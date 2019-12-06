# go标准库源码学习（一）短小精悍的Once
## 概述
Once在标准库中的功能被描述如下：

>Once is an object that will perform exactly one action.

即保证某个动作只执行一次。这很好理解，延迟初始化、单例（懒汉式）就是这种场景。下面用Once实现一个懒汉式的单例：
```go
type Model struct {}

var instance *Model
var once sync.Once

func GetModel() *Model {
	once.Do(func() {
		instance = &Model{}
	})
	return instance
}
```
值得注意的是，Once.Do被设计成严格保证传入的函数只执行一次，且当Once.Do返回时，保证抢占Once执行权成功的goroutine传入Do的函数已经执行完成。
这意味着如果两个goroutine同时首次执行Once.Do，肯定只有一个能执行，那么另一个必须等待直到它执行完成之后方能返回。是不是有点双重检测锁(DCL)的味道了？
##源码
Once的源码异常简单，纯代码只有不到20行，作为go语言极为常用和基础的同步工具，当真对得起"短小精悍"这个形容。这里将代码全部贴出
```go
type Once struct {
	done uint32
	m    Mutex
}
func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 {
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```
可以看到它内部确实实现了双重检测锁。

这里有几个值得注意的点：
+ Do方法中的第一层检测` atomic.LoadUint32(&o.done) == 0`是否应该用CAS来实现？
+ 如果不是，为什么不直用`o.done == 0`（就像java单例的双重检测锁那样）？
+ doSlow方法的临界区内，为什么第二层检测时直接读取done，而写值却用了atomic操作？

##源码分析
首先第一个问题，其实在源码的注释中已经解释了：
```go
	// Note: Here is an incorrect implementation of Do:
	//
	//	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	//		f()
	//	}
	//
	// Do guarantees that when it returns, f has finished.
	// This implementation would not implement that guarantee:
	// given two simultaneous calls, the winner of the cas would
	// call f, and the second would return immediately, without
	// waiting for the first's call to f to complete.
	// This is why the slow path falls back to a mutex, and why
	// the atomic.StoreUint32 must be delayed until after f returns.
```
前文提到了，Once需要保证当Once.Do返回时，抢占Once执行权成功的那个goroutine传入Do的函数已经执行完成。因此这里并不能使用CAS，因为在并发执行时，
CAS抢夺失败的那个goroutine会直接返回，而此时有可能抢夺成功的那个还没执行完成，所以不能达到设计目标。因此源代码中使用
`atomic.LoadUint32(&o.done) == 0`和`o.done == 0`结合互斥锁实现了双重检测锁，将同时执行的goroutine阻塞在Mutex处直到抢夺成功的函数执行完成。

再来看第二个问题。

大家应该还记得，java版本的双重检测锁中，我们需要在检测变量前加个volatile保证可见性:
```java
public static volatile Object instance;
```
同样的问题go也会有，但go并没有`可见性`的概念。事实上根据[go内存模型官方文档](https://golang.org/ref/mem)，虽然go没有JMM的工作内存和主内存之类的区分，
但它确实有自己的一套观测(observe)规则和Happens before规则。关于"可观测"和"可见"，我认为是两个不同的维度：可见性是针对变量来描述的——
你可以说变量a具有可见性；而可观测是针对读写操作来描述的——你只能说对变量a的某个写操作能保证被某个它的读操作观测到。

内存模型文档中提到：
>When multiple goroutines access a shared variable v, they must use synchronization events to establish happens-before
> conditions that ensure reads observe the desired writes.

因此如果不做同步而直接用`o.done == 0`的话肯定无法观测到doSlow方法中设置的done值。

但是go并没有给我们提供`volatile`这个关键字，那我们该如何保证可观测呢？我以前写过一片文章提到过这个问题[为什么golang没有volatile?](https://github.com/RealBar/Blogs/blob/master/golang/%E4%B8%BA%E4%BB%80%E4%B9%88golang%E6%B2%A1%E6%9C%89volatile%3F.MD)
文中提到过，go鼓励开发者"通过通信去共享内存，而不是通过共享内存去通信"。但是目前看来，这个观念的转换过程并不是那么顺利——至少在这里，1.13.4的标准库源码中Once.go的源码中还在使用共享变量进行通信……

言归正传，Once.done是一个共享变量，那么怎么保证其并发可观测呢？办法有很多，加锁可以，但是基于原子操作的atomic.Load*/Store*是一个更轻量且高效的选择。
至于为什么atomic操作能保证可观测将会在下一小节说明。

第三个问题就比较值得玩味了。

`临界区(Critical Section)`指的是并发场景下只能有一个线程/协程执行的代码区域。在go中，临界区是Mutex上锁和解锁的中间执行区：
```go
fun xxx(){
    mutex.lock()
    defer mutex.unlock()
    x=10
}
```
这里`x=10`就处于临界区内。那么，它能否保证可观测呢？——或者说，能否保证另一个goroutine的atomic.Load*读取到这里设置的值呢？

答案是不能保证，至少在官方的内存模型文档中没有给出保证。准确来说，go的内存模型只列举了6种同步机制（atomic操作不在其中，下一小节会解释原因）：
- Initialization
- Goroutine creation
- Goroutine destruction
- Channel communication
- Locks
- Once
其中对Locks的Happens before规则描述为：
>For any sync.Mutex or sync.RWMutex variable l and n < m, call n of l.Unlock() happens before call m of l.Lock() returns.

这里m，n指的是调用函数的实际时间点。注意这里只规定了unlock先于lock发生，因此只能保证这两个操作之间的可观测。而如果在临界区外部对变量进行读写则
无法保证能被观测到，正如Once中的情形一样。这也就解释了为什么doSlow中写done时即使是在临界区内也要使用atomic.StoreUint32进行写操作。

至于doSlow中第二重检测时为什么是直接读取done而没有用atomic操作，因为这个读操作和临界区内的写操作均处于临界区内，已经保证了对写操作的可观测性，自然不需要atomic了。

## atomic操作真能保证可观测吗？
既然官方的内存模型中并没有提到atomic操作的可观测性，那它真能保证可观测吗？

我们试着打开atomic的文档，第二段话就解释了为什么内存模型文档中没有提到它：
>These functions require great care to be used correctly. Except for special, low-level applications, synchronization is 
>better done with channels or the facilities of the sync package. 

意思很明显——"atomic函数是给底层、特殊应用使用的，同步应该使用channel或者sync包下的工具"——就是让你最好别用呗。

说了这么多，它到底能保证可观测性吗？上源码，以Store为例，函数声明在这里runtime/internal/atomic/atomic_amd64x.go
```go
func Store(ptr *uint32, val uint32)
```
实现在runtime/internal/atomic/asm_amd64.s
```
TEXT runtime∕internal∕atomic·Store(SB), NOSPLIT, $0-12
	MOVQ	ptr+0(FP), BX
	MOVL	val+8(FP), AX
	XCHGL	AX, 0(BX)
	RET
```
注意到，Store的实现中使用了一个交换指令XCHGL，在x86架构上它实际上包含了一个LOCK前缀。是不是比较眼熟？没错，他就是java的volatile变量读写操作
经过JIT编译后携带的指令前缀，volatile就是通过它保证了可见性。因此到这里我们可以确定atomic操作的可观测性。

## 意外收获
阅读Once源码还有个意外收获。注意到Once的定义处有这样一段注释
```go
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
其中提到了为什么要把done这个成员变量放在结构体的第一位，以及一个神秘的词——`hot path`。什么是`hot path`呢？参考stackoverflow上的
[回答](https://stackoverflow.com/questions/59174176/what-does-hot-path-mean-in-the-context-of-sync-once)，它指泛指编译后的go程序中
极度频繁运行的指令集合，其特点是所有的`hot path`中的调用都会被内联编译以保证运行速度。

什么是内联编译？就是每次调用都会生成一次完整的操作指令直接执行，而不是通过跳转指令跳转至定义处执行。这样做的好处是提高运行速度，但缺点也很明显：某个调用
很频繁的话就会大量浪费空间，造成编译后的程序体积膨胀。

Once作为go语言官方提供的6种同步机制之一，被广泛应用于各个场景，其Do方法的调用是极度频繁的，而每次Do的调用都会访问done，因此done也被列为
`hot path`之一。而为了保证其内联编译的体积，这里做的一个优化就是将done放置于结构体的第一位，好处是能直接通过结构体地址读取其值
（原理类似c语言中数组的地址等于首位元素的地址），节省了用于计算偏移量的指令，缓解了内联编译后的体积膨胀问题。

## 总结
Once的核心内容其实就是一个go版的双重检测锁，需要注意的是其使用了atomic.Load和atomic.Store操作来保证操作的可观测性，但是第二重检测时由于
临界区内保证了其能观测到更新操作，因此使用了直接读取。

另外，我们通过注释发现了go的`hot path`机制，并且了解到结构体的首位在访问时能具有一定优势，能节省一定指令数量，从而可以在诸如`hot path`等内联编译
场景下优化体积膨胀的问题。