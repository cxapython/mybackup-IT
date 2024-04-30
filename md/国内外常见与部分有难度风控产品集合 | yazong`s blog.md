> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [1997.pro](http://1997.pro/archives/1713518394359)

> 国内风控 阿里云盾 难点 控制流

阿里云盾
----

### 难点

控制流平坦化 / wasm / 滑块轨迹 / 并发环境

### 140

#### demo 网址

[https://promotion.aliyun.com/ntms/act/captchaIntroAndDemo.html](https://promotion.aliyun.com/ntms/act/captchaIntroAndDemo.html)

#### 特征

aliyun 域名，加密参数 140# 开头

### 227

#### 普通滑块 demo 网址

[https://ditu.amap.com/detail/get/detail](https://ditu.amap.com/detail/get/detail?id=B00155L3DH)

#### **水果滑块 demo 网址**

[https://zhaoshang.tmall.com/maintaininfo/liangzhao.htm?xid=d05d9e6324a811f4e2cb7918db4c88d3](https://zhaoshang.tmall.com/maintaininfo/liangzhao.htm?spm=a1z10.1-b.1997427721.5.ba9c5f3ba2vn3f&xid=d05d9e6324a811f4e2cb7918db4c88d3)

#### 特征

fireyejs.js 文件，加密参数 227! 开头

网易易盾
----

### 难点

参数杂 / 并发环境

### **demo 网址**

[https://dun.163.com/trial/sense](https://dun.163.com/trial/sense)

### 特征

163 域名，data、fp、cb 等参数

腾讯天御 (原防水墙)
-----------

### 难点

jsvmp / 动态 js / 并发 ip 要求 / aigc 图库

### **demo 网址**

[https://cloud.tencent.com/product/captcha](https://cloud.tencent.com/product/captcha)

### 特征

tdc.js 文件，collect 加密参数

顶象
--

### 难点

动态 js / 检验环境多 / 验证码类型多

### **demo 网址**

[https://www.dingxiang-inc.com/business/captcha](https://www.dingxiang-inc.com/business/captcha)

### 特征

验证码带有顶象字样或图标，ac 加密参数

数美
--

### 难点

无

### **demo 网址**

[https://www.ishumei.com/trial/captcha.html](https://www.ishumei.com/trial/captcha.html)

### 特征

fverify 请求、organization 加密字段

同盾
--

### 难点

动态 js

### 无感

#### **demo 网址**

[https://www.juneyaoair.com/home](https://www.juneyaoair.com/home)

#### 特征

fm.js 文件，blackbox 参数，a、b、c、d... 或者 h、i、j、k... 加密参数

### 滑块

#### **demo 网址**

[https://sec.xiaodun.com/onlineExperience/slidingPuzzle](https://sec.xiaodun.com/onlineExperience/slidingPuzzle)

#### 特征

tdCaptcha.js 文件，p1 p2 p3... 加密参数

vaptcha
-------

### 难点

手势识别

### **demo 网址**

[https://www.vaptcha.com/#demo](https://www.vaptcha.com/#demo)

### 特征

vaptcha-sdk.js 文件，丑丑的手势识别图片

极验
--

### 难点

无

### **demo 网址**

[https://www.geetest.com/Register](https://www.geetest.com/Register)

### 特征

geetest 标识及域名

友验
--

### 难点

jsvmp

### **demo 网址**

[https://www.fastyotest.com/demo](https://www.fastyotest.com/demo)

### 特征

yotest.js 文件

ciyverify
---------

### 难点

jsvmp 多堆栈协程

### **demo 网址**

[https://ciyverify.com/#/?id=%e6%bb%91%e5%9d%97%e6%b5%8b%e8%af%95](https://ciyverify.com/#/?id=%e6%bb%91%e5%9d%97%e6%b5%8b%e8%af%95)

### 特征

cyverification.js 文件

瑞数
--

### 难点

jsvmp

### demo 网站

#### vmp 版本

专利局

#### 其它版本

略

#### 特征

cookie，url 后缀，无限 debugger，202 或 412 返回

谷歌
--

### 难点

恶心的混淆 / 类 vmp / 多异步操作 / 模型精度要求较高

### v2

#### 无感

[https://www.google.com/recaptcha/api2/demo?invisible=true](https://www.google.com/recaptcha/api2/demo?invisible=true)

#### 点选

[https://www.google.com/recaptcha/api2/demo](https://www.google.com/recaptcha/api2/demo)

#### **企业版**

[https://2captcha.com/demo/recaptcha-v2-enterprise](https://2captcha.com/demo/recaptcha-v2-enterprise)

#### 特征

### v3

#### 无感

[https://recaptcha-demo.appspot.com/recaptcha-v3-request-scores.php](https://recaptcha-demo.appspot.com/recaptcha-v3-request-scores.php)

#### 企业版

[https://2captcha.com/demo/recaptcha-v3-enterprise](https://2captcha.com/demo/recaptcha-v3-enterprise)

### **特征**

recaptcha__zh_cn.js 为第一语言中文的谷歌文件

/recaptcha/enterprise.js 为企业版

hcaptcha
--------

### 难点

问题类型多，图库更新很频繁

### 普通版

[https://accounts.hcaptcha.com/demo](https://accounts.hcaptcha.com/demo)

### pro 版

[https://www.hcaptcha.com/pro](https://www.hcaptcha.com/pro)

### 企业版

[https://watchwarranty-int.gucci.com/](https://watchwarranty-int.gucci.com/)

### 特征

hcaptcha.js 文件，hcaptcha 手掌 logo

**Cloudflare(5 秒盾)**
--------------------

### 难点

动态 js，检测环境巨多，代码变更快，并发风控严格

### demo 网址

三种模式:

[https://](https://react-turnstile.vercel.app/basic)[peet.ws/turnstile-test/](http://peet.ws/turnstile-test/)

### 特征

cf 验证框，403 返回，cf_bm、cf_clearance

akamai
------

### 难点

动态混淆，并发风控严格，代码变更快

### demo 网站

[https://www.ihg.com.cn/rewardsclub/cn/zh/enrollment/join](https://www.ihg.com.cn/rewardsclub/cn/zh/enrollment/join)

### 特征

sensor_data 加密字段, cookie 携带_abck

亚马逊云验证
------

### 难点

小车路径终点识别，点选类别较多

### demo 网站

[https://sciencesetavenir.fr/](https://sciencesetavenir.fr/)

### 特征

awswaf 域名, challenge.js，cookie 携带 aws-waf-token

Incapsula
---------

### 难点

不同网站版本不同，检测参数较多

### demo 网站

#### 普通版

[https://www.luxair.lu/](https://www.luxair.lu/)

### rbzid 版

[https://premier.hkticketing.com/](https://premier.hkticketing.com/)

#### **utmvc 版**

[https://www.flyscoot.com/zh](https://www.flyscoot.com/zh)

### 特征

cookie 携带 reese84，incap_ses_xxx

funcaptcha
----------

### 难点

问题类型多，图库更新频繁，ip 质量要求偏高

### demo 网址

[https://www.amazon.com/aaut/verify/flex-offers/challenge?challengeType=ARKOSE_LEVEL_2&returnTo=https://www.amazon.com&headerFooter=false](https://www.amazon.com/aaut/verify/flex-offers/challenge?challengeType=ARKOSE_LEVEL_2&returnTo=https://www.amazon.com&headerFooter=false)

### 特征

arkoselabs 域名，重复操作多次点选

Datadome
--------

### 难点

封指纹严重，并发风控严格，小更新频繁

### demo 网址

[https://www.hermes.com/](https://www.hermes.com/)

### 特征

cookie 携带 datadome，f12 被封设备

PerimeterX
----------

### 难点

不同网站版本不同，检查参数多，wasm，ip 质量要求高

### demo 网址

[https://www.walmart.com/blocked?url=Lw==](https://www.walmart.com/blocked?url=Lw==)

### 特征

cookie 携带_px2、_px3、_pxde

Kasada
------

### 难点

jsvmp，动态 js，大量环境检测

### demo 网址

[https://www.sephora.com/](https://www.sephora.com/) 登录

### 特征

ips.js 文件，x-kpsdk 请求头，通常伴随 akamai 验证

F5 shape
--------

### 难点

jsvmp，动态 js，巨量环境检测

### demo 网址

[https://www.southwest.com/](https://www.southwest.com/)

### 特征

-A -B -C -D -F -Z 后缀请求头