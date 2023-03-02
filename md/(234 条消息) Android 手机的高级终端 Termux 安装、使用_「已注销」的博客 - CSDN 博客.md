> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/lyshark_lyshark/article/details/125848602)

>  From：[https://www.sqlsec.com/2018/05/termux.html](https://www.sqlsec.com/2018/05/termux.html "https://www.sqlsec.com/2018/05/termux.html")
> 
> [Termux](https://so.csdn.net/so/search?q=Termux&spm=1001.2101.3001.7020) 高级终端安装使用配置教程 ：[https://www.cnblogs.com/cutesnow/p/11430833.html](https://www.cnblogs.com/cutesnow/p/11430833.html "https://www.cnblogs.com/cutesnow/p/11430833.html")  
> 神器 Termux 的使用记录：[https://cloud.tencent.com/developer/article/1609398](https://cloud.tencent.com/developer/article/1609398 "https://cloud.tencent.com/developer/article/1609398")

[adb](https://so.csdn.net/so/search?q=adb&spm=1001.2101.3001.7020) shell 下使用 termux：[https://github.com/alwaystest/blog/issues/68](https://github.com/alwaystest/blog/issues/68 "https://github.com/alwaystest/blog/issues/68")

ttyd --- Share your terminal over the web：[https://github.com/tsl0922/ttyd](https://github.com/tsl0922/ttyd "https://github.com/tsl0922/ttyd")

termux-app：[https://github.com/termux/termux-app](https://github.com/termux/termux-app "https://github.com/termux/termux-app")  
python 脚本在手机或安卓系统上运行：[https://www.zhihu.com/question/28654702](https://www.zhihu.com/question/28654702 "https://www.zhihu.com/question/28654702")  
安装 [python 库](https://so.csdn.net/so/search?q=python%E5%BA%93&spm=1001.2101.3001.7020) + tasker 调用：[https://www.jianshu.com/p/c239a7eaadba](https://www.jianshu.com/p/c239a7eaadba "https://www.jianshu.com/p/c239a7eaadba")  
AidLearning-FrameWork ：[https://github.com/aidlearning/AidLearning-FrameWork](https://github.com/aidlearning/AidLearning-FrameWork "https://github.com/aidlearning/AidLearning-FrameWork")

安卓

*   **termux**(**安卓 5.0 以上**)。Termux 是 Android 手机上一个高级的终端模拟器软件，开源且不需要 root，支持 apt 管理软件包，十分方便安装软件包，完美支持 Python、PHP、Ruby、Go、Nodejs、MySQL 等。Termux  相当于在安卓上搭建了一个 Linux 平台，所以在 Linux 上 能干的事情很多在手机上也都办得到。由于安卓平台的开放性，类似 termux 的手机神器还有很多。不说各类强大的编程 IDE，单是 termux 这样的 Linux 平台类软件就很多，如 GnuRoot 系列，LinuxDisplay 系列等。这其中 termux 很受人欢迎。随着智能设备的普及和性能的不断提升，如今的手机、平板等的硬件标准已达到了初级桌面计算机的硬件标准, 用心去打造完全可以把手机变成一个强大的工具.。**termux 还有许多插件**：
    
    ![](https://img-blog.csdnimg.cn/img_convert/f1960594926b07deb8cfc6bc821c353c.png)
    
*   **gnuroot debian**。GNU 属于大而全，里面啥模块都有，安装包也大，termux 如果不够用就直接用 GNU 。GnuRoot 可以执行 python，java，c，php 。(gnu 更方便，直接 apt install python-scipy 之类搞定)。

IOS 推荐

*   pythonista(付费)。**pythonista** 只针对 python

1、在 Android 上安装 Termux
======================

安装 Termux 的三种方法：

*   1. Google Play。Google Play 下载的**版本**比酷安要新，有能力建议下载 Google PLay **版本。**
*   2.  Fiord 。 F-Droid 客户端：[https://f-droid.org/packages/com.termux/](https://f-droid.org/packages/com.termux/ "https://f-droid.org/packages/com.termux/")
*   3. 直接下载 Termux 的 APK 安装包进行安装，但是这种方式安装后将不会收到更新通知。 安装 **Termux 应用程序** ( [https://opensource.com/article/20/8/termux](https://opensource.com/article/20/8/termux "https://opensource.com/article/20/8/termux") )。

Termux 是一个强大的终端仿真器，它提供了所有最流行的 Linux 命令，加上数百个额外的包，以便于安装。它不需要任何特殊的权限，可以使用默认的 Google Play 商店 ( ​[https://play.google.com/store/apps/details?id=com.termux](https://play.google.com/store/apps/details?id=com.termux "https://play.google.com/store/apps/details?id=com.termux") )，或者开源应用仓库 F-Droid  ( [https://f-droid.org/repository/browse/?fdid=com.termux](https://f-droid.org/repository/browse/?fdid=com.termux "https://f-droid.org/repository/browse/?fdid=com.termux") ) 来安装。安装后如图所示：

![](https://img-blog.csdnimg.cn/img_convert/f121baab43eab5ea425d63091331b05f.jpeg)​

*   1. 第一部分是 termux 官方网站和相关资源， github 和官方 wiki 有很多资源供进一步学习。
*   2. 第二部分介绍了个包管理器命令 pkg，给出了四个命令。最后的 help 是通用的，前面分别是搜索 / 安装 / 升级包。跟 linux 的 apt/apt-get, python 的 pip 差不多，实际上直接用 apt 命令也可以的。

安装 Termux 后，启动它并使用 Termux 的 pkg 命令执行一些必要的软件安装。

*   订阅附加仓库 root-repo ：**pkg install root-repo**
*   执行更新，使所有安装的软件达到最新状态： **apt update     // 更新源  
            apt upgrade  // 升级软件包**
*   安装 Python：**pkg install python**

![](https://img-blog.csdnimg.cn/img_convert/263b968a82b328e3c2164a603bfc9708.jpeg)​

 安装和自动配置完成后，就可以构建你的应用了。

2、基本操作
======

长按屏幕
----

显示菜单项 (包括复制、粘贴、更多)，此时屏幕出现可选择的复制光标

![](https://img-blog.csdnimg.cn/img_convert/1b38547d0c5771074bd0a50eed19a1b8.png)

从左向右滑动
------

显示隐藏式导航栏，可以新建、切换、重命名会话 session 和调用弹出输入法

 ![](https://img-blog.csdnimg.cn/img_convert/c3f3f6db4aacbd96fed78675b82a488e.png)

显示扩展功能按键
--------

扩展功能键是什么? 就是 PC 端常用的按键如: ESC 键，CTR 键，TAB 键, 但是手机上难以操作的一些按键.

 ![](https://img-blog.csdnimg.cn/img_convert/853811f4ad2d8c343addf09e026e1aed.png)

*   方法一：从左向右滑动, 显示隐藏式导航栏，长按左下角的 KEYBOARD。
*   方法二：使用 Termux 快捷键：**音量 +** + **Q 键**

3、更新源、升级软件包
===========

下载安装后，要首先 **更新、升级软件包**，国内使用 termux 安装包多少有点尴尬，所以更换 Termux 清华大学源，加快软件包下载速度。

*   编辑文件：**vim /data/data/com.termux/files/usr/etc/apt/sources.list**
*   输入：**deb https://mirrors.ustc.edu.cn/termux/apt/termux-main stable main**

清华大学开源软件镜像站：[https://mirrors.tuna.tsinghua.edu.cn/help/termux/](https://mirrors.tuna.tsinghua.edu.cn/help/termux/ "https://mirrors.tuna.tsinghua.edu.cn/help/termux/")

就将原来的官方源，替换为清华源了。(可以将原来的源加上 # 来注释掉) 按 ESC 然后输入 :wq 保存并退出。上面是官方推荐的方法，其实还有更简单的方法，类似于 Linux 下直接编辑源文件：

```
pkg update
pkg install vim curl wget git unzip unrar
```

换源其实就是手动修改下面的三个文件

编辑 `$PREFIX/etc/apt/sources.list` 修改为如下内容

# The termux repository mirror from TUNA: deb [https://mirrors.tuna.tsinghua.edu.cn/termux/termux-packages-24](https://link.zhihu.com/?target=https%3A//mirrors.tuna.tsinghua.edu.cn/termux/termux-packages-24 "https://mirrors.tuna.tsinghua.edu.cn/termux/termux-packages-24") stable main

编辑 `$PREFIX/etc/apt/sources.list.d/science.list` 修改为如下内容

# The termux repository mirror from TUNA: deb [https://mirrors.tuna.tsinghua.edu.cn/termux/science-packages-24](https://link.zhihu.com/?target=https%3A//mirrors.tuna.tsinghua.edu.cn/termux/science-packages-24 "https://mirrors.tuna.tsinghua.edu.cn/termux/science-packages-24") science stable

编辑 `$PREFIX/etc/apt/sources.list.d/game.list` 修改为如下内容

# The termux repository mirror from TUNA: deb [https://mirrors.tuna.tsinghua.edu.cn/termux/game-packages-24](https://link.zhihu.com/?target=https%3A//mirrors.tuna.tsinghua.edu.cn/termux/game-packages-24 "https://mirrors.tuna.tsinghua.edu.cn/termux/game-packages-24") games stable

换好源后，记得 update，但是不需要 upgrade：**apt update**

# 安装基本工具

```
apt update	 // 更新源
apt upgrade  // 升级软件包
 
apt install git    // 分布式管理工具
apt install wget   // 下载工具
apt install vim    // vim编辑器
apt install tar    // 解压缩工具
apt install less   // termux下vim支持触摸移动光标移动位置
```

```
Ctrl+A -> 将光标移动到行首
Ctrl+C -> 中止当前进程
Ctrl+D -> 注销终端会话
Ctrl+E -> 将光标移动到行尾
Ctrl+K -> 从光标删除到行尾
Ctrl+L -> 清除终端
Ctrl+Z -> 挂起(发送SIGTSTP到)当前进程
```

**4、常用快捷键**
===========

**Ctrl 键**是终端用户常用的按键，但大多数触摸键盘都没有这个按键。为此 Termux 使用**音量减小**按钮来模拟 Ctrl 键。 例如，在触摸键盘上按音量减小 + L 发送与在硬件键盘上按 Ctrl + L 相同的输入。

```
音量加+E -> Esc键
音量加+T -> Tab键
音量加+1 -> F1(和音量增加+ 2→F2等)
音量加+0 -> F10
音量加+B -> Alt + B，使用readline时返回一个单词
音量加+F -> Alt + F，使用readline时转发一个单词
音量加+X -> Alt+X
音量加+W -> 向上箭头键
音量加+A -> 向左箭头键
音量加+S -> 向下箭头键
音量加+D -> 向右箭头键
音量加+L -> | (管道字符)
音量加+H -> 〜(波浪号字符)
音量加+U -> _ (下划线字符)
音量加+P -> 上一页
音量加+N -> 下一页
音量加+. -> Ctrl + \(SIGQUIT)
音量加+V -> 显示音量控制
音量加+Q -> 显示额外的按键视图
```

**音量加键** 也可以作为产生特定输入的 **特殊键**。

```
pkg search <query>              搜索包
pkg install <package>           安装包
pkg uninstall <package>         卸载包
pkg reinstall <package>         重新安装包
pkg update                      更新源
pkg upgrade                     升级软件包
pkg list-all                    列出可供安装的所有包
pkg list-installed              列出已经安装的包
pkg show <package>              显示某个包的详细信息
pkg files <package>             显示某个包的相关文件夹路径
```

**5、基本命令**
==========

Termux 除了支持 apt 命令外，还在此基础上封装了 pkg 命令，pkg 命令向下兼容 apt 命令。

```
~ > echo $HOME
/data/data/com.termux/files/home
 
~ > echo $PREFIX
/data/data/com.termux/files/usr
 
~ > echo $TMPPREFIX
/data/data/com.termux/files/usr/tmp/zsh
```

目录结构 和 特殊环境变量 **PREFIX**
------------------------

```
Enter a number, leave blank to not to change: 14
Enter a number, leave blank to not to change: 6
```

长期使用 Linux 的朋友可能会发现，这个 HOME 路径看上去可能不太一样，为了方便，Termux 提供了一个特殊的环境变量：**PREFIX**

![](https://img-blog.csdnimg.cn/img_convert/1d76f4d1310582df9db0883c716d2e91.png)

**6、 更换配色**
===========

使用 zsh 来替代 bash 作为默认 shell。可以使用一键安装脚本来安装，执行下面这个命令确保已经安装好了 curl，没有的话根据它的提示安装，你没安装的话，执行了下面这条语句，它会给你一条安装 curl 的语句的。

```
apt install python3
apt install python3-pip
```

Android6.0 以上会弹框确认是否授权，允许授权后 Termux 可以方便的访问 SD 卡文件。

![](https://img-blog.csdnimg.cn/img_convert/4cd54ca77c43859ad480e5356bc12874.png)

 脚本允许后先后有如下两个选项：

```
python2 -m pip install --upgrade pip
python -m pip install --upgrade pip
```

分别选择 **背景色** 和 **字体** 想要继续更改挑选配色的话，继续运行脚本来再次筛选:

```
pkg install clang
pip install ipython
pip3.6 install ipython
```

exit 退出，重启 sessions 会话生效配置，如想深入使用，请访问? GitHub

访问外置存储优化
--------

执行过上面的 zsh 一键配置脚本后，并且授予文件访问权限的话，会在家目录生成 storage 目录，并且生成若干目录，软连接都指向外置存储卡的相应目录

![](https://img-blog.csdnimg.cn/img_convert/4dcd6058ebfbe4e5c3399e3ffbe970da.png)

 **创建 QQ 文件夹软连接**

手机上一般经常使用手机 QQ 来接收文件，这里为了方便文件传输，直接在 storage 目录下创建软链接。

**QQ**

```
set fileencodings=utf-8,gb2312,gb18030,gbk,ucs-bom,cp936,latin1
set enc=utf8
set fencs=utf8,gbk,gb2312,gb18030
```

**TIM**

```
pkg install python, python2
pkg install python-dev, python2-dev
 
或者
 
apt install python python-dev python2 python2-dev
```

最后效果图如下：

![](https://img-blog.csdnimg.cn/img_convert/a6ee1addd23fb79489fda6817ed119c8.png)

这样可以直接在`home`目录下去访问 QQ 文件夹，非常方便文件的传输，大大提升了工作效率。[http://mirrors.tuna.tsinghua.edu.cn/termux](http://mirrors.tuna.tsinghua.edu.cn/termux "http://mirrors.tuna.tsinghua.edu.cn/termux")

oh my zsh 主题配色
--------------

编辑 **.zshrc** 配置文件

```
$ . ./venv/bin/activate
(env)$
```

第一行可以看到，默认的主题是 agnoster 主题：

![](https://img-blog.csdnimg.cn/img_convert/a73533b6b5475638df644dcdfdef84c6.png)

在 .oh-my-zsh/themes 目录下放着 oh-my-zsh 所有的主题配置文件。

下面是几款还可以的主题

**agnoster**

![](https://img-blog.csdnimg.cn/img_convert/c4e9aba3dce8b198a0eb31af63dbdcc0.png)

 **robbyrussell**

![](https://img-blog.csdnimg.cn/img_convert/2cbfdac3409f589b61ac39479973835b.png)

**jaischeema** 

![](https://img-blog.csdnimg.cn/img_convert/d9347f69034fc21e71b6efcd24070af2.png)

**re5et** 

![](https://img-blog.csdnimg.cn/img_convert/a7789f9c469974f9da3fde701d0c420f.png)

**junkfood** 

![](https://img-blog.csdnimg.cn/img_convert/b36cb20b3e7db0358dd3ad9c267c1746.png)

**cloud** 

![](https://img-blog.csdnimg.cn/img_convert/b547f99bf94f0847869578ccae4ac1d7.png)

**random** 

当然如果你是个变态的话, 可以尝试`random`主题, 每打开一个会话配色主题都是随机的.

```
apt install python 
apt install clang 
apt install fftw 
apt install libzmq 
apt install freetype 
apt install libpng 
apt install pkg-config
 
或者一条命令
 
apt install python clang fftw libzmq freetype libpng pkg-config
```

编辑启动问候语
-------

默认的启动问候语如下：

![](https://img-blog.csdnimg.cn/img_convert/572d1532f5a2d9ead84ea8f5cd941704.png)

这个对于初学者有一定的帮助在前期，随着对 Termux 的熟悉，这个默认的问候语就会显得比较臃肿。编辑问候语文件直接修改问候语:

```
apt-get install clang
apt-get install libxml2 libxml2-dev libxslt libxslt-dev
pip install lxml
```

![](https://img-blog.csdnimg.cn/img_convert/56d9e662c9a0eabcb1768369dc48f487.png)

**7、 管理员身份**
============

**手机没有 root**
-------------

利用 **proot** 工具来模拟某些需要 root 的环境：**pkg install proot**

然后终端下面输：**termux-chroot**

就可以模拟 root 环境，在这个 proot 环境下面，相当于是进入了 home 目录，可以很方便地进行一些配置。

![](https://img-blog.csdnimg.cn/img_convert/9896ef446180ccc4dd26eb0a18d1132f.png)

 在管理员身份下，输入 exit 可回到普通用户身份。

### 访问 sdcard 

如果要访问 sdcard 的目录，需要先运行：

```
apt install libffi libffi-dev openssl openssl-dev libxml2 libxml2-dev libxslt libxslt-dev
pip install scrapy
```

完成授权后，在 $HOME 目录会多出一个 storage 目录。安装完毕以后，换 Termux 包管理器换为国内的清华源，加快软件包下载速度。

### ssh 连接

安装 SSH 服务：**pkg install openssh**  
设置密码：**passwd**   
查询手机 ip，以实际手机 ip 为准：**ifconfig**  
查询当前用户：**whoami**  
确认 ssh 服务的监听端口：**netstat -ntlp | grep sshd**

信息确认后就可以在电脑端 cmd 下输入连接了，命令如下 (前提是电脑端 openssh 已经安上了)：**ssh u0_a123@192.168.0.1 -p 8022** 

这里假定用户名为 u0_a123(whoami 查询可得)。ip 为 192.168.0.1(ifconfig 查询可得)。至此，Termux 基本环境就搭好了！

开启 ssh 的指令是：

sshd  
sshd -p 9000  
上面的一个指令默认打开的端口是 8022，后一个指定了新的端口 9000。其他需要的软件自行安装。

### 安卓版 Linux --- Termux

：[https://zhuanlan.zhihu.com/p/92664273](https://zhuanlan.zhihu.com/p/92664273 "https://zhuanlan.zhihu.com/p/92664273")

### 安卓版 Linux --- Aid Learning

：[https://zhuanlan.zhihu.com/p/92161002](https://zhuanlan.zhihu.com/p/92161002 "https://zhuanlan.zhihu.com/p/92161002")

Termux 是一款安卓版的 Linux。 Aid Learning 是 Termux 的高仿！而且自带界面，自带 Python！还是国产的。Aid Learning 安装完毕后，需要等待，后台开始下载各种库。

官网：[https://www.aidlux.com/product](https://www.aidlux.com/product "https://www.aidlux.com/product")

下载地址：[https://www.pianwan.com/app/121392](https://www.pianwan.com/app/121392 "https://www.pianwan.com/app/121392")

​

aid learning，一般又称 AidLux。

【AidLux 是什么】：

*   AidLux 是一个基于 ARM 构建，同时支持多生态融合 (Android+Linux) 环境的 AI 应用开发和部署平台，为开发者带来强大、简单、无限创意可能的奇妙体验！

【AidLux 简介】：

*   基于 Android 底层 Linux kernel 构建了完整 Linux 的环境，并且与 Android 环境同时提供于用户访问。在为用户提供和原生 Linux 系统类似的命令行使用体验 (如通过 `apt` 命令进行包管理) 的同时，构建了图形化桌面环境，用户可以直接通过触摸屏或浏览器访问。
*   AidLux 补全了 AI 运行所需的所有基础科学计算包 / 库，支持了业界主流深度学习框架，并内置自主研发的 AI 智能加速技术，为开发者提供了一个 “AI 就绪” 的应用开发平台。

【AidLux 强大的功能】

*   1、一部设备同时运行两个系统环境，既是一部 Android 设备，同时也是一部 Linux 设备。两个生态的资源优势可同时被加以利用；
*   2、集成主流 AI 框架 (caffe、mxnet、keras、MNN、pytorch、tensorflow、ncnn、MindSpore、PaddlePaddle、TNN、opencv)，无需配置，直接使用；
*   3、海量的 AI 案例，人脸识别、人脸关键点识别、肢体识别、手势识别、头发识别、物体分类、物体跟踪、3D 检测 -、身体交换、换脸、人体抠图等。
*   4、内置创新性的 CPU+GPU+NPU 智能加速技术，通过 “硬件 + 框架 + Op" 多层优化，赋予深度学习运算性能的大幅度提升。并且提供统一 API 接口，在方便开发者调用的同时，还支持不同 AI 框架模型自动转换；
*   5、支持多种开发语言：C/C++，Python，Java，JavaScript，Ruby，PHP，Go，Shell 等；
*   6、支持多种开发工具：AidCode，Wizard，VSCode，Jupyter notebook，pycharm，积木编程 (青少年)；
*   7、扩展性好：内置了极简的外设极速互连模块，通过 USB 和网络等方式控制 Arduino、机械臂、高清摄像机、深度相机等；
*   8、丰富的 Linux 软件：Git，MySql，Hadoop，Nginx，Apache，Vim，SSH，ROS，PCL 点云，Eigen，Home Assistant 和 g2o 等多种工具；

### 极致安卓之 --- Termux 安装完整版 Linux

> _Termux 并非完整版 Linux，而是一个模拟环境，如果想基于 Termux 安装完整版 Linux，比如 Ubuntu、Debian、Kali 等，请参考：_ ：[https://zhuanlan.zhihu.com/p/95865982](https://zhuanlan.zhihu.com/p/95865982 "https://zhuanlan.zhihu.com/p/95865982")

安装基础件 proot-distro：**pkg install proot-distro**   
或者 **apt install proot-distro**  
查看 proot-distro 的使用帮助为：**proot-distro help**

**proot-distro list**     查看可以安装的 Linux 系统。

安装以上系统就简单了： proot-distro install <alias> 

比如，我要安装 ubuntu 20.04，指令为： **proot-distro install ubuntu-20.04** 

安装完成后，进入 Linux 发行版环境的指令为，比如安装的 ubuntu 为：**proot-distro login ubuntu-20.04**

每次进入 ubuntu 的命令太长，可以在 Termux 环境新建一个 sh 文件，比如新建 u20.sh：**vim u20.sh**

输入如下内容 (就是 esc 键 + i 键)：**proot-distro login ubuntu-20.04**

然后退出 (esc 键 +: 键，再输入 wq，回车)，最后，在终端输入 **./u20.sh** 就进入了真正的 linux 环境了。之后，传统操作比如换源，安装软件等等，一条龙走起来吧。输入 exit 可以退出登录的 linux 系统。

以上就是官方版的纯种 Linux 安装全过程。只要是国内源亲测安装没有 bug，非常顺畅。

装完之后，现在开始安装 python 环境

```
pkg install mariadb    安装 mariadb
mysql_install_db       安装基本数据
mysqld                 启动 mariadb 服务
```

### 极致安卓 --- Termux/Aid Learning 安装宇宙最强 VS Code

：[https://zhuanlan.zhihu.com/p/106593146](https://zhuanlan.zhihu.com/p/106593146 "https://zhuanlan.zhihu.com/p/106593146")

### 把安卓手机性能发挥到极致之 --- Termux/Aid Learning 使用 Fortran

：[https://zhuanlan.zhihu.com/p/92280533](https://zhuanlan.zhihu.com/p/92280533 "https://zhuanlan.zhihu.com/p/92280533")

Termux 运行 gcc、gfortran

### proot 介绍

wiki：[https://wiki.termux.com/wiki/PRoot#Installing_Linux_distributions](https://wiki.termux.com/wiki/PRoot#Installing_Linux_distributions "https://wiki.termux.com/wiki/PRoot#Installing_Linux_distributions")

**手机已经 root**
-------------

安装 tsu，这是一个 su 的 termux 版本，用来在 termux 上替代 su`：`**pkg install tsu**

然后终端下面输入：**tsu**  即可切换 root 用户，这个时候会弹出 root 授权提示，给予其 root 权限即可。

![](https://img-blog.csdnimg.cn/img_convert/dd1f3ec0c0b48f87e2bf75cf934bc549.png)

 在管理员身份下输入 exit 可回到普通用户身份。

8、 安装 python 和 必要模块
===================

安装 Python2 和 Python3
--------------------

*   安装 python2.7：**pkg install python2**  安装完成后，使用 **python2** 命令启动
*   安装 python3：**pkg install python** 安装完成后，使用 **python** 命令启动。 **注意：pkg install python** **安装的是最新版的 Python**

升级 pip 版本
---------

```
mysql                        直接进入 mariadb 数据库
exit                         退出数据库
mysql_secure_installation    修改密码命令, 进行密码相关的安全设置
 
下面根据个人偏好来进行设置,没有绝对的要求
Remove anonymous users? [Y/n] Y                #是否移除匿名用户
Disallow root login remotely? [Y/n] n          #是否不允许root远程登录
Remove test database and access to it? [Y/n] n #是否移除test数据库
Reload privilege tables now? [Y/n] y           #是否重新加载表的权限
 
使用密码登录数据库
$ mysql -uroot -p
Enter password:****
```

安装 ipython
----------

ipython 是一个 python 的交互式 shell，支持变量自动补全，自动缩进，支持 bash shell 命令，内置了许多很有用的功能和函数。学习 ipython 将会让我们以一种更高的效率来使用 python。  
先安装 clang，否则直接使用 pip 安装 ipython 会失败报错.

```
mkdir www
vim www/index.php
tree www/
```

然后分别使用 ipython 和 ipython2 进入 py2 和 py3 控制台:

编辑器
---

终端下有 vim 神器，并且官方也已经封装了 vim-python，对 vim 进行了 Python 相关的优化。

```
termux-chroot
vim /etc/php-fpm.d/www.conf
```

**解决 termux 下的 vim 汉字乱码**

在家目录下新建 **.vimrc** 文件：**vim .vimrc**

添加内容如下：

```
worker_processes  1;
events {
    worker_connections  1024;
}
 
 
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
 
 
    server {
 
        listen       8080;
        server_name  localhost;
        root   /data/data/com.termux/files/usr/share/nginx/html;
        index  index.html index.htm;
 
 
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   /data/data/com.termux/files/usr/share/nginx/html;
        }
 
 
        location ~ \.php$ {
            root           html;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  /usr/share/nginx/html$fastcgi_script_name;
            include        fastcgi_params;
        }
    }
 
}
```

然后 source 下变量：**source .vimrc**

效果图：

![](https://img-blog.csdnimg.cn/img_convert/9457ca030b0149ffb573c8dbb56b548e.png)

创建 Python 虚拟环境
--------------

有些第三方模块会依赖 python-dev，所以也可以装上

```
root   /data/data/com.termux/files/usr/share/nginx/html;
fastcgi_param  SCRIPT_FILENAME  /usr/share/nginx/html$fastcgi_script_name;
```

也可以创建一个 Python 虚拟环境。这是 Python 开发者的常见做法，它有助于让你的 Python 项目独立于你的开发系统 (在本例中是你的手机)。在你的虚拟环境中，你将能够安装特定于你应用的 Python 模块。

```
php-fpm
nginx
```

** 你的新虚拟环境 (**注意，开头的两个点用空格隔开**)

```
wget https://cn.wordpress.org/wordpress-4.9.4-zh_CN.zip
pkg install unzip
unzip wordpress-4.9.4-zh_CN.zip
```

请注意你的 shell 提示符现在以 (env) 开头，表示你在虚拟环境中。现在使用 pip 安装 Flask Python 模块。

```
cd wordpress
php -S 127.0.0.1:8080
```

 安装 科学计算包：jupyter、numpy、matplotlib、pandas
-----------------------------------------

方法来自这篇文章：[Running Jupyter and the Scipy stack on Android](https://link.jianshu.com?t=http%3A%2F%2Fwww.leouieda.com%2Fblog%2Fscipy-on-android.html "Running Jupyter and the Scipy stack on Android")

安装这四个包，首先安装下依赖。

```
root   /data/data/com.termux/files/home/wordpress;
        index  index.html index.htm index.php;
fastcgi_param  SCRIPT_FILENAME  /data/data/com.termux/files/home/wordpress$fastcgi_script_name;
```

这四个包安装方法类似，不过实际中安装时很多人会踩坑，其中 jupyter 是最容易安装的，一般没问题。安好了就可以拥有 ipython 和 jupyter notebook 两大神器了。

```
php-fpm
nginx
```

这里 pip 之前加了东西，看到网上说是链接到数学库编译的意思。剩下三个其中 numpy 是基础包，是 pandas 和 matplotlib 的依赖包，方法类似。

```
mkdir hexoblog  # 手动创建一个目录
cd hexoblog
hexo init       # 初始化hexo环境
hexo g          # 生成静态文件
hexo s          # 启动hexo
```

**ipython 和 matplotlib**

用 ipython 写代码可以使用各种魔法操作，termux 里的界面看着也很漂亮，

termux 下 ipython 界面如下图所示

![](https://img-blog.csdnimg.cn/img_convert/ef95f96502521f045cecce3161e1b743.png)

 不过可以看到里面 ```import matplotlib.pyplot``` 报错了，主要是缺后端绘图界面支持。命令行作图确实也不大现实。但还是可以在手机上使用 matplotlib 的，毕竟还有 jupyter notebook 嘛。

在 jupyter notebook 作图如下：

![](https://img-blog.csdnimg.cn/img_convert/ef8c762dd1a83f7e06863569cd133786.png)

安装 numpy，matplotlib 可能遇到的问题

numpy，matplotlib 可能在安装时可能会有问题，这个跟各个模块的版本号有关系。

安装报错不妨多试几个版本。就是在最后加上版本号信息诸如 ``pip install numpy==1.12.1```  ```pip install matplotlib==1.2.0```

当然如果你还要一起安装下面两个模块的话，还可以有别的解决方法。

**安装 scipy 和 scikit-learn**

这里安装后一个 scikit-learn 需要先安装 scipy。安装时要用到 gcc 来编译，不过从某版本开始 termux 官方版把 gcc 去除了。

参照官方 wiki 和 Github 的大致安装方法如下：

1. 安装 curl：**pkg install curl**

2. 命令行输入以下命令：$ **curl -L https://its-pointless.github.io/setup-pointless-repo.sh | sh** 安装一个叫 gnupg 的东西，链接到了 termux 社区一位贡献者 (its-pointless) 编译的源中，其中把 numpy 和 scipy 直接编译好都包括进去了。只需要  **pkg install numpy, scipy** 即可。

Github 里原话是这样的

This script essentially installs gnupg on your device and downloads and adds a public key to your apt keychain ensuring that all subsequent downloads are from the same source.

3. 上面已经说了，就是 ```pkg install numpy, scipy```

4. 最后安装 scikit-learn 就很简单了，直接 ```pip install scikit-learn``` 就行。

假如前面那个方法按照 numpy 报错，可以采用该方法

安装 爬虫模块 (requests，BeautifulSoup4，lxml，scrapy)
---------------------------------------------

常见的几个比如 requests，BeautifulSoup4，lxml，scrapy。

```
git clone https://github.com/ziahamza/webui-aria2.git
cd webui-aria2
node node-server.js
```

前两个很简单，直接 pip 安装就行。后两个有一些依赖，而且安装 scrapy 前必须要先装 lxml。

**lxml**

```
cd ~
curl https://getcaddy.com | bash -s personal http.filemanager
```

**scrapy**

```
cd ~
vim Caddyfile
```

**安装 nodejs**
-------------

命令：**pkg install nodejs**

安装比较方便，但是在安装的时候报错了

```
:8080 {
filemanager / /sdcard
timeouts none
gzip
}
```

查了下是这边版本的问题

![](https://img-blog.csdnimg.cn/img_convert/222973f162250002f90711e3760c1a99.png)

 官方的解决方法如下 [disable concurrency in case of libuv/libuv#1459](https://github.com/rvagg/node-worker-farm/commit/0b2349c6c7ed5c51e234e418fad226875313e773 "disable concurrency in case of libuv/libuv#1459")

![](https://img-blog.csdnimg.cn/img_convert/9cf2f765120dd6430358403accac4e32.png)

解决 npm 安装报错
-----------

```
{
  "health": "GOOD",
  "percentage": 67,
  "plugged": "UNPLUGGED",
  "status": "DISCHARGING",
  "temperature": 24.600000381469727
}
```

我这里修改 length 的是`4，`这个好像和 CPU 有关，总之这里的 length 得指定一个数字.

 ![](https://img-blog.csdnimg.cn/img_convert/3106c6552b9ad936eda5db787415389d.png)

 然后在重新安装下 **npm install hexo-cli -g** 成功.

**安装 MariaDB (MySQL)**
----------------------

MariaDB 数据库管理系统是 MySQL 的一个分支，主要由开源社区在维护，采用 GPL 授权许可。开发这个分支的原因之一是：甲骨文公司收购了 MySQL 后，有将 MySQL 闭源的潜在风险，因此社区采用分支的方式来避开这个风险。

```
pkg install nyancat
nyancat
```

 启动完成后，这个会话就一直存活，类似与 debug 调试一样，只有新建会话才可以操作。

![](https://img-blog.csdnimg.cn/img_convert/6e95505f2c780535698afa0629c89efb.png)

关于隐藏会话可以使用 nohup 命令和 tmux 命令，这里我建议使用 tmux 命令

设置 **MariaDB 密码**
-----------------

 新建 termux 会话，由于 mariadb 安装的时候没有设置密码，所以当前的 mariadb 密码为空。

![](https://img-blog.csdnimg.cn/img_convert/c8bc09f8aedfe3efa65b06923789a6f4.png)

```
npm install mapscii -g
mapscii
```

![](https://img-blog.csdnimg.cn/img_convert/d3f84260d211f3216672049cda8f80f4.png)

安装 tmux
-------

Tmux 是一个优秀的终端复用软件，类似 GNU Screen，但来自于 OpenBSD，采用 BSD 授权。一旦你熟悉了 tmux 后， 它就像一个加速器一样加速你的工作效率。

 安装 tmux 的命令：**pkg install tmux**

新建 mysql 会话
-----------

上面介绍的 mysqld 后会一直卡在那里，强迫症表示接受不了，重启手机，现在尝试使用 tmux 来管理会话。

```
(env)$ cat << EOF >> hello_world.py
> from flask import Flask
> app = Flask(__name__)
>
> @app.route('/')
> def hello_world():
>     return 'Hello, World!'
> EOF
(env)$
```

可以看到最下面的提示，表明现在是在 mysql 的会话下面操作

![](https://img-blog.csdnimg.cn/img_convert/4d889fff51559eae262056a342129458.png)

启动 mysqld 并断开会话
---------------

**启动 mysqld**

```
(env) $ export FLASK_APP=hello_world.py
(env) $ export FLASK_ENV=development
(evn) $ python hello_world.py
```

**让会话后台运行**  
使用快捷键组合 `Ctrl`+`b` + `d`，三次按键就可以断开当前会话。

使用 mysql
--------

现在那个`mysqld`会话被放在后台运行了, 整个界面看上去很简介, 使用

```
(env) $ export FLASK_ENV=””
(env) $ flask run –host=0.0.0.0
```

可以优雅的使用数据库了。效果图

 ![](https://img-blog.csdnimg.cn/img_convert/d80818a1971c5476905df96e4577352a.gif)

关于 tmux 更多进阶的用法这里不在过多介绍了。

安装 PHP、**nginx**
----------------

termux 封装的 php 版本是 php 7.2.5

安装 nginx 包：**pkg install nginx**

安装命令：**pkg install php** ，查看 php 版本

![](https://img-blog.csdnimg.cn/img_convert/efa85ac4b2f2195f8c1a960b03e7cb00.png)

自 PHP5.4 之后 PHP 内置了一个 Web 服务器。

### **编写测试文件**

在家目录下建一个 www 文件夹：mkdir www  
在 www 文件夹下新建一个 index.php 文件，其内容为：**<?php phpinfo();?>**

具体操作如下：

```
cd ～
pkg install wget
wget https://Auxilus.github.io/metasploit.sh
bash metasploit.sh
```

### **启动 WebServer**

```
git clone https://github.com/sqlmapproject/sqlmap.git
cd sqlmap
python2 sqlmap.py
```

### **启动 nginx**

Nginx 是一个高性能的 Web 和反向代理服务器, 它具有有很多非常优越的特性.

默认的普通权限无法启动 nginx，需要模拟 root 权限才可以，切换 root 用户 (切换命令：**termux-chroot** ) 进入模拟的 root 环境，尝试能不能解析默认的 index.html 主页，这个文件在 termux 上的默认位置为 /data/data/com.termux/files/usr/share/nginx/html/index.html

在模拟的 root 环境下启动 nginx：**nginx**

termux 上 nginx 默认的端口是 8080，查看下 8080 端口是否在运行：**netstat -an |grep 8080**

![](https://img-blog.csdnimg.cn/img_convert/c0f4eae44be2aba6d5235c158d6422e6.png)

 然后手机本地直接访问：[http://127.0.0.1:8080](http://127.0.0.1:8080 "http://127.0.0.1:8080")

![](https://img-blog.csdnimg.cn/img_convert/1041503b66111674f4055d980effb0bd.png)

这样一个默认的 nginx 服务就起来了，但是意义不大，得配置一下可以解析 php 才会有更大的意义.

停止 nginx 服务
-----------

这里是直接杀掉占用端口的进程，具体端口以实际情况为准.

```
pip2 install requests
git clone https://github.com/reverse-shell/routersploit
cd routersploit
python2 rsf.py
```

重启 nginx 服务
-----------

```
git clone https://github.com/gkbrk/slowloris.git
cd slowloris
chmod +x slowloris.py
```

nginx 解析 PHP
------------

成功解析的话，下面安装 wordpress 等 cms 就会轻松很多。  
nginx 本身不能处理 PHP，它只是个 web 服务器，当接收到 php 请求后发给 php 解释器处理，nginx 一般是把请求发 fastcgi 管理进程处理，PHP-FPM 是一个 PHP FastCGI 管理器，所以这里得先安装 php-fpm。

> 这里默已经安装了 nginx 和 php，没有安装的话，使用 **pkg install php nginx** 来进行安装，参考上面部分进行配置

### 安装并配置 php-fpm

**安装 php-fpm**

```
pkg install php
git clone https://github.com/Tuhinshubhra/RED_HAWK.git
cd RED_HAWK
php rhawk.php
```

**配置 php-fpm**  
进入`proot`环境, 然后编辑配置文件`www.conf`(先进 proot 可以更方便操作编写相关配置文件)

```
git clone https://github.com/Mebus/cupp.git
cd cupp
python2 cupp.py
```

定位搜索 listen 找到

```
git clone https://github.com/UltimateHackers/Hash-Buster.git
cd Hash-Buster
python2 hash.py
```

将其改为

```
git clone https://github.com/shawarkhanethicalhacker/D-TECT.git
cd D-TECT
python2 d-tect.py
```

### 配置 nginx

在`proot`环境下, 然后编辑配置文件`nginx.conf`

```
git clone https://github.com/m4ll0k/WPSeku.git
cd WPSeku
pip3 install -r requirements.txt
python3 wpseku.py
```

下面给出已经配置好的模板文件, 直接编辑替换整个文件即可:

```
git clone https://github.com/UltimateHackers/XSStrike.git
cd XSStrike
pip2 install -r requirements.txt
python2 xsstrike
```

里面的网站默认路径就是`nginx`默认的网站根目录:

```
root   /data/data/com.termux/files/usr/share/nginx/html;
fastcgi_param  SCRIPT_FILENAME  /usr/share/nginx/html$fastcgi_script_name;
```

要修改网站默认路径的话, 只需要修改这两处即可.

### 建立 php 测试文件

在 /usr/share/nginx/html 目录下新建一个 phpinfo.php 文件，其内容是：**<?php phpinfo();?>**

![](https://img-blog.csdnimg.cn/img_convert/4248e665b219d67efe7cc21481308f99.png)

### 启动 php-fpm 和 nginx

在`proot`环境下面分别启动`php-fpm`和`nginx`, 这里的`nginx`不在`proot`环境下启动后会出一些问题, 感兴趣的可以自己去研究看看.

```
php-fpm
nginx
```

### 浏览器访问测试

浏览器访问 `http://127.0.0.1:8080/phpinfo.php` 查询`php`文件是否解析了. ![](https://img-blog.csdnimg.cn/img_convert/3152f3dc9969b4cdad80a54ba619f141.png)

**搭建 WordPress**
----------------

这里只是用 wordpress 做个典型案例来讲解，类似地可以安装 Discuz、DeDecms 等国内主流的 PHP 应用程序。

### 方法一：使用 PHP 内置的 Web Server

确保安装并配置了 php 和 mariadb，没有安装好的话，参考本文中具体细节部分来进行安装。  
新建数据库

```
mysql -uroot -p*** -e"create database wordpress;show databases;"
```

*** 这里是 mysql 的密码

**下载解压 wordpress**

```
wget https://cn.wordpress.org/wordpress-4.9.4-zh_CN.zip
pkg install unzip
unzip wordpress-4.9.4-zh_CN.zip
```

**启动 PHP Web Server**

到解压后的 wordpress 目录下，执行

```
cd wordpress
php -S 127.0.0.1:8080
```

然后浏览器访问 127.0.0.1:8080 开始进行 wordperss 的安装。

![](https://img-blog.csdnimg.cn/img_convert/ef13eff29932a35feb12ccb6a3738851.png)

### **方法二：nginx + PHP + Mariadb**

上面使用的方法一是直接使用 PHP 自带的 PHP Web Server 来运行的，看上去不够严谨~，所以这里用 nginx 来部署 wordpress。确保已经安装 PHP、php-fpm、mariadb，配置可以参考上面说明。这里主要介绍使用 nginx 去解析 wordpress 源文件。 当前解压后 wordpress 的绝对路径是:

```
/data/data/com.termux/files/home/wordpress
```

**编辑 nginx.conf**

```
vim /etc/nginx/nginx.conf
```

修改为如下几处：

```
root   /data/data/com.termux/files/home/wordpress;
        index  index.html index.htm index.php;
fastcgi_param  SCRIPT_FILENAME  /data/data/com.termux/files/home/wordpress$fastcgi_script_name;
```

![](https://img-blog.csdnimg.cn/img_convert/a0d4536cb0489667fc0a8a8c347272bf.jpeg)

**启动 php-fpm 和 nginx**

在 proot 环境下面分别启动 php-fpm 和 nginx，这里的 nginx 不在 proot 环境下启动后会出一些问题，感兴趣的可以自己去研究看看。

```
php-fpm
nginx
```

**安装 wordpress**

浏览器访问：http://127.0.0.1:8080/wp-admin/setup-config.php 进行安装.

![](https://img-blog.csdnimg.cn/img_convert/0dce52026fc4918d7df9f85b240704cf.png)

 同理安装其他博客也就轻而易举了, 可玩性大大增加~

**搭建 hexo 博客**
--------------

没错还能搭建 Hexo，但是我的 hexo 是用的电脑。但是这并不代表手机就不能玩了，你要是觉得不方便，还可以用电脑来控制。

### **安装 hexo**

```
npm install hexo-cli -g
```

### **部署 hexo 博客环境**

然后建立一个目录，然后到这个目录下初始化 hexo 环境

```
mkdir hexoblog  # 手动创建一个目录
cd hexoblog
hexo init       # 初始化hexo环境
hexo g          # 生成静态文件
hexo s          # 启动hexo
```

![](https://img-blog.csdnimg.cn/img_convert/1847b9b6f8bc8f2727e2663c75246acd.png)

 然后就跑起来一个最基本的 hexo 博客。关于 hexo 博客的详细教程，建议搭建去参考 hexo 官方文档。

![](https://img-blog.csdnimg.cn/img_convert/14f24c0d8bf0a186337c6aeb6ad27272.png)

termux ssh 连接电脑
---------------

有时候要操作电脑，这个时候有了 termux，躺在床上就可以操作电脑了，岂不是美滋滋~~  
安装 openssh

```
pkg install openssh
```

然后就可以直接 ssh 连接你的电脑了

> 前提是电脑安装了 ssh 服务

```
$ ssh sqlsec@192.168.1.8
```

手机连接操作电脑效果图:

![](https://img-blog.csdnimg.cn/img_convert/d064358f58a81d3ff1a441879c85f355.png)

电脑 ssh 连接 Termux
----------------

emmm 这个需求比较鸡肋，但是写文字嘛就得写全了~

**安装 openssh**

同样也需要`openssh`才可以

```
pkg install openssh
```

**启动 sshd**

安装完成后,`sshd`服务默认没有启动, 所以得手动启动下:

```
sshd
```

因为手机上面低的端口有安全限制, 所以这里的`openssh`默认的`sshd`默认的服务是`8022`端口上的.`ssh` 的用户名用`whoami`命令看下.

![](https://img-blog.csdnimg.cn/img_convert/4c7e53c7bfd42ae6297707761d28976e.png)

可以看到 `sshd` 启动后，端口才可以看到. 

**PC 端生成公钥**

`ssh`登录是 key 公钥模式登录, 首先在 PC 端生成秘钥:

```
sqlsec@ubuntu:-> ssh-keygen -t rsa
```

执行完成后，会在家目录下创建 3 个文件：`id_rsa`, `id_rsa.pub` , `known_hosts`

![](https://img-blog.csdnimg.cn/img_convert/7ccc71dfff29dfd8e4f9c0fb06d1f4e5.png)

**拷贝公钥到手机**

然后把公钥`id_rsa.pub`拷贝到手机的`data\data\com.termux\files\home\.ssh`文件夹中.

**将公钥拷贝到验证文件中**

在`Termux`下操作

```
cat id_rsa.pub > authorized_keys
```

![](https://img-blog.csdnimg.cn/img_convert/dcfb0c6f3c9ff217c8ff9c0e0f53e91b.png)

**PC 端 连接 手机 termux**

```
sqlsec@ubuntu-> ssh -p8022 u0_a119@192.168.1.3
```

**效果图** ![](https://img-blog.csdnimg.cn/img_convert/9684489c85efc41ff001f25e931e4410.png)

 pc 端连接手机 termux 真心鸡肋呀~(忍不住自己吐槽下自己)

使用 Aria2 打造自己的下载工具
------------------

Aria2 是一个轻量级多协议和多源命令行下载实用工具。它支持 HTTP / HTTPS, FTP, SFTP, bt 和 Metalink。通过内置 Aria2 可以操作 json - rpc 和 xml - rpc。配置好的话还可以高速下载百度云文件.

### 安装 aria2

```
pkg install aria2
```

### 本地启动服务

```
aria2c --enable-rpc --rpc-listen-all
```

这个`rpc`服务默认监听的是`6800`端口, 启动后方便下面的 Web 界面连接操作.

### webui-aria2

这是个 Aria2 的热门项目, 把 Aria2 封装在了 Web 平台, 操作起来更加简单便捷。

```
git clone https://github.com/ziahamza/webui-aria2.git
cd webui-aria2
node node-server.js
```

> 需要 node 来运行, 没有安装的 话使用`pkg install nodejs`来安装

使用效果图 , 速度蛮快的 , 有兴趣的可以研究如何利用`aria2`来下载百度云文件, 等你们来探索.

![](https://img-blog.csdnimg.cn/img_convert/37dc1cdc81b178caccf0df1fa2feabc2.png)

多功能文件分享
-------

官方项目地址：[https://github.com/mholt/caddy](https://github.com/mholt/caddy "https://github.com/mholt/caddy")

### 安装 caddy

官方: 到目前为止，在 Android 上运行 Caddy 有两种方式：`Termux`和`adb`, 所以那就顺便折腾一下看看吧:

```
cd ~
curl https://getcaddy.com | bash -s personal http.filemanager
```

这一步可能执行要`3`番钟左右, 耐心等待一下即可.

### 编写配置文件

```
cd ~
vim Caddyfile
```

内容如下:

```
:8080 {
filemanager / /sdcard
timeouts none
gzip
}
```

这里的`8080`端口号可以随意指定, 因为手机权限比较低, 所以一般设置`1024`以上的端口.

注意`8080`和`{`之间有一个`空格`

注意 **`/ / sdcard`** 两个斜杠之间也有一个空格

### 启动 caddy

```
caddy
```

![](https://img-blog.csdnimg.cn/img_convert/2e60a69315ec04b9f1e9a1979cec4f14.png)

### 效果

浏览器访问:`http://127.0.0.1:8080`即可, 局域网内的用户访问手机 ip 地址即可.

默认账号和密码为`admin`,`admin`.

![](https://img-blog.csdnimg.cn/img_convert/cf9187027af861a73a12c028ec7e7cc2.png)

可以在设置界面里面 `设置简体中文`, 可以修改`更新默认密码`.

可以直接查看文件, 也支持`Linux`命令搜索.

 ![](https://img-blog.csdnimg.cn/img_convert/d291e3048e387c4f9522ec20ffd09a59.png)

 ![](https://img-blog.csdnimg.cn/img_convert/041f06ef48df4303022c959ca3cdf500.png)

Termux-api
----------

Termux:API，用于访问手机硬件, 实现更多的可玩性, 可以实现如下等功能:

*   访问电池信息
*   获取相机设备信息
*   获取本机设备信息
*   获取设置剪贴板信息
*   获取通讯录信息
*   获取设置手机短信
*   拨打号码
*   振动设备

### 安装 Termux-api

Termux-api Google Play 下载地址：[https://play.google.com/store/apps/details?id=com.termux.api](https://play.google.com/store/apps/details?id=com.termux.api "https://play.google.com/store/apps/details?id=com.termux.api")

![](https://img-blog.csdnimg.cn/img_convert/ea4907046de062c86fb5c2a80cc48c45.png)

如何在电脑上下载 Google play 上的应用？：[https://www.zhihu.com/question/22382577](https://www.zhihu.com/question/22382577 "https://www.zhihu.com/question/22382577")

### 安装 Termux-api 软件包

安装完`Termux-api`APP 后,`Termux`里面必须安装对应的包后才可以实现操作手机底层.

```
pkg install termux-api
```

下面只列举一些可能会用到的, 想要获取更多关于`Termux-api`的话, 那就去参考官方文档.

### 获取电池信息

```
termux-battery-status
```

可以看到电池的 - 健康状况 - 电量百分比 - 温度情况等

```
{
  "health": "GOOD",
  "percentage": 67,
  "plugged": "UNPLUGGED",
  "status": "DISCHARGING",
  "temperature": 24.600000381469727
}
```

### 获取相机信息

```
termux-camera-info
```

### 获取与设置剪贴板

**查看当前剪贴板内容**

```
termux-clipboard-get
```

**设置新的剪贴板内容**

```
termux-clipboard-set PHP是世界上最好的语言
```

**效果演示**

![](https://img-blog.csdnimg.cn/img_convert/fc982528305700d94823dc3345507680.png)

### 获取通讯录列表

```
termux-contact-list
```

![](https://img-blog.csdnimg.cn/img_convert/8b904a9fda434b1235783f8677abf93a.png)

### 查看短信内容列表

```
termux-sms-inbox
```

![](https://img-blog.csdnimg.cn/img_convert/5ae1fd232a5e7fb589f719ddad73f391.jpeg)

### 发送短信

```
termux-sms-send
```

支持同时发送多个号码，实现群发的效果，官方介绍如下:

```
termux-sms-send -n number(s)  recipient number(s) - separate multiple numbers by commas
```

**发送测试**

```
termux-sms-send -n 10001 cxll
```

 ![](https://img-blog.csdnimg.cn/img_convert/579456c6fe7c8f0f8b483fceeb8c9f5c.jpeg)

### 拨打电话

```
termux-telephony-call
```

拨打电话给`10001`中国电信, 查看下话费有没有欠费~?

```
termux-telephony-call 10001
```

 ![](https://img-blog.csdnimg.cn/img_convert/f33bae31964501e53647e08d2acdbf88.jpeg)

### WiFi 相关

**获取当前 WiFi 连接信息**

```
termux-wifi-connectioninfo
```

**获取最近一次 WiFi 扫描信息**

```
termux-wifi-scaninfo
```

![](https://img-blog.csdnimg.cn/img_convert/a282732f868cf9c37ec9a3a1a727be60.jpeg)

直接操作调动系统底层的话，可以通过编程来实现自动定时短信发送，语音播报等 DIY 空间无线

一些无聊的尝试
-------

一些无聊有趣的版块, 如果你是一个正经讲究人, 可以跳过这个板块以节约你的阅读时间.

### nyancat 彩虹猫

**彩虹貓** (英语：**Nyan Cat**) 是在 2011 年 4 月上传在 Youtube 的视频，并且迅速爆红于网络，並在 2011 年 YouTube 浏览量最高的视频中排名第五.

```
pkg install nyancat
nyancat
```

![](https://img-blog.csdnimg.cn/img_convert/5907a73154e8bd4d1a0b43013f9eaa16.jpeg)

 什么鬼~ 完全 Get 不到国外人的趣味点~

### 终端二维码

Linux 命令行下的二维码，主要核心是这个网址：[http://qrenco.de/](http://qrenco.de/ "http://qrenco.de/")

```
echo "http://www.sqlsec.com" |curl -F-=\<- qrenco.de
```

![](https://img-blog.csdnimg.cn/img_convert/c85dc145b4f298d4e706db607a0385cc.png)

如果你不嫌无聊的话还可以扫描这个二维码, 然后就打开我的博客了.

### 终端地图

一个基于`nodejs`编写的命令行下的地图.

```
npm install mapscii -g
mapscii
```

进入终端地图 ![](https://img-blog.csdnimg.cn/img_convert/dda387aa63d7e7320f8af5311e30a03b.png)

 **操作方法**

*   方向键 移动
*   `a`和`z`键 放大缩小
*   `q`键 退出

终端下的地图! 讲究人~ 如果你足够无聊的话, 还可以尝试能不能在这个地图上找到自己所在的位置.

安装 Linux
--------

甚至还可以在 Termux 里面在安装其他的 Linux 发行版.

由于本文篇幅已经过长了，这里不在叙述了，感兴趣，能折腾的自己去找一些资料。下面列出目前网友们用 `Termux` 可以成功安装的发行版:

*   Ubuntu
*   Arch
*   Fedora
*   Kali Nethunter

**Ubuntu**

![](https://img-blog.csdnimg.cn/img_convert/01297918eda73d4a8093044cbd98cafb.png)

 **Fedora**

![](https://img-blog.csdnimg.cn/img_convert/8fe354e2aa7bc42393bf0a4cab2d4e46.png)

### **安装步骤**

*   1. 下载安装脚本：**wget http://funs.ml/file/atilo**
*   2. 设置执行权限：**chmod +x atilo**
*   3. 运行 atilo：**./atilo**

![](https://img-blog.csdnimg.cn/img_convert/5f7739b730c884b95f229e6435601ad3.jpeg)

 通过它告诉我们的用法，我们就可以来安装了，注意流量哦，记得用 WiFi，土豪随意。

4. 比如安装 Arch 试试

```
./atilo arch
```

然后稍等一会儿，安装完成之后会提示你通过 startarch 指令启动：

```
startarch
```

5. 如果你不想要了，也可以删除

```
./atilo -r arch
```

**内网穿透**
--------

使用 ngrok 或者 frp 可以将 Termux 上面搭建的网站映射到外网上去，手机建站也不是不可能了。

关键字： frp 内网穿透

关键字： frp 内网穿透 安卓 手机

Python Jupyter Notebook
-----------------------

Jupyter notebook(又称 IPython notebook)，支持运行超过 40 种编程语言。Python 的一个强大的模块，成功安装的话可以实现比 caddy 的效果，支持 web 下的终端操作，支持代码高亮运行。由于这里需要安装大量文件，加上用户需求比较少，这一块感兴趣的话可以自己去探索。

![](https://img-blog.csdnimg.cn/img_convert/72521d47134a63e8990cb80ca57309c4.png)

下载工具
----

### you-get

是一款命令行工具，用来下载网页中的视频、音频、图片，支持众多网站，包含 41 家国内主流视频、音乐网站，如 网易云音乐、AB 站、百度贴吧、斗鱼、熊猫、爱奇艺、凤凰视频、酷狗音乐、乐视、荔枝 FM、秒拍、腾讯视频、优酷土豆、央视网、芒果 TV 等等，只需一个命令就能直接下载视频、音频以及图片回来，并且可以自动合并视频。而对于有弹幕的网站，比如 B 站，还可以将弹幕下载回来

### BaiduPCS-Go

仿 Linux shell 文件处理命令的百度网盘命令行客户端。

项目地址：[https://github.com/iikira/BaiduPCS-Go](https://github.com/iikira/BaiduPCS-Go "https://github.com/iikira/BaiduPCS-Go")

可以完美在 Termux 上运行。

 ![](https://img-blog.csdnimg.cn/img_convert/cde40ce969112e626a44dd192a9eddb2.png)

 相对来说 国外的 Termux DIY 的氛围比国内好很多，Youtube 上的视频都有很高的播放量。

![](https://img-blog.csdnimg.cn/img_convert/f30678302431b86640ed6db8d6986d9f.png)

9、termux / Tasker 联合使用
======================

**termux-taske**r：[https://github.com/termux/termux-tasker](https://github.com/termux/termux-tasker "https://github.com/termux/termux-tasker")

安装 termux-task.apk 。具体使用方法：

1. Tasker 任务里添加插件 > termux:task，然后添加用 termux 编写的脚本了。

2. 脚本放置位置是有要求的，就是要放到 ```~/.termux/tasker``` 文件夹里。需要在 termux 里创建该目录 (如下代码所示)，然后放入脚本就行。

mkdir -p .termux/tasker

3. 这个跟文件系统有关系。比如 ```~/.termux```. ~ 表示 $HOME, 对于 termux 来说也就是这个路径 "/data/data/com.termux/files/home". 手机未 root 时 这个目录只有 termux 才有权限访问。

4. 实际测试时发现，termux 中的可执行程序开头必须加上声明行才可以使用，不然都是当成 sh 脚本运行的。比如对于 python 文件，开头要加上一行：#!/data/data/com.termux/files/usr/bin/python

5. python 程序中有文件操作时，没办法直接写一个相对路径，写上绝对路径是可以的。

比如之前提到的 ```.termux/tasker``` 文件夹中的 xxx.py，

假如程序中有个写入文件 ```data/xxx.csv```，要换成下面的绝对路径：/data/data/com.termux/files/home/.termux/tasker/data/xxx.csv

如下图，为 Tasker 中添加 Termux 脚本的界面，这里添加了一个 py 脚本，选择在 termux 中运行

Tasker 添加 termux 脚本

![](https://img-blog.csdnimg.cn/img_convert/70203f562e1749504b71825ff3d28d96.png)

 下图即为脚本执行界面

![](https://img-blog.csdnimg.cn/img_convert/a49b4139343232906c6f01a584849089.png)

10、在 Android 上写 Python 代码，部署 web 服务
===================================

你已经准备好了。现在你需要为你的应用编写代码。要做到这一点，你需要有经典文本编辑器的经验。我使用的是 vi 。如果你不熟悉  vi ，请安装并试用  vimtutor ，它 (如其名称所暗示的) 可以教你如何使用这个编辑器。如果你有其他你喜欢的编辑器，如  jove 、 jed 、 joe 或  emacs ，你可以安装并使用其中一个。

现在，由于这个演示程序非常简单，你也可以直接使用 shell 的 heredoc 功能，它允许你直接在提示符中输入文本。

```
(env)$ cat << EOF >> hello_world.py
> from flask import Flask
> app = Flask(__name__)
>
> @app.route('/')
> def hello_world():
>     return 'Hello, World!'
> EOF
(env)$
```

这只有六行代码，但有了它，你可以导入 Flask，创建一个应用，并将传入流量路由到名为 `hello_world` 的函数。

![](https://img-blog.csdnimg.cn/img_convert/93c586fde6584a9808218f2459aaa60e.jpeg)​

 现在你已经准备好了网页服务器的代码。现在是时候设置一些 [环境变量](https://opensource.com/article/19/8/what-are-environment-variables "环境变量") ，并在你的手机上启动一个网页服务器了。

```
(env) $ export FLASK_APP=hello_world.py
(env) $ export FLASK_ENV=development
(evn) $ python hello_world.py
```

![](https://img-blog.csdnimg.cn/img_convert/7458588029583175c1031666569e1e2f.jpeg)​

 启动应用后，你会看到这条消息:

```
serving Flask app… running on http://127.0.0.1:5000/
```

这表明你现在在 localhost(也就是你的设备) 上运行着一个微型网页服务器。该服务器正在监听来自 5000 端口的请求。

打开你的手机浏览器并进入到 `http://localhost:5000` ，查看你的网页应用。

![](https://img-blog.csdnimg.cn/img_convert/d468c756ef28f9673fb0e06d61379fd9.jpeg)​

 你并没有损害手机的安全性。你只运行了一个本地服务器，这意味着你的手机不接受来自外部世界的请求。只有你可以访问你的 Flask 服务器。

为了让别人看到你的服务器，你可以在 `run` 命令中加入  `--host=0.0.0.0` 来禁用 Flask 的调试模式。这会打开你的手机上的端口，所以要谨慎使用。

```
(env) $ export FLASK_ENV=””
(env) $ flask run –host=0.0.0.0
```

按 `Ctrl+C` 停止服务器 (使用特殊的  `Termux` 键来作为  `Ctrl` 键)。

11、信息安全
=======

因为 Termux 完美的支持 Python 和 Perl 等语言，所以有太多优秀的信息安全工具值得大家去发现了， 这里就不一一列举了。总的来说可玩性还是比较高的。

Metasploit
----------

**安装 Ｍetasploit**

Termux 官方提供的自动话脚本安装方法如下:

```
cd ～
pkg install wget
wget https://Auxilus.github.io/metasploit.sh
bash metasploit.sh
```

注：在 x86 平台下自动化安装失败，想在 x86 平台下安装的参考　[官方的文档](https://wiki.termux.com/wiki/Metasploit_Framework "官方的文档") 手动去安装．　　

这个过程平均耗时大约 3 分钟左右 (使用国内的清华源的情况下)．　　

**配置 msf 数据库缓存**

意外发现数据库居然都配置好了，启动`msfconsole会`自动连接数据库了．　　

![](https://img-blog.csdnimg.cn/img_convert/5c9e3a76459ce78b0bdca70c5d47400f.png)

 接下来重建数据库缓存

```
msf > db_rebuild_cache
```

这个时候立刻去搜索发现缓存依然没有建立，只能使用慢速搜索，这里其实是这个缓存建立需要时间，只要稍微等待一下就可以了．

然后就可以实 现`msf` 秒搜索的效果了，无需等待，感觉比电脑上还要快呐　　

![](https://img-blog.csdnimg.cn/img_convert/4a4227037e0b10b90a5d5aca5ce55ea6.png)

### 解决 metasploit 启动后无法连接数据库

使用自动化脚本安装好 Metasploit 后使用 db_status 发现数据库是处于连接状态的，然后在使用 db_rebuild_cache 重新建立缓存，等待大约 3 分钟后，便可以使用快速搜索了，没毛病~，但是在一段日子过后，可能会出现以下情况:

![](https://img-blog.csdnimg.cn/img_convert/5f92820910b86fab8d30f961bbe9e4a3.png)

> msfconsole  
> [-] Failed to connect to the database: could not connect to server: Connection refused  
>         Is the server running on host "127.0.0.1" and accepting  
>         TCP/IP connections on port 5432?

报这个错误是因为 postgresql 数据库没有启动造成的。解决方法就是启动数据库:

本方法只针对 termux 上使用自动化脚本安装 msf

```
pg_ctl -D $PREFIX/var/lib/postgresql start
```

启动数据库后重新进入 msfconsole 会发现启动没有报错了， db_status 查看下数据库连接，也正常了:

![](https://img-blog.csdnimg.cn/img_convert/bf28ae18f1d8e8f37c0180adc50321cf.png)

Nmap
----

端口扫描必备工具

安装命令：**pkg install nmap**

![](https://img-blog.csdnimg.cn/img_convert/5f8cea5f717e5ea7051eec0152dddbcd.png)

hydra
-----

Hydra 是著名的黑客组织 THC 的一款开源暴力破解工具这是一个验证性质的工具，主要目的是：展示安全研究人员从远程获取一个系统认证权限。

 安装命令：**pkg install hydra**

![](https://img-blog.csdnimg.cn/img_convert/697d0c2b79743bb969625ebcf03f68d6.png)

sslscan
-------

SSLscan 主要探测基于 ssl 的服务，如 https。SSLscan 是一款探测目标服务器所支持的 SSL 加密算法工具。SSlscan 的代码托管在 [https://github.com/DinoTools/sslscan](https://github.com/DinoTools/sslscan " https://github.com/DinoTools/sslscan")

安装命令：**pkg install sslscan**

![](https://img-blog.csdnimg.cn/img_convert/d942c4b289f4da0f1f656d7e7839cf9a.png)

whatportis
----------

whatportis 是一款可以通过服务查询默认端口，或者是通过端口查询默认服务的工具，简单易用。在渗透测试过程中，如果需要查询某个端口绑定什么服务器，或者某个应用绑定的默认端口，可以使用 whatportis 查询。

 安装命令：**pip2 install whatportis**

![](https://img-blog.csdnimg.cn/img_convert/290553d0723b0fecc1280f6948448efa.png)

SQLmap
------

SQLmap 是一款用来检测与利用 SQL 注入漏洞的免费开源工具，

官方项目地址： [https://github.com/sqlmapproject/sqlmap](https://github.com/sqlmapproject/sqlmap "https://github.com/sqlmapproject/sqlmap")

直接 git clone 源码

```
git clone https://github.com/sqlmapproject/sqlmap.git
cd sqlmap
python2 sqlmap.py
```

sqlmap 支持 pip 安装了，所以建议直接 **pip install sqlmap** 来进行安装，然后终端下直接 sqlmap 就可以了，十分方便。

![](https://img-blog.csdnimg.cn/img_convert/ec29be13eb3086bf76adf51c3bfd61bf.png)

RouterSploit
------------

RouteSploit 框架是一款开源的路由器等嵌入式设备漏洞检测及利用框架。

```
pip2 install requests
git clone https://github.com/reverse-shell/routersploit
cd routersploit
python2 rsf.py
```

![](https://img-blog.csdnimg.cn/img_convert/5361d510ede2319cb3ab696f414eedec.png)

Slowloris
---------

低带宽的 DoS 工具

```
git clone https://github.com/gkbrk/slowloris.git
cd slowloris
chmod +x slowloris.py
```

![](https://img-blog.csdnimg.cn/img_convert/cc8a7372959ea8bede1c0bd894902c9f.png)

RED_HAWK
--------

一款采用 PHP 语言开发的多合一型渗透测试工具，它可以帮助我们完成信息采集、SQL 漏洞扫描和资源爬取等任务。

```
pkg install php
git clone https://github.com/Tuhinshubhra/RED_HAWK.git
cd RED_HAWK
php rhawk.php
```

![](https://img-blog.csdnimg.cn/img_convert/dfd67d3fb3acccba0558e2c4c838ca41.png)

Cupp
----

Cupp 是一款用 Python 语言写成的可交互性的字典生成脚本。尤其适合社会工程学，当你收集到目标的具体信息后，你就可以通过这个工具来智能化生成关于目标的字典。

```
git clone https://github.com/Mebus/cupp.git
cd cupp
python2 cupp.py
```

![](https://img-blog.csdnimg.cn/img_convert/c4c4e350d4d87f2700e59d1eb1d4c8ea.png)

Hash-Buster
-----------

Hash Buster 是一个用 python 编写的在线破解 Hash 的脚本，官方说 5 秒内破解，速度实际测试还不错哦~

```
git clone https://github.com/UltimateHackers/Hash-Buster.git
cd Hash-Buster
python2 hash.py
```

![](https://img-blog.csdnimg.cn/img_convert/44465f7eacc6673256d056f1e08bc79b.png)

D-TECT
------

D-TECT 是一个用 Python 编写的先进的渗透测试工具,

*   wordpress 用户名枚举
*   敏感文件检测
*   子域名爆破
*   端口扫描
*   Wordperss 扫描
*   XSS 扫描
*   SQL 注入扫描等

```
git clone https://github.com/shawarkhanethicalhacker/D-TECT.git
cd D-TECT
python2 d-tect.py
```

![](https://img-blog.csdnimg.cn/img_convert/1aed00dfac4380a07f9be4a94f4aecb7.png)

WPSeku
------

WPSeku 是一个用 Python 写的简单的 WordPress 漏洞扫描器，它可以被用来扫描本地以及远程安装的 WordPress 来找出安全问题。被评为 2017 年最受欢迎的十大开源黑客工具.

```
git clone https://github.com/m4ll0k/WPSeku.git
cd WPSeku
pip3 install -r requirements.txt
python3 wpseku.py
```

![](https://img-blog.csdnimg.cn/img_convert/b9d07a0fadf49bda735c7bf9e566d853.png)

XSStrike
--------

XSStrike 是一种先进的 XSS 检测工具。它具有强大的模糊测试引擎.

```
git clone https://github.com/UltimateHackers/XSStrike.git
cd XSStrike
pip2 install -r requirements.txt
python2 xsstrike
```

![](https://img-blog.csdnimg.cn/img_convert/2c4b662faf609206213395f547429350.png)