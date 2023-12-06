> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-270117.htm)

> [原创] 小红书 app shield so 逆向算法分析

const 小白龙_龙哥 = function () {}
=============================

> **_const a = 龙哥是全能的傻宝，每天都会换一种姿势来分享各种骚操作_**  
> **_const b = 10 月份开始准备密码学课程，11 月份准备更完 aes 密码学系列课程_**  
> **_const c = 除了密码学，还有各种 unidng 骚操作，反 unidbg，反反 unidbg，全是出自傻宝_**  
> **_const d = [赶紧点击加入星球吧，买到就是赚到 !](https://t.zsxq.com/BeeyBIQ)_**  
> **_return 无敌的龙哥，全能的傻宝_**

 

**_逆向小白一只，只能研究研究大佬完剩下的了，小红书 app 版本 6.87_**

 

**_直入正题，前面的就不分析了_**

AES 分析
======

> **_这里推荐一篇 aes 文章，有兴趣的大佬可以看看_**  
> **_https://bbs.pediy.com/thread-253884.htm_**  
> **_一般标准的 aes 逆向咱们需要关注一下几个点_**  
> **_1、明文是什么 ?_**  
> **_2、密钥是什么 ?_**  
> **_3、数据填充方式是什么 ?_**  
> **_4、初始化向量 IV 是什么 ?_**  
> **_5、工作模式是什么 ?_**  
> **_下面开始今天的正文_**

 

![](https://bbs.pediy.com/upload/attach/202111/939409_297AMGXC7W9S5MT.jpg)

 

来看 `sub_117A0` 函数把，有三个参数，分别是 `参数一：JNIEnv，参数二：main_hmac，参数三：main`，点进去看看

 

![](https://bbs.pediy.com/upload/attach/202111/939409_NJDSVE4845QPHES.jpg)

 

前面是是一些赋值操作，`CallStaticObjectMethodV(v11, v14, dword_B5084, v10);` 这里是调用 `baseDecode` 函数，参数是 `main_hmac` 字段，再往下就是获取字符串长度等，最后调用 `aes` 函数

 

![](https://bbs.pediy.com/upload/attach/202111/939409_JECNGUED8WN6YMT.jpg)

 

点进来，做了一些处理，然后继续调用 `aes` 函数

 

![](https://bbs.pediy.com/upload/attach/202111/939409_YJW8DMWDSWSPPXD.jpg)

 

这个函数，前面是定义了一些常量，中间有 `for` 循环，应该是初始化一些数据，在就是调用两个函数，先来看看 `init` 函数

### aes init

> **_init 有三个参数_**  
> **_参数一：目前看不出是啥_**  
> **_参数二：128_**  
> **_参数三：内存地址，应该是用来存放数据的_**

 

![](https://bbs.pediy.com/upload/attach/202111/939409_NMRHYXKQXJ5YSN4.jpg)

 

点进函数，这里先是调用了 `key_expansion` 函数，这个应该就是获取 `aes` 密钥的函数了  
下面是两个 `for` 循环，具体作用未知，来看看 `key_expansion` 函数

### key expansion

![](https://bbs.pediy.com/upload/attach/202111/939409_RKXE83XG2NS3A6W.jpg)

 

这个函数就比较清晰了，参数二传进来的是 `128` 所以这里是 `AES-128`，而 `AES-128` 的密钥一般是 `16` 个字节  
再往下是个 `while` 循环，一共循环 10 次，`aes init` 的逻辑比较简单，基本就是初始化数据，看看另外一个函数 `aesFunction`

### aes function

> **_这个函数的参数还是比较多的，就不一一分析了_**

 

![](https://bbs.pediy.com/upload/attach/202111/939409_E9WKN6X8A5C5DY3.jpg)

 

这里有两个加解密函数， `encrypt decrypt`，咱们这次主要看解密的，先看看 `sub_4C194` 函数

 

![](https://bbs.pediy.com/upload/attach/202111/939409_AE2CGSBNZ9JTQGE.jpg)

 

核心逻辑是这个 `while` 循环，每次循环都会先调用 `a6` 函数，也就是 `XYAES_decrypt` 函数，一共会执行 `base64DenodeLength / 16` 次，执行完后再来个小的 `for` 循环，具体作用未知，下面来看看 `XYAES_decrypt` 函数

### xyaes decrypt

![](https://bbs.pediy.com/upload/attach/202111/939409_YJZDPNYV93U544Q.jpg)

 

这个函数也比较清晰，都是运算逻辑，这里应该就是 `aes` 的轮训加密运算了

 

**_Tips：至此整个逻辑就分析的差不多了，这个样本的 `aes` 还算比较友好，没有做过多的符号混淆，虽然算法魔改了，但是逻辑都很清晰，下面来使用 `unidbg` 动态调试，辅助分析看看，一些数据_**

### unidbg

![](https://bbs.pediy.com/upload/attach/202111/939409_8F8J6EY8TER3BFB.jpg)

 

现在入口处 `aes1` 函数下个端点，看看传了那些函数

 

![](https://bbs.pediy.com/upload/attach/202111/939409_4AUD6CK6XHYAX7E.jpg)

 

成功断下来，`r1 r3` 这两个参数是数字，应该是 `r0 r2` 的长度

 

![](https://bbs.pediy.com/upload/attach/202111/939409_XYCKMY4CBCJHME5.jpg)

 

查看 `r0` 的数据，应该是 `main_hmac base64Decode` 出来的数据

 

![](https://bbs.pediy.com/upload/attach/202111/939409_57VP4J7B6CFM7SW.jpg)

 

使用 `CyberChef` 解码看一下，结果一样

 

![](https://bbs.pediy.com/upload/attach/202111/939409_NCT78RHQ2QF7K2N.jpg)

 

查看 `r2` 数据，发现是 `device_id` 长度刚好 `36` 个字节

 

![](https://bbs.pediy.com/upload/attach/202111/939409_Z42SXTFHSUBBZYN.jpg)

 

参数五在 `msp` 堆栈里，查看数据，应该是 `buffer`，`aes` 入口参数分析的差不多了

 

![](https://bbs.pediy.com/upload/attach/202111/939409_9599K4TRXAJQ69M.jpg)

 

在来 `aesInit` 函数下个端点，看看参数是啥

 

![](https://bbs.pediy.com/upload/attach/202111/939409_8THPXB5XXPHQQP9.jpg)

 

`r0` 的数据是，`device_id` 的前 `16` 个数据

 

![](https://bbs.pediy.com/upload/attach/202111/939409_VUHT9DC63MPCCBY.jpg)

 

`r2` 寄存器的数据，暂时看不出来是啥

 

![](https://bbs.pediy.com/upload/attach/202111/939409_579C7N77JR5E28M.jpg)

 

点进来在 `key_expansion` 函数在下个端点

 

![](https://bbs.pediy.com/upload/attach/202111/939409_ZBWJCNU3PZHEG38.jpg)

 

这三个参数都是我们传进来的，单步调试，然后在查看 `r2` 寄存器的数据 `0xbffff488`，分析下 `key_expansion` 就知道数据最终是存到参数三的地址里了

 

![](https://bbs.pediy.com/upload/attach/202111/939409_DN9CHC59XBWNEKY.jpg)

 

这里猜测应该就是生成的 `aes key` 了

 

![](https://bbs.pediy.com/upload/attach/202111/939409_VFVNE6TJSJMKA23.jpg)

 

这里的两个 `for` 循环都是取 `v16 + 60` 的数据，`v16 = a3 = r2` 咱们直接看下数据

 

![](https://bbs.pediy.com/upload/attach/202111/939409_DWK6YKARAT6Q2DR.jpg)

 

查看 `241` 个字节，**_在 arm 里地址加一就是加四个字节_**，`0xa = 10`

 

![](https://bbs.pediy.com/upload/attach/202111/939409_MPZ9BMCQKKWTAEZ.jpg)

 

在 `key_expansion` 里可以看到这里赋的值

 

![](https://bbs.pediy.com/upload/attach/202111/939409_S3JMZABXNVSGC5U.jpg)

 

再来这里下断点，查看输入

 

![](https://bbs.pediy.com/upload/attach/202111/939409_NEUB6TST88TH4SM.jpg)

 

参数一是 `hmac` 解码的结果，参数三是数字 `0x60` 应该是 `hmac` 的长度

 

![](https://bbs.pediy.com/upload/attach/202111/939409_HU3PC6ZFM2VE4GU.jpg)

 

参数二是 `buffer` 应该是用来存放结果的，可以先记录地址，参数四应该是前面 `aes-init` 的结果

 

![](https://bbs.pediy.com/upload/attach/202111/939409_W6A92DPVFXBWRS4.jpg)

 

![](https://bbs.pediy.com/upload/attach/202111/939409_SBPH4A9SNBR587S.jpg)

 

参数五的数据，是前面的四个常量

 

![](https://bbs.pediy.com/upload/attach/202111/939409_JK73V2ZZ7XWMA9S.jpg)

 

输入 `n` 单步执行，再次查看 参数二的数据，发现有数据了，应该是 `aes` 的解密结果

##### ---------------------------- 华丽的分割线 ----------------------------

MD5 分析
======

![](https://bbs.pediy.com/upload/attach/202111/939409_7YSXAP9CGJWXFJ7.jpg)

 

`intercept` 函数的 `0x93EB0` 就是 `md5` 的入口函数，有四个参数，先不分析具体传的啥了

 

![](https://bbs.pediy.com/upload/attach/202111/939409_P5V92HQVEVNYWEM.jpg)

 

这里的 `sub_44418` 函数，是 `md5 init` 逻辑，点进去看看

 

![](https://bbs.pediy.com/upload/attach/202111/939409_ESRKSRVGPCASWSU.jpg)

 

经过动态调试走的这个分支

 

![](https://bbs.pediy.com/upload/attach/202111/939409_S4KDXXMYX4STKTH.jpg)

 

点进来，这些逻辑看不出来是啥

 

![](https://bbs.pediy.com/upload/attach/202111/939409_6MJZ7GUDYVD2EPS.jpg)

 

往下走，就是核心逻辑了，这里的两个 `for` 循环，会拿前面 `aes decrypt` 结果跟 `0x36 0x5c` 进行异或，到的新的 `hex` 数据，然后在调用 `md5 init update transform` 等函数，来看看 `sub_4C4F4 init` 函数

 

![](https://bbs.pediy.com/upload/attach/202111/939409_BQBBQRCKE4EXSA3.jpg)

 

没啥逻辑，就最后获取一个内存地址，是个函数跳转过去，暂时静态没法分析，稍后动态调试看看

 

![](https://bbs.pediy.com/upload/attach/202111/939409_GA2GST8AKSCD347.jpg)

 

`sub_4C5C0 update transform` 函数也是同样的逻辑，好家伙，看来只能上 `unidbg` 了啊

### unidbg

> **_先来看看 md5Handler 入口函数的输入是啥，unidbg 下个断_**

 

![](https://bbs.pediy.com/upload/attach/202111/939409_D37QG6W2BRDQWUS.jpg)

 

参数一，参数二，看不出来是啥

 

![](https://bbs.pediy.com/upload/attach/202111/939409_UBYZ4D6VSQ5FJ23.jpg)

 

参数三报错了，参数四是字符串 `main`

 

![](https://bbs.pediy.com/upload/attach/202111/939409_539MZ4SYATW54BK.jpg)

 

在 `sub_44418 md5 init` 函数入口处，在下个断点

 

![](https://bbs.pediy.com/upload/attach/202111/939409_VCG9KSJNPJWSB42.jpg)

 

断下来，参数二跟参数四都是数字，应该就是，参数一参数三的长度了

 

![](https://bbs.pediy.com/upload/attach/202111/939409_DG6H9N3W6A836EF.jpg)

 

参数一看不出来是啥，感觉中间应该是内存地址，参数二看起来有点熟悉，是前面 `aes` 解密的结果

 

![](https://bbs.pediy.com/upload/attach/202111/939409_PFU6AJSWZBD5R28.jpg)

 

这里下个断点

 

![](https://bbs.pediy.com/upload/attach/202111/939409_2XMP5VBXECF942U.jpg)

 

参数一未知，参数二是 `aes` 解密结果，加上未知地址

 

![](https://bbs.pediy.com/upload/attach/202111/939409_G6Q3BYKH443E6FT.jpg)

 

参数三是数字 `64`，参数四也是未知的

 

![](https://bbs.pediy.com/upload/attach/202111/939409_92JMBSQN7KCHKFX.jpg)

 

接着来到了这里，`r0 ^ 0x36`，所以我们要在获取 `r0` 数据下个断点

 

![](https://bbs.pediy.com/upload/attach/202111/939409_SDQR6RFBNJRPZG6.jpg)

 

这里是 `r0 + 0x14 = 0x402a6014` 在取出 `0x402a6014` 地址的第一个字节也就是 `0x9f`

 

![](https://bbs.pediy.com/upload/attach/202111/939409_7PC2P223K6Y8XFX.jpg)

 

`0x402a6014` 地址的数据就是 `aes` 解密结果，长度 `64` 这里只需要循环 `64` 次即可

 

![](https://bbs.pediy.com/upload/attach/202111/939409_55XVGUHR32MYFMN.jpg)

 

再来这里下断点

 

![](https://bbs.pediy.com/upload/attach/202111/939409_H7996VS3CCM7TBG.jpg)

 

这里执行 `r1` 寄存器的函数，`s` 跟进去，然后在复制内存地址，`ida` 里跳过去

 

![](https://bbs.pediy.com/upload/attach/202111/939409_RNU8KGSDUM6HZGZ.jpg)

 

来到了这里，调用了 `md5Init` 函数，参数二是个常量

 

![](https://bbs.pediy.com/upload/attach/202111/939409_EJ89DEYPAPV3FBQ.jpg)

 

进入 `md5Init` 函数，在开头处下个断点

 

![](https://bbs.pediy.com/upload/attach/202111/939409_WN64ARSNNZHMJ7V.jpg)

 

参数一是个 `buffer` 应该是用来存放结果的，参数二是个常量数据

 

![](https://bbs.pediy.com/upload/attach/202111/939409_YTCRC3RKA8T9FSV.jpg)

 

可以复制这个地址 `ida` 跳转过去

 

![](https://bbs.pediy.com/upload/attach/202111/939409_RDFXXNT6FKJCFHD.jpg)

 

这里可以看到密密麻麻的常量数据，随便复制一个 `google` 搜索一下，原来是 `md5` 的 `k` 值

 

![](https://bbs.pediy.com/upload/attach/202111/939409_BAWYQR7A2YTGKS2.jpg)

 

再来分析 `md5Init` 函数

> **_先是调用 `_aeabi_memclr4` 清空 `a1 348` 字节数据_**  
> **_在把 md5 的四个常量 存进前 16 个字节里，这里的四个常量看起来不是标准的_**  
> **_参数二如果不为空的话，在吧 k 值的 256 个字节数据，复制到参数一的 23_ 4 字节后面 ***

 

![](https://bbs.pediy.com/upload/attach/202111/939409_Q8HS5EPQP9SE3PF.jpg)

 

接着回到 `sub_4C5C0 md5 update transform` 函数，下个断点

 

![](https://bbs.pediy.com/upload/attach/202111/939409_C493ZAAY6T3NXAK.jpg)

 

`s` 进入函数，`ida` 跳转过去，进入 `sub_4CFAA` 函数，开头下断点

 

![](https://bbs.pediy.com/upload/attach/202111/939409_PCMWSTMNYB6Q224.jpg)

 

查看参数一的 `348` 个字节，正是前面 `md5init` 函数结果

 

![](https://bbs.pediy.com/upload/attach/202111/939409_ZDGU7DAEYKWX23W.jpg)

 

参数二还是 `aes` 结果，参数三是个数字 `0x40`

 

![](https://bbs.pediy.com/upload/attach/202111/939409_S5T67PNE48WAKAA.jpg)

 

这里就是最后一个处理函数了，判断赋值，最后调用 `md5Transform` 函数进行运算获取结果，运算结果就不看了，再回到入口函数处

 

![](https://bbs.pediy.com/upload/attach/202111/939409_MKMNXTVR55GRMG8.jpg)

 

第一个 `md5 init` 函数，分析完了，下面两个也是核心逻辑

##### ---------------------------- 华丽的分割线 ----------------------------

XYData 分析
=========

![](https://bbs.pediy.com/upload/attach/202111/939409_YB3263P2CVUY6B5.jpg)

 

前面是获取 `md5 加密结果 build sappid device_id` 等参数，然后调用 `sub_404E8` 函数，参数都加了注释

 

![](https://bbs.pediy.com/upload/attach/202111/939409_B2J3MH92GXYQ3B5.jpg)

 

点进来，这里我基本都加了注释，总结这个函数的核心逻辑就是拼接字符串

 

![](https://bbs.pediy.com/upload/attach/202111/939409_B3BZ2GZUKDEWQV4.jpg)

 

再往下还有两个函数，其中第一个是核心函数，执行完在拼接 `XY` 就是最终的结果了，先来看看 `sub_406FC` 这个函数

 

![](https://bbs.pediy.com/upload/attach/202111/939409_PBCF679KNHFVFNC.jpg)

 

点进来，主要分析这两个

 

![](https://bbs.pediy.com/upload/attach/202111/939409_6VWSNSZRBET53YJ.jpg)

 

这是 `xydata` 函数，看不出来是啥

 

![](https://bbs.pediy.com/upload/attach/202111/939409_3DVD6TTPTPND7Q8.jpg)

 

点进 `sub_40C74` 函数里，里面又调用了两个

 

![](https://bbs.pediy.com/upload/attach/202111/939409_HNTW57JR9UAY2S9.jpg)

 

进入 `sub_4AF1C` 函数，有个三个参数，`内存地址，数字，字符串`，里面都是运算逻辑，感觉是初始化一些数据

 

![](https://bbs.pediy.com/upload/attach/202111/939409_ETEMBH378RFV64P.jpg)

 

在看 `sub_4A94C` 函数，这里面就是真正的数据处理逻辑了。回到开始，查看 `sub_40CFC` 函数

 

![](https://bbs.pediy.com/upload/attach/202111/939409_6YSXXZG29M46QBU.jpg)

 

这里核心两个函数，一个简单的数据处理，一个 `base64` 编码函数

### unidbg

![](https://bbs.pediy.com/upload/attach/202111/939409_Y2HHM57N3BN3P5E.jpg)

 

这个函数执行前，下个断点

 

![](https://bbs.pediy.com/upload/attach/202111/939409_6C2UJT4CMZNWURE.jpg)

 

先记录下 `r0` 的地址 `0xbffff6a8`，这是用来存放结果的

 

![](https://bbs.pediy.com/upload/attach/202111/939409_QFMH8UG7MZW5ZNB.jpg)

 

查看参数三的数据，正是 `build id`

 

![](https://bbs.pediy.com/upload/attach/202111/939409_WGNAZ7ZC2XGCCR9.jpg)

 

参数七在 `msp` 里，查看数据正是 `md5` 的加密结果，其余的参数就不看了，`n` 执行

 

![](https://bbs.pediy.com/upload/attach/202111/939409_YZHN4FNV8ZGG5YG.jpg)

 

在查看 `r0 0xbffff6a8` 地址的数据，密密麻麻的，我们需要的数据就在其中一个地址里，就不去一一找了

 

![](https://bbs.pediy.com/upload/attach/202111/939409_G7FAMXC2MNMX572.jpg)

 

进入 `sub_406FC` 函数，在开头处下断点

 

![](https://bbs.pediy.com/upload/attach/202111/939409_EA5Y4MZF5P53JQP.jpg)

 

参数一不知道是啥，参数二就是前面 `sub_404E8` 函数的执行结果

 

![](https://bbs.pediy.com/upload/attach/202111/939409_D88WKDS3JUQNTGK.jpg)

 

前面的就不分析了，直接在 `sub_40C74` 函数执行前下个断点

 

![](https://bbs.pediy.com/upload/attach/202111/939409_V7K7WWV4RK3TMRE.jpg)

 

参数一，是 `build device_id sappid` 等参数的拼接结果，参数二是 `0x53`

 

![](https://bbs.pediy.com/upload/attach/202111/939409_93HA9DFQF77DCZZ.jpg)

 

参数三是 `buffer` 用来存放数据，参数四目前不知道是啥，这里先记录下参数三的地址 `0x402920c0`

 

![](https://bbs.pediy.com/upload/attach/202111/939409_9NNXGARFF7J5JPE.jpg)

 

进入 `sub_40C74` 函数，在 `sub_4A94C` 这里下断点

 

![](https://bbs.pediy.com/upload/attach/202111/939409_6VY32AV7949A7CQ.jpg)

 

参数一是上面一个函数的执行结果，参数二三四是传进来的，`n` 执行

 

![](https://bbs.pediy.com/upload/attach/202111/939409_U93VAYFCBK2VQ97.jpg)

 

查看参数三地址的数据，这里就是运算结果

 

![](https://bbs.pediy.com/upload/attach/202111/939409_7QGU7DN999HXKR4.jpg)

 

复制到 `CyberChef` 看看 `base64` 结果，不太一样，继续往后分析

 

![](https://bbs.pediy.com/upload/attach/202111/939409_B59VXTNDURB8HWY.jpg)

 

这里下个断点

 

![](https://bbs.pediy.com/upload/attach/202111/939409_QW5Z98BRT7UPUKU.jpg)

 

查看参数一的数据，这个就是最终结果

 

![](https://bbs.pediy.com/upload/attach/202111/939409_PXDA5EC7TWTZ7MK.jpg)

 

`n` 执行，查看 `r0` 数据，`base64` 的结果已经出来了

 

![](https://bbs.pediy.com/upload/attach/202111/939409_CHFP4FDSHGVZ2XK.jpg)

 

在使用 `CyberChef` 看看，随便复制一段，结果一样

##### ---------------------------- 华丽的分割线 ----------------------------

 

![](https://bbs.pediy.com/upload/attach/202111/939409_SBH2UV3CB97NPA9.jpg)

> 分析完后再来个 `python` 还原的运行结果，正常请求

> 原文链接: [小红书 app shield so 加密算法分析破解还原: https://www.qinless.com/608](https://www.qinless.com/608)

[【公告】看雪团队招聘安全工程师，将兴趣和工作融合在一起！看雪 20 年安全圈的口碑，助你快速成长！](https://job.kanxue.com/position-read-1104.htm)

最后于 2021-11-5 18:25 被爬山的小脑虎编辑 ，原因：

[#加密算法](forum-4-1-5.htm)

上传的附件：

*   [libshield-so.zip](javascript:void(0)) （3.30MB，48 次下载）