#golang基础（六）标准包context
### 干嘛的？
看看官方文档怎么说
> Package context defines the Context type, which carries deadlines,
  cancellation signals, and other request-scoped values across API boundaries
  and between processes.

这里要关注几个重点：
1. Context的内容包括：截止时间点，取消信号，以及其他请求范围内的值。
2. Context的使用场景是跨api或者跨进程的信息传递。