> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-272295.htm)

> [原创]zygisk 原理简单分析和以及 zygisk 插件开发 (1)

### zygisk 是什么

来自面具作者的文章对于 zygisk 的介绍

> Zygisk 是 Zygote 的 Magisk。这将在 Zygote 进程中运行 Magisk 的一部分，使 Magisk 模块更强大。这也是 Magisk"退出舞台" 理念中非常重要的一部分。当某一个进程位于上述拒绝列表中时，Magisk 将清理该过程的内存空间，以确保不会应用任何修改（P.S.1）。当然，Zygisk 仍然是 WIP（开发中）的，一旦实施准备好测试，会公布更多细节。

#### 对于 zygisk 的理解

*   zygisk 依然是 magisk，但是是更高级的版本，带了注入 Zygote 功能。
*   zygisk 的开启是可选的，在 magisk 设置。如不开启将和原来差不多。
*   zygisk 将提供超过之前 magisk 的功能，尤其是注入 hook。所以可以基于 zygisk 开发出 lsp(新的 xp 框架) 而且不基于 riru。
*   riru 和 zygisk 对 Zygote 的修改部分重复了，所以二者不能很好共存。
*   zygisk 没有 hide，因为作者加入了谷歌之后不再做检测对抗了。

### magisk 实现原理

在 magisk 上有一行说明: systemless root

 

那么这个 systemless 是什么东西

 

在了解 systemless 之前我们还要了解什么是 root 以及如何获得 root

#### root 原理 (来自网络)

**下面是来自网络的 root 解释，年代已久不过我感觉还是有用的。**

 

root 权限就是 root(管理员权限) 用户的权限，该权限可以修改根目录下的文件

 

为了让我们的应用使用 root 权限

 

首先我们需要一个处理 root 请求的 su 进程

 

我们链接手机到电脑 输入 adb shell

 

再输入 su 如果没 root 就会提示 su 文件找不到

 

ok 既然没有这个文件那我们就 push 这个 su 执行文件到  
/system/bin 下面吧

 

不行，因为你没有 root 权限所以不能向 system 传输文件

 

同样的因为你不能传输文件也就不能植入 su 应用程序

 

也就是没法让你获得 root 权限 这就是死循环

###### 之前的 root 方式

 

这个不细说了

*   利用系统漏洞提权注入。这个方式很多，早期版本的系统用的多。现在系统越来越安全基本难以实现。
    
*   定制内核修改 system 分区的方式。
    
*   厂商会定制开发版本自带 root 权限。在于本身 ASOP 就支持 userdebug 版本模式去编译的。
    

##### systemless root

之前的 root 会对 system 目录做文件修改

 

而 systemless root 则不会对 system 目录做任何修改

 

这是为了应对高版本的系统的检测

 

基于 systemless 的 root 就是 systemless root

 

magisk 是实现 systemless root 的一个方式，实际上在 magisk 之前就有 systemless root。只不过 magisk 把这个发扬光大了。

#### magisk 实现方式

在之前做过 magisk 的分析，我把答案抄过来。

##### magisk 刷入过程

*   安装 magisk apk
*   提取 boot.img
*   制作修改的 patch_boot.img
*   刷入 patch_boot.img 开机即可

不难得出核心在于针对 boot.img 文件的修改

 

那么修改的 patch_boot.img 具体做了什么是我们最关心的

##### 启动日志分析

开机 打开面具的日志这里

 

![](https://bbs.pediy.com/upload/attach/202204/832784_5BTUUB5KNMVV34Z.png)

 

可以看到这里在

 

`/sbin/.magisk/mirror` 下面建立了一个系统根目录的镜像目录

 

我们打开文件的时候就会发现 sbin 里面还拥有一个 su 执行文件和模块插件等其他文件

 

然后进行 bind mounted 操作，也就是让这个文件系统生效。

 

这时候 ps -A 其实就可以看到 su 进程，也就有了授权 root 的能力。

 

接下来就可以用 magisk 来管理 root 权限，请求 root 的都会出现在`超级用户`这个列表里。

 

同时基于 bind mount 可以针对系统文件进行替换以及在这里执行 sh 脚本，这就是我们写的 magisk 模块的实现了。

> 非常多的细节可以在 [https://topjohnwu.github.io/Magisk/details.html](https://topjohnwu.github.io/Magisk/details.html) 里面得到，或许我的理解会有误差。

### zygisk 实现原理

在了解 zygisk 之前，先了解 magisk 的功能实现原理。我这里分析 zygisk 关于注入 Zygote 的逻辑。

#### 实现原理猜想

在分析之前我就在猜想他的实现原理了。

> xposed 注入 Zygote 原理是替换 app_process，而 magisk 可以做到替换系统文件。那么 magisk 是不是可以替换 app_process 来注入 Zygote。

 

虽然 zygisk 是新一点东西 好在已经有人分析过了。可以踩在巨人的肩上了。

#### 验证猜想

在 magisk 仓库里面搜索 `app_process` 果然找到了相关代码

 

打开 [main.cpp](https://github.com/topjohnwu/Magisk/blob/9e8218089bdca649aa27a76961fbfba8a24ef3f5/native/jni/zygisk/main.cpp) 看到一行注释

 

`// Entrypoint for app_process overlay`

 

确认无疑了, 对 app_process 进行了替换

#### 源码分析

[  
Zygisk 源码分析](https://gist.github.com/5ec1cff/bfe06429f5bf1da262c40d0145e9f190#file-zygisk-md) 实际上这篇文章已经写的非常详细了，我加上了一点自己的理解。算是拾人牙慧了。

##### 太长不看版本

先总结一下整个流程

*   `module.cpp`中判断是否开启`zygisk_enabled`, 是就进入  
    mount_zygisk 进行 `app_process`的 mount，将  
    `app_process`替换成`magisk`
*   `main.cpp` 是`magisk`执行文件，作用是替换`app_process`工作 (原理是用 LD_PRELOAD 注入)
*   `entry.cpp`加载 Zygisk，分为两个阶段加载。一阶段 dlopen 载入二阶段，二阶段进行 hook。 其他的细节包括 hide hook 处理看源码分析。

##### 注入阶段

###### native/jni/core/module.cpp

 

搜索 `app_process` 出现最多的就是这个 module.cpp 文件

 

打开文件看到`bind_mount` 就知道这个类对文件进行了挂载。

 

对`app_process`进行处理的函数在`magic_mount`中

> mount 是 Linux 下的一个命令，它可以将分区挂接到 Linux 的一个文件夹下，从而将分区和该目录联系起来，因此我们只要访问这个文件夹，就相当于访问该分区了。

 

![](https://bbs.pediy.com/upload/attach/202204/832784_UP2TFAZ8N7TSMS7.png)

 

新建文件夹后`mount_zygisk`处理挂载文件夹

###### mount_zygisk 函数

 

![](https://bbs.pediy.com/upload/attach/202204/832784_HEHQW7HWX9DREUD.png)

 

两件事: 文件移动和文件夹挂载

 

app_process 32/64 放到了 zygisk 下面  
magisk 32/64 放到了 `MAGISKTMP` 下面

 

完成文件的`app_process`到`magisk`的替换

 

做完这些之后，我们重启手机启动运行的`app_process`其实就是运行`magisk`

 

我们其实可以看到这些文件的

 

![](https://bbs.pediy.com/upload/attach/202204/832784_2KUQFDJ7T7SWMSA.png)

##### 加载阶段

这部分核心研究跑起来的 magisk 跑起来之后具体做了什么

 

包括 hook 的实现逻辑

###### native/jni/zygisk/main.cpp

 

运行了 magisk 文件，现在进入他的函数分析他的逻辑。

 

`Entrypoint for app_process overlay`

 

对着这个注释往下看

###### app_process_main 函数

 

通过 socker 通信设置几个变量的值 LD_PRELOAD INJECT_ENV_1  
MAGISKTMP_ENV

 

`ZygiskRequest::SETUP`是通信的标志

 

![](https://bbs.pediy.com/upload/attach/202204/832784_SPK8FMMQYKYUSUX.png)

 

这里非常核心的一个就是 LD_PRELOAD 变量的值设置，这将作用到后面的注入

> LD_PRELOAD 是 Linux 系统的一个环境变量，它可以影响程序的运行时的链接（Runtime linker），它允许你定义在程序运行前优先加载的动态链接库。

##### native/jni/zygisk/entry.cpp

找到消息接收的地方

 

![](https://bbs.pediy.com/upload/attach/202204/832784_6NUS7AJW9YSMEU8.png)

###### 进入 setup_files 函数

 

![](https://bbs.pediy.com/upload/attach/202204/832784_Y43ZSM4SQ4NV4G5.png)

 

buf 获取了可执行路径  
sendfd 发送持有的真正的 app_process 文件 fd  
write_string 发送 MAGISKTMP 路径

 

到这里我开始发现与参考文件不同的地方了

 

尤其是

 

![](https://bbs.pediy.com/upload/attach/202204/832784_4EBXM3RNV62GGH5.png)

 

根本对不上，zygisk 下面也没有对应的 so 文件

 

那就是改版了，下面就自己分析新版 zygisk 的原理吧

##### native/jni/zygisk/entry.cpp

![](https://bbs.pediy.com/upload/attach/202204/832784_WUGERUUC2XQ2SJA.png)

 

zyg_init 函数

 

这里进行两阶段的加载操作

*   判断`INJECT_ENV_1` 进行一阶段加载 这个在 `app_process_main`进行设置的
    
*   在一阶段加载加载完成阶段设置二阶段标志 `INJECT_ENV_2`
    

下面分析两个阶段的事情

###### first_stage_entry

 

一阶段: 好在作者加了大段注释 ，我谷歌翻译一遍

 

![](https://bbs.pediy.com/upload/attach/202204/832784_5ENZ5BJF25SY8NM.png)

 

在这里我发现 LD_PRELOAD 是个非常重要的变量

 

修改这个值就可以加载我们的注入的程序

 

这里一阶段就是来加载 `LD_PRELOAD` 的文件

 

这个文件就是之前 `app_process_main`设置的

 

获取 fd 的值这里最终将这个值写入到 `MAGISKFD_ENV` 中

 

同时进行第一次的 `android_dlopen_ext`

###### second_stage_entry

 

获取`MAGISKFD_ENV`的值进行 path

 

进入 `android_dlopen_ext` 再一次 dlopen 并立马关闭

 

恢复 变量 `MAGISKTMP_ENV` `MAGISKFD_ENV`

 

![](https://bbs.pediy.com/upload/attach/202204/832784_3DP44KQMFWSZGAE.png)

 

在结束的时候, 调用了

*   sanitize_environ();
*   hook_functions();

`sanitize_environ` 字面意思就是 消毒环境。就是用于对抗检测的，消灭证据。具体实现这里不是重点。

 

`hook_functions` 具体的 hook 实现部分

#### hook 实现

我们已经将自己的文件注入进去，现在要实现 hook

###### native/jni/zygisk/hook.cpp

 

核心的 hook_functions 处于这个类

 

![](https://bbs.pediy.com/upload/attach/202204/832784_D5YKSSDURV76D89.png)

 

我们核心关注`XHOOK_REGISTER(ANDROID_RUNTIME, jniRegisterNativeMethods);` 这部分的逻辑

 

在之前的 riru 分析中，我们已经了解过这种 hook 的实现方法。基于 xhook，对 register 方法进行 hook 后做个指针的切换。

 

几个重要的宏定义

 

`XHOOK_REGISTER` 宏  
使用 xhook register

 

![](https://bbs.pediy.com/upload/attach/202204/832784_U73PGAUGJF392U7.png)

 

`DCL_HOOK_FUNC` 宏  
对函数进行 hook  
![](https://bbs.pediy.com/upload/attach/202204/832784_6X3VMCR5RAC6A9E.png)

 

`DCL_PRE_POST` 宏  
定义函数的 per 和 post 两种

 

![](https://bbs.pediy.com/upload/attach/202204/832784_GGTE47BUQBAGSW4.png)

 

![](https://bbs.pediy.com/upload/attach/202204/832784_5BGFHT8KZ6DKWZ4.png)

 

比如 fork 定义了 fork_pre 和 fork_post

 

挑一对 pre 和 post 来分析

##### nativeSpecializeAppProcess_pre/nativeSpecializeAppProcess_post

![](https://bbs.pediy.com/upload/attach/202204/832784_N22CM37FZNS2JH9.png)

 

可以发现核心就是两个

*   run_modules_pre(module_fds);
    
*   run_modules_post();
    

也就是运行 modules 插入的 hook 代码

 

这个两个方法需要记得后面继续分析。

###### hookAndSaveJNIMethods

 

执行 hook 操作的方法 核心的 hook 逻辑

 

![](https://bbs.pediy.com/upload/attach/202204/832784_TURK35ATT92QWFD.png)

###### HOOK_JNI

 

三个函数的 hook

*   HOOK_JNI(nativeForkAndSpecialize)
*   HOOK_JNI(nativeSpecializeAppProcess)
*   HOOK_JNI(nativeForkSystemServer)

这三个函数是 app 进程启动的 Zygote 中三个相关函数，是应用进程或者系统服务进程被 fork 出来的时候会调用的方法。

 

![](https://bbs.pediy.com/upload/attach/202204/832784_QZ2NF7P2BJ3HZGW.png)

 

实现 hook 后引入

##### native/jni/zygisk/jni_hooks.hpp

实现具体 hook 的地方，在这里区分了版本和手机类型。将新旧方法做了保存和指针更换处理。

 

![](https://bbs.pediy.com/upload/attach/202204/832784_Y7AXMYBR2X3TMSN.png)

 

比如这个就是安卓 Q 版本的 hook 处理逻辑，这里有个重要的`HookContext`结构体。

 

核心的三句

*   ctx.nativeSpecializeAppProcess_pre();
    
*   reinterpret_cast<decltype(&nativeSpecializeAppProcess_q)>(nativeSpecializeAppProcess_orig)( env, clazz, uid, gid, gids, runtime_flags, rlimits, mount_external, se_info, nice_name,  
    is_child_zygote, instruction_set, app_data_dir);
    
*   ctx.nativeSpecializeAppProcess_post();
    

`pre`和`post`调用用于执行插入的 hook code。

 

而中间那句就是执行原方法`orgin`

### 最后

分析到这里差不多，未完待续。后面还有关于如何 zygisk 屏蔽应用以及如何 load moudules 代码的有时间再说。

 

参考

 

[  
Zygisk 源码分析](https://gist.github.com/5ec1cff/bfe06429f5bf1da262c40d0145e9f190#file-zygisk-md)

[【公告】 [2022 大礼包]《看雪论坛精华 22 期》发布！收录近 1000 余篇精华优秀文章!](https://bbs.pediy.com/thread-271749.htm)

最后于 1 小时前 被小黄鸭爱学习编辑 ，原因： 勘误

[#基础理论](forum-161-1-117.htm) [#NDK 分析](forum-161-1-119.htm) [#HOOK 注入](forum-161-1-125.htm) [#源码分析](forum-161-1-127.htm)