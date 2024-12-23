> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.1997.pro](https://www.1997.pro/archives/1728550214689)

> 国庆前抖音罗盘 a_bogus 突然更到了 1.0.1.15 版本，拿去年的短 ab 跑间歇性给数据，当时由于短 ab 断断续续的还能跑，假期在即就没管它了，昨天突然就 g 了，只能被迫开干。

国庆前抖音罗盘 a_bogus 突然更到了 1.0.1.15 版本，拿去年的短 ab 跑间歇性给数据，当时由于短 ab 断断续续的还能跑，假期在即就没管它了，昨天突然就 g 了，只能被迫开干。

由于只做商家后台不做 dy 前台，所以中间的长 ab 版本都没跟过，从星球找了一份 1.0.1.5 的纯算，对照着开始改。

[![](https://www.1997.pro/upload/image-eufk.png)](https://www.1997.pro/upload/image-eufk.png)

插桩简单看了下后，发现和 1.0.1.5 整体流程貌似区别不大，而且整个流程日志只有四万多行，没必要费劲反编译，直接在 1.0.1.5 的基础上开改。

初始化时间和加密数组
----------

[![](https://www.1997.pro/upload/image-czak.png)](https://www.1997.pro/upload/image-czak.png)

url 参数处理
--------

[![](https://www.1997.pro/upload/image-rydq.png)](https://www.1997.pro/upload/image-rydq.png)

这里算法从短 ab 到 1.5 长 ab 再到 1.15 都没改，加盐 sm3 哈希，变的只有 salt

请求 body 处理
----------

[![](https://www.1997.pro/upload/image-gyme.png)](https://www.1997.pro/upload/image-gyme.png)

同样的加盐哈希，get 方法的话就是空字符串加盐哈希

ua 处理
-----

短 ab 和 1.5 都是标准 RC4，但是 1.15 可以明显看到进行了魔改：

[![](https://www.1997.pro/upload/image-uwxv.png)](https://www.1997.pro/upload/image-uwxv.png)

S 数组变成了 255 254 ... 0

[![](https://www.1997.pro/upload/image-ylpk.png)](https://www.1997.pro/upload/image-ylpk.png)

往下看，密钥生成部分出现了乘法，明显进行了魔改，不过逻辑很清晰：

[![](https://www.1997.pro/upload/image-zhlu.png)](https://www.1997.pro/upload/image-zhlu.png)

加密部分还是 RC4 的原生 xor，实际上各种加密算法的魔改基本都是在密钥轮进行的，很少有魔改加密轮的，因为没能力 / 不安全。

RC4 之后的魔改 base64 还是和之前一样，没改变：

[![](https://www.1997.pro/upload/image-xzis.png)](https://www.1997.pro/upload/image-xzis.png)

[![](https://www.1997.pro/upload/image-pryj.png)](https://www.1997.pro/upload/image-pryj.png)

后续又是一次 sm3

版本号处理
-----

[![](https://www.1997.pro/upload/image-imzz.png)](https://www.1997.pro/upload/image-imzz.png)

直接 split 后数字化。

短数组生成
-----

对着日志撸，逻辑很清晰

[![](https://www.1997.pro/upload/image-lplu.png)](https://www.1997.pro/upload/image-lplu.png)

[![](https://www.1997.pro/upload/image-bzgn.png)](https://www.1997.pro/upload/image-bzgn.png)

[![](https://www.1997.pro/upload/image-omvt.png)](https://www.1997.pro/upload/image-omvt.png)

随机数生成算法
-------

[![](https://www.1997.pro/upload/image-ojyb.png)](https://www.1997.pro/upload/image-ojyb.png)

这一部分短 ab 没有，直接复制大佬的 1.5 算法。

两个位置用到，分别是：

和

传参其实并不都是 Math.random() * 65535，但是能用就懒得改。

短数组转长数组
-------

先是 xor 生成校验位，然后拼接生成短数组 n[88]。

[![](https://www.1997.pro/upload/image-pnkf.png)](https://www.1997.pro/upload/image-pnkf.png)

接着一顿操作把 94 位数组变成了 125 位数组。

这里询问一个搞完的大佬几个数字是否是常量，结果大佬直接把这部分代码发我了 = =

[![](https://www.1997.pro/upload/image-syfe.png)](https://www.1997.pro/upload/image-syfe.png)

不过大佬发的是 1.0.1.17 版本抖音短视频的 sdk，我改了下后发现死活过不去，还以为是 1.17 在 1.15 基础上又变了，最后调试后发现原来是弱智 GPT 提取代码时搞错了几个运算符，GPT 给我拉了一坨大的......

生成 ab
-----

[![](https://www.1997.pro/upload/image-yeot.png)](https://www.1997.pro/upload/image-yeot.png)版本号数组拼接上面的 125 数组后 RC4，然后另外两个常量生成的字符串再拼接，最后同样的魔改 B64，得到的 a[93] 就是最终 ab。

经测试 1.0.1.15 可以通过 dy 1.0.1.17 的签名认证。

扩展
--

上面的整个流程除了 arr94_to_arr125 外，其它的流程和 1.5 流程变化不大，硬说变化就是数组位置、加密方法的改变。

这里看到 arr94_to_arr125 时候就很好奇，上面的其它加密都是魔改 rc4 sm3 b64 的，而这个通过随机数算法加密生成的新数组，在抖音服务器那边如何解密还原呢？问了 GPT，回答又是答非所问的一坨，于是问了下密码学大佬，发现这里的常量数组 [145, 110, 66, 189, 44, 211] 选的很有深意，因为它们的二进制为：

每一组的两个都是互补的，通过位运算将加密后，完全可以通过结果的四个数字逆推得到原始数组，消除中间随机数带来的影响。

神奇的密码学~