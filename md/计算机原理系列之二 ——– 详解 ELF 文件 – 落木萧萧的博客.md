> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [luomuxiaoxiao.com](https://luomuxiaoxiao.com/?p=139)

> [toc]

[toc]

下面我们介绍一种文件格式：`ELF格式`，全名为`可执行和可链接格式（Executable and Linkable Format）`。维基百科中这样描述：

> 在计算机科学中，**_ELF 文件是一种用于可执行文件、目标文件、共享库和核心转储（`core dump`）的标准文件格式_**。其中核心转储是指： 操作系统在进程收到某些信号而终止时，将此时进程地址空间的内容以及有关进程状态的其他信息写出的一个磁盘文件。这种信息往往用于调试。

一、ELF 文件类型
----------

通俗点说由汇编器和链接器生成的文件都属于 ELF 文件。通常我们接触的 ELF 文件主要有以下三类：

*   **可重定位文件**（`relocatable file`） 它保存了一些可以和其他目标文件链接并生成可执行文件或者共享库的二进制代码和数据。
*   **可执行文件**（`excutable file`）它保存了适合直接加载到内存中执行的二进制程序。
*   **共享库文件**（`shared object file`）一种特殊的可重定位目标文件，可以在加载或者运行时被动态的加载进内存并链接。

总之，ELF 文件是一种文件格式。但凡是一种格式，总要有一些规则，下面我们来介绍 ELF 文件的格式规则。

二、ELF 文件结构
----------

一个典型的 ELF 文件包括 **ELF Header**、**Sections**、**Section Header Table** 和 **Program Header Table**。其位置分布如下图所示：  
![](http://luomuxiaoxiao.com/wp-content/uploads/2018/10/cs01-elf.png)  
图 1 ELF 文件构成

### 1. ELF Header

每个 ELF 文件都存在一个 ELF Header 用来描述其结构和组成。ELF Header 其实对应的是一个结构体，该结构体定义如下：

```
#define EI_NIDENT 16

   typedef struct {                     
       unsigned char e_ident[EI_NIDENT];
       uint16_t      e_type;            
       uint16_t      e_machine;         
       uint32_t      e_version;         
       ElfN_Addr     e_entry;           
       ElfN_Off      e_phoff;           
       ElfN_Off      e_shoff;           
       uint32_t      e_flags;           
       uint16_t      e_ehsize;          
       uint16_t      e_phentsize;       
       uint16_t      e_phnum;           
       uint16_t      e_shentsize;       
       uint16_t      e_shnum;           
       uint16_t      e_shstrndx;        
   } ElfN_Ehdr;
```

其中

```
ElfN_Addr       Unsigned program address, uintN_t
ElfN_Off        Unsigned file offset, uintN_t
```

上述结构体的各成员意义如下：

*   `e_ident`：包含一个 magic number、ABI 信息，该文件使用的平台、大小端规则
*   `e_type`：文件类型, 表示该文件属于可执行文件、可重定位文件、core dump 文件或者共享库
*   `e_machine`：机器类型
*   `e_version`：通常都是 1
*   `e_entry`: 表示程序执行的入口地址
*   `e_phoff`: 表示 Program Header 的入口偏移量（以字节为单位）
*   `e_shoff`: 表示 Section Header 的入口偏移量（以字节为单位）
*   `e_flags`: 保存了这个 ELF 文件相关的特定处理器的 flag
*   `e_ehsize`: 表示 ELF Header 大小（以字节为单位）
*   `e_phentsize`: 表示 Program Header 大小（以字节为单位）
*   `e_phnum`: 表示 Program Header 的数量 （十进制数字）
*   `e_shentsize`: 表示 Section Header 大小（以字节为单位）
*   `e_shnum`: 表示 Section Header 的数量 （十进制数字）
*   `e_shstrndx`: 表示字符串表的索引，字符串表用来保存 ELF 文件中的字符串，比如段名、变量名。 然后通过字符串在表中的偏移访问字符串。

例如，machine 的值为`0x3e`, 十进制为 62，其代表的意义可以从文件 [elf-em.h](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/elf-em.h#L31) 中找到，如下：

```
#define EM_X86_64  62  /* AMD x86-64 */
```

表示，这个 ELF 文件可以运行在 x86_64 的机器上。

### 2. Section

在 ELF 文件中，数据和代码分开存放的，这样可以按照其功能属性分成一些区域，比如程序、数据、符号表等。这些分离存放的区域在 ELF 文件中反映成`section`。ELF 文件中典型的`section`如下：

*   `.text`: 已编译程序的二进制代码
*   `.rodata`: 只读数据段，比如常量
*   `.data`: 已初始化的全局变量和静态变量
*   `.bss`: 未初始化的全局变量和静态变量，所有被初始化成 0 的全局变量和静态变量
*   `.sysmtab`: 符号表，它存放了程序中定义和引用的函数和全局变量的信息
*   `.debug`: 调试符号表，它需要以`'-g'`选项编译才能得到，里面保存了程序中定义的局部变量和类型定义，程序中定义和引用的全局变量，以及原始的 C 文件
*   `.line`: 原始的 C 文件行号和`.text节`中机器指令之间的映射
*   `.strtab`: 字符串表，内容包括`.symtab`和`.debug`节中的符号表

特殊的，  
1）对于**可重定位的文件**，由于在编译时，并不能确定它引用的外部函数和变量的地址信息，因此，编译器在生成目标文件时，增加了两个 section：

*   `.rel.text` 保存了程序中引用的外部函数的重定位信息，这些信息用于在链接时重定位其对应的符号。
*   `.rel.data` 保存了被模块引用或定义的所有全局变量的重定位信息，这些信息用于在链接时重定位其对应的全局变量。

2）对于**可执行文件**，由于它已经全部完成了重定位工作，可以直接加载到内存中执行，所以它不存在`.rel.text`和`.rel.data`这两个 section。但是，它增加了一个 section：

*   `.init`: 这个 section 里面保存了程序运行前的初始化代码

上述描述的各个文件中包含的这些 section 是必须存在的，当然除了这些 section，每种文件还有一些其他的 section 用来存放编译器或者链接器所需要的辅助信息，详情请参考**_参考阅读 1_**，在这里就不过多的讨论了。

### 3. Section Header Table

上述各个 section 的大小和位置等具体信息的存放是由 Section Header Table 来描述的。Section Header Table 是一个结构体数组，对应的结构体定义如下：

```
typedef struct {
    uint32_t   sh_name;
    uint32_t   sh_type;
    uint64_t   sh_flags;
    Elf64_Addr sh_addr;
    Elf64_Off  sh_offset;
    uint64_t   sh_size;
    uint32_t   sh_link;
    uint32_t   sh_info;
    uint64_t   sh_addralign;
    uint64_t   sh_entsize;
} Elf64_Shdr;
```

其中各成员的意义如下：

*   `sh_name`：表示该 section 的名字相对于`.shstrtab` section 的地址偏移量。
*   `sh_type`：表示该 section 中存放的内容类型，比如符号表，可重定位段等。
*   `sh_flags`： 表示该 section 的一些属性，比如是否可写，可执行等。
*   `sh_addr`： 表示该 section 在程序运行时的内存地址
*   `sh_offset`： 表示该 section 相对于 ELF 文件起始地址的偏移量
*   `sh_size`： 表示该 section 的大小
*   `sh_link`： 配合 sh_info 保存 section 的额外信息
*   `sh_info`：保存该 section 相关的一些额外信息
*   `sh_addralign`：表示该 section 需要的地址对齐信息
*   `sh_entsize`：有些 section 里保存的是一些固定长度的条目，比如符号表。对于这些 section 来讲，sh_entsize 里保存的就是条目的长度。

### 4. Program Header Table

前面讲过了，section 基本是按照目标文件内容的功能来划分的一些区域，而根据其内容在内存中是否可读写等属性，又可以将不同的 section 划分成不同的`segment`。其中每个 segment 可以由一个或多个 section 组成。其关系如图 1 所示。

在可执行文件中，ELF header 下面紧接着就是 Program Header Table。它描述了各个 segment 在 ELF 文件中的位置以及在程序执行过程中系统需要准备的其他信息。它也是用一个结构体数组来表示的。具体代码如下：

```
typedef uint64_t  Elf64_Addr;
typedef uint64_t  Elf64_Off;
typedef uint32_t  Elf64_Word;
typedef uint64_t  Elf64_Xword;

typedef struct {
    Elf64_Word      p_type;         // 4
    Elf64_Word      p_flags;        // 4
    Elf64_Off       p_offset;       // 8
    Elf64_Addr      p_vaddr;        // 8
    Elf64_Addr      p_paddr;        // 8
    Elf64_Xword     p_filesz;       // 8
    Elf64_Xword     p_memsz;        // 8
    Elf64_Xword     p_align;        // 8
} Elf64_Phdr;
```

各个字段的具体含义如下：

*   `p_type`：描述了当前 segment 是何种类型的或者如何解释当前 segment，比如是动态链接相关的或者可加载类型的等
*   `p_flags`：保存了该 segment 的 flag
*   `p_offset`：表示从 ELF 文件到该 segment 第一个字节的偏移量
*   `p_vaddr`：表示该 segment 的第一个字节在内存中的虚拟地址
*   `p_paddr`：对于使用物理地址的系统来讲，这个成员表示该 segment 的物理地址
*   `p_filesz`：表示该 segment 的大小，以字节表示
*   `p_memsz`：表示该 segment 在内存中的大小，以字节表示
*   `p_align`：表示该 segment 在文件中或者内存中需要以多少字节对齐

三、实践
----

为了进一步加深对 ELF 文件整体结构的理解，我们取一个 64bit 的 linux 上的可执行文件`hello`来验证。  
首先，使用`READELF`工具查看其 ELF Header 信息：

```
$ readelf -h hello
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x400430
  Start of program headers:          64 (bytes into file)
  Start of section headers:          6616 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         9
  Size of section headers:           64 (bytes)
  Number of section headers:         31
  Section header string table index: 28
```

从 ELF Header 中我们得到如下信息：

*   ELF Header 大小（`Size of this header`）为 64 byte
*   Program Header Table 的起始地址（`Start of program headers`）是 64 byte，其内部共有 9 个（`Number of program headers`）Program Header，每个大小为 56 byte（`Size of program headers`）
*   Section Header Table 的起始地址（`Start of section headers`）是 6616 byte，其内部共有 31 个（`Number of section headers`）Section Header，每个大小为 64 byte（`Size of this header`）

其次，使用`READELF`命令读取 section header table 信息：

```
$ readelf -S hello
There are 31 section headers, starting at offset 0x19d8:

Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  [ 0]                   NULL             0000000000000000  00000000
       0000000000000000  0000000000000000           0     0     0
  [ 1] .interp           PROGBITS         0000000000400238  00000238
       000000000000001c  0000000000000000   A       0     0     1
  [ 2] .note.ABI-tag     NOTE             0000000000400254  00000254
       0000000000000020  0000000000000000   A       0     0     4
  [ 3] .note.gnu.build-i NOTE             0000000000400274  00000274
       0000000000000024  0000000000000000   A       0     0     4

    ... snip ...

  [27] .comment          PROGBITS         0000000000000000  00001038
       0000000000000035  0000000000000001  MS       0     0     1
  [28] .shstrtab         STRTAB           0000000000000000  000018cc
       000000000000010c  0000000000000000           0     0     1
  [29] .symtab           SYMTAB           0000000000000000  00001070
       0000000000000648  0000000000000018          30    47     8
  [30] .strtab           STRTAB           0000000000000000  000016b8
       0000000000000214  0000000000000000           0     0     1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), l (large)
  I (info), L (link order), G (group), T (TLS), E (exclude), x (unknown)
  O (extra OS processing required) o (OS specific), p (processor specific)
```

根据上述 section 信息，我们得到:

*   section 的起始地址是`0x238`（第一个 section `.interp`的起始地址）
*   section 的结束地址是`.shstrtab`的结束地址即：`0x18cc + 0x10c = 0x19d8`

根据我们上面的结论，我们可以得到该 ELF 文件的组成图如下:

<table><thead><tr><th>name</th><th>start_addr</th><th>size</th><th>end_addr</th></tr></thead><tbody><tr><td>ELF Header</td><td>0</td><td>0x40</td><td></td></tr><tr><td>Program Header Table</td><td>0x40</td><td>0x1F8</td><td></td></tr><tr><td>Section</td><td>0x238</td><td></td><td>0x19D8</td></tr><tr><td>Section Header Table</td><td>0x19D8</td><td>0x7C0</td><td></td></tr></tbody></table>

我们可以计算出各部分的 end_addr 上述空白部分，得到：

![](http://luomuxiaoxiao.com/wp-content/uploads/2018/10/cs02-table.png)

显然，各个 section 正好和下一个 section 首尾相接。然后，通过`ls -l hello`命令查看该二进制文件的大小，得到该文件大小为`8600 byte`，而`Section Header Table`的 end_addr `0x2198`换算成十进制大小正好是`8600 byte`。从而证明我们上述的分析是没问题的。

四、总结
----

1.  ELF 文件包括可重定位文件、可执行文件、共享库和 Core dump 文件。其基本结构包括 ELF Header，Program Header Table，Sections 和 Section Header Table。其中不同的 sections 组成了不同的 segment。
2.  ELF Header 描述了该 ELF 文件中 Program Header Table 的位置、大小，Section Header Table 的位置、大小和程序执行的入口地址等信息。
3.  Program Header Table 描述了各个 segment 的类型、虚拟地址、相对文件的偏移地址等信息。
4.  Section Header Table 描述了各个 section 的名字、类型、相对文件的偏移地址等信息。

五、参考阅读
------

1.  [Executable and Linkable Format (ELF)](http://www.cs.cmu.edu/afs/cs/academic/class/15213-s00/doc/elf.pdf)
2.  [UNIX/LINUX 平台可执行文件格式分析](https://www.ibm.com/developerworks/cn/linux/l-excutff/index.html)
3.  [ELF man page](http://man7.org/linux/man-pages/man5/elf.5.html)
4.  [打造史上最小可执行 ELF 文件](https://www.w3cschool.cn/cbook/7to1eozt.html)
5.  [_SYSTEM V:APPLICATION BINARY INTERFACE_ Edition 4.1](http://www.sco.com/developers/devspecs/gabi41.pdf)
6.  [Linux 内核如何装载和启动一个可执行程序](http://www.codexiu.cn/linux/blog/8995/)