## goroutine实现原理

```
在操作系统的OS Thread和编程语言的User Thread之间，实际上存在3中线程对应模型:

	1 N:1是说，多个（N）用户线程始终在一个内核线程上跑，context上下文切换确实很快，但是无法真正的利用多核。
	2 1:1是说，一个用户线程就只在一个内核线程上跑，这时可以利用多核，但是上下文switch很慢，频繁切换效率很低。
	3 M:N是说， 多个goroutine在多个内核线程上跑，这个看似可以集齐上面两者的优势，但是无疑增加了调度的难度。

goroutine google runtime默认的实现为M:N的模型，于是这样可以根据具体的操作类型（操作系统阻塞或非阻塞操作）调整goroutine和OS Thread的映射情况，显得更加的灵活。

在goroutine实现中，有三个最重要的数据结构，分别为G M P:

    G：代表一个goroutine
    M：代表 一个OS Thread
    P：一个P和一个M进行绑定，代表在这个OS Thread上的调度器
```



## goroutine 调度机制

```
```



## go并发模型

```
Go实现了两种并发形式。
	1 多线程共享内存。其实就是Java或者C++等语言中的多线程开发。
		
	2 Go语言特有的，也是Go语言推荐的：CSP（communicating sequential processes）并发模型。
	不要以共享内存的方式来通信，相反，要通过通信来共享内存。
	Go的CSP并发模型，是通过goroutine和channel来实现的
```

