> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/mUGlYInAHhIY_yLzTo8ENQ)

  

  

  

  

  

        收到来信甚感欣慰，正好最近有很多人问这个问题，系统的写一下逆向过程，希望能够帮助到大家。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9dKyPsW1te6rR5Kdpls3vkx2rFWnd9J1kSbbCmOBV5ppExY2r6AH1Vw/640?wx_fmt=png)

  

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG973eJagNDX9z0Peaufkj8DBOUyTnXxCrw2WS2iaWSubhOHSgBQFlsZ9Q/640?wx_fmt=png)

文章目录

     **1、流程分析**

     **2、远程调用**

     **3、本地调用**

 **4、分析结果**

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG973eJagNDX9z0Peaufkj8DBOUyTnXxCrw2WS2iaWSubhOHSgBQFlsZ9Q/640?wx_fmt=png)

Vol.01 流程分析

逆向加密参数的第一步，分析流程。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9CHjvmBd4oWxeia1OQcclPIHFogtv0Hy2gWLsrmzWMPjx4BaE9gy7UCg/640?wx_fmt=png)

通过堆栈信息进入源码中并断点。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9lQCbFxbwRksvJgz1YLUc57FAMibKvMAnT2p84XBcOsYRlL6GM2AiadJQ/640?wx_fmt=png)

> 图中 apply 方法能劫持另外一个对象的方法，继承另外一个对象的属性 apply 方法接收两个参数。

断点之后，可知 t = this  是 XMLHttpRequest 对象，观察请求，当前请求对象的_url 中包含了 signature 和 x-bogus。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9wX4GxRTzMK3QciafyEmzSz9zQU84qzibicrcJgSaoLibc7npMbDBnwKQ7A/640?wx_fmt=png)

那么现在需要找到未带有 signature 和 x-bogus 的请求对象。

在 e.nativeXMLHttpRequestSend 时往前调试 7 步左右，发现一处和 XMLHttpRequest 有关的方法。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9DrDQSgKzYSjSjpmNaWHiccJaK1zLIgovvfibae9vWHcYU3Q9YSKw4Z9A/640?wx_fmt=png)

在方法末尾的 send 处打上断点，然后放掉所有请求，重新触发断点。

此时可发现，在该断点的对象中，_url 还未包含两个加密参数。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9mgh0M0xnPe5e5WO8uoXs8JOSe84q9p1ALdibR6FXXpzG4YvEcC5C5bA/640?wx_fmt=png)

那么 F11 继续往下走，可见进入了混淆的 webmssdk 中，其对请求对象被赋予了某些操作。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9KeNiaTicaJfGksvnevXZCfX6BSTF4mHOPnOPs1pwmQvKLrrTicjphCSrw/640?wx_fmt=png)

当 F8 跳出这里回到最终的 nativeXMLHttpRequestSend 时，可发现_url 中已经产生了两个签名。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG90pvrroyTqrTkcCUicXoAwarF1iajWws6oe2gVHQeBiakiaSxs1E1UJ62Hg/640?wx_fmt=png)

只需要这两个断点即可分析大概的流程，既先构建一个 XMLRequest 对象，然后通过改写 send 方法将其交给 webmssdk 中的方法加密，加密之后再回到 XMLHttpRequest.prototype.send 中，重置 send 方法完成请求。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9nvyZsYjCLg0ZXeMsWAzKQonR3HRrsx9ckJgIJVHKZHiboBoFTPkQ2sA/640?wx_fmt=png)

> 每个函数都有一个 prototype 属性，这个属性是指向一个原型对象，原型对象包含函数实例共享的方法和属性，

> 通俗来讲，当通过 new 来生成一个类的对象时，prototype 对象的属性就会成为实例化对象的属性。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9NaPWnrnsW8ibZMcC7iaqyDWibewiaiawuD1vwAicLrdsmniadDL8OJ3aerXWA/640?wx_fmt=png)

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG973eJagNDX9z0Peaufkj8DBOUyTnXxCrw2WS2iaWSubhOHSgBQFlsZ9Q/640?wx_fmt=png)

Vol.02 远程调用

目前已经知道参数是怎么来的，就可以通过 RPC 方法来模拟生成。

虽然我们还没有具体追到加密的 JS 中，没有看到加密方法，但是由于其加密过程的特殊性，是基于操作 XMLRequest 对象的方法来进行调用，所以我们可以复刻过程生成参数

本地的调用不要着急，先按步骤来学习。

> RPC 是指跨进程间的远程调用过程，此处的意思是本地操作浏览器执行一些 JS 方法并返回结果。

在浏览器构建请求进行测试。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9xRBibXQ2WolJgGqQwuB91uxUnricVQqlnnbOrRqKUPUeNn2uMn0nwBZw/640?wx_fmt=png)

执行之后，查看控制台打印出的内容。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG99TVcamQM5ZxsWDHquaYKTMVa4mp3BWZecTzt09NqO2CwqnuZNfq0Kw/640?wx_fmt=png)

当我们在当前环境中发送一个请求，其返回的内容中包含了带有签名的链接，同时可看到 responseText 中已经返回了数据，那么说明整体的加密和请求都包含在了 send 中。

所以将这部分代码在本地模拟，比如通过 selenium 操作 chromderver 在网站环境中发送请求，即可进行简单的采集。

 (还有兴趣的继续往下阅读吧)

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9NaPWnrnsW8ibZMcC7iaqyDWibewiaiawuD1vwAicLrdsmniadDL8OJ3aerXWA/640?wx_fmt=png)

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG973eJagNDX9z0Peaufkj8DBOUyTnXxCrw2WS2iaWSubhOHSgBQFlsZ9Q/640?wx_fmt=png)

Vol.03 本地调用

上述分析中，已经明确了加密参数的生成流程，接下来需要处理 webmssdk 文件。

乐意花时间的可以尝试还原一下，或者慢慢调试下。

> 在线解混淆站点：

        cnlans.com:8887

我们需要在本地补上 XMLReuqest，执行 send 方法，同时把 webmssdk.js 中的代码复制到本地运行。

先复制 webmssdk.js 运行，根据报错在开头补环境。

```
报错 Request is not defined，补：Request = function Request() {};
报错 Headers is not defined，补：Headers = function Headers() {};
报错 document is not defined，补：document = {}
报错 window is not defined，补：window = {}
报错 document.addEventListener is not a function，补：document.addEventListener =  function () {}
```

继续运行无报错，添加 XMLReuqest 代码并发送请求。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9gHckAarEDgm3NPkibttIrcG4JdE0Ekv9TmzqbsgR1aE1So008YYpReA/640?wx_fmt=png)

虽然代码运行了，但是一直没有结束，代码中有 setInterval 和 setTimeout 定时执行着方法。

```
setTimeout = function(){return function(){}};
setInterval = function(){};
```

把定时器方法修改后，继续运行，发现没有报错也没有返回有用的结果。很正常，毕竟补的环境还都是 {}。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9Kt9MjbUVsr4mmBBJxRQ0E5BcIZCvngMMq7EOagg2ibdLH7WvmbGqbxw/640?wx_fmt=png)

现在先把你所了解的、看到的环境信息都补上去。

比如 screen、navigator、document、location、canvas、localStorage、sessionStorage、PluginArray、Image 等等。

结果补完之后还是啥都没有, 此时就需要动手分析源码了。

在本地代码断点可以看出，req.send() 走了一次就结束了，方法没走到 webmssdk.js 的代码中，说明我们的调用没有成功。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9Fb4nQl1Upg7kXlJvFTqehBIWicf7tM0mJlENM2l4GJORibYMPBfkpAxw/640?wx_fmt=png)

用我们第一段中的代码调试，打上断点。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9Hiahx16I8wicUC6zLlweAFkTvcFmq4VPIvvzYlcDaplx0yItJvAzEtXw/640?wx_fmt=png)

执行代码看断点怎么走的，和我们本地执行不同的是，目前可直接进入 function _0x65f4c7()。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG93JwbgEVEegpOwoozzR8Um8PDJq74KFp6HNOFNu036PiamOL8K5mrzgQ/640?wx_fmt=png)

那现在就需要对比本地和浏览器的区别在呢，像这类情况一般都是缺环境，或者是浏览器环境中有一些初始化的加载。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG986EF4WRO7d2qs7zRwHUfkvuMtoKS1F7Tzwvj6jSZTpdiaVP9jekFk8A/640?wx_fmt=png)

byted_acrawler 是该页面独有的。(主要是最早的版本就需要 byted_acrawler.init，所以一眼就看到了)

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9lhxST4rOAXdibnyhyoDIuccDp1ibWiaZeHnl7XJjkYoZTqz8F8TH6pFVQ/640?wx_fmt=png)

加上 init 之后，再次运行代码。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9FiaGbHomUwYJ124Pa6bia2ib4ialicUAPhx5Vyy46BCPMM2jryGrjFoQCeA/640?wx_fmt=png)

```
报错：Cannot read properties of undefined (reading 'onabort')
定义：XMLHttpRequest.prototype.upload = function (){}
```

```
报错：Cannot read properties of undefined (reading 'init')
```

意思是 window 中未定义 byted_acrawler，所以更没有 init 了。

所以可以猜这段代码中的 byted_acrawler，没有附加到我们定义的 window 中，要么是缺环境，要么是补的 window 和源码的 this 不匹配。那么加上 window = this; 指向当前 moudles

再次执行，成功返回结果。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9NaPWnrnsW8ibZMcC7iaqyDWibewiaiawuD1vwAicLrdsmniadDL8OJ3aerXWA/640?wx_fmt=png)

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG973eJagNDX9z0Peaufkj8DBOUyTnXxCrw2WS2iaWSubhOHSgBQFlsZ9Q/640?wx_fmt=png)

Vol.04 分析结果

打印结果，发现 req 的 onload 中携带了 send 后加载的 url，可发现经过两次计算，分别加上了 X-Bogus 和_signature。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9FzCicLYFia2ibKCbrsUc9W2DWzicxLk20IPqZic7a7rJP8qaiaItmujicqLuw/640?wx_fmt=png)

本地能够成功调用之后，就可以着手还原这套调用流程，这块笔者不写了。

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG910ibjVTiaQtiaa98hWEUiaV27VCQgxoqwQfvzBSwOlQADf2sRBDiccc47UA/640?wx_fmt=png)

本文主要是以教程为主，分享分析的流程和逆向的思路，不提供代码了。不过大概补了下图的内容就能返回结果，**但是不保证能请求成功**。

补环境用文字描述太过折磨，要补的东西有些多，一些简单的跳过了，各位自行调试，珍重！

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9ricwMHJ3EUfOX1IoqcPEFTiby18CNxXgIia0xPKjeDtVb1ImoWNxmoYsg/640?wx_fmt=png)

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG9NaPWnrnsW8ibZMcC7iaqyDWibewiaiawuD1vwAicLrdsmniadDL8OJ3aerXWA/640?wx_fmt=png)

![图片](https://mmbiz.qpic.cn/mmbiz_png/xxA783bXQVsh6FrZVnLzCAf5vT8FpHG973eJagNDX9z0Peaufkj8DBOUyTnXxCrw2WS2iaWSubhOHSgBQFlsZ9Q/640?wx_fmt=png)

**关注《Pythonlx》阅读更多内容**