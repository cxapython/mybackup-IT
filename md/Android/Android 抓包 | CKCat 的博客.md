> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [ckcat.github.io](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/)

> 使用 Charles 抓包 PC 端共享无线网络，使用 ipconfig 命令查看 ip 地址： 手机端设置代理

1.  PC 端共享无线网络，使用 ipconfig 命令查看 ip 地址：

![](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/2020-12-30-16-09-26.png)

2.  手机端设置代理

![](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/2020-12-30-16-10-42.png)

3.  抓包工具设置代理

![](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/2020-12-30-16-12-38.png)

4.  安装证书

手机端访问 chls.pro/ssl 下载证书并安装。其中可以通过 Magisk 插件 move Certificates () 将证书从用户证书移动到系统证书。后续即可进行抓包了。

> brup 配置好代理后，下载证书地址 [http://burp](http://burp/) 。

手动将用户证书移动到系统证书。

```
mount -o rw, remount /

mv -f /data/misc/user/0/cacerts-added/123abc456.0 /system/etc/security/cacerts

mount -o ro, remount /
```

[](#Charles-设置 "Charles 设置")Charles 设置
--------------------------------------

首先要把 Charles 当做一个 SOCKS5 的代理服务器，所以要先设置 SOCKS5 代理服务器。打开 Proxy 设置选项，开启 SOCKS 服务器，我这里开启的端口为 8889 ，也可以随意填写。

![](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/2020-12-30-16-52-51.png)

下面介绍 3 款手机端的 VPN 应用。

[](#Postern-设置 "Postern 设置")Postern 设置
--------------------------------------

在手机上打开 [Postern](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/Android%E6%8A%93%E5%8C%85/Postern-3.1.2.apk) 应用，设置 SOCKS5 代理服务器的地址。

![](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/2020-12-30-16-55-15.png)

然后设置 VPN 规则，我这里设置为所有地址都需要走 SOCKS5 代理服务器。

![](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/2020-12-30-16-57-27.png)

然后开启 VPN 即可使用 Charles 抓包。

[](#SocksDroid-设置 "SocksDroid 设置")SocksDroid 设置
-----------------------------------------------

这里还有另一款 VPN 工具 [SocksDroid](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/Android%E6%8A%93%E5%8C%85/SocksDroid.apk) 可以使用，这里也顺便说一下其配置方法，除了设置服务器 IP 和端口外，还需要设置一下 DNS 服务器，以适应国内网络环境。

![](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/2020-12-30-17-01-17.png)  
![](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/2020-12-30-17-01-48.png)

[](#drony-设置 "drony 设置")drony 设置
--------------------------------

最后再介绍一款 VPN 应用 [drony](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/(Android%E6%8A%93%E5%8C%85/drony_1.3.155.apk)) ，通过以下方法进行设置。  
进入设置页面，选中无线网络，然后已连接的无线网络，设置设置服务器 IP 和端口，并将代理类型设置为 socket5 。

![](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/2020-12-30-17-09-53.png)

如果针对某一个应用进行抓包，可以通过规则设置，界面如下图

![](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/2020-12-30-17-16-04.png)

1.  选择 Local proxy chain 。
2.  Application 选择需要强制代理的 APP 。
3.  Hostname 及 Port 不填 表示所有的都会被强制代理。

设置好需要抓包的应用后直接保存即可。然后返回日志界面，开启代理即可。

![](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/2020-12-30-17-18-46.png)

[](#HTTP-HTTPS转发到Burp-Suite "HTTP/HTTPS转发到Burp Suite")HTTP/HTTPS 转发到 Burp Suite
-------------------------------------------------------------------------------

在 Charles 中，打开 External Proxy Settings 选项卡，选择把数据转到 Burp Suite 的代理服务器中。

![](https://ckcat.github.io/2020/12/30/Android%E6%8A%93%E5%8C%85/2020-12-30-17-21-25.png)

参考连接：

```
https://mp.weixin.qq.com/s/ahPbBSfkkBsv4oy265rI2Q
https://www.cnblogs.com/lulianqi/p/11380794.html
```