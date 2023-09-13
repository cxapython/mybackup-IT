> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [cloud.tencent.com](https://cloud.tencent.com/developer/article/1145344?areaId=106001)

> ELF 文件装载链接过程及 hook 原理 ELF 文件格式解析 可执行和可链接格式 (Executable and Linkable Format，缩写为 ELF)，常被称为 ELF 格式，在计算机科学中，是一种用于执行档、目的档、共享库和核心转储的标准文件格式。

ELF 文件格式解析
----------

可执行和可链接格式 (Executable and Linkable Format，缩写为 ELF)，常被称为 ELF 格式，在计算机科学中，是一种用于执行档、目的档、共享库和核心转储的标准文件格式。

ELF 文件主要有四种类型：

*   可重定位文件（Relocatable File） 包含适合于与其他目标文件链接来创建可执行文件或者共享目标文件的代码和数据。
*   可执行文件（Executable File） 包含适合于执行的一个程序，此文件规定了 exec() 如何创建一个程序的进程映像。
*   共享目标文件（Shared Object File） 包含可在两种上下文中链接的代码和数据。首先链接编辑器可以将它和其它可重定位文件和共享目标文件一起处理，生成另外一个目标文件。其次，动态链接器（Dynamic Linker）可能将它与某个可执行文件以及其它共享目标一起组合，创建进程映像。

以一个简单的目标文件为例：

```
#include <stdio.h>
int global_init_var = 84;
int global_uninit_var;
void func1(int i)
{
    printf("%d\n",i);
}
int main(void)
{
    char *str = "hello";
    static int static_var = 85;
    static int static_var2;
    int a = 1;
    int b;
    func1(static_var + static_var2 + a + b);
    return 0;
}

```

### ELF 文件结构

![](https://ask.qcloudimg.com/http-save/yehe-1878670/wkhwss59py.jpeg)

ELF 文件结构

### 链接视图和执行视图

ELF 文件在磁盘中和被加载到内存中并不是完全一样的，ELF 文件提供了两种视图来反映这两种情况: 链接视图和执行视图。顾名思义，链接视图就是在链接时用到的视图，而执行视图则是在执行时用到的视图。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/6usgw6prrw.jpeg)

链接视图和执行视图

程序头部表(Program Header Table)，如果存在的话，告诉系统如何创建进程映像。 节区头部表 (Section Header Table) 包含了描述文件节区的信息，比如大小，偏移等。

### ELF 文件头（ELF Header）

定义了 ELF 魔数、硬件平台等、 入口地址、程序头入口和长度、 段表的位置和长度及段的数量、 段表字符串表（.shstrtab）所在的段在段表中的下标。

可以在”/usr/include/elf.h” 中找到它的定义（Elf32_Ehdr）。 ELF 各个字段的说明:

![](https://ask.qcloudimg.com/http-save/yehe-1878670/yrxesg5bmg.jpeg)

### 段表 (Section Header Table)

描述了各个段的信息，比如每个段的段名、段的长度、在文件中的偏移、读写权限及段的其它属性。 段表的结构是一个以 Elf32_Shdr 结构体（段描述符）为元素的数组。 每个 Elf32_Shdr 结构体对应一个段。 使用 readelf 工具查看 ELF 文件的段:

![](https://ask.qcloudimg.com/http-save/yehe-1878670/s428v3xapk.jpeg)

**段描述符（Elf32_Shdr）**的各个成员及含义：

![](https://ask.qcloudimg.com/http-save/yehe-1878670/sqc3bk15tt.jpeg)

**段的类型 (sh_type)** 对于编译器和链接器，主要决定段的属性的是段的类型 (sh_type) 和段的标志位(shflags)。段的类型相关常量以 SHT 开头，列举如下表。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/2qd3jhdjau.jpeg)

**段的标志位 (sh_flag)** 表示该节在进程虚拟地址空间中的属性，比如是否可写，是否可执行等。相关常量以 SHF_开头，如下表：

![](https://ask.qcloudimg.com/http-save/yehe-1878670/nf2xfp24oy.jpeg)

**段的链接信息 (sh_link、sh_info)** 如果节的类型是和链接相关的，比如重定位表、符号表等，那么 sh_link 和 sh_info 两个成员包含的意义如下。对于其他段，这两个成员没有意义。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/epa4sxggtq.jpeg)

### 代码段 (.text)

使用 objdump 工具查看代码段的内容，”-d” 参数将所有包含指令的段反汇编。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/f6ugl2jp16.jpeg)

### 数据段 (.data) 和只读数据段(.rodata)

.data 段保存的是那些已经初始化了的全局静态变量和局部静态变量。前面 SimpleSection.c 代码里面一共有两个这样的变量，都是 int 类型的，一共刚好 8 字节。 在 SimpleSection.c 里在调用”printf” 的时候，用到了一个字符串常量”%d\n”, 它是一种只读数据，所以被放到了”.rodata” 段。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/d092psvdql.jpeg)

### BSS 段 (.bss)

.bss 段存放的未初始化的全局变量和局部静态变量。.bss 段不占磁盘空间。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/dheivp9ity.jpeg)

### 字符串表（.strtab）

在 ELF 文件中，会用到很多字符串，比如节名，变量名等。所以 ELF 将所有的字符串集中放到一个表里，每一个字符串以’\0’分隔，然后使用字符串在表中的偏移来引用字符串。 比如下面这样：

![](https://ask.qcloudimg.com/http-save/yehe-1878670/e5ql6yetzs.jpeg)

那么偏移与他们对用的字符串如下表:

![](https://ask.qcloudimg.com/http-save/yehe-1878670/m9tt4kb5km.jpeg)

这样在 ELF 中引用字符串只需要给出一个数组下标即可。字符串表在 ELF 也以段的形式保存，常见的段名为”.strtab”或”.shstrtab”。这两个字符串表分别为字符串表 (String Table) 和段表字符串表(Header String Table)，字符串表保存的是普通的字符串，而段表字符串表用来保存段表中用到的字符串，比如段名。

### 符号表（.symtab）

在链接的过程中需要把多个不同的目标文件合并在一起，不同的目标文件相互之间会引用变量和函数。在链接过程中，我们将函数和变量统称为符号，函数名和变量名就是符号名。 每一个目标文件中都有一个相应的符号表 (System Table)，这个表里纪录了目标文件所用到的所有符号。每个定义的符号都有一个相应的值，叫做符号值 (Symbol Value)，对于变量和函数，符号值就是它们的地址。 符号表是一个 Elf32_Sym(32 位) 的数组，每个 Elf32_Sym 对应一个符号。这个数组的第一个元素，也就是下标为 0 的元素为无效的” 未定义” 符号。 他们的定义如下：

![](https://ask.qcloudimg.com/http-save/yehe-1878670/bnrkmjnxym.jpeg)

**符号类型和绑定信息 (st_info)** 该成员的低 4 位标识符号的类型 (Symbol Type)，高 28 位标识符号绑定信息 (Symbol Binding)，如下表所示。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/nbxw2szfgd.jpeg)

![](https://ask.qcloudimg.com/http-save/yehe-1878670/xrjj157ihw.jpeg)

**符号所在段 (st_shndx)** 如果符号定义在本目标文件中，那么这个成员表示符号所在段在段表中的下表，但是如果符号不是定义在本目标文件中，或者对于有些特殊符号，sh_shndx 的值有些特殊。如下：

![](https://ask.qcloudimg.com/http-save/yehe-1878670/ncw17oeg7v.jpeg)

**符号值 (st_value)** 每个符号都有一个对应的值。主要分下面几种情况：

*   如果符号不是”COMMON”类型的 (即 st_shndx 不为 SHN_COMMON)，则 st_value 表示该符号在段中的偏移，即符号所对应的函数或变量位于由 st_shndx 指定的段，偏移 st_value 的位置。比如 SimpleSection 中的”func1”，”main” 和”global_init_var”。
*   在目标文件中，如果符号是”COMMON” 类型 (即 st_shndx 为 SHN_COMMON)，则 st_value 表示该符号的对齐属性。比如 SimleSection 中的”global_uninit_var”。
*   在可执行文件中，st_value 表示符号的虚拟地址。

下图为使用 readelf 工具来查看 ELF 文件的符号:

![](https://ask.qcloudimg.com/http-save/yehe-1878670/bru1g41czi.jpeg)

比如，Num13 行指的是符号表中的第 13 个元素，符号名为 main，它是函数类型，定义在第一个段（即. text 段）的第 001b 偏移处，大小为 64 字节。

### 重定位表 (.rel.text)

SimpleSection.o 中有一个叫”.rel.text”的段，它的类型 (sh_type) 为”SHT_REL”，也就是说它是一个重定位表。链接器在处理目标文件时，需要对目标文件中的某些部位进行重定位，即代码段和数据中中那些绝对地址引用的位置。对于每个需要重定位的代码段或数据段，都会有一个相应的重定位表。比如”.rel.text”就是针对”.text”的重定位表，”.rel.data”就是针对”.data”的重定位表。

静态链接
----

这节以下面两个文件为例

```
extern int shared;
int main(){
    int a = 100;
    swap(&a,&shared);
}

```

```
int shared = 1;
void swap(int* a, int* b){
    *a ^= *b ^= *a ^= *b;
}

```

当我们有两个目标文件时，如何将他们链接起来形成一个可执行文件？ 对于多个输入目标文件，链接器如何将它们的各个段合并到输出文件？输出文件中的空间如何更配给输入文件？ 下图为现在链接器采用的空间分配策略。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/hzhiisg3ly.jpeg)

整个链接过程分两步：

*   第一步 空间与地址分配 扫描所有的输入目标文件，并且获得它们的各个段的长度、属性和位置，并且将输入目标文件中的符号表中所有的符号定义和符号引用收集起来，统一放到一个全局符号表中。
*   第二步 符号解析与重定位 使用第一步中收集到的信息，读取输入文件中段的数据、重定位信息，并且进行符号解析与重定位、调整代码中的地址等

使用 ld 链接器将”a.o” 和”b.o” 链接起来：

```
$ld a.o b.o -e main -o ab

```

查看链接前后各个段的属性

![](https://ask.qcloudimg.com/http-save/yehe-1878670/5dtv0fonz3.jpeg)

VMA 表示虚拟地址，LMA 表示加载地址，正常情况下这两个值应该一样。

整个链接过程前后，目标文件各段的分配、程序虚拟地址:

![](https://ask.qcloudimg.com/http-save/yehe-1878670/a3q2fn5pky.jpeg)

在 Linux 下，ELF 可执行未见默认从地址 0x08048000 开始分配。

### 符号解析与重定位

编译器在将”a.c” 编译成指令时，它如何访问”shared” 变量？如何调用”swap” 函数？ **重定位表 (Relocation Tabel)** 专门用来保存与重定位相关的信息，链接器根据它知道哪些指令时要被调整的，以及如何调整。 对于 32 位的 Intel x86 系列处理器来说，重定位表的结构是一个 Elf_32Rel 结构的数组，每个数组元素对应一个重定位入口。定义如下：

![](https://ask.qcloudimg.com/http-save/yehe-1878670/pf4f25xknv.jpeg)

可以使用 objdump 来查看目标文件的重定位表：

![](https://ask.qcloudimg.com/http-save/yehe-1878670/6s84z1m6ov.jpeg)

将”a.o”的代码段反汇编可以看到，此时编译器并不知道 “shared” 的地址，暂时把地址 0 看做”shared”的地址。 0xE8 是一条近址相对位移调用指令，后面 4 个字节就是被调用函数的相对于调用指令的下一条指令的偏移量。 此处”swap”函数的地址是 0x2b-4=0x27, 可以看出 0xfffffffc 也是一个临时地址。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/4ow33n71b7.jpeg)

**指令修正方式**

![](https://ask.qcloudimg.com/http-save/yehe-1878670/7nxx3w3wa8.jpeg)

指令修复的结果

![](https://ask.qcloudimg.com/http-save/yehe-1878670/ofm3c1nc9j.jpeg)

* * *

可执行文件的装载与进程
-----------

程序执行时所需要的指令和数据必需在内存中才能够正常运行。 页映射将内存和所有磁盘中的数据和指令按照 “页（Page）” 为单位划分成若干个页，以后所有的装载和操作的单位就是页。

进程的建立需要做下面三件事情：

*   创建一个独立的虚拟地址空间
*   读取可执行文件头，并且建立虚拟空间与可执行文件的映射关系。
*   将 CPU 的指令寄存器设置成可执行文件的入口地址，启动运行。

对于第 2 步，当操作系统捕获到缺页错误时，它应该知道程序当前所需的页在可执行文件中的哪一个位置。 这种映射关系是保存在操作系统内部的一个数据结构 **VMA**。 例如下图中，操作系统创建进程后，会在进程相应的数据结构中设置有一个. text 段的 VMA：它在虚拟空间中的地址为 0x08048000~0x08049000，它对应 ELF 文件中偏移为 0 的. text，它的属性为只读，还有一些其他的属性。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/ev8xvtdb4s.jpeg)

**页错误** 在上面的例子中，程序的入口地址为 0x08048000，当 CPU 开始打算执行这个地址的指令时，发现页面 0x08048000~0x08049000（虚拟地址）是个空页面，于是它就认为这是一个页错误。CPU 将控制权交给操作系统，操作系统将查询虚拟空间与可执行文件的映射关系表，找到空页面所在的 VMA，计算相应的页面在可执行文件中的偏移，然后在物理内存中分配一个物理页面，将进程中该虚拟页与分配的物理页之间建立映射关系，然后把控制权再还给进程，进程从刚才页错误的位置重新开始执行。

### 链接视图和执行视图

以下面的程序为例。

```
#include <stdlib.h>
int main()
{
    while(1)
    {
        sleep(1000);
    }
    return 0;
}

```

下面的 elf 文件被重新划分成了三个部分，有一些段被归入可读可执行的，他们被统一映射到一个 CODE VMA；另一部分段是可读可写的，它们被映射到了 DATA VMA；还有一些段在程序执行时没有用，所以不需要映射。 ELF 与 Linux 进程虚拟空间映射关系（一个常见进程的虚拟空间）如下图。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/t5u13lenmd.jpeg)

#### 程序头表 (Program Header Table)

用来保存 “Segment” 的信息, 描述了 ELF 文件该如何被操作系统映射到虚拟空间。因为 ELF 目标文件不需要被装载，所以它没有程序头表，而 ELF 的可执行文件和共享库文件都有。 使用 readelf 查看程序头表。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/ep7mmlfe3u.jpeg)

跟段表结构一样，程序头表也是一个结构体数组，其结构体用 Elf32_Phdr 表示。 下表是 Elf32_Phdr 结构的各个成员的基本含义。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/19xfv7svdg.jpeg)

### 堆和栈

VMA 除了被用来映射可执行文件中的各个”segment” 以外，操作系统通过使用 VMA 来对进程的地址空间进行管理，包括堆和栈。 在 Linux 下，可以通过查看”/proc” 来查看进程的虚拟空间分布：

![](https://ask.qcloudimg.com/http-save/yehe-1878670/fm5gffz76z.jpeg)

我们可以看到进程中有 5 个 VMA, 只有前两个是映射到可执行文件中的两个 Segment。另外三个段的文件所在设备主设备号及文件节点号都是 0，则表示他们没有映射到文件中，这种 VMA 叫做匿名虚拟内存区域。另外有一个很特殊的 VMA 叫 “vdso”，它的地址已经位于内核空间了（即大于 0xC0000000 的地址），事实上它是一个内核的模块，进程可以通过访问这个 VMA 来跟内核进行一些通信。 操作系统通过给进程空间划分出一个个 VMA 来管理进程的虚拟空间；基本原则是将相同权限属性的、有相同映像文件的映射成一个 VMA。

* * *

动态链接
----

以下面的代码为例

```
#ifndef LIB_H
#define LIB_H
void foobar(int i);
#endif


#include <stdio.h>
void foobar(int i){
    printf("Printing from Lib.so %d\n",i);
    sleep(-1);
}


#include "Lib.h"
int main(){
    foobar(1);
    return;
}


#include "Lib.h"
int main(){
    foobar(2);
    return;
}

```

将 Lib.c 编译成一个共享对象文件：

```
  $gcc -fPIC -shared -o Lib.so Lib.c

```

分别编译链接 Program1.c 和 Program2.c：

```
 $gcc -o Program1 Program1.c ./Lib.so

```

```
  $gcc -o Program2 Program2.c ./Lib.so

```

查看进程的虚拟地址空间分布：

![](https://ask.qcloudimg.com/http-save/yehe-1878670/5krc0f0obq.jpeg)

上图中的 ld-2.6.so 实际上是 Linux 下的动态链接器，它与普通共享对象一样被映射到了进程的地址空间，在系统开始运行 program1 之前，首先会把控制权交给动态链接器，由它完成所有的动态链接工作以后再把控制权交给 program1, 然后开始执行。

通过 readelf 查看 Lib.so 的装载属性：

![](https://ask.qcloudimg.com/http-save/yehe-1878670/wbi1991x56.jpeg)

与普通程序不同的是，动态链接模块的装载地址是从地址 0x00000000 开始的，这个地址是无效的，共享对象的最终装载地址在编译时时不确定的，而是在装载时，装载器根据当前地址空间的空前情况，动态分配一块足够大小的虚拟地址空间给相应的共享对象。

### 地址无关代码 (PIC)

装载时重定位是解决动态模块中有绝对地址引用的方法之一，但是它有一个很大的缺点是指令部分无法在多个进程之间共享，这样就失去了动态链接节省内存的一大优势。我们还需要有一种更好的方法解决共享对象指令中对绝对地址的重定位问题。其实我们的目的很简单，希望程序模块中共享的指令部分在装载时不需要因为装载地址的改变而改变，所以实现的基本思想就是把指令中那些需要被修改的部分分离出来，跟数据部分放在一起，这样指令部分就可以保持不变，而数据部分可以在每个进程中拥有一个副本。

模块中各种类型的地址引用方式如下图：

![](https://ask.qcloudimg.com/http-save/yehe-1878670/slkp9ox7q2.jpeg)

#### 全局偏移表 (GOT)

用于模块间数据访问，在数据段里建立一个指向外部模块变量的指针数组。当代码需要引用该全局变量时，可以通过 GOT 中相对用的项间接引用，它的基本机制如下图。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/ulgwommo8c.jpeg)

当指令中需要访问变量 b 时，程序会先找到 GOT，然后根据 GOT 中变量所对应的项找到变量的目标地址。每个变量都对应一个 4 字节的地址，链接器在装载模块的时候会查找每个变量所在的地址，然后填充 GOT 中的各个项，以确保每个指针所指向的地址正确。由于 GOT 本身是放在数据段的，所以它可以在模块装载时被修改，并且每个进程都可以有独立的副本，相互不受影响。

#### 延迟绑定 (PLT)

动态链接下对于全局和静态的数据访问都要进行复杂的 GOT 定位，然后间接寻址；对于模块间的调用也要先定位 GOT，然后再进行间接跳转。程序开始执行时，动态链接器都要进行一次链接工作，会寻找并装载所需的共享对象，然后进行符号查找地址重定位等工作，如此一来，程序的运行速度必定会减慢。

延迟绑定的实现 函数第一次被用到时才进行绑定（符号查找、重定位等），如果没有用到则不进行绑定。

GOT 位于 .got.plt section 中，而 PLT 位于 .plt section 中。这些都是数据段，不同进程拥有其副本。 GOT 保存了程序中所要调用的函数的地址，运行一开时其表项为空，但数组大小在编译时已经确定了，运行时会实时的更新表项。相当于链接器给动态加载器布置了填空题作业。 一个符号调用在第一次时会解析出绝对地址更新到 GOT 中，第二次调用时就直接找到 GOT 表项所存储的函数地址直接调用了。 `printf`函数的调用过程如下图

![](https://ask.qcloudimg.com/http-save/yehe-1878670/g3qwef0ke4.jpeg)

#### GDB 调试分析延迟绑定机制

为了加深理解可以用 GDB 动态调试，Examine 下断点前后 GOT 表的内存的变化。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/vlp75176wg.jpg)

![](https://ask.qcloudimg.com/http-save/yehe-1878670/52o7npewnx.jpg)

![](https://ask.qcloudimg.com/http-save/yehe-1878670/pijhhqabtk.jpg)

动态加载器解析结束，可以看到 got 表项正确指向了 libc 动态库中 printf 的地址

![](https://ask.qcloudimg.com/http-save/yehe-1878670/u9k87gr870.jpg)

### 动态链接的相关结构

#### .interp 段

在动态链接的 ELF 可执行文件中，有一个专门的段叫做”.interp” 段。里面保存的是一个字符串，记录所需动态链接器的路径。 从下图可以看出，Android 用的动态链接器是 linker

![](https://ask.qcloudimg.com/http-save/yehe-1878670/ed63k1b5ry.jpeg)

#### .dynamic 段

这个段里保存了动态链接器所需要的基本信息，比如依赖哪些共享对象、动态链接符号表的位置、动态链接重定位表的位置、共享对象初始化代码的地址等。 .dynamic 段里保存的信息有点像 ELF 文件头。 .dynamic 段的结构是由 Elf32_Dyn 组成的数组。 Elf32_Dyn 结构由一个类型值加上一个附加的数值或指针，对于不同的类型，后面附加的数值或者指针有着不同的含义。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/4rrufdp6wk.jpeg)

#### 动态符号表 (.dynsym)

为了表示动态链接模块之间的符号导入导出关系，ELF 专门有一个叫做动态符号表的段用来保存这些信息。 与”.symtab” 不同的是，”.dynsym” 只保存了与动态链接相关的符号，对于那些模块内部的符号，比如模块私有变量则不保存。很多时候动态链接模块同时拥有”.dynsym” 和”.symtab” 两个表，”.symtab” 中往往保存了所有符号，包括”.dynsym” 中的符号。

#### 动态符号字符串表 (.dynstr)

在动态链接时用于保存符号名的字符串表。

#### 符号哈希表 (.hash)

由于动态链接下，需要在程序运行时查找符号，为了加快符号的查找过程，往往还有辅助的符号好戏表。 用 readelf 查看 elf 文件的动态符号表及它的哈希表。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/s1y4d2sp4u.jpeg)

### 动态链接重定位表

在动态链接中，导入符号的地址在运行时才确定，所以需要在运行时将这些导入符号的引用修正，即需要重定位。

“.rel.dyn” 段对数据引用的修正，它所修正的位置位于”.got” 以及数据段； “.rel.plt” 段对函数引用修正，它所修正的位置位于”.got.plt”。 用 readelf 来查看一个动态链接的文件的重定位表：

![](https://ask.qcloudimg.com/http-save/yehe-1878670/dmsfdslzik.jpeg)

R_386_JUMP_SLOT 和 R_386_GLOB_DAT 这两个类型的重定位入口表示，被修正的位置只需要直接填入符号地址即可。 比如，printf 这个重定位入口，它的类型为 R_386_JUMP_SLOT，它的偏移为 0x000015d8，它位于”.got.plt” 中，下图为其结构。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/zgq0jdmzwy.jpeg)

当链接器需要进行重定位时，它先查找”printf” 的地址，“printf” 位于 libc.so 中。假设链接器在全局符号表里面找到”printf” 的地址为 0x08801234, 那么链接器就会将这个地址填入到”.got.plt” 中偏移为 0x000015d8 的位置中去，从而实现了地址的重定位。 R_386_GLOB_DAT 是对”.got” 的重定位，它跟 R_386_JUMP_SLOT 的做法一样。

* * *

android arm 架构的一种 hook 实现方案
---------------------------

具体实现来自 Andrey Petrov 的 [blog](https://cloud.tencent.com/developer/tools/blog-entry?target=http%3A%2F%2Fshadowwhowalks.blogspot.com%2F2013%2F01%2Fandroid-hacking-hooking-system.html).

```
 #include "linker.h"  
 static unsigned elfhash(const char *_name)  
 {  
   const unsigned char *name = (const unsigned char *) _name;  
   unsigned h = 0, g;  
   while(*name) {  
     h = (h << 4) + *name++;  
     g = h & 0xf0000000;  
     h ^= g;  
     h ^= g >> 24;  
   }  
   return h;  
 }  
 static Elf32_Sym *soinfo_elf_lookup(soinfo *si, unsigned hash, const char *name)  
 {  
   Elf32_Sym *s;  
   Elf32_Sym *symtab = si->symtab;  
   const char *strtab = si->strtab;  
   unsigned n;  
   n = hash % si->nbucket;  
   for(n = si->bucket[hash % si->nbucket]; n != 0; n = si->chain[n]){  
     s = symtab + n;  
     if(strcmp(strtab + s->st_name, name)) continue;  
       return s;  
     }  
   return NULL;  
 }  

 int hook_call(char *soname, char *symbol, unsigned newval) {  
  soinfo *si = NULL;  
  Elf32_Rel *rel = NULL;  
  Elf32_Sym *s = NULL;   
  unsigned int sym_offset = 0;  
  if (!soname || !symbol || !newval)  
     return 0;  
  si = (soinfo*) dlopen(soname, 0);  
  if (!si)  
   return 0;  
  s = soinfo_elf_lookup(si, elfhash(symbol), symbol);  
  if (!s)  
    return 0;  
  sym_offset = s - si->symtab;  
  rel = si->plt_rel;  
    
  for (int i = 0; i < si->plt_rel_count; i++, rel++) {  
   unsigned type = ELF32_R_TYPE(rel->r_info);  
   unsigned sym = ELF32_R_SYM(rel->r_info);  
   unsigned reloc = (unsigned)(rel->r_offset + si->base);  
   uint32_t page_size = 0;
   uint32_t entry_page_start = 0;
   unsigned oldval = 0;  
   if (sym_offset == sym) {  
    switch(type) {  
      case R_ARM_JUMP_SLOT:  
          
         page_size = getpagesize();
         entry_page_start = reloc& (~(page_size - 1));
         int ret = mprotect((uint32_t *)entry_page_start, page_size, PROT_READ | PROT_WRITE); 

         oldval = *(unsigned*) reloc;  
         *((unsigned*)reloc) = newval;  
         return 1;  
      default:  
         return 0;  
    }  
   }  
  }  
  return 0;  
 } 

```

用法：

```
hook_call("libandroid_runtime.so", "connect", &my_connect);  

```

1. 调用 dlopen 拿到 so 的句柄, 得到 soinfo, 它包含了符号表、重定位表、plt 表等信息。 2. 查找需要 hook 的函数的符号，得到它在符号表中的索引。具体实现是 soinfo_elf_lookup 函数。

![](https://ask.qcloudimg.com/http-save/yehe-1878670/yadesn7pyq.jpeg)

bucket 数组包含 nbucket 个项目，chain 数组包含 nchain 个项目，下标都是从 0 开始。bucket 和 chain 中都保存了符号表的索引。chain 表项和符号表存在对应。符号表项的数目应该和 nchain 相等，所以符号表的索引也可以用来选取 chain 表项。哈希函数能够接受符号名并返回一个可以用来计算 bucket 的索引。如果哈希函数针对某个名字返回了数值 x，则 bucket[x%nbucket] 给出了一个索引 y，该索引可用于符号表，也可用于 chain 表。如果该符号表项不是所需要的，那么 chain[y] 则给出了具有相同哈希值的下一个符号表项。我们可以沿着 chain 链一直搜索，直到所选中的符号表项包含了所需要的符号，或者 chain 项中包含值 STN_UNDEF。

3. 遍历 plt 表，直到匹配第 2 步中找到的符号索引。 如果是 JUMP_SLOT 类型（函数调用），替换为新的符号地址（函数指针）。

另外，程序中调用 mprotect 的作用是： 修改一段指定内存区域的保护属性。 [函数原型](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Flinux.die.net%2Fman%2F2%2Fmprotect)为：`int mprotect(const void *start, size_t len, int prot);` mprotect() 函数把自 start 开始的、长度为 len 的内存区的保护属性修改为 prot 指定的值。 需要指出的是，指定的内存区间必须包含整个内存页（4K）。区间开始的地址 start 必须是一个内存页的起始地址，并且区间长度 len 必须是页大小的整数倍。

参考
--

*   《程序员的自我修养》
*   《深入理解计算机系统》
*   [Andrey Petrov’s blog](https://cloud.tencent.com/developer/tools/blog-entry?target=http%3A%2F%2Fshadowwhowalks.blogspot.com%2F2013%2F01%2Fandroid-hacking-hooking-system.html)
*   [Redirecting functions in shared ELF libraries](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fwww.apriorit.com%2Fdev-blog%2F181-elf-hook)

本文参与 [腾讯云自媒体分享计划](https://cloud.tencent.com/developer/support-plan)，分享自作者个人站点 / 博客。

原始发表：2016 年 12 月 24 日，

如有侵权请联系 [cloudcommunity@tencent.com](mailto:cloudcommunity@tencent.com) 删除