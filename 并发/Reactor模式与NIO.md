# Reactor模式

## 定义

Reactor(事件驱动）是一种广泛应用在服务器端开发的设计模式。那么什么是事件驱动呢？

最早的网络编程思想，服务器不断监听新的套接字连接，如果有，调用`handle`处理，但处理时这种方法无法并发，吞吐量低，前面的请求未处理完，后面的请求只能阻塞。

之后引入了多线程的概念，之前的请求阻塞不会影响后续请求，增大了服务器吞吐量，如下图所示。

![pic](https://github.com/solo941/notes/blob/master/并发/pics/150046-20170901082644655-2144636819.png)

但线程的创建消耗资源，连接数变大时，`JVM`资源不足。线程池可以缓解创建销毁线程的开销，但仍存在线程切换的开销。对于服务器，CPU处理速度快于IO速度。试想一个进程一个数据读了500ms，期间进程切换到它3次，但是CPU却什么都不能干，这很不划算。

这时一个聪明的办法是把`IO`操作单独拉出来存储在线程池里，所有`handler`向一个中间人(Reactor)注册感兴趣的事件`event handler`，`IO`就绪后，由中间人产生事件通知`handler`处理。它是一种异步、非阻塞的模式。

### 单线程

它是由一个线程来接收客户端的连接，并将该请求分发到对应的事件处理 handler 中，整个过程完全是异步非阻塞的；并且完全不存在共享资源的问题。所以理论上来说吞吐量也还不错。

![pic](https://github.com/solo941/notes/blob/master/并发/pics/150046-20170901082719280-29887948.png) 

> 但由于是一个线程，对多核 CPU 利用率不高，一旦有大量的客户端连接上来性能必然下降，甚至会有大量请求无法响应。
> 最坏的情况是一旦这个线程哪里没有处理好进入了死循环那整个服务都将不可用！

### 多线程

![pic](https://github.com/solo941/notes/blob/master/并发/pics/150046-20170901082834187-1581301551.png)

将处理器的执行放入线程池，多线程进行业务处理。但Reactor仍为单个线程。

### 主从多线程

![pic](https://github.com/solo941/notes/blob/master/并发/pics/112151380898648.jpg)

继续改进：对于多个CPU的机器，为充分利用系统资源，将Reactor拆分为两部分。`mainReactor`负责监听连接，`accept`连接给`subReactor`处理，为什么要单独分一个Reactor来处理监听呢？因为像TCP这样需要经过3次握手才能建立连接，这个建立连接的过程也是要耗时间和资源的，单独分一个Reactor来处理，可以提高性能。`subReactor`可以有一个或者多个，每个`subReactor`都会在一个独立线程中执行，并且维护一个独立的`NIO Selector`。这样的好处很明显，因为`subReactor`也会执行一些比较耗时的IO操作，例如消息的读写，使用多个线程去执行，则更加有利于发挥CPU的运算能力，减少IO等待时间。

## 参考资料

[高性能IO之Reactor模式](https://www.cnblogs.com/doit8791/p/7461479.html)

[Netty源码解读（四）Netty与Reactor模式](http://ifeve.com/netty-reactor-4/)

[Netty(二) 从线程模型的角度看 Netty 为什么是高性能的？](https://crossoverjie.top/2018/07/04/netty/Netty(2)Thread-model/)