> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [xiaoyaozi.blog.csdn.net](https://xiaoyaozi.blog.csdn.net/article/details/42779915?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~default-1.essearch_pc_relevant&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2~default~CTRLIST~default-1.essearch_pc_relevant)

> 一、  Thrift 服务器端几种工作模式分析与总结 Thrift 为服务器端提供了多种工作模式，本文中将涉及以下 5 中工作模式：TSimpleServer、TNonblockingServer、THsHaSe......

相关示例代码见：**http://download.csdn.net/detail/hjx_1000/8374829**  

**五、  Thrift 服务器端几种工作模式分析与总结**

Thrift 为服务器端提供了多种工作模式，本文中将涉及以下 5 中工作模式：TSimpleServer、TNonblockingServer、THsHaServer、TThreadPoolServer、TThreadedSelectorServer，这 5 中工作模式的详细工作原理如下：

**1.      TSimpleServer 模式**

TSimpleServer 的工作模式只有一个工作线程，循环监听新请求的到来并完成对请求的处理，它只是在简单的演示时候使用，它的工作方式如图 5.1 所示：

![](https://img-blog.csdn.net/20150116174733671?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaG91aml4aW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

图 5.1 TSimpleServer 的工作模式

TSimpleServer 的工作模式采用最简单的阻塞 IO，实现方法简洁明了，便于理解，但是一次只能接收和处理一个 socket 连接，效率比较低，主要用于演示 Thrift 的工作过程，在实际开发过程中很少用到它。

**2.      TNonblockingServer 模式**

TNonblockingServer 工作模式，该模式也是单线程工作，但是该模式采用 NIO 的方式，所有的 socket 都被注册到 selector 中，在一个线程中通过 seletor 循环监控所有的 socket，每次 selector 结束时，处理所有的处于就绪状态的 socket，对于有数据到来的 socket 进行数据读取操作，对于有数据发送的 socket 则进行数据发送，对于监听 socket 则产生一个新业务 socket 并将其注册到 selector 中，如下图 5.2 所示：

![](https://img-blog.csdn.net/20150116174813571?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaG91aml4aW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

图 5.2、TNonblockingServer 工作模式

上图 5.2 中读取数据之后的业务处理就是根据读取到的调用请求，调用具体函数完成处理，只有完成函数处理才能进行后续的操作；

**TNonblockingServer 模式优点：**

相比于 TSimpleServer 效率提升主要体现在 IO 多路复用上，TNonblockingServer 采用非阻塞 IO，同时监控多个 socket 的状态变化；

**TNonblockingServer 模式缺点：**

TNonblockingServer 模式在业务处理上还是采用单线程顺序来完成，在业务处理比较复杂、耗时的时候，例如某些接口函数需要读取数据库执行时间较长，此时该模式效率也不高，因为多个调用请求任务依然是顺序一个接一个执行。

**3.      THsHaServer 模式（半同步半异步）**

THsHaServer 类是 TNonblockingServer 类的子类，在 5.2 节中的 TNonblockingServer 模式中，采用一个线程来完成对所有 socket 的监听和业务处理，造成了效率的低下，THsHaServer 模式的引入则是部分解决了这些问题。THsHaServer 模式中，引入一个线程池来专门进行业务处理，如下图 5.3 所示；

![](https://img-blog.csdn.net/20150116174804281?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaG91aml4aW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

图 5.3 THsHaServer 模式

**THsHaServer 的优点：**

与 TNonblockingServer 模式相比，THsHaServer 在完成数据读取之后，将业务处理过程交由一个线程池来完成，主线程直接返回进行下一次循环操作，效率大大提升；

**THsHaServer 的缺点：**

由图 5.3 可以看出，主线程需要完成对所有 socket 的监听以及数据读写的工作，当并发请求数较大时，且发送数据量较多时，监听 socket 上新连接请求不能被及时接受。

**4.      TThreadPoolServer 模式**

TThreadPoolServer 模式采用阻塞 socket 方式工作，, 主线程负责阻塞式监听 “监听 socket” 中是否有新 socket 到来，业务处理交由一个线程池来处理，如下图 5.4 所示：

![](https://img-blog.csdn.net/20150116174847158?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaG91aml4aW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

图 5.4 线程池模式工作过程

**TThreadPoolServer 模式优点：**

线程池模式中，数据读取和业务处理都交由线程池完成，主线程只负责监听新连接，因此在并发量较大时新连接也能够被及时接受。线程池模式比较适合服务器端能预知最多有多少个客户端并发的情况，这时每个请求都能被业务线程池及时处理，性能也非常高。

**TThreadPoolServer 模式缺点：**

线程池模式的处理能力受限于线程池的工作能力，当并发请求数大于线程池中的线程数时，新请求也只能排队等待。

**5.      TThreadedSelectorServer**

TThreadedSelectorServer 模式是目前 Thrift 提供的最高级的模式，它内部有如果几个部分构成：

（1）  一个 AcceptThread 线程对象，专门用于处理监听 socket 上的新连接；

（2）  若干个 SelectorThread 对象专门用于处理业务 socket 的网络 I/O 操作，所有网络数据的读写均是有这些线程来完成；

（3）  一个负载均衡器 SelectorThreadLoadBalancer 对象，主要用于 AcceptThread 线程接收到一个新 socket 连接请求时，决定将这个新连接请求分配给哪个 SelectorThread 线程。

（4）  一个 ExecutorService 类型的工作线程池，在 SelectorThread 线程中，监听到有业务 socket 中有调用请求过来，则将请求读取之后，交个 ExecutorService 线程池中的线程完成此次调用的具体执行；

![](https://img-blog.csdn.net/20150116174844953?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaG91aml4aW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

图 5.5 TThreadedSelectorServer 模式的工作过程

如上图 5.5 所示，TThreadedSelectorServer 模式中有一个专门的线程 AcceptThread 用于处理新连接请求，因此能够及时响应大量并发连接请求；另外它将网络 I/O 操作分散到多个 SelectorThread 线程中来完成，因此能够快速对网络 I/O 进行读写操作，能够很好地应对网络 I/O 较多的情况；TThreadedSelectorServer 对于大部分应用场景性能都不会差，因此，如果实在不知道选择哪种工作模式，使用 TThreadedSelectorServer 就可以。