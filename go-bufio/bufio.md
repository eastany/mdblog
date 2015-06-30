Reader
--------
```
type Reader struct {
	buf          []byte
	rd           io.Reader // reader provided by the client
	r, w         int       //读写的位置
	err          error
	lastByte     int
	lastRuneSize int
}

func NewReaderSize(rd io.Reader, size int) *Reader {
	b, ok := rd.(*Reader) //类型断言rd是否是Reader
	if ok && len(b.buf) >= size { //如果是Reader且buf大小大size，则返回rd
		return b
	}
	if size < minReadBufferSize {
		size = minReadBufferSize
	}
	r := new(Reader)
	r.reset(make([]byte, size), rd) //初始化Reader
	return r
}

func NewReader(rd io.Reader) *Reader {
	return NewReaderSize(rd, defaultBufSize)   //defaultBufSize=4096
}




```
