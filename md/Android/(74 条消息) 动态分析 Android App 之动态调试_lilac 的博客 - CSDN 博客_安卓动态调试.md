> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38851536/article/details/100026480?spm=1001.2014.3001.5502)

_这个系列一共有五篇左右，内容主要介绍如何在 Java 层**动态分析和调试 Android App**，和网上其他教程相比，**内容更充实，体系更健全，深入而浅出**。  
闻道有先后，术业有专攻，希望能给刚入门 Android 逆向的同侪们些微帮助。出于各种原因，文章有两个遗憾，一是只包含了 Java 层代码的动态分析和调试，Jni 和 Native 层并没有涉及；二是对 Hook 框架的介绍和使用不是很充分，因为 Hook 值得另外很多个五篇去写。逆向太深太广了，吾辈将上下而求索。_

本篇内容所涉及到的资源  
链接：https://pan.baidu.com/s/14ZF-7pop4NbrDPydtRQOeg  
提取码：8fs8

一、认识动态分析 Android 程序
-------------------

我们将需要运行应用程序才能实施和完成的分析方法统称为动态分析方法，不要被这个名词吓到了，**抓包、动态调试、观察 App 页面的 UI 设计和交互、使用 Xposed/Frida Hook App 中某个函数、Smali 插桩**等等，这些都可以称为动态分析，有时候我们也会认为 Smali 插桩是一种静态的技术，但这里不用过分区分和计较，我们最应该关注的是技术和思路。  
这个系列会逐一讲解和介绍这些动态分析工具的使用，为了防止我讲的不够清楚，每一个知识点和技术都会配上数篇详细可靠的同类型文章。当然，你也可以直接 Google 获取这些知识，但需要稍加甄别搜索到的内容。我们第一篇说一下 Smali 动态调试。

### 1.1 什么是 Smali 动态调试？

调试分为源码级调试和反汇编级别调试，源码级调试是什么自然不用说，程序员大多都使用过诸如 Pycharm 这样的 IDE 对自己的程序进行过源码级调试，从而了解程序运行情况，分析程序的执行流程，观察变量的动态值等。而我们进行逆向分析时，手里不可能有 App 的源码，这就需要进行反汇编级的调试，也就是我们常说的的 Smali 动态调试。

当我学习 Smali 时，产生过各种各样的困惑，smali 是什么？我写的是 java 代码，怎么变成 smali 了？为什么可以 smali 代码可以调试？Jadx 反编译出来的 java 代码如此优雅，能不能根据这些代码进行源码级调试？  
我们一一来探讨这些问题。

从 Java 源码到编译打包成 APK 文件，会经过非常复杂和繁多的步骤，我们这里只关注代码的编译过程。

Android 平台上主要使用 Java 语言来开发程序，但 Android 上的程序运行机制和标准的 Java 程序并不一样。  
我们先看一下 Java 程序从编写到执行的过程

```
第一步：编写Java代码
第二步：所有的Java代码通过Java编译器被编译成java字节码，即.class文件
第三步：Java字节码在Java虚拟机上被解释成机器语言后，程序执行
```

![](https://img-blog.csdnimg.cn/20190823195911714.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
手机系统内存和处理器速度有限，为了解决运行效率的问题，以及摆脱和 Java 母公司的版权纠纷，Google 开发了 Dalvik 虚拟机，Android 程序就运行在其上，我们来看一下 App 中 Java 代码的编译过程。

```
第一步：编写Java代码
第二步：所有的Java代码通过Java编译器(javac)被编译成Java字节码，即.class文件
第三步：Java字节码通过Android 的dx工具转换为Dalvik字节码，即.dex文件
第四步：Dalvik字节码在Dalvik虚拟机上运行
```

![](https://img-blog.csdnimg.cn/20190823195922975.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
现在我们已经知道解压 Apk 后，那些奇怪的 dex 是哪来的了，那 Smali 又是哪来的呢？  
我们看一下 Smali 代码长啥样  
![](https://img-blog.csdnimg.cn/20190823221057566.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
首先澄清一下这两个概念

反汇编：将可执行的文件中的二进制经过分析转变为汇编程序。  
反编译：将可执行的程序经过分析转变为高级语言的源代码格式，一般完全的转换不太可能，因为有编译器的优化等因素在里面。

各种各样反编译工具几乎都用到了 Baksmali 这个工具，我们来看一下它和孪生兄弟的介绍：

> Smali，Baksmali 分别是指安卓系统里的 Java 虚拟机（Dalvik）所使用的一种. dex 格式文件的汇编器，反汇编器。  
> 其语法是一种宽松式的 Jasmin/dedexer 语法，而且它实现了. dex 格式所有功能（注解，调试信息，线路信息等）。  
> Smali，Baksmali 分别是冰岛语中编译器，反编译器的叫法。

也就是说，Smali 代码是利用 Baksmali 反汇编 (disassemble)dex 文件得到的一种类汇编代码，它完整的实现了. dex 格式所有功能 (注解，调试信息，线路信息等)，谈起 Smali 和 dex 之间的关系，我们常常称为转化 (convert), 即还原度极高，这也是 Smali 可以胜任动态调试的重要原因。而利用 Smali 汇编器，我们可以修改 Smali 代码，重新编译成 dex 文件，进一步可以对 Apk 进行重打包。

![](https://img-blog.csdnimg.cn/20190823200601541.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
在黑产中，这也是无数盗版应用、破解版应用、功能增强应用、去广告版应用的实现原理。  
![](https://img-blog.csdnimg.cn/20190823205312635.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

而我们这些只是想分析 App 通信协议的程序员，也可以简单的修改 Smali 进行 “Smali 插桩”，通过 log 探针输出一些可疑的信息，或者根据探针是否触发，探测程序运行逻辑。

提到 Smali 插桩，玩法也不少，Android 逆向人员在仰望星空时，或多或少都渴望过一种破解 App 的暴力美学——让每一行代码都自动吐出来一句话 “爷，我在这儿，我是干嘛的，我前面又是啥。”  
换成可以通过代码实施的方案，也就是在每个方法内打印调用栈或者 log 输出一下它在哪个方法里，该怎么做呢？方法非常多，我们看一下暴力插桩的三个思路。

1. 直接对 smali 代码进行文法分析，写一些正则表达式的判断，实现在每行代码后面加上 log 输出的 Smali 代码，然后使用 Smali 汇编器编译成 dex 文件，进而重打包 Apk，最后运行 App 查看 log 输出。具体实现可以参考这篇文章，实现出来的效果也很不错实现出来的效果也很好。  
《Android 应用逆向——分析反编译代码之大神器》 https://blog.csdn.net/charlessimonyi/article/details/52027563  
![](https://img-blog.csdnimg.cn/20190823211707302.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
2. 直接对 Dex 进行操作，可以使用的工具有 ReDex，Dexter 等。但由于 Dalvik 发展尚浅，且由于 Dalvik 字节码比 Java 字节码的结构更加紧凑，所以修改起来比较复杂，笔者暂时没有看到逆向中应用 Dex 插桩的具体实现。

3. 既然 Dex 紧凑而且不好搞，那能不能搞 Java 字节码进行插桩呢？答案是完全可以的，对 Java 字节码进行操作的工具非常多且成熟，可以通过 AspectJ、Asm、javassit 等工具对 Java 字节码进行操作。下面这个工具，就是将 Dex 文件转换为 Java 字节码，再使用 asm 操作字节码，最后再用 dx 工具 (Android Java 代码编译流程第三步中提到的 Android 自带工具) 编译成 dex，进而重打包 Apk，最后运行 App 查看 log 输出。  
《带你开发一款给 Apk 中自动注入代码工具 icodetools》  
https://blog.csdn.net/jiangwei0910410003/article/details/53386071

Smali 和 Smali 插桩就介绍到这儿了，感兴趣的同学可以去试一下。  
讲道理，我们应该先花一万字讲一下 smali 语法和 smali 如何实现基础的插桩，但动态调试 Smali 实在是方便又迷人，我发誓，在掌握 Smali 动态调试后，你们很快就会将又麻烦又容易出错的 Smali 插桩忘在脑后。

但这绝不意味着我们就不需要能看懂和理解 Smali 语法了，原因主要有两个

1.  Smali 动态调试基于 Smali 代码，断点该下在哪里，程序跑到哪儿了，如何查看调试中的变量值…… 这些都需要你稍微懂一些 Smali 语法。
2.  Jadx 之类的工具虽然号称做到了 Dex 到 Java 的一站式反编译 (Dex to Java decompiler)，但在其内部，是先将 Dex 反汇编成 smali，再用 asm 将 smali 转成 class 字节码，最后解析查看 Java 代码。如果说 Dex 到 Smali 是一种 convert(转化), 得到的是可调式、精准靠谱的源码，那么 Dex 到 Java 就是一种 translation(翻译)，虽然看着可能还不错，但其实和源码相差甚远，只能称为 Java 伪代码，正因为相差甚远，所以我们才无法使用 Jadx 反编译出来的 java 代码进行源码级的调试，而只能进行稍显晦涩的 Smali 调试。所以你可能会在使用 Jadx 反编译过程中遇到部分逻辑无法翻译成 java 代码的情况，或者翻译的 Java 代码和真实的 Smali 逻辑有差异，这个时候就需要老老实实看 Smali 代码。

在下一篇中，我们将结合小红书应用来讲解 Smali 的语法，这一篇的主角还是实现如何进行动态调试。

### 1.2 为什么要进行 Smali 动态调试？

动态调试能更充分的展示程序的运行逻辑，简而言之倍儿爽。

### 1.3 不能进行 Java 调试吗？

在上面我们已经讲的很清楚了，反编译得到的 Java 代码只是一种翻译而来的伪代码，它无法支撑其源代码级的动态调试。

接下来我们开始动态调试 Smali 之旅。

二、推开调试之门
--------

出于安全考虑，Android 系统并不允许应用被随意调试，官方文档称需要满足二者之一的条件。

1.App 的 AndroidManifest.xml 中 Application 标签必选包含属性 android:debuggable=“true”；  
2./default.prop 中 ro.debuggable 的值为 1；

我们先来看第一个条件有没有办法满足，首先，发行版的 App 都会将 debuggable 设置为 false，使第三方不能直接调试分析 APP，这也是厂商出于安全的考虑，那我们就需要反编译 Apk，修改后进行重打包，这也是绝大多数教程的做法，但我个人非常非常不建议这么操作，因为**重打包容易遭受无妄之灾**，这也是我不喜欢 Smali 插桩的原因——它们需要**重打包 App**。  
你想研究 App 的通讯协议和加密字段，这已经足够让人焦头烂额，你可能会遇到繁杂的代码、诡异的反抓包，So 层的加密…… 而如果你对 App 进行重打包，那就要面对 App 额外的保护措施，比如重打包失败，签名验证等。因为重打包这个操作主要是开发盗版 App 和破解版 App 做的事，这对厂商来说更加难以忍受，只是修改一个 debuggable 字段就要揽上这么多事，显然吃力不讨好。

那第二个条件好满足吗？default.prop 文件非常好找，它就在 Android 的根目录下，我们可以通过 ES 文件浏览器找到它。  
![](https://img-blog.csdnimg.cn/20190824105841635.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20190824105901446.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
很不幸的发现，这台手机的 debuggable 标识为 0，不可调试。一个朴素的想法是直接修改这个值不就可以了？但是这是不可以的，这个值只在系统启动时，也就是开机时才会读取和加载一次。那重启？抱歉，每次重启，这个值就会恢复默认。所以就造成了一个死循环。  
那我们是怎么解决它的呢？有这样几种办法。  
1. 改写系统文件，修改 ro.debuggable 为 1，重新编译系统镜像文件，刷入设备。  
难度稍大，但一劳永逸，缺点是对新手很不友好。可以参考这篇文章，https://bbs.pediy.com/thread-197334.htm。  
2. 注入 init 进程，修改内存中 ro.debuggable 的值，这个也是之前惯常的做法。  
通过大佬写的 mprop 工具可以修改内存中所有的属性值，只需要按照操作步骤，cmd 敲七八行即可，还有人出了一键式的 bat 脚本。资源放在了我分享的百度资源里，大家也可以去制作者那儿下载。https://bbs.pediy.com/thread-223294.htm。 需要注意的是，因为是修改内存值，所以文件中的 ro.debuggable 值并不会变化，且每次重启设备都要重新注入。  
3. 使用开发版 / 测试版的手机系统，ro.debuggable 值常常为 1  
4. 使用模拟器，比如雷电模拟器、Genymotion 等，许多模拟器天然支持动态调试，尽管 defalut.prop 中值并不为 1，打开 adb shell，用 getprop ro.debuggable 命令查看内存中的 debuggable 值却为 1。  
![](https://img-blog.csdnimg.cn/20190824112942723.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20190824113012567.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
5.Xposed Hook 系统判定函数，Android 系统凭什么判断某个 App 是一个可调试的应用？从读取 AndroidManifest.xml 中 android:debuggable 属性值，到打开这个应用，里面有非常多的门路，找一个合适的时机进行 hook，就可以实现狸猫换太子，这需要逆向分析人员了解 Android 源码，我们这里不去说它，因为成熟的工具已经有很多了，只需要下载 Apk，在 Xposed 中激活后重启手机，就可以一劳永逸。

*   BDOpener 开启 APK 调试与备份选项的 Xposed 模块 https://security.tencent.com/index.php/opensource/detail/17
*   Xinstall 如何不重打包调试 Android 应用 https://www.open-open.com/lib/view/open1426304176732.html
*   BuildProp Enhancer Make all application attribute android:debuggable=“true” https://repo.xposed.info/module/com.jecelyin.buildprop
*   XDebug make all app on your phone debuggable https://github.com/deskid/XDebug

我个人平时用 BDopener，网盘资源中存放了数种开启调试的工具，请自行选择，Xinstall 是个十分优秀的 Xposed 框架，我们日后还会用到它。

三、选择调试目标
--------

我猜测你很可能选择了雷电模拟器，或者在真机上装了 BDOpener，我并不觉得意外，因为这两种方法确实最为便捷，之所以我们要讲那么多种方法，是为了避免意外，有的机型或者有的 App 存在闪退行为，这样你就可以求助于另外一个方法。

我们演示在雷电模拟器上调试新浪博客的一个 sign 参数，不难，但又不是纸玩具，非常适合我们进行测试。  
下载新浪博客最新版，我在百度云里也放了 apk。开启 Fiddler/Charles 抓包工具后，打开 App。  
![](https://img-blog.csdnimg.cn/20190824122620905.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
点开一条博客  
![](https://img-blog.csdnimg.cn/20190824122647263.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
查看 Fiddler，多出四五条数据，根据数据包大小和内容找到我们需要的那一条。  
![](https://img-blog.csdnimg.cn/20190824122815136.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
这是一个 GET 请求，字段有九个  
![](https://img-blog.csdnimg.cn/20190824122845904.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
deviceid、chno，appver，appid 是固定不变的，Is_default 可以不填，login_uid 因为没有登录也不用管，article_id 是每篇文章的 id，显然这个 sign 是比较好玩的。  
它是 64 位十六进制数，猜测是两个 md5 拼接，不太确定哦。

我们接下来使用 Jadx 反编译 Apk，搜索 url 链接的末尾部分，即 get_article_info.php，放一张之前教程的截图  
![](https://img-blog.csdnimg.cn/20190824132358733.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20190824132424541.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
只有 m 字符串是符合要求，双击代码进去看一下，在这个 config(配置) 包里，以类变量的方式存放着大量的字符串，如果想引用它，就是 b.m  
![](https://img-blog.csdnimg.cn/20190824132454254.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20190824132751184.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
右键查看用例，看一下这个 url 在哪儿被使用了，发现只有一处  
![](https://img-blog.csdnimg.cn/20190824133523805.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
双击第二行的代码，查看详细引用  
![](https://img-blog.csdnimg.cn/20190824132831816.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
它包装了一个 a 方法来取我们的目标 URL，再次查找用例  
![](https://img-blog.csdnimg.cn/20190824133740410.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
点开第一个，你会发现其实它就是 a 方法上面的那个方法  
![](https://img-blog.csdnimg.cn/201908241338331.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
看到这些代码你应该感到喜闻乐见了，我们发送网络请求，首先就要进行字段的获取和拼接，在 Java 中往往由集合 map 完成，格式类似于 {”id“:3,“name”:“lilac”}，put 存入，get 取出，这儿就是一个典型的 Hashmap。

它第一步初始化一个 map，之后巴拉巴拉放进去很多东西，看着和我们 get 请求中的字段一致。接下来我们用 Smali 动态调试**跟踪一下集合 m 从初始化到塞满东西的全部过程**。  
首先我要说明，这儿不用动态调试也是完全可以的，但 App 并不总是很简单，可以一目了然。

首先我们要获取反编译的 Smali 代码，因为我们的调试就是基于 Smali 的。你可以使用 Apktool 敲几行命令完成，但我个人更喜欢可视化的界面，市面上有很多集成了这些工具，可视化拖拽操作的工具。  
我这边演示 windows 下的操作，工具放在了网盘里，也可以自行搜索下载。  
![](https://img-blog.csdnimg.cn/20190824135630246.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
操作选择反编译 apk——拖拽 apk 到源文件——点击操作，依据电脑性能和 Apk 的大小，反编译所需时间要几十秒到十分钟不等，反编译完成后自动弹出文件目录。  
![](https://img-blog.csdnimg.cn/20190824140042457.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
mac 可以下载这个工具 https://github.com/Jermic/Android-Crack-Tool ，界面和操作几乎和 windows 中一样。  
![](https://img-blog.csdnimg.cn/20190824135534877.gif)  
都讲这么久了，我们都还没说调试工具，是这样的，几乎所有的主流 Java IDE 配上 smalidea 插件都可以对 Smali 进行动态调试，除此之外，JEB 也可以直接调试 Smali，IDA 也有调试 DEX 的能力，还有 Qtrace 等等工具，但调试 Smali，我只推荐 **Android Studio+smalidea 插件**这个组合，操作简单，功能强大，效果也很稳定。

接下来打开 Android Studio（注：Android studio 版本需要大于 3.0，我个人是 3.5Beta 版）  
先下载 smalidea 插件，可以直接用我的百度云链接，也可以去官网下载 https://bitbucket.org/JesusFreke/smali/downloads/  
![](https://img-blog.csdnimg.cn/20190824141033174.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
Android Studio–>Settings–>Plugins–>Install plugin from desk…，安装插件；需要注意 smalidea 路径最好不要有中文路径，可能会出问题。安装好后重启生效。  
![](https://img-blog.csdnimg.cn/20190824142433648.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
打开项目【Open an existing Android Studio Project】，选择 sinablog 文件夹，等待其加载，过程会持续数分钟。  
![](https://img-blog.csdnimg.cn/20190824142801952.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
加载完成之后，你需要配置一下 JDK 和 SDK，SDK 并不一定要和我一样，29，22…… 或者别的其他版本都可以。  
![](https://img-blog.csdnimg.cn/20190824144519630.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20190824155044657.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

我们要在想要跟踪的程序起始处下断电，重新上一下图，它是类 com.sina.sinablog.network.d 中的一个 a 方法，我们要在 smali 文件夹中找到它。  
![](https://img-blog.csdnimg.cn/20190824143222549.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
在目录中，我们可以看到两个 smali 文件夹，这是由于 dex 分包造成的，你暂时可以不用理解它，smali1 文件夹找不到对应的包，就去下一个找即可，注意要切换到 Project 目录。  
![](https://img-blog.csdnimg.cn/20190824155421532.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20190824155827107.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20190824155928415.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
接下来找到 a 方法，我们需要了解 Smali 语法才能读懂它，这一部分我打算下一节结合 Smali 插桩讲，大家也可以自行搜索和学习。  
我们现在只需要知道 ".method" 和 “.end method” 分别是方法开始和结束的地方即可。下图红框即 a 方法的两个重载方法，它们对应着我们前面分析的 Jadx 伪代码。  
![](https://img-blog.csdnimg.cn/20190824160215154.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
对比一下 Jadx 反编译的结果  
![](https://img-blog.csdnimg.cn/20190824160251815.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
显然 Smali 代码更长的那个才是我们需要的重载方法，在代码左边空白处单击即可下断点。  
![](https://img-blog.csdnimg.cn/2019082416051319.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

下好断点后，就可以准备开始运行了。  
首先，红框一圈出来的设备信息处，必须要有一个蓝色的设备在运行，如果你的是黑色，可以重启一下模拟器。  
![](https://img-blog.csdnimg.cn/20190824160705526.png)  
![](https://img-blog.csdnimg.cn/20190824160821987.png)  
确定设备没问题后，点击那个带箭头的小虫子  
![](https://img-blog.csdnimg.cn/20190824160918970.png)  
![](https://img-blog.csdnimg.cn/20190824161124725.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
我们现在要找到我们应用的包名，查看应用包名以及其他信息的方法和工具非常多，我这里推荐一个非常优雅的工具 Apk Messenger, 在反编译的第一步，我们需要对应用进行查壳，我也建议使用它，因为它实在是难得的 UI 设计好看的反编译工具。这是它的官网，https://www.ghpym.com/apkinfo.html ，百度云也放了相应的资源。

![](https://img-blog.csdnimg.cn/20190824161447514.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
直接拖拽 Apk 进去  
![](https://img-blog.csdnimg.cn/20190824161857155.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
细心的小伙伴可能会发现，它判定应用进行了腾讯加固，这似乎是一个误报。这是为什么呢？事实上，Apk 的查壳工具都并不算聪明，是通过检查 Apk 目录中是否由加固软件的特征文件判断的，我们来看一下 APK Messenger 的判断库。  
![](https://img-blog.csdnimg.cn/20190824162543942.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
再用 360 压缩或者别的压缩工具打开 Apk，会在 assets 目录下找到这些。  
![](https://img-blog.csdnimg.cn/20190824163134521.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
所以不用去管它，我们反编译内容正常，就不用考虑它为什么是个 “假加固” 的事了。  
现在我们知道了包名，在模拟器中打开 App，在列表中找到 com.sina.sinablog 附加调试即可，反复调试 Smali 代码时，可能会出现进程列表里没有这个 App 的状况，重启 App 即可。  
![](https://img-blog.csdnimg.cn/20190824163523936.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20190824163532653.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
网上很多教程都让你用 DDMS 查看端口，再用 adb 动态转发端口之类的，其实这些步骤一般都是不需要的。  
我们现在已经开始了 Smali 调试，只需要触发断点即可。  
![](https://img-blog.csdnimg.cn/20190824163714979.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
点击一条博客  
![](https://img-blog.csdnimg.cn/2019082416562479.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
关于 Android Studio 调试工具如何使用，网上已经有非常多的文章了，不熟悉的可以看一下这篇文章 https://blog.csdn.net/yy1300326388/article/details/46501871

我们这里需要使用到下图这些功能  
![](https://img-blog.csdnimg.cn/20190824170344225.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
我们来看一下 Smali 代码，我们下一篇才会讲 Smali 语法，但如果大家先学习了 Smali 语法，会对 Smali 调试有非常大的帮助。![](https://img-blog.csdnimg.cn/20190824171255177.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
运行完 move-result-object v0 这一行后，我们在 Watches 监视器中添加 v0，接下来我们就可以一路 F8，感受它的变化了。  
初始化 hashmap 时，那个方法已经塞进去五个字段了，一步步 F8，你会发现 v0 里的字段越来越多，没过多久，Get 请求的九个字段就全部躺在了 v0 中。  
![](https://img-blog.csdnimg.cn/20190824171831655.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20190824171933178.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
光靠 F8 一行一行走是没办法得知的，你可以退出调试模式，更加精细的看一下，F7 进入到子方法，Shift+F8 跳出方法，这样子多走几遍，你就理解了。  
在 m 方法中，获得了 5 个字段  
![](https://img-blog.csdnimg.cn/20190824172516669.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
出了 m 方法后，得到了三个字段  
![](https://img-blog.csdnimg.cn/20190824172853707.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
我们用 Jadx 的 Java 代码上标记一下  
![](https://img-blog.csdnimg.cn/20190824173142614.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)![](https://img-blog.csdnimg.cn/20190824174038963.png)  
SIGN 是怎么生成的呢？  
![](https://img-blog.csdnimg.cn/20190824173310516.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
不熟悉 Smali 可以在 Jadx 中对应看一下  
![](https://img-blog.csdnimg.cn/20190824174059108.png)  
sign 值是由 CpltUtil.invoke() 方法生成的，参数是两个字符串，第一个是固定字符串 “/apicheck/blog”，第二个参数是八个参数的字符串，我们可以用计算器查看一下。

在如图这一步，v0 即包含了 8 个字段的 map，点击图中红框的计算器，它叫 Evaluate Expression，可以在这儿运行各种各样的表达式  
输入 new JSONObject(v0).toString();  
你会发现 JSONObject 飘红，按照提示进行导包  
![](https://img-blog.csdnimg.cn/20190824175225537.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
导包后 Evaluate  
![](https://img-blog.csdnimg.cn/20190824175249355.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
好吧，报错，那我们就不转 JsonObject 了，直接 toString 转成字符串看看什么样  
![](https://img-blog.csdnimg.cn/20190824175435525.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/2019082417544743.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
结果为 {is_default=0, login_uid=, blog_uid=1260074450, article_id=4b1b35d20102yvlj, appver=6.1.2, appid=2, deviceid=e9ec21f2f9a7dc8f9c4e10694bdc6143, chno=515_104}，和预期一样。  
接下来我们在 Jadx 中看一下这个 CpltUtil.invoke() 方法

![](https://img-blog.csdnimg.cn/20190824181019773.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
一看这名字，似乎是个 native 方法，Ctrl + 左键进入  
![](https://img-blog.csdnimg.cn/20190824181142817.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
竟然是一个 native 层的加密函数，在之后 native 层破解时我们再提它，大家可以先试一下。  
去 lib 库中找到 libcrossplt.so 文件，在 ida 中反编译，这个函数是静态注册的，所以很容易就可以在 Exports 列表中找到它，之后 F5 反汇编成 c 代码，静态分析 c 代码或者 ida 动态调试即可。

不入 so 层，就还没有入门逆向，大家加油，下一篇可能讲 Smali 相关的东西。  
![](https://img-blog.csdnimg.cn/20190824181838553.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)