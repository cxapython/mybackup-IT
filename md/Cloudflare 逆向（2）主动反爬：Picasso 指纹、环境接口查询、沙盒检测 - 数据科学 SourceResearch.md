> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.resourch.com](https://www.resourch.com/archives/11.html)

> Cloudflare 主动爬虫程序检测技术当我们访问受 Cloudflare 保护的网站时，浏览器中会运行风控检测，其中包含以下内容验证码验证码会降低用户体验，Cloudflare 是否显示验证码和用...

Cloudflare 主动爬虫程序检测技术
---------------------

当我们访问受 Cloudflare 保护的网站时，浏览器中会运行风控检测，其中包含以下内容

### 验证码

验证码会降低用户体验，Cloudflare 是否显示验证码和用户设置有关：

*   站点配置： 管理员是否启用了验证码
*   风险级别： 管理员启用了验证码后，Cloudflare 仅在流量可疑时才会跳出验证码。比如，如果用户使用 Tor 客户端浏览网站，会显示验证码，但如果用户运行标准浏览器，则不会显示验证码。

![](http://www.resourch.com/usr/uploads/2022/12/3423754764.png)

Cloudflare 早期使用 reCAPTCHA 作为他们的主要验证码提供商。但 2020 年后他们已经迁移到专门使用 hCaptcha，可能是 reCAPTCHA 的接码平台过多。

我们后面会介绍如何绕过验证码

### Canvas 指纹识别

Canvas 允许浏览器获取 Web 客户端的设备环境，包括访问网页的系统的浏览器、操作系统、图形硬件参数等等。

![](http://www.resourch.com/usr/uploads/2022/12/3460976170.webp)

Canvas 是一个 HTML5 API，可以使用 javascript 在网页上绘制图形和动画。那么这样的一个东西，是怎么实现指纹的呢？

第一步，网页会使用浏览器的 CanvasAPI 绘制图像；  
第二步，对该图像进行 hash 处理以生成指纹。

现代浏览器在进行 Canvas 处理时，都会直接调用系统硬件，指纹中会包含你的设备信息，Canvas 指纹的计算很复杂：

*   硬件：显卡，不同显卡渲染出的图形实际上是有肉眼难以察觉的微小差别的
*   底层软件：GPU 驱动、操作系统（字体、抗锯齿、渲染算法）
*   高级软件：网页浏览器（图像处理引擎：OpenGL、Skia 等）

因此，这是一项 **物理上不可克隆** 的功能，指纹中会包含你的设备信息，上面的内容只要稍微变动一点，就会产生唯一的指纹，canvas 指纹可以准确区分设备类别。

**【注意】** canvas 指纹包含的信息不足以充分跟踪请求者的身份，它的主要目的是准确区分设备。

Cloudflare 使用一种特定的画布指纹识别方法，具体请搜索 Google 的 `Picasso Canvas Fingerprint` 算法，[这里有一位台湾老哥的详解](https://ithelp.ithome.com.tw/articles/10300957)，这个算法不仅能够生成设备指纹，还能够起到 proof of work 的效果。这对于爬虫检测来说十分有效，因为基于 http 请求的爬虫难以通过仿造 canvas 指纹来逃避检测，Cloudflare 拥有大量合法画布指纹 + 用户的数据集。并且使用了机器学习，可以通过查找画布指纹与预期指纹之间的不匹配来检测是否存在指纹欺骗

### 事件跟踪

Cloudflare 将事件侦听器添加到网页中来侦听用户操作，例如鼠标移动、单击或键盘按键。一般来说真实用户需要使用鼠标或键盘进行浏览。如果 Cloudflare 发现用户没有鼠标或键盘操作，就可以假设用户是机器人。

### 环境接口查询

这是一个非常广泛的内容，浏览器有数百个可用于机器人检测的 Web API，这导致了 selenium 等基于浏览器自动化的爬虫也难逃其手（说实话 selenium 已经够不稳定够慢了，我都不想用。。。）差不多有以下 4 类：

#### 特定浏览器的 API

某些浏览器有，某些没有的 API。例如，`window.chrome` 是仅存在于 Chrome 中的属性。如果你伪造的指纹是 Chrome 浏览器，但却用 Firefox 发送请求，他们就会知道出了问题。

#### 时间戳 API

Cloudflare 利用时间戳 API（例如 `Date.now()` 或  
`window.performance.timing.navigationStart` 来跟踪浏览器的加载速度。

如果返回的时间戳不像人类活动，用户将被阻止。一些示例包括：浏览速度不像人类或时间戳不匹配（例如上传的时间戳早于页面加载前的`navigationStart`时间戳）。

#### 自动化检测

Cloudflare 向浏览器查询仅存在于自动 Web 浏览器环境中的属性。例如，window 对象下如果有 `__selenium_unwrapped` 或 `callPhantom` 属性，就表明存在 Selenium 和 PhantomJS。

#### 沙盒检测

在这里我说的沙盒是指在非浏览器环境中模拟浏览器，例如使用 nodejs 来模拟浏览器的 dom+js 环境。Cloudflare 可以检测你是否用了 nodejs，例如，脚本可能会查找仅存在于 NodeJS 中的 `process` 对象（不止这些，剩下的还有其他检测内容）。

#### js 注入检测

Cloudflare 的 js 进行了变态般的混淆，如果你想通过扣代码来还原它，那难度可以说是非常大的，因为它们还可以通过在相关函数上使用原型链检测来判断是否对关键函数进行了修改

`Function.prototype.toString.call(functionName)`

上面这段代码经过了层层混淆，你很难定位他们检测的位置，致使扣代码也无济于事。如果找到了解决方案重写 toString 方法也是可以的，但是对于动态生成的 js 来说，会增加很多工作量

下一章我们会讲讲 Cloudflare 最变态的内容，地狱级 js 混淆