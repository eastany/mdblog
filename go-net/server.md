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
