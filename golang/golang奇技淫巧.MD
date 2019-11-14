#golang奇技淫巧

平时写代码总是能碰到一些具有一定参考价值的技巧，这些技巧可能并不是那么的惊天地、泣鬼神，但往往如此十分有价值且实用。
故而这里做一个记录。

###Must函数
有时候我们调用函数时确实不需要在返回值处检查error，希望如果处理失败则直接panic，然后由统一的上层拦截器去处理。
这种函数一般都被称为Must函数，它们的返回值只可能为合法值：

    func MustValidate(req *http.Reuest){
        ...
        if notvalid{
            panic(errors.New(...))
        }
    }

### slice解构
内置的append函数可以直接append slice吗？可以！但是要解构，直接传slice是会报错的：

    a := []int{1,2,3}
    b := []int{4,5,6}
    append(a,b)  // 报错

为什么呢？我们看看append的函数签名：
    
    func append(slice []Type, elems ...Type) []Type
关于...的含义，其实有两重：
1. 放在函数声明中，用于声明可变参数及类型
2. 放在正常函数体中，用于解构运算
这里append函数中声明elems为Type类型的可变参数，而如果我们直接传入[]Type，类型不匹配。所以我们需要先吧[]Type转换为一个一个的Type，
然后一个一个用逗号分隔传入append。

这么麻烦？求解脱？

解构运算符就是来解决这个问题的，我们直接在[]Type变量前添加"..."就能直接传入append函数了！

    a := []int{1,2,3}
    b := []int{4,5,6}
    append(a,b...) //成功

"..."这个操作符非常常见，例如在gin中

    // Use adds middleware to the group, see example code in GitHub.
    func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
    	group.Handlers = append(group.Handlers, middleware...)
    	return group.returnObj()
    }
