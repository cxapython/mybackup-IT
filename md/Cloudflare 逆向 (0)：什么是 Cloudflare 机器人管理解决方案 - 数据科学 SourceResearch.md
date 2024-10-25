> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.resourch.com](https://www.resourch.com/archives/3.html)

> 什么是 clodflareCloudflare 是一家美国网络基础设施和网站安全公司，提供内容交付网络和 DDoS 缓解服务。它的服务发生在网站访问者和 Cloudflare 客户的托管服务提供商之...

什么是 clodflare
-------------

Cloudflare 是一家美国网络基础设施和网站安全公司，提供内容交付网络和 DDoS 缓解服务。它的服务发生在网站访问者和 Cloudflare 客户的托管服务提供商之间，充当网站的反向代理。Cloudflare 是一家致力于互联网性能和安全的公司。我们最常知道的是他们的 CDN 服务以及等待窗口。

Cloudflare 机器人管理解决方案（Bot Management Solution）是 Cloudflare 推出的一款强大的反爬产品，在 Cloudflare 的 WAF 中已经得到广泛应用。

那么
--

*   什么是 Cloudflare 机器人管理解决方案
*   Cloudflare 的反爬策略
*   有 Cloudflare 反爬策略的逆向工程和绕过方案吗？

Cloudflare 机器人管理器（Bot manager）、5 秒盾
-----------------------------------

Cloudflare 提供网络应用防火墙（WAF）来保护应用程序免受多种安全威胁，如跨站脚本（XSS）、凭证填充和 DDoS 攻击。在访问网站时，我们经常会遇到一个被称作 “5 秒盾” 的东西。

Cloudflare 的 Bot Manager 是 WAF 的核心组成部分之一，它可以减轻恶意爬虫的攻击，同时不影响真实用户。

没有网站希望故意阻止谷歌或其他搜索引擎抓取其网页，因此 Cloudflare 承认爬虫的重要性，并维护了一个白名单，用于识别已知的良性爬虫。但他们认为所有不在白名单中的爬虫流量都是恶意的。

如果你以前曾爬取过受 Cloudflare 保护的网站，肯定遇见过这种情况：

*   错误码 1010：浏览器签名被加入黑名单
*   错误码 1015：被限制速率
*   错误码 1020、1012：拒绝访问，原因不明

响应头还会包含 403 Forbidden 的 HTTP 状态码，根据上述提示进行代码修改仍然会遇到错误

Cloudflare-Scrape？
------------------

一些朋友可能曾使用过 [cloudflare-scrape](https://github.com/Anorov/cloudflare-scrape)，这是一个基于 Python 的反 WAF 爬虫库，2021 年后基本告别世界，因为 Cloudflare 不仅修改了 JavaScript 反爬，还加上了多种反爬方案。

Cloudflare 目前使用的反爬虫策略可分为两类：被动和主动。被动反爬一般为后端进行的指纹检查，而主动反爬则依赖于在客户端进行用户行为检查。