#golang基础（五）goroutine全解析
###协程？不，它更像线程
很多人把goroutine称为协程，其实是不准确的。

协程(coroutine)的定义，引自wiki：
> Coroutines are computer program components that generalize subroutines for non-preemptive multitasking, 
>by allowing execution to be suspended and resumed. 

协程的定义中最重要的有两点：
+ non-preemptive：非抢占
+ allowing execution to be suspended and resumed：由程序主动释放cpu执行权

而在如今的goroutine中，第二点可以通过调用方法实现
    
    runtime.Gosched()
而第一点，在Go1.4后已经无法满足，因为Golang在1.4版本中加入了抢占式调度的逻辑。

### 使用方法？简单得令人发指
你只需要在普通的函数调用前，加一个go

    i := 0
    go fmt.Printf("%d",i)
   
这样就已经开启了一个goroutine。是的，就是这么简单。

###main函数退出，则进程退出
goroutine和java的Thread有一点很重要的区别：

java的main Thread在退出后，如果还有非daemon线程在运行，则进程不会退出；
而在golang中，如果main函数的goroutine退出，则整个进程退出，不管是否有还有goroutine在运行。

###"在不知道如何停止的情况下，永远不要启动goroutine"
请记住这句至理名言，他将在危难时刻救你一命。
实际上这句话强调一点：你要清楚你所写的goroutine的退出条件，尤其当你要在某些可能会被重复调用的函数中开启goroutine时（如
构造函数）。

这种说法来自[这篇文章](https://commandercoriander.net/blog/2018/05/22/how-to-leak-a-goroutine-then-fix-it/)，
[中文译文在这](https://blog.csdn.net/wujilei5/article/details/93512768)。
文中列举了一种可能发生goroutine泄露并引起内存泄露的情况。值得注意的是，文中一种停止chan阻塞写法很漂亮，这里介绍下。
首先看问题：如果给你这样一段代码：

    func NewMap(expiration time.Duration) *Map {
        m := &Map{
            data:       make(map[string]expiringValue),
            expiration: expiration,
        }
    
        // start a worker goroutine
        go func() {
            for range time.Tick(expiration) {
                m.removeExpired()
            }
        }()
    
        return m
    }
我们知道，for range chan会循环调用直到chan的发送方主动关闭chan。如果要你给goroutine加个打断循环的方法，你会怎么做？
那还不简单？把time.Tick(expiration)的结果取出来放在Map的成员里，写个Map的方法关闭这个chan就行了啊！

是吗？你仔细看看time.Tick函数的返回值类型，是<-chan Timer，这意味着它是个只能接收、不能发送的chan，也就不能调用close进行关闭(编译报错)！

那咋办？我们看看原文是怎么优雅地解决的：

    func NewMap(expiration time.Duration) *Map {
        m := &Map{
            data:       make(map[string]expiringValue),
            expiration: expiration,
            done:       make(chan struct{}),
        }
    
        // start a worker goroutine
        go func() {
            ticker := time.NewTicker(expiration)
            defer ticker.Stop()
            for {
                select {
                    case <-ticker.C:
                        m.removeExpired()
                    case <-m.done:
                        return
                }
            }
        }()
    
        return m
    }
    func (m *Map) Close() error {
        close(m.done)
        return nil
    }
不得不说，这段代码是真的优雅，不仅使用select实现了goroutine的终止，而且还考虑到了time.Tick方法的存在的泄露问题（参考官方文档），
使用了time.NewTicker替代并主动调用stop防止了ticker的泄露。