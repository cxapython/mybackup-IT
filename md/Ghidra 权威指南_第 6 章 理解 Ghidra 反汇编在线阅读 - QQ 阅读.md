> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [book.qq.com](https://book.qq.com/book-read/46418409/15)

> QQ 阅读提供 Ghidra 权威指南, 第 6 章 理解 Ghidra 反汇编在线阅读服务, 想看 Ghidra 权威指南最新章节, 欢迎关注 QQ 阅读 Ghidra 权威指南频道, 第一时间阅读 Ghidra 权威指南最新章节!

第 6 章 理解 Ghidra 反汇编
-------------------

本章将介绍一些重要的基础技能，帮助你更好地理解 Ghidra 反汇编。我们将从基本的导航技术开始，在汇编代码中移动并检查遇到的每个制品（artifacts）。当从一个函数导航到另一个函数时，需要根据反汇编代码中的线索来解码每个函数的原型。因此，我们会讲解如何确定一个函数所接收参数的个数及其数据类型。由于一个函数所做的大部分工作都与其维护的局部变量有关，我们还会讲解函数如何使用堆栈来存放局部变量，以及在 Ghidra 的帮助下，准确理解函数如何利用它为自己保留的堆栈空间。无论是调试代码、分析恶意程序、还是开发漏洞利用，了解如何解码函数在堆栈上分配的变量，是帮助理解程序行为的基本技能。最后，讲解 Ghidra 提供的搜索选项以及它们如何帮助理解反汇编代码。

### 6.1 反汇编导航

在第 4 章和第 5 章中，我们看到 Ghidra 将许多常见逆向工程工具的功能集成到了 CodeBrowser 中显示。在显示中导航是掌握 Ghidra 所需的一项基本技能。一些静态反汇编代码，例如 objdump 等工具生成的清单，除了上下滚动，不提供任何导航功能。即使有文本编辑器提供了类似 grep 的搜索功能，在这样的代码清单中也很难导航。而 Ghidra 提供了出色的导航功能，除了提供文本编辑器的标准搜索功能，还提供了全面的交叉引用列表，其行为类似于网页的超链接。在大多数情况下，只需要双击即可导航到感兴趣的地方。

#### 6.1.1 名称和标签

当反汇编一个程序时，程序中的每个位置都分配有一个虚拟地址。因此，通过虚拟地址可以导航到程序中任何我们感兴趣的地方。不幸的是，在我们的脑子里维护一个地址目录是非常困难的事情。这一事实促使早期的程序员为他们想引用的程序位置分配符号名，从而简化这件事情。为程序地址分配符号名与为程序操作码分配助记符没有什么不同，通过使标识符更容易记忆，让程序变得更容易阅读和编写。Ghidra 延续了这个传统，为虚拟地址分配标签，并允许用户修改和扩展标签集。前面我们已经看到了如何使用与符号树窗口相关的名称，双击一个名称会让清单窗口（和符号引用窗口）跳转到被引用的位置。虽然在名称和标签这两个术语的使用上存在差异（例如，函数有名称，同时也在符号树的一个单独分支中有标签），但在导航上下文中，这两个术语基本上可以互换，因为它们都代表了导航目标。

Ghidra 在自动分析阶段使用二进制文件中的现有名称（如果有的话）或根据二进制文件中的位置引用方式自动生成一个符号名。除了象征目的，显示在反汇编窗口中的任何标签都是潜在的导航目标，类似于网页上的超链接。这些标签与标准超链接的主要区别是，没有以任何方式高亮显示，以表明它们可以被跟踪，而且 Ghidra 通常需要双击来跟踪标签，而传统超链接只需要单击。

**标签命名规则**

Ghidra 在分配标签时为用户提供了很大的灵活性，但某些命名模式具有特殊意义，并为 Ghidra 所保留。当下面这些前缀后紧跟下划线和地址时为保留模式，在分配标签时应避免使用：EXT、FUN、SUB、LAB、DAT、OFF 和 UNK。此外，标签中不允许有空格和不可打印字符。从好的方面来说，标签最多可以包含 2000 个字符，如果有超过这个限度的风险，请仔细数一数！

#### 6.1.2 在 Ghidra 中导航

在图 6-1 所示的图表中，实心箭头指示的每个符号代表一个命名的导航目标。在清单窗口中双击它们中的任意一个，都会使 Ghidra 清单窗口（及所有连接的窗口）重新定位到选中的位置。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_82_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Df5fb14b7a68df36ddf3f949c3291ceb0f7bd7319)

图 6-1：显示导航目标的清单

出于导航目的，Ghidra 将另外两个显示实体作为导航目标。首先，交叉引用（图 6-1 中用虚线箭头表示）被视为导航目标，双击底部的交叉引用地址将跳转到引用位置（本例中为 00401331）。关于交叉引用更详细的内容在第 9 章中讲解。将鼠标指针悬停在这些可导航的对象上，将显示一个包含目标代码的弹出窗口。

其次，另一种导航目标实体是十六进制数。如果十六进制数表示二进制文件中的一个有效虚拟地址，那么关联的虚拟地址将显示在其右侧，如图 6-2 所示。双击显示的值将重新定位反汇编窗口到相关的虚拟地址处。在图 6-2 中，双击实心箭头所指的任意一个值都会跳转显示，因为这些值都是二进制文件中的有效虚拟地址，而双击其他任何值都不会有什么反应。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_83_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D637d9291a1b5abc6b8e2bd7ecb175172f76bb541)

图 6-2：显示十六进制导航目标的清单

#### 6.1.3 Go To 对话框

如果你知道想要导航的地址或名称时（例如，导航到 ELF 二进制文件的 main 函数以开始分析），可以滚动清单来找到地址，滚动符号树窗口中的函数文件夹来找到所需名称，或者使用 Ghidra 的搜索功能（本章稍后讲解）。但是，最简单的方法是使用 Go To 对话框（如图 6-3 所示），在反汇编窗口中选择 Navigation→Go To 或热键 G 可以打开。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_83_2.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D8b1f163b44904d463289b72e27f047c30584020b)

图 6-3：Go To 对话框

导航到二进制文件中的任意位置非常简单，只需要指定一个有效地址（区分大小写的符号名或十六进制数），然后单击 “OK” 按钮，显示窗口就会立即跳转到该地址。输入对话框的值可通过下拉历史列表查看和使用，可以快速回到之前请求过的地址处。

#### 6.1.4 导航历史

Ghidra 支持根据你导航反汇编代码的历史进行前进或后退导航。每次你在反汇编中导航到新位置时，当前位置就会被添加到历史列表中。通过 Go To 窗口或 CodeBrowser 工具栏中的左右箭头按钮可以遍历此列表。

在图 6-3 的 Go To 窗口中，文本框右侧的箭头会打开历史列表，可以从中选择。在 CodeBrowser 工具栏的左侧可以看到类似浏览器的前进和后退按钮，如图 6-4 所示。每个按钮都关联了一个下拉历史列表，提供对导航历史中任意位置的即时访问，而不必一步步回溯。图 6-4 展示了一个与后退箭头关联的下拉历史列表示例。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_84_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Da34a7ac83d429e6d91d463162003ce931b5138dd)

图 6-4：带历史列表的前进和后退导航箭头

ALT-left 箭头（Mac 上的 OPTION-left 箭头）用于后退导航，是需要记住的最有用的热键之一。当你跟随一连串的函数调用深入到好几层，然后决定要返回反汇编中的起始位置时，后退导航非常方便。ALT-right 箭头（Mac 上的 OPTION-right 箭头）则用于在历史列表中做前进导航。

现在我们已经对 Ghidra 中的反汇编导航有了更清晰的认识，但是仍然没有对导航目标赋予什么意义。下一节我们将研究为何某些函数（特别是涉及栈帧的）会成为逆向工程师的重要导航目标。

### 6.2 栈帧

由于 Ghidra 是一个底层分析工具，它的许多功能和显示都要求用户对底层的汇编语言有所了解，特别是机器语言的生成和高级语言的内存管理。Ghidra 特别关注编译器处理局部变量的声明和访问的方式。你可能已经注意到，在大多数函数清单的开头，有很多行是专门用于局部变量的。这些行是 Ghidra 通过堆栈分析器对每个函数进行详细堆栈分析的结果。这种分析是必要的，因为编译器会将函数的局部变量（在某些情况下是函数的传入参数）放在栈上分配的内存块中。本节将回顾编译器如何处理局部变量和函数参数，以帮助更好地理解 Ghidra 的清单视图。

#### 6.2.1 函数调用机制

一个函数调用可能需要内存来存放以参数形式传递给函数的信息，以及为函数执行保留临时存储空间。参数值及其内存地址需要存放在函数可以找到的地方。临时空间通常由程序员通过声明局部变量来分配，这些变量可以在函数内使用，但在函数完成后就不能再访问了。栈帧（也称为激活记录）是在程序运行时栈上分配的内存块，专门用于函数调用。

编译器使用栈帧来让函数参数和局部变量的分配和释放对程序员透明。对于在栈上传递参数的调用约定，编译器在将控制权转移给函数本身之前，插入了将函数参数放入栈帧的相关代码，同时分配足够的内存来存放函数的局部变量。在某些情况下，函数的返回地址也存放在新栈帧中。栈帧还支持递归 [[1]](https://book.qq.com/book-read/Text/Section0011.xhtml#n9E673B56CD104734887A7D992A99BB11)，对函数的每个递归调用都有自己的栈帧，从而实现每个调用的隔离。

调用函数时会发生以下操作：

（1）调用者根据被调用函数所采取的调用约定，将其所需的所有参数放到相应的位置。如果是在运行时栈上传递参数，则程序堆栈指针可能会发生改变。

（2）调用者通过调用指令（如 x86 CALL、ARM BL 和 MIPS JAL）将控制权转移给被调用函数。返回地址被存放到程序栈或寄存器中。

（3）必要时，被调用函数会配置栈帧指针 [[2]](https://book.qq.com/book-read/Text/Section0011.xhtml#n6CC3CC9ACA984CBCA156A9FB49BFC43B)，将调用者希望保持不变的寄存器值保存下来。

（4）被调用函数为其可能需要的所有局部变量分配空间。通过调整程序栈指针，可以在运行时栈上获得空间。

（5）被调用函数执行其操作，可能会访问传递给它的参数，并生成执行结果。结果通常放在一个或多个特定的寄存器中，调用者可以在被调用函数返回后查看。

（6）当函数完成其操作时，为局部变量保留的所有堆栈空间都会被释放。这一步通常是步骤（4）的逆操作。

（7）恢复在步骤（3）中保存下来的值到相应寄存器。

（8）被调用函数将控制权返还给调用者。这一步骤的典型指令包括 x86 RET、ARM POP 和 MIPS JR。根据使用的调用约定，此操作还可能会从程序栈中清除一个或多个参数。

（9）一旦调用者重新获得控制权，它可能需要将程序堆栈指针恢复为步骤（1）之前的值，从而删除程序堆栈中的参数。

步骤（3）和步骤（4）通常在进入函数时执行，称为函数序言（prologue）。类似地，步骤（6）到步骤（8）称为函数尾声（epilogue）。除了步骤（5），所有这些操作都是与函数调用相关开销的一部分，这在高级语言编写的程序源代码中可能并不明显，但在汇编语言里很容易观察到。

**它们真的消失了吗？**

当我们说从堆栈中删除项目，以及删除整个栈帧时，我们的意思是堆栈指针被调整了，指向了栈中位置较低的数据，被删除的内容不能再通过 POP 操作访问。但该内容在被 PUSH 操作覆盖之前，它仍然存在。从编程的角度来看，这符合删除的条件。从数字取证的角度来看，你只需要再仔细一点就能找到这些内容。从变量初始化的角度来看，这意味着栈帧中任何未经初始化的局部变量都有可能包含上一次使用该栈帧时遗留在内存中的值。

#### 6.2.2 调用约定

当从调用者向被调用者传递参数时，调用函数必须完全按照被调用函数所期望的方式来存放参数，否则就会出现严重的问题。调用约定决定了调用者应该把函数所需的参数存放在哪里，是程序栈还是寄存器加程序栈。当参数被传递到程序栈上时，调用约定也决定了在被调用函数结束后，由谁负责将它们从栈上清除，是调用者还是被调用者。

无论你在逆向哪种架构，如果不了解所使用的调用约定，那么理解围绕函数调用的代码就会很困难。在接下来的章节里，我们将回顾在编译 C 和 C++ 代码时常见的一些调用约定。

**栈和寄存器参数**

函数参数可以在寄存器中传递，也可以在程序栈中传递，或者两者相结合。当参数被放在栈上时，调用者执行内存写入操作（通常是 PUSH）来将参数放到栈上，然后被调用函数执行内存读取操作来访问该参数。为了加快函数调用过程，一些调用约定使用寄存器来传递参数，这样做不需要执行内存读写操作，因为参数在指定的寄存器中可以直接被函数使用。基于寄存器的调用约定有一个缺点，就是处理器的寄存器数量是有限的，而函数的参数数量可以是任意的，所以必须正确处理那些需要更多参数的函数，多余的参数通常被放到栈上。

**C 调用约定**

C 调用约定（C calling convention）是大多数 C 语言编译器在生成函数调用时使用的默认调用约定。在函数原型中使用_cdecl 关键字，可以在 C/C++ 程序中强制使用这种调用约定。cdecl 调用约定规定，调用者按从右到左的顺序将参数放在栈上，并且在被调用者执行结束后，由调用者（而不是被调用者）负责将参数从栈中清除。对于 32 位的 x86 二进制文件，cdecl 将所有参数放在栈上。对于 64 位的 x64 二进制文件，cdecl 在不同操作系统上有差异，在 Linux 上，最多 6 个参数被依次放在 RDI、RSI、RDX、RCX、R8 和 R9 寄存器中，其余参数则放在栈上。对于 ARM 二进制文件，cdecl 将前 4 个参数放在 R0 到 R3 寄存器中，其余参数则放在栈上。

栈分配的参数按从右到左的顺序放在栈上，在函数被调用时，最左边的参数总是位于栈顶。因此，无论函数有多少个参数，都可以轻松找到第一个参数，这使得 cdecl 调用约定非常适合可变参数函数（例如 printf）。

要求调用者清理栈上的参数，意味着你经常会在被调用函数返回后，看到紧跟着一段调整栈指针的指令。对于可变参数函数，调用者明确知道它传递的参数个数，也就可以很容易地做出正确的调整，而被调用函数是无法提前知道它会收到多少参数的。

在下面的例子中，我们来看 32 位的 x86 二进制文件如何调用函数，每个函数都使用不同的调用约定。第一个函数的原型如下：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_86_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D552bcfe008ef659ca41c562b050fc90553068db4)

默认情况下，这个函数将使用 cdecl 调用约定，4 个参数按从右到左的顺序入栈，并且由调用者负责清理栈上的参数。函数调用示例如下：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_87_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Ded1dc05e03af6bb50fe049d3195daa91edeada78)

编译器可能会生成如下代码：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_87_2.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D94c62396fb3641ca0afba7480d54160d7a7fe9f0)

4 个 PUSH 指令操作❶将程序的栈指针（ESP）改变了 16 个字节（32 位架构上的 4*sizeof（int）），在从 demo_cdecl 返回后立即恢复❷。以下技术已经在某些版本的 GNU 编译器（gcc 和 g++）中使用，它遵循 cdecl 调用约定，同时无须调用者在每次调用 demo_cdecl 之后显式地清除栈上的参数。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_87_3.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D2201ce842258857ffb20de6c070be1957a33e492)

在这个例子中，demo_cdecl 的参数入栈时没有改变程序的栈指针。注意，无论哪种方式，在函数调用时栈指针都指向最左边的参数。

**标准调用约定**

在 32 位的 Windows DLL 文件中，微软大量使用了一种称为标准调用约定（standard calling convention）的调用方式。在源代码中，通过在函数声明时使用_stdcall 修饰符来实现，如下所示：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_87_4.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Deb46660dcc37d5845a7c424e4942f391677ddc66)

为了避免 “标准” 一词可能造成的误会，在本书的其余部分，我们将这种调用约定称为 stdcall 调用约定。

stdcall 调用约定要求将栈分配的函数参数按从右到左的顺序放在栈上，但在被调用函数结束后，由被调用函数负责清理栈上的参数。这样做只适用于固定参数的函数，而可变参数函数，如 printf，不能使用 stdcall 调用约定。

demo_stdcall 函数有 3 个整型参数，在栈上总共占用 12 字节（32 位架构上的 3*sizeof（int））。x86 编译器使用一种特殊形式的 RET 指令，在从栈顶弹出返回地址的同时调整栈指针以清除栈上的参数。在 demo_stdcall 这个例子中，我们可能会看到如下指令用于返回到调用者：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_87_5.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D4c84c6a8bcbf96fa52ad86fbea4c0e9aa2587f28)

使用 stdcall 就不需要在每次函数调用后清除栈上的参数，从而使程序体积更小，运行更快。按照约定，微软对所有从 32 位共享库（DLL）文件中导出的固定参数函数使用了 stdcall 调用约定。当你在为共享库组件生成函数原型或者开发二进制兼容的替代品时，记住这一点是非常重要的。

**x86 的 fastcall 调用约定**

微软 C/C++ 和 GNU gcc/g++（3.4 及以后版本）编译器支持 fastcall 调用约定，这是 stdcall 约定的一个变体，将前两个参数分别放在 ECX 和 EDX 寄存器中。其余参数则按从右到左的顺序放在栈上，被调用函数在返回时负责清除栈上的参数。fastcall 调用约定的声明示例如下：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_88_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Df6a11f01b623c74a991ee7012fc5bfdbcc732dc4)

函数调用示例如下：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_88_2.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3De59721dfd6a541f474c414bfd85be6c32b114b52)

编译器可能会生成如下代码：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_88_3.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Dc06f2c29661c4958feeaa6d7aa797dd82bb9b72d)

从 demo_fastcall 返回时不需要对栈进行调整，因为 demo_fastcall 负责在返回给调用者时从栈中清除参数 y 和 z。需要注意的是，虽然函数有 4 个参数，但因为前 2 个参数是以寄存器的形式传递的，所以被调用函数只需要从栈中清除 8 字节。

**C++ 调用约定**

C++ 类中的非静态成员函数必须提供一个用于调用该函数的对象指针（this 指针）[[3]](https://book.qq.com/book-read/Text/Section0011.xhtml#nD0AD066DE6FF4C4EA9EDBD28800A6708)。用于调用该函数的对象地址必须由调用者作为参数提供，但 C++ 语言标准并没有规定应该如何传递，所以不同编译器可能会使用不同的实现方式。

在 x86 上，微软的 C++ 编译器利用 thiscall 调用约定，在 ECX/RCX 寄存器中传递 this 指针，并要求非静态成员函数清理栈上的参数，就像 stdcall 一样。GNU g++ 编译器将 this 指针视为非静态成员函数隐含的第一个参数，其他方面的行为与 cdecl 约定一样。因此，对于 g++ 编译的 32 位代码，在调用非静态成员函数之前，this 指针被放在栈顶，调用者负责在函数返回后清理栈上的参数（总是至少有一个）。关于编译后 C++ 程序的其他特征在第 8 章和第 20 章中讲解。

**其他调用约定**

要想完全覆盖每一种调用约定可能需要单出一本书。调用约定通常是与操作系统、编程语言、编译器和处理器相关的。当遇到由不常见编译器生成的代码时，可能需要自己研究一下。不过，还有一些情况值得一提：代码优化、自定义汇编代码和系统调用。

当函数被导出给其他程序使用时（例如库函数），它们必须遵守公认的调用约定，以便程序员很容易地与这些函数进行对接。另一方面，如果一个函数只供程序内部使用，那么该函数的调用约定就只需程序内部知道即可。在这种情况下，编译器可以进行优化，使用更快的调用约定来替代原来的代码。例如，在微软 C/C++ 中使用 / GL 选项，指示可以执行整个程序的优化，这可能使跨函数的寄存器使用得到优化。而在 GNU gcc/g++ 中使用 regparm 关键字，可以让最多 3 个参数使用寄存器来传递。

程序员使用汇编语言来开发程序时，可以完全控制参数如何传递给任意函数。除非想要将函数提供给其他人使用，否则他们可以自由地以最合适的方式传递参数。因此，在分析自定义汇编代码时要格外小心，例如被混淆的程序和 shellcode。

系统调用是一种特殊的函数调用，用于请求操作系统的服务。系统调用通常会影响从用户模式到内核模式的状态转换，以使操作系统内核为用户的请求提供服务。在不同的操作系统和处理器中，系统调用的启动方式也有所不同。例如，32 位的 Linux x86 系统调用可以使用 INT 0x80 指令或 sysenter 指令来启动，其他 x86 操作系统可能只能使用 sysenter 指令或备用中断号，而 64 位的 x64 代码则使用 syscall 指令。在许多 x86 系统中（Linux 除外），系统调用的参数被放在运行时栈上，系统调用号放在 EAX 寄存器中，而 Linux 系统调用通过特定的寄存器传递参数，当参数个数过多时，偶尔也会通过栈来传递。

#### 6.2.3 栈帧的其他思考

在任何处理器上，寄存器都是一种有限的资源，需要在程序中的所有函数之间合作共享。当一个函数（func1）正在执行时，它会觉得自己完全控制了所有的处理器寄存器。当 func1 调用了另一个函数（func2）时，func2 也会有相同的感觉，并根据自己的需要使用所有可用的处理器寄存器，但如果 func2 对寄存器进行任意修改，则可能会破坏 func1 所依赖的值。

为了解决这个问题，所有的编译器都遵循明确定义的寄存器分配和使用规则。这些规则通常被称为平台的应用程序二进制接口（ABI）。ABI 将寄存器分为两类：调用者保存的寄存器和被调用者保存的寄存器。当一个函数调用另一个函数时，调用者需要将调用者保存的寄存器保存下来，以防止内容丢失。被调用者保存的寄存器则必须由被调用者进行保存，然后被调用函数才能任意使用它们。这些操作通常是作为函数序言的一部分进行的，在返回之前，调用者保存的寄存器会在函数尾声中恢复。调用者保存的寄存器称为 clobber 寄存器，因为被调用函数无须先进行保存就可以随意修改它们的内容。相反，被调用者保存的寄存器称为 no-clobber 寄存器。

英特尔 32 位处理器的 System V ABI 规定，调用者保存的寄存器包括 EAX、ECX 和 EDX，被调用者保存的寄存器包括 EBX、EDI、ESI、EBP 和 ESP[[4]](https://book.qq.com/book-read/Text/Section0011.xhtml#n41CB00586E0B470CAFFB48EF13C6BFB8)。你可能会注意到，编译器通常更喜欢在函数中使用调用者保存的寄存器，因为它们不必在进入和退出函数时保存和恢复其内容。

#### 6.2.4 局部变量布局

与规定参数如何传入函数的调用约定不同，函数局部变量的内存布局没有约定来进行规范。在编译函数时，编译器必须计算函数局部变量所需的空间，以及保存 no-clobber 寄存器所需的空间，并确定这些变量是可以分配给处理器寄存器，还是必须分配在程序栈上。这些分配的具体方式与函数调用者和被调用者都无关，而且一般来说，仅根据对函数源代码的检查，是不可能确定函数局部变量布局的。有一点是确定的：编译器必须至少拿出一个寄存器来记住函数新分配栈帧的位置。这个寄存器很明显是栈指针，指向当前函数的栈帧。

#### 6.2.5 栈帧示例

当执行一项复杂任务时，例如对二进制文件进行逆向工程，你应该有效地利用时间。在理解一个反汇编函数的行为时，花在检查通用代码序列上的时间越少，那么花在复杂序列上的时间就越多。函数序言和尾声就是这样的通用代码序列，重要的是你能够识别它们，理解它们，然后迅速地转到其他更有趣、需要更多思考的代码上去。

Ghidra 在每个函数开头的局部变量列表中总结了它对函数序言的理解，虽然它可能使代码更易读，但并没有减少你需要阅读的反汇编代码的数量。在下面的示例中，我们会讲解两种常见的栈帧类型，并回顾创建每种栈帧所需的代码，以便你在遇到类似的代码时，可以快速略过从而把握函数的核心内容。

下面是在 32 位 x86 计算机上编译的函数：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_90_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Ddba0ef1e9fa8b635aea1c72356b5748fd8e69db4)

demo_stackframe 的局部变量需要 76 字节（三个 4 字节的整数和一个 64 字节的缓冲区）。这个函数可以使用 stdcall 或者 cdecl，并且栈帧看起来是一样的。

**示例 1：通过栈指针访问局部变量**

图 6-5 显示了调用 demo_stackframe 的一个栈帧示例。在这个例子中，编译器选择使用栈指针来访问栈帧中包含的变量，而将其他所有寄存器留作其他用途。如果有任何指令改变了栈指针的值，则编译器必须确保在所有后续的局部变量访问中考虑这一变化。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_91_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D694d3a1e07f657d48bae0384bc171db69b019875)

图 6-5：在 32 位 x86 计算机上编译函数的栈帧示例

该栈帧的空间是在 demo_stackframe 的入口处，通过下面这一行序言设置的：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_91_2.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D5386fdae76eb9da6e2cd7c1a4cf5f70dabf923ba)

图 6-5 中的偏移列（Offset）表示 x86 寻址模式（本例中为基址 + 位移），用来引用栈帧中的每个局部变量和参数。在本例中，ESP 被用作基址寄存器，每个位移都是栈帧中从 ESP 到变量开头的相对偏移量。然而，图 6-5 中的位移只有当 ESP 的值不发生变化时才是正确的。不幸的是，栈指针经常发生变化，因此编译器必须不断调整，才能确保在引用栈帧内的任意变量时使用了正确的偏移量。如下代码是在函数 demo_stackframe 中对 helper 的调用：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_91_3.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D878feb6a859d70892ee8bb40f4bc4ca535742820)

第一个 PUSH❶指令按照图 6-5 中的偏移量将局部变量 y 放入栈。乍一看，第二个 PUSH❷指令似乎会错误地再次放入局部变量 y。但是，因为栈帧中的所有变量都是相对于 ESP 来引用的，并且第一次 PUSH❶改变了 ESP，所以图 6-5 中的所有偏移量都需要进行调整。因此，在第一次 PUSH❶之后，局部变量 z 的新偏移量为 [ESP+4]。在检查那些使用栈指针引用栈帧变量的函数时，必须注意栈指针的所有变化，并相应地调整后续的所有偏移量。

demo_stackframe 执行完成后，它需要返回给调用者。最终，RET 指令会从栈顶将所需的返回地址弹出到指令指针寄存器中（本例中为 EIP）。在弹出返回地址之前，需要将局部变量从栈顶移除，以便在执行 RET 指令时栈指针能正确地指向保存的返回地址。在本例中（假设使用了 cdecl 调用约定），函数尾声如下所示：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_91_4.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Da8bf412c409e70df025419ad0f750589a0968542)

**示例 2：让栈指针休息一下**

如果使用第二个寄存器来定位栈帧中的变量，那么栈指针就可以自由地改变，而不需要为栈帧中的每个变量重新计算偏移量。当然，编译器需要保证第二个寄存器不会改变，否则，就会出现与上一个例子相同的问题。在本例中，编译器首先需要为此选择一个寄存器，然后生成代码在进入函数时初始化该寄存器。

为此目的选择的寄存器称为帧指针（frame pointer）。在前面的例子中，ESP 被用作帧指针，我们称之为基于 ESP 的栈帧。大多数体系结构的 ABI 会有建议，将哪一个寄存器用作帧指针。帧指针通常被认为是一个 no-clobber 寄存器，因为调用函数可能已经将其用于相同的目的。在 x86 程序中，EBP/RBP（扩展了基指针）寄存器通常专门用作帧指针。默认情况下，大多数编译器会生成代码，使用栈指针以外的寄存器来作为帧指针，尽管通常存在使用栈指针来作为帧指针的选项（例如，GNU gcc/g++ 提供了 - fomit-frame-pointer 编译器选项，它生成的函数不使用第二个寄存器来作为帧指针。）

在使用专用帧指针寄存器时，demo_stackframe 生成栈帧的序言代码如下所示：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_92_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D91c8d90c35ab8281c8fa66f4e94916999a00258f)

PUSH❶指令保存当前调用者的 EBP 值，因为 EBP 是一个 no-clobber 寄存器。在被调用函数返回时，必须恢复调用者的 EBP 值。如果调用者的其他寄存器（例如 ESI 和 EDI）也需要保存，那么编译器可以在保存 EBP 的同时保存它们，也可以推迟到分配完局部变量之后再保存它们。因此，栈帧中并没有一个保存寄存器的标准位置。

EBP 被保存之后，就可以通过 MOV❷指令将其修改为指向当前栈帧的位置，也就是复制一份当前栈指针（当前时刻唯一指向栈帧的寄存器）的值。最后，与基于 ESP 的栈帧一样，为局部变量分配空间❸。生成的栈帧布局如图 6-6 所示。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_92_2.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3De50dd463e2dae58604c9a7654868f36f7cabeced)

图 6-6：基于 EBP 的栈帧示例

有了专门的帧指针，所有的变量偏移量现在都可以相对于帧指针寄存器来进行计算，如图 6-6 所示。最常见的情况是（不一定是），正偏移用来访问在栈上分配的函数参数，而负偏移用来访问局部变量。由于使用了专门的帧指针，所以，栈指针可以自由改变，而不会影响栈帧上的任何变量。现在对 helper 函数的调用可以这样实现：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_93_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D9ea8e5e71f8801a6a0d342a2320c1dfd8b22f7b0)

第一个 PUSH❹指令改变了栈指针，但是不会影响后续访问变量 z 的 PUSH 指令。

在使用帧指针的函数的尾声部分，必须在返回前恢复调用者的帧指针。如果是使用 POP 指令来恢复帧指针，那么在弹出帧指针的旧值之前，必须先清除栈帧中的局部变量。由于当前帧指针指向栈帧中 Saved EBP 的位置，在使用 EBP 作为帧指针的 32 位 x86 程序中，这一恢复过程可以通过如下代码实现：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_93_2.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Df5da86560d94c7029f0abfeeb3f064e2c52842ea)

这种操作非常普遍，以至于 x86 架构提供了 LEAVE 指令来完成同样的任务：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_93_3.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Db14fe28077c9b26c7200e85bb50450aa038bb079)

虽然其他处理器架构使用的寄存器和指令肯定会有所不同，但创建栈帧的基本过程将保持不变。不管是哪种架构，你都要熟悉典型的序言和尾声序列，以便快速略过并专注于分析函数中更有趣的代码。

### 6.3 Ghidra 栈视图

栈帧是一个运行时概念，没有栈，没有运行中的程序，栈帧也就无法存在。虽然这是事实，但并不意味着你在使用 Ghidra 等工具进行静态分析时就应该忽略栈帧的概念。二进制文件中包含了为每个函数创建栈帧所需的所有代码。通过仔细分析这些代码，即使函数并没有运行，我们也可以详细了解任何函数的栈帧布局。事实上，Ghidra 会进行一些最复杂的分析，专门用于确定它所反汇编的每个函数的栈帧布局。

#### 6.3.1 Ghidra 栈帧分析

在初始化分析中，Ghidra 会跟踪栈指针在一个函数执行时的行为，记录每一个 PUSH 或 POP 操作以及任何可能改变栈指针的算术操作，如添加或减去常数值。该分析的目的是确定分配给函数栈帧的局部变量区域的确切大小，确定给定函数中是否使用了专门的帧指针（例如识别 PUSH EBP/MOV EBP，ESP 序列），并识别函数栈帧中对变量的所有内存引用。

例如，Ghidra 如果发现如下指令在 demo_stackframe 函数体中，它会认为函数的第一个参数（本例中为 a）被加载到 EAX 寄存器（参考图 6-6）中。Ghidra 可以区分内存中访问函数参数的引用（位于保存的返回地址下方）和访问局部变量的引用（位于保存的返回地址上方）。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_94_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D84c602ea648f78bdc801b8a15f3249f69431bbd1)

Ghidra 还采取了额外的措施来确定栈帧中哪些内存地址被直接引用。例如，虽然图 6-6 中的栈帧有 96 字节大小，但我们可以看到被引用的变量只有 7 个（4 个局部变量和 3 个参数）。因此，你可以把注意力集中在 Ghidra 认为重要的那 7 个变量上，而不用花太多时间去想那些 Ghidra 没有命名的字节。在识别和命名栈帧中各个变量的过程中，Ghidra 还能识别出变量之间的空间关系。这在某些时候非常有用，例如漏洞利用开发，Ghidra 可以很容易地确定哪些变量可能因缓冲区溢出而被覆盖。Ghidra 的反编译器（在第 19 章中讲解）也在很大程度上依赖栈帧分析，它利用分析结果来推断函数接收多少个参数，以及在反编译代码中需要声明哪些局部变量。

#### 6.3.2 清单视图中的栈帧

理解一个函数的行为通常可以归结为理解该函数所操作的数据类型。在阅读反汇编清单时，要想理解函数所操作的数据，其中一个方法是查看函数栈帧的分解内容。Ghidra 为函数栈帧提供了两个视图：摘要视图和详细视图。我们通过如下版本的 demo_stackframe 来理解这两种视图，使用 gcc 进行编译：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_94_2.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D70e92f58c902f55575d17955fe9a61fccc796007)

由于局部变量只在函数运行时存在，任何没有在函数中以有意义的方式使用的局部变量实际上都是没有用的。如下代码是一个与 demo_stackframe 功能相同（或者说是优化）的版本：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_94_3.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D1bf50e258b51637d06f82fa5b8d3e5f566aeca88)

在原始版本的 demo_stackframe 中，局部变量 x 和 y 分别由参数 k 和 j 初始化。局部变量 z 被初始化为字面值 10，命名为 buffer 的 64 字节局部数组的第一个字符被初始化为字符'A'。图 6-7 是该函数对应的 Ghidra 反汇编，使用了默认的自动分析。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_95_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3De9bb4ca6ebc40f64780bc67f3f62008dbd290f42)

图 6-7：demo_stackframe 函数的反汇编

当我们开始熟悉 Ghidra 的反汇编符号时，这个清单中有许多要点要介绍。这里我们重点关注反汇编代码中的两个部分，它们提供了特别有用的信息。我们从栈帧摘要开始，如下所示（可以参考图 6-7，看看该栈帧摘要的上下文）。为了简化讨论，我们用局部变量和参数这两个术语来区分两种类型的变量，而变量这个术语用于统称。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_95_2.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D5f0702242c4b2a4f75ed2599b734669afa98f835)

Ghidra 提供了一个栈帧摘要视图，列出了栈帧中直接引用的每个变量，以及每个变量的重要信息。Ghidra 给每个变量分配了有意义的名字（在第三列），当你在反汇编清单中看到它们时，就获得了关于参数类型的信息：传递给函数的参数名以 param_作为前缀，而局部变量名以 local_作为前缀。因此，可以很容易地区分这里两种类型的变量。

变量名前缀还与变量的位置信息有关。对于参数，如 param_3，名称中的数字对应于参数在函数参数列表中的位置。对于局部变量，如 local_10，数字是一个十六进制偏移量，表示变量在栈帧中的位置。位置信息也可以在清单的中间一列中找到，位于名称的左边。该列有两个组成部分，用冒号隔开：Ghidra 对变量大小的估计（以字节为单位），以及变量在栈帧中的位置（在进入函数时该变量与初始栈指针值的偏移量）。

图 6-8 是这个栈帧示例的表格形式。如前所述，参数位于保存的返回地址下方，因此与返回地址有一个正偏移。局部变量位于保存的返回地址上方，因此有一个负偏移。栈上局部变量的顺序与本章前面看到的源代码中声明的顺序不一致，因为编译器可以根据各种内部因素，在栈上自由排布局部变量，例如字节对齐和数组相对于其他局部变量的位置等。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_96_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D9ef49ba78ab06e25fab7771825b7f05f2fab201a)

图 6-8：栈帧示例表格

#### 6.3.3 反编译辅助栈帧分析

回顾一下前面提到的这段等效函数代码：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_96_2.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D11596271d0a0652dc412b67cebcdb175738d058d)

Ghidra 反编译器为此函数生成的代码如图 6-9 所示，与优化过的等效函数代码非常相似，因为反编译器仅包含了原始函数的可执行部分（除了包含 param_1 参数）。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_96_3.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Dda105b5e1550306c190dcd197984564b8c167870)

图 6-9：demo_stackframe 反编译器窗口

你可能已经注意到，函数 demo_stackframe 接受三个整型参数，但反编译清单中只包含其中的两个（param_1 和 param_2）。哪一个不见了，为什么？事实证明，Ghidra 的反汇编器和反编译器对命名的处理方式略有不同，虽然它们都命名了所有直到最后一个引用的参数，但反编译器只命名到最后一个以有意义的方式使用的参数。反编译器参数 ID 分析器（Decompiler Parameter ID）是 Ghidra 提供的众多分析器中的一个，在大多数情况下默认关闭（仅对小于 2MB 的 Windows PE 文件启用）。当这个分析器启用时，Ghidra 使用反编译器派生的参数信息来命名反汇编清单中的函数参数，demo_stackframe 的例子如下所示：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_97_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D68d84ba477d9d8499ac0893d40142d6eb6dd3be1)

注意，param_3 不再出现在函数参数列表中，因为反编译器已经确定它在函数中没有以任何有意义的方式被使用。这个特殊的栈帧会在第 8 章中讲解。如果你希望 Ghidra 在完成默认的自动分析之后，执行反编译器参数 ID 分析，可以选择 Analysis→One Shot→Decompiler Parameter ID。

#### 6.3.4 局部变量作为操作数

让我们将注意力转移到如下清单的反汇编代码部分：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_97_2.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D1355c3b212e77a6d27d443e5e2e2fe276b6922c1)

该函数在基于 EBP 的栈帧上使用了通用函数序言❶。编译器在栈帧中分配了 88 字节（0x58 等于 88）的局部变量空间❷。这比估计的 76 字节略多一点，说明编译器偶尔会用额外的字节来填充局部变量空间，以便在栈帧中保持特定的内存对齐。

Ghidra 的反汇编清单和我们之前分析的栈帧有一个重要区别，就是在反汇编清单中没有类似于 [EBP-12] 的内存引用（例如，你可能会在 objdump 中看到）。相反，Ghidra 将所有的常数偏移量替换成了栈帧中的符号名及它们与函数初始栈指针的相对偏移量。这也说明了 Ghidra 的目标是生成更高层次的反汇编代码。与处理常数相比，处理符号化的名称更加容易，我们还可以修改变量名，以便理解变量的用途。尽管如此，Ghidra 还是在 CodeBrowser 窗口的右下角显示了当前指令不带任何标签的原始形式，以供参考。

在这个例子中，由于我们有源代码可供比较，所以可以利用反汇编中的各种线索，将 Ghidra 生成的变量名修改回源代码中对应的名称。

（1）demo_stackframe 接受三个参数 i、j 和 k，分别对应于变量 param_1、param_2 和 param_3。

（2）局部变量 x（local_10）初始化为参数 k（param_3）。❸

（3）同样，局部变量 y（local_14）初始化为参数 j（param_2）。❹

（4）局部变量 z（local_18）初始化为常数 10。❺

（5）64 字节的字符数组中的第一个字符 buffer[0]（local_58）初始化为 A（ASCII 0x41）。❻

（6）调用 helper 的两个参数被放到栈上❼。在此之前，8 字节的栈调整与两个 PUSH 相结合，让栈发生了 16 字节的变化。因此，程序维护了早期 16 字节的栈对齐。

#### 6.3.5 Ghidra 栈编辑器

除了栈摘要视图，Ghidra 还提供了一个详细的栈编辑器，其中记录了分配给栈帧的每个字节。当你在 Ghidra 的摘要视图中选择了一个函数或栈变量时，可以右击并从弹出的快捷菜单中选择 Function→Edit Stack Frame 来打开栈编辑器。demo_stackframe 函数的编辑器窗口如图 6-10 所示。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_98_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Dc5043ced266d13d7e20697fe9a348355486fa928)

图 6-10：栈编辑器示例

由于详细视图包含了栈帧中的每个字节，所以它会比摘要视图占用更多的空间。图 6-10 显示的部分有 29 字节，但也只是整个栈帧的一小部分。与前面的清单相同，local_10❸、local_14❹和 local_18❺在反汇编清单中被直接引用，它们的内容是以 dword（4 字节）写入来初始化的。基于 32 位数据操作的事实，Ghidra 能够推断出这些变量大小都是 4 字节，因此将每个变量标记为 undefined4 （未知类型的 4 字节变量）。

通过栈编辑器，我们可以编辑栈帧字段、更改显示格式，以及添加有用的补充信息。例如，可以为保存在 0x0 处的返回地址添加一个名称。

**基于寄存器的参数**

ARM 调用的约定使用多达 4 个寄存器来向函数传递参数，而不使用堆栈。一些 x86-64 调用的约定使用多达 6 个寄存器，一些 MIPS 调用的约定使用多达 8 个寄存器。基于寄存器的参数比基于堆栈的参数更难识别。

考虑以下两个汇编语言片段：

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_99_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Deb45bba80ea6a333ab2d8b8301fa05b9bba0711d)

在第一个函数中，保存的返回地址下方的栈区域被访问❶，可以得出结论，该函数至少需要一个参数。与大多数高级反汇编器一样，Ghidra 通过对栈指针和帧指针进行分析，来识别访问函数栈帧成员的指令。

在第二个函数中，RDI 在初始化之前就被使用❷。唯一符合逻辑的结论是，RDI 一定是在调用者中被初始化了，在这种情况下，RDI 用于将信息从调用者传递给 regargs 函数（即它是一个参数）。在程序分析术语中，RDI 在进入 regargs 时是活（live）的。为了确定函数期望从寄存器中获得参数的数量，可以观察函数内寄存器被写入（初始化）之前，它们的内容是否被读取和使用，由此来判断函数内所有活的寄存器。

不幸的是，这种数据流分析通常超出了大多数反汇编程序的能力，包括 Ghidra。另一方面，反编译器必须执行这种类型的分析，并且通常可以很好地识别出基于寄存器的参数的使用情况。Ghidra 的反编译器参数 ID 分析器（Edit→Options for<prog>→Properties→Analyzers）可以根据反编译器执行的参数分析来更新反汇编清单。

栈编辑器视图提供了关于编译器内部工作的详细信息。在图 6-10 中，可以很明显地看到，编译器在保存的帧指针 - 0x4 和局部变量 x（local_10）之间插入了 8 个额外的字节。这些字节占据了栈帧中从 - 0x5 到 - 0xc 的位置。除非你自己是一位编译器专家，或者愿意深入研究 GNU gcc 的源代码，否则你能做的就是猜测为什么这些额外字节是以这种方式分配的。在大多数情况下，这些额外字节可以归结于填充对齐，并且它们的存在不会影响程序的行为。在第 8 章中，我们会回顾栈编辑器视图，并讲解如何使用它处理更复杂的数据类型（如数组和结构体）。

### 6.4 搜索

如本章开头所述，Ghidra 让你在反汇编代码中导航、定位和发现新内容变得更容易。它还设计了许多数据显示来归纳特定类型的信息（名称、字符串、导入表等），让它们可以很容易地被找到。然而，要想有效地分析反汇编清单，往往需要寻找新的线索来提供更多信息。为此，Ghidra 提供了一个搜索菜单，用于搜索和定位我们感兴趣的数据。默认搜索菜单如图 6-11 所示。在本节中，我们将研究如何使用 CodeBrowser 提供的文本和字节搜索功能来探索反汇编清单。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_100_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Dddfa4d514f8ee10388666601b0ee798f11b25bb6)

图 6-11：Ghidra 搜索菜单项

#### 6.4.1 搜索程序文本

Ghidra 文本搜索等同于通过反汇编清单视图进行子字符串搜索。通过选择 Search→Program Text 可以打开文本搜索对话框，如图 6-12 所示。有两种搜索类型可供使用：Listing Display 搜索 CodeBrowser 窗口中可以看到的内容，而 Program Database 搜索整个程序数据库（超出 CodeBrowser 窗口）。除了搜索类型，还有几个选项可用于指定搜索的方式和内容。

要想在匹配项之间进行导航，可以使用文本搜索对话框中的 Next 和 Previous 按钮，或者选择 Search All 在新窗口中打开搜索结果，从而轻松导航到任意匹配项。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_101_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3Dbacd97e5a884912acd48e198d185b7cc9dd23e84)

图 6-12：文本搜索对话框

**修改窗口名称**

搜索窗口是 Ghidra 中可以随意修改名称的窗口类型之一，这有助于你在工作过程中跟踪搜索窗口。只需右击标题栏，并输入一个有意义的名字即可修改窗口名称。一个小技巧是将搜索字符串和助记符同时包含进去，从而帮助记忆。

#### 6.4.2 搜索内存

如果想搜索特定的二进制内容，例如已知的字节序列，那么文本搜索就不太管用了。此时，需要使用 Ghidra 的内存搜索功能。通过选择 Search→Memory 或热键 S 可以打开内存搜索对话框，如图 6-13 所示。如果要搜索一个十六进制字符序列，搜索字符串应该指定为一个以空格分隔的列表，包含两位数字的十六进制数，且不区分大小写，例如 c9 c3，如图 6-13 所示。如果序列中包含不确定的字符，可以使用通配符 * 或？。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_101_2.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D61ddedef931312fc7ae724232fc388420c13cce9)

图 6-13：内存搜索对话框

使用 Search All 选项搜索字节序列 c9 c3 的结果如图 6-14 所示。可以对任意列进行排序、修改窗口名称或者进行筛选。该窗口还提供了一些右键选项，包括删除行和操作匹配项等。

![](https://epubserver-1252317822.cos.ap-shanghai.myqcloud.com/C9D5D1/25638894401717006/epubprivate/OEBPS/Images/44551_102_1.jpg?sign=q-sign-algorithm%3Dsha1%26q-ak%3DAKIDulTzKUTIxRj5HF2X2TkyFPOJJg7ykMmz%26q-sign-time%3D1693539136%3B1693546336%26q-key-time%3D1693539136%3B1693546336%26q-header-list%3D%26q-url-param-list%3D%26q-signature%3D2a8f763b77439827697a27db518e043349aff134)

图 6-14：内存搜索结果

搜索值可以是字符串、十进制数、二进制数和正则表达式等。字符串、十进制数和二进制数都提供了各自上下文的格式选项。正则表达式用于搜索特定的模式，但因为在处理上有些限制，所以只能向前搜索。Ghidra 使用 Java 内置的正则表达式语法，详细内容可以查看 Ghidra 帮助文档。

### 6.5 小结

本章的目的是为你提供最基本的技能，以便有效地理解 Ghidra 反汇编清单，并围绕它进行导航。你与 Ghidra 之间的所有交互方式，目前已经讲解了绝大部分。然而，执行基本的导航、理解重要的反汇编结构（如堆栈）和搜索反汇编代码的能力，对于逆向工程师来说只是冰山一角。

掌握了这些技能之后，下一步就是学习如何使用 Ghidra 来满足特殊需求。在下一章中，我们将开始研究如何对反汇编清单进行最基本的修改，以便在理解二进制文件内容和行为的基础上，获得新知识。

[[1]](https://book.qq.com/book-read/Text/Section0011.xhtml#n9E673B56CD104734887A7D992A99BB11s) 当一个函数直接或间接调用自身时，就会产生递归。每次函数递归调用自身时，都会创建一个新的栈帧。如果没有明确定义停止情况（或者在合理数量的递归调用中没有触发停止情况），不受控制的递归会消耗掉所有可用的堆栈空间并使程序崩溃。

[[2]](https://book.qq.com/book-read/Text/Section0011.xhtml#n6CC3CC9ACA984CBCA156A9FB49BFC43Bs) 栈帧指针是一个指向栈帧内部某个地址的寄存器。通过栈帧内变量与栈帧指针的相对位置偏移，来引用这些变量。

[[3]](https://book.qq.com/book-read/Text/Section0011.xhtml#nD0AD066DE6FF4C4EA9EDBD28800A6708s) C++ 类可以定义两种类型的成员函数：静态成员和非静态成员。非静态成员函数用于操作特定对象的属性，因此必须有一些方法来知道它们到底在操作什么对象（this 指针）。静态程序颜色属于整个类，用于操作该类所有实例中的共享属性，它们不需要（也不接受）this 指针。

[[4]](https://book.qq.com/book-read/Text/Section0011.xhtml#n41CB00586E0B470CAFFB48EF13C6BFB8s) 参见链接 6-16。