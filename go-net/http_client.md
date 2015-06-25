HTTP Client
=======
http client的封装实现
```
type Client struct {
	Transport RoundTripper    //
	CheckRedirect func(req *Request, via []*Request) error   //对重定向请求的处理策略定义，
	//默认是重定向的请求正常进行，重定向最多10次
	Jar CookieJar         //cookie管理器，如果为空则忽略cookie
	Timeout time.Duration //超时时间，从发起请求开始计时
}
```
RoundTripper为请求接口,具体实现函数是RoundTrip，

```
type Transport struct {
	idleMu     sync.Mutex
	wantIdle   bool // user has requested to close all idle conns
	idleConn   map[connectMethodKey][]*persistConn
	idleConnCh map[connectMethodKey]chan *persistConn
 
	reqMu       sync.Mutex
	reqCanceler map[*Request]func()
 
	altMu    sync.RWMutex
	altProto map[string]RoundTripper // nil or map of URI scheme => RoundTripper
	Proxy func(*Request) (*url.URL, error)
 
	//定义非加密链接
	Dial func(network, addr string) (net.Conn, error)
 
	//定义加密链接
	DialTLS func(network, addr string) (net.Conn, error)
 
	//加密配置
	TLSClientConfig *tls.Config
 
	//TLS握手超时时间
	TLSHandshakeTimeout time.Duration
 
	//true表示在不同的request之间链接不重用
	DisableKeepAlives bool
 
	DisableCompression bool
	MaxIdleConnsPerHost int
 
	// ResponseHeaderTimeout＝写入所有request的数据后，等待服务端返回头信息，不包含读取应答body
	ResponseHeaderTimeout time.Duration
 
	// TODO: tunable on global max cached connections
	// TODO: tunable on timeout on cached connections
}
```

针对HTTP Client实现，Transport实现了RoundTripper接口

```
func (t *Transport) RoundTrip(req *Request) (resp *Response, err error) {
	if req.URL == nil {
		req.closeBody()
		return nil, errors.New("http: nil Request.URL")
	}
	if req.Header == nil {
		req.closeBody()
		return nil, errors.New("http: nil Request.Header")
	}
	if req.URL.Scheme != "http" && req.URL.Scheme != "https" {
	  //自定义的协议
		t.altMu.RLock()
		var rt RoundTripper
		if t.altProto != nil {
			rt = t.altProto[req.URL.Scheme]
		}
		t.altMu.RUnlock()
		if rt == nil {
			req.closeBody()
			return nil, &badStringError{"unsupported protocol scheme", req.URL.Scheme}
		}
		return rt.RoundTrip(req)
	}
	if req.URL.Host == "" {
		req.closeBody()
		return nil, errors.New("http: no Host in request URL")
	}
	treq := &transportRequest{Request: req}
	cm, err := t.connectMethodForRequest(treq)
	if err != nil {
		req.closeBody()
		return nil, err
	}
 
	// 从缓存中获取连接
	pconn, err := t.getConn(req, cm)
	if err != nil {
		t.setReqCanceler(req, nil)  //失败后，设置取消回调函数
		req.closeBody()
		return nil, err
	}
 
	return pconn.roundTrip(treq) //调用连接的处理函数，如下
}

func (pc *persistConn) roundTrip(req *transportRequest) (resp *Response, err error) {
	pc.t.setReqCanceler(req.Request, pc.cancelRequest) //设置取消回调
	pc.lk.Lock()  //计数增加
	pc.numExpectedResponses++
	headerFn := pc.mutateHeaderFunc
	pc.lk.Unlock()
 
	if headerFn != nil {  //如果头处理函数不为空，则处理头
		headerFn(req.extraHeaders())
	}
	requestedGzip := false   //GZIP压缩处理逻辑
	if !pc.t.DisableCompression &&
		req.Header.Get("Accept-Encoding") == "" &&
		req.Header.Get("Range") == "" &&
		req.Method != "HEAD" {   //如果启用压缩，且请求头中无Accept-Encoding和Range声明，请求方法不是HEAD，则默认开启GZIP
		requestedGzip = true
		req.extraHeaders().Set("Accept-Encoding", "gzip")
	}
 
	if pc.t.DisableKeepAlives { //如果未启用场链接，则设置链接close
		req.extraHeaders().Set("Connection", "close")
	}
 
	// 边写边等待应答，避免服务器在读完请求之前就返回.这里可以学习学习实现
	writeErrCh := make(chan error, 1)
	pc.writech <- writeRequest{req, writeErrCh}
 
	resc := make(chan responseAndError, 1)
	pc.reqch <- requestAndChan{req.Request, resc, requestedGzip}
 
	var re responseAndError
	var pconnDeadCh = pc.closech
	var failTicker <-chan time.Time
	var respHeaderTimer <-chan time.Time
WaitResponse:
	for {
		select {
		case err := <-writeErrCh:
			if err != nil {
				re = responseAndError{nil, err}
				pc.close()
				break WaitResponse
			}
			if d := pc.t.ResponseHeaderTimeout; d > 0 {
				timer := time.NewTimer(d)
				defer timer.Stop() // prevent leaks
				respHeaderTimer = timer.C
			}
		case <-pconnDeadCh:
			// The persist connection is dead. This shouldn't
			// usually happen (only with Connection: close responses
			// with no response bodies), but if it does happen it
			// means either a) the remote server hung up on us
			// prematurely, or b) the readLoop sent us a response &
			// closed its closech at roughly the same time, and we
			// selected this case first, in which case a response
			// might still be coming soon.
			//
			// We can't avoid the select race in b) by using a unbuffered
			// resc channel instead, because then goroutines can
			// leak if we exit due to other errors.
			pconnDeadCh = nil                               // avoid spinning
			failTicker = time.After(100 * time.Millisecond) // arbitrary time to wait for resc
		case <-failTicker:
			re = responseAndError{err: errClosed}
			break WaitResponse
		case <-respHeaderTimer:
			pc.close()
			re = responseAndError{err: errTimeout}
			break WaitResponse
		case re = <-resc:
			break WaitResponse
		}
	}
 
	pc.lk.Lock()
	pc.numExpectedResponses--
	pc.lk.Unlock()
 
	if re.err != nil {
		pc.t.setReqCanceler(req.Request, nil)
	}
	return re.res, re.err
}
```

