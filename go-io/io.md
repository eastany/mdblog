Reader  Writer  multiReader multiWriter
=============
IO的基础接口
```
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}

type Closer interface {
	Close() error
}

type Seeker interface {
	Seek(offset int64, whence int) (int64, error)
}
```

上述基础的接口可以进行组合，如：
```
type ReadWriter interface {
	Reader
	Writer
}

type ReadCloser interface {
	Reader
	Closer
}

type ReadSeeker interface {
	Reader
	Seeker
}
```
...等等
对读写接口的二次封装接口
```
type ReaderFrom interface {
	ReadFrom(r Reader) (n int64, err error)
}
type WriterTo interface {
	WriteTo(w Writer) (n int64, err error)
}
type ReaderAt interface {
	ReadAt(p []byte, off int64) (n int, err error)
}
type WriterAt interface {
	WriteAt(p []byte, off int64) (n int, err error)
}
```
下面几个主要的函数的介绍

ReadAtLeast 至少读取min个字节
------------------------------
```
func ReadAtLeast(r Reader, buf []byte, min int) (n int, err error) {
	if len(buf) < min {
		//缓存buf长度不够
		return 0, ErrShortBuffer
	}
	for n < min && err == nil {
		//循环读取，直到读取的字节数不小于min或者nil不为空时结束
		var nn int
		nn, err = r.Read(buf[n:])  //继续接着读取
		n += nn
	}
	if n >= min {
		err = nil
	} else if n > 0 && err == EOF {
		err = ErrUnexpectedEOF
	}
	return
}

func ReadFull(r Reader, buf []byte) (n int, err error) {
	return ReadAtLeast(r, buf, len(buf))
}
```

CopyN 从src复制指定字节数到dst
-----------
```
func CopyN(dst Writer, src Reader, n int64) (written int64, err error) {
	written, err = Copy(dst, LimitReader(src, n))
	if written == n {
		return n, nil
	}
	if written < n && err == nil {
		// src stopped early; must have been EOF.
		err = EOF
	}
	return
}

func Copy(dst Writer, src Reader) (written int64, err error) {
	//如果src有WriteTo函数，直接转换，避免分配空间和内存copy
	if wt, ok := src.(WriterTo); ok {
		return wt.WriteTo(dst)
	}
	// 如果dst有ReadFrom函数，直接转换，避免分配空间和内存copy
	if rt, ok := dst.(ReaderFrom); ok {
		return rt.ReadFrom(src)
	}
	buf := make([]byte, 32*1024)
	for {
		nr, er := src.Read(buf)
		if nr > 0 { //读到了数据，往dst中写入
			nw, ew := dst.Write(buf[0:nr])
			if nw > 0 {
				//写入字节数计数
				written += int64(nw)
			}
			
			//如果写入出错，退出
			if ew != nil {
				
				err = ew
				break
			}
			if nr != nw { //读写字节数不一致
				err = ErrShortWrite
				break
			}
		}
		if er == EOF { //读到EOF
			break
		}
		if er != nil {
			err = er
			break
		}
	}
	return written, err
}
```

Seek(off,off_type) off_type=0是从开始算起偏移，1是从你当前算偏移，2是从结尾开始算偏移。

PIPE
--------
```
type pipe struct {
	rl    sync.Mutex // gates readers one at a time
	wl    sync.Mutex // gates writers one at a time
	l     sync.Mutex // protects remaining fields
	data  []byte     // data remaining in pending write
	rwait sync.Cond  // waiting reader
	wwait sync.Cond  // waiting writer
	rerr  error      // if reader closed, error to give writes
	werr  error      // if writer closed, error to give reads
}

func (p *pipe) read(b []byte) (n int, err error) {
	// One reader at a time.
	p.rl.Lock()
	defer p.rl.Unlock()
 
	p.l.Lock()
	defer p.l.Unlock()
	for {   //阻塞等待
		if p.rerr != nil {
			return 0, ErrClosedPipe
		}
		if p.data != nil {  //有数据来了，跳出循环
			break
		}
		if p.werr != nil {
			return 0, p.werr
		}
		p.rwait.Wait() 
	}
	n = copy(b, p.data)
	p.data = p.data[n:]
	if len(p.data) == 0 {
		p.data = nil
		p.wwait.Signal() //通知写端
	}
	return
}
```
func (p *pipe) write(b []byte) (n int, err error) 同read

PipeReader是对pipe的包装，实现了Reader接口
PipeWriter是对pipe的包装，实现了Writer接口

```
func Pipe() (*PipeReader, *PipeWriter) {
	p := new(pipe)
	p.rwait.L = &p.l
	p.wwait.L = &p.l
	r := &PipeReader{p}
	w := &PipeWriter{p}
	return r, w
}
```
因为pipe是带锁的所以支持安全的并发读写。

MultiReader是对多个Reader的逻辑连接，读取的时候按顺序读取
```
func MultiReader(readers ...Reader) Reader {
	r := make([]Reader, len(readers))
	copy(r, readers)
	return &multiReader{r}
}
```

DEVNULL  io.util.Discard是/dev/null
-------
```
var Discard io.Writer = devNull(0)
```
