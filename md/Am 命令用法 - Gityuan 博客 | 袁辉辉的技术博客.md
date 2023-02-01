> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [gityuan.com](http://gityuan.com/2016/02/27/am-command/)

> 袁辉辉, Gityuan, Android 博客, Android 源码, Flutter 博客，Flutter 源码

> 基于 Android 6.0 的源码剖析， 分析 am 命令

一、概述[](#一概述)
------------

作为一名开发者，相信对 adb 指令一定不会陌生。那么在手机连接 adb 后，可通过 am 命令做很多操作：

#### 拨打电话[](#拨打电话)

通过 adb，可以直接拨打电话 10086

```
adb shell am start -a android.intent.action.CALL -d tel:10086
```

#### 打开网站[](#打开网站)

比如，打开网站 www.gityuan.com

```
adb shell am start -a android.intent.action.VIEW -d  http://gityuan.com
```

#### 启动应用[](#启动应用)

比如，启动包名为 com.gityuan.app，主 Activity 为`.MainActivity`，且 extra 数据以”website” 为 key, “gityuan.com” 为 value。通过 java 代码要完成该功能虽然不复杂，但至少需要一个 android 环境，而通过 adb 的方式，只需要在 adb 窗口，输入如下命令便可完成:

```
am start -n com.gityuan.app/.MainActivity -es website gityuan.com
```

am 命令还可以启动 Service、Broadcast，杀进程，监控等功能，这些功能都非常便捷调试程序，接下来讲述关于 am 更多更详细的功能。

二、Am 命令[](#二am命令)
-----------------

### 2.1 命令列表[](#21-命令列表)

命令格式如下：

```
am [subcommand] [options]
```

命令列表如下：

<table><thead><tr><th>命令</th><th>功能</th><th>实现方法</th></tr></thead><tbody><tr><td>am start <code>[options</code>] <code>&lt;INTENT</code>&gt;</td><td>启动 Activity</td><td>startActivityAsUser</td></tr><tr><td>am startservice <code>&lt;INTENT</code>&gt;</td><td>启动 Service</td><td>startService</td></tr><tr><td>am stopservice <code>&lt;INTENT</code>&gt;</td><td>停止 Service</td><td>stopService</td></tr><tr><td>am broadcast <code>&lt;INTENT</code>&gt;</td><td>发送广播</td><td>broadcastIntent</td></tr><tr><td>am kill <code>&lt;PACKAGE</code>&gt;</td><td>杀指定后台进程</td><td>killBackgroundProcesses</td></tr><tr><td>am kill-all</td><td>杀所有后台进程</td><td>killAllBackgroundProcesses</td></tr><tr><td>am force-stop <code>&lt;PACKAGE</code>&gt;</td><td>强杀进程</td><td>forceStopPackage</td></tr><tr><td>am hang</td><td>系统卡住</td><td>hang</td></tr><tr><td>am restart</td><td>重启</td><td>restart</td></tr><tr><td>am bug-report</td><td>创建 bugreport</td><td>requestBugReport</td></tr><tr><td>am dumpheap <code>&lt;pid</code>&gt; <code>&lt;file</code>&gt;</td><td>进程 pid 的堆信息输出到 file</td><td>dumpheap</td></tr><tr><td>am send-trim-memory <code>&lt;pid</code>&gt; <code>&lt;level</code>&gt;</td><td>收紧进程的内存</td><td>setProcessMemoryTrimLevel</td></tr><tr><td>am monitor</td><td>监控</td><td>MyActivityController.run</td></tr></tbody></table>

am 命令实的实现方式在 Am.java，最终几乎都是调用`ActivityManagerService`相应的方法来完成。

*   比如命令`am start -a android.intent.action.VIEW -d http://gityuan.com`， 启动 Acitivty 最终调用的是 ActivityManagerService 类的 startActivityAsUser() 方法来完成的。
*   再比如 `am kill-all`命令，最终的实现工作是由 ActivityManagerService 的 killBackgroundProcesses() 方法完成的。

接下来，说说`[options`] 和 `<INTENT`> 参数含义和使用。

### 2.2 Activity 启动命令[](#22-activity启动命令)

先来介绍一下启动 Activity 命令`am start [options] <INTENT>`使用 options 参数，接下来列举 Activity 命令的 [options] 参数：

*   -D: 允许调试功能
*   -W: 等待 app 启动完成
*   -R `<COUNT`>: 重复启动 Activity COUNT 次
*   -S: 启动 activity 之前，先调用 forceStopPackage() 方法强制停止 app.
*   –opengl-trace: 运行获取 OpenGL 函数的 trace
*   –user `<USER_ID`> `|` current: 指定用户来运行 App, 默认为当前用户。
*   –start-profiler `<FILE`>: 启动 profiler，并将结果发送到 `<FILE`>;
*   -P `<FILE`>: 类似 –start-profiler，不同的是当 app 进入 idle 状态，则停止 profiling
*   –sampling INTERVAL: 设置 profiler 取样时间间隔，单位 ms;

启动 Activity 的实现原理： 存在 - W 参数则调用 startActivityAndWait() 方法来运行，否则 startActivityAsUser()。

### 2.3 trim-memory 命令[](#23-trim-memory命令)

收紧内存命令

```
am send-trim-memory  <pid> <level>
```

例如： 向 pid=12345 的进程，发出 level=RUNNING_LOW 的收紧内存命令

```
am send-trim-memory 12345 RUNNING_LOW。
```

那么 level 取值范围为： HIDDEN、RUNNING_MODERATE、BACKGROUND、RUNNING_LOW、MODERATE、RUNNING_CRITICAL、COMPLETE。

### 2.4 Intent 参数[](#24-intent参数)

Intent 的参数和 flags 较多，本文为方便起见，分为 3 种类型参数，常用参数，Extra 参数，Flags 参数。

#### 2.4.1 常用参数[](#241-常用参数)

*   `-a <ACTION>`: 指定 Intent action， 实现原理 Intent.setAction()；
*   `-n <COMPONENT>`: 指定组件名，格式为 {包名}/.{主 Activity 名}，实现原理 Intent.setComponent(）；
*   `-d <DATA_URI>`: 指定 Intent data URI
*   `-t <MIME_TYPE>`: 指定 Intent MIME Type
*   `-c <CATEGORY> [-c <CATEGORY>] ...]`: 指定 Intent category，实现原理 Intent.addCategory()
*   `-p <PACKAGE>`: 指定包名，实现原理 Intent.setPackage();
*   `-f <FLAGS>`: 添加 flags，实现原理 Intent.setFlags(int)，紧接着的参数必须是 int 型；

实例

```
am start -a android.intent.action.VIEW
am start -n com.gityuan.app/.MainActivity
am start -d content://contacts/people/1
am start -t image/png
am start -c android.intent.category.APP_CONTACTS
```

**(1). 基本类型**

<table><tbody><tr><td>参数</td><td>-e/-es</td><td>-esn</td><td>-ez</td><td>-ei</td><td>-el</td><td>-ef</td><td>-eu</td><td>-ecn</td></tr><tr><td>类型</td><td>String</td><td>(String)null</td><td>boolean</td><td>int</td><td>long</td><td>float</td><td>uri</td><td>component</td></tr></tbody></table>

比如参数 es 是 Extra String 首字母简称，实例：

```
am start -n com.gityuan.app/.MainActivity -es website gityuan.com
```

此处`-es website gityuan.com`，等价于 Intent.putExtra(“website”, “gityuan.com”);

**(2). 数组类型**

<table><tbody><tr><td>参数</td><td>-esa</td><td>-eia</td><td>-ela</td><td>-efa</td></tr><tr><td>数组类型</td><td>String[]</td><td>int[]</td><td>long[]</td><td>float[]</td></tr></tbody></table>

比如参数 eia，是 Extra int array 首字母简称，多个 value 值之间以逗号隔开，实例：

```
am start -n com.gityuan.app/.MainActivity -ela weekday 1,2,3,4,5
```

此处`-ela weekday 1,2,3,4,5`，等价于 Intent.putExtra(“weekday”, new int[]{1,2,3,4,5});

**(3). ArrayList 类型**

<table><tbody><tr><td>参数</td><td>-esal</td><td>-eial</td><td>-elal</td><td>-efal</td></tr><tr><td>List 类型</td><td>String</td><td>int</td><td>long</td><td>float</td></tr></tbody></table>

比如参数 efal，是 Extra float Array List 首字母简称，多个 value 值之间以逗号隔开，实例：

```
am start -n com.gityuan.app/.MainActivity -efal nums 1.2,2.2
```

此处`-efal nums 1.2,2.2`，等价于先构造 ArrayList 变量，再通过 putExtra 放入第二个参数。

### 2.4.3 Flags 参数[](#243-flags参数)

在参数类型 1 中，提到有`-f <FLAGS>`，是通过`Intent.setFlags(int )`方法，来设置 Intent 的 flags. 本小节也是关于 flags，是通过`Intent.addFlags(int )`方法。如下所示，所有的 flags 参数。

```
[--grant-read-uri-permission] [--grant-write-uri-permission]
[--grant-persistable-uri-permission] [--grant-prefix-uri-permission]
[--debug-log-resolution]
[--exclude-stopped-packages] [--include-stopped-packages]
[--activity-brought-to-front] [--activity-clear-top]
[--activity-clear-when-task-reset] [--activity-exclude-from-recents]
[--activity-launched-from-history] [--activity-multiple-task]
[--activity-no-animation] [--activity-no-history]
[--activity-no-user-action] [--activity-previous-is-top]
[--activity-reorder-to-front] [--activity-reset-task-if-needed]
[--activity-single-top] [--activity-clear-task]
[--activity-task-on-home]
[--receiver-registered-only] [--receiver-replace-pending]
```

例如，发送 action=”broadcast.demo” 的广播，并且对于 forceStopPackage() 的应用不允许接收该广播，命令如下：

```
am broadcast -a broadcast.demo --exclude-stopped-packages
```

* * *

**微信公众号** [**Gityuan**](http://gityuan.com/images/about-me/gityuan_weixin_logo.png) **| 微博** [**weibo.com/gityuan**](http://weibo.com/gityuan) **| 博客** [**留言区交流**](http://gityuan.com/talk/)

* * *