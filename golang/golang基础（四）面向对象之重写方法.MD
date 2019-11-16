#golang基础（四）面向对象之重写方法
###方法重写
overwrite被认为是面向对象理所当然的特性之一，那我们看看在golang中要如何实现这个特性吧。

下面看个实际案例，某个中间件希望在业务代码进行网络请求操作时记录一些信息（请求URL、耗时、返回码等等），由于golang的标准库的http.Client
没有实现无侵入的插件架构，因此只能在中间件中封装一个client供业务方使用，其中的细节涉及到了http RoundTrip方法的重写。

中间件的简化版实现如下：

    type TransportWrapper struct {
        http.Transport
        logger log.Logger
    }
    
    func (t *TransportWrapper) RoundTrip(req *http.Request) (*http.Response, error) {
        t.logger.Printf("sending to %s...", req.URL)
        start := time.Now()
        response, err := t.Transport.RoundTrip(req)
        latency := time.Since(start)
        if err != nil {
            t.logger.Printf("error %s", err.Error())
        } else {
            t.logger.Printf("get resp to %s...", req.URL)
        }
        t.logger.Printf("req takes %dms", latency.Milliseconds())
        return response, err
    }
    
    func Register() * http.Client{
    	l := log.New(os.Stdout,"",0755)
    	dt := http.DefaultTransport.(*http.Transport)
    	t := TransportWrapper{
    		Transport: *dt,
    		logger:    l,
    	}
    	c := http.Client{
    		Transport:     t,
    		Timeout:       3000 * time.Millisecond,
    	}
    	return &c
    }
使用示例：
    
    func TestRoundTripper(t *testing.T) {
        req, err := http.NewRequest("GET", "http://baidu.com", nil)
        checkError(err, t)
        client := Register()
        for i := 0; i < 10; i++ {
            fmt.Printf("No.%d req", i)
            _, err := client.Do(req)
            checkError(err, t)
            time.Sleep(time.Second)
        }
    }
    
    func checkError(err error, t *testing.T)  {
    	if err != nil{
    		t.Error(err)
    	}
    }
代码逻辑很简单，重点关注的是TransportWrapper以及其与RoundTripper实现关系。下面简单说明。

注意到，TransportWrapper继承自http.Transport；而 http.Transport实现了RoundTripper接口：

    type RoundTripper interface {
    	RoundTrip(*Request) (*Response, error)
    }
    
    func (t *Transport) RoundTrip(req *Request) (*Response, error) {
    	return t.roundTrip(req)
    }
注意这里使用指针做receiver。http.Client的定义中，transport成员被声明为RoundTripper类型：

    type Client struct {
        Transport RoundTripper
        ...
    }
因此我们只需要让嵌入了监控代码的TransportWrapper重写RoundTripper这个接口就好了。怎么做？直接写一个RoundTrip方法就好了。

但这里注意一点：在java中，子类在重写父类方法a时，如果想调用父类方法a，直接super.a(..)就好了。
但是golang中没有super关键字，那要怎么做呢？如上边代码中，必须利用匿名字段引用父类型后调用。

    response, err := t.Transport.RoundTrip(req)