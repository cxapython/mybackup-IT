> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [nszdhd1.github.io](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/)

> so 加密笔记基础知识一、ELF 文件（更详细可看 elf 讲解） ELF 是 Linux 下的可执行连接格式文件，类似于 windows 下的 PE 文件。

[](#基础知识 "基础知识")基础知识
--------------------

### [](#一、ELF文件 "一、ELF文件")一、ELF 文件

（更详细可看 [elf 讲解](https://blog.csdn.net/jinking01/article/details/105387253)）

ELF 是 Linux 下的可执行连接格式文件，类似于 windows 下的 PE 文件。

ELF 文件（目标文件）格式主要三种：

1.  可重定向文件 (Relocatable file)：文件保存着代码和适当的数据，用来和其他的目标文件一起来创建一个可执行文件或者是一个共享目标文件。（目标文件或者静态库文件，即 linux 通常后缀为. a 和. o 的文件）这是由汇编器汇编生成的 .o 文件。后面的链接器(link editor) 拿一个或一些 Relocatable object files 作为输入，经链接处理后，生成一个可执行的对象文件 (Executable file) 或者一个可被共享的对象文件(Shared object file)，内核可加载模块 .ko 文件也是 Relocatable object file
    
2.  可执行文件 (Executable file)：文件保存着一个用来执行的程序。（例如 bash，gcc 等）
    
3.  共享目标文件：即 .so 文件。如果拿前面的静态库来生成可执行程序，那每个生成的可执行程序中都会有一份库代码的拷贝。如果在磁盘中存储这些可执行程序，那就会占用额外的磁盘空 间；另外如果拿它们放到 Linux 系统上一起运行，也会浪费掉宝贵的物理内存。如果将静态库换成动态库，那么这些问题都不会出现
    

#### [](#1-1-ELF-文件结构 "1.1 ELF 文件结构")1.1 ELF 文件结构

elf 文件主要有两种格式，一是链接状态、二是运行状态。

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200722200900804.png)

这里再放一张 32 位数据类型，方便理解。

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200722202815692.png)

首先看图：

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200722202418781.png)

一个 ELF 文件由以下三部分组成：

*   ELF 头 (ELF header) - 描述文件的主要特性：类型，CPU 架构，入口地址，现有部分的大小和偏移等等；
*   程序头表 (Program header table) - 列举了所有有效的段(segments) 和他们的属性。 程序头表需要加载器将文件中的节加载到虚拟内存段中；
*   节头表 (Section header table) - 包含对节(sections) 的描述。

```
typedef struct
{
    unsigned char  e_ident[EI_NIDENT]; 
    Elf32_Half  e_type;         
    Elf32_Half  e_machine;      
    Elf32_Word  e_version;      
    Elf32_Addr  e_entry;        
    Elf32_Off   e_phoff;        
    Elf32_Off   e_shoff;        
    Elf32_Word  e_flags;        
    Elf32_Half  e_ehsize;       
    Elf32_Half  e_phentsize;        
    Elf32_Half  e_phnum;        
    Elf32_Half  e_shentsize;        
    Elf32_Half  e_shnum;        
    Elf32_Half  e_shstrndx;     
} Elf32_Ehdr;
```

这里可以借助 010editor 和 readelf 来理解文件，蓝色的这一块就是名叫 libhooklib.so 文件的 elf 头。

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200722204307424.png)

一个可执行文件或共享目标文件的程序头表 (program header table) 是一个数组， 数组中的每一个元素称为 “程序头(program header)”，每一个程序头描述了一个 “段(segment)” 或者一块用于准备执行程序的信息。一个目标文件中的 “段 (segment)” 包含一个或者多个 “节(section)”。程序头只对可执行文件或共享目标 文件有意义，对于其它类型的目标文件，该信息可以忽略。在目标文件的文件头 (elf header) 中，e_phentsize 和 e_phnum 成员指定了程序头的大小。

根据 elf 头，可以看到 Program header table 在文件中的偏移为 52（0x34）。

```
typedef struct
{
    Elf32_Word  p_type;         
    Elf32_Off   p_offset;       
    Elf32_Addr  p_vaddr;        
    Elf32_Addr  p_paddr;        
    Elf32_Word  p_filesz;       
    Elf32_Word  p_memsz;        
    Elf32_Word  p_flags;        
    Elf32_Word  p_align;        
} Elf32_Phdr;


enum segment_type
{
    PT_NULL,    
    PT_LOAD,    
    PT_DYNAMIC, 
    PT_INTERP,  
    PT_NOTE,    
    PT_SHLIB,   
    PT_PHDR     
};
```

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200722205838088.png)

这里需要了解一下 PT_LOAD、PT_DYNAMIC、PT_PHDR 这三个类型的程序头。

##### [](#PT-PHDR "PT_PHDR")PT_PHDR

PT_HDR 段保存了程序头表本身的位置和大小。Phdr 表保存了所有的 Phdr 对文件（以及内存镜像）中段的描述信息

##### [](#PT-LOAD "PT_LOAD")PT_LOAD

一个可执行文件至少有一个 PT_LOAD 类型的段。这类程序头描述的是可装载的段，也就是说，这种类型的段将被装载或映射到内存中。

例如，一个需要动态链接的 ELF 可执行文件通常包含以下两个可装载的段（类型为 PT_LOAD）：

存放程序代码的 text 段；

存放全局变量和动态链接信息的 data 段。

上面的两个段将会被映射到内存中，并根据 p_align 中存放的值在内存中对齐。

Text 段（代码段）的权限设置为 PF_X|PF_R 时，可读可执行。

Data 段的权限设置为 PF_W|PF_R，可读可写。

##### [](#PT-DYNAMIC "PT_DYNAMIC")PT_DYNAMIC

动态段是动态链接可执行文件所特有的，包含了动态链接器所必需的一些信息。在动态段中包含了一些标记值和指针，包含但不限于以下内容：

> *   运行时需要链接的共享库列表；
> *   全局偏移表（GOT）的地址——ELF 动态链接库部分；
> *   重定位条目的相关信息。

在程序中，段（segment）并不等于节（section）。段是程序执行的必要组成部分，在每个段中，会有代码或者数据被划分为不同的节。节头表是对这些节的位置和大小描述，主要用于链接和调试。节头表对于程序的执行来说不是必需的，没有节头表，程序仍然可以正常执行，因为节头表没有对程序的内存布局进行描述，对程序内存布局描述是程序头表的任务。

如果二进制文件中缺少节头，并不意味着节就不存在。只是没有办法通过节头来引用节，对于调试器或者反编译程序来说，只是可以参考的信息变少了而已。

每一个节都保存了某种类型的代码或者数据。数据可以是程序中的全局变量，也可以是链接器所需要的动态链接信息。正如前面所提到的，每个 ELF 目标文件都有节，但是不一定有节头，尤其是有人故意将节头从节头表中删除了之后。当然，默认是有节头的。

ELF 文件节头结构如下：

```
typedef struct
{
    Elf32_Word  sh_name;        
    Elf32_Word  sh_type;        
    Elf32_Word  sh_flags;       
    Elf32_Addr  sh_addr;        
    Elf32_Off   sh_offset;      
    Elf32_Word  sh_size;        
    Elf32_Word  sh_link;        
    Elf32_Word  sh_info;        
    Elf32_Word  sh_addralign;       
    Elf32_Word  sh_entsize;     
} Elf32_Shdr;
```

*   .text 节是保留了程序代码指令的代码节。一段可执行程序，如果存在 Phdr，.text 节就会存放在 text 段中。由于. text 节保存了程序代码，因此节的类型为 SHT_PROGBITS。
*   .rodata 节保存了只读数据，因此只能存放于一个可执行文件的只读段中。也因此，只能在 text 段（不是 data 段）中找到. rodata 节。由于. rodata 节是只读的，因此节类型为 SHT_PROGBITS。
*   过程链接表（Procedure Linkage Table，PLT），.plt 节中包含了动态链接器调用从共享库导入的函数所必需的相关代码。由于其存在于 text 段中，同样保存了代码，因此节类型为 SHT_PROGBITS。
*   .data 节存在于 data 段中，保存了初始化的全局变量等数据。由于其保存了程序的变量数据，因此节类型被标记为 SHT_PROGBITS。
*   .bss 节保存了未进行初始化的全局数据，是 data 段的一部分，占用空间不超过 4 字节，仅表示这个节本身的空间。程序加载时数据被初始化为 0，在程序执行期间可以进行赋值。由于. bss 节未保存实际的数据，因此节类型为 SHT_PROGBITS。
*   .got 节保存了全局偏移表。.got 节和. plt 节一起提供了对导入的共享库函数的访问入口，由动态链接器在运行时进行修改。如果攻击者获得了堆或者. bss 漏洞的一个指针大小的写原语，就可以对该节任意进行修改。.got.plt 节跟程序执行有关，因此节类型被标记为 SHT_PROGBITS。
*   .dynsym 节保存了从共享库导入的动态符号信息，该节保存在 text 段中，节类型被标记为 SHT_PROGBITS。
*   .dynstr 节保存了动态符号字符串表，表中存放了一系列字符串，这些字符串代表了符号的名称，以空字符作为终止符。
*   .rel.* 节 重定位节保存了重定位相关的信息，这些信息描述；了在链接或者运行时，对 ELF 目标文件的某部分内容或者进程镜像进行补充或者修改。重定位节保存了重定位相关的数据，因此节类型被标记为 SHT_REL。
*   .hash 节有时也称为. gnu.hahs，保存了一个用于查找符号的散列表。
*   .symtab 节保存了 ElfN_Sym 类型的符号信息，因此节类型被标记为 SHT_SYMTAB。
*   .strtab 节保存的是符号字符串表，表中的内容会被. symtab 的 ElfN_Sym 结构中的 st_name 条目引用。由于其保存了字符串表，因此节类型被标记为 SHT_STRTAB。
*   .shstrtab 节保存节头字符串表，该表是一个以空字符终止的字符串的集合，字符串保存了每个节的节名，如. text、.data 等。有一个名为 e_shsrndx 的 ELF 文件头条目会指向. shstrtab 节，e_shstrndx 中保存了. shstrtab 的偏移量。由于其保存了字符串表，因此节类型被标记为 SHT_STRTAB。
*   .ctors（构造器）和. dtors（析构器）这两个节保存了指向析构函数和析构函数的指针，构造函数是在 main 函数执行之前需要执行的代码，析构函数是在 main 函数之后需要执行的代码。

##### [](#符号表 "符号表")符号表

```
typedef struct { 
Elf32_Word st_name; 

Elf32_Addr st_value; 
Elf32_Word st_size; 
unsigned char st_info; 
unsigned char st_other; 
Elf32_Half st_shndx; 

} Elf32_sym;
```

**符号是对某些类型的数据或代码（如全局变量或函数）的符号引用，函数名或变量名就是符号名**。例如，`printf()`函数会在动态链接符号表`.dynsym`中存有一个指向该函数的符号项（以`Elf_Sym`数据结构表示）。在大多数共享库和动态链接可执行文件中，存在两个符号表。即`.dynsym`和`.symtab`。

**`.dynsym`保存了引用来自外部文件符号的全局符号**。如`printf`库函数。**`.dynsym`保存的符号是`.symtab`所保存符合的子集，`.symtab`中还保存了可执行文件的本地符号 **。如全局变量，代码中定义的本地函数等。

既然`.dynsym`是`.symtab`的子集，那为何要同时存在两个符号表呢？

通过`readelf -S`命令可以查看可执行文件的输出，一部分节标志位（`sh_flags`）被标记为了 **A（ALLOC）、WA（WRITE/ALLOC）、AX（ALLOC/EXEC）**。其中，`.dynsym`被标记为 ALLOC，而`.symtab`则没有标记。

ALLOC 表示有该标记的节会在运行时分配并装载进入内存，而`.symtab`不是在运行时必需的，因此不会被装载到内存中。**`.dynsym`保存的符号只能在运行时被解析，因此是运行时动态链接器所需的唯一符号 **。`.dynsym`对于动态链接可执行文件的执行是必需的，而`.symtab`只是用来进行调试和链接的。

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200723202546045.png)

##### [](#字符串表 "字符串表")字符串表

类似于符号表，在大多数共享库和动态链接可执行文件中，也存在两个字符串表。即`.dynstr`和`.strtab`，分别对应于`.dynsym`和`symtab`。此外，还有一个`.shstrtab`的节头字符串表，用于保存节头表中用到的字符串，可通过`sh_name`进行索引。

字符串表中包含有若干个以 ’ null ’ 结尾的字符序列，即字符串。在目标文件中， 这些字符串通常是符号的名字或者节的名字。在目标文件的其它部分中，当需要引 用某个字符串时，只需要提供该字符串在字符串表中的序号即可。

字符串表中的第一个字符串（序号为 0）永远是空串，即”null”，它可以用于表 示一个空的名字或者没有名字。所以，字符串表的第一个字节是’null’。由于每一 个字符串都是以’null’结尾，所以字符串表的最后一个字节也必然为’null’。

字符串表也可以是空的，不含有任何字符串，这时，节头中的 sh_size 成员必须 是 0。

一个目标文件中可能有多个字符串表，其中一个称为 “节名字表 (.shstrtab)”， 它包含所有节的名字。每一个节头的 sh_name 成员应该是一个索引值，它指向节 名字表中的一个位置，从这个位置开始到接下来第一个’null’字符为止的这个字符 串，正是这个节的名字。

##### [](#重定位 "重定位")重定位

```
typedef struct {
 Elf32_Addr r_offset;
 Elf32_Word r_info;
} Elf32_Rel;
typedef struct {
 Elf32_Addr r_offset;
 Elf32_Word r_info;
 Elf32_Sword r_addend;
} Elf32_Rela;
```

重定位 (relocation) 是把符号引用与符号定义连接在一起的过程。比如，当程序 调用一个函数时，将从当前运行的指令跳转到一个新的指令地址去执行。在编写程 序的时候，我们只需指明所要调用的函数名（即符号引用），在重定位的过程中， 函数名会与实际的函数所在地址（即符号定义）联系起来，使程序知道应该跳转到 哪里去。

重定位文件必须知道如何修改其所包含的 “节” 的内容，在构建可执行文件或 共享目标文件的时候，把节中的符号引用换成这些符号在进程空间中的虚拟地址。 包含这些转换信息的数据也就是“重定位项(relocation entries)”。

### [](#关于linker "关于linker")关于 linker

参考：[so 加载过程](https://cloud.tencent.com/developer/article/1193474)、[参考 2](https://segmentfault.com/a/1190000007008889)

作用：1. 关于解密时机的选择

​ 2. 静态文件可以修改的地方

### [](#ELF-静态注入 "ELF  静态注入")ELF 静态注入

参考：[腾讯](https://gslab.qq.com/portal.php?mod=view&aid=163)

作用：无源码情况下解密（主要是分无源码和有源码）

[](#正文开始 "正文开始")正文开始
--------------------

### [](#so节加密demo "so节加密demo")so 节加密 demo

demo 是我将一个 testfunc 的方法放在 .itext, 然后将 itext 节加密，这样就把 testfunc 保护起来了

```
void testfunc()__attribute__((section(".itext"))){
    LOGD("im encrypt");
}
```

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200826211930404.png)

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200826211956696.png)

加密 python：

```
import lief
import math

sobin =lief.parse("libnative-lib.so")

sobin.add_library("libpro.so") 

itext = sobin.get_section(".itext")

offset = itext.offset
size = itext.size
print(size)

sobin.header.section_header_offset = offset 
sobin.header.entrypoint = size


copyfile("libnative-lib.so","libout.so")


infile =  open("libnative-lib.so","rb")
file = infile.read()
out = open("libout.so","wb")

enc = []
for i in itext.content:
    enc.append(struct.pack('h',~i)[0])

print(hex(file.find(bytes(itext.content))))


out.write(file.replace(bytes(itext.content),bytes(enc)))
out.close()
```

解密 cpp 文件：

```
void init_getString() __attribute__((constructor));
unsigned long getLibAddr();

void init_getString(){
    char name[15];
    unsigned int nblock;
    unsigned int nsize;
    unsigned long base;
    unsigned long text_addr;
    unsigned int i;
    Elf32_Ehdr *ehdr;
    Elf32_Shdr *shdr;

    base = getLibAddr();

    ehdr = (Elf32_Ehdr *)base;

   
  
   
    
    text_addr = ehdr->e_flags + base;
    nsize = ehdr->e_entry & 0xffff;
    npage =nsize / PAGE_SIZE + ((nsize % PAGE_SIZE == 0) ? 0 : 1);
    
    if(mprotect((void *) (text_addr / PAGE_SIZE * PAGE_SIZE), 4096 * npage, PROT_READ | PROT_EXEC | PROT_WRITE) != 0){
        __android_log_print(ANDROID_LOG_INFO, "JNITag", "mem privilege change failed");
    }

    for(i=0;i< nsize; i++){
        char *addr = (char*)(text_addr + i);
        *addr = ~(*addr);
    }

    if(mprotect((void *) (text_addr / PAGE_SIZE * PAGE_SIZE), 4096 * npage, PROT_READ | PROT_EXEC) != 0){
        __android_log_print(ANDROID_LOG_INFO, "JNITag", "mem privilege change failed");
    }

    __android_log_print(ANDROID_LOG_INFO, "JNITag", "Decrypt success");
}

unsigned long getLibAddr(){
    unsigned long ret = 0;
    char name[] = "libnative-lib.so";
    char buf[4096], *temp;
    int pid;
    FILE *fp;
    pid = getpid();
    sprintf(buf, "/proc/%d/maps", pid);
    fp = fopen(buf, "r");
    if(fp == NULL)
    {
        puts("open failed");
        goto _error;
    }
    while(fgets(buf, sizeof(buf), fp)){
        if(strstr(buf, name)){
            temp = strtok(buf, "-");
            ret = strtoul(temp, NULL, 16);
            break;
        }
    }
    _error:
    fclose(fp);
    return ret;
}
```

会遇到的情况：  
对 so 进行 section 加密后，提示. dynamic section header was not found  
遇到问题：Android7.0 后 JNI 库必须保留 Section Headers。由于加密时修改了 shoff 值，导致加载 so 库值解析 Section Headers 解析不了，.dynamic section header was not found。

修改测量：shoff 和 entry 目的是为了存储加密的偏移大小和加密的大小。使用 e_flags 和 e_entry 来储存段的偏移和大小

### [](#腾讯so加密 "腾讯so加密")腾讯 so 加密

首先，当王者荣耀加载 libil2cpp.so 时，会先加载 libtprt.so

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200827143911255.png)

libtprt.so 对自身的 tptext 节进行了加密，分析 init_array, 可以看到解密函数如下，关键函数时 mprotect，因为解密会用到。

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200827144338193.png)

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200827144827705.png)

根据分析解密函数：

```
tprt = lief.parse(name)
tptext_section = tprt.get_section(".tptext").content

print(len(tptext_section))
offset = tprt.get_section(".tptext").offset
out = b""
for i in range(len(tptext_section)):
    out += (tptext_section[i] ^ 0xb8).to_bytes(1,byteorder='little')


print(len(out))
tpp = tprt_bin[:offset]+out+tprt_bin[offset+len(tptext_section):]
with open("tpp.so", 'wb') as fp:
    fp.write(tpp)
```

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200827145222734.png)

tptext 节里的函数主要用于初始化各类保护，会被 jni_onload 调用。现在 tprt 已经被成功加载，现在就需要解密 libil2cpp.so（linker 加载先加载链接库，再执行构造函数）

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200827150229908.png)

问题 1：关于 ADRL 作用 [http://blog.sina.com.cn/s/blog_4b5210840100c80i.html](http://blog.sina.com.cn/s/blog_4b5210840100c80i.html)

取得基址后，根据 base+0x1d0 处的偏移获得解密函数地址，调用解密函数，在此处调用 libtprt.so 的导出函数 g_tprt_pfn_array，init_proc -> 调用 g_tprt_pfn_array（“.text”,?,3）-> 解密 .text 段

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200827152258578.png)

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200827152450124.png)

查看 lintprt.so 的导出函数，

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200827152727259.png)

![](https://nszdhd1.github.io/2020/08/27/SO%E5%8A%A0%E5%AF%86/image-20200827152830212.png)

end：到目前为止，都很简单，加密的难点在于如何修改 elf 文件要在能正常运行的情况下添加需要的数据。