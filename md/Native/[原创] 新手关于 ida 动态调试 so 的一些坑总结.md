> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-259633.htm)

> [原创] 新手关于 ida 动态调试 so 的一些坑总结

目录

*   [动态调试步骤](#动态调试步骤)
*            [复制 `android_server` 到设备中，并执行。](#复制`android_server`到设备中，并执行。)
*            [用 `pm` 确定要调试 apk 的包名](#用`pm`确定要调试apk的包名)
*            [用 `am` 启动被调试应用](#用`am`启动被调试应用)
*            [设置 IDA 调试器](#设置ida调试器)
*            [开始附加程序](#开始附加程序)
*            [jdb 连接](#jdb连接)
*            [触发断点](#触发断点)
*            [调试快捷键](#调试快捷键)
*   [可能的错误](#可能的错误)

 

虽然 Java 层代码包含了许多有用的信息，但是一般稍微注重安全的应用都会把核心代码放到 Native 层，所以对 Native 层的调试就显得尤为重要了。

动态调试步骤
======

> 使用工具：
> 
> am + pm +IDA, 其中 am 和 pm 为安卓系统自带

复制`android_server`到设备中，并执行。
---------------------------

`android_server`的目录为：`IDA目录`>`dbgsrv`>`android_server`

> 注意：
> 
> `android_server`分版本的，使用对应的版本。

```
//复制到设备上
adb push android_server /data/local/tmp
 
//修改权限，使之能执行
chmod 777 /data/local/tmp/android_server
 
//执行
cd /data/local/tmp
./android_server
 
adb forward tcp:23946 tcp:23946
```

等待附加。

 

![](https://bbs.pediy.com/upload/attach/202005/869791_TQJCV6H8VC5FK5U.jpg)

用`pm`确定要调试 apk 的包名
------------------

pm(package manager) 包管理工具.

 

列出所有的包信息:`pm list packages [filter]`

*   pm 过滤器
    *   -d: 只显示禁用的应用的包名
    *   -e: 只显示可用的应用的包名
    *   -s: 只显示系统应用的包名
    *   -3: 只显示第三方应用的包名

![](https://bbs.pediy.com/upload/attach/202005/869791_EZJK655Y3G5ZFVM.jpg)

用`am`启动被调试应用
------------

> am 是 activity manager 的缩写

 

`am`启动程序命令：`am start -D -n com.example.testarm/.MainActivity`

> `am start -D -n`调试模式打开应用
> 
> `com.example.testarm`要调试启动的包名
> 
> `.MainActivity`Lunch Activity

 

启动后等待调试器的链接。  
![](https://bbs.pediy.com/upload/attach/202005/869791_NVBFYY56EP86ZF6.jpg)  
<img src="upload/attach/202005/869791_NVBFYY56EP86ZF6.jpg" />

设置 IDA 调试器
----------

*   用 IDA 打开想要调试的 so 库。
    
*   选择`Remote ARM Linux/Android debugger`。
    
    ![](https://bbs.pediy.com/upload/attach/202005/869791_F33S5P9P34T995K.jpg)
    
*   设置调试选项
    
    ![](https://bbs.pediy.com/upload/attach/202005/869791_HPYY355GC63BGWT.jpg)
    
    ![](https://bbs.pediy.com/upload/attach/202005/869791_ZH35GV76ZXTJFZV.jpg)
    

开始附加程序
------

**设置主机和端口**

 

![](https://bbs.pediy.com/upload/attach/202005/869791_XS5ZBFEYMSWQEW8.jpg)

 

![](https://bbs.pediy.com/upload/attach/202005/869791_9HC2KHBJS29R8P8.jpg)

 

**选择要调试的程序进行附加**

 

弹出对话框表示全部加载完成了.

 

![](https://bbs.pediy.com/upload/attach/202005/869791_UBG8M4KQ8G2RAUH.jpg)

 

此时会显示出`PC`的位置

 

![](https://bbs.pediy.com/upload/attach/202005/869791_7DDDAX7S64FDVFR.jpg)

 

**IDA 按`F9`, 继续执行.**

jdb 连接
------

```
jdb -connect com.sun.jdi.SocketAttach:hostname=localhost,port=8700
```

> 8700 为 apk 运行的端口, 根据实际情况更改.

 

**确定 port 的方法**

*   使用 ddms(monitor)
    
    ![](https://bbs.pediy.com/upload/attach/202005/869791_3WNMK3J9GTE4V4E.jpg)
    
    **注意: 以前 monitor 为 Android Studio 自带, 从 2019 年下半年开始的 Android Studio 删除了这些工具.**
    
    提取的 ddms:https://www.jianguoyun.com/p/DWps1OsQ9oe6CBjP15oD (访问密码：HrhFnH)
    

触发断点
----

![](https://bbs.pediy.com/upload/attach/202005/869791_Z2DMR3K998BAJXV.jpg)

 

**same**

 

![](https://bbs.pediy.com/upload/attach/202005/869791_2P6XU5Q9C6E9GW3.jpg)

 

**Yes**

调试快捷键
-----

`F2`下断点

 

`F7`单步步入

 

`F8`单步步过

 

`F9`执行到下个断点

可能的错误
=====

*   由于没有设置_参数_, 所以经常有下面的错误提示, 忽略或者随便给个参数
    
    ![](https://bbs.pediy.com/upload/attach/202005/869791_N9A2TUKR38KV2G4.jpg)
    
*   没有进行端口映射
    
    ![](https://bbs.pediy.com/upload/attach/202005/869791_FA53HBBAP3WZJ2S.jpg)
    
    > `adb forward tcp:23946 tcp:23946`
    
*   android_server 未开启
    
    ![](https://bbs.pediy.com/upload/attach/202005/869791_WBR26B8YG6RXKAG.jpg)
    
*   可附加的程序过少
    
    ![](https://bbs.pediy.com/upload/attach/202005/869791_DP56MT5ZZC32F38.jpg)
    
    启动`android_server`的用户权限低. 用 root 用户运行`android_server`来监听.
    
*   ida 调试版本的 so 和正在运行的 so 不一致
    
    ![](https://bbs.pediy.com/upload/attach/202005/869791_62KN8D78U3D3QKT.jpg)
    
*   jdb 连接失败
    
    ![](https://bbs.pediy.com/upload/attach/202005/869791_T9GX6WKM37SCTC8.jpg)
    

*   **ida 打开的 so 文件名要和运行 apk 中的 so 名一致. 如果不一致会导致断点无效.**

[第五届安全开发者峰会（SDC 2021）10 月 23 日上海召开！限时 2.5 折门票 (含自助午餐 1 份）](https://www.bagevent.com/event/6334937)

最后于 2020-5-20 17:44 被 nisodaisuki 编辑 ，原因： 增加和修改错误