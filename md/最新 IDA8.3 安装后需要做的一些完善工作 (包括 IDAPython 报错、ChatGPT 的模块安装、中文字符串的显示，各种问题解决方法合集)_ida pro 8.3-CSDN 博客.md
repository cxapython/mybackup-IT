> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/donglxd/article/details/135243027)

        前段时间看到论坛上说的 IDA8.3 泄露的梗, 个人表示对这个提前放出安装包的中国朋友点赞, 好东西就是应该尽快免费给大伙分享, 老是藏着掖着揣兜里有什么用那?

        既然最近有空, 就来鼓捣下这个 IDA8.3 吧! 首先去下面的博客下载安装包

([【逆向工具】IDA Pro v8.3 高级增强版 - 极核 GetShell (get-shell.com)](https://get-shell.com/1395.html "【逆向工具】IDA Pro v8.3 高级增强版 - 极核GetShell (get-shell.com)")), 因为是绿色版, 所以下载后解压到自己喜欢的目录即可, 我在电脑 D 盘新建了一个名为 "IDA8.3" 的文件夹并解压到里面, 如图:

![](https://img-blog.csdnimg.cn/direct/10787ee9b6674e928ebcf617349a825e.png)

 进入目录后, 可以看到 ida8.3 的所有安装文件, 如下图:

![](https://img-blog.csdnimg.cn/direct/73eca216521a487a81af3c18a56cd4c1.png)

其中, 我选中的两个文件是 ida.exe 和 ida64.exe 分别是分析 x86 和 x64 位程序用的, 给这两个文件都建立一个桌面快捷方式，名称随意，方便使用。如下图:

![](https://img-blog.csdnimg.cn/direct/84d9db051c7747e89a2c811696795490.png)

我们双击其中的一个测试下是否可以使用, 因为我安装是 Python3.12 最新版, 所以弹出如下错误:

![](https://img-blog.csdnimg.cn/direct/0e8ef2b4f68e400c8ff3360ea1394b11.png)

这个错误是因为 Python3.12 不再使用 "imp" 模块, 所以我们需要安装 python3.11 到 python3.8 之间版本的 python 才行。如果你是第一次安装, 别忘记设置 pip 源, 如果不会请动动你的手指搜索下。

为了方便, 我在安装 python 时使用了自定义安装到 D 盘，省的找目录麻烦。(如图 1)

安装 Python3.11 后还是会弹出刚刚的错误提示，看了下 IDA8.3 目录里的 readme 说明, 里面说了要使用 idapyswitch.exe 来设置下 python3.11 目录, 需要使用 cmd 命令提示符打开 ida8.3 的目录，如下图:

![](https://img-blog.csdnimg.cn/direct/a845f9f8624f4ac69bb311e4d4c40680.png)

 ![](https://img-blog.csdnimg.cn/direct/190b24790b354c80b06e133dc429672b.png)

然后执行 idapyswitch.exe 命令设置你 python 目录里的 python3.dll(具体说明请使用 idapyswitch.exe -h 查看) 

![](https://img-blog.csdnimg.cn/direct/2b2c1abc2fba438fa280fdeef0881af7.png) 请看成功运行命令后, 打开 ida8.3 就不报错了, 如图:

![](https://img-blog.csdnimg.cn/direct/e6c7c1d99c964309a3df3c1625177e07.png)

这时, 我们第一个小问题解决了。

打开 IDA8.3 加载一个程序后, 会发现输出窗口提示错误, 如下图:

![](https://img-blog.csdnimg.cn/direct/30e2cd2ebd0e4e7db0bb7d6ad89dfb54.png)

显然是 IDA8.3 的发布者集成了 OpenAI 插件, 但是我们的 python3.11 没有安装这个库导致的，所以解决的方法也很简单, 如下图, 使用刚刚的 cd 命令进入 python3.11 目录后, 执行 pip 安装就可以了.

```
//原始是一条注释,我们把//和后面的；分号去掉
// CULTURE="all"; // all letters of all Unicode blocks, plus all codepoints in .clt files
//修改后
CULTURE="all"// all letters of all Unicode blocks, plus all codepoints in .clt files
```

 安装结果如下 (如安装缓慢, 请使用 pip 的国内源或使用代理安装, 我的博客有使用代理进行 pip 安装的教程, 国内源教程请自行搜索):[Window 通过代理上网时, PIP 安装包还是很慢或者提示 Retrying after connection broken by ‘ProxyError‘错误_加了代理为什么 pip 还是走 pypi.org-CSDN 博客![](https://csdnimg.cn/release/blog_editor_html/release2.3.6/ckeditor/plugins/CsdnLink/icons/icon-default.png?t=N7T8) https://blog.csdn.net/donglxd/article/details/133920536](https://blog.csdn.net/donglxd/article/details/133920536 "Window通过代理上网时,PIP安装包还是很慢或者提示Retrying after connection broken by ‘ProxyError‘错误_加了代理为什么pip还是走pypi.org-CSDN博客")

![](https://img-blog.csdnimg.cn/direct/2b399974508f4f43bda10e110aa8ebaa.png)

安装 openai 库后重新进入 ida, 结果又有新的错误，如图:

 ![](https://img-blog.csdnimg.cn/direct/74eba59af3e148118f96843e93879da6.png)

一葫芦画瓢，用 pip 安装下这个 anytree 即可，命令如下:

```
pip install anytree
```

 ![](https://img-blog.csdnimg.cn/direct/f86fefc210634abaa3794edf02a31602.png)

再进入 ida 看看, 发现没有错误了，如图:

![](https://img-blog.csdnimg.cn/direct/0b8ef7dda7e148ffadb633e14902a1a3.png)

第二个问题也解决了。

现在我们来说说之前我博客发表的 IDA7.6 时的解决中文字符串显示的问题, 这次我用简单点的方法来帮助大家解决, 当然用快捷方式的方法也是有用的。

现进入 ida8.3 的安装目录中的 cfg 文件夹找到 ida.cfg 用记事本打开, 如图:

![](https://img-blog.csdnimg.cn/direct/933b6beb070843a1b6576dcd64228a83.png)

按 CTRL+F 搜索 DCULTRUE, 可以看到这里的参数和之前教程中的一样, 只是 D 大写了, 可能有些没有成功的朋友是没有在这个 "-DCULTRUE" 和程序路径之前加空格或者没有大写。

![](https://img-blog.csdnimg.cn/direct/b1ec4a0edc0a4d40b87d21858bb4b6af.png)

在搜索到的这句下面第三句, 修改如下:

```
//原始是一条注释,我们把//和后面的；分号去掉
// CULTURE="all"; // all letters of all Unicode blocks, plus all codepoints in .clt files
//修改后
CULTURE="all"// all letters of all Unicode blocks, plus all codepoints in .clt files
```

修改后 (这句之后的注释也可以去掉) 如图所示:

![](https://img-blog.csdnimg.cn/direct/5b82422620104bc09894dd4c56e2d2e6.png)

最后别忘了保存这个 ida.cfg 配置文件, 然后重新打开 IDA 加载之前的项目, 重新分析下程序, 如下图所示.

![](https://img-blog.csdnimg.cn/direct/fc1a33ab6da146f5aec28434e18f16a0.png)

然后使用 shift+F12 打开 string 字符串窗口, 右击任意字符串, 点击 Setup 设置下 ida 识别的最小字符串数量, 如图:

![](https://img-blog.csdnimg.cn/direct/ce6cbc5ff60744cca28386c448d3a84e.png)

在打开的窗口, 把最小识别长度设置为 1, 如下图:

![](https://img-blog.csdnimg.cn/direct/a7b0b9f7c2e24bc0bf1f0efb0ffc237d.png)

最后点击 Rebuild 重建下字符串.

![](https://img-blog.csdnimg.cn/direct/afe3647e22c14061bc7036c2b7e1cd31.png)

可以看到, 我们把 1 个字的字符串也识别出来了, 如下图:

![](https://img-blog.csdnimg.cn/direct/3ecd79d0f3b543bcbc23b7f1876c903c.png)

这样我们第三个问题也就解决了, 至此 IDA8.3 顺利安装成功了, 快开启你的逆向之旅吧! 

PS:2024.01.15 更新帖:

关于极核发布的 IDA8.3 又有更新, 简化了安装步骤, 点击程序目录中的 “IDA_Pro_8.3_绿化工具. exe” 即可一键安装 IDA8.3, 我之前文章中的部署 python 和第三方库等操作就不用了, 如下图所示:

![](https://img-blog.csdnimg.cn/direct/008a6d8974dc4e0e9d23b80307e9d73d.png)

使用也很简单, 按提示操作, 输入 1 回车后, 再输入 y 即可, 最后按任意键退出。一些 IDA8.3 的安装说明请看程序目录中的 "说明. txt" 文档.

顺便提一句, 如果你的 IDA7.7 以上使用我的搜索 UTF-8 中文字符串有问题, 可能是 IDA 的 StrongCC 中文插件造成的, 请看上面的 "说明. txt" 里关闭掉它 (CPACP = false), 然后重新分析程序、重新构建字符串即可。