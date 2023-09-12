> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [crifan.github.io](https://crifan.github.io/reverse_debug_frida/website/use_frida/frida_cli/frida_stalker.html)

> 如下场景：

*   `Frida`的`Stalker`
    *   别称：`frida-stalker`
        *   有人翻译成：`潜行者`
    *   作用：汇编指令级别的 hook
    *   用途：追踪真实代码指令的执行过程
    *   官网文档
        *   [Stalker 介绍](https://frida.re/docs/stalker/)
    *   API 接口
        *   接口形式
            *   普通用户常直接调用：`JS的API`
                *   [Stalker - JavaScript API | Frida](https://frida.re/docs/javascript-api/#stalker)
            *   内部接口：Gum 接口
                *   TypeScript type definitions
                    *   [DefinitelyTyped/index.d.ts](https://github.com/DefinitelyTyped/DefinitelyTyped/blob/master/types/frida-gum/index.d.ts)
        *   接口内容
            *   最核心接口
                *   `Stalker.follow([threadId, options])`
            *   其他
                *   `Stalker.exclude(range)`
                *   `Stalker.unfollow([threadId])`
                *   `Stalker.parse(events[, options])`
                *   `Stalker.flush()`
                *   `Stalker.garbageCollect()`
                *   `Stalker.invalidate(address)`
                *   `Stalker.invalidate(threadId, address)`
                *   `Stalker.addCallProbe(address, callback[, data])`
                *   `Stalker.removeCallProbe`
                *   `Stalker.trustThreshold`
                *   `Stalker.queueCapacity`
                *   `Stalker.queueDrainInterval`
    *   用法
        *   概述
            *   核心逻辑
                *   在 frida 命令上，和普通 frida 一样，都是调用 js
                    
                    ```
                    frida -U -n akd -l ./fridaStalker_akdSymbol2575.js
                    
                    
                    ```
                    
                *   js 内部逻辑
                    *   最初要初始化：计算出当前要 hook 的函数所属的模块和地址
                    *   在普通的`Interceptor.attach`的`onEnter`中，加上`Stalker.follow`
                        *   在`transform`中，计算是否是原始函数的代码
                            *   如果是，再去：实现特定的调试的目的
                                *   打印真实执行的指令的信息`instruction.toString()`
                                *   打印当时的变量的值`context`
                                *   等等
        *   详解
            *   举例
                *   [Stalker 示例代码](https://crifan.github.io/reverse_debug_frida/website/frida_example/frida/ios_objc/stalker/)

何时需要用到 Frida 的 Stalker
----------------------

如下场景：

*   你想要搞懂该函数内部的执行的逻辑，想要查看哪个函数，甚至是哪个代码块 codeblock，被执行了
*   还比如你想要搞懂，当传入不同参数时，函数内部代码执行的流程路径，是否有何不同。

就可以去用：

*   `Stalker.follow()`

Frida 的 Stalker 中 transform 的逻辑
-------------------------------

Frida 的 Stalker 中 transform 的逻辑：

除了官网文档：

[https://frida.re/docs/stalker/](https://frida.re/docs/stalker/)

介绍了内部具体实现机制和过程之外：

对于，想要搞懂如何利用 transform 去调试代码来说：

需要明白的逻辑是：

此处代码：

```
Interceptor.attach(funcRealStartAddr, {
    onEnter: function(args) {
...
        var curTid = Process.getCurrentThreadId();
        console.log("curTid=", curTid);
        Stalker.follow(curTid, {
            events: {
                call: true, 
                ret: false, 
                exec: false, 
                block: false, 
                compile: false 
            },
            
            transform: function (iterator) {
                ...


```

触发到的`transform的iterator`来说：

*   每次触发 = 每个 iterator：都（对应着）单个 block=（basic）代码块
*   此处代码块有 2 种
    *   非原始函数代码 == isAppCode=false
        *   对应着应该是 Stalker 内部实现原理说的，copy 拷贝出的代码
            *   其中会额外加上很多逻辑，用于实现 Stalker 的功能和逻辑
            *   估计就是这里说的这些内容
                *   [[原创] sktrace：基于 Frida Stalker 的 trace 工具 - Android 安全 - 看雪 - 安全社区 | 安全招聘 | kanxue.com](https://bbs.kanxue.com/thread-264680.htm)
                    *   每当执行到一个基本块，Stalker 都会做以下几件事：
                        1.  对于方法调用，保存 lr 等必要信息
                        2.  重定位位置相关指令，例如：ADR Xd, label
                        3.  建立此块的索引，如果此块在达到可信阈值后，内容未曾变化，下次将不再重新编译（为了加快速度）
                        4.  根据 transform 函数，编译生成一个新的基本块 GumExecBlock ，保存到 GumSlab 。void transform(GumStalkerIterator iterator, GumStalkerOutput output, gpointer user_data) 可以控制读取，改动，写入指令。
                        5.  transform 过程中还可通过 void gum_stalker_iterator_put_callout (GumStalkerIterator self,GumStalkerCallout callout, gpointer data, GDestroyNotify data_destroy) 来设置一个当此位置被执行到时的 callout。通过此 void callout(GumCpuContext cpu_context, gpointer user_data) 获取 cpu 信息。
                        6.  执行一个基本快 GumExecBlock，开始下一个基本快
        *   所以无需操作具体内部过程，直接忽略即可
    *   原始函数代码 == isAppCode=true
        *   是实际运行的代码，才是我们所要关注的代码
            *   才会真正的去处理：比如判断是否是对应的（某个偏移量的）代码，然后去打印查看调试寄存器的值等等

此处去通过计算是否是 原始函数代码 isAppCode 决定是否处理

具体详见示例代码：[___lldb_unnamed_symbol2575$$akd](https://crifan.github.io/reverse_debug_frida/website/frida_example/frida/ios_objc/stalker/akd_lldb_unnamed_symbol2575.html)

`Stalker.follow`中的`events`的属性含义
-------------------------------

*   `Stalker.follow`中的`events`的属性含义
    *   概述
        *   Stalker.follow 中的 events 中某个属性是 true，含义是：当出现对应指令，则触发对应 event 事件
    *   详解
        *   属性
            *   对应的属性的含义是
                *   call：call 指令
                    *   Intel 的：call 指令
                    *   ARM 的：BL 类的指令
                        *   普通的 = arm64 的：BL、BLR 等
                        *   arm64e 的，带 PAC 的：BLRAA、BLRAAZ、BLRAB、BLRABZ 等
                *   ret：ret 指令
                *   exec：所有指令
                *   block：（单个）block 的（所有）指令
                *   compile：特殊，（单个）block 被编译时，仅用于测试代码覆盖率？
            *   除去特殊的 compile 参数，其他几个参数，按照范围大小去划分，更容易理解：
                *   exec：所有代码的级别
                    *   block：单个代码块的级别
                        *   某些特殊指令的级别
                            *   call：单独的 call 指令
                            *   ret：单独的 ret 指令
        *   event 事件
            *   会触发 onReceive(events) 函数
                *   其中可以 events 是二进制（的 blob），需要去用 Stalker.parse() 解析后才能看懂
    *   -》events 和 onReceive 的作用
        *   暂时不完全懂，只是知道，可以设置参数，决定 call、ret 等指令的触发时去打印，其他用途暂时不清楚

`Stalker.follow()`内部实现原理
------------------------

当用户调用`Stalker.follow()`时，内部调用：

*   要么是：`gum_stalker_follow_me()` ：去跟踪当前的线程 thread
    *   函数原型
        
        ```
        GUM_API void gum_stalker_follow_me (GumStalker * self, GumStalkerTransformer * transformer, GumEventSink * sink);
        
        
        ```
        
    *   底层`JS`引擎：是`QuickJS`或`V8`
*   要么是：`gum_stalker_follow(thread_id)`：去跟踪当前 process 进程中的其他某个线程 thread

### `gum_stalker_follow_me`的内部原理

```
#ifdef __APPLE__

.globl _gum_stalker_follow_me

_gum_stalker_follow_me:
#else
.globl gum_stalker_follow_me

.type gum_stalker_follow_me, %function

gum_stalker_follow_me:
#endif
stp x29, x30, [sp, -16]!

mov x29, sp

mov x3, x30

#ifdef __APPLE__

bl __gum_stalker_do_follow_me

#else
bl _gum_stalker_do_follow_me

#endif
ldp x29, x30, [sp], 16

br x0


```

-》

*   内部原理
    *   LR=Link Register=X30 = 链接寄存器
        *   AArch64 架构中，根据 LR 去决定从哪里开始跟踪
        *   当遇到 BR，BLR 等跳转指令时，会去设置 LR
            *   LR 被设置为，当前函数返回后，继续运行的地址
        *   由于只有一个 LR，如果被调用函数调用了其他函数，此时 LR 的值就会被保存起来，比如保存到 Stack 栈上，后续当 RET 指令执行之前，会重新把 LR 从 Stack 栈中加载到寄存器中，最终返回到调用者
    *   FP=Frame Pointer=X29 = 帧指针
        *   FP 始终指向 Stack top 栈的顶部，表示当前函数被调用时的栈的位置
        *   所以就可以通过固定的偏移量去访问到，所有通过 Stack 栈传入的参数和基于栈的局部变量
        *   且每个函数都有自己的 FP，所以需要调用新函数时，保存之前的 FP，返回之前函数时，恢复 FP。
        *   在刚进入新函数后，备份 FP 后，就可以去设置：mov x29, sp，把 SP 给 X29=FP 了。

保持了原先传入`x0-x2`的 3 个参数，额外加上`x3`=`x30`=`LR`，所以再去调用函数，就对应上参数了：

```
gpointer_gum_stalker_do_follow_me (GumStalker * self, GumStalkerTransformer * transformer, GumEventSink * sink, gpointer ret_addr)


```

### `gum_stalker_follow`的内部原理

和`gum_stalker_follow_me()`类似，但有额外参数：thread_id

```
void
gum_stalker_follow (GumStalker * self,
                    GumThreadId thread_id,
                    GumStalkerTransformer * transformer,
                    GumEventSink * sink)
{
  if (thread_id == gum_process_get_current_thread_id ())
  {
    gum_stalker_follow_me (self, transformer, sink);
  }
  else
  {
    GumInfectContext ctx;

    ctx.stalker = self;
    ctx.transformer = transformer;
    ctx.sink = sink;

    gum_process_modify_thread (thread_id, gum_stalker_infect, &ctx);
  }
}


```

其中：`gum_process_modify_thread()`，不属于`Stalker`，但属于`Gum`

回调 callback 会去修改：`GumCpuContext`

#### GumCpuContext

GumCpuContext 的定义：

```
typedef GumArm64CpuContext GumCpuContext;

struct _GumArm64CpuContext
{
  guint64 pc;
  guint64 sp;


  guint64 x[29];
  guint64 fp;
  guint64 lr;
  guint8 q[128];
};


```

相关：

```
static void
gum_stalker_infect (GumThreadId thread_id,
                    GumCpuContext * cpu_context,
                    gpointer user_data)


```

*   `gum_process_modify_thread()` 内部实现
    *   Linux/Android：ptrace
        *   GDB 也用的这个：挂载到进程上，读写寄存器

Stalker 每次只处理一个代码块 block

内部机制：

新申请一块内存，写入给原始代码中加了调试代码后的代码

加的指令，用于生成事件、提供其他 Stalker 所支持的功能。

以及根据情况去 relocate 重定位指令代码。

比如对于下面代码，要重定位：

*   ADR Address of label at a PC-relative offset.
*   `ADR Xd, label`
*   `Xd` Is the 64-bit name of the general-purpose destination register, in the range 0 to 31.
*   `label` Is the program label whose address is to be calculated. It is an offset from the address of this instruction, in the range ±1MB.

底层通过 Gum 的 Relocator

[frida-gum/gumarm64relocator.c at main · frida/frida-gum · GitHub](https://github.com/frida/frida-gum/blob/main/gum/arch-arm64/gumarm64relocator.c)

现在，回想一下我们说过潜行者一次工作一个块。那么我们如何检测下一个块呢？我们还记得每个块也以分支指令结尾，如果我们修改这个分支以分支回 Stalker 引擎，但确保我们存储分支打算结束的目的地，我们可以检测下一个块并在那里重定向执行。这个相同的简单过程可以一个接一个地继续。

Stalker = 潜行者

现在，这个过程可能有点慢，因此我们可以应用一些优化。首先，如果我们多次执行相同的代码块（例如循环，或者可能只是一个多次调用的函数），我们不必重新检测它。我们可以重新执行相同的检测代码。出于这个原因，我们保留了一个哈希表，其中包含我们之前遇到的所有块以及我们放置块的检测副本的位置。

其次，当遇到呼叫指令时，在发出检测的呼叫后，我们随后会发出一个着陆板，我们可以返回该着陆板而无需重新进入 Stalker。Stalker 使用记录真实返回地址（real_address）和此着陆垫（code_address）的 GumExecFrame 结构构建了一个侧堆栈。当一个函数返回时，我们发出代码，该代码将根据 real_address 检查侧堆栈中的返回地址，如果匹配，它可以简单地返回到 code_address，而无需重新进入运行时。这个着陆板最初将包含进入 Stalker 引擎以检测下一个块的代码，但稍后可以将其反向修补以直接分支到该块。这意味着可以处理整个返回序列，而无需输入和离开 Stalker。

如果返回地址与存储的 GumExecFrame real_address 不匹配，或者我们在侧堆栈中的空间不足，我们只需从头开始重新构建一个新的。我们需要在应用程序代码执行时保留 LR 的值，以便应用程序不能使用它来检测 Stalker 的存在（反调试），或者如果它将其用于除简单返回之外的任何其他目的（例如，在代码部分中引用内联数据）。此外，我们希望 Stalker 能够随时取消关注，所以我们不想不得不返回我们的堆栈来更正我们在此过程中修改的 LR 值。

最后，虽然我们总是用对 Stalker 的调用来替换分支以检测下一个块，但根据 Stalker.trustThreshold 的配置，我们可能会对此类检测代码进行反向修补，以将调用替换为下一个检测块的直接分支。确定性分支（例如，目的地是固定的，分支不是有条件的）很简单，我们可以将分支替换为下一个块。但是我们也可以处理条件分支，如果我们检测两个代码块（一个是分支，一个是不是）。然后，我们可以将原始条件分支替换为一个条件分支，该条件分支将控制流定向到获取分支时遇到的块的检测版本，然后是另一个检测块的无条件分支。我们还可以部分处理目标不是静态的分支。假设我们的分支是这样的：

BR X0 这种指令在调用函数指针或类方法时很常见。虽然 X0 的值可以更改，但通常它实际上总是相同的。在这种情况下，我们可以将最终的分支指令替换为代码，该代码将 X0 的值与我们的已知函数进行比较，如果它将分支与代码的检测副本的地址匹配。然后，如果不匹配，则可以将无条件分支返回到 Stalker 引擎。因此，如果函数指针 say 的值发生了变化，那么代码仍然有效，无论我们最终到达哪里，我们都将重新输入 Stalker 和乐器。但是，如果如我们预期的那样它保持不变，那么我们可以完全绕过 Stalker 引擎并直接进入仪器化功能。