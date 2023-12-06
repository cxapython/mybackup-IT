> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/mH_9FpJsHLSJj6-APn_54w)

这个算法我搞了大约一周才有结果，总的来说不是很难，但却相当费屁股。因为每天下班后，我需要一直**坐着**对它分析（好像别的动作也不适合），而且这一坐就是三四个小时。好在皇天不负有心人，我终于把它搞出来了！只不过现在的屁股有点不太能回血。。。

![](https://mmbiz.qpic.cn/mmbiz_jpg/7QYmakzsJiaHwpmVXoDXTOrCd2flicL9OOJFcwXoCyQBQWIich5gq0pr1swSS90xJGAoUj8eaiaVRmOCzyYpGfV6rA/640?wx_fmt=jpeg)

**对算法的个人理解：**

鉴于百度对这种算法没有固定的词条，谷歌出来的结果又看不太明白，所以我只能凭借多日来对它的研究经验，强行给它加个解释，当然，这种解释仅仅代表个人观点，不喜勿喷。

**电影《误杀》当中，李维杰之所以能够逃出生天，不仅是因为他阅片无数，可以模仿电影桥段，面对警局的调查瞒天过海。更重要的是，他在面临审讯时，始终在创造或者完善一种 “不在场” 的逻辑，只要这种逻辑一旦合理，那么所有证据都将失去意义。而今天我们要研究的代码也一样，它正是采用这种逻辑上的保护，让自己变得看似无懈可击。**

**三个重要的调试工具：**

在研究之前，我们先了解 **chrome 浏览器**自带几个断点调试工具，如果没有这几个工具出现，就算你头顶再秃发际线再高，也对我们即将研究的代码没有丝毫办法。

我们先简单写个 HTML 静态页面

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiaiaPH0y6KN5faLsRu8FC8g7697KboYDSer1mZ3ubuULFZbsp6SO2uQNw/640?wx_fmt=png)

然后用浏览器打开后查看源码

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiazRuO5GSDK9MF7qOReEIYCA8prtE4KgxtGdkxPVOqLdSRVYafVY82kA/640?wx_fmt=png)

随便在代码左侧的行数上面单击鼠标右边，会发现有一个功能面板

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiaHAnDlJAE1hKX46fUJydCZtZz39BYGNONF6MMdEOMc4320QqkAICQkw/640?wx_fmt=png)

我们只需要看前三个功能  

1. **Add breakpoint** 添加断点，这个功能比较简单通过标记后，刷新浏览器。如果代码会在此处运行，直接 **debugger**

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiaJ53z4TB5GsPbxS257VMP1YulwW1ic4O7B8GAAiaVMZQDxV4N1TDb07Xw/640?wx_fmt=png)

  
2. **Add conditional breakpoint** 添加断点条件，系统会自动根据你添加的代码逻辑进行 **debugger****（相当智能）**

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiaApRmFyssrYBSibr4taFmp4qTAoBCcD8uTxjfZXvCUlicaYNIQTOpXbFQ/640?wx_fmt=png)

3. **Add logpoint** 断点日志，又叫插桩，如果我们想查看某一变量的值，我们可以通过这种方式在 **console** 控制台对它进行打印

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiaSCg9r4pGoLlttCtYZxwqpiareyV4QuD0nd8OErzYARxaJAEOGhazBQw/640?wx_fmt=png)

效果如下：

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiaDuRMbKg3svw7ibWBym0Hn9sMEuJqbSGibRLhx5gxkx31mCKSpG3B8KrA/640?wx_fmt=png)

**分析加密：**

我们进入到目标网站通过分析请求会发现一个动态的 **_signature** 加密参数

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOia3yVRXRCtcuzCvFUQ6pPl8OAPWOUrnot6R1jorWgxdPgOG0W7XqdwLw/640?wx_fmt=png)

**定位加密：**

不同于以往，我们这次使用 **ajax** 请求拦截方式，来对这个参数进行拦截

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiadL7Lbice0NAkEP6yEtFUF477OBOOgX2O7HU8aaJDWXicic6fCjcSIQibicg/640?wx_fmt=png)

通过堆栈的调用信息，得出以下结果

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOia1hykOBly8C9yI31Bicfv30z4a29CESkTzp5bjFHn8QuAyD0PRnxtticQ/640?wx_fmt=png)

打上断点实现最终定位

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiaECKKDS9Oz4goeYwOJ83viboV16OhibeWxoQy49zwc3e9ajUFd6mbbWmw/640?wx_fmt=png)

换一种形式调用

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOia3jFH0ibJfu5PSYYpoMiaXBQFX3Zo2Er9kibiaX0Byaleom944lmibhphCSA/640?wx_fmt=png)

**window 被赋予的属性：**

我们分析得知： **_signature** 参数由 **window.byted_arawler.sign** 方法实现，然而这个方法指向 **acrawler.js** 文件中一个莫名其妙的位置

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOia7jdl7fGU5WdMLA7nKp4wQxnCs3TgjBHXqayoJGzj1AeHHQgJNOibNWg/640?wx_fmt=png)

所以我们目的也有了初步的明确：只要让 **window** 对象拥有 **byted_acrawler** 属性，那么整套算法就会被我们完全击破

![](https://mmbiz.qpic.cn/mmbiz_jpg/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiadcaK6icpVcVeGcbcm4d1gRFKuNBtysDX0MAwG92bySjbl7MGM5m6lIw/640?wx_fmt=jpeg)

**惯用的方法：  
**

我们在本地新建一个 **HTML** 文件把代码整体拷贝过来，然后用浏览器打开

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiacbU7fguW041Szerv97ADJDW7QA1NiakTAl8VyhQt9bby376ichyVWquQ/640?wx_fmt=png)

我们惊奇的发现，这个 **byted_acrawler**  属性已经存在于 **window** 对象之中，但是当我们借用 n**ode** 去执行它的时候，却秒被现实打脸

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiaTkt9yk2HvgUuanRR8Oahnib6ah7Sg9cyibL9HK6H7WMXxbQ0jsrnWziaA/640?wx_fmt=png)

**如何分析：**  

我们先整体看下代码，除了 **if-else** 就是 **else-if**  活活的一个无限套娃形式，而且里面的逻辑，只用单一的字母表示，根本没有任何阅读性

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOia2yYjOTeyj4OQoyIAdNJ16OGEe7Tb6YXvlEmbhE2OQvSuJfAuvNQA0Q/640?wx_fmt=png)

虽然通篇都是 **if-else**  但它肯定也符合一定的逻辑规律，如若不然，它一定会报错。**只不过官方这样做的目的，让它失去可读性，从而导致我们无法对分析。试想，我们可不可以，把它换一种形式进行表示出来，从而恢复它的可读性呢？**

**![](https://mmbiz.qpic.cn/mmbiz_jpg/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOia6bPHrdPwicvG7zqFfpiaEA0qsrgI2xkaOuoCRtJDjItkDcOHrY1tFFnQ/640?wx_fmt=jpeg)**

**平坦化流程：**

这是一个全新的概念，也是对抗 **jsvmp** 算法必会的概念，虽然方式比较简单，但工程量比较庞大。我们需要将代码中所有的 **if-else** 替换成 **switch-case** 但问题在于 **switch** 哪个变量 **case** 那种情况？

观察一下这段代码：

```
var j = parseInt("" + b[O] + b[O + 1], 16);
```

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOia1938SRicDzs1SS5k0yryuHYP8r8l5lLKhywyXQFTpJWjZdCHEyibKYLg/640?wx_fmt=png)

验证一下，整体流程会不会因为 **j** 变量的动态变化，进入到不同的逻辑分支？  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiaqJyAwwYbxqskRiboJpKTibCXyoTAe9KWV0lrbFzOWv6edIWMYPOs2HSg/640?wx_fmt=png)

不妨我们通过开篇所讲的**条件断点**试一下，每次 **j=16** 时会不会进入同一个逻辑分支中，如果能，说明 **j** 所有 **if-else** 的动态参数。如果不能，那就再去验证其他参数。

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiaAzL8Azc4fUX2hLIZ7EzyCglicA89P0ia6IkjNaicib6FfD5SWv2Wib6Uryw/640?wx_fmt=png)

经验证，我们可以确定 **j=16** 时都会进入到上述的分支，所以我们不妨可以这样将它表示出来

```
 switch (j) {
   case 0:
     // todo somethind
     break;
   case 16:
      q = S[R--],
      w = S[R--],
      (A = S[R--]).x === G ? A.y >= 1 ? S[++R] = K(b, A.c, A.l, q, A.z, w, null, 1) : (S[++R] = K(b, A.c, A.l, q, A.z, w, null, 0),
          A.y++) : S[++R] = A.apply(w, q);
      break;
    default:
      //todo something
      return;  
 }
```

最终我们梳理的结果是这样的

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOia97sRfNLkNERH8KqUCMxvUxCdqexsJnw1cICwFDklw5ldSB1lV0nia5g/640?wx_fmt=png)

这里注意一下**：****x 会有一个偏移量计算，每次运行都会 x>>4 并且 x 和 A 的值相等**

**环境对比：**

等我们对代码平坦化流程之后，我们需要考虑之前的一个问题，为什么浏览器环境就可以拥有 **byted_acrawler** 而 **node** 环境却是 **undefined** 呢？可能的原因是，它肯定存在某种环境的检测，导致 node 环境失效

我们先从运行的开端分析一下（这也是很多人会忽略的细节）

```
glb = "undefined" == typeof window ? global : window
```

这段代码放到浏览器运行，代表的是 **window** 对象

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiaEozMBFvqPHb8QFkqT6EEHraiaCwKibcLiazic6cr4VKZJmFuHpR8ibvzpjQ/640?wx_fmt=png)

如果把它放到 **node** 环境运行，它代表却是 global 对象

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiaGeC7t9TKJY65OSTKZjx9VTla0f2GNiaic9rqdVZvBEYAqcH2kuBYsaTw/640?wx_fmt=png)

所以我们需要改变下 **node** 环境，这里我采用的是 **jsdom** 来对环境进行一下伪装，这样的话，我们就改变了初始的环境

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOia51xdoJH272XwvNqJyt2HibI4ltzCwrGZ0DFfVrx3ohj30g3UZh3hV8w/640?wx_fmt=png)

**逻辑对比：**

修改完环境之后，我们再次运行我们本地文件，我们会发现抛出一个错误  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiaT5I8EsAr4ptN2ib3hDGef9RAZN4Jv7gUzwHCHXPfnFwRrBW1XY2ISWg/640?wx_fmt=png)

这是 **case 26** 流程所抛出的，所以我们直接在官方打条件断点进行比对

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiad3QtCbpsHCMyppXpKsxJFVZuQJPKiaxjF9az2VAIqhLicf59p55cKJow/640?wx_fmt=png)

这是官方的结果  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiacPda28FtsJia4CM7WRqhXahrnNrI1h7vmLxdibWSkl98osqC38WmUmvQ/640?wx_fmt=png)

这是本地的结果

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOialhL3aMZIIBvPmFymDFFBXuEdicerHlGdTsyu3HpWmIEia6oibweFN7LBA/640?wx_fmt=png)

经过对比，官方 **S** 中拥有 **Object** 函数而本地却是 **undefined** 所以我们要给它添加这种属性  

```
window.Object = function (){
    return ["native code"];
}
```

添加完之后自然有了这种属性  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiamIvqHbybZjNV5sQKicAmFV2FdXmjC3zRrMRazh2FiaPOwMmAAkla5cXQ/640?wx_fmt=png)

**环境补全：**  

鉴于这个对比过程太过漫长，我这里主要列出几个重要的属性，其他的请自行校验：

1. 补齐 **canvas** 属性

```
window.HTMLCanvasElement.prototype.getContext = function () {
    return {
        fillText(text, x, y, maxWidth) {
            return ["native code"];
        },
        arc(x, y, radius, startAngle, endAngle, anticlockwise) {
            return ["native code"];
        },
        stroke() {
            return ["native code"]
        },
        getExtension() {
            return ["native code"]
        },
        getParameter() {
            return ["native code"]
        }
    }
}
```

2. 补齐 **localStorage**（这个会影响 **_signature**  生成的长度）

```
(function () {
    let localStorage = {
        "__tea_cache_tokens_2018": "{\"user_unique_id\":\"verify_kut5p98i_BUC0aRbH_QZ5B_4lu4_BQzz_F22NI2z6snKh\",\"web_id\":\"7018918857491351074\",\"timestamp\":1634353853331}",
        "__tea_cache_tokens_24": "{\"web_id\":\"7018898163830441479\",\"ssid\":\"522d279c-ba4b-4c85-a61c-98b7f1a05b71\",\"user_unique_id\":\"7018898163830441479\",\"timestamp\":1634219409505}",
        "__tea_cache_first_24": "1",
        "tt_scid": "z1OscwP3.P6dJQt3WyFHyZ65WZCoGUYARVlW9x0SAYAXAN.mmr9m1v6FlmpA1M98f94d",
        "__tea_cache_first_2018": "1",
        "_byted_param_sw": "lmr6SespEoX9rv9+V/8="
    }
    for (var p in localStorage) {
        window.localStorage.setItem(p, localStorage[p])
    }
})()
```

3. 修改 **jsdom User-Agent** （必须跟请求中保持一致）  

```
resourceLoader = new jsdom.ResourceLoader({
    strictSSL: false,
    userAgent: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.77 Safari/537.36",
});
const jsdomConfig = {
    resources: resourceLoader,
    url: "https://www.toutiao.com/",
}
```

**简洁化处理：**

如果细心的话会发现，每个分支的代码，只运行了一小部分就跳出了所在的分支，剩下的代码形同虚设般存在。之所以会这样，是因为分支内部的 **A** 值会影响代码的整体走势，所以我们只需要判断满足 **A** 的代码就可以，其他的都是在尸位素餐

精简之前  

```
case 25:
    (A = x) > 8 ? (C = S[R--],
        S[R] = typeof C) : A > 6 ? S[R] = --S[R] : A > 4 ? S[R -= 1] = S[R][S[R + 1]] : A > 2 && (q = S[R--],
        (A = S[R]).x === G ? A.y >= 1 ? S[R] = K(b, A.c, A.l, [q], A.z, w, null, 1) : (S[R] = K(b, A.c, A.l, [q], A.z, w, null, 0),
            A.y++) : S[R] = A(q));
    break;
```

精简之后  

```
case 25:

    S[R -= 1] = S[R][S[R + 1]]

    break;
```

**总结：**

1.   对于这种混淆的代码而言，流程平坦化是首选，但如何进行才是关键
    
2.   代码不仅想知道你的运行环境，它更想知道这种环境是否真的适合它
    
3.   搞这种算法水平不一定要高，但屁股一定要好（敲重点）  
    

**最终效果：**

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGD9xc8IaomARkhzOOW5yOiakq4YH909V8IibHic2kzlejP2kzCBcj2omUicZNpVZvweX02kwSb4Lec2Q/640?wx_fmt=png)

**最后声明：**

**✧✧** **本片文章只用于学习交流，切勿用于商业用途，如果有侵权行为，请与作者联系，本人会在第一时间将其删除。**✧✧****