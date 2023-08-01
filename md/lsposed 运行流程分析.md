> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/revercc/p/17034028.html)

xposed 适用的最高版本为 android 8.0，针对高版本的 ART HOOK 框架可以使用比较有名的 lsposed。它使用了 lsplant ART HOOK 框架（早期使用 YAHFA）并提供了和 xposed 一样的接口 API 与其进行了兼容，同时 lsposed 本身是一个基于 magisk 的 riru/zygisk 插件，所以在分析 lsposed 运行流程之前先分别分析一下 riru 和 zygisk 的运行流程。

riru 运行流程分析[#](#riru运行流程分析)
===========================

riru 的目的就是为了能够将插件 so 在 zygote 刚开始启动的时候注入到此进程中，首先 riru 自己需要先注入到 zygote 进程中。不同版本的 riru 使用了不同的方法将自己注入到 zygote 进程中。

*   早期：通过替换系统 so 库：libmemtrack.so 来实现劫持注入。
*   中期：使用 public.libraries.txt，在 zygote 启动时会加载此文件中的所有 so 库。
*   现在：修改系统属性`ro.dalvik.vm.native.bridge`进行注入，zygote 会加载此系统属性值对应的 so 库。

libriruloader.so 的. initarray[#](#libriruloaderso的initarray)
------------------------------------------------------------

目前最新的 riru 版本（V26）通过修改系统属性`ro.dalvik.vm.native.bridge`将 libriruloader.so 注入到 zygote 进程中，然后查看此 so 的. initarray 其会先调用 dlopen 将 libriru.so 加载，最后调用 libriru.so 的 init 函数。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230107235815178-1290659209.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230107235815178-1290659209.png)

libriru.so 的 init[#](#libriruso的init)
-------------------------------------

libriru 的 init 函数分别调用了 PrepareMapsHideLibrary，InstallHooks 和 Load。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108000908673-99127123.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108000908673-99127123.png)

### PrepareMapsHideLibrary[#](#preparemapshidelibrary)

PrepareMapsHideLibrary 加载 libriruhide.so 并获取其导出函数 riru_hide

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108001319388-1503685170.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108001319388-1503685170.png)

### InstallHooks[#](#installhooks)

XHOOK_REGISTER 是一个宏，其通过 GOT 表 hook libandroid_runtime.so 的 jniRegisterNativeMethods，因为 libandroid_runtime.so 的 jni 函数都是通过 jniRegisterNativeMethods 注册的，所以 hook 后可以主动调用原 jniRegisterNativeMethods 为 libandroid_runtime.so 注册回调函数并将`nativeForkAndSpecialize ,nativeSpecializeAppProcess, nativeForkSystemServer`这三个函数指针修改。这三个 jni 函数会在 Zygote 进程 java 层 fork 应用进程和系统进程时被调用，通过修改这三个函数的指针就可以在 zygote fork 新进程的时候得到执行时机。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108003705536-1613719515.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108003705536-1613719515.png)

### Load[#](#load)

*   调用 LoadModule 函数，通过 dlopen 加载所有的 riru 模块 so 并调用其 init 函数。
*   调用 HideFromMaps 函数通过之前获取的 libriruhide.so 的导出函数 riru_hide 隐藏所有加载的 riru 模块 so 和 libriru.so 本身。
*   调用所有加载的 riru 模块 so 的 onModuleLoaded 函数。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108003747523-420987685.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108003747523-420987685.png)

libriruhide.so 的导出函数 riru_hide，此函数会调用 do_hide 隐藏指定内存块，通过备份后再重新 map 会原地址的方法欺骗 map 表。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108005412486-1668355649.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108005412486-1668355649.png)

zygisk 运行流程分析[#](#zygisk运行流程分析)
===============================

zygisk 目的和 riru 一样都是为了在 zygote 进程中运行自己的模块，其通过修改 app_process 程序的入口，通过设置环境变量 LD_PRELOAD 后运行原来的 app_process 程序从而注入 zigisk 自己的 so。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108012209093-818312614.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108012209093-818312614.png)

zygisk 自己的 so 注入到 zygote 进程中后和 riru 一样也会加载所有的模块 so，同时也会利用相同的方式隐藏这些 so 模块。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108012423173-1762282441.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108012423173-1762282441.png)

整体流程大致和 riru 相似。

lsposed 运行流程分析[#](#lsposed运行流程分析)
=================================

由 riru 运行流程分析可知，其在加载模块 so 之后会先后进行如下几步操作（模块 so 就是 liblspd.so）：

1.  加载完模块 so 后调用 so 中的 init 函数
2.  隐藏模块 so
3.  调用模块 so 的 onModuleLoaded 函数
4.  当 zygote fork 生成新 apk 时会调用模块 so 中设置的回调函数`nativeForkAndSpecialize(pre/post) ,nativeSpecializeAppProcess(pre/post) , nativeForkSystemServer(pre/post)`  
    以 nativeForkAndSpecialize(pre/post) 为例，lsposed 设置这两个回调函数后，当 zygote fork 一个新的 app 进程时会分别调用这两个回调函数。lsposed 设置 nativeForkAndSpecializepost 回调函数会调用 MagiskLoader::OnNativeForkAndSpecializePost，此函数会调用一系列函数在 zygote fork 的 apk 运行前进行一些初始化。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108013835555-225303293.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108013835555-225303293.png)

LoadDex[#](#loaddex)
--------------------

先调用 PreloadedDex 将 lspd.dex 文件从磁盘 map 到内存中。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108015813797-1232544188.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108015813797-1232544188.png)

LoadDex 实例化一个 InMemoryClassLoader 并设置 parent 为系统类加载器 systemClassLoader，同时加载了 lspd.dex。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108015659791-869607590.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108015659791-869607590.png)

InitArtHooker and InitHooks[#](#initarthooker-and-inithooks)
------------------------------------------------------------

进行一些初始化，其中 InitHooks 内部会获取前面实例化的 InMemoryClassLoader 类加载器中加载的所有 dex 文件（实际就是 lspd.dex）的 DexFile 对象，然后调用 DexFile_setTrusted 使此 dex 文件中的类能够绕过 android 9.0 开始的对私有系统 frameword API 的限制访问，但是查看 DexFile_setTrusted 源码发现，此函数只有在 apk 处于调试状态下才能生效。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108130155060-988545754.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108130155060-988545754.png)

SetupEntryClass[#](#setupentryclass)
------------------------------------

相当于找到 lspd.dex 的入口类`org.lsposed.lspd.core.Main`

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108021102009-622937876.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108021102009-622937876.png)

forkCommon[#](#forkcommon)
--------------------------

forkCommon 会调用 initXposed 和 bootstrapXposed  
[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108022030325-1163216533.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108022030325-1163216533.png)

### initXposed[#](#initxposed)

initXposed 进行一些初始化，通过前面加载的 lspd.dex 中提供的 xposed API 进行一些 hook 操作。  
[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108022437104-2069340199.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108022437104-2069340199.png)

### bootstrapXposed[#](#bootstrapxposed)

bootstrapXposed 调用 lspd.dex 的 loadModules。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108022645847-165271131.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108022645847-165271131.png)

loadModules 调用 getModulesList 加载并所有的 xposed 模块，调用重载的 loadModule（内部调用 InitModule）初始化 xposed 模块中需要 hook 的函数。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108023149873-361591506.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108023149873-361591506.png)

loadModules 调用 getModulesList，getModulesList 内部经过层层调用最后会调用 LoadModule，此函数会读取 xposed 模块 apk 文件中的 assets/xposed_init 中注册的类名称。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108030427293-1987398922.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108030427293-1987398922.png)

loadModules 调用重载的 loadModule，此函数通过一个自定义的类 LspModuleClassLoader 继承于 ByteBufferDexClassLoader ，ByteBufferDexClassLoader 继承于 BaseDexClassLoader，BaseDexClassLoader 继承于 ClassLoader，定义一个 classloader 设置 parent 为之前创建的 InMemoryClassLoader 并加载对应的 xposed 模块 apk。最后还会调用 InitModule 对 xposed 模块中需要 hook 的函数进行初始化。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108025138079-163462339.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108025138079-163462339.png)

检测 lsposed 指纹[#](#检测lsposed指纹)
==============================

检查栈回溯[#](#检查栈回溯)
----------------

可以通过主动抛出一个异常并检查栈回溯信息看是否存在一些特殊方法调用，例如：`de.robv.android.xposed`

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108131426782-1084356518.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108131426782-1084356518.png)

内存漫游获取 classloader[#](#内存漫游获取classloader)
-----------------------------------------

getInstancesOfClasses 可以获取某个类的所有实例，可以通过调用此函数获取所有的 ClassLoader 实例并进一步查看此 ClassLoader 加载的所有类信息，看是否存在特殊的类名称。但是此函数是 hide api，调用的话需要先绕过 hide api 限制。

检查 / proc/pid/maps[#](#检查procpidmaps)
-------------------------------------

无论 lsposed 是基于 riru 还是 zigisk，其 so 模块都会被隐藏。虽然隐藏后虽然内存块是匿名的，但是内存块还是包含可执行属性，正常情况下是很少出现匿名的可执行内存的。可以通过检测 map 表是否存在匿名的并且具有可执行属性的内存判断是否存在 lsposed 模块。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108132437312-1092601981.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108132437312-1092601981.png)

检查 riru[#](#检查riru)
-------------------

riru 会将 libriruloader.so 放在`/system/lib64`目录下并注入到 zygote 进程中，通过`fopen("/system/lib64/libriruloader.so", "r");`判断是否存在此文件。

[![](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108134122471-519922001.png)](https://img2023.cnblogs.com/blog/2052882/202301/2052882-20230108134122471-519922001.png)

检查 zygisk[#](#检查zygisk)
-----------------------

zygisk 通过修改 zygote 进程的环境变量 LD_PRELOAD 注入 zygisk 的 so 文件，在应用程序中可以检测环境变量。不过 shamiko 项目可以对这些变化进行抹去`https://github.com/LSPosed/LSPosed.github.io/releases/tag/shamiko-126`

参考：  
[https://bbs.kanxue.com/thread-269094.htm](https://bbs.kanxue.com/thread-269094.htm)  
[https://bbs.kanxue.com/thread-263018.htm](https://bbs.kanxue.com/thread-263018.htm)  
[https://liwugang.github.io/2022/01/01/android_env_detection.html](https://liwugang.github.io/2022/01/01/android_env_detection.html)  
以上均属个人观点，仅供参考。