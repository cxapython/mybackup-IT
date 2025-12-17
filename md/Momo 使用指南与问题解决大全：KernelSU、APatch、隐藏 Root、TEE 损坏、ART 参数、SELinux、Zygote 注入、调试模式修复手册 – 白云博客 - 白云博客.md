> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.bybk.cc](https://www.bybk.cc/2400.html)

> 适用 #KernelSU##APatch## 隐藏 root#Momo 应用相信大家略有耳闻，不知道的也没关系，本篇教

适用[#KernelSU#](https://www.coolapk.com/t/KernelSU?type=12)[#APatch#](https://www.coolapk.com/t/APatch?type=12)[#隐藏 root#](https://www.coolapk.com/t/%E9%9A%90%E8%97%8Froot?type=12)

![](https://www.bybk.cc/wp-content/uploads/2024/07/20240709004821176-09375690db1129bef9a496aa4125a37b29690e8b.jpeg)

[Momo](https://www.bybk.cc/tag/474.html "查看所有关于 Momo 的文章") 应用相信大家略有耳闻，不知道的也没关系，本篇[教程](https://www.bybk.cc/tag/69.html "查看所有关于 教程 的文章")我也会重新介绍它。当然除了介绍 Momo 外，本篇教程还要讲讲如何解决 Momo 里的一些提示。相信你看完本篇教程教程后，也可以像上图那样过 Momo。忘说了，上图中 Momo 显示 “校验” 或? 就是大家口中所说的通过 Momo 检测、Momo 已 (绕) 过、环境完美。

本篇教程共分为十部分，第一部分介绍 Momo 的[使用场景](https://www.bybk.cc/tag/585.html "查看所有关于 使用场景 的文章")以及用法；第二部分介绍 Momo 中哪些提示影响隐藏；第三部分是 (Momo 提示)“[服务无响应](https://www.bybk.cc/tag/588.html "查看所有关于 服务无响应 的文章")” 的解决方法；第四部分是 “[TEE 损坏](https://www.bybk.cc/tag/589.html "查看所有关于 TEE损坏 的文章")” 的解决方法；第五部分是 “[ART 参数异常](https://www.bybk.cc/tag/590.html "查看所有关于 ART参数异常 的文章")”的解决方法；第六部分是 “SELinux 处于宽容模式” 的解决方法；第七部分是 “Zygote 被注入” 的解决方法；第八部分是“[权限系统异常](https://www.bybk.cc/tag/593.html "查看所有关于 权限系统异常 的文章") / [包管理服务异常](https://www.bybk.cc/tag/594.html "查看所有关于 包管理服务异常 的文章")” 的解决方法；第九部分是 “处于[调试环境](https://www.bybk.cc/tag/595.html "查看所有关于 调试环境 的文章")” 的解决方法；第十部分是 “已[开启调试模式](https://www.bybk.cc/tag/596.html "查看所有关于 开启调试模式 的文章")” 的解决方法。

一、Momo 的使用场景以及用法
----------------

(1) [面具用户](https://www.bybk.cc/tag/598.html "查看所有关于 面具用户 的文章")看了我大号 [@topmiaohan](https://www.coolapk.com/u/topmiaohan) 的[隐藏 root](https://www.bybk.cc/tag/469.html "查看所有关于 隐藏root 的文章") 保姆级系列教程，并严格按照我教程里的步骤去隐藏，可还是有打不开和闪退的应用 (比如银行类金融类游戏类应用)

(2)[KernelSU](https://www.bybk.cc/tag/586.html "查看所有关于 KernelSU 的文章") 或 [APatch](https://www.bybk.cc/tag/587.html "查看所有关于 APatch 的文章") 用户遇到打不开和闪退的应用 (比如银行类金融类游戏类应用)

当出现以上 (1)(2) 场景中描述的情况时，可从本篇教程的置顶评论里下载 Momo，然后安装，安装完不要着急打开。如果你是面具用户，就按照我大号隐藏 root 保姆级教程里对银行类金融类应用隐藏 root 的步骤，去对 Momo 隐藏 root，然后再打开 Momo；如果是 KernelSU 或 APatch 用户，(由于它们会 “自动” 对 Momo 隐藏 root 的缘故)就不需要操作什么，直接打开 Momo 就行。

二、Momo 中影响隐藏的提示
---------------

如果对 Momo 隐藏 root 后打开 Momo，Momo 有提示我以下列举的这几项，你就去解决一下，也就是让 Momo 不显示 (不提示) 我以下列举的这几项。为什么这么说呢？因为经过我的长期研究发现，以下我列举的这几项，每一项都或多或少的可能会对隐藏 root 产生影响。有时候想办法让 Momo 检测不到我列举的这几项，也就有可能打开那些闪退或者无法运行的应用。对于以下 Momo 中的这些提示我也写了解决方法，大家可以根据自身 Momo 的提示来选择去看不同的部分。

Momo 提示中影响隐藏 root 的提示 (基于 4.4.1 版本的 Momo 测试) 如下：

找到 [Zygisk](https://www.bybk.cc/tag/468.html "查看所有关于 Zygisk 的文章") 找到可执行程序 “su”

(如果你是面具用户，那么这两项解决方法见我大号 [@topmiaohan](https://www.coolapk.com/u/topmiaohan) 的隐藏 root 保姆级系列教程)

服务无响应。请允许 Momo 启动服务，部分设备可能需要关闭电池优化。

(解决方法见本篇教程第三部分)

TEE 损坏

(解决方法见本篇教程第四部分)

ART 参数异常

(解决方法见本篇教程第五部分)

SELinux 处于宽容模式

(解决方法见本篇教程第六部分)

Zygote 被注入

(解决方法见本篇教程第七部分)

权限系统异常 / 包管理服务异常

(解决方法见本篇教程第八部分)

处于调试环境

(解决方法见本篇教程第九部分)

已开启调试模式

(解决方法见本篇教程第十部分)

Momo 提示中不影响隐藏 root 的提示 (基于 4.4.1 版本的 Momo 测试) 如下：

找到 Magisk

分区挂载异常

设备正在运行非原厂系统

找到被 Magisk 模块修改的文件

数据未加密，挂载参数被修改

init.rc 被修改

存在 Magisk 或 TWRP 等可疑文件

非 SDK 接口的限制失效

Bootloader 未锁定

发现代码注入

Seccomp 未开启

三、服务无响应
-------

“服务无响应” 这个提示，常出现在真我、OPPO、一加等机型，出现这个提示，你就没法进一步用 Momo 查看系统环境了。究其原因好像因为这些机型的系统会干扰最新 Momo(4.4.1 版本) 的检测服务，导致 4.4.1 版本的 Momo 无法进一步检测。

解决方法：如果你不是上述机型，可以在评论区留言咨询。如果你是上述机型，请卸载 4.4.1 版本的 Momo，安装旧版本 4.3.1 的 Momo(与 4.4.1 版本区别不大)，4.3.1 版本的 Momo 可在本篇教程的置顶评论里获取。

![](https://www.bybk.cc/wp-content/uploads/2024/07/20240709004822456-c1748dc2a6b4a4792f920d9a5c96c15e921f48c2.jpeg)

四、TEE 损坏
--------

Momo 提示 “TEE 损坏” 说明你设备的 TEE 处于永久损坏或者 “假死” 状态，且无论是永久损坏还是 “假死” 状态，它们都是很难修复的，不过 “TEE 损坏” 时，还有一个情况需要你解决一下，具体请往下看。

Momo 提示 “TEE 损坏” 时，还会连带出现的 “Bootloader 未锁定”，这个连带出现的“Bootloader 未锁定” 需要你去解决一下(因为它会影响隐藏)，解决方法如下：

如果你是官方面具、阿尔法面具、KernelSU 的 “TEE 损坏” 用户，那么安装 [Shamiko](https://www.bybk.cc/tag/597.html "查看所有关于 Shamiko 的文章") 后 Momo 就不会提示 “Bootloader 未锁定” 了；如果你是 kitsune Mask/kitsune Lite(德尔塔面具)的用户，由于不能安装 Shamiko 的关系，你可以安装一个我自研的面具模块 (一个名为“改善系统环境之重置敏感属性” 的 Magisk 模块)，该模块可在本篇教程的置顶评论里获取。一般安装完重启开机后，Momo 应该就不会再提示 “Bootloader 未锁定” 了。

五、ART 参数异常
----------

如果你的 Momo 提示 “ART 参数异常”，请首先检查安装的 [LSPosed](https://www.bybk.cc/tag/476.html "查看所有关于 LSPosed 的文章") 版本是不是最新。目前最新的 LSPosed 版本是 1.9.2(7058)，如果你不是该版本，请升级到该版本。如果你不知道从哪里下载 1.9.2(7058) 的 LSPosed，可以看下本篇教程的置顶评论。

如果你已经安装了最新版本 LSPosed，但 Momo 仍然提示 “ART 参数异常”，请检查你的安装版本是不是安卓 9？如果不是，请跳过本段内容看下一段！如果是，请用 MT 管理器删除 / data/adb/modules/zygisk_lsposed 或 / data/adb/modules/riru_lsposed 下的 system.prop 文件。删除完请重启手机，开机后卸载重新安装 Momo，Momo 应该就不会提示“ART 参数异常” 了。

如果你已经安装了最新版本 LSPosed，且安卓版本大于 9，但 Momo 仍然提示 “ART 参数异常”，请检查你的 Momo 是不是还提示了 “SELinux 处于宽容模式”？

如果是，请先看以下第六部分来解决 Momo 提示 “SELinux 处于宽容模式”，因为 SELinux 处于宽容模式也会导致 Momo 提示 “ART 参数异常”。如果 Momo 没有提示 “SELinux 处于宽容模式”，但 Momo 仍然提示 “ART 参数异常”，你可以卸载重装一下 Momo 再看看。

六、SELinux 处于宽容模式
----------------

Momo 提示 “SELinux 处于宽容模式” 的原因和解决方法我都写在了以下这两篇教程里，你可以根据你系统选择其中的一篇来查看。如果你是官方系统请看第一期，如果你是非官方系统 (比如官改系统、第三方系统、移植系统) 请看第二期。

[[链接]@By_miaohan 的图文…](https://www.coolapk.com/feed/52960070?shareKey=ZGE5ZDI1NTZhYjVhNjYwYTllMWE~&shareUid=4161227&shareFrom=com.coolapk.market_13.4.1)

[[链接]@By_miaohan 的图文…](https://www.coolapk.com/feed/45100028?shareKey=NjlmNTAzMTNhODQ4NjYwYTllNDk~&shareUid=4161227&shareFrom=com.coolapk.market_13.4.1)

七、Zygote 被注入
------------

Momo 提示 “Zygote 被注入”，是因为你启用 Xposed 模块的作用域选择了 Momo。考虑到有相当一部分小白不知道啥是“Xposed 模块” 和“作用域”，所以我先简单介绍下这两个词语的意思。

首先说一下 “Xposed 模块”，“Xposed 模块” 用专业的话解释可能对于初学者很难理解。所以如果你是初学者，知道 LSPosed 模块界面 (下图图二界面) 显示的那些应用就是 Xposed 模块就够了。以下图图二界面为例，我们可以看到启用的 Xposed 模块中有 “隐藏应用列表”。我们点一下“隐藏应用列表”，会进入图三界面。图三界面会显示你安装的所有应用，并且每个应用后面都带一个 □ 供你 ✓ 选，这就是作用域选择界面。我们从图三中还可以看到，“系统框架(推荐应用)” 已被自动 ✓ 选，这说明 “隐藏应用列表” 的在 LSPosed 里的作用域是“系统框架”。这时候我们如果在作用域里勾选 Momo、NativeTest(如图四)，就会看到 Momo 提示“Zygote 被注入”，NativeTest 也一堆异常。

![](https://www.bybk.cc/wp-content/uploads/2024/07/20240709004823589-02bf0fc6dc564a7be703dc47ce31f81401f62c92.jpeg)

总之，无论是 “隐藏应用列表”，还是其他的 Xposed 模块，它们的作用域只能选择“推荐应用”，不能选择非“推荐应用”，尤其是银行类金融类游戏类应用更加不能选择。一句话，在 LSPosed 里勾选银行类金融类游戏类应用(如下图) 是错误的！是不被允许的！部分人银行类金融类应用打不开或游戏封号，就是因为在 LSPosed 某个模块的作用域里选择了银行类金融类或者游戏类应用。

![](https://www.bybk.cc/wp-content/uploads/2024/07/20240709004824163-0d860f350db2a534b8312a635460bf940f4e0635.jpeg)

八、权限系统异常 / 包管理服务异常
------------------

Momo 提示 “权限系统异常、包管理服务异常”，是因为你安装了 “[核心破解](https://www.bybk.cc/tag/599.html "查看所有关于 核心破解 的文章")” 软件或者你是官改系统第三方系统。

解决方法：打开 “核心破解”，关闭“禁用软件包管理器签名验证” 选项，重启手机。等开机后打开 Momo，就看不到 Momo 提示 “权限系统异常或包管理服务异常” 了。如果还是不行，卸载 “核心破解” 软件，重启手机再试试。

如果你是官改系统第三方系统或者未安装 “核心破解” 却提示 “权限系统异常” 或“包管理服务异常”，那基本上是没有解决方法了。

九、处于调试环境
--------

出现这个提示的是因为你的系统开启了[全局调试](https://www.bybk.cc/tag/600.html "查看所有关于 全局调试 的文章")，当系统开启了全局调试容易被银行类金融类应用检测。

解决方法：要解决 Momo 提示 “处于调试环境”，可以安装一个我自研的 Magisk 模块(一个名为“改善系统环境之关闭全局调试” 的 Magisk 模块)，该模块可在本篇教程的置顶评论里获取。一般安装完重启开机后，Momo 就不会再提示 “处于调试环境” 了。

十、已开启调试模式
---------

已开启调试模式，是因为你在开发者选项里打开了 [USB 调试](https://www.bybk.cc/tag/601.html "查看所有关于 USB调试 的文章")，你去关闭 USB 调试就不会提示了。如果你不会进开发者选项，可以百度你机型的品牌。比如你是小米 10 手机，你可以百度搜索关键句 “小米手机如何进入开发者选项”。

如果你关闭 USB 调试后，Momo 还是提示 “已开启调试模式”，你可以强制结束 Momo 的后台运行再打开 Momo 试试。如果依旧不行，重启手机再试试。如果重启手机还不行，说明你的系统可能会在开机时自动开启 USB 调试。对于这种情况你可以安装一个我自研的 Magisk 模块(一个名为“改善系统环境之开机自动关闭 USB 调试” 的 Magisk 模块)，该模块可在本篇教程的置顶评论里获取。一般安装完重启开机后，Momo 就不会再提示 “已开启调试模式” 了。