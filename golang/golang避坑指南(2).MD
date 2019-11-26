# golang避坑指南(2)goroutine之踩坑实录
### goroutine中直接引用外部变量
如果在goroutine中的匿名函数直接引用了外部变量，例如：
```go
func TestGo(t *testing.T) {
	c := make(chan struct{},1)
	m := 0
	a := func() {
		time.Sleep(time.Second)
		fmt.Println(m)
		c <- struct{}{}
	}
	go a()
	m += 1000
	<- c
}
```
你认为输出结果会是什么？

按照一般理解，m是一个int类型的局部变量，要想被别的函数引用，只能传值。因此在匿名函数内部引用的应该是函数定义时传递的值，所以应该是0？

如果你是这么想的，那么恭喜你掉坑了。不信？运行一下看看：
```
1000
```
居然是1000？难道匿名函数中对于外部的局部变量，包括值类型都是引用传递的吗？我们不妨再试一试：
```
func TestGo(t *testing.T) {
	m := 0
	f := func() {
		m += 1000
	}
	f()
	fmt.Println(m)
}
```
在这个程序中，我们定义一个局部变量m，然后在定义的匿名函数中直接对m进行赋值操作，然后再调用这个无参函数，打印一下m的值，我们发现输出的结果是：
```
1000
```
