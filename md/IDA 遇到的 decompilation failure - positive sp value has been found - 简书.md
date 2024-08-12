> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/605dee5e7160)

最近在用 IDA 看东西时，经常遇到如下报错

![](http://upload-images.jianshu.io/upload_images/2570919-1342d092da224cdf.png)

后来查了查资料，解决了这个问题，觉得好神奇，仿佛打开了新大陆，特地记录一下

参考的文章：https://blog.csdn.net/userpass_word/article/details/83588244

我的 IDA 是 7.0.170914 版本，Mac 版本

先是在 IDA 的 Options 选择 General，在弹出框中将 Stack pointer 选中

![](http://upload-images.jianshu.io/upload_images/2570919-790887593e57bf7f.png)

然后 IDA 的汇编代码中就有了一串 000 的绿色数字，其中方法的最后一部分是 - 90

![](http://upload-images.jianshu.io/upload_images/2570919-b6b11418eeefc02a.png)

在 - 90 的上一行 32C 的位置，右键，选择 Change stack pointer

![](http://upload-images.jianshu.io/upload_images/2570919-df648c6d90f81e87.png)

在新的弹框中输入 - 0x90 这个数字来源于上图的最后的 90

![](http://upload-images.jianshu.io/upload_images/2570919-d93eba5b5d98792b.png)

选择 OK 后，IDA 界面就变成了下图，用 fn+F5 就能看到伪代码了，神奇

![](http://upload-images.jianshu.io/upload_images/2570919-d1c57a37e495ec55.png)