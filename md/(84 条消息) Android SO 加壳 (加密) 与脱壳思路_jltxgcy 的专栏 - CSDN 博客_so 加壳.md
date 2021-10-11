> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/jltxgcy/article/details/52205210)

 **0x01 常见的 Android SO 加壳 (加密) 思路**

    **1.1 破坏 Elf Header**

    将 Elf32_Ehdr 中的 e_shoff, e_shnum, e_shstrndx, e_shentsize 字段处理，变为无效值。由于在链接过程中，这些字段是无用的，所以可以随意修改，这会导致 ida 打不开这个 so 文件。

    **1.2 删除 Section Header**

 同样的道理，在链接过程中，Section Header 是没有用到的，可以随意删除，这会导致 ida 打不开这个 so 文件。

    **1.3 有源码加密 Section 或者函数**

    一是对 section 加壳，一个是对函数加壳。参考 [Android 逆向之旅 --- 基于对 so 中的 section 加密技术实现 so 加固](http://blog.csdn.net/jiangwei0910410003/article/details/49962173)，[Android 逆向之旅 --- 基于对 so 中的函数加密技术实现 so 加固](http://blog.csdn.net/jiangwei0910410003/article/details/49966719)。

  **  1.4 无源码加密 Section 或者函数**

    将解密函数放在另一个 so 中，只需保证解密函数在被加密函数执行前执行即可。和其他 so 一样，解密 so 的加载也放在类的初始化 static{} 中。我们可以在解密 so 加载过程中执行解密函数，就能保证被加密函数执行前解密。执行时机可以选在 linker 执行. init_array 时，也可以选在 OnLoad 函数中。当然，解密 so 一定要放在被解密 so 后加载。否则，搜索进程空间找不到被解密的 so。

    详细介绍和代码：[无源码加解密实现 && NDK Native Hook](http://bbs.pediy.com/showthread.php?t=192047) 。

    **1.5 自定义 loader 来加载 SO，即从内存加载 SO**

    我们可以 DIY SO，然后使用我们自定义的 loader 来加载，破解难度又加大了。详解的介绍参考 S[O 文件格式及 linker 机制学习总结 (1)](http://bbs.pediy.com/showthread.php?t=197512)，[SO 文件格式及 linker 机制学习总结 (2)](http://bbs.pediy.com/showthread.php?t=197559)。

 **1.6 在原 so 外面加一层壳**

     packed so 相当于把 loader 的代码插入到原 so 的 init_array 或者 jni_onload 处，然后重新打包成 packed so，加载这个 so，首先执行 init_array 或者 jni_onload，在这里完成对原 so 的解密，从内存加载，并形成 soinfo 结构，然后替换原 packed so 的 soinfo 结构。

 **1.7 llvm 源码级混淆**

    ![](https://img-blog.csdn.net/20160924160335934?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

    Clang (发音为 /klæŋ/) 是 LLVM 的一个编译器前端，它目前支持 C, C++, Objective-C 以及 Objective-C++ 等编程语言。Clang 对源程序进行词法分析和语义分析，并将分析结果转换为 Abstract Syntax Tree ( 抽象语法树 ) ，最后使用 LLVM 作为后端代码的生成器。

    在 Android llvm 源码级混淆比较成熟的是 Safengine。

    如果想自己研究 llvm 源码级混淆，可以参考 [Android LLVM-Obfuscator C/C++ 混淆编译的深入研究](http://blog.csdn.net/wangbaochu/article/details/45370543)，通过修改 NDK 的编译工具，来实现编译混淆。

 **1.8 花指令**

    通过在 C 语言中，内嵌 arm 汇编的方式，可以加入 arm 花指令，迷惑 IDA。

 **1.9 so vmp 保护**

    写一个 arm 虚拟机虚拟执行 so 中被保护的代码，在手机上效率是一个问题。    

 **0x02 对应的脱壳思路**

 **1.1 针对破坏 Elf Header 和删除 Section Header**

 参考 [ELF section 修复的一些思考](http://bbs.pediy.com/showthread.php?t=192874)。

    里面有个知识点，需要说明下：

    从 DT_PLTGOT 可以得到__global_offset_table 的偏移位置。由 got 表的结构知道，__global_offset_table 前是 rel.dyn 重定位结构，之后为 rel.plt 重定位结构，都与 rel 一一对应。我们还是以 libPLTUtils.so 为例，下载地址：[http://download.csdn.net/detail/jltxgcy/9602803](http://download.csdn.net/detail/jltxgcy/9602803)。

    arm-linux-androideabi-readelf -d ~/Public/libPLTUtils.so：

```
Dynamic section at offset 0x2e8c contains 32 entries:
  Tag        Type                         Name/Value
 0x00000003 (PLTGOT)                     0x3fd4
 0x00000002 (PLTRELSZ)                   64 (bytes)
 0x00000017 (JMPREL)                     0xc74
 0x00000014 (PLTREL)                     REL
 0x00000011 (REL)                        0xc24
 0x00000012 (RELSZ)                      80 (bytes)
 0x00000013 (RELENT)                     8 (bytes)
 0x6ffffffa (RELCOUNT)                   7
 0x00000006 (SYMTAB)                     0x18c
 0x0000000b (SYMENT)                     16 (bytes)
 0x00000005 (STRTAB)                     0x51c
 0x0000000a (STRSZ)                      1239 (bytes)
 0x00000004 (HASH)                       0x9f4
 0x00000001 (NEEDED)                     Shared library: [liblog.so]
 0x00000001 (NEEDED)                     Shared library: [libdl.so]
 0x00000001 (NEEDED)                     Shared library: [libstdc++.so]
 0x00000001 (NEEDED)                     Shared library: [libm.so]
 0x00000001 (NEEDED)                     Shared library: [libc.so]
 0x0000000e (SONAME)                     Library soname: [libPLTUtils.so]
 0x0000001a (FINI_ARRAY)                 0x3e80
 0x0000001c (FINI_ARRAYSZ)               8 (bytes)
 0x00000019 (INIT_ARRAY)                 0x3e88
 0x0000001b (INIT_ARRAYSZ)               4 (bytes)
 0x00000010 (SYMBOLIC)                   0x0
 0x0000001e (FLAGS)                      SYMBOLIC BIND_NOW
 0x6ffffffb (FLAGS_1)                    Flags: NOW
 0x6ffffff0 (VERSYM)                     0xb74
 0x6ffffffc (VERDEF)                     0xbe8
 0x6ffffffd (VERDEFNUM)                  1
 0x6ffffffe (VERNEED)                    0xc04
 0x6fffffff (VERNEEDNUM)                 1
 0x00000000 (NULL)                       0x0
```

    我们看到 PLTGOT 的 value 为 0x3fb4。

    然后 arm-linux-androideabi-readelf -r ~/Public/libPLTUtils.so：

```
Relocation section '.rel.dyn' at offset 0xc24 contains 10 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
00003e80  00000017 R_ARM_RELATIVE   
00003fb4  00000017 R_ARM_RELATIVE   
00003fb8  00000017 R_ARM_RELATIVE   
00003fbc  00000017 R_ARM_RELATIVE   
00003fc0  00000017 R_ARM_RELATIVE   
00003fc8  00000017 R_ARM_RELATIVE   
00003fcc  00000017 R_ARM_RELATIVE   
00004004  00000402 R_ARM_ABS32       00000000   puts
00003fc4  00000915 R_ARM_GLOB_DAT    00000000   __gnu_Unwind_Find_exid
00003fd0  00001f15 R_ARM_GLOB_DAT    00000000   __cxa_call_unexpected

Relocation section '.rel.plt' at offset 0xc74 contains 8 entries:
 Offset     Info    Type            Sym.Value  Sym. Name
00003fe0  00000216 R_ARM_JUMP_SLOT   00000000   __cxa_atexit
00003fe4  00000116 R_ARM_JUMP_SLOT   00000000   __cxa_finalize
00003fe8  00000416 R_ARM_JUMP_SLOT   00000000   puts
00003fec  00000916 R_ARM_JUMP_SLOT   00000000   __gnu_Unwind_Find_exid
00003ff0  00000f16 R_ARM_JUMP_SLOT   00000000   abort
00003ff4  00001116 R_ARM_JUMP_SLOT   00000000   memcpy
00003ff8  00001c16 R_ARM_JUMP_SLOT   00000000   __cxa_begin_cleanup
00003ffc  00001d16 R_ARM_JUMP_SLOT   00000000   __cxa_type_match
```

    我们可以看到 0x00003fb4 恰好是其分界线。

    在 ida 中，是这样的。

```
.got:00003FB4 ; ===========================================================================
.got:00003FB4
.got:00003FB4 ; Segment type: Pure data
.got:00003FB4                 AREA .got, DATA
.got:00003FB4                 ; ORG 0x3FB4
.got:00003FB4 global_fun_ptr  DCD global_fun          ; DATA XREF: Java_com_example_ndkplt_PLTUtils_pltTest+16o
.got:00003FB4                                         ; Java_com_example_ndkplt_PLTUtils_pltTest+18r ...
.got:00003FB8 __aeabi_unwind_cpp_pr0_ptr DCD __aeabi_unwind_cpp_pr0 ; DATA XREF: sub_E54+1Cr
.got:00003FB8                                         ; .text:off_E98o
.got:00003FBC __aeabi_unwind_cpp_pr1_ptr DCD __aeabi_unwind_cpp_pr1 ; DATA XREF: sub_E54+28r
.got:00003FBC                                         ; .text:off_E9Co
.got:00003FC0 __aeabi_unwind_cpp_pr2_ptr DCD __aeabi_unwind_cpp_pr2 ; DATA XREF: sub_E54+34r
.got:00003FC0                                         ; .text:off_EA0o
.got:00003FC4 __gnu_Unwind_Find_exidx_ptr DCD __imp___gnu_Unwind_Find_exidx
.got:00003FC4                                         ; DATA XREF: sub_EA4+8r
.got:00003FC4                                         ; .text:off_F9Co
.got:00003FC8 off_3FC8        DCD aCallTheMethod      ; DATA XREF: sub_EA4+48r
.got:00003FC8                                         ; .text:off_FA0o
.got:00003FC8                                         ; "call the method"
.got:00003FCC off_3FCC        DCD 0x2304              ; DATA XREF: sub_EA4+4Cr
.got:00003FCC                                         ; .text:off_FA4o
.got:00003FD0 __cxa_call_unexpected_ptr DCD __cxa_call_unexpected ; DATA XREF: sub_150C+3A8r
.got:00003FD0                                         ; .text:off_18F8o
.got:00003FD4 _GLOBAL_OFFSET_TABLE_ DCD 0             ; DATA XREF: .plt:00000CBCo
.got:00003FD4                                         ; .plt:off_CC4o
.got:00003FD8                 DCD 0
.got:00003FDC                 DCD 0
.got:00003FE0 __cxa_atexit_ptr DCD __imp___cxa_atexit ; DATA XREF: __cxa_atexit+8r
.got:00003FE4 __cxa_finalize_ptr DCD __imp___cxa_finalize ; DATA XREF: __cxa_finalize+8r
.got:00003FE8 puts_ptr        DCD __imp_puts          ; DATA XREF: puts+8r
.got:00003FEC __gnu_Unwind_Find_exidx_ptr_0 DCD __imp___gnu_Unwind_Find_exidx
.got:00003FEC                                         ; DATA XREF: __gnu_Unwind_Find_exidx+8r
.got:00003FF0 abort_ptr       DCD __imp_abort         ; DATA XREF: abort+8r
.got:00003FF4 memcpy_ptr      DCD __imp_memcpy        ; DATA XREF: memcpy+8r
.got:00003FF8 __cxa_begin_cleanup_ptr DCD __imp___cxa_begin_cleanup
.got:00003FF8                                         ; DATA XREF: __cxa_begin_cleanup+8r
.got:00003FFC __cxa_type_match_ptr DCD __imp___cxa_type_match
.got:00003FFC                                         ; DATA XREF: __cxa_type_match+8r
.got:00003FFC ; .got          ends
```

    我们在 0x00003fb4 地址前，看到了：

```
00004004  00000402 R_ARM_ABS32       00000000   puts
00003fc4  00000915 R_ARM_GLOB_DAT    00000000   __gnu_Unwind_Find_exid
00003fd0  00001f15 R_ARM_GLOB_DAT    00000000   __cxa_call_unexpected
```

    在这个地址后面，看到了：

```
00003fe0  00000216 R_ARM_JUMP_SLOT   00000000   __cxa_atexit
00003fe4  00000116 R_ARM_JUMP_SLOT   00000000   __cxa_finalize
00003fe8  00000416 R_ARM_JUMP_SLOT   00000000   puts
00003fec  00000916 R_ARM_JUMP_SLOT   00000000   __gnu_Unwind_Find_exid
00003ff0  00000f16 R_ARM_JUMP_SLOT   00000000   abort
00003ff4  00001116 R_ARM_JUMP_SLOT   00000000   memcpy
00003ff8  00001c16 R_ARM_JUMP_SLOT   00000000   __cxa_begin_cleanup
00003ffc  00001d16 R_ARM_JUMP_SLOT   00000000   __cxa_type_match
```

    **1.2 针对有源码加密 Section 或者函数**  
    使用 dlopen 来加载 so，返回一个 soinfo 结构体如下：

```
struct soinfo {
    const char name[SOINFO_NAME_LEN]; Elf32_Phdr *phdr; //Elf32_Phdr 实际内存地址 int phnum;
    unsigned entry;
    unsigned base; //SO 起始
    unsigned size; //内存对齐后占用大小
    int unused; // DO NOT USE, maintained for compatibility. unsigned *dynamic; //.dynamic 实际内存地址
    unsigned wrprotect_start; //mprotect 调用 unsigned wrprotect_end;
    soinfo *next; //下一个 soinfo unsigned flags;
    const char *strtab; //.strtab 实际内存地址 Elf32_Sym *symtab; //. symtab 实际内存地址
    //hash 起始位置:bucket – 2 * sizeof(int) unsigned nbucket; //size = nbucket * sizeof(int) unsigned nchain; //size = nchain * sizeof(int) unsigned *bucket;
    unsigned *chain;
    unsigned *plt_got; //对应.dynamic: DT_PLTGOT Elf32_Rel *plt_rel; //函数重定位表
    unsigned plt_rel_count;
    Elf32_Rel *rel; //符号重定位表 unsigned rel_count;
    ....
};
```

    得到这个结构体时，已经执行了 init_array，已经实现了解密。剩下的工作就是如何恢复原 so 了，这部分参考 [ELF section 修复的一些思考](http://bbs.pediy.com/showthread.php?t=192874)和从[零打造简单的 SODUMP 工具](http://bbs.pediy.com/showthread.php?t=194053)。

    **1.3 针对无源码加密 Section 或者函数和自定义 loader 来加载 SO，即从内存加载 SO**

    和针对有源码加密 Section 或者函数类似，但不像原来那样，只要在 ndk 开发中调用 dlopen 即可。从 soinfo 结构体恢复 so 文件的时机，要选择在 Android 源码中，具体时机，如果日后有需要，再做研究。