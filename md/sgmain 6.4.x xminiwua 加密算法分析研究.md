> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/mckitPK5uWMRP5S1odbAXQ)

仅供学习研究 。请勿用于非法用途，本人将不承担任何法律责任。
==============================

前言
==

> ❝
> 
> `sgmain 6.4.x` 版本的 `x-mini-wua` 参数加密算法研究分析 样本 `tianmao-8.11.0`
> 
> ❞

本文主要使用 `ida + unidbg + frida` 动静态分析  
之前有位大佬发了篇 `x-sign` 的加密流程（已经被删），我也是基于此来的灵感，大概的流程是

```
1、SGSAFETOKEN_IN = READFILE("file/app_SGLib/SG_INNER_DATA")<br style="visibility: visible;">2、json = AES_DECRYPT(SGSAFETOKEN_IN)<br style="visibility: visible;">3、sdfsd = BASE64_DECODE(json.getString("sdfsd"))<br style="visibility: visible;">4、x1 = AESDECRYPT(sdfsd)
5、x2 = XOR(x1)
6、result = "HHnB" + BASE64_ENCODE(AES_ENCRYPT(x2))
```

本文基于上面的流程，简单说一下，不会扩展到其他的细节上

「Tips：因为 unidbg 跑出来的 x-mini-wua 是短的，所以只能辅助分析大概的算法流程，具体细节博主是使用 frida hook 的方式来验证的」

0x1 读取文件
========

`SG_INNER_DATA` 文件在手机目录下就能找到。读取出来获取 `SGSAFETOKEN_IN` 字段

0x2 AES 解密 SGSAFETOKEN_IN
=========================

这个如何定位有两种方法（假设你已经分析过 xsign 参数了，那对 sgmain 的 aes 会有所了解）

*   1、直接去 `hook aes decrypt` 函数
    
*   2、通过 `traceRead` 看看 `SGSAFETOKEN_IN` 在哪里读的
    

![](https://mmbiz.qpic.cn/mmbiz_png/MWQJibR3pCZib8bf2GtgtPjUkYiaVs1Aq5gSm1ZjnEeyF5QrXwqZE9kCgqzt1xTuxwibOlY0uicr7YwBI066YDg3HicA/640?wx_fmt=png)

博主这里就直接说答案了，在这里就是 `aes cbc decrypt` 逻辑，通过 `inlinehook` 可以直接获取到 `data key iv result` 等数据

![](https://mmbiz.qpic.cn/mmbiz_png/MWQJibR3pCZib8bf2GtgtPjUkYiaVs1Aq5gFayckxibSbwDVeoGH6jvDxiazdmQlTJOIOUs7wSAPG9lpGqBOZOXnlYQ/640?wx_fmt=png)

分析到这里，就卡住了，因为 `SGSAFETOKEN_IN` 解密出来的结果是个 `json` 有四个字段，但是没有 `sdfsd`，可能是这个版本名称不一样

然后经过 `n` 多次的 `trace` 也还是没有找到想要的答案，这里各位读者大佬可以自己尝试呀，过程很心酸

最后还是决定从后往前追（当然也是遇到了 `n` 多个问题，有些问题卡了很久，这个后面在说）

0x6 base64 encode && aes encrypt
================================

最后的结果是 `"HHnB" + BASE64_ENCODE(AES_ENCRYPT(x2))`

![](https://mmbiz.qpic.cn/mmbiz_png/MWQJibR3pCZib8bf2GtgtPjUkYiaVs1Aq5grHm6bibFTXHt7DhO2tTmP14o5I6pN95uj3KLLGXGXtQJEk64gUyolcA/640?wx_fmt=png)

这里直接进行 `base decode` 结果可以正常出来，然后就可以进行 `unidbg trace` 了

*   1、在最后的结果下断点，使用 `shr` 搜索堆
    
*   2、搜索到之后就可以使用 `traceWrite` 了
    
*   3、不出意外的情况下，`trace` 到的地址就是 `aes encrypt`
    
*   4、然后根据代码接口，就可以判断出这是个 `aes cbc` 模式的加密，因为 `密钥编排、iv ^ data 的特征都很明显，还有 pkcs7 的填充标志`
    

以上都是博主亲测出来的流程，只不过以文字的方式描述，而没有加上图片，各位读者大佬如果没有思路可根据以上的思路尝试一下

0x5 x2 = xor(x1)
================

这一步最复杂，为何复杂，因为博主的 `unidbg` 没有跑出全的 `x-mini-wua` 所以使用 `unidbg` 不好去分析，这一步也是卡了很久

![](https://mmbiz.qpic.cn/mmbiz_png/MWQJibR3pCZib8bf2GtgtPjUkYiaVs1Aq5gEbpEmqxKA0xkbP8bM1iaQic3gESrBD7w6422Srzica8NibUMYS05fia3PUA/640?wx_fmt=png)

这里就是最后 `aes encrypt` 加密的数据

![](https://mmbiz.qpic.cn/mmbiz_png/MWQJibR3pCZib8bf2GtgtPjUkYiaVs1Aq5gd6l9ib8Nqg6TXa1ia9Xy2Qm0ibMEhnZ0HgIkiciclY2MwPtqNtCoNvtd34g/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/MWQJibR3pCZib8bf2GtgtPjUkYiaVs1Aq5gibFicP0vUic7n52dXFiaBpmIR5UQibEHP1IZxU5FcrmK9ibu24Cd69ytRiatw/640?wx_fmt=png)

博主在进行 `trace` 的过程，发现这些字节是在 `n` 多处 `write` 的

而且 `so` 还看不到 `c` 代码，很难根据 `arm` 分析出数据的计算逻辑，这一步也是卡了很久，具体分析的过程细节就不说了，简单说下思路

*   1、使用 `unidbg` 进行 `hook trace`，根据结果，来猜测尝试
    
*   2、使用 `frida hook` 完成验证，`hook` 出一些关键数据，进行计算
    

0x4 aes decrypt sdfsd
=====================

这一步就比较简单了，没啥可说的

![](https://mmbiz.qpic.cn/mmbiz_png/MWQJibR3pCZib8bf2GtgtPjUkYiaVs1Aq5gxQ37gSVWXv4ibOJD78GEeqDeuOukXyPO5EuRsqbHpUtjLf0Wfzd5rvQ/640?wx_fmt=png)

最后放上一张测试请求成功的图片