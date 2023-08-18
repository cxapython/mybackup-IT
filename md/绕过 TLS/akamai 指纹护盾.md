> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7226930744769577017)

1. 前言
=====

在之前的项目中，发现一些网站使用不同的客户端会得到不同的结果，比如使用浏览器访问正常没问题，但使用 python 写脚本或者 curl 请求就会被拦截，当时也尝试数据包 1:1 还原，但还是不能解决。

测试指纹拦截站点：[ascii2d.net](https://link.juejin.cn?target=https%3A%2F%2Fascii2d.net%2F "https://ascii2d.net/")

最近拜读了师傅的文章《[绕过 Cloudflare 指纹护盾](https://link.juejin.cn?target=https%3A%2F%2Fsxyz.blog%2Fbypass-cloudflare-shield%2F "https://sxyz.blog/bypass-cloudflare-shield/")》，很有感触，感觉之前遇到的应该就是这个问题；之前写爬虫遇到类似这种指纹护盾（反爬机制），也都是尝试通过 selenium 模拟浏览器来绕过的，这一次也算是见了世面，学到了一些新的东西。

> 本次内容主要分为 2 部分，1 是绕过 TLS 指纹识别，2 是绕过 Akamai 指纹（HTTP/2 指纹）识别

2. TLS 指纹相关
===========

2.1. 什么是 TLS 指纹
---------------

TLS 指纹是一种用于识别和验证 TLS（传输层安全）通信的技术。

TLS 指纹可以通过检查 TLS 握手过程中使用的**密码套件、协议版本和加密算法等信息**来确定 TLS 通信的特征。由于每个 TLS 实现使用的密码套件、协议版本和加密算法不同，因此可以通过比较 TLS 指纹来判断通信是否来自预期的源或目标。

TLS 指纹可以用于检测网络欺骗、中间人攻击、间谍活动等安全威胁，也可以用于识别和管理设备和应用程序。

TLS 指纹识别原理（ja3 算法）：[github.com/salesforce/…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fsalesforce%2Fja3 "https://github.com/salesforce/ja3")

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/47d9a95352c349ceaf7427719fd0777e~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258A%25E5%258D%2588112740792.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8A%E5%8D%88112740792.png")

2.2. 测试 TLS 指纹
--------------

测试一下不同客户端之间的指纹差异（ja3_hash）

> 深入分析的话可以用 wireshark 抓 TLS 包进行对比分析

测试网站：[tls.browserleaks.com/json](https://link.juejin.cn?target=https%3A%2F%2Ftls.browserleaks.com%2Fjson "https://tls.browserleaks.com/json")

*   CURL v7.79.1

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/79216daa1c6e48128cb3a9c81faf8e7c~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258A%25E5%258D%2588112944762.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8A%E5%8D%88112944762.png")

*   CURL 7.68.0

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f62435e3500f4ae2afd94d83364e8166~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258A%25E5%258D%2588113055858.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8A%E5%8D%88113055858.png")

*   Chrome 112.0.5615.137（正式版本） (x86_64)

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ec1d1f4e2171474dba56ea7310a86a92~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258A%25E5%258D%2588113234072.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8A%E5%8D%88113234072.png")

*   Burp Chromium 103.0.5060.114（正式版本） (x86_64)

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2249ead9ed64435c98e4d18300b840d3~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258A%25E5%258D%2588113630016.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8A%E5%8D%88113630016.png")

*   Python 2.11.1

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2ff02d6b8a9349769a8c91eda2521563~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258A%25E5%258D%2588113510067.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8A%E5%8D%88113510067.png")

可见不同的客户端都存在区别，针对最后一个 python 的 ja3_text 做一个简单的说明

*   第一个值 `771`：表示 JA3 版本，即用于生成指纹的 JA3 脚本的版本。
*   第二个值 `4866-4867-4865-49196-49200-49195-49199-163-159-162-158-49327-49325-49188-49192-49162-49172-49315-49311-107-106-57-56-49326-49324-49187-49191-49161-49171-49314-49310-103-64-51-50-52393-52392-49245-49249-49244-49248-49267-49271-49266-49270-52394-49239-49235-49238-49234-196-195-190-189-136-135-69-68-157-156-49313-49309-49312-49308-61-60-53-47-49233-49232-192-186-132-65-255`：表示加密套件，即客户端可以支持的加密算法。
*   第三个值 `0-11-10-35-22-23-13-43-45-51-21`：表示支持的压缩算法。
*   第四个值 `29-23-30-25-24`：表示支持的 TLS 扩展，如 SNI。
*   第五个值 `0-1-2`：表示支持的 elliptic curves，即椭圆曲线算法。

2.3. 绕过 TLS 指纹
--------------

既然都知道原理了，那么绕过就是伪造成合法客户端就行，简单来说，就是伪装 ja3_text 值，让其不被拦截即可，以修改支持的加密算法为主。

### 2.3.1. 方法零：使用原生 urllib

```
import urllib.request
import ssl

url = 'https://tls.browserleaks.com/json'
req = urllib.request.Request(url)
resp = urllib.request.urlopen(req)
print(resp.read().decode())

# 伪造TLS指纹
context = ssl.create_default_context()
context.set_ciphers("ECDHE-RSA-AES128-GCM-SHA256+ECDHE+AESGCM")

url = 'https://tls.browserleaks.com/json'
req = urllib.request.Request(url)
resp = urllib.request.urlopen(req, context=context)
print(resp.read().decode())
```

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d2999f0a54e14cfb8261a55637031c35~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258B%25E5%258D%258870919048.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8B%E5%8D%8870919048.png")

### 2.3.2. 方法一：使用其他成熟库🌟

可以试试 [`curl_cffi`](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fyifeikong%2Fcurl_cffi "https://github.com/yifeikong/curl_cffi")这个库，主打的就是模拟各种指纹

Python binding for curl-impersonate via cffi. A http client that can impersonate browser tls/ja3/http2 fingerprints.

> 除了这个，也可以去尝试下 pyhttpx、pycurl

```
pip install --upgrade curl_cffi
```

测试代码：

```
from curl_cffi import requests

print("edge99:", requests.get("https://tls.browserleaks.com/json", impersonate="edge99").json().get("ja3_hash"))
print("chrome110:", requests.get("https://tls.browserleaks.com/json", impersonate="chrome110").json().get("ja3_hash"))
print("safari15_3:", requests.get("https://tls.browserleaks.com/json", impersonate="safari15_3").json().get("ja3_hash"))

# 支持代理
proxies = {"https": "http://localhost:7890"}
r = requests.get("https://tls.browserleaks.com/json", impersonate="chrome101", proxies=proxies)
print(r.json().get("ja3_hash"))
```

效果如下：

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1aad298799dd4efa8788978454c584dc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258B%25E5%258D%258844440082.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8B%E5%8D%8844440082.png")

支持伪造的浏览器列表如下：

```
# curl_cffi.requests.session.BrowserType
class BrowserType(str, Enum):
    edge99 = "edge99"
    edge101 = "edge101"
    chrome99 = "chrome99"
    chrome100 = "chrome100"
    chrome101 = "chrome101"
    chrome104 = "chrome104"
    chrome107 = "chrome107"
    chrome110 = "chrome110"
    chrome99_android = "chrome99_android"
    safari15_3 = "safari15_3"
    safari15_5 = "safari15_5"
```

### 2.3.3. 方法二：挂一层客户端代理

这里是用 burp 去完成 TLS 认证过程，前提是 burp 的 TLS 指纹不会被拦截。

[![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258B%25E5%258D%258830421464.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8B%E5%8D%8830421464.png")

Burp 的 TLS 指纹可通过如下方式进行修改

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/25dff5708a534595af42d32b10f0c48f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258B%25E5%258D%258873736723.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8B%E5%8D%8873736723.png")

### 2.3.4. 方法三：修改 requests 底层代码

requests 库的 SSL/TLS 认证是基于 urllib3 库实现的，所以改底层就是改 urllib3 的代码

查看`urllib3`安装位置

```
python3 -c "import urllib3; print(urllib3.__file__)"

/Library/Frameworks/Python.framework/Versions/3.7/lib/python3.7/site-packages/urllib3/__init__.py
```

修改相关 SSL 代码，文件地址一般为`site-packages/urllib3/util/ssl_.py`

```
DEFAULT_CIPHERS = ":".join(
    [
        "ECDHE+AESGCM",
        "ECDHE+CHACHA20",
        "DHE+AESGCM",
        "DHE+CHACHA20",
        "ECDH+AESGCM",
        "DH+AESGCM",
        "ECDH+AES",
        "DH+AES",
        "RSA+AESGCM",
        "RSA+AES",
        "!aNULL",
        "!eNULL",
        "!MD5",
        "!DSS",
    ]
)
```

操作的空间很多，像我这种脚本小子一般就以删除和调换位置为主，对比如下：

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/685aec0cbfee444f85865169dcd6a0cd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258B%25E5%258D%258840250059.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8B%E5%8D%8840250059.png")

3. Akamai 指纹相关（HTTP/2 指纹）
=========================

3.1. 什么是 Akamai 指纹
------------------

Akamai Fingerprint 是 Akamai Technologies 公司提供的一种防止恶意机器人和自动化攻击的技术，它基于浏览器指纹识别技术。

浏览器指纹是一种用于识别 Web 浏览器的技术，它通过收集并分析浏览器的各种属性和行为，如用户代理字符串、插件、字体、语言、屏幕分辨率等信息来识别浏览器。浏览器指纹在互联网安全领域得到了广泛应用，可以用于检测和识别恶意机器人、欺诈行为、网络钓鱼等。

Akamai Fingerprint 利用了浏览器指纹技术，将其与其他安全技术结合起来，以识别和拦截自动化攻击。它可以在不影响用户体验的情况下，对访问网站的浏览器进行识别和验证，防止自动化攻击、账户滥用和数据泄露等安全问题。

可以在 [tls.peet.ws/api/all](https://link.juejin.cn?target=https%3A%2F%2Ftls.peet.ws%2Fapi%2Fall "https://tls.peet.ws/api/all") 看到详细的指纹，主要有如下内容

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/19d51c4a10534a26a556146e0c415e28~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258B%25E5%258D%258885345384.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8B%E5%8D%8885345384.png")

指纹为：`1:65536,2:0,3:1000,4:6291456,6:262144|15663105|0|m,a,s,p`

1.  `1:65536`: `HEADER_TABLE_SIZE`，即头部表大小为 64KB，指的是用于存储请求头和响应头的大小，它是可以调整的。这个字段指明了使用 64KB 的头部表大小。
    
2.  `2:0`: `HTTP2_VERSION`，指示此请求使用的 HTTP/2 版本。0 表示 H2，表示启用了 HTTP/2 协议。
    
3.  `3:1000`: `MAX_CONCURRENT_STREAMS`，即最大并发流数，指的是在任何给定时间内，客户端和服务器端可以并行发送的最大请求数量。这个字段指明了最大并发流数为 1000。
    
4.  `4:6291456`: `INITIAL_WINDOW_SIZE`，即初始流窗口大小，指的是初始的流控窗口大小，即客户端可以发送的最大字节数量。这个字段指明了初始流窗口大小为 6MB（即 6291456 字节）。
    
5.  `6:262144|15663105|0|m,a,s,p`: 以竖杠 “|” 分隔。具体含义如下：
    
    *   `6:262144`: `max header list size`，即动态表大小，指的是接收方可以接收的最大 HTTP 头部大小。这个字段指明了动态表大小为 256KB（即 262144 字节）。
    *   `15663105`: `WINDOW_UPDATE`，表示收到了`WINDOW_UPDATE`帧，并且窗口大小增加了 15663105 个字节。
    *   `0`: `no compression`，表示不启用头部压缩。
    *   以 `:` 开头的 header 的第一个字符参与编码，多个逗号隔开。如 `:method`、`:authority`、`:scheme`、`:path` 编码为 `m,a,s,p`

可在 [Passive Fingerprinting of HTTP/2 Clients](https://link.juejin.cn?target=https%3A%2F%2Fwww.blackhat.com%2Fdocs%2Feu-17%2Fmaterials%2Feu-17-Shuster-Passive-Fingerprinting-Of-HTTP2-Clients-wp.pdf "https://www.blackhat.com/docs/eu-17/materials/eu-17-Shuster-Passive-Fingerprinting-Of-HTTP2-Clients-wp.pdf") 中查看详细细节

3.2. 测试 Akamai 指纹
-----------------

测试网站：[tls.browserleaks.com/json](https://link.juejin.cn?target=https%3A%2F%2Ftls.browserleaks.com%2Fjson "https://tls.browserleaks.com/json")

*   CURL

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c4b0cc40b3814272af929f72057d2828~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258B%25E5%258D%258850603033.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8B%E5%8D%8850603033.png")

*   Chrome

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e137ddcb28724d98833de80f8fedf389~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258B%25E5%258D%258850716193.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8B%E5%8D%8850716193.png")

*   Python

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bf0c89c3accd4ec0b483e38d4a4436c0~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258B%25E5%258D%258850803277.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8B%E5%8D%8850803277.png")

可以看到用 python requests 直接为空，爬虫小子直接被拦截在外了。

3.3. 绕过 Akamai 指纹
-----------------

伪造指纹中特定的字段即可。

### 3.3.1. 方法一：使用其他成熟库🌟

还是刚才的 [`curl_cffi`](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fyifeikong%2Fcurl_cffi "https://github.com/yifeikong/curl_cffi")这个库，因为这个库主打的就是模拟各种指纹

Python binding for curl-impersonate via cffi. A http client that can impersonate browser tls/ja3/http2 fingerprints.

```
pip install --upgrade curl_cffi
```

测试代码：

```
from curl_cffi import requests

print("edge99:", requests.get("https://tls.browserleaks.com/json", impersonate="edge99").json().get("akamai_hash"))
print("chrome110:", requests.get("https://tls.browserleaks.com/json", impersonate="chrome110").json().get("akamai_hash"))
print("safari15_3:", requests.get("https://tls.browserleaks.com/json", impersonate="safari15_3").json().get("akamai_hash"))
```

效果如下：

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cd5fa990e4af4f488dbaa6606be990cc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258B%25E5%258D%258851109672.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8B%E5%8D%8851109672.png")

支持伪造的浏览器列表如下：

```
# curl_cffi.requests.session.BrowserType
class BrowserType(str, Enum):
    edge99 = "edge99"
    edge101 = "edge101"
    chrome99 = "chrome99"
    chrome100 = "chrome100"
    chrome101 = "chrome101"
    chrome104 = "chrome104"
    chrome107 = "chrome107"
    chrome110 = "chrome110"
    chrome99_android = "chrome99_android"
    safari15_3 = "safari15_3"
    safari15_5 = "safari15_5"
```

4. 最终效果
=======

[ascii2d.net](https://link.juejin.cn?target=https%3A%2F%2Fascii2d.net%2F "https://ascii2d.net/") 存在 CloudFlare 的指纹护盾，拒绝爬虫，测试一下。

直接 CURL，被拦截

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/61c31740fc5745aea0fbfcb38303834b~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258B%25E5%258D%258853411888.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8B%E5%8D%8853411888.png")

绕过

```
from curl_cffi import requests

req = requests.get("https://ascii2d.net", impersonate="chrome110")
print(req.text)
```

可正常获取页面

[![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f0bd5433b9e4038a75e38480b5e14a4~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)](https://link.juejin.cn?target=https%3A%2F%2Fblog.gm7.org%2F%25E4%25B8%25AA%25E4%25BA%25BA%25E7%259F%25A5%25E8%25AF%2586%25E5%25BA%2593%2F01.%25E6%25B8%2597%25E9%2580%258F%25E6%25B5%258B%25E8%25AF%2595%2F07.WAF%25E7%25BB%2595%25E8%25BF%2587%2F02.%25E7%25BB%2595%25E8%25BF%2587TLS%3Aakamai%25E6%258C%2587%25E7%25BA%25B9%25E6%258A%25A4%25E7%259B%25BE.assets%2Fimage-20230427%25E4%25B8%258B%25E5%258D%258853458127.png "https://blog.gm7.org/%E4%B8%AA%E4%BA%BA%E7%9F%A5%E8%AF%86%E5%BA%93/01.%E6%B8%97%E9%80%8F%E6%B5%8B%E8%AF%95/07.WAF%E7%BB%95%E8%BF%87/02.%E7%BB%95%E8%BF%87TLS:akamai%E6%8C%87%E7%BA%B9%E6%8A%A4%E7%9B%BE.assets/image-20230427%E4%B8%8B%E5%8D%8853458127.png")

5. 参考
=====

*   [绕过 Cloudflare 指纹护盾](https://link.juejin.cn?target=https%3A%2F%2Fsxyz.blog%2Fbypass-cloudflare-shield%2F "https://sxyz.blog/bypass-cloudflare-shield/")
*   [SSL 指纹识别和绕过](https://link.juejin.cn?target=https%3A%2F%2Fares-x.com%2F2021%2F04%2F18%2FSSL-%25E6%258C%2587%25E7%25BA%25B9%25E8%25AF%2586%25E5%2588%25AB%25E5%2592%258C%25E7%25BB%2595%25E8%25BF%2587%2F "https://ares-x.com/2021/04/18/SSL-%E6%8C%87%E7%BA%B9%E8%AF%86%E5%88%AB%E5%92%8C%E7%BB%95%E8%BF%87/")
*   [HTTP2 指纹识别 (一种相对不为人知的网络指纹识别方法)](https://link.juejin.cn?target=https%3A%2F%2Fwww.cnblogs.com%2Fyudongdong%2Fp%2F16654636.html "https://www.cnblogs.com/yudongdong/p/16654636.html")