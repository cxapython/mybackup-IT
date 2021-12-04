> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/houjixin/article/details/42779835)

****相关示例代码见：****http://download.csdn.net/detail/hjx_1000/8374829**  
**

**三、  Thrift 的工作原理**

**1. 普通的本地函数调用过程**

例如，有如下关于本地函数的调用的 java 代码，在函数 caller 中调用函数 getStr 获取两个字符串的拼接结果：

![](https://img-blog.csdn.net/20150116172708315?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaG91aml4aW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

代码 3.1

 **本地函数调用调用方和被调用方都在一个程序内部**，只是 cpu 在执行调用的时候切换去执行被调用的函数，执行完被调用函数之后，再切换回来执行调用之后的代码，其调用过程如下图 3.1 所示：

![](https://img-blog.csdn.net/20150116172837359?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaG91aml4aW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

图 3.1

 站在调用方的角度，在本地函数调用过程中，执行被调用函数期间，调用方会被卡在那里一直等到被调用函数执行完，然后再继续执行剩下的代码。

**2.Thrift 的 RPC 调用过程**

 **远程过程调用（RPC）的调用方和被调用方不在同一个进程内，甚至都不在同一台机子上**，因此远程过程调用中，必然涉及网络传输；假设有如下代码 3.2 所示的客户端代码：

![](https://img-blog.csdn.net/20150116173003098?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaG91aml4aW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

代码 3.2

上述代码含义在 “二” 中已经有详细解释，其调用过程如下图 3.2 所示：

![](https://img-blog.csdn.net/20150116173044625?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaG91aml4aW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

图 3.2 thrift 的 RPC 调用过程

Thrift 框架的远程过程调用的工作过程如下：

 **(1)** 通过 IDL 定义一个接口的 thrift 文件，然后通过 thrift 的多语言编译功能，将接口定义的 thrift 文件翻译成对应的语言版本的接口文件；

 **(2)** Thrift 生成的特定语言的接口文件中包括客户端部分和服务器部分；

 **(3)** 客户端通过接口文件中的客户端部分生成一个 Client 对象，这个客户端对象中包含所有接口函数的存根实现，然后用户代码就可以通过这个 Client 对象来调用 thrift 文件中的那些接口函数了，但是，客户端调用接口函数时实际上调用的是接口函数的本地存根实现，如图 3.2 中的箭头 1 所示；

 **(4)** 接口函数的存根实现将调用请求发送给 thrift 服务器端，然后 thrift 服务器根据调用的函数名和函数参数，调用实际的实现函数来完成具体的操作，如图 3.2 中的箭头 2 所示；

 **(5)**Thrift 服务器在完成处理之后，将函数的返回值发送给调用的 Client 对象；如图 3.2 中的箭头 3 所示；

(6) Thrift 的 Client 对象将函数的返回值再交付给用户的调用函数，如图 3.2 中的箭头 4 所示；

**说明：**

        [1] 本地函数的调用方和被调方在同一进程的地址空间内部，因此在调用时 cpu 还是由当前的进行所持有，只是在调用期间，cpu 去执行被调用函数，从而导致调用方被卡在那里，直到 cpu 执行完被调用函数之后，才能切换回来继续执行调用之后的代码；

        [2]RPC 在调用方和被调用方一般不在一台机子上，它们之间通过网络传输进行通信，一般的 RPC 都是采用 tcp 连接，如果同一条 tcp 连接同一时间段只能被一个调用所独占，这种情况与 [1] 中的本地过程更为相似，这种情况是**同步调用**，很显然，这种方式通信的效率比较低，因为服务函数执行期间，tcp 连接上没有数据传输还依然被本次调用所霸占；另外，这种方式也有优点：实现简单。

        [3] 在一些的 RPC 服务框架中，为了提升网络通信的效率，客户端发起调用之后不被阻塞，这种情况是**异步调用**，它的通信效率比同步调用高，但是实现起来比较复杂。

**二、  Thrift 源码分析**

        源码分析主要分析 thrift 生成的 java 接口文件，并以 TestThriftService.java 为例，以该文件为线索，逐渐分析文件中遇到的其他类和文件；在 thrift 生成的服务接口文件中，共包含以下几部分：

        （1）异步客户端类 AsyncClient 和异步接口 AsyncIface，本节暂不涉及这些异步操作相关内容；

        （2）同步客户端类 Client 和同步接口 Iface，Client 类继承自 TServiceClient，并实现了同步接口 Iface；Iface 就是根据 thrift 文件中所定义的接口函数所生成；Client 类是在开发 Thrift 的客户端程序时使用，Client 类是 Iface 的客户端存根实现， Iface 在开发 Thrift 服务器的时候要使用，Thrift 的服务器端程序要实现接口 Iface。

        （3）Processor 类，该类主要是开发 Thrift 服务器程序的时候使用，该类内部定义了一个 map，它保存了所有函数名到函数对象的映射，一旦 Thrift 接到一个函数调用请求，就从该 map 中根据函数名字找到该函数的函数对象，然后执行它；

        （4）参数类，为每个接口函数定义一个参数类，例如：为接口 getInt 产生一个参数类：getInt_args，一般情况下，接口函数参数类的命名方式为：接口函数名_args;

        （5）返回值类，每个接口函数定义了一个返回值类，例如：为接口 getInt 产生一个返回值类：getInt_result，一般情况下，接口函数返回值类的命名方式为：接口函数名_result;

        参数类和返回值类中有对数据的读写操作，在参数类中，将按照协议类将调用的函数名和参数进行封装，在返回值类中，将按照协议规定读取数据。

        Thrift 调用过程中，Thrift 客户端和服务器之间主要用到传输层类、协议层类和处理类三个主要的核心类，这三个类的相互协作共同完成 rpc 的整个调用过程。在调用过程中将按照以下顺序进行协同工作：

        （1）     将客户端程序调用的函数名和参数传递给协议层（TProtocol），协议层将函数名和参数按照协议格式进行封装，然后封装的结果交给下层的传输层。此处需要注意：要与 Thrift 服务器程序所使用的协议类型一样，否则 Thrift 服务器程序便无法在其协议层进行数据解析；

        （2）     传输层（TTransport）将协议层传递过来的数据进行处理，例如传输层的实现类 TFramedTransport 就是将数据封装成帧的形式，即 “数据长度 + 数据内容”，然后将处理之后的数据通过网络发送给 Thrift 服务器；此处也需要注意：要与 Thrift 服务器程序所采用的传输层的实现类一致，否则 Thrift 的传输层也无法将数据进行逆向的处理；

        （3）     Thrift 服务器通过传输层（TTransport）接收网络上传输过来的调用请求数据，然后将接收到的数据进行逆向的处理，例如传输层的实现类 TFramedTransport 就是将 “数据长度 + 数据内容” 形式的网络数据，转成只有数据内容的形式，然后再交付给 Thrift 服务器的协议类（TProtocol）；

        （4）     Thrift 服务端的协议类（TProtocol）将传输层处理之后的数据按照协议进行解封装，并将解封装之后的数据交个 Processor 类进行处理；

        （5）     Thrift 服务端的 Processor 类根据协议层（TProtocol）解析的结果，按照函数名找到函数名所对应的函数对象；

        （6）     Thrift 服务端使用传过来的参数调用这个找到的函数对象；

        （7）     Thrift 服务端将函数对象执行的结果交给协议层；

        （8）     Thrift 服务器端的协议层将函数的执行结果进行协议封装；

        （9）     Thrift 服务器端的传输层将协议层封装的结果进行处理，例如封装成帧，然后发送给 Thrift 客户端程序；

        （10）    Thrift 客户端程序的传输层将收到的网络结果进行逆向处理，得到实际的协议数据；

        （11）    Thrift 客户端的协议层将数据按照协议格式进行解封装，然后得到具体的函数执行结果，并将其交付给调用函数；

上述过程如图 4.1 所示：

![](https://img-blog.csdn.net/20150116173311796?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaG91aml4aW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

图 4.1、调用过程中 Thrift 的各类协同工作过程

       上图 4.1 的客户端协议类和服务端协议类都是指具体实现了 TProtocol 接口的协议类，在实际开发过程中二者必须一样，否则便不能进行通信；同样，客户端传输类和服务端传输类是指 TTransport 的子类，二者也需保持一致；

       在上述开发 thrift 客户端和服务器端程序时需要用到三个类：传输类（TTransport）、协议接口（TProtocol）和处理类（Processor），其中 TTransport 是抽象类，在实际开发过程中可根据具体清空选择不同的实现类；TProtocol 是个协议接口，每种不同的协议都必须实现此接口才能被 thrift 所调用。例如 TProtocol 类的实现类就有 TBinaryProtocol 等；在 Thrift 生成代码的内部，还需要将待传输的内容封装成消息类 TMessage。处理类（Processor）主要在开发 Thrift 服务器端程序的时候使用。

**1.      TMessage**

       Thrift 在客户端和服务器端传递数据的时候（包括发送调用请求和返回执行结果），都是将数据按照 TMessage 进行组装，然后发送；TMessage 包括三部分：消息的名称、消息的序列号和消息的类型, 消息名称为字符串类型，消息的序列号为 32 位的整形，消息的类型为 byte 类型，消息的类型共有如下 17 种：

**public****final class**TType {

 **public staticfinal byte**_STOP_   =0;

 **public staticfinal byte**_VOID_   =1;

 **public staticfinal byte**_BOOL_   =2;

 **public staticfinal byte**_BYTE_   =3;

 **public staticfinal byte**_DOUBLE_ = 4;

 **public staticfinal byte**_I16_    =6;

 **public staticfinal byte**_I32_    =8;

 **public staticfinal byte**_I64_    =10;

 **public staticfinal byte**_STRING_ = 11;

 **public staticfinal byte**_STRUCT_ = 12;

 **public staticfinal byte**_MAP_    =13;

 **public staticfinal byte**_SET_    =14;

 **public staticfinal byte**_LIST_   =15;

 **public staticfinal byte**_ENUM_   =16;

}

Byte 共可表示 0~255 个数字，这里只是用了前 17 个

**2.      传输类（TTransport）**

       传输类或其各种实现类，都是对 I/O 层的一个封装，可更直观的理解为它封装了一个 socket，不同的实现类有不同的封装方式，例如 TFramedTransport 类，它里面还封装了一个读写 buf，在写入的时候，数据都先写到这个 buf 里面，等到写完调用该类的 flush 函数的时候，它会将写 buf 的内容，封装成帧再发送出去；

       TFramedTransport 是对 TTransport 的继承，由于 tcp 是基于字节流的方式进行传输，因此这种基于帧的方式传输就要求在无头无尾的字节流中每次写入和读出一个帧，TFramedTransport 是按照下面的方式来组织帧的：每个帧都是按照 4 字节的帧长加上帧的内容来组织，帧内容就是我们要收发的数据，如下：

  +---------------+---------------+

  |   4 字节的帧长  |   帧的内容       |

  +---------------+---------------+

**3.      协议接口（TProtocol）**

       提供了一组操作协议接口，主要用于规定采用哪种协议进行数据的读写，它内部包含一个传输类（TTransport）成员对象，通过 TTransport 对象从输入输出流中读写数据；它规定了很多读写方式，例如：

readByte()

readDouble()

readString()

…

       每种实现类都根据自己所实现的协议来完成 TProtocol 接口函数的功能，例如实现了 TProtocol 接口的 TBinaryProtocol 类，对于 readDouble() 函数就是按照二进制的方式读取出一个 Double 类型的数据。

       类 TBinaryProtocol 是 TProtocol 的一个实现类，TBinaryProtocol 协议规定采用这种协议格式的进行消息传输时，需要为消息内容封装一个首部，TBinaryProtocol 协议的首部有两种操作方式：一种是严格读写模式，一种值普通的读写模式；这两种模式下消息首部的组织方式不一样，在创建时也可以自己指定使用那种模式，但是要注意，如果要指定模式，Thrift 的服务器端和客户端都需要指定。

       严格读写模型下的消息首部的前 16 字节固定为版本号：0x8001，如图 4.2 所示；

![](https://img-blog.csdn.net/20150116173517720?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaG91aml4aW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

图 4.2 二进制协议中严格读写模式下的消息组织方式

       在严格读写模式下，首部中前 32 字节包括固定的 16 字节协议版本号 0x8001，8 字节的 0x00，8 字节的消息类型；然后是若干字节字符串格式的消息名称，消息名称的组织方式也是 “长度 + 内容” 的方式；再下来是 32 位的消息序列号；在序列号之后的才是消息内容。

       普通读写模式下，没有版本信息，首部的前 32 字节就是消息的名称，然后是消息的名字，接下来是 32 为的消息序列号，最后是消息的内容。

![](https://img-blog.csdn.net/20150116173506750?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvaG91aml4aW4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

图 4.3 二进制协议中普通读写模式下的消息组织方式

       通信过程中消息的首部在 TBinaryProtocol 类中进行通过 readMessageBegin 读取，通过 writeMessageBegin 写入；但是消息的内容读取在返回值封装类（例如：getStr_result）中进行；

**（1）     TBinaryProtocol 的读取数据过程：**

在 Client 中调用 TBinaryProtocol 读取数据的过程如下：

> readMessageBegin()
> 
> …
> 
> 读取数据
> 
> …
> 
> readMessageEnd()

readMessageBegin 详细过程如下：

      [1] 首先从传输过来的网络数据中读取 32 位数据，然后根据首位是否为 1 来判断当前读到的消息是严格读写模式还是普通读写模式；如果是严格读写模式则这 32 位数据包括版本号和消息类型，否则这 32 位保存的是后面的消息名称

      [2]读取消息的名称，如果是严格读写模式，则消息名称为字符串格式，保存方式为 “长度 + 内容” 的方式，如果是普通读写模式，则消息名称的长度直接根据 [1] 中读取的长度来读取消息名称；

      [3]读取消息类型，如果是严格读写模式，则消息类型已经由 [1] 中读取出来了，在其 32 位数据中的低 8 位中保存着；如果是普通读写模式，则要继续读取一字节的消息类型；

      [4] 读取 32 为的消息序列号；

      读取数据的过程是在函数返回值的封装类中来完成，根据读取的数值的类型来具体读取数据的内容；在 TBinaryProtocol 协议中 readMessageEnd 函数为空，什么都没有干。

**（2）     TBinaryProtocol 的写入数据过程：**

      在 sendBase 函数调用 TBinaryProtocol 将调用函数和参数发送到 Thrift 服务器的过程如下：

> writeMessageBegin(TMessage m)
> 
> …
> 
> 写入数据到 TTransport 实现类的 buf 中
> 
> …
> 
> writeMessageEnd();
> 
> getTransport().flush();

writeMessageBegin 函数需要一个参数 TMessage 作为消息的首部，写入过程与读取过程类似，首先判断需要执行严格读写模式还是普通读写模式，然后分别按照读写模式的具体消息将消息首部写入 TBinaryProtocol 的 TTransport 成员的 buf 中；

**4.      Thrift 客户端存根代码追踪调试**

      下面通过跟踪附件中 **thrift 客户端**代码的 test() 函数，在该函数中调用了 Thrift 存根函数 getStr，通过追踪该函数的执行过程来查看整个 Thrift 的调用流程：

      （1）客户端代码先打开 socket，然后调用存根对象的

m_transport.open();

String res = testClient.getStr("test1","test2");

      （2）在 getStr 的存根实现中，首先发送调用请求，然后等待 Thrift 服务器端返回的结果：

send_getStr(srcStr1, srcStr2);

return recv_getStr();

      （3）发送调用请求函数 send_getStr 中主要将参数存储到参数对象中，然后把参数和函数名发送出去：

getStr_args args = new getStr_args();// 创建一个该函数的参数对象

args.setSrcStr1(srcStr1);// 将参数值设置带参数对象中

args.setSrcStr2(srcStr2);

sendBase("getStr", args);// 将函数名和参数对象发送出去

      （4）sendBase 函数，存根类 Client 继承自基类 TServiceClient，sendBase 函数即是在 TServiceClient 类中实现的，它的主要功能是调用协议类将调用的函数名和参数发送给 Thrift 服务器：

oprot_.writeMessageBegin(new TMessage(methodName,TMessageType.CALL, ++seqid_));// 将函数名，消息类型，序号等信息存放到 oprot_的 TFramedTransport 成员的 buf 中

args.write(oprot_);// 将参数存放到 oprot_的 TFramedTransport 成员的 buf 中

oprot_.writeMessageEnd();

oprot_.getTransport().flush();// 将 oprot_的 TFramedTransport 成员的 buf 中的存放的消息发送出去；

      这里的 oprot_就是在 TProtocol 的子类，本例中使用的是 TBinaryProtocol，在调用 TBinaryProtocol 的函数时需要传入一个 TMessage 对象（在本节第 2 小节中有对 TMessage 的描述），这个 TMessage 对象的名字就是调用函数名，消息的类型为 TMessageType._CALL_，调用序号使用在客户端存根类中（实际上是在基类 TServiceClient）中保存的一个序列号，每次调用时均使用此序列号，使用完再将序号加 1。

      在 TBinaryProtocol 中包含有一个 TFramedTransport 对象，而 TFramedTransport 对象中维护了一个缓存，上述代码中，写入函数名、参数的时候都是写入到 TFramedTransport 中所维护的那个缓存中，在调用 TFramedTransport 的 flush 函数的时候，flush 函数首先计算缓存中数据的长度，将长度和数据内容组装成帧，然后发送出去，帧的格式按照 “长度 + 字段” 的方式组织，如：

  +---------------+---------------+

  |   4 字节的帧长  |    帧的内容       |

  +---------------+---------------+

      （5）recv_getStr，在调用 send_getStr 将调用请求发送出去之后，存根函数 getStr 中将调用 recv_getStr 等待 Thrift 服务器端返回调用结果，recv_getStr 的代码为：

getStr_resultresult = new getStr_result();// 为接收返回结果创建一个返回值对象

  receiveBase(result, "getStr");// 等待 Thrift 服务器将结果返回

      （6）receiveBase，在该函数中，首先通过协议层读取消息的首部，然后由针对 getStr 生成的返回值类 getStr_result 读取返回结果的内容；最后由协议层对象结束本次消息读取操作；如下所示：

> iprot_.readMessageBegin();// 通过协议层对象读取消息的首部
> 
> ……
> 
> result.read(iprot_);// 通过返回值类对象读取具体的返回值；
> 
> ……
> 
> iprot_.readMessageEnd();// 调用协议层对象结束本次消息读取

在本节第 4 小节中有对 readMessageBegin 函数的描述；

**5.           处理类（Processor）**

      该类主要由 Thrift 服务器端程序使用，它是由 thrift 编译器根据 IDL 编写的 thrift 文件生成的具体语言的接口文件中所包含的类，例如 2.5 节中提到的 TestThriftService.java 文件，处理类（Processor）主要由 thrift 服务器端使用，它继承自基类 TBaseProcessor。

例如，2.5 节中提到服务器端程序的如下代码：

TProcessor tProcessor =

**New**TestThriftService.Processor<TestThriftService.Iface>(_m_myService_);

      这里的 TestThriftService.Processor 就是这里提到的 Processor 类，包括尖括号里面的接口 TestThriftService.Iface 也是由 thrift 编译器自动生成。Processor 类主要完成函数名到对应的函数对象的映射，它内部维护一个 map，map 的 key 就是接口函数的名字，而 value 就是接口函数所对应的函数对象，这样服务器端从网络中读取到客户端发来的函数名字的时候，就通过这个 map 找到该函数名对应的函数对象，然后再用客户端发来的参数调用该函数对象；在 Thrift 框架中，每个接口函数都有一个函数对象与之对应，这里的函数对象继承自虚基类 ProcessFunction。

      ProcessFunction 类，它采用类似策略模式的实现方法，该类有一个字符串的成员变量，用于存放该函数对象对应的函数名字，在 ProcessFunction 类中主要实现了 process 方法，此方法的功能是通过协议层从传输层中读取并解析出调用的参数，然后再由具体的函数对象提供的 getResult 函数计算出结果；每个继承自虚基类 ProcessFunction 的函数对象都必须实现这个 getResult 函数，此函数内部根据函数调用参数，调用服务器端的函数，并获得执行结果；process 在通过 getResult 函数获取到执行结果之后，通过协议类对象将结果发送给 Thrift 客户端程序。

      Thrift 服务器端程序在使用 Thrrift 服务框架时，需要提供以下几个条件：

      （1）定义一个接口函数实现类的对象，在开发 Thrift 服务程序时，最主要的功能就是开发接口的实现函数，这个接口函数的实现类 **implements** 接口 Iface，并实现了接口中所有函数；

      （2）创建一个监听 socket，Thrift 服务框架将从此端口监听新的调用请求到来；

      （3）创建一个实现了 TProtocol 接口的协议类对象，在与 Thrift 客户端程序通信时将使用此协议进行网络数据的封装和解封装；

      （4）创建一个传输类的子类，用于和 Thrift 服务器之间进行数据传输；