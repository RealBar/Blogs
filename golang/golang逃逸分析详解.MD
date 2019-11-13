#golang 逃逸分析详解
##缘起
前段时间跟项目组leader聊到golang编码规范时，我提到一个问题。

我：“golang函数传参是不是应该跟c一样，尽量不要直接传结构体，而要传结构体指针？“

leader：“不对，咱们项目很多都是直接传结构体的。“

我：“那样不会造成不必要的内存copy开销吗？”

leader：“确实会有，但这样可以减小gc压力，因为传值会在栈上分配，而一旦传指针，结构体就会逃逸到堆上。“

我：“有道理。。。“

由于之前是搞java的，关于逃逸分析在golang的上规则还不是很熟，因此，后来在心里一直记得：“一旦将某个局部变量以指针的方式传出，该变量就会逃逸到堆”。

但是我内心还是对这种说法一直存在疑惑，这样的话那我们在高实时要求的业务，调用函数岂不是不能传struct了？如今golang编译器应该不会这么低能吧？这里借机会好好学习一下。

##什么是逃逸分析？
相信熟悉java的朋友对逃逸分析不会太陌生，这里引述周志明大大的原画：

在计算机语言编译器优化原理中，逃逸分析是指分析指针动态范围的方法，它同编译器优化原理的指针分析和外形分析相关联。当变量（或者对象）在方法中分配后，其指针有可能被返回或者被全局引用，这样就会被其他过程或者线程所引用，这种现象称作指针（或者引用）的逃逸(Escape)。
你学java时，老师在讲解jvm内存结构可能跟你说过这样一句话：“new出来的东西都在堆上，栈上存的是它的引用。”其实在现代JVM上这句话是不准确的，因为逃逸分析机制。

简单来说JVM的逃逸分析会在运行时检测当前方法栈帧内new出来的对象的引用是否被传出当前栈帧，传出则发生逃逸，未传出则未发生逃逸，例如：

    public void test(){
        List<Integer> a = new ArrayList<>();
        a.add(1); // a 未发生逃逸，因此在栈上分配
    }
    
    public List<Integer> test1(){
        List<Integer> a = new ArrayList<>();
        a.add(1);
        return a  //a 发生逃逸，因此分配在堆上
    }

对于未发生逃逸的变量，则直接在栈上分配内存。因为栈上内存由在函数返回时自动回收，因此能减小gc压力。

##准备
首先要明确几点：

+ 不同于jvm的运行时逃逸分析，golang的逃逸分析是在编译期完成的。
+ Golang的逃逸分析只针对指针。一个值引用变量如果没有被取址，那么它永远不可能逃逸。
+ 本文golang运行环境：go version go1.13.4 darwin/amd64

另外，验证某个函数的变量是否发生逃逸的方法有两个：

1. go run -gcflags "-m -l" (-m打印逃逸分析信息，-l禁止内联编译)；例：

        ➜  testProj go run -gcflags "-m -l" internal/test1/main.go
        \#command-line-arguments
        internal/test1/main.go:4:2: moved to heap: a
        internal/test1/main.go:5:11: main make([]*int, 1) does not escape
2. go tool compile -S main.go | grep runtime.newobject（汇编代码中搜runtime.newobject指令，该指令用于生成堆对象）,例：

        ➜  testProj go tool compile -S internal/test1/main.go | grep newobject
            0x0028 00040 (internal/test1/main.go:4) CALL    runtime.newobject(SB)

>备注：关于-gcflags "-m -l"的输出，有两种情况:
>+ moved to heap:xxx
>+ xxx escapes to heap
>
>根据我个人的实验结果，二者都表示发生逃逸，当xxx变量类型为指针时，出现下一种；当xxx变量为值类型时，为上一种。有兴趣的可以用上边的命令跑一下下边的代码

    type S int
    func main() {
        a := S(0)
        b := make([]*S, 2)
        b[0] = &a
        c := new(S)
        b[1] = c
    }

>关于这个问题在stack overflow上有个回答https://stackoverflow.com/questions/51518742/what-is-the-meaning-of-the-output-from-go-run-gcflags-m-xxx-go，应该是错的，至少go版本13.4是错的。

##Golang 逃逸分析
那么究竟什么时候，什么情况下会发生逃逸呢？下面就是本文所主要探究的内容。

###情况1
首先说一种最基本的情况：

> **在某个函数中new或字面量创建出的变量，将其指针作为函数返回值，则该变量一定发生逃逸。**

这是golang基础教程中经常举的，用于区别c/c++例子：

    func test() *User{
        a := User{}
        return &a
    }
这种情况较为基础，这里不再赘述。

###情况2
验证本文开头的说法是否正确，即当某个值取指针传给另一个函数，该值是否发生逃逸：

####example1

    type User struct {
        Username string
        Password string
        Age      int
    }
    
    func main() {
        a := "aaa"
        u := &User{a, "123", 12}
        Call1(u)
    }
    
    func Call1(u *User) {
        fmt.Printf("%v",u)
    }
看一下逃逸情况：

    ➜  testProj go run -gcflags "-m -l" main.go
    # command-line-arguments
    ./main.go:18:12: leaking param: u
    ./main.go:19:12: Call1 ... argument does not escape
    ./main.go:19:13: u escapes to heap
    ./main.go:14:23: &User literal escapes to heap
果然发生了逃逸，这里将指针传给一个函数Call1并打印，如果不打印，只对u进行读写呢？修改一下Call1

####example2

    type User struct {
        Username *string
        Password string
        Age      int
    }
    
    func main() {
        a := "aaa"
        u := &User{&a, "123", 12}
        Call1(u)
    }
    
    func Call1(u *User) int{
        u.Username="bbb"
        return u.Age * 20
    }
结果：

    ➜  testProj go run -gcflags "-m -l" main.go
    # command-line-arguments
    ./main.go:15:12: Call1 u does not escape
    ./main.go:11:23: main &User literal does not escape
居然没有逃逸！为什么example1发生了逃逸呢？也许你会说，example1里Call1把u传给了fmt.Printf了啊，那我们再做个实验，Call1多传几次，但还是只对u进行读写：

####example3

    func main() {
        a := "aaa"
        u := &User{a, "123", 12}
        Call1(u)
    }
    
    func Call1(u *User) int {
        return Call2(u)
    }
    
    func Call2(u *User) int {
        return Call3(u)
    }
    
    func Call3(u *User) int {
        u.Username = "bbb"
        return u.Age * 20
    }
结果：

    ➜  testProj go run -gcflags "-m -l" main.go
    # command-line-arguments
    ./main.go:23:12: Call3 u does not escape
    ./main.go:19:12: Call2 u does not escape
    ./main.go:15:12: Call1 u does not escape
    ./main.go:11:23: main &User literal does not escape
可以看到，依然没有发生逃逸。

那究竟为什么example1会逃逸呢？

我们点进去看看fmt.Printf的源码，最终我找到了被传入的u被赋值给了pp指针的一个成员变量：

    func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
        p := newPrinter()
        p.doPrintf(format, a)
    ...
    }
    
    func (p *pp) doPrintf(format string, a []interface{}) {
    ...
    p.printArg(a[argNum], rune(c))
    ...
    }
    
    func (p *pp) printArg(arg interface{}, verb rune) {
        p.arg = arg
        p.value = reflect.Value{}
    ...
    }
而这个pp类型的指针p是由构造函数newPrinter返回的，根据我们情况1，p一定发生逃逸，而p引用了传入指针，由此我们可以总结：

> **被已经逃逸的变量引用的指针，一定发生逃逸。**
 
###情况3
我们再看上面备注中的代码例子：

func main() {
	a := make([]*int,1)
	b := 12
	a[0] = &b
}
结果：

    ➜  testProj go run -gcflags "-m -l" main.go
    # command-line-arguments
    ./main.go:7:2: moved to heap: b
    ./main.go:6:11: main make([]*int, 1) does not escape
sliace a并没有发生逃逸，但是被a引用的b依然逃逸了。类似的情况同样发生在map和chan中：

    func main() {
        a := make([]*int,1)
        b := 12
        a[0] = &b
    
        c := make(map[string]*int)
        d := 14
        c["aaa"]=&d
    
        e := make(chan *int,1)
        f := 15
        e <- &f
    }
结果：

    ➜  testProj go run -gcflags "-m -l" main.go
    # command-line-arguments
    ./main.go:7:2: moved to heap: b
    ./main.go:11:2: moved to heap: d
    ./main.go:15:2: moved to heap: f
    ./main.go:6:11: main make([]*int, 1) does not escape
    ./main.go:10:11: main make(map[string]*int) does not escape
由此我们可以得出结论：

> **被指针类型的slice、map和chan引用的指针一定发生逃逸**

>备注：stack overflow上有人提问为什么使用指针的chan比使用值的chan慢30%，答案就在这里：使用指针的chan发生逃逸，gc拖慢了速度。问题链接https://stackoverflow.com/questions/41178729/why-passing-pointers-to-channel-is-slower

##总结
我们得出了指针必然发生逃逸的三种情况（go version go1.13.4 darwin/amd64)：

+ 在某个函数中new或字面量创建出的变量，将其指针作为函数返回值，则该变量一定发生逃逸（构造函数返回的指针变量一定逃逸）；
+ 被已经逃逸的变量引用的指针，一定发生逃逸；
+ 被指针类型的slice、map和chan引用的指针，一定发生逃逸；
同时我们也得出一些必然不会逃逸的情况：

+ 指针被未发生逃逸的变量引用；
+ 仅仅在函数内对变量做取址操作，而未将指针传出；
有一些情况可能发生逃逸，也可能不会发生逃逸：

+ 将指针作为入参传给别的函数；这里还是要看指针在被传入的函数中的处理过程，如果发生了上边的三种情况，则会逃逸；否则不会逃逸；