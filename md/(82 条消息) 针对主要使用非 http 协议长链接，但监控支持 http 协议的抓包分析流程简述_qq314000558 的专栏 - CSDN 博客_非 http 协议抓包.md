> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq314000558/article/details/105958847)

分析
--

### 1、Charles 抓包

一开始爬 **美团外卖 App** 我是直接 Charles 上手就干的，但我抓了一天都没抓到有用的数据我就开始找资料，遗憾的是网上没有一篇关于 **美团外卖 App** 抓包分析的文章，我是真的一篇都没看到（这里指的是移动 app，不是网页）

不过好在在我查找资料的过程也并非无任何收获，我得知美团使用了一种叫 “**[移动长连接](https://baike.baidu.com/item/%E9%95%BF%E8%BF%9E%E6%8E%A5/568486?fr=aladdin)** “ 的技术导致我抓不到包；

接着我在网上找了关于该名词的解释以及 mt 发表的一篇文章：[移动网络优化实践](https://tech.meituan.com/2017/03/17/shark-sdk.html)，用我所理解的话来说就是： **打开 APP 的时候移动端和服务器建立起 tcp 连接后，这个连接就不断开了，后续的请求和接收都走该通道，这个 tcp 长连接一般都是会使用自定义的协议，而非 http，所有普通的抓包软件都无法抓到这类请求（如有误请指正）**。

### 2、长连接分析

文章里有这么一段：  
**“当 TCP 通道无法建立或者发生故障时，可以使用 UDP 面向无连接的特性提供另一条请求通道，或者绕过代理长连服务器之间向业务服务器发起 HTTP 公网请求”**；  
![](https://img-blog.csdnimg.cn/20200506205414900.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxMzE0MDAwNTU4,size_16,color_FFFFFF,t_70)  
也就是说，美团为了以防万一提供了降级方案，正是因为这个方案给了我们机会，既然 tcp 通道发生故障会切换到 http 请求，那么只要让这个条件成立就好了。这个可以通过屏蔽 ip 实现，随即有了一个新问题，怎么找到这个 ip。

首先移动端长连接与服务器的通信肯定是 [TCP](https://baike.baidu.com/item/TCP/33012) 协议；然后由于是长连接，那么移动端肯定不会轻易的去断开这个连接，也就是不会发送 fin 包，有了这两个特征就可以开始筛选 ip 了。

### 3、ip 筛选

### >PC 端

pc 端的话用 [**wireshark**](https://www.wireshark.org/) 就完事了；

首先打开模拟器，然后打开 wireshark，选择网卡就开始抓包了，然后打开美团外卖 App，搜索一个关键字；

首先你搜索一个关键字肯定会得到很多结果，那么就可以断定返回的数据包应该比较大，而且根据上图还可以知道是加密过的，那么就很好找了，我这里找到了

![](https://img-blog.csdnimg.cn/20200506205420850.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxMzE0MDAwNTU4,size_16,color_FFFFFF,t_70)

接着就是屏蔽它了，Linux 直接用 iptables 就行，我没 linux 但我可以提供命令：

```
iptable -A INPUT -s ***.**.***.181 -j DROP #屏蔽
iptable -D INPUT -s ***.**.***.181 -j DROP #解除屏蔽
```

这命令可以在安卓上直接生效，linux 可能需要 service iptables save 来生效

![](https://img-blog.csdnimg.cn/20200506205313518.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxMzE0MDAwNTU4,size_16,color_FFFFFF,t_70)

Mac 下没有 iptables 可以用 **pfctl**，首先写规则

```
sudo vim /etc/pf.conf
```

![](https://img-blog.csdnimg.cn/20200506205516122.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxMzE0MDAwNTU4,size_16,color_FFFFFF,t_70)

然后使其生效

```
sudo pfctl -evf /etc/pf.conf
```

接着就可以直接用 **Charles** 抓包了

![](https://img-blog.csdnimg.cn/20200506205557551.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxMzE0MDAwNTU4,size_16,color_FFFFFF,t_70)

### >Android

接下来我要说的是 Android 上，我抓 mt 是为了写爬虫，我在电脑上抓到了请求的连接但那些参数加密我需要 hook，所以不得不在手机上弄，所以我就需要在手机上屏蔽 ip。我先说在手机上抓长连接 ip，目前手机上能抓 tcp 的除了 tcpdump 以外好像就只有 **httpcanary** 了，主要是它能显示请求的协议；

![](https://img-blog.csdnimg.cn/20200506205617702.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxMzE0MDAwNTU4,size_16,color_FFFFFF,t_70)  
找到 ip 后屏蔽即可，然后再次请求

![](https://img-blog.csdnimg.cn/2020050620563574.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxMzE0MDAwNTU4,size_16,color_FFFFFF,t_70)  
其中有一条 80 多 k 的请求，而且这个 host 之前没见过，点进去看 response 就得知成功了