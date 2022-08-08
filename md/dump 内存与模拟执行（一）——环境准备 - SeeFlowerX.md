> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.seeflower.dev](https://blog.seeflower.dev/archives/165/)

> 前言我 dump 内存采用的方式是 termux + lldb 手机需要有 root 权限环境准备首先下载最新版 termuxhttps://github.com/termux/termux-app/rele...

我 dump 内存采用的方式是`termux + lldb`

**手机需要有 root 权限**

首先下载最新版 termux

*   [https://github.com/termux/termux-app/releases/tag/v0.118.0](https://github.com/termux/termux-app/releases/tag/v0.118.0)

下载安装后打开 termux，先安装几个必要的软件（普通用户下安装）

```
pkg install nano
pkg install openssl
pkg install lrzsz
pkg install lldb
```

简要说明

*   `nano` 用于编辑文件
*   `lrzsz` 用于传输文件，搭配`ttyd`会比较方便，再也不用 adb pull 了
    
    *   下载文件：`lsz file.zip`
    *   上传文件：`lrz`
*   `openssl` 是避免某些操作的时候可能出现`library "libssl.so.3" not found`的情况
*   `lldb` 当然是用来 dump 内存

![](https://blog.seeflower.dev/images/Snipaste_2022-07-30_15-28-04.png)

为了方便在 PC 端操作，需要配置 termux 的环境，PC 上进入 termux shell 环境这里推荐两个方式

*   `adb shell` + `配置环境变量`
*   `termux + ttyd` + `网页远程配置`

**adb 方式**

使用 adb 要么有线连接，要么无线连接

然后在电脑上执行下面的命令

```
adb shell
su
```

**ttyd 方式**

在 termux 中安装`ttyd`，命令如下

```
pkg install ttyd
```

安装完成后执行下面的命令开启服务端

```
ttyd bash
```

如果想调整字体大小，颜色配置等，可以参考执行下面的命令

```
ttyd -t fontSize=22 -t 'theme={"background": "#282A36", "foreground": "#F8F8F2", "cursor": "#F8F8F2", "selection": "#44475A"}' bash
```

然后就可以在 PC 上通过`http://{手机ip}:7861`访问手机的 shell 了，可以多开

然后切换到 su

**配置环境变量**

通过`adb方式`或者`ttyd方式`，现在已经可以在 PC 端方便操作了

前面说到要配置 termux 环境，主要指的是 termux 的环境变量

打开 termux 后输入`env`命令执行即可看到 termux 普通用户下的环境变量

有很多，这里简化部分，最终我们需要配置的环境命令如下：

```
export SHELL=/data/data/com.termux/files/usr/bin/bash
export COLORTERM=truecolor
export HISTCONTROL=ignoreboth
export PREFIX=/data/data/com.termux/files/usr
export TERMUX_IS_DEBUGGABLE_BUILD=1
export TERMUX_MAIN_PACKAGE_FORMAT=debian
export PWD=/data/data/com.termux/files/home
export TERMUX_VERSION=0.118.0
export EXTERNAL_STORAGE=/sdcard
export LD_PRELOAD=/data/data/com.termux/files/usr/lib/libtermux-exec.so
export HOME=/data/data/com.termux/files/home
export LANG=en_US.UTF-8
export TERMUX_APK_RELEASE=GITHUB
export TMPDIR=/data/data/com.termux/files/usr/tmp
export TERM=xterm-256color
export SHLVL=2
export PATH=/data/data/com.termux/files/usr/bin:$PATH
```

现在复制上面的命令，粘贴到你的 shell 中执行

`adb方式`

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_21-03-30.png)

`ttyd方式`

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_21-07-01.png)

至此 dump 内存的环境准备工作做好了

  
星球广告 & 交流群  
![](https://blog.seeflower.dev/images/xqyh.png)