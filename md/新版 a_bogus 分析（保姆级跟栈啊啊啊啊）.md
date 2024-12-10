> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/bB7jbTOhejCV36H3j99DsA)

本文导火索：不靠某音吃饭，所以啥时候更新的咱也不知道，刚好有老板滴滴我，说帮看看明天就 v 我 50，那就来吧！！！  

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AI2XXzibgzXEUa2OtRlicSs5ibAVic5mVZG7UTY1r42YhJoacSr6EGFBUdag/640?wx_fmt=png&from=appmsg)

那么接下来进入正题：

目标：视频详情接口：L2F3ZW1lL3YxL3dlYi9hd2VtZS9kZXRhaWwv

直接上 XHR 断点

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AIrfY9yu7uEF3haGia0OyVGncDib959nia8SgNQrgSJGI20knelnlN6fHfA/640?wx_fmt=png&from=appmsg)

还是熟悉的配方，上一篇文章已经知道了 bdms 是 vmp 文件，估计加密还是在这里面生成的，按道理应该在前一个堆栈打断点

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AICfHxPyN2dQ95RmG0myia09iaeefHHIQCxq5O2pMDntbNWTITWqppdWiaQ/640?wx_fmt=png&from=appmsg)

但新版某音已经不管用了，这里按正常逻辑打日志断点：y.openArys 啥啥啥的你只能看到已经加密好的结果，甚至你的每一次刷新页面，这个断点的位置都可能不是 y.send。没错，堆栈会变![](https://res.wx.qq.com/t/wx_fed/we-emoji/res/v1.3.10/assets/newemoji/Boring.png)

但是 vmp 加密也不可能一个断点一个断点去调试，毕竟人家的调用次数可能比我年薪（以块为单位）都多

这里我选择往前继续找合适的堆栈位置下断，再往下一层

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AIRic9pEiaaXoorMRuXGYg5icf7qBpZ7LyUyZ5FU4UrKmPzFyCUT3VKXhgw/640?wx_fmt=png&from=appmsg)

这里有 params，并且这个时候 a_b 还没有加入大部队，先从这里开始下断单步调试。条件断点：e.url === "自己去看 url 是啥"。刷新页面成功断住，咱先单步走，记住上一步执行到哪，以防随时跳转

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AIGGOFiakib6x2PiaP9DnFNEH9D7iaj5YXYX3TictGgvlzeOic2bJP1Lju8zKg/640?wx_fmt=png&from=appmsg)

前面走完并没有出现我们要找的，这里注意 y.send(p)，在这个位置再次单步执行就直接 apply，并且 this 中已经有了 a_b

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AIf5xIdicTzpt4EEV59y6icyDIXytaRxkYNbqFGCxxXibOalNWnrItd7yPQ/640?wx_fmt=png&from=appmsg)

说明 y.send(p) 进入了全新的世界

重新刷新页面，再次断在了前面下断点的地方，这次我们直接在 y.send(p) 下断跳过来，然后进入（F11）。这里需要注意，前面说过了抖音每次刷新都可能走不同逻辑，这里就是其中一个点，有时候 F11 进入会直接进到 vmp 文件，当然不是说不行，这也可以是个入口，只是没那么好梳理。我进入的位置是这样：

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AIWLdTZicV9Kw4fiajyvkvyBH4S4V38dcyj4cg1SGcWibWyqQaY5DFzyaVw/640?wx_fmt=png&from=appmsg)

我们继续单步执行，然后你就会发现，在执行完 t.xhrAsyncSend ? Promise.resolve().then(n).catch((function() 这行代码之后又直接跳到了 apply 结果已经生成的位置，说明这里又进了某个关键逻辑。再次刷新页面重复前面的步骤，这一次我们在 t.xhrAsyncSend ? Promise.resolve().then(n).catch((function() 的地方进去

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AI4Iw2uPg3KjJ0FLP8Q9rUtUftUM8YxMOib4ibgfB83VfAdLVkrKcicvsgQ/640?wx_fmt=png&from=appmsg)

这就没必要单步了，总共三行代码，后面两行是 delete 不用走了。直接 F11 进去

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AInO8pLjtNzOI8ODole1Qjj0oR8jEmYmXmzb6sXhWFcvWeB3eX3Zg56w/640?wx_fmt=png&from=appmsg)

来，继续单步

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AIRvQFMCotHH48AHzD9qFialq4daZhia0icHAqgv4W7mnXoSDNrrkQzCNag/640?wx_fmt=png&from=appmsg)

到 forEach 这里再单步还是会结束，所以这里 F11 进去。走到这里不得不提一嘴

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AI3I22wesnnnpvMWHHrx59qEqHDcpUqA8unlGpH9AmFFpGUG3vTibX91A/640?wx_fmt=png&from=appmsg)

 t.origin.apply(e, u) 调用的是 vmp 文件，并且传入了没有 a_b 的 url 作为参数。这个 url 肯定是要作为加密值所需参数传入的，所以这里很可疑，感兴趣可疑进去调试一下。我已经走过了就不走了，直接告诉大家结果，这里并不是加密位置，进去再出来 a_b 也没有生成。然后这里面还有多个 t.origin.apply(e, u)，单步执行就能看到。而真正的加密入口在第四次进入 forEach 之后

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AI6FNzxTl1RmraOe25bGkmeiaJQuUG046AicALhHkZoMibdSsAYKQmtVKFQ/640?wx_fmt=png&from=appmsg)

也就是 send 条件中这个，e 中包含了我们所需的所有东西。这里 F11 进去就会正式进入 vmp 文件

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AI62aWRxChOJAicHOjeh2Hy3dgveXotdlpPiclJWoz2iclLQ08whIIxokOw/640?wx_fmt=png&from=appmsg)

进来后这里有个 d 函数，这个函数也是我们一开始打 xhr 断点时堆栈中显示的关键函数，进入 d 就开始没法看了，看花眼的 if else。没办法单步调试了，毕竟前面也说了这玩意执行次数比我年薪都多。那怎么办？vmp 技巧：搜 apply。而在这个函数中 apply 只有两个，并且只有一个是函数调用

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AIq8iaDDtJBb3ruV5plN8xibj6Zibqnt0ZGebrdmnuBl08PiaYUPzXCygh9Q/640?wx_fmt=png&from=appmsg)

这个也也是我们 xhr 断点显示的一处关键堆栈。终于可以开始日志断点了家人们。怎么打日志断点不用教了吧？我们把 mnde 都打印一下。

下好断点后先别打钩，不然一会日志太多容易卡死。选择前面的 t.origin.apply(e, u) 下条件断点（其它断点可以删掉了）。只保留这个，以及日志断点跟 xhr 断点。并且日志断点不打钩。

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AIYOavZRH3gk1yic5FphCmniaxMofqEF2zzpMLNFbKrcH45eyoRe3GYYWg/640?wx_fmt=png&from=appmsg)

断住后再勾选日志断点，然后等走到 xhr 断点再取消日志断点。

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AIFb5d4IdDKEPr1icniadLWjeicT28EVsENoQqvCcicpRMUEt2LvJ9iac1KHQ/640?wx_fmt=png&from=appmsg)

然后看日志，可以看到 a_b 确实是这个位置出的，并且是由 m 值一个字符一个字符拼接的。不过这不是重点，毕竟我们不搞（搞不来）纯算，补环境不需要在意这个。只需要知道 a_b 在运行到某个环节的时候赋值给了 e，而我们要做的就是给他加密所需参数，再拿回这个值就可以了。最好的位置当然就是在进入 vmp 文件的前一步，也就是 t.origin.apply(e, u)。

vmp 文件全部拷出，导出 ab

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AITBL1yEgN3Sa3LZF8kSHWP9G9hKysRkGTVWaIswPLMCxoKDlbAmULGw/640?wx_fmt=png&from=appmsg)

怎么导出也教了哦

然后写程序主入口让其运行。

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AIuPEiaRh8v8ap1EVIEibAoWYP6UzsdKjglt3RTCMjtc2HGBibYkW20PHtw/640?wx_fmt=png&from=appmsg)

再就是补上所需环境了，补完就可以调用啦

![](https://mmbiz.qpic.cn/mmbiz_png/BKF2CicIdvGZO13lJzMfiaV8Cjv5Izia2AIBF990krPmv7iaXHpTPNKQQR2cbicXj8pZC0v2BcfJ1WKNgfejKLzv28A/640?wx_fmt=png&from=appmsg)

顺利拿到结果

再提醒两点：1、日志断点的流程非常重要，补完环境记得浏览器联调，对比一下自己的跟网站的流程是否大差不差，如果出不来结果，那大概率就是环境缺失，那就认真去参照日志断点；2、拿到 ab 值后程序会报错 throw l，毕竟我们没法真的去 send，这里要么在导出 ab 后直接 process.exit(1) 结束进程，但是这样跟 python 交互比较麻烦不建议，要么就是在报错的位置 try catch 一下。

嗯，就这样。收工！

如果有问题私聊没回复，那可能就是时间太久回复不了了，不急的可以在文章下面留言哈