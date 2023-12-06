> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/EmwuL3ToKwDFwCILZTM1AQ)

**“****本文共计 6640 字，阅读大约需要 13 分钟****”**

**写在前面：**

不同于往期，今天我们不光要对算法进行破解，更要利用我们所掌握的知识对它进行还原，以便能够领悟到它的精髓所在。同时，还要感谢微信昵称 1900，的小伙伴，对我提供相关的资源以及文章参考，让我有能力或者说有运气接触到此类大厂的算法，并经过日渐的深入研究，窥探到了它内部的乾坤。

**0x01 对 VM 的理解**

有很多从事爬虫行业的小伙伴都对 **VM** 这个概念感到困惑，甚至有时还会感到莫名的恐惧。因为，一旦网站采用了这种手段进行加密，那无异于就是对我们的发量发起了挑战。究其原因是因为它所表现出的是：**极差的阅读性，极难的调试性，和极强的迷惑性**，比如：

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaG2wicsKqw1l1Y9vEjUaCuvicTOqcn1ajlGO1mfrt2QSAeQzwBtVEqUOWYnPd6PFfzVHdolagYC2bvg/640?wx_fmt=png)

一个死循环，嵌套无数控制分支，就如同一个迷宫一样，让人有进无出。更抓狂的是，我们所熟知的 **javascript** 内置函数，都是靠它一些特殊的指令所生产而来，而不是直接调用的，比如：我们把一个字符串转换成 **base64** 编码

正常转换：

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaG2wicsKqw1l1Y9vEjUaCuvicerA2UJryibdYMQicqic9lRAA9EC2azzFjHYyNcQKxUKw6cJNcx6JDPWKw/640?wx_fmt=png)

VM 转换:

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaG2wicsKqw1l1Y9vEjUaCuvicZT1pOmLpVI5VsV51nZlHSTXO1CCUBWrKhdjiakX66N7lOoZicEWuAbYQ/640?wx_fmt=png)

由此可见，**VM** 算法加密不光对代码本身做了保护，更是对代码的运行机制做了修改。让人懵逼之余，还会夹杂着一丝丝绝望。而目前我所知的突破 **VM** 算法的办法就是使用 **AST** 也就是对代码平坦化流程操作。但是，对于今天研究的 **TX** 滑块算法，使用平坦化流程策略，会出现一些问题。对于这些问题，我会在遇到时重点进行研究，并给与解决方案。

**0x02 分析请求定位参数**

对这种算法有个简单认识后，我们开始进入我们今天的正题。打开浏览器，进入到滑块网站（这里注意一下，因为滑块是前端的 **iframe** 页面，我们直接访问它 **iframe** 地址即可）

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGwqAbAsJO99VzKxXp0asAFnohhuoNpgPvsibsyrA2eHXIzSw7MJh94T2EFGe3NnYticN37nucCOibfw/640?wx_fmt=png)

然后我们拖动滑块，查看网络请求，会发现这样一个验证信息

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGwqAbAsJO99VzKxXp0asAFxJ9bdL8S7PLxoWFbFBCmA6T4WmPhaKa345HdfKibattsVSpka41lVeA/640?wx_fmt=png)

**post** 表单里面有很多参数，我只截取了一部分，当然最重要的就是这个 **collect** 参数，因为只有它会随着滑块滑动的不同位置而产生变化。由此，我们给出初步推测：这个参数应该是服务端校验滑块是否正确的主要依据。换句话说，它肯定包含滑块的某种信息，只不过被加密了而已。所以我们来全局寻找下这个参数。

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGwqAbAsJO99VzKxXp0asAFSFgwh9nzhLuwaL6vvxFC85ywogVhFlS6IodbanHLpicBEu03eYZN6jQ/640?wx_fmt=png)

我们会在一个以 **tcaptcha** 为前缀的文件中寻觅到它的身影，直接打上断点去分析，这个 **decodeURIComponent(R())** 方法所返回的结果正是 **collect** 的加密内容

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGwqAbAsJO99VzKxXp0asAF7vxTiciaCo59JMb3LANTQFoxBpHT44cpO060ibdxP4qznvMuLDyibKOL6A/640?wx_fmt=png)

然后通过 **R** 所指向的函数位置，我们有了这样的定位

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGwqAbAsJO99VzKxXp0asAFBVZ4Y26uRv0AF0jpdy0riblhTkbEx6NyzcouAiaeYsFyq7tOmIRlsSDg/640?wx_fmt=png)

而通过 **a.getData(!0)** 我们就能获取到 **collect** 的加密值

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGwqAbAsJO99VzKxXp0asAFL9HD7k1FHvC17jMddOd6KYXvG2EGv1aJVqZf0CAicCCAR3UKbcRsh4w/640?wx_fmt=png)

怪了！**getData** 直接传了个 **true** 就把 **collect** 的加密全部获取出来了。有点不符合常理，其原因就是跟我事先猜想相违背。它传入的参数不应该是 **true** 应该是一些包含**滑块轨迹或者滑块位置**的信息阿！

**0x03 初识滑块轨迹**

经过冥思苦想反复研究之后，我终于发现了其中的猫腻。它虽然没直接传入滑块的相关参数，却采用了一种不易被人察觉方式对滑块信息进行了暗箱操作。然而我们可以在代码中发现一个，与之 **getData** 相对的函数 **setData** 。顾名思义：一个是个获取数据，一个是设置数据

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGwqAbAsJO99VzKxXp0asAFVqCMcWeJHIicgjBGAsUbatibevibpRsU6UibgWeWrsV4FWyFqree5Z57icg/640?wx_fmt=png)

我们尝试打上断点，看看 **setData** 在入参的过程中，是否有我们所需要的数据

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGwqAbAsJO99VzKxXp0asAFWrGTyVrYjWbJK4pHzGTcLKtElRGY82eDV1yfoddIKwSd7otv2a33Vw/640?wx_fmt=png)

当滑动滑块后，进行到第二次断点停留后，奇迹发生了  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGwqAbAsJO99VzKxXp0asAFn3ib3jQba5WhLt30E7ibkcv3Gu3AlPVhuf7TTg9T0ub3wLPSWMjbVQSQ/640?wx_fmt=png)

虽然这些数据让我们感到既熟悉又陌生，但其中的一个关键词，却能证明我们找对了方向。那就是 **slideValue** ，直接翻译过来：**滑块数据。**如果非要弄明白这些数据的成因的话，我们可以根据调用栈的信息，在同一 **js** 文件下有了这样的定位

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGwqAbAsJO99VzKxXp0asAFdjicicQ0hDjibNQCTazHFfLicVBHDdCEJDOCB75iaKmcABVK6fgEtgIj5Zw/640?wx_fmt=png)

我们通过上述图片所展示的数据可以看到，三个信息：**x**，**y**，**t** 所以我们可以给出结论，它们分别代表：滑块 **x** 轴的坐标，滑块 **y** 轴的坐标，以及拖动滑块的时间间隔，然后它们又经过下列函数，变成了 **setData** 的入参参数

```
for (var e = [], t = w.length - 1; t >= 0; t--)
        t > 0 ? (w[t].x = Math.ceil(w[t].x - w[t - 1].x),
        w[t].y = Math.ceil(w[t].y - w[t - 1].y),
        w[t].t = w[t].t - w[t - 1].t) : (w[t].x = Math.ceil(w[t].x),
        w[t].y = Math.ceil(w[t].y)),
        0 == w[t].x && 0 == w[t].y || e.push([w[t].x, w[t].y, w[t].t])
```

**0x04 插桩、Hook 应用与闭包机制（选读）**

滑块轨迹，是一种在时间的作用下驱动滑块而产生的变量。就跟日常行驶机动车一样，只要车开走，就会留下行驶的轨迹。但一般而言，车的轨迹无法被人捕捉和保留，那么代码是如何做到人能力之外的事情的呢？我们这里引入一个概念：**Hook** **鼠标位置。**直白点说就是：**给鼠标安装监听器。**尝试下列代码：

```
window.addEventListener("mousemove", function(event){console.log(event)}, false)
```

你会发现，每次挪动鼠标都会被浏览器给监听下来，并且能准确的输出你鼠标的位置  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaHXTLXL5tXzU3GQMSP6rOMEJAOLibcFicHKjTNz75erk7S63uibH0u8H5QQZWibzuB8ymCicfic7vyiaf6VA/640?wx_fmt=png)

那在本项目中怎样把滑块数据向上述一样直观的打印出来呢？答案：**插桩**。我们可以在监方法中加入断点日志，而这个监听方法在以 **slide** 为前缀的文件中的 2131 行左右。我们可以以此形式对它进行打印  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaHXTLXL5tXzU3GQMSP6rOMEQ8oYmqapCugOugCiaQPKV96VW5lW8QFjpuDTzvGsccZ9lScdYcHr4qA/640?wx_fmt=png)

  
由此，我们可以看到与滑块滑动的同步数据

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaHXTLXL5tXzU3GQMSP6rOMEhAkWauuKHmdIyLPJveuBmDFVxVzLgQFOo4nDRQUiaJj4MXM7pdY8N6g/640?wx_fmt=png)

还有一个问题就是，这个 **slideValue** 的值是如何保存在 **setData** 函数中，又是如何被 **getData** 所引用出来的呢？我们直接找到 **setData** 的原始对象（可以根据代码的赋值关系找到，这里就就不做讲解了）

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaHXTLXL5tXzU3GQMSP6rOMEI0sfpguQ1gmdQSc6xtXWpvobpJdyvjaA01rKKcxJicVZvJhgJzfHic3A/640?wx_fmt=png)

然后经过一番努力后探寻下（藏得很深），才能寻觅到 **slideValue** 的踪迹

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaHXTLXL5tXzU3GQMSP6rOMEYruo7wbicntHLFMnZtRO4ofpPibpsfD1mvxEFmfezVLkPIjKoHcmic3lg/640?wx_fmt=png)

结果有点超乎你的想象，我们需要打开四个叠层才能看到 **slideValue** 的数据，然而它又扎根在 **scope** 闭包土壤中，无法撼动。这个闭包概念我也不是太懂，只知道一条：**只有当这个函数运行后，我们才能拿到里面数据的操作权限。**但我尝试很多方式对 **setData** 进行调用，无一例外都失败了，根本不能对里面的数据触碰分毫。如果前端小妹用手卷起麻花辫，硬是想看你秀出这一波，咋办？好吧，那只能再次拿出我们的终极杀招：**Hook** 大法，来满足她所有的期望了。我们可以加入以下的代码

```
Object.defineProperty(window.TDC, "setData",{
    configurable: true,
    enumerable: true,
    writable: true,
    value: function (e) {
          if(e.hasOwnProperty("slideValue")){window.slideValue=e}
    }
})
```

然后滑动滑块后，直接读取 **window** 里面的 **slideValue** 属性，就可以拿到滑块数据的所有信息，当然作为 **js** 的返回值也可以，而且屡试不爽

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaHXTLXL5tXzU3GQMSP6rOMEYOH87L08AU2PHIicSCNXyoOTVu9gVOsQcSQovAicf5ET81DIriamVP08A/640?wx_fmt=png)

至此，我们了解了滑块的运行原理，懂得了服务端的验证机制，明白了监听器的使用方式。下面我们将进行本文最核心的内容：

**0x05 开启 VM 的潘多拉魔盒**  

> 前进，前进，不择手段的前进！
> 
> 托马斯 · 维德

其实，本文最核心的内容，也能代表出大家目前心中最深的困惑：**getData** 或者 **setData** 究竟是个啥？为啥能够引起我们这么多的关注？不好意思。这个问题没有答案，因为这两个函数，无名，无姓，像两个俄罗斯套娃一样，既可以相互独立，又可以彼此统一，具体会在下面的代码中表现出来

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGk2ib1YdsACxzImdtozS8ia5OYTibGUauVCkkdXynIF3KNEMKIfnia1coefna3u7RdOV7wyxWLXicgpSA/640?wx_fmt=png)

上述，你所见的正是 **setData** 和 **getData** 函数所指向的位置。更有甚者，与它们之外的所有函数都指向了这个位置：比如：**getInfo** 和 **clearTc** 就如同《三体》中的水滴一样，它们存在着一种令人瞠目结舌的共性  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGk2ib1YdsACxzImdtozS8ia5piaIQSvlgqicegGOu8kUaTmwGYXzEjqwLxL96EV9ia8iakPSkF5kQSajMA/640?wx_fmt=png)

比起前面死循环、假指令而言，这又是 **VM** 的算法的的第三个特点：**所有的函数都指向了同一函数位置**，我们姑且把这一特点称之为：伪方法（勿喷，纯个人观点）。**死循环**、**假指令**、**伪方法**，这根本就不是他妈的代码啊！俨然就是一个骗局！所以我们接下来的事，就是利用其中的规律，将它的骗局彻底曝光在群众的视野下

**0x06 探寻规律**  

代码需要遵从一定的规律，才可以达到探寻的目的，如果没有规律可循，就算你请来探长也无济于事。所以我们要把找规律作为第一要务：

我们执行 **getData(true)** 时，这个 **true** 参数会以 **arguments** 数组的形式向函数内部传递

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGk2ib1YdsACxzImdtozS8ia5eyODludulPNwpGsMo6ibJE3UanicO8iccJXSdOVoiamnh0x8paDqv3osDA/640?wx_fmt=png)

然后又经过 **__TENCENT_CHAOS_VM** 这个函数进入 **VM** 的结构体

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGk2ib1YdsACxzImdtozS8ia5d4HmfQsSAc5vXVZDCZic3iaI0uAJpc15nO0fkcBVXMFOdSn0GFGjw3yQ/640?wx_fmt=png)

然后会在这个位置进行无限次循环，而且每一次循环 **U** 变量都会 +1

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGk2ib1YdsACxzImdtozS8ia5cyjlvqqMJj7grfj48RLLhvU2UoRFibSKX3R4LN20FAWwUoCoG0AxzxA/640?wx_fmt=png)

**Q** 是一个包含 63 个匿名函数的大数组，只要 **[M[U++]]** 产生变化，就会根据不同的结果值，对大数组里的匿名函数进行调用，而且有序不乱各司其职

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGk2ib1YdsACxzImdtozS8ia55YCkPDD2Qicdp7THD5qRaNlqt2EwwiaIaxwush7ZdNic2HQT4QWia8Aic5w/640?wx_fmt=png)

而后，它们会将工作的结果统一存放在 **B** 这个新数组中并会以冒泡的形式（不太准确）对每个匿名函数返回的结果进行排序

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGk2ib1YdsACxzImdtozS8ia5VhoRtmMFicxFjUfVuQulF6uMqxPKx3GgF7jrfKSBYic0wqTBVtDPtQKA/640?wx_fmt=png)

当 **w** 为 **true** 的时候，它会结束这个循环，将结果推向最终位置进行返回  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGk2ib1YdsACxzImdtozS8ia5M96F5VSQWKibE9MFhCcs4fPlfyx98ibjlNMIdlHgWBryrNc0SgmhEIiaw/640?wx_fmt=png)

好了，了解到这里，我们可以总结出如下的两条规律：

1.   **[M[U++]]** 的变化会对 **Q** 数组里面的匿名函数进行不同程度的调用
    
2.   **Q** 数组内部的所有索引值跟  **[M[U++]]** 的结果存在一一对应关系  
    

利用如上的两条规律，我们就可以对它进行另一种操作：流程平坦化。这里注明一点：**所谓的流程平坦化，并不是直接破解算法，而是将代码以另一种形式进行展示，便于我们分析**  

**0x07 局部被覆盖 | 全局被抹杀**   

先前说过对代码实现流程平坦化会存在一些问题，而这些问题会具体表现在全局变量和局部的赋值关系上。在实现之前，希望你能牢记这一点：当 **w = true** 时 **VM** 算法会将内部的循环进行停止

我们依照 **Q** 与 **[M[U++]]** 的对应关系，将 **[M[U++]]** 赋值给 **index** 然后在循环底部增加 **switch** **case** 逻辑分支（注：**由于 TX 的代码每次开启新的验证页面都不会相同，所以接下来的代码跟先前的可能有出入，但不会影响任何逻辑**）

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaG5QKQX71YCWLjM7UDldTXb5pNMj8hKD1M1PekNAzcJwiakVl0dcoicqibgvcpqaqcExWZCHKpHJpMpw/640?wx_fmt=png)

目前，像我这种喜欢 “硬刚” 的人而言，对于展开代码这项工作，没有找到任何捷径，都是一点点的复制粘贴过来。如果你们有更好的方式，可以向我分享一下。然后我们运行代码后，问题瞬间就会暴漏出来

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaG5QKQX71YCWLjM7UDldTXbjvjnn8zbGzlq1ZKZeB2nWcbibb3sgNkXYe3AtwEz0f2gN7vuw7Oe8tQ/640?wx_fmt=png)

究其原因，是因为 **D** 变量也就是先前说的 **w** 变量在 **index=48** 时布尔值得状态被篡改，导致意外跳出循环，引起了这一局部变量被覆盖的错误

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaG5QKQX71YCWLjM7UDldTXbriaajerS0riaSHJBGIPjMk9PkIqYbZiaZicVv6oaPOEBvn3Of1unWEJZ0Q/640?wx_fmt=png)

那有人问了，我们可以把它拿到 **for** 循环外部去，不就可以完美的避开这一问题了？像这样：

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaG5QKQX71YCWLjM7UDldTXbiaWtJib4485cDTXrSB3NJSVJ4NNO2p3Ob47UibrUD14B9Rd84BXdVpcgg/640?wx_fmt=png)

我们运行代码后，一个新的问题又会让我们陷入迷茫

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaG5QKQX71YCWLjM7UDldTXbHfTuozD0hzOMOIZgiaHicgUrnCCdvIAmq73329ZCSy95fwiaHh7e2Viahw/640?wx_fmt=png)

究其原因，是因为 **T** 所对应的函数定义在全局，而我们所定义的函数无法对其引用所导致的错误（至于是什么原因，我也一头雾水）

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaG5QKQX71YCWLjM7UDldTXbNSeicDMmdZaCicH8OEebTePcauZ5TgPd3kurnuHmvfXn1UmvfE7S6Dfg/640?wx_fmt=png)

这是函数内部报错的具体位置  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaG5QKQX71YCWLjM7UDldTXbXiaE47JqL9ADecMNqc3iaJSmvwD8HFGpJaLGWUxKLnGIWdjpszY073OA/640?wx_fmt=png)

当然，如果有人像产品经理一样不讲理，硬是要求你把这个模式设计出来咋办？摘掉假发撞他，不对，我们可以把 **json** 作为结构体，在数组中进行展开，这样的话代码结构就会一目了然

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaG5QKQX71YCWLjM7UDldTXbnb1gI2sGhicUVIvibWUnaEiaIF0YibnCcu4alOWtEa5OE1LvUORZq8pG8w/640?wx_fmt=png)

当然，循环底部也需要小小的改动  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaG5QKQX71YCWLjM7UDldTXbIicEUEHoeZroqzicWFZzfYCjtZ0oicBibYzN5o5GqX8lILtNDcF0K07yfg/640?wx_fmt=png)

**0x08 以一种能理解的方式去解读**  

当我们把代码展开，面对形形色色的逻辑分支，即便你已经对它的运行过程熟稔于心，但还是碍于刚刚接触的缘故，没办法用语言对它进行描述。所以今天我想独辟蹊径，用一种极为形象的比喻，来加深你对它的理解。我们姑且把这种形式称之为：**工厂化结构复原。因为后期你会发现，它的运行机制确实如工厂一样，复杂但很有序。**

当我们运行 **getData(true)** 的时候，可以看到它由 **index = 48** 开始进入代码的主体。我们可以把这一起始位置称之为：**入料**

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy1k4lyoQrXNd8fzyN9Ztdmo2h9JxpJd8HiaTJniaqBIbB39ciaKPyhUfnZw/640?wx_fmt=png)

然后它又来循环体进行工作分配，我们把这个循环分配的环节称之为：**传送**

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy1oUXyVAGlNeIsqzMTXmINfN4XeMs8kx5EoibTOl8CJPBDS9Udz95oEOQ/640?wx_fmt=png)

传送会将一些 “材料” 的信息输入到数组中，并与各个函数进行相关联，我们把这一环节称之为：**加工**  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy10EzeCbFaC6GuERFUhjokJXj6Tic5rkvJ1ExMuBhDVcicAJHNGM7LuickQ/640?wx_fmt=png)

在加工的过程，当 **index = 26** 时，它会对你电脑一些的特征进行检测，比如：系统时间、本地域名、浏览器标识等。我们把这个环节称之为：**质检**

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy1QZGr2iaxc6lDT6FL1O1d6B6jjEFVKYIgknbPeibryHhZOfZA6AE1NVSw/640?wx_fmt=png)

质检完成后，在 **index = 36** 时它们会把这些收集来的信息进行拼接处理，我们把这一环节称之为：**装配**  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy11PGiaeCwAWwzBqGgM9T1pvaicozticGFFpBic1picLdjwaqpjFRn79mQmew/640?wx_fmt=png)

在装配成功后，由传送发送 **index = 3** 的停止指令（**D=true**），整套循环停止，然后把成品返回，我们把这一环节称之为：**出货**

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy1kryctZV0g76e9BTbg7U4dyEW5ujwgicKyIjURoFlos0hQx1SO8rVtVg/640?wx_fmt=png)

**我们再次梳理一遍：入料 -> 加工 -> 质检 -> 装配 -> 出货，简直和工厂生产的模式一模一样。当然需要注明一点：本概念属于本作者独创，如果其他作者想引用，请联系作者授权或者附带《逆向简史》公共号来源说明。**

**0x09 以点带面的分析法则**

阅读至此，想必大家心中都会对这个算法有了系统性的了解，接下来我们就要进入到本篇文章的尾声：**推导与算法还原（注：阅读本章节请务必将上一章读完，因为这章很多的定义都是取自上一章）**  

我们先从**出货**位置入手，去分析它在得到 **collect** 加密前都经历了哪些环节。所以直接加入断点日志（注：为了方便阅读我对这步进行了重新赋值操作）

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy1et8rvibH9CA6RLOlqfODoZibFfZ73BsQoR49TpgKYbkaqe9WVK0Yxo1Q/640?wx_fmt=png)

然后可以分析出 **collect** 是由三个阶段的加密结果拼接而成，最后进行了 **urlencode** 编码（我这里截图范围有限，你们往上翻阅即可发现其中规律）

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy1G1vv9cQ61kY93icQJ7ajH6NHtSDJMcSF0pxTcPLE6ialOYSBW9ewMBug/640?wx_fmt=png)

按逆向推理顺序，**出货**之前要对货物进行**装配**，所以我们也要从**装配**的位置，既 **index = 36** 加入断点日志

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy1NOzhA2KAicOrUt8EiagU0QVx7XOWX0LXg4QiaNmSq8ZWWL5NveyrHpaHw/640?wx_fmt=png)

对比下结果，发现，加密是由**装配**位置中的一堆乱码数据转变而来，这堆乱码数据我们暂且称之为：**unicode** 编码

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy1tkmN3394vF0fuxeaHvUYOcfXZGia0KA3GUFWfDnJBV1f7Yns56icSESg/640?wx_fmt=png)

这些 **unicode** 编码数据长度固定，我们可以利用此项规律创建条件断点，来分析**出货**加密前的临界操作  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy1zv4kb6rZr5flr6qtd7es5rLgNmZneODZlcklOf2JKr9W3ia0ib7M9QQg/640?wx_fmt=png)

最终我们得出结论，**出货**的数据是由**装配**中的 **unicode** 数据 **base64** 编码而来

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy1EsL0ibmakqm3eEKL7cLr1qvvlWnrSBHLuIrrh5pCxgzn0op1NT0rf7A/640?wx_fmt=png)

同时，我们通过对**装配**位置的打印结果也有了新发现：那些 **unicode** 乱码的数据是由先前的**质检时**所收集的数据**每隔四个步长有序化拆分而成**，比如滑块轨迹，只不过在中间经历了一段超出你认知之外的事情：那就是算法加密  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy1dDhUIrSR4ExENvb6HVCBPODtBQiaexbLo2QLPaclrGPsph2AblJzhqQ/640?wx_fmt=png)

好吧，我已经感觉到屏幕外一脸懵逼的你们，像这样  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy1icv4BWAFkCyLH7cUk09KuIS27mXJQAyPQ4a48iaiaXYqRlYyh3ibOAj3Rw/640?wx_fmt=png)

画个图直观一点  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaFttGTQpdBNLnFaBH1Ulsy10a4hib00Jcsmg7XDnIeORZaDic1ssqa9NvNjfXQHVsyuHBYa3icTiaKUwg/640?wx_fmt=png)

图画的比较烂，但大致可以表明意思。其中最核心的东西就是 **TX** 算法。有待思考的问题是：这种算法是如何将：字符串 Valu 变成 EvÈo 只要把这过程搞明白，我们就会彻底突出重围，迎来胜利的曙光

**0x10 终章：细节才是关键**  

我们先拿一个字符串做分析，并将算法的过程列举出来  

```
#include <stdint.h>
void encipher(unsigned int num_rounds, uint32_t v[2], uint32_t const key[4]) {
    unsigned int i;
    uint32_t v0=v[0], v1=v[1], sum=0, delta=0x9E3779B9;
    for (i=0; i < num_rounds; i++) {
        v0 += (((v1 << 4) ^ (v1 >> 5)) + v1) ^ (sum + key[sum & 3]);
        sum += delta;
        v1 += (((v0 << 4) ^ (v0 >> 5)) + v0) ^ (sum + key[(sum>>11) & 3]);
    }
    v[0]=v0; v[1]=v1;
}

void decipher(unsigned int num_rounds, uint32_t v[2], uint32_t const key[4]) {
    unsigned int i;
    uint32_t v0=v[0], v1=v[1], delta=0x9E3779B9, sum=delta*num_rounds;
    for (i=0; i < num_rounds; i++) {
        v1 -= (((v0 << 4) ^ (v0 >> 5)) + v0) ^ (sum + key[(sum>>11) & 3]);
        sum -= delta;
        v0 -= (((v1 << 4) ^ (v1 >> 5)) + v1) ^ (sum + key[sum & 3]);
    }
    v[0]=v0; v[1]=v1;
}
```

然后分为三段进行研究，既：

```
① Valu -> [1701079404, 1970037078]
```

```
② 1970037078 -> -15304460731
```

```
③ -15304460731 -> EvÈo
```

鉴于中间也就是第②部的转换比较复杂，我们先从两端开始进行，在**装配**位置设置条件断点

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHPDdal1adYDvPluvNDpxjM6KfyPTJfJiaHnedfLuONRichoibngBRwrxogQ/640?wx_fmt=png)

在完成转换前，将所有的**加工**过程进行打印，也就是说观察一下，它所经历了哪些步骤。这里面存在个疑问，为什么不直接打印 **index** 而是打印 **B[Q]** ？因为直接在 **index** 前面设置断点会出现一个问题，这个问题我也没想明白，当然你打印 **B[Q++]** 会改变原始值，直接报错

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHPFIVoPsuU8EDh7LOhdklWTzKojd7MEOJiaic2JSLAG9gIgk8c23MOKs8g/640?wx_fmt=png)

这些是第①步转化过程中所经历的步骤（大约 200 步左右）  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHP7hRE34IzB6KIUQggmzSh8qQ3NjicwcwWQIZRanTKCnSiayLCs4cP75dg/640?wx_fmt=png)

虽然步骤很多，但其中也存在十分明显的规律，既：它们使用 **charCodeAt** 依次获取字符串 0~3 范围内的 **unicode** 编码（不是之前那个）

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHP4ReiavvmJKyOFmdHQohMExhL8Ww1WIaYGmCHkDhoWM4DhjwhXA02zyw/640?wx_fmt=png)

然后把每次的结果分别按照 0，8，16，24 进行左移位（<<）计算再次并以递归的形式进行逻辑或（|）运算。表述很难，直接上图

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHP8woRHeBDfPN86OibHHtmlpuu2quTEIiaVD2OdWr3icOBr1A2zYaPqbpfQ/640?wx_fmt=png)

有人觉得不对，第①步所对应的是个列表，为什么只返回一个整型数字？原因是它不只是对一个字符串进行运算，而是按拆分的规律对一组数据分别的运算: ['lide', 'Valu']  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHPhdYibwekJzo56HnxEohelibps9mLWIz3TmpXicnEBmzr7fm3LY3tNGMEQ/640?wx_fmt=png)

仿照上述的规律，我们写出第①步所对应代码  

```
// 字符串转char code
function stringToCharCode(str) {
    var move = [0, 8, 16, 24]
    var orChar = 0, charCode;
    for (var m of move) {
        inde = move.indexOf(m)
        charCode = str.charCodeAt.apply(str, [inde]) << m | orChar;
        orChar = charCode;
    }
    return charCode;
}
```

第③步和第①步分析的规律如出一辙，甚至比它还简单些，直接上图

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHP2F2hH291UkG8bWEv54BKOjL8wX0RIEM1THvHd450sict5OgXnMB1U1A/640?wx_fmt=png)

按照此运算规律，直接可以写出第③所对应的代码

```
// 编码转换unicode
function stringToUnicode(char){
    var move = [0, 8, 16, 24];
    var unicodes = [];
    for(var m of move){
        var code = char >> m & 255;
        unicodes.push(code);
    }
    return String.fromCharCode.apply(String, unicodes);
}
```

第②应该是最难的，它是一位剑桥大学的教授所设计出来的算法，而对于我这种大石桥还没毕业的学渣而言，很难看懂。我是根据一个特征把它百度出来的。更要命的是，**TX** 还在这个算法的某些地方做了改动

具体来看下它的规律，我们还是加入断点日志（为了便于观察我把 **Q** 的值也进行了打印）

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHPlzWKr9OwO0ibibynFRoOUQxV9Yic3ic2eVrAYH0GRchArBxCicNOmMYB0tA/640?wx_fmt=png)

我们把结果复制到文本中进行分析  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHPBs74cXnmXB6VcRjzGibWW441iagQgWNDazRUa4QK3j3knCKGFwiaIdheQ/640?wx_fmt=png)

如上第②经历了三千个步骤（有点吓人），是第①的十倍不止。但它其中有个明显的规律：**经历****了 32 组轮询计算**

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHPibHNsmpcEsLELWIxR4HApJDx1bFDPTKoM9mMibGbp7t4M0ZwtttqHufA/640?wx_fmt=png)

而且再调试的过程种发现，每一组轮询在计算之初的经历了类似下面的过程  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHPx8Vp2REJiagHvIbZK8qrR8wXg24boqyUYoVtDXvbuJ5Z0gVJImMcy2g/640?wx_fmt=png)

起初我也不太明白，甚至给我搞得有些绝望。因为我始终觉得它肯定符合某种算法运算规律，但就是不知道怎么进行搜索。直到我遇到绿色字体那个数值（2654435769），才看出了端倪，迎来了希望。这是百度的结果

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHPP9nEArjicaTib5YOLeZ8DLP0iaLNgicuwWZOSQcDY83ofUbfRlJK6BGz2Q/640?wx_fmt=png)

了解到了它应该是一个名为：**x-tea** 的算法，我又随手谷歌了一下，以下是 **C** 语言版的源码

```
#include <stdint.h>
void encipher(unsigned int num_rounds, uint32_t v[2], uint32_t const key[4]) {
    unsigned int i;
    uint32_t v0=v[0], v1=v[1], sum=0, delta=0x9E3779B9;
    for (i=0; i < num_rounds; i++) {
        v0 += (((v1 << 4) ^ (v1 >> 5)) + v1) ^ (sum + key[sum & 3]);
        sum += delta;
        v1 += (((v0 << 4) ^ (v0 >> 5)) + v0) ^ (sum + key[(sum>>11) & 3]);
    }
    v[0]=v0; v[1]=v1;
}
void decipher(unsigned int num_rounds, uint32_t v[2], uint32_t const key[4]) {
    unsigned int i;
    uint32_t v0=v[0], v1=v[1], delta=0x9E3779B9, sum=delta*num_rounds;
    for (i=0; i < num_rounds; i++) {
        v1 -= (((v0 << 4) ^ (v0 >> 5)) + v0) ^ (sum + key[(sum>>11) & 3]);
        sum -= delta;
        v0 -= (((v1 << 4) ^ (v1 >> 5)) + v1) ^ (sum + key[sum & 3]);
    }
    v[0]=v0; v[1]=v1;
}
```

这里面需要两个参数：含有四个值密钥的数组和一对数组序列值，但这个算法被 **TX**  修改过，应该是：>> 5 它修改成了 >>> 5。上述例子就能看出来，于是我们转成 **JS** 代码

```
function encryptTeaAlgorithm(v) {
    var key = [1164667728, 1114199370, 1213157446, 1346923371];
    var v0 = v[0], v1 = v[1]
    var sum = 0;
    var delta = 0x9E3779B9;
    for (var i = 0; i < 32; i++) {
        v0 += (((v1 << 4) ^ (v1 >>> 5)) + v1) ^ (sum + key[sum & 3]);
        sum += delta;
        v1 += (((v0 << 4) ^ (v0 >>> 5)) + v0) ^ (sum + key[(sum >> 11) & 3]);
    }
    return [v0, v1];
}
```

这里面还有个坑，就是那四个密钥，**TX** 在里面做了手脚。给人得错觉是直接读取出来的

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHP1xTYuJu4BbxDRt4q4FY6xcH3y3wfice7zW2GmKHiaqtMsgmpLPzyibOfQ/640?wx_fmt=png)

其实它的计算出来的，具体计算方法是：  

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHP9GzQIzViabykClPSrKKosrUmCzGWG8ITHblP5wEKZddzHynkFuwSg5w/640?wx_fmt=png)

好了，来把第②步的实现一下

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHPzVJTo6CDvEXTO1gTknZPaPALj9WYE4XKKTfEeDuA9BodJyGTbKmkxQ/640?wx_fmt=png)

我们如果把上述过程梳理明白，直接就能实现它的算法，最后附上一张效果图：

![](https://mmbiz.qpic.cn/mmbiz_png/7QYmakzsJiaGst4OBic2icwBicI07B4lOgHP8oWqOmzS9NGC7ZaWoIm3BC18qMu82VLwibM9AG699XYjKaZu9glWpKg/640?wx_fmt=png)

**结语：**  

从破解算法到创作这篇文章大约花费了我两周左右的时间。可以说，掠去了所有下班时间和休息日的的自由，连相亲都没能顾得上（开个玩笑)。虽然在此期间所经历的过程艰辛漫长，甚至有几度想要直接放弃，但一要想到比我优秀的人比我更卷，我就没敢将这种想法付诸于行动。**所以分析不易，创作更难，如果本文对您有帮助的话，可以点个在看支持下，谢谢！**  

**最后声明：**

********✧✧** **本篇文章只用于学习交流，切勿用于商业用途，如果侵犯权益，请与作者联系，本人会在第一时间将其删除。**✧✧**********