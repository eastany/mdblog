HTTP Transfer
===========
transferWriter 封装了请求和应答的写操作，检查属性，其定义如下
```
type transferWriter struct {
	Method           string
	Body             io.Reader
	BodyCloser       io.Closer
	ResponseToHEAD   bool
	ContentLength    int64 // -1 means unknown, 0 means exactly none
	Close            bool
	TransferEncoding []string
	Trailer          Header
}
```
生成transferWriter对象使用
```
func newTransferWriter(r interface{}) (t *transferWriter, err error) {
	t = &transferWriter{}   //实例化返回对象
 
	// Extract relevant fields
	atLeastHTTP11 := false
	switch rr := r.(type) {
	case *Request://请求类
		if rr.ContentLength != 0 && rr.Body == nil {   //检查内容长度与body是否匹配
			return nil, fmt.Errorf("http: Request.ContentLength=%d with nil Body", rr.ContentLength)
		}
		t.Method = rr.Method
		t.Body = rr.Body
		t.BodyCloser = rr.Body
		t.ContentLength = rr.ContentLength
		t.Close = rr.Close
		t.TransferEncoding = rr.TransferEncoding
		t.Trailer = rr.Trailer
		atLeastHTTP11 = rr.ProtoAtLeast(1, 1)
		if t.Body != nil && len(t.TransferEncoding) == 0 && atLeastHTTP11 { //如果body不为空，传输编码未指定
			if t.ContentLength == 0 {
				//如果内容长度为0
				var buf [1]byte
				n, rerr := io.ReadFull(t.Body, buf[:])
				if rerr != nil && rerr != io.EOF {
					t.ContentLength = -1
					t.Body = &errorReader{rerr}
				} else if n == 1 {
					// Oh, guess there is data in this Body Reader after all.
					// The ContentLength field just wasn't set.
					// Stich the Body back together again, re-attaching our
					// consumed byte.
					t.ContentLength = -1
					t.Body = io.MultiReader(bytes.NewReader(buf[:]), t.Body)
				} else {
					// Body is actually empty.
					t.Body = nil
					t.BodyCloser = nil
				}
			}
			if t.ContentLength < 0 {
			  //数据分片
				t.TransferEncoding = []string{"chunked"}
			}
		}
	case *Response:
		if rr.Request != nil {
			t.Method = rr.Request.Method
		}
		t.Body = rr.Body
		t.BodyCloser = rr.Body
		t.ContentLength = rr.ContentLength
		t.Close = rr.Close
		t.TransferEncoding = rr.TransferEncoding
		t.Trailer = rr.Trailer
		atLeastHTTP11 = rr.ProtoAtLeast(1, 1)
		t.ResponseToHEAD = noBodyExpected(t.Method)
	}
 
	// Sanitize Body,ContentLength,TransferEncoding
	if t.ResponseToHEAD {
		t.Body = nil
		if chunked(t.TransferEncoding) {
			t.ContentLength = -1
		}
	} else {
		if !atLeastHTTP11 || t.Body == nil {
			t.TransferEncoding = nil
		}
		if chunked(t.TransferEncoding) {
			t.ContentLength = -1
		} else if t.Body == nil { // no chunking, no body
			t.ContentLength = 0
		}
	}
 
	// Sanitize Trailer
	if !chunked(t.TransferEncoding) {
		t.Trailer = nil
	}
 
	return t, nil
}
```
TransferWriter 写Header
```
func (t *transferWriter) WriteHeader(w io.Writer) error {
	if t.Close {
		if _, err := io.WriteString(w, "Connection: close\r\n"); err != nil {
			return err
		}
	}
	if t.shouldSendContentLength() {
		if _, err := io.WriteString(w, "Content-Length: "); err != nil {
			return err
		}
		if _, err := io.WriteString(w, strconv.FormatInt(t.ContentLength, 10)+"\r\n"); err != nil {
			return err
		}
	} else if chunked(t.TransferEncoding) {
		//分片
		if _, err := io.WriteString(w, "Transfer-Encoding: chunked\r\n"); err != nil {
			return err
		}
	}
 
	// Write Trailer header
	if t.Trailer != nil {
		keys := make([]string, 0, len(t.Trailer))
		for k := range t.Trailer {
			k = CanonicalHeaderKey(k)
			switch k {
			case "Transfer-Encoding", "Trailer", "Content-Length":
				return &badStringError{"invalid Trailer key", k}
			}
			keys = append(keys, k)
		}
		if len(keys) > 0 {
			sort.Strings(keys)
			// TODO: could do better allocation-wise here, but trailers are rare,
			// so being lazy for now.
			if _, err := io.WriteString(w, "Trailer: "+strings.Join(keys, ",")+"\r\n"); err != nil {
				return err
			}
		}
	}
 
	return nil
}
```
