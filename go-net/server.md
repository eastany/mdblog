```
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
实现ServeHTTP方法即可对http请求指定路径做出处理响应。
