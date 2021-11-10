> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/otHHxU-g7dUvbvgczVujWQ)

**抓包**  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCrnWZLibUuicpLXmOria38omkKYRBticjvic36dBiaiamHREPicxt4ria0G6IgJg/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCC2UPMLsswg5GYUmZIiccsZr8TMk3mbd4WvicOLgriat3GOVRxN90ibeZ22A/640?wx_fmt=png)![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCbZyUKwUK1CQEJKpkEiczZTjpxibU33K612oVmYrLMf4lGOdqeCRRcB7A/640?wx_fmt=png)

可以看到有个 sig 参数，每次不一样的，猜测是签名

返回数据是乱码，猜测是加密后的 bytes，经过 app 解密后才显示到页面上

所以文本目标：

一、sig 加密流程

二、响应数据解密流程

**sig 加密流程分析**

****1. jadx 分析 java 层函数****  

先分析签名 sig：  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCC2c037NHDUuMibeg5REIyqTf6ibuKqVibffzIvribbq5JW7B7ibXvTtmolRg/640?wx_fmt=png)

双击进入

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCXfGXD9McgM2tmhNpxrCOL3VDZN9jiaaia9Mr0tfhrPcZVr5ic3A5hq4eA/640?wx_fmt=png)

简单看一下上面的参数，跟抓包的对应上了，那应该就是这里赋值的，继续追 a 函数  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCoYYtgG7atODRdWlFwdADthE1LXyntzKicy5Bo5LOgDabNMSLZpDic8Bw/640?wx_fmt=png)

继续追 a 函数

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCYzOVBLBYiak6kHGbknLOsibjYcp9ncFD8OpNwuUOtTPBwJwalt4jflWg/640?wx_fmt=png)

继续追 signature 函数

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCIfasYajTKEm761bCUF5CbpZz5ByiauZ7MGicbRg2pubRBLgrPJRPbnHg/640?wx_fmt=png)

逐渐接近真相

![](https://mmbiz.qpic.cn/mmbiz_jpg/icBZkwGO7uH6nE726poJiaiahicok6OOiaaRrFyZO7WEJNdDCz5q78sUgUZ41Kk4h98UHOeQxTMSiargCA7VvWlm0aFw/640?wx_fmt=jpeg)

映入眼帘的是 5 个 native 函数。。  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCVNBJvq2jjeYevZQ540PEc6fEneETibpWaIQytw7RkbrfM2Ztjictre7Q/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_jpg/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCXxYXI02Hl5QGPNr4hzcuMiaafLRRjUILPZJC4Rby1jPfrvdLXavx8Lw/640?wx_fmt=jpeg)  

不就是 so 文件吗，别慌，看这 5 个函数名取的秒阿~~

第 1、2 个函数响应数据解密？

第 3 个请求 data 加密？

第 4 个密码加密？

第 5 个生成签名？

![](https://mmbiz.qpic.cn/mmbiz_jpg/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCa7TAxeZSR47Sz3XcpeibgRAG1hgAes7NpCHZk29piaIhKdHDcKJx5CCg/640?wx_fmt=jpeg)

祭出 frida 让它原形毕露；  

先解决 sig；

这里有个技巧，如果直接 hook nativeSignature 函数，那第一个参数类型是 bytes，返回的数据类型也是 bytes，需要各种转换

所以可以偷懒直接 hook 这个函数，就省下来几秒钟![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCcZC4Iu8AZAYsibQaAZzoaKoVaNNicAwTPiad4O1diaFpzJicOtvnbNiclGjA/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCXXZuIZabx6SxiaQlsAeSkeKL0JvvlxKibTGCFKBtkWvjXD5oUUbiaBKHQ/640?wx_fmt=png)

hook 结果：  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCl0IgwuBn2n8MnmqiclshscSQPOxdCeCGS6icKbwwJ32bYibZFJtM1uRKA/640?wx_fmt=png)

sig 跟抓包的对比一下，发现是正确的  

然后又看到 sig 是 32 位的 16 进制字符串

32 位？16 进制？md5？

![](https://mmbiz.qpic.cn/mmbiz_jpg/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCmquVefNxcaEmPZocqDvQd5VJEcic2iaStticPRqsaMskUq0wIjtJmMkTA/640?wx_fmt=jpeg)

赶紧把 hook 出来的 arg1 在线 md5 加密一波！万一呢  

结果不出我所料！ 并没有这么简单![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCDK8KCBWxIch7ibHicicHOD3DXUDoslISqgVynj0MyLXod55RGaheeGTdw/640?wx_fmt=png)  应该是加了盐，盐值应该就是 arg2

看来还是要靠这个女人呢

 ![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCOBdpp6sWVsRibrnu5tEXbsGnyjGrc6GI70kuUaJ1kjpmqHuPPkiawyiaA/640?wx_fmt=png)

**2. ida 分析 **native 层函数****

搜索关键词

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCkYVCRdRFBgYPvTibnVKia2bvPUVLicNmK7WZCD4Zn66Tw2NwHQI7dn5ibw/640?wx_fmt=png)

分析，找关键函数

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCyytVzRSIXaVUQqhzqHRTaEzgRUnacoDGzxoufe9l9qZNngQDjoPs8g/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCx2wSU5acMZsCesqsfYLzfg8ibv1VUaOKff96KPxvIcGPvy7Dh2TX3Cg/640?wx_fmt=png)

这些基操不在说了，不会的看前几篇公众号，定位到两个关键函数，hook 之

hook sub_A464：

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCU6mvvaltW1BudHcUyt1rkHjnJ4KEszto00RctibAOyXibhXlZMdm7sag/640?wx_fmt=png)

sub_A464 进去看看是啥玩意

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCC0woEDLNm7ppEuKmNm6KS1KUVssVB8TtJ5es5wp9HJwaiakrUlPoucpQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCiak3bOjQNCWpTKYoknGpic9voJN3OUZTqA3ROayze2sJ52XqHCb7h5Xg/640?wx_fmt=png)

明显的 某 5 编码特征，在线一波，发现这次对了

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCcviaRKdWAjBpib3mIDibicuicNhx7UeDfAzEe3RlBGaKk7TEFmiay12g2ZSg/640?wx_fmt=png)

sub_A438 就不用 hook 了，明显是拼接字符串  

所以总结：sig 就是把 arg1 + 两个 32 位的盐传进 sub_A464 进行 md5 编码生成 v17，然后 v17 传给 sub_A438 和 V2.0 拼接

**响应数据解密流程**  

**1. jadx 分析 java 层函数**  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCMwTMzrV2T6AjD6k9VgeqvmgyfGpQ5j4VibOyY6eSXOia9kM2gm1Mx46Q/640?wx_fmt=png)

看看这个函数哪里调用了

追到这，这是个拦截器，简单分析下  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCJ9q3IJuXvF88VywTtAdIAqbSEKR6lhtEgOndYGEoffxiaFVcKwdKArg/640?wx_fmt=png)

参数 1 是 a3.body().bytes()，a3 是这儿来的

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCC2JzmUGedGsexk3NUaYaOdervic0xhPaZZBeCmepPt5BSAmgniaEhgFMg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCFkqh4rmro0ja0mRoZ5H2lqNLtv0k2t3lbX4LfsiafKLUq3KBSn1UneA/640?wx_fmt=png)

3、4、5 参数是这玩意，指定加密方式

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCy4gJ8QCKuxlMyWhCXZnunhjEmicpJYiaTIVSQSRpa6OUWF8rkp4sDzTg/640?wx_fmt=png)

应该没毛病，所以 hook  nativeDecodeContent 函数看看参数是啥  

hook 结果

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCBicmp3hriaTI7VHTeEiafFyt7HmU9W3wmc0DmiccEShdkLlDxN22Sia7G9Q/640?wx_fmt=png)

把结果转成字符串，没毛病  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCSu6OomHobuciaa4qHyDU3t1Fc9ODdacl18GlIiaE4LbreJqclUb2LTeQ/640?wx_fmt=png)

**2. ida 分析 native 层函数**  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCaLVNXgXSeMoVibYph5qcfPH8ppuSz3aSkMyycS5SFuekDDLxbicH7E6w/640?wx_fmt=png)

手动美化一下  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCC0DActdrvKiaEyccYfTvG0M0rfQx50W9w9giaqSjib2rl2ic1UcZxNT4vhQ/640?wx_fmt=png)

然后就是不断的分析

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCC3yMZUSYSMUECZNzeyFlExMoFCvYU9oYjAHAKiaW3j5ibbrWTl7V4ICZQ/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCOl3OtUrs5llq1g62Ua3ib8AO0twQOXibIxTW97Wz5fqU6JSFTicSgBtZw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCsfPIdlFF0Ic3hUPRePpicCzNBT1uRVyxBOooA5kcSQLyTfibGt64QibYg/640?wx_fmt=png)

sub_A6F0：

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCClFDmTicpyB0XArU7rLYCwr956Vt8d6kMcia6Quib0zoWqUgiajR7scd4Dg/640?wx_fmt=png)

sub_B36C：

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCaRUtoOTWTEqGldfePUVoAg8y2FtV7lbNtRKWBSlnpPsoBUHvzLiaVMw/640?wx_fmt=png)

sub_B3B6：

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCEYgEg1INbLGhwEE7MdmFKncVDCE6PArkQJm2jB0y6O01PS3a9ibTgvA/640?wx_fmt=png)

加密特征很明显，明眼人一眼就能看出来是啥加密

![](https://mmbiz.qpic.cn/mmbiz_jpg/icBZkwGO7uH6nE726poJiaiahicok6OOiaaRryIsdGibhpWiaUO7jC7hM8T59tWgvtlZgia6UMiaic9L7aJBcxwHahzzBZXQ/640?wx_fmt=jpeg)

hook sub_A6F0：  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCz5tFazuicGCFojswAoA6RAv9khowKz4t7icXTicTcE9ZA5ZhcuvibuYgPg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCaJgEZqsMkicZoDS6VTFldVf3WreU1letB1mZI49KOuReT2dmAB9nJpQ/640?wx_fmt=png)

sub_A6F0 解密出来的 bytes 传进 A75C，  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCiaMia3tU0YQbvIkTNIgB4Xv9icqsvSewalr04YCrnVexOicnIgeX2hxbvw/640?wx_fmt=png)

hook：

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCf2YzghVdYynQ5RoPHNwuqa5LyOMsZIqvkI1jAr8XK8iaoNtCbLEn7kQ/640?wx_fmt=png)

可以看到经过上面函数解密，A75C 参数 1 已经有点像答案了  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCEmd0nebg8pic82Z2eNmRN7c0biceYicJmqbaia5ypWoHEKuOtVjjg8TAAA/640?wx_fmt=png)  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCeuZXRY1dgdMxicnuYiaGPMdlIuDg5rgCpbx3cyPSFj2O537lUibnVWRuQ/640?wx_fmt=png)

A75C 返回值就是完全解出来的答案，看看 A75C 函数做了什么  

![](https://mmbiz.qpic.cn/mmbiz_jpg/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCIMiak9Qcwia2btsic1SMftibUjxUOtZDHeeKXrBslrLxOoCXJPUW0yyGBA/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCC2Z6lGSiaWCAqDTRpf7RuzRrYyn1YjjjoUOMR7cDcqX581kQEXbX14zQ/640?wx_fmt=png)

就是 lz4 解压，hook 一波  

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCbicSic5HAxt71WFRGvXfPT0PwlicKKibuHMuTQvBagTX6pJwmZX8F4Cxdg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCccbtngchiciaJ7icOum5sFCxLheYcLHToSD2kibZtEQgntjdVgibhSsBBbQ/640?wx_fmt=png)

没毛病

总结：sub_A75C 函数解密, sub_A75C 函数解压

![](https://mmbiz.qpic.cn/mmbiz_png/icBZkwGO7uH5hiaxSp14Wz4rzTwerJuiaCCJdr2iaBwg1YD5cKUD3pXCExBlksB7Y9JSYFGNCtAkLIicXicicWh0bXxZQ/640?wx_fmt=png)

ps：这个会了模拟登录就会了  

**主要学习记录和分享思路，关键 key 已经隐去，如有侵犯，请联系我删除**

![](https://mmbiz.qpic.cn/mmbiz_jpg/icBZkwGO7uH6nE726poJiaiahicok6OOiaaRrIZoialicKqZwHeLxSVxjQk17Od3lzYwGjBQKrLNicRHiaDKiaDaqviakgvCQ/640?wx_fmt=jpeg)