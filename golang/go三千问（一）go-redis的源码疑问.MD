#go三千问（一）go-redis的源码疑问

今天在看go-redis源码时碰到这样一段代码：
```go
//go-redis v7.0.0-beta4  commands.go
args := make([]interface{}, 3, 4)
	args[0] = "set"
	args[1] = key
	args[2] = value
	if expiration > 0 {
		if usePrecise(expiration) {
			args = append(args, "px", formatMs(expiration))
		} else {
			args = append(args, "ex", formatSec(expiration))
		}
	}
```
让我感觉奇怪的是，args在make时被指定为length=3，capacity=4；在赋值了前三个元素后，明明在之后的判断中会被添加2个元素，按道理make的capacity
应该传5才对啊，这里为什么要传4？不会引起扩容降低性能吗？带着这个疑问，我做了以下实验：
```go
func TestSliceCap(t *testing.T) {
	a := make([]interface{}, 3, 4)
	fmt.Println(cap(a))
	a = append(a,"da",int64(44))
	fmt.Println(cap(a))
}
```
打印结果为：
```
4
8
```
很明显发生了扩容，而这次扩容明显是不必要的，因此可以视为一个bug。

目前我已经提交PR到官方仓库:https://github.com/go-redis/redis/pull/1204