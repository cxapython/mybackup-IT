> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/191d8ba2af38)

> **本系列文章共十篇：**

[【爬虫成长之路】（一）爬虫系列文章导读](https://www.jianshu.com/p/8e03db02b75b)  
[【爬虫成长之路】（二）各篇需要用到的库和工具](https://www.jianshu.com/p/9e8373a41416)  
[【爬虫成长之路】（三）【大众点评】selenium 爬虫](https://www.jianshu.com/p/8f7c1dd88fa4)  
[【爬虫成长之路】（四）【大众点评】selenium 登录 + requests 爬取数据](https://www.jianshu.com/p/8326186c0273)  
[【爬虫成长之路】（五）【大众点评】浏览器扫码登录 + 油猴直接爬取数据](https://www.jianshu.com/p/3e8b69e73e0a)  
[【爬虫成长之路】（六）【大众点评】mitmproxy 修改 HttpOnly 字段获取完整 cookie+requests 请求数据](https://www.jianshu.com/p/ed86100c0be0)  
[【爬虫成长之路】（七）【大众点评】PC 微信小程序 + requests 爬取数据](https://www.jianshu.com/p/07be1478d0b8)  
**[【爬虫成长之路】（八）【大众点评】安卓 APP 爬虫](https://www.jianshu.com/p/191d8ba2af38)**

> 本章标题是安卓 APP 爬虫，说实话，如果爬虫的攻防对抗升级到了 APP 层面，这差不多是爬虫的最高形态了，之所以会升级到 APP 层面，如果有把前面的文章中的实验都做一遍，就不难发现，虽然爬虫能爬取数据，但是爬不了多少数据就会被封，所以如果需要更多的数据，就不得不转移到 APP 这个层面来。APP 能够爬取数据的前提是目标 APP 不强制要求登录，如果强制要求登录，那 APP 端也爬不动。幸运的是大众点评 APP 并没有强制要求登录，所以我们可以从 APP 端入手。

> 本文需要用到的工具：`Fiddler`、`IDA`，`JADX-gui`、`frida`、`objection`、已 root 安卓手机或安卓模拟器、大众点评 APP v10.41.15...  
> 本文需要用到的库：`requests`...

这里对这几个工具作个简单的介绍：

1.  Fiddler：HTTP/HTTPS 抓包软件，可重放请求，也可修改请求和响应；
2.  IDA：反编译工具，可以将二进制文件反编译成汇编或伪代码，还可以动态调试，功能十分强大；
3.  [JADX-gui](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fskylot%2Fjadx)：可以将 APK 直接反编译成 JAVA 代码，绝大部分都能还原回来，最重要的是可以 Go to Definition，还可以查找引用，这个非常好用；
4.  [frida](https://links.jianshu.com/go?to=https%3A%2F%2Ffrida.re%2F)：一款可以使用 JS 进行 HOOK 的全平台的框架，使用非常简单，无需配置；
5.  [objection](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fsensepost%2Fobjection)：基于 frida 开发的使用命令行进行 HOOK 的工具，不用写代码就能完成 HOOK 工作，十分好用。

**如果 frida 和 objection 不会使用的同学，可以参考下官方文档和下面的这些文章：**

1.  [Frida 安装和使用](https://www.jianshu.com/p/bab4f4714d98)
2.  [FRIDA 系列文章](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fr0ysue%2FAndroidSecurityStudy)
3.  [实用 FRIDA 进阶：内存漫游、hook anywhere、抓包](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.anquanke.com%2Fpost%2Fid%2F197657)
4.  [frida 入门总结](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.52pojie.cn%2Fthread-1128884-1-1.html)
5.  [一篇文章带你领悟 Frida 的精髓（基于安卓 8.1）](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.freebuf.com%2Farticles%2Fsystem%2F190565.html)
6.  [雷电模拟器安装 frida-server 教程](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.52pojie.cn%2Fthread-1344344-1-1.html)
7.  [Frida Android hook](https://links.jianshu.com/go?to=https%3A%2F%2Feternalsakura13.com%2F2020%2F07%2F04%2Ffrida%2F)
8.  [objection 常用方法](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.zhangkunzhi.com%2Findex.php%2Farchives%2F328%2F)
9.  [记一次 APP 加密通信后的分析过程](https://links.jianshu.com/go?to=https%3A%2F%2Fsec.mrfan.xyz%2F2019%2F07%2F28%2F%25E8%25AE%25B0%25E4%25B8%2580%25E6%25AC%25A1APP%25E5%258A%25A0%25E5%25AF%2586%25E9%2580%259A%25E4%25BF%25A1%25E5%2590%258E%25E7%259A%2584%25E5%2588%2586%25E6%259E%2590%25E8%25BF%2587%25E7%25A8%258B%2F)
10.  [Frida 构造 Java 函数所需的 Map<String, List<String>> 参数](https://www.jianshu.com/p/20bc3d2da714)

一、需求分析
======

这一篇总共需要爬取两个页面的数据，分别是：

1.  某用户的`评论详情`页面

二、获取目标页面的 URL
=============

由于是需要从 APP 入手，第一步还是抓包，对普通应用程序来说，只需要在手机上安装好证书，连上电脑的 WIFI，就能在 PC 端抓到手机的包，但是在抓大众点评的时候你会发现可以抓到一些提交日志信息相关的包，实际上有数据的包一个也抓不到，这是什么原因呢？

**面对 PC 抓不到手机的包，在手机和 PC 配置都没有出错的前提下，一般有以下这两种情况：**

> 1.  APP 检测到了使用代理了，直接拒绝工作，此时 APP 上不会更新任何数据；
> 2.  APP 所采用的不是 HTTP/HTTPS 通信协议，导致 Fiddler 抓不到包，但是使用 wireshark 可以看到明显是有产生数据包的。

具体是哪一种，在使用过程中，我们可以看到大众点评的 APP 明明有新的评论数据刷新出来，但是 Fiddler 里面确是一条数据都没有，而使用 wireshark 的时候却可以明显看到有数据包产生，这说明大众点评 APP 使用的不是 HTTP/HTTPS 协议，而是使用了 TCP 协议或者自定义的协议。在逆向大众点评的 APP 之前，我们先去 search 一下，在 [美团点评移动网络优化实践](https://links.jianshu.com/go?to=https%3A%2F%2Ftech.meituan.com%2F2017%2F03%2F17%2Fshark-sdk.html)看到了这张图，这也印证了大众点评并不是在使用 HTTP 通信。从实战报告中我们也可以知道，他们提供了 3 种方案用于完成通信。

![](http://upload-images.jianshu.io/upload_images/4537502-b2b48e21c4f18567.png) 美团完整的网络通道拓扑图

> 美团技术团队：图中网络通道 SDK 包含了三大通信通道：
> 
> 1.  CIP 通道：CIP 通道就是上文中提到的自建代理长连通道。CIP 是 China Internet Plus 的缩写，为美团点评集团的注册英文名称。App 中绝大部分的请求通过 CIP 通道中的 TCP 子通道与长连服务器（CIP Connection Server）通信，长连服务器将收到的请求代理转发到业务服务器（API Server）。由于 TCP 子通道在一些极端情况下可能会无法工作，我们在 CIP 通道中额外部署了 UDP 子通道和 HTTP 子通道，其中 HTTP 子通道通过公网绕过长连服务器与业务服务器进行直接请求。CIP 通道的平均端到端成功率目前已达 99.7%，耗时平均在 350 毫秒左右。
> 2.  WNS 通道：出于灾备的需要，腾讯的 WNS 目前仍被包含在网络通道 SDK 中。当极端情况发生，CIP 通道不可用时，WNS 通道还可以作为备用的长连替代方案。
> 3.  HTTP 通道：此处的 HTTP 通道是在公网直接请求 API Server 的网络通道。出于长连通道重要性的考虑，上传和下载大数据包的请求如果放在长连上进行都有可能导致长连通道的拥堵，因此我们将 CDN 访问、文件上传和频繁的日志上报等放在公网利用 HTTP 短连进行请求，同时也减轻代理长连服务器的负担。

到此为止我们有两种方法可以完成我们的爬虫：

1.  使用 TCP 通道进行爬取，需要逆向 APP；
2.  使用 HTTP 通道；

但是在 TCP 通道可用的情况下，APP 是不会采用 HTTP 通信的（上文提到，HTTP 通常只用来上传日志和下载图片），有数据的内容不会通过 HTTP 进行传输，所以我们可以想办法阻塞掉 TCP 通道。一开始我选用的方案是在 wireshark 中查看目标服务器的 IP，然后在 windows 的防火墙里封禁相应的 IP，在尝试多次之后，方向他的 IP 实在是太多了。。。大概禁了七八个吧，然后发现 APP 上的数据不更新了，可能 HTTP 也是走的这些通道吧，或者根本就没降级到 HTTP。。。这个我不确定，没仔细看，有兴趣的同学可以自行验证。

那如果需要抓到 HTTP 的包，就必须对 APP 进行逆向了，这里我使用的是 JADX-gui 这款软件，电脑内存要大些才好，真的是有多少内存就吃多少，最少也留个 3G 的空闲内存吧。 这里怎么逆向 APP 我就不展开讲解了，网上有蛮多的教程，如果不会的话自行去百度一下吧。这个说个比较重要的思路，如果老版本的 APP 可以使用，那就尽量从老版本的 APP 开始入手，这样遇到各种加密和混淆的概率低一些，也方便入手。

在逆向完成后，能够看到 jadx-gui 反编译后的 JAVA 代码了，现在要做的就是根据一些关键字去定位到相应的代码，比如我们知道点评的域名信息是`**.dianping.com/**`，就可以从中搜索了，**定位代码和理清其逻辑关系，这一步是这篇教程中最费时的一步。**这里具体如何操作，我就不演示了，反正挺费时间的，需要有耐心。

**下面给几个关键的步骤，以便读者可以快速的获取到相关的接口：**  
我们知道传输的数据都是经过加密了的，所以我们可以从加密函数入手，利用 frida 来 HOOK 相应的接口即可，

```
// 源程序包路径：com.dianping.nvnetwork.tunnel.tool.c
  private static Key a(byte[] bArr) throws Exception {
        Object[] objArr = {bArr};
        ChangeQuickRedirect changeQuickRedirect = a;
        if (PatchProxy.isSupport(objArr, null, changeQuickRedirect, true, "862c9f994092c26eb84b9d83c437a3ac", RobustBitConfig.DEFAULT_VALUE)) {
            return (Key) PatchProxy.accessDispatch(objArr, null, changeQuickRedirect, true, "862c9f994092c26eb84b9d83c437a3ac");
        }
        return SecretKeyFactory.getInstance("DES").generateSecret(new DESKeySpec(bArr));
    }

    public static byte[] a(byte[] bArr, byte[] bArr2) throws Exception {
        Object[] objArr = {bArr, bArr2};
        ChangeQuickRedirect changeQuickRedirect = a;
        if (PatchProxy.isSupport(objArr, null, changeQuickRedirect, true, "f7a471aa6ce2c3ab09d975bac8d088d0", RobustBitConfig.DEFAULT_VALUE)) {
            return (byte[]) PatchProxy.accessDispatch(objArr, null, changeQuickRedirect, true, "f7a471aa6ce2c3ab09d975bac8d088d0");
        }
        Key a2 = a(bArr2);
        Cipher instance = Cipher.getInstance("DES");
        instance.init(2, a2);
        return instance.doFinal(bArr);
    }

    // 通过对所有的加密函数进行HOOK，最终发现所有的URL相关加密都会经过这个函数，因此HOOK这个函数
    public static byte[] b(byte[] bArr, byte[] bArr2) throws Exception {
        Object[] objArr = {bArr, bArr2};
        ChangeQuickRedirect changeQuickRedirect = a;
        if (PatchProxy.isSupport(objArr, null, changeQuickRedirect, true, "791dd5351de3cfac5e41752ae0c020dc", RobustBitConfig.DEFAULT_VALUE)) {
            return (byte[]) PatchProxy.accessDispatch(objArr, null, changeQuickRedirect, true, "791dd5351de3cfac5e41752ae0c020dc");
        }
        Key a2 = a(bArr2);
        Cipher instance = Cipher.getInstance("DES");
        instance.init(1, a2);
        return instance.doFinal(bArr);
    }
```

```
//frida js HOOK 代码
function bin2string(array){
    var result = "";
    for(var i = 0; i < array.length-1; ++i){
        result+= (String.fromCharCode(array[i]));
    }
    return result;
}

function main(){
    Java.perform(function x() {

        var c = Java.use("com.dianping.nvnetwork.tunnel.tool.c");

        c.b.overload("[B","[B").implementation = function(bArr, bArr2){
            var result = this.b(bArr, bArr2);
            var str = bin2string(bArr);
            if(str.includes("pragma-unionid") && str.includes("pragma-dpid") && str.includes("mainid")){
                console.log(str);// 打印传入待加密的参数
            }
            return result;
        }
    }
}

setImmediate(main)
```

通过 HOOK DES 加密函数，最终找到相关接口及 Header 信息，Header 及其他被加密的信息如下：

```
{
        "m": "GET",
        "h": {
            "pragma-device": "4***3",//IMEI
            "network-type": "wifi",
            "pragma-os": "MApi 1.4 (com.dianping.v1 10.36.3 om_sd_** NXT-DL00; Android 8.0)",//类似于UserAgent
            "pragma-uuid": "11139***2",//UUID可用UUID生成算法生成
            "pragma-unionid": "158bca***83",// 需要向服务器请求获得
            "User-Agent": "MApi 1.4 (com.dianping.v1 10.36.3 om_sd_** NXT-DL00; Android 8.0)",
            "pragma-dpid": "158bca***83",// 需要向服务器请求获得
            "picasso": "no-js",
            "M-SHARK-TRACEID": "11158b***a"//本地生成
        },
        "u": "http://***", //  目标URL
        "i": "11***4"// 请求序号
    }
```

其实到这里虽然拿到了这些重点参数，但是此时仍然是建立了 TCP 连接而不是 HTTP 连接，所以，直接用这些数据去请求的话必然是失败的，这里就要迫使 APP 降级采用 HTTP 连接了，这里参考了 github 的一位大佬的代码，我找了好一会没找到入口在哪。。。

```
//frida JS代码，绕过CIP和WNS代理，直接走HTTP通道
var nvnetwork_g = Java.use("com.dianping.nvnetwork.g");
nvnetwork_g.g.overload().implementation = function(){
    console.log("----------------------------- Hook g()---------------------------");
    return 3;
}
```

这样就可以使用 Fiddler 抓包了，最终结果如下。

**评论详情 URL 接口：**

```
# 评论详情URL接口：
http://mapi.dianping.com/mapi/note/getfeedcontent.bin?***
```

三、请求头 Header、URL 参数解析
=====================

**评论 URL 参数**

这里有两个重点参数，分别是`mainid` 和 `cx`，这个`cx`和小程序里的`cx`是不一样的，这点需要注意一下。

<table><thead><tr><th>序号</th><th>名称</th><th>值</th><th>说明</th></tr></thead><tbody><tr><td>1</td><td><code>mainid</code></td><td>***</td><td>评论 ID</td></tr><tr><td>2</td><td>feedtype</td><td>1</td><td>评论类型</td></tr><tr><td>3</td><td>lng</td><td>***</td><td>经度</td></tr><tr><td>4</td><td>lat</td><td>***</td><td>纬度</td></tr><tr><td>5</td><td>displaypattern</td><td>2</td><td>显示模式，固定</td></tr><tr><td>6</td><td>bubblepagetype</td><td>null</td><td>-，固定</td></tr><tr><td>7</td><td><code>cx</code></td><td>***</td><td>加密生成的参数，带了时间戳，重点参数</td></tr><tr><td>8</td><td>pagecityid</td><td>1</td><td>-，固定</td></tr><tr><td>9</td><td>optimus_partner</td><td>76</td><td>-，固定</td></tr><tr><td>10</td><td>optimus_risk_level</td><td>71</td><td>风险等级，固定</td></tr><tr><td>11</td><td>optimus_code</td><td>10</td><td>-，固定</td></tr><tr><td>12</td><td>picsize</td><td>...</td><td>一些分辨率信息，可适当调整</td></tr></tbody></table>

**Header 参数**  
看上面的注释就好了，就不重复说明了，说下重点参数。  
在 Header 里有几个重点参数，分别是：`pragma-device`、`pragma-os`、`pragma-uuid`、`pragma-unionid`、`pragma-dpid`、`M-SHARK-TRACEID`、`i`。

<table><thead><tr><th>序号</th><th>名称</th><th>值</th><th>说明</th></tr></thead><tbody><tr><td>1</td><td><code>pragma-device</code></td><td>4***3</td><td>设备的 IMEI</td></tr><tr><td>2</td><td><code>pragma-os</code></td><td>MApi 1.4 (com.dianping.v1 10.36.3 om_sd_** NXT-DL00; Android 8.0)</td><td>类似于 UA</td></tr><tr><td>3</td><td><code>pragma-uuid</code></td><td>11139***2</td><td>本地生成</td></tr><tr><td>4</td><td><code>pragma-unionid</code></td><td>158bca***83</td><td>需要提交请求换取</td></tr><tr><td>5</td><td><code>pragma-dpid</code></td><td>158bca***83</td><td>同<code>pragma-unionid</code></td></tr><tr><td>6</td><td><code>M-SHARK-TRACEID</code></td><td>11158b***a</td><td>本地算法生成，需要看源码</td></tr><tr><td>7</td><td><code>i</code></td><td>11***4</td><td>请求序号，依此递增</td></tr></tbody></table>

四、请求头 Header、URL 参数构造
=====================

首先构造 Header 里的参数，`pragma-device`是`IMEI`，这个比较容易构造，`pragma-os`类似于`UserAgent`，也比较好构造，`pragma-uuid`是用`UUID`算法生成的，剩下的`pragma-unionid`和`pragma-dpid`其实可以一致，或者`pragma-dpid`可以留空，那最关键的就是获取到`pragma-unionid`和`M-SHARK-TRACEID`了，那如何构造`pragma-unionid`和`M-SHARK-TRACEID`呢？这就需要看源码了，经过一番定位查找，结果如下。

**unionid 参数生成，JAVA 源码：**

```
// 源程序获取unionid函数，包路径：com.meituan.android.common.unionid.oneid.OneIdHelper
private static void getOneIdByNetwork(final DeviceInfo deviceInfo, final OneIdNetworkHandler oneIdNetworkHandler, final List<IOneIdCallback> list, String str, final String str2) {
    Object[] objArr = {deviceInfo, oneIdNetworkHandler, list, str, str2};
    ChangeQuickRedirect changeQuickRedirect2 = changeQuickRedirect;
    if (PatchProxy.isSupport(objArr, null, changeQuickRedirect2, true, "24218b6da0953568d0f2ba37b8d38f2d", RobustBitConfig.DEFAULT_VALUE)) {
        PatchProxy.accessDispatch(objArr, null, changeQuickRedirect2, true, "24218b6da0953568d0f2ba37b8d38f2d");
    } else if (deviceInfo == null || oneIdNetworkHandler == null) {
        Log.e(TAG, "getoneIdByNetwork: one of the parameters is null");
    } else {
        _oneid_request(deviceInfo, oneIdNetworkHandler, list, str, str2, 1);
        try {
            MonitorManager.addEvent(deviceInfo.stat, "oaid", 0, true);
            OaidManager.getInstance().getOaid(sContext, new OaidCallback2() {
                /* class com.meituan.android.common.unionid.oneid.OneIdHelper.AnonymousClass1 */
                public static ChangeQuickRedirect changeQuickRedirect;

                @Override // com.meituan.android.common.unionid.oneid.oaid.OaidCallback
                public void onSuccuss(boolean z, String str, boolean z2) {
                }
...

public static void _oneid_request(DeviceInfo deviceInfo, OneIdNetworkHandler oneIdNetworkHandler, List<IOneIdCallback> list, String str, String str2, int i) {
    String request = OneIdNetworkHandler.request(sContext, str, deviceInfo, str2, i);
    if (!TextUtils.isEmpty(request)) {
        if (!TextUtils.isEmpty(lastOneid) && !lastOneid.equals(request)) {
            JSONObject jSONObject = new JSONObject();
            try {
                jSONObject.put("req", deviceInfo.toString());
                jSONObject.put("url", str);
                jSONObject.put("new", request);
                jSONObject.put("old", lastOneid);
                LogMonitor.watch(LogMonitor.ONEID_CHANGE_TAG, "", jSONObject);
            } catch (Exception e) {
                c.a(e);
                e.printStackTrace();
            }
        }
...
```

**unionid 参数生成，python 版本：**

```
import time
import random
import uuid
import json
import requests


class UnionidHelper:
    '''
    UnionidHelper 是用来生成获取unionid所需参数的
    [注意]初始化一次只能获取一次里面的参数，否则参数将是重复的，无法生成新的unionid
    '''

    def __init__(self, device_info=None, brand=None,model=None,app_source=None,app_version=None,imei1=None,imei2=None,
                androidId=None,osName=None,os_version=None,serialNumber=None,bluetoothMac=None,wifiMac=None):
        '''
        device_info 字典中含有其他参数时，则其他参数可以不用传入
        '''
        self.brand = brand if 'brand' not in device_info else device_info['brand']
        self.model = model if 'model' not in device_info else device_info['model']
        self.app_source = app_source if 'app_source' not in device_info else device_info['app_source']  # 应用来源(不带前缀om_sd_)
        self.app_version = app_version if 'app_version' not in device_info else device_info['app_version']
        self.imei1 = imei1 if 'imei1' not in device_info else device_info['imei1']
        self.imei2 = imei2 if 'imei2' not in device_info else device_info['imei2']
        self.androidId = androidId if 'androidId' not in device_info else device_info['androidId'] # len = 16
        self.osName = osName if 'osName' not in device_info else device_info['osName'] # 手机中的版本号
        self.os_version = os_version if 'os_version' not in device_info else device_info['os_version']  # android version
        self.serialNumber = serialNumber if 'serialNumber' not in device_info else device_info['serialNumber']
        self.bluetoothMac = bluetoothMac if 'bluetoothMac' not in device_info else device_info['bluetoothMac']
        self.wifiMac = wifiMac if 'wifiMac' not in device_info else device_info['wifiMac']

        self.localid = LocalId()
        
        self.url_unionid_register = 'http://api-unionid.meituan.com/unionid/android/register'

        self.header = {
            "Accept-Charset": "UTF-8",
            "uuidRequestId": self.localid.gen_localId(),
            "uuidSessionId": self.localid.gen_localId(),
            "retrofit_exec_time": str(int(time.time()*1000)),
            "Accept-Encoding": "gzip",
            "Content-Type": "application/json;charset=UTF-8",
            "Content-Length": '0',
            "User-Agent": f"Dalvik/2.1.0 (Linux; U; Android {os_version}; {self.model} Build/{osName})",  # 待修改
            "Host": "api-unionid.meituan.com",
            "Connection": "Keep-Alive"
        }

        self.logInfo = {
            "processName": "com.dianping.v1",
            "events": [
            {
                "markKey": "buCallStart",
                "markValue": 121,
                "incrementalId": 0,
                "opName": 0,
                "threadName": "Aurora#2",
                "timestamp": int(time.time()*1000),
                "uptimeMillis": 37244436+random.randint(-5,10)
            },
            {
                "markKey": "dpid",
                "markValue": 130,
                "incrementalId": 1,
                "opName": 0,
                "threadName": "Aurora#2",
                "timestamp": int(time.time()*1000),
                "uptimeMillis": 37245536+random.randint(-5,10)
            },
            ...
            ],
            "rtt": {
            "sessionId": ""
            }
        }

        self.data = {
            "appInfo": {
                "app": "com.dianping.v1",
                "version": self.app_source,
                "appName": "dianping_nova",
                "sdkVersion": "1.16.11",
                "userId": "",
                "downloadSource": f"om_sd_{self.app_source}"     # 待修改
            },
            "idInfo": {
                "localId": self.localid.gen_localId(),
                "unionId": "",
                "uuid": "",
                "dpid": "",
                "requiredId": str(random.randint(1,2))       # 1:registerOrUpdate, 2:startDpid, 4:registerOrUpdateUuid
            },
            "logInfo": json.dumps(self.logInfo),
            "environmentInfo": {
                "platform": "android",
                "osName": self.osName,    # 待修改
                "osVersion": self.os_version,
                "clientType": "6"
            },
            "deviceInfo": {
                "keyDeviceInfo": {
                "imei1": self.imei1,
                "imei2": self.imei2,
                "meid": "",
                "androidId": self.androidId,  # 6db88d1534ef1089
                "oaid": "",
                "appid": {
                    "share": "",    # raw: AndroidMuMu$6db88d1534ef1089
                    "local": {
                    "oldid": f"{self.brand}{self.model}${androidId}",  # raw: AndroidMuMu$6db88d1534ef1089
                    "newid": ""
                    }
                }
                },
                "secondaryDeviceInfo": {
                "serialNumber": self.serialNumber,   # ZX1G42CPJD
                "bluetoothMac": self.bluetoothMac.lower(),
                "wifiMac": self.wifiMac.lower(),
                "simulateId": "",
                "uuid": uuid.uuid4().__str__()
                },
                "brandInfo": {
                "brand": self.brand,
                "deviceModel": self.model
                }
            },
            "communicationInfo": {
                "jntj": "",
                "jddje": "",
                "nop": "unknown"
            },
            "mark": json.dumps({
                "dpid": 9,
                "unionId": 9,
                "appid_share": 131,
                "appid_local": 130
            })
        }


if __name__ == '__main__':
    unionid_helper = UnionidHelper()

    session = requests.session()
    session.headers.clear()
    session.headers.update(unionid_helper.header)
    res = session.post(url=unionid_helper.url_unionid_register, data=json.dumps(unionid_helper.data))

    res_data = json.loads(res.text)
    code = res_data['code']
    if code == 0:
        print(res_data['data']['unionId'])
    print(res.content)
```

**M-SHARK-TRACEID 参数生成算法， python 版本：**

```
def gen_M_SHARK_TRACEID(pragma_unionid):
    '''
    M-SHARK-TRACEID : 113888...6253a7a3161510...360d09cda
    结构：11 388...62 53a7a3 161510...360 d09cda
         [fix] [unionid] [uuid(pre-6bit)] [timestamp(ms)] [uuid(pre-6bit)]
    '''
    prefix = '11'

    uuid_pre = uuid.uuid4().__str__()[:6]
    uuid_end = uuid.uuid4().__str__()[:6]
    timestamp = str(int(time.time()*1000))
    return prefix + pragma_unionid + uuid_pre + timestamp + uuid_end
```

**localId 生成算法 (unionid 获取时需要这个参数)，JAVA 源码：**

```
// 包路径：com.meituan.android.common.unionid.oneid.util.TempIDGenerator
public class TempIDGenerator {
    public static String generate() {
        SecureRandom secureRandom = new SecureRandom();
        byte[] bArr = new byte[50];
        byte[] bArr2 = new byte[24];
        byte[] bArr3 = new byte[24];
        secureRandom.nextBytes(bArr2);
        secureRandom.nextBytes(bArr3);
        for (int i = 0; i < bArr2.length; i++) {
            bArr2[i] = (byte) (bArr2[i] & 15);
            bArr3[i] = (byte) (bArr3[i] & 15);
        }
        System.arraycopy(bArr2, 0, bArr, 0, bArr2.length);
        System.arraycopy(bArr3, 0, bArr, 26, bArr3.length);
        handleBytes(bArr2);
        handleBytes(bArr3);
        byte checker = getChecker(bArr2);
        byte checker2 = getChecker(bArr3);
        bArr[24] = checker;
        bArr[25] = checker2;
        return byteArrayToHexString(bArr);
    }

    private static void handleBytes(byte[] bArr) {
        for (int i = 0; i < bArr.length; i += 2) {
            bArr[i] = (byte) (bArr[i] * 2);
            while (bArr[i] >= 10) {
                bArr[i] = (byte) ((bArr[i] % 10) + ((bArr[i] / 10) % 10));
            }
        }
    }

    private static byte getChecker(byte[] bArr) {
        int i = 0;
        for (byte b : bArr) {
            i += b;
        }
        byte b2 = (byte) (10 - ((byte) (i % 10)));
        if (b2 == 10) {
            return 0;
        }
        return b2;
    }

    private static String byteArrayToHexString(byte[] bArr) {
        StringBuffer stringBuffer = new StringBuffer(bArr.length);
        for (byte b : bArr) {
            stringBuffer.append(Integer.toHexString(b));
        }
        return new String(stringBuffer);
    }
}
```

**localId 生成算法 (unionid 获取时需要这个参数)，python 版本：**

```
import time
import random
import uuid
import json
import requests


class LocalId:
    '''
    大众点评localId生成算法(对应:TempIDGenerator.generate())
    '''
    def __init__(self):
        self.bArr2 = None  # [random.randint(0,15) for i in range(24)]
        self.bArr3 = None  # [random.randint(0,15) for i in range(24)]
        self.bArr = None  # self.bArr2 + [0,0]+ self.bArr3

    def handleBytes(self, bArr):
        '''
        func:将偶数位的数先乘2再变成小于10的数
        '''
        for i, v in enumerate(bArr):
            if i % 2 != 0:
                continue
            bArr[i] *= 2
            while bArr[i] >= 10:
                bArr[i] = bArr[i] % 10 + bArr[i] // 10 % 10
        return bArr

    def getChecker(self, bArr):
        _sum = sum(bArr)
        b2 = 10 - _sum % 10
        return b2 if b2 != 10 else 0

    def gen_localId(self):
        self.bArr2 = [random.randint(0,15) for i in range(24)]
        self.bArr3 = [random.randint(0,15) for i in range(24)]
        self.bArr = self.bArr2 + [0,0]+ self.bArr3

        self.bArr2 = self.handleBytes(self.bArr2)
        self.bArr3 = self.handleBytes(self.bArr3)
        check2 = self.getChecker(self.bArr2)
        check3 = self.getChecker(self.bArr3)

        self.bArr[24] = check2
        self.bArr[25] = check3

        bArr_str = ''.join('{:1x}'.format(x) for x in self.bArr)
        return bArr_str
```

**cx 生成算法，JAVA 源码：**  
省略了...

**cx 生成算法，python 版本：**

由于需要用到的参数过多，这里就不贴出来了了，太影响阅读体验了，就给个大致思路吧：

> **加密过程**： 待加密字符串 (str) -> 编码 (byte) -> des 加密 (byte) -> base64 编码 (byte) -> url 编码 (str) -> 密文 (str)  
> **解密过程**： 密文 (str) -> url 解码 (str) -> base64 解码 (byte) -> des 解密 (byte) -> 解码成字符串 (str)

**生成 cx 时用到的 DES 加解密算法**：

```
from pyDes import des, CBC, PAD_PKCS5
import base64
import urllib.parse
import json
import uuid
import time


class DES_Encrypt:

    def __init__(self):
        # 秘钥
        self.KEY = 'k***'

    def des_encrypt_byte(self, content):
        """
        DES 加密
        :param s: 原始字符串
        :return: 加密后字符串，byte
        """
        secret_key = self.KEY  # 密码
        iv = secret_key  # 偏移
        # secret_key:加密密钥，CBC:加密模式，iv:偏移, padmode:填充
        des_obj = des(secret_key, CBC, iv, pad=None, padmode=PAD_PKCS5)
        # 返回为字节
        secret_bytes = des_obj.encrypt(content, padmode=PAD_PKCS5)
        # 返回为16进制
        # return binascii.b2a_hex(secret_bytes)
        return secret_bytes

    def des_descrypt_byte(self, content):
        """
        DES 解密
        :param s: 加密后的字符串，16进制
        :return:  解密后的字符串
        """
        secret_key = self.KEY
        iv = secret_key
        des_obj = des(secret_key, CBC, iv, pad=None, padmode=PAD_PKCS5)
        decrypt_str = des_obj.decrypt(content, padmode=PAD_PKCS5)
        return decrypt_str
```

> 到这里，发起请求的全部重点参数其含义以及如何生成的就都清楚了，其中的重点参数：`pragma-unionid`、`pragma-dpid`、`M-SHARK-TRACEID`、`cx`、`localId`的生成算法都给出来了，现在就可以去发起请求了。

五、response 响应解析
===============

按照以往的 WEB 爬虫来说，能够成功发起请求，爬虫基本上就算完成了，但是对于 APP 爬虫，尤其是这种做了大量加密的 APP 来说，其返回的 response 必然也会进行相应的加密，这里的解密也参考了另一位 github 大佬的代码，点评 APP 把解密的算法放到了 so 库里面，所以这时候借助 JADX-gui 就行不通了，我们先来看下 JADX-gui 中解出来的解密算法：

```
// 包路径：com.dianping.util.NativeHelper
public class NativeHelper {
    public static final boolean a;

    private static native boolean a();

    public static native boolean nd(byte[] bArr, byte[] bArr2, byte[] bArr3, byte[] bArr4);

    public static native byte[] ndug(byte[] bArr, byte[] bArr2, byte[] bArr3);

    public static native boolean ne(byte[] bArr, byte[] bArr2, byte[] bArr3, byte[] bArr4);

    public static native byte[] nug(byte[] bArr);

    static {
        boolean z;
        b.a("1af3893fc7dcb0905311776314368f4d");
        try {
            if (!aa.a("nh", NativeHelper.class)) {
                System.loadLibrary(b.b("nh"));
            }
            z = a();
        } catch (Throwable th) {
            c.a(th);
            ab.c("failed to load native helper");
            z = false;
        }
        a = z;
    }
}
```

里面没有算法的具体实现，说明其具体实现在 so 层，这时候就要用 IDA 来查看其使用的是什么算法了。

在 APP 解压出来的文件中，找到`libnh.so`这个文件（从`System.loadLibrary(b.b("nh"))`这里可以知道），查看其导出函数，再一顿操作转成可读性稍好的 C 代码，这里明显能从函数名看出来使用了 AES/CBC 加密，

![](http://upload-images.jianshu.io/upload_images/4537502-d8e71a1ed4d1ac53.png) ne 加密函数

![](http://upload-images.jianshu.io/upload_images/4537502-0f179898de005653.png) ndug 解密函数

关于 AES 加解密，直接调库就好了；关于 AES 的秘钥和偏移，可以从 JADX-gui 反编译后的文件中找到，这里的思路是根据加解密函数去查找，看哪里调用了这个函数，传入的参数是哪里来的，就能找到了。

到这里，很大一部分工作就完成了，如果实际测试一下可以发现，解密后的数据仍然不是我们需要的数据，还需要进一步进行处理，如果源码追踪是仔细一点，可以看到这一步就是做了一个变量名的映射，然而重新建立这个映射关系还是需要花点时间的，源程序的部分映射关系如下：

```
//包路径：com.dianping.model.AdLog
...
@SerializedName("feedback")
public String a;
@SerializedName("impUrl")
public String b;
@SerializedName("clickUrl")
public String c;
@SerializedName("thirdpartyMonitorImpUrls")
public String[] d;
@SerializedName("thirdpartyMonitorClickUrls")
public String[] e;
@SerializedName("ext")
public String f;

...
public void writeToParcel(Parcel parcel, int i) {
    parcel.writeInt(2633);
    parcel.writeInt(this.isPresent ? 1 : 0);
    parcel.writeInt(35360);
    parcel.writeString(this.f);
    parcel.writeInt(31004);
    parcel.writeStringArray(this.e);
    parcel.writeInt(53501);
    parcel.writeStringArray(this.d);
    parcel.writeInt(3264);
    parcel.writeString(this.c);
    parcel.writeInt(43874);
    parcel.writeString(this.b);
    parcel.writeInt(7952);
    parcel.writeString(this.a);
    parcel.writeInt(-1);
}
```

重建映射关系后，就能解析出完成的 JSON 数据了，下面给出一个简化版本。

> ### 注：
> 
> 下面`response`解码部分本来是不打算放出来的，一是原创不是我，二是影响阅读体验，但问的同学是在太多了，所以考虑下，还是放出来了。

```
def decrypt_aes(key, iv, content):
    generator = AES.new(key=key, mode=AES.MODE_CBC, iv=iv)
    decrypt = generator.decrypt(content)
    return decrypt

def decode_aes(content, model=None):
    aes_data = decrypt_aes(key=key, iv=iv, content=content)
    ungzip_data = gzip.decompress(aes_data)
    return ungzip_data

# 请求数据并做AES解密，再做变量名的重映射
res = session.get(url_user_comment, proxies=None, timeout=20)
body = decode_aes(res.content)
if res.status_code == 200:
    res_data = json.dumps(decode_model(body), indent=4)
    res_data = json.loads(res_data)
```

像下面这种`model`文件是有相互依赖关系的，有时解析某个`model`内的数据时会依赖其他`model`的数据，这里最好就全部解析一遍。如果需要爬取的内容较多，也需要编写多个类似于下面的`model`文件，可以用程序来处理，将`java`的`model`转为`python`的`model`。

```
# encoding: utf-8

from model import BaseModel, add_model

@add_model(0x9bc3)
class AdLog(BaseModel):
    
    field_map = {'a': 'feedback', 'b': 'impUrl', 'c': 'clickUrl', 'd': 'thirdpartyMonitorImpUrls', 'e': 'thirdpartyMonitorClickUrls', 'f': 'ext'}

    def j_flag_2633(self):
        """
        0xa49 -> :sswitch_0
        :return:
        """
        self.result[self.field_map.get('isPresent', 'isPresent')] = self.archive_d_b()
    def j_flag_35360(self):
        """
        0x8a20 -> :sswitch_1
        :return:
        """
        self.result[self.field_map.get('f', 'f')] = self.archive_d_g()
    def j_flag_31004(self):
        """
        0x791c -> :sswitch_2
        :return:
        """
        self.result[self.field_map.get('e', 'e')] = self.archive_d_n()
    def j_flag_53501(self):
        """
        0xd0fd -> :sswitch_3
        :return:
        """
        self.result[self.field_map.get('d', 'd')] = self.archive_d_n()
    def j_flag_3264(self):
        """
        0xcc0 -> :sswitch_4
        :return:
        """
        self.result[self.field_map.get('c', 'c')] = self.archive_d_g()
    def j_flag_43874(self):
        """
        0xab62 -> :sswitch_5
        :return:
        """
        self.result[self.field_map.get('b', 'b')] = self.archive_d_g()
    def j_flag_7952(self):
        """
        0x1f10 -> :sswitch_6
        :return:
        """
        self.result[self.field_map.get('a', 'a')] = self.archive_d_g()
```

> ##### 下面的 model 解码程序是 github 大佬的，本来只能兼容比较旧的版本，我修改后能兼容到我测试所用的版本，之后的版本没有继续测试了，没有大版本更新的话是可以不作修改继续使用的，具体情况请自行测试或回退几个版本测试。

```
# encoding: utf-8

import struct
import os
from io import BytesIO
import logging
import time

flag_model_map = {}


class BaseModel:
    field_map = {}

    def __init__(self, data):
        self.result = {}
        if isinstance(data, BytesIO):
            self.data = data
            self.raw_data = data
        else:
            self.raw_data = data
            self.data = BytesIO(data)

    def unpack(self, fmt, stream):
        size = struct.calcsize(fmt)
        buf = stream.read(size)
        try:
            return struct.unpack(fmt, buf)
        except struct.error as e:
            logging.error("数据不全:{}".format(buf))

    def main(self):
        self.result = self.archive_d_a_archive_c()
        return self.result

    def decode(self):
        while True:
            j_flag = self.archive_d_j()
            if j_flag < 1:
                break
            j_flag_func_name = 'j_flag_{}'.format(j_flag)
            if hasattr(self, j_flag_func_name):
                getattr(self, j_flag_func_name)()
            else:
                try:
                    self.archive_d_i()
                except ValueError as e:
                    logging.error("数据错误不解析了 j_flag:{} 当前model:{}".format(j_flag, self.__class__))
                    raise e

    def archive_d_b(self):
        flag, = self.unpack(">b", self.data)
        if flag == 0x54:
            data = 0x1
        elif flag in [0x46, 0x4e]:
            data = 0x0
        else:
            logging.error("archive_d_b抛错:unable to read boolean")
            raise ValueError()
        logging.info("找到bool:{}".format(data))
        return data

    def archive_d_c(self):
        flag, = self.unpack(">b", self.data)
        if flag == 0x49:
            data, = self.unpack(">i", self.data)
        elif flag == 0x4e:
            data = 0x00
        else:
            logging.error("archive_d_c抛错")
            raise ValueError()
        logging.info("找到int:{}".format(data))
        return data

    def archive_d_d(self):
        flag, = self.unpack(">b", self.data)
        if flag == 0x4c:
            data, = self.unpack(">q", self.data)
        elif flag == 0x4e:
            data = 0x0
        else:
            logging.error("archive_d_d抛错")
            raise ValueError()
        logging.info("archive_d_d找到string:{}".format(data))
        return data

    def archive_d_j(self):
        flag, = self.unpack(">b", self.data)
        if flag == 0x4d:
            data, = self.unpack(">h", self.data)
            data &= 0xffff
        elif flag == 0x5a:
            data = 0x00
        else:
            logging.error("archive_d_j抛错")
            raise ValueError()
        logging.info("当前model{}".format(self.__class__))
        return data

    def archive_d_e(self):
        flag, = self.unpack(">b", self.data)

        if flag == 0x44:
            data, = self.unpack(">d", self.data)
        elif flag == 0x4e:
            data = 0x0
        else:
            logging.error("archive_d_e抛错")
            raise ValueError()
        logging.info("当前model{}".format(self.__class__))
        return data
 
    def archive_d_g(self):
        flag, = self.unpack(">b", self.data)
        if flag == 0x53:
            length, = self.unpack(">h", self.data)
            length = 0xffff & length
            data, = self.unpack(">{}s".format(length), self.data)
        elif flag == 0x42:
            length, = self.unpack(">i", self.data)
            # length = 0xffff & length
            data, = self.unpack(">{}s".format(length), self.data)
        elif flag == 0x4e:
            data = b""
        else:
            logging.error("archive_d_g抛错")
            raise ValueError()
        logging.info("找到string:{}".format(data.decode()))
        return data.decode()

    def archive_d_n(self):
        data = []
        flag, = self.unpack(">b", self.data)
        if flag == 0x4e:
            data = [""]
        elif flag == 0x41:
            length, = self.unpack(">h", self.data)
            length = 0xffff & length
            for i in range(length):
                data.append(self.archive_d_g())
        else:
            logging.error("archive_d_n抛错")
            raise ValueError()
        logging.info("找到string:{}".format(data))
        return data

    def archive_d_i(self):
        flag, = self.unpack(">b", self.data)
        if flag == 0x41:
            length, = self.unpack(">h", self.data)
            length = length & 0xffff
            for i in range(length):
                self.archive_d_i()
        elif flag == 0x44:
            self.unpack(">d", self.data)
        elif flag == 0x49:
            self.unpack(">i", self.data)
        elif flag == 0x4c:
            self.unpack(">q", self.data)
        elif flag == 0x4f:
            self.unpack(">h", self.data)
            while self.archive_d_j() > 0:
                self.archive_d_i()
        elif flag == 0x53:
            position, = self.unpack(">h", self.data)
            position = (position & 0xffff) + self.data.tell()
            self.data.seek(position)
        elif flag == 0x55:
            self.unpack(">i", self.data)
        elif flag in [0x46, 0x4e, 0x54, ]:
            pass
        elif flag in [0x42, 0x43, 0x45, 0x47, 0x48, 0x4a, 0x4b, 0x4d, 0x50, 0x51, 0x52]:
            raise ValueError("unable to skip object:")

    def archive_d_a_archive_c(self):
        flag, = self.unpack(">b", self.data)
        if flag == 0x4e:
            logging.info("创建了一个空对象")
            return {}
        elif flag == 0x4f:
            data, = self.unpack(">h", self.data)
            data &= 0xffff
            model_class = flag_model_map.get(data, None)
            if model_class:
                model_instance = model_class(self.data)
                model_instance.decode()
                return model_instance.result
            else:
                logging.error("archive_d_a_archive_c未找到此model：{}".format(hex(data)))
                # raise ValueError()
                # return f'"archive_d_a_archive_c未找到此model：{format(hex(data))}"'
                print(f'"archive_d_a_archive_c未找到此model：{format(hex(data))}"')
        else:
            logging.error("archive_d_a_archive_c抛错")
            raise ValueError()

    def archive_d_b_archive_c(self):
        result = []
        flag, = self.unpack(">b", self.data)
        if flag == 0x4e:
            logging.info("创建空对象")
            return []
        elif flag == 0x41:
            length, = self.unpack(">h", self.data)
            length = 0xffff & length
            for i in range(length):
                data = self.archive_d_a_archive_c()
                logging.info("创建了一个对象:{}".format(data))
                result.append(data)
            return result
        else:
            logging.error("抛错")
            raise ValueError()


def add_model(flag):
    def wrapper(cls):
        if flag_model_map.get(flag):
            # raise ValueError("model已存在:{}".format(flag))
            print("model已存在:{}".format(flag))
            return cls
        flag_model_map[flag] = cls
        return cls

    return wrapper


def decode_model(data):
    """

    :param data:
    :return: dict
    """
    if not isinstance(data, bytes):
        data = bytes.fromhex(data)
    logging.debug("需要解密的body:{}".format(data.hex()))
    basemodel = BaseModel(data)
    basemodel.main()
    result = basemodel.result
    return result


def import_all_model():
    all_list = os.listdir(os.path.dirname(__file__))
    for i in all_list:
        if "__" not in i and ".py" in i:
            __import__("model." + i.replace(".py", ""))

t1 = time.time()
print('正在加载model...')
import_all_model()
print(f'[{round(time.time()-t1, 2)}s] model加载完成!')
```

六、优缺点分析
=======

<table><thead><tr><th>序号</th><th>优点</th><th>缺点</th></tr></thead><tbody><tr><td>1</td><td>程序运行更快</td><td>参数构造麻烦</td></tr><tr><td>2</td><td>-</td><td>response 解析麻烦</td></tr><tr><td>3</td><td>-</td><td>需要对 APP 进行逆向</td></tr><tr><td>4</td><td>-</td><td>需要理清 APP 的各个模块间的逻辑关系</td></tr></tbody></table>

七、结语
====

这是 APP 爬虫，对于新手来说还是难度很大的，对于有逆向 APP 经验的同学来说，这里复杂的可能就是理清逻辑关系了，文中给出了关键的思路和代码，有了这些思路，相信对于想入门高阶爬虫的同学来说，多少还是有点帮助的。但是在此再次声明，虽然 APP 爬虫可以大规模爬取数据，但是最好还是不要给对方服务器造成压力，影响其服务的正常运行，做任何爬虫都是这样的，主要还是以学习爬虫思想为主，了解爬虫的常见对抗升级方式。

注：
==

> 1.  如果您不希望我在文章提及您文章的链接，或是对您的服务器造成了损害，请联系我对文章进行修改；
> 2.  本文仅爬取公开数据，不涉及到用户隐私；