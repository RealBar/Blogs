# 为什么golang没有volatile?
### 缘起
从java转golang程序员可能会有一个疑问：为什么golang没有volatile关键字？这样的话如何保证可见性？
在java中我们一般可以通过给某个变量加上volatile修饰符来保证不同线程对该变量读写时的可见性，并凭借JVM的happens-before机制避免了data race，
关于data race和happens-before机制，这里不做过多阐述，有疑问者可自行google。
正是由于上述JVM全方位保姆式的完善保护机制，java程序员们习以为常地写出无锁代码：
```java
public class Connection{
    private static volatile Boolean initialized = false;
    public void init(){
        //init process...
        initialized = true;
    }
    public Boolean isInitialized(){
        return initialized;
    }
}
```
上述代码可以在一个线程中调用init，然后在其他线程中循环调用isInitialized去检测初始化是否完成，不会产生任何可见性问题和data race。

但是如果使用golang写个类似的例子试试：
```go
var inited = false

func Setup(){
	time.Sleep(time.Second)
	inited = true
}
func main() {
	go Setup()

	for{
		if inited{
			break
		}
		time.Sleep(100 * time.Millisecond)
	}

	fmt.Println("setup succeed")
}
```
你可以使用go run -race 运行上边的程序，你会看到警告：
```
➜  go-learning git:(master) ✗ go run -race main/data_race.go
==================
WARNING: DATA RACE
Write at 0x0000012272c1 by goroutine 7:
  main.Setup()
      /Users/yalouwang/workspace/go/go-learning/main/data_race.go:20 +0x47

Previous read at 0x0000012272c1 by main goroutine:
  main.main()
      /Users/yalouwang/workspace/go/go-learning/main/data_race.go:26 +0x65

Goroutine 7 (running) created at:
  main.main()
      /Users/yalouwang/workspace/go/go-learning/main/data_race.go:23 +0x46
==================
setup succeed
Found 1 data race(s)
exit status 66
```
-race用于检测代码中可能存在的data race。这里的告警信息写得很清楚，一个goroutine的写操作和另一个goroutine的读操作可能发生data race，有风险。

你检查完代码后发现，应该在inited前加一个"volatile"；但是刚刚想要改代码，却发现golang中根本没有这个关键字！

于是你大声问道：golang的volatile去哪儿了？
### 解决问题
先看看如何解决上边的data race问题。实际上在golang中，互斥锁在并发操作中是非常常见的，当某个变量存在并发访问的可能时，请一定记得加锁：
```
var inited = false
var lock sync.Mutex

func Setup(){
	time.Sleep(time.Second)
	lock.Lock()
	inited = true
	lock.Unlock()
}

func main() {
	go Setup()

	for{
		lock.Lock()
		b := inited
		lock.Unlock()
		if b{
			break
		}
		time.Sleep(100 * time.Millisecond)
	}

	fmt.Println("setup succeed")
}
```
再次运行go run -race，我们发现不再告警了。

好了，解决了旧问题，新的问题又来了：加锁不就降低并发性能了吗？

### 新的理念
在c/c++以及java中，我们往往习惯设置共享对象或基本类型，然后在不同的线程中对它进行读写操作从而达到线程间通信的目的。这样做的好处就是
操作简单，坏处也很明显：容易造成data race，而为了避免data race又不得不加锁，加锁又降低了并发性能。

所以golang提出了一个新的理念：
>Do not communicate by sharing memory; instead, share memory by communicating.
>
>——出自 effective go

乍一看有点玄乎，但实际上它表达的意思很简单：**别再用共享变量啦，快来试试偶们的channel吧！**

关于channel的使用相信大家已经很熟了，我们用chan改造一下上边的代码：
```
func Setup2() <-chan bool {
	time.Sleep(time.Second * 3)
	c := make(chan bool,1)
	c <- true
	return c
}

func main() {
	if <-Setup2(){
		fmt.Println("setup succeed")
	}
}
```
我们得到了一个无锁的版本，是不是十分简单且优雅呢？事实上使用chan的好处不仅在于解决了并发访问的data race和锁的问题，而且还提高了代码的运行效率——chan的接收端goroutine会被挂起
直到chan中有值可读，相比于传统方法的循环检测共享变量，这种方式效率明显要高不少，且优雅。

### 所以，volatile呢？
相信大多数读者看到这里，对开头的问题已经有答案了。

volatile呢？他早就被被golang的创造者们扔进了历史的垃圾桶啦！放弃它和它老旧的共享变量思想吧，是时候使用channel重构你的代码了！