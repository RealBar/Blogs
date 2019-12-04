#go奇技淫巧

平时写代码总是能碰到一些具有一定参考价值的技巧，这些技巧可能并不是那么的高大上，但往往如此十分有价值且实用。
故而这里做一个记录。

###Must函数
我们在处理一些可能返回错误的逻辑时一般会返回error，让调用方检查返回值来判断是否发生来错误：
```
func Validate(req *http.Reuest)error{
    ...
    if notvalid{
        return NotValidError
    }
    return nil
}
```
但是有时候我们调用函数时确实不需要在返回值处检查error，希望如果处理失败则直接panic，然后由统一的上层拦截器去处理。
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

解构运算符就是来解决这个问题的，我们直接在[]Type变量后添加"..."就能直接传入append函数了！

    a := []int{1,2,3}
    b := []int{4,5,6}
    append(a,b...) //成功

"..."这个操作符非常常见，例如在gin中

    // Use adds middleware to the group, see example code in GitHub.
    func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
    	group.Handlers = append(group.Handlers, middleware...)
    	return group.returnObj()
    }

### copy？append？拼接两个slice用哪个？
上边说了，可以直接使用append拼接两个slice，
+ 方法一：


    sliceC := append(sliceA,sliceB...)
十分简单且省事。
+ 方法二：


    sliceC := make([]int,len(sliceA + sliceB))
    copy(sliceC,sliceA)
    copy(sliceC[len(sliceA):],sliceB)
原理也很简单，过程稍微复杂一点，先分配一个大slice，再两次copy。

那么问题来了，有了方法一，为什么还要方法二？我们不妨比较一下两者的性能：

    func TestSliceAppend(t *testing.T) {
    	println("preparing data...")
    	ran := rand.New(rand.NewSource(time.Now().Unix()))
    	l := 10 * 1024 * 1024
    	a := make([]int, l)
    	for i := 0; i < l; i++ {
    		a[i] = ran.Int()
    	}
    	b := make([]int, l)
    	for i := 0; i < l; i++ {
    		b[i] = ran.Int()
    	}
    	println("preparing data finished")
    	t1 := time.Now()
    	c := append(a, b...)
    	d1 := time.Since(t1)
    	fmt.Printf("append uses:%vms,%d\n", d1.Milliseconds(), c[1024])
    
    	t2 := time.Now()
    	d := make([]int, 2*l)
    	copy(d, a)
    	copy(d[l:], b)
    	d2 := time.Since(t2)
    	fmt.Printf("copy uses:%vms,%d\n", d2.Milliseconds(), d[1024])
    }
结果：

    preparing data...
    preparing data finished
    append uses:129ms,319467484622772473
    copy uses:106ms,319467484622772473
经过多次测试，第二种方法均比第一种快1/6。

由此我们基本得出结论：**在拼接大slice时，优先使用第二种方法**

### 对象设置参数项太多？看看grpc怎么做
> grpc-go/clientconn.go

    // Dial creates a client connection to the given target.
    func Dial(target string, opts ...DialOption) (*ClientConn, error) {
    	return DialContext(context.Background(), target, opts...)
    }
>grpc-go/dialoptions.go

    // dialOptions configure a Dial call. dialOptions are set by the DialOption
    // values passed to Dial.
    type dialOptions struct {
    	unaryInt  UnaryClientInterceptor
    	streamInt StreamClientInterceptor
    
    	chainUnaryInts  []UnaryClientInterceptor
    	chainStreamInts []StreamClientInterceptor
    
    	cp          Compressor
    	dc          Decompressor
    	bs          internalbackoff.Strategy
    	block       bool
    	insecure    bool
    	timeout     time.Duration
    	scChan      <-chan ServiceConfig
    	authority   string
    	copts       transport.ConnectOptions
    	callOptions []CallOption
    	// This is used by v1 balancer dial option WithBalancer to support v1
    	// balancer, and also by WithBalancerName dial option.
    	balancerBuilder balancer.Builder
    	// This is to support grpclb.
    	resolverBuilder             resolver.Builder
    	channelzParentID            int64
    	disableServiceConfig        bool
    	disableRetry                bool
    	disableHealthCheck          bool
    	healthCheckFunc             internal.HealthChecker
    	minConnectTimeout           func() time.Duration
    	defaultServiceConfig        *ServiceConfig // defaultServiceConfig is parsed from defaultServiceConfigRawJSON.
    	defaultServiceConfigRawJSON *string
    	// This is used by ccResolverWrapper to backoff between successive calls to
    	// resolver.ResolveNow(). The user will have no need to configure this, but
    	// we need to be able to configure this in tests.
    	resolveNowBackoff func(int) time.Duration
    }
    type DialOption interface {
    	apply(*dialOptions)
    }
    
### 不能Seek的Reader如何找回已经read的数据？
也许有这种情况：当一个函数中你从reader中读取了一些数据后，在另一个处理过程中你需要哪些已经被读取的数据和剩下的数据一起完成业务处理。
如果reader实现了Seeker，那相当简单：

    // reader implements Seeker
    reader.read(buf)
    reader.Seek(0,SeekStart)
但是如果reader没有实现Seeker接口怎么办？有一个妙招：

    tmpBuf := new(bytes.Buffer)
    newReader := io.TeeReader(reader,tmpBuf)
    newReader.read(buf)
    fullReader := io.MultiReader(tmpBuf,reader)
利用TeeReader将读取的数据存一份，然后用MultiReader将存的数据和剩下的数据拼接起来，就构成了完整的数据。
这个方法的实际应用场景就是图片的验证和解析。验证调用image.DecodeConfig需要读取图片头部信息，之后解析调用image.Decode需要读取全部的图片信息，
所以如果数据来源reader不是Seeker，那必定需要此种方法将读取的图片头存起来，然后拼接起来交给Decode：

    var header bytes.Buffer
    conf, err := png.DecodeConfig(io.TeeReader(streamReader, &header))
    if err != nil || !confValid(conf) { ... }
    im, err := png.Decode(io.MultiReader(&header, streamReader)

### 如何优雅地设置超时？
有编程经验的人都知道在耗时调用前一定要加超时，如http.Client、redis客户端、数据库客户端等都直接或间接提供了超时配置。那么如果要你亲手实现一个
超时机制，你会怎么做呢？
做法1
```go

```
