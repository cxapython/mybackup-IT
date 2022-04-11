> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-254042.htm)

> [原创] 一种还原白盒 AES 秘钥的方法

一种还原白盒 AES 秘钥的方法
================

背景
--

在日常逆向分析工作有遇到过一个白盒 AES 算法，在网上找到这样一篇还原该白盒算法秘钥的文章：  
[DFA 分析白盒 AES 算法](https://blog.quarkslab.com/differential-fault-analysis-on-white-box-aes-implementations.html) ，通过学习该文章，总结了一些心得，在这里分享下。

Differential Fault Analysis
---------------------------

详细的理论可以看上面那篇 blog，我这里只挑一些重点来说明下。

### AES 算法

AES 算是我们日常开发中最常用一种对称加密算法了，加密过程如下：  
![](https://bbs.pediy.com/upload/attach/201908/608531_U9BPS8UR8GEY9Z5.png)  
主要有这四种操作：  
1.S 盒字节代换  
2. 行移位  
3. 列混淆  
4. 轮秘钥加

### 白盒 AES 算法

白盒算法是将秘钥混淆到算法中，让攻击者即便能够获取算法的内部细节（能够动态调试），也无法 还原出秘钥的一种算法，常见的白盒算法有：白盒 AES, 白盒 SMS4。

### DFA 分析 AES-128 加密

由 AES 加密算法流程可以看出: 第 10 次轮秘钥加之前是没有列混淆的。如果我们在第九轮列混淆之前构造如下两组数据：  
![](https://bbs.pediy.com/upload/attach/201908/608531_Q5YR8Q2TVPDT9CS.png)  
从上图两个状态矩阵可以看出，状态矩阵中只有第一个字节不一样。如果当前状态继续往下推导，可以有如下：

*   MixColumns
*   AddRoundKey K9
*   SubBytes
*   ShiftRows
*   AddRoundKey K10  
    详细推导过程可以参照原文 blog：  
    最终可以推导得到如下表达式：  
    ![](https://bbs.pediy.com/upload/attach/201908/608531_B3BXH7GEG3GY6JW.png)  
    四个表达式表示了 Z 与（Y0,Y1,Y2,Y3）的关系。  
    **对 Y0 从 0~255 就能得到对应 Z 的取值集合，同理对 Y1,Y2,Y3 取值，都能得到一个 Z 的取值范围（这里是多对一映射）。  
    所以最终 Z 的取值只能是这 4 个 z 的取值范围的交集**  
    Z 的取值范围确定后，对应也可以确定一组（Y0,Y1,Y2,Y3）的值，继而由：![](https://bbs.pediy.com/upload/attach/201908/608531_78WCYY343DMRT7C.png)  
    可以得到一组 K10 的（0,7,10,13）的位置值。  
    同理可以改变 X 的值，通过合并得到唯一确定的 K10（0,7,10,13）的值。  
    同理改变其他位置上一个字节的值，可以得到另外 3 组位置的值，继而可以还原得到整个 K10 的值，再根据 AES 秘钥拓展算法，最终可以还原原始的加密秘钥。

### DFA 算法实现

这里的算法分为两个部分：  
1. 产生这些 fault 数据的方法:  
该算法在 [DFA 产生 Fault 数据](https://github.com/SideChannelMarvels/Deadpool/blob/master/README_dfa.md)  
该算法主要是通过静态修改二进制文件方式来修改 R9 的一个字节，从而输出 Fault 数据的。（这里面具体的实现细节没太弄懂，有兴趣都可以自行去看代码，后面的实战，我主要用 IDA 动态 Patch 方式实现输出 Fault 数据，没有用到这里算法。）

 

2. 拿到这些 fault 数据后，怎么样还原出白盒 AES 秘钥  
该算法实现是在 github 上：[phoenixAES](https://github.com/SideChannelMarvels/JeanGrey/tree/master/phoenixAES)  
下面主要对该算法的学习，解读。  
首先该算法的输入一个格式化的文件，每一行分别是输入状态矩阵（16 进制字符串表示），输出状态矩阵（16 字节 128 位）

#### 算法输入

经过分析代码可以发现，其实代码中没有用到输入的值，只需对输出值进行计算，并且第一行的输出值定为 golden_ref, 即没有做改变的原始正常的值，其他输出值为 diff, 即在第九轮状态更改一字节后输出的值。

#### 算法输出

最终算法输出的是还原的第 10 轮的秘钥，要得到原始 AES 秘钥，还需逆推一下。

#### 算法关键点说明

```
_AesFaultMaps= [
# AES decryption
 [[True, False, False, False, False, True, False, False, False, False, True, False, False, False, False, True],
  [False, True, False, False, False, False, True, False, False, False, False, True, True, False, False, False],
  [False, False, True, False, False, False, False, True, True, False, False, False, False, True, False, False],
  [False, False, False, True, True, False, False, False, False, True, False, False, False, False, True, False]],
# AES encryption
 [[True, False, False, False, False, False, False, True, False, False, True, False, False, True, False, False],
  [False, True, False, False, True, False, False, False, False, False, False, True, False, False, True, False],
  [False, False, True, False, False, True, False, False, True, False, False, False, False, False, False, True],
  [False, False, False, True, False, False, True, False, False, True, False, False, True, False, False, False]]
]
```

该算法实现中有这样一个列表，这个列表是来判断当前 diff 是属于哪一类，对应秘钥哪 4 个位置，比如  
拿加密来说，在第九轮列变换之前改变状态矩阵第一列中某一个字节的值，最后的结果只会在（0,7,10,13）的位置上发生改变，也就对应第 10 轮秘钥的（0,7,10,13）位置上的值。同理改变第二列上的值，只会改变（1,4,11,14）位置上的值。

#### 状态矩阵乘法

![](https://bbs.pediy.com/upload/attach/201908/608531_6ANN576P268635A.png)  
AES 加密列变换阵数值只在：1，2 ，3 中取，  
AES 解密列变换的数值只在：9, 13, 11, 14 中取  
这里的列表  
_AesMult[1] 可以表示（0~255）中分别与 1 相乘的结果。  
_AesMult[9] 可以表示（0~255）中分别与 9 相乘的结果。

#### 核心算法

```
def _get_compat(diff, tmult, encrypt):
    print ("_get_compat","diff:%x" % diff, "tmult:%x" %tmult)
    ibox = [_AesSBox, _AesInvSBox][encrypt]
    itab = [0]*256
    for i,mi in enumerate(_AesMult[tmult]):
        itab[mi] = i
    candi = [itab[ibox[j ^ diff] ^ ibox_j] for j, ibox_j in enumerate(ibox)]
    return candi
```

这里 j=S(Y0), ibox_j = invSBox(Y0) = Y0  
由上述推导公式：diff = S(Y0) + S(Y0+2z) 可以变换得（加号均表示异或）：  
diff + S(Y0) = S(Y0+2z)  
invSBox(diff + S(Y0)) = Y0 + 2z  
invSBox(diff + S(Y0)) + Y0 = 2z  
candi = [itab[ibox[j ^ diff] ^ ibox_j] for j, ibox_j in enumerate(ibox)]  
所以这里这句是返回是 S(Y0) 从 0~255 取值时，对应 z 的取值。  
所以后面会对 z 进行求交集操作，然后反过来根据 z 的值，去取 S(Y0) 的取值列表，最后通过多组数据的求解归并得到唯一确定的 S(Y0) 的值，最后求秘钥时会有一个：  
key[Keys[j]]=list(c[index][0][j])[0] ^ Gold[j]  
![](https://bbs.pediy.com/upload/attach/201908/608531_NAM9KHQQAU9WX3T.png)

实战
--

讲了这么多理论算法实现，还是来个具体的示例更容易说明问题，这里选的列子是这篇 blog 上提的：  
[LIFE 破解白盒 AES](https://blog.quarkslab.com/when-sidechannelmarvels-meet-lief.html)  
它里面提到的解法，是通过用 LIEF 将 Android so 转化成 Linux 上的可执行程序，然后对接上述的 DFA  
静态产生 Fault 数据方法得到一组数据，然后调用 [phoenixAES] 进行秘钥还原。我这边主要是通过 IDA 动态 Patch 来得到一组 Fault 数据。

 

里面的 APK 是 SECCON2016 CTF 中的一题：[SECCON2016 Online CTF-Binary / Crypto500 Obfuscated AES](https://github.com/SECCON/SECCON2016_online_CTF/tree/master/Binary/500_Obfuscated%20AES)

### java 层分析

![](https://bbs.pediy.com/upload/attach/201908/608531_MRXA94QR4D69G3K.png)  
主要从资源文件 arrays 中随机取一个字符串传入到 native 函数 a 进行加密，然后每隔 0.01s 递归调用一次，所以程序运行后界面上的 encrypted_flag 一直在变化。  
arrays 列表：  
![](https://bbs.pediy.com/upload/attach/201908/608531_CEY3UNETQCX4EZN.png)

### native 层分析

用 IDA 打开 lib-native.so 可以发现该 so 经过了 ollvm 混淆了，而且题目也说清楚了这个是一个 ollvm 混淆的白盒 AES 算法。  
通过 IDA 静态分析，可以定位到静态注册的 JNI 函数：Java_kr_repo_h2spice_crypto500_MainActivity_a  
简单分析一下可以定位到关键函数： TfcqPqf1lNhu0DC2qGsAAeML0SEmOBYX4jpYUnyT8qYWIlEq  
通过 frida hook 一下该函数查看该函数的输入输出值：

```
def hook_AES():
    jsCode = """
 
    function hookOAES() {
        var fnPtr = Module.findExportByName("libnative-lib.so", "_Z48TfcqPqf1lNhu0DC2qGsAAeML0SEmOBYX4jpYUnyT8qYWIlEqPhS_");
        var oldfnPtr = new NativeFunction(fnPtr, 'int', ['pointer', 'pointer']);
        Interceptor.replace(fnPtr, new NativeCallback(function (input, output) {
            send("***********OAES***********")
            var arg0 = Memory.readByteArray(input,16);
            console.log("a0:" + hexdump(arg0));
 
            var arg1 = Memory.readByteArray(output,16);
            console.log("a1 before:" + hexdump(arg1));
 
            var ret = oldfnPtr(input,output);
 
            var arg2 = Memory.readByteArray(output,16);
            console.log("a1 after:" + hexdump(arg2));
 
            return ret;
        }, 'int', ['pointer', 'pointer']));
    }
 
    Java.perform(function(){
        send("***************Start hook***************");
 
        var Main = Java.use("kr.repo.h2spice.crypto500.MainActivity");
        Main.a.overload("java.lang.String").implementation = function (input) {
            input = "Gew1cqzKp5K8sejh3FlTZlS/CISCpO81WmZ/oU4SJOk=";
            console.log("native-input:" + input);
            var ret = this.a(input);
            console.log("native-output:" + ret);
            return ret;
        }
        Main.onCreate.overload("android.os.Bundle").implementation= function (bundle) {
             console.log("MainActivity onCreate!!!");
             return this.onCreate(bundle);
        }
       hookOAES();
    });
    """
    return jsCode
```

这里有两点说明下：  
1. 在 hook native 层导出函数时用的 replace 的方式，而不是 attach 方式是为了能够在函数执行完后再打印一遍输入参数，因为在 C 中会经常传个指针，在函数中操作的结果也是保存在这个指针中。在本列中 TfcqPqf1lNhu0DC2qGsAAeML0SEmOBYX4jpYUnyT8qYWIlEq 函数 a0 参数是加密前 java 层传下来的数据，a1 是经过加密后的数据。

 

2. 这里也 hook 了 Java 层的 JNI 函数，为了让每次传下来的值保持一致，这里我是随机选取一个值："Gew1cqzKp5K8sejh3FlTZlS/CISCpO81WmZ/oU4SJOk="，这样没隔 0.01s 都会触发一次调用。

 

经过上面的 hook 分析可以得出：  
java 层传下来的加密数据都是: 32 字节的数据，所以在 C 层调用了两次 TfcqPqf1lNhu0DC2qGsAAeML0SEmOBYX4jpYUnyT8qYWIlEq 函数，分别加密前 16 字节和后 16 字节（ECB 模式），然后将加密后的数据 Base64 一下返回给 java 层，显示到界面。

### 秘钥还原

有了上面分析，结合 phoenixAES 算法，要还原秘钥，只需要想办法去产生 R9 Fault 数据，也就是在  
调用 TfcqPqf1lNhu0DC2qGsAAeML0SEmOBYX4jpYUnyT8qYWIlEq 函数内部，某个时间点（刚好在 R9 列变换前）更改状态矩阵的一个字节的数据，得到加密后的数据，对比输出数据与 golden_ref 是否刚好只有 4 字节不同。（如果只有 1 字节改变，说明 patch 太晚，如果有 16 字节不同说明 patch 太早）。

 

**所以问题就变成如何刚好找到 Patch 的时间点**，由于 so 被混淆了不能很直观的看到函数执行流程，这里用 IDA Python 去打印该函数的核心块的执行过程：

 

这里我调试是 armeabi-v7a 的 so, 其中断点 4 个位置分别是：  
1.OAES 函数的开始  
2.OAES 函数的结尾  
3.OAES 函数其中一处明显的子函数调用  
4.OAES 函数其中一处明显的数据处理块的起始地址。  
运行完可以明显观察到子函数调用了 10 次，每次调用之间调用了 4 次核心处理模块共九组，是不是可以类比 AES10 轮加密操作，所以我选择在第九组核心操作前进行 patch 一字节，然后观察到最后的输出结果果然是符合预期的，最后的 patch 代码如下：

```
# -*- coding: UTF-8 -*-
import idc
import idaapi
import breakFunctions
import time
import linecache
import dumpmemory
import random
 
soName = 'libnative-lib.so'
module_base  = breakFunctions.findModleBaseByName(soName)
gloden_ref = "868FC14BCCC36E3C657B73271A32DCD7"
 
# AESEnc
sub_4250  = module_base + 0x4250
sub_59A6  = module_base + 0x59A6
 
sub_46A4  = module_base + 0x46A4
 
sub_4DBC  = module_base + 0x4DBC
 
def main():
 
    idc.AddBpt(sub_4250)
    idc.MakeComm(sub_4250, "### WBoxAES  START###")
    print "[+]set breakpoint addr=>0x%X %s" % (sub_4250, "sub_4250")
 
    idc.AddBpt(sub_59A6)
    idc.MakeComm(sub_59A6, "### WBoxAES  END###")
    print "[+]set breakpoint addr=>0x%X %s" % (sub_59A6, "sub_59A6")
 
    idc.AddBpt(sub_4DBC)
    idc.MakeComm(sub_4DBC, "### BL      Z48lrsFdMdlAT0vSMVedxmqOkCBF7sCTbhCjYEp1rLP8vatWEGDPh###")
    print "[+]set breakpoint addr=>0x%X %s" % (sub_4DBC, "sub_4DBC")
 
    idc.AddBpt(sub_46A4)
    idc.MakeComm(sub_46A4, "### sub_46A4 ###")
    print "[+]set breakpoint addr=>0x%X %s" % (sub_46A4, "sub_46A4")
 
    auto_run_test()
 
def auto_run_test():
    logpath = "C:\\Users\\felix.li\\Desktop\\Crypto500_WAESCrack1.txt"
    f = open(logpath, "a")
    f.write(time.strftime("%Y-%m-%d %H:%M:%S", time.localtime(time.time())) + "*****START******\n")
 
    for i in range(16):
        runTo(sub_4250)
        count = 0
        is_this_round = True
        while True:
            addr = idc.GetEventEa()
            if addr == sub_4250:
                f.write("OAES START\n")
                print ("OAES START\n")
 
                R0 = idc.GetRegValue("R0")
                R1 = idc.GetRegValue("R1")
 
                srcR0 = toHexString(getMemory(R0, 16))
                print  srcR0
 
                if srcR0.lower() != "19ec3572accaa792bcb1e8e1dc595366":
                    print "not this round"
                    f.write("not this round\n")
                    is_this_round = False
                    break
                f.write("srcHexstr:" + srcR0 + "\n")
 
                ret_address = R1
                print hex(ret_address)
 
            if addr == sub_59A6:
                f.write("OAES END\n")
                print ("OAES END")
 
                print  hex(ret_address)
                desret = toHexString(getMemory(ret_address, 16))
                print "resHexstr:" + desret
                f.write("resHexstr:" + desret + "\n")
 
                break
            if addr == sub_4DBC:
                # f.write("BL lrsFdMdlAT0vSMVedxmqOkCBF7sCTbhCjYEp1rLP8vatWEGD\n")
                print ("BL lrsFdMdlAT0vSMVedxmqOkCBF7sCTbhCjYEp1rLP8vatWEGD")
 
            if addr == sub_46A4:
                # f.write("sub_46A4\n")
                print ("sub_46A4")
                count = count + 1
                if count == 33:
                    print hex(ret_address)
                    desret = toHexString(getMemory(ret_address, 16))
                    print "patch current state:" + desret
                    idc.PatchByte(ret_address + random.randint(0,15), random.randint(0,255))
 
            idaapi.continue_process()
            idaapi.wait_for_next_event(idc.WFNE_ANY, -1)
            event = idc.GetDebuggerEvent(idc.WFNE_ANY, -1)
 
            if (event <= 1):
                break
        if not is_this_round:
            idaapi.continue_process()
            idaapi.wait_for_next_event(idc.WFNE_ANY, -1)
 
def runTo(ea):
 
    addr = idc.GetEventEa()
    while addr != ea:
        idaapi.continue_process()
        idaapi.wait_for_next_event(idc.WFNE_ANY, -1)
        addr = idc.GetEventEa()
```

这里在调试时需要用 frida hook java 层的输入以保持每次输入数据都一样，所以这里的调试步骤为：  
1. 运行 android_server -p12345  
2. 运行 frida_server  
3.adb shell am start -D -n kr.repo.h2spice.crypto500/.MainActivity 进入调试模式  
4. 运行 frida hook java 层代码（c 层 hook 注释掉）  
5.ida attach  
6.jdb -connect com.sun.jdi.SocketAttach:hostname=127.0.0.1,port=8700  
7. 运行 IDA Python patch 脚本

 

注意这里的第 4 步不能跟第 5 步互换，不然 frida 会报错，通过这种方式意外的发现可以 hook 到 Actity 的 OnCreate 函数，我之前用 frida spawn 的方式老是报错。

 

整理脚本的输出的 Fault 结果，放入 [phoenixAES] 中:

```
# Cryptor500 OAES
with open('tracefile', 'wb') as t:
    t.write("""
    868FC14BCCC36E3C657B73271A32DCD7
    998FC14BCCC36E7F657B61271AC0DCD7
    86C5C14B32C36E3C657B73911A32E8D7
    868FAF4BCC6D6E3C267B73271A32DC1B
    8630C14B4BC36E3C657B73A41A32B7D7
    868FE74BCC096E3C957B73271A32DC9E
    86ADC14B7DC36E3C657B73881A32A4D7
    868FC1FFCCC3A13C653E7327A032DCD7
    9B8FC14BCCC36E3F657B0A271A87DCD7
    BC8FC14BCCC36E09657BF3271AAEDCD7
    8622C14B62C36E3C657B73E71A3224D7
    868FA34BCC8B6E3C1B7B73271A32DC4C
    868FC1CACCC3D73C654473270032DCD7
    658FC14BCCC36E52657B79271AE4DCD7
    867EC14B23C36E3C657B73A91A328AD7
    258FC14BCCC36E83657B55271A26DCD7
    E78FC14BCCC36EE1657B44271AA7DCD7
     """.encode('utf8'))
 
phoenixAES.crack_file('tracefile',[],True,False,3)
```

还原出 K10 秘钥为: 040D08DA68001026F3DC0D68897148B4  
再调用 [Key scheduling reversers](https://github.com/SideChannelMarvels/Stark) 中的  
aes_keyschedule 逆推得到 round0 秘钥 即 AES 秘钥。

```
$ aes_keyschedule 040D08DA68001026F3DC0D68897148B4 10
 
K00: 6C2893F21B6185E8567238CB78184945
K01: C013FD4EDB7278A68D00406DF5180928
K02: 6F12C9A8B460B10E3960F163CC78F84B
K03: D7537AE36333CBED5A533A8E962BC2C5
K04: 2E76DC734D45179E17162D10813DEFD5
K05: 19A9DF7F54ECC8E143FAE5F1C2C70A24
K06: FFCEE95AAB2221BBE8D8C44A2A1FCE6E
K07: 7F4576BFD46757043CBF934E16A05D20
K08: 1F09C1F8CB6E96FCF7D105B2E1715892
K09: A7638E006C0D18FC9BDC1D4E7AAD45DC
K10: 040D08DA68001026F3DC0D68897148B4
```

得到该 AES 秘钥为：6C2893F21B6185E8567238CB78184945

 

然后按道理应该循环遍历 arrays 中加密的 flag, 解出 flag。  
刚好加密的 flag 为第一个数据：g1UlZafiuGdCgpTkWYjaZg3kE6qCd7kF3kV+nMKcGHc=

 

验证如下：  
![](https://bbs.pediy.com/upload/attach/201908/608531_PMBMB6DDJ7TK749.png)  
flag 为：SECCON{owSkwPeH1CHQdPV9KWrSmz9n}

总结
--

1. 要通过 DFA 还原白盒 AES 的秘钥，其实只需要想办法构造一组合理 Fault 数据，后面的输入到 phoenixAES 的实现中，即可还原秘钥。  
2. 在构造 Fault 数据一般有两种方式：

```
* 官方给的静态更改的方式，这种方式好处是全自动化，不需要分析任何二进制代码。
* IDA 动态Patch的方式，这种需要分析代码，找准patch的时机。
```

3. 在动态 Patch 过程中，注意观察输出数据，如果输出只有 1 字节改变，说明 patch 太晚，如果输出有 16 字节不同说明 patch 太早，如果输出刚刚为 4 字节，且 4 字节的位置也能符合规则，则说明时机找准了。

[【公告】看雪招聘大学实习生！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)

最后于 2019-8-22 10:46 被 lfyyy 编辑 ，原因：