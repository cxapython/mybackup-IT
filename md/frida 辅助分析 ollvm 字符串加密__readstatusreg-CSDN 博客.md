> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/Qiled/article/details/115235485)

frida 辅助分析 ollvm 字符串加密
----------------------

> 近期逆向任务繁重且艰巨，因此把工作期间得学习心得做下笔记还是十分得有必要得。防止后期遗忘。  
> 个人博客: http://www.zhuoyue360.com/

#### 文章目录

*   [frida 辅助分析 ollvm 字符串加密](#fridaollvm_1)
*   [一. Ollvm 混淆简单版分析 - datadiv](#Ollvm__datadiv_6)
*   *   [1. 如何辨别 OLLVM 字符串混淆?](#1_OLLVM_8)
    *   [2. 初步分析](#2__15)
    *   [3. 编写通用方法打印字符串](#3__110)
*   [二. Ollvm 字符串混淆 - 找不到 datadiv](#_Ollvm__datadiv_161)
*   *   [1. 初步分析](#1__167)
*   [三 Ollvm 字符串混淆 - 再次升级](#_Ollvm___264)

一. Ollvm 混淆简单版分析 - datadiv
--------------------------

> 使用 2.0.0apk 案例

### 1. 如何辨别 OLLVM 字符串混淆?

把 so 文件拖入 IDA. 到导出表`Exports`中，通过`Name`排序可看到`.datadiv_decode123123131`类似字符串，其就是 OLLVM 字符串解密的特征

![](https://img-blog.csdnimg.cn/20210326181219503.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1FpbGVk,size_16,color_FFFFFF,t_70#pic_center)

### 2. 初步分析

此时可以看到这些`byte_XXX`, 这些都是编译过程中把字符串给加密了. 然后再进行异或，进行解密，解密完成以后才进行使用。  
![](https://img-blog.csdnimg.cn/2021032618123385.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1FpbGVk,size_16,color_FFFFFF,t_70#pic_center)

然后我们在 010Editor 这款软件中进行尝试解密。新建一个`十六进制文件`  
具体步骤:

1.  byte_37118 ^= 0xD2u; `byte_37118 DCB 0xF9`
2.  byte_37119 ^= 0xD2u; `byte_37119 DCB 0xD2`
3.  他们 2 都是异或 0xD2, 在 010Editor 的工具, 十六进制计算中存在异或选项. `无符号字节,十六进制`
4.  此时就可以拿到解密以后的结果 `+.`

其他的也就类似了.

```
  byte_3711F ^= 0xEDu;
  byte_3711C ^= 0xEDu;
  byte_3711D ^= 0xEDu;
  byte_3711E ^= 0xEDu;
  byte_37120 ^= 0xEDu;
  byte_37121 ^= 0xEDu;
  byte_37122 ^= 0xEDu;
  // 异或0xED以后的结果:
  // salt0+

```

这种手动计算其实是比较麻烦的, 所以需要找到计算函数. 去`JNI_OnLoad`函数中进行操作.(这边我已经对函数进行的名称的重新及类型的重新定义)  
这里主要看到`RegisterNatives`, 它的第三个参数`v7`是一个函数的数组, 然后 v7 来自于`&off_33D60`

```
jint JNI_OnLoad(JavaVM *vm, void *reserved)
{
  jint v2; // w19
  JNIEnv *env1; // x20
  __int64 v5; // x0
  JNIEnv *env; // [xsp+8h] [xbp-48h] BYREF
  __int128 v7; // [xsp+10h] [xbp-40h] BYREF
  __int64 (__fastcall *v8)(); // [xsp+20h] [xbp-30h]
  __int64 v9; // [xsp+28h] [xbp-28h]

  v9 = *(_ReadStatusReg(ARM64_SYSREG(3, 3, 13, 0, 2)) + 40);
  env = 0LL;                                    // env
  v2 = 65540;
  if ( (*vm)->GetEnv(vm, &env, 65540LL)
    || (env1 = env, v8 = sub_E76C, v7 = *&off_33D60, (v5 = (*env)->FindClass(env, &xmmword_37050)) == 0)
    || (((*env1)->RegisterNatives)(env1, v5, &v7, 1LL) & 0x80000000) != 0 )
  {
    v2 = -1;
  }
  return v2;
}

```

> — DCQ（DCQU）用于分配一片以 8 字节为单位的连续的存储单元并用指定的数据初始化。

我们双击点进去, 按`D`键把他装换成`DCQ`的指针 , 我们静态看, 看不出什么东西. 用 frida 去看下它内容是什么样子的

```
.data.rel.ro:0000000000033D60                 AREA .data.rel.ro, DATA, ALIGN=3
.data.rel.ro:0000000000033D60                 ; ORG 0x33D60
.data.rel.ro:0000000000033D60 off_33D60       DCQ aYcmd               ; DATA XREF: LOAD:0000000000000EE8↑o
.data.rel.ro:0000000000033D60                                         ; .text:000000000000EA80↑o ...
.data.rel.ro:0000000000033D60                                         ; "ycmd;\n"        // 这边我们双击进去可以看到真实地址.

```

**Frida 代码**

```
function hook_native(){
    //先找到基址
    var base_hello_jni = Module.findBaseAddress("libhello-jni.so");
    if(base_hello_jni){
        // base_hello_jni是一个指针, 指针需要使用.add方法, 直接用运算符是会出错的
        var addr_37070 = base_hello_jni.add(0x37070);
        console.log("addr_37070",addr_37070.readCString());
    }
}


function main(){
    hook_native()
}

setImmediate(main)



// 返回结果 
[Pixel 2::com.example.hellojni_sign2]-> hook_native()
addr_37070 sign1


```

**总结: OLLVM 默认的字符串混淆, 静态的时候没法看见字符串, 执行起来之后, 先调用`.init_array`里面的函数来解密字符串, 解密以后, 内存中的字符串就是明文状态了.**

### 3. 编写通用方法打印字符串

看到`sub_E76C`函数, 看到其中, 有如下代码  
通过上面的分析, 我们可以猜到, 这种`byte_xxx`是 ollvm 加密的字符串, 我们可以去 frida 中编写个字符串打印算法

```
  sprintf(s, &byte_37040, v19);
  sprintf(&s[2], &byte_37040, BYTE1(v19));
  sprintf(&s[4], &byte_37040, BYTE2(v19));
  sprintf(&s[6], &byte_37040, BYTE3(v19));
  sprintf(&s[8], &byte_37040, BYTE4(v19));
  sprintf((s | 0xA), &byte_37040, BYTE5(v19));
  sprintf((s | 0xC), &byte_37040, BYTE6(v19));
  sprintf((s | 0xE), &byte_37040, HIBYTE(v19));
  sprintf(&v18, &byte_37040, v20);
  sprintf(&v18 + 2, &byte_37040, BYTE1(v20));
  sprintf(&v18 + 4, &byte_37040, BYTE2(v20));
  sprintf(&v18 + 6, &byte_37040, BYTE3(v20));
  sprintf(&v18 + 8, &byte_37040, BYTE4(v20));
  sprintf(&v18 + 10, &byte_37040, BYTE5(v20));
  sprintf(&v18 + 12, &byte_37040, BYTE6(v20));
  sprintf(&v18 + 14, &byte_37040, HIBYTE(v20));

```

**Frida 代码:**

```
function print_str(addr){
    var base_hello_jni = Module.findBaseAddress("libhello-jni.so");
    if(base_hello_jni){
        // base_hello_jni是一个指针, 指针需要使用.add方法, 直接用运算符是会出错的
        var addr_str = base_hello_jni.add(addr);
        console.log("addr ",addr_str,"  add_str",ptr(addr_str).readCString());
    }

}
// 测试一下
[Pixel 2::com.example.hellojni_sign2]-> print_str(0x37040)
addr  0x79ce0fd040   add_str %02x
[Pixel 2::com.example.hellojni_sign2]-> print_str(0x3712C)
addr  0x79ce0fd12c   add_str salt2+
[Pixel 2::com.example.hellojni_sign2]-> print_str(0x37124)
addr  0x79ce0fd124   add_str salt1+
[Pixel 2::com.example.hellojni_sign2]-> print_str(0x3711C)
addr  0x79ce0fd11c   add_str salt0+
[Pixel 2::com.example.hellojni_sign2]-> print_str(0x37192)
addr  0x79ce0fd192   add_str e

```

看到上面得返回结果，可以看到，只要我们愿意，所有得字符串都是可以解密出来的

二. Ollvm 字符串混淆 - 找不到 datadiv
----------------------------

> 这种情况只有在 64 位才有出现

> 2.0.1 apk

这个案例 APK 我们是找不到 datadiv 的，那么我们该怎么下手呢？

### 1. 初步分析

从`.init_array`下手. 到`IDA View-A`的窗口，按下`Ctrl+s`，进到`.init_array`  
![](https://img-blog.csdnimg.cn/20210326181249156.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1FpbGVk,size_16,color_FFFFFF,t_70#pic_center)

我们点进去看看.

```
  byte_37030 ^= 0xC6u;
  byte_37033 ^= 0xC6u;
  byte_37034 ^= 0xC6u;
  byte_37035 ^= 0xC6u;
  byte_37036 ^= 0xC6u;
  byte_37037 ^= 0xC6u;
  v0.n128_u64[0] = 0xC6C6C6C6C6C6C6C6LL;
  v0.n128_u64[1] = 0xC6C6C6C6C6C6C6C6LL;
  stru_37010[0] = veorq_s8(stru_37010[0], v0);
  stru_37010[1] = veorq_s8(stru_37010[1], v0);
  byte_37031 ^= 0xC6u;
  byte_37032 ^= 0xC6u;
  byte_37038 ^= 0xC6u;
  byte_37039 ^= 0xC6u;
  byte_3703A ^= 0xC6u;
  byte_3703B ^= 0xC6u;
  byte_3703C ^= 0xC6u;
  byte_3703D ^= 0xC6u;
  byte_3703E ^= 0xC6u;
  byte_37040 ^= 0x94u;
  byte_37041 ^= 0x94u;
  byte_37042 ^= 0x94u;
  byte_37043 ^= 0x94u;
  byte_37044 ^= 0x94u;

```

我们看看其汇编代码. 得知 V0 是 0xC6

```
MOVI            V0.16B, #0xC6
ADRL            X9, xmmword_37050
EOR             V5.16B, V5.16B, V0.16B
EOR             V6.16B, V6.16B, V0.16B

```

再看看`stru_37010[0]`的值  
看到其 IDA 加载地址是 037010, 文件地址是 36010  
![](https://img-blog.csdnimg.cn/20210326181257787.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1FpbGVk,size_16,color_FFFFFF,t_70#pic_center)

然后有 010editor 拖入进去, 跳转到 0x36010, 提取出 16 个字符. 在进行一个异或`0xC6`.  
得到结果.  
`Hello from JNI !`

看到下一个函数, 其实也差不多. 回到`JNI_Onload`

```
.data.rel.ro:0000000000033D60                 ; ORG 0x33D60
.data.rel.ro:0000000000033D60 off_33D60       DCQ str_sign1           ; DATA XREF: LOAD:0000000000000EE8↑o
.data.rel.ro:0000000000033D60                                         ; .text:000000000000EA80↑o ...
.data.rel.ro:0000000000033D60                                         ; str_sign1
.data.rel.ro:0000000000033D68                 DCQ stru_37080

```

按下 X, 查看交叉应用这个`str_sign1`. 然后进去. 然后通过下面的代码解密, 然后现在这个`str_sign1`只有一个字符, 我们需要把它变成 6 个字符

```
  str_sign1 ^= 0xE7u;
  byte_37071 ^= 0xE7u;
  byte_37072 ^= 0xE7u;
  byte_37073 ^= 0xE7u;
  byte_37074 ^= 0xE7u;
  byte_37075 ^= 0xE7u;

```

在 str_sign1 按下右键 -> Array -> Array size 修改成 6

```
ta:0000000000037070 str_sign1       DCB 0x94                ; DATA XREF: std__string___4921590060622252445+178↑o
.data:0000000000037070                                         ; std__string___4921590060622252445+1B4↑o ...
.data:0000000000037071 byte_37071      DCB 0x8E                ; DATA XREF: std__string___4921590060622252445+1E8↑r
.data:0000000000037071                                         ; std__string___4921590060622252445+234↑w
.data:0000000000037072 byte_37072      DCB 0x80                ; DATA XREF: std__string___4921590060622252445+1F0↑r
.data:0000000000037072                                         ; std__string___4921590060622252445+23C↑w
.data:0000000000037073 byte_37073      DCB 0x89                ; DATA XREF: std__string___4921590060622252445+1F8↑r
.data:0000000000037073                                         ; std__string___4921590060622252445+244↑w
.data:0000000000037074 byte_37074      DCB 0xD6                ; DATA XREF: std__string___4921590060622252445+200↑r
.data:0000000000037074                                         ; std__string___4921590060622252445+24C↑w
.data:0000000000037075 byte_37075      DCB 0xE7                ; DATA XREF: std__string___4921590060622252445+204↑r
.data:0000000000037075                                         ; std__string___4921590060622252445+254↑w
.data:0000000000037076                 ALIGN 0x20

```

然后到`Pseudocode`窗口重新 F5, 则可以看到, 此时的代码就比较漂亮了.

```
  str_sign1[0] ^= 0xE7u;
  str_sign1[1] ^= 0xE7u;
  str_sign1[2] ^= 0xE7u;
  str_sign1[3] ^= 0xE7u;
  str_sign1[4] ^= 0xE7u;
  str_sign1[5] ^= 0xE7u;

```

到 010editor 异或下 0xE7.  
得出结果`sign1.`  
其他的还是可以使用之前第一点的`print_str`打印出来的.

三 Ollvm 字符串混淆 - 再次升级
--------------------

> 在. init_array 它可以拥有很多的函数. 反调试, 字符串解密, 检测还有一些其他的解密都可以写在这边.

依然是去. init_array 看. 悲伤的发现它们并不是一个解密字符串的函数了

```
_QWORD *sub_6CEC()
{
  _QWORD *result; // x0

  result = qword_3FEE0;
  qword_3FEE0[0] = 0LL;
  qword_3FEE0[1] = 0LL;
  qword_3FEE0[2] = 0LL;
  qword_3FEE0[3] = 0LL;
  qword_3FF00 = 0LL;
  return result;
}

```

其他函数也一样是找不到解密函数的, 然后我们去找一找 jnionload  
然后把基本的`JavaVM*`改一下,`JNIEnv*`改一下,

```
jint JNI_OnLoad(JavaVM *vm, void *reserved)
{
  jint v2; // w19
  JNIEnv *env_1; // x20
  unsigned __int64 i; // x9
  __int128 v6; // q0
  unsigned __int64 v7; // x9
  __int64 v8; // x0
  JNIEnv *ENV; // [xsp+8h] [xbp-48h] BYREF
  __int128 v10; // [xsp+10h] [xbp-40h] BYREF
  __int64 (__fastcall *v11)(); // [xsp+20h] [xbp-30h]
  __int64 v12; // [xsp+28h] [xbp-28h]

  v12 = *(_QWORD *)(_ReadStatusReg(ARM64_SYSREG(3, 3, 13, 0, 2)) + 40);
  ENV = 0LL;
  v2 = 65540;
  if ( (*vm)->GetEnv(vm, (void **)&ENV, 65540LL) )
    return -1;
  env_1 = ENV;
  strcpy((char *)&dword_3E1B4, "sign1");
  *(_QWORD *)&xmmword_3E1E8 = &dword_3E1B4;
  for ( i = 0LL; i != 39; ++i )
    byte_3E1BA[i] = byte_22E80[i + 280 + -28 * (i / 0x1C)] ^ byte_22E80[i + 308];
  *(_QWORD *)&v6 = byte_3E1BA;
  *((_QWORD *)&v6 + 1) = sub_6E4C;
  *(__int128 *)((char *)&xmmword_3E1E8 + 8) = v6;
  v11 = sub_6E4C;
  v7 = 0LL;
  v10 = xmmword_3E1E8;
  do
  {
    byte_3E195[v7] = byte_22E80[v7 + 150 + -25 * (v7 / 0x19)] ^ byte_22E80[v7 + 175];
    ++v7;
  }
  while ( v7 != 30 );
  v8 = (__int64)(*env_1)->FindClass(env_1, byte_3E195);
  if ( !v8
    || (((__int64 (__fastcall *)(JNIEnv *, __int64, __int128 *, __int64))(*env_1)->RegisterNatives)(
          env_1,
          v8,
          &v10,
          1LL) & 0x80000000) != 0 )
  {
    return -1;
  }
  return v2;
}

```

我们可以看到 V9 的最终来源其实是指向了 `dword_3E1B4`, `dword_3E1B4`的值是`sign1`, 这边使用的 IDA 版本号是`7.5`, 不同 IDA 出来的结果很有可能不一样, 比如在 MAC 上 7.0 的 IDA 出来的结果是 `1ngis`. 所以选择一个好的版本可以减少我们很多奇奇怪怪的`??????`

然后参数是哪来的.

```
*(__int128 *)((char *)&xmmword_3E1E8 + 8) = v6;

```

v6 是哪来的

```
  for ( i = 0LL; i != 39; ++i )
    byte_3E1BA[i] = byte_22E80[i + 280 + -28 * (i / 0x1C)] ^ byte_22E80[i + 308];
  *(_QWORD *)&v6 = byte_3E1BA;

```

然后这个`byte_22E80`是什么? 双击进去, 发现这里是一个表

```
rodata:0000000000022E80 ; Segment type: Pure data
.rodata:0000000000022E80                 AREA .rodata, DATA, READONLY, ALIGN=4
.rodata:0000000000022E80                 ; ORG 0x22E80
.rodata:0000000000022E80 ; _BYTE byte_22E80[352]
.rodata:0000000000022E80 byte_22E80      DCB 0xEA, 0x4C, 0x26, 0xF9, 0x30, 0xAC, 0xE3, 0x96, 0xCD
.rodata:0000000000022E80                                         ; DATA XREF: Java_com_example_hellojni_HelloJni_stringFromJNI+C↑o
.rodata:0000000000022E80                                         ; Java_com_example_hellojni_HelloJni_stringFromJNI+20↑o ...
.rodata:0000000000022E80                 DCB 0x30, 0x45, 0x20, 0x73, 0x56, 0xE5, 0xD2, 0xEC, 0xAA
.rodata:0000000000022E80                 DCB 1, 0x8E, 0x62, 0x60, 0xB2, 0xA7, 0x9B, 0x8E, 0xB7
.rodata:0000000000022E80                 DCB 0x86, 0xF0, 0xCF, 0xF0, 0xD0, 0xD5, 0x26, 0xCA, 0x66
.rodata:0000000000022E80                 DCB 0x99, 0xEB, 0x3C, 0x28, 0xD7, 0xCB, 0xF7, 0xE1, 0x97
.rodata:0000000000022E80                 DCB 0xE0, 0x82, 0xA0, 0x9D, 0xF0, 0x9F, 0x68, 0x83, 0x46
.rodata:0000000000022E80                 DCB 0xB8, 0xCB, 0x1C, 0x23, 0xDD, 0xCA, 0xEB, 0xE7, 0xDB
.rodata:0000000000022E80                 DCB 0xE3, 0x94, 0xEF, 0x87, 0xB9, 0xA1, 0x4E, 0xEA, 0x27
.rodata:0000000000022E80                 DCB 0xDB, 0xA2, 0x1C, 1, 0xC0, 0xCA, 0xAD, 0xBA, 0x9A
.rodata:0000000000022E80                 DCB 0xF0, 0xC8, 0xAE, 0xCA, 0xD0, 1, 0xA8, 0xB0, 0x3E
.rodata:0000000000022E80                 DCB 0xCC, 0xD1, 0xB9, 0x6C, 0x47, 0x8C, 0xA0, 0xAD, 0xEA
.rodata:0000000000022E80                 DCB 0x3F, 0xF3, 0x97, 0x64, 0x22, 0x7F, 0x48, 3, 0x1B
.rodata:0000000000022E80                 DCB 0x10, 0x1C, 0x4E, 0xD6, 0x64, 0x42, 0x4C, 0xE5, 0xD0
.rodata:0000000000022E80                 DCB 0x23, 0xD1, 0xD5, 0x9F, 0x9B, 0x5B, 0xBD, 0x5E, 0x42
.rodata:0000000000022E80                 DCB 0xFB, 0x3B, 2, 7, 0x4F, 0x7A, 0x7B, 0x1B, 0xCF, 0x86
.rodata:0000000000022E80                 DCB 0x63, 0x82, 0xE0, 0x41, 0x3F, 0x7B, 0xD7, 0x97, 0x2F
.rodata:0000000000022E80                 DCB 0xA6, 0xE7, 0x40, 0x91, 0x96, 0xA1, 0x4C, 0x8F, 0xD0
.rodata:0000000000022E80                 DCB 0x23, 0x21, 0x74, 0x6B, 0x81, 0xB7, 0x91, 0x1F, 0x79
.rodata:0000000000022E80                 DCB 0x66, 0x5C, 0xC6, 0x1D, 0x11, 0x7B, 0xA5, 0x11, 0xD8
.rodata:0000000000022E80                 DCB 0xCB, 0xC6, 0xF3, 0xC2, 0x23, 0xE2, 0xFF, 0x46, 0x59
.rodata:0000000000022E80                 DCB 0x15, 6, 0xF1, 0xDB, 0xF4, 0x30, 0x11, 3, 0x30, 0xAA
.rodata:0000000000022E80                 DCB 0x72, 0x7B, 0x15, 0xCC, 0x3E, 0x90, 0xAE, 0xAA, 0x9F
.rodata:0000000000022E80                 DCB 0xCE, 6, 0xE1, 0xB9, 0x23, 0x40, 0x42, 0x5A, 0x18
.rodata:0000000000022E80                 DCB 0x1F, 0x33, 0x9C, 0x47, 0x5D, 0xB0, 0x2D, 0xAD, 0x23
.rodata:0000000000022E80                 DCB 0x6F, 0xCC, 0xEC, 0x84, 0x8E, 0x7A, 5, 0xD1, 0x87
.rodata:0000000000022E80                 DCB 0xFB, 0xA3, 0x97, 0xE0, 0x11, 0xC0, 0x36, 0xAE, 0xDA
.rodata:0000000000022E80                 DCB 0xEB, 0x89, 0x7B, 0x19, 0x17, 0x8E, 0xCE, 0xF4, 0xF4
.rodata:0000000000022E80                 DCB 0x18, 0x8C, 0x31, 0xCD, 0xDD, 0x3C, 0x38, 0x92, 0xD
.rodata:0000000000022E80                 DCB 0x14, 0xA9, 0x82, 0xEE, 0x15, 0x28, 0x17, 0xA3, 0x86
.rodata:0000000000022E80                 DCB 0x28, 0xF7, 0x13, 0x79, 5, 0xAF, 0xCA, 0xE, 0x90, 0x1E
.rodata:0000000000022E80                 DCB 0x97, 0x57, 0x59, 0xCB, 0xBA, 0xB9, 0xED, 0x80, 0xF
.rodata:0000000000022E80                 DCB 0x5F, 0x9C, 0x55, 0x1C, 0xD0, 0x1B, 0x8B, 0x8E, 0x17
.rodata:0000000000022E80                 DCB 0x6E, 0x8F, 0x98, 0x1A, 0xA8, 0xEE, 0xBD, 0xEF, 0x63
.rodata:0000000000022E80                 DCB 0x4E, 0x12, 0x4E, 0x64, 0x34, 0xB1, 0x74, 0x63, 0xA8
.rodata:0000000000022E80                 DCB 0x43, 0x35, 0xFD, 0x23, 0x7D, 0xFF, 0x77, 0xEA, 0xE0
.rodata:0000000000022E80                 DCB 0x70, 0x41, 0xDC, 0xEC, 0x68, 0xC1, 0x80, 0xDA, 0xD4
.rodata:0000000000022E80                 DCB 0x4A, 2, 0x78, 0x2F, 0x12, 0x55, 0x9E, 0x18, 2, 0xEE
.rodata:0000000000022E80                 DCB 0x68, 0x70, 0xCF, 0x21, 0x6E, 0xB9, 0x75, 0xEC, 0xB5
.rodata:0000000000022E80                 DCB 0x17, 0, 0, 0, 0, 0


```

然后应该在最后赋值的地方打印. 那么应该看到下面这行代码.

```
*(_QWORD *)&v6 = byte_3E1BA;

```

又或者说打印 RegisterNative  
以下展示 hook,libart 的 registerNative

```
function hook_libart() {
    var module_libart = Process.findModuleByName("libart.so");
    var symbols = module_libart.enumerateSymbols();     //枚举模块的符号

    var addr_GetStringUTFChars = null;
    var addr_FindClass = null;
    var addr_GetStaticFieldID = null;
    var addr_SetStaticIntField = null;
    var addr_RegisterNatives = null;
    for (var i = 0; i < symbols.length; i++) {
        var name = symbols[i].name;
        if (name.indexOf("art") >= 0) {
            if ((name.indexOf("CheckJNI") == -1) && (name.indexOf("JNI") >= 0)) {
                if (name.indexOf("GetStringUTFChars") >= 0) {
                    console.log(name);
                    addr_GetStringUTFChars = symbols[i].address;
                } else if (name.indexOf("FindClass") >= 0) {
                    console.log(name);
                    addr_FindClass = symbols[i].address;
                } else if (name.indexOf("GetStaticFieldID") >= 0) {
                    console.log(name);
                    addr_GetStaticFieldID = symbols[i].address;
                } else if (name.indexOf("SetStaticIntField") >= 0) {
                    console.log(name);
                    addr_SetStaticIntField = symbols[i].address;
                }else if(name.indexOf("RegisterNatives")>= 0 ){
                    console.log(name)
                    addr_RegisterNatives = symbols[i].address;
                }
            }
        }
    }

    //hook jni的一些函数
    // if (addr_GetStringUTFChars) {
    //     Interceptor.attach(addr_GetStringUTFChars, {
    //         onEnter: function (args) {
    //             //打印调用栈
    //             // console.log('addr_GetStringUTFChars onEnter called from:\n' +
    //             //     Thread.backtrace(this.context, Backtracer.FUZZY)
    //             //         .map(DebugSymbol.fromAddress).join('\n') + '\n');
    //         }, onLeave: function (retval) {
    //             // retval const char*
    //             console.log("addr_GetStringUTFChars onLeave:", ptr(retval).readCString(), "\r\n");
    //         }
    //     });
    // }

    // if (addr_FindClass) {
    //     Interceptor.attach(addr_FindClass, {
    //         onEnter: function (args) {
    //             console.log("addr_FindClass:", ptr(args[1]).readCString());
    //         }, onLeave: function (retval) {

    //         }
    //     });
    // }
    // if (addr_GetStaticFieldID) {
    //     Interceptor.attach(addr_GetStaticFieldID, {
    //         onEnter: function (args) {
    //             console.log("addr_GetStaticFieldID:", ptr(args[2]).readCString(), ptr(args[3]).readCString());
    //         }, onLeave: function (retval) {

    //         }
    //     });
    // }
    // if (addr_SetStaticIntField) {
    //     Interceptor.attach(addr_SetStaticIntField, {
    //         onEnter: function (args) {
    //             console.log("addr_SetStaticIntField:", args[3]);
    //         }, onLeave: function (retval) {

    //         }
    //     });
    // }
    if(addr_RegisterNatives){
        Interceptor.attach(addr_RegisterNatives, {
            onEnter: function (args) {
                console.log("addr_RegisterNatives:", hexdump(args[2]));
                console.log("addr_RegisterNatives:",ptr(args[2]).readPointer().readCString());
                //  +8 是因为arm64指针长度为8, 加8就到下一个参数去了
                console.log("addr_RegisterNatives:",ptr(args[2]).add(0x8).readPointer().readCString());
                
            }, onLeave: function (retval) {

            }
        });
    }
}


```

```
[Pixel 2::com.example.hellojni_sign2]-> addr_RegisterNatives:              0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F  0123456789ABCDEF
7fc13f3140  b4 51 c9 ca 79 00 00 00 ba 51 c9 ca 79 00 00 00  .Q..y....Q..y...
7fc13f3150  4c de c5 ca 79 00 00 00 64 ea 97 21 8c 14 d4 68  L...y...d..!...h
7fc13f3160  c0 c1 4c e5 79 00 00 00 70 4e 26 e5 79 00 00 00  ..L.y...pN&.y...
7fc13f3170  80 32 3f c1 7f 00 00 00 68 32 3f c1 7f 00 00 00  .2?.....h2?.....
7fc13f3180  50 32 3f c1 7f 00 00 00 28 d9 0b e5 79 00 00 00  P2?.....(...y...
7fc13f3190  00 72 42 e5 79 00 00 00 60 00 00 00 00 00 00 00  .rB.y...`.......
7fc13f31a0  20 33 3f c1 7f 00 00 00 00 84 4a e5 79 00 00 00   3?.......J.y...
7fc13f31b0  40 4a 14 6a 7a 00 00 00 00 00 43 00 79 00 00 00  @J.jz.....C.y...
7fc13f31c0  50 32 3f c1 00 00 59 00 14 4a 4e 49 5f 4f 6e 4c  P2?...Y..JNI_OnL
7fc13f31d0  6f 61 64 00 00 00 00 00 00 00 00 00 00 00 00 00  oad.............
7fc13f31e0  00 ea 4b e5 79 00 00 00 43 00 00 00 59 00 00 00  ..K.y...C...Y...
7fc13f31f0  00 ea 4b e5 79 00 00 00 64 ea 97 21 8c 14 d4 68  ..K.y...d..!...h
7fc13f3200  40 4a 14 6a 7a 00 00 00 60 00 00 00 00 00 00 00  @J.jz...`.......
7fc13f3210  e0 32 c9 d8 79 00 00 00 57 00 00 00 00 00 00 00  .2..y...W.......
7fc13f3220  00 84 4a e5 79 00 00 00 1c 33 3f c1 7f 00 00 00  ..J.y....3?.....
7fc13f3230  20 33 3f c1 7f 00 00 00 80 32 c9 d8 79 00 00 00   3?......2..y...
addr_RegisterNatives: sign1  // 方法名称
addr_RegisterNatives: (Ljava/lang/String;)Ljava/lang/String;  // 参数签名

```

再来打印一下 byte_3E1BA.  
这个是 print_str, 之前我们用过的.

```
[Pixel 2::com.example.hellojni_sign2]-> print_str(0x3E1BA)
addr  0x79cac951ba   add_str (Ljava/lang/String;)Ljava/lang/String;

```

**接下来用 inline 打印. 这里要打开 opcode, opcode bytes 是 4**

```
.text:00000000000072F0 6C 03 80 92                 MOV             X12, #0xFFFFFFFFFFFFFFE4
.text:00000000000072F4 08 01 3A 91                 ADD             X8, X8, #byte_22E80@PAGEOFF
.text:00000000000072F8 6B E9 06 91                 ADD             X11, X11, #byte_3E1BA@PAGEOFF ; 2 : 3E1BA找到了
.text:00000000000072FC
.text:00000000000072FC             loc_72FC                                ; CODE XREF: JNI_OnLoad+F4↓j
.text:00000000000072FC 2D FD 42 D3                 LSR             X13, X9, #2 ; 3 : 可以看到有循环
.text:0000000000007300 AD 7D CA 9B                 UMULH           X13, X13, X10
.text:0000000000007304 AD FD 41 D3                 LSR             X13, X13, #1
.text:0000000000007308 AD 21 0C 9B                 MADD            X13, X13, X12, X8
.text:000000000000730C 0E 01 09 8B                 ADD             X14, X8, X9
.text:0000000000007310 AD 01 09 8B                 ADD             X13, X13, X9
.text:0000000000007314 CE D1 44 39                 LDRB            W14, [X14,#0x134]
.text:0000000000007318 AD 61 44 39                 LDRB            W13, [X13,#0x118]
.text:000000000000731C AD 01 0E 4A                 EOR             W13, W13, W14 ; 4 : EOR 异或, 打印异或完之后的东西
.text:0000000000007320 6D 69 29 38                 STRB            W13, [X11,X9] ; 5: 我们来打印这个地方
.text:0000000000007324 29 05 00 91                 ADD             X9, X9, #1
.text:0000000000007328 3F 9D 00 F1                 CMP             X9, #0x27 ; '''
.text:000000000000732C 81 FE FF 54                 B.NE            loc_72FC
.text:0000000000007330 EA FF FF F0                 ADRP            X10, #sub_6E4C@PAGE
.text:0000000000007334 60 01 67 9E                 FMOV            D0, X11 ; 1. 现在找3E1BA

```

```
function inline_hook(){
    var base_hello_jni = Module.findBaseAddress("libhello-jni.so");
    console.log(base_hello_jni);
    if(base_hello_jni){
        console.log(base_hello_jni);
        var addr_07320 = base_hello_jni.add(0x07320);
        // inline hook 和 hook函数是一样的
        Interceptor.attach(addr_07320,{
            onEnter:function(args){
                // 
                // 64位: X0-X30, XZR(零寄存器)   要学汇编! 不然没法玩inline hook
                console.log("addr_07320 x13",this.context.x13, " x14",this.context.x14);
            },
            onLeave:function(retval){}
        })

        
    
    }
}

```

然后我们会发现没法 hook 成功，是因为 frida 注入脚本的时候，我们要 hook 的 so 文件还没加载。这时候我们需要来 hook `dlopen`,`dlopen`加载成功以后在去执行`inline_hook`

```
void * dlopen（const char * filename ，int flag ）;

```

dlopen 有 2 个参数, 我们打印第一个 `filename`

```
function hook_dlopen(){

    var dlopen = Module.findExportByName(null,"dlopen");
    Interceptor.attach(dlopen,{
        onEnter:function(args){
            console.log("dlopen",ptr(args[0]).readCString());
        },
        onLeave:function(retval){}
    })
}

```

发现没有我们需要的.

```
[Pixel 2::com.example.hellojni_sign2]-> dlopen /vendor/lib64/egl/libGLESv2_adreno.so
dlopen /vendor/lib64/egl/libEGL_adreno.so   
dlopen /vendor/lib64/egl/libGLESv2_adreno.so
dlopen /system/lib64/libEGL.so
dlopen /system/lib64/libGLESv2.so
dlopen /system/lib64/libGLESv1_CM.so
dlopen /vendor/lib64/egl/eglSubDriverAndroid.so
dlopen libadreno_utils.so
dlopen /vendor/lib64/egl/libEGL_adreno.so

```

在 android6.0 的时候, hook`dlopen`的时候是可以加载所有的 so 的, 但是在高版本的时候就没有了, 我们需要 hook 另外一个函数, 叫做`android_dlopen_ext`, 所以我们去 hook 他

```
function hook_dlopen(){

    var dlopen = Module.findExportByName(null,"dlopen");
    Interceptor.attach(dlopen,{
        onEnter:function(args){
            console.log("dlopen",ptr(args[0]).readCString());
        },
        onLeave:function(retval){}
    });

    var android_dlopen_ext = Module.findExportByName(null,"android_dlopen_ext");
    Interceptor.attach(android_dlopen_ext,{
        onEnter:function(args){
            console.log("android_dlopen_ext",ptr(args[0]).readCString());
        },
        onLeave:function(retval){}
    });
}

```

返回结果, 我们可以看到 libhello-jni.so 那么此时就可以去 hook 他了

```
Spawned `com.example.hellojni_sign2`. Resuming main thread!
[Pixel 2::com.example.hellojni_sign2]-> android_dlopen_ext /data/app/com.example.hellojni_sign2-VfsaSmMy0zmiYY_Le5Ylfw==/oat/arm64/base.odex
android_dlopen_ext /vendor/lib64/egl/libEGL_adreno.so
dlopen /vendor/lib64/egl/libGLESv2_adreno.so
android_dlopen_ext /data/app/com.example.hellojni_sign2-VfsaSmMy0zmiYY_Le5Ylfw==/lib/arm64/libhello-jni.so
dlopen /vendor/lib64/egl/libEGL_adreno.so
android_dlopen_ext /vendor/lib64/egl/libGLESv1_CM_adreno.so
dlopen /vendor/lib64/egl/libGLESv2_adreno.so
android_dlopen_ext /vendor/lib64/egl/libGLESv2_adreno.so
dlopen /system/lib64/libEGL.so
dlopen /system/lib64/libGLESv2.so
dlopen /system/lib64/libGLESv1_CM.so
dlopen /vendor/lib64/egl/eglSubDriverAndroid.so
android_dlopen_ext /vendor/lib64/hw/gralloc.msm8998.so
dlopen libadreno_utils.so
dlopen /vendor/lib64/egl/libEGL_adreno.so
android_dlopen_ext /vendor/lib64/hw/android.hardware.graphics.mapper@2.0-impl.so
android_dlopen_ext /vendor/lib64/hw/gralloc.msm8998.so


```

然后再修改下代码, 整体代码大概如下

```
function inline_hook(){
    var base_hello_jni = Module.findBaseAddress("libhello-jni.so");
    console.log(base_hello_jni);
    if(base_hello_jni){
        console.log("base_hello_jni",base_hello_jni);
        var addr_07320 = base_hello_jni.add(0x07320);
        // inline hook 和 hook函数是一样的
        Interceptor.attach(addr_07320,{
            onEnter:function(args){
                // 
                // 64位: X0-X30, XZR(零寄存器)   要学汇编! 不然没法玩inline hook
                console.log("addr_07320 x13",this.context.x13);
            },
            onLeave:function(retval){}
        })


    
    }
}
var is_hook_hello_jni = true;
function hook_dlopen(){

    var android_dlopen_ext = Module.findExportByName(null,"android_dlopen_ext");
    Interceptor.attach(android_dlopen_ext,{
        onEnter:function(args){
            var so_name = ptr(args[0]).readCString();
            if(so_name.indexOf("libhello-jni.so")){
                is_hook_hello_jni = true
            }
            console.log("android_dlopen_ext",so_name);
        },
        onLeave:function(retval){
            if(is_hook_hello_jni){
                inline_hook()
            }
        }
    });
}

```

再把 x13 的值放到 010editor 中查看. 就可以看到签名了.

至此，ollvm 字符串混淆的学习就暂时结束了。晚点找点案例来实战，找到案例了再来更新~