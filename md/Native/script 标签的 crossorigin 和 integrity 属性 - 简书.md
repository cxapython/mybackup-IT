> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/3eb536f7c002)

通过 script 标签加载跨域资源是一种常用的跨域请求解决方式。script 标签去请求资源的时候，当引入跨域的脚本时有错误时，由于浏览器安全策略，是拿不到错误信息的。当本地尝试使用 window.onerror 去记录脚本的错误时，跨域脚本的错误只会返回 Script error。

#### 设置 crossorigin 属性后

crossorigin 属性值 anonymous（默认），user-credentials

crossorigin 的作用有三个：

1.crossorigin 会让浏览器启用 CORS 访问检查，检查 http 相应头的 Access-Control-Allow-Origin

2. 对于传统 script 需要跨域获取的 js 资源，控制暴露出其报错的详细信息

3. 对于 module script，控制用于跨域请求的凭据模式

但是对于跨域 js 来说，只会给出很少的报错信息：'error: script error' ，通过使用 crossorigin 属性可以使跨域 js 暴露出跟同域 js 同样的报错信息。但是，资源服务器必须返回一个 Access-Control-Allow-Origin 的 header，否则资源无法访问。

#### 总结

1. 设置了 crossorigin 就相当于开启了 CORS 校验。

2. 开启 CORS 校验之后，跨域的 script 资源在运行出错的时候，window.onerror 可以捕获到完整的错误信息。

3.crossorigin=use-credentials 可以跨域带上 cookie。

#### integrity 属性

integrity 属性是资源完整性规范的一部分，它允许你为 script 提供一个 hash，用来进行验签，检验加载的 JavaScript 文件是否完整。

integrity="sha256-PJJrxrJLzT6CCz1jDfQXTRWOO9zmemDQbmLtSlFQluc=" 告诉浏览器，使用 sha256 签名算法对下载的 js 文件进行计算，并与 intergrity 提供的摘要签名对比，如果二者不一致，就不会执行这个资源。

intergrity 的作用有：

1. 减少由托管在 CDN 的资源被篡改而引入的 XSS 风险

2. 减少通信过程资源被篡改而引入的 XSS 风险（同时使用 https 会更保险）

3. 可以通过一些技术手段，不执行有脏数据的 CDN 资源，同时去源站下载对应资源