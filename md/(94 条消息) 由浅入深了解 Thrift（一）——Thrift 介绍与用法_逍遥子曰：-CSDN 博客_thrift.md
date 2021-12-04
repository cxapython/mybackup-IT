> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [xiaoyaozi.blog.csdn.net](https://xiaoyaozi.blog.csdn.net/article/details/42778335)

> 相关示例代码见：http://download.csdn.net/detail/hjx_1000/8374829 一、Thrift 简单介绍 1.1、 Thrift 是什么？能做什么？Thrift 是 Facebo......

****相关示例代码见：****http://download.csdn.net/detail/hjx_1000/8374829****

**一、  Thrift 简单介绍**

**1.1、  Thrift 是什么？能做什么？**

Thrift 是 Facebook 于 2007 年开发的跨语言的 rpc 服框架，提供多语言的编译功能，并提供多种服务器工作模式；用户通过 Thrift 的 IDL（接口定义语言）来描述接口函数及数据类型，然后通过 Thrift 的编译环境生成各种语言类型的接口文件，用户可以根据自己的需要采用不同的语言开发客户端代码和服务器端代码。

例如，我想开发一个快速计算的 RPC 服务，它主要通过接口函数 getInt 对外提供服务，这个 RPC 服务的 getInt 函数使用用户传入的参数，经过复杂的计算，计算出一个整形值返回给用户；服务器端使用 java 语言开发，而调用客户端可以是 java、c、python 等语言开发的程序，在这种应用场景下，我们只需要使用 Thrift 的 IDL 描述一下 getInt 函数（以. thrift 为后缀的文件），然后使用 Thrift 的多语言编译功能，将这个 IDL 文件编译成 C、java、python 几种语言对应的 “特定语言接口文件”（每种语言只需要一条简单的命令即可编译完成），这样拿到对应语言的“特定语言接口文件” 之后，就可以开发客户端和服务器端的代码了，开发过程中只要接口不变，客户端和服务器端的开发可以独立的进行。

Thrift 为服务器端程序提供了很多的工作模式，例如：线程池模型、非阻塞模型等等，可以根据自己的实际应用场景选择一种工作模式高效地对外提供服务；

**1.2、  Thrift 的相关网址和资料：**

（1）  Thrift 的官方网站：[http://thrift.apache.org/](http://thrift.apache.org/)

（2）  Thrift 官方下载地址：[http://thrift.apache.org/download](http://thrift.apache.org/download)

（3）  Thrift 官方的 IDL 示例文件（自己写 IDL 文件时可以此为参考）：

[https://git-wip-us.apache.org/repos/asf?p=thrift.git;a=blob_plain;f=test/ThriftTest.thrift;hb=HEAD](https://git-wip-us.apache.org/repos/asf?p=thrift.git;a=blob_plain;f=test/ThriftTest.thrift;hb=HEAD)

（4）  各种环境下搭建 Thrift 的方法：

[http://thrift.apache.org/docs/install/](http://thrift.apache.org/docs/install/)

该页面中共提供了 CentOS\Ubuntu\OS X\Windows 几种环境下的搭建 Thrift 环境的方法。

**二、  Thrift 的使用**

Thrift 提供跨语言的服务框架，这种跨语言主要体现在它对多种语言的编译功能的支持，用户只需要使用 IDL 描述好接口函数，只需要一条简单的命令，Thrift 就能够把按照 IDL 格式描述的接口文件翻译成各种语言版本。其实，说搭建 Thrift 环境的时候，实际上最麻烦的就是搭建 Thrift 的编译环境，Thrift 的编译和通常的编译一样经过词法分析、语法分析等等最终生成对应语言的源码文件，为了能够支持对各种语言的编译，你需要下载各种语言对应的编译时使用的包；

**2.1、  搭建 Thrift 的编译环境**

本节主要介绍如何搭建 Unix 编译环境，搭建时有以下要求：

**基本要求：**

G++、boost、lex、yacc

**源码安装要求：**

如果使用源码安装的方式，则还需要下列工具：

Autoconf、automake、libtool、pkg-config、lex 和 yacc 的开发版、libssl-dev

**语言要求：**

搭建 C++ 编译环境：boost、libevent、zlib

搭建 java 编译环境：jdk、ApacheAnt

具体搭建环境时可以参考 “一” 中所列官网的安装方法。

**2.2、  搭建 JAVA 下 Thrift 开发环境**

在 java 环境下开发 thrift 的客户端或者服务器程序非常简单，只需在工程文件中加上下面三个 jar 包（版本可能随时有更新）：

  libthrift-0.9.1.jar

  slf4j-api-1.7.5.jar

             slf4j-simple.jar

**2.3、  编写 IDL 文件**

使用 Thrift 开发程序，首先要做的事情就是使用 IDL 对接口进行描述， 然后再使用 Thrift 的多语言编译能力将接口的描述文件编译成对应语言的版本，本文中将 IDL 对接口的描述文件称为 “Thrift 文件”。

**（1）  编写 Thrift 文件**

使用 IDL 对接口进行描述的 thrift 文件命名一般都是以 “.thrift” 作为后缀：XXX.thrift，可以在该文件的开头为该文件加上命名空间限制，格式为：namespace 语言 命名空间的名字；例如：

**namespace javacom.test.service**

IDL 文件中对所有接口函数的描述都放在 service 中，service 的名字可以自己指定，该名字也将被用作生成的特定语言接口文件的名字，接口函数需要对参数使用序号标号，除最后一个接口函数外，要以 “，” 结束对函数的描述。

例如，下面一个 IDL 描述的 Thrift 文件（该 Thrift 文件的文件名为：test_service.thrift）的全部内容：

```
namespace java com.test.service

include "thrift_datatype.thrift"

service TestThriftService
{

	
	thrift_datatype.ResultStr getStr(1:string srcStr1, 2:string srcStr2),
	
	thrift_datatype.ResultInt getInt(1:i32 val)
	
}


```

代码 2.1

这里的 TestThriftService 就被用作生成的特定语言的文件名，例如我想用该 Thrift 文件生成一个 java 版本的接口文件，那么生成的 java 文件名就是：TestThriftService.java。

**（1）  编写 IDL 文件时需要注意的问题**

[1] 函数的参数要用数字依序标好，序号从 1 开始，形式为：“序号: 参数名”;

[2] 每个函数的最后要加上 “,”，最后一个函数不加；

[3] 在 IDL 中可以使用 /*……*/ 添加注释

**（2）  IDL 支持的数据类型**

IDL 大小写敏感，它共支持以下几种基本的数据类型：

[1]**string**， 字符串类型，注意是全部小写形式；例如：string aString

[2]**i16**, 16 位整形类型，例如：i16 aI16Val;

[3]**i32**，32 位整形类型，对应 C/C++/java 中的 int 类型；例如：      I32  aIntVal

[4]**i64**，64 位整形，对应 C/C++/java 中的 long 类型；例如：I64 aLongVal

[5]**byte**，8 位的字符类型，对应 C/C++ 中的 char，java 中的 byte 类型；例如：byte aByteVal

[6]**bool**, 布尔类型，对应 C/C++ 中的 bool，java 中的 boolean 类型； 例如：bool aBoolVal

[7]**double**，双精度浮点类型，对应 C/C++/java 中的 double 类型；例如：double aDoubleVal

[8]**void**，空类型，对应 C/C++/java 中的 void 类型；该类型主要用作函数的返回值，例如：void testVoid(),

**除上述基本类型外，ID 还支持以下类型：**

[1]**map**，map 类型，例如，定义一个 map 对象：map<i32, i32> newmap;

[2]**set**，集合类型，例如，定义 set<i32> 对象：set<i32> aSet;

[3]**list**，链表类型，例如，定义一个 list<i32> 对象：list<i32> aList;

**（3）  在 Thrift 文件中自定义数据类型**

在 IDL 中支持两种自定义类型：枚举类型和结构体类型，具体如下：

[1]**enum**， 枚举类型，例如，定义一个枚举类型：

```
enum Numberz
{
  ONE = 1,
  TWO,
  THREE,
  FIVE = 5,
  SIX,
  EIGHT = 8
}

```

注意，枚举类型里没有序号

[2]struct，自定义结构体类型，在 IDL 中可以自己定义结构体，对应 C 中的 struct，c++ 中的 struct 和 class，java 中的 class。例如：

```
struct TestV1 {
       1: i32 begin_in_both,
       3: string old_string,
       12: i32 end_in_both
}

```

注意，在 struct 定义结构体时需要对每个结构体成员用序号标识：“序号:”。

**（4）  定义类型别名**

Thrift 的 IDL 支持 C/C++ 中类似 typedef 的功能，例如：

`typedef``i32  Integer` 

`就可以为i32类型重新起个名字Integer。`

**2.4、  生成 Thrift 服务接口文件**

搭建 Thrift 编译环境之后，使用下面命令即可将 IDL 文件编译成对应语言的接口文件：

**thrift --gen <language> <Thrift filename>**

例如：如果使用上面的 thrift 文件（见上面的代码 2.1）：test_service.thrift 生成一个 java 语言的接口文件，则只需在搭建好 thrift 编译环境的机子上，执行如下命令即可：

thrift --gen **java** test_service.thrift

这里，我直接在 test_service.thrift 文件所在的目录下执行的命令，所以直接使用文件名即可（如图 2.1 的标号 1 所示），如果不在 test_service.thrift 所在的目录中，则需要具体指明该文件所在的路径。

![](https://img-blog.csdn.net/20150116160957791?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaG91aml4aW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

图 2.1 

如图 2.1 中标号 2 所示，生成的 gen-java 的目录，目录下面有 com、test、service 三级目录，这三级目录也是根据 test_service.thrift 文件中命名空间的名字：com.test.service 生成的，进入目录之后可以看到生成的 java 语言的接口文件名为：TestThriftService.java，这个文件的名字也是根据 test_service.thrift 文件的 service 名字来生成的（见代码 2.1）。

**2.5、  编写服务器端的 java 代码**

编写 thrift 服务器程序需要首先完成下面两步工作：

（1）先将 2.2 节中的三个 jar 包添加到工程里，如图 2.2 的标号 2 所示。

（2）将生成的 java 接口文件 TestThriftService.java 拷贝到自己的工程文件中，如图 2.2 的标号 1 所示。

![](https://img-blog.csdn.net/20150116161125812?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaG91aml4aW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

图 2.2

服务端程序需实现 TestThriftService.Iface 接口，在实现接口中完成自己要提供的服务，服务器端对服务接口实现的代码如下所示：

```
package com.test.service;

import org.apache.thrift.TException;

public class TestThriftServiceImpl implements TestThriftService.Iface

public String getStr(String srcStr1, String srcStr2) throws TException {

long startTime = System.currentTimeMillis();

String res = srcStr1 + srcStr2; 

long stopTime = System.currentTimeMillis();

		System.out.println("[getStr]time interval: " + (stopTime-startTime));

public int getInt(int val) throws TException {

long startTime = System.currentTimeMillis();

long stopTime = System.currentTimeMillis();

		System.out.println("[getInt]time interval: " + (stopTime-startTime));
```

代码 2.2

服务器端启动 thrift 服务框架的程序如下所示，在本例中服务器采用 TNonblockingServer 工作模式：

```
package com.test.service;

import org.apache.thrift.TProcessor;

import org.apache.thrift.protocol.TBinaryProtocol;

import org.apache.thrift.server.TNonblockingServer;

import org.apache.thrift.server.TServer;

import org.apache.thrift.transport.TFramedTransport;

import org.apache.thrift.transport.TNonblockingServerSocket;

import org.apache.thrift.transport.TTransportException;

private static int m_thriftPort = 12356;

private static TestThriftServiceImpl m_myService = new TestThriftServiceImpl();

private static TServer m_server = null;

private static void createNonblockingServer() throws TTransportException

TProcessor tProcessor = new TestThriftService.Processor<TestThriftService.Iface>(m_myService);

TNonblockingServerSocket nioSocket = new TNonblockingServerSocket(m_thriftPort);

		TNonblockingServer.Args tnbArgs = new TNonblockingServer.Args(nioSocket);

		tnbArgs.processor(tProcessor);

		tnbArgs.transportFactory(new TFramedTransport.Factory());

		tnbArgs.protocolFactory(new TBinaryProtocol.Factory());

		m_server = new TNonblockingServer(tnbArgs);

public static boolean start()

			createNonblockingServer();

		} catch (TTransportException e) {

			System.out.println("start server error!" + e);

		System.out.println("service at port: " + m_thriftPort);

public static void main(String[] args)
```

代码 2.3

在服务器端启动 thrift 框架的部分代码比较简单，不过在写这些启动代码之前需要先确定服务器采用哪种工作模式对外提供服务，Thrift 对外提供几种工作模式，例如：TSimpleServer、TNonblockingServer、TThreadPoolServer、TThreadedSelectorServer 等模式，每种服务模式的通信方式不一样，因此在服务启动时使用了那种服务模式，客户端程序也需要采用对应的通信方式。

另外，Thrift 支持多种通信协议格式：TCompactProtocol、TBinaryProtocol、TJSONProtocol 等，因此，在使用 Thrift 框架时，客户端程序与服务器端程序所使用的通信协议一定要一致，否则便无法正常通信。

以上述代码 2.3 采用的 TNonblockingServer 为例，说明服务器端如何使用 Thrift 框架，在服务器端创建并启动 Thrift 服务框架的过程为：

[1] 为自己的服务实现类定义一个对象，如代码 2.3 中的：

TestThriftServiceImpl_m_myService_ =**new**TestThriftServiceImpl();

这里的 TestThriftServiceImpl 类就是代码 2.2 中我们自己定义的服务器端对各服务接口的实现类。

[2] 定义一个 TProcess 对象，在根据 Thrift 文件生成 java 源码接口文件 TestThriftService.java 中，Thrift 已经自动为我们定义了一个 Processor；后续节中将对这个 TProcess 类的功能进行详细描述；如代码 2.3 中的：

TProcessor tProcessor = **New**TestThriftService.Processor<TestThriftService.Iface>(_m_myService_);

[3] 定义一个 TNonblockingServerSocket 对象，用于 tcp 的 socket 通信，如代码 2.3 中的：

TNonblockingServerSocketnioSocket = newTNonblockingServerSocket(m_thriftPort);

在创建 server 端 socket 时需要指明监听端口号，即上面的变量：m_thriftPort。

[4] 定义 TNonblockingServer 所需的参数对象 TNonblockingServer.Args；并设置所需的参数，如代码 2.3 中的：

```
TNonblockingServer.Args tnbArgs = new TNonblockingServer.Args(nioSocket);

tnbArgs.processor(tProcessor);

tnbArgs.transportFactory(new TFramedTransport.Factory());

tnbArgs.protocolFactory(new TBinaryProtocol.Factory());
```

在 TNonblockingServer 模式下我们使用二进制协议：TBinaryProtocol, 通信方式采用 TFramedTransport，即以帧的方式对数据进行传输。

[5] 定义 TNonblockingServer 对象，并启动该服务，如代码 2.3 中的：

> m_server = new TNonblockingServer(tnbArgs);
> 
> …
> 
> m_server.serve();

**2.6、  编写客户端代码**

Thrift 的客户端代码同样需要服务器开头的那两步：添加三个 jar 包和生成的 java 接口文件 TestThriftService.java。

```
		 m_transport = new TSocket(THRIFT_HOST, THRIFT_PORT,2000);

TProtocol protocol = new TBinaryProtocol(m_transport);

		 TestThriftService.Client testClient = new TestThriftService.Client(protocol);

String res = testClient.getStr("test1", "test2");

			 System.out.println("res = " + res);
```

代码 2.4

由代码 2.4 可以看到编写客户端代码非常简单，只需下面几步即可：

[1] 创建一个传输层对象（TTransport），具体采用的传输方式是 TFramedTransport，要与服务器端保持一致，即：

m_transport =**new** TFramedTransport(**new**TSocket(_THRIFT_HOST_,_THRIFT_PORT_, 2000));

这里的 THRIFT_HOST, THRIFT_PORT 分别是 Thrift 服务器程序的主机地址和监听端口号，这里的 2000 是 socket 的通信超时时间；

[2] 创建一个通信协议对象（TProtocol），具体采用的通信协议是二进制协议，这里要与服务器端保持一致，即：

TProtocolprotocol =**new** TBinaryProtocol(m_transport);

[3] 创建一个 Thrift 客户端对象（TestThriftService.Client），Thrift 的客户端类 TestThriftService.Client 已经在文件 TestThriftService.java 中，由 Thrift 编译器自动为我们生成，即：

TestThriftService.ClienttestClient =**new** TestThriftService.Client(protocol);

[4] 打开 socket，建立与服务器直接的 socket 连接，即：

m_transport.open();

[5] 通过客户端对象调用服务器服务函数 getStr，即：

String res = testClient.getStr("test1","test2");

                System._out_.println("res =" +res);

[6] 使用完成关闭 socket，即：

m_transport.close();

        这里有以下几点需要说明：

[1] 在同步方式使用客户端和服务器的时候，socket 是被一个函数调用独占的，不能多个调用同时使用一个 socket，例如通过 m_transport.open() 打开一个 socket，此时创建多个线程同时进行函数调用，这时就会报错，因为 socket 在被一个调用占着的时候不能再使用；

[2] 可以分时多次使用同一个 socket 进行多次函数调用，即通过 m_transport.open() 打开一个 socket 之后，你可以发起一个调用，在这个次调用完成之后，再继续调用其他函数而不需要再次通过 m_transport.open() 打开 socket；

**2.7、  需要注意的问题**

（1）Thrift 的服务器端和客户端使用的通信方式要一样，否则便无法进行正常通信；

Thrift 的服务器端的种模式所使用的通信方式并不一样，因此，服务器端使用哪种通信方式，客户端程序也要使用这种方式，否则就无法进行正常通信了。例如，上面的代码 2.3 中，服务器端使用的工作模式为 TNonblockingServer，在该工作模式下需要采用的传输方式为 TFramedTransport，也就是在通信过程中会将 tcp 的字节流封装成一个个的帧，此时就需要客户端程序也这么做，否则便会通信失败。出现如下问题：

服务器端会爆出如下出错 log：

```
2015-01-06 17:14:52.365 ERROR [Thread-11] [AbstractNonblockingServer.java:348] - Read an invalid frame size of -2147418111. Are you using TFramedTransport on the client side?

```

客户端则会报出如下出错 log：

```
org.apache.thrift.transport.TTransportException: java.net.SocketException: Connection reset
	at org.apache.thrift.transport.TIOStreamTransport.read(TIOStreamTransport.java:129)
	at org.apache.thrift.transport.TTransport.readAll(TTransport.java:84)
	at org.apache.thrift.protocol.TBinaryProtocol.readAll(TBinaryProtocol.java:362)
	at org.apache.thrift.protocol.TBinaryProtocol.readI32(TBinaryProtocol.java:284)
	at org.apache.thrift.protocol.TBinaryProtocol.readMessageBegin(TBinaryProtocol.java:191)
	at org.apache.thrift.TServiceClient.receiveBase(TServiceClient.java:69)
	at com.browan.freepp.dataproxy.service.DataProxyService$Client.recv_addLiker(DataProxyService.java:877)
	at com.browan.freepp.dataproxy.service.DataProxyService$Client.addLiker(DataProxyService.java:862)
	at com.browan.freepp.dataproxy.service.DataProxyServiceTest.test_Likers(DataProxyServiceTest.java:59)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:606)
	at org.junit.runners.model.FrameworkMethod$1.runReflectiveCall(FrameworkMethod.java:47)
	at org.junit.internal.runners.model.ReflectiveCallable.run(ReflectiveCallable.java:12)
	at org.junit.runners.model.FrameworkMethod.invokeExplosively(FrameworkMethod.java:44)
	at org.junit.internal.runners.statements.InvokeMethod.evaluate(InvokeMethod.java:17)
	at org.junit.runners.ParentRunner.runLeaf(ParentRunner.java:271)
	at org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:70)
	at org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:50)
	at org.junit.runners.ParentRunner$3.run(ParentRunner.java:238)
	at org.junit.runners.ParentRunner$1.schedule(ParentRunner.java:63)
	at org.junit.runners.ParentRunner.runChildren(ParentRunner.java:236)
	at org.junit.runners.ParentRunner.access$000(ParentRunner.java:53)
	at org.junit.runners.ParentRunner$2.evaluate(ParentRunner.java:229)
	at org.junit.runners.ParentRunner.run(ParentRunner.java:309)
	at org.eclipse.jdt.internal.junit4.runner.JUnit4TestReference.run(JUnit4TestReference.java:50)
	at org.eclipse.jdt.internal.junit.runner.TestExecution.run(TestExecution.java:38)
	at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.runTests(RemoteTestRunner.java:467)
	at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.runTests(RemoteTestRunner.java:683)
	at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.run(RemoteTestRunner.java:390)
	at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.main(RemoteTestRunner.java:197)
Caused by: java.net.SocketException: Connection reset
	at java.net.SocketInputStream.read(SocketInputStream.java:196)
	at java.net.SocketInputStream.read(SocketInputStream.java:122)
	at java.io.BufferedInputStream.fill(BufferedInputStream.java:235)
	at java.io.BufferedInputStream.read1(BufferedInputStream.java:275)
	at java.io.BufferedInputStream.read(BufferedInputStream.java:334)
	at org.apache.thrift.transport.TIOStreamTransport.read(TIOStreamTransport.java:127)
	... 31 more

```

（2）在服务器端或者客户端直接使用 IDL 生成的接口文件时，可能会遇到下面两个问题：

[1]、Cannotreduce the visibility of the inherited method fromProcessFunction<I,TestThriftService.getStr_args>

[2]、The typeTestThriftService.Processor<I>.getStr<I> must implement theinherited abstract methodProcessFunction<I,TestThriftService.getStr_args>.isOneway()

**问题产生的原因：**

问题 [1] 是继承类的访问权限缩小所造成的；

问题 [2] 是因为存在抽象函数 isOneWay 所致;

**解决办法：**

问题 [1] 的访问权限由 protected 修改为 public；

问题 [2] 的解决办法是为抽象函数添加一个空的函数体即可。

**2.8、  应用技巧**

**（1）  为调用加上一个事务 ID**

在分布式服务开发过程中，一次事件（事务）的执行可能跨越位于不同机子上多个服务程序，在后续维护过程中跟踪 log 将变得非常麻烦，因此在系统设计的时候，系统的一个事务产生之处应该产生一个系统唯一的事务 ID，该 ID 在各服务程序之间进行传递，让一次事务在所有服务程序输出的 log 都以此 ID 作为标识。

在使用 Thrift 开发服务器程序的时候，也应该为每个接口函数提供一个事务 ID 的参数，并且在服务器程序开发过程中，该 ID 应该在内部函数调用过程中也进行传递，并且在日志输出的时候都加上它，以便问题跟踪。

**（2）  封装返回结果**

Thrift 提供的 RPC 方式的服务，使得调用方可以像调用自己的函数一样调用 Thrift 服务提供的函数；在使用 Thrift 开发过程中，尽量不要直接返回需要的数据，而是将返回结果进行封装，例如上面的例子中的 getStr 函数就是直接返回了结果 string，见 Thrift 文件 test_service.thrift 中对该函数的描述：

stringgetStr(1:string srcStr1, 2:string srcStr2)

在实际开发过程中，这是一种很不好的行为，在返回结果为 null 的时候还可能造成调用方产生异常，需要对返回结果进行封装，例如：

```
struct ResultStr
{
  1: ThriftResult result,
  2: string value
}


```

其中 ThriftResult 是自己定义的枚举类型的返回结果，在这里可以根据自己的需要添加任何自己需要的返回结果类型：

```
enum ThriftResult
{
  SUCCESS,           
  SERVER_UNWORKING,  
  NO_CONTENT,  		 
  PARAMETER_ERROR,	 
  EXCEPTION,	 	 
  INDEX_ERROR,		 
  UNKNOWN_ERROR, 	 
  DATA_NOT_COMPLETE, 	 
  INNER_ERROR, 	 
}


```

此时可以将上述定义的 getStr 函数修改为：

ResultStr  getStr(1:string srcStr1, 2:string srcStr2)

在此函数中，任何时候都会返回一个 ResultStr 对象，无论异常还是正常情况，在出错时还可以通过 ThriftResult 返回出错的类型。

**（3）  将服务与数据类型分开定义**

在使用 Thrift 开发一些中大型项目的时候，很多情况下都需要自己封装数据结构，例如前面将返回结果进行封装的时候就定义了自己的数据类型 ResultStr，此时，将数据结构和服务分开定义到不通的文件中，可以增加 thrift 文件的易读性。例如：

在 thrift 文件：thrift_datatype.thrift 中定义数据类型，如：

```
namespace java com.browan.freepp.thriftdatatype
const string VERSION = "1.0.1"



enum ThriftResult
{
  SUCCESS,           
  SERVER_UNWORKING,  
  NO_CONTENT,  		 
  PARAMETER_ERROR,	 
  EXCEPTION,	 	 
  INDEX_ERROR,		 
  UNKNOWN_ERROR 	 
  DATA_NOT_COMPLETE 	 
  INNER_ERROR 	 
}


struct ResultBool 
{
  1: ThriftResult result,
  2: bool value
}


struct ResultInt
{
  1: ThriftResult result,
  2: i32 value
}


struct ResultStr
{
  1: ThriftResult result,
  2: string value
}


struct ResultLong
{
  1: ThriftResult result,
  2: i64 value
}




struct ResultDouble
{
  1: ThriftResult result,
  2: double value
}


struct ResultListStr 
{
  1: ThriftResult result,
  2: list<string> value
}


struct ResultSetStr 
{
  1: ThriftResult result,
  2: set<string> value
}


struct ResultMapStrStr 
{
  1: ThriftResult result,
  2: map<string,string> value
}


```

代码 2.5

在另外一个文件 test_service.thrift 中定义服务接口函数，如下所示：

```
namespace java com.test.service

include "thrift_datatype.thrift"

service TestThriftService
{

	
	thrift_datatype.ResultStr getStr(1:string srcStr1, 2:string srcStr2),
	
	thrift_datatype.ResultInt getInt(1:i32 val)
	
}


```

代码 2.6

由于在接口服务定义的 thrift 文件 test_service.thrift 中要用到对数据类型定义的 thrift 文件：thrift_datatype.thrift，因此需要在其文件前通过 include 把自己所使用的 thrift 文件包含进来，另外在使用其他 thrift 文件中定义的数据类型时要加上它的文件名，如：**thrift_datatype.**ResultStr

**（4）  为 Thrift 文件添加版本号**

在实际开发过程中，还可以为 Thrift 文件加上版本号，以方便对 thrift 的版本进行控制，如代码 2.5 所示。