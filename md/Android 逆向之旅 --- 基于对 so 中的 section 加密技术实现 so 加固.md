> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/roccheung/p/5797267.html)

一、前言
====

好长时间没有更新文章了，主要还是工作上的事，连续加班一个月，没有时间研究了，只有周末有时间，来看一下，不过我还是延续之前的文章，继续我们的逆向之旅，今天我们要来看一下如何通过对 so 加密，在介绍本篇文章之前的话，一定要先阅读之前的文章：

so 文件格式详解以及如何解析一个 so 文件

[http://blog.csdn.net/jiangwei0910410003/article/details/49336613](http://blog.csdn.net/jiangwei0910410003/article/details/49336613)  

这个是我们今天这篇文章的基础，如果不了解 so 文件的格式的话，下面的知识点可能会看的很费劲  

下面就来介绍我们今天的话题：对 so 中的 section 进行加密

二、技术原理
======

加密：在之前的文章中我们介绍了 so 中的格式，那么对于找到一个 section 的 base 和 size 就可以对这段 section 进行加密了

解密：因为我们对 section 进行加密之后，肯定需要解密的，不然的话，运行肯定是报错的，那么这里的重点是什么时候去进行解密，对于一个 so 文件，我们 load 进程序之后，在运行程序之前我们可以从哪个时间点来突破？这里就需要一个知识点：

**__attribute__((constructor));**  

关于这个，属性的用法这里就不做介绍了，网上有相关资料，他的作用很简单，就是优先于 main 方法之前执行，类似于 Java 中的构造函数，当然其实 C++ 中的构造函数就是基于这个属性实现的，我们在之前介绍 elf 文件格式的时候，有两个 section 会引起我们的注意：

![](http://img-blog.csdn.net/20151121101215972?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

对于这两个 section, 其实就是用这个属性实现的函数存在这里，

在动态链接器构造了进程映像, 并执行了重定位以后, **每个共享的目标都获得执行 某些初始化代码的机会**。这些初始化函数的被调用顺序是不一定的, 不过所有共享目标 初始化都会在可执行文件得到控制之前发生。  
类似地, 共享目标也包含终止函数, 这些函数在进程完成终止动作序列时, 通过 atexit() 机制执行。动态链接器对终止函数的调用顺序是不确定的。  
共享目标通过动态结构中的 DT_INIT 和 DT_FINI 条目指定初始化 / 终止函数。通常 这些代码放在. init 和. fini 节区中。  

这个知识点很重要，我们后面在进行动态调试 so 的时候，还会用到这个知识点，所以一定要理解。

所以，在这里我们找到了解密的时机，就是自己定义一个解密函数，然后用上面的这个属性声明就可以了。

三、实现流程
======

第一、我们编写一个简单的 native 代码，这里我们需要做两件事：

1、将我们核心的 native 函数定义在自己的一个 section 中，这里会用到这个属性：**__attribute__((section (".mytext")));**

其中. mytext 就是我们自己定义的 section.

说到这里，还记得我们之前介绍的一篇文章中介绍了，动态的给 so 添加一个 section:

[http://blog.csdn.net/jiangwei0910410003/article/details/49361281](http://blog.csdn.net/jiangwei0910410003/article/details/49361281)

2、需要编写我们的解密函数，用属性： __attribute__((constructor)); 声明

这样一个 native 程序就包含这两个重要的函数，使用 ndk 编译成 so 文件

第二、编写加密程序，在加密程序中我们需要做的是：

1、通过解析 so 文件，找到. mytext 段的起始地址和大小，这里的思路是：

找到所有的 Section, 然后获取他的 name 字段，在结合 String Section，遍历找到. mytext 字段

2、找到. mytext 段之后，然后进行加密，最后在写入到文件中。

四、技术实现
======

前面介绍了原理和实现方案，下面就开始 coding 吧，

**第一、我们先来看看 native 程序**

```
#include <jni.h>
#include <stdio.h>
#include <android/log.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <elf.h>
#include <sys/mman.h>

jstring getString(JNIEnv*) __attribute__((section (".mytext")));
jstring getString(JNIEnv* env){
	return (*env)->NewStringUTF(env, "Native method return!");
};

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
  text_addr = ehdr->e_shoff + base;
  
  nblock = ehdr->e_entry >> 16;
  nsize = ehdr->e_entry & 0xffff;

  __android_log_print(ANDROID_LOG_INFO, "JNITag", "nblock =  0x%x,nsize:%d", nblock,nsize);
  __android_log_print(ANDROID_LOG_INFO, "JNITag", "base =  0x%x", text_addr);
  printf("nblock = %d\n", nblock);
  
  if(mprotect((void *) (text_addr / PAGE_SIZE * PAGE_SIZE), 4096 * nsize, PROT_READ | PROT_EXEC | PROT_WRITE) != 0){
    puts("mem privilege change failed");
     __android_log_print(ANDROID_LOG_INFO, "JNITag", "mem privilege change failed");
  }
  
  for(i=0;i< nblock; i++){  
    char *addr = (char*)(text_addr + i);
    *addr = ~(*addr);
  }
  
  if(mprotect((void *) (text_addr / PAGE_SIZE * PAGE_SIZE), 4096 * nsize, PROT_READ | PROT_EXEC) != 0){
    puts("mem privilege change failed");
  }
  puts("Decrypt success");
}

unsigned long getLibAddr(){
  unsigned long ret = 0;
  char name[] = "libdemo.so";
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

JNIEXPORT jstring JNICALL
Java_com_example_shelldemo_MainActivity_getString( JNIEnv* env,
                                                  jobject thiz )
{
#if defined(__arm__)
  #if defined(__ARM_ARCH_7A__)
    #if defined(__ARM_NEON__)
      #define ABI "armeabi-v7a/NEON"
    #else
      #define ABI "armeabi-v7a"
    #endif
  #else
   #define ABI "armeabi"
  #endif
#elif defined(__i386__)
   #define ABI "x86"
#elif defined(__mips__)
   #define ABI "mips"
#else
   #define ABI "unknown"
#endif

    return getString(env);
}



```

下面来分析一下代码：

1、定义自己的段

```
jstring getString(JNIEnv*) __attribute__((section (".mytext")));
jstring getString(JNIEnv* env){
	return (*env)->NewStringUTF(env, "Native method return!");
};


```

这里的 getString 返回一个字符串，提供给 Android 上层，然后将 getString 定义在. mytext 段中。

2、获取 so 加载到内存中的起始地址

```
unsigned long getLibAddr(){
  unsigned long ret = 0;
  char name[] = "libdemo.so";
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

这里的代码其实就是读取设备的

**proc/<uid>/maps**

中的内容，因为这个 maps 中是程序运行的内存映像：

![](http://img-blog.csdn.net/20151121103426202?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  
我们只有获取到 so 的起始地址，才能找到指定的 Section 然后进行解密。

3、解密函数

```
void init_getString(){
  char name[15];
  unsigned int nblock;
  unsigned int nsize;
  unsigned long base;
  unsigned long text_addr;
  unsigned int i;
  Elf32_Ehdr *ehdr;
  Elf32_Shdr *shdr;
  
  //获取so的起始地址
  base = getLibAddr();
  
  //获取指定section的偏移值和size
  ehdr = (Elf32_Ehdr *)base;
  text_addr = ehdr->e_shoff + base;
  
  nblock = ehdr->e_entry >> 16;
  nsize = ehdr->e_entry & 0xffff;

  __android_log_print(ANDROID_LOG_INFO, "JNITag", "nblock =  0x%x,nsize:%d", nblock,nsize);
  __android_log_print(ANDROID_LOG_INFO, "JNITag", "base =  0x%x", text_addr);
  printf("nblock = %d\n", nblock);
  
  //修改内存的操作权限
  if(mprotect((void *) (text_addr / PAGE_SIZE * PAGE_SIZE), 4096 * nsize, PROT_READ | PROT_EXEC | PROT_WRITE) != 0){
    puts("mem privilege change failed");
     __android_log_print(ANDROID_LOG_INFO, "JNITag", "mem privilege change failed");
  }
  //解密
  for(i=0;i< nblock; i++){  
    char *addr = (char*)(text_addr + i);
    *addr = ~(*addr);
  }
  
  if(mprotect((void *) (text_addr / PAGE_SIZE * PAGE_SIZE), 4096 * nsize, PROT_READ | PROT_EXEC) != 0){
    puts("mem privilege change failed");
  }
  puts("Decrypt success");
}

```

这里我们获取到 so 文件的头部，然后获取指定 section 的偏移地址和 size

```
//获取so的起始地址
base = getLibAddr();

//获取指定section的偏移值和size
ehdr = (Elf32_Ehdr *)base;
text_addr = ehdr->e_shoff + base;

nblock = ehdr->e_entry >> 16;
nsize = ehdr->e_entry & 0xffff;

```

这里可能会有困惑？为什么这里是这么获取 offset 和 size 的，其实这里我们做了一点工作，就是我们在加密的时候顺便改写了 so 的头部信息，将 offset 和 size 值写到了头部中，这样加大破解难度。后面在说到加密的时候在详解。

**text_addr 是起始地址 + 偏移值，就是我们的 section 在内存中的绝对地址**

**nsize 是我们的 section 占用的页数**  

然后修改这个 section 的内存操作权限  

```
//修改内存的操作权限
if(mprotect((void *) (text_addr / PAGE_SIZE * PAGE_SIZE), 4096 * nsize, PROT_READ | PROT_EXEC | PROT_WRITE) != 0){
	puts("mem privilege change failed");
	__android_log_print(ANDROID_LOG_INFO, "JNITag", "mem privilege change failed");
}

```

这里调用了一个系统函数：mprotect

第一个参数：需要修改内存的起始地址

必须需要页面对齐，也就是必须是页面 PAGE_SIZE(0x1000=4096) 的整数倍  

第二个参数：需要修改的大小

占用的页数 * PAGE_SIZE

第三个参数：权限值

最后读取内存中的 section 内容，然后进行解密，在将内存权限修改回去。

然后使用 ndk 编译成 so 即可，这里我们用到了系统的打印 log 信息, 所以需要用到共享库，看一下编译脚本 Android.mk

```
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)
LOCAL_MODULE := demo
LOCAL_SRC_FILES := demo.c
LOCAL_LDLIBS := -llog
include $(BUILD_SHARED_LIBRARY)

```

关于如何使用 ndk, 这里就不做介绍了，参考这篇文章：

[http://blog.csdn.net/jiangwei0910410003/article/details/17710243](http://blog.csdn.net/jiangwei0910410003/article/details/17710243)

**第二、加密程序**

1、加密程序 (Java 版)

我们获取到上面的 so 文件，下面我们就来看看如何进行加密的：

```
package com.jiangwei.encodesection;

import com.jiangwei.encodesection.ElfType32.Elf32_Sym;
import com.jiangwei.encodesection.ElfType32.elf32_phdr;
import com.jiangwei.encodesection.ElfType32.elf32_shdr;

public class EncodeSection {
	
	public static String encodeSectionName = ".mytext";
	
	public static ElfType32 type_32 = new ElfType32();
	
	public static void main(String[] args){
		
		byte[] fileByteArys = Utils.readFile("so/libdemo.so");
		if(fileByteArys == null){
			System.out.println("read file byte failed...");
			return;
		}
		
		/**
		 * 先解析so文件
		 * 然后初始化AddSection中的一些信息
		 * 最后在AddSection
		 */
		parseSo(fileByteArys);
		
		encodeSection(fileByteArys);
		
		parseSo(fileByteArys);
		
		Utils.saveFile("so/libdemos.so", fileByteArys);
		
	}
	
	private static void encodeSection(byte[] fileByteArys){
		//读取String Section段
		System.out.println();
		
		int string_section_index = Utils.byte2Short(type_32.hdr.e_shstrndx);
		elf32_shdr shdr = type_32.shdrList.get(string_section_index);
		int size = Utils.byte2Int(shdr.sh_size);
		int offset = Utils.byte2Int(shdr.sh_offset);

		int mySectionOffset=0,mySectionSize=0;
		for(elf32_shdr temp : type_32.shdrList){
			int sectionNameOffset = offset+Utils.byte2Int(temp.sh_name);
			if(Utils.isEqualByteAry(fileByteArys, sectionNameOffset, encodeSectionName)){
				//这里需要读取section段然后进行数据加密
				mySectionOffset = Utils.byte2Int(temp.sh_offset);
				mySectionSize = Utils.byte2Int(temp.sh_size);
				byte[] sectionAry = Utils.copyBytes(fileByteArys, mySectionOffset, mySectionSize);
				for(int i=0;i<sectionAry.length;i++){
					sectionAry[i] = (byte)(sectionAry[i] ^ 0xFF);
				}
				Utils.replaceByteAry(fileByteArys, mySectionOffset, sectionAry);
			}
		}

		//修改Elf Header中的entry和offset值
		int nSize = mySectionSize/4096 + (mySectionSize%4096 == 0 ? 0 : 1);
		byte[] entry = new byte[4];
		entry = Utils.int2Byte((mySectionSize<<16) + nSize);
		Utils.replaceByteAry(fileByteArys, 24, entry);
		byte[] offsetAry = new byte[4];
		offsetAry = Utils.int2Byte(mySectionOffset);
		Utils.replaceByteAry(fileByteArys, 32, offsetAry);
	}
	
	private static void parseSo(byte[] fileByteArys){
		//读取头部内容
		System.out.println("+++++++++++++++++++Elf Header+++++++++++++++++");
		parseHeader(fileByteArys, 0);
		System.out.println("header:\n"+type_32.hdr);

		//读取程序头信息
		//System.out.println();
		//System.out.println("+++++++++++++++++++Program Header+++++++++++++++++");
		int p_header_offset = Utils.byte2Int(type_32.hdr.e_phoff);
		parseProgramHeaderList(fileByteArys, p_header_offset);
		//type_32.printPhdrList();

		//读取段头信息
		//System.out.println();
		//System.out.println("+++++++++++++++++++Section Header++++++++++++++++++");
		int s_header_offset = Utils.byte2Int(type_32.hdr.e_shoff);
		parseSectionHeaderList(fileByteArys, s_header_offset);
		//type_32.printShdrList();
		
		//这种方式获取所有的Section的name
		/*byte[] names = Utils.copyBytes(fileByteArys, offset, size);
		String str = new String(names);
		byte NULL = 0;//字符串的结束符
		StringTokenizer st = new StringTokenizer(str, new String(new byte[]{NULL}));
		System.out.println( "Token Total: " + st.countTokens() );
		while(st.hasMoreElements()){
			System.out.println(st.nextToken());
		}
		System.out.println("");*/

		/*//读取符号表信息(Symbol Table)
		System.out.println();
		System.out.println("+++++++++++++++++++Symbol Table++++++++++++++++++");
		//这里需要注意的是：在Elf表中没有找到SymbolTable的数目，但是我们仔细观察Section中的Type=DYNSYM段的信息可以得到，这个段的大小和偏移地址，而SymbolTable的结构大小是固定的16个字节
		//那么这里的数目=大小/结构大小
		//首先在SectionHeader中查找到dynsym段的信息
		int offset_sym = 0;
		int total_sym = 0;
		for(elf32_shdr shdr : type_32.shdrList){
			if(Utils.byte2Int(shdr.sh_type) == ElfType32.SHT_DYNSYM){
				total_sym = Utils.byte2Int(shdr.sh_size);
				offset_sym = Utils.byte2Int(shdr.sh_offset);
				break;
			}
		}
		int num_sym = total_sym / 16;
		System.out.println("sym num="+num_sym);
		parseSymbolTableList(fileByteArys, num_sym, offset_sym);
		type_32.printSymList();

		//读取字符串表信息(String Table)
		System.out.println();
		System.out.println("+++++++++++++++++++Symbol Table++++++++++++++++++");
		//这里需要注意的是：在Elf表中没有找到StringTable的数目，但是我们仔细观察Section中的Type=STRTAB段的信息，可以得到，这个段的大小和偏移地址，但是我们这时候我们不知道字符串的大小，所以就获取不到数目了
		//这里我们可以查看Section结构中的name字段：表示偏移值，那么我们可以通过这个值来获取字符串的大小
		//可以这么理解：当前段的name值 减去 上一段的name的值 = (上一段的name字符串的长度)
		//首先获取每个段的name的字符串大小
		int prename_len = 0;
		int[] lens = new int[type_32.shdrList.size()];
		int total = 0;
		for(int i=0;i<type_32.shdrList.size();i++){
			if(Utils.byte2Int(type_32.shdrList.get(i).sh_type) == ElfType32.SHT_STRTAB){
				int curname_offset = Utils.byte2Int(type_32.shdrList.get(i).sh_name);
				lens[i] = curname_offset - prename_len - 1;
				if(lens[i] < 0){
					lens[i] = 0;
				}
				total += lens[i];
				System.out.println("total:"+total);
				prename_len = curname_offset;
				//这里需要注意的是，最后一个字符串的长度，需要用总长度减去前面的长度总和来获取到
				if(i == (lens.length - 1)){
					System.out.println("size:"+Utils.byte2Int(type_32.shdrList.get(i).sh_size));
					lens[i] = Utils.byte2Int(type_32.shdrList.get(i).sh_size) - total - 1;
				}
			}
		}
		for(int i=0;i<lens.length;i++){
			System.out.println("len:"+lens[i]);
		}
		//上面的那个方法不好，我们发现StringTable中的每个字符串结束都会有一个00(传说中的字符串结束符)，那么我们只要知道StringTable的开始位置，然后就可以读取到每个字符串的值了
       */
	}
	
	/**
	 * 解析Elf的头部信息
	 * @param header
	 */
	private static void  parseHeader(byte[] header, int offset){
		if(header == null){
			System.out.println("header is null");
			return;
		}
		/**
		 *  public byte[] e_ident = new byte[16];
			public short e_type;
			public short e_machine;
			public int e_version;
			public int e_entry;
			public int e_phoff;
			public int e_shoff;
			public int e_flags;
			public short e_ehsize;
			public short e_phentsize;
			public short e_phnum;
			public short e_shentsize;
			public short e_shnum;
			public short e_shstrndx;
		 */
		type_32.hdr.e_ident = Utils.copyBytes(header, 0, 16);//魔数
		type_32.hdr.e_type = Utils.copyBytes(header, 16, 2);
		type_32.hdr.e_machine = Utils.copyBytes(header, 18, 2);
		type_32.hdr.e_version = Utils.copyBytes(header, 20, 4);
		type_32.hdr.e_entry = Utils.copyBytes(header, 24, 4);
		type_32.hdr.e_phoff = Utils.copyBytes(header, 28, 4);
		type_32.hdr.e_shoff = Utils.copyBytes(header, 32, 4);
		type_32.hdr.e_flags = Utils.copyBytes(header, 36, 4);
		type_32.hdr.e_ehsize = Utils.copyBytes(header, 40, 2);
		type_32.hdr.e_phentsize = Utils.copyBytes(header, 42, 2);
		type_32.hdr.e_phnum = Utils.copyBytes(header, 44,2);
		type_32.hdr.e_shentsize = Utils.copyBytes(header, 46,2);
		type_32.hdr.e_shnum = Utils.copyBytes(header, 48, 2);
		type_32.hdr.e_shstrndx = Utils.copyBytes(header, 50, 2);
	}
	
	/**
	 * 解析程序头信息
	 * @param header
	 */
	public static void parseProgramHeaderList(byte[] header, int offset){
		int header_size = 32;//32个字节
		int header_count = Utils.byte2Short(type_32.hdr.e_phnum);//头部的个数
		byte[] des = new byte[header_size];
		for(int i=0;i<header_count;i++){
			System.arraycopy(header, i*header_size + offset, des, 0, header_size);
			type_32.phdrList.add(parseProgramHeader(des));
		}
	}
	
	private static elf32_phdr parseProgramHeader(byte[] header){
		/**
		 *  public int p_type;
			public int p_offset;
			public int p_vaddr;
			public int p_paddr;
			public int p_filesz;
			public int p_memsz;
			public int p_flags;
			public int p_align;
		 */
		ElfType32.elf32_phdr phdr = new ElfType32.elf32_phdr();
		phdr.p_type = Utils.copyBytes(header, 0, 4);
		phdr.p_offset = Utils.copyBytes(header, 4, 4);
		phdr.p_vaddr = Utils.copyBytes(header, 8, 4);
		phdr.p_paddr = Utils.copyBytes(header, 12, 4);
		phdr.p_filesz = Utils.copyBytes(header, 16, 4);
		phdr.p_memsz = Utils.copyBytes(header, 20, 4);
		phdr.p_flags = Utils.copyBytes(header, 24, 4);
		phdr.p_align = Utils.copyBytes(header, 28, 4);
		return phdr;
		
	}
	
	/**
	 * 解析段头信息内容
	 */
	public static void parseSectionHeaderList(byte[] header, int offset){
		int header_size = 40;//40个字节
		int header_count = Utils.byte2Short(type_32.hdr.e_shnum);//头部的个数
		byte[] des = new byte[header_size];
		for(int i=0;i<header_count;i++){
			System.arraycopy(header, i*header_size + offset, des, 0, header_size);
			type_32.shdrList.add(parseSectionHeader(des));
		}
	}
	
	private static elf32_shdr parseSectionHeader(byte[] header){
		ElfType32.elf32_shdr shdr = new ElfType32.elf32_shdr();
		/**
		 *  public byte[] sh_name = new byte[4];
			public byte[] sh_type = new byte[4];
			public byte[] sh_flags = new byte[4];
			public byte[] sh_addr = new byte[4];
			public byte[] sh_offset = new byte[4];
			public byte[] sh_size = new byte[4];
			public byte[] sh_link = new byte[4];
			public byte[] sh_info = new byte[4];
			public byte[] sh_addralign = new byte[4];
			public byte[] sh_entsize = new byte[4];
		 */
		shdr.sh_name = Utils.copyBytes(header, 0, 4);
		shdr.sh_type = Utils.copyBytes(header, 4, 4);
		shdr.sh_flags = Utils.copyBytes(header, 8, 4);
		shdr.sh_addr = Utils.copyBytes(header, 12, 4);
		shdr.sh_offset = Utils.copyBytes(header, 16, 4);
		shdr.sh_size = Utils.copyBytes(header, 20, 4);
		shdr.sh_link = Utils.copyBytes(header, 24, 4);
		shdr.sh_info = Utils.copyBytes(header, 28, 4);
		shdr.sh_addralign = Utils.copyBytes(header, 32, 4);
		shdr.sh_entsize = Utils.copyBytes(header, 36, 4);
		return shdr;
	}
	
	/**
	 * 解析Symbol Table内容 
	 */
	public static void parseSymbolTableList(byte[] header, int header_count, int offset){
		int header_size = 16;//16个字节
		byte[] des = new byte[header_size];
		for(int i=0;i<header_count;i++){
			System.arraycopy(header, i*header_size + offset, des, 0, header_size);
			type_32.symList.add(parseSymbolTable(des));
		}
	}
	
	private static ElfType32.Elf32_Sym parseSymbolTable(byte[] header){
		/**
		 *  public byte[] st_name = new byte[4];
			public byte[] st_value = new byte[4];
			public byte[] st_size = new byte[4];
			public byte st_info;
			public byte st_other;
			public byte[] st_shndx = new byte[2];
		 */
		Elf32_Sym sym = new Elf32_Sym();
		sym.st_name = Utils.copyBytes(header, 0, 4);
		sym.st_value = Utils.copyBytes(header, 4, 4);
		sym.st_size = Utils.copyBytes(header, 8, 4);
		sym.st_info = header[12];
		//FIXME 这里有一个问题，就是这个字段读出来的值始终是0
		sym.st_other = header[13];
		sym.st_shndx = Utils.copyBytes(header, 14, 2);
		return sym;
	}
	

}


```

在这里，我需要解析 so 文件的头部信息，程序头信息，段头信息

```
//读取头部内容
System.out.println("+++++++++++++++++++Elf Header+++++++++++++++++");
parseHeader(fileByteArys, 0);
System.out.println("header:\n"+type_32.hdr);

//读取程序头信息
//System.out.println();
//System.out.println("+++++++++++++++++++Program Header+++++++++++++++++");
int p_header_offset = Utils.byte2Int(type_32.hdr.e_phoff);
parseProgramHeaderList(fileByteArys, p_header_offset);
//type_32.printPhdrList();

//读取段头信息
//System.out.println();
//System.out.println("+++++++++++++++++++Section Header++++++++++++++++++");
int s_header_offset = Utils.byte2Int(type_32.hdr.e_shoff);
parseSectionHeaderList(fileByteArys, s_header_offset);
//type_32.printShdrList();

```

关于这个解析的工作说明这里就不解析了，看之前解析 elf 文件的那篇文章。

获取这些信息之后，下面就来开始寻找我们的段了，只需要遍历 Section 列表，找到名字是. mytext 的 section 即可，然后获取 offset 和 size, 对内容进行加密，回写到文件中。

下面来看看核心方法：

```
private static void encodeSection(byte[] fileByteArys){
	//读取String Section段
	System.out.println();

	int string_section_index = Utils.byte2Short(type_32.hdr.e_shstrndx);
	elf32_shdr shdr = type_32.shdrList.get(string_section_index);
	int size = Utils.byte2Int(shdr.sh_size);
	int offset = Utils.byte2Int(shdr.sh_offset);

	int mySectionOffset=0,mySectionSize=0;
	for(elf32_shdr temp : type_32.shdrList){
		int sectionNameOffset = offset+Utils.byte2Int(temp.sh_name);
		if(Utils.isEqualByteAry(fileByteArys, sectionNameOffset, encodeSectionName)){
			//这里需要读取section段然后进行数据加密
			mySectionOffset = Utils.byte2Int(temp.sh_offset);
			mySectionSize = Utils.byte2Int(temp.sh_size);
			byte[] sectionAry = Utils.copyBytes(fileByteArys, mySectionOffset, mySectionSize);
			for(int i=0;i<sectionAry.length;i++){
				sectionAry[i] = (byte)(sectionAry[i] ^ 0xFF);
			}
			Utils.replaceByteAry(fileByteArys, mySectionOffset, sectionAry);
		}
	}

	//修改Elf Header中的entry和offset值
	int nSize = mySectionSize/4096 + (mySectionSize%4096 == 0 ? 0 : 1);
	byte[] entry = new byte[4];
	entry = Utils.int2Byte((mySectionSize<<16) + nSize);
	Utils.replaceByteAry(fileByteArys, 24, entry);
	byte[] offsetAry = new byte[4];
	offsetAry = Utils.int2Byte(mySectionOffset);
	Utils.replaceByteAry(fileByteArys, 32, offsetAry);
}

```

我们知道 Section 中的 sh_name 字段的值是这个 section 段的 name 在 StringSection 中的索引值，这里 offset 就是 StringSection 在文件中的偏移值。当然我们需要知道的一个知识点就是：StringSection 中的每个 name 都是以 \ 0 结尾的，所以我们只需要判断字符串到结束符就可以了，判断方法是 Utils.isEqualByteAry:

```
public static boolean isEqualByteAry(byte[] src, int start, String destStr){
	if(destStr == null){
		return false;
	}
	byte[] dest = destStr.getBytes();
	if(src == null || dest == null){
		return false;
	}
	if(dest.length == 0 || src.length == 0){
		return false;
	}
	if(start >= src.length){
		return false;
	}

	int len = 0;
	byte temp = src[start];
	while(temp != 0){
		len++;
		temp = src[start+len];
	}

	byte[] sonAry = copyBytes(src, start, len);
	if(sonAry == null || sonAry.length == 0){
		return false;
	}
	if(sonAry.length != dest.length){
		return false;
	}
	String sonStr = new String(sonAry);
	if(destStr.equals(sonStr)){
		return true;
	}
	return false;
}

```

这里我们加密的方法很简单，加密完成之后，我们需要做的是回写到 so 文件中，当然这里我们还需要做一件事，就是将我们加密的. mytext 段的偏移值和 pageSize 保存到头部信息中：

```
//修改Elf Header中的entry和offset值
int nSize = mySectionSize/4096 + (mySectionSize%4096 == 0 ? 0 : 1);
byte[] entry = new byte[4];
entry = Utils.int2Byte((mySectionSize<<16) + nSize);
Utils.replaceByteAry(fileByteArys, 24, entry);

```

这里又有一个知识点需要说明？大家可能会困惑，我们这样修改了 so 的头部信息的话，在加载运行 so 文件的时候不会报错吗？这个就要看看 Android 底层是如何解析 so 文件，然后将 so 文件映射到内存中的了，下面我们来看看系统是如何解析 so 文件的？

源代码的位置：Android linker 源码：**bionic\linker**

在 linker.h 源码中有一个重要的结构体 soinfo，下面列出一些字段：  

```
struct soinfo{
    const char name[SOINFO_NAME_LEN]; //so全名
    Elf32_Phdr *phdr; //Program header的地址
int phnum; //segment 数量
unsigned *dynamic; //指向.dynamic，在section和segment中相同的
//以下4个成员与.hash表有关
unsigned nbucket;
unsigned nchain;
unsigned *bucket;
unsigned *chain;
//这两个成员只能会出现在可执行文件中
unsigned *preinit_array;
unsigned preinit_array_count;

```

指向初始化代码，先于 main 函数之行，即在加载时被 linker 所调用，在 linker.c 可以看到：__linker_init -> link_image ->

```
call_constructors -> call_array
unsigned *init_array;
unsigned init_array_count;
void (*init_func)(void);
//与init_array类似，只是在main结束之后执行
unsigned *fini_array;
unsigned fini_array_count;
void (*fini_func)(void);
}

```

另外，linker.c 中也有许多地方可以佐证。其本质还是 linker 是基于装载视图解析的 so 文件的。

基于上面的结论，再来分析下 ELF 头的字段。

1) e_ident[EI_NIDENT] 字段包含魔数、字节序、字长和版本，后面填充 0。对于安卓的 linker，通过 verify_elf_object 函数检验魔数，判定是否为. so 文件。那么，我们可以向位置写入数据，至少可以向后面的 0 填充位置写入数据。遗憾的是，我在 fedora 14 下测试，是不能向 0 填充位置写数据，链接器报非 0 填充错误。

2) 对于安卓的 linker，对 e_type、e_machine、e_version 和 e_flags 字段并不关心，是可以修改成其他数据的 (仅分析，没有实测)

3) 对于动态链接库，e_entry 入口地址是无意义的，因为程序被加载时，设定的跳转地址是动态连接器的地址，这个字段是可以被作为数据填充的。

4) so 装载时，与链接视图没有关系，即 e_shoff、e_shentsize、e_shnum 和 e_shstrndx 这些字段是可以任意修改的。被修改之后，使用 readelf 和 ida 等工具打开，会报各种错误，相信读者已经见识过了。

5) 既然 so 装载与装载视图紧密相关，自然 e_phoff、e_phentsize 和 e_phnum 这些字段是不能动的。

从上面我们可以知道，so 中的有些信息在运行的时候是没有用途的，有些东西是不能改的。

2、加密程序 (C 版)

上面说的是 Java 版本的，下面再来一个 C 版本的：

```
#include <stdio.h>
#include <fcntl.h>
#include "elf.h"
#include <stdlib.h>
#include <string.h>

int main(int argc, char** argv){
  char *encodeSoName = "libdemo.so";
  char target_section[] = ".mytext";
  char *shstr = NULL;
  char *content = NULL;
  Elf32_Ehdr ehdr;
  Elf32_Shdr shdr;
  int i;
  unsigned int base, length;
  unsigned short nblock;
  unsigned short nsize;
  unsigned char block_size = 16;
  
  int fd;
  
  fd = open(encodeSoName, O_RDWR);
  if(fd < 0){
    printf("open %s failed\n", argv[1]);
    goto _error;
  }
  
  if(read(fd, &ehdr, sizeof(Elf32_Ehdr)) != sizeof(Elf32_Ehdr)){
    puts("Read ELF header error");
    goto _error;
  }
  
  lseek(fd, ehdr.e_shoff + sizeof(Elf32_Shdr) * ehdr.e_shstrndx, SEEK_SET);
  
  if(read(fd, &shdr, sizeof(Elf32_Shdr)) != sizeof(Elf32_Shdr)){
    puts("Read ELF section string table error");
    goto _error;
  }
  
  if((shstr = (char *) malloc(shdr.sh_size)) == NULL){
    puts("Malloc space for section string table failed");
    goto _error;
  }
  
  lseek(fd, shdr.sh_offset, SEEK_SET);
  if(read(fd, shstr, shdr.sh_size) != shdr.sh_size){
    puts("Read string table failed");
    goto _error;
  }

  lseek(fd, ehdr.e_shoff, SEEK_SET);
  for(i = 0; i < ehdr.e_shnum; i++){
    if(read(fd, &shdr, sizeof(Elf32_Shdr)) != sizeof(Elf32_Shdr)){
      puts("Find section .text procedure failed");
      goto _error;
    }
    if(strcmp(shstr + shdr.sh_name, target_section) == 0){
      base = shdr.sh_offset;
      length = shdr.sh_size;
      printf("Find section %s\n", target_section);
      break;
    }
  }
  
  lseek(fd, base, SEEK_SET);
  content = (char*) malloc(length);
  if(content == NULL){
    puts("Malloc space for content failed");
    goto _error;
  }
  if(read(fd, content, length) != length){
    puts("Read section .text failed");
    goto _error;
  }
  
  nblock = length / block_size;
  nsize = length / 4096 + (length % 4096 == 0 ? 0 : 1);
  printf("base = %x, length = %x\n", base, length);
  printf("nblock = %d, nsize = %d\n", nblock, nsize);
  printf("entry:%x\n",((length << 16) + nsize));

  ehdr.e_entry = (length << 16) + nsize;
  ehdr.e_shoff = base;
  
  for(i=0;i<length;i++){
    content[i] = ~content[i];
  }

  lseek(fd, 0, SEEK_SET);
  if(write(fd, &ehdr, sizeof(Elf32_Ehdr)) != sizeof(Elf32_Ehdr)){
    puts("Write ELFhead to .so failed");
    goto _error;
  }

  lseek(fd, base, SEEK_SET);
  if(write(fd, content, length) != length){
    puts("Write modified content to .so failed");
    goto _error;
  }
  
  puts("Completed");
_error:
  free(content);
  free(shstr);
  close(fd);
  return 0;
}

```

这里就不做详细解释了  

我们在上面加密完成之后，我们可以验证一下，使用 readelf 命令查看一下：

![](http://img-blog.csdn.net/20151121113929510?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

哈哈，加密成功，我们在用 IDA 查看一下：

![](http://img-blog.csdn.net/20151121113936017?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)![](http://img-blog.csdn.net/20151121113941660?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

会有错误提示，但是我们点击 OK, 还是成功打开了 so 文件，但是我们 ctrl+s 查看段信息的时候：

![](http://img-blog.csdn.net/20151121113947720?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

也是没有看到我们的段信息，我们可以看一下我们没有加密前的效果：

![](http://img-blog.csdn.net/20151121113954100?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

既然加密成功了，那么下面我们得验证一下能否运行成功

**第三、Android 测试 demo**

我们在获取加密之后的 so 文件之后，我们用 Android 工程测试一下：

```
package com.example.shelldemo;

import android.app.Activity;
import android.os.Bundle;
import android.view.Menu;
import android.view.MenuItem;
import android.widget.TextView;

public class MainActivity extends Activity {

	private TextView tv;
	private native String getString();
	
	static{
		System.loadLibrary("demo");
	}
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		setContentView(R.layout.activity_main);
		
		tv = (TextView) findViewById(R.id.tv);
		tv.setText(getString());
	}
}


```

运行结果：

![](http://img-blog.csdn.net/20151121113041585?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  

看到了，运行成功了。

**案例下载地址：[http://download.csdn.net/detail/jiangwei0910410003/9288051](http://download.csdn.net/detail/jiangwei0910410003/9288051)**

五、技术总结
======

1、Elf 文件格式的深入了解

2、两个属性的了解：__attribute__((constructor)); __attribute__((section (".mytext")));

3、程序的 maps 内存映像了解

4、修改内存属性方法

5、Android 系统如何解析 so 文件 linker 源码

六、梳理流程步骤
========

加密流程：  
1)  从 so 文件头读取 section 偏移 shoff、shnum 和 shstrtab  
2)  读取 shstrtab 中的字符串，存放在 str 空间中  
3)  从 shoff 位置开始读取 section header, 存放在 shdr  
4)  通过 shdr -> sh_name 在 str 字符串中索引，与. mytext 进行字符串比较，如果不匹配，继续读取  
5)  通过 shdr -> sh_offset 和 shdr -> sh_size 字段，将. mytext 内容读取并保存在 content 中。  
6)  为了便于理解，不使用复杂的加密算法。这里，只将 content 的所有内容取反，即 *content = ~(*content);  
7)  将 content 内容写回 so 文件中  
8)  为了验证第二节中关于 section 字段可以任意修改的结论，这里，将 shdr -> addr 写入 ELF 头 e_shoff，将 shdr -> sh_size 和 addr 所在内存块写入 e_entry 中，即 ehdr.e_entry = (length << 16) + nsize。当然，这样同时也简化了解密流程，还有一个好处是：如果将 so 文件头修正放回去，程序是不能运行的。  
解密时，需要保证解密函数在 so 加载时被调用，那函数声明为：init_getString __attribute__((constructor))。(也可以使用 c++ 构造器实现， 其本质也是用 attribute 实现)  
解密流程：  
1)  动态链接器通过 call_array 调用 init_getString  
2)  Init_getString 首先调用 getLibAddr 方法，得到 so 文件在内存中的起始地址  
3)  读取前 52 字节，即 ELF 头。通过 e_shoff 获得. mytext 内存加载地址，ehdr.e_entry 获取. mytext 大小和所在内存块  
4)  修改. mytext 所在内存块的读写权限  
5)  将 [e_shoff, e_shoff + size] 内存区域数据解密，即取反操作：*content = ~(*content);  
6)  修改回内存区域的读写权限  
(这里是对代码段的数据进行解密，需要写权限。如果对数据段的数据解密，是不需要更改权限直接操作的)  

六、总结
====

这篇文章主要介绍了如何对 so 中的 section 进行加密，然后将我们的 native 函数存到这个 section 中，从而达到对我们函数的实现的加密，这样对于后续的破解工作加大难度，但是还是那句话，没有绝对的安全，这种方式还是很容易破解的，动态调试 so, 在 init 出下断点，就可以跟到我们这里的 init_getString 函数的实现了。关于动态调试的知识点大家不要着急，后续我会详细讲解的，所以说攻与防是永不停息的战争。下一篇我会继续介绍如何对指定的函数进行加密，难度加大。。期待~~

PS: 关注微信，最新 Android 技术实时推送

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAATEAAAEvCAYAAAAtufaDAAAgAElEQVR4Aey9CaxlWVX/v999c1X1UD1U09XzAEgjgyCIdPxJfr/8STRRCU7RKOTnD3EAZRBBkVlERhlkULEhxJgQg5IQE02McQiCMggyNNB0NzTQI91dPdbwpvvPZ5/7ubVq9z73nnvffVXdTe/kvrX32mt917D32fcM+507129K6vf7aW5uLlPqvV4vt7e2tjK1L6U0lIN3fy5lTPpuLLHfWOAZP/LKqCOG/O3ELwaUTyz6o73Yp55jQ5919ZBhDCnKR4zIR5dijKVuGw5yFuvRvvWILU9/a3o1+8qJBRUj1o0BHn6jp014syzRp1jHBjb1pWZfv6ClrvqR1vyOerGu3ij7NbwHKm8hJjEmIvIJruwrefe3BOAvg8hEt94Wg4OtnLHU5GPcsV+dLlS90j91ox9Rlrp9Ljy29ctYaMeDPMopqwy0VtChQNWXyseecYhBuyz6q55UWftt2x8pdYp+WY86jUTz1/ijz7G/a1197WKPunxwYp22tpEtYxNHOagxiFPagN/WN6l95B+oxTxF/+c2Nzc5Fxt+a5Nw2haTKYVvvQao3ommDrix6LNnFaV/caKpU8YqX6ztxN/mn5hQ61GWemwjQ1tf1VMuxhvxjD8ebPDi+Ed56+Ijqx/qiak/tqXqRj369JX+qGsbqhx0FN94xbStXgaa8E/0ibr+CKMt2varU/oaZaxDIyZ12+IhU44VPMqk9gdqDzhCThjPmBOCWDBZJmNzc3P4zfGAi7KDw0tLS/eRihPuRMbPWCwsLNzHvzhGo/xDbnFx8T6DXAIiZ8xOCmTgdcEXT7/Egr+xsTHEVk6qf+rJj/rYj21lutASfxYLWBe7UWY7/uPv/Pz8MH7iKXMVbcU6OUN2O/Yj3v2xTm7IUZmT4RFD8Hxe97rXpc9+9rM5mffHQLbj08rKSo7v0Y9+dI41HsAk5t57702veMUr0te//vXhmel27HXRdfIhe9JJJ6XXv/716dJLLz3mi0SZAwcOpJe//OXp29/+9n0GEv2TTz45veENb0gXXXRRjo+YHPDawkAffPFvvfXW9Pu///vpxhtvHOrFGE477bSMf9555w3PCuwnlzfffHPWv+WWW2QPKTbOOuusrP+whz1saFPb6H/nO9/J+sRZK8hSjCnK0Ld///70x3/8x+mMM84Y4htjlJ2mLg7UujjY5gBj3jB/7rnnntxlbMqVNPZzlnXJJZekP/qjP8rzABsW7ZEjim37weHL78tf/nJ61atelQ4fPpy7Ir6yD0RKvKxNT3jCE9KrX/3qnGtyMczR1tZWf2Njo7++vt4/ePBg/2lPexoz5UH5WV5e7v/Lv/xLfpRBzBTjp33rrbf2H/OYx5yw2E8++eT+Zz7zmezX4DJ/WKd9/fXX9y+55JJW/0499dT+5z73uazDeKJDfH5yx+CPPGSQpf2Nb3yjf95557Xin3HGGf0vf/nLQ5/E09drrrmmv3///lZ9+q6++upj9LGLfcqVV17Zx8a08+/888/vX3fddTluYzJOfd0J6lz61Kc+1d+zZ8/U/j/60Y/uf/e73x2OW1dftf9v//Zv/ZWVlantT5v346XH2sQaRbyMqyXf2I+r8+7du/MKx8rO6vdgKK7YxLa8vDwMiW+qWFjdj0f8fpPql+1du3YNLydL35DBvz179txnfNTH93g52vZNHLGtQzmbAN9vOfsiPpertVLqg+X8UR9s/BMXah1M7d9+++33Oduo2ZQnPmeyYFjApo8S6/Z3perqq/bEFofYGINDhw7JOiY+mDVd8s1luGOLXLSpjqC2S/vcKmEOxUt6fVb3gUgZ0zI/MY58T8xASYo3dpmA1B8MxcEmJmMlLuO1Dj2R8XvQR3/KujHE8SEO4or66HFw1IryykQ9caOM9VHzARmK+tSVV187tpGJ4xH1owz8UUVZ8ZWFb4l1eV2pulL02ur4UPoxzk6cc+K2UbHstw11DmC/zGuUeyDWicf5VPrfIxkxIbFeCj/Y2iSmjLdsH4+YselnO/YihnUoRSq+bahnXvKUOR40LrSzsh9jqsW+U3FhN3662plV3NHeTmBG/PtTPV9Omvj7k2PHyxdjf7B9czmJXail5jW2laUv1pWN1DxBy7rtKF/W1YNqS9pFv8Qr2+LDF09ays6qLb50VrgP4XTLQH46SfK7DgATzknXzcTxlZokFjxDfpbxxDOLWuRtp8Q12e3wHE9iq8UY+fZDrRMHdYqy1OF5+WMskYesH3NBG1lx2vS1lY22/BEzdkd8+PotjbI7UceOH/D1sWbfnO2EHxGzZttxiXL3p3rM4SR+DfeJEWCXwo1dbrTVktRFf6dl1tbWOvtmzLOaWOCRH3FrsR45cqSzfzX9rrzoQ6xH/ZLPmMLj4QfUMbbOuJMrYqgV+jiA48MT5NSnDxnGqCzKQNsKfbX8Rl3uB9Xw2zBnxV9fX8/54ua+exHNnzZoTzI/1ZuGmpOoC0/fIv/+UMe3acfumH87GhUMRphAr33ta/N+DZ4W+I0zSu949TFB2B/DPpkvfelLwwOni/3agHfRK2XYp8U+pYsvvnh45oEMvmHj7rvvzvu8rr766on8K+3Muo1vfPbt25fe//735/1yNRvso2Ofzl133VXrzk/X/vAP/zDvV6sJoPeyl71suI8qymCf/VXsNaPEBYA+2ueee2564xvfmNivFov5ZX/Zi1/84qH/6IklRtSbdZ39ae9973sT+xFjwT+OlWuvvTbvg7vzzjtj9zGxHtMxo4axX3bZZXmfpP6ZnxmZmRqGLzYW///+7/9Or3nNayZe6CdaxPgm/uEf/uH0oz/6o1M7vJOKnCG8613vyiYcuGivbdDiARPlu9bF5VuO3DBZaoWD+M1vfnPuqvlX0zkePP1ncv/Ij/xIq8mbbrop/c7v/E667rrrqjJnn312+pM/+ZN0/vnnV/u/9a1vpRe+8IV5M21VYAyTLQhPf/rT82bWmug111yTXvCCF6Tvfve7te4d5zHu7373u9Opp55atXXllVfmMyHz7bzbibkgNo6IzyL7Yz/2Y/dZZKvOngAmZ/CsMfrb1YXh5WQMuqZMPx++LaGcPmPw/lD07eDBg8NLCXhlKXm2J01aGy7fKOQHypmq+eE0mW9i/CNvFG2X9RL7eLfxtSzEg/+c6bLQUY85s84kZI8UGHxi/NTb9KM9bLUVMDkbZJEgf/hBQUd8fCj9a8ObFR97+IZtYmSMyQkfCn2caTg34tjPyocSxzGBrz0uZZmDjov5K3WPd5v8cJXH2DKW+tvVj+G/HY1TcEBIAHWoyRinu9P9BM1H37AXB1H7xmBbuWkSV2LYxgcmBx/qDohtfYj+yRPjRNLamOKf/pMrPqX/xAkffTFi/NTBYMKW+pPEKz72wKPoH+2af5PgTyuLP3z0Dxz8gWedPnzlQ6HPembs0B9saKv0b4dMTgyLj+QLqr9dQTovYqMASZBJUs62DkHvj6Xm+6z89GA9EbEb1zSxzGrMavGfiFxMk4Pt6nTNv8fJdu1No49ti37MauzFHUW1OUqmS9+2FjGTUFINx0ls3b5p6SwPArDwa1alzbfIj3XtwjOH8mjLK6ky09Ca/YjTZjfyo3ytjix59ZtVXailFrN9o2jEEq+ko/R3si/mVj/hRf44++q1yU2C1YYhH1sRb6fyGO1oY9K86HONbmsRwxEXAZNhG2M6XNKaI1158cDQZlfdUs7kQuMHObCx1aXohxilDnyK/chHfNtQZaAxl2JGWf1TTxvo1XTpV0e8SKOfJR88+7EHTmxT149oW73IQ1eMaKetjmzUxxYfedaR40ORUo++jYq/zf44vnb1S/v6AH9UUU6qvDTqcjkIX9nYN2ldjGgn5nRSvFHy0QZ2YzvWR2G09W1rEauBckOb+x47UQiem6dQB2A7dsBg0EiieNDV1dVhexJ8bt7WJgGY2NAON8e1F/GZoOjHfyCO/W5rKfXFxT655+ZyrWDTfUL6hFysc1PaGEoMsPGxzT59yLgXroyRPnxEf9JCjOCTg1H71Eqb0Q59xO8CPEo26nWp4x+lpNgo7ShT4iJHfG373IjfL4BSt2s7+kI9+kJ9J/ex4T838C3Y2248YG1rEcMJE8HEJwGvfOUr0+c///k84eDFpOn8NBQcJiD7kB7/+Mfng4WkbKfov9/OPMJ/5zvfmdjHU/qtLJRi3FJ4DNAjHvGIvAiIqTz9PFXjETz7xWr9LD5vfetbE+/1Kgs4LK6/93u/l04//fThghjxefLE+8TKt0Do4969e/MWjwsuuOAYHxkn/OE9YL/7u7/b+j4x3tDA++ZOOeWU7J450gfi4n1abCWRpwwK7KNjiwl5puiX9cys/BHrjjvuyPvAeMoXcemnzdMtcxcXYu1ceOGF6U1vetNwC0TEqJgdssTXD/GkCNJHfOSGuuM7BAkV7YrH4k756le/mn7+539+eKArByWehz/84XmfnG+7sD9Aj61qE0FzhH3m7uc+97m8z7LtS2IseIsAfmKD45bj1y8SfYg+tUCMZvNeHt4HxTt6Dh061P/xH/9xjtJ+r9fLlLpt3lX0D//wD/k1Pr6viQa6tO++++7+5Zdffoye+rOgi4uL/X/6p38a2vd9QsZw5513Du1H/+fm5vp8eN/WJz7xiawW30mkvu/FEncaGrHA8wOfz6hy++239y+77LLW/O3du7f/la98pRWCd1FdeumlrfqnnXZa/4tf/GLWd/zwyVx885vfHPk+sbPOOqt/7bXXttrnfWe806ttrHlXGTLTFt5Xtm/fvlb8NruMPX2Pfexj+7fddtu05jvpkUs+jLtjTpvy2c9+Nr8vjbmpTyVtiwE+77rjnXdi65D4zO1TTjklY4uLnscCx+Ydd9wxnJPOzSNHjmQoji2OsVE+bKcP+6wR+Gtu8GFtbS3bZ21ZXV0d+qst/WdtYo1S3/inPhPzW4BVlLrUbwkvbaLc6OW0vRcMPpwJcDlJEbdda3SPqz801jmdt13agF/ytKJOPDuE5zey/Xz7US8/yPEN6OWy+YuxchmGDBheWtJPG7+4DEUfLPTlK8O7pkr/ov/oMX5QPvoMNljY52yRb1U+yIhNnT7tYwcdCn3Iq4/v9OkLdsyrOrbFRxZ8MMCjX/+QsW1es+HiD3Y5W+OMCTn9L8Tu09Q/7WlLQfm09Y26esainFSZSGs+wSN/zP9aP/qTFn1CzzpnSNhwL2j0c1L8KM/YkXvXhtgX7Zf8ru2pFzGNx4EiGUxCeFAnZFdn2uTA5cNAbjex6jtw0jIOfFE2xhonkf1gGGuJJ46y6CsPjw+6DDR91OERa8RUDl3klNc3ecqZK/0BS2x0KPQhbx2qXfixT1zs8tEPdJig+qMd+qM9+fpJPzoU+mjHUrbF1z/69Q8929Coi4x9yIHDBz6+bKdoH3xiKIt+KEd/6Y86+ljDUa+tT4xJqH5EHXjmlzqfWRWwauM8C/yJF7EYWFtdxxwY2/cniu9OMvyyHXltdWWNJ+ZBLKj60lIevn3SEksd+6U1/NgHTsSyHmXEULbs07ZUDNtQeOpJ5UPhqSeN+sqoW1tY7INaLzFK7LJdyk/SBgu7YlqPfOv2RXz14HX1P+rvRF0/9HuUb7OyH/MwK0xwJl7EDN6gcSwmQufk276/0uhnjCPW8Z22xRxA5VuP7VKePr9NxUDGOv3qS8Uo25GvvjxoxIp86+ioV9aVKSkLjDqxr00fH9r61G+Ly/5IJ5GNflqXRswudfWk6MQ6bXITc26/Oehi50TI4OckeT0RPo6zOfEiFgOO9ThoGLU9zoET2a//0Vd48vWNtoPtgSxPmYghTxox27Bd3GpnIuC04cNXV3uRln7aJ17pm/L0K6N9+kpbyIghNUcxlljXB+WjHXhlgVfaLWVsiyVFlzofsaXqTEsjZsQobc/KXrQxq7r5mRXeicCZeBHTSYKPCXCg4gAqe3+kbf6yTYT7ApYYI7FxY9kYowz1KEsbOXna4/5RxI96/uNw1KVOUX/QHBL5pU/aHgoWFfTih0WGNveLeABQKzws0J79YmgPfbaClAW5Ur/0mYWKm/c1G8iWeSttEIMPFuwDS9+MMfZRL/2wfxyNflqPWNS1PQ7rof7pMzD1IuYAOWgO4vSuHF/N6Lff8jy5eulLX5quuuqq4eVB9IqDm31GvHLFg0McaZRXxtyw/4x9WLxXKh5QyrGA8q4xij5FvLa6+tpBLtajnnwpfnuj+8wzz0xXXHFFftMBfH0wNhaI2m9O0g8evyf5wQ9+MD8lLfXBIn/IUOhnUVIXH/jdyZe85CXptttuG/KNhXwxPr5mR//Fos0rgN7ylrck9sPR1gd0sc/TMV5HA1+9XJnyj76LJ2b0TT+mNPGQWocMTLyIlQNW2ogDWPbdn9rEwYfJ7UTjTOgTn/hE+uIXv1h1lYOQDZeUGKc5kZbKyPJhkfr4xz+evva1r5UiY9sltm1orAsEL/oY+dah6iLLIsX74saVEldbnKU+9alPHamOrvosLhQXSxapf/3Xfx0uVCOBKp1sD3ja057W+r4xVbRve1oqTqTUzSm45mZaGw8GPfMhnXVMEy9iDhiOlAMGb6ccnXXg+K7/+sxB5V4Wzgw8uOxnEYv/NqFP4kDNQY0HPgcaeO7jEkPadskktnK2odqyrws1JvXRoa79iEmdgg4xQNWzLh6XyzVdfSKvypaUPnIcbainHcdEfqT4zn8tRB/0RYqNWORHXte6PiFvPeJRt68r5oNRzjxIZx3jxIuYE++BPjjEwceDwrhok2xora82AOpKR+VGfA447DiwUbdmo+RF+ViPcvDFj3zr9keZuIC04aJvX9SFr742pMjxiXqxrZx51zf5XSmLFJ8aNjyK2MpIu9pQTjzaca7I147y36vUMZfOOg/b2+03a2+OM960k3dWbjrZp8FDN+rHunhdeEwsJ5dUPagHZ8SM/ejQ9qNcjUZ8sV3UkadfGWkNZxxvlC/ahWJDOgoTGQq0VrcPGv2mro0M8NCfHcnAxGdiO+LFCQadZKK1TeK2EMqJX8rFSW8fZzPl4kFf6Wf0hX7b4tRotIc87cir2Y08bahDu8YrbWtL+TaKXvSJOrJtOYl29Ame+GV/5BMXOsYX9aOeePLAsFBXL/Kjjv2Rp/5DdPsZeGgRG+QwTrQyrXFyImdbWsrHNjJRrrRT9nGwUqRiRb0Sk7b9UvXG0YilL1KxbJdYJZ+2OsqW7VIHOWQiXx1zULbFLimLUcQp++mz3zo25JXytumPPsY6MnERVBa+df2HF+viP0S3l4HvuUUsTqJYHzWRlYN689hv8a7pB5+nk2LV9OjjCWF581lZHipw05qneB4g9Fkv95mpN4piM8buP5iP0mnrY0HAf4pxRmz4tX1g4uE/MXJzf9KCHfehaVsqFjLRvr6VcspPQsFi3PgnaooLMPVZ2pnEp+8V2e+5RcwJFSfXuAVJHd6T9bznPW/4FoRJJj8YHKTsQaP47U1dHN5FxU+eXXTRRblfvvZ9Xxj7qGKhH1kWIPagUSJ+lLUesdXnXVwvetGL0g033DD0SfkulPecvf3tb8+/D2lOsSM++8DAL/0Xm/etEb9vmdBH+0dRbPDkl31g2I6LCG0WmG984xv5fWT+7iM6lEnstPmADX5v9G1ve1t+X5k+gC2+9towHuJPl4HvuUUsponJxWTzIIt9tTqLED/wud2C3WjTNmcx7LPixYq1wssO+d3HcfvMxCsxsFkeSPI46DlL+Y//+I/E70NOU/jxXRZa7UNjnT72yfkDuaUNXtbIj8/y+5XTFsaTgt2y8NLGf//3f0/8yO5OFOJijlDKPGuvjW//Q3TyDHxPL2KkywOta+riN3xXnSjnohF51jkAvVTkslFbTnzfF4bPnFl4wKoPVTbyrLfFCh89MN3Hhu1RWGJC1ecyEAz04FEiBn21fWDY4jKdRZyzSerEZvzRVq0e7dmvfdtQLjex75t7o29RbtK6/pM7fdY+NqId+ZPaeEi+PQPf84tYe2rqPbWFoy45nhsnt9IcBEx0DngPCOXgYZ82dFa+gKct969pR79GUfzVJ+Rs1w5Y/VYGeXT94Adx0q9Po2yXfeBQIn6UqdmP/dPWY/xgGA9+8Hmo7FwGHlrEdi63nZE98KSdFacQLG2UB9ksD7iuWMiVfk0RWlbR5qzwtuPH/cWXaWN4oOjNfBFz4KDWt5sMsWaFt11/xulP6qfynHl48EFjHZu0lYVa7+qP8uBwRoI9eNrSHnjKRuzYH/mxLpY82tjSjvxIu+AqX5OFJ75+21ZPCt+PvO1SbYKDL9HH2I787drcKf1Z5wY/Y352wu+ZL2JMWMusnWcSRHzt3J/odiaBk9yJH9vGKI/2JPmNshHDeqTKSrUNVS7yrCPPRxmpejU8dUdRcNSNmFFHvnL2xTYyLqj0xz7lZ0n1CUzq2ov8WdrrihV9iTrwo2/6G2W2Uyf3EX87WFF3posYDnI/w3+ijoZmUectCZadSIbY01IGnXdpkYNpCje2uaHvDzXESUS87DNjH9VJJ500DXy+cc7TMx8elPjcVGefE+NXK0xCHi60fZGAV8PHd/uizZqNGk+dUfade9y4V14aMckxN+B3ojA25JCnsLEYP2M7TUF/lqXMi/jQeIzNyib2GLvacYFN7U9rbyaLGE5yacIBzPuc/N3DaZ1q0yMJ3//935+7awlp0ztefPY3sU/q0ksvPeYbv6t9Jv8b3/jG/PuPUYf8MtAcoH/wB3+QeO8X7dpkVBZ9+5Vli8FrXvOa1n1aLI78rmT8XcvoB/u7XvjCF+YtGNGOk/DGG29Mz372s6sHAvLEhwxFnYjfVucAYH6x9QP7zi8x9OWcc87JvxtKfijGjy6FfVwf/vCH85dBZszoj/lli8VznvOc4fvU9A8z+EL+eTIa+V1cML4usl1k9Fe/eGpL+YEf+IH00Y9+dPgmky5YXWTwnzE87bTT8hpB25gYm0nzUdqcySIGqI495jGPKW3sSBt795eiL5zFPPnJT06PfvSjp3KNg/PFL35x+spXvlLVZxKwSD7qUY+q9o9j8uO4L3jBC9I111xTFWWf17ve9a50ySWXVPs5SFlI2wpnIZ/+9KfbuqfmO8lZBNnH5g/kloD8OC4+UMoDFR6+P+UpTynVZtb+0pe+lP7rv/4rL1QzAw1AzrPAmqhqHqHmJ2Ky2fryyy+fCHM7wtqWTos1s0XMpLDimqxpnRqlx8pN0Nprk91uYtpwa3zjJXYv1eI+r5pO5KnPAcjlDr5zpgkexXjp45ISPlshPMMosZRXF3l4bfjgIAM+MiU+bWS4lBx3SVTzKfoH1qQF3ynkhIUIG7UY7UNWHerOFeg09rPxEX/A5GyGseFyjNsB2i3VprHv/JCWmF3b5gRqXUzaO5Uf/cMGY6dN+WVbflc6s0XMpIybxF0dGyenvTY5EuPAtMnMih994UCjDe2aCwfRhYs2k90Jbxy0wRRXWsZh7PLR54M8fXzcD4aMPPCj3+Kra2zi1qg+1/pmwQOfjzkB03q0HXNAv3LEMOsCPh/ypX/YwIdZFOObBVaJYW7gU9+J/JQ2teUYRR9qsuN4M1vExhk63v0kJk7q423fhUG7+OOgyYNOO9GjXrTlhNCWffKj7Qd7Peaoa6xt49RV/yG58RkoczzNOEUrEy9iGvTgiGAnqq4v+qYfJsv2LGhtMZA3yl48e9BP5PnYN6l/xq2euLSp65dysd+zBmSt638pH9tiSrUtjTbkRVrT0y5y0ZZYkYesH3HVl6pnf6TIjCrqSkvZcfqlfNkep99mt8SZpg12/EyDMQsdY5R6siGd1EbnRUyDXmJgSN6kRmctH/2IkyTya/7SX8psxzexxC0HBf6s8hfjxGfafKJN7MmPvkVd6vaVsUe+uGKWsrZH4UW7yktjX7RlvzT6JA8a7SojVS7akBep8tA2We209Ue8WFde/dh3vOrMPf0w1uNlO9oxv/qgT9Neyi4AJFg0VNYxxId9QLySRkdKuRPR1n9uPI8qJksZ2urKG0WR5+ZxW7LZYsLNXR6l13CZRO4jo58PmKVfo3ywT13bUHjkgPtdlhijN/bZShH5ynJTOvqvX9o6fPhw3kfWtk+Nxcc9UuroF1h84s137Up5uouPPhwp7XvjvM0++vjojXX19YFxI/8U+kofzR+0LPD8B/KIW8qNaqPnP9jX5Bg38lezX5OflGd8UW/aWCLGpHX8MP/o0mZsfANIG15bXhYMok1AQCYoE+xlL3tZ3u9BW11lTiTFfybBlVdemd3w2zz6VMZYtqNsre4+MF6VU8YPFgcQWyDcC1VicPDR/33f931Z37MydCfNJfLqoM+HfVy//du/nfdTgQ0vFjaxvvrVr877zOCjH2VYfF/xilfkfVhRzzrxv+ENb8jjLw8KBlj8JiT2eW9YxLbOPi62cJT73NRniwn7wPiSrBW2mKDftoix9eL5z39+dRFjvPjNTPR55xg2zT99HERf//rX8xaUmn3k2cKBPn5QiKtLwQ42HvnIR+b3pcXNxGJAsR/jx+YsCrYpX/jCF9IznvGMHKt2Z4G/HQxjJEeMP19U8krcNp87L2IAkgwXidLA/a3dlgj50i5+mzy+iZ/4xCemxz72sVU1fpOSH39lstQKiwALhUVc29NQMIyFL5lPfepTwxcjlngsHuxha3tf2U033ZS+/OUvt+rzw7fsA+SljbWCPmdabYU+9mm1vS+MlxZ+/vOfb/0SwC4bMv0B3tIOiwDve/MHdst+NrtykJh38hbzxxncJz/5yeFvi5b64MaDTJxSrq3N4ss+LOZBrcBn13/0qSY3LY+5xz62+3OZJvbO98QMnBVTQ5MOohg7SVloy8lpG7ulz8YyyicXCbA93efUl1zQZz+LiJMQirz2kOFSxt3R2INn/yj7o/q0jQxnEywUYGIH+xRjZB8YfkLkVQIAACAASURBVMOHaps2uhygk+xTExt9ckFuvJTVZnZg8Ic+95qhowwx4C/2uSQEiw8y0Yb72NirZtxggKs+MsQSx0YZckMfuny0r49gMEacicU+sLDBpSD69E1S9JWcE395SQs2fnMZTX0nCzHqD3bKPBj3pDFux+fog2New4t+x/7hPbE2gShMfZSRUvZEtmM8DogUv6xD+UT50u8oy4Sm7YGCrLrxACFPfNS1LTZ8++RNQ8HQPvrYoc3BQJ2iHdr470c+beOJ+uJqI+qLqwz6fGpFffqQ4UBiIZIPRrSvD2Irp330qcOnQEt9+OjzifpZYfBH/cgDV2ztiwV/mqJ9/cRX/RKbPudWtDuNvVE64zYrj9I90X218cKn4SLWJnCiHZ/GPrGUE6HGmxZbPXNW2qIfXsmP7dgvXyr+TlJ856CZ1GaUN35pm79RRxl0xukp20an1dc2fllvszELvvG3+Stfqvx2bM8CYzv2j6duXsRInt8yJhJq/Xg6NCtb0XfrHLQWJzBxW7cPGuNXXxrl1C0nDe2SF/UivvzIK+v6pGwb1ccYK7zoZ/TNPvAivw1fGc9Wohx9lliXJ6Uvxhf51tto9FcZeLGA77iWfPr8qEebIo06k9RLffEZC86MbYPp+MCLerTtk6+edJRPyPhRf5T8A6FvXNzDG/sEQ9A7fU1+opJGIpjYfCwOMn3U+cR+5aSlPnxxrSsrtlS+VHu0lQG/Lf/RRzGiLnX9GyVrn7q0rUNtZ2bljwdYxFEf3ih9Y1ZXavyjdCuuDFnqDxktFfFLPxCHR2xRpgajLn2lrH3w+dgWh7GVB8WebWQcP+VLSj/yFKj2lbO/5CurrvIPNGr+Sr/vc2OfG9I8JTHBpQJtB6hMVk22xjOpZV8bv5SrtdWtDZR+ckPVOnLKMvjyubFcxm+fN82jfTGQsU5/bKsPXxlorCvDDV7eJlDqI+sNYXHU1x900Md/8eyDos82kNoWAvrpG1XARMbXyZQ24j49+mr+gQ/fD21xSvlRvtiHjvrySspc5gCA4n98QqwsGDH+mv/Iwteetm3Tzxyp3XdClrExRuaceLkywEZfGam+MDct8qJt6nEfGvqxH90aT8wudLv6XWzUZMgX+auVfCaGYwwwSXrTm96U92uMWsRqQJFXS7DBQ8cVEx9lI2YNq+yPNsTjhqqvsSE+i3Uegb/vfe+rTnJkuaEc9aN/Yo2j+gK1rg77j6644or8lIq+Ep/x4TU58PE5ysDjPWAf/OAH81NC+6TY4OBlnxj7qSI2MhSeDl533XW5Hvuts4XiWc96Vl4M5WXhwR/02SNGqfXD1xbUepvsAHYsUV9aKrhg8Aqin/u5n8vzvJTFF54O+puYZb+Y8O1z3shj68pHPvKR+2zaBBsf2B/Gb2vSjk8JaYNx2WWX5fd5uQhqx37mZ/yCgm8Bn3ft1d4HBo4Yym+Xalsft4s3Sh9bxMfxUVvIhjf2PSg8SEeBPhj6HAQpMTGxxr0PjUGLA4d+yZskP+iKwVnw4x73uLHq0T7CxsAi9/jHP75Vn/eB8a6yq6++ulVmVAeLFPu4xhXjKeXMU1u/cZR6o9pRJ9ajjvniTPGzn/1s7Np2HWyOHShnQU960pNGYsYcIEjbfLDIjdMXXBza2meR431232tleE+MlY5kQikmd5KETKPThj8pFvIUJ0RsRx5nYxb5UAvxqysvUr995SkbMeyrUeWlUc8xoI9P7BPLL5soIxYyfIuXurbdhwWuj/nFVabtnpxyZfzypeDwqRXs8mnrb+PXsORNquP4t+k5/8UvaYwBDOORlvkr7UT79kUc9cGrFfOvPWVsO3/b9JWfNTWGWeNGPGIy/sgfLmIGXROKCtPUDdBBM+HaBDPKUI9+RLnSvpgRwzoUXe1FWfnKQCmjbA1EhiTixToCsR3r4kvpsw417sgfGgwVdWShhw7UPjGkLly0nezqK2O7jY47yNv05GsHarGu3/JHUWTVGyVX9rFITKsrlnZrOC5SypYUXfT4UMq242+/VBzl26j6NXl44KmrzHaoWFDrJZ78WdvWznCLhQyoRqHTlOisGFDr2iixlYE6eOqUmLbFUNd2G416YkvbdGp8caCUEsN++pQpcaLPyotT9olhv/JS+dFGjRf7j3d9nD/j+vEXGWJ2MTUvOxmLfkG1L0+fJrEvRtQRr+yzLXW8azTiRb/Q5WOxDUaJox1lI7Uv6tDvWERcZemnHkuUi1iRH+XFKP1VZnhjXwY0Akf+JHUx1ImBEDT9Bq8MVEetj6PgokOJNmLdvigrbsnLQB3/aFcqJjR+I9KPncjTLn3qK2cflGLbOlQd+pQTS/koI7Y8aTYw+FPjxX7q2ir5Xdolfg0LGT9i0jYmeHHewI8f+kt9caT0T1IYt3gGp31oLYYSW//hG0fNZ/X0P8pGDOslRV+eWNq0D5+VsU5bOXmZUfljvxjGL0ak+u+8N2ZhI9Y4+2J5pmsbvbzFQsOCQ2u82D+ujhFxcNZ6NB5tKJ8FJ/yD7ij92BfrE5o5RjziUPdjrGU/yvTZT+x81Iu5QBZ+yYOvjs7U7ESZiIFsPBjF6ELFifa66JUy4sCPfoLLx/zU9KJt6hELeQ6WErOGU/K6tLUlbdMp/dIfKXoxjhInysU+daTi6I8UfomhT+raruGjG7GijHX7xdNe1NWGstKoIx408pWN/dQjP9bvs0+sVNxuW+fAoc4/CXODOTqhDXj+k63y9tWo2OhRF99/bqYd7ZT4NcxJeWDGA493cvkYvcSKe3jo02/lwNnOPwFzEGPDb77SBvbYK+Y+NO12pSyAvOkhxttVVznHjLZjFseJfVLkryzIkFvmT7lPTf1yn1qJQV54AhjzU8q0tbHB433GJ+7XivKcJeA/hVxT9A1K3sgfdUvs5+k0+ug6N6KseOoooy0xS2o/44f/JWYpP20b/+NeTHD0FcpDJ9/1No0N8MtjCJy5/k5FNAjAQDDDWx5e9KIXpc997nN5ImnaJDM53vrWt+bHxCTcG9H21wIXA8oA8ZNk/HQWE9U+9TkIeJ8Xr3Ppil+zaUyRUgfza1/7Wh4obUZ9tnDwTikGAt+UYXJT531Kz3ve8xKvpIn+R4xaHV3wOPjf/e53p4c//OHZFw9W88Dijn/xYK/htfHYX4Z/7CXTZpts5CvLHrd//ud/ThdccMEx/hk/8wP/oLGYK16F8853vrO6WRcbLHDo+yVp3NE++dm7d2+GFzfaqtWV4z1xvE+MeRYL+MRAfOKrYx9zmdccMT/Zr6dP4DBOzB32maFf7gUTCxqLGNCyRFmwmXu8pojjz/xGmVJ/krYxsr3nHe94x3AvF3z7iP8Tn/hEeulLXzp8gt7Vhhg/9EM/lNcHYiFn8Ck7fiaGEZ0gmSwwbb9NyCLGtyzFBOtoZlb+0I8sH/C/+MUvps985jMVyWbHuru1xa8KjmFGXer4wIfk+uO+bRDIqxNxkGeRYR/WVVdd1aY+ks/ByYFM0Q51c8Q3Wdu70EYCDzrZZxb/a6CLTrQf5c1ZzAVfMqP2ubFZlfHFj7YCLqXMLTzOcn7wB38wvxSxTX8Un0WI8Tlw4EBVjLkVN6oao8KchfC+N+egfOnhw4eP0fdLyJikyps725EiW/ZzbGEfOztRmF8s5tGufmCPvPE+M3M0qQ/sgwO/LMdlEXNCEZCny6zMOuSqyiQmERQHLCakdJ622Op4OguOyRKLPhYaZXNlm3/E1g8Hsc2Gsdb6yUktP+NcxAfsoxvj07doK/o3Dtd+dPCbMzhzal8Xam5KWfn66Xyo8ckNZ1hc0plD5SJujWc/vhsDcuB0KfhFXvmSwT6+ULQFDl+g5F9MY1IGeTCYg/GLBj54+ObcheeYlpQ+S+yTJ9WuFD7HBMcYcVBin3rT0Bh/qY8Nc8FJCvZZRB3rUr7WFj/mJ8rt6CIWk0zdNoERhIEYKBOhTKwJiE7HupjigwkGWOKLYTvqT1PXJrqxjl0nsX2T4us/tKu/+hDl4fEpS41XypRtdIiLg21afcdVig3q4snHhnX7tI8P5gWecqW/bW10wNdGHKs2Hfjo6YP24Uf71GP+aaOjfq4UD3dq+upEPXlSsaJM5MnXh6jX5n+pP2nb+LEV7cW6MtGHrnbQ5VMrO7qI1Qw+WHgMDkl10KzH+NqSHmWoKyct+6dpg9WGF32fBns7OnFSs4joY9cFZTu2tbUdjK66xGl8x9NuV/+Oh5zx77St476IOYmhsU6gBC1v2sCjvkmUJ50WGz0nJBQ8vlUmwR0lS5/9sT7O36ijbJs+fscYlJeKZXtSWtPXF6j2pSW+vsG3rqztUie22+yj63zwjKmGV9OP+KPqpZ+1trmIONqUxr5J6zEm65HW7E9qo5SPfhszPOvk27PfUncW7eO6iBGU91RMbAyCYLkMnLaAGfVLG+CXvK620HNg1BFLKn8cFScOPjrg6P+kmOjHS2h9ACfaGYerbKkn3jRUm+XiIb8r5na/5MqYavbbxmaUj+LoX8xh1INvDiJffcbPeuwfV0dHv0v92FZmHN6k/dqQYifaIi8UFjKKcrkxgz87uohFZ61zc46naARZFm/awle+lGlrKz8K35vm0+A7KDxhciFu86WNDwaPzxnMGL++o4ePbflpw5XPHihxxbSNDAeQT9AiX30mG0+ASv/sH0ejfk2WrSXc1OZXofCv5kNNT1l0jasmh33y60FTyjA32B4Rn4DrA7jceCf+Nv0SzzYY6PMhv+rLR446DxXAr8UAD/8stPVNXhtVjkUQ++KLIWVrBfZ9+NOGNw0fGxy/2HcLR4yfnPhUNPInsWVcpc6OLmI6C+XA4ABlnxaTiKCiU8jAY48TxVW7dLjWVpeD+D3veU/eR1Tio8fgTYOPn3wYoN/8zd/Me5Fq+DXf4OEfBf/YB8R7o1hQxIBSeF/SBz7wgXygqZM7Ov7h6RN7lShi6js89ln9+q//euvvQvI+sve+972JnzbTvy6m8RU7/JQa72PjJ9loU+yD8q4u9kmxiE1a0GcRcHuF+NEG+8+w7+9CljbQfe5zn3ufDZf6qD6/SyluiVFre3bFHrWf/umfbl0k+N3ND33oQ8csVtoBg/nhRmR86locK7Yv8bufLhYxR2Dxk3cf/vCHh5t1J7ExzhewiP+Zz3zmfb7k8YP5yHE/6ZPJaLfN3x1dxHBAw1A+bPbsUtTrIqsdFj5+mLZLEV/aRYdH0+xT4jNN4SzBjZJOsGifRYjfhZxFAVcb4rFF4X/+53/yZlp5kfK7lCwUlFI3yrXV+SZmIzE/Ulsr3/zmN/Pvlt5www217m3z+JJ8whOeMPxx4BKQg4wDnU27tcLiGrcfxLGpyZc8cjfqfWvkn316tf9IKLEmaTtWvLGXPZKOYYnB8cH4cEa8E4WrFDbUEufxLHkRMwka9gCYdBDVjxTsiMe3hvYiX53SZk1G2YgdedZrFLyIKUZNtuTxbcIpP/pMCL+BS7myjTyFPTLlqbz2zUlXzNKGbXyM8cHXBn1eUlPXJnXsEpv+6bO4XSiXMxxAUD6eTYONjS77vMbZGZUf7Xu5b4zwiavNPnLIkBvqlGniR0/9GIf4fEnhAwsl+DUb5Mzxihhd6u4DcxGJ40t82me/VrRPvWZTnjhtPohN3vkiY4xq4wTOOKw2G6P41VfxoFBL8Cigtj5xpLVBbtOFHxMpxih+lBmFq5x0lGzsY3DwqW2goqx1bagLv21APfDVnZZqs9TXB2n0Bd52C+NrDB6QYFKnDxt88G8nJjR2+GADe9qgPsq+Y7rd+Nty6HjjA/5RHCPpdm2jb36pG7t12uZHm9Lt2gaHj/bb8rAdOzGeiDN8FY8COkN7VgFGg7GuDah1+23jg3X7oJGvvvwoN009Yk+jP0rHnEqRrcU3CmOn+vDjwVQeqPHEuTHr8TAnUBZUCnXnvP205e2kP5PE1+bH8B37BiONQU5iqKusyYLGenRUvj7V+kr9NvvqStvkxCtty2/Tk1/DlydFNuLDj/i2o/wofPumpdrTL33Rx9iW12ZLWb+JxZZPm49nRSXOOHx0y6KN2FfaK3WmbWMj2umCo7wU36yjr6/U4Zf90UbUi/wude0wNuA4RujaJz5tS/Qn8mO/GOj7sX8SWsMfp58XMYUMQDoNoFiTUm1KR9mOfdbR42O7tN/Gj3KlTNnWBjplX8Rpi4FJ48RRRqyIZ10asanHOK1HvFI+ttswkWFh8duZdpSljo3YH3Gp26+s/dFH44/Yym2Xcm8GXO2bk1nZiuM3ia/a1z/9EsN+2talykhjLuV1pY5vaV99+PZhn49tZWp+KQMlRzUZ9ZHpiqvOODq8nEQQcJ6ecWNQY+MAZtmPTfaxcANyXGlLBNsgfMJUYojPfQHqFHGgxu+NV9qx8PRFXi0/8HgCif/KRf22PUzI+nEfWg0fv8Go+R/tWI8+WDduZSJFBvv+bFnsox73IJV9tNFHhlcKUdeWdeYW2wjatkBwkPOEzYWutGH85UJqrsC2YJNCn37YNymNWOTftrQLHrLxXVi0S7+4MU7+y/iNj3mF/VKvi31kwGWbgzf+1Yv4HH9thePC3ywtYweD/WFsEYn73dqwSr768RgrZdrawy0WBEgSX/ayl6X//M//zAdK6WgbyKz4PDVhH9lTnvKUY55umWTt0KZI8RP/2Qz5W7/1W3kLBBO+9J/k8j4oHsNzwHAwRGwOst/4jd9IX/3qV4fY2oSSH3/uLE40/WAA2afEe6Holy9G3KdW2kaWg599XLxPrNQFQ3y2qZT+1+TlkYdY1x+psdxyyy3pl3/5l/NTVPsiZfJ/61vfyqyYW+v8LuUv/dIv5SdU0aYY7L/ifWBs5i378Y+tF+zj4r1dtMW1ztaNP//zP8+/rykm1H7Gl20iFPKrfpSdpu6iybj+3d/93XAf1Dj8MkaefjKG+qcvyDFfmXfMXw7kWIiPMWL7EPOLhQYeny7F8WV7zU/8xE9kW1EPHOYT2z/Yx+gTbG3Qh3++j4x5EGNHjjbzkvwwz9t8K3OCH8bPa3p43xn4+hz9bKsfs8UCMN5lNWqvSxvQLPgsYnyTW0wO7TIpZRsZks3vKrb5z+CUE4SYLSxSV155Zd5LJW8Syjcl+7zG/XalmAxUjAP7vLOK/Uy1Ur4vDJnof01nEh6Th/inLejjf1u58MIL84F47rnnVkXYbMsjekoce4VZpHjf2L59+2Tdh5oPaYkj/z6KIxiOEfa38z42TET71KN/fInywtB4DES3mC/MkWkL+CxkbYX5iw39Uk6fOQvjdzsZ51rhLJF9aI5hTWYUj316frnHvIzSoe+Ye2IwGCgAWHknWQ2jIR2AWkyMCZEPRY5PfJ8Y/CirvnplWxz959tA//Uj4qMvvpQEqt8Wv5j6ESl97JMCj4HGh1jg40s8S4A3zj46yMR9ZsYEfqzX7Mlrk7MfSty1ou98UYwqnrVoCz145IaDhLxwIIIDHzn6yJV7zMA3J9EWOsigL26UA8uPevSXMvZ1peiDC8U29bIoE/uIy3wobz/Uun3kgAWg7X1jcR+bOpNSxldfoRR4xBXPwCKufjJ+zEHi4mNRH+oCp45xaku+urSxzQmMxw6yyis3ih5zTwxAnAOgdHQUyHb7DFTb4sE3GOqxxHaUEyP6r6x94IArXxvwlYn60W5bXSwGkjrUCWyf2DXbpQwHbPRL36J9dcRTHn7JQ8/+iFHWxy1SpXzZJkZK9I22+SQvfCgxP9Rt586WP8j4BUWdmIzLuFG1bp9wtqXwrUuVjdQ+bBqb/dqiHftq8YATZcSQmifa2hR/u2MDZg1DfMcOuTYf9a+UxVc+xGzcYPChjTxUW8ZH22Om1DN+ZEeVnoYUov1gLsZnwkz8TsTsIECtx3zD059p7YshFQ8qDzquqDdOblb92NNmF/9G2RXLeJGlLq524FuXtvFKe8ozb8oPsvKiXTEiTxz7Iq35S39cMKL8qLpYo2RG+TJKr+wrbYkLjfVSj3apW5MZxxvu2NfYOIVx/dPixIBH2SiDtg3lo/2IF3nK+82ALerqj7I9aV/Nrn7pq22x1eGgcPLCU145aeyrxQCGOBFDO+JAsYnMqDKuX1wpWNT5gF8W+6Jc5MnXd/pG+WCsyFngqQfFD9vImOuooy5Uvjql/bKtbvRZnr7Yhkb9aEsZ7donv0aR8RNxlY1YpX/2wY+61qViSUs+OBT9iHVk+dhHPkp9cbvS4Y397QJ1Ndgmh30C4mPRJ6nJsT9Sk6Is1LpyERv52M9E9oN87FN/FK3Jw4s+xzpY+qAclPsDlIhnPeZHLPug1vVTGQ9a7dlf0lK/7J+kHbGs1y6RIybxebmjDv3WS334xqSMMaNnPVJsKFtS8SOuutGP6LN1dWJbfDDEkSoHlQdFR70oQ13/5Jc2Y2zKlFRsKf3WxdMfdW0zj6J87Bcj8tSDN6oe+8Qv8cSt0XxPrDRSE+zCwxmeUHCTzoREvRov9nNT0/sl8MGLOtR5QuLNw6hLnRuD3HjkUX6ZGPq5cYkMr6Qpcenn6Y0HkbZLG23t0h742oBSxGSyWbfPfvKH/0wY5ORDeTwfJ1LUdQLzZCsuhNEvcsdeJbchRP1oJxsNf5TDBvuMzFEQyVV8w0fuWaFjjPoQ9yCZAxTFRw59xzfyqUd99MRXTv+gtcL4kt+a/+KDKW7MNTz0jJ92WYgbfPViP/jo+y612GeduUn88RigD1vExD64aDfWkcMub8jg+KNgE5lIc0fInRjIcOwQn+8DU5Y+sH3gIL+NKq9t5OBRIo82uOV4IVPKZeWWP8f821GLTCc2zrAIveUtbxn+biSOxEIg0cGyHwzeeURQMQiDdB8Yj4kZaBOjDey/5CUvGb5TS76UwXnzm9+crr322owvX8ok42fBKNq0bxwtfSEWPhRjQcYcKB/zwRYK3ifGZIavD9QpHCS+D00efDF5xcyv/uqv5r1c2I42qHOAvf71r8/v/bJP3Wwg+Fprs/j/2q/9WuKVOtgXwzrvE3v/+98/fJ9Y9JE6B9f+/fszdFzo8JVYzz///PQ3f/M3eRFDHp4Y2OIgO+uss7K+fCh9YDCu+Md+u7Igw3vO/uIv/iLnodbPlxwbcZEVN9Kvf/3reR+fGz4jBjq8y419bGDoO/rMK+JlHxjvo2vbQkH8f/VXf5W/bCM2GOCxiDGGtCn66ViztedjH/tYdZGOeNSNK+KwB/JnfuZnhl8iUQd54uYYxO6oEv2LclFP+8RVFuSibNlftmeyiOk0yWRD4OMe97jSTuc2zhMYWLHAZzKwh+oLX/hC7BrWOcvgIGGvSq2wR+w73/nOyL0yNb1xvFrCyYl5QT/WbaMXdZnoXd6Hho746tNmRzX75NyQW/rN/ipeeMgYTVPYzMpCUhZs4wdfIvzuJgfjqBJ9Ri7qd3mfmvFLtXX48OE8N9reF8aXw6Me9aiR+8w8qPQJG/qLPnOv7aWOnEEyR40JDIr67tOqLYJZMKW8z3Dc+8aMW9+wg9/M/+3sY2OB4l15xLmdgj9+wME329F3bZgf2sZkXxc6vJyMQF0Uo4yO4SwJoM0BVTst1g5BqRd54NIXi20WtrZ9XMiwj4VLKfCYUH7biwUPGWTxzQlrP7TGi/21uv7FPnwwLvlRjroy8ml7EKAT+WLExR15cegnplp+xOFMiHEhRvIklv3iaUsfkEeGM1kvVaMMehR8Z/xLfPrEdk7YVlcZ2vJibPTT1mfbUIvxQ6MuOvht/MaADEV71MWXh4xyzCcW8RiDOsRO7tXPwAOf1XefFYsZRRvg4ZPzt/QvYtXwxXHu2tYu+tTli2cb38lNzBF99isP1UbklXX1tFn6gbx91sWAH/vkj6LH7sYcJTmiT8OIkGTa0HKwo3PWoWUhCbE/Jo46/dDIR5629rFN3YTqG+2afunDLNoxhugH2LaRUU4ftQ1/XBFHOSZkGZ845oe8cEBCzWHNB3DURZaPbe1Bo655lypX+lnDQRYbyrbJtPHRJ34+0Sfw+Bg/vtEWxzqUAj/qZ+bgDxjmTHm6qMtXXnypMshFXfXhxTyrJ560jU+/+UOGD5hRPtqNdeWNj77Yr+0uVCz0ow/iRX+64I2TOfaabZz0mP7onEnQ8di2Dpz9EboM3HaUifVoN/JPZN0Jre/6KMU3JhzFHMQ++eWEL3MnvvIZcHAglnXtyI/UyQsPOf2PMqP0kSv9j7qxrlyJF2NBvvSplI+Yo+pRL9ZH6Wy3j1iMcxIs/YPWPmApU+JGe+oq3zaPolyJN21bzHI8p8UbpzeTMzGN6LzteCAQEG0TjWysoxODtg6NemKXtMTSF6hFnu1pqHairjxotBf9hq9c1LVOH/JR3z55Ed+6esgqRx0+ZxzWoy/KKo+scvDkQ0t8+7JC8UddbPGJRRvK2CffdqkHH552y7Mo9bpQbUvVERta+qPMrCl2tBuxtQ+1Hvupt/HJU+wr8Y1PPpRPzDlfrBFD2/DUkyeN8mLS1yav3qzoTBYxHSeYeODE4HDYU10DjP3WpchQNxHaGBU48tG+GOpg335x7dsOFYvLGCdEjKMNWz36a3piKGcb+Vh34sX8goceclA+8LRTYka86C/8iE+9LGLRR36jTpTVF3mxTZ0itQ525MnPwhP+wT/9j5jWpcYzIfwx4mDUcIyn1ncMwBQNY6upEps2qdvWH3WcN8rKL9vyofZ5KyPm0XqUn3V9JotYdJQngDyK92CJfdR5ROzN9XHBmJyYqMiL+vD58PiaV8rU5Hj6QqIp+FKTiZhlnUnCPh78rxX2McVFMsYe5bVtv35A2afDzXf7ol6tri59PDXjMXzbWx7YwuFEj/hg8GHMeHLGDd6ynza+RXv6Q5/6yLgPz34o/dyH48mbPsDTTtR3jKI+9VK/7B/XJj7flBBl9Z9xdZ+XPPyathhfxBA3Ysb+yO9SjzYYt9oYRRkw9QFKEY5NLAAAIABJREFUsZ+HMswRHlBMUsAht+5jE0/8SbCmkZ3JIobTfHjE/eIXvzhPBIIqg+AJyDve8Y50+eWX5wPFA36U42DEpJSY6NJPYQF93vOel5NZ2kcGHu/qolCftDDB/+zP/ixvg9CniEE8vG6GPurKGEOUjXXl2N/EPq+2fWxRp1ZnEX3d616X90PFfu1zkLIPT3vK2Gbxec5znjP8XUr7pTzdrb1PzFzyHrBf+IVfyFstxFQXGba/XHHFFYnfX7QfSh/5+va3v53t17ZIIMcrfNBnPxqlNhe0Fym6FPLKPigWw1jAwQdywz43NhurY+6i/KT10k+xxdmODX3ni4Gfo2MfGschpbSjvRpFlq03H/nIR/KTytLnmk7koc/84+mtX1L0bye2iD+qfuxojpLs0MdEcJGoibOIxY1+BtiWsHH9pQ3su1m17IvtiNtloPWPA40Xv43aiwMeH21gt82GfOX5JmVDZNv7xGIMtTo78X1nV61fX7QX7TPx3GfGZtZpCvqjfOcsgYXQ3EixRZ2Dj/eZsR+tVpg72KAYQ02uxgOfLSKj3pfmFhL9igdjDbONhz5zEcqnLPLMf9nftV3isHWDhcwtHF1xlGMRYp8em36nLcQdy3ZjjFht9ZkuYhhpGzgmBJsh46WYAbZNSPkkRtm2QOS32bc/YnXFVA5dDjTaLDgsahT7tYEP8Pg40ewr5aMuOart84q6tbr2yC0HOX5ySVY7COHxwa5UfeLRvn3RnjFFXlkHq4yZNj75LW3MUjGwzxzBtj7RV+orPynVL6n64msbv+BJletKo14ZY1eMLnLRDvKcYXKi4BztgoEM+Wa+oM/88XZCmac2vBhj1Cn9a9PfLn/mixiOx6Cig0zkWMZNFBMijbpt9VH21dGu7XFU+1APMCgfiwNGjNbVU0Ya7YMR86U+tMyX+iUVD5zoV/SvpgMPXT+0mczg6EepN66NboxHG2LGdg3LuI0pynfNRw03+hTrNXxsy69hjePpu3kVb5zepP01XPNXxjgOG3k+zh99H6dHP3rIQ7GvX+Xc7oI1jczMF7FpnDjeOiR71qU2cPJqtmKfi03k1XQm4RFj1ziRm6XtSfwkdhenE+UDdk9kDibJV1dZY+oqvx05xw26E3N5nG8TL2I63Abc9cBp0x/H1z7U+jid7fZrB2p8UOtt+MqoJw7y9MlXXx5t+qJ8lLFeUvRZEKApuVBDj96XqWFG3rhJ2GCXlru10e1vNXnDT22hnfsGOSljp01/9LPNorpdZMVQFqof9ukb1MU29n0v1cnNuGIukXPM1JOKQZuPebVtf1c68SKmIw64hqLz8sZRsZCjPg4DGR+/U4/642zNop9k66N0HK5yUuVtQ42D+nYGlEWB+xrgzM15qXt0AdM2VJthfcu2uR9i/1AmKm6jjn/zC0dfTW0OXMyMX7tSTeJbv+8tibnhMm1/f3AvUL1MCT8fe8OK4kOqfBzfYefgC4W2uVU+ysR62V+2o+xO1WdpE6xmTh2dq21+R7vo0B6lj4z3lrvmt7Q90SKGQfZJcYM2OisoPPbh8ASqSzFIZKmPKiaCbQ68jiXqjtKbRZ++8ZYAtnG0PT3bri2e3vGEyNfNlHgcZAcOHMg3Xss+2vST/5tvvmk4cfjynEsc8M22D/YBOVnEoI/CYnLqqaeks87alxfBcoz5AsG+XyTqd6X4d/uBA/nmc02HPWrsI+PmclxyHGt8P7o493Nc4GwNzhB68/NHX6UziLlmp+SJT+7JX60gQ585UacmW/KQPd5lEv/G+ab/HNeMvzko9XgwwvgpT3+sox9zqD54zD3f7tHme5vdzosYwDz5eOtb3zr8XUi/QQVn8r3gBS9IH//4x8cuMujEbz7aMWADNBHYYhF5z3vekxeSNtmoN6u68bFZlvdxtf0u5HbtsVnwVa96VeL3FWM+tM8EYB8Z2xjiQPf7nHmldNttt6ZnPetZaZVfpGGvWo9vwpzBvEyddtredMVf/mW69OEPT5u8xSKfFSHQT1upn87cd0b667/+67wVYS71sg642f7cXN7E+pz/9/+G8evX2LgHiwyvQfr5n/v5tLS0CGhKc73mRLCPB/20b9+Z6T3vfk/CT7xi8bXg4fIy7xM7M6W5rZTm5rPPSDA3kL/woovS3/7t36b1dc4mc+BpDuw5DqZhZdAXsAdz7/rrr0+/8iu/csw2IO1D2Qzqu8rGxU6/MtKIFevj+qPsuDrzIs6NcfJd+vnS4myJn2vjdzFZjKLP2EPmiU98Yt5HyUmOfoDPcY7+pz71qbw+NGfUzfjQD5aLGNtckK8VMGtlokUMQ2wIbHvnE8FxIFLaEmnwMciaYyVPeeyfqMKZEhN91G8rbsc3XqbHAsY7uWqFX+ZeWSl/l7EZWBYBJsc1V189OHzvi3DmGWemI4eas2TGwQWuOU/rp8WlxfSIRzzyvooDzo033ZS3QNBsJlSz1GAwn81xAB2dm8fgYGN9bS1ddVX9NzURvvjQxenSSy9J+8855xjdYxtMcM8dm6XKM03OBLq8j+1YvKMt5jcvPqxttj0qNb5Gbp2vUOe8mmW7JqPspLQZ15ZBmBRsIK+/7kNjoakVzpRZgJBv5kcjpT5XMexj42RnlqXzImZy3IPCAeO1LH04zyIGf1QxOKh15K0bsBjw5VFvW6WV3wlq7CTfSzFin5Uvxs6Zrt9SfLNxUFGMn8lz9FLOiXp0IeF0bGl5IW1scu+unxbm+e1NxiafTqWVldXB2Rf57uX7/Whv9bHTnKlsbHoW08ssxDirw0fGfqP1dyfr35LD8Rh8i/YGZ19I9zMP7F7iftbC4kJaY5/SJve+UkK2KcaaUm++8TPfESOsQX5cyDY3MyefmQ5tD8/q6MuWScAwr4wj40nu2WtH3uO8E8d5YLuNou/ccOyiLP1dsaLeJPWa3Un0o6zzk7nPFwVz0PiQIx5y5x5DeNF+qY9u7NfWtDnpvIjpiAMM5RMdtk+nRlGDkDpptKMu/fbB06b9x4OaXGxbZyDiQG7HD2MGjzofbBmrOWragzMvrpHyISxtJs7a2kZaWmaB5cWQ64OzLfzezBMtn6n3U9pMXDCCwLWWB/dc6s03i1dzsOcrPlaxoU9bm82pPvhNkTbuhNaxKRl04Md9S8PbWN9Icznu5l+2cr4zbPYyLz9zaTNH7VrUn2sWwXyJSUx5Tjb+aidfSmYFOUdpzC15d1yp23dUunsNfQoY1m1HlNgX+dutb9f/Nvvmpzb3I4+48/gVl9XqT5PbNp3Oi1hbUNPyHTypgw3VWfocDOWmtffg02tZLjgr3uynjc0mjwuLi+mM0/elxcWltP/s/WlldXl4MsK9tP7cVr73lA+5vB6Cy4cljoP66FLJmdzjf+AJ6cx9Z6XlXbvyAd8cqsg0/nj25HjmvDOmg7OuXp9F6OhTXiY1k/3QoSPp7P37h5fL3hXJl7zNiWQ+u5pLvCVj4KLrb587ehR9dwlOqbn4zKtJZvZyzM3i0gvza5bzI87f7FVz3Z5NOJ+1d0yeZM6A6sMMoKaGMDaP3Uin8U/90qHjuojhuKs1VKcM1r7I12FlpglejGkpNvno37Q4xlXqy4canzaRjTxle71mgWkOV47qJp/czF9f30yru1bS//rR/5XOP+/i9L//z/9Jj33MY9LS4lI695xz09YmZ7RckA0P8bwI5dvjsPLZDaf8rB5z3H/P5z97Tzst/em73pnWN8OrrQdLB+sHS4j+Na2jkQ4Xu8EWieFCNNDnDG9+cSHtPf20xHlZzgWI5CSfLQ6w8tPW5swsrw3ZX7LQWGjuzWEdvcF9vxxDP6UtFu2Bo4O8mls9xa4feZGW8mUfcwQZKVjUuVyFT9szFHRtI0MdmVE2oj3r4kMt2Iht+ODXinwoOqV98fUfOXVKXGSiXf3w9gt68CilnTZeFh7x57gtYiaA62qKgbQlpBbgiDh2tMsBYyD0exr/xukw+OJrk8Cs0+cE4QAWj7mZz0TyPrp+uvjSi9PTn/7/pRf89gvSWWeenXaftCctLTZ5d7Op68IcB3g+k2kWtHyvibMV7OaserO+l+Z6c2n//uYNEuMTjnb2qiJ6FL2RQbaR3+JMLV9yHn36SI+HZ74EDpdozULcxCBGzg2balna8gHnCtvYbXru6xa59Z4juTW/95U8lqMc48McwSb1xvbRgxZeeT/J8QSRujrQrgVcivdr1ZXiH3X9LHH1ARr9Vk58MJDhE7HU147yNaq+OtrYDj1ui5hJ5Akf+6wYTIM0eALhCQc3uGsFOR5xc4PbAarJ7RSPtyg4yY2niy1liZcnkPGf4KM+e+DYK8MrbWqFLRbInH02C0mc5M0CQH7O2HdG/tm8pzz5Kemkk/ekXm8+n02trW8lbnfNzzfbGvJiIEq8gT7vYpa/Kgf7subSPNsauMTsNzf+m+Wp8aE552k8npsb/FN8PjcaLCTDJZHFpMFv/B/EAHtwwz9fjuYDOF84Zk2XPKSzxuAAZ/FlYZ3jkrhxqLnUzIseZ178hwC7MRqf8LBZnhtfOZDiPGIB4jU8ftGWY8CDHeZfeQA6vswNnmzygCvbGvjp/GZswefmeK2wT40n0G1P/2o68MTn2KIur5Rn3jH/PO5iP3rsLMB/niLGIj5PJ/Hf+EoZ3n6BvvPb3KrPFiXetFI+nbSfB0e1fWTRTq1+3BYxHMXJF77whdX3feEcixf7wJ761Kfmm9BMKgv6JOH5z39+/sk1BgLe8Sw8gfF9WuVEHuWHfrIR8C//8i/zb0fKi3pMnle/+tX5vVqRb51J/trXvjadd965gxv2+WqruezLl079tLq6ki688IJ8FsQTv63+XFrf4MYWl5kbaXNjPa2tDR6R97fSkfWDmbfQY4liQ+xcSptbaYGzisXFtMkvI+Ub7lxWzucnldz854yOm/BM1K0tfphjPvM2ecNH3qc1WOA4kOdSvoTNx3Qesq0s25tfzItqzkV/Ls3z3wZpLtvN3nBGwxzg6WU+eycT+Mhi2k99vgjneGI5l/qbG9ku/uL3rtXdqceP+A4XsuYsKC+WLoIDylxiAeJ3Iz/60Y/muefCZO5p85on9pGVWzCcC1dddVX6qZ/6qdZF8IILLsi/K8l+R4o2iB8feN/ZL/7iL3b+kVp9k3J8MIdcnJ1j2uEVUrwPL7700D4weA3UM5/5zNZF6rLLLsv5qZ1kgMP2CeIvFymw8cXfxSy/JOjjWP/kJz+Z3wfIIq7vxjaKHtdFDMeuu+66Vn/YJBffhRQTjC4TjXdd8duKD6RCHBQG79JLL01Mhlrh4GAfWts7r/gW5MWAl1326Jr6gNdPR47cm+bnF/KHb/+77r433+y//fYD6YYbrk/33H1XOnjo3nTPPXemW26+KR24/Za0sXYoLc7Ppb0nn5x2LS+n0049OZ18yinpyNpGuvuuu9ORzfW8GPGgYGlxOZ+hEQ+Tj4OG2zH3HjqYbrr55nTg9jvS4UOH8laP/K3PZsiN5heIenPzaX6uefLKN/bi4nKa782nxaWltGv37rR37+lp90knpdXV3WlpdSWt7NqdVlZX83+JLK8s53t9bJbNG3m3mktPFtEjBw+me++9O3+TE/tZ+85KZ5z5sLT/3PPyPT1WwXkWXCr5VltzqRkPFs6QeF9cW2Ec/WKNcxN52nxJs5C1FeLlx4+52qgV9NmnxhnVtMW5FvX1lcWT3910EY0y1NmNj/8shrXC/CM/be8b4wqLhbDtTJKXYqLPcV4rXIHgq/6WMnGsYt9xW8Q0yqQ20dEp+KzwnooiH/vRQYaJRp0DiDMjShm0beisCr6A57fuJLjGgS4TlTan5MRjH9jEQ1zUOVi8dFWO3CCDDhQ+IeZ9YPmuUXNQsz8s9ebTDTd8J33rum/l3e3nnnN+Ov+8c/Nu+ZtuuiHdeutWWt84nHbvWU133zWX1g+tp7n5hdTfWEu9pYW0uryYTtmzK61vrKde2kgbm5w1zaelleW0tLor+5kXsV7zO54b65vptgO3p7vvuiPdvrmW7uUFhvjIWRQ+DhYxTp4X5xnnhbS4wCK2lsd9oTefFubm08rictqzuivtOfnktLJ7d1pe3ZVWd+9Ku3eflPbsob6S58AiZ2Nz3LfaTFubh9O9LNZ3LqWFhV664/YD6ZvXXp2uv/476d6770qXPvL78lke/izMLw0ukY/OL8cASs4ZJ3JrgUfuHTv46ihjm7FzvOQ5luSLsxTGLtqg7thyDCCHLnzwYqFNX8nXJ21GHXnYZd5hA2wKONTd/4Uf0T4y+g/Ff8+0zBG4Ykq1CVUPiiz62DWGmF/9pk+MyLMe6XFfxExedMI6fdHxWiDKkAxlpeLYlsrfLp0FHgNPXAxojA9+nPzGic/Woeo38vPNU8VevoXdbGPoz+f7Ptd+46r8nwVrRzbS6aedng4fvCedcdrp+RfAl5aX88LCxtiT9qymXSsL6e4Dt6bFXkpLvV5aWeilXatL6aTdy6k3vzudfBKL1mLenjG/tJwWFpfyP3JzBjU/z6RfzJtUd92wku6840C6dXkprawspeWt5XzJySVds4k2JbY15MVrYTFvxl1YWErLS8tp9+pq2rNrNddXlpfTbuornoGt5kug1d270+pqg72Q7+1xkbmZ+lu91NvcSFvrR9L6keW0OHdqOrJ7V/rubbem//n8p1O/v5Eeedn3p8W8ODSzIOY+1skrxbGwjox91ClxPogBzy+fLBTe0UYf496MXWMHHrriu4jC9yNOF6pvUVbfoNpXLlLqzDHnG/YpsY0+H4r5iDEga/zULcZi7NjStvriwde2+qPozBexSYwbxCgHHwx9bXHC5xMHO9Zj7G0YzLPBIZVvbrOx844Dt6XPffYz6Zabb077znpY2tpaSzfe8O10ZO1IuuDiR6bTznhYWsu70+fT3XfNp62N9bQ410/rh+9NC1vNmyI21tdSL22lk3fvSr2Fk9ICZ8lLi2lhaTmf1eT7UmwunZ9PC/PL6cjaWrqdxXGTe2jrzf2qXjO92PGf7+dz5sTBz9lO3r3BU0Uu75oDdiv/8G1zlsLlJ/f02JLLJaj7z/LO/OFCsZHY/Jo22eXP/9zxq/MchEfS0lIvnXLSrnwGePVVX0mn792bzjz7nDTXW8qXlL3BAWr2zDV55mBybORD28aAvi7zfpR+tNMFK8rHurrSNt9ifC4eEWdc3ViinagT57Ey0ihXq6Nbk63x0J/5ImZCoG1GS8dNqIkp+2ttZGv4k2DUcHeCp0/mBhvwzBG0jKeMzXbz3chDPjaQ9hPPGvN2Lp7CcU9+czNd981vpFtvvTlffu1aWc5bKLj6YnG58eZb0qmn70tn7T8npR6Lx1bqzfXT0nxK9955W5pbP5z6G82/Vy3lX//Zkx/EcJOch5M86eOTLyXz3icW4oXmftngySc+ctMfh5ozi8F+L/5TgIcAg2/oXq/Z+IoPnFkju7DAojg4W5nv5UWS88ytfvMNv8VCyEOIPP6DJWiOf3ZHajP/Yzsr5uHDB9Piwnzae+qedMstt6Ybvn1dOv1h+7MeV955MW1W1mMWJw8+8u24waNu36RzxLFFv8lHc5A6H7DFRzvandSO80rfoRTsYJeifW3DU874aMvLSoM/+kWfMdEl3zrtyMOWsUGtKxPtlrpiQqPPtC1TL2K1IOExGSk6pqEycPg4LA6UBJts9Sal0Y5JAiPyJ8WclbwHaomnn3EAY26QH8oMlZkoNJob1ZnNRtcj6+m2W29Pp+89PZ1x5ulpz8qufPOcex793kI6dPhI6t91dzr11N3ptDPPzNsTbr+ln/rrR1LaXEtpbSn11w+lpaXmUpAzLt6KsbC8lDY56PlfysElIZcV+b+QuITFmbnB9tnBbRwW103+PYlKnhNsq2Eh5NK5meh5McmbaxuZjY31PAca/lziaecCB1UTaX5YsJz9aC6b85YJFnPwetjaTMsLC+lIfyvN86aLuZR2ra6kW268IR2664500ulnswLmf8tqjvFjL+ecj9nhwR8OHuJj4S7HJcqNq4Pj5Rx24kEJLm2OG32QgtvVrjrSqAcvXg7SR9EP4htVSkz11PH4VU6+awF8dLwc1b5ytMFALmKoz720yFdvoVSwYxzVgQgKj20EZ599dnbWPvjUpSW2WBxo3BictoBz+umn55uU2p4WSz19mwUeWyQcQPDFLKm2Ix3KDJh5/g3XB3LLWclcWl/jLGo+XXLJRWmVH93op7S6uitxVrWVFtLa/Eq64+Cdqb91OO0/e1/adc55+czr0MGD+QBaO3gPj1HT4spimltYTnPzi8zyfMbH/1azzSJf/nFgseufJ4RsK8UQByEXa3lLGZeA3D9pdmbl/7nc6uctG5xFceZETPmz1SxoHsT5/WD95ht7c4sfPmHybuZLz0UOAs78su7g7IizUg4Q/gF+vpfuWj+UFtkOsrWR1/hdq6uJ/yn1/z6bFDY6zklsc5DwhNiDxvxn7F4v7+GiTlFPmS6UA5TfRD08+Em1iEGdJ8nso4pbILrgjpPRDvMP++x3JA756BM/TyctsS/yYt1cSHkowLvwXGzAsCDDU1HsI1cWZNmZgL4LmTL0MSblu8rsX8B5DJQDp0Ab1XENIMci9Pa3vz3vVRG3Tb/GR4ctBPhCfZKCH2zWY58Z+2Fq8SCD39BpirrSiBHzUfK1xwJ2/vnn527ik69u1Guru1mzia/Ha7Waa6pNNkyxqDSXPnt2705LC3NpiydyvY20yFkRT/LmttLeXUvp7nvvTDddv5727z8vPeyci9PBwynd3r85raeltDDHv4dvpd7SnrS4fFLa3Oql/mbz1tgNFqzFXppbwF5KC0vzeeFMPa5nN/NlX77flS9zG594dxl+cYtseGnI5SELQh6L5sns+gZPBptLHs5a59fW8p6wjSX2j22mxYVmAZvrc2nbz7bmejzJYz+cl4hYICVcM/bT4gJPI1Na4+0vLMD5346YA5z5r+cDCn32uV1zzdXp//7fX8nbNGL+HW8WHzajUiYZM+ci2w/YR+UXmfNRW7xmive5tW2BUG4Sqp/Y4j10P/uzPztcRGMczEcWET7qjLOjnMfqk5/85PSP//iPw2PP+JBD5gtf+EL6yZ/8yXy1pi42lHv84x+f/v7v/z6f8Ubb9JNDFmHWGHWUGX3+qNQIijMYsXiQ2p6UgofDJqaLvvbRYUPhdt4p1cXedmUcQHOn/9Iaftk3vMDiLIbVJF9W5lOl5sZ7j132TJ5+muuzyXUzbfY5leeSopd2r8ynew/ek26+8aZ0+r6HpfMvuDgvBIc3NlPaOJQWV3qpv7CS1jb76aSl1fx/jVv9dTZbpTk2xPKFwL2PPAn5w7g1lwLNkwY2qPLhjMf50bxyJ9+LQ5+Fjv9nzP9ojn5zrw/afDirY38GZ2pcgnIJ2Wx85d+lmpt0bOGYy5e4LGTcO8vTcau5X5jfisHJ02CxjLltxsEL1eZ3Kdmn5UIVZWdRZxEEv61wBsaG27Z9ZG16XfmcaWE/7sXsqouc81Yd5qRzGMr+sdo+O2V4KSYL6VrLm59ZO9DnTK2c79qs0eHlZOlgTTjyDECeRmungsgYSFlXP/aLZd84qi7U3b6c0rrfpQtexNCeMUZqX5S3nz74LKZQPmWhT3+kyklLHXEbfrMgZN28T6rZEc/N7mbRmkvLS4tpmf1Gva3UZ9EZ3kzlxhSL2VpaXVrO/t1z14E0z79DnbEvXXThhYmndrfedH1aWV7Ku95ZGDY2ttLi3HrqLXADneWShbH5h81+4rJxMc3lBYxTLZ448rihl+a2emlus596ffLBhGdRYtMra95AH98G90EGJ0nNwsaZVV6o5vJl6VxaTP28fYSFk0UNH1j4mk+Taf5yLcu/IzUnXf5HVY9/SeCTv3Txp8nm5mZzD2qB2Ob53cbmdy+PzXkj28Y72ju6xpg55lGSOcFxw3yFel+5JguPeRL7bDt/lMEGdfBYGMCGYocTBeWVox150cdYRwZcP+pAsSGefmhXqpw4nJlyvJIHKHrUxcmVwR/46snPZ2I6I7MLjUDULTogj7aFOvyoa5889eRPSglSH6KutvVHe8rIp01deZOpXI2CZXIjrrqRR93iZUbst288zTeemgUlpxjc5swoP+FbXEi9ObYp9FN/o3n6x490cJso31LaYE/YfFpd7qU7D9ySNjbW0jnnnJcuvejilDbW0sF7DqQjRzbSybt25UvWzTkuJPPy1ZwBYW6Tm/TYYQHlNGo+JRYuFrJ8Zsh48zZp6KDOAjdYoPKZGPW8EOUTrtTPC15zVsYmXu7D8a9C/fwgffAAIc8h9yANFnWWzcFeO27y41O+l8d/FXBWxpPTvAD7hQrlQG4OCvM9PJusnHkosx3qmEcMxt+PMTgHo9wk9TjPwBQP+y5gUSbW2+yAQZGWcvC1FeXAto96tG8f8vCNHxoxcmPwp7TfgwEQH4o0KsW6/SWQ/CgrNrxSPspttx6xreNP+cGOPOtxUE0ufdQtxgaNMvJr8tEPcaAOjryIIa8rRdclkemVD8rMac5UxIHP1ga+Cfub62k+baaNI4fS+pF70/ISC9BGuv27N6Xrr/922rW6K5177oVpvreY7rjrrnRo7UjKC9hCP23ObzQ37rGVd5Hxehk23C6k/hz3rbhcbBZSFpG5uY3Uy69fzBd+zZ4wljcua1nMuNfGpWPzv9rN3Bu+0bXX8Lmc5KxuuEBRJ2pWx6NnBM2CtZh6C1yKsOWD+1/zzaX1wkKziIV5SO48C/PAI0/hO9f0bYs6DwDBpm0p/Fh3PuSxrcxh51/ZL1870WkxI2+SuvqRUo82jUHaBR/ZKB/rXfSVWYiOwBSoZiAegBwQyBAMH+uRakQZ2taRi0UMeGVflNMHZJSzTp/4xEURt5ShTR8l4sBTF6qeeaJNXT/klzjRdjbS8kef7Y6+xDoifja0AAAgAElEQVT9tq1zKeVbS5u+wRkhpzr5CWAvbbBC5IOdG+H8w/R82uAfwPucrfTS+to9aWP9SDp86Ei68fq1xEsD16jfeGNa37g77T19Me2dW0qb3FvrNW+NaL4kWWTyjtW8bG72yQuesUG18Yub61tpIy8M+UwsvwmjOV9kIUOBJ5H5iWbiocBm6s9vpsT6w717Fqs5Lk/Idz8tLOAz+BTHDlvINgtW/ofxvHSyzDZfGvzPZLNvjVMxzhaaS2TmMPlHn3FsxjKDt/6JY9AqNBgr55djjC42KPaVGOJDrUdZ9OWXurEPGfWauI4ep236JZ5tLve4JEUPLO1oQz7yYmPbOvKUqKdu1FE+C0/w55gb+xj2ehyDJkEHHHTaXFdjNBqOjqtbOhvlo5/qQq3HfurwuWamlP7R58KKDf0TI9q1LkXGOrRm337xpJGPHgXfXOSUg4od6+ioBx0XH9iU5n5Rk5N8yQaT+0DYyQcvBy4TrrkHxURkywGXevwfYz5zme+l9UNH0hr/rL2+mdYO3puuvfpQ+va3vp0++Z//keYX1tNJJ/fTmWevDBaPflqY25XmBzvxeSsPe8c4m8rnRiwG3IfrbaUeZ22cZaWN5qY7KwoPMPPG1ZQ2ttbz08KMlfeXraWNrcNpY+tg6vcOp9Q7kreXZdzeSuotrKeFRXAzSF78mldSN08j8yLW42yQN23wD+dbiYeYnI3lhYr/J2W1Gnx3MhZs58n74QbzmH+fiuOZE138cawKdrVZyjp2VeExTP3yyeYY8dytjscCecAH/Sj9a8NUHgqGPoiPXqyXbeXR5RgFB9vaF5+5Lx8qZqzXfFxAUDAEeDLCf5tjsCwkg1d98JShBgwW+7SYHNEBcLTDk5/Dhw8P+6ONqA8/+qU93ofU5h//Hc8bAGr+oU9M7MPxn1SjbfpJ5nbeVwY+8ZMni36X8ThwykGJnzcFnHPOOdX8sE+m+Xef5nhsMKk3Z16sJGwW5a0TB09eSasrCynlH93gW3RwZpI3PM6ltc21dPjIWt47Nc9it3EoHTq4no4cOZBuvPk76Ts3fDPdePM30+rJ62nvvpT2n3dq2rW6nHat8MXFvrGUNnuL+S7ZFgvH/EZKCxupz304Pvmfe9gc29xJy1u2WJF4ZU/u6yd2329uHU5pay3N95dSPy9Wa6m3sJbml46kucV+PiObXzicFhaPpPmlteZuPWeAnKXllydyhsg/Kaf8gIKtEuQ2n6Uypvk+TbO45XyRpD4Hcz/deedd+UkZ8XCg8ZN3XFK2FW6KMz4elG1yJd85wFO52j60Ur5seyxNqy8e+9DYh8Vvx+qTfeMo8mxxuPnmm4fbHMbp2K8tnoqyh9Qvavrpc3Fj7TFWKTKxLmakx9zY5+B+29veljfDlYoYYxV93etel17+8pdHjFxHnkH+0z/90/S0pz0tn9HhnDhMLBYv3if26U9/OjsOJkUZ7L/vfe9Ll19+ebbFZDEBYLF/5r3vfW9+RKxOdAT8V77ylcPfZaRPG9R5hP3+978/PelJTxomj35l2AT43Oc+N78KB3vyo41aXV/Yp3bFFVekxz3uccfETz9YUjFsq88C/aEPfah1kWdxPPfc/fmUhtxscEaVz/zyiVW+pGKi/Ou//Et6zGMekX7wCY9JK8uL6dC9R/LTQ7cncEZCfPy7EWdn62tH8j+IH7z7YFpb20x7di2kSx9+Xjp4+KZ0/Y3fSF+9+svp4OZp6fQzTk2nnXIw7Vo9JS0tsWfnpLS1xXXfYv7XprnFzTQ3v5E201ra2ORp00KzNWNjq/ktSM54OBtkj1f+Z+Re6s9vpYWVhfSwc/elR3zfpWn/OWenPSfvSbt280BhPt/UX1rupcWVg2l987a0vrmcVpd2Dy8p2V6xyf97csmZ/12J96bxIGMrzed9cptpcbX5HURO7RgH7td997vfTc9//vPSF77wxZwLFjKebPO6Ikoce8eJ1+h84AMfyJsus1DHP9lmr5fn5bOf/exjNpV2geDYYbx4X9dznvOc4atyoo+jcPSf7Qsf+chH8iLknBulZ5/+Y/8Zz3hGXoS62gZD++zf/NjHPpbXiYhtP8eP+8CMGblYVy/So6cMA2P/P3nvFW1pctV57uPPuSa9KZeV5WWqkEUCBBJMIyEaNDC4hYTQtFqYBQjQg7TU8DCz5oXptTAPgDCi8QiEGVqsQQNqxPQsQKglIYQsqiqqVJWlMlmV7trjzazfjvM/d9/I71yXWUDPROZ3I76I7cLtE2Z/EYwC5jl+CXTmUBEMSkhnERVlEmG++MUvzrWVYSTFBaU44auw8Xm4l3GeY5TIL928M51Qgig6ORUevAgjH4ce7mTLI9win1840Yem6Cov4IgnPk4whFFMt912m8fP++OL967840jZ9/kchXWu48eO2NEjh31tqt9jnYvpHp2771NBN3/wHcWyTTiepbtp3c0121xds05v6Erh7jtuseVDY9vsXbBLV87bqHTZLq8s2ImjF+z40ZO2fPSkHT5y0mrVRas3FtPYqtK3sQ3cDGNhacEWmst26fKaDUZdq7da1ut1PL3f71qzVbfTp07Z3ffcbs+/9247c+tNVmsw9WM9DCXLNBTlRLkMrNPvWLdXs1b9qI0mbc9Ho9byTYKx9b3uMF4d+chzZJ12xyqVuq+B8fnR4sKC75R6XUzNDp588kl7+OEvbCtqlEXuVGd0MM6DY7R9EEfb0NQO/NgudqInOPBpm7TzgzjkvuOOO3xEdRB81kk5GFJ9dL800C2U37zzxKCnvOIrrHj1mZyv24kpEiQ6Mi5qP8UzDNRQGl+wwFP5skERvdxHCGnaiA8uaShBTcUkMLwJy488RZ80HtbzoAu86AsPWOgTL0caTryQA/l4J1zES7jRF36kT5ziI6x4Ki7CkAZPwcS0tOqE4ksjCT8Zwrs4fMgEeZn4cTnPec49dvNNN2CXapxGwcI2Ssyx2YFjN3DMOVzsVo5s3O/boLNpvfaacXQPFvnVEkfajGxz9bI9MV61jQ3OHWtZe2PVVtcu2NLqMTt6/LQdOXrKjh45ZaVJ3calro0nfVeCRw+dsMUWN3nXrXv+gvWGQxsy/Rt37cTpJfuSL3mevfTlL7IzZ05ZrYEJRd/Gk44NbeBrdm7q4Ma66QQLPzNtXLVebdOq1ZbV6k2rNVirq9tSY8k/tepPej76cmt9L5uJ1TGxYLqIiQZte1rn1C/GmameUm2QRNmr/FVHeqd90QeYkeC214+gt3zwgAGeds1IL9bvFuTeQrRd2udOA4UiSvBW31AeyD9utzwAI/nlq28V8SqKE3/5lGEq962lrCiH0vYqn392FDurCMiHkCqRjCuMr7BgIh3FSTjRE07EB490Cknp4OMiPu8q/JSa/oqW5OMdmrk8+Tu0xU98gBE9pUVeO4UjfXAlT8QRH9EGJuKRLjzBTnNJaaQ1Gzf0nC4Pkej6Kyk4FBdneXESxbBHryQ/rHmNrJYMuvz0i9Iw2TVUJiWrctwNHZwDEP0D7LFVFqvWYg10Zd0uPL1iR1YW7fKhpm2cPGxLR5Zs6fBFu7JyyQ4fv2Anjl+2QyisSccmk3SED6fHPvTAI3bXPc+zM7febI88fs42e5t2/FjTvvF/+lr70pe92MpVzD461h5u2nDYMdbe+4N2Op3Dv6107ezlwdQURbnR6VmtvmyTfsUWRpywcdTa/YHVjHPGqnaxt2lrayu+RjbCMnw88lNgGYVWK9sX7lXXqp+pftPrzKceYn1RP7jt9TMDnwXAES5+jieaM4Q9BKLMe8VXvwIeGfTAbrc8AKN8gEdYzx7EdZCcvwYS8Fb5iFbsC+JH2k5y+nRSwCK6G5IY5n7EJw26e3UUzPV0RZnO4+CJjBSyXA6j+L36wlfeeYeP4kVH8bzH8ld67lM6SU2lwQRr22Mu10BPpb7uAPAa9QY26PatVU0L8KXxxCqcBlGrGAtCw6mCL485IaJs1XHZKuOSlQdjKzHaLk+sMhra8++8x87ecoN95v5P2UMP328rl9Zs9eK6LR9bsGOnjtjGZtcurazYxQsX7PTxm61eWrbbb7vJhpsDe+KxC2aTvn3hC5+3M7edtYVFs1M33WD/4ze/2u646xbrj1dsc+WKVesTG0261h9sWo0vJNm1ZH3LR0X8sKWS4KsAzi0rW9N6nU0rlxtp5Dbq2GLrsPXHVTv34KP2l3/2QRt2x3bXrXfa8WOn/XPJRq3qU1y/GDgv2H2853W4D9R/UdDr3beuNTN5f1BfwJeCE4+9lLlPJwEUMAzyTPOOhoy/AmIyz6djQjOnJz6Rpzq8OjM04SXYeTwULx45jvDlCz734QsuTrD4Cufw+bvg5Ec6yEa8fOHqXX7EjfiEXb3PdPzUwBXNNb1urezfH7Lxl+ruysqKVUtDG/XrfmAgZ0twbgC2V5irYuQ5Goxt0Btbv92ztZU127y8aptXVm2zs2mthbp1KmadhYbddvtZu/P2O+wfP/85+5v/9iE7f+EpW13p2OXLG3bk0podOrZoR48dse56z44vn7ZbWJivL1tn/WO2ubFmG71V+/z9H7db7rjZXv5lL7Cbbz1qm/1L1vf1sZ71Oh2blAc2KfWt3+3bBHsxLsb0bx8ryb7LbdAYQSRFZpO6lSsNG407Nhh0bHN9xR743EP2V//lr2zj0obdeuNZ6/XafCpgg27XLg8v2sbapg16bV+bUx2ozOV7WRf8mFJHSqOd5G1F9HJfdQu82ja8Ij9w9I4v+hFO+PhRlpzfQd8lJ77oR1rw3e8UMuLHsHjF/JHO+zz+wheOaCh+m4mFgPCjExJzWp6DOAnI5gAuVghhHCMiKTQKTWFP3OEPtHFRGUX6Soe+Rl2Kk69C1PZvxN+BtSeJBusNOX3Rzcs0xgtffGIacRqBzdIT1+lrrKuS9bpd++ynP23NysQWWhUb93vG8TX1atlHOSMMTjl7a1ji6yIbdkfW3eza+sqqra5esf6gZ9VG2U+8OP/UEVu/smbLRw9bZdSwu2+9z9ZXR3Zp/ZKtb2zYpUurtnSs4buWK0fW7cLSRTt9eMWnloPBhrU3Lltv1LZSi4MMV+z+h//ejtxQtcXlpjUWWOsaWne47goMJdbttd2glSNzsHRg+lf1oy84qaJkpdHQ+n7W2YINOuvWbByyjY01+8RHP20PfPYhs37ZlpaaNhh07bHHvmCXL162R8990UasiNUW7AUv+dLpymBqf7rQAnOLvTjaF+1f7TKv05wG9SoYfLVPtXfB6x2fdq/RiHCBI8yac4wT/m4+OLGNEZZsMV605ed04Y+LODnMXt9zGvDUs1cagpuZWECUh909dh8gmDOik6ri8zQRlF+UDk3d2EOF5jAsjGPmwA5mnia6O/ns3nAv47wdTBZy4REdMqkR7SZfxCsKY0ejio7p0C1yipcfYa6KEwmfO3KqKw2bu3tQEFiy0xFL1u/17HOf+bR11i7ZUrNupXE6epqLNbj0AwsuN0Xgu8QB31WybDb243QGfUZC2GZBfGxrKyv2+LnH/eozbh/q9gfWGNfscOWQtQdmm1eu2IW1jm1e3LTV5Q07tLhiFxau2KHWYXvskUft0uVnrL5UsS970UvtnvvutMvtS3b+mXN2snTcFid1N25lDc0YNY57NmGcOD3JAot+fhD6fqIFo3pOfE0KpN1pW7XcspUr6/YPf/9Z+8I/PWZf8ZKvtC9/ySvsT/7g/fbYQ49adVSxKnZntZZ1+iNrLR+1133zt9nhk6kgsbeLNpGUNz9gmF7oh0j1QRrtkXh26LSwrvS9+NBgZx87raKjdqDP7jY3BtEHcgc+fVP9QjLlcEXvwilKIw5aOPo2+Qc+x6G/YuOZx8+jOS9eckNPYfof7+IPbuQjOEyk0B+5cyUGAoRQUu94xzvsb//2b/3XIBIizEMl5kxyoryrYPAlBErk537u566y8xIfRmnYeWGPIvwi2nkc+MCzc/MTP/ETfi2V4iIsBSUTEn7txIN44GlE2JFJiUfcvYShKfrQvK5OgwV2F6eVzDRxMh5amRVxH81OrMwiUq/v08P6ctOqqC0W9ktjPxUVk1evD+pomI6u4fTV4ZhRZJqq+ukT5ZJ1hh0/KpqbiSbtro+OGvWqr681Ky1brlbd9GHzUseeOb9qF8vrdnhx1VqtJVu9csW/rlxcWLTbzp61G06fthPlY9YZrTvjfpcpJJ8a9Ww45AaoZCibNlmTHD7AZgRWoq7KNqSeSlUfQbIx8fT5Kzbo9ewFL7zPXvTSF9pNN99gX/rSl9gTDz5u3dW2LbZK1tno2OWNrh0ZpzsooQXdU6dO2y/8wi+5QqKuqLtHHnnEvvu7v9s7spfRdISvHznMG3a6V3Kn+qZ9cV7Ye97zHjdxUPvEx8HvoYcesje84Q1XKUnB0i4xWBXOTvxiGvmLijm2TfiSRv4/8YlPuJ0k/TDyEH+UN09Mi3zmheGBkx/hiBN93TuJPJGHZMR+9N3vfrcPFMiD6G2bG1JZsgWJjK5HWAzVyXOaCE0lYTFPZR7E8QuHpsYoUQVTRKcoDfl45o3iiujsFKf87gQjOfDnOehsS/chWLIL867u3/wQSnFjv+g2mbs0603uSjNjZ65ctUa1ZMOR6E1sXE2nXGCAobPBODUCmw1syfjneH6dGueI+ScBbn/GiRjNSt0ajabVJwvWHXWtNxhYd21gw876dLdhYqdOnrLj3KrtI+CJ1UpVX8jnB5MLTNLJrUPjrDLZwKHc3E0//uYsfi8HNjJY52LXdWi2vLxkz3/e8+348ZNWb9Ts8uoVtwi/6aYb7Itr56w8mlidHzeOw6YfeV8it0zbqnbzzRgOp2kenVideV5dMFJA0R3U0T6xA8SoOTq1AxQU96pey72Tke5ew2pfzGSuxQ5sr/xyOPUVDLXp+1rSyeH4EZCsMW2bnRjENN2iUvULFBGKiMT0orBwoD+PJjA0bP1KzONfRF+FwHqFtDi0oJE7wUqmmD5PvgizWxgaPNAXryIc8ZdfBEOcp/vQi/6WLN2dvqstRljYWGEhmqaAnMk6KdWsNzRr9yfWqNas0eKyWT5+Hvu9jNAcjwb+JB3IKHsqs6sozgNjipp+QdPWAp2fEPlCFKzvoV+36kLN6pWatTtdt0sbc33aZGgYvN51910+8kDptbsDv3FpXB7651GDQc/tyjxfjAR9Z5KssIANb5QPGzxx6tGzaqVpzWbNKtZMx2WXJ9buta1WqvvlurecudkuPPqkm5mwGQA95HGl73+5fWlim5tpVEGbZ6oSRyDz6mWnOp1Xh7Rp2j3tk3bJAx0e4nlYhqDtxjW3IhmK4ubxVXyOw3vMh8LwpixUDjnerD2K8B590ZEPvxgWGcoA/ioTxVN+lI1sOBUvf9tIjEgRx1dYwLv5ESeGVUjESUlFWoKNaYqLcLuFwYEXD7QivXm4wlH6XnAEu5OvPO8Gk/PP4T3dFQsp0xHb9GyuCVMj5l8MrX00V7bFoyfsnufda0vLyzZot63f3bBqq24Liy03W8BebDTo+UkWpX7PBtZLU1GuSIMPp0vAya9XSzvEfCLEdJQdQ88X7257lt4r1ao1avV0yzfxNrZ2e9NuOnHazp69xfrDro0GNb8hfL29bpUaGzlj6w+wtO/7KGk04YJejCD5DpRk/qSHj7txKM86U9vR0DY6a1aroCgP+ekXG5urNu6Obal01I6fOG7NhaZvWExK7MYyVR74nQBOczra5EeO8lVbiXVPPknLXVFcDpO/g6NHvKAf24jeBQeNg/DKefMe+RS9iw8+CkQyKL6I5n7ixF9+xBVP1UXkLzjJIV/x8n13UqMXIucBCmE3X4LiKyyc/P168BPtnfyYpxiWjMQVybYTzWtNg5/44sewaEf5Zh3aR1pp5OTnbKUDZ5IBKyPpZsu++jWvtVPHj1uvvWEP/9MDtr5y2XrdtmGAOui0rddpW3t93dOZNtJwWBfzD7MpC/6V6dzUIReCTPWJf8yddoEZiXHHpHF+PnXN1W8cUzFONxpVyyVbWlzw87x6vb5Nyn3b7GzYyuolazS5+ZubktJnQihnRkqceeaHiKEGXX9MldhUAOTr2shazWWrl2u++dDtdqzd61qj3rJas2ndYdsWllvWaNWt3+lOiy0Z9kINl75dSDvi0yj3qIP/vzi1dymNf858U876wbgeZe7nicUMKHPqQDFtpzDwCIZQPBSOaIFHXHwnTjz2U5A5DdGRX8SfODnwJQt8Y2EW0RZexFHctfiRnsLyRTfK5xO56ZQPuElavPJjZ/zIMO/oHLZasmpryUdkN95yxs7ecZefyvz0M+ftoQcfsItPPW7rVy7blYsXbU2LuhwNPGB9DIVEPaULSFhHYhTEGWQ+GPLPlaajL790Aw2HLmF6OrISSs2/XSzZUmvBT8i4snLFji+d9O8me/2+odBcYY0wm+AUC24zYiE32YcxwpMFb5rNwjSdMYQS46jt0XDTquWJ1WsTq9fZneVEWbOuda1VW7BGs+4XmPjJHVWOZZpOR33zY6q8sk+MkAWnNjmvLQhOdZT7sT0pbTeagnu2/N1kjnwla4yL4f3QEp5w8GnTPHLEFT1K34u/bToJMS2qwYj3vTrgGdHhKAiGh7EhxLDogkOlk6a43fgVwREHDfjj80T+vM9zwp2XnseLfh6/33fxLcoPtGI8PBM8SiXlZXZpLkdCezpqjvWwig04Zmaz7aMgrNUXlxftxmbLmotL9viRI/b0E4/58IrpXL/fs363a+UeJg5p5OPTU7+oKI3Gyn4xCI0NnZXOeEZBcEqELrHlW06u8ahwuit3PnI89sSsvblpx0vHrVZvWG3Et7EsoGOqwxQvrVUx1fPr1ci4Zw8N4zmcvus0E+6SNOv32xgVWqtp1lo0P82Vs8N6pZ4NK+lwRfhAy+3L9MnPtD17qy5oE7RH1qso61j++6lbaOSOOOhprUnp4nNQXqJzEB+e6hfy6T88kvcgdHfCoU/qEU/gpQNYE4vxO9GKaduMXUlgd48dOv2iRODdwiwMYuNy7ty5aafbrjwQkPO8+Io9r7i9CI9MEV84qhAW/tjZmccf/FOnTvniYc5/t7zBi8rVvYHivRveXtIlPxWMfJRjdJKVBvb008+44mFKxASQadTxk8dnmxjIxfTOn0rFMC3ujMbWaC7YqZtvnt7fOPaTENZWV6yzvu7KjtHWuMx0zg2z0o8BV8D5ZSR0+3TjODIyWKIsvHb9BiJzZcaH5tz32Oco7MHA6vWaf4DNyO7o0cM2qQ5tY+2CdfsjvwdyNBz42porMRbepzZwSb+kSZ/PZxN3zzHHXfv1cShzbMkG2MqN/bZvyqPX6dlowKZOOtKaeanrlelSm8pVZa53yo1y56y63I5LsLJjoh6KHJ0QOzD1ndhGoBHTivCfrbgoBzyQRTIS5sGxqI71APmMjnRosHtJ+xd8hNkpLP7Q5ay/qKxEC3lkowa84neiqzTvLWICcc4TY6tTccqAiEYGMQxBRnHYeb3zne+c4YsRsND/5V/+ZXvVq17lnQDBoSvags19yYKd2bve9S57yUte4vg5HIX84z/+434emHAiDPic98V5YnRC+BfBRRzCkhEFyXlOn//85x13N7lzOru9Y6fGeWL33XefjwhQGDjJirHhv/t3/7N94Qtf8KOlif+SL3mh/cIvvsvNBSQPVvukDUdjwyqiN+inUSpTNh/kMD2sTs+fT19GUBY8HBGN4ywvv1wD5eWNHC3AbUfYanmhpGMPsTLHkr1SthEXClVKNqDRlyuuxE4cO263n7nNDt90zEqPY2d4zs8W67CT6aOetPCOAnLlBX9G076bigJCYA6BnRq7Vmrp86Nyy7gBqdsd2GK16TcqodD4vsovRumNrFpON5SzV5AuI0mjJEaPbFaovGgDlBfHxPzJn/zJbESvdC+PqR0XdmQYnYKjdIU5r4v6w+A6d8Dy400dq83hU+bPtiNvuUOeyJ/3F7/4xfbnf/7ns3yBozwi58c//nF7y1vesm9bMfH/6Ec/aq997WsL84ws6B0UnXjmMs973/aTD6EbbrhhVkG878WJKUpkZWXFO1kRHiMlbMFw4EAfX+EinBgHPL+UnIkUcQjjOGeJjk4nF+2Iz6FrOf+YvluYX+AnnnhiLv3d8HdLx36IKcc8B//Hv/jFxJ8yNHM7LC7IxblRK9OBCQpsaOsba9Zpb/romFMcjiwtWmnQs5WVK9bttP2GIzo3nbrM1JALeFl3s3Q3Zb3OBDEt7Puyk9//OH0vVR1O+CieIaM0m1ir0bRSo+KnR7caLTd0bSwv2Kg3skF/aL3uwDptjFyT3KyRpU8QyEQ6fpoz9alX/44SdVXjHs2S1ZoLtsyhjOUlPyV2ZW3FDVoH7YEt1c1HppsX1qzb6dq4z3Q1HVVNeyhrhIs+nrY9fIUZicgWiTg5tTXat35YlIYf8WUHJpwIR1htVfGRj+KebT/nKfkxMyH/ebrywvlredpeZFWe6XvXYmc3j9fs3kllRFpTgueIihc86cKhkymTDM3jO5qckZimSsLHVzjnpXfx5F2amlGfGpT40yn0ywYfxYvOPP6kRx6Cz31o09CRF97kD5fLr3d8OdHHjzjACB7aylPEVZ5QNuLP95CMtBYW6ukyWx8xscPXt1qNT0Qu2D/904P29Pnzbl/D6ODEkSNWHnStt75i62tr/vH1sJ9Gbb5+xLaAr4VxEGEpTTPLTBVZt2AzwfctfZTEdA2FhcmGm1tg7e+nZTACq5tVSz61u3Jp1Z45f9EGF0b24P3/ZJtrHet3htbZ6LtJhp8sgbLix8zLcroyxxHX04VgYPi15ZyzI4eO273PfZEtt47a0089Y5cvfdL44H3YH9pSbcPWSqu28dSaddc20+nV5dQO/Ljq6XXpcFJ9UBeqE3wpVpU/Pu2I9hTXtISj+sWnPSnWDyIAACAASURBVNAu0whza8omHtBSXUa8ZzsM3ygv+YlykCYYwSmOePJD38mnmfuVG1o8RS7yLUrfKW42EhNx+UKKmVE4T9N7XjCKB0+P4vDhlSuamK4wcJILHoTp7LHDQ580fBx0RVu4eicdOOKJw+cRrvjKVzw+8Pg02BgvWNGOvtIEH9OIK5Iv4iidnHlnn3Z6Xw0CXx1xkk6Hffihh/2E2iefeMJHZNhN8Su4sXrZKoOuVQY9G7Q5BHHD+v2O7xByegQ2Vf4NJjuOvkuZFBj8fdLjU7xkCEuct0fWntzcgmkg077p5Y6GEuvauUfOWWfcs4717aHHHrRKs2TdHuf5D9Ogi3U4pnuuwWgUTEVTHfJNJ0+1Zsa3n6MBxq91u/OO59jNJ8/ap0uftgfvf9Daww2bDEbW6WzY+uqqbVxYt3FvkL45KHEoIqKhZNO0SuVJGad8bLUv2hRxqivB4qt9K04wqivBqB0KXvVNuuobX/i0KfjqXfSebV/yiA/visOX/CoTvQtGeHvxwcGpD+pduCoXve/HdyWmQs8JwSgyi2GY6B1/twogPcLEcKS1m/DiKXz5ipdPPM882QQXece4XA7RUvw8ukq/Xv48mdwW1Jls/bIBy3HUD97/gJ179BGrNxrWXEiW6CuDy7Y+Hll92LUmWmPINW1tt7Av2cjYzLNa+kojlWkadfn3ADRoTDhYXELnoHGwB6tWrFyvTUdmGKsObTRE0SdcwhefuWhXNldsUB7ZanfdhuWR33SESZgrPBTwdMmGndfEm8bFj8V00Z4Lf53xyDbXe3bp0hVrVpbt0tOXbNDp27Dbc7/fHdlkY2yT/tAX+kf+9fjET8NgJKFvDlzbFlRQrFP1CcBUB/Jz1JnMecIO75HXDmDPSlLMB2HllXzE95x5xMvT9A7MvPKYFy/cg/qz88REQIyUIcXLL4onDg0rLQsscXqEi68RDGkKCz7iUxgqNIX1DrzC+NCSi/GE9UQcwYIneNGQLxh84niifEVw4gFN0kVbNHiPNCIPhUmPvESLdNJ450GpcFsQC+iMxYhjaKQpD8qLxskCOvQwLrVB14adTU4U8ws7hr2ulSZc3JEW8n0+54anwFf9qU51F4MwV5eet6RciEmX2lZsUsHkgs9WWN5Ku6NcA9fm056BWb80svawbW12JDnf3++JnI56OLliOo31fHrZJSWGAdgQMzCucxtN7LHHn7S//Mv/akcXjtmlpy/YyuUV67f7bnLBd6NcQcfFvE2m+7WyWb1q9UHdqjV+r6cjPB/5bdUn5aqyVb2qYyOP6lIwXtaqsHAEFPGiJToCE41IV3SiT7rehRv9nG5MmxcWDr7kizyI5yFOsDEMDnLF/Ef8eXwVL1r4+8ET/m6+j8Si4Ah7EKdMas0rp6EM6Psn4GKGkCGuY6Hg5CQfc3OtWSguwhBHJ8YJP8KBq3hVDLARZqf882ueT2HFXz60RE9+TFO4yKc8tOYVywc6pBHHDUe8p5uOJv4pDyMWVzGlstXqdVtcXrKFNjvMZtXpeebkvUKZun0EC+nDNHVkrsWhiv710taUnM97Go1aOp+fXc6pAiiVS+YX5TLBZLHMdSe/5hVfOK83E+S4PPG7LMfDdELG5rBjG/02Hzm5cSprbBUUn+/A0vlRxen6NWR0w1b/QoB4PvoeWr83sNFwaJ9f61iL3ckBtrXc5DRMhz4O06dRNXZex3y90LAx3+N1B1MFrFJP5clCNuWKDKpXQURf5Q8M5UjbiU5tijTqiEc4EY7wvHjSoEP7zennNPb7Th7hi1xaFyZurw5YlVPsQzvhiz58eXAxvBPuvDTRydPdTgyGYsApFuzy0ZkVJ4EiMmkxnncyqN0/0kQ3wkGf3UMqLMZDmwrkjKc777wzspqF2aKm4cnBEyc+yIydC7uXkj/yAFc3rcR40aPxIB95iOnioy1g4BUnXHwaOffqRTs4yUY6eWaHh3zGeNGAP7ufEV9p+Jh4YGvU5wZvuvxk7PzSrdepPrD3OnL4qPPa3NzwxXYUD0pmodX0QwdLnXU3QyjR2bjsdohiSkoFI1EUSq1S8YcVMP84gF/jqcLyC2rZyHSzCE63KPvIqjUeW4+LQjB6LZs1gGdExOiyDx8OW5z4hSGswVUqrAWhpMouJzZp7ED6+pWvyaHK+DFLN493e0Pb3OxbrVyzpfrQGiza93vWrFasXqtblc2jRtOak6pN+lMTCx9mMnUtuZmGypO6OH+es8G4Felqw2zqN7YB8Lhz8dZbb517ZRt1w21ZnBum9iEavFOvtA/9UCpNPj9g7A7SznCiIVnYHaV97lfJiQ747A4ih3iqPOQLNk/nnRNm2H2l/eaOdOTmPLR5jv5H/oHlUb7kC18yzKOTx89GYhQMSujtb3+7fehDH/KKhZgY5Ih5vGAxWEPAWNBK4yyit73tba6IcnzoU4k/9VM/5fYq4EMnOt5pKLj8l5M0Col7Kaks3nMexGFMSrzwCePwOQrl+77v++xzn/vcTAl64vQPMmHsJ3ilQReHDRB2aM997nM9/4oXD+hjZ/SZz3xGqDM5iaCRvPnNb067ezOIrQA7jJzHdvvtmJik0QANEgNleKTTHirWbC7YTadvtGcuPOOGn/z61htVO7bYtHq3aRcf33D7KhbQIUOnKqMEsMViccynDyUrc3aZT28YpmFagbJCeyUbNMZLXDPi34ZzKu+EHc2aTTB25WPtVsvGGM4zcplwVHZaqGc30weEWN/7oWaoyiTHkHy40kJ1jWzEmTucZIGiZCSB7RdyVTatVy7bUmvRJmNOgGjZpD+xKmdj1Rv+WRI8qrWGLS+Ylei4k/R5U6lU9Xrk3slPfvLTXgeqq63S3gqRRvmiYGQHlsp7e/vkGJs3velNPgjI02k72JG9973v9aN4eJcy0w/uvffea3/6p3+6re8gBbSA4Zy9N77xjbOjeojfixPcpz71KfvGb/zGq/IrWQWX01T+uTcSOzrNFgQHHvJ97GMf8/bLIIA40aOv8aPxZV/2ZfYrv/IrPhoUTfngf/jDH3Y7NHZBKZ/cAVvkZkqMRJhiyMfJqiJehLRb3Dxc6EsJ5DTAQQmhCPi1UyWDUyS84lRQ0KMgpOQi/Vwe4YiGYOGJfPvNv+jBHyUp+aEfeWPDhkKZ56hofmnnOb5WOHHiuJ09e6t/upNop3ynckpheDS4fHb6AXe1hvIp20K9YpVxz0dYA7doRz7GdCgVOjjDRRoPiiStdzEFIV/I5sBsPgLjo6Y0gkEON54tV/3WJC4iGaDIvE2Rm7Jx92NpUrESU74RP3LsEaAG0YnwxA5s4h+S8yG4K2k+Qkf2at2/jRyPyq4cJ4OBWX1khw8vWWtp0SpjRnK+9G/NZsP5YV/B6AvFV62izdKhiEhDnrE/40cDC/LdnOqQHxHsKPPTRSl7yogRGCMR6OLULoTPLKOocwJLPPXG1zKpLrc6rPoCbRNakba/7PEPyoG2vV8n+RmFMdPBaBwnWSSf7DNJU94jL5TfLbfcctWROiq/effFikYRTdJm00kaKULBCEelzCtwES3yYZQzUyHgK+M5LvFobHiC751mChTpFdEQTfGWD7rgFUe+iBNNhfFJk5KZl3/h5fLzThp2QviMaskPdFWO5Elh8Y10iCtyCTaNPlP5cKN2z6eInMaKzMKFB41saWnRFhe4l5FzszasySdAk6Gf54W2In+MqspYwJcmxidAPqXkfDJP31orAtbzPeGj65qNWUOinuh8fokHC+PUezqEAlk4StrXtbg70qe+E5vwidCw7IoSJYKyAo8TNHDk3s0qfJSZjFPLE+wNp4a1hrnFxPPSWGhatVbzE2nhUKvWME3zfPRH2HUxHW4ap4hxam2N+nVjV7ikdphGFGoj1J+k2N4J1Rbw8zoEgzpJdZRG+IT1TrrwqCfCzmVa17x72U7bT6rf7T/c8ISe2pYTOOAf8YOeHPyjvIqXL/k1opI8wuGdvEk+5Uf4eidvwPKoHEgjfid80YkyKw5/NhITAARxEBfziHCQsOiIpjIvnqJJOnE8ZFIZVXr0BRvjCAtfPOO7cKIPjmBjmLgYn/OJ7/CQQ2bJT6VHt1N+BFfEU3H4acpI2bAYzWdC9LzUMaEB74sXL9hw0LV6rWKHlpesXiu7USonVbBW1awxBWQbMU2v/JMcX7BG8XqT8JEQtGhc5KPZbPk0kiN9iGeEA28/HNF/FEauuKsckoiiQ0EyGkIhlcx6g7p1qgOrDsfGYMz/jJKRK0a7s51DB0/5G7rCI0/TM9OYyjLJrJitbGza5dWLduTQgt1+21m74+ztttxo2TPnHrfulVUb8WXAuGt9LhcZDqwEL2hPq4qyTAoj1T/xSYklhar6wE9wqT3Edkk5yBHmgW58lK44vePn7QF84oCNtPVOmuLFK9IjPC9ecLF/Kw4fHvOc0sBV/iVHlJfwTg6cHF908PXMoyE58vSZEssT/qXfo8CEySCFqEz/S8s3j3+UO8ob4/eCOxeGM2e8KzPhY6EKg6upHuNj786m8Y3a2VvP2H33PtdHPa1a3QaTgZXGFXvq4gXbXL2SPtguM7rju8eKVVkzKnOcNUfdcDJr+tXUIu5ggJIaWMX4mLzhi+TcHs5qf73W9MtqJ+OOW+HTHTCjGI5GNtShhP2RjYdjv+eSOzC5Zo4PuPmYk9EXo0BUAjrPT5lllEZepwv8aJ8x62eVrm222RAY29JCxZaPLNqtd56xO+66zY4sLNutN5+0Jx961C49/pT1OkNrjBvWHFWt2qi7rPPKdUuJuRTbwGhzaoP4eV3qXWl6FxG9yydeNAUjZaA2E2EVli+cf2kfeYr6Y543weTy612+4HL83fI5O56awoOYtKn83QgoHdx5zBUvvwiWNJzk0JCVONIUn49uSIcej/AVhw9u5Au+eCmPEV9pTqzgT1G64qCncAFqYZTkJjHi5mHgiEuyjv2sLoY4qVzATeR73Z6dZ11tNLT/4ZVf6SYIm2tr1lqq2fqFNfYBrVVnWgNuspZHaQ1GvTTK8xvDmRal68mYcjE6qzdQbijCiZ/V5YoK21fWtpgScpcln91gZsDGSb1mrWbD6uWSfyHQ6datbn2/uLecdJOvY3EuGorLrT9YRuAzJr+whNEX63KYbJRtiA3YeGS1BbNas2w33HjE7rnnNrv7rtvt1rM3W7XJ6G9sh44ftd76ho06Xdtc7VhvULNKp+PlxCdSNJP0NUIaFVBulGvyVQdbyw2qF2BUv6qbeX5hRU8jqS+eSHd73V6tGNT2BSe+RXyAkQMuviv+oH7Ot4i+ykh9C16E1Z/Ji6aTko33aJaiPq508VV8Lr+PxAREor6PYk1nv05MczzFyxc/+YLnXXZkab1i2jMFkPmRnsLzMhpRBYufy6C0CB/DRemKY9QSy00VSbrCkRZh+JMuOURLfoT3dbYqSrhszUYDZBtjTuBKHpOFsjVbLWs163b+ySfsiS8+Zvc97zm23Kjb0+eftCcef8ynkK1GOsc8LayzcsX0DmWS1qnSmhFroigXZE8W+MkWDJ6sZdE4uVEc4/00egPKT7ngR4V1jnLdati2jYa+UYD5hM8INcLicyWXHYU68ZHbgPUf+iFV78dhczZZyRYXKlZrNuzYyUN2441H7M7bbrY77zprx44ftsXFBVeIfa4U3Nx0pYq9XK3Gt6Ssp+HGPgWeTVsnE9/Fps/Hso5hlT2dDKddM8GozgSHT5ziIxxh2ibTczrsfhxtB5rg0QakBItoACe+RenXKw4eeT6RjT6g8hIvyYtPHigD4QJDHO/kExqsrUUnet1uMtyOaY4PIsQhwsMpEdhpUeDXuzCghx0UwhQ55GC3iBtPJFOEQ1bkYxezSDbiwMfeBFjBkC/CxLE7ktthCS7yKgpTJvAHv8hxSobS4BmdeOTxEUb0UeRFbvnQsp1/6mmfvs1MLFoLduqGG/woHP+wmWnYaGQb66v2/3zwg9awsR1ZXrSHPv85e/qJc1YbbFiNT4bQP2XWvLhNmw6SLhPx9jBVQth3UVd+vLSP0FJnZgTHmhgKs16t27gytvKEDlr3XVMp85Hf7l22LheCcOIryosZcAVd6tJONwO4hSltCjC1ZSqLshlOzGrVkh05XrObbjlmJ0+ftJtuOGmnTx23EyeO2KFDC+nLhcHIz+tvt4fWvbxmg7WObVzhI/CujUtN30CQsS7lSl1Q1tgs3XHH7d4utupF6zpbIxrhsOsMntpTUR0VxanuUYLs4HE/qdwWX8Vc7YMPHOZL7BDKFjOHJJ7dbXV68c3hrve78kC/5Dgj6j93yHLkyBG/TSkqMckIDXZ30T0osuhIQx/Q98Qrps+mkzReiP/0T//0VcaeEWG/YQkJHpnjPK6/+qu/8oaDYNFRCT/6oz9aqKQQns79G7/xG8b9c1SUftHgwYMd2g/8wA8Y9jBRiYkHhfzbv/3bbq8CbxokPg/wuKJCUhxb7NxL+fznP9/5KV70ocH2u/KlvOPnsMIhTbKyBf+bv/mb3hAijuhg7PpDP/RDfugj+MQ//wUvtP/07nfbmVtuIsINRckJnfbiM+ftff/HH9qhhYYv8C82G1ayrlX8dm1uxa65wSr1guJgJMW6l3hTvuwyoliS/Ci1jh/37FewsWM2osEl630u7mXaB6yP0vwEhIE16nU7UT9mfZRrf92Xwnw5jEX66QGLPuJj18HPpRvbidN1O3S4bqdvPGRnbjttx04dtkOHluz0idO20GTXleUybmwqWafbsc2Nvq1dWre1py5bpZO+TihhrMaaXbXmXzJw6xGy0XZOnTpp73rXL1i3yy4vGwn86OhJ9aVyUFkzO+C4JGik8nBxd/0jOvfff7990zd9k7e7XZEyAGhgf/gHf/AHV5k4iD72h9/1Xd81u3xX7TAjdV1elX/1HWR4+ctfbh/4wAdm7R9GwEk+ziMj//lICzhgOOfv/e9//2xkFgUlnf6rH3jxB2amxMSQThgZR0IHCcMch8+vOg2hyAmOX5sih0yYDuRaXnjgUKDYYjEao2HmlcgvYFEBFvGLceKBDNhqYcujyhOcYCSH4uWr0COc0hSHzKJPXI7DKO/ylctTOzZGmmO78UyyqROtISYeo/RJUWXCCatD47TCxUbVDrVqVq4fMhv0vCNTM2kKwPExaQhPB0e5U0/IU6mXrVFv+HE/rJ0tLbHjmnYLWQfrD1lLZdF+5AqNckEhevnwvaKP9mpWa9TtUL9rT6+vuH1YCRuuWtU3CkrD6THXbrYxtBtuWLIXvvhuO33TITt0tGpLh6vWXK7a4sKCNWp8PpNOjt3Y6FivO7TVlY5deGrdNlfb1hhWrD40a3BOGvZp1Ykb2/omBHdw+uQ5TV0YGZBHlDVT8S0ltr3zqWxVJ/IVv1eftscXG/t1tAN4MnLETg1bSpzah9oiozDBKm2/vA4Kj3womPxeWeRQm0KJk39GpMDLSWYGB4y2tJQ0Lw8RFxrbJudKxOeZR0TM9+KLJgVNZvKhYqQxjydy8NC5eHCSDV94apCkC05pxNFYgYlOtGNcHhYvaNEQ8clLdOIjP+IoHOFjmHTweCgffOhLVt5xpPHhNPDkjw+iFxpNt/fyyev05FVuHSqNOIpmZAv1qi21Klad9P1oHK7PnoxSHvxkigm7k9wcxJK/WaXEVDMtJfT6vWSISlvgm8npfYnIwmTQJ4hsUPKDwTHWZaaFaSSGfMjd7w19Mb/Ft9itpi0cWrLexro35F5/bAM+RaIssea3kTWaFbvh5DG77cxNduKGRSs3ejYqbWCFa+Vq3QaDjo37A+tubtqTj5+31Ssbtrk2tPbayJYbh61qfDc5sSrHEvl9lenooHQqRipHlXePD+CnI/zUFShFyrdYiQGLk+8ve/xDWRwED/KUr9oDPg+0RE9papvEq83sUbxrBov8FEYO9XvaA2F8hcWU/NG2gVf+iFP+BMe7HsXhVwGGKY8A8PUegQ8SVobALRIspxnh95MmWOFTYDzRKS3G7SdMuSgP+Dxy0CY956n0nXzJJRriI/pK550wj/PxcMgjU1M+6OaI5/HAGiyIN6pWLY2t216zvn8TyveJHCON8mpwJ4ibN0CFLlqbrsexroYdGIv2/HLSf2lgdBT/FpKO7kdG16xaS6Nr8P0rAUZwfH5ExJj1qoFN2m0bVdOSwGhjzbrcecn3lBPkK/spE9i8NqplK08mtr6yaqXyptUWxjapda05ZkG4aQ1r2JPnn7Snn3jC1q9s2uLCIWvWGtYecjrH0GqNljUw2madj51Nt9gfWYkNkVBfiKY6lJ9KgM5TrMR2qsO9pKke9wIbYYSHL1lpIzw4pZMW3/3lWfoTZSEsefD1LlmifITzvql8AK/84cf4nbIxO4onR8jfdyLyrzFNBffPJZvKSw1JfInfi2Lbm7ysWl3tYux4uojOCRRHDi3akcWm1UpDH71UDMtqPgKv+miLdStfRGc05UdPl6xcSyehcnJss9L0T5L4leQyD2SsN+p+ggY2XBXfaav7Z0GYRqAE6o2mj8agC/1Wq5mU5GTsZ+8P3CYs/Rgw8hpgezYuW63CiRlmpcHA+httW7m4YuNR3ZrLFRtWOlZandiVCz0bdCf2zPlnrDQc2m23nLUX3PdiG/fN/uGjn7Erz6zZqJrW6MiPm5G4ER2de3rqhkqwqCCvLtr/bmP21p4Onj3aNTzw1fahprhIOaYTL5wi2Ii31/BVtx3tlvlrFUCdXHT2Iqhg8VEIyIjPO473GEe8+BAvOPwcNuKJJnB6oB/xi/jHPAhWeNCPTr8wkT5xwMmXTMITDcwWksU+8jHt4ZKO6VcBzieZNiy0GrZcP24njh21VmVik37XKo2xjasLNuKKN9Z/WJSfHm2EUSpzSWTqtLGpGvktRSygp3PFatZs1n2BHhMIppm4Urlqk0rVR2uD/sC4V5IRHCM2cu1TTPf9XAobN6p+IgbTVRb7B9x61OGTk7KdPn7YzUbWV1YMm9aNS+vWbTM0M5tUBlZpVqxU2fCNgO5m344sLNih5aN+XDXfiN946rTVBmVDh1Un6SRaP9rH7TXSzqffX+lKjfaSShdfj0+S2eDY50hM7UZ+bAOqw2vx1Vbw4ZHz4Z22g7+bO4hswpEPj9heeSeNuPgIPpdPMKKzm8y7pV9lJybGuyHuBQ5h5SQ4UxMcDX2vTnR8TWi6U8ZOau4YYXgHmq4tKX03fOboOPDhAXyUL+IDQ96L+Itf7gufytSiZqSvcMwffHLHwX5b8qVt7M12e6vxct9juWTLiwtWn9R8yjjsb1pp2OfQVrd+n3CYIsfrjNKpJRxciBLz7xhR+Mlk3jcN3NaryneJVTdc5bQKP+ffzTKwTysbSm3Iwj4mqeVyWjNEcD9ltucGqoRZL+MkWNeXE5QwR02b1WtmJ49j93XWjhw6bP1OO001h11bWbvip8EOxgOrLfDVwch3Nn0qbHXbbA9d6S40Fnyxu3N5zSbdvtUxzfC5bPooAL3FMn/6JiCt5yWFxQ8cTyppZFJbyMt+p3fyjVP72Ysy2Ylenqb2gQ+PvG2IvzZjwKeNqt1FekVxMb0oLP5qu9BQ/5evOL1HOlG+IrkOIlOkP+spIoQdF+eJzROG3QPu5ZPQkVgejgITJjPs7N19990ezuHnvYsXlcQpGw888MA2/pKdQmZ39Z577tkmv/DZ3eMkgIgvXOTDRINTKHJ8ycUWL2dKRXyl4dMBoh0aNHlw+KSfPXvWO7rixR8YTDg4BYF8SmZHnv7ycacnO1Q0KqaQo8nIztx8s3/szO8F3zFibFrn4+vBwDqbbRt3N41p5LiKghlab9RNZ1SwpuejuXTOFn2eHTyM+Nmp41QKfmaIwyC1MmQC2bf+IBkcVmp1p8MIjO8TmU6iGNEIvp7mGzncpJTWolB2g87AbDC2Evp3MPERV83K1sIAkkX44dDq5KxcsaXWsi23FuzEqGubvZ51B1176vIF600GdujoYlKINrbNQcfX38r1qg+02LEdjkpW5nMmRicjPgBHbfODmh7KjtD2Okin4j7++BOzRea8DlQXuS84dgc5rkc7/DnctbzDg7aJnZlOkRA90uhbnGUmhUNc7mj/tE8pFdJVBjls/o5i1n204EQ85Z/+g/4AVnH4PLR9rAaut4JHTldiMIE4i7bveMc77G/+5m88o8TLITS/AJwH9JrXvMYLK/9FEGz0lVkygRL42Z/92QPZoSELSgo7sk9/Op0BlcvHFu/P//zPG+cexTTCyEEh/8iP/IifF8Z7hOGdxsG9ls973vNiFmZhlPtb3/pWV2IRnzAOY9ff+73fm52HRp5xajTYGP36r/+6l7MnTP9IPuzAfvAHfzBdyRbkE68jhw/7vZ233X6790cW3mmYx44cTYcfOr+Sr+t321wIUrbJoG82HlqbyzfcbiudzcVIkoMEXTbvBBwsyBk15jd7w3M45EYiPqBmq2/kJ1dw/DM3eFup7eU3mTCyYxrL5bppp5Oz8/kEqF5vcRqhp2MH1h0NrcLUtmdW6pk1JjWr16vWKNWtt9axcbtnKLVhv+cbFK0l8nbCNxtWN1etv7FhKyO28is2KQ+tM+7a2qht1XLDJvWSjZgmVxpunIvphV+6WSv5yK5W72IFNy1xrqKjYzFyZ3qWfoDuv/8B+87vfL1faacyj/U0LyxYDDWxQ+TH6Nlw3Hf6+te/3tsx9PP2S//ghy7GA0cd079f9KIXuXy0GWTej4MmePSRHFcjRM4T+57v+R43g8plAAcTK548bT9yFMHORmIkQhyrWd0vF5khBA0/t9MqIqq4iE8cNI4fP67kffnQQglRSWh7aIk+YRwjRBoQtipUmuIFBy4HE4KvigVPcOBig8OvVcQHBhooGZ55/LE7Yro3z8GTX9PcQRsZGIFR/kX0+T3hRwYeyEfnU76Y4k23Av2sLxqKj6abzLpVpQAAIABJREFUdStjGzUaWnd6KS5rW+zScZkH9Hr+uQ83D3Exbhoto3yRBwt74jFvGAw56G7k54X5BbtuC0a5+31uaVeSi3rHfHbC8TrYi7EuVpqddDEkj+W6NdnNJLHWsEa1Zo0amwWcj9+3BqdmMPljS551to22jfkMZTCwZqVsC6ynVZi6lWxUHtikwtBxZPWFmlXrNeuuAovIaWrMybWumJHTmwmjCE0ptzoyH50jwzPPPO2W8Z7/8COe11l8FyxtGzs/fOKul6OeaTv0S2YCtGGc6n83PoKj/3LeHj+2B5FPONBTOPJGNzAafTYUVeSTh68ydtVaT+zkIPFO49booigTkbgyKj+m7ScMvh40PnLgonySBdlQPjj9OoAb4yI+aXLQEE3iwVde98Nf9CVT5AEv0YrpyCfe4ql3cIClS3jc9ANtn755Y+LTnGo6Lx9YN2V3JL8ABOXGihXdFrspzg9DibFGNObhViGmcm6XWvE1L0ZWpenoDMt8zsPnuBymZrUGtwZRvqypaQpZ9jUvdgB7bkfHN5HwoOynJUw9sEFYKdny8oItLS96GfsXAuyeciZHFbMN6mScjrBm5DaupKvXqhVbWGIUULZqkylzycYD7rBct1FzwUeVzXrFbyHHiNbt3li0L9WszlV0HJzoZwAlXYYOpS68bH16tNXGFae2sZtPvai9UJexjqGVtwHVqfhH+oqLONCGDrRpH4TnuYgnGMkAHT3IjNuJlvCjD3yOo3dk44ltN+ISLpIvh9nv+2x3UsSjr7CYx/fdGJEx4JVB4ep9N3zhig7vFI7o4CssWryLPr7CqrCIr8YmXHzhgwes8MQr4isux9c76TjJEN+jbIoXP73jKwwdyLHeQ/N1/DL3PTJFZPEaXmm66KMkjF65To0r2GxoXF3GWlm1wSGCI5uweQGiD0nSN4y+xlZKa21p9AWNso+SqvX06ZFVW+nWJNgNBzbgBqIhyxDJoJWPuFEeHKuD8uIYafYJ0oGJI4/vjQbWd8v5VJ++JFFppCOnubKNaSwKctQ362NQO7beqOdqGNlP3njcSq2xjSt9t4frtNdstHzYFqp1v0h4Y9xPCtHrv2oo4ca4bJVayyplnfq6VTexLgjHco/lr3ot8gWHr3qMdQ/dIqd48BTOffCUThphHsUX0d0pTvLh48RvJ5z9pNGvivrWfmjsF9ankzCVUwHp/aB+pCP6xKnwoKsCFCzvKgBVmPyDyAEt8CONyEs0JRNw1+pEH978Kun9Wukm/NTJRcsX430ZiwMIk05iDcpPhqB8OZd+UrVyo2nVSvrYmywy3WNH0suatazpyJXpIqM1lBtwLPKXKgNjCFVv1PxYa0/gZH1O0uBUhhYzw5KfHAHNHusePT5j6lu/j2FsUmjQHk64m3LkphcoNkZ40O5PhmY1Dlqs+Jn8nItfrTdcadfqNavU0ukPR5olWzrSsvLCxDqDdUO5MoXpD3s2Kg/9diY+DGUhfzKgsw+sWuIUjfLsUESUvbf2AkVwrXUFPg/liov1H9tW5KOw2iptkbgIr/q+Vv/ZoBllgr4e4pW3CFMUvla5qipwMVWHVmGKKe88e2Eo4YGN9KGVv4u+/EhflUmcaAou+oIjDvq8M2zGV34IR968y8Ww8PElC+nCJ44nlo/gxAu6xIkG4chDfPfqb+GmPPDO2VreVdC7biVKftLOYLfXs43NTauVR9bwW35KPgIb9zq+0O5TTEZkKDBXfgznoM37yKek6YNovndNx6vUe1X/YNy5+lFfaYcPpYlR63A8cct8ppN8euTTV0aD45JPDQeDMfsLPkLjliUUGLuY3RGfQ/X9I/NWvWkVY7ex4sq0mha/0jSUa9sosHLVGgs1nxq2WlWbVMvWGw6sw4ftDZRq3T9SZ/Q2GE78dI3OoGytWsuVNyQoKf8aIfx477UuVNcRXm1B9a/6Vv07zzmKiboEXji0Wznxgo54CFYwRf5We0mpeseHFo+ceOj9IL7ki3TFcy/0BJvLVyRbLAfRnk0nFcGiHC5foNY7i8E4MfaXgj95ugq/SLCInuPFtHlh0cRnh1J+Ds/CuXZUhQOMwsLD19pgpAGulKPKg3TJzO6QGqHiYnqktZ8w8kCP9aAmH2OXSn7OfFJbaLGtRsnOoJ8g4TM+dhTTJ0ZoD45pBpKTKfyD5+laN18Xlvncx21eU6eCDpT5mJy1MMcsmTWnl0RgX0Zex2Ps6tKdkO1OxwZTy33W3fyoazZoMc9gmuiXhZRs1ON+0PStXLPVcKXpH5BjN9bkeJ10Kuy4y8feY19nK1dG1in1rdbuWu1ozSblqo0wtrWybWy0XZnV2GxvlH2nclz1tX0b2tC645LVsAfzC3S9wrYVf6yrbQkFL0WwagvUvzoycFJqkKHOihzxql/gi+BEh5EdvMRjHr08XvRpv/SBorad4+znXfKp74ErnrvRAU55Rq69yCd40fbppBjic2YSdlJxKCyhKAR2APUuIrkf6ZFGharyuW1lJzs0dt7YxhW88HMe8R1YeFK5jz76qO8wqqKJl2N3kwdXRJ9Oyb188Ac/4oLDOWWcIsCRKNEJjl0fzEhwxCkeP/KLuHsJCxf5sBNiusgoirUnGs6tZ874rp6vjzEFdPWTDivEhmtA+Tj80C3wfRGfNRFmc+wkstY2VZLIQxKKxEdjvPBBNae7Vmt+YCGjGPjzXaXzRJv6Ca3p9FfWxZjGOgwjY8pywq00SbGNOcGCOyP9hNf0mRPVxH2SjBLLpZpZZeBfFGy2+QB8Yii7cc3s8qV1Gzcw+Rhapc0NWTVbHW/YUr1ty9VFP1G22qxbbWQ27IxsOC75dJVjgPy4a+qef2EknteB6i2Pp6yxldIPodKBp71w36nSRGNe3cd4ZOGdo6iwpdIPoegrHRtCzuvSeWIxD+DTtmWLFdMUJh0bx4OYWEiWIh/6KLJz587NFKx4FsHHOOAEi17AjIQyJD/RAcMpNOgHXEyfmVggBJqQex81GhMwBEDiwdYJJ+3rLwV/lC5mVDJ0sfPCnoR0CQ86cGhhzgv76q/+av/FUYMoIH9VFLQohB/+4R/2jh1pCxgZuE4LXoSjAx4TDM4jQ44ihwL/pV/6JbcjA155Ey8UP+UDbf0IAKP0Ipr7ieOYoTf/+3+ffkmneThz5qz93nt/z+66/TbnywfcnIHf7/Vtobbg61WMplBE7GKijLDKx4jVjVqryJcaDO8UCyYbbAS4/sIUY9y34WjgC+0TjlFh7OfH8DB1TDcWkUdOhnVlyIkV1XRPpOvT8fReEIp8wk7exC39WS9DIfucdsjZYiPrdpGr78qRdz/qp4yNUcdG1ZE1SjXrDHtWb5Ws2qAdjv06uMVa29aq69Yc16w8dBuKJDPKt1631uKCnynmKp5jtafLDUX1E+uW+qGtUqcYaf/+7//+7PJc1T8w4MAHE50cv6iOgcFBg9EVfe8f//Ef/TwwfixzBzz2i/DnR1a4+OLHeWLf+Z3f6e0Yumrj4oV95ete97pZu815XMs7/OjfPOK3F3qSE/8jH/mIff3Xf72j8S4nmK/6qq9yOzfKijoRjJtYwFSRUlIqGBHary/8mCHCKysrbkuSKzHoM4rRdFUC7ocv9LHoxxXhR1kiXcVT6cKP6aKFjRa2ZHy1AKzigRUNxfHOwzuP0iPd/YbheenChdSpWN9wA8QFPzXVZRgOrNVatLO33WFPnnvUj+oZ9nq2tMDq+9jNG1hvSu0jLXD7xbd+kmoqM/oWd3iw4uUykw/XM+koJdL96MXpTilndkHJPyT3qWs6DprDEXmg4WttfO40xtyL/Ubd0oROgwJrjNiTTWzMzumInTPyh/kHmo+Ng5GvA/YuDWyyNsLEzKoNpv3mH4+PqhPrlvs+Aju6eMhvO6rUGv6tJUcO8aUDjd/vo5yu76pOYp1RjnpX/QiOHybs/LDIJ05wSuddisPrI4z2IrzoCkZ0UGacp4ctotLwSQefUQh2aLRB4ZAOT2Tb7b5U2XFFmoT34iRD5BvxlD+VRUzbSxg8puPYmeVOvCmXIvqzkZgS8XmECEHCesdXXGSW45AW40TDG1L4dRMN0qkIHrnIE1p7cfPkA5c0KZ8iesLN+Sheihdc6PAuJ7p6j34Rr5i+nzCjpK3fKLNup2f3f/5Bu+3Mrb44vrCwZF/x5a+wT3384zbsblprYdmVUIvp2KRvJda+UFKUp3Yip6NSPvchr/49IYavwFEf0zqBL+eDeX7KDPnpPBRsuoaNGkpTFX4l0y4nU0lGPUO/PYkbxX0W64dK+IiOyS/TTG0cjdMnV+D4x+Qc7Gh8+jT2ww2HbS7mHbrC4ux9rPdb9Ynx2REmJc1Gy/pdrkzywZyxyVGq1uylL3qpLS0f9ekzc1t4kw/yG+uxqC6AUZ2jaDTdU7sQHfmCh1YMF9EmDjyc2r/oCh754Ck/b2tKQzbJEOmKDj40IkxM2yksmeTnsDFe+clhdnqP+Dmc8o3uKILbdp4YyAICkXDsrLwjYJGQShO+BBFska+4HFbv+MDkNGN6UTinWwRTFLcbnmRBHpUPdGLeFSZeMLvRLZJlftz0CGoMVMucu/60/Z9/+n575au+0o4sL7tt18u+/BX2sY/8N/vIh//GJuWKr3mN2fErtxjTuEJgdOWW/qhEjGD9ZqHUWVjLYgBU4gZu2sH0pAy6Wr251ZBoG26Vz05npeImFf6FkpcPZcR3k1Xz3QVX+tOr4CbJTs3tySg/Mos9Gd/YwY8LRzAPqXImWM0NcvkCoDpiQ2BsFf+WcmxDFGpnbKXmBBM2qzdLNuAi4F7XjwDqoHAmFXv+c++1F774ZX7ih3/8Ph4ZJ+DiyJ+cwqpnxcunbqVoVKfERUe80mJ8DKuN4PNIKQIT8UUn+sgoOeWTLlr4OznqTHx2gsvTogx5Wv6ODJJJeDmM3pUuX/HRV5pkj2mEZ+eJ8SLmESgvlPy9CFZMgRXNnfAijTx8ULyczrPxrryR3ygnjUtx0b9WGdQ8scvCKp3z7bmA45FHHrXz55+xw4cO2aRas5O33GL/9pu+2Z658LSd+8Ij6Tagctm4NdvG7KD2041CTBqZHpbHhpkrwyMOD/RIFuR9OIOh6NRKHOXEv+kpsOSNLkHjmlTrVqr2jV1LlN1owg43So5OrZu9sQWr25h1MToT+WA31UdtDNHSCBe1xpcF1XrFF3lRuMBz5PSwXrJ+vWf1Hkf5dP0jeFb8OXtsOK5Zb1DhpE9f1LdK1W4+c6t9zWtfZ8duvcOt9xllDrrYsPW9zlBK5OMgLioR4au+1TYoG7UNpeW+6IjGXnzRBFZh+XvB/+eA2Uu5InMsj4PINTOxiAxVGFSACliMvMHuUOnCRRjh4It+9BWOOKKPr18owSlNtIsyDC09eTp09ORp+32XTMKT4uJdafjIonfByp8Xr/Tcl3IoVyZ+mKD/qJbK9rGPfcT+06//mv1v/+v/Ykutuo1LVXvZK7/GR1f/5f96vz167hFbu7Jqm+2OlcrY0bGw74K6AqKcXdbZYqmb2/tx0YzQ2GChs/MjX6lMPxr3NbPp1JDvDjGgddtVRvAowlQOTHE42XXAVV6uuPiucnr8D1Oo6UjPjf1nR0qj7EpWG2EOkOzRXKWi5HwIWbZaveknbbCxwEkZVubwxYZ1B+m7ShbY77jzTnv1v32dvfgrv8YVGGtzaG/WXlh7ZbdRSoy2haMeeWK9qW6Ji+1S9aN65h0YHHGqX+HrXbQFKzrzfOHhiz80cZJJPPCVNo/evHjoi9c8mL3SLspjEX3xk19EnzjyXZSGnDMTC17EGIKE9Y4vN4+Q0qOgOR3xkB/pKk7bv0U7hMyJtWMp2cRXvhqj3ov8ebhFsDFOBU1clF3xyq9wBLMTP3CEL7ydfF9khz8LVj7mKdnG+oq97z//iX3tv/lae+2r/42VJ0Mr1Sr2sle9ym665Sb7zCc/ZQ/c/4A9/sVHbGP18uyUVv/0yE9xmH7ozUW3dAIUL4v5fl8kiitNN1nwL5eq6ZhnX+xPIxjWtPhdYyGeXcuSVUx2ZowWUWAcmDgcpNEX+eXbS9ej03bGboKXHyfCQgGbJj4Gr0y/y6QNMsL1MePQLy6BE18VMHKrVZp+ocny4qLdePMN9uKXvdDuft7z7NY7nm/jct1vWmrWq7aysuo7aNQJbTnWEQo32nqpHtSmSKf9qQ0qPfqCVZzo805Y7/KJ363+gQUGvvQLrSuLh/okPiNM5Hy2XJ6/3fjMy5vylOMX0Vdcd87O52xhPxKDgZjIlzCcWYQpAgWmOHCFgx1Nft5YhCsKg4tDWOy0ZKYQZSJMJorsvASHTNi5Rf6RHyMO5JedjfD26ktO4EVX5SOftBiOsOAIL/KMdGN8UTiZPZCSzu7ihAmrVe3Rhx+2//DOd9g9//l9fiv2cNgzDnA4c9dz7MZbb7ev+po16/cxxuTEijTyovMnWdNCuA+1mP5458aMYkhGp+8aUbomc8WR8oIqJA6Z+FaJNbStdSampL7LyCgInZc04cysA2VJvI/eCKf5qdPkRiQ+vmRKimwoxgkGZ3zjCYsJSgwEFG9teuQ2hziaLRxetFqjaWOrWomjqkvmbYtRW6WyZRRNHigDHkZmz3nOc3x3MJXL9mkmp6NwJyonScR6VF/gBzjeTSkatGuF8zolHvydnHjRbrGjKjoOBzrIhY0n/WSvTnLhY9pB/5DSyGnAl/wBqwcYlWH0Iy7x5BG9gR2b8iMYyUC/jfRFj3T6LnduFpVVoRKLTBTG51fq7W9/u33oQx+aDcMlCD6/EL/2a79mr371q/3XQEP1CKNwXlC8U0lve9vbfIdLfAXPOw9H1eBiOpnEYWz6i7/4i7PzvFQ4nmjmCvCNb3zjzE4tl0Fw+/VVsPLBl0wKI28uD2ng7EeONPKYSjid1rNLiDJ4+qmnbOXKZRsOb7XhAEPYpp+EWmnW7FAd2yJGWHRMFvP5n8otzf2gyaQtxSc/TbFIcT3iAhc2malAUIo0p3xczmm8kpPWm+Ep4Otw6Cl4SprZTEBKRXJt7/yUY9J3wKE4qn7pSKfdtg/+3//VfuZnfsZ+8RfeZbfccmY6NU474dQLnQRD0ve9731X1Yfq7uGHH7Y3v/nNVx2FozpEgfzRH/3R7O5R6BbVeVGc8l/kq61/7nOf83sbwceJjvz77rvP/vAP/3CbwXURvTwO+uTh7//+7+0Nb3iDKzPixFf5e+lLX2q/9Vu/5cpeMkALuPg+j/6HP/xhe9Ob3nTVaBc9wejxK77iK1x/MOJUnkQXHvxIFM3QvEVGgREgIpIBZQYfWw00vphIYN75JSuy88phxSPSJY5GiDHqQR18WAvBqp5GyTuPaGMMu9NUYDe+kjeHIz7msSisOMoTF2nFcE676F20ZmnsJgaFyIWwbhyKgWuNT62mO4xpQDNDS6qCVyVsqaBUalugMz0yVS9buFswhLbh6UX+DDR1EKKllshTwk+Kbxae4aQYV23cvOuvQQ1PFSVk/FvO4dAuX77i54NhQM2ls/CgI1SraTGfuuDRjy3+vPPegMOGi/YpW0LVm+pD9mNJOIk4zVdoi8ITHO1e7UJxRT59i76XO/Hn1GDOMsOODB7EKw1fTmmSA5+8YwcZ4SI8YfoO5YM9Z05XtIDLaZA/6NM3hRfhSccxCII+vsoDuEhP+DFu191Jpz79A6KUAEzEnGTeeRBWbp4gpMdMFAkmGrkf8ZSmjOJrPQAlhiyCR1bioszC36sfCy7iKF6+0vJ8KV0yASd5lCbcCKM4+fPSoKFfqvX1DdvcbPu7fsFIv3KFNbFkre/9nrGTr7lo9CMu2302A6jPo0ePuZLgfUtlQSO1B0bK2HfFKSWUsBfjoENuOYdOngdkoCxQEn7kkM8Xt2QAPvE/OusIqV+mzkkadU8nZ8bARa1/9md/5tOXv/7rv/ZROFNFyoc2DC9waCOSh7hUFts7Drz1AAtMdMRRprHzAUM8LsLHfItmpEU4wuRpkZbSiAMnz8dOtCIP8k05UH4xPqcPnPqQ8hbhCfMgjx7eU3vboh9x4KE8ASdY4cc8CE6+5PPdSQGSKEEQkkcVKwQJIDjFAxed4IiL4QijsNLlK36vfsyUChdf+YEO72qwe6V7LXDiLdmK8kZcfCI/4ce4ncKir3ySV+qEdQ6+ksBxIuh//I//u49Oms3G7FSHJGOaQCbltNWwwCOdUQDT9Xe+8z/Y8eMnXFH4t5WuiNMnN6x5/ORP/qQrIkblkgl8FAtrne985zv9GziZOEAfOOD5rAp8RswoBLUp8gQ+Ix3wWTsBP9Ux+ObrRFh7/9iP/Zjfo0CHZMTEoYtueDvNBzgoMToL4ej0jrypTFKq3vGRVXIJV+/Kr3zSSYNujCOed8XJFz188Ypxwsvj4nuUFb7in8PEeMoCB+5uDpo84pPLLhrAiGYMK054ghffSJ844Sq9yPfppCpBfgTMmcS0PCzBiAcv0othpUf4nNZB3yWv+Old9PbCE5y9wIlmkR/xJQtwUZ7rwSfnDU06KaMv+NLZxR//E5/4B/88JcfbyztTFUZ3x46lm9DVwKALT9Y0P/7xj/tlJ0X0mOYDgxJCKUZ8Rkgo3L/7u7+bu6TA514sWjOdQUkJn7JGBtL4CFnTPcoCBaYyVzkIL8YrDC3C+KpD8MDRe1HeiBOO8MEjjE8aYR45weldfoRR3E4+skkRRTjxzOWSHMTjxE/vkca8sGiTLrxIR3knLk+PNJUW4yJNwtDI6YkX6X6eGIRUsQDHQhdxxQtZ8ErnnUfpUbgYFgwjhevl4MkDbcmOj5M8hKnoCBv5E4+cwo9p+w3H/EY5iugrDh7Aqnzw9+qUR8pU+dOUSe8aGWm6ybqRRl05H9p26GuUItK5okIB6Yx2yYjcKBYUFPQlh8pBZUsa+JxGAEzEF13kLMKHB2m6B4FwxGe9RR/3k3eVKzxReDjJAS2lK540XC4zccALB57IF2GJiyM70Ygwop8rG2CJk8yiT3yk4wynsigsX3CSEx8nnoKb5ytvwsvhRB8fWXlUlkoTP9WJaJAuOQQrXzgRFhlEX3j4xKtsBC9/Np1UBBWkSlIcPnEwV4OQLxiYwEwL+8RLCGUEn19L/AgnGtfqQxM54ctIJHdMUZA75iOH4R18YK7Fga/8Q0fveSVHHshHHnaTL+LEMLh0Bh4WaXHUC/HIgpLxM748j4wsInYKb+V7+4gBQ1Ho8gEyIyJMXZQXcFAq8OKhERY50pgSgp9kScoAeJQYIzHC8/DJB3AoLK0/wRu6KFaZHvCuOgRHigwfZYt/EEf+Gdnm8sEPR9sWb94lAz4PTmXmL9M/5AUHfeTN+1aE3SkMHfImejvBFqXRZ4rkUz6gSz3vRF/tR3nWu/jxLnrExfSd5I86SWUpmrP9cojx8CU8ayc5IAgULo2QrdwoCGm8Uwk0JlwsDMKqXM5cYvFX6TETjngNfyhgzlziSBLo5nmgkXGawb333ntVGmxpnJxHRge7FqeyEX/lkfIT/QijdDoxdnbC268MjFKw84GPOkKixec9fGd5cfbjQX3MG4nBVzKlcOqAnEDBJ06UkxSjYKl7pnHcu4iSUcdWHqhv1sRoXyhA5FM+4UUDZk0LMwft8kkGlRV1R0ejnnGKhxfxdGC1NTV6aLCehqP+sbOijRAPvmQgXXEOnP0BDtkxo2DNLjrSkAHZKYfokEf5oF1hB0n5RV4KY0PF5oNsISMdwsQzXc6VqOjTfjDD0FE9Of5O78jJWXWiLZrgKAz9z372s17OsdyUf0bY2HnhYnrkWxQv+pQrfZcyjHCiT/tmBzanXxqPx5PUoNPoCDuOonsnQYQ49zq+4hWvmGVMAsKIB0Y0SAmBgITxeVj8pVHFdNE4qC/6NNLv//7vL7yXEtpU7rvf/e6r7qUkDdloJNjJsC6jzkCaZGVn7QMf+IC97GUv88pWR9lNbnVoOvm3fdu32YMPPjijGXHp/NxLSWdAnv06plNvectbXFFux6du0qgMswM11K3dxd04JVkok6NHj/holzypXMAmzPY48nNkTK6kaDt0UuTjAuPt8iV8Rmjgc63Yloxb5U95ayornvAFlrpFQX3d132dK0Picx7gU4f4pEX5dysB0rlXEvmop+jEi3ZPGu96gKOs4PmJT3xidt5XET6Hbf7qr/7qbCCQw6BAvv3bv/2qo3oERxlr91dxe/WRl37JJlBebqJB/qAPbHS8k8eXv/zl9t73vtd/UJR/2gz1Q/7/4i/+wr7lW75lNmLNaUC/yMxD9F/5ylfae97zHtcv0JUc265sgyhKhkYGUMwMCBqOq5EBI6dGIRwxULroaZojeKVfiy+eaHJ+LdhmFz/RRR5+KegEUX5whU/eKWyc4oS/Xx98eOZ0GOXm5Ss4GoLky5VEzj/Sj2HoF+U/4ZPX2AD3pyiRCUVZ5JCB0RCjLUZMeb7JIz8Syj/v0QHPDyCbB+Dn6YJFBlxMJ45644nxwpFPZ+KHZCcYwUYfeHgwlaYTM1LEiU6e15gW6cCf8pPBNmng0lahj/Kn/NRHRF/5Y5StuCKe/HBoUyPy3UsYetAuoit8Rt/YyuUOPGSk/0S3G70IS5ip+m70i+TbNvaFKdoclwvAe4wrIiY84OTy8Dw8wR/Uhy6PFKsaBnGSXXHwoNCVH4VpZIRxSjuoPDHfokGclGQRfeRT+Uhu4Rb5gkVm5a2I7xau6gUfBab3LYj5IcFfrfjEEx9ZkIsOhUw43sm35ASOR/IDR9krjnjehT9fpv2nwGO/TjjII5lzGpKdeMLxXTjg84ge8YIFT/QVLx7Cp0wVjvQFdy2+ZDoIDeGiO2J+kJU87cXleY44oh9neDHdzxNTw4sJEFWBEa93EYzCkq54wUVaeTjHzdMP+g5dyYxPvuQkl+RUY5IPXAwL73r7UT6F4ZHLt1sZKa/ykZ1HNEVvvvw14ecXAAAgAElEQVRXK6P5sC6hJ4t+Dit+klvyOGborIJTPL7qiTThyxcf8YWucAUvmL34orMXWMFEnCgXYVzuE5fjiFZME0z0RV9+pE/exSvSi2HRinHXM1xEX3HyI788Ln/fCVZpwpGvePkzi/3dCkcIu/nXi85ufHZLL8pwHsd7Lm8Osxuf65Wey7EXusJRxwZHcfjz8yIFJn8v3IDZOzz8Jct8ObbzFfz22PRWlFYUV4T7bMepHcmHH7IV5bsobj/yXSv+fnhdCyz5p10i77Mt8+w8MQmsDiE/xkeh+PUsakTE5RWod3z96orufv1In3DukJF4+RSgcKL8TFeY4uCrkJUn4Yu2+IAPDPCCFYx88RJN3hUGJoaFozj8SF/pRT6wOW3JRDyy7tdBU7LsF1fwyMCjckUW3nGSL9YN8YLBn+ckl2gp/5EuadDO6Yvm9cifaM3zyTeOvEhm4pCJd8kHjNKVpnJQOak8lC/lPeI6s+v4B1n268CRjMKN+Vc+8Olz83gctH62rYlBROYFzL+jU8EiBMIwP93JSXDB6H1eBgS3V1/0BI/syIj8hHP5gWNhUvP2Ilsh8qR1hyJ8Fh41L99v/pEJl8utONIpGxbHgdHapCPt8Q8ykcd5+d8jmQODSX78WM8KU+bIFztjZNad2lkRRxnEsorhiEOYNonbjX6Ot9d31Z38HE+y4UuWCKM46nQnOzPaHHWY173Kj7ahMLzmyRN57zc8r252oiMc9T1gkQ1Z8SUndc8OaFHf2on+bmnblBhMsePKd89EhMJl94CrnySY0ihUHuyE2GGL6Spw4rCTYodKGRT+bj40wIE+Bo/QIk6OMI0FOx4Kqog+jeCJJ56wT33qU0JzX7SoBM6MoqFF2gLGUBJbITVKxcunfCg/4JR/0ZEv2OgLlsrlw2V19J1wIr7ClCvlQz4P4uCLrRDldxAHPteOsfumMoWOwrQdzEfYgYyOfAKDiUX8cVG5AAsMcnEcjupHdEUL+y+uVWP3ucihJMnffjuR5JtXH5KD9rMTfdoO164V2YFBg91Jyi+38yKN9oxpjkZ6Rfm71jj6lc7syvOqMijiQRqKjLpFTt4jvsJYB7zwhS+c/dCKlvJH+413Vyp9N780gcLUEYQQlS3GMY3Kf+tb32qcC5SnA8evCMeevOY1r3FByRAPjkxC9/Wvf7199KMfLcQXr3k+Ro6/+7u/a9iLIEv+i4X82LmokUc6pNF4vvd7v9cN6orkp/FgB4QxLPA5DFvI2Dlhj5SnwQvlzXlOnLtEh0bZkX9o4dj+xo7pk5/8pMfrF0xyAoud0W6jPMHnPtv/nPeEIiuSP4fXu2BRMt/6rd86s2OT3ILbzUd+zAOoF9EUDu8oqd/5nd+Zq2TAA19tRrh0XMqSi1+Rr8iMAPoYQpJ/mUAIX7JgaIqdEka11B/xe3HIQ1294AUvsA9+8INuagGe2oDko16/4zu+w/tQThdeGLLSPzAlkUzAKYwCo31hJlTkaFM72XEV4ewlTvnjXkfssDQb2AuuYMgD/VN2coqPPv2y6No1cJGBcwo5748fq9g3JN83fMM32B//8R97/yBO5e8jMYjIUcA80ZHOgxD82tDYIRLxIMivqH4peBcT0QIeW64ifMHM86GFktGvaBFt4mRjIzqSER8lhCIr4g8u+YYHRpvkgzziRINORgMrwgeORqbCj/IJXzLN88HViRPzYHaKR14UKfZM8Mxl4F2yKI13wSK/8rwTn3lpyB9toCIcPPglpoylZCSPZAFeYcmUx1E+lH/ugMfGik4U6QOHXOSLUzJEP8ff6R3aOznRVP5zWzrSSWOkRftETlzEQ0kzCiF/O7WB3WTZSc55aaKJDIySNRKUfPPwiuLBgZ5wRZt39IPqJuJSNvBmJIgTToTZKVwWQ5iIcRECcDxq5Dks7wjCU+REX+miUwSbxwkXHOEXweRx8T2XP/JXXogDDidfNHJ84ZAu+ZAt0hWu0vV+vX3Jgi+5c38nnoKlMfFcbyf5dirfnKdwYjxxqv+YrnCkTz5UZ8ofP0wKy4/054VFfzcc+CMf8MKBpsJF8pGuMoc+MDn+PLmuV7zko3xUbtdCW/SgEcPkLz7ioXKNAyCl7cW/ancyZywiKlgxlDBKz98Vn/vz8HO4onfhFqUpLhaa4vBz+YsqizjhCz6nIRmK8qu0iLPf8EFpCE9yR18yKG+85+EieOHtx5ccRTjqrOKd+0U4eRz0xUO+YHiP+Yj0FS9Y3nN8pc3zRW9eOvHQFN0iX3JEZSW6kinH24nf9UyTbNG/HvRj/hRWWYkX7zFtP3x9JKaCVyODQAzvh+C/dlg1kH/tch5Evpi3eeGD0L1eOLHBXi+ae6Fz0M6xF9oHgaFuJNP/V/vZvHIh73pUBvNg9xq/7dtJiKpQ806goSYwPPySRMc7D/jg5kND4okTXfmRxrxwhBUN/L0UgnCBRwb9AmpaAk/igMvlF33SeMQbHNGdJzPx4AtXcOKPX+RU/kVp8+KiLMqn1g4jjuQhLuYNHBy8iedRmUT8vYTz/EYc0uDBI5nxJUuEjWGVCT5yqQxFA3yl4fOorkgTnGBU94ov4hXjrndYMhXlW/mD50HkA3+/Dhx4qRwpJ8JF8u2XtvIBrZifSDvGH4T+NhMLCKiCc2Ka69M5YJp3EjLOrgKL3wio41JEB3xgtPMWMyGYeb5g8TFfwI9b8fPwYjz8/9/2zuTnlqrq/9df3qlx9JqIGmMTBzZRQbGLUQOKDSjBCJHEBgfqQCfGxH/ACTNi7KLGGAcKahDFPqLBDhNAJI4MYaKBgfoHMMI3nwqf8/s+665dzTnnufdB3UndtfdqvmvttXftqufUvlU+eCB+fsS2EBcFHvGN8HlqQ/8o6CwlH7kTBBv8uJem5m8C3fMfY6GP5AefNf9z0NhR6DsPb4jbhW3ObiQznirHD3Hpr8pHbU9M43O8qj6xM8bgdz6wZ4wP6Vv1OWrn3Mi6+uTIopw5wRP8Q+Ib5UZfHdUG/+TIOd7pHsrTFzj0m7HNXOyDf+I3MUDZ58LTkTz5BCa57MF5+ctf3p7EdJ69OjxqNtgMkEnm/3R34MSeo+iCg38es3OiUk/sOXtlLLLsAzN++cbCIjW3D4wnq+7x0UaMSpVLkXNisU+IE62LnUnEdw2ZyFuKPsgvj+l5gtrlfwmTced1M/Xp9JKdcvyzj4s80z/jUg6f90Xxlg1kXQ7UTaou+xfZpsBrrrvC3GSf3aOPPtqJp7nJ627Y6pFFfC4wxH+MC0z2P+v4NS/2X8pTQfZRdXMMHeYf5+dokcOe8XMBF9f+6Tf7nnXywvhsvUFIjK5OHMxHnhyzDcaLEnxiWoqrw0ze7k4MJ1ylPvWpT53j6zB1EcMhi9QXv/jFc7fcckvrmEn8sY997Nwf/vCHExM0k8g+EYonWQYzVweDweW7lCR5S8cdTBY/vovJROkKk4R9Ory3yQSnHjGzj46y5B97/aJPnX1cvC+KPKdMHywivC+KzcTkf2uO2EJw0003TeOUORd/jhIPj9fZJ+SL7eb0U6YvLmDsw2IhqP2jzUbjG2+88bxJnFhdHVt88MJI4mMLSVfYosD78MgDxTHSno2c7FOrm23RI99cQIifRTbtp8bGf/TZmSFjbKEcjvWLXvSic9///vfbcUePBYb30bkFw/6xaLGwsY/t1ltv3W2R6Hx3PHDAf+CBB6Z9blxExe70t/LApr98V5L4uFmgzxT8HOprese+YFAWCiYBThKcQFg8+HOAvR55J6QenedgMyKJdaCU2xn8UJdPe01B30VkjX7q4I99SlytOAm42sKjgEudRZqrccafGOpWXtfOQdIOH1yN9Cc/7bVL3to6+XaCr7UhJg5sGV/2mTm+a2OxP+QOLAo8jiz4GO0jS71aNz4uAtwlsgjpE11wmW+MG/MDio3+6QfzlXEn/3WfFnrosAgbf41ha1vfnV3GnnL6QB+z2Hdk7qNCnvjW6QP9Y55TzAHUuti2seVg7rO4MH+4kxdT/UOo+Wdtqbi0ieWQ8v8EgHKQLIp8wZUbhBQ5dSZS2lV79SalMgjyjkWrb2OHGjeTmpg94HNoK903Jv2AwyCKpx/l4Fs3ln196qPSER569XAcjWlkm3x1sbVecWtMyhNnqU4eu/j0iUzczLm+qz12HGJCrS/FsiQ3JvSynm3jUk6bej2MiTlr0Za2deTVVn9g6AdqX6GJa97E7PzJW0vFYqEE/9jlvA+FZEet49QkGBDUgNCjzaFNUusGn3ry1tKK1dl1OvKMGcoB39gzLvvb4a/lJR42tKX61I9tddb6SD37KEWW9dRVpl/0OGxDzZV8baDIiV097fQhnm0pfIpU/hpqHMYlNQ5iQIe2PrKtHnbUtRe39qGLCV2OLLalKevq+MliG3vrGYu4xpu2WUeunVRMKfrIxKSehbwg81Cmvu0tVFspttSJlyMX0S246u6WRR3UTqk4R0c2YlbbEb/qHbudfonZJDr4+FNHeswYxMQ3R7aP5Wc0FiN8Y5CqV+NTbuzo6UuZtluoGIk7Z49e+qOeGNqqo0w+NG3Ug9/ppp066NVDGTTxk0+9FnHk02Y+7lOyL9iLLSbUkrLkKz9Nan5qvPv63P2wD4DgdjpBPeHR4airJ22vdNobJLbWE5M6uikDY66gz7GlqE8cXmmIVz7+OeBlLGt9pI0Y0oph/Nik/6o3ai9N8Iylw+js5UGxN0fdWKBD3MavP/vS+VzDE89Y0gZsYhnJ0jc6GSM4/EQCfmeffuxL8qjLhzKuHBT9Eht1x1x+h5P24qInRuJax4bY0dFGio51KLpiyRcn22mHTeYIX+hWfWz2KcZOXsSUIhvla62vabOrgBiN9jGRGAqdxWl9DOtvaSQEPJ7AnUYBO+Pd4oOnp/xdTvzdPip+2GSfDCUTvuTDQWBAlvaxGbs29gfbNcVxmNM1dn2lbmcvjzGj/8QyF0+Haz86WfqvdfXnYtaGucn8yqJfeDyseSzeSaae/etk6GQM2iQ1Nn/8hmZx7rtHT5l2tsktOs4x+VAxkmfdcy3xE5s6Bb0RvlgdtT9QzpFjbDFJP+bftQWZMafeUt1xqnrTaJgQKPtweIJEUp0gOoTHE5z7779/kskXlM6zh+fSSy+dXQTSn7b4orM85marQ1fwzz4Ynr6gO3eipb2ddx8Ykwh7+PYBylMZfWuTOKO6uiwCfPcPCj6YUmyZJLzvKt+Hhg6HGCMfac9E7Qp+H3rooelEBpOSuPSb962Rh+Sjhz5P/si/r0sRQxwWfuLH3n6hU4/J8cp/9EE84ILPyUhbGVD44y0QXnyQ2Qfr5IU9gD4BTTn27FFyQQBTO/2oX0OXz9M1tiF0T5iZi+TOBUDsxGJu8dk2ngDrP+XU9WVM8sDnFVDiq6ccyh5Mzk3nR2Igp2CXfNoc4PNJPfLHfj5K1cv2pND8Iz40Mcg/awt+Ekf/DdR5rLRL4e6SAjiT/Oabbx5uViSBfNfxk5/8ZGJMdRxwkn75y18+77uUOrdj5xk/wSB5vE/o7rvvnjpLxynasz3is5/97LlXvvKVu8EeYXV8FqkPfehD034b5CZcXdpu4ViKVZuknPzsk/PKVn2wSPA+pFe84hXTSenVl9yr2/ml//DZWsD7qPwupXnRhlfAsM+JDa/Iav7YYsD7ttgvpY3xo89mUvLPp8G6wiuK7rjjjmkhxF6MrBtTZ1956tJ/7rDYB8Y+IvxQkOuDNnl1I27KrLNI8T43+62tcu1pW8y9/uQnFYdFin18aZN6nB8sJOpLjYeN2uzzSv9pP6qDg03FV1989jdee+21m/HBwQc3H7fddtt5i6D90N9Wav5ZX7hQkb/EzPpWbPR3HwqhgTPuEnI/igHgiEWG283RPjKukhycbCQcW45RSWx0uN10gmTHqItDbOAz6eWN8OWLxSTmTwriZwFx8NWDqpu8tXVsWQQzrox9hE8c2HhUf8aEnP6zz0kbMZFxJ2b+Ogx0uBCM7LnTYKGv44sdMeQdHDz5+LKesVqvsdR26nGXTXzwwExs6p4A2uhXGe/rUpYY1rVHX17qwx8V5tzcPjxxsM+6eNh7lwgvY1dniXa42nDOJb78jqZvckJsnN/cZTLHzD221ud860NcbeBjZ1u5+omfvK4+8j/tE9OBSralAM7JdFgniHxwEst6YlLncGFRB4y0J9kUdOWrK9VmUnziH/HVkabOMev6S0wWTv1Kqx7tWlLXxQu95GMDL/NTcbA1v+kn62JWW9rEzxh3xXi0T8xOHx46HrbtX9qIJTbUOnrWoeqmfdZTru/kpe7WunGknTxpyo7lNzHX1tO3sUHlS+XRpr50qAf1MKauzXjDt1Qd+VB8d2X6c1IQA9SANjL5tpF3ztRNe23TOTwxOCk8sTrdtMt61aWNf0+yjMW6fvUN/9hFTGni28/0n/3obNKeun1Iap/hZT3x9FOpmPL1h23aU+8mnPpSbcCzriwpcgo6qSsf6pF2a+pimosah/4q33jmfHQ2a/S1k87ZHCLbB18baOac/MGTEhfyrlTbTs+8Y49+HmLqH1ktHQ+d3SKWgWrsSdcFpA4UeWefOtbBNFCoJwZ1O6XuHBWn0wErC7iVl/LTqK/xZ5/1v8ZmSdeBHmGN+OKmvXVl2Hb2HU+bEXVMRpiML3/+q5c4HU+5sUjhZ109+dX/SDft/t3qdZzJPedzLehV3dRBhl1nX3liSRNn6xiceDoJIM4o1BMMPn+qwO+KfPSoq5uTREx46otlmyQgzySKAQ89DvzA70rFt0/aiK/PDqPjVdyqg9x+pwy+JfMDL/vin4LqzlHsElfdUf7QR4Z/joyzxiTWWpqx4MMD+7mcoacOMRgfsdHmNx7lU+WJ38SsS/FPgXJgq19o1pWhb12aORG7UmOu/FE7fdfYRjYXmk+MFOIz9+SEvqZsTVyOATRL5WcbX7bxaY7FoE3dB2GJS333Kh4NuPqNCmD5iFo9bOksch7vUvdRuDod1U4ZbfeSOIGVQfnhkScca/HTljqxg4sffgQ/rUJ8+Kgl89PlkR/O5/Jf8WybRyag+ev6x0MZ8jcaR8ZuNFHSF3V9Wofqn0nJsVSqDjGTly05MA4o/SL3S30wrqrHQw8w5kqNeU63k+XcyHqne6F45I5C7pkD3dy8ELEw7pzj9dw3591/ICeuEysWneHbdjyBIcEWOwm4Tz/kpQ6TmMf7PN2gzoRATyza7APiMTl85WLQ5nUkPmWrPkgu7/u69957T+BqX6m+pTyZ5DUzJOU0JhB95jE6ye4K/eI1OzwBNIbU48kmMkrte+qN6uSH17H42TMxzD/7kxhf94FVHJ5KsghStK06YkmRW+fCxT4j9grCE0O5bW2yDY+3Z7DPjv1KaWOdRZ69RvRT2/TD+JJ/Fuvka599qfbMPT7p1i3+2nGCs8+OC8HaYhxQxtZvR+LfGNZinZaecTAvOLcYxy5nx/SPT31Y51N6bJ/iPDImfKLH+cK74BinWp7y+OOP/wsFjBjA973vfdP7xHSggaBs2BsNNDYsYE6yisHg893IK664YlptuRIaIPgcLADgV1tkTFL2ebGhr8qNc45ypWWfFCcaePtgzOGzR4h9QMRH38irBX8MAAsJV5zOP/pgMIhZiBN9XvrHdw9f+tKX7hZB+BR0sGd8uNhok33kFTV895MPGCdfX/jFf8YtNn7Yx3XXXXdNFyJ09KGO8Wvf9VFfSdXjAkV87FerBR0uQHfeeee06ZW2E5q4yTebQd/97ndPW0Sq/Zo2ect9Xtrghz5xgbj99tvPe12OeiNq/4zPT7rBv9jFMYSSQ+aneb2QsZGL17/+9dP79jg/6vxEzrpCfFV24h37BM1Cwd0WHemS3PHsLDKuNhwmRxltrqROcPnYZFAE2RWxud2ci6+zFR/fxMA+JCbsMQeL+GpfaizIOUnmCjpbSvqkn75LCgxlUu6yOObyd4h/8ulmVHHM/VyfGBdsWWSZO8SHXWKgw/5AS+JaRx9793Jprw1UXG1Shn5nIw8b8lu/bZoYXZ3YOTH9qaXTuVg8+wblnOBu7EIX8kqOuIMmvyxW3bmJHgexQi2738RgIGA1tm4HVd5Cqy3YBJbBdQGlbwNNLHnaro0J/WpjG3pIMT4og0GhLl9s2of6qlji6S/btZ7+kdX4xN5C0wd2I8zUs5768qRdDMxN5dkXdZFxdDJ1jE8qf46KydimnbGMbGsczg3xRnYXg38xYzKPUvtf2yP+tIh1HWAAcsAEWKIjmw5PXSgxZNBZ12fG2eGp11H1xYVmvbM5hGffKsaIX/XWtO1D9i37RB2ZesrArifkGn9VR9w8OfWJrnJj0D7jSD1tzZFUu9qWnxQd9aQpP6Ruf4xfOoepDTpr9OewTlNmrqSn6atip0/zJa26XXv3344uVIJrwLTxPeJ3QW/ldT62YqzRzxzWPq2xn9NJbPSyTT3b9le9lM35WCPLcUKfdt5d61PZGswni07t+5Ml7n/3OE/8OUlnmZBM+joxD00EeGJ3WPVEq21txNgSn1j554h4Uk/8pTsU/auvPTQnedZTZylu72rSJrF8ciNNPeoZn7KMNbGUJ+3iI3/YiZ36ypJn3bzbXkPxoR9jBYe8wIfad+XmDP5SAWOuiJU6+hn1B7l5SHv5xGXs6FFHJm76SvvkW1+KX7196CimxDrE/xI+cnLl+NZ8Z+5Sdt4WC/di8CPfMQuDw4/K9clmBjPnjw6CcUh86R+8LMaxdpDU7zDgIa8+4C9N0sSzLhaLsP85d+4xP345lmIUP+lcfOSvmxedn8Skbi6k9slY0QEbH6MYeKjDj778SJ7F33FPa5+XsabPrNt/qLGkXB5jxo/XXQ5Tf64+ys2czTFlp+mfsV2zV9F8268Ts4ETmH1aPEEj8U44lQ+hOGby+WTHQPBhfQ5fex5zU/aJjwT59Kz6JA6uAOwzov9Vjk/iZ58ST1DXxp19wp69LixEaU+dwgI/2mdGPJwAvM+KR/Rpjy1tJgHfteQJbMqzL/rKuKxjT3zkKe2V83QQ2T4lY8g6WLbBZp8Q+9VqIR72kbHPji0iaWes7C8jR6MCPv3zAqBf7blA8t3KeqEd4cnXniej2LtIyZcSH69hYqG2IKMQC9tjGH/t1ZEyb4jfRdH4le9LiYFznyeT7mPrsHhy6DvBkI/821/7BiVmnhqDXxdC9ZjX99xzz+4ilfjo8CUo9pnCT9lT/iXCEycCA7F1ELsOdzwcswDUK2mn2/EI9ZD48M9AOAn0AS4HSWafUf3upAljEvE+sMsvv3xa8MTRnkG46qqrphffYQOfoj0nId8VZKFRZgxQ/PM+sAcffHD350fKwSF/+lUmPoMM/kte8pLpRGBiKkOXj8peeeWV04kG3xis89LBH/7wh9PLA5XpAwoe+YOeRmFycwGpk1xffFfy+uuvP/ePf/xj6leNUfvKt39+1zG3aoiNDu8Lu+aaa3afbhNHe/bn/eIXv9h99xI+hYsfY8IFhvGr76TTngXoW9/61vS6G/1C8UNOfR9Y3UeGjL69+tWvPved73xnmgNpf4w6Md53333nrrvuuvNeDKr/N73pTVP8a/43TsZk/373u9+du+GGG6ZFHJ75VZd1odtiRWz0/41vfOP0vjkutsRk/qc7sQTjlvw0i47xaX2NP/W9i1lj0+noU7zsO/r8ycJiQpKUaYNc3hx2laUNdwO8s4mJj48sefFIG3XgcbWuhfiQMblGdtmHap9t42PSdDbGjJ9OnljWU7fW1QELbBbJWrBBxkdtuYvhjgF9+LV0vNRhfo/ezModrCVx9DXqr3xsuMi610oM5OSTucX8rX8NIGMR1L92XSyc5NjrU51DKP5ZQHKnfsZgHZ/4Z45s8Z/4ximmbSh3oOYu+Zm/5FvfPZ2UcdrUzkvX+kv9rK+1V0/bpCYUnnc51OVry4nkSSxvRNNeX+AxoLaxpQ6Polx+9T8pzfzTxZe+ZkwnUfqHMWc7J6t+UndU1yblxEM785BybdZScMQSGzzHJOXV7xof5l8f2hizc6diGwtxWMdWHPWViwsVO3m1LqZ4yqut+PLV3+K/+tIWn4mffOMZUeNhobWeuuc9nUSog84gjS9Gfd+YanIzdgcLHommwEu+vEm45z/EbvxZB44JXnmdmxqTOl28ypLqP3lZNwZpykZ1Y8LG+kh3Cz9PeuPJfm71JQa0yzf8Q0vGJ1bOKWNAL+voLvlXXz36sCYH6plP48oYjuU/fRmb8Ur131FtUiZPmjLq052YQqgdpb7GaQU8q237Iq1xmoORvOrv09YHtvqBl8c+uFtsMoZqZ0yVv9ROu6wv2SnPmDzh98ERbw0VX3/YZBxrMKqO41j5XVv/nWwtL/1RBzNxu3r2MeVrfaqHbfpPfq0f4kesOXri/07izEHtApwDerLIXKQzXvsKRU4eUs9BgEd+0OM3LYv28NDl8M9SdLRPHjbyobQpnX/9QB2f5G2pEwP+8JM+wYUP5aAvytfiY88xV7r4009iwK94xG2O0m7OJxj4xc7SYSPr8OGhn/bodhjk1wdX9hU75waU336yX/DUEdc4R1R7Y7ANlkWeOvDhZRse7X2LWODSX/smH2rujQ0ex7HKeX9O5ol2LCdnDYekOpjEZkKh/DAL7R5z88Msj+ex5QlJLfwwyo/z2OeP9OrlPiswjAF9T5RD9sHpRwquB77ot/hOKHThU9ie4JOfesKKuUT11+kRw9L8wp4izToxE7/xdj7meNU2848d+N0+Nf1hj46xmSPb2te5oz1zgjniFg9jddFznxtxUcRVLyky9NRFRjzGlLpZ1w7KgX5ipO6WOlhLY0v/9AXF5hjlxD4xks0+FR7x0rljOTlGoIdgmDiSzGNunoDRN/lg02YysY+Hpy/dZODp0cMPPzy96aOLhxOAz6nlYKUeT3Z4OmUxv/+XYY0AACAASURBVMaBfz6bxVs25KkLZZHhfW34WVPASBxOoMsuu2x6pU/yxSIv4PMUcEsBi76Az/YRKEUf9vOxxx6bXpfDxaAr5J1tEF4gsPdAn7yyzcAtCB3GiEcMvEpIbPSMyzjBf81rXtM+IUOXT91hrz68jI+xxZ4nyGIbD+1LLrnkxHchU8Z8Yw+VF0FlI2oMymmztSP3qaWMc5unmoyPcxsb4qqxareGigEmT/XxXzFpc+6xx4+FnnKIzxrXbosFoCTw05/+9O67j8d0VB1fyLaJZpKyz4b3FjGoJN6rJPEwCT/3uc+duAvTFjl7mHjfFS/u6wqThO8mvvjFL54GCVsLuaSND/Mq1Qcbgb/yla/sbsnTljqvqGEf25///OfzYldXSt+crPLYp/a1r31t96eNfONg/xXfVWQ/1j6F952xz4wNiUzW7D+xPPLII9N3LfkAs33Gj3UWGex5bxgxGb8UPvuscsxqnPZFTGOAz4nEHLDAQw4fzOc973nTdxfFUA8Kj4sM45eY8IkPe/rN/Oriw4Z5w/xxH1niI+cOjr8EthR8Y0dszIv3vve9090qePaDOjFxgfze97632+snX70tfrHBnoOxJofsM7vxxhvbhRg91hcuYPqDZ32L76p74jcxhDgiyQ5sNaCtc+hZKSa1S4onwVys9oW7gSz2FR6Dxd0Qe1kyP9ryZwJ3ISxmTCz8GpeY8mgnNnWOOf/cydg/qbhzVD/QXESxAYeDuFikmWSMP+3RyVh9gYuu+6+MrVLsuIsUX7n29C8L8oydmFyE4GdRV57YtqHaKBNbGfjkJ0vaUNdGvm1sOnv45IZFhrs4/iT1Tq2L2dgyhrl61RcfG2XE6NxNLOQc9iVlS/W0sQ4W/nOeVhz96bvK92mf+E2MYBiIpUIAFOmS/oWSL8VD/0y48dPWrtaNe8RXnhQs9F0AtCWv1vWXdtSRWzodePLFUr/S1FWmLbEQH4d6+l6Dq654tqH2E5/y9Y+MoytgVdu01xcUvm2xaHMkRqeDXD5UH7UOrrLqg7YY6tnO2NLeulRMqfa211LwxKRvHLWkXH37S3tf3/gRm3rmPmPQl36kqXNIfVrEuo7g6NjODgn0EFsXlOyP9ey7A4LMyZ56xGAbat3YaIvhhLIN5UgdZeImnvpQ4s+2/irFXoyso6evxHHCg+8ENFdQsTo/yZvTTT3qxiVN+cgfOsSN3DjTTlwocvX0oa04tpXLp20dKpb6ytBLnnaTceRafXQp4FH0m3ZZn5T2/Eds/YwoMRnXWlc1RtoVQ//OI7Grrfxj0N2O/RrMMcAvFgZ9McEmtfZPnYzRRKdMO/FS37p2tqHYaSv1ZKftCaINPHGoZ1udNRQ7ihjaZPz68aRSX919qbj72o/sRn1KfXXgWc8cyENunPCsi6WN/NQRQ5o2qc84V530q92Fpsao39qWP6K1T7QrBu3KG+Edi3/ih/01oDnx1+hfaB0XCvyazJp8ZS4kyjv9bqBqn7Rf4oPvFUpf0s6PuOpgS8zwcxy05Xe6rmAvBr+NUMSmrlxZhyFP/+LBh2dsUHA41BFfHfQ5lBs/PIs2ttWpfOVbqH6xyTo+yKE8aeaq86MelIN+UrSj7ZF9R1cdMbTt/CRPfXhZFz/58MCt8wM7ZMaQ+F1dP1JsresPLI7kJ5YxIJ/TSxv9QLsy/TmJYG1H1ia5c3aheCZHasKq/9pn9dUb2SlfotqLyyA4iEu2nZyHBj7dqXuR0OdHVfn45qDYT2jdo5R++NF8Kb5u/OW5zwyMDocftokffW3SP/HnQmre1KFtX+Qdm5o3/Ugzl9QrX97cGDt+jtEhsetfChYP5XIOiG9OkZl37eyvutCOp9x+qicObevojIqxjOQd35jdw1l1Tvy3oyqsbQaIfVbsY7Iz0qp7IdsmjQSxz4mnXyaUOLKecWmHvPZjZDPiJ26t64fJy+tWeDpFLuWP9KsvBpF9PmyVyKIeTwd9ugZPvn5YZHjMD44ycOw77/Fioo8KJyHbR3yCKob4+OcxP+/NyiI+W0T4XN5zn/vcSaw9DXSe8Yxn7N7kgExc7XmyCT792FrESp9iwONEIXf0j4Ven8aWbe2SgkFemX8uUvpUj+8qvupVrzqxz89+Qnk6zPhor11HExtbCvl/wxvecAIfPnLODfbg8ZQ0CzLt5dtOH8qgmYusq4N92tpmCxHzl7aHeNp2FF3Gh3cJct7UsnoRw5hJfPPNN0/v9XF1FBBHlC54dY5F8ZV+xGWSs0/mt7/97dRZYsxkGSP68sHhELPqiC3t/CpLKj48fbBJ8yMf+ci0oTj9pN1cnfeFffe7350+4JpxiAXlbop+M1609U2bfWA33XTT7ruTiYFf7FzEUiYO+8D4bqf7uPSrLicp392si5h9euYznzntU+Ljul3h7i23UICrb/TZv8Y+pNH7xDrMNTx9PP/5z5/2qbGY2ifskdtX24nLAsHi8NBDD03zj4soRQzxOYG//e1vT4tN2lNHhwvcu971rt1mXu2rbrbRYWyhLMC874562uqfGLkA2Rdp4i3VtZGO9JEbA/GRI97Dx3dn5/4aEFdb8MVyC1P1uftzMo2qEm3kHACxs5urhQ51Iq328tWv8jVtfIvT6df4jTd1q44ycaXwrUP3KdiJIR5tbvm9E6sXgpEfY2AC8icZ+WdSMDks5sc2NPtrnbsYrvhO/KqvXvKtI+NChn8vEOmH/9HAn4vi2z/z4J+b/M8FsOyX+JUi91D22GOPTfhiyl9LtYNaqBNr3uFlfKOciCEFjwsp/YenHXXGi7FnDOl/FnyzwHgHqF3qZB0545c+kMPzIpD61tHnoGhrW505qq62zr8aL+3kZZ3+M4fAEEef2e7q8CjgWad98t5StIZq5MREJU8E5VLk1nFqhxvoVSyxpGmUSZKPnrqVqpO005GXel298w8v+WBxcLeRfOr6yXr6UV+9SjtdbMg5uqmfY5Z21tHVn7ykKRNXX+jJSxvq6CDzsK2euBlfxaJ96DzSj9SYaYOt//Sd9exLYohjfMiUay+1z9mmXvVtqy/t7OAlhrodVU/8Sjub5KGfecp4Uq/Wc+3QRqputqnblqKXddqrF7HqRCAohwHKp21d2wtBc0Cs67e25R9Ku36aD7HVkQ8lHvjGlXXtoPLVl1d1sp062HlyVf/VZk2beDyqPr7ySDl8CrZZurYY6GlH3bxVfuLN1fEFXvU5wkvf4iZGlXfxqYOs8yvuFiomNhlPtjs87aBZ166zkae+/pxT8tXLPs7JxEGHOoc5cpE0LnX0kXTzIpbGBpGOdJYdTL2031oHW1x9JgZ+LOhajMn2adD0nfgZR/LNScqz3ukmj7r6I9+pU22P2V4Th/6MFWpde+OtfOVSsdZS8LCtVPt9cMUzVrGkzFNOSErWaWszomLM0YzZfnXYiWHMVc84UndN3Rgq1TbjkgdFP2NRJk7GY12qbtKDFjGBDIo2f/tnsSNzQaT+XB0/TAgx53RT5xi+9cWfg+DlYmodmVdi8uCgYEu95kbMpNk/44bqE3wOsPSrH/W0S74+xIcqVwb1xEuedfuAb+NIm6X+aZ+22MM3ZjCs69f+QonbMTB+7W1jL08MecZYfRgH8iUdbYmHYsw1PnNszonbAoY4UmWV2i8ovvw9Wjv82l941isObWVQ62ASm/2uduhpi28O+4Y/7PhNL/uvDfKqax6MG2z0UrfGMNc+yiKGc4Oee/IwF8hWGT4vdCHp7lVhIC3wKfzo7Kta/OiDOlB+dGWw54pYnQ4/mvOjKJMCP11hHDicFOiASRvK00fkownbYcpj8rENAepEVAalfx1fHfquzkhvxAeDvpODfWI3BqhzVZ5zyfi2zmHH1H1wOTf0AWXuOL6OURdP2lAnPvSJi/yNxr7arW0bP9jmIm31j9/Ov/bExznAA4wsjhd54byYy6/5wWfWE6/W58+oqj1oOylwymNi9gNxotHukjKAWWQzwfkcmW9LOCb2ovNz56bks8+HrQ5eXdKOAeJ9SrzlwpyknKdW9fG7cvrC4NK/+vROLBYQ8vv3v/99wq/9Z5Jhz0TTBnz1WAR43xWvvIGXOuixQLAPiydsWdRjgv7+97+ftmjAE9c6Wx+6LRpiIeOzXf/7v//bxq9eR/HBFhW+S8nT3TXFuKQsIt0+M+QUcH/zm9/stkCYIyk61jvKK4bIr/kT13OBLRy5ACkHa66ox7y6++67p/E1ljm7KgPHuFMGn3OL3LgA6xM965zX+M8+iAdlCwz71MDQxr7Rfs5znjPlt16o0GGt4FN6vu8MffjaZ7y1/pTHH398+vQkRqyg7PP58Y9/vFuENPDqz/uI3va2t01XQ4PBliCY5NgzUbuTXKx9KYsE+2z4/h2J8gqAfw4m4Tve8Y6df2KimAgWvx/96EfnXve61+3ix075UlzoMkG9slR9Jhnv42LDYoeJvXdC6Rdd2k9/+tOn+F74whdO7YrBScx3Dfk+YldYXHkfF99HJD+MgQUsx8i8KDMWFsd3vvOdu31s8LOAwSNyxz1l1MElP9iJCd+69hlXxRi1weClhMw/9qttKfrnhZbMD/azEQt8C+1D4gOHD8vefvvtJxZB8aH0m/ylb3jkDXrvvfdO5xaLBSXj0350tzQZHPgPc4ZzuPoVlnEnfgp9yILNa1/72nPf/OY3d4ucOsios3/z/e9//4mtLGKRAxZA9kH614b26aerH3wnVjvMCc5iwgJTT5YugLW82qHaXoNTY11jo44D0f2ZiIyDOxn3gTHgnb+Olz64G+NOjDx6smtDXsHnjq7Dd7c+eOQncwQGbSehmOgqyyusMSVFj0UqcatcvKRZZxEf2SdWreObfVxMcO80zU/VtW2/XCTyzxhktcDbJz76w3ixCBgf2KN+pm/q2a4xZZt+EN9plrlY6GPn3/4j5/zwHIHPQdysB8xZFknOk1wbGEdyh30WYhnlMPUOXsRwYscNGgfwCDTl6XhL3ckKlvUt9upiv2/RVjrCSXkOVKdv3lJmzsBJLHWSJz45oQ41P9VeO3xyqAeuPql3Melb+ZJO6mddO2nK5urEio19qHTOVl0x1IWfcYzq6s9RTs4s5lbfxp4+kClP26W6Y76kN5Lb7+o7YxvZwu/81/6nffrRB9Qj5dRrO7FG9YMXsRFwBjzSWcs/JtZanyO9THLVQbZPrGkjPtS6fkb4aa8uPPnypHlC6yepemeF2g+ocRIb9aVS9cVastsiT8wuJvPt4iZ27Y/806TGKj2Gr8TKfNtfc2IeWAi1gXYL45a4Tm0R2xLEf3WPn4GcTB06cidSJ/935XlCnWb/9CElz+ZbHv6zfprxXAxs+maf9e98q3zl+9KjL2IGum9AF8qOODNW6iS3/l1e4+Fq4iBAtxavTmknDrflXqWIw/ikHS9xttb1m3b2z6tmyqgvXTW7/iXGkn3qdnVyYB7MS+r5p00nS71R/Rjxp++aY2KvBR6/15Eb/BtD4mizlD9t1a90yb7q13aHD49Y6avjQ9344aGjb/kVG7n24qUOmOlL2dEXsTpoOjqLNBNt3J4ES/Gqv6RX5Q5k5dPmR09+XAc7f4RWlx/ufSKbsSvfSp1MThgmkPvgqO9T5vq3D17aMDb8qG8OUlbrS/mx79XuGPE7NzrazS95jD1zYN/c05djxF9zku0OXx4PnfhRvz4gsn88sCIndWxc4HwoQnuumFd1jr6ICfxkoDmRqTN5HnzwwWmflyt+9oOFhXca8TUjT/yUL9U7+4yBJ4dsz2CrQ1fYwsGTSUradbpreTkhmIB8zo5Pl9WJBh5Plf70pz/t9kFVH0xC3hfmE9Aq58km9uDsU1hgf/3rX0/vs6v25IOT5GUve9n0hJD2XI66/nGRwL6ehPri6TDfTsy3XSirVN/6gTJ2c/Zs/SD/XX6wZ/yx50leV5iXzM+6yBML85nvQua3HzuMOR7vA2P7DljEUwv7wNhH1vlnIfvjH//YLtAugv/85z/P3XXXXdNTzMSnjg7v0WMfJO2UH30Rc/BqB89a2ySYIOJjI+rHP/7xadOf8oybk+SOO+6YPuBKUr3CpE5XF4tJ8KUvfWna0Nflif1B11577W6fVsXCJ1dqSmdf9ZfaxmU/2Kf21a9+dXg156O67LPiI6/YGoN13sP19a9//dyznvWs1jXvGXv7298+fJ9ZaxR9xf4DH/jAdBJVXWJhH9lPf/rTc7y3jAsS/bKPVT/bnJTkls2ofNeSl352hX7Tf/Jgnzs9YgHPk925wkbo66+/frfhOW2xYYHgfWAsRrXgjwvs1VdfPS1GyM2/8XOCs88qt9qoh879998/va9stOG6+rQtPh+WZp+mWyiUQ4mPfWB8V7Mu8sSJ3H1oaUed/FDYJ8c+SAr6WcBgfyjf9eQiY27ROfoiVp1nIGep7gQwwbaZ/FzxOQHkGTfJ26d/iQMufxLhh4FAxgEuV3oGmr04yvQNVTd5h9T1KyWGboJ6QnInie6oIOMqTP+0Qdc6sjn7Ea58cEb71MAld/RBH3NjpY7YxskYeyepPX4Zt4y/sxcLOw944kA5weteK/j4IH7uZhmDtNc/MVjSv3Uo8Y/smVMWbWzP0dRdio+7SC60aSN2x1MG5ZxgfGshF8jIA/Vajr6IVQdntW0yoCSHBFPPgYZPSV3rW/qlDT70Ay/5tqEOtv63+Nqiq39p9jVxjC15XT37V7GyX53tWp4+Ul9sx854zWPqWteGdupRR0ZJmvry007cOYpd4mhfL5jq6cdY1K9tMau8s1dHm7l4lanrfKTtkTrJU1f5WtrZuXiD35X/2EXMZDCoHDVBDrZ60I6X8jV1/eSAW4fqQ7oGs9PBfl8M44BSMr7OV/LUTR6LS8dPnWPUs7/6S176GPG1Q5d6RxNna73zmzz9S7f6x458g3ns+I0pacZHHb/Zn6356fSX8P6jFzEGo678SwnrknwWeU60fWPD/tBCLo+Bs08c+4zjPjb7xHbaNvYD6t3pafu8mPj/0YsYg8xJBnXgvWvIwfdETN6WQRMbG/+2h+qXhZQ6vM7/ki/7AAWLg99XMt61fTAWfGojz/iM0X7p39t++NrW2LHt7NHDDl/7FnNojsEijqT4zjjxRRsb+cZiH+yndN/4xD3E3vwlFjxig2ZxXOTRtv/o25/s5yH5109H9SHtdODt4//oi5iJGQV5lvgk1MGkTgLdJ8UiUAs/yLIXZk3JPDhwTCCeHEHrhAOTH2T5YRTbzv+SX2x9cgMdlbmJkiey9vKIjx9tR/bkjsXAvEKtg4UMnZG9/val/PCee+kqDj9KkyPiyGI8xo/MMct68tJ+S/0QDOYec3AufvtiTOmPvvPD+T5zS7xDqOfaIRid7dEXsUxa5/As8oyZp2+8pof9KCwyJF0ZcfPUjW0SlOR3fULuQiblyRSvKWI/EJMtdagzwXjMzlYFfdQYqi+x4RMf+7D+9re/VbWpzUnMY3L0ElcMJjmP8TmZu/6xz+myyy6btiIAWHV4HxRP9jps4+MxOd+3zKI+fh944IF2n1Tqj+o83f3lL385ve+t0+FVRrwuJp8O4puxZjz4FB05Mh9i2JbK34fug2GeedXSFVdcceIJHnjIiZ9XAbGQW6ovxgd7xlmZtlyoyA/jXxdJ8Q6h+CN+9uHh0z6JyRjwPjreaVYXYnVG9P/3eKTxb8bP5DmQTmJO7ltuuWU4iNhyN0JJnC5FKaeOLzYrfuITn5j+bNF3YjHI7EPjA8X7FDYL8t1C3jemz8RhceZdcXyfMP/cJBZygP2HP/zhc3/9619P2IvF9yLvvPPO6eV2iWsdDO4AXaDh208o+8fYR9ZNUnzwUj32kUEdE7HnqD6I+4Mf/OAUuzFjZ519YD/4wQ+mlzJ2eMYPHjZnpRgLLwy87bbbdjmt8XXx2xdyzj4y9lmZL+1pY8s+LeYPFwN4VU/9rdSx5IWit9566/S/UeyTWLTZyHzddddN21CId63/oy9iax0b/IWmGR+J84QyqdyNeTWTlzF2POWJ3dXhcTcGBvXEos1mWq6I3M1wNWTws4iZdsjpAzzixo47OnDUFyP/jDAGZcYDFvbVB3r8OYMPFnL10566MesbHLHg+WduyvGJHTE7HsrFX0Ox4c9dsNIe/+CSG8aXuy0KfPWMUT+2lcu/mJSYjD3jIMYar+3UG9mTG8a1s0n7Q+vEaf7xxQEP//D3LUdfxE47Eft2tLNz8KGUjD1PhORblyZu8sRMOXUGDD0OdDg8eW3jmxNOPPjWxdNevlQM9KynLtiWtEkedhSp9urU+OB3WNprl23jgIct7ZRXn2IsUTDIMUW8xNaXOsYt1Ub/0iW/W+T60Ka25UuNDWrekGkHz3kFH70atxhplzz1xdT3Vor9CCNjTFz08e+YzGGknfWjL2KjDujwrFDjdPCIK+tdG5522Q95UDCWivrqOXi0tWdiOmHFVQ8dY5Wiqy042KSf2ta3fO3TRp1K8ZPxOTnFyjiwNW59wEMXPQ4XbOSnWfQn1RdtfGffracu9WMUcdKH9UqX/ImlnjkURz568KD2VR0xpPCVaQ+VJ1WWbTA4kqeeMvzrK3XldbZidPToi5iBdM7OEi+TZ1wmL/tAPdvqVoptYma96tJOX9Q90lfVcYJqrw9tpMhZOLBXp4sBHvK0y3pnU+X6gJ91sRMjdVIupv1Nm9Oqp0/8pm9P8tQhjtTZNy4wxRUj29Y7PfU7qh0y4xTD/qmTcmVp1+HD017cbKd8VFdfOb7FSh71LeWgRcxkpENXWf5EyqANuLNJ+65uRyumumJK5c9RdEeTVT729gF96tw1VD+2O1nGYD/EVEYbjNo/9fStH+OqfNqZf++QxEeWRXwpsvSRusroI7j2tcagL/mJgY04+kRPn96xqZO2a+v2PzHsNzT90dancXfUWKVrY+n00r/9lqJPjsxT9UebOYI+dQ5t1a1U3cxHjSttwNeH2OYNPWLjt1Xq2sEzl1DszSu+UqZvsOFnAU//1b7T1/agRcxO2Fkoj8mha/dTGcgayqNxHg9T9EmdOGgbzxos7VJXHLGliUuSR4Wnm/5wjU3ageUxsq/70Kr/xAODtjq0GXjznz/i6y/xMxbqTqjqQ1sofZ/b55a6+9TBNr+1b2vxcvJrI4+nbtaR0W/82Gd9JhVjCzW30K7oT5ltftxmDq39kVs/xite+lUG1Q966iiHx0Mn5k+dO+YMOQ91nOP680EYtNuH5qLMFh2wjNs4jIVzmzlqW3z9+9RUvvSgRaw640Tgu3s8QWEyVrlO96Xgsk2A0g3IVn/qO5CJaV0ZPlmYeZ0JWyWUwxeHCVA/t4VMXQb/0ksvnV61wsDI18dTn/rU3ee+pk4+8U9iVL5tdMgP+7Be8IIXTNjGpQ573HzNi76VVV35UGVM0F/96lfn2GoBz4VPedpsqWNPPHy3kUmcPrfgsADwXcr6BM/8ssWj2yKjfyk+a362xKE9GPrWvsM1f8ydn//859NCJoZ26DB+zB8Wi1Gs4ksrDm39ia0u59ZVV121u1FQLuUVR7zvy0VLPnjMBbbGXHnllbs7SeXgs5Dxrjls9WcfbPMpPj4HWRdR5PDYouFFTmzoQYsYACaETjBBPvOZz5zXiXR4SJ3OOEHzBDKGLdhgcWBrMqXiILcg4z1M7PPyu5LwstDmaoKdVw/l8JiEn//854fvE0OHhS4XOO07ir4xQ3kP1he+8IXzfGuLPvlD1/zBW1vYjPjRj350NwnX2hojfqhnsQ/Kujvt1O/q9IWc8T6xb3zjG7P7wLzL0W/2Ieudn7U8cDywsc8jfMeb95XdeOONu/zqDzsweOEh+/yYR2JVio087SsVT775Y6M17wszXuVQbPhwMu9DYxtLLdjwQkfsPUerDgsQ81v/+vGGh43UvE9NftrDYwHEnnr2ce9FTCAD0iGTpK7Uyg6l+DJ4KZhZ10fHU1YpuvYHSkle1R/tw6p6tS02iz1XFheS1DNuKbKsp25X9yRFpj/7kli1vx1W5WHDnxRLxXj1P6e/RXeEox8oJ4QLNfqJTz3bI7xD+caj/zU+seHo8os9dzL1wgg+Nsj1aV2fo76oZ4zisEjUgl/OaeYrf43UCw185rN6zEHwMwbxE1selAOczr824knl772IJRB1AwI4ZTo6Fh1h4z+L8VR+6mRdXCmyrNuWB243qRJzVAfDQ1zj1Qa5vKSdvrrKbCe1ro5+wOawZF1e0jV9xhdlCQud1LWe/tbU7Zv+bEPFVKZPddbgr9XBh0fa6NtYUkbdWKBdfr3YQSuGbal46QP/yDOO5FVbZWJkm3oe6kjBykM+VD/Y2648+pj+0n5U33sRsyMCE8ycc2XQQ8rIXr6UeLoJcYhvbMXfF8dBw96cUa+Dhx/lSfWbOGIpM0Z01BMPHeXQPCqOeFto+ttidyxd+wz15Afb31Lsr3Eey2+HY547WfLUk6Ys60vy1M26fZWCQ72j2KmXGJU/igXbPMQQUzt1kBuHOlJtK1Vf/uZFzCCg1gHDsW3rGQwy+TrfSsHIiak9uPqS1pjUPSvUOI295gd+8mxnv8SAh27mh3oW7ZOvTfK0Ud/2WipWxoat/LU4h+jZr4phDDW2qrdvG1yPrRjYEZ90q/0WfXxQpOnXeuKZN6m2tsVJm1oXt9OVJ161pV3tbSNbvYjpgL/NqUN1DtCau541Ol0H5OHXOORB5Rubsqpb2+pdTGoOKyUmeMZsH5Nn3OrQJgcUeOpCraubbcYFOygXCQ/xky6NIbZgU6T6hJf1qoP+Ev4EXP4RE0o/7Ev6A9t4ivnqJvbc0Xkh1S9t6lL5nT90qr0BYNf1Xzyo/dvSH+zQh1rXZ/I7OXrEpMz4aVvICTrQ5CuHp5/sn3xl6MvTVirf3GFjWb2IYcTht+N8VC3QWaDsM/KHweyksXU8ZWeRMnAWBz/7oByeg+yfjRWUJAAABj1JREFUTdptofwgzpMnfOlvrb0xbLVbiz+np29+UGYO5MONakeezJX5qzq1rZ77qFhIstjnug/NuKp93UKQWF1dHOb2ln1kHdY+PB/U4X9pHxh95TBmqHUXwH1iSBvwsqxexAiMwbvnnnumpxPUXRUT8LTqmRh8mBgpPE5AXidDcWJNjSf+AeOsFfs1isv+5cAlD3v7xVOj++67b/q+IWMjX9vqy7Z67HO7/PLLp/dSZTzqMYF5XYt7udQRn7dvsE+LhYT8wzdWdNUTT3vb+AefcdxSjJ/NlOyzYguCmOLQ5gJM/zgZtVE+R437aU972rSPCj+10F+2eHgRRY6P9MOrltgHRT+zoMN48V1Ixq9uFBeD97D95Cc/afeRJV6tY5/joFye/ZNfqfZ8Mu+tb33rbh9X2tF/tmhwEZUv1Z5zk29Pomuf8IWeOtW3bWz4pCD75NAXe7J//PHH/wUAB8l7z3veM+1FIakY1sIgjW4bq+7FaHO1tD/6t8Ns9GSfDS8+ZBG2H8rVr1Q8JhmbAZloo/xUW9ris5nwZz/72fRiQq7G3jUp72zxPSfHhnFC59FHH53ie/jhh6c2tvoXR17ni82G5Oe5z31uJ56+t8hJ+Je//KXF5ySmf89+9rNb+yUmL3PkJGHTJP2Zi7XDwob5ab7EkPJRYOK75JJLJnPGcEshHuZXd16AA553LcYgvn3hHLOuDIo+LwTkfWq8nJBS9dAZ7cFKrGPXzR8vlGQf1+ivMOZzzmlzwLlGXliAb7jhhmkRrH2bixkccv7mN7/53O233z6NMbkWf/WdmE4YhK23w9peCLolORcinpEPB2Akh09f0Kt9kpcUfdvckTHII/vqUzsmG3VOFBfGjAPZ6AROPRYS9NYuEqk7h1/jrm36y/y0P8ozbupbi3nEzkVqhAF+5wMeOP6pm5jkHT48jw4fWbePrNM9Jo/YiRH/LFLE2vVRn8g6OfOB8fVGQ/0lqv8R7uZFbC7JS8H8V34yAwwK+RwVJ4I09eQlhjzHiAmT8rTPujrYUXcBdBFykRE/bbOuX+3RFztt9YOtdW0Sb5+6MUAtGUPGpHyJGjvUnIxs1O3kyDIu+y7VRozUVXbIIi/GVpp9JrY8wMr4rdtX+4KePHS29EP/iZV92LyIpfF/62c/A92JMBe1EwVa67bn7Jlw2lLPSa0d8uRnXZ196Vx/52Rr/XU5MH7okg/7jr96cnbYa+O6UHrEmAd+M27r5kF6mvH9dxE7zez+B2M7maGe3PJoy4fKlz7Z0mbc0qX40as5WbI5y/IcT+O0j2tzot0+9LxFzCupP9DtA3rWbEwkfbK+b4xibMmPPs3tvr7X2Olja3xMxDU24OsDGwr9G9mnjvGrb16S3+Er35fihz9fHLt9cY5pV/sONjxiJAcUc3dMv/tgEROxGFfFsC/SKrft2K6ZZ9pAl/z/D8Hl4Tt7+IH0363waNsfRveZIJwIh+Sn+t8nhm5MwGEC8eMrPmjvM37Y+tCmxkYbGTqj3zOQEUOW0cTu+MQ8h5+4+9TZGuEP1Njbxy6WffC32OAbv1DjYG4yv2oOt+Cehq7jne8DW5OzqsMDJ7bniLc2VvU996rd/6QjVjxep8F+H1ZNk1uNnmxt+8ij4e59ZGv7w1M3HvPyfcIt+dE/+6fYK0SBx3GMHIvPuPE+p0ceeWQTtnEQG+80M76pEr95EP9b3vKWaauFNurSD75ryF6sfQu+2cLBFoPET7wRP3VqHRtOBN4nRo6yILvYxRh4ldLVV1+924d3jLlxjL6ZPz71t/RkMv0RP7b2jy0811xzzd5bLNgj1t3FPWXaJBaeuRqetStBhHdQlWTmPqJ9wLhaemXYan8M/3M+mTSM377xsTDPTdIl/EP7t4Q/1/c1sqX+rcE4TR36v3X7wWnGU7FZQJgf+xbmJfOTfu5TRv6fwmZXAZmEONrXiThnmTKR6Sd9TLoUs/qHLvD7+r8Q8ZEP4qPY3+p3rv/Ye4zsK176wsaj0zsGjxPB2KTHwN2KoW8oxbm47wVoq/999J0fxr4WQ33oIf0b+T/vTsykrg3wyaZHIg4ph+bnUP9LsZ92fEv4h/ZvCX+p/0vyQ+Nbwj9Uftr9PzS+Q/N3aP86//8HBO8uBb51RkwAAAAASUVORK5CYII=)