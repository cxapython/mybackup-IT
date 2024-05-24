> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.qlgoo.com](https://www.qlgoo.com/3669/)

> 大早上起来醉生梦死，弥留之际发现有人说 8.3 泄漏了，一下子垂死病中惊坐起，我寻思今天也不是除夕啊怎么还提前过大年呢？由于我用的是 MacBook Pro，所以我首先整合了一份 Windows 下的 8.3，并打包为 Crossover 版本供 macOS 使用。

大早上起来醉生梦死，弥留之际发现有人说 8.3 泄漏了，一下子垂死病中惊坐起，我寻思今天也不是除夕啊怎么还提前过大年呢？由于我用的是 MacBook Pro，所以我首先整合了一份 Windows 下的 8.3，并打包为 Crossover 版本供 macOS 使用。虚拟机下速度感觉跟 7.7 差不多，但事实上 8.3 还是非常牛逼的。

一切内容基于原帖 https://www.52pojie.cn/thread-1861384-1-1.html 修改。不建议大家汉化使用，7.7 的汉化版本到现在使用的时候都有问题。注册表乱码你还删不掉乱码的注册表项。

1.  KeyGen 了我自己的 ID，大家自己可以自己改。
2.  增加了以下增强插件: 2.1 VS2019 黑色主题

2.2 增加了 WPeChatGPT 插件，密钥自己整，放到 Program Files (x86)/IDA Pro 8.3 (x86, x86_64)/plugins/WPeChatGPT.py 中。2.3 增加了 retsync 插件。2.4 增加了 Patching 增强插件，比自带的 patching 更强。2.5 增加了 StrongCC。2.6 增加了 Signsrch。

所有插件文件均提取自论坛 7.7 英文版整合包中的文件，无任何修改，仅集成到新版本中。

目前还在研究怎么把 7.7 的插件塞进 8.3 中，解决 arm64 没法 F5 的问题，如果有更新我会继续在本帖说明。

1.  增加 IDA 7.7 版本 arm/arm64/mips/powerpc 的反汇编插件到 IDA 中, 使用了 github 开源项目来劫持 DLL 导出表实现兼容。[https://github.com/x0rloser/ida_dll_shim](https://github.com/x0rloser/ida_dll_shim)
2.  添加了默认配置和字体注册表文件, Jetbrains ttf 字体下载后自行安装: [https://www.jetbrains.com/zh-cn/lp/mono/](https://www.jetbrains.com/zh-cn/lp/mono/)
3.  在 tg 上的 IDA 讨论组中经过”Drovosek”与 “croviso” 老外指点，通过修改 arm64.dll 文件的偏移 0x8DA16 从 0x75 改为 0xEB pass 修复掉一个 arm64 反汇编 ObjectC 代码时引起的 INTERR 50740 is “return registers must be part of SPOILED” 错误。
4.  使用时请将 IDA 直接解压到 C 盘，否则注册表中的 IDA Python DLL 会找不到导致无法加载 python 环境。如果有其他需要可以自己修改注册表中”Python3TargetDLL”=”C:\IDA_Pro_8.3\python38\python3.dll” 这一项目。
5.  IDA 8.3 的 arm64 反编译插件来自旧版本 7.7，出现崩溃和不兼容的问题是必然的，请不要报告这种问题了，基本解决不了或者解决了也会有更多的问题。目前也只是测试过基本上可以反编译并显示伪代码，M 系列芯片的 Mac 转译有概率会卡死, 望周知。最好的解决办法就是有人共享出 8.3 版本的 arm64.dll，不过这基本跟做梦没区别。

总的来说，已经支持了旧版本 arm64 插件导入使用，虽然可能会出现问题，但这是必然的。最后感谢 github 开源项目作出的贡献，很惭愧在这方面我没有做出什么有用的贡献。

请大家自行检查，我个人使用没有任何问题，但不排除原帖本身可能存在的各种问题，我仅做整合自用，请各位下载后自行检查 md5 值。

MD5 (/Program Files (x86)/IDA Pro 8.3 (x86, x86_64).zip) = 重新上传后不准了

MD5 (/Volumes/data/IDA Pro-7.7 QiuChenly v1.3.cxarchive) = 50091dbaf4f13403972d7d4c35b038f1

MD5 (/Volumes/data/IDA Pro-8.3 QiuChenly v1.0.cxarchive) = 7996f7e281e9381da3a8cbc25dff45f5

链接: [https://pan.baidu.com/s/1VayRmDOl_XhJU-3MwtRDvw?pwd=qcly](https://pan.baidu.com/s/1VayRmDOl_XhJU-3MwtRDvw?pwd=qcly) 提取码: qcly

我上传了我自己用的 Crossover 版本，这个可以自己安装 Crossover 导入后使用。

这两个是给 macOS 使用的版本：

1.  IDA Pro 7.7 52 论坛整合版
    
2.  IDA Pro 8.3 2024.1.1 提前泄漏整合版
    

自行安装 Crossover 导入使用。

================= 2023.12.15 Update 2.0 =================IDA Pro 8.3 v2.0Mac Wine 打包整合版本 + Windows 压缩包版本解压导入注册表即用: 链接: [https://pan.baidu.com/s/1501YPXWSkLaaHfcqVwIlfw?pwd=qcly](https://pan.baidu.com/s/1501YPXWSkLaaHfcqVwIlfw?pwd=qcly) 提取码: qcly

md5 Win.reg IDA_Pro_8.3_QiuChenly_v2.0.rar IDA\ Pro-8.3\ QiuChenly\ v2.0.cxarchiveWindows 需要导入此注册表配合字体使用最佳，字体下载：[https://www.jetbrains.com/zh-cn/lp/mono/](https://www.jetbrains.com/zh-cn/lp/mono/%3C/font%3E[/font)[[/font](https://www.jetbrains.com/zh-cn/lp/mono/%3C/font%3E[/font)]MD5 (Win.reg) = e8b9855f2c75054e893bda22115879bbWindows 压缩包解压版 MD5 (IDA_Pro_8.3_QiuChenly_v2.0.rar) = 83a46fc6b26c96ac2f745803fcdc5530Mac Wine Crossover 解压版 激活方法在 https://github.com/QiuChenlyOpenSource/InjectLib/tree/main/tool 中 Crossover Activation Script.command 运行即可。MD5 (IDA Pro-8.3 QiuChenly v2.0.cxarchive) = 1afa66e712b3aec3ad9977d147f12ecb

Windows 下使用不要忘记在环境变量加入 C:\IDA_Pro_8.3\python38 路径，否则加载不了 python 插件。

泄漏版来源与论坛原文 This IDA copy only have x86 and x64 decompiler.               

Well… we sad to day this:The original supplier, Doe, hold a copy of IDA Pro 8.3 andwe exchanged it to get it. We mark the date of release:Janurary 1st 2024                        

auth (auth.lol) and Doe agreed. auth also prepared for itSuddenly, on the 52pojie forum, Kenny0521, take our unpreparedrelease, posted on 52pojie, then spread around the world…not Dr.FarFar of course.                                      

We’re holding the original release, but… it’s over… they are going to nuke our release as stolen release, even through we got it first.

This is the end of BanG Software Private/Pirate ArchiveFarewell.- bang1338

Closed, blame @Kenny0521 on Telegram and 52pojie for this happened.And farewell.

The leader of this team is Bang1338, contact via Telegram or Discord,or contact: teambgspa@protonmail.com。