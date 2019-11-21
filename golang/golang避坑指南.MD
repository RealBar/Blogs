# golang避坑指南(1)interface之坑多多

### interface{}和void*/Object是一样的吗？
先来看一段关于interface的官方说明

    Under the covers, interfaces are implemented as two elements, a type and a value. The value, called the interface’s
    dynamic value, is an arbitrary concrete value and the type is that of the value. For the int value 3, an interface
    value contains, schematically, (int, 3).
事实上，interface的实现分为两种eface,iface，它们结构如下：

    type iface struct {
    	tab  *itab
    	data unsafe.Pointer
    }
    
    type eface struct {
    	_type *_type
    	data  unsafe.Pointer
    }
iface为有方法声明的interface，eface为空的interface即interface{}。我们可以看到eface结构体中只存两个指针：一个_type类型指针用于存数据的
实际类型，一个通用指针（unsafe.Pointer）存实际数据；iface则比较复杂，这里不做详细展开了，你只需要记住它也是两个指针，和eface一样其中一个用来
存数据，另一个itab指针用来存数据类型以及方法集。因此interface类型的变量所占空间一定为16。

明白了原理，我们看一段简单代码：

example1

    type Student struct {
        Name string
    }
    var b interface{} = Student{
        Name:     "aaa",    
    }
    var c  = b.(Student)
    c.Name = "bbb"
    fmt.Println(b.(Student).Name)
你觉得输出是什么？

如果你的答案是bbb，恭喜你，你掉坑了！不信你运行试试：

    //example1 output
    aaa

这个坑道理很简单，根据我们开头讲的interface原理，很多从java或c/c++转过来的程序员都把interface{}看成了Object或void*。
的确，很多场景（如传参、返回值）它们的确很类似；但请注意，在底层实现上它们是完全不同的。
java的Object以及c语言的void*可以通过通过强转为某个类型获取指向原数据的一个目标类型的引用或指针，因此如果在这个引用或指针上进行修改操作，原数据
也会被修改；但是golang的interface和具体类型之间的转换、赋值是将实际数据复制了一份进行操作的。例如上例中的

    var c  = b.(Student)
实际的过程是首先将b指向的数据复制一份，然后转换为Student类型赋值给c。

记住了吗？好，我们看个类似的例子：

example2

	type Student struct {
		Name string
	}
	a := Student{Name:"aaa"}
	var b interface{} = a
	a.Name = "bbb"
	fmt.Println(b.(Student).Name)
这次，输出结果又会是什么？

如果你给出的答案是aaa，恭喜你脱坑了！

### 遇到interface类的返回值你要注意了
看个简短的例子：

example3

    func GetReader(id int64) io.Reader {
        var r *MyReader = nil
        if id > 0 && id < 10000{
            r = openReader(id)
        }
        return r
    }
    func main() {
        r := GetReader(-2)
        if r == nil {
            fmt.Println("bad reader")
        } else {
            fmt.Println("valid reader")
        }
    }
其中MyReader为某个实现了io.Reader的结构体类型，openReader根据传入参数返回一个MyReader结构体指针。

你觉得这段程序会输出什么呢？会输出"bad reader"吗？

答案是刚好相反:
    
    //example3 output
    valid reader

为了解释这个结果，我们再看两段简单的代码：

example4

    var b interface{} = nil
    fmt.Println(b == nil)
example5

    var a *Student = nil
    var b interface{} = a
    fmt.Println(b == nil)
输出分别为：

    //example4 output
    true
    
    //example5 output
    false
相信仔细对比过后，你应该已经有答案了：**当一个指针赋值给interface类型时，无论此指针是否为nil，赋值过的interface都不为nil**。

ok，结论已经有了，那为什么是这样呢？还记得本文开头介绍的interface底层实现吗？无论是iface还是eface，都有两个指针，指向数据的是通用指针，
还有一个指针用于指定数据类型或方法集；当我们将一个nil指针赋值给interface时，实际是对interface的这两个指针分别赋值，虽言数据指针data为nil，
但是类型指针_type或tab并不是nil，他将指向你的空指针的类型，因此赋值的结果interface肯定不是nil啦！

### 什么？interface还能嵌入struct？
众所周知，一个新定义的type要想实现某个interface，一定需要将该interface的所有方法都实现一遍。

对吗？

老规矩，先上例子：

example6

    type Talkable interface {
    	TalkEnglish(string)
    	TalkChinese(string)
    }
    
    type Student1 struct {
    	Talkable
    	Name string
    	Age  int
    }
    
    func main(){
        a := Student1{Name: "aaa", Age: 12}
        var b Talkable = a
        fmt.Println(b)
    }
以上的代码时100%能编译运行的。输出为：

    // example6 output
    {<nil> aaa 12}
这是一种取巧的方法，将interface嵌入结构体，可以使该类型快速实现该interface。所以，本小节开头的话并不成立。但是如果我们调一下方法呢？

example7

    ...
    func main(){
        a := Student1{Name: "aaa", Age: 12}
        a.TalkEnglish("nice to meet you\n")
    }
可以预见到的，报错了：

    //example7 output
    panic: runtime error: invalid memory address or nil pointer dereference 
并没有实现interface的方法，当然会报错。我们可以只实现interface的一部分方法，比如我只需要用到Talkable的TalkEnglish方法：

    func (s *Student1) TalkEnglish(s1 string) {
    	fmt.Printf("I'm %s,%d years old,%s", s.Name, s.Age, s1)
    }
或者只需要讲中文：

    func (s *Student1) TalkChinese(s1 string) {
    	fmt.Printf("我是 %s, 今年%d岁,%s", s.Name, s.Age, s1)
    }
总而言之，嵌入interface的好处就是可以帮在整体类型兼容某个接口的前提下，允许你你针对你的应用场景只实现interface中的一部分方法。但是在使用时要
注意没有实现的方法在调用时会panic。

###总结
interface时golang编程中使用得非常频繁的特性，我们需要明白它的底层结构，以及一些编译和运行时的特殊之处，能帮我们避免一些不必要的麻烦：
+ interface很类似void*，但在值类型的变量和interface类型变量相互赋值时，会发生数据的复制。
+ 将某个类型的指针赋值给interface，interface的值永远不可能是nil；
+ interface可以嵌入结构体，帮类型快速实现接口，但是注意如果调用未实现的方法则会panic；