title: 线程-信号
category: 线程
date: 2017-4-19
tags: [thread,signal]
toc: false
comments: true
---
#### 1,线程与信号

1. 每个线程都有自己的信号屏蔽字，但是信号的处理是先进程中所有线程共享的。这就意味着单个线程可以阻止某些信号，但当某个线程修改了与某个给定信号相关的处理行为以后，所有的线程都必须共享这个处理行为的改变。
2. 进程中的信号是递送给单个线程的。（Signals are delivered to a single thread in the process. If the signal is related to a hardware fault, the signal is usually sent to the thread whose action caused the event. Other signals, on the other hand, are delivered to an arbitrary thread.)对于该句的理解是：因为进程中所有线程共享信号的处理，所以发送给任意进程（只要该进程没有屏蔽该信号），对该信号的处理都是相同的。

<!--more-->


