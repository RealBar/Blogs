#go三千问（二）何为"hot path"？

###描述
go version 1.13.4的源码中有这样一段
```go
//sync/once.go
type Once struct {
	// done indicates whether the action has been performed.
	// It is first in the struct because it is used in the hot path.
	// The hot path is inlined at every call site.
	// Placing done first allows more compact instructions on some architectures (amd64/x86),
	// and fewer instructions (to calculate offset) on other architectures.
	done uint32
	m    Mutex
}
```
这里提到了"hot path"，究竟是何方神圣呢？目前基本没有中文资料，google上搜英文，也很少有资料