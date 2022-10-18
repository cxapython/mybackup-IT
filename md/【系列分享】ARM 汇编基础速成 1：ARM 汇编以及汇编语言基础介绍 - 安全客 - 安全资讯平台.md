> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/86383)

> 安全客 - 安全资讯平台

![](https://p3.ssl.qhimg.com/t01bdd3e4e75c7311fd.png)

译者：[arnow117](http://bobao.360.cn/member/contribute?uid=941579989)

预估稿费：200RMB

投稿方式：发送邮件至 linwei#360.cn，或登陆网页版在线投稿

**写在前面  
**

欢迎来到 ARM 汇编基础教程，这套教程是为了让你可以在 ARM 架构下进行漏洞利用打基础的。在我们能开始写 ARM 的 shellcode 以及构建 ROP 链之前，我们需要先学习相关的 ARM 汇编基础知识。

这些基础知识包括：

[Part 1：ARM 汇编介绍](https://azeria-labs.com/writing-arm-assembly-part-1/)

[Part 2：数据类型寄存器](https://azeria-labs.com/arm-data-types-and-registers-part-2/)

[Part 3: ARM 指令集](https://azeria-labs.com/arm-instruction-set-part-3/)

[Part 4: 内存相关指令：加载以及存储](https://azeria-labs.com/memory-instructions-load-and-store-part-4/)

[Part 5：重复性加载及存储](https://azeria-labs.com/load-and-store-multiple-part-5/)

[Part 6: 分支和条件执行](https://azeria-labs.com/arm-conditional-execution-and-branching-part-6/)

[Part 7：栈以及函数](https://azeria-labs.com/functions-and-the-stack-part-7/)

为了能跟着这个系列教程动手实践，你可以准备一个 ARM 的运行环境。如果你没有 ARM 设备（比如说树莓派或者手机），你可以通过 QEMU 来创建一个，[[教程在这]](https://azeria-labs.com/emulate-raspberry-pi-with-qemu/)。如果你对于 GDB 调试的基础命令不熟悉的话，可以通过 [[这个]](https://azeria-labs.com/debugging-with-gdb-introduction/) 学习。在这篇教程中，我们的核心关注点为 32 位的 ARM，相关的例子在 ARMv6 下编译。

**为什么是 ARM？**

前面说过，本系列教程的核心目的，是为那些想学习在 ARM 架构下进行漏洞利用的人而准备。可以看看你身边，有多少设备是 ARM 架构的， 手机，路由器，以及 IOT 设备，很多都是 ARM 架构的。无疑 ARM 架构已经成为了全世界主流而广泛的 CPU 架构。所以我们面对的越来越多的安全问题，也都会是 ARM 架构下的，那么在这种架构下的开发以及漏洞利用，也会成为一种主流趋势。

我们在 X86 架构上进行了很多研究，而 ARM 可能是最简单的广泛使用的汇编语言。但是人们为什么不关注 ARM 呢？可能是在 intel 架构上可供漏洞利用的学习资料比 ARM 多得多吧。比如 [[Corelan Team]](https://www.corelan.be/index.php/2009/07/19/exploit-writing-tutorial-part-1-stack-based-overflows/) 写的很棒的 intel X86 漏洞利用教程，旨在帮助我们可以更准确更高效的学习到关键的漏洞利用基础知识。如果你对于 x86 漏洞利用很感兴趣，那我觉得 Corelan Team 的教程是一个不错的选择。但是在我们这个系列里，我们要创造一本高效的 ARM 架构下的漏洞利用新手手册。

**ARM VS. INTEL**

ARM 处理器 Intel 处理器有很多不同，但是最主要的不同怕是指令集了。Intel 属于复杂指令集（CISC）处理器，有很多特性丰富的访问内存的复杂指令集。因此它拥有更多指令代码以及取址都是，但是寄存器比 ARM 的要少。复杂指令集处理器主要被应用在 PC 机，工作站以及服务器上。

ARM 属于简单指令集（RISC）处理器，所以与复杂指令集先比，只有简单的差不多 100 条指令集，但会有更多的寄存器。与 Intel 不同，ARM 的指令集仅仅操作寄存器或者是用于从内存的加载 / 储存过程，这也就是说，简单的加载 / 存储指令即可访问到内存。这意味着在 ARM 中，要对特定地址中存储的的 32 位值加一的话，仅仅需要从内存中加载到寄存器，加一，再从寄存器储存到内存即可。

简单的指令集既有好处也有坏处。一个好处就是代码的执行变得更快了。（RISC 指令集允许通过缩短时钟周期来加速代码执行）。坏处就是更少的指令集也要求了编写代码时要更加注意指令间使用的关系以及约束。还有重要的一点，ARM 架构有两种模式，ARM 模式和 Thumb 模式。Thumb 模式的代码只有 2 或者 4 字节。

ARM 与 X86 的不同还体现在：

ARM 中很多指令都可以用来做为条件执行的判断依据

X86 与 X64 机器码使用小端格式

ARM 机器码在版本 3 之前是小端。但是之后默认采用大端格式，但可以设置切换到小端。

除了以上这些 ARM 与 Intel 间的差异，ARM 自身也有很多版本。本系列教程旨在尽力保持通用性的情况下来讲讲 ARM 的工作流程。而且当你懂得了这个形式，学习其他版本的也很容易了。在系列教程中使用的样例都是在 32 位的 ARMv6 下运行的，所以相关解释也是主要依赖这个版本的。

不同版本的 ARM 命名也是有些复杂：

![](https://p3.ssl.qhimg.com/t01c8b371c998e1a4b0.png)

**写 ARM 汇编**

在开始用 ARM 汇编做漏洞利用开发之前，还是需要先学习下基础的汇编语言知识的。为什么我们需要 ARM 汇编呢，用正常的变成语言写不够么？的确不够，因为如果我们想做逆向工程，或者理解相关二进制程序的执行流程，构建我们自己的 ARM 架构的 shellcode，ROP 链，以及调试 ARM 应用，这些都要求先懂得 ARM 汇编。当然你也不需要学习的太过深入，足够做逆向工作以及漏洞利用开发刚刚好。如果有些知识要求先了解一些背景知识，别担心，这些知识也会在本系列文章里面介绍到的。当然如果你想学习更多，也可以去本文末尾提供的相关链接学习。

ARM 汇编，是一种更容易被人们接受的汇编语言。当然我们的计算机也不能直接运行汇编代码，还是需要编译成机器码的。通过编译工具链中 **as** 程序来将文件后缀为 ".s" 的汇编代码编译成机器码。写完汇编代码后，一般保存后缀为 ".s" 的文件，然后你需要用 **as** 编译以及用 **ld** 链接程序:

```
$ as program.s -o program.o
$ ld program.o -o program
```

![](https://p0.ssl.qhimg.com/t01f9b13c73d8c74949.gif)

**汇编语言本质**

让我们来看看汇编语言的底层本质。在最底层，只有电路的电信号。信号被格式化成可以变化的高低电平 0V(off) 或者 5V(on)。但是通过电压变化来表述电路状态是繁琐的，所以用 0 和 1 来代替高低电平，也就有了二进制格式。由二进制序列组成的组合便是最小的计算机处理器工作单元了，比如下面的这句机器码序列就是例子。

```
1110 0001 1010 0000 0010 0000 0000 0001
```

看上去不错，但是我们还是不能记住这些组合的含义。所以，我们需要用助记符和缩写来帮助我们记住这些二进制组合。这些助记符一般是连续的三个字母，我们可以用这些助记符作为指令来编写程序。这种程序就叫做汇编语言程序。用以代表一种计算机的机器码的助记符集合就叫做这种计算机汇编语言。因此，汇编语言是人们用来编写程序的最底层语言。同时指令的操作符也有对应的助记符，比如：

```
MOV R2, R1
```

现在我们知道了汇编程序是助记符的文本信息集合，我们需要将其转换成机器码。就像之前的，在 [[GNU Binutils]](https://www.gnu.org/software/binutils/) 工程中提供了叫做 **as** 的工具。使用汇编工具去将汇编语言转换成机器码的过程叫做汇编 (assembling)。

总结一下，在这篇中我们学习了计算机是通过由 0101 代表高低电平的机器码序列来进行运算的。我们可以使用机器码去让计算机做我们想让它做的事情。不过因为我们不能记住机器码，我们使用了缩写助记符来代表有相关功能的机器码，这些助记符的集合就是汇编语言。最后我们使用汇编器将汇编语言转换成机器可以理解的机器码。当然，在更高级别的语言编译生成机器码过程中，核心原理也是这个。

 **拓展阅读**

1. [Whirlwind Tour of ARM Assembly.]([https://www.coranac.com/tonc/text/asm.htm](https://www.coranac.com/tonc/text/asm.htm))

2. [ARM assembler in Raspberry Pi.]([http://thinkingeek.com/arm-assembler-raspberry-pi/](http://thinkingeek.com/arm-assembler-raspberry-pi/))

3. Practical Reverse Engineering: x86, x64, ARM, Windows Kernel, Reversing Tools, and Obfuscation by Bruce Dang, Alexandre Gazet, Elias Bachaalany and Sebastien Josse.

4. [ARM Reference Manual.]([http://infocenter.arm.com/help/topic/com.arm.doc.dui0068b/index.html](http://infocenter.arm.com/help/topic/com.arm.doc.dui0068b/index.html))

5. [Assembler User Guide.]([http://www.keil.com/support/man/docs/armasm/default.htm](http://www.keil.com/support/man/docs/armasm/default.htm))