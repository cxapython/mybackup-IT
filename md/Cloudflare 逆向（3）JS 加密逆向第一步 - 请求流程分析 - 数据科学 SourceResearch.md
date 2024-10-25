> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.resourch.com](https://www.resourch.com/archives/19.html)

> Cloudflare 5 秒盾在我们访问国外网站时经常会遇到一个检测页面：Checking if the site connection is secure 这是 Cloudflare 的 Web 应用防火...

Cloudflare 5 秒盾
---------------

在我们访问国外网站时经常会遇到一个检测页面：

> Checking if the site connection is secure

这是 Cloudflare 的 Web 应用防火墙（WAF），一般我们称之为 5 秒盾，Cloudflare 将其称为 "等待室"，如果检测到异常情况，就会弹出 hCaptcha 验证码或者拒绝请求。

![](http://www.resourch.com/usr/uploads/2022/12/3090899296.png)

基于协议的爬虫如果不处理加密参数就会被 banip，ban 的次数多了就可能被添加到黑名单中。可以选择使用基于自动化测试框架的爬虫，如 selenium、playwright 等，配合指纹浏览器来过检

但是使用自动化会导致经常被置于检测页面好几秒，相比于协议的一秒上百并发，自动化爬虫一分钟也请求不了几个页面，另外，5 秒盾不一定是 5 秒，具体的检测时间取决于目标站点的安全配置和爬虫的行为。如果目标网站的安全配置较高，则检测页面可能需要花费十多秒的时间。

这个检测页面覆盖了全站，包括 HTML 页面、后端接口以及**静态资源**，如果想要继续爬取，必须绕过盾检测页面。

本节我们会初步研究 Cloudflare 的 JavaScript 反爬机制，梳理请求发送的流程，以便更好地分析后续的反爬技术。

对 Cloudflare 的 js 反爬进行逆向工程
--------------------------

### 第一步：检查请求日志

首先，在浏览器中按下 F12 键打开开发者工具，切换到网络选项卡，并勾选禁用缓存和保留日志。然后打开目标网站。

触发检测页面后，将重定向到实际网站，并发现以下请求内容：

#### 第一个请求：获取初始化 js

首先，请求目标 URL 后，会返回一个反爬页面的 HTML。在 `script` 标签中有一个重要的匿名函数。该函数负责一些初始化内容，并加载初始的反爬脚本。。

```
(function () { 
 window._cf_chl_opt = { 
    cvId: '2', 
    cType: 'non-interactive', 
    cNounce: '12107', 
    cRay: '744da33dfa643ff2', 
    cHash: 'c9f67a0e7ada3f3', 
    /* ... */ 
    }; 
    var trkjs = document.createElement('img'); 
    /* ... */ 
    var cpo = document.createElement('script'); 
    cpo.src = '/cdn-cgi/challenge-platform/h/g/orchestrate/jsch/v1?ray=744da33dfa643ff2'; 
    window._cf_chl_opt.cOgUHash = /* ... */ 
    window._cf_chl_opt.cOgUQuery = /* ... */ 
    if (window.history && window.history.replaceState) { 
        /* ... */ 
    } 
    document.getElementsByTagName('head')[0].appendChild(cpo); 
})();
```

`(function(){...})();` 表示立即执行该匿名函数。Cloudflare 的所有 JavaScript 都是动态生成的，每次请求的内容都不一样，因此上述的 JavaScript 可能与实际获取的内容不同。

#### 第二个请求：获取第一个加密 js

![](http://www.resourch.com/usr/uploads/2022/12/4285455035.png)

第二步，向 `/cdn-cgi/jsch/v1?ray=::rayId::` 发送初始 GET 请求，其中 `rayId` 是上述 window._cf_chl_opt.cRay 的值。此 GET 请求会动态返回一个经混淆的 JavaScript，每次请求返回的内容都不一样。  
![](http://www.resourch.com/usr/uploads/2022/12/2960609011.png)

#### 第三个请求：获取第二个加密 js

第三步，在另一个 URL 上发送 POST 请求，以获取一段 base64 编码的字符串。

这里不提供 URL，因为 URL 是由之前获取的两个字符串拼接而成的。构建 URL 路径需要使用两个字符串，通过不断查找，我们可以确定字符串的位置：

一个是初始化脚本中定义的`parsedStringFromJS`字符串， 另外一个是`window._cf_chl_opt.cHash`的值。

所以我们最后构造出来的 url 是`/parsedStringFromJS/window._cf_chl_opt.cHash`

这个 post 请求的作用是负责获取二级 js 加密脚本，请求体是一个 form，格式为：`v_rayID=initialChallengeSolution`。响应体是一个 base64 编码的字符串

题外话，做逆向的时候的一定要学会找参数，可以通过 xhr 断点，重写相关 js 对象的方法加入 debugger 语句来 hook，或者通过 f12 中的请求调用栈来定位从哪里获取的参数

#### 第三四个请求：获取真正的 html 内容

第四步，最后一个 POST 请求是请求主站的 url，并且包含了一些加密参数，例如：

```
md: <string> 
r: <string> 
sh: <string> 
aw: <string>
```

请求后给了我们目标网页的实际 HTML 和一个叫做 cf_clearance 的 cookie，获取了 cf_clearance 我们就可以自由访问网站了

这些参数凭空产生，在 F12 中并未提供太多信息，特别是所有的数据似乎要么是加密的，要么是随机的。因此，尝试通过黑盒逆向工程来绕过 Cloudflare 的可能性大大降低。

我们可以通过直接请求真实 IP 地址来绕过 CDN，但绝大多数套了 cdn 的网站都通过反向代理工具避免爬虫获取真实 IP。

因此，现在我们别无选择。这些请求是从哪里来的？上述请求参数代表什么？返回的 base64 字符串有何用途？根据前文的分析，第一个请求的内容是未加密的 JavaScript，我们将在下文逐步在这段 JavaScript 中寻找答案。