> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhaoxincheng.com](http://zhaoxincheng.com/index.php/2021/03/14/ios%E7%B3%BB%E7%BB%9F%E6%8A%93%E5%8C%85%E5%85%A5%E9%97%A8%E5%AE%9E%E8%B7%B5%E4%B9%8B%E7%9F%AD%E9%93%BE/)

> 前言作为爬虫工程师的基本功，抓包是经常遇到且非常重要的技能之一，尤其是现在的移动端的发展，更是在 App 抓包这里做了很多防护措施，本文主要是针对 i

前言
--

作为爬虫工程师的基本功，抓包是经常遇到且非常重要的技能之一，尤其是现在的移动端的发展，更是在 App 抓包这里做了很多防护措施，本文主要是针对 iOS 系统 App 的数据包抓取进行大概总结，主要是针对常规 App 数据包抓取、突破路由直连抓包以及通过简单的逆向 Hook 手段突破 SSL Ping 防护抓取数据包。

常规抓包
----

对于常规无抓包防护的 App 数据抓取，可以直接在手机上通过设置 HTTP 代理的方式，配合 Charles 便可以抓取到目标 App 的数据包，不需要使用其他复杂的或者逆向 Hook 技术进行辅助。

*   设备：iPhone 5s、Mac
*   Mac 抓包工具：Charles

### 抓包实践

手机和电脑连接同一个 wifi，在一个局域网即可。

路由直连
----

### 防护介绍

路由直连是客户端流量直接到达服务器而不经过代理或者未设置代理服务器。参考`ZXRequestBlock`项目：

```
+(NSURLSession *)zx_sessionWithConfiguration:(NSURLSessionConfiguration *)configuration                                    delegate:(nullable id<NSURLSessionDelegate>)delegate                               delegateQueue:(nullable NSOperationQueue *)queue{    if (!configuration){        configuration = [[NSURLSessionConfiguration alloc] init];    }    if(isDisableHttpProxy){//开启防代理抓包        configuration.connectionProxyDictionary = @{};//设置代理为空字典    }    return [self zx_sessionWithConfiguration:configuration delegate:delegate delegateQueue:queue];}
```

发现该项目是将`connectionProxyDictionary`设置空字典，从而绕过中间人代理的数据抓取分析。  
关于`connectionProxyDictionary`在官方文档中这样描述：  
![](http://zhaoxincheng.com/wp-content/uploads/2021/03/2021031904314396.png)  
也就是说：这个属性可以设置网络代理，默认值是 NULL，使用系统的代理设置。  
该示例 App 也就是通过将`connectionProxyDictionary`设置为空字典，从而绕过 Charles 等抓包软件的抓取。即 App 中的所有网络请求正常，但是所有的网络请求不经过 Charles 代理软件，针对这样的抓包防护，可以通过建立虚拟网卡的方式进行数据流量抓取。

### 抓包实践

在 Mac 上通过 Shell 连接到手机，查看手机端的网卡信息。

可以看到代理软件给在手机上建立了一个虚拟网卡，然后将流量转发到 Charles 上边，从而抓到该示例 App 的数据包。

Frida 绕过 iOS SSL Pinning
------------------------

### SSL Pinning 防护介绍

SSL Pinning 称为证书锁定（SSL/TLS Pinning）也就是 SSL 证书绑定，顾名思义，将服务器提供的 SSL/TLS 证书内置到移动端开发的 App 客户端中，当客户端发起请求时，通过比对内置的证书和服务器端证书的内容，以确定这个连接的合法性。  
SSL Pinning 又分为单向证书校验和双向证书校验：

*   单向证书校验是 App 应用对服务端证书进行校验，但是服务端不会对 App 的证书进行校验。
*   双向证书校验是 App 应用对服务端证书进行校验，服务端也会对 App 的证书进行校验。  
    如今，不少应用程序已在其移动应用中实现了 SSL Pinning，具体表现抓包的时候，App 显示`无法连接网络`或者`请求失败`之类的提示，这样的防护是无法完成具体的抓包分析，抓包成了分析的第一座壁垒。这层壁垒在本文中使用 Frida 和基于 Frida 开发的工具`objection`进行突破。  
    这里使用`应用商店`作为案例，突破单向证书校验。
    *   设备： 越狱的 iPhone 5s、Mac
    *   Mac 抓包工具：Charles

### 安装 Frida

*   Frida 是一个很常用的开源 Hook 工具，只需要编写一段 Javascript 代码就能对指定的函数进行 Hook，而且它基本上可以主流平台全覆盖，除了 iOS，还有 Android 和 PC 端的应用也可以用它来进行 Hook，非常方便。
    
*   启动 Cydia 然后通过 Manage -> Sources -> Edit -> Add 这个操作步骤把 [https://build.frida.re](https://build.frida.re/) 这个代码仓库加入进去。然后在 Cydia 里面找到 Frida 的安装包，最终将 iOS 设备插入电脑，并开始使用 Frida。
    
*   电脑上使用 python3 环境，然后`pip install frida-tools`
    

### Frida Hook 应用商店

电脑 usb 连接手机，并打开`应用商店`，然后在 Mac 上的终端输入`frida-ps -Ua`，可以看到该应用商店的信息了。  
![](http://zhaoxincheng.com/wp-content/uploads/2021/03/2021031809170070.png)

打开该应用商店，并进行数据包抓取，可以清楚的发现该应用无网络，很清晰的看到 SSL Pinning 的特征。  
![](http://zhaoxincheng.com/wp-content/uploads/2021/03/2021031904120812.png)

Frida 脚本：

```
/* *  Description: iOS 12 SSL Bypass based on blog post https://nabla-c0d3.github.io/blog/2019/05/18/ssl-kill-switch-for-iOS12/ *  Author:     @macho_reverser */ // Variablesvar SSL_VERIFY_NONE = 0;var ssl_ctx_set_custom_verify;var ssl_get_psk_identity; /* Create SSL_CTX_set_custom_verify NativeFunction *  Function signature https://github.com/google/boringssl/blob/7540cc2ec0a5c29306ed852483f833c61eddf133/include/openssl/ssl.h#L2294 */ssl_ctx_set_custom_verify = new NativeFunction(    Module.findExportByName("libboringssl.dylib", "SSL_CTX_set_custom_verify"),    'void', ['pointer', 'int', 'pointer']); /* Create SSL_get_psk_identity NativeFunction * Function signature https://commondatastorage.googleapis.com/chromium-boringssl-docs/ssl.h.html#SSL_get_psk_identity */ssl_get_psk_identity = new NativeFunction(    Module.findExportByName("libboringssl.dylib", "SSL_get_psk_identity"),    'pointer', ['pointer']); function custom_verify_callback_that_does_not_validate(ssl, out_alert) {    return SSL_VERIFY_NONE;} var ssl_verify_result_t = new NativeCallback(function (ssl, out_alert) {    custom_verify_callback_that_does_not_validate(ssl, out_alert);}, 'int', ['pointer', 'pointer']); function bypassSSL() {    console.log("[+] Bypass successfully loaded ");     Interceptor.replace(ssl_ctx_set_custom_verify, new NativeCallback(function (ssl, mode, callback) {        ssl_ctx_set_custom_verify(ssl, mode, ssl_verify_result_t);    }, 'void', ['pointer', 'int', 'pointer']));     Interceptor.replace(ssl_get_psk_identity, new NativeCallback(function (ssl) {        return "notarealPSKidentity";    }, 'pointer', ['pointer'])); } bypassSSL();
```

上边脚本保存为 killSSL.js 文件，使用 Frida 加载脚本进行 Hook: `frida -U --no-pause -f com.***.AppStore -l killSSL.js` 打开该应用商店之后即可抓包。  
![](http://zhaoxincheng.com/wp-content/uploads/2021/03/2021031809181532.png)

该脚本主要是通过 Hook 更 App 中`ssl_ctx_set_custom_verify`、`ssl_get_psk_identity`，替换其回调函数，从而不会去检查服务端的证书链。

### 使用 objection 快速绕过 SSL Pinning

objection 是基于 Frida 开发的并且功能强大、命令众多，而且不用写一行代码，便可实现诸如内存搜索、类和模块搜索、方法 Hook 打印参数返回值调用栈等。

#### objection Hook SSL Pinning 实践

上述脚本适用于 iOS12 系统，但在 iOS9、iOS10、iOS11 等系统无法使用，objection 正好进行了整合，在`objection`的源码中（具体路径`objection/blob/master/agent/src/ios/pinning.ts`）可以看到它对 iOS 的多个版本以及`Framework`等，通过观察查看源码，发现 objection 是通过 Hook AFNetworking 等常用库和一些底层的 methods 实现绕过证书绑定。  
![](http://zhaoxincheng.com/wp-content/uploads/2021/03/2021031612403633.png)  
在 Mac 终端执行`pip install objection`  
再去执行 `objection -g com.**.AppStore explore`，此时`objection`已经附加在该应用商城了，然后在 objecton 的终端执行 `ios sslpinning disable` 就可以快速过绕过 SSL Pinning，当然该命令是否能奏效还要视具体的 App 而定。  
![](http://zhaoxincheng.com/wp-content/uploads/2021/03/2021031809191092.png)  
最后效果同 Frida 绕过 SSL Pinning。

小结
--

iOS 端的短链抓包在很多 App 上面都没有做相关防护，从而使抓包变得更容易，但是针对有抓包防护的 App，都可以根据其抓包中出现的现象进行逐一突破，目前常用的也就是本文中提到的路由直连以及 SSL Pinning，都可以使用代理软件或者逆向 Hook 技术进行突破。

参考
--