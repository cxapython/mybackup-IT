> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458379268&idx=1&sn=4a6a0dd7491cde7cbf226e8552056947&chksm=b180d48e86f75d98defc719ac21f3bacf8e6e5fd6d2b87f0d89ae19a753d1091bfa045c29924#rd)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDrxP8MjqpAf6IuRicVdhymnuSJkPLhNtfJon7NLp1FM4llZbtYXgCLxQ/640?wx_fmt=png)

本文为看雪论优秀文章  

看雪论坛作者 ID：Avacci

该题源自看雪高研 3W 班 9 月第三题。

目标 app 只有一个很朴素的界面。点击 “CHECK” 按钮会在下方不断打印加密后的字符串。目标就是分析整个加密流程。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDbzTJu7iadHFD5u3tCD5oFGMh8KMPLCytVXVlqMMHQp8gCfYMicKagTXA/640?wx_fmt=png)

由于 Yang 神手下留情，整个加密流程其实并不复杂，但是由于使用了 ollvm 的字符串混淆及控制流平坦化，为静态分析增加了不少难度。这种时候就要善于借助各种其他工具和技术来加快分析过程。

很多时候，我们并不需要弄清一个变量的值在运行时被解密的具体细节，可以直接在运行时 hook 出其解密后的值即可。也往往不需要分析明白一个关键加密算法函数中的每个子函数的作用，如果发现了某种常用算法的特征（不一定要完全匹配，可能是变种），就应该大胆猜测并及时做重放测试（CyberChef 真的很方便）。毕竟猜错了没什么损失，猜对了可能挽回几年阳寿。

先将 apk 拖入 GDA 中，定位到点击按钮的处理函数：  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDramV2YqkYqiaWZKm5u2aL32cEMO0vKuMmmBhehaCcGGC5Vx2EmLj9yA/640?wx_fmt=png)

uUIDCheckS0 为点击 CHECK 按钮后生成的值，生成算法包含在函数 UUIDCheckSum 中，传入参数是由 RandomStringUtils.randomAlphanumeric(36) 生成的一个 36 个字符的随机字符串。  

可以利用 Log 确定每次调用 UUIDCheckSum 的入参和返回值。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDFaVo6DbLCDPxsSZvicIvOCXTwEbE0UcdDuCYibE6ZBEicwPs1uDECssSw/640?wx_fmt=png)

由于 UUIDCheckSum 是 native 函数，其实现在 so 中。将 apk 解压后把唯一的 so 文件拖入 ida。  

在函数窗口搜索，定位到 UUIDCheckSum 具体实现。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDQicjXzhsYcOydkj0zuEWz5ibd9ibibgznIIVVGZUpibt5ESML245mm7adDw/640?wx_fmt=png)

本来想先定位返回值再回溯，结果发现 ida f5 识别出来的方法没有返回值。猜测函数没有被正确识别，比如函数尾部被截断了之类的。  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDrsibiby6kVfcKScuxT6vqrkwN10w1rialhUoXLFDkKNdsVUebRTtzsF9w/640?wx_fmt=png)

回到汇编代码，查看并定位到函数尾部。  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSD9nWcly4tibHGH87Zbl2ibzRJicPY5WAUQfcvbribsFdbSuPkfQ287y3MJA/640?wx_fmt=png)

结果发现函数尾部接着的是一段. datadiv_decode 开头的代码，之前学习视频里有提到这是一段解密用的代码段。  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDuicFgkmRaw97lwrWA060ARqyDvqz2v2CtPLcM6riaKmPl3WVNDTkSfpA/640?wx_fmt=png)

看了下调用位置，在. init_array 中：  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDbeT1JUpYBice6jLOYQ4qouexavCNEOHqNicWCKy09CBrShPaiacb6Vd8Q/640?wx_fmt=png)

观察了一下该段代码反汇编后的结果，  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSD5vCvkCL26dfk4duuSicNOY6XRVXm3lAIhXDGyUJeytxWOlBwuib9tjUA/640?wx_fmt=png)

看起来是对几个字符串解密，比如 stru_37010，byte_37090, byte_370A0, stru_370B0。可以用 frida hook 得到解密后的值，看看哪些比较有用。  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDSx2hMAXrgRXMlr7azM2sUtJ6TJVrC9uJWia5gKWibKVlbBeFXibibBGMIw/640?wx_fmt=png)

比较感兴趣的是 stru_37010 解密后字符串:  

0123456789-_abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ

看到这串排列整齐的字符序列，首先想到会不会是用到了某种 base64 编码。先去定位下用到这串字符串的位置。  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDeX7CAHasbQ0bLBvTtAZyGGRzQ6UUvczpHibHV1AXqICMNPchAyfTiawQ/640?wx_fmt=png)

两处定位看反编译的结果都比较诡异，可能是因为加密的值被优化了。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDTDYV8kJOUhbEwSbSVsZ7r8K39t7ibCficBrpZrkSbcyRhL6oS1UiaF4ibA/640?wx_fmt=png)

但看汇编代码，确实是在这里调用的。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDUArfRMFhpqLCaTS7JUciaAyLiaX88Qrt6nRvQwCc3DSPjmc8Jt1ZLRaw/640?wx_fmt=png)根据调用栈逐层往上找，发现是在 UUIDCheckSum 中的 sub_F9B8 方法里用到。

那么，猜测一下，这个 sub_F9B8 是以  
0123456789-_abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ  
为编码表的 base64 加密函数？需要验证一下

我先用 frida 写了两个 hook 函数，一个 hook 了 sub_F9B8 函数，还有一个 hook 在 sub_F9B8 之前调用的 sub_FCB4 函数。因为 sub_FCB4 以之前生成的随机数作为输入，很可能 sub_FCB4 会用到处理后的结果。  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDxeiaUaKSTdzel77ia9u3XW6YiacX90a0qKv6FlxVpRwytiarrDPBCn7iclg/640?wx_fmt=png)

Hook 函数的实现：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDrxP8MjqpAf6IuRicVdhymnuSJkPLhNtfJon7NLp1FM4llZbtYXgCLxQ/640?wx_fmt=png)Hook 上后点击按钮，得到了结果。

生成的随机数：  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDwicKLvWAoIWXgJXWMv9Jdqm5NzRcBqeEic7jdUxCfWqNFc4pNwsRwicWA/640?wx_fmt=png)

经过 sub_FCB4 处理后的中间结果：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSD5Hau05AEiaDXTxZqWu5xXYsg6qDpgdOW1DMhldqE7nyEwqAicvT16uiag/640?wx_fmt=png)

sub_F9B8 的第二个传入参数：  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDjWw0cxH8kHknga4ENicbRhEXKc7Hko3mAIEzdhnjRUL6mhWy4ad0rdA/640?wx_fmt=png)打印出的最终结果（Logcat）

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDNcia6ad5edtBicUUen8icyiaxBAK8Fz324iaVTOmhVC8DvvXFjdmNfeqhTg/640?wx_fmt=png)

到 CyberChef 上验证猜想。由于猜测 sub_F9B8 是 base64，所以将中间值  
2n5ixhQ{-oLqe-4nEi-Xr3dv-u6Rq4uo`ga3 作为输入，设定下编码表。发现结果果然符合最终结果。  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDAeKmv7d4CQasXicUCYE8rFj8qOVeWmBRPV4PxRfYibSVHK3hNt7iajPbg/640?wx_fmt=png)

运气不错，验证成功！

接下来就只要研究 sub_FCB4 就行了。粗略地观察了一下反汇编后的代码，是对传入的随机字符串进行逐字节处理。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDbAEEibynYHzsMChocao2JnRCGXJCJ62Q5w9X9j7hqVsPFxLia2KUvdxg/640?wx_fmt=png)

先取一次运行的输入输出值，放在 010editor 中做下参照。

输入：  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDMskJVdf8C5hic7DUeIeO5p2Bw3AFP8Z8rCdOUxzPZBNdOdEvUZR44tQ/640?wx_fmt=png)

输出：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDmo3l31Rv8icYkZPiaNcbAGricr1GRMWhR95ibteVftqEm3YIicD8Hgy6whw/640?wx_fmt=png)

由于函数不太复杂，打算先静态分析下。首先：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDUXuiamNibsIXiaKtqmNknUAZY3skAFiaolhvr0V7jcej66krVpSsaoGgsg/640?wx_fmt=png) 

输出的第 23 个字节是输入的第 24 个字节，然后定位到代码。  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDrGQ8rt2JYibzL9B1ElzLPfqOicYjSO9gKWG5XvQBnbsbqK0w2JB5GseA/640?wx_fmt=png)

如果 index 不为 8、13、14、18、24 则结果就是原字节异或 1（除最后两个字节外）。

如果是 14：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDNOLa9EB1lLrmxtUiaGIhQRJnhWBwb0t48dqcGhibxeB43crDA6342qcw/640?wx_fmt=png)

输出字节为 52。

其他的第 8、13、18、24 字节值都为 45。  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDTqQibuxX9GnicAdM43wZdFCyTVl2lcYLQicvx3MbcFAiawLUCMticoYU30A/640?wx_fmt=png)

还剩最后两个字节，在函数尾部处理。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDHyCbeIfsk6WNOejAn94V3XhDXNvmSvgfGXO7LdkvOnC0We1c4Em9Lw/640?wx_fmt=png)

这里 v14 和 v16 的值不太好看出来，决定用 ida trace 看看。 

在 trace 的 log 中定位到处理最后两个字节的位置。  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDV5EuUOsB75JUMMM2ymPUwMBzj6jSibj2UQENvc7Q13c8PEwdk4P5LTA/640?wx_fmt=png)先看最后一个字节：

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDDoMqroQQAO1jibbHr2L8xIBRQicM3Jg0jVW0aMfgedScnCfuduoEIaaQ/640?wx_fmt=png)

这个字节是通过一个字符串指定偏移处的值得到，偏移值存在 X8 寄存器中。

逐步往前跟踪 X8 寄存器值的来源，定位到：  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDYfh4JUvtJNmhCAHU69OibmtOKhpz7icVTdAe9UvnmtAlDcA09InzSk5Q/640?wx_fmt=png)

也就是是 W20^W21。

W20 是之前处理的输入字符串的最后一个字节。而 W21 来源于 [SP,#0x50+var_44]。这个位置也是异或后结果保存的地方。所以猜测可能之前处理过程中也会不断将输入的字节逐一异或处理。

搜索了一下，果然有很多。  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDkHkPdvicwEYy2KfR9sQSTOdDOGg9nnKwjcu7mhrpeic1UCL5DzWXC4Xg/640?wx_fmt=png)

第一次用 0xFF 异或输入字符串的第一个字节，将结果与第二个字节异或，以此类推。

然后之前那些特殊位置的字节不参与处理。获取最终结果的低四位值。

同理发现倒数第二个字节的来源是将输入的字节逐个相加。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDge8ptjulbWNmgIOC8yHv0Cc4jYxHZNicMNxbicYU31UiaHV0vVwibEibIPA/640?wx_fmt=png)

获取最后的和的低四位值。

至此，分析结束。

![](https://mmbiz.qpic.cn/mmbiz_gif/b96CibCt70iaa8r7PJoyAtlfHAKe8RosE3wYVKBac55p1HPBJHZS42ywnG4yYtD3jo9A9e5kawBZs4IE6R1C4wibw/640?wx_fmt=gif)

- End -  

  

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8FvGmlNic2JthsIpRKaAIOSDO1yEvfUueQbAuZCqAywqqWQ45RpOqiaZPQnWXsoCxnGs2aPf0QSUkuA/640?wx_fmt=png)

  

**看雪 ID：Avacci**

https://bbs.pediy.com/user-home-879855.htm

 * 本文由看雪论坛 Avacci 原创，转载请注明来自看雪社区。

 安卓应用层抓包通杀脚本发布！
---------------

《高研班》2021 年 3 月班正在火热招生中！👇
--------------------------

[![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EXz6dPudYZkBtWllcJ06WrD7fOUJBWDgX7Iwby00JSCYWusgIcEib8yhdATV9yXlrFykZdzeBK2og/640?wx_fmt=jpeg)](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458378436&idx=1&sn=4c52b0fc0f1600a0fb2f988e7190a232&chksm=b180d04e86f759583d9c6b49384de080a900b6c0936a7b75345ee380ded28a4793038e78d1ed&scene=21#wechat_redirect)  

* 戳图片了解详情  

  

**#** **往期推荐**

*   [一个缓冲区溢出复现笔记](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458379185&idx=1&sn=ff36aa1ab3e28a75d92e450d3f0703a4&chksm=b180d33b86f75a2da6f37f8d78ce49da477367aa228b06955116a0774e6b6806b725882c1236&scene=21#wechat_redirect)
    
*   [Sandboxie 循序渐进](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458378917&idx=1&sn=533008b3fc3dd17add05e5e00c3a8668&chksm=b180d22f86f75b3900410e401c1b9fbb512164a7cacb2f95c395643cb65bb7f1dcee1518795b&scene=21#wechat_redirect)
    
*   [【逆向解密】WannaRen 加密文件的解密方法](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458378868&idx=1&sn=495643d2a8b0167a290737a90236475f&chksm=b180d2fe86f75be8f3cb0be95e76731b13b9052fd06ac6e08250f14b9a30114da95710e1826f&scene=21#wechat_redirect)
    
*   [](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458378868&idx=1&sn=495643d2a8b0167a290737a90236475f&chksm=b180d2fe86f75be8f3cb0be95e76731b13b9052fd06ac6e08250f14b9a30114da95710e1826f&scene=21#wechat_redirect)[使用 unicorn 来 trace 还原 ollvm 混淆的非标准算法](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458378834&idx=1&sn=da5c20cd044273714f16a833bb4cf031&chksm=b180d2d886f75bced123ea7262edeea620c5282c5953f7444ef094bcccda04e5704a194f3f74&scene=21#wechat_redirect)  
    
*   [fart 的理解和分析过程](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458378618&idx=1&sn=bd6aab7728e0a440643b02996fe9c9a7&chksm=b180d1f086f758e6cdd5a9de680dc2aa095c862494e4349cfede9f26e7ca33a5a3b20cb7a682&scene=21#wechat_redirect)
    

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_jpg/Uia4617poZXP96fGaMPXib13V1bJ52yHq9ycD9Zv3WhiaRb2rKV6wghrNa4VyFR2wibBVNfZt3M5IuUiauQGHvxhQrA/640?wx_fmt=jpeg)

公众号 ID：ikanxue  

官方微博：看雪安全

商务合作：wsc@kanxue.com

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/1UG7KPNHN8EbEJaHl4j4oA4ejnuzPAicdP7bNEwt8Ew5l2fRJxWETW07MNo7TW5xnw60R9WSwicicxtkCEFicpAlQg/640?wx_fmt=gif)

**球分享**

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/1UG7KPNHN8EbEJaHl4j4oA4ejnuzPAicdP7bNEwt8Ew5l2fRJxWETW07MNo7TW5xnw60R9WSwicicxtkCEFicpAlQg/640?wx_fmt=gif)

**球点赞**

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/1UG7KPNHN8EbEJaHl4j4oA4ejnuzPAicdP7bNEwt8Ew5l2fRJxWETW07MNo7TW5xnw60R9WSwicicxtkCEFicpAlQg/640?wx_fmt=gif)

**球在看**

![图片](https://mmbiz.qpic.cn/sz_mmbiz_gif/1UG7KPNHN8EbEJaHl4j4oA4ejnuzPAicd7icG69uHMQX9DaOnSPpTgamYf9cLw1XbJLEGr5Eic62BdV6TRKCjWVSQ/640?wx_fmt=gif)

点击 “阅读原文”，了解更多！