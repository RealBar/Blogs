#golang基础（五）goroutine全解析
###协程？不，它更像线程
很多人把goroutine称为协程，其实是不准确的。

协程(coroutine)的定义，引自wiki：
> Coroutines are computer program components that generalize subroutines for non-preemptive multitasking, 
>by allowing execution to be suspended and resumed. 


###main函数退出，则进程退出

