#go三千问（二）何为"hot path"？

###描述
context/context.go有如下代码
```go
// closedchan is a reusable closed channel.
var closedchan = make(chan struct{})

func init() {
	close(closedchan)
}
```
