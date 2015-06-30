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

func (b *Reader) fill() {
	// Slide existing data to beginning.
	if b.r > 0 {
		copy(b.buf, b.buf[b.r:b.w])  //把当前写入的数据 移到开始
		b.w -= b.r
		b.r = 0
	}
 
	if b.w >= len(b.buf) {  //如果缓冲区大小小于写入偏移
		panic("bufio: tried to fill full buffer")
	}
 
	// Read new data: try a limited number of times.
	for i := maxConsecutiveEmptyReads; i > 0; i-- {
		n, err := b.rd.Read(b.buf[b.w:])   //继续读入数据
		if n < 0 {
			panic(errNegativeRead)
		}
		b.w += n
		if err != nil {
			b.err = err
			return
		}
		if n > 0 {
			return
		}
	}
	b.err = io.ErrNoProgress
}
```

func (b *Reader) Peek(n int) ([]byte, error)  
