> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/tVa_35pt0Knp4KfgxN9lVw)

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichhZsQrsibXscNj4oTNvvgXCTg572LJk2FT1xjHHzpfhTzp9kanGibO8Mw/640?wx_fmt=jpeg)

本文为看雪论坛优秀文章  

看雪论坛作者 ID：沧桑的小白鼠

本人菜鸟，初入逆向，前段时间把 sig 算法通过 ida 阅读伪代码和 arm 汇编进行了还原，花了 4 天时间，殊不知已有大神已经发有分析帖子了 

_（link：https://bbs.pediy.com/thread-254328.htm）_

一、前言  

目标 APP 的 sig3 算法前面已经做足了功夫，这里只陈述一下我是如何找到 sig3 算法的，并没有把算法还原，因为 so 层 onload 有大量 ollvm，目前还处于学习阶段，还没有触及，日后学成在继续发帖分享下学习笔记。

二、定位入口  

本次目标 APP 版本是 7.2.4.13202，这里使用 Charles 抓包知道了 sig3 的全称：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMich5qwIK7fx9qnacKEdNQiabx151GPibn3Y9mnwnicicm96DiaPlrUxsTkBoPw/640?wx_fmt=jpeg)  

sig3 全称是 _NS_sig3 ，这里将 app 拖入 jadx-gui 进行分析，使用字符串搜索大法进行初步查找：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichjQ1wpFq9yfC1fwrtZ55VdMEzkRnxO4QNJFWDPxl0afTPImu7GXALFg/640?wx_fmt=jpeg)  

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichLiba8ZxiczSJNKB7RhqswDSiaiaSSSUPxrhj8XNS3nfE0gHo9iaYJOGbRhA/640?wx_fmt=jpeg)  

通过框选区域发现，a3 是来自 a2.a 方法，且 str2 通过上方代码得知是 sig 签名结果，b2 是一个 url，但是执行了 b() 可能进行了一些处理，这里暂时先不管 b2 数据是什么，先最终到最终 a 调用的地方。

三、算法最终定位  

String a3 = a2.a(b2 + str2);  跟进 a 方法的声明：  

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichdh0FA7OPfsrDpggXeBCX3Q6hKpNyrDeuy55xGPc9ac4Eibsm5x2L1vg/640?wx_fmt=jpeg)

 通过查看 a 的方法声明，发现是一个 interface，这里直接右键查看调用例：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichgZKoOPwHdU2S74lOcoFWsotiaoHfeoqpIQTzXGfIzKhVEOFiaGib4FMxQ/640?wx_fmt=jpeg)  

发现只有一处地方使用了 implements，进去查看具体代码实现：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichCu3xltu3YqLdVJYR8IQDk0SbVj0C5UibJXXy79vktGJM9230icUK4uHg/640?wx_fmt=jpeg)  

继续跟进 getSigWrapper 方法实现：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichhygx6T3enDZa8aicLvLgNnRWItKNgja1MKlYr5kuEMDdwnKCnnCwBkQ/640?wx_fmt=jpeg)  

继续跟进 j.c(str) 方法实现：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMicho8zTRYrhABv0xVuGp776Lka82q4QzXyiaWqq6p3gmsWe2RwdgO8xGtA/640?wx_fmt=jpeg)  

发现还在继续深入调用  String atlasSign = KSecurity.atlasSign(str);，继续跟进方法声明。

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichqYfa6sCaJIPkWpldwWuPVsqLe2cfsRrZFYIS454gCwiaeeSuJfHicwnQ/640?wx_fmt=jpeg)  

继续阅读下代码，我们可以直接双击 str 变量，查看他引用地方：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichN1RIx5dPY1sgnGqjQU0yBbv2sSOP4p2E6BU0B9Wo2EJIhS33268uLA/640?wx_fmt=jpeg)  

发现是执行 d().a(str) ，继续跟进查看实现方法：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichSjsTfJJR2dMXul5PRZ0kYwdaepzV8eOrcfupH5oFZlKia3xjuInIP9Q/640?wx_fmt=jpeg)  

这里传了个参数 2，false，继续跟进。

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichx0w8w0opXkrOeK4iaszd4ia7icuy9FwvjzCDg14hBpXn23J1hMKuicCzHw/640?wx_fmt=jpeg)  

这里我们看到，他 new 了一个 i()，并且一直在里面赋值，通过简单的课文阅读理解，我们去掉无用的代码：

```
i iVar = new i();
iVar.a(KSecurity.getkSecurityParameterContext().getAppkey());
iVar.a((Map<String, String>) null);
iVar.b(0);
iVar.a(z2);
iVar.b(str.getBytes("UTF-8"));
if (i.a(this.a).b().a(iVar, "0335") && iVar.j() != null) {
```

最终发现是调用了 i.a(this.a).b().a(iVar, "0335") 实现，继续跟进查看：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichVza0icQqtIeicMjgnvKFgslwFckstOHVMUDgeRFoyJSADkSWAibseicKDA/640?wx_fmt=jpeg)  

发现是 interface，继续右键查看用例。

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichM6XicCXSfodkMvONJxApCXVOicyXayicXkicUxvZRk19FJeWEAXT9iamicyw/640?wx_fmt=jpeg)  

找到引用的地方，双击找到具体实现 a(i,string) 的方法：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichvKVdZCib6PhEUibJXH2QiaWPaDtcW5Es5YNlZFsbbWdqZqiabEfx3gXl3g/640?wx_fmt=jpeg)  

我们继续进行简单的语文阅读理解。

```
String trim = new String(iVar.i()).trim();
String str2 = (String) this.a.getRouter().a(10405, new String[]{trim}, KSecurity.getkSecurityParameterContext().getAppkey(), -1, false, KSecurity.getkSecurityParameterContext().getContext(), null, Boolean.valueOf(iVar.b()));
iVar.c(str2.getBytes("UTF-8"));
```

我们是否还记得，他需要算法的参数都放在了 iVar，这里他使用 iVar.i() 将要签名的信息拿出来，紧接着调用 this.a.getRouter().a 获得结果，发现 a 这里传了很多东西，估计是最终算法，我们先继续跟一下 a 方法。

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichOPHibfPKsS3OtyQZHs6nR4lGatbMLrnUQDfCzVC1lKU735DqmVDuT8w/640?wx_fmt=jpeg)  

好家伙，又是 interface，继续右键，我们稳住。

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMich5eF07wetXNDs5Sk3hJEhWS5KZq3dqDcZPqrBeuSI4vibN1DahRQOSrg/640?wx_fmt=jpeg)  

继续跟进查看实现：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichdXib9XZiaCfX5IMBhzIXf6BubtMEicYeHMtym7mTu7kjKvObGRHEtVEPw/640?wx_fmt=jpeg)  

return JNICLibrary.doCommandNative(i, objArr);  我们发现，最终是传给了 navite 进行了计算。

那么我们继续 hook 下他都传了什么信息，这里我选择的 hook 点是：

```
com.kuaishou.android.security.a.a.a.a(com.kuaishou.android.security.kfree.c.i,java.lang.String)
```

因为最后面这 2 个传的东西太多，而且很多都是固定值，懒得码这么多代码。

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichQicGTePgx5ialSDRiczQ455B1EB1fZYibYNmfdyP5iaptIQKcxuqTPY7wVw/640?wx_fmt=jpeg)  

编写好的 frida 代码，先将手机的 frida 运行起来，具体 frida 的搭建文章我们可以阅读： 

_https://zhuanlan.kanxue.com/article-350.htm_

_https://www.jianshu.com/p/dadacd02fefd_

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichP8vGuYAic64daQLoPtLia5Ep4hpUgRxW0kSt3RaO6Wx1jzSu5jlm1YDA/640?wx_fmt=jpeg)  

我们看到传的参数是域名的路径和 sig（我们第一次看到他是 str+str2 的合并，原来 str 是域名路径）

这里我们回到 a 方法，查看他最终传给 navite 的参数：

参数一：10405

参数二：new String[]{trim}

参数三: KSecurity.getkSecurityParameterContext().getAppkey()

参数四：-1

参数五：false

参数六：KSecurity.getkSecurityParameterContext().getContext()

参数七：null

参数八：Boolean.valueOf(iVar.b())

参数二就是 hook 打印的内容。

参数三是一个固定的 appkey，通过查看 appkey 的调用例找到了

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichAJcP4tALsD9lQPhLcTJRJPaDcUpb06kKRoD1yMaJIQPvicQeS1Pslvg/640?wx_fmt=jpeg)  

这里因为文章长度问题，就不具体展示细节。

参数八：就是一开始传的 false 逻辑变量，这里我们可以回到一开始传的时候查看他的赋值就知道了，这里的 b() 只是获取当时赋值的结果：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichCwEOhmCPhjky4yXDnBFKnhmXDy8LABIB1dGsab7cR923PEo8VZnszA/640?wx_fmt=jpeg)  

参数全部找到了，由于 so 层看到了大量 ollvm，我就没有进一步还原，只能通过 hook 方式主动调用了。

四、so 文件定位  

因为看到这个 so 的加载挺难找的，这里也记录下我的查找过程。我们进入 JNICLibrary.doCommandNative(i, objArr); 的定义发现并没有找到加载 so 文件。

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichR2uHqQXKHRKNyCEhnxRwUJks5GynTrgsNmSia4ujP30FxTW3uuXh3SA/640?wx_fmt=jpeg)  
![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichYichdnndJN2TgDpBwkGhl1yIgY7oNTRhjZ56YbknkpOq5BMlAKGtvibA/640?wx_fmt=jpeg)  

这里我选择了右键查看 b 的调用例，发现有一处 new 的地方：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMich49fpVMu3QqJnywXoxZsayaR1FDdxFoqmB1suoBia9JCnXJQ6rsog9eA/640?wx_fmt=jpeg)  
  

我们跟进去看看。

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichznRhy2l5arDvj4UoGqrL3mrx3b1sEy3fbQsdjCCZcr6ZKjMz7tzxwg/640?wx_fmt=jpeg)  
  

通过种种代码的文字提示，发现他是在这里调用  boolean a = f.a(context); 进行加载，貌似这个是在提示加载消耗的时间。继续跟进 f.a() 查看：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMich6sb4NaGdENc07v0dia4Yt2PKAJ5yZLlbwP5JPaI9ZrDA9rcRo4PmqCA/640?wx_fmt=jpeg)  
  

发现看到了一个名字叫：kwsgmain，加上底部提示的 retry load so ok，怀疑这个就是 so 文件名字，查看 lib 发现确实存在该文件。

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichqpczvaofO3J7FzIiaOaSK5tLHx2DZoYwGF3FFf71Nr3fA09ADlDPdxg/640?wx_fmt=jpeg)  
通过后续的继续跟进这个字符串的调用，最终发现确实是这个文件：

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichn2YUIApV9L1EbgxAWYuy9ahIOVHgLJicELhs5BBqNSufuOxmWjW5tjg/640?wx_fmt=jpeg)  

但是拖进 ida 看到 onload 那一刻起，我目前先放弃了。先继续啃课本。

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMich9cV4Fiab6jIdS8fmmlBv0rUwiab0oxaxGyWCFZ5XnWu8ThbicpdibllQPQ/640?wx_fmt=jpeg)  

![](https://mmbiz.qpic.cn/mmbiz_gif/b96CibCt70iaa8r7PJoyAtlfHAKe8RosE3wYVKBac55p1HPBJHZS42ywnG4yYtD3jo9A9e5kawBZs4IE6R1C4wibw/640?)  

- End -

![](https://mmbiz.qpic.cn/sz_mmbiz_png/1UG7KPNHN8EWpplp2mMHeJ9b4YUxdMichU6LNg0sy3HhiaXO94JjrJe60ibz6G7zTPFq9kZ92fzCCicJvNUTf8Td6A/640?wx_fmt=png)  

  

  

**看雪 ID：****沧桑的小白鼠**

https://bbs.pediy.com/user-885524.htm 

* 这里由看雪论坛 沧桑的小白鼠 原创，转载请注明来自看雪社区。

推荐文章 ++++

![](https://mmbiz.qpic.cn/mmbiz_jpg/US10Gcd0tQFGib3mCxJr4oMx1yp1ExzTETemWvK6Zkd7tVl23CVBppz63sRECqYNkQsonScb65VaG9yU2YJibxNA/640?)

[*](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458298320&idx=3&sn=cd7d24d6c8c2ebe22be92bcea5656e61&chksm=b181975a86f61e4c05c9f6816f7df28ca0c77576499e78e87aa68e51090752a298b38f5ae82e&scene=21#wechat_redirect) [手把手教你入门 V8 突破利用](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458304446&idx=1&sn=17a437cae897baea913277aad4ce4d9a&chksm=b1818f3486f60622b14a134d2c7fe3e45855e2f810668f438f02c420dcd39607d077cd5a73d0&scene=21#wechat_redirect)

[*](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458298312&idx=2&sn=52e40cb171b6699505f079f96b53e973&chksm=b181974286f61e54c425e5ea29fe706b830898abbee1293d89d4c9786c3596c2688cca240083&scene=21#wechat_redirect) [Android 微信逆向 - 实现发朋友圈动态](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458304420&idx=1&sn=022001e527e8613765e804d1b6cac57e&chksm=b1818f2e86f60638dc54e63fca554a2f0675cd32144adad912159f3cafbf1ce799aec69bfbfe&scene=21#wechat_redirect)  

[*](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458298985&idx=1&sn=ba40d13b0a84d1ca744f4ef2067da81f&chksm=b1819ae386f613f581ce1fecb7a71db0ddd14170480124681fa01330ed05e3e6103999bb0207&scene=21#wechat_redirect) [病毒样本半感染型分析的方法](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458304388&idx=1&sn=e94b84465f5a237bcdd99e027d98945c&chksm=b1818f0e86f60618431bed7d91fead3b49b64a51796027895fc91898ded8df475e5c7b9f6d4e&scene=21#wechat_redirect)  

[*](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458298286&idx=2&sn=3735d8e22374ae73839c96806464d54f&chksm=b181972486f61e32c8a39b467842ccff53cf65724ed40a65df8730e5a8969e2a2dd04388519a&scene=21#wechat_redirect) [对宝马车载 apps 协议的逆向分析研究](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458303493&idx=1&sn=072a250e780ab340aac1a571ad713329&chksm=b1818c8f86f60599d34eb4b7e7867ebfc78d15d35180d91c33dfdb91a6226f1abfbdedd1deff&scene=21#wechat_redirect)

*  [x86_64 架构下的函数调用及栈帧原理](http://mp.weixin.qq.com/s?__biz=MjM5NTc2MDYxMw==&mid=2458303473&idx=1&sn=fe368c6bec6190ef59be725b163663d8&chksm=b1818b7b86f6026d1d45679948ffc52642a4fdb7fc83f9f3c54258efb29483602ec667c86ea7&scene=21#wechat_redirect)

好书推荐![](https://mmbiz.qpic.cn/mmbiz_png/b96CibCt70iaajvl7fD4ZCicMcjhXMp1v6UQQ68afWhJytuHspOcDRtNqnosZfRiaqD9E6ZQs5jaeMyw9vTrDd3DTA/640?)  

﹀  

﹀

﹀

![](https://mmbiz.qpic.cn/mmbiz_jpg/Uia4617poZXP96fGaMPXib13V1bJ52yHq9ycD9Zv3WhiaRb2rKV6wghrNa4VyFR2wibBVNfZt3M5IuUiauQGHvxhQrA/640?)

公众号 ID：ikanxue  

官方微博：看雪安全

商务合作：wsc@kanxue.com

![](https://mmbiz.qpic.cn/mmbiz_gif/1UG7KPNHN8FxuBNT7e2ZEfQZgBuH2GkFjvK4tzErD5Q56kwaEL0N099icLfx1ZvVvqzcRG3oMtIXqUz5T9HYKicA/640?wx_fmt=gif)

戳 “阅读原文” 一起来充电吧！