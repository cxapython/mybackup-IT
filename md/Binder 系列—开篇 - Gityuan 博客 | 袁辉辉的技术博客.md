> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [gityuan.com](https://gityuan.com/2015/10/31/binder-prepare/)

> 袁辉辉, Gityuan, Android 博客, Android 源码, Flutter 博客，Flutter 源码

> 基于 Android 6.0 的源码剖析

一、概述
----

Android 系统中，每个应用程序是由 Android 的`Activity`，`Service`，`Broadcast`，`ContentProvider`这四剑客的中一个或多个组合而成，这四剑客所涉及的多进程间的通信底层都是依赖于 Binder IPC 机制。例如当进程 A 中的 Activity 要向进程 B 中的 Service 通信，这便需要依赖于 Binder IPC。不仅于此，整个 Android 系统架构中，大量采用了 Binder 机制作为 IPC（进程间通信）方案，当然也存在部分其他的 IPC 方式，比如 Zygote 通信便是采用 socket。

Binder 作为 Android 系统提供的一种 IPC 机制，无论从事系统开发还是应用开发，都应该有所了解，这是 Android 系统中最重要的组成，也是最难理解的一块知识点，错综复杂。要深入了解 Binder 机制，最好的方法便是阅读源码，借用 Linux 鼻祖 Linus Torvalds 曾说过的一句话：`Read The Fucking Source Code`。

二、 Binder
---------

### 2.1 IPC 原理

从进程角度来看 IPC 机制

![](https://gityuan.com/images/binder/prepare/binder_interprocess_communication.png)

每个 Android 的进程，只能运行在自己进程所拥有的虚拟地址空间。对应一个 4GB 的虚拟地址空间，其中 3GB 是用户空间，1GB 是内核空间，当然内核空间的大小是可以通过参数配置调整的。对于用户空间，不同进程之间彼此是不能共享的，而内核空间却是可共享的。Client 进程向 Server 进程通信，恰恰是利用进程间可共享的内核内存空间来完成底层通信工作的，Client 端与 Server 端进程往往采用 ioctl 等方法跟内核空间的驱动进行交互。

### 2.2 Binder 原理

Binder 通信采用 C/S 架构，从组件视角来说，包含 Client、Server、ServiceManager 以及 binder 驱动，其中 ServiceManager 用于管理系统中的各种服务。架构图如下所示：

![](https://gityuan.com/images/binder/prepare/IPC-Binder.jpg)

可以看出无论是注册服务和获取服务的过程都需要 ServiceManager，需要注意的是此处的 Service Manager 是指 Native 层的 ServiceManager（C++），并非指 framework 层的 ServiceManager(Java)。ServiceManager 是整个 Binder 通信机制的大管家，是 Android 进程间通信机制 Binder 的守护进程，要掌握 Binder 机制，首先需要了解系统是如何首次[启动 Service Manager](http://gityuan.com/2015/11/07/binder-start-sm/)。当 Service Manager 启动之后，Client 端和 Server 端通信时都需要先[获取 Service Manager](http://gityuan.com/2015/11/08/binder-get-sm/) 接口，才能开始通信服务。

图中 Client/Server/ServiceManage 之间的相互通信都是基于 Binder 机制。既然基于 Binder 机制通信，那么同样也是 C/S 架构，则图中的 3 大步骤都有相应的 Client 端与 Server 端。

1.  **[注册服务 (addService)](http://gityuan.com/2015/11/14/binder-add-service/)**：Server 进程要先注册 Service 到 ServiceManager。该过程：Server 是客户端，ServiceManager 是服务端。
2.  **[获取服务 (getService)](http://gityuan.com/2015/11/15/binder-get-service/)**：Client 进程使用某个 Service 前，须先向 ServiceManager 中获取相应的 Service。该过程：Client 是客户端，ServiceManager 是服务端。
3.  **使用服务**：Client 根据得到的 Service 信息建立与 Service 所在的 Server 进程通信的通路，然后就可以直接与 Service 交互。该过程：client 是客户端，server 是服务端。

图中的 Client,Server,Service Manager 之间交互都是虚线表示，是由于它们彼此之间不是直接交互的，而是都通过与 [Binder 驱动](http://gityuan.com/2015/11/01/binder-driver/)进行交互的，从而实现 IPC 通信方式。其中 Binder 驱动位于内核空间，Client,Server,Service Manager 位于用户空间。Binder 驱动和 Service Manager 可以看做是 Android 平台的基础架构，而 Client 和 Server 是 Android 的应用层，开发人员只需自定义实现 client、Server 端，借助 Android 的基本平台架构便可以直接进行 IPC 通信。

### 2.3 C/S 模式

BpBinder(客户端) 和 BBinder(服务端) 都是 Android 中 Binder 通信相关的代表，它们都从 IBinder 类中派生而来，关系图如下：

![](https://gityuan.com/images/binder/prepare/Ibinder_classes.jpg)

*   client 端：BpBinder.transact() 来发送事务请求；
*   server 端：BBinder.onTransact() 会接收到相应事务。

三、 提纲
-----

在后续的 Binder 源码分析过程中所涉及的源码，会有部分的精简，主要是去掉所有 log 输出语句，已减少代码篇幅过于长。通过前面的介绍，下面罗列一下关于 Binder 系列文章的提纲：

*   [Binder 系列 1—Binder Driver 初探](http://gityuan.com/2015/11/01/binder-driver/)
*   [Binder 系列 2—Binder Driver 再探](http://gityuan.com/2015/11/02/binder-driver-2/)
*   [Binder 系列 3—启动 Service Manager](http://gityuan.com/2015/11/07/binder-start-sm/)
*   [Binder 系列 4—获取 Service Manager](http://gityuan.com/2015/11/08/binder-get-sm/)
*   [Binder 系列 5—注册服务 (addService)](http://gityuan.com/2015/11/14/binder-add-service/)
*   [Binder 系列 6—获取服务 (getService)](http://gityuan.com/2015/11/15/binder-get-service/)
*   [Binder 系列 7—framework 层分析](http://gityuan.com/2015/11/21/binder-framework/)
*   [Binder 系列 8—如何使用 Binder](http://gityuan.com/2015/11/22/binder-use/)
*   [Binder 系列 9—如何使用 AIDL](http://gityuan.com/2015/11/23/binder-aidl/)
*   [Binder 系列 10—总结](http://gityuan.com/2015/11/28/binder-summary/)

文章是从底层驱动往上层写的，这并不适合大家的理解，建议读者还是从上层往底层看。下面说说这个系列文章之间的彼此联系，也是对你阅读顺序的一个建议，更好的建议，大家可以上微博跟 **[@Gityuan](http://weibo.com/gityuan)**，或许邮件跟我进行交流与反馈：

首先阅读 [Binder 系列 5—注册服务 (addService)](http://gityuan.com/2015/11/14/binder-add-service/) 和 [Binder 系列 6—获取服务 (getService)](http://gityuan.com/2015/11/15/binder-get-service/)，这两个过程都需要于 ServiceManager 打交道，那么这两个过程在开始之前都需要 [Binder 系列 4—获取 Service Manager](http://gityuan.com/2015/11/08/binder-get-sm/)，既然要获取 Service Manager，那么就需要先 [Binder 系列 3—启动 Service Manager](http://gityuan.com/2015/11/07/binder-start-sm/)。在看 Binder 服务的注册和获取这两个过程中，不断追溯下去，最终调用到底层 Binder 底层驱动，这时需要了解 [Binder 系列 1—Binder Driver 初探](http://gityuan.com/2015/11/01/binder-driver/)和 [Binder 系列 2—Binder Driver 再探](http://gityuan.com/2015/11/02/binder-driver-2/)。

看完 Binder 系列 1~ 系列 6，那么对 Binder 的整个流程会有一个比较清晰的认知，这还只是停留在 Native 层 (C/C++)。接下来，可以看看上层 [Binder 系列 7—framework 层分析](http://gityuan.com/2015/11/21/binder-framework/)的 Binder 架构情况，Java 层 Binder 架构的核心逻辑都是交由 Native 架构来完成，更多的是对 Binder 的一个封装过程，只有真正理解了 Native 层 Binder 架构，才能算掌握的 Binder。

前面的这些都是讲述 Binder 整个流程以及原理，再接下来你可能想要自己写一套 Binder 的 C/S 架构服务。如果你是**系统工程师**可能会比较关心 Native 层和 framework 层分别该如何实现自己的自定义的 Binder 通信服务，见 [Binder 系列 8—如何使用 Binder](http://gityuan.com/2015/11/22/binder-use/)；如果你是**应用开发工程师**则应该更关心 App 是如何使用 Binder 的，那么可以查看文章 [Binder 系列 9—如何使用 AIDL](http://gityuan.com/2015/11/23/binder-aidl/)。

最后是对 Binder 的一个简单总结 [Binder 系列 10—总结](http://gityuan.com/2015/11/28/binder-summary/)。

四. 源码目录
-------

从上之下, 整个 Binder 架构所涉及的总共有以下 5 个目录:

```
/framework/base/core/java/               (Java)
/framework/base/core/jni/                (JNI)
/framework/native/libs/binder            (Native)
/framework/native/cmds/servicemanager/   (Native)
/kernel/drivers/staging/android          (Driver)


```

#### 4.1 Java framework

```
/framework/base/core/java/android/os/  
    - IInterface.java
    - IBinder.java
    - Parcel.java
    - IServiceManager.java
    - ServiceManager.java
    - ServiceManagerNative.java
    - Binder.java  


/framework/base/core/jni/    
    - android_os_Parcel.cpp
    - AndroidRuntime.cpp
    - android_util_Binder.cpp (核心类)


```

#### 4.2 Native framework

```
/framework/native/libs/binder         
    - IServiceManager.cpp
    - BpBinder.cpp
    - Binder.cpp
    - IPCThreadState.cpp (核心类)
    - ProcessState.cpp  (核心类)

/framework/native/include/binder/
    - IServiceManager.h
    - IInterface.h

/framework/native/cmds/servicemanager/
    - service_manager.c
    - binder.c


```

#### 4.3 Kernel

```
/kernel/drivers/staging/android/
    - binder.c
    - uapi/binder.h


```

* * *

**微信公众号** [**Gityuan**](http://gityuan.com/images/about-me/gityuan_weixin_logo.png) **| 微博** [**weibo.com/gityuan**](http://weibo.com/gityuan) **| 博客** [**留言区交流**](http://gityuan.com/talk/)

* * *