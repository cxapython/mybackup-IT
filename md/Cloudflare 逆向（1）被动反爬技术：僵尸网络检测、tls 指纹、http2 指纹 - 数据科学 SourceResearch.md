> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.resourch.com](https://www.resourch.com/archives/4.html)

> Cloudflare 被动反爬技术 Cloudflare 如何检测爬虫？Cloudflare 的反爬可以分为两类： 被动 和 主动。被动爬虫检测技术包括后端执行的指纹检查等，而主动检测技术依赖于浏览器执...

Cloudflare 被动反爬技术
=================

Cloudflare 如何检测爬虫？
------------------

Cloudflare 的反爬可以分为两类： **被动** 和 **主动**。被动爬虫检测技术包括后端执行的指纹检查等，而主动检测技术依赖于浏览器执行的检查。以下是我在爬取过程中发现的 Cloudflare 采用的一些检测方案：

### 僵尸网络检测

Cloudflare 维护着已知与恶意爬虫相关的设备、IP 地址和行为模式的目录。

顶级黑名单中的设备会被永远禁止访问。次级黑名单（如代理、翻墙软件的节点）可能需要通过验证码、用户行为检测等验证。

### IP 地址信誉（风险评分）

用户的 IP 地址信誉（类似浏览器的反诈功能）基于地理位置、ISP 和历史访问记录等多个因素生成。

例如，属于数据中心或已知 VPN 提供商的 IP，必定会产生大量非人类请求，因此，这些节点的声誉将比住宅 IP 地址差。

站点还可以选择限制从其服务区域以外的区域访问站点，因为目标客户的流量不应来自那里（例如日本本土的迪士尼商城不会为日本地区外的用户提供服务，用户请求会被直接阻止）

### HTTP 请求头

Cloudflare 使用 HTTP 请求头中的 `User-Agent` 字段来确定是否为机器人。如果请求头中的 User-Agent 不是浏览器用户的 UA，例如 `python-requests/2.22.0` ，会被标记为机器人。

如果请求缺少浏览器中存在的标头，或者标头与浏览器不匹配，例如 Firefox 中产生了`sec-ch-ua-full-version-list`（Chrome 特性）标头，爬虫仍会被识别。

![](http://www.resourch.com/usr/uploads/2022/12/2668075855.png)

### TLS 指纹识别

这种技术使 Cloudflare 的反机器人能够轻松识别客户端是否合法，对于 python 来说比较致命，因为 python 的`ssl`库无法完全更改 tls 指纹（椭圆曲线部分，见下方内容）

有多种 TLS 指纹识别方法（如 JA3、JARM 和 CYU），每个实现都会生成一个独一无二的客户端的静态指纹。

TLS 指纹识别非常具有杀伤力，因为同一浏览器同一版本的指纹相同，但浏览器不同版本之间的 TLS 指纹、和 requests 库的实现不同，因此 Cloudflare 很有可能只维护了很小一段的白名单指纹。

TLS 指纹的构造发生在 TLS 握手期间。首先，客户端会在握手时发送一个`hello`消息：

![](http://www.resourch.com/usr/uploads/2022/12/165911060.png)

打开 Wireshark 抓取的包，我们可以看到其中包括 TLS 版本、密码套件、扩展，还有椭圆曲线参数等，以计算给定客户端的哈希值。

![](http://www.resourch.com/usr/uploads/2022/12/4227855332.png)

接下来，cloudflare 会在预先收集的指纹库中查找该哈希，判断是否在白名单内。假设客户端的指纹 hash 与允许的指纹 hash（即浏览器的指纹）匹配。在这种情况下，Cloudflare 会将客户端请求中的 UA 与存储的指纹 hash 关联的 UA 进行比较。

如果它们匹配，则安全系统假定请求源自标准浏览器。如果客户端的 TLS 指纹与请求的 UA 不匹配，则请求明显是来源于爬虫。

### HTTP/2 指纹识别

HTTP/2 是 HTTP2.0 协议版本，于 2015 年 5 月 14 日发布，目前所有主流浏览器都支持该协议，python 中的 requests 库依然无法有效进行应对

http2 的主要目标是通过引入 header 压缩和多路复用来提高 web 性能。因此为了拓展，HTTP/2 中的消息由帧组成，有十种不同用途的帧，帧始终是流的一部分。

我们从 http2 的第一帧看起，`SETTINGS`是客户端发送的第一帧，里面有一些特殊配置：

#### Chrome

![](http://www.resourch.com/usr/uploads/2022/12/2795111599.png)

#### PYTHON

![](http://www.resourch.com/usr/uploads/2022/12/148736766.png)

很明显，其中的差异可以用作指纹生成

http2 在 http1.1 的基础上添加了新的参数和值。我们都知道，http 的 header 包含客户端的确切版本，虽然可用于识别客户端，但是很容易被任何 http 库或其他工具伪造！而在 http2 中，虽然 header 一样可以伪造，但是下列伪标头在不同客户端的顺序中是不一样的

`:method`  
`:authority`  
`:scheme`  
`:path`

这同样可以用作反爬！

另外，http2 为了改善客户端的接收环境，声明了一个类似 tcp 窗口的`流窗口`大小，因此，这同样可以构成指纹的一部分

下面是详细的可以用作指纹的内容：

*   http2 帧：`SETTINGS_HEADER_TABLE_SIZE`， `SETTINGS_ENABLE_PUSH`，  
    `SETTINGS_MAX_CONCURRENT_STREAMS`， `SETTINGS_INITIAL_WINDOW_SIZE`，  
    `SETTINGS_MAX_FRAME_SIZE`， `SETTINGS_MAX_HEADER_LIST_SIZE`， `WINDOW_UPDATE`
*   http 流信息： `StreamID:Exclusivity_Bit:Dependant_StreamID:Weight`
*   伪标头字段的顺序： `:method` `:authority` `:scheme` `:path`

与 TLS 指纹识别一样，每个客户端都将具有静态 HTTP/2 指纹，Cloudflare 会始终验证请求中的指纹是否与存储在其数据库中的白名单对匹配。

[这里有个网站可以测试 http2 指纹](https://privacycheck.sec.lrz.de/passive/fp_h2/fp_http2.html)

下一篇文章讲讲主动反爬