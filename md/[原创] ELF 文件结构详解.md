> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-255670.htm)

> [原创] ELF 文件结构详解

博客：www.wireghost.cn

ELF 文件结构详解
==========

链接与装载视图
-------

链接与装载视图
-------

Elf 文件有 2 个平行视角：一个是程序链接角度，一个是程序装载角度。从链接的角度来看，Elf 文件是按 “Section”（节）的形式存储；而在装载的角度上，Elf 文件又可以按 “Segment”（段）来划分。实际上，Section 和 Segment 难以从中文的翻译上加以区分。因为很多时候 Section 也被翻译成段，比如 Section Header Table，有的资料叫段表、有的称为节区。后面在讲解时，就不对其加以区分。。  
![](https://bbs.pediy.com/upload/attach/201911/540885_3ACMWPTB4ZNCF2F.png)  
![](https://bbs.pediy.com/upload/attach/201911/540885_22DF75KJ7RUXEC6.png)

关于动态链接与静态链接
-----------

链接分为 2 种方式：一种是静态链接、一种是动态链接。  
静态链接是在编译链接时直接将需要执行的代码拷贝到调用处；动态链接则是在编译的时候不直接拷贝可执行代码，而是通过记录一系列符号和参数，在程序运行或加载时将这些信息传递给操作系统，由系统负责将所需的动态库加载到内存，然后当程序运行到指定的代码时，去共享执行内存中已经加载的动态库可执行代码，最终达到运行时链接的目的。  
程序是静态链接还是动态链接，由编译器的链接参数指定。具体来说，在 Android 上用 C++ 进行 ndk 编程时，可以通过设置 Application.mk 的相关内容，将相应的运行库作为动态库或静态库。

编写例子 so
-------

为了方便自己学习和记忆，编写了一个例子 so。这里我仅贴了部分代码，用于后面的分析与测试。  
![](https://bbs.pediy.com/upload/attach/201911/540885_49B5UJMWSGN38AP.png)

目标文件中的数据类型
----------

在介绍 Elf 文件格式前，先看看 Elf 文件中用到的数据类型：

<table><thead><tr><th>Name</th><th>Size</th></tr></thead><tbody><tr><td>Elf32_Addr</td><td>4</td></tr><tr><td>Elf32_Half</td><td>2</td></tr><tr><td>Elf32_Off</td><td>4</td></tr><tr><td>Elf32_Sword</td><td>4</td></tr><tr><td>Elf32_Word</td><td>4</td></tr><tr><td>unsigned char</td><td>1</td></tr></tbody></table>

Elf 文件头
-------

Elf 文件头描述了整个文件基本属性，如段表偏移、程序头部偏移等重要信息。它的定义如下：  
![](https://bbs.pediy.com/upload/attach/201911/540885_JF8P55PXXQ9BRUW.png)  
![](https://bbs.pediy.com/upload/attach/201911/540885_BTRT49TSXD6AEV7.png)  
![](https://bbs.pediy.com/upload/attach/201911/540885_V3ZTWSV5VR9GARA.png)

[](#节区头部（段表）)节区头部（段表）
---------------------

前面有提过，Elf 文件链接时是以 Section 的形式进行存储。其中，节区头部（段表）就是保存这些 Section 基本属性的结构。它描述了 Elf 中各个节的信息，比如每个节的名称、长度、在文件中的偏移、读写权限及其他属性，是一个以 Elf32_Shdr 结构体为元素的数组，而这个结构体又被称为段描述符。  
因为 sh_name 是在段表字符串表中的索引，所以实际在解析时需要先定位到. shstrtab 表，该表是专门用来存放 Section 名称的字符串表。而它对应的描述符在段表数组中的下标，则在 Elf 文件头中有给出，通常都是最后一个下标。在拿到节区名称后，再通过 sh_offset、sh_size 确定每一个节区在文件中的位置与长度。  
最后，用 readelf 命令来查看下目标文件中的段，对照相应输出来分析确认段表结构 (PS：段表数组中，第一个元素总是无效的描述符，全部为 0)  
![](https://bbs.pediy.com/upload/attach/201911/540885_CBDZHUVMTHFDTGE.png)  
![](https://bbs.pediy.com/upload/attach/201911/540885_4STNUWGHASUPCVS.png)

代码段
---

.text 代码段中保存程序指令，具体可以去查看 arm、thumb 指令集的 opcode。。  
![](https://bbs.pediy.com/upload/attach/201911/540885_JXKA2E93UN3P24R.png)  
![](https://bbs.pediy.com/upload/attach/201911/540885_XKRUA68SPE4GE8J.png)

数据段和只读数据段
---------

".rodata" 段存放的是只读数据，一般是程序里的只读变量（如 const 修饰的变量）和字符串常量；".data" 段保存的是已经初始化的全局静态变量和局部静态变量。  
PS：有时候编译器会把字符串常量放到 ".data" 段，而不是单独放到 ".rodata" 只读数据段。

BSS 段
-----

.bss 段中存放的是未初始化的全局变量和局部静态变量，这个 Section 在 Elf 文件中没有被分配空间。。

自定义 section
-----------

在声明一个函数或变量时，可以加上 **attribute**((section("自定义 section 名"))) 前缀的方式，将其添加到自定义段。

字符串表
----

Elf 文件中用到的字符串，如段名、函数名、变量名称等，均保存在字符串表中。其中，shstrtab 段表字符串表仅用来保存段名，而 strtab 或 dynstr section 则是存放普通字符串，如函数、变量名等符号名称，字符串之间以 "00" 截断。。  
![](https://bbs.pediy.com/upload/attach/201911/540885_QWSRUD8VVZFT9JA.png)

符号表
---

在链接过程中，函数和变量统称为符号，函数名或变量名就是符号名。符号表的段名为 symtab 或 dynsym，它是一个 Elf32_Sym 结构的数组。  
![](https://bbs.pediy.com/upload/attach/201911/540885_REP4N2BHTCDZZK3.png)  
使用 readelf 命令来查看目标文件的符号信息：  
![](https://bbs.pediy.com/upload/attach/201911/540885_B6CR4HU5MTC55YX.png)  
从以上输出可以看到，第一个元素即下标为 0 的符号，总是一个未定义的符号。  
其中 st_size 符号大小，它的取值有以下几种情况：对于数据类型的符号，它的值为该数据类型的大小；对于函数类型的符号，它的值为该方法的长度；如果该值为 0，表示该符号大小为 0 或未知。  
一般情况下，st_value 符号值，为相应符号的偏移（“COMMON” 块除外，表示该符号的对齐属性）。如本地定义的 JNI_OnLoad 方法，这个符号的值为 0xdcd，实际上这个函数的地址就是 0xdcc（因为指令集的切换加 1）。  
此外，让我们注意下红框中的几个符号。其中 printf、__android_log_print 这 2 个符号因为定义在其他库文件中，所以对应的符号值和大小都是 0；而全局变量 a 和声明的全局函数指针 global_printf 的符号值，分别为 0x4010、0x4014，都已经超过了文件长度，那么这些值实际上是在表示什么呢？通过动态调试，我们得知它其实是程序在内存中的虚拟地址。  
![](https://bbs.pediy.com/upload/attach/201911/540885_CTKKU4JMJBP4X4W.png)  
![](https://bbs.pediy.com/upload/attach/201911/540885_482RGWRKPN9WZ5K.png)  
这里再简单说明下，在链接过程中，链接器并不关心模块内部的非导出符号，如 start 这个函数。它是通过本地注册的方式声明的，实际寻址时可以通过 registerNatives 找到该方法，像这种函数会被编译器优化掉，变成偏移。  
![](https://bbs.pediy.com/upload/attach/201911/540885_76W8EUUPUG57RYR.png)  
PS：对符号表的理解，是 elf hook 的基础（导出表、导入表 Hook）

程序解释器
-----

".interp" 段用于指定解释器路径，里面保存的就是一个字符串，Android 下固定为 "/system/bin/linker"  
![](https://bbs.pediy.com/upload/attach/201911/540885_25NXH5AFJRGMP7H.png)

[](#全局偏移表（got）)全局偏移表（GOT）
-------------------------

在位置无关代码中，一般不能包含绝对虚拟地址。当在程序中引用某个共享库中的符号时，编译链接阶段并不知道这个符号的具体位置，如上面的__android_log_print，只有等到动态链接器将所需要的共享库加载到内存后，也就是运行阶段，符号的地址才会最终确定。因此，需要有一个数据结构来保存符号的绝对地址，这便是 GOT 表。这样，程序就可以通过引用 GOT 来获得某个符号的地址。  
在 Linux 下，GOT 被拆分成 ".got" 和 ".got.plt"2 个表。其中 ".got" 用来保存全局变量引用的地址，".got.plt" 用来保存函数引用的地址。此外，".got.plt" 的前三项保留，用于存放特殊的数据结构地址：第一项保存的是 ".dynamic" 动态节区的地址；第二项保存的是本模块 ID，指向已经加载的共享库的链表地址（前面提到加载的共享库会形成一个链表）；第三项保存的是_dl_runtime_ resolve 函数的地址（用于查找指定模块下的特定方法）.  
而在 Android 平台，got 表并没有细分 ".got"、".got.plt"，但仔细观察可以发现，它其实有通过_GLOBAL_OFFSET_TABLE_ 来区分上下两个结构。。  
![](https://bbs.pediy.com/upload/attach/201911/540885_NCYWBB5NNM6X7C2.png)

[](#过程链接表（plt）)过程链接表（PLT）
-------------------------

在支持懒绑定的情况下，当发生对外部函数的调用时，程序会通过 PLT 表将控制交给动态链接器，后者解析出函数的绝对地址，修改 GOT 中相应的值，之后的调用将不再需要链接器的绑定。Android 虽然内核基于 Linux，但其动态链接机制却不是 ld.so 而是自带的 linker。由于 linker 是不支持懒绑定的，所以在进程初始化时，动态链接器首先解析出外部过程引用的绝对地址，一次性的修改所有相应的 GOT 表项。  
基于上文的说明，再来简单分析下 Android 平台中 Elf 文件的 PLT 过程链接表。可以发现，plt 其实也是代码段，除 PLT[0] 外，其它所有 PLT 项的形式都一样，且包括 PLT[0] 在内的每个表项都占 16 个字节，所以整个 PLT 就像个数组。其中，PLT[0] 内容固定，跳转到 GOT[2] 即_dl_runtime_ resolve 函数，查找特定模块下的指定方法，并填充到 GOT 表。而其他 PLT 普通表项则相当于一个函数的桩函数（stub），通过引用 GOT 表中函数的绝对地址，来把控制转移到实际的函数。  
PS：这一部分知识可以用来实现 GOT、PLT 表 hook，即导入表 hook。。  
![](https://bbs.pediy.com/upload/attach/201911/540885_NRNCNE9XG3ST2X4.png)  
![](https://bbs.pediy.com/upload/attach/201911/540885_X8QN739ZEEYFDT3.png)  
![](https://bbs.pediy.com/upload/attach/201911/540885_76D4PVJFZ5X942N.png)

重定位表
----

在前面介绍符号表、got 表、plt 表时，其实就已经涉及到了重定位。重定位是将符号引用与符号定义进行链接的过程。例如，当程序调用了一个函数时，相关的调用指令必须把控制传输到适当的目标执行地址。  
在 Elf 文件中，以 ".rel" 或 ".rela" 开头的 section 就是一个重定位段。它是一个 Elf32_Rel 结构数组，每个元素对应一个重定位入口。  
本例中的重定位表是 ".rel.dyn" 和 ".rel.plt"，它们分别相当于静态链接中的 ".rel.data" 和 ".rel.text"。".rel.dyn" 实际上是对数据引用的修正，它所修正的位置相当于 ".got" 以及数据段；而 ".rel.plt" 则是对函数引用的修正，所修正的位置位于 ".got.plt"。然后，使用 "readelf -r" 命令，查看重定位表，并依此进行对比分析。。  
![](https://bbs.pediy.com/upload/attach/201911/540885_AWDP8XAPHMQR75G.png)  
![](https://bbs.pediy.com/upload/attach/201911/540885_NM5DHZRWQ2RGNPF.png)  
接下来，结合代码看看 Android 系统的 Linker 是如何实现重定位的。将例子 so 拖到 Ida 中，查找到对应 start 方法的偏移函数。然后，将几个重要的地址先找出来，分别是：  
![](https://bbs.pediy.com/upload/attach/201911/540885_YXJG6UUNWRXNMED.png)  
![](https://bbs.pediy.com/upload/attach/201911/540885_BP65NRH66EYHK7T.png)

### 全局函数指针调用外部函数

global_printf 方法是我们声明的指向 printf 函数的全局指针，调用 global_printf 方法时，R3 的值是 * global_printf，而 global_printf 的值 0x4010 刚好在. rel.dyn 中的 R_ARM_ABS32 的重定位项，因此可以得出结论：通过全局函数指针的方式调用外部函数，它的重定位类型是 R_ARM_ABS32，并且位于. rel.dyn 节区。  
继续分析 global_printf 的调用流程的调用流程，首先定位到 global_printf_ptr（0x3FD0），该地址位于. got 节区，GLOBAL_OFFSET_TABLE 的上方。然后再通过 global_printf_ptr 定位到 0x4010（位于. data 节区），最后再通过 0x4010 定位到最终的函数地址，因此 R_ARM_ABS32 重定位项的 Offset 指向最终调用函数地址的地址（也就是函数指针的指针），整个重定位过程是先位到. got，再从. got 定位到. date。下面是. got 段区的 16 进制表示：  
![](https://bbs.pediy.com/upload/attach/201911/540885_R3K2AA7YZPB7TJ7.png)  
结果发现 0x4028 地址中的数据全为 0，当动态链接时，linker 会覆盖 0x00004010 地址的值，指向 printf 的真正地址（而不是现在的 0x00004028）

### 直接调用外部函数

再来看下直接调用 printf 函数时的情况，对应 0xD16 的 BLX 指令，它会跳转. plt 节。最后，PC 指向 * printf_ptr，其中 printf_ptr 的地址为 0x3FE8，位于. got.plt 节区，而 0x3FE8 地址值的正好是前面有提到的 0x4028，于是可以得出结论：直接调用外部函数，它的重定位类型是 R_ARM_JUMP_SLOT，位于. re.plt 节区，其 Offset 指向最终调用函数地址的地址（也就是函数指针的指针）。整个过程是先到. plt，再到. got，最后才定位到真正的函数地址。。  
![](https://bbs.pediy.com/upload/attach/201911/540885_SEP8TS7UTRY4DDX.png)

[](#动态节区（dynamic）)动态节区（dynamic）
-------------------------------

Dynamic 段是动态链接中 Elf 最重要的一个 section，这里面保存了动态链接器所需的基本信息，如依赖于哪些共享对象、动态链接符号表的位置、共享对象初始化代码的地址等。  
动态节区是一个数组，每个元素都是 Elf32_dyn 结构体。它的定义如下所示，由一个类型值加上一个附加的数值或指针，对于不同的类型，后面附加的数值或指针有着不同含义。。  
![](https://bbs.pediy.com/upload/attach/201911/540885_VP8RDSZBACD9HS5.png)  
![](https://bbs.pediy.com/upload/attach/201911/540885_3JRB2FXGCCA3TDZ.png)

程序头表（Program Header Table）
--------------------------

前面已经就链接视图将重要的一些 Section 做了详尽解析，这里再从装载角度介绍下程序头表，它是一个以 Elf32_Phdr 结构体为元素的数组。  
然后，再来简单介绍下 Segment 这个概念。因为程序在加载的过程中，是根据权限映射到内存空间的，而一个 Segment 可以包含一个或多个属性类似的 Section。  
其中，p_memsz 的值不可以小于 p_filesz，否则就是不符合常理的。如果 p_memsz 大于 p_filesz，就表示该 Segment 在内存中所分配的空间超过在文件中的实际大小，多余的部分全部填充 0，如 BSS 段。。  
使用 readelf 命令查看程序头表，进行对比分析。其中第一项 Program Header，用来描述程序头表自身的位置和大小；第二项和第三项为 Load Segment，只有这部分会被加载到内存，并因为权限属性的不同，被划分成了多个部分（具体到这个 so 则是 2 块）。它包括代码段、数据段等；第三项是 Dynamic Segment，它提供了 Dynamic 动态节区的偏移和大小，并通过寻址到 Dynamic 节进而获取动态链接器所需的基本信息。  
![](https://bbs.pediy.com/upload/attach/201911/540885_FNNN83QSY7NH6AR.png)  
![](https://bbs.pediy.com/upload/attach/201911/540885_QERW5H7A7UGJSGR.png)  
使用 “cat /proc/pid/maps” 命令，查看 libtest.so 在内存中的映射，发现它被分成了 3 个子空间，继续看第二列。其中，r 表示只读，w 表示可写，x 表示可执行，p 表示私有（s 表示共享）。这一部分基本对应前面的程序头表，至于为什么会多出一块只读部分，个人的理解是程序头表只是根据权限将属性相近的段划到一个 Segment，加载的时候还是按照权限进行映射的。最明显的就是，rodata 只读数据段和代码段放到了一个 Segment，但代码段是可读可执行的。  
![](https://bbs.pediy.com/upload/attach/201911/540885_93E99PCM4XXYMXR.png)  
最后，再来简单的说下页对齐。在进行内存映射时，实际是以一个 “页（Page）” 为单位进行映射的，而在 Android 平台下，页的单位为 0x1000（4096）字节。这里再来看下上图中的第一行信息，libtest.so 映射到内存空间中的起始地址为 0x403ec000，这一块地址空间的权限为可读可执行，在程序头表中相应的 Load 段的大小为 0x2444，二者相加为 0x403EE444。但是，它的结束地址却不是 0x403EE444，而是 0x403ef000，多出的那一部分字节正是为了做页对齐。。

总结
--

至此，Elf 文件结构的学习和总结告一段落。这部分知识，可以应用在 Elf hook 和 Elf 的加固上，其中 Elf hook 已经有了一个简单的认识，后续有时间我会进行相关说明并整理相应的代码实现。。

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)