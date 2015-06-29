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

