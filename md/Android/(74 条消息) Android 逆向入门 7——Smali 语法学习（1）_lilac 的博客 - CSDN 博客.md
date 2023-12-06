> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38851536/article/details/100173323?spm=1001.2014.3001.5502)

_这一节我们一起探讨 smali 语法和 smali 在 Android 逆向中的应用，它是 Android 逆向世界中不可或缺的一部分。_

![](https://img-blog.csdnimg.cn/2019083115215915.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
简单的来说，Dex [反编译](https://so.csdn.net/so/search?from=pc_blog_highlight&q=%E5%8F%8D%E7%BC%96%E8%AF%91)的结果就是 Smali，Smali 和 dex 之间的关系，我们常常称为转化 (convert)；Dex 是晦涩的二进制文件，Smali 是人可以读懂的代码，而 Jadx 等工具就是解析 Smali 文件，翻译成 Java 代码，其准确度差了不止一个档次了。

我们尝试一下新的形式，用问与答的形式表述一下 Smali 学习的一些困惑。

问：学习 Smali 有啥用？  
答：可以实现对 Apk 源代码的修改。

问：我只是想搞懂 Apk 的运行逻辑，我为什么要修改 Apk 源代码？  
答：因为 Apk 程序的运行是非常复杂的，你很难光靠静态分析 Jadx 反编译得到的 Java 代码就搞懂程序。

问：你这样说太虚了，举个例子把。  
答：在上一节中，我们找到了疑似 sign 算法生成的地方，可是我们怎么确定这儿就是 sign 生成的地方？

![](https://img-blog.csdnimg.cn/20190831174807701.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

问：可以抓一次数据包，如果这个函数的返回值和抓包得到的 sign 一致，那么就可以判断了吧？  
答：抓包很好做，那怎么拿到这个函数的返回值呢？

问：我好像不知道。  
答：我们之前对代码的分析是静态的，现在我们要动态分析程序和代码，以小红书 sign 为例，它最后 return lowerCase2，那我们在上面一行 print(lowerCase2)，然后查看日志不就行了吗？

问：print 是 Python 中用来打印输出的吧，Android 用什么？  
答：对对对，Java 中打印用 System.out.println() 方法打印输出，Android 提供了更方便的 Log 输出，形式如 Log.d(“tag”,“Info”)，我们可以通过 Android Logcat 日志查看器查看和监测一台 Android 设备运行时产生的所有日志，再通过 tag 标签 / 包名 / 进程 id 等标识筛选和监测我们需要的那一类 Log 日志即可。

问：之前通过 Android Studio，我确实通过 Logcat 菜单查看过 Log 日志，可是那是开发 App 用的工具，逆向时我们怎么通过它检测 Log 呢？  
答：逆向 App 时，Android Studio 也是非常强大的工具，连接设备后，我们确实也可以通过 Android Studio 内置的 Logcat 工具栏查看日志，但 Android Studo 太庞大了，我们可以使用 Android SDK 提供的 Android Devices Monitor，它更加轻量级，但功能完善。打开 SDK 目录——tools——ddms.bat/monitor.bat 都可以。

![](https://img-blog.csdnimg.cn/20190831175933932.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

问：我现在已经知道怎么查看日志了，那该怎么在程序中插入一行 Log.d(“疑似 sign 值”, lowerCase2) 呢？  
答：将 Dex 返汇编成 Smali 文件，找到这个类对应的 Smali 文件，在对应的位置插入 Log.d(“疑似 sign 值”, lowerCase2) 这句 Android 代码所对应的 Smali 代码，然后将 Smali 代码回编译成 Dex 文件，重打包成 Apk 在手机上运行，手机连接电脑，即可在 Android Device Monitor(简称 DDMS) 中查看 Logcat。

问：这就是 Smali 插桩对吗？  
答：是的，Smali 插桩可以输出一些关键的信息值，比如函数的参数，函数的返回值，除此之外，Smali 插桩还可以帮助我们探测程序的运行逻辑，比如你在 if 语句内放一个 Log.d(”Test“，0), 如果 Logcat 中没有看到这句日志，就说明程序没有走 if 这个代码块。

> 引用一下维基百科的插桩定义：程序插桩，最早是由 J.C. Huang  
> 教授提出的，它是在保证被测程序原有逻辑完整性的基础上在程序中插入一些探针（又称为 “探测仪”），通过探针的执行并抛出程序运行的特征数据，通过对这些数据的分析，可以获得程序的控制流和数据流信息，进而得到逻辑覆盖等动态信息，从而实现测试目的的方法。

问：那 Smali 除了插桩还能干啥？我听说 Hook 技术也可以实现插桩。  
答：确实，Hook 技术很强大，你可以 Hook 一个函数，它就受你摆布，不论是狸猫换太子换成另外一个函数，还是简单的输出它的参数和返回值都是可以的，Android 平台的主流 Hook 框架目前是 Xposed 和 Frida，它们都很方便而且强大。

问：那我们为什么不学它们而学 Smali?  
答：因为 Smali 是基础中的基础，搞 Android 逆向，就不能不懂 Smali 代码和 arm 汇编，更因为 Smali 动态调试需要你理解 Smali，单论插桩能力，Frida 插桩甩出 Smali 插桩三条街，我们之后很快就会提到它。

问答结束，下一讲中，我们将使用 Smali 插桩技术在 Log 中输出疑似 Sign 生成函数的参数和返回值。