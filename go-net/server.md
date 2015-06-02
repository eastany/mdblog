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

__func (c *conn) hijack() (rwc net.Conn, buf *bufio.ReadWriter, err error)__ 是链接劫持的具体实现，把Conn跟对应的buf导出，原conn响应的置空，调用者则接管了该连接。



