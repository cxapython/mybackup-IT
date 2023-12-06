> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [kevinspider.github.io](https://kevinspider.github.io/so1/)

> ARM 基础三级流水 PC 代表程序技术器, 三级流水是一个指令执行的三个阶段 取指 (从存储器中装载一条指令) 译码 (识别将要执行的指令) 执行 (处理指令并将结果写回寄存器) PC 总是指向正在......

[](#ARM-基础 "ARM 基础")ARM 基础
--------------------------

### [](#三级流水 "三级流水")三级流水

PC 代表程序技术器, 三级流水是一个指令执行的三个阶段

1.  取指 (从存储器中装载一条指令)
2.  译码 (识别将要执行的指令)
3.  执行 (处理指令并将结果写回寄存器)

PC 总是指向`正在取指`的指令, 而不是指向正在执行或者正在译码的指令;

假设当前在 arm 指令下, 每条指令占用 4 个字节; 那么 pc 指向的是正在取指的指令地址, cpu 正在译指的地址是 PC-4; cpu 正在执行的指令地址是 pc-8;

也就是说 PC 所指向的地址和现在所执行的指令地址相差 8;

[](#ELF-文件 "ELF 文件")ELF 文件
--------------------------

*   为什么要学习 ELF parser
    *   文件格式要跟着解析器一起学习
*   为什么要 dump so;
    *   很多加固和混淆的 so 在静态看的时候是无法正常查看的
    *   在静态 –> so 解密 –> 加载到内存中 –> dump 下来的是经过解密之后的 so
*   为什么 dump 下来的 so 无法直接反汇编
    *   在 elf 文件装载过程中会丢失信息, 需要重新建立这些信息才能反汇编
*   dump 下来的 so 如何进行修复
*   为什么修复之后就可以反汇编了

### [](#结构综述 "结构综述")结构综述

ELF 文件的开头是一个文件头, 它描述了整个文件的文件属性, 包括文件是否可执行, 是静态链接还是动态链接以及入口地址 (如果是可执行文件), 目标硬件, 目标操作系统等信息;

文件头还包括一个段表 (Section Table), 段表其实是一个描述文件中各个段的数组, 段表描述了文件中各个段在文件中的偏移位置以及段的属性等, 从段表中可以得到每个段的所有信息;

文件头后面就是就是各个段的内容, 比如代码段保存的就是程序的指令, 数据段保存的就是程序的静态变量等; 一般 C 语言编译后执行语句都编译成机器码, 保存在. text 段中; 已经初始化的全局变量和局部静态变量都保存在. data 段; 未初始化的全局变量和局部静态变量一般放在. bss 段中; .bss 段只是为未初始化的全局变量和局部静态变量预留位置而已, 它并没有内容, 所以它在文件中也不占据空间;

总体来说, 程序源代码编译之后主要分成两种段: 程序指令和程序数据; 代码段属于程序指令, 而数据段和. bss 段属于程序数据;

### [](#文件头 "文件头")文件头

可以使用 readelf 命令来查看 elf 文件; `readelf -h <soPath>`

```
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          52 (bytes into file)
  Start of section headers:          29216 (bytes into file)
  Flags:                             0x5000200, Version5 EABI, soft-float ABI
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         8
  Size of section headers:           40 (bytes)
  Number of section headers:         27
  Section header string table index: 26
```

ELF Header 数据结构

1.  e_ident 数组
    1.  EI_MAG0 文件标识 –> 7f
    2.  EI_MAG1 文件标识 –> 45 –> E
    3.  EI_MAG2 文件标识 –> 4c –> L
    4.  EI_MAG3 文件标识 –> 46 –> F
    5.  EL_CLASS 文件类型
        1.  0 非法类型
        2.  1 32 位目标 ELFCLASS32
        3.  2 64 位目标 ELFCLASS64
    6.  EI_DATA
        1.  0 ELFDATANONE 非法数据编码
        2.  1 ELFDATA2LSB 小端 高位在前
        3.  2 ELFDATA2MSB 大端 低位在前
    7.  EI_VERSION ELF 头部的版本号码, 目前必须是 EV_CURRENT
    8.  EI_PAD 标记 e_ident 中未使用字节的开始, 初始化为 0;
2.  e_type 目标文件类型
    1.  0 ET_NONE 未知目标文件格式
    2.  1 ET_REL 可重定位文件
    3.  2 ET_EXEC 可执行文件
    4.  3 ET_DYN 共享目标文件, 一般为 so 文件
    5.  4 ET_CORE Core 文件
    6.  0xff00 ET_LOPPOC 特定处理器文件
    7.  0xffff ET_HIPROC 特定处理器文件
3.  e_machine 给出文件的目标体系结构类型
    1.  0 EM_NONE 未指定
    2.  1 EM_M32 AT&T WE 32100
    3.  2 EM_SPARC SPARC
    4.  3 EM_386 INTEL80386
    5.  4 EM_68K MOTOROLA 68000
    6.  5 EM_88K MOTOROLA 88000
    7.  7 EM_860 INTEL 80860
    8.  8 EM_MIPS MIPS RS3000
    9.  其他的都是保留的, 特定处理器 elf 名称会使用机器名进行区分
4.  e_version 目标文件版本
    1.  0 EV_NONE 非法版本
    2.  1 EV_CURRENT 当前版本
5.  e_entry 程序入口的虚拟地址, 如果目标文件没有程序入口, 可以为 0
6.  e_phoff 程序头部表格 Program Header Table 的偏移量, 如果程序没有头部表格, 可以为 0
7.  e_shoff 节区头部表格 Section Header Table 的偏移量, 如果文件没有节区头部表格, 可以为 0
8.  e_flags 保存与文件相关的特定处理器的标志
9.  e_ehsize elf 头部的大小 (以字节计算)
10.  e_phentsize 程序头部表格的表项大小, 按字节计算
11.  e_phnum 程序头部表格的表项数数量, 可以为 0
12.  e_shentsize 节区头部表格的表项大小, 按字节计算
13.  e_shnum 节区头部表格的表项数目, 可以为 0
14.  e_shstrndx 节区头部表格中与节区名称字符串表相关的表项的索引, 如果文件没有节区名称字符串表, 此参数可以为 SHN_UNDEF

### [](#段表 "段表")段表

ELF 文件中有很多各种各样的段, 每个段表 Section Header Table 就是保存这些段的基本属性的结构; 段表是 ELF 文件中除了文件头以外最重要的结构, 它描述 了 ELF 的各个段的信息; 比如段的段名, 段的长度, 在文件中的偏移, 读写权限, 段的其他属性;

ELF 文件的段结构就是由段表决定的, 编译器, 链接器和装载器都是根据段表来定位和访问各个段的属性的; 段表在 ELF 文件中由 ELF 文件头的 e_shoff 成员决定;

可以使用 `readelf -S <soPath>` 来查看段表结构;

```
There are 27 section headers, starting at offset 0x7220:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .note.androi[...] NOTE            00000134 000134 000098 00   A  0   0  4
  [ 2] .note.gnu.bu[...] NOTE            000001cc 0001cc 000024 00   A  0   0  4
  [ 3] .dynsym           DYNSYM          000001f0 0001f0 000770 10   A  4   1  4
  [ 4] .dynstr           STRTAB          00000960 000960 0015ea 00   A  0   0  1
  [ 5] .gnu.hash         GNU_HASH        00001f4c 001f4c 000284 04   A  3   0  4
  [ 6] .hash             HASH            000021d0 0021d0 000368 04   A  3   0  4
  [ 7] .gnu.version      VERSYM          00002538 002538 0000ee 02   A  3   0  2
  [ 8] .gnu.version_d    VERDEF          00002628 002628 00001c 00   A  4   1  4
  [ 9] .gnu.version_r    VERNEED         00002644 002644 000040 00   A  4   2  4
  [10] .rel.dyn          REL             00002684 002684 000318 08   A  3   0  4
  [11] .rel.plt          REL             0000299c 00299c 0002a0 08   A  3   0  4
  [12] .plt              PROGBITS        00002c3c 002c3c 000404 00  AX  0   0  4
  [13] .text             PROGBITS        00003040 003040 00253c 00  AX  0   0  4
  [14] .ARM.exidx        ARM_EXIDX       0000557c 00557c 000390 08  AL 13   0  4
  [15] .ARM.extab        PROGBITS        0000590c 00590c 0004a4 00   A  0   0  4
  [16] .rodata           PROGBITS        00005db0 005db0 000627 00   A  0   0  1
  [17] .data.rel.ro      PROGBITS        00007bc4 006bc4 000188 00  WA  0   0  4
  [18] .fini_array       FINI_ARRAY      00007d4c 006d4c 000008 00  WA  0   0  4
  [19] .dynamic          DYNAMIC         00007d54 006d54 000118 08  WA  4   0  4
  [20] .got              PROGBITS        00007e6c 006e6c 000194 00  WA  0   0  4
  [21] .data             PROGBITS        00008000 007000 000004 00  WA  0   0  4
  [22] .bss              NOBITS          00008004 007004 000001 00  WA  0   0  1
  [23] .comment          PROGBITS        00000000 007004 0000b5 01  MS  0   0  1
  [24] .note.gnu.go[...] NOTE            00000000 0070bc 00001c 00      0   0  4
  [25] .ARM.attributes   ARM_ATTRIBUTES  00000000 0070d8 000036 00      0   0  1
  [26] .shstrtab         STRTAB          00000000 00710e 00010f 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  y (purecode), p (processor specific)
```

段表结构比较简单, 它是一个以`Elf32_Shdr`结构体为元素的数组, 数组元素的个数等于段的个数, 每个`Elf32_Shdr`结构体对应一个段, `Elf32_Shar`又被称为段描述符, 上面的 so 文件一共有 0-26 个段描述符, 也就是由 27 个元素的数组; ELF 段表的第一个元素是无效的段描述符, 它的类型是 NULL; 除此之外每个段描述符都对应一个段, 也就是说该 so 文件共有 26 个有效的段;

简而言之, 段表就是一个大数组, 数组中每一个元素都是一个段描述符的结构体;

#### [](#段描述符 "段描述符")段描述符

1.  sh_name: Section name 段名; 段名是个字符串, 它位于一个叫做 .shstrtab 的字符串表, sh_name 是段名字符串在 .shstrtab 中的偏移;
2.  sh_type: Section type 段的类型, 参考后面 –> 段的类型
3.  sh_flags: Section flag 段的标志位, 参考后面 –> 段的标志位
4.  sh_addr: Section Address 段虚拟地址; 如果该段可以被加载, 那么 sh_addr 为该段被加载后在进程地址空间中的虚拟地址, 否则为 0
5.  sh_offset: Section Offset 段偏移; 如果该段存在于文件中, 则表示该段在文件中的偏移; 否则无意义; 例如, sh_offset 对于 BSS 段来说就没有意义
6.  sh_size: Section Size 段的长度;
7.  sh_link 和 sh_info: Section link 和 Section Information 段链接信息, 参考后面 –> 段的链接信息
8.  sh_addralign: 段地址对齐; 有些段对段地址对齐有要求, 由于地址对齐的数量都是 2 的指数倍, sh_addralign 表示地址对齐数量中的指数, 3 表示 2^3, 如果 sh_addralign 为 0 或 1, 则表示该段没有对齐要求
9.  sh_entsize: Section Entry Size 项的长度; 有些段包含了一些固定大小的项, 比如符号表, 它包含的每个符号占用的大小都是一样的, 对于这种段, sh_entsize 表示每个项的大小, 如果为 0, 则表示该段不包含固定大小的项;

#### [](#段的类型 "段的类型")段的类型

段的名字只在链接和编译的过程中有意义, 对于编译器和链接器而言, 主要决定段的属性的是段的类型和段的标志位, 段的类型如下, 主要以商量 SHT 开头

<table><thead><tr><th>常量</th><th>值</th><th>含义</th></tr></thead><tbody><tr><td>SHT_NULL</td><td>0</td><td>无效段</td></tr><tr><td>SHT_PROGBITS</td><td>1</td><td>程序段, 代码段, 数据段都是这种类型的</td></tr><tr><td>SHT_SYMTAB</td><td>2</td><td>表示该段的内容是符号表</td></tr><tr><td>SHT_STRTAB</td><td>3</td><td>表示该段的内容为字符串表</td></tr><tr><td>SHT_RELA</td><td>4</td><td>重定位表, 该段包含了重定位信息, 可以参考后面的重定位</td></tr><tr><td>SHT_HASH</td><td>5</td><td>符号表的哈希表, 参考符号表</td></tr><tr><td>SHT_DYNAMIC</td><td>6</td><td>动态链接信息, 参考动态链接</td></tr><tr><td>SHT_NOTE</td><td>7</td><td>提示信息</td></tr><tr><td>SHT_NOBITS</td><td>8</td><td>表示该段在文件中没有内容, 比如. bss 段</td></tr><tr><td>SHT_REL</td><td>9</td><td>该段包含了重定位信息, 具体参考重定位</td></tr><tr><td>SHT_SHLIB</td><td>10</td><td>保留</td></tr><tr><td>SHT_DNYSYM</td><td>11</td><td>动态链接的符号表, 参考动态链接</td></tr></tbody></table>

#### [](#段的标志位 "段的标志位")段的标志位

段的标志位表示该段在程序虚拟地址空间中的属性, 比如是否可写, 是否可执行等; 相关常量通常以 SHF_开头;

<table><thead><tr><th>常量</th><th>值</th><th>含义</th></tr></thead><tbody><tr><td>SHF_WRITE</td><td>1</td><td>表示该段在进程空间站中可写</td></tr><tr><td>SHF_ALLOC</td><td>2</td><td>表示该段在进程空间中需要分配空间, 像代码段, 数据段和. bss 段都会有这个标志位</td></tr><tr><td>SHF_EXECINSTR</td><td>4</td><td>表示该段在进程空间中可以被执行, 一般指代码段</td></tr></tbody></table>

系统保留段的属性

<table><thead><tr><th>Name</th><th>sh_type</th><th>sh_flag</th></tr></thead><tbody><tr><td>.bss</td><td>SHT_NOBITS</td><td>SHF_ALLOC+SHF_WRITE</td></tr><tr><td>.comment</td><td>SHT_PROGBITS</td><td>none</td></tr><tr><td>.data</td><td>SHT_PROGBITS</td><td>SHF_ALLOC+SHF_WRITE</td></tr><tr><td>.data1</td><td>SHT_PROGBITS</td><td>SHF_ALLOC+SHF_WRITE</td></tr><tr><td>.debug</td><td>SHT_PROGBITS</td><td>none</td></tr><tr><td>.dynamic</td><td>SHT_DYNAMIC</td><td>SHF_ALLOC+SHF_WRITE</td></tr><tr><td>.hash</td><td>SHT_HASH</td><td>SHF_ALLOC</td></tr><tr><td>.line</td><td>SHT_PROGBITS</td><td>none</td></tr><tr><td>.note</td><td>SHT_NOTE</td><td>none</td></tr><tr><td>.rodata</td><td>SHT_PROGBITS</td><td>SHF_ALLOC</td></tr><tr><td>.rodata1</td><td>SHT_PROGBITS</td><td>SHF_ALLOC</td></tr><tr><td>.shstrtab</td><td>SHT_STRTAB</td><td>none</td></tr><tr><td>.strtab</td><td>SHT_STRTAB</td><td>如果该 ELF 文件中有可装载的段需要用到该字符串表, 那么该字符串表也将被装载到进程空间, 则有 SHF_ALLOC 标志位</td></tr><tr><td>.symtab</td><td>SHT_SYMTAB</td><td>同字符串表</td></tr><tr><td>.text</td><td>SHT_PROGBITS</td><td>SHF_ALLOC+SHF_EXECINSTR</td></tr></tbody></table>

#### [](#段的链接信息 "段的链接信息")段的链接信息

如果段的类型是与链接相关的, 不论是动态链接或者静态链接, 比如重定位表, 符号表等; 那么 sh_link 和 sh_info 这两个成员所包含的意义如下图; 对于其他类型的段, 这两个成员没有意义;

<table><thead><tr><th>sh_type</th><th>sh_link</th><th>sh_ info</th></tr></thead><tbody><tr><td>SHT_DYNAMIC</td><td>该段所使用的字符串在段表中的下标</td><td>0</td></tr><tr><td>SHT_HASH</td><td>该段所使用的符号表在段表中的下标</td><td>0</td></tr><tr><td>SHT_REL\SHT_RELA</td><td>该段所使用的相应符号表在段表中的下标</td><td>该重定向表所作用的段在段表中的下标</td></tr><tr><td>SHT_SYMTAB/SHT_DYNSYM</td><td>操作系统相关</td><td>操作系统相关</td></tr><tr><td>other</td><td>SHN_UNDEF</td><td>0</td></tr></tbody></table>

#### [](#重定位表 "重定位表")重定位表

链接器在处理目标文件时, 需要对目标文件中某些部位进行重定位, 即代码段和数据段中那些对绝对地址的引用的位置; 这些重定位信息都会记录在 ELF 文件的重定位表里面; 对于每个需要重定位的代码段或者数据段, 都会有一个相应的重定位表; 比如 `.rel.text`就是对`.text`段的重定位表; `.rel.data`就是对 `.data`段的重定位表;

一个重定位表同时也是 ELF 的一个段, 那么这个段的类型就是 SHT_REL 类型; 它的 sh_link 表示符号表的下标; 它的 sh_info 表示它作用于那个段; 比如 .rel.text 作用于 .text, 而. text 段的下标是 1, 那么 .rel.text 的 sh_info 为 1;

#### [](#字符串表 "字符串表")字符串表

ELF 文件中用到了很多字符串, 比如段名, 变量名等; 因为字符串的长度往往是不定的, 所以很难用固定的结构来表示; 常见的方式是把字符串集中起来放在一个表里, 然后使用字符串在表中的偏移来引用字符串;

![](https://kevinspider-1258012111.cos.ap-shanghai.myqcloud.com/2021-07-19-063649.png)

在 ELF 文件中引用字符串值需要给出一个数字下标即可; 不用考虑字符串的长度问题; 一般字符串在 ELF 文件中也以段的形式保存, 常见的段名为`.strtab`或者 `.shstrtab`; 这两个字符串表分别为字符串表和段表字符串表; .strtab 字符串表就是用来保存普通的字符串, 比如符号的名字; 段表字符串表用来保存段中用到的字符串, 最常见的就是段名 (sh_name);

e_shstrndx 是 ELF32_Ehdr 的最后一个成员, 是 Section header string table index 的缩写, 我们知道段表字符串本身也是 ELF 文件中的一个普通的段, 知道它的名字往往叫做 .shstrtab; 那么这个 e_shstrndx 就是表示 .shstrtab 在段表中的下标; 即段表字符串表在段表中的下标;

只有分析 ELF 文件头, 才可以得到段表和段表字符串表的位置, 从而才能解析整个 ELF 文件;

#### [](#代码段 "代码段")代码段

`objdump -h <soPath>` -h 参数可以将 ELF 文件中各个段的基本信息打印出来; 或者使用 -x 参数, 打印更详细的信息;

`objdump -s -d <soPath>` -s 参数可以将所有段的内容以十六进制的方式打印出来, -d 参数可以将所有的包含指令的段反汇编;

#### [](#数据段和只读数据段 "数据段和只读数据段")数据段和只读数据段

.data 段保存的是已经初始化的全局静态变量和局部静态变量;

.rodata 段存放的是只读数据, 一般是程序中只读变量,(如 const 修改的变量) 和字符串常量; 有时候编译器会把字符串常量放在 .data 段, 而不会单独放在. rodata 段;

#### [](#bss-段 "bss 段")bss 段

.bss 段存放的是未初始化的全局变量和局部静态变量;

#### [](#其他段 "其他段")其他段

<table><thead><tr><th>常用的段名</th><th>说明</th></tr></thead><tbody><tr><td>.rodata1</td><td>Read only Data. 在这种段里存放的是只读数据, 比如字符常量, 全局 const 变量, 和 .rodata 一样</td></tr><tr><td>.comment</td><td>存放的是编译器版本信息, 比如字符串: “GCC: (GNU) 4.2.0”</td></tr><tr><td>.debug</td><td>调试信息</td></tr><tr><td>.dynamic</td><td>动态链接信息</td></tr><tr><td>.hash</td><td>符号哈希表</td></tr><tr><td>.line</td><td>调试时的行号表</td></tr><tr><td>.note</td><td>额外的编译器信息, 比如程序的公司名, 发布版本号等</td></tr><tr><td>.strtab</td><td>String Table 字符串表, 用于存储 ELF 文件中用到的各种字符串</td></tr><tr><td>.symtab</td><td>Symbol Table 符号表</td></tr><tr><td>.shstrtab</td><td>Section String Table 段名表</td></tr><tr><td>.plt .got</td><td>动态链接的跳转表和全局入口表</td></tr><tr><td>.init .fini</td><td>程序初始化与终结代码段</td></tr></tbody></table>

#### [](#自定义段 "自定义段")自定义段

正常情况下, GCC 编译出来的目标文件中, 代码会被放在. text 段中; 初始化的全局变量和局部静态变量会被放到. data 段中; 未初始化的全局变量和静态变量会被放到. bss 段中; 如果想要某些部分的代码或者变量放到自己指定的段中, 可以通过通过下面代码指定变量所处的段:

```
__attribute__((section("FOO"))) int global = 42;
__attribute__((section("BAR"))) void foo()
(
)
```

我们通过在全局变量或者函数前面加上`__attribute__((section("name")))`属性就可以将相应的变量或者函数放到以 name 命名的段中;

### [](#符号 "符号")符号

#### [](#符号定义 "符号定义")符号定义

##### [](#链接本质 "链接本质")链接本质

链接过程的本质就是将多个不同的目标文件之间相互粘在一起; 或者说是就像拼积木一样拼装成一个整体; 在链接中, 目标文件之间相互拼合实际上是对目标文件之间对地址的引用;

##### [](#定义和引用 "定义和引用")定义和引用

比如目标文件 B 要用到目标文件 A 中的函数 foo, 那么目标文件 A 定义了函数 foo, 目标文件 B 引用了目标文件 A 中的函数 foo; 这个概念也同样适用于变量;

##### [](#符号名和符号值 "符号名和符号值")符号名和符号值

在链接中, 我们将函数和变量统称为符号, 函数名和变量名就是符号名;

每个目标文件都会有一个相应的符号表, 这个表里记录了目标文件中所用到的所有符号; 每个定义的符号都有一个对应的值, 叫做符号值, 对于函数和变量来说, 符号值就是他们的地址;

##### [](#符号分类 "符号分类")符号分类

*   全局符号: 定义在本目标文件的全局符号, 可以被其他目标文件引用;
*   外部符号: 在本目标文件中引用的全局符号, 却没有定义在本目标文件, 也叫做符号引用;
*   段名: 一般是编译器产生的, 它的值就是该段的起始地址
*   局部符号: 只在编译单元内部可见; 这些局部符号对于链接过程没有作用. 链接器也往往忽略它们;
*   行号信息: 目标文件指令和源代码中代码行的对应关系, 可选;

一般值得关注的就是全局符号和外部符号; 因为链接过程只关心全局符号的相互粘合;

#### [](#查看文件符号 "查看文件符号")查看文件符号

`nm <soPath>`

`readelf -s <soPath>`

### [](#符号表结构 "符号表结构")符号表结构

ELF 文件中的符号往往是一个文件中的一个段, 段名一般是`.symtab`; 符号表的结构比较简单, 它是一个 Elf32_Sym 结构的数组; 数组中的每个元素对应一个符号; 数组中的第一个元素也就是下标 0 的元素为无效的未定义符号;

数组中每个元素的结构如下:

<table><thead><tr><th>名称</th><th>含义</th></tr></thead><tbody><tr><td>st_name</td><td>符号名, 包含了该符号名在字符串表中的下标</td></tr><tr><td>st_value</td><td>符号相对应的值, 这个值和符号有关, 可能是一个绝对值, 也可能是一个地址等; 不同的符号, 它所对应的值含义不同; 参考 –&gt; 符号值</td></tr><tr><td>st_size</td><td>符号大小, 对于包含数据的符号, 这个值是该数据类型的大小; 如果该值为 0, 则表示该符号大小为 0 或者未知;</td></tr><tr><td>st_info</td><td>符号类型和绑定信息</td></tr><tr><td>st_other</td><td>该成员目前是 0, 没用</td></tr><tr><td>st_shndx</td><td>符号所在的段, 参考 –&gt; 符号所在段</td></tr></tbody></table>

#### [](#st-info-符号类型和绑定信息 "st_info 符号类型和绑定信息")st_info 符号类型和绑定信息

该成员低 4 位表示符号的类型 (Symbol Type), 高 28 位表示符号绑定信息 (Symbol Binding):

符号绑定信息表:

<table><thead><tr><th>名称</th><th>值</th><th>说明</th></tr></thead><tbody><tr><td>STB_LOCAL</td><td>0</td><td>局部符号, 对于目标文件的外部不可见</td></tr><tr><td>STB_GLOBAL</td><td>1</td><td>全局符号, 外部可见</td></tr><tr><td>STB_WEAK</td><td>2</td><td>弱引用, 参考 –&gt; 弱符号和强符号</td></tr></tbody></table>

符号类型表:

<table><thead><tr><th>名称</th><th>值</th><th>说明</th></tr></thead><tbody><tr><td>STT_NOTYPE</td><td>0</td><td>未知类型符号</td></tr><tr><td>STT_OBJECT</td><td>1</td><td>该符号是个数据对象, 比如变量, 数组等</td></tr><tr><td>STT_FUNC</td><td>2</td><td>该符号是个函数或者其他科执行代码</td></tr><tr><td>STT_SECTION</td><td>3</td><td>该符号表示一个段, 这种符号必须是 STB_LOCAL</td></tr><tr><td>STT_FILE</td><td>4</td><td>该符号表示文件名, 一般都是该目标文件所对应的源文件名; 它一定是 STB_LOCAL 类型的, 而且它的 st_shndx 一定是 SHN_ABS</td></tr></tbody></table>

#### [](#st-shndx-符号所在段 "st_shndx 符号所在段")st_shndx 符号所在段

如果符号定义在本目标文件中, 那么这个成员表示符号所在的段在段表中的下标; 如果符号不是定义在本目标文件中, 或者对于有些特殊符号, sh_shndx 的值有些特殊:

<table><thead><tr><th>名称</th><th>值</th><th>含义</th></tr></thead><tbody><tr><td>SHN_ABS</td><td>0xfff1</td><td>该符号包含了一个绝对的值, 比如文件名符号就属于这种类型</td></tr><tr><td>SHN_COMMON</td><td>0xfff2</td><td>该符号是一个 COMMON 块类型的符号, 一般未初始化的全局符号定义就是这种类型的; 参考 –&gt; COMMON 块</td></tr><tr><td>SHN_UNDEF</td><td>0</td><td>表示该符号未定义, 这个符号表示该符号在本目标文件被引用到, 但是定义在其他目标文件中;</td></tr></tbody></table>

#### [](#st-value-符号值 "st_value 符号值")st_value 符号值

每个符号都有对应的值, 如果这个符号是一个函数或者变量, 那么符号的值就是这个函数或者变量的地址; 但是这个地址需要分下面的情况区别对待:

1.  在目标文件中, 如果是符号的定义且该符号不是 COMMON 块 (st_shndx 不是 SHN_COMMON), 则 st_value 表示该符号在段中的偏移; 也就是说符号所对应的函数或者变量位于由 st_shndx 指定的段, 偏移 st_value 的位置;
2.  在目标文件中, 如果符号是 COMMON 块, 即 st_shndx 为 SHN_COMMON 则 st_value 表示该符号的对齐属性;
3.  在可执行文件中, st_value 表示符号的虚拟地址; 这个虚拟地址对于动态链接器来说十分有用;

使用 readelf 可以查看符号表

```
Symbol table '.dynsym' contains 119 entries:
   Num:    Value  Size Type    Bind   Vis      Ndx Name
     0: 00000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 00000000     0 OBJECT  GLOBAL DEFAULT  UND __stack[...]@LIBC (2)
     2: 00000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_atexit@LIBC (2)
     3: 00000000     0 FUNC    GLOBAL DEFAULT  UND strlen@LIBC (2)
     4: 00000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_f[...]@LIBC (2)
     5: 00000000     0 FUNC    GLOBAL DEFAULT  UND __stack[...]@LIBC (2)
     6: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZN4test5TestBEv
     7: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZNKSt6__ndk16lo[...]
     8: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZNKSt6__ndk18io[...]
     9: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSt11logic_err[...]
    10: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSt12length_er[...]
    11: 00000000     0 FUNC    GLOBAL DEFAULT  UND snprintf@LIBC (2)
    12: 00000000     0 OBJECT  GLOBAL DEFAULT  UND _ZNSt6__ndk15cty[...]
    13: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSt6__ndk16loc[...]
    14: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSt6__ndk16loc[...]
    15: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSt6__ndk18ios[...]
    16: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSt6__ndk18ios[...]
    17: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSt6__ndk18ios[...]
    18: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZNSt6__ndk18ios[...]
    19: 00000000     0 FUNC    GLOBAL DEFAULT  UND abort@LIBC (2)
    20: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZSt18uncaught_e[...]
    21: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZSt9terminatev
    22: 00000000     0 OBJECT  GLOBAL DEFAULT  UND _ZTINSt6__ndk18i[...]
    23: 00000000     0 OBJECT  GLOBAL DEFAULT  UND _ZTISt12length_error
    24: 00000000     0 OBJECT  GLOBAL DEFAULT  UND _ZTVN10__cxxabiv[...]
    25: 00000000     0 OBJECT  GLOBAL DEFAULT  UND _ZTVN10__cxxabiv[...]
    26: 00000000     0 OBJECT  GLOBAL DEFAULT  UND _ZTVN10__cxxabiv[...]
    27: 00000000     0 OBJECT  GLOBAL DEFAULT  UND __sF@LIBC (2)
    28: 00000000     0 OBJECT  GLOBAL DEFAULT  UND _ZTVSt12length_error
    29: 00000000     0 FUNC    GLOBAL DEFAULT  UND dladdr@LIBC (3)
    30: 00000000     0 FUNC    GLOBAL DEFAULT  UND _ZdlPv
    31: 00000000     0 FUNC    GLOBAL DEFAULT  UND fprintf@LIBC (2)
    32: 00000000     0 FUNC    GLOBAL DEFAULT  UND _Znwj
    33: 00000000     0 FUNC    GLOBAL DEFAULT  UND fflush@LIBC (2)
    34: 00000000     0 FUNC    GLOBAL DEFAULT  UND __aeabi_memcpy
    35: 00000000     0 FUNC    GLOBAL DEFAULT  UND __aeabi_memset
    36: 00000000     0 FUNC    GLOBAL DEFAULT  UND __android_log_write
    37: 00000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_allocate_e[...]
    38: 00000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_begin_catch
    39: 00000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_end_catch
    40: 00000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_free_exception
    41: 00000000     0 FUNC    GLOBAL DEFAULT  UND __cxa_throw
    42: 00000000     0 FUNC    GLOBAL DEFAULT  UND __gxx_personality_v0
    43: 00000000     0 FUNC    GLOBAL DEFAULT  UND __aeabi_memclr
    44: 00000000     0 FUNC    GLOBAL DEFAULT  UND __gnu_Unwind_Fin[...]
    45: 00007c08    16 OBJECT  WEAK   DEFAULT   17 _ZTTNSt6__ndk119[...]
    46: 00003319    12 FUNC    WEAK   DEFAULT   13 _ZTv0_n12_NSt6__[...]
    47: 000062b6    50 OBJECT  WEAK   DEFAULT   16 _ZTSNSt6__ndk113[...]
    48: 000034d9    88 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    49: 00003739    14 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    50: 00007cc4    64 OBJECT  WEAK   DEFAULT   17 _ZTVNSt6__ndk115[...]
    51: 000062a0    22 OBJECT  WEAK   DEFAULT   16 _ZTSN6Zhenxi10Lo[...]
    52: 000034d1     4 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    53: 00003709    32 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    54: 00007cb8    12 OBJECT  WEAK   DEFAULT   17 _ZTINSt6__ndk115[...]
    55: 00003d03    92 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk113ba[...]
    56: 00007c18    40 OBJECT  WEAK   DEFAULT   17 _ZTCNSt6__ndk119[...]
    57: 00007c40    12 OBJECT  WEAK   DEFAULT   17 _ZTINSt6__ndk19b[...]
    58: 00007bd8     8 OBJECT  WEAK   DEFAULT   17 _ZTIN6Zhenxi10Lo[...]
    59: 000032d9    12 FUNC    WEAK   DEFAULT   13 _ZTv0_n12_NSt6__[...]
    60: 00003085   128 FUNC    GLOBAL DEFAULT   13 JNI_OnLoad
    61: 00003ad9    96 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk112ba[...]
    62: 00003a91    72 FUNC    WEAK   DEFAULT   13 _ZNKSt6__ndk115b[...]
    63: 00003391    40 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    64: 00003a07    28 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk119ba[...]
    65: 000063a3    52 OBJECT  WEAK   DEFAULT   16 _ZTSNSt6__ndk115[...]
    66: 00007c4c    24 OBJECT  WEAK   DEFAULT   17 _ZTINSt6__ndk113[...]
    67: 00007cb0     8 OBJECT  WEAK   DEFAULT   17 _ZTINSt6__ndk115[...]
    68: 0000355f    34 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    69: 00003a25   108 FUNC    WEAK   DEFAULT   13 _ZN6Zhenxi10LogM[...]
    70: 00003d61   168 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk113ba[...]
    71: 00003185    88 FUNC    GLOBAL DEFAULT   13 Java_com_kejian_[...]
    72: 000031dd     4 FUNC    GLOBAL DEFAULT   13 Java_com_kejian_[...]
    73: 00003149    60 FUNC    WEAK   DEFAULT   13 _ZN6Zhenxi10LogM[...]
    74: 00003531    46 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    75: 00003bfd    40 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk113ba[...]
    76: 00003581    76 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    77: 000032cb    14 FUNC    WEAK   DEFAULT   13 _ZTv0_n12_NSt6__[...]
    78: 00007be0    40 OBJECT  WEAK   DEFAULT   17 _ZTVNSt6__ndk119[...]
    79: 00003105    28 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk1lsIN[...]
    80: 000033c9     2 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    81: 00003747    14 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    82: 00007c64    12 OBJECT  WEAK   DEFAULT   17 _ZTINSt6__ndk119[...]
    83: 00003629   224 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    84: 00007bc8    16 OBJECT  WEAK   DEFAULT   17 _ZTVN6Zhenxi10Lo[...]
    85: 00003b39   196 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk124__[...]
    86: 00006315    73 OBJECT  WEAK   DEFAULT   16 _ZTSNSt6__ndk119[...]
    87: 000032b5    22 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk113ba[...]
    88: 00007c70    64 OBJECT  WEAK   DEFAULT   17 _ZTVNSt6__ndk115[...]
    89: 00003729    16 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    90: 000033cd   236 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    91: 00003e21    82 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk112ba[...]
    92: 000033cb     2 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    93: 00003797    18 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk111ch[...]
    94: 000039f5    18 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk111ch[...]
    95: 00003831   168 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk112ba[...]
    96: 00003755     6 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    97: 000062e8    45 OBJECT  WEAK   DEFAULT   16 _ZTSNSt6__ndk19b[...]
    98: 0000375b     6 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
    99: 000033b9    16 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
   100: 00003309    16 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk119ba[...]
   101: 00003781    22 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk112ba[...]
   102: 00003349    48 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
   103: 00008004     0 NOTYPE  GLOBAL DEFAULT  ABS _edata
   104: 00003325    12 FUNC    WEAK   DEFAULT   13 _ZTv0_n12_NSt6__[...]
   105: 000035cd    92 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
   106: 000032a5    16 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk113ba[...]
   107: 00008005     0 NOTYPE  GLOBAL DEFAULT  ABS _end
   108: 00003121    40 FUNC    WEAK   DEFAULT   13 _ZN6Zhenxi10LogM[...]
   109: 00003245    16 FUNC    WEAK   DEFAULT   13 _ZN6Zhenxi10LogM[...]
   110: 0000393d    36 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk112ba[...]
   111: 0000635e    69 OBJECT  WEAK   DEFAULT   16 _ZTSNSt6__ndk115[...]
   112: 000037a9   104 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk112ba[...]
   113: 000038d9    16 FUNC    WEAK   DEFAULT   13 _ZNKSt6__ndk121_[...]
   114: 000032e5    36 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk119ba[...]
   115: 00003961   120 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk112ba[...]
   116: 00008004     0 NOTYPE  GLOBAL DEFAULT  ABS __bss_start
   117: 00003761     6 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
   118: 000034d5     4 FUNC    WEAK   DEFAULT   13 _ZNSt6__ndk115ba[...]
```

第一列 num 表示符号表数组的下标; 第二列 value 就是符号值, 即 st_value; 第三列 Size 表示符号大小, 即 st_size; 第四列和第五列分别为符号类型和绑定信息, 即 st_info 的低 4 位和高 28 位; 第六列 Vis 目前没用; 第七列 Ndx 就是 st_shndx, 表示符号所属的段; 最后一列是符号名称, 即 st_name;

### [](#特殊符号 "特殊符号")特殊符号

使用 ld 作为链接器来链接生产科执行文件时, 它会为我们定义很多特殊的符号, 这些符号我们可以直接声明并使用, 我们称这些链接器产生的符号为特殊符号;

*   `__executable_start`: 该符号是程序起始地址, 不是入口地址, 是程序最开始的地址
*   `__etext`或者`_etext`或者`etext`: 该符号是代码段结束地址, 也就是代码段最末尾地址
*   `_edata`或者`edata`: 该符号是数据段结束地址, 即数据段最末尾的地址
*   `_end`或`end`: 该符号位程序结束地址

### [](#符号改编和函数签名 "符号改编和函数签名")符号改编和函数签名

函数签名, 包含了一个函数的信息, 包括函数名, 它的参数类型, 它所在的类和名称空间及其他信息; 函数签名用于识别不同的函数; 函数的名字只是函数签名的一部分, 同名函数因为参数类型不同, 所在的类不同, 名称空间不同等都会产生不同的函数签名;

在编译器和链接器处理符号时, 会使用符号改编 (name mangline) 对函数和变量进行改编, 形成符号名; 也就是说 C++ 的源代码编译后的目标文件中所使用的符号名是相应的函数和变量经过修饰之后的名称;

可以使用 `c++filt -n 修饰后的符号名`来还原原始的函数名或者变量名;

### [](#extern-“C” "extern “C”")extern “C”

C++ 为了和 C 兼容, 在符号管理上, C++ 有一个用来声明或定义一个 C 的符号的 `extern "C"`关键字用法

```
extern "C" {
    int func(int);
    int var;
}
```

C++ 编译器会将在`extern "C"`的大括号内的代码当做 C 语言的代码处理; 所以此时 C++ 的符号改编机制就不会起作用;

我们可以通过 C++ 的宏 `__cplusplus`来判断当前编译单元是不是 C++ 代码;

```
#ifdef __cpluscplus
extern "C" {
#endif

void *memset {void *, int, size_t};
#ifdef __cplusplus
}
#endif
```

如果当前编译单元是 C++ 代码, 那么 memset 会在 `extern "C"`里面被声明; 如果是 C 代码, 则直接声明;

### [](#弱符号和强符号 "弱符号和强符号")弱符号和强符号

符号重复定义: 多个目标文件中含有相同名字全局符号的定义, 这些目标文件链接的时候将会出现符号重复定义的错误; 比如: 目标文件 A 和目标文件 B 都定义了一个全局整型变量 global, 并将它们都初始化了, 那么链接器将 A 和 B 进行链接的时候会报错;

在 C/C++ 中默认函数和初始化了的全局变量为强符号, 未初始化的全局变量为弱符号; 可以通过`__attribute__((weak))`来将任何一个强符号转化为弱符号; 这里的强符号和弱符号是针对定义来说的, 不是针对符号的引用;

```
extern int ext; 

int weak; 
int strong = 1; 

__attribute__((weak)) weak2 = 2; 

int main(){
    return 0;
}
```

*   weak 和 weak2 是弱符号
*   strong 和 main 是强符号
*   ext 不是强符号也不是弱符号, 是一个外部变量的引用

全局符号处理规则:

1.  不允许强符号被多次定义 (不同的文件中不能有同名的强符号); 如果有多个强符号定义, 则链接器报符号重复定义错误
2.  如果一个符号在某个目标文件中是强符号, 在其他文件中都是弱符号, 那么选择强符号
3.  如果一个符号在所有目标文件中都是弱符号, 那么选择其中占用空间最大的一个; 比如目标文件 A 定义全局变量 global 为 int 型, 占 4 个字节; 目标文件 B 定义 global 为 double 型, 占 8 个字节; 那么目标 A 文件和目标 B 文件链接后, 符号 global 占 8 字节;

### [](#弱引用和强引用 "弱引用和强引用")弱引用和强引用

如果没有找到该符号的定义, 链接器就会报未定义错误, 这种被称为强引用; 与之相应的还有一种弱引用, 在处理弱引用时, 如果该符号有定义, 则链接器将该符号的引用决议; 如果该符号未被定义, 则链接器对于该引用不会报错;

在 GCC 中, 我们可以通过`__attribute__((weakref))`这个扩展关键字来声明对一个外部函数的引用为弱引用; 比如:

```
__attribute__((weakref)) void foo();

int main()
{
    foo();
}
```

将上面代码编译成一个可执行文件, GCC 并不会报链接错误; 但是运行这个可执行文件, 会发生运行错误; 因为 main 函数调用 foo 函数时, foo 函数的地址为 0, 于是发生了非法地址访问的错误; 可以改为:

```
__attribute__((weakref)) void foo();

int main(){
    if (foo) foo();
}
```

这种弱符号和弱引用对于库来说十分有用, 比如库中定义的弱符号可以被用户定义的强符号所覆盖, 从而使得程序可以使用自定义版本的库函数; 或者程序可以对某些扩展功能模块的引用定义为弱引用; 当我们将扩展模块与程序链接起来时, 功能模块就可以正常使用了, 如果我们去掉了某些功能模块, 那么程序也可以正常链接, 只是缺少了相应的功能; 这使得程序的功能更加容易裁剪和组合;

### [](#调试信息 "调试信息")调试信息

目标文件里面还有可能保存的是调试信息; 几乎所有现代的编译器都支持源代码级别的调试, 比如我们可以在函数里面设置断点, 可以监视变量变化, 可以单步等;

在 GCC 编译时加上 - g 参数, 编译器就会在产生的目标文件里面加上调试信息. 通过 readelf 可以查看到目标文件中增加了很多 debug 相关的段;

现在的 ELF 文件采用 DWARF 的标准的调试信息格式; 在 Linux 下, 可以使用 strip 命令来去掉 ELF 文件中的调试信息;

`strip foo`

[](#静态链接 "静态链接")静态链接
--------------------

有两个目标文件, 如何将它们链接起来形成一个可执行文件, 这就是链接的核心内容: 静态链接;

```
extern int shared;

int main()
{
    int a = 100;
    swap( &a, &shared );
}
```

```
int shared = 1;

void swap(int* a, int* b){
    *a ^= *b ^= *a ^= *b;
}
```

假设程序只有两个模块 a.c 和 b.c; 使用 gcc 将 a.c 和 b.c 分别编译成目标文件 a.o 和 b.o; `gcc -c a.c b.c`

经过编译获得了 a.o 和 b.o 两个目标文件, b.c 中一共定义了两个全局符号, 一个是全局变量 shared, 和一个函数 swap; a.c 里面定义了一个全局符号就是 main; 模块 a.c 里面引用了 b.c 里面的 swap 和 shared; 接下来就是将两个目标文件链接在一起形成一个可执行文件 ab;

### [](#空间与地址分配 "空间与地址分配")空间与地址分配

对于链接器来说, 整个链接过程中, 它就是将几个输入目标文件加工后合并成一个输出文件; 在这个案例中, 输入文件就是 a.o 和 b.o; 输出就是可执行文件 ab; 可执行文件的代码段和数据段都是由输入的目标文件合并而来的, 这就产生了问题: 对于多个输入的目标文件, 链接器如何将它们的各个段合并到输出文件中? 或者说, 输出文件中的空间是如何分配给输入文件的?

#### [](#按序叠加 "按序叠加")按序叠加

最简单的方案就是按序叠加, 将输入的目标文件依照次序叠加起来;

但是这样会产生问题, 在有多个输入文件的情况下, 输出文件会产生多个零散的段; 比如一个稍大的应用程序可能会有数百个目标文件, 如果每个目标文件都分别有. text 段和. data 段和. bss 段, 最后输出文件将会有很多个零散的段; 这种做法很浪费空间, 因为每个段都必须要有一定的地址和空间对齐要求, 比如对于 x86 硬件来说, 段的装载地址和空间的对齐单位是页, 也就是 4096 字节, 那么就是说如果一个段的长度只有 1 个字节, 它也要在内存中占用 4096 字节; 这样会造成空间浪费;

#### [](#相似段合并 "相似段合并")相似段合并

更实际的做法是将相同性质的段合并到一起, 比如将所有输入文件的. text 合并到输出文件的. text 段; 将所有输入文件的. data 段, .bss 段全部合并到输出文件的. data 段和. bss 段中;

.bss 段在前面说了, 在目标文件和可执行文件中都不占用文件的空间, 但是它在装载的时候占用地址空间, 所以链接器在合并各个段的同时, 也将. bss 段合并, 并且分配了虚拟空间;

#### [](#空间和地址 "空间和地址")空间和地址

空间和地址有两个含义:

1.  第一个是输出的可执行文件的空间
2.  第二个是在装载后的虚拟地址中的虚拟地址空间

对于有实际数据的段, 比如 .text 和 .data 来说, 他们在文件中和在虚拟地址中都要分配空间, 因为它们在这两者中都存在; 而对于. bss 这样的段来说, 分配空间的意义只局限于虚拟地址空间, 因为它在文件中并没有内容; 事实上, 我们在这里谈的空间只关注于虚拟地址空间的分配, 因为这个关系到链接器后面关于地址计算的步骤, 而可执行文件本身的空间分配与链接过程关系并不大;

现在链接器空间分配策略都是采用相似段合并的方式, 并且都使用两步链接的方法:

1.  空间与地址分配
    1.  扫描所有输入目标文件, 并且获得他们的各个段的长度, 属性和位置
    2.  将输入文件中的符号表中的所有符号定义和符号引用都收集起来, 统一放到一个全局符号表中
    3.  将输入文件的所有段合并, 并计算出申诉出文件中各个段合并后的长度和位置, 建立映射关系
2.  符号解析和重定位
    1.  从第一步获取到输入文件中端的数据, 重定位信息, 并且进行符号解析与重定位, 调整代码中的地址等;
    2.  符号解析和重定位是链接过程的核心

使用 ld 将 a.o 和 b.o 链接起来: `ld a.o b.o -e main -o ab`; -e main 表示将 main 作为程序的入口, ld 链接器默认的程序入口为_start; -o ab 表示链接输入文件名为 ab, 默认为 a.out;

![](https://kevinspider-1258012111.cos.ap-shanghai.myqcloud.com/2021-07-20-053440.png)

VMA Virtual Memory Address 表示虚拟地址, LMA 表示 Load Memory Address, 即加载地址, 正常情况下这两个值应该是一样的, 但是在有些嵌入式系统中, 特别是那些程序放在 ROM 的系统中时, LMA 和 VMA 是不相同的; 这里只需要关注 VMA 即可;

#### [](#符号地址的确定 "符号地址的确定")符号地址的确定

在第一步的扫描和空间分配阶段, 链接器按照前面介绍的空间分配方法进行分配, 这时候输入文件中的各个段在链接后的虚拟地址就已经确定了; 比如. text 段起始地址是: 0x08048094; .data 段的起始地址为 0x0849108;

当前面一步完成之后, 链接器开始计算各个符号的虚拟地址; 因为各个符号在段内的相对位置是固定的, 所以这个时候其实 main, shared 和 swap 的地址就已经确定了; 只不过链接器要给每个符号加上一个偏移量, 让它们能够调整到正确的虚拟地址; 比如假设 a.o 中的 main 函数相对于 a.o 的 .text 段的偏移是 X; 通过 objdump 看到 main 方法在 a.o 中. text 段的最开始, 但是经过链接合并之后, 所以 main 这个符号在最终的输出文件中的地址为 0x08048094 + 0; 通过这样的方法可以获取到所有符号的地址;

### [](#符号解析与重定位 "符号解析与重定位")符号解析与重定位

#### [](#重定位 "重定位")重定位

在完成了空间和地址分配之后, 链接器就进入了符号解析与重定位的步骤, 这也是静态链接的核心内容; 在分析符号解析和重定位之前, 看一下原始的 a.o 中是如何处理这两个外部符号 shared 变量和 swap 函数的:

![](https://kevinspider-1258012111.cos.ap-shanghai.myqcloud.com/2021-07-20-060631.png)

*   程序的代码里面使用的都是虚拟地址, 这里 main 的起始地址时 0x00000000; 因为在未进行前面提到的空间分配之前, 目标文件代码中的起始地址以 0x00000000 开始, 等到空间分配完成之后, 各个函数才会确定自己在虚拟空间中的位置;
*   在 a.c 的源代码被编译成目标文件时, 编译器并不知道 shared 和 swap 的地址, 因为它们定义在其他目标文件中, 所以编译器就暂时把地址 0 看做是 shared 的地址; 0xFFFFFFFC 也是一个临时的假地址, 用作 swap 的临时地址, 因为编译器不知道 swap 的真正地址;
*   编译器把这两条指令的地址部分暂时用地址 0x00000000 和 0xFFFFFFFC 代替, 把真正的地址计算工作留给了链接器; 通过前面的空间和地址分配, 链接器完成地址和空间分配之后, 就可以知道所有符号的虚拟地址了, 那么链接器就可以根据符号的地址对每个需要重定位的指令进行修正;

看一下 ab 代码中, main 函数中的两个需要重定位的入口已经被修正: shared 和 swap 都已经获得了真实地址;

![](https://kevinspider-1258012111.cos.ap-shanghai.myqcloud.com/2021-07-20-061650.png)

#### [](#重定位表-1 "重定位表")重定位表

链接器如何知道哪些指令是需要被调整的? 事实上在 ELF 文件中, 有一个叫重定位表的结构专门用来保存这些与重定位相关的信息; 对于重定位的 ELF 的文件来说, 它必须包含有重定位表, 用来表述如何修改相应的段里的内容; 每一个需要被重定位的 ELF 段都有一个对应的重定位表, 而一个重定位表往往就是 ELF 文件中的一个段, 所以其实重定位表也可以叫做重定位段; 比如代码段 .text 如果要被重定位, 那么会有一个相应的 .rel.text 的段保留了代码段的重定位表; 如果代码段. data 有要被重定位的地方, 会有一个相应的 .rel.data 的段保存数据段的重定位表;

![](https://kevinspider-1258012111.cos.ap-shanghai.myqcloud.com/2021-07-20-062648.png)

使用 `objdump -r` 可以查看重定位表, 可以看到 a.o 中所有引用了外部符号的地址; 每一个要被重定位的地方叫做重定位入口; a.o 中有两个需要被重定位的重定位入口, 重定位入口的 OFFSET 偏移表示该入口在要被重定位的段中的位置; RELOCATION RECORDS FOR [.text] 表示这个重定位表示代码段的重定位表; 所以偏移表示代码段中需要被调整的位置, 对照前面反汇编的结构, 0x1c 和 0x27 分别就是 mov 指令和 call 指令的地址部分;

重定位表的结构也很简单, 是一个 Elf32_Rel 结构的数组, 每个数组元素对应一个重定位入口, 定义如下:

```
typedef struct {
    Elf32_Addr r_offset;
    Elf32_Word r_info;
}Elf32_Rel;
```

1.  r_offset: 重定位入口的偏移, 对于可重定位文件来说, 这个值就是重定位入口所要修正的位置的第一个字节相对于段起始的偏移; 对于可执行文件或者共享对象文件来说, 这个值就是该重定位入口所要修正的位置的第一个字节的虚拟地址;
2.  r_info: 重定位入口的类型和符号, 这个成员的低 8 位表示重定位入口的类型, 高 24 位表示重定位入口的符号在符号表中的下标;

#### [](#符号解析 "符号解析")符号解析

链接是因为我们目标文件中用到的符号被定义在其他目标文件里, 所以要将它们链接进来; 其实在前面重定向的过程中, 也伴随着符号的解析过程; 重定位的过程中, 每个重定位的入口都是对一个符号的引用; 当链接器需要对某个符号的引用进行重定位时, 它就要确定这个符号的目标地址; 这个时候链接器就会去查找由所有输入文件符号表组成的全局符号表, 找到相应的符号后进行重定位;

![](https://kevinspider-1258012111.cos.ap-shanghai.myqcloud.com/2021-07-20-065935.png)

*   Global 类型的符号除了 main 是定义在代码段中的, 其他两个 shared 和 swap 都是 UND, 即 undefind 未定义类型;
*   这种未定义的符号都是因为该目标文件中有关于它们的重定位项;
*   在链接器扫描完所有的输入目标文件后, 所有这些未定义的符号都应该能够在全局符号表中找到, 否则链接器就会报未定义错误;

### [](#COMMON-块 "COMMON 块")COMMON 块

链接器本身不支持符号的类型, 即变量类型对于链接器来说是透明的, 它只知道一个符号的名字, 并不知道类型是否一致; 以下分三种情况讨论类型不一致的的时候链接器如何处理:

1.  两个或两个以上的强符号类型不一致 –> 直接报错, 符号多重定义错误
2.  有一个强符号, 其他都是弱符号, 出现类型不一致 –> 以强符号所占空间为准
3.  两个或者两个以上的弱符号类型不一致 –> 以输入文件中占用空间最大的那个为准

使用 COMMON 块的方法是一种取巧的行为, 因为编译器和链接器允许不同类型的弱符号存在, 但最本质的原因还是链接器不支持符号类型, 即链接器无法判断各个符号的类型是否一致;

编译器为什么使用 COMMON 块来标记弱符号, 而不是将弱符号当做局部变量在. bss 段中分配空间来处理?

1.  编译器不能确定弱符号最终的大小; 因为如果编译器包含了弱符号 (未初始化的全局变量就是典型的弱符号). 那么该符号最终占用的空间大小此时还是未知的, 有可能其他编译单元中该符号所占用的空间比本编译单元该符号所占用空间要大, 所以此时编译器并不能为该弱符号在. bss 段中分配空间;
2.  链接器在链接过程中可以确定弱符号的大小, 因为当链接器读取所有输入目标文件后, 任何一个弱符号的最终大小都可以确定, 所以它可以在最终输出文件的 BSS 段为其分配空间
3.  总体来看, 未初始化全局变量最终还是被放在 bss 段中, 只是是由链接器将其放入 bss 段的;

开发人员可以通过 `__attribute__((nocommon));`扩展来让未初始化的全局变量不以 COMMON 块的形式处理

```
int global __attribute__((nocommon));
```

一旦一个未初始化的全局变量是不以 COMMON 块的形式存在的, 那么它就相当于一个强符号, 如果其他目标文件中还有一个同名的强符号定义, 那么链接时就会发成符号重复定义错误;

[](#可执行文件的装载与进程 "可执行文件的装载与进程")可执行文件的装载与进程
-----------------------------------------

1.  什么是进程的虚拟地址空间?
2.  为什么进程要有自己独立的虚拟地址空间?
3.  装载的几种方式?
4.  虚拟地址空间分布情况, 代码块, 数据段, bss 段, 堆, 栈分别在进程地址空间中怎么分布, 它们的位置和长度如何决定?

### [](#进程虚拟空间地址 "进程虚拟空间地址")进程虚拟空间地址

每个程序运行起来以后, 它都将拥有自己独立的虚拟地址空间, 这个虚拟地址空间的大小由计算机的硬件平台决定, 具体说就是由 CPU 的位数决定; 硬件决定了地址空间的最大理论上限, 即硬件的寻址空间大小; 比如 32 位的硬件平台决定了虚拟地址空间的地址为 0 ~ 2^32-1, 也就是 0x0000 0000 ~ 0xFFFF FFFF, 也就是常说的 4G 虚拟空间大小; 而 64 位的硬件平台具有 64 位寻址能力, 它的虚拟地址空间达到了 2^64 字节, 也就是 0x0000000000000000~0xFFFFFFFFFFFFFFFF, 总共是 17179869184GB;

一般来说, 我们可以通过 C 语言指针大小的位数来判断虚拟空间的大小, C 语言指针大小的位数与虚拟空间的位数相同, 如 32 位平台下面指针为 32 位, 即 4 个字节; 64 位平台下的指针为 64 位, 即 8 个字节;

32 位下, 程序在运行的时候不是完全使用 4GB 虚拟地址空间, 4GB 虚拟地址空间被划分成两部分, 其中操作系统本身要用去一部分, 从地址 0xC0000000 到 0xFFFFFFFF 共 1GB; 剩下的从 0x00000000 到 0xBFFFFFFF 共 3GB 的空间留给进程使用;

从原则上说我们的进程最多可以使用 3GB 的虚拟地址空间, 但是并不是进程只能使用 3GB 的空间, 这个空间如果理解成虚拟地址空间的话, 那么 32 位的 cpu 只能使用 32 位的指针, 最大寻址范围是 0 到 4GB; 如果这个空间理解成计算机的内存空间, 那么其实可以使用更大的空间, 因为像 intel 从 1995 年开始就采用了 36 位的物理地址, 也就是可以访问 64GB 的物理内存; intel 在扩展了 36 位地址线之后, 修改了页映射方式, 使得新的映射方式可以访问到更多的物理内存, 这种地址扩展方式被叫做 PAE;

### [](#装载的方式 "装载的方式")装载的方式

静态装载: 将程序运行所需要的指令和数据全部都装入内存中, 这样程序就可以顺利运行了; 但是很多情况下, 程序所需要的内存是远大于物理内存的数量的, 所以静态装入的方式现在很少使用;

动态装载: 覆盖装入和页映射是两种很典型的动态装载方式, 动态装载的思想就是程序要用到哪个模块, 就将哪个模块装入内存, 如果暂时不用, 就不装入内存, 存放在磁盘中;

#### [](#覆盖装入 "覆盖装入")覆盖装入

在没有发明虚拟存储之前应用广泛, 现在几乎被淘汰了;

覆盖装入的方法将挖掘内存潜力的任务交给了程序员, 程序员在编写程序的时候必须手动将程序划分成若干个块, 然后编写一个小的辅助代码来管理这些模块应该何时驻留在内存何时被替换掉; 这个小的辅助代码就是覆盖管理器; 最简单的案例, 一个主程序 main, main 会分别调用模块 A 和模块 B; 但是 A 和 B 之间不会相互调用, 那么我们可以用覆盖装入, 让模块 A 和模块 B 在内存中相互覆盖, 即这两个模块共享块内存区域; 当 main 调用模块 A 的时候, 覆盖管理器保证将模块 A 从文件中读入内存, 当 main 调用模块 B 时, 则覆盖管理器将 B 从文件中读入内存, 因为此时 A 不再使用, 所以 B 会装入原先 A 所占用的内存空间;

#### [](#页映射 "页映射")页映射

页映射是虚拟存储机制的一部分, 它随着虚拟存储的发明而诞生; 与覆盖装入的原理相似, 页映射也不是一下子就把程序的所有数据和指令都装入内存, 而是将内存和所有磁盘中的数据和指令按照页 Page 为单位划分成若干个页; 以后所有的装载和操作的单位都是页; 以目前的情况, 硬件规定的页大小都是 4096 字节, 8192 字节, 2MB 和 4MB 等;

为了演示页映射的基本机制, 假设我们的 32 位机器有 16KB 的内存, 每个页大小为 4096 字节, 则一共有 4 个页;

![](https://kevinspider-1258012111.cos.ap-shanghai.myqcloud.com/2021-07-21-021059.png)

假设所有的指令和数据总和有 32KB, 那么程序总共被分为 8 个页, 我们给他们编号为 P0-P7; 内存 16KB, 程序总共 32KB, 所以无法同时将 32KB 的程序装入内存, 我们将按照动态装入的原理来进行整个装入过程;

*   如果程序刚开始执行的入口在页 P0 中, 此时装载管理器发现程序 P0 不在内存中, 于是将内存 F0 分配给 P0, 将 P0 内容装入 F0;
*   运行一段时间后, 程序发现需要使用 P5, 于是装载器将 P5 装入 F1,
*   当程序用到 P3 和 P6 的时候, 它们分别被装入 F2 和 F3;

![](https://kevinspider-1258012111.cos.ap-shanghai.myqcloud.com/2021-07-21-021608.png)

当要使用到新的程序页的时候, 就需要使用内存页来加载这个新的程序页; 比如如果现在要使用到 P4, 则必须放弃内存页中原有的内容来装载 P4; 我们可以依据先进先出的原则, 放弃 F0 原有的内容, 用 F0 来加载 P4; 也可以根据最少使用算法 LUR, 发现哪个使用的较少, 可以将对应的内存页释放来加载 P4;

### [](#从操作系统角度看可执行文件的装载 "从操作系统角度看可执行文件的装载")从操作系统角度看可执行文件的装载

从上面的页映射的动态装入的方式可以看出, 可执行文件中的页可能被装入内存中的任意页; 在虚拟存储中, 现代硬件的 MMU 都提供了地址转换的功能, 有了硬件的地址转换和页映射机制, 操作系统动态加载可执行文件的方式和静态加载有了很大的区别;

#### [](#进程的建立 "进程的建立")进程的建立

从操作系统的角度来看, 一个进程最关键的特征是它拥有独立的虚拟地址空间, 这让它有别于其他进程; 创建一个进程, 然后装载相应的可执行文件并且执行在有虚拟存储的情况下, 最开始只要做三件事:

1.  创建一个独立的虚拟地址空间
2.  读取可执行文件头, 并且建立虚拟空间与可执行文件的映射关系
3.  将 CPU 的指令寄存器设置为客户自行文件的入口地址, 启动运行

首先是创建虚拟地址空间; 虚拟地址空间由一组页映射函数将虚拟空间的各个页映射至相应的物理空间, 所以建立虚拟空间实际上并不是创建空间空间, 而是创建映射函数所需要的相应的数据结构; Linux 下创建虚拟地址空间实际上只是分配了一个页目录 Page Directory 就可以了, 甚至都不需要设置页映射关系, 这些映射关系等到后面程序发生页错误的时候再设置;

读取可执行文件头, 并且建立虚拟空间与可执行文件的映射关系: 上一步的页映射关系函数是对虚拟空间到物理内存的映射关系; 而这一步所做的是虚拟空间和可执行文件的映射关系; 当程序发生页错误的时候, 操作系统将从物理内存里面分配一个物理页, 然后将该缺页从磁盘中读取到内存中, 再设置缺页的虚拟页和物理页的映射关系; 当操作系统捕获到缺页错误时, 它应当知道程序当前所需要的页在可执行文件中的哪个位置, 这就是虚拟空间和可执行文件之间的映射关系;

将 CPU 指令寄存器设置成可执行文件入口, 启动运行; 操作系统通过设置 CPU 的指令寄存器将控制权交给进程, 由此进程开始执行; 从进程的角度来看就是操作系统执行了一个跳转指令, 直接跳转到可执行文件的入口地址; 这个地址就是 ELF 文件头中保存的入口地址;

#### [](#页错误 "页错误")页错误

上面的步骤执行完成后, 其实可执行文件的真正指令和数据都没有被装入内存中; 操作系统只是通过可执行文件头部的信息建立起可执行文件和进程虚拟空间之间的映射关系;

假设上面的例子, 程序的入口地址是 0x08048000, 即刚好是. text 段的起始地址, 当 CPU 开始打算执行这个地址的指令时, 发现页面 0x08048000~08049000 是一个空页面, 于是它就认为这个是一个页错误; CPU 将控制权交给操作系统, 操作系统通过前面我们装载过程中提到的第二步建立的虚拟空间页和 ELF 文件映射的数据结构, 查询出空页面所在的 VMA 虚拟内存区域 (virtual memory areas), 计算出相应的页面在可执行文件中的偏移, 然后再物理内存中分配一个物理页面, 将进程中该虚拟页与分配的物理页之间建立映射关系, 然后把控制权交给进程, 进程从刚才页错误的位置重新开始执行;

随着进程的执行, 页错误会不断产生, 操作系统也会为进程分配相应的物理页面来满足进程执行的需求;

![](https://kevinspider-1258012111.cos.ap-shanghai.myqcloud.com/2021-07-21-033123.png)

### [](#进程虚拟空间分布 "进程虚拟空间分布")进程虚拟空间分布

#### [](#ELF-文件链接视图和执行视图 "ELF 文件链接视图和执行视图")ELF 文件链接视图和执行视图

ELF 文件被映射的时候, 是以系统的页长度作为单位的, 那么每个段在映射时的长度应该是系统页长度的整数倍; 如果不是, 那么多余部分也会占用一个一个页; 一般的 ELF 文件都有十几个段, 那么内存空间就会产生大量的浪费;

操作系统一般只关心一些和装载相关的问题, 最重要的就是段的权限, 可读, 可写, 可执行; ELF 文件中, 段的权限往往只有为数不多的几种组合, 基本上是三种:

1.  以代码段为代表的权限为可读可执行的段
2.  以数据段和 bss 段为代表的权限为可读可写的段
3.  以只读数据段为代表的权限为只读的段

对于相同权限的段, 把它们合并到一起当做一个段进行映射; 比如有两个段分别为 .text 和 .init, 他们分别是程序的可执行代码和初始化代码, 并且它们的权限相同, 都是可读并且可执行的; 假设. text 为 4097 字节, .init 为 512 字节, 这两个段分别映射的话就要占用三个页面; 但是如果将它们合并成一起映射的话, 就只要占据两个页面;

![](https://kevinspider-1258012111.cos.ap-shanghai.myqcloud.com/2021-07-21-060209.png)

ELF 可执行文件引入了一个概念叫做 Segment; 一个 Segment 包含一个或者多个属性类似的 Section; 如果将. text 段和. init 段合并在一起看做是一个 Segment 的话, 那么装载的时候会将它们看做是一个整体一起映射, 也就是说映射以后在进程虚拟空间只有一个相对应的 VMA, 而不是两个, 这样做的好处是可以很明显减少页面内部碎片, 从而减少内存空间;

从链接的角度来看, ELF 文件是按 Section 存储的, 从装载的角度来看, ELF 文件又可以按照 Segment 划分;

Segment 的概念实际上是从装载的角度重新划分了 ELF 的各个段, 在将目标文件链接成可执行文件的时候, 链接器会尽量把相同权限属性的段分配在同一空间; 比如可读可执行的段都放在一起, 典型的是代码段; 可读可写的段都放在一起, 典型的是数据段; ELF 把这些属性相似的有连在一起的段叫做 Segment, 而系统正是按照 Segment 而不是 Section 来映射可执行文件的;

使用 `readelf -l <soPath>` 可以查看 ELF 的 Segment;

使用 `readelf -S <soPath>`可以查看 ELF 的 Section;

描述 Section 属性的结构叫做段表, 描述 Segment 的结构叫做程序头 (Program Header), 它描述了 ELF 文件该如何被操作系统映射到进程的虚拟空间;

总体来说, Segment 和 Section 是从不同的角度来划分同一个 ELF 文件; 这个在 ELF 中被称为不同的视图 View; 从 Section 的角度来看 ELF 文件就是链接视图 (Linking View); 从 Segment 的角度来看就是执行视图 (Execution View); 当在 ELF 装载过程中, 段专门指 Segment, 在其他情况下, 段指的是 Section;

ELF 可执行文件中有一个专门的数据结构叫做程序头表, 用来保存 Segment 的信息; 因为 ELF 目标文件不需要被装载, 所以它没有程序头表; 而 ELF 的可执行文件和共享库文件都有, 和段表结构一样, 程序头表也是一个结构体数组, 结构如下:

```
typedef struct {
    ELF32_Word p_type;
    ELF32_Off p_offset;
    ELF32_Addr p_vaddr;
    ELF32_Addr p_paddr;
    ELF32_Word p_filesz;
    ELF32_Word p_memsz;
    ELF32_Word p_flags;
    ELF32_Word p_align;
}Elf32_Phdr;
```

<table><thead><tr><th>成员</th><th>含义</th></tr></thead><tbody><tr><td>p_type</td><td>Segment 的类型, 基本上我们这里只关注 Load 类型的 Segment; Load 类型的常量为 1; 还有几种诸如 Dynamic 和 Interp 在动态链接的时候会碰到</td></tr><tr><td>p_offset</td><td>Segment 在文件中的偏移</td></tr><tr><td>p_vaddr</td><td>Segment 的第一个字节在进程虚拟地址空间中的起始位置; 整个程序头表中, 所有 Load 类型的元素按照 p_vaddr 从小到大排序</td></tr><tr><td>p_paddr</td><td>Segment 的物理装载地址, 一般情况下物理装载地址就是 LMA, p_paddr 的值在一般情况下和 p_vaddr 是一样的;</td></tr><tr><td>p_filesz</td><td>Segment 在 ELF 文件中所占空间的长度, 它的值可能是 0; 因为有可能这个 Segment 在 ELF 文件中不存在</td></tr><tr><td>p_memse</td><td>Segment 在进程虚拟地址空间所占用的长度, 它的值可能是 0;</td></tr><tr><td>p_flags</td><td>Segment 的权限属性, 比如可读 R, 可写 W 和可执行 X</td></tr><tr><td>p_align</td><td>Segment 的对齐属性, 实际对齐字节等于 2 的 p_align 次; 比如 p_align 等于 10, 那么实际的对齐对齐属性就是 2 的 10 次方;</td></tr></tbody></table>

对于 Load 类型的 Segment 来说, p_memsz 的值不可以小于 p_filesz, 否则就是不符合常理的; 如果 p_memsz 的值大于 p_filesz 就表示该 Segment 在内存中所分配的空间大小超过文件实际的大小, 这部分多余的部分会全部填充为 0; 这样做的好处就是, 我们在构造 ELF 可执行文件时, 不需要额外再设立 bss 的 Segment 了, 可以把数据 Segment 的 p_memsz 扩大, 这些额外的部分就是 BSS;

数据段和 bss 的唯一区别就是, 数据段从文件中初始化内容, 而 bss 段的内容全部初始化为 0, 这也就是我们在前面例子中只看到两个 Load 类型的段, 而不是三个, bss 已经合并到数据类型的段里面;

#### [](#堆和栈 "堆和栈")堆和栈

VMA 除了被用来映射可执行文件中的各个 Segment 以外, 它还可以有其他的作用, 操作系统通过 VMA 来对进程的地址空间进行管理; 进程在执行的时候它还需要用到堆和栈等空间; 事实上它们在进程的虚拟空间中的表现也是以 VMA 的形式存在的; 很多情况下, 一个进程中的堆和栈分别都有一个对应的 VMA; 在 Linux 下, 我们可以通过查看 `/proc`来查看进程的虚拟空间分布;

![](https://kevinspider-1258012111.cos.ap-shanghai.myqcloud.com/2021-07-21-070315.png)

第一列是 VMA 的地址范围, 第二列是 VMA 的权限, r 表示可读, w 表示可写, x 表示可执行; p 表示私有, s 表示共享; 第三列是偏移, 表示 VMA 对应的 Segment 在映像文件中的偏移; 第四列表示映像文件所在设备的主设备号和次设备号, 第五列表示映像文件的节点号; 最后一列是映像文件的路径;

可以看到进程中有 5 个 VMA, 只有前两个是映射到可执行文件中的两个 Segment; 另外三个段的文件所在设备主设备号和次设备号及文件节点都是 0, 则表示它们没有映射到文件中, 这种 VMA 叫做匿名虚拟内存区域; Heap 和 Stack 它们大小分别为 140KB 和 88KB; 这两个 VMA 几乎在所有的进程中存在, 我们在 C 语言中最常用的 malloc() 内存分配函数就是从堆里面分配, 堆由系统库管理;

*   代码 VMA, 权限只读, 可执行, 有映像文件
*   数据 VMA, 权限可读可写可执行, 有映像文件
*   堆 VMA, 权限可读写, 可执行, 无映像文件, 匿名, 可以向上扩展
*   栈 VMA, 权限可读写, 不可执行, 无映像文件, 匿名, 可向下扩展

![](https://kevinspider-1258012111.cos.ap-shanghai.myqcloud.com/2021-07-21-071820.png)

### [](#Linux-内核装载-ELF-过程简介 "Linux 内核装载 ELF 过程简介")Linux 内核装载 ELF 过程简介

[](#动态链接 "动态链接")动态链接
--------------------

### [](#为什么要动态链接 "为什么要动态链接")为什么要动态链接

### [](#简单的动态链接案例 "简单的动态链接案例")简单的动态链接案例

### [](#地址无关代码 "地址无关代码")地址无关代码

### [](#延迟绑定 "延迟绑定")延迟绑定

### [](#动态链接相关结构 "动态链接相关结构")动态链接相关结构

### [](#动态链接的步骤和实现 "动态链接的步骤和实现")动态链接的步骤和实现

### [](#显示运行时链接 "显示运行时链接")显示运行时链接

[](#Linux-共享库的组织 "Linux 共享库的组织")Linux 共享库的组织
--------------------------------------------

### [](#共享库版本 "共享库版本")共享库版本

### [](#符号版本 "符号版本")符号版本

### [](#共享库系统路径 "共享库系统路径")共享库系统路径

### [](#共享库查找过程 "共享库查找过程")共享库查找过程

### [](#环境变量 "环境变量")环境变量

### [](#共享库的创建和安装 "共享库的创建和安装")共享库的创建和安装

[](#内存 "内存")内存
--------------

### [](#程序的内存分布 "程序的内存分布")程序的内存分布

### [](#栈与调用惯例 "栈与调用惯例")栈与调用惯例

### [](#堆与内存管理 "堆与内存管理")堆与内存管理

[](#运行库 "运行库")运行库
-----------------

### [](#入口函数与程序初始化 "入口函数与程序初始化")入口函数与程序初始化

### [](#C-C-运行库 "C/C++运行库")C/C++ 运行库

### [](#运行库与多线程 "运行库与多线程")运行库与多线程

### [](#C-全局构造与析构 "C++全局构造与析构")C++ 全局构造与析构

### [](#fread-实现 "fread 实现")fread 实现