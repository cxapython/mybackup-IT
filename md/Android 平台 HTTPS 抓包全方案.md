> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/l13OLrXJbRrtUkQlV1q6fg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/v1LbPPWiaSt5u4qlRDV7JCPTlhLruwgCE7IwG1NBPXaJxfvDKvhCWSDe0PMuVbEb1pavYAiaAjbELXcNC1UPzbJw/640?wx_fmt=jpeg)

/   今日科技快讯   /

美团股价今日开盘不久就攀至 133 港元以上，依据实时汇率，市值突破 1000 亿美元。至此，美团成为腾讯、阿里之后，国内第三家市值跨过千亿美元门槛的互联网公司。

/   作者简介   /

本篇文章转自 MegatronKing 的博客，分享了 Android 中 https 抓包相关的解决方案，希望对大家有所帮助！

原文地址：

> https://juejin.im/post/5cc313755188252d6f11b463

/   前言   /

HTTP 协议发展至今已经有二十多年的历史，整个发展的趋势主要是两个方向：效率和安全。效率方面，从 HTTP1.0 的一次请求一个连接，到 HTTP1.1 的连接复用，到 SPDY/HTTP2 的多路复用，到 QUIC/HTTP3 的基于 UDP 传输，在效率方面越来越高效。安全方面，从 HTTP 的明文，到 HTTP2 强制使用 TLSv1.2，到 QUIC/HTTP3 强制使用 TLSv1.3，越来越注重数据传输的安全性。总而言之，HTTP 协议的发展对用户是友好的，但是对开发者而言却不那么友善。

抓包是每个程序员的必修技能之一，尤其是在接口调试和程序逆向方面具有广阔的用途。但是，随着越来越多的通信协议使用加密的 HTTPS，而且系统层面也开始强制规定使用 HTTPS，抓包似乎是显得越来越难了。

本篇博客，主要详解 Android 平台下，HTTPS 抓包的常见问题以及解决办法。工欲善其事必先利其器，博客中以 HttpCanary 作为抓包工具进行讲解。更多 HttpCanary 的资料，请见：

GitHub 地址：

> https://github.com/MegatronKing/HttpCanary

/   抓包原理   /

几乎所有网络数据的抓包都是采用中间人的方式（MITM），包括大家常用的 Fiddler、Charles 等知名抓包工具，HttpCanary 同样是使用中间人的方式进行抓包。

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt7ra2p8nyTdv7W6EgMicBFsTJEk9ib9IoUia7ezeuxDupDicmvjMumQXkP8pLST9dUGwnLiaaciaLzicG8FA/640?wx_fmt=png)

从上面这个原理图，可以看出抓包的核心问题主要是两个：

*   MITM Server 如何伪装成真正的 Server
    
*   MITM Client 如何伪装成真正的 Client
    

第一个问题，MITM Server 要成为真正的 Server，必须能够给指定域名签发公钥证书，且公钥证书能够通过系统的安全校验。比如 Client 发送了一条 https://www.baidu.com / 的网络请求，MITM Server 要伪装成百度的 Server，必须持有 www.baidu.com 域名的公钥证书并发给 Client，同时还要有与公钥相匹配的私钥。

MITM Server 的处理方式是从第一个 SSL/TLS 握手包 Client Hello 中提取出域名 www.baidu.com，利用应用内置的 CA 证书创建 www.baidu.com 域名的公钥证书和私钥。创建的公钥证书在 SSL/TLS 握手的过程中发给 Client，Client 收到公钥证书后会由系统会对此证书进行校验，判断是否是百度公司持有的证书，但很明显这个证书是抓包工具伪造的。为了能够让系统校验公钥证书时认为证书是真实有效的，我们需要将抓包应用内置的 CA 证书手动安装到系统中，作为真正的证书发行商（CA），即洗白。这就是为什么，HTTPS 抓包一定要先安装 CA 证书。

第二个问题，MITM Client 伪装成 Client。由于服务器并不会校验 Client（绝大部分情况），所以这个问题一般不会存在。比如 Server 一般不会关心 Client 到底是 Chrome 浏览器还是 IE 浏览器，是 Android App 还是 iOS App。当然，Server 也是可以校验 Client 的，这个后面分析。

/   安装 CA 证书   /

抓包应用内置的 CA 证书要洗白，必须安装到系统中。而 Android 系统将 CA 证书又分为两种：用户 CA 证书和系统 CA 证书。顾明思议，用户 CA 证书是由用户自行安装的，系统 CA 证书是由系统内置的，很明显后者更加真实有效。

系统 CA 证书存放在 / etc/security/cacerts / 目录下，名称是 CA 证书 subjectDN 的 Md5 值前四位移位取或，后缀名是. 0，比如 00673b5b.0。考虑到安全原因，系统 CA 证书需要有 Root 权限才能进行添加和删除。

对于非 Root 的 Android 设备，用户只能安装用户 CA 证书。

无论是系统 CA 证书还是用户 CA 证书，都可以在设置 -> 系统安全 -> 加密与凭据 -> 信任的凭据中查看：

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt7ra2p8nyTdv7W6EgMicBFsTMbAQRHS9icUEHZDDpuaMJkTuE6UchWSiaKjvFqnUsvkYNhpMtnsbs0Qg/640?wx_fmt=png)

/   Android7.0 的用户 CA 限制   /

Android 从 7.0 开始系统不再信任用户 CA 证书（应用 targetSdkVersion >= 24 时生效，如果 targetSdkVersion <24 即使系统是 7.0 + 依然会信任）。也就是说即使安装了用户 CA 证书，在 Android 7.0 + 的机器上，targetSdkVersion>= 24 的应用的 HTTPS 包就抓不到了。

比如上面的例子，抓包工具用内置的 CA 证书，创建了 www.baidu.com 域名的公钥证书发给 Client，系统校验此证书时发现是用户 CA 证书签发的，sorry。。。那么，我们如果绕过这种限制呢？已知有以下四种方式（低于 7.0 的系统请忽略）。

#### 配置 networkSecurityConfig

如果我们想抓自己的 App，只需要在 AndroidManifest 中配置 networkSecurityConfig 即可：

```
<?xml version="1.0" encoding="utf-8"?>
<manifest ... >
    <application android:networkSecurityConfig="@xml/network_security_config"
       ... >
        ...
    </application>
</manifest>
```

```
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="true">
        <trust-anchors>
            <certificates src="system" />
            <certificates src="user" />
        </trust-anchors>
    </base-config>
</network-security-config>
```

这样即表示，App 信任用户 CA 证书，让系统对用户 CA 证书的校验给予通过。更多相关信息，详见  

Network security configuration：

> https://developer.android.com/training/articles/security-config

#### 调低 targetSdkVersion < 24

如果想抓一个 App 的包，可以找个历史版本，只需要其 targetSdkVersion <24 即可。然而，随着 GooglePlay 开始限制 targetSdkVersion，现在要求其必须>=26，2019 年 8 月 1 日后必须 >=28，国内应用市场也开始逐步响应这种限制。绝大多数 App 的 targetSdkVersion 都将大于 24 了，也就意味着抓 HTTPS 的包越来越难操作了。

#### 平行空间抓包

如果我们希望抓 targetSdkVersion >= 24 的应用的包，那又该怎么办呢？我们可以使用平行空间或者 VirtualApp 来曲线救国。平行空间和 VirtualApp 这种多开应用可以作为宿主系统来运行其它应用，如果平行空间和 VirtualApp 的 targetSdkVersion < 24，那么问题也就解决了。

在此，我推荐使用平行空间，相比部分开源的 VirtualApp，平行空间运行得更加稳定。但必须注意平行空间的版本 4.0.8625 以下才是 targetSdkVersion < 24，别安装错了。当然，HttpCanary 的设置中是可以直接安装平行空间的。

#### 安装到系统 CA 证书目录

对于 Root 的机器，这是最完美最佳的解决方案。如果把 CA 证书安装到系统 CA 证书目录中，那这个假 CA 证书就是真正洗白了，不是真的也是真的了。由于系统 CA 证书格式都是特殊的. 0 格式，我们必须将抓包工具内置的 CA 证书以这种格式导出，HttpCanary 直接提供了这种导出选项。

操作路径：设置 -> SSL 证书设置 -> 导出 HttpCanary 根证书 -> System Trusted(.0)。

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt7ra2p8nyTdv7W6EgMicBFsTc8sibPViazBEafXWibYSzw3eITnHL0kARV4p7D9gOP725IAlb3eJWUYew/640?wx_fmt=png)

PS. 很不幸的 HttpCanary v2.8.0 前导出的证书名称可能不正确，建议升级到 v2.8.0 以上版本操作。

导出. 0 格式的证书后，可以使用 MT 管理器将. 0 文件复制到 / etc/security/cacerts / 目录下，或者通过 adb remount 然后 push 也可（这里稍微提一下，别在 sdcard 里找这个目录）。

/   Firefox 证书安装   /

火狐浏览器 Firefox 自行搞了一套 CA 证书管理，无论是系统 CA 证书还是用户 CA 证书，Firefox 通通都不认可。这种情况，我们需要将 CA 证书通过特殊方式导入到 Firefox 中，否则 Firefox 浏览网页就无法工作了。

HttpCanary v2.8.0 版本提供了 Firefox 证书导入选项。在设置 -> SSL 证书设置 -> 添加 HttpCanary 根证书至 Firefox 中：

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt7ra2p8nyTdv7W6EgMicBFsTNaZ0fDdH1dg92ehGlqO7RTOWgKYaDXQBFccMbMGQ9fZiadF5BpStT1A/640?wx_fmt=png)

点击右上角复制按钮将 url 复制到粘贴板，然后保持此页面不动，打开 Firefox 粘贴输入复制的 url。

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt7ra2p8nyTdv7W6EgMicBFsTsQeS7LVI6d7FcNpdRGyWRFzmLESmfeReGicBORBx2V76oiaIW3cEKWPQ/640?wx_fmt=png)

出现下载证书弹框后，一定要手动勾上：信任用来标志网站和信任用来标志电子邮件用户。然后确定即可。

/   公钥证书固定   /

证书固定（Certificate Pinning）是指 Client 端内置 Server 端真正的公钥证书。在 HTTPS 请求时，Server 端发给客户端的公钥证书必须与 Client 端内置的公钥证书一致，请求才会成功。

在这种情况下，由于 MITM Server 创建的公钥证书和 Client 端内置的公钥证书不一致，MITM Server 就无法伪装成真正的 Server 了。这时，抓包就表现为 App 网络错误。已知的知名应用，比如饿了么，就采用了证书固定。

另外，有些服务器采用的自签证书（证书不是由真正 CA 发行商签发的），这种情况 App 请求时必须使用证书固定。

证书固定的一般做法是，将公钥证书（.crt 或者. cer 等格式）内置到 App 中，然后创建 TrustManager 时将公钥证书加进去。很多应用还会将内置的公钥证书伪装起来或者加密，防止逆向提取，比如饿了么就伪装成了 png，当然对公钥证书伪装或者加密没什么太大必要，纯粹自欺欺人罢了。

证书固定对抓包是个非常麻烦的阻碍，不过我们总是有办法绕过的，就是麻烦了点。

#### JustTrustMe 破解证书固定

Xposed 和 Magisk 都有相应的模块，用来破解证书固定，实现正常抓包。破解的原理大致是，Hook 创建 SSLContext 等涉及 TrustManager 相关的方法，将固定的证书移除。

#### 基于 VirtualApp 的 Hook 机制破解证书固定

Xposed 和 Magisk 需要刷机等特殊处理，但是如果不想刷机折腾，我们还可以在 VirtualApp 中加入 Hook 代码，然后利用 VirtualApp 打开目标应用进行抓包。当然，有开发者已经实现了相关的功能。详见：

案例 1：

> https://github.com/PAGalaxyLab/VirtualHook

案例 2：

> https://github.com/rk700/CertUnpinning

不过，这里 CertUnpinning 插件的代码有点问题，要改改。

#### 导入真正的公钥证书和私钥

如果 Client 固定了公钥证书，那么 MITM Server 必须持有真正的公钥证书和匹配的私钥。如果开发者具有真正服务端的公钥证书和私钥，（比如百度的公钥证书和私钥百度的后端开发肯定有），如果真有的话，可以将其导入 HttpCanary 中，也可以完成正常抓包。

在设置 -> SSL 证书设置 -> 管理 SSL 导入证书 中，切换到服务端，然后导入公钥证书 + 私钥，支持. p12 和. bks 格式文件。

/   双向认证   /

SSL/TLS 协议提供了双向认证的功能，即除了 Client 需要校验 Server 的真实性，Server 也需要校验 Client 的真实性。这种情况，一般比较少，但是还是有部分应用是开启了双向认证的。比如匿名社交应用 Soul 部分接口就使用了双向认证。使用了双向认证的 HTTPS 请求，同样无法直接抓包。

关于双向认证的原理

首先，双向认证需要 Server 支持，Client 必须内置一套公钥证书 + 私钥。在 SSL/TLS 握手过程中，Server 端会向 Client 端请求证书，Client 端必须将内置的公钥证书发给 Server，Server 验证公钥证书的真实性。

注意，这里的内置的公钥证书有区别于前面第 5 点的公钥证书固定，双向认证内置的公钥证书 + 私钥是额外的一套，不同于证书固定内置的公钥证书。

如果一个 Client 既使用证书固定，又使用双向认证，那么 Client 端应该内置一套公钥证书 + 一套公钥证书和私钥。第一套与 Server 端的公钥证书相同，用于 Client 端系统校验与 Server 发来的证书是否相同，即证书固定；第二套 SSL/TLS 握手时公钥证书发给 Server 端，Server 端进行签名校验，即双向认证。

用于双向认证的公钥证书和私钥代表了 Client 端身份，所以其是隐秘的，一般都是用. p12 或者. bks 文件 + 密钥进行存放。由于是内置在 Client 中，存储的密钥一般也是写死在 Client 代码中，有些 App 为了防反编译会将密钥写到 so 库中，比如 S 匿名社交 App，但是只要存在于 Client 端中都是有办法提取出来的。

#### 双向认证抓包

这里以 S 匿名社交 App 为例，讲解下如何抓取使用了双向认证的 App 的 HTTPS 包。如果服务器使用了 Nginx 且开启了双向认证，抓包时会出现 400 Bad Request 的错误，如下：

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt7ra2p8nyTdv7W6EgMicBFsT60UNdsGN8Su2IN63mwWv9JlR3TdnOt6U158nRpmZbwlr2ovlSYxwZQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt7ra2p8nyTdv7W6EgMicBFsTrwwrC4azBC4BX9M96moVHoyP3B1J3TPhZo61yVl5ON8pZBibCblC4sw/640?wx_fmt=png)

有些服务器可能不会返回 404，直接请求失败。接下来看，如何使用 HttpCanary 配置双向认证抓包。

首先，解压 APK，提取出. p12 或者. bks 文件，二进制的文件一般存放都在 raw 或者 assets 目录。

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt7ra2p8nyTdv7W6EgMicBFsTj6dYhLiaVicl1aAqw8obsZKBicibkqZOmORnIc3DEImQvQq6djRf2AMibMQ/640?wx_fmt=png)

将 client.p12 文件导入手机，然后在 HttpCanary 的设置 -> SSL 证书设置 -> 管理 SSL 导入证书中，切换到客户端（因为需要配给 MITM Client），然后导入. p12 文件。

由于双向认证的公钥证书和私钥是受密钥保护的，所以需要输入密码：

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt7ra2p8nyTdv7W6EgMicBFsTmibM00N8xIE4icUJglNsZica3uIPiaoaeGwGup2QicydGRQS5drLHJ2Hu9Q/640?wx_fmt=png)

一般通过逆向可以从 APK 中提取出密钥，具体操作这里略过。输入密钥后，需要输入映射域名，这里使用通配符 * 映射所有相关域名：

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt7ra2p8nyTdv7W6EgMicBFsTibNHIQhhdaAU60D45uKsvAqMEgRo6DLHf0a23ZXVrd2JAB6KoLchROQ/640?wx_fmt=png)

导入完成后如下：

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt7ra2p8nyTdv7W6EgMicBFsTFic0SbwSOYicM1kB28B8E0OVJKQicoAKQNmgPNFAEzrrnKX7XU932gopg/640?wx_fmt=png)

可以点进证书详情查看细节，这个 client.p12 文件包含公钥证书和私钥，是用于双向认证的。

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt7ra2p8nyTdv7W6EgMicBFsTjWQhZkDK2JEZtMMYVlbyxicm2eFONeDCt8L4bCk2bicdJJpRmS05EEHw/640?wx_fmt=png)

配置完成后，重新进行抓包，看看效果。

![](https://mmbiz.qpic.cn/mmbiz_png/v1LbPPWiaSt7ra2p8nyTdv7W6EgMicBFsTISS9eTR2k0oBfA6vBb74BPWDR46co7rIYfxicyndEChibGrjHVyJhIDQ/640?wx_fmt=png)

可以看到，之前 400 Bad Request 的两个要求双向认证的请求成功了！  

/   SSL 重协商  /

有些服务器可能会开启 SSL 重协商，即 SSL/TLS 握手成功后发送请求时服务器会要求重新握手。这种情况一般比较少，但是也不排除，已知的应用比如 10000 社区 就使用了 SSL 重协商。

由于 Android 系统对 SSL 重协商是有限支持，所以部分系统版本抓包会失败，表现为网络异常。在 Android 8.1 以下，SslSocket 是完全支持 SSL 重协商的，但是 SSLEngine 却是不支持 SSL 重协商的，而 HttpCanary 解析 SSL/TLS 使用的是 SSLEngine。在 Android 8.1 及以上，SSLEngine 和 SslSocket 统一了实现，故是支持 SSL 重协商的。

所以，如果确认服务器使用了 SSL 重协商，请使用 8.1 及以上版本系统进行抓包。

/   非 Http 协议抓包   /

如果确认了以上几点，HttpCanary 仍然抓包失败，那么极有可能使用的并非是 HTTP 协议。比如像微信聊天，视频直播等，使用的就不是 HTTP 协议，这种情况需要使用其它的抓包工具，比如 Packet Capture 这种直接解析 TCP/UDP 协议的，但是往往非 HTTP 协议的数据包即使抓到了也无法解析出来，因为大概率都是二进制而非文本格式的。

/   总结   /

抓包是个技术活儿，需要对网络协议有大致的了解，对抓包感兴趣的同学可以多查阅 TCP、UDP、SSL/TLS、HTTP 等相关资料。

HttpCanary 是专业的 HTTP 协议抓包工具，专注 HTTP 协议三十年（吹过头了），不过目前还不支持 QUIC/HTTP3 这种新协议，等 QUIC/HTTP3 正式应用起来再说吧。

推荐阅读：

[](http://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650249794&idx=1&sn=25ff1aedf2c5208fd7d7be44a453eb31&chksm=88636b2dbf14e23b9cabfd52a845d30af55c8cd3dfc33fc1d1e5956a906d412c34202a4eff5c&scene=21#wechat_redirect)[写一篇最好懂的 https 讲解](http://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650248619&idx=1&sn=412fff24e4d105bd6237ff4068323955&chksm=886364c4bf14edd2648d0eb887d6e74880213163ad404acfca08a04e7c8e175ffa9949207a75&scene=21#wechat_redirect)  

[如何优雅地恢复 Recyclerview 的滚动位置](http://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650249693&idx=1&sn=4e44b26f66d96b9d3f6780ff81e61b02&chksm=886368b2bf14e1a476d9aab3d05f1567d24813b1dd333d5c6e57447dd2d7dd9000887ed00b1c&scene=21#wechat_redirect)  

[ViewModel 源码，学起来！](http://mp.weixin.qq.com/s?__biz=MzA5MzI3NjE2MA==&mid=2650249601&idx=1&sn=a8b80625255d5d71a0909a5d9b8961bf&chksm=886368eebf14e1f8be8aeac8830790e945f64b0651b7bbc2825dab79d558a95b2ebebc4860e8&scene=21#wechat_redirect)  

欢迎关注我的公众号

学习技术或投稿

![](https://mmbiz.qpic.cn/mmbiz/wyice8kFQhf4Mm0CFWFnXy6KtFpy8UlvN0DOM3fqc64fjEj9tw23yYSqujQjSQoU1rC0vicL9Mf0X6EMR4gFluJw/640.png?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_jpg/v1LbPPWiaSt6FSn51QbdP1ic92cjsQM7LkBCfnaJMtcibMw9vYtdQ6QQM3CcFFbGqMoNucFlBRJw9E6VQWYk30ficw/640?wx_fmt=jpeg)

长按上图，识别图中二维码即可关注