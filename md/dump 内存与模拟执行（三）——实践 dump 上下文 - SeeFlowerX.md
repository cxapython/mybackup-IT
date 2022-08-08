> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.seeflower.dev](https://blog.seeflower.dev/archives/169/)

> 前言假定你已经完成下面的操作，那么现在要实际开始操作了 termux + lldb 的环境准备好了写好了 dump 上下文的 lldb 脚本步骤首先在 PC 上进入 termux 的环境中（root 用户）以绿洲 v...

2022-08-06 [默认分类](https://blog.seeflower.dev/category/default/)

假定你已经完成下面的操作，那么现在要实际开始操作了

*   `termux + lldb`的环境准备好了
*   写好了 dump 上下文的 lldb 脚本

首先在 PC 上进入 termux 的环境中（root 用户）

以`绿洲 v4.5.6`为例（64 位），目标是`com.weibo.xvideo.NativeApi`的`s`函数

通过 frida hook RegisterNative，可以得到其 native 函数位于`liboasiscore.so + 0x116CC`

先查看其主进程`pid`

```
ps -ef | grep oasis
```

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_22-39-09.png)

然后执行`lldb`进入 lldb 交互界面

再执行`attach {pid}`附加到主进程

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_22-41-08.png)

这个时候 APP 会被暂停，此时再按`c`即可恢复运行

恢复运行后我们依然可以执行命令，这点相比 gdb 要好很多，gdb 附加真的很慢而且只能在暂停的时候执行命令（？）

有时候会遇到这样的信号，这个时候 APP 就突然暂停了

![](https://blog.seeflower.dev/images/Snipaste_2022-07-31_10-32-24.png)

通过下面的命令查看信号处理设置

```
process handle SIGSEGV
```

输出如下

```
NAME         PASS   STOP   NOTIFY
===========  =====  =====  ======
SIGSEGV      true  true   true
```

那么可以在刚附加的时候就把这个信号 PASS 动作设置为 true，STOP 动作设置为 false，这样可以避免 APP 突然暂停

```
process handle SIGSEGV -p true -s false
```

如果其他 APP 存在反调试，可能会发`SIGQUIT SIGSTOP SIGKILL`之类的信号

暂停的时候可以打调用栈看看，说不定能找到反调试信息

可以用下面的命令查看目标 so 的基址信息

```
image list -o -f liboasiscore.so
```

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_22-48-56.png)

这里只是演示下，lldb 可以直接通过下面的命令实现对`模块名+偏移`的断点设置，相比 gdb 的操作方便得多

```
breakpoint set -s liboasiscore.so -a 0x116CC
```

现在断点下好了，将 APP 切换到后台，再进入 APP 就能触发`s`函数的调用

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_22-51-42.png)

如果前面没有载入脚本，那么先通过下面的命令载入

```
command script import lldb_dumper.py
```

然后执行`dumpctx`命令即可开始 dump 上下文

提前做了过滤，所以不会太花时间

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_22-53-12.png)

最后脚本会将 dump 好的内容打包

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_22-54-15.png)

如果是用的`ttyd+浏览器远程`，那现在可以使用`lsz`命令直接下载文件，如果是 adb 方式，记得手动 pull 出来

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_22-55-43.png)

现在就完成了 dump 上下文了

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_22-57-22.png)

**这里还没有结束**

查看`liboasiscore.so + 0x116CC`的反汇编代码，可以看到这样的代码

```
v35 = *(_QWORD *)(_ReadStatusReg(ARM64_SYSREG(3, 3, 13, 0, 2)) + 40);
```

对应的汇编是`MRS X8, #3, c13, c0, #2`

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_23-05-56.png)

也就是在获取`TPIDR_EL0`这个系统寄存器，有关说明可以参考下面的文章

*   [Android 逆向中的 Canary 机制](https://cataloc.gitee.io/blog/2021/04/24/Android%E9%80%86%E5%90%91%E4%B8%AD%E7%9A%84Canary%E6%9C%BA%E5%88%B6/#%E5%89%8D%E8%A8%80)

还有哪些系统寄存器？请参考

*   [https://github.com/TrungNguyen1909/aarch64-sysreg-ida/blob/main/aarch64_sysreg.py](https://github.com/TrungNguyen1909/aarch64-sysreg-ida/blob/main/aarch64_sysreg.py)

不过我们只需要获取到`TPIDR_EL0`就足以应付后续的模拟执行了

为了能稳妥的模拟执行，必须拿到这个寄存器的值，经过实践，发现无法直接在最开始断点的时候就拿到

只能在执行完`MRS`这条指令之后去获取 x8 的值

所以还需要在`MRS`的下一行断点，也就是`liboasiscore.so + 0x116F4`

```
breakpoint set -s liboasiscore.so -a 0x116F4
```

下完断点后按 c 执行，等断在`0x116F4`，然后通过`register read x8`读取`TPIDR_EL0`的值，记录下来

注意，不一定都是把值赋给 x8，请结合实际情况确定是哪个寄存器

**现在才算是完成上下文 dump 了**

如果要结束附加进程，建议使用下面的命令（最好取消断点，再按`c`让 APP 恢复运行）

```
process detach
```

直接 quit 也能退，但是会导致 APP 崩溃或者无响应，不建议

* * *

  
星球广告 & 交流群  
![](https://blog.seeflower.dev/images/xqyh.png)