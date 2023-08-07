> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [cloud.tencent.com](https://cloud.tencent.com/developer/article/1004417)

> 当一个 App 发布之后，突然发现了一个严重 bug 需要进行紧急修复，这时候公司各方就会忙得焦头烂额：重新打包 App、测试、向各个应用市场和渠道换包、提示用户升级、用户下载、覆盖安装。

> 作者：johncz

#### 1. 背景

当一个 App 发布之后，突然发现了一个严重 bug 需要进行紧急修复，这时候公司各方就会忙得焦头烂额：重新打包 App、测试、向各个应用市场和渠道换包、提示用户升级、用户下载、覆盖安装。有时候仅仅是为了修改了一行代码，也要付出巨大的成本进行换包和重新发布。

这时候就提出一个问题：有没有办法以补丁的方式动态修复紧急 Bug，不再需要重新发布 App，不再需要用户重新下载，覆盖安装？

虽然 Android 系统并没有提供这个技术，但是很幸运的告诉大家，答案是：可以，我们 QQ 空间提出了热补丁动态修复技术来解决以上这些问题。

#### 2. 实际案例

空间 Android 独立版 5.2 发布后，收到用户反馈，结合版无法跳转到独立版的访客界面，每天都较大的反馈。在以前只能紧急换包，重新发布。成本非常高，也影响用户的口碑。最终决定使用热补丁动态修复技术，向用户下发 Patch，在用户无感知的情况下，修复了外网问题，取得非常好的效果。

### 3. 解决方案

该方案基于的是 android dex 分包方案的，关于 dex 分包方案，网上有几篇解释了，所以这里就不再赘述，具体可以看[这里](https://cloud.tencent.com/developer/tools/blog-entry?target=https://m.oschina.net/blog/308583)

简单的概括一下，就是把多个 dex 文件塞入到 app 的 classloader 之中，但是 android dex 拆包方案中的类是没有重复的，如果 classes.dex 和 classes1.dex 中有重复的类，当用到这个重复的类的时候，系统会选择哪个类进行加载呢？

让我们来看看类加载的代码：

![](https://mc.qcloudimg.com/static/img/d23f02e403bcd2538c2fa2c1d676fe3b/image.jpg)

一个 ClassLoader 可以包含多个 dex 文件，每个 dex 文件是一个 Element，多个 dex 文件排列成一个有序的数组 dexElements，当找类的时候，会按顺序遍历 dex 文件，然后从当前遍历的 dex 文件中找类，如果找类则返回，如果找不到从下一个 dex 文件继续查找。

理论上，如果在不同的 dex 中有相同的类存在，那么会优先选择排在前面的 dex 文件的类，如下图：

![](https://mc.qcloudimg.com/static/img/c54a807208cd978e5cd2a89cea5e89bc/image.jpg)

在此基础上，我们构想了热补丁的方案，把有问题的类打包到一个 dex（patch.dex）中去，然后把这个 dex 插入到 Elements 的最前面，如下图：

![](https://mc.qcloudimg.com/static/img/1ccd98a9a909d3736ef0bf557c0f36dc/image.jpg)

好，该方案基于第二个拆分 dex 的方案，方案实现如果懂拆分 dex 的原理的话，大家应该很快就会实现该方案，如果没有拆分 dex 的项目的话，可以参考一下谷歌的 multidex 方案实现。然后在插入数组的时候，把补丁包插入到最前面去。

好，看似问题很简单，轻松的搞定了，让我们来试验一下，修改某个类，然后打包成 dex，插入到 classloader，当加载类的时候出现了（本例中是 QzoneActivityManager 要被替换）：

![](https://mc.qcloudimg.com/static/img/b379f158255e7c49195a80f8bd1794f0/image.jpg)

为什么会出现以上问题呢？

从 log 的意思上来讲，ModuleManager 引用了 QzoneActivityManager，但是发现这这两个类所在的 dex 不在一起，其中：

1.  ModuleManager 在 classes.dex 中
2.  QzoneActivityManager 在 patch.dex 中

结果发生了错误。

这里有个问题, 拆分 dex 的很多类都不是在同一个 dex 内的, 怎么没有问题?

让我们搜索一下抛出错误的代码所在，嘿咻嘿咻，找到了一下代码：

![](https://mc.qcloudimg.com/static/img/0e528e4f96135c64b8c9f7483a4227a8/image.jpg)

从代码上来看，如果两个相关联的类在不同的 dex 中就会报错，但是拆分 dex 没有报错这是为什么，原来这个校验的前提是：

![](https://mc.qcloudimg.com/static/img/a4a0a85345bc41eb7fcad06dcf61dcc1/image.jpg)

如果引用者（也就是 ModuleManager）这个类被打上了 CLASS_ISPREVERIFIED 标志，那么就会进行 dex 的校验。那么这个标志是什么时候被打上去的？让我们在继续搜索一下代码，嘿咻嘿咻~，在 DexPrepare.cpp 找到了一下代码：

![](https://mc.qcloudimg.com/static/img/dc75f613f62cdc686051cf37a9d4fdbf/image.jpg)

这段代码是 dex 转化成 odex(dexopt)的代码中的一段，我们知道当一个 apk 在安装的时候，apk 中的 classes.dex 会被虚拟机 (dexopt) 优化成 odex 文件，然后才会拿去执行。

虚拟机在启动的时候，会有许多的启动参数，其中一项就是 verify 选项，当 verify 选项被打开的时候，上面 doVerify 变量为 true，那么就会执行 dvmVerifyClass 进行类的校验，如果 dvmVerifyClass 校验类成功，那么这个类会被打上 CLASS_ISPREVERIFIED 的标志，那么具体的校验过程是什么样子的呢？

此代码在 DexVerify.cpp 中，如下：

![](https://mc.qcloudimg.com/static/img/8ee22db1a82ceb2e81292bd4ec414d40/image.jpg)

1.  验证 clazz->directMethods 方法，directMethods 包含了以下方法：
2.  static 方法
    1.  private 方法
    2.  构造函数
3.  clazz->virtualMethods
4.  虚函数 = override 方法?

概括一下就是如果以上方法中直接引用到的类（第一层级关系，不会进行递归搜索）和 clazz 都在同一个 dex 中的话，那么这个类就会被打上 CLASS_ISPREVERIFIED：

![](https://mc.qcloudimg.com/static/img/3826b6ec1c42f563828ebb82b6432749/image.jpg)

所以为了实现补丁方案，所以必须从这些方法中入手，防止类被打上 CLASS_ISPREVERIFIED 标志。

最终空间的方案是往所有类的构造函数里面插入了一段代码，代码如下：

```
System.out.println(AntilazyLoad.class);

```

}`

![](https://mc.qcloudimg.com/static/img/d950cfd1e6117fa2362006ecd1ca9647/image.jpg)

其中 AntilazyLoad 类会被打包成单独的 hack.dex，这样当安装 apk 的时候，classes.dex 内的类都会引用一个在不相同 dex 中的 AntilazyLoad 类，这样就防止了类被打上 CLASS_ISPREVERIFIED 的标志了，只要没被打上这个标志的类都可以进行打补丁操作。

然后在应用启动的时候加载进来. AntilazyLoad 类所在的 dex 包必须被先加载进来, 不然 AntilazyLoad 类会被标记为不存在，即使后续加载了 hack.dex 包，那么他也是不存在的，这样屏幕就会出现茫茫多的类 AntilazyLoad 找不到的 log。

所以 Application 作为应用的入口不能插入这段代码。（因为载入 hack.dex 的代码是在 Application 中 onCreate 中执行的，如果在 Application 的构造函数里面插入了这段代码，那么就是在 hack.dex 加载之前就使用该类，该类一次找不到，会被永远的打上找不到的标志)

其中:

![](https://mc.qcloudimg.com/static/img/50aa7d75fbee8d0489d7dd63f3eec911/image.jpg)

之所以选择构造函数是因为他不增加方法数，一个类即使没有显式的构造函数，也会有一个隐式的默认构造函数。

空间使用的是在字节码插入代码, 而不是源代码插入，使用的是 javaassist 库来进行字节码插入的。

隐患:

```
    虚拟机在安装期间为类打上CLASS_ISPREVERIFIED标志是为了提高性能的，我们强制防止类被打上标志是否会影响性能？这里我们会做一下更加详细的性能测试．但是在大项目中拆分dex的问题已经比较严重，很多类都没有被打上这个标志。

```

如何打包补丁包：

1.  空间在正式版本发布的时候，会生成一份缓存文件，里面记录了所有 class 文件的 md5，还有一份 mapping 混淆文件。
2.  在后续的版本中使用 - applymapping 选项，应用正式版本的 mapping 文件，然后计算编译完成后的 class 文件的 md5 和正式版本进行比较，把不相同的 class 文件打包成补丁包。

备注: 该方案现在也应用到我们的编译过程当中, 编译不需要重新打包 dex, 只需要把修改过的类的 class 文件打包成 patch dex, 然后放到 sdcard 下, 那么就会让改变的代码生效。

> 文章来源公众号：QQ 空间终端开发团队 (qzonemobiledev)

**相关推荐**

#### [微信 Android 热补丁实践演进之路](https://www.qcloud.com/community/article/164816001481011793?fromSource=gwzcw.59524.59524.59524&from_column=20421&from=20421)

#### [【腾讯 TMQ】不会做 bug 分析？套路走起~](https://www.qcloud.com/community/article/970622001487728567?fromSource=gwzcw.59525.59525.59525&from_column=20421&from=20421)