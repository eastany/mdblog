## http server 
```
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
实现ServeHTTP方法即可对http请求指定路径做出处理响应。
ServeHTTP需要写应答头并将数据写到ResponseWriter返回，并信号通知server已经完成处理，可以转移到该连接的下一个请求上。
如果ServeHTTP出现了panic，则server认为当前请求受影响。随后会recover，记录堆栈日志，挂起该连接。

```
type ResponseWriter interface {
	Header() Header
	Write([]byte) (int, error)
	WriteHeader(int)
}
```
ResponseWriter是handler用来实现应答的组装。
Header 是header信息的map，作为WriteHeader的数据源。所以调用顺序是Header()->WriteHeader()
Write 则是将应答的data写入到conn中，如果WriteHeader没调用的话，先调用WriteHeader，再写数据。还有就是Content-Type没有设置的话，采用默认的规则去匹配，设置该值。
WriteHeader 经常用来返回错误码。一般没有被显式调用的话，第一次调用Write的时候会先调用它

```
type Hijacker interface {
	Hijack() (net.Conn, *bufio.ReadWriter, error)
}
```
Hijacker 连接劫持。每个连接过来后，被劫持交给handler处理。ResponseWriter实现了该接口，用来实现HTTP的应答。

```
type conn struct {
	remoteAddr string               // network address of remote side
	server     *Server              // the Server on which the connection arrived
	rwc        net.Conn             // i/o connection
	w          io.Writer            // checkConnErrorWriter's copy of wrc, not zeroed on Hijack
	werr       error                // any errors writing to w
	sr         liveSwitchReader     // where the LimitReader reads from; usually the rwc
	lr         *io.LimitedReader    // io.LimitReader(sr)
	buf        *bufio.ReadWriter    // buffered(lr,rwc), reading from bufio->limitReader->sr->rwc
	tlsState   *tls.ConnectionState // or nil when not using TLS
 
	mu           sync.Mutex // guards the following
	clientGone   bool       // if client has disconnected mid-request
	closeNotifyc chan bool  // made lazily
	hijackedv    bool       // connection has been hijacked by handler
}
```
### conn
作为HTTP请求的连接定义，具体的属性如上

__func (c *conn) hijack() (rwc net.Conn, buf *bufio.ReadWriter, err error)__   是链接劫持的具体实现，把Conn跟对应的buf导出，原conn相应的置空，调用者则接管了该连接。

__func (c *conn) closeNotify() <-chan bool__   
关闭通知，标记客户端已断开连接。具体实现为：复制一些未处理的数据，做收尾工作，

__func (c *conn) noteClientGone()__
标记客户端已断开。向close chan发送断开信号

```
type switchWriter struct {
	io.Writer
}
```
switchWriter可以在运行时改变Writer，所以并发写入并不安全

```
type liveSwitchReader struct {
	sync.Mutex
	r io.Reader
}
```
liveSwitchReader可以在运行时改变Reader，如果持有锁的话，那么并发读就是安全的

```
type response struct {
	conn          *conn
	req           *Request // request for this response
	wroteHeader   bool     // reply header has been (logically) written
	wroteContinue bool     // 100 Continue response was written
 
	w  *bufio.Writer // buffers output in chunks to chunkWriter
	cw chunkWriter
	sw *switchWriter // of the bufio.Writer, for return to putBufioWriter
 
	// handlerHeader is the Header that Handlers get access to,
	// which may be retained and mutated even after WriteHeader.
	// handlerHeader is copied into cw.header at WriteHeader
	// time, and privately mutated thereafter.
	handlerHeader Header
	calledHeader  bool // handler accessed handlerHeader via Header
 
	written       int64 // number of bytes written in body
	contentLength int64 // explicitly-declared Content-Length; or -1
	status        int   // status code passed to WriteHeader
	closeAfterReply bool
	requestBodyLimitHit bool
	trailers []string
 
	handlerDone bool // set true when the handler exits
 
	// Buffers for Date and Content-Length
	dateBuf [len(TimeFormat)]byte
	clenBuf [10]byte
}
```
注释写的比较清楚，作为HTTP的响应。

```
// Read next request from connection.
func (c *conn) readRequest() (w *response, err error) {
	if c.hijacked() {	//如果已被劫持处理，则返回，出错
		return nil, ErrHijacked
	}
 
	if d := c.server.ReadTimeout; d != 0 {	//设置读超时
		c.rwc.SetReadDeadline(time.Now().Add(d))
	}
	if d := c.server.WriteTimeout; d != 0 {
		defer func() {			 //用defer设置写超时，比较准确
			c.rwc.SetWriteDeadline(time.Now().Add(d))
		}()
	}
 
	c.lr.N = c.server.initialLimitedReaderSize()	// 初始化读的缓冲区长度
	var req *Request
	if req, err = ReadRequest(c.buf.Reader); err != nil {//读取请求
		if c.lr.N == 0 {
			return nil, errTooLarge
		}
		return nil, err
	}
	c.lr.N = noLimit  
 
	req.RemoteAddr = c.remoteAddr
	req.TLS = c.tlsState
 
	w = &response{
		conn:          c,
		req:           req,
		handlerHeader: make(Header),
		contentLength: -1,
	} //实例化响应对象
	w.cw.res = w
	w.w = newBufioWriterSize(&w.cw, bufferBeforeChunkingSize)
	return w, nil
}

func (w *response) Header() Header {
	if w.cw.header == nil && w.wroteHeader && !w.cw.wroteHeader {
		w.cw.header = w.handlerHeader.clone()
	}
	w.calledHeader = true
	return w.handlerHeader
}
```


```
func (w *response) WriteHeader(code int) {
	if w.conn.hijacked() {
		w.conn.server.logf("http: response.WriteHeader on hijacked connection")
		return
	}
	if w.wroteHeader {
		w.conn.server.logf("http: multiple response.WriteHeader calls")
		return
	}
	w.wroteHeader = true
	w.status = code
 
	if w.calledHeader && w.cw.header == nil {
		w.cw.header = w.handlerHeader.clone()
	}
 
	if cl := w.handlerHeader.get("Content-Length"); cl != "" {
		v, err := strconv.ParseInt(cl, 10, 64)
		if err == nil && v >= 0 {
			w.contentLength = v
		} else {
			w.conn.server.logf("http: invalid Content-Length of %q", cl)
			w.handlerHeader.Del("Content-Length")
		}
	}
}
```


```
const rstAvoidanceDelay = 500 * time.Millisecond
```
rstAvoidanceDelay是在关闭socket之前关闭连接的写之后需要等待的时间，可以让客户端收到FIN报文，并且可以处理收到RST请求报文时的未处理数据。RST在BSD系统中出现

```
// Serve a new connection.
func (c *conn) serve() {
	origConn := c.rwc // copy it before it's set nil on Close or Hijack
	defer func() {
		if err := recover(); err != nil {   //捕获异常
			const size = 64 << 10
			buf := make([]byte, size)
			buf = buf[:runtime.Stack(buf, false)]    //保存当前goroutine的堆栈信息
			c.server.logf("http: panic serving %v: %v\n%s", c.remoteAddr, err, buf)
		}
		if !c.hijacked() { //如果没有被接管，则关闭链接做处理
			c.close()
			c.setState(origConn, StateClosed)
		}
	}()
 
	if tlsConn, ok := c.rwc.(*tls.Conn); ok {
		//如果时HTTPS的链接
		if d := c.server.ReadTimeout; d != 0 {
			c.rwc.SetReadDeadline(time.Now().Add(d))
		}
		if d := c.server.WriteTimeout; d != 0 {
			c.rwc.SetWriteDeadline(time.Now().Add(d))
		}
		if err := tlsConn.Handshake(); err != nil {
			c.server.logf("http: TLS handshake error from %s: %v", c.rwc.RemoteAddr(), err)
			return
		}
		c.tlsState = new(tls.ConnectionState)
		*c.tlsState = tlsConn.ConnectionState()
		if proto := c.tlsState.NegotiatedProtocol; validNPN(proto) {
			if fn := c.server.TLSNextProto[proto]; fn != nil {
				h := initNPNRequest{tlsConn, serverHandler{c.server}}
				fn(c.server, tlsConn, h)
			}
			return
		}
	}
 
	for {	//主循环体，负责接收请求
		w, err := c.readRequest()  //收到请求w
		if c.lr.N != c.server.initialLimitedReaderSize() {
			// If we read any bytes off the wire, we're active.
			c.setState(c.rwc, StateActive)
		}
		if err != nil {
			//接收请求出错
			if err == errTooLarge {  //请求过大，直接返回413
				// Their HTTP client may or may not be
				// able to read this if we're
				// responding to them and hanging up
				// while they're still writing their
				// request.  Undefined behavior.
				io.WriteString(c.rwc, "HTTP/1.1 413 Request Entity Too Large\r\n\r\n")
				c.closeWriteAndWait()
				break
			} else if err == io.EOF { //结束字符
				break // Don't reply
			} else if neterr, ok := err.(net.Error); ok && neterr.Timeout() {
				break // Don't reply
			}
			io.WriteString(c.rwc, "HTTP/1.1 400 Bad Request\r\n\r\n")
			break
		}
 
		// 支持100-continue
		req := w.req
		if req.expectsContinue() { //
			if req.ProtoAtLeast(1, 1) && req.ContentLength != 0 {
				// Wrap the Body reader with one that replies on the connection
				req.Body = &expectContinueReader{readCloser: req.Body, resp: w}
			}
			req.Header.Del("Expect")
		} else if req.Header.get("Expect") != "" {
			w.sendExpectationFailed()
			break
		}
 
		//把请求处理丢到goroutine中，并行处理, 所以这段函数调用是在goroutine中，参见Serve
		serverHandler{c.server}.ServeHTTP(w, w.req)
		if c.hijacked() {
			return
		}
		w.finishRequest()
		if w.closeAfterReply {
			if w.requestBodyLimitHit {
				c.closeWriteAndWait()
			}
			break
		}
		c.setState(c.rwc, StateIdle)
	}
}
```


-----------
```
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	hosts bool // whether any patterns contain hostnames
}
type muxEntry struct {
	explicit bool
	h        Handler
	pattern  string
}
```
ServeMux是HTTP链接复用器，也就是路由表，定义请求的URL与处理handler的映射，长匹配优先级高于短匹配，

下面这个函数做匹配
```
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
	var n = 0
	for k, v := range mux.m {
		if !pathMatch(k, path) {
			continue
		}
		if h == nil || len(k) > n {
			n = len(k)
			h = v.h
			pattern = v.pattern
		}
	}
	return
}
```

HTTP Server
====
```
type Server struct {
	Addr           string   
	Handler        Handler      
	ReadTimeout    time.Duration
	WriteTimeout   time.Duration 
	MaxHeaderBytes int           //头最大长度
	TLSConfig      *tls.Config   //https配置
 
	
	TLSNextProto map[string]func(*Server, *tls.Conn, Handler)
	ConnState func(net.Conn, ConnState)   //一个很常用的编码规范，可以借鉴。设置连接状态
	ErrorLog *log.Logger
	disableKeepAlives int32 // accessed atomically.
}

func (srv *Server) ListenAndServe() error {
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)   //开始监听
	if err != nil {
		return err
	}
	return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})  //调用Serve函数,看下面
}

func (srv *Server) Serve(l net.Listener) error {
	defer l.Close()
	var tempDelay time.Duration // accept失败后sleep的时间
	for {    //负责收连接，然后丢到协程中即可
		rw, e := l.Accept()  //accept到请求
		if e != nil {
			if ne, ok := e.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return e
		}
		tempDelay = 0  //设置等待时间为0
		c, err := srv.newConn(rw)   //新建连接对象
		if err != nil {
			continue
		}
		c.setState(c.rwc, StateNew) // before Serve can return
		go c.serve()  //丢到协程中去处理
	}
}
```
