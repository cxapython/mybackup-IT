> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-220753.htm)

> [翻译]ARM 汇编基础教程——ARM 指令集

**ARM 汇编基础教程**

——ARM 指令集

原文链接：https://azeria-labs.com/arm-instruction-set-part-3/

翻译：ljcnaix

ARM 和 Thumb
===========

ARM 处理器有两种工作状态 ARM 和 Thumb（Jazelle 此处先不考虑）。这两种工作状态和运行模式没有任何关系。比如不论是 ARM 还是 Thumb 状态的代码都可以运行在用户模式下。这两种工作状态之间最大的差异是指令集，ARM 状态的指令长度是 32 位的，Thumb 状态的指令长度是 16 位的（也可能为 32 位）。了解如何使用 Thumb 工作状态对于编写 ARM 平台的漏洞利用是至关重要的。当我们编写 ARM shellcode 时，需要使用 16 bit 的 Thumb 指令代替 32 bit 的 ARM 指令，从而避免在指令中出现’\0’截断。

容易引起混淆的是，不同的 ARM 版本，支持的 Thumb 指令集并不相同。在某些版本中，ARM 引入了扩展的 Thumb 指令集（也就是 Thumb-2），它支持 32 bit 指令以及条件执行。这在原本的 Thumb 指令中都是不受支持的。为了在 Thumb 状态下支持条件执行，“it” 指令被引入。然而，可能是为了简化指令集，这个指令在后来的版本中被删除了。我认为这种设计反而增加了兼容的复杂度。不过，当然我认为没必要知道所有 ARM 版本的 ARM/Thumb 指令集变体，我建议你也不必在这上面浪费太多时间。你只需要知道目标设备的版本和该版本对 Thumb 指令有哪些特殊支持，然后调整你的代码就好了。ARM Infocenter 可以帮助你了解各个 ARM 版本的具体细节（http://infocenter.arm.com/help/index.jsp）。

我们已经知道了 Thumb 有不同的版本，下面我们对不同的版本做一下简单的介绍，注意不同的命名只是为了区分不同的版本（换句话说，处理器只知道它运行在 Thumb 状态，其它一概不知）。

*   Thumb-1（16 位指令）：用于 ARMv6 和更早的版本。
    
*   Thumb-2（16 位和 32 位指令）：对 Thumb-1 的扩展，添加了更多指令并允许它们为 16 位或 32 位宽（ARMv6T2，ARMv7）。
    
*   ThumbEE：在 Thumb-2 基础上包含了针对动态代码生成（代码在执行前或执行期间编译代码）的一些变更和补充。
    

ARM 和 Thumb 的区别：

*   条件执行：ARM 状态下的所有指令都支持条件执行。某些 ARM 处理器版本允许使用 IT 指令在 Thumb 中进行条件执行。条件执行提高了代码密度，因为它减少了要执行的指令数量，并节省了昂贵的分支指令。
    
*   32 位 ARM 和 Thumb 指令：32 位 Thumb 指令具有.w 后缀。
    
*   桶形移位器是另一种 ARM 模式特有的功能。它可以将多个指令合并成一个。例如，您可以通过使用如下指令（将移位包含在 MOV 指令内）左移 1 位 “Mov R1，R0，LSL＃1; R1 = R0 * 2” 从而代替两个乘法指令（只用乘法指令将寄存器的值乘以 2，并使用 MOV 将结果存储到另一个寄存器中）。
    
    要切换处理器的执行状态，必须满足以下两个条件之一：
    
*   我们可以使用分支指令 BX（分支和状态切换）或 BLX（分支，返回和状态切换），并将目标寄存器的最低有效位置 1。这可以通过将 1 添加到偏移量（如 0x5530+1）来实现。你可能会认为这会导致对齐问题，因为指令总是 2 或 4 字节对齐的。然而，这么做不会导致问题，因为处理器在读取指令时是忽略最低有效位的。更多的细节将在第 6 篇：条件分支中介绍。
    
*   如果当前程序状态寄存器中的 T 位置 1，我们知道我们处于 Thumb 模式。
    

ARM 指令简介
========

这一节的目的是简要介绍 ARM 的指令集和它的基本用法。作为汇编语言的基本单位，了解指令的用法，指令间的如何关联以及将指令进行组合能实现什么功能对于学习汇编语言是至关重要的。

ARM 汇编由 ARM 指令组成。ARM 指令通常跟一到两个操作数，我们使用如下模板描述：

MNEMONIC{S}{condition} {Rd}, Operand1, Operand2

需要指出的是，只有部分指令用到了指令模板中的所有域。模板中各字段的作用如下所示：

MNEMONIC     - 指令的助记符如 ADD

{S}          - 可选的扩展位

               - 如果指令后加了 S，将依据计算结果更新 CPSR 寄存器中相应的 FLAG

{condition}  - 执行条件，如果没有指定，默认为 AL(无条件执行)

{Rd}         - 目的寄存器，存储指令计算结果

Operand1     - 第一个操作数，可以是一个寄存器或一个立即数

Operand2     - 第二个 (可变) 操作数

             - 可以是一个立即数或寄存器甚至带移位操作的寄存器

助记符、S 扩展位、目的寄存器和第一个操作数的作用很好理解，不多做解释，这里补充解释一下执行条件和第二个操作数。设置了执行条件的指令在执行指令前先校验 CPSR 寄存器中的标志位，只有标志位的组合匹配所设置的执行条件指令才会被执行。第二个操作数被称为可变操作数，因为它可以被设置为多种形式，包括立即数、寄存器、带移位操作的寄存器，如下所示：

#123         - 立即数

Rx           - 寄存器比如 R1

Rx, ASR n    - 对寄存器中的值进行算术右移 n 位后的值

Rx, LSL n    - 对寄存器中的值进行逻辑左移 n 位后的值

Rx, LSR n    - 对寄存器中的值进行逻辑右移 n 位后的值

Rx, ROR n    - 对寄存器中的值进行循环右移 n 位后的值

Rx, RRX      - 对寄存器中的值进行带扩展的循环右移 1 位后的值

最后我们来看一些满足上述指令模板的常见指令，这些指令将会在后续的例子中出现。

<table border="1" cellspacing="0" cellpadding="0"><tbody><tr><td width="78" valign="top">&nbsp;<p align="center">指令</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p align="center">描述</p>&nbsp;</td><td width="78" valign="top">&nbsp;<p align="center">指令</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p align="center">描述</p>&nbsp;</td></tr><tr><td width="78" valign="top">&nbsp;<p align="center">MOV</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>移动数据</p>&nbsp;</td><td width="78" valign="top">&nbsp;<p align="center">EOR</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>单比特异或</p>&nbsp;</td></tr><tr><td width="78" valign="top">&nbsp;<p align="center">MVN</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>取反码移动数据</p>&nbsp;</td><td width="78" valign="top">&nbsp;<p align="center">LDR</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>加载数据</p>&nbsp;</td></tr><tr><td width="78" valign="top">&nbsp;<p align="center">ADD</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>数据相加</p>&nbsp;</td><td width="78" valign="top">&nbsp;<p align="center">STR</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>存储数据</p>&nbsp;</td></tr><tr><td width="78" valign="top">&nbsp;<p align="center">SUB</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>数据相减</p>&nbsp;</td><td width="78" valign="top">&nbsp;<p align="center">LDM</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>多次加载</p>&nbsp;</td></tr><tr><td width="78" valign="top">&nbsp;<p align="center">MUL</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>数据相乘</p>&nbsp;</td><td width="78" valign="top">&nbsp;<p align="center">STM</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>多次存储</p>&nbsp;</td></tr><tr><td width="78" valign="top">&nbsp;<p align="center">LSL</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>逻辑左移</p>&nbsp;</td><td width="78" valign="top">&nbsp;<p align="center">PUSH</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>压栈</p>&nbsp;</td></tr><tr><td width="78" valign="top">&nbsp;<p align="center">LSR</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>逻辑右移</p>&nbsp;</td><td width="78" valign="top">&nbsp;<p align="center">POP</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>出栈</p>&nbsp;</td></tr><tr><td width="78" valign="top">&nbsp;<p align="center">ASR</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>算数右移</p>&nbsp;</td><td width="78" valign="top">&nbsp;<p align="center">B</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>分支</p>&nbsp;</td></tr><tr><td width="78" valign="top">&nbsp;<p align="center">ROR</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>循环右移</p>&nbsp;</td><td width="78" valign="top">&nbsp;<p align="center">BL</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>带返回的分支</p>&nbsp;</td></tr><tr><td width="78" valign="top">&nbsp;<p align="center">CMP</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>比较操作</p>&nbsp;</td><td width="78" valign="top">&nbsp;<p align="center">BX</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>带状态切换的分支</p>&nbsp;</td></tr><tr><td width="78" valign="top">&nbsp;<p align="center">AND</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>单比特与</p>&nbsp;</td><td width="78" valign="top">&nbsp;<p align="center">BLX</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>带返回、状态切换的分支</p>&nbsp;</td></tr><tr><td width="78" valign="top">&nbsp;<p align="center">ORR</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>单比特或</p>&nbsp;</td><td width="78" valign="top">&nbsp;<p align="center">SWI/SVC</p>&nbsp;</td><td width="210" valign="top">&nbsp;<p>系统调用</p>&nbsp;</td></tr></tbody></table>

[[2022 冬季班]《安卓高级研修班 (网课)》月薪三万班招生中～](https://www.kanxue.com/book-section_list-84.htm)

上传的附件：

*   [0x03ARM 汇编基础教程之 ARM 指令集. docx](javascript:void(0)) （25.38kb，327 次下载）