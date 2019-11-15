#golang基础（三）面向对象之继承方法
###方法继承
众所周知golang使用struct的匿名字段进行继承，子type可以直接访问父type或其成员。

需要注意的是，父struct方法也能继承，如下：

    type Human struct {
    	Name string
    	Age  int
    }
    
    func (h *Human) Greet() string {
    	return fmt.Sprintf("Hello, I'm %s, %d yeats old.", h.Name, h.Age)
    }
    
    type Student struct {
    	Human
    	Class      int
    	Speciality string
    }
    func TestStudent(t *testing.T) {
    	s := Student{
    		Human:      Human{"aaa", 123},
    		Class:      3,
    		Speciality: "science",
    	}
    	println(s.Greet())
    }
此处特别提醒，当receiver type为pointer时，有个小坑，看下面：
    
    type Speakable interface {
    	Greet() string
    }
    
    func Speak(s Speakable)string{
    	return s.Greet()
    }
    
    func TestStudent(t *testing.T) {
        s := Student{
            Human:      Human{"aaa", 123},
            Class:      3,
            Speciality: "science",
        }
        Speak(s) // 报错:Student does not implement Speakable (Greet method has pointer receiver)
    	Speak(&s) // OK
    }

总结起来就是：*当某个类型实现某个接口的方法的receiver是指针时，必须使用这个类型的指针才能赋值到该接口类型的变量上。*

[stack overflow上的相关回答](https://stackoverflow.com/questions/40823315/x-does-not-implement-y-method-has-a-pointer-receiver)
