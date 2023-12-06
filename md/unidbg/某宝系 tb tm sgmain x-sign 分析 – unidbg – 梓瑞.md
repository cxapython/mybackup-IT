> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.qinless.com](http://www.qinless.com/179)

> 仅供学习研究 。请勿用于非法用途，本人将不承担任何法律责任。 前言 apk 版本, 天猫 8.11.0, 本次主…

> `apk 版本, 天猫 8.11.0`, 本次主要说分析下如何使用 `unidbg` 跑通 `x-sign`，博主也是初学者，有啥问题可以加博主一起讨论哈，非常欢迎

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_5278d11d58a0833bd887c3c97faac89f.jpg)

主要分析这个 `x-sign`，知道目标后，直接开始使用 `unidbg`

具体的函数定位就不说了，各位看官应该都知道 `JNICLibrary.doCommandNative` 这个是 `so` 入口

这里先把框架搭起来，`IOResolver<AndroidFileIO>` 这个接口类可以直接使用 `unidbg` 的虚拟文件系统，方便补文件。运行，如果没啥错误就说明一切正常，继续下一步

下面开始对各个 `so` 进行初始化，具体的初始化流程，可以使用 `frida hook` 查看整体流程，使用 `jnitrace` 不全，比较容易出问题

libsgmainso
-----------

这里是初始化 `sgmain` 具体流程是 `frida hook` 的结果，写好后在 `main` 函数里调用一下，开始运行

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_b9bc2b26bbe1ed773a60c2935920d452.jpg)

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_875fa4e82d6c5ba0773751bc3a96cdec.jpg)

这里就开始报常见的环境错误了, 我们给他加上

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_344b26215cb139df89295a5d433b778e.jpg)

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_3c34e1c39b2f6dd010324dbb98d0c0ba.jpg)

这里继续，返回 `apk` 的 `base.apk` 路径

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_5af90a83aa71df7382369feda455e4e1.jpg)

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_811f602074c25ee6dc7b42fe52e1b90e.jpg)

这里连续报的两个错误，可以统一处理，返回 `files` 文件夹路径

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_3a22cc0782350f4d534cef58822aab19.jpg)

这里需要返回 `apk 的 native lib/arm` 路径

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_9e3e2365d5800c0af518088c73fb4056.jpg)

这个是 `sgmain` 抛异常的类，我们需要把 `msg code` 打印下，看看报了啥错误

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_4bcf2f62ff838116d11dccb20d191481.jpg)

继续执行，果然报错了 `code = 123`。这里查找错误原因有两种，第一种是 百度搜索 阿里聚安全 `123` 是啥错误，第二种是自己查看 `log` 日志，猜测试错。过程还是比较坑的，我这里踩了很多坑，就直接说答案吧是缺少了文件。具体缺少啥文件，搜索 `resolve.pathname:` 这个字眼，不知道补哪些就能补的都补下

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_a9bd3b53d56799052023576c4abedeb6.jpg)

我这里是补了左侧的几个文件，跟 `base.apk`，文件就在手机里对应的目录下，`pull` 下来就行，运行后发现不再报 `123` 错误了

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_0abe0c7a046920d64a4587ba1fbc6ddf.jpg)

这里报了两个错误，可以一起补

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_f9bbc663554b5524f78daa8db71c275b.jpg)

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_106d2689a9fee51534c77d4ddd413c83.jpg)

中间又补了几个错误之后，这里是读取文件里的 `json` 内容，因为是固定的我就直接取出来了，这个补完之后再补一个常见的就 `OK` 了

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_bb71c4e661855d1cec304c8a754daab1.jpg)

这里算是 `sgmain` 就补完了，全都正常返回了 `0`，这个结果我是 `frida call` 出来的结果

libsgsecuritybodyso
-------------------

新增个函数，`main` 函数里调用

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_ea6fb8e2086331acb2cdf1b40a541412.jpg)

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_bd14b1bd65bede96b0756c41ea8be30a.jpg)

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_919e252b521b817c6f2bf5d820c13711.jpg)

上图都是执行结果却啥补啥，就行

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_db707d8316d9fd0ea6beb2eb174a7ab8.jpg)

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_325de0e19d65a6a918af353f730ec546.jpg)

补到最后报了个指针错误，因为啥也不知道，还是使用相同的方法补文件，经过测试补 `dev/__properties__` 即可，继续运行，`OK` 了

libsgavmpso
-----------

这个比较简单，跟上诉逻辑差不多，就不说了，缺啥补啥

get x-sign
----------

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_5362ac7fed4f88969482870c9ee4ac26.jpg)

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_94cf50e0ddc290a74a3504b2b6c42001.jpg)

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_4db73971f8fc5863a63103d5359c91cd.jpg)

![](https://www.qinless.com/wp-content/uploads/2021/09/wp_editor_md_283153a5b0b0d7f63badcdabcd5632bc.jpg)

上图环境报错全部补完之后，就可以出结果了，经过测试，这个结果是可以使用的

> 想学习 `unidbg` 的伙伴，[可点击加入星球](https://t.zsxq.com/BeeyBIQ)，星主是一位全能的傻宝，每天都会分享各种姿势骚操作