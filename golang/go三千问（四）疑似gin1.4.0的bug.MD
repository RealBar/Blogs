#go三千问（四）疑似gin1.4.0的bug

###描述
配置如下response
```go
type UniversalResponse struct {
	Code int         `json:"code"`
	Msg  string      `json:"msg"`
	Data interface{} `json:"data,omitempty"`
}
```
但是使用gin.context.JSON写入上述返回值时，收到的body里有"data":null
