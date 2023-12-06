> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7032937512641249316)

第 1 章 背景知识
==========

1.1 iOS 逆向工程简介
--------------

```
从iOS应用的功能或行为入手，通过脱壳、运行时分析、静态分析、动态调试、Hook、注入等一系列技术手段，推导出目标文件的结构、流程、算法、代码等，称为“iOS应用逆向工程”（后续简称为“iOS逆向工程”）。
对于从未接触过逆向的人来说，这将是一个全新的领域，充满刺激与挑战。
```

### 1.1.1 iOS 逆向工程学习路线

```
逆向工程是一个庞大的学术体系，初学者通常感觉无从下手、难度较大。
笔者根据多年的经验，将基本的学习路线加以整理，尽可能让读者少走弯路、快速进入iOS逆向之门。
下面列出的内容在这个阶段都只强调“基础入门”，读者大可不必去刻意学习高深的东西，等入门以后自然会兴趣使然。
```

```
1.逻辑推理能力
  逆向工程是技术与思维的紧密结合，大多数情况下需要根据逻辑来猜测结果，如果不具备任何逻辑推理能力，就不太适合学习逆向工程。

2.iOS开发基础
  逆向工程的层次高低取决于正向开发知识的积累，
  对于初学者，建议至少要了解Objective-C、Swift的基本特性以及Xcode的使用方法，等熟练之后，还需要了解一些底层知识，例如Mach-O文件结构、Runtime机制等。
  
3.C/C++语言基础
  iOS平台上虽然主要使用Objective-C编写应用，但是考虑到跨平台以及执行效率问题，很多底层逻辑往往都会使用C/C++编写，所以C/C++是极为重要的技术基础。
  其实不管是哪个平台的逆向，C/C++都是必须掌握的语言。
  读者在学习C/C++时，需要重点关注指针的概念和使用，以及C++虚函数及其特性。
  
4.ARM汇编基础iOS设备都使用ARM架构的CPU，所以对ARM汇编指令的了解是必不可少的。
  如今的设备都是64位，因此学习重点应该放在ARM64上，ARMv7稍微了解一下即可。
  根据笔者的经验，学习ARM汇编只需要了解常用的
    赋值、跳转、算术运算、移位运算、堆栈操作、内存读写指令和函数调用约定即可，在分析时如果发现不认识的指令，再去查阅相关的ARM手册。
```

### 1.1.2 iOS 逆向工程基本流程

```
严格来说，iOS逆向工程并没有固定的操作流程，
但是根据以往的经验，在具备一定基础的情况下，可以参照以下步骤完成。

● 应用脱壳：iOS端App在上线之前会由苹果商店进行FairPlay DRM数字版权加密保护（加壳），因此，要对应用进行分析，就必须先得到解密（脱壳）后的二进制文件。
  当然也有一些不在苹果商店上架（如重签名）的App是没有进行加壳的，可以跳过这一步。

● 去除保护：现在越来越多的App开发人员为了维护自己的关键技术不被侵犯，采用了各式各样的保护技术，如反调试、越狱检测、混淆等。要想对这类App进行逆向，首先要判断并详细分析其保护代码，在掌握它的运行机制后将其去除。

● 运行时分析：利用Cycript等工具向目标App中注入代码，从表面现象入手来获取当前的界面布局及控制器，从而定位可疑方法或函数，然后进行各种调用测试，最终定位被逆向功能的入口。

● 静态分析：可以采用class-dump先导出头文件来大致扫描一下感兴趣的类名、函数名等。找到感兴趣的函数名后，用Hopper Disassembler、IDA Pro等强大的工具查看其实现方式、交叉引用等。

● 动态调试：当静态分析无法得到想要的结果（如某函数经过混淆）的时候，就要使用动态调试技术来查看其调用栈，分析其逻辑等。

● Hook测试：通过Hook快速梳理类与类之间的调用顺序，修改函数参数、返回值等测试效果，从而找到突破口。

● 抓包分析：如果逆向的目标具备网络特征，则可以先进行抓包分析确定关键点，然后模拟或篡改数据包测试后，再结合上面的方法具体分析。

总之，iOS逆向工程的基本流程就是：
  先想尽一切办法寻找切入点（突破口），
  接着通过静态分析与动态调试相结合、跟踪和分析目标程序的核心代码、理解其设计思路等，从而还原出相似代码。
```

### 1.1.3 iOS 逆向工程意义所在

```
通常，对目标程序进行逆向是为了在没有源代码的情况下了解程序。
需要进行逆向的情况包括但不限于以下几种。
  ● 学习优秀的设计和实现方法。
  ● 分析应用存在的安全问题。
  ● 分析恶意应用的行为。
  ● 分析应用的通信协议。
iOS逆向工程可以让读者从另一层面了解程序的结构及逻辑、深入洞察程序的运行过程、分析目标程序使用的通信协议，以及学习先进的算法等。
```

1.2 iOS 越狱平台简介
--------------

```
众所周知，iOS系统的安全性相对较高，其原因之一是系统较高的封闭性，这也导致在未越狱的情况下，人们无法了解它的内部结构、运行机制等。
直到国内外越狱团队公开了越狱工具，其神秘面纱才被一层层揭开。
要想更好地研究iOS逆向工程，就必须先了解越狱平台及将自己的设备越狱。
```

### 1.2.1 iOS 越狱及其定义

```
iOS越狱是利用系统漏洞来获取设备Root权限的一种技术手段。
iOS设备越狱后可以下载并安装苹果公司不允许出现在App Store上的应用，安装各种插件等。对于逆向研究人员来说，可以更加方便地调试应用、洞察程序的执行过程等。
从越狱的完美度来划分，目前业界给出了四种定义。

1.引导式越狱当处于此状态的iOS设备重启后，之前进行的越狱程序就会失效，用户将失去Root权限，需要将设备连接电脑后再次使用越狱工具进行引导开机，才能正常使用。

2.非完美越狱与引导式越狱一样定义为“在设备重启时，之前的越狱就会失效，用户将失去Root权限”，若想再获得Root权限，则需再次进行越狱。
它们的最大区别在于：“非完美越狱”的设备在重启后至少能作为未越狱的设备正常使用。

3.完美越狱指设备在重启后依然能保持越狱状态，这也是最理想的越狱方式。
由于苹果的保护越来越强，目前“完美越狱”的版本止于iOS 9.1（使用由盘古团队发布的越狱工具），之后的都是“非完美越狱”。

4.永久越狱最高级别的越狱方式基于iOS设备的BootROM漏洞进行。
这个漏洞源于芯片，而且相应区域是只读的，不能通过软件进行写入，也就是说这个漏洞无法通过系统更新来修复。
该方式不受系统版本影响，只要越狱工具开发者稍作适配就能通用于未来的任何版本，所以称为“永久越狱”。
BootROM漏洞非常罕见，截至目前仅出现过两次，一次是早年影响A4芯片的漏洞，另一次是近期公开的影响A5～A11芯片的漏洞（checkm8）。
```

### 1.2.2 iOS 越狱商店

```
iOS越狱商店类似于苹果官方的App Store，但它的权限更高，可操作性更强，里面主要发布一些系统补丁、插件（tweaks）等。
通过安装各种插件，可以增加更多的实用功能，例如修改控制中心、增强指纹识别功能、访问系统文件及安装命令行工具等。
目前有Cydia与Sileo两款越狱商店，下面分别进行介绍。
```

```
1.Cydia
Cydia由Jay Freeman（saurik）开发，是老牌的iOS越狱商店。
在iOS 11之前，Cydia的安装也成为“已越狱”的一种明显标志。
Cydia有五个选项卡，分别是Cydia、软件源、变更、已安装、搜索，如图1-1所示。
  1）Cydia：提供了很多帮助指引，及一些精选插件等。
  2）软件源：所谓“源”实际上就是可供下载及安装应用的URL。
  这里以添加PP助手源为例：在左上角单击“添加”，在弹出的对话框中输入“http://apt.25pp.com”，然后单击“添加源”。
  3）变更：如果安装了某个源对应的软件并且软件版本有变动，将会在下面用红色提示。
  4）已安装：显示所有已安装的插件，分为用户、专业人士、最近三个类别。
  5）搜索：用来搜索需要安装插件的关键字，这是用得最多的。
```

```
2.Sileo
由于CoolStar在发布iOS 11越狱工具Electra时，Jay Freeman（saurik）没有同步对Cydia进行更新，导致纯64位设备无法正常运行Cydia，所以CoolStar采用了逆向的方式对Cydia进行改造并集成在Electra工具中，使之能兼容64位设备。

但是这种方式令Jay Freeman（saurik）非常不满，CoolStar迫不得已开发出了Sileo来尝试替代Cydia，并在新版Electra、Chimera工具中正式集成了它。

相比Cydia，Sileo的加载速度更快，整体风格和App Store极为类似，页面底部分别为精选、新内容、软件源、软件包和搜索选项。

Sileo如今已经完美支持iOS 11及以上的所有版本，若Cydia不再更新的话，终有一天会被Sileo替代，那时将会掀起一波改朝换代的浪潮。
```

### 1.2.3 iOS 系统目录

```
iOS源自macOS，而macOS又基于UNIX系统内核，因此其目录结构基本与UNIX系统相同。

实际上iOS系统包含两类目录，
一类是保留的UNIX传统目录，
另一类是iOS/macOS特有的目录，
越狱后用PP助手等工具可以对此一探究竟。
```

```
1.UNIX传统目录
/bin：“binary”的简称，存放提供用户级基础功能的二进制文件，如cat、chmod、chown等。

/boot：存放能使系统成功启动的所有文件，iOS中此目录为空。

/dev：“device”的简称，存放BSD设备文件。

/etc：“EtCetera”的简称，存放系统脚本及配置文件，如passwd、hosts等。iOS中，此目录是一个符号链接，实际指向/private/etc。

/lib：存放系统文件、内核模块及设备驱动等。iOS中此目录为空。

/mnt：“mount”的简称，存放临时的文件系统挂载点。iOS中此目录为空。

/sbin：“system binaries”的简称，提供系统级基础功能的二进制文件，如mount、reboot等。

/tmp：临时文件存放目录。在iOS中，此目录是一个符号链接，实际指向/private/var/tmp。

/usr：存放大量的工具及第三方程序。
  /usr/lib目录存放了各种dylib（动态链接库），
  /usr/include目录存放了所有的标准C头文件，
  /usr/bin目录存放着一些后期安装的用户命令，如zip、unzip等，后面章节需要用到的debugserver也将放到此目录。

/var：“variable”的简称，存放经常变化的文件，如日志文件。
   /var/mobile和/var/root分别存放了mobile用户和root用户的文件，是非常重要的目录。
```

```
2.iOS特有目录
/Applications：存放所有系统应用以及从Cyida安装的应用，但不包括从AppStore下载安装的应用。

/Developer：当设备连接到Xcode时，此目录下会存放一些开发调试相关的文件和工具。

/Library：存放一些提供系统支持的数据，其中的/Library/MobileSubstrate/DynamicLibraries是越狱开发者最感兴趣的目录，里面存放了所有基于CydiaSubstrate的插件。

/System：仅包含一个Library子目录，是iOS文件系统中最重要的目录之一，存放着大量的系统组件。
  /System/Library/CoreServices里面的SpringBoard.app是iOS桌面管理器，是用户与系统交互的重要媒介；
  /System/Library/Frameworks和/System/Library/PrivateFrameworks目录存放iOS系统中的公开及私有框架。
  /System是系统中最重要的目录，一般情况下不去轻易修改它。
  
/User：用户目录，实际指向/var/mobile。
  这个目录下存放着大量的用户数据，比如用户照片、录音、短信数据及语音等。
  
  由于篇幅所限，这里只列出一些重要的目录，感兴趣的读者可以自行研究其他目录结构。
  但是在不完全了解某个目录功能的情况下，不要随意修改文件，以免造成“白苹果”（不能进入系统），切记！
```

### 1.2.4 iOS 沙盒结构

```
沙盒也叫沙箱（Sandbox），其原理是把应用程序生成和修改的文件重定向到自身文件夹中。
在沙盒机制下，每个程序之间的文件夹不能互相访问。
为了保证系统安全，iOS采用了这种机制。
iOS应用程序在安装时会创建属于自己的沙盒文件，沙盒的根目录有三个文件夹，分别是Documents、Library、tmp。下面分别进行介绍。

Documents：用于存储应用程序的数据，该路径可通过配置来实现iTunes文件共享，可被iTunes备份。

Library：该目录下有两个子目录，其中，
  Preferences目录保存应用程序的偏好设置文件，iTunes同步设备时会备份该目录；   
  Caches目录主要存放缓存文件，用来保存应用程序运行时产生的需要持久化的数据，这些数据一般存储体积比较大，但又不是特别重要（比如网络请求数据等），需要用户负责删除，iTunes同步设备时不会备份该目录。
  另外，Library下可创建子文件夹，用来放置希望备份但不希望被用户看到的数据。该路径下的文件夹，除Caches以外，都会被iTunes备份。
 
tmp：保存应用程序运行所需的临时数据，使用完毕后再将相应的文件从该目录删除。
  应用程序没有运行时，系统也可能会清除该目录下的文件。
  iTunes同步设备时不会备份该目录。
```

### 1.2.5 iOS 应用结构

```
App是“Application”的简称，也即应用。
对于普通用户来说，只有从App Store下载App的概念，而对于安全研究人员来说，还可以从越狱商店下载App。
它们两者有一定的差别。

安装目录的差别：
  /Applications目录存放的是iOS系统自身的App以及从越狱商店安装的App（后面把来自越狱商店的App也归类为系统App）；
  App Store下载的App则存放在/var/mobile/Containers/Bundle/Application目录。

安装包的差别：
  越狱商店的App安装包格式通常为deb，
  而App Store的App安装包格式通常是ipa。

文件权限的差别：
  越狱商店的App能够以root权限运行，
  而App Store的App只能以mobile权限运行。

除了上述差别外，App安装包内部的结构几乎是一模一样的。
可以随便取一个ipa文件来研究一下其内容。

ipa文件实则是zip文件，对其右击，在弹出的快捷菜单中选择“打开方式”→“归档实用工具”，将文件内容解压出来，使用tree可以看到如下目录结构：

由此可知，一个ipa包内必然存在一个Payload文件夹，里面存放的.app文件夹才是真正的包内容。
在.app文件夹上右击，在弹出的快捷菜单中选择“显示包内容”即可看到完整内容。
下面对App安装包中的一些关键内容进行讲解。

Info.plist文件：应用的主要配置文件，包括“Bundle identifier”和“Executablefile”等参数，这两个参数也是越狱插件开发需要的重要配置信息。

可执行文件：Info.plist文件中的“Executable file”参数所对应的可执行文件。
该文件就是通常所说的目标文件，逆向工程就是针对它展开的。

*.lproj文件夹：存放各种本地化的字符串（.strings），是iOS逆向工程寻找切入点的途径之一。

资源文件：包括图片资源、配置文件、音/视频素材等。
以上就是一个App安装包最基本的组成部分，除此之外还可能存在以下内容。

Frameworks：当前应用中使用的一些第三方框架或Swift动态库。

PlugIns：当前应用的扩展文件。

Watch：如果当前应用支持Apple Watch，则会有此文件夹及对应的应用。
```

### 1.2.6 iOS 文件权限

```
iOS是一个多用户操作系统，每个用户扮演着不同的角色，对系统的控制权也各不相同，
如root用户拥有系统最高控制权，可以执行所有命令，
而mobile用户只能执行一些权限比较低的命令。

“组”是用户的一种组织方式，一个“组”可以包含多个用户，一个用户也可以归属于多个“组”。

文件所有权是iOS的一个重要组成部分，提供了一种安全的方法来存储文件。
当使用ls-l命令的时候，会将文件相关的各种权限展示出来：
  ls -l

类似“-rwxr-xr--”这样的就是文件的权限信息。
权限共由10个bit来划分，
最前面1个bit表示文件类型，如：“d”表示目录、“l”表示符号链接、“-”表示普通文件。
接下来的9个bit平均分成3组，分别如下。
 所有者权限：决定文件的所有者可以对文件执行的操作。
 所属组权限：决定属于该组的成员对他所拥有的文件能够执行的操作。
 其他人权限：表示其他人对于该文件能够进行的操作。
 
文件权限划分可用图1-7直观表示。

iOS用3个bit来描述权限信息，从高位到低位分别是r（read，读）、w（write，写）及x（execute，执行）权限。

所以从“-rwxr-xr--”中就能很清晰地解读得出：
  所有者具有读、写及执行权限；
  所属组具有读和执行权限；
  其他人仅具有读权限。
  
那么这些文件权限如何设置呢？
其实很简单，用二进制来表示即可：如果某一位为1，则这一位代表的权限生效，为0则无效。
例如，要使某个普通文件的所有者具有r、w、x权限，所属组具有r、w权限，其他人只能具备r权限，则按照图1-7来进行二进制换算拆解可得到0-111-110-100。
同时二进制的0111110100转成八进制是764，再使用chmod命令设置权限即可，具体如下。
  chmod 764 chinapyg.txt
  ls -al
  
除r、w、x权限外，文件还可以拥有SUID、SGID及sticky等特殊权限，
由于这类应用在常规逆向工程中的出现频率不高，所以不做详述，读者稍微了解一下即可。
```

1.3 本章小结
--------

```
本章第一小节科普了iOS逆向工程的概念、学习路线及基本流程，旨在引导读者正确认识iOS逆向工程；
第二小节介绍了越狱的定义、越狱商店的演变过程、系统目录、沙盒结构、应用结构及文件权限等，旨在让读者对iOS越狱领域有个初步的了解。
```

第 2 章 环境搭建
==========

2.1 开发环境
--------

```
无论是正向还是逆向开发，都少不了编写代码的环节，
一个好的开发环境能使编写代码的过程高效流畅，让开发者只需将精力全部放到代码逻辑上而不必管一些烦琐的配置。
```

### 2.1.1 Xcode

```
iOS只支持在macOS中以Xcode方式开发第三方应用，
Xcode由苹果公司提供，可以通过App Store或者苹果开发者官网下载。
```

```
1.下载Xcode
（1）从苹果开发者官网下载首先打开网站https://developer.apple.com，找到Xcode下载链接，如图2-1所示。
    单击“Download”链接，如果账号没有登录，则会有“Sign In”提示，如图2-2所示。
    使用注册的账号登录即可（若没有账号，可单击“Create Apple ID”按钮免费注册）。
    用户在成功登录之后可进入开发工具的下载页面，左边可以进行快速筛选，如笔者输入了“Xcode”且只勾选“Developer Tools”复选框，如图2-3所示。
    这样就可以下载各个版本的Xcode及命令行工具了。
  
（2）从App Store下载在macOS端打开App Store，在右上角搜索文本框输入“Xcode”后进入下载页面，单击“获取”按钮即可下载，如图2-4所示。
    从App Store下载是最方便的，但是只能下载最新版本，若需要下载历史版本，则只能选择从官网下载。
```

```
2.配置Xcode
要让用Xcode编写的应用能在真机运行，还需要登录开发者账号。
运行Xcode，选择“Xcode”菜单下的“Preferences”命令。
切换到“Accounts”选项卡，单击左下角的“+”按钮，在弹出的账号类型选择对话框中选择“Apple ID”，然后输入Apple ID和密码（没有的话，单击“Create AppleID”按钮免费创建一个），单击“Next”按钮即可，如图2-5所示。
```

```
3.真机部署
 1）打开Xcode，选择“File”菜单下的“Project”命令。
    选中“iOS”选项卡里面的“Single View App”模板，单击“Next”按钮后，会进入设置界面，如图2-6所示。
    根据自己的需要填写产品名称、组织名称和公司标识符等信息。
    比如笔者将产品名称设置为“FirstOC”，语言设置为“Objective-C”，单击“Next”按钮，按提示选择保存路径后单击“Create”按钮，一个简单的iOS应用工程就创建成功了。

2）修改“Development Target（开发目标设备）”。
   如果iOS设备系统版本比Xcode默认的版本低，则会提示“OS version lower than deployment target”，如图2-7所示。
   此时将“Development Target”设置为合适的版本即可，如图2-8所示。

3）部署到真机，选择“Product”→“Run”命令（或者单击左上角的三角形按钮）运行程序，会出现一个错误提示对话框，显示需要信任证书，如图2-9所示。
  这是因为设备还没有添加信任证书。
  在设备中依次打开“设置”→“通用”→“描述文件与设备管理”，在“开发者应用”下选择已登录的账号，在弹出的对话框中单击“信任****（您的证书名称）”，并再次单击“信任”按钮即可，如图2-10所示。
  再次运行，程序已经成功在设备上运行起来，虽然界面上除了一个白板什么内容都没有，但它却是一个完整的应用。至此，第一个iOS应用部署成功。
```

### 2.1.2 Homebrew

```
Homebrew是macOS下的一款软件包管理工具，它具有以下优点。
  无需sudo权限安装软件，简单、易用、安全。
  安装路径都在/usr/local，免去了配置环境变量的麻烦。
  解决了程序的依赖问题。
  方便切换各种包的版本。
Homebrew是基于Ruby的（macOS已经自带），所以安装过程也就变得十分简单，只需在终端输入如下代码即可。
$ /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

Homebrew的常用操作方法见表2-1。
  安装软件 brew install xxxx
  卸载软件 brew uninstall xxxx
  更新软件 brew ungrade xxxx
  搜索软件 brew search xxxx
  查看安装列表 brew list
  更新Homebrew brew update

如果实在不习惯用命令行来安装，也可以使用Cakebrew工具，它是Homebrew的前端。
Cakebrew具备非常友好的图形界面，使用起来更加简单快捷，如图2-11所示。
Cakebrew的下载地址是https://www.cakebrew.com/。
https://www.cakebrew.com/
后面的很多小工具都基于Homebrew进行安装，读者务必掌握其常用操作方法。
```

2.2 越狱环境
--------

```
在正式学习iOS逆向工程之前，必须先选择一台合适的设备。
虽然设备在越狱以后都会安装基础框架及依赖，但要想更好地掌控自己的设备，还得先配备好一些实用的小工具。
```

### 2.2.1 iOS 越狱设备的选择

```
究竟如何挑选一台适合自己的设备来学习iOS逆向工程呢？
笔者建议先从https://canijailbreak.com/了解各个版本的越狱支持情况，部分信息如图2-12所示。
了解一些基本信息后，可以从以下方面综合考虑。
● 选择支持ARM64架构的设备（iPhone 5S及以上），因为ARMv7架构逐渐被淘汰已是大势所趋。● 选择iOS 9及以上版本，因为很多App不能在iOS 9之前的版本运行。
● 优先考虑“完美越狱”的设备，因为它们重启后不需要重新越狱。

随着BootROM（checkm8）漏洞的公开，基于该漏洞的“永久越狱”工具checkra1n悄然发布，支持iPhone 5S～iPhone X之间的任何设备。
因为BootROM漏洞无法修复，所以该工具也能支持iOS 12、iOS 13以及未来所有版本设备的越狱。
在https://checkra.in可以下载该越狱工具，喜欢高版本系统的读者可以考虑。
```

### 2.2.2 Cydia Substrate（iOS 11 以下）

```
Cydia Substrate是Cydia官方为了方便越狱开发而提供的一个基础框架，它封装了很多函数，加入了安全的防崩溃保护机制，使插件开发变得简单高效。
在Cydia中搜索“Cydia Substrate”即可安装它。

Cydia Substrate主要由MobileHooker、MobileLoader及Safe mode组成，下面分别进行介绍。

1.MobileHooker
MobileHooker用来对函数进行替换（即Hook），开发人员可以在源代码中添加substrate.h头文件，调用相应的函数来实现Hook功能。
主要用到的函数有MSHookMessageEx()和MSHookFunction()，这两个函数的具体使用方法将在第9章详细讲解。

2.MobileLoader
MobileLoader是实现注入的核心功能模块。其实现原理也将在第9章详细讲解。

3.Safe mode
Safe mode即安全模式。
如果第三方插件导致SpringBoard崩溃，MobileLoader会在捕获到SIGABRT、SIGILL、SIGBUS、SIGSEGV、SIGSYS异常后进入安全模式。

在安全模式里，所有基于Cydia Substrate的第三方插件将被禁用，以便于排查问题。
遗憾的是安全模式并不是万能的，还是会存在一些因为第三方插件而导致无法进入系统的情况，这时候就只能尝试使用下面的方法强制进入安全模式再排查问题。
  ● 按住开机键+〈Home〉键约10秒，iOS设备会自动关闭。
  ● 同时按住开机键和〈音量+〉键，直到进入系统。
```

### 2.2.3 Substitute（iOS 11～iOS 13）

```
Cydia Substrate暂时还不支持iOS 11，因此Electra/Chimera越狱工具集成了由CoolStar修改的Substitute框架。

此框架提供的API已尽可能地和Cydia Substrate保持一致，所以很多越狱插件都能在iOS 11～iOS 13系统上兼容运行。
已安装的Substitute如图2-13所示。
```

### 2.2.4 必备小工具

```
将手机越狱后，需要安装一些必备的工具来增强可操控性。
相信读者学会如何在Cydia中搜索安装软件后，下面的操作应该已经非常熟练。
```

```
1.Apple File Conduit"2"(Apple File Conduit"2")
  Apple File Conduit"2"又称为“afc2add”（简称“AFC2”），用于激活助手类工具对iOS设备所有路径的访问权限。
  为了更好地进行iOS逆向研究，请务必安装此插件。
```

```
2.adv-cmds在手机上执行最频繁的应该就是ps命令了，它用来查看当前运行进程的PID以及应用的路径。
  但这个命令并不是系统自带的，如果尝试在设备上执行ps命令时出现“-sh:ps:command not found”的提示，则需要先安装adv-cmds。
```

```
3.AppSync Unified
  AppSync Unified是iOS设备越狱后的必备补丁，用来绕过系统对应用的签名验证，可以随意安装和运行脱壳后的ipa。
  建议添加插件作者的源（网址为https://cydia.akemi.ai/），搜索“AppSyncUnified”即可安装，该插件支持iOS 5～iOS 13版本系统。
```

```
4.Filza File Manager
  Filza File Manager是手机上的文件管理器（简称“Filza”），用来浏览手机文件、修改文件的权限等，支持iOS 7～iOS 13系统。
```

```
5.NewTerm 2
  NewTerm 2是一款能运行在手机上的终端工具，可以执行各种命令、下载文件、编辑文件等。NewTerm 2能完美支持iOS 7～iOS 13系统，添加源“http://cydia.hbang.ws/”，搜索“NewTerm 2”即可安装。
```

2.3 SSH 配置
----------

```
Secure Shell（SSH）是建立在应用层基础上的安全协议，用于计算机之间的加密登录，可以在不安全的网络中为网络服务器提供安全的传输环境。

SSH最初是UNIX系统上的程序，后来迅速扩展到其他操作平台。
```

### 2.3.1 安装 OpenSSH(use)

```
OpenSSH的主要用途是方便在Windows或者macOS上远程输入命令操作iOS设备。

（1）安装
如果当前的iOS设备系统为iOS 11.0～iOS 12，越狱工具Electra、Chimera已经内置OpenSSH，并默认开放22端口，读者可以直接跳过下面OpenSSH的安装部分。
如果当前的越狱设备系统是iOS 8或iOS 9，则需要在Cydia里面搜索并安装OpenSSH。

（2）测试
依次打开iOS设备的“设置”→“Wi-Fi”页面，再选择已经连接的网络，可以看到设备的IP地址，如图2-14所示。

如笔者这里显示的IP地址为192.168.45.71，则在macOS终端输入：
  ssh root@192.168.45.71
按〈Enter〉键，此时提示输入密码（默认为“alpine”），输入后顺利登录（注意，这里输入密码后是无任何显示的）。
```

### 2.3.2 配置 dropbear

```
Dropbear是一个相对轻量级的SSH服务器，
如果iOS设备的系统介于iOS 10.0和iOS 10.2之间，并且使用了最新版的yalu工具进行越狱，则会默认安装dropbear。

首先进入iOS设备终端，执行“ps aux|grep dropbear”命令，如果看到如下内容，说明只能使用数据线连接：
 /usr/local/bin/dropbear -F -R -p 127.0.0.1:22
    
若需要用Wi-Fi进行连接，则修改yalu102.app下面的dropbear.plist文件。
运行psaux|grep yalu102命令找到其安装目录，然后使用PP助手导出dropbear.plist到macOS上，用Xcode打开它，将其中的“127.0.0.1:22”改为“22”后再导回设备，如图2-16所示。
  
当然也可以直接用Filza File Manager在设备上进行修改。
重启设备，即可顺利登录。至此，SSH已经配置成功。
```

### 2.3.3 免密码登录

```
每次连接SSH的时候都提示输入密码，是不是觉得非常麻烦呢？
下面介绍一种免密码登录的方法。

（1）生成秘钥对
  ssh-keygen -t rsa

（2）将公钥上传到iOS设备
  scp ~/.ssh/is_rsa.pub root@192.168.45.7:/var/root/.ssh/authorized_keys
    
  现在再次尝试SSH连接，无须输入密码就能直接登录了。
  如果iOS设备上不存在/var/root/.ssh/目录，则需要先登录设备创建目录：
  ssh root@192.168.45.71
  cd /var/root
  mkdir .ssh
```

### 2.3.4 USB 连接设备 (use)

```
前面讲的都是用Wi-Fi方式连接设备，这样有个很大的弊端，即如果网络不畅通，使用起来就会非常不方便，尤其是后面动态调试的时候会很卡顿。

本节所讲的USB连接方式就可以解决这个问题。
下面的步骤将当前连接的iOS设备的22端口映射到了macOS的2222端口。

（1）安装usbmuxd
  brew install usbmuxd

（2）端口映射
  iproxy 2222 22 [设备UDID]
  如果有多台iOS设备连接到了同一台macOS，则需要在最后加上需要映射设备的UDID参数，反之可以省略。

（3）连接到设备
  需要另外开启一个终端用2222端口连接：
  ssh --p 2222 root@127.0.0.1
  
  对于iOS 9越狱设备，步骤就比较简单：直接使用PP助手的“打开SSH通道”功能即可，如图2-17所示。
  读者在使用SSH连接设备时，可能会出现“Host key verification failed.”的提示，导致SSH连接失败。
  可能的提示信息如下。
  解决方法：根据上面的提示信息找到“/Users/piao/.ssh/known_hosts”，直接把known_hosts中127.0.0.1的信息删除即可，命令行操作如下。
  ssh-keygen -f /Users/piao/.ssh/know_hosts -R 127.0.0.1
  
  该命令同时会生成known_hosts.old，即原文件的备份。
  更加“暴力”的方法是直接删除known_hosts文件：
  rm /Users/piao/.ssh/know_hosts
```

### 2.3.5 修改默认密码

```
越狱后iOS设备就拥有了最高权限，但root默认密码（“alpine”）是众所周知的，如果开启了SSH而不修改默认密码的话，一旦设备暴露在网络中就很容易被入侵，为了安全考虑，建议立即改掉。

SSH连接iOS设备后，修改root用户默认密码的具体方法如下。
 passwd

如果修改mobile用户的默认密码，则将命令改为“passwd mobile”，其他操作不变。
若使用手机终端（如NewTerm 2）来进行操作，则需要先输入su命令提升到root权限才能修改成功。
```

### 2.3.6 使用 scp 传输文件

```
在配置好SSH之后，可以使用scp工具在iOS设备和计算机之间互相传输文件。

macOS系统自带了scp，
对于非iOS 10的系统，越狱工具也集成了scp，
对于iOS10系统，yalu越狱工具并没有提供scp工具，所以执行命令时会出现下面的错误提示。
sh:scp:command not found
解决方法：添加源（网址为https://coolstar.org/publicrepo/），搜索“scp”安装即可。
  
scp的使用方法很简单，具体如下。
#把macOS上的"chinapyg.txt" 文件复制到ios设备的/tmp/目录
scp -P2222 ./chinapyg.txt root@localhost:/tmp/
  
#把ios设备tmp目录下的"chinapyg.txt"文件复制到macOS
scp -P2222 root@localhost:/tmp/chinapyg.txt ./
```

2.4 实用工具推荐
----------

```
在学习过程中，常常需要一些实用的工具来协助完成某些复杂的操作。
选择合适的工具，不仅能提高工作效率，还能避免很多人为差错。

本节介绍的工具在后续章节都会用到，请提前安装好。
```

```
（1）iTerm2
  iTerm2（网址为https://www.iterm2.com）作为macOS系统自带终端（Terminal）的最佳替代品，具备很多能够提升效率的实用功能，如窗口分割、热键窗口、智能搜索、自动完成、快速复制等。
```

```
（2）SecureCRT
  SecureCRT（网址为https://www.vandyke.com）是一款功能非常强大的跨平台SSH终端软件（支持SSH1和SSH2）。
    它可以新建很多会话，典型的应用场景是在用SSH连接手机的时候可以简化操作，从而避免重复输入一些连接命令。
```

```
（3）Sublime Text
  Sublime Text（网址为https://www.sublimetext.com）是一个跨平台的编辑器，同时支持Windows、Linux及macOS操作系统，拥有漂亮的用户界面和强大的插件扩展功能。
```

```
（4）010 Editor
  010 Editor（网址为https://www.sweetscape.com/010editor）是功能极其强大的跨平台十六进制编辑工具，同时支持Windows、Linux及macOS操作系统，它最出色的就是独特的二进制模板技术，能够方便地分析任何二进制文件格式。
```

```
（5）jtool
  jtool（网址为http://www.newosxbook.com/tools/jtool.html）完全可以替代系统自带的otool命令来解析Mach-O文件，它还提供了许多新颖的功能，如二进制搜索、代码签名等。
    最重要的是，它可以运行在macOS、iOS甚至Linux平台上，后面很多章节会用到它。
```

```
（6）tree通常情况下，通过ls命令可以看到当前文件夹中的内容，但是不能看到子文件夹的内容，且展示方式并不人性化。
    tree是一个操作极其简单的命令行工具，它可以像Windows的文件管理器一样清楚明了地显示目录结构。
    它可以使用brew进行安装：
    brew insatll tree
```

2.5 本章小结
--------

```
本章分别讲述了开发环境及越狱环境的搭建过程，
另外着重讲解了各版本系统的SSH配置方法及技巧，
最后推荐了日常工作中的一些实用工具。
对本章内容的掌握与否决定后续章节的操作是否顺畅，请读者仔细阅读。
```

第 3 章 逆向基础
==========

3.1 Mach-O 文件格式
---------------

```
Mach-O是iOS/macOS系统上应用程序的文件格式，它并不像Windows平台中的PE文件那样复杂。
了解Mach-O文件的格式，对于静态分析、动态调试、自动化测试及安全都很有意义。
在掌握了二进制文件的数据结构以后，一切都会显得没那么神秘。
```

### 3.1.1 通用二进制文件

```
由于iOS应用需要支持不同的CPU架构，所以默认编译打包出来的是一个UniversalBinary（通用二进制，也称为胖二进制）格式文件。
用file命令即可查看文件信息：
  file AVPlayer
上面的结果显示程序支持ARMv7和ARM64两种架构。
实际上通用二进制文件只是将不同架构的Mach-O文件打包在一起，再在文件起始位置加上fat_header结构来说明所包含的Mach-O文件支持的架构和偏移地址信息。

通用二进制文件的结构可用图3-1直观表示。

可以在mach-o/fat.h头文件中查看通用二进制文件的定义：
  ● magic：魔数（特征字段），被定义为FAT_MAGIC常量，取值固定为“0xcafebabe”，加载器通过该特征来判断这是什么样的文件。
  ● nfat_arch：指当前的通用二进制文件包含了多少个不同架构的Mach-O文件。
    fat_header后会紧跟fat_arch结构，有多少个不同架构的Mach-O文件，就会有多少个fat_arch，用于描述对应Mach-O文件的具体信息，其定义如下。
  
  ● cputype：标识CPU的架构，如ARM、ARM64、MIPS、X86_64等。
     它的类型是cpu_type_t，其定义位于mach/machine.h文件。
     在iOS平台中cputype通常如下。
  ● cpusubtype：标识具体的CPU类型，区分不同版本的处理器，它的类型是cpu_subtype_t。在iOS平台中cpusubtype通常如下。

  ● offset：指定该CPU架构数据相对于文件开头的偏移量。
  ● size：指定该CPU架构数据的大小。
  ● align：指定数据的内存对齐边界，取值必须是2的N次方。
  可以使用otool工具的-f参数查看fat_header结构，-v参数显示详细参数：
    otool -f -v AVPlayer
  fat_arch之后就是每个Mach-O文件的分布了。
  Mach-O文件由三部分组成：Mach-O头部（Header）、加载命令（Load Commands）、数据块（Data），如图3-2所示。
  接下来将对整个Mach-O文件结构进行简要分析。
```

### 3.1.2 Mach-O 头部

```
Mach-O头部（Header）保存了CPU架构、大小端序、文件类型、加载命令数量等一些基本信息。
通过头部就能按顺序向下解析整个Mach-O文件。
```

```
1.Mach-O头部定义
头部的定义可以在mach-o/loader.h中找到。
32位与64位架构的CPU分别使用mach_header与mach_header_64结构体来描述Mach-O头部。
它们的结构大同小异，本书仅讨论64位，其定义如下：、

● magic：魔数（特征字段），用于标识当前设备是大端序还是小端序，iOS是小端序。
  对于64位架构的程序，它被定义为常量MH_MAGIC_64，取值固定为“0xfeedfacf”。

● cputype、cpusubtype：与fat_header结构体中的定义完全一样。
● filetype：表示Mach-O文件的类型，如可执行文件、符号文件、内核扩展文件等，可在mach-o/loader.h中找到具体定义和取值。
   iOS逆向工程中接触得最多的有两种类型。
    #define MH_EXECUTE 0x2 可执行文件
    #define MH_DYLIB 0x6   动态库文件
    
● ncmds：表示Mach-O文件中加载命令的数量。
● sizeofcmds：表示Mach-O文件加载命令所占的总字节大小。
● fags：表示Mach-O文件的一些标志信息，读者可在mach-o/loader.h中找到其具体的定义和取值。
   其中的“#define MH_PIE 0x200000”值得注意，这个标记只在MH_EXECUTE中使用，意思是启用ASLR（Address Space LayoutRandomization，地址空间布局随机化）来增加程序的安全性。
   简单地说，程序每次启动后，加载地址都会随机变化，这样程序里的所有代码引用都是错的，需要重新对代码地址进行计算修正才能正常访问。
● reserved：系统保留字段，仅在mach_header_64结构中存在。
   简单总结一下：头部用于校验Mach-O文件的合法性及确定文件的运行环境。
   读者可用otool的-h参数分析该部分的数据。
```

```
2.使用可视化工具分析Mach-O文件
otool使用的是命令行操作方式，另外还有一些操作更便捷、显示更直观的可视化工具可供选择。
（1）使用MachOView分析
MachOView是macOS平台下的一款老牌Mach-O文件分析工具，主要用来解析各种信息。

MachOView属于免费开源项目，源代码和编译好的文件均可在https://github.com/gdbinit/MachOView下载。

直接将Mach-O文件拖动到MachOView图标上即可进行分析，界面如图3-3所示。

（2）使用010 Editor分析

010 Editor的模板功能非常强大，不过官方自带的Mach-O模板目前不能分析ARM64架构的程序，所以需要选择第三方的模板。
在010 Editor中依次单击“Templates”→“View Installed Templates”，在弹出的对话框中单击“Add”按钮添加一个模板，指定模板路径，最后单击“OK”按钮即可，如图3-4所示。

之后就可以在“Templates”菜单下看到刚刚添加的模板菜单项了，选择后会对当前文件进行分析，如图3-5所示。
```

### 3.1.3 加载命令

```
加载命令（Load Commands）紧跟在Mach-O头部之后，明确地告诉加载器如何处理二进制数据，有些命令是由内核处理的，有些是由动态链接器（dyld）处理的。

使用MachOView查看加载命令，如图3-6所示。

load_command的数据结构定义如下：
此结构仅有两个字段，cmd表示当前加载命令的类型，cmdsize表示当前加载命令的大小。
根据不同的加载命令类型，内核会使用不同的函数来解析，cmd的完整定义可以在mach-o/loader.h头文件里找到，

图3-6中的cmd包含以下部分。
● LC_SEGMENT_64：定义一个段，加载后被映射到内存中。段中又包含节（section）。
● LC_DYLD_INFO_ONLY：记录了动态链接的重要信息，动态链接器要根据它来进行地址重定向。

● LC_SYMTAB：文件所使用的符号表，找到后分别获取符号表偏移量、符号数、字符串表偏移量、字符串表大小。
● LC_DYSYMTAB：动态链接器所使用的符号表，找到后获取间接符号表偏移量。

● LC_LOAD_DYLINKER：默认的加载器路径（/usr/bin/dylb）。
● LC_UUID：Mach-O文件的唯一标识。dSYM文件（符号文件）和崩溃堆栈中都存在这个值，用于分析对应的崩溃位置。

● LC_VERSION_MIN_IPHONEOS：Mach-O文件要求的最低系统版本，和Xcode中配置的target有关。
● LC_SOURCE_VERSION：构建二进制文件的源代码版本。

● LC_MAIN：程序的入口。动态链接器获取该地址，然后程序跳转到该处执行。
● LC_ENCRYPTION_INFO_64：文件加密信息，包括加密标记、加密数据的偏移和大小，脱壳时需要用到。
● LC_LOAD_DYLIB：依赖的动态库，包括动态库路径、当前版本、兼容版本等。
  在未越狱环境注入动态库时需要操作它。

● LC_RPATH：@rpath的路径，指定动态链接器搜索路径列表，以便定位框架（framework）。
● LC_FUNCTION_STARTS：函数起始地址表，使调试器或其他程序能够判断一个地址是否在该表范围内。
● LC_DATA_IN_CODE：定义在代码段内的非指令表。
● LC_CODE_SIGNATURE：代码签名信息，校验签名及修复脱壳后的应用闪退需要用到。

下面将对几个重要的命令类型进行讲解。
```

```
1.LC_SEGMENT

1.LC_SEGMENTLC_SEGMENT和LC_SEGMENT_64是段加载命令，每个段都定义了一个虚拟内存区域，动态链接器负责把这个区域映射到进程地址空间。
LC_SEGMENT使用segment_command结构体表示，
LC_SEGMENT_64使用segment_command_64结构体表示，它们的结构大同小异。

segment_command_64定义如下：
● cmd：表示当前加载命令的类型。
● cmdsize：表示当前加载命令的大小。
● segname：段名称，占用16字节。
● vmaddr：段的虚拟内存地址。
● vmsize：段的虚拟内存大小。
● fileoff：段在文件中的偏移量。
● filesize：段在文件中的大小。
● maxprot：段页面的最高内存保护（r、w、x）。
● initprot：段页面的初始内存保护（r、w、x）。
● nsects：段中包含节的数量。一个段可以包含0个或者多个节。
● fags：段的标志信息。

系统从fileoff处加载filesize大小的内容到虚拟内存vmaddr处，大小为vmsize，
段页面的权限由initprot进行初始化，这些权限可以被修改，但是不能超过maxprot的值。

例如，逆向时关注的代码段（__TEXT）的初始化和最高内存权限都是可读/可执行/不可写，这就是未越狱状态下不能Inline Hook的原因。

段名称用下划线和大写字母表示，如图3-6所示，示例程序包含四个段：
● __PAGEZERO：静态链接器创建了__PAGEZERO作为可执行文件的第一个段，该段在虚拟内存中的位置及大小都为0，不能读写、不能执行，用来处理空指针。
  开发者尝试访问NULL指针时，会得到一个EXC_BAD_ACCESS错误。
● __TEXT：包含了可执行的代码和其他一些只读数据。
  静态链接器设置该段的虚拟内存权限为可读、可执行，进程被允许执行这些代码，但是不能修改。
● __DATA：包含了将会被更改的数据。
  静态链接器设置该段的虚拟内存权限为可读写。
● __LINKEDIT：包含了动态链接库的原始数据，如符号、字符串和重定位表条目等。
```

```
一个段可以包含0个或者多个节，32位和64位分别用section和section_64结构体表示，它们的定义基本相同。
section_64定义如下：
  ● sectname：节的名称，占用16字节。
  ● segname：节所属的段名称，占用16字节。
  ● addr：节在内存的起始位置。
  ● size：节占用的内存大小。
  ● offset：节的文件偏移地址。
  ● align：节的字节对齐大小。
  ● reloff：重定位入口的文件偏移。
  ● nreloc：需要重定位的入口数量。
  ● fags：节的类型和属性。
  ● reserved1/2/3：系统保留字段。
  段和节的一个命名规则是，大写代表的是段，小写代表的是节。
  示例程序的__TEXT段和__DATA段所包含的节如图3-7所示。
```

```
（1）__TEXT段
● __text：主程序代码。
● __stubs、__stub_helper：用于帮助动态链接器绑定符号。

● __const：const关键字修饰的常量。

● __objc_methname：OC方法名。
● __cstring：只读的C语言字符串。

● __objc_classname：OC类名。
● __objc_methtype：OC方法类型（方法签名）。

● __gcc_except_tab、__ustring、__unwind_info：GCC编译器自动生成，用于确定异常发生时栈所对应的信息（包括栈指针、返回地址及寄存器信息等）。
```

```
（2）__DATA段
● __got：全局非懒绑定符号指针表。
● __la_symbol_ptr：懒绑定符号指针表。
● __mod_init_func：C++类的构造函数。
● __const：未初始化过的常量。
● __cfstring：Core Foundation字符串。
● __objc_classlist：OC类列表。
● __objc_nlclslist：实现了+load方法的Objective-C类列表。
● __objc_catlist：OC分类（Category）列表。
● __objc_protolist：OC协议（Protocol）列表。
● __objc_imageinfo：镜像信息，可用它区别Objective-C 1.0与2.0。

● __objc_const：OC初始化过的常量。
● __objc_selrefs：OC选择器（SEL）引用列表。
● __objc_protorefs：OC协议引用列表。
● __objc_classrefs：OC类引用列表。
● __objc_superrefs：OC超类（即父类）引用列表。
● __objc_ivar：OC类的实例变量。

● __objc_data：OC初始化过的变量。
● __data：实际初始化数据段。
● __common：未初始化过的符号声明。
● __bss：未初始化的全局变量。

为了加快程序启动速度，iOS引入了“懒绑定”和“非懒绑定”的概念，这属于动态链接器加载Mach-O的范畴，读者大致了解一下即可。
  非懒绑定：在动态链接器加载程序的时候就会绑定真实调用地址，之后直接使用即可。可将其理解为“主动绑定”。
  懒绑定：只有方法被调用的时候才会去寻找对应的调用地址，然后再执行。可将其理解为“被动绑定”。
  感兴趣的读者可以通过“xcrun dyldinfo”来查看更加详细的绑定信息，如图3-8所示。
```

```
2.LC_LOAD_DYLIB
LC_LOAD_DYLIB指向的都是程序依赖库的加载信息。
使用MachOView查看LC_LOAD_DYLIB命令，可以发现程序加载的一些常见库，如CFNetwork、UIKit等。如图3-9所示。

LC_LOAD_DYLIB加载命令对应的数据结构为dylib_command，定义如下：
● name：依赖库的完整路径。动态链接器在加载动态库时，会通过此路径进行加载。
● timestamp：依赖库构建时的时间戳。
● current_version：当前版本号。
● compatibility_version：兼容版本号。
   该结构对应的命令还可能是LC_LOAD_WEAK_DYLIB，它们都表示需要加载一个动态库。
   通过LC_LOAD_WEAK_DYLIB声明的依赖库是可选的，如果缺少这些库也没有什么影响，主程序会继续执行；
   而LC_LOAD_DYLIB则不同，依赖库若是没有找到，加载器会放弃并结束该进程。
   
依赖库的加载路径是有讲究的，在日常分析过程中，除了大家熟悉的“/System/Library/”和“/usr/lib/”系统路径之外，还可能见到@rpath、@executable_path之类的路径。例如：
“@rpath”是由LC_RPATH加载命令指定的，iOS平台上通常都是存放应用自身的framework文件，
 默认路径为“@executable_path/Framework”，
 
而“@executable_path”则是可执行文件的目录。
这些路径其实是可以修改的，在macOS上提供了一个install_name_tool工具，就可以直接修改它们：
  # 修改依赖库路径
  # install_name_tool -change @executable_path/iThunderBypass.dylib @rpath/iThunderBypass.dylib iThnuder
  
  在未越狱平台注入动态库的原理就是向Mach-O文件添加一条LC_LOAD_DYLIB加载命令，将路径指向自己的动态库，这时就会见到类似于“@executable_path/iThunderBypass. ylib”或者“@rpath/iThunderBypass.dylib”的依赖库了。在第9章将会讲解此方法。
```

3.LC_CODE_SIGNATURE
-------------------

```
LC_CODE_SIGNATURE是代码签名加载命令，通常位于最后一个段中，描述了Mach-O文件的代码签名信息，在iOS中如果签名校验不通过，进程会立即被内核用SIGKILL命令杀死，
所以越狱后有一环节就是绕过代码签名校验。
它属于链接信息，使用linkedit_data_command结构体表示：

dataoff表示数据相对于__LINKEDIT段的文件偏移，datasize表示数据大小。
与代码签名相关的数据定义可以在xnu内核代码中找到（见https://opensource.apple. com/source/xnu/xnu-3789.70.16/bsd/sys/codesign.h.auto.html），
整个代码签名的头部使用一个CS_SuperBlob结构体定义，它的原型如下：
```

```
下面讲解每个字段的含义。
● magic：表示Blob的类型，可取值如下：
  对于代码签名信息头部来说，magic必须为CSMAGIC_EMBEDDED_SIGNATURE，表示采用了嵌入式签名信息。
● length：所有Blob的总大小。
● count字段：共有多少个Blob。
● index[]字段：每个Blob的索引信息。其结构CS_BlobIndex定义如下：
  offset表示该Blob距离代码签名起始位置的偏移，
  type表示该Blob的类型，
  可取值如下：
用MachOView并不能很好地解析代码签名信息，可以改用010 Editor的Mach-O模板直观地进行解析，如图3-10所示。
  第1列是解析出来的结构，
  第2列是每个字段的数值，
  第3列是偏移量，
  第4列是结构体或字段占用的空间大小，
  最后一列显示一些注释信息。
  由解析结果可知，存在5个Blob，它们的类型和偏移值见表3-1。
```

```
（1）CSSLOT_CODEDIRECTORY第1个Blob指向的是CSSLOT_CODEDIRECTORY，它是一个CS_CodeDirectory结构体：
  这个结构里面保存了一些相当重要的hash信息，后面介绍脱壳后解决闪退问题时就要修复它。
  使用010 Editor查看CSSLOT_CODEDIRECTORY数据，如图3-11所示。
  
● hashOffset：表示codeHash信息距离当前结构起始地址的偏移
 （示例为0xBB，实际上就是0x2440144+0xBB=0x24401FF）。
 这些codeHash的计算规则是：将整个代码段按pageSize指定的大小（通常为0x1000）进行分页，然后依次计算，并从0x24401FF开始写入，nCodeSlots表示分了5017页，也即存在5017条codeHash信息。
 hashSize表示每个codeHash占用的大小，与hashType指定的算法有关，iOS目前使用SHA-1和SHA-256算法，长度分别为20和32。

● identOffset：表示标识符距离当前结构起始地址的偏移（示例为0x34，实际上就是0x2440144+0x34=0x2440178），
  查看得知该标识符为“AVPlayer.eplayworks.com”，长度为24字节。
  
● nSpecialSlots：表示特殊条目的specialHash数量，这些条目紧跟在标识符后面，通常为5个，
  第1个是CSSLOT_ENTITLEMENTS对应结构数据的hash值，
  第2个通常为空，
  第3个是资源目录的hash值，也就是AVPlayer.app/_CodeSignature/CodeResources文件的hash值，
  第4个是CSSLOT_REQUIREMENTS对应结构数据的hash值，
  第5个是Info.plist文件对应的hash值。
  用jtool的-sig参数可以分析这些特殊条目的数据，具体如下：
  jtool -arch arm64 --sig -v AVPlayer
```

```
（2）CSSLOT_REQUIREMENTS
 第2个Blob指向的是CSSLOT_REQUIREMENTS，它是一个CS_SuperBlob结构体，保存了整个数据的长度和REQUIREMENT的个数，如图3-12所示。
 这里对应的magic是CSMAGIC_REQUIREMENT，长度为84，按照后面的提示用codesign查看数据：
 codesign --display -r- AVPlayer
```

```
（3）CSSLOT_ENTITLEMENTS
 第3个Blob指向的是CSSLOT_ENTITLEMENTS，是权限相关的一些配置信息，它是一个CS_GenericBlob结构体：
 其data数据实则是一个plist文件，用codesign或jtool都可以提取：
 在第7章配置调试器或兼容iOS 11运行的时候，都将添加或修改这些配置。
```

```
（4）CSSLOT_ALTERNATE_CODEDIRECTORIES
 第4个Blob仍然指向CSSLOT_CODEDIRECTORY，是一个备用的Blob，结构和第1个Blob一样。  
 这里采用了SHA-256计算hash，在iOS 11之前，只用了第1个Blob，
而在iOS 11中，第4个Blob里面的hash信息如果不对，就会造成闪退。
```

```
（5）CSSLOT_SIGNATURESLOT
  第5个Blob指向CSSLOT_SIGNATURESLOT。
  它是签名时使用的证书，其结构也是CS_GenericBlob，如图3-13所示。
  在010 Editor中将data数据导出为cer文件，使用macOS的“快速查看xxx.cer”功能对保存的文件进行预览，如图3-14所示。
  读者也可以使用codesign命令快速查看文件的简要签名信息，如下：
  codesign -vvv -d AVPlayer
```

3.2 ARM 汇编基础
------------

```
对于从其他平台转向iOS的逆向研究者来说，ARM是一门全新的语言，可能听说过，但从未接触过。想要更好地研究iOS平台的逆向，其底层汇编语言是必须掌握的。
以笔者多年的逆向工程经验来看，从高级语言过渡到汇编语言需要了解三个重要概念：
寄存器、常用指令及栈。
本章将会按照这个顺序进行讲解，当大家理解了汇编语言的精髓时，看到的所有代码都将是源代码。
```

### 3.2.1 寄存器

```
寄存器是CPU的一个组成部分，里面存放着指令、数据和地址等供CPU计算使用，速度比内存快。
寄存器通常可分为通用寄存器、浮点寄存器和状态寄存器。
```

```
1.通用寄存器
ARM64有31个通用寄存器，每个寄存器可以存取一个64位的数据。
 当使用X0～X30时，它就是一个64位的数；
 当使用W0～W30时，实际访问的是这些寄存器的低32位，写入时会将高32位清零。
在指令编码中，0b11111（31）用来表示ZR（0寄存器），仅表示参数0，并不表示ZR是一个物理寄存器，如图3-15所示。
```

```
除了X0～X30寄存器外，还有一个SP寄存器也非常重要。
下面介绍iOS逆向工程中关注频率最高的一些寄存器。
● X0～X7：用来传递函数的参数，如果有更多的参数则使用栈来传递；X0也用来存放函数的返回值。
● SP（Stack Pointer）：栈指针寄存器。指向栈的顶部，可以通过WSP寄存器访问栈指针的最低有效32位。
● FP（Frame Pointer）：即X29，帧指针寄存器。指向栈的底部。
● LR（Link Register）：即X30，链接寄存器。
  存储着函数调用完成时的返回地址，用来做函数调用栈跟踪，程序在崩溃时能够将函数调用栈打印出来就是借助LR寄存器来实现的，如图3-16所示。
● PC（Program Counter）：保存的是将要执行的下一条指令的内存地址。
  通常在调试状态下看到的PC值都是当前断点处的地址，所以很多人认为PC值就是当前正在执行的指令的内存地址，其实这是错误的。
 大家可以在Xcode中对任意一个地址设置断点，然后实际观察一下（需要在左下角选择“All”才会显示所有变量），如图3-16所示。
```

```
2.浮点寄存器
因为浮点数的存储及运算的特殊性，所以CPU中专门提供了FPU以及相应的浮点数寄存器来进行处理。
ARM64有32个浮点寄存器（V0～V31），每个寄存器大小是128位。
开发者可以通过Bn（Byte）、Hn（Half Word）、Sn（Single Word）、Dn（Double Word）、Qn（Quad Word）来访问不同的位数，如图3-17所示。
```

```
3.状态寄存器
状态寄存器用来保存指令运行结果的一些信息，比如相加的结果是否溢出、是否为0及是否为负数等。
CPU的某些指令会根据运行的结果来设置状态寄存器的标志位，而某些指令则是根据这些状态寄存器中的值来进行处理。
ARM64体系的CPU提供了一个32位的CPSR (Current Program Status Register)寄存器来作为状态寄存器，
  低8位（包括M[0:4]、T、F和I）称为控制位，程序无法修改，
  第28～31位的V、C、Z和N均为条件代码标志位，
  它们的内容可被算术或逻辑运算的结果改变，如图3-18所示。
  状态寄存器的内容由CPU内部进行置位，在程序中不能将某个数赋值给它。
  
下面介绍4个条件代码标志位及其含义。
● N（Negative）：当两个有符号整数进行运算时，N=1表示结果为负数，N=0表示运行结果为正数或零。
● Z（Zero）：Z=1表示运算结果为零；Z=0表示运算结果为非零。
● C（Carry）：当加法运算结果产生进位时（无符号数溢出），C=1，否则C=0；
             当减法运算结果产生借位（无符号数溢出）时，C=0，否则C=1。
● V（Overfow）：在加/减法运算中，当操作数和运算结果为二进制补码表示的带符号数时，V=1表示符号位溢出。
```

### 3.2.2 指令集

```
ARM内核属于RISC（Reduced Instruction Set Computer）结构，所以其指令长度固定、指令格式种类少、寻址方式简单。
ARM处理器内部的指令译码采用硬布线逻辑、不使用微程序控制，以减少指令的译码时间，使大部分指令可以在一个时钟周期内完成。
指令的基本格式如下：
  <opcode> {<code>}{s} <Rd>,<Rn> {,<shift_op2>}
  操作码 条件码          目标寄存器 操作数 操作数(立即数，寄存器移位方式)
其中，尖括号是必需的，花括号是可选的，各关键字的含义见表3-2。

下面介绍一些逆向工程中常见的ARM64指令集，完整资料请读者参考ARM64官方手册。

1.常用算术指令（见表3-3）表
2.常用跳转指令
（1）条件跳转（见表3-4）
（2）无条件跳转（见表3-5）
3.常用逻辑指令（见表3-6）
4.常用数据传输指令（见表3-7）
5.常用地址偏移指令（见表3-8）
6.常用移位运算指令（见表3-9）
7.常用加载/存储指令（见表3-10）
加载/存储指令都是成对出现，有时也会遇到这些命令的一些扩展，比如LDRB、LDRSB等，它们的含义见表3-11。

ARM指令的一个重要特点是可以条件执行，每条ARM指令的条件码域包含4位条件码，共16种。
几乎所有指令均根据CPSR中条件码的状态和指令条件码域的设置有条件地执行。
当指令执行条件满足时，指令被执行，否则被忽略。
指令条件码及其助记符后缀见表3-12。
```

### 3.2.3 栈及传参规则

```
1.栈的特性
  栈就是指令执行时存放函数参数、局部变量的内存空间。
  栈的特性是“先进后出”（即First In Last Out，也称为FILO）。
  栈只有一个入口，先进去的就到最底下，后进来的就在最上面，取数据时，就是从入口端取，所以说栈是先进后出、后进先出。
  iOS是小端模式，所以栈的增长方向是从高地址到低地址的，栈底是高地址，栈顶是低地址。栈的特性可以用图3-19概括。
```

```
2.传参规则
下面通过编写实例来研究一下函数调用时的参数是如何传递的。
（1）参数少于8个时的传参
先用Xcode新建一个iOS工程，添加一个mul函数并调用，代码如下：

在调用处添加断点，然后依次选择Xcode菜单中的“Debug”→“Debug Workfow”→“Always Show Disassembly”命令（当然也可以直接用命令进行转换，这样不需要真机即可得到ARM64汇编代码：xcrun-sdk iphoneos clang-S-arch arm64main.m），断点命中后，就直接以汇编代码呈现了：

笔者对代码加入了一些注释，并按照逻辑将其划分为五个部分，它们的含义如下。
● ①、⑤：这两部分是对称的，称为方法头和方法尾。
  在编译器生成汇编时，首先会计算需要的栈空间大小，
  在方法头部，通过向下移动SP指针在栈上分配空间，把当前FP和LR保存起来，然后将当前FP切换到新栈帧，以抬高栈底；
  在方法尾部，恢复栈上的FP和LR到寄存器，然后向上移动栈指针SP，来释放栈上的内存。
  这两部分其实也是为了保持栈平衡，前面开辟了多少空间，调用结束后就需要归还多少，从而恢复成函数调用前的样子。如果每次只开辟空间而不归还，栈很快会被消耗完毕。
● ②、③：主要是调用mul和print函数的一些操作。
● ④：因为ARM寄存器不支持直接把值写入内存，
     所以先把返回值0写入W8中，
     然后把W0写入[SP+0xC]的内存中，
     最后把X8（存储了0）写入X0，作为main函数的返回值。

接着看一下mul函数的汇编代码和注释：
那么mul函数为什么需要开辟多达16字节的栈空间？
  这是因为ARM规定SP必须按照16字节对齐，
  读者若将mul的参数增加到5个进行测试，
  会看到需要开辟0x20的栈空间。
  mul函数内的栈分布如图3-20所示。
```

```
（2）参数多于8个时的传参
  前面说过寄存器X0～X7用来传递前8个参数，这在上述汇编代码里面也能得到验证。
  如果超过8个参数又该如何传递呢？
  现在添加另外一个函数，将参数扩充到10个：

中断后读取汇编代码，可以清晰地看到最后两个参数通过栈来传递，如图3-21所示。
可以通过读取内存进一步求证：
  memory read $sp
```

```
（3）浮点数的传参现在来看一下浮点数。
 将代码修改如下：
 中断后可以看到前8个参数不再使用X0～X7传递，而是使用D0～D7传递，第9个参数开始仍然使用栈传递。
 如下所示：
 还有一些其他数据类型的参数传递，读者可以自行了解。
```

### 3.2.4 内联汇编

```
在逆向工程中，有些场合（比如反调试）必须用汇编语言来实现，
除了使用纯汇编源文件以外，还可以在高级语言中嵌入汇编代码来实现，这种方式称为“内联汇编”。
内联的汇编指令与通常的ARM指令有所区别，被嵌入的代码在形式上表现为独立定义的函数体，遵循过程调用标准。
本节均以C语言内联汇编为基准进行讲解。
```

```
1.语法格式
该语法共包括四个部分：汇编语句模板、输出、输入和会被破坏的寄存器。
各部分用“:”隔开，汇编语句模板必不可少，其他三部分可选，如果使用了后面的部分，而前面部分为空，也需要用“:”隔开，相应部分内容为空。
其中，参数volatile是可选的，如果用了它，则告诉编译器不允许对该内联汇编进行优化，否则当启用优化选项(-O)进行编译时，汇编代码就会被编译器优化得面目全非。
```

```
2.实例演示
为了方便讲解，先在Xcode里面写入一个加法函数，代码如下：
● 第一行：汇编语句模板
  w0、%w1和%w2代表指令的操作数，称为占位符，内联汇编根据它们将C语言表达式与指令操作数对应起来。
  模板后面的小括号内是C语言表达式，本例有3个：sum、a、b，它们按照出现的顺序分别与指令操作数%0、%1、%2对应。
  操作数最多只能有10个（即用%0～%9表示）。
  百分号后面的字母表示这个变量的位宽，“w”表示32位，“x”表示64位。
  
● 第二行：输出参数。
  sum前面的限制字符串是“=r”，其中“=”表示sum是输出操作数，“r”表示需要将sum与某个通用寄存器相关联，先将操作数的值读入寄存器，然后在指令中使用相应寄存器，而不是sum本身。
  当然指令执行完后需要将寄存器中的值存入变量sum，从表面上看好像是指令直接对sum进行操作，实际上编译器做了隐式处理，这样可以少写一些指令。

● 第三行：输入参数。
  “r”表示该表达式需要先放入某个寄存器，然后在指令中使用该寄存器参与运算。
  现在来看一下反汇编代码，编译器将嵌入的占位符用实际的寄存器替代了，具体如下：
  读者可能注意到上面的内联代码并没有“会被破坏的寄存器”，下面试着添加几个参数来研究一下。 
  再次反汇编查看代码，会发现编译器申请了更大的栈空间，将“会被破坏的寄存器”用STP指令保存了起来，待调用完成后又使用LDP指令进行了恢复，所以这部分的实际意义就是保护指定的寄存器不被破坏，具体如下：
```

```
3.纯裸函数
读者应该已经注意到，编译器会“自作聪明”地将上面编写的内联汇编生成头部和尾部的栈平衡代码，然而在某些特殊场合，开发者并不需要编译器处理栈平衡，只想“所见即所得”，这就要使用下面介绍的裸函数来处理了。
裸函数使用“__attribute__((naked))”关键字表示，实例如下：
反汇编后的代码如下：
这显然是上面要求的效果。需要注意的是，如果使用了裸函数，则在函数内部不能使用局部变量，否则编译不通过。
使用内联汇编+裸函数能做很多意想不到的事情，限于篇幅，笔者也只能点到为止了。
```

### 3.2.5 Objective-C 的汇编机制

```
Objective-C中的方法调用是通过消息机制来实现的（即[object method:arg]），现在编写一个简单的方法来研究一下它的汇编代码到底是怎样的。
  代码如下：
  反汇编代码如下：
在上述代码内部已经将Objective-C代码转换为objc_msgSend调用，在其调用处可以将寄存器内容打印出来，如下所示：
可以看到，在Objective-C中，
   X0寄存器保存了对象本身，
   X1寄存器保存了方法名，
   从X2～X7开始便是参数，其他的参数通过栈传递，由于是64位的程序，所以每8个字节一组。本节所讲的Objective-C汇编传参规则请读者一定要记住，这将贯穿全书内容。
```

3.3 本章小结
--------

```
本章主要讲解Mach-O文件格式和ARM汇编基础知识，这些都是iOS逆向工程的必备知识点。
笔者已经尽量将这些枯燥的理论知识进行精简，仅保留了一些常用的知识，并通过一些图解及实例来加深印象。
所谓“万丈高楼平地起”，希望读者把逆向基础知识完全掌握后再进行后面的学习。
即将开始真正的iOS逆向工程之旅，准备好了吗？
```

第 4 章 应用脱壳
==========

```
iOS端App在上线之前会由苹果商店进行FairPlayDRM数字版权加密保护（称为“加壳”）。
要对应用进行分析，就必须先解密（称为“脱壳”），从而得到原始未加密的二进制文件。
本章将讨论各种各样的脱壳技术。
```

4.1 检测是否加壳
----------

```
如何检测应用是否加壳了呢？笔者讲解两种常规的方式。

1.使用otool检测用otool可以看到二进制文件的信息里有一个cryptid字段，
  cryptid=1表示已加壳，cryptid=0表示未加壳。具体如下。
  otool -l WhatsApp | grep crypt

2.使用MachOView检测如果不想输入命令行，则可以使用MachOView来查看。
  将目标文件拖入MachOView，展开“Load Commands”节点，
  选择“LC_ENCRYPTION_INFO_64”选项，右边可以看到“Crypt ID”，如图4-1所示。
```

4.2 Clutch
----------

```
Clutch是一款全自动脱壳工具，其原理是把应用运行时的内存数据按照一定格式导出，并重新打包为ipa文件。
```

### 4.2.1 安装 Clutch

```
从https://github.com/KJCracks/Clutch/releases直接下载最新版，复制到iOS设备的/usr/bin/目录，然后添加执行权限，操作如下。
   # macOS执行
   scp -P 2222 -r ./Clutch root@localhost:/usr/bin/
   # ios执行
   chmod +x /usr/bin/Clutch
   
 输入Clutch命令，如果输出了帮助信息则表示安装配置成功，具体如下。
   Clutch
```

### 4.2.2 Clutch 脱壳实战

```
使用-i参数打印从App Store安装的所有应用列表：
  Clutch -i
  
这里用序号为1的“WhatsApp Messenger”进行脱壳演示：
  Clutch -d net.whatsapp.WhatsApp
  
上述信息提示脱壳完成，重新打包后的文件为
/private/var/mobile/Documents/Dumped/net.whatsapp.WhatsApp-iOS7.0-(Clutch-2.0.4).ipa。

值得一提的是，最终脱壳出的文件架构和使用的iOS设备有关，
如笔者的iPhone 6设备，脱壳出来的就是ARM64架构，如果放到ARMv7架构的设备上是不能正常运行的。
如果需要多架构可用，则需要再准备一台ARMv7架构的设备进行脱壳，然后用lipo命令将两个架构进行合成。
PP助手下载的越狱版应用就是多架构可用的。
```

4.3 dumpdecrypted
-----------------

```
上一节提到的Clutch使用过程是“傻瓜式”的，但很多应用使用它脱壳会失败。
本节介绍另外一款脱壳工具dumpdecrypted，它在使用上会比Clutch稍微麻烦一些，但非常灵活，效果也很不错。
```

### 4.3.1 编译 dumpdecrypted

```
Dumpdecrypted是开源的，需要先编译、签名，再将其复制到iOS设备中。
从https://github.com/stefanesser/dumpdecrypted.git可下载最新源代码。
```

```
1.编译dumpdecrypted
  进入dumpdecrypted文件夹，输入make命令编译生成dumpdecrypted.dylib：
  ~/Path/to/dumpdecrypted
  make
  ls -al

编译dumpdecrypted的时候可能会出现如下警告，然后导致编译失败：
  make
  解决方法：编辑Makefile文件，将“-F$(SDK)/System/Library/PrivateFrameworks”删除后再重新编译。
```

```
2.给dumpdecrypted.dylib签名
  下面提供两种签名方式，读者自行选择。
  1）使用ldid签名：
    ~/path/to/dumpdecrypted
    ldid -S dumpdecrypted.dylib
  2）使用codesign签名：
    ~/path/to/dumpdecrypted
    codesign -f -s - ./dumpdecrypted.dylib
```

```
3.复制dumpdecrypted.dylib到设备
 用scp命令将dumpdecrypted.dylib复制到iOS设备的/usr/lib/目录：
 scp -P2222 ./dumpdecrypted.dylib root@localhost:/usr/bin/
到此为止，准备工作就完成了。
```

### 4.3.2 dumpdecrypted 脱壳实战

```
为了操作方便，笔者选择先进入tmp目录（如果脱壳失败，请进入沙盒的Documents再进行），脱壳后的文件就会保存于此。
接着用ps命令查看目标文件的完整路径，
最后使用DYLD_INSERT_LIBRARIES环境变量将dumpdecrypted.dylib注入就可以脱壳了。
具体如下。

cd /tmp

ps -ax ｜ grep WhatsAPP

DYLID_INSERT_LIBRARIES = /usr/bin/dumpdecrypted.dylib /var/containers/Bundle/Application/666666/WhatsApp.app/WhatsApp

WhatsApp.decrypted为脱壳后的文件，使用scp命令或者PP助手导出到macOS上即可。
```

4.4 bfinject
------------

```
如果当前的设备系统是iOS 11及以上版本，那么Clutch、dumpdecrypted不进行改造的话，目前都无法正常使用，这时候可以选择bfinject工具包，它集成了脱壳工具及Cycript等。
```

### 4.4.1 安装 bfinject

```
GitHub上有两个版本的bfinject，https://github.com/BishopFox/bfinject中是原始版本，
只能支持iOS 11～iOS 11.1.2，https://github.com/MJavad/bfinject中是修改版本，
经笔者测试已经完美支持iOS 11～iOS 11.4.1。

1）将修改版clone到macOS上：
git clone https://github.com/MJavad/bfinject

2）连接iOS设备并在根目录创建bfinject文件夹：
        ssh root@localhost -p 2222
        cd /
        mkdir bfinject

3）另外开启一个终端，将bfinject.tar上传到iOS设备：
    cd bfinject
    ls | grep *.tar
    bfinject.tar
    scp -P2222 ./bfingect.tar root2localhost:/

4）进入iOS设备的/bfinject目录并解压：
    cd /bfinject
    bfinject root #tar xvf bfinject.tar 

至此，bfinject就已经安装好了。
```

### 4.4.2 bfinject 脱壳实战

```
依然用WhatsApp来做实验。先确保目标App已经运行，并进入bfinject目录，使用如下命令进行脱壳。
  ./finject -P WhatsAPP - L decrypt

如果顺利完成，界面会弹出“Decryption Complete”对话框，如图4-2所示。
由于稍后会使用scp命令将文件复制到macOS上，所以无须理会该对话框上的NetCat提示，直接单击“No”按钮即可。

脱壳完成的文件存放在应用沙盒的Documents目录下，名为“decrypted-app.ipa”，
如果打开了控制台就能更加快捷地从日志进行定位，如图4-3所示。

使用scp命令将decrypted-app.ipa复制到macOS，利用PP助手或者Xcode安装decrypted-app.ipa，
如果能正常运行则到此结束，否则请看下一节。
```

### 4.4.3 修复闪退

```
如果脱壳后的ipa包安装后运行闪退，则需要稍微处理一下。
具体如下。

1.解压ipa包
  ~/path/to/WhatsApp
  unzip decrypted-app.ipa
  
2.导出ent.xml
codesign -d --entilements - ./Payload/WhatsApp.app > ent.xml

3.使用codesign重签名
codesign -s - --entilements ent.xml -f ./Payload/WhatsApp.app/WhataApp

4.压缩成新的ipa包
 zip -r WhatsApp_ok.ipa Payload
 
 重新安装处理后的WhatsApp_ok.ipa，即可解决闪退问题。
 至此，bfinject的脱壳过程就全部完成了。
```

4.5 CrackerXI（iOS 11～iOS 13）
----------------------------

```
CrackerXI是脱壳工具的后起之秀，专为iOS 11～iOS 13量身打造，是目前为止最为傻瓜式的脱壳工具。

CrackerXI的安装比较简单，直接在Cydia中添加源地址http://cydia.iphonecake.com/，然后搜索安装即可。

安装完成后，会在桌面出现CrackerXI图标，打开后会呈现已安装的App列表。

在脱壳之前，先进入“Settings”页面进行一些必要的设置，如图4-4所示。如果安装后没有出现相应图标，则进入终端执行“uicache-a”即可。

现在进入“App List”页面，单击需要脱壳的App，在随后弹出的对话框中单击“YES，Full IPA”按钮，即会自动进行脱壳并重新打包成ipa文件，完成后会弹出图4-5所示的对话框。

至此，脱壳就很简单地完成了。
```

4.6 Frida-ios-dump
------------------

```
Frida-ios-dump基于Frida（一个跨平台的轻量级Hook框架）提供的强大功能，
通过注入JS实现内存dump，
然后利用Python自动复制到macOS生成最终的ipa文件。

Frida的具体安装步骤请读者查看第5章。
```

### 4.6.1 一键快速脱壳

```
Frida-ios-dump的原理和dumpdecrypted一样，都是通过把内存中已解密的数据dump再修复Mach-O，
但dumpdecrypted仅能dump主程序，对于框架需要自行修改源代码才能完成，而且操作上比较麻烦。
而Frida提供的强大功能，使得frida-ios-dump只需一键即可快速完成脱壳。

首先从GitHub下载frida-ios-dump，并查看帮助：
git clone https://github.com/AloneMonkey/frida-ios-dump
cd frida-ios-dump
./dump.py -h

Frida-ios-dump支持直接填写应用名或BundleID进行脱壳，
参数-o指定ipa文件的输出路径，
参数-l显示所有已安装的应用。

以微信脱壳为例，命令如下。
  ./dump.py 微信 -o ./WeChat_Decrypted.ipa

笔者发现由当前frida-ios-dump版本脱壳后的WeChat_Decrypted.ipa仅能在iOS8上运行，
其他系统上会出现“Service exited due to signal:Killed:9”错误（闪退），
下一小节来完美修复它。
```

### 4.6.2 完美修复闪退

```
使用dumpdecrypted和bfinject脱壳后同样会发生闪退情况，之前都是用codesign重签处理。

既然重签能够运行，就说明闪退是由于签名校验失败所致，下面将从根源上解决这个问题。

Cluth脱壳的程序是能正常运行的，对其源代码研究后发现它进行了hash（散列，又叫“哈希”）值的修正处理。

在学习Mach-O文件格式时讲过，LC_CODE_SIGNATURE加载命令存放的是一些与签名有关的数据，
而里面最重要的是
CSSLOT_CODEDIRECTORY和
CSSLOT_ALTERNATE_CODEDIRECTORIES，
它们包含了代码段的SHA-1和SHA-256校验信息。
Cluth源代码里面有一个步骤修正了SHA-1的hash值，所以在iOS 9上运行没问题，但是iOS 11校验了SHA-256的hash值，而该值又没有修正，所以仍然会闪退。

笔者根据校验原理写了一个名为“FixMachO”的macOS端小工具，它能自动修正签名段的hash值，其关键代码如图4-7所示。

将FixMachO放到dump.py的同级目录下，接着修改dump.py文件，让脚本在生成ipa文件之前先调用FixMachO，如图4-8所示。
```

### 4.6.3 ipa 文件安装失败处理

```
如果将脱壳后的ipa文件安装到不同类型的设备，有可能会出现“DeviceNotSupported”错误，控制台中的错误提示如下。

这是因为设备支持列表中没有目标设备的类型。
将ipa文件解压，找到Info.plist文件，在UISupportedDevices项添加自己的设备类型（或者直接删除UISupportedDevices项），如图4-9所示。

将处理后的Info.plist文件重新打包放入ipa文件再安装即可。
```

4.7 使用 lipo 分离架构
----------------

```
前文已经说过了，最终脱壳出的文件架构和使用的iOS设备有关。

Mach-O是胖文件格式，可能存在多种架构，那些没被脱壳的架构已经没有存在的意义，将其剔除还可以节省不少空间。

macOS平台自带的lipo工具就是负责这项工作的。
lipo的功能非常强大，不但能合并多个Mach-O文件到一个胖文件格式，也能从一个胖文件格式中分离指定架构的Mach-O文件。
下面的例子使用lipo工具的-info参数查看目标文件的架构，然后使用-thin参数分离ARMv7和ARM64架构：
 lipo -info WeChat
 lipo WeChat -thin armv7 -output WeChat_armv7
 lipo WeChat -thin arm64 -output WhChat_arm64
 
另外，如果想在64位设备上运行32位程序，只需要提取ARMv7架构即可，因为在iOS 11系统之前，指令集都是向下兼容的，但是到了iOS 11及以后的系统，就只保留了ARM64架构，也是本书着重讲ARM64的原因。
```

4.8 本章小结
--------

```
脱壳乃逆向分析的前奏，是必须掌握的技能。
本章从易到难讲述了各种脱壳工具的使用与技巧，相信总有一个方法适合你。
最重要的是本章从根源上分析了脱壳后闪退的原因及修复办法，这都是很多资料不曾提及的。
```

第 5 章 运行时分析
===========

```
Objective-C是一门面向对象的动态编程语言，只有在程序运行时才会去确定对象的类型，并调用类与对象相应的方法。
于是，iOS逆向领域出现了一些工具，利用这些运行时（Runtime）特性，使得开发者能够在程序运行时动态修改类及对象中的所有属性及方法。

在iOS逆向工程中，运行时分析是一种简单快捷但又非常实用的定位目标函数的手段。
学习本章之前，假设读者已经具备了相关的开发基础知识。
```

5.1 class-dump
--------------

```
class-dump是一个命令行工具，它利用Objective-C语言的运行时特性将二进制文件中的类、方法及属性等信息导出为头文件。
class-dump算是一个入门级工具，它不需要开发者具备复杂的逆向工程基础，即可轻松了解程序的大致结构。
从http://stevenygard.com/projects/class-dump/下载最新安装包，双击打开，将“class-dump”拖动到/usr/local/bin目录下即可使用。
安装包里面还包含了源代码，读者也可以尝试自行编译安装。
```

```
以WhatsApp为例来讲解它的使用方法：
  class-dump --arch arm64 -a -A -H ./Payload/WhatsApp.app/WhatsApp -o ./Headers
  
-arch：指定架构（本例只有一个架构，也可以不写）；
-a：显示变量的偏移地址；
-A：显示方法实现的地址；
-H：生成头文件；
-o：指定头文件的输出位置。

命令执行完成后，进入Headers文件夹可以看到一共生成了3004个头文件，如图5-1所示。
任意打开一个头文件查看，变量名、方法名、偏移地址等毫无保留地显示出来了，如图5-2所示。在这些头文件里面可以根据关键字找到很多线索，在后面的实战章节会有具体应用。
```

5.2 Cycript
-----------

```
Cycript（官网为http://www.cycript.org/）是由Cydia创始人Saurik推出的一款脚本语言，它混合了Objective-C与JavaScript语法解释器，能够探测和修改运行中的应用程序。

Cycript主要用于注入目标进程来实现运行时调试，
它的优点是重启程序后所有的修改都会失效，对原生程序或代码完全无副作用。
```

### 5.2.1 越狱环境安装 Cycript

```
越狱环境下，直接在Cydia中搜索“Cycript”安装即可。安装成功后，在iOS设备上输入“cycript”，设备将会显示cy#的提示符：
  cycript

这是一个JavaScript控制台，输入的所有内容都由JavaScript内核运行，如果输入有误，将会提示语法错误：
  var a=5
  Math.pow(a,7)
  var a=5

这时可以使用〈Control+C〉快捷键取消或使用〈Control+D〉快捷键退出Cycript。
在越狱手机上，可以使用“-p”选项注入目标进程，进入命令行交互模式后，调用目标函数。

下面的命令先注入WeChat，然后调用NSHomeDirectory()函数获取沙盒路径：
cycript -p WeChat
NSHomeDirectory()
```

### 5.2.2 Cycript 实战

```
在实际逆向工程中，通常会使用Cycript进行注入程序、查看程序界面、调用程序函数、修改程序返回值等操作，以验证自己的猜想。

下面以WhatsApp为例来介绍实际的分析方法。启动WhatsApp进入欢迎页面，单击“同意并继续”按钮会进入下一个页面，如图5-3所示。
本小节的目的是找到该按钮的响应事件，并在Cycript里面调用它。

在正式分析功能之前，有必要先讲一下MVC的概念。
MVC（Model ViewController）是一种非常经典的设计模式，
其中，Model作为数据管理者，View作为数据展示者，Controller作为数据加工者，Model和View又都是由Controller根据业务需求调配的，如图5-4所示。

由图可知，Controller在整个MVC框架中起了主导性的作用，若要获得按钮响应事件（action），则首先要想办法得到按钮所在的Controller。
下面就一步步来分解。
```

```
1.分析视图层次结构

众所周知，苹果系统中的私有函数在正向开发中是不能使用的，但在逆向分析中可以发挥很大的用途，其中有一个recursiveDescription函数可以用来递归打印任意视图层次结构，
再结合.toString()就能使得输出格式看上去更加清晰，如下所示：
 
 cycript -p WhatsApp
 [[UIApp keyWindow] recursiveDescription].toString()
 
Cycript有一个“友情提示”功能，即当开发者不清楚函数名时，只要输入关键字后按Tab键就会列出所有匹配项，例如：
 recursive ^TAB ^TAB
```

```
初学者看到如此多的输出内容估计会很烦恼，那么这里再提供一个简化版的方法：
 在UIWindow里有一个名为“_autolayoutTrace”的私有函数，该函数的返回值是一个字符串，
 这个字符串则包含了UIWindow中整个视图的层次结构。具体如下：
[[UIApp keyWindow] _autolayoutTrace].the oString()

现在看起来就简单多了。
```

```
2.定位“同意并继续”按钮

寻找控件的思路有很多，可以根据不同情况来选择最优雅的办法，比如本例的按钮是在最下边，所以可以根据位置来定位，找到疑似对象：
  UIButton:0x6666666 '\u200e'

另外一种直观的方法是根据按钮的文字来找。
但遗憾的是，Cycript对中文显示不太友好，只能通过编写脚本进行转码：
 echo -e "\u200e"
 python -c 'print u"\u200e"'

这样操作比较麻烦且不那么优雅，有没有一种快速无误又不用转码的方法来验证自己的猜想呢？
答案是肯定的，可以用一些小技巧，比如：
  [#0x155d9f220 setHidden:YES]

在Cycript中，如果知道一个对象在内存中的地址，就可以通过“#”操作符来获取这个对象。
现在对0x1023627a0这个UIButton进行隐藏测试，会发现“同意并继续”按钮消失了，这说明目标找对了。
```

```
3.获取按钮对应的控制器
按照MVC设计标准，通过Controller就很容易得到Model，因为C层既访问V层，也能访问M层。
基于这个思路，先使用nextResponder逐层打印响应链，最终就能得到Controller。
这里调用了两次nextResponder来得到目标：
 [#0x155d9f220 nextResponder]
 [#0x155db5310 nextResponder]
```

```
4.获取按钮响应事件UIButton继承自UIControl，
  而UIControl可以通过allTargets与allControlEvents查看所有的对象与事件，
  再使用“actionsForTarget:forControlEvent:”找到响应事件，具体如下：
 [#0x155d9f220 allTargets]
 [#0x155d9f220 allControlEvents]
 [#0x155d9f220 actionsForTarget:#0x1571301d0 forControlEvevnt:64]
```

```
5.验证结果找到了可疑函数，最后一步就是对分析结果进行验证了：
 [#0x1571301d0 acceptAction:nil]

此时页面已经有所动作，跳到了下一个页面，说明“同步并继续”按钮的响应事件已经成功调用，如图5-5所示。
```

### 5.2.3 Cycript 高级用法

```
1.查找某个对象
如果知道一个类对象存在于当前进程中，却不知道它的内存地址，则不能通过“#”操作符来获取它，这时，choose命令就派上用场了，它将返回符合条件的所有记录，加下标能访问某个元素。

具体如下：
  choose [UITableView]
  choose(UITableView)[0]
```

```
2.增加Category
  在Cycript里也可以使用Objective-C的Category。具体如下：
  //为UIDevice类增加一个deviceName方法，返回固定字符串
  @implementation UIDevice (MyUIDevice)
  - deviceName {return "iPhones XS Max";}
  @end
  
  choose(UIDevice)]
  
  [#0x1065ac830 deviceName]
```

```
3.编写自定义函数
 在实战中，有一个环节是通过nextResponder逐层打印响应链，得到ViewController，层次很深的话，将会非常麻烦。
 此时，不妨试试利用Cycript编写自定义函数来解决这个问题。具体如下：
 function findVC(view){
   var cur_view  = view;
   while(cur_view isKindOfClass:NSClassFromString(@"UIViewController")]!=true){
     cur_view  = [cur_view nextResponder];
   }
   return cur_view;
 }
 
 findVC(#0x155d9f220)
 
 下面这个函数用来打印某对象的所有属性：
  function printIVars(obj){
    var ivars = {};
    for(i in * obj){
      try{
        ivars[i]=(*obj)[i];
      }catch(e){}
    }
    return ivars;
  }
  
  printIvars(#0x1571301d0)
```

```
4.使用.cy文件封装函数上面的一些自定义命令在每次分析时都要手动输入一次，非常麻烦，
  利用.cy文件可以将常用的Cycript命令封装在一个文件中。
  
  在/usr/lib/cycript0.9/com/saurik/substrate/目录下可以找到一个MS.cy样例：
  
  cat /usr/lib/cycript0.9/com/saurik/substrate/MS.cy
  (function(exports){
    export.hookFunation = function(func,hook,old){
    }
    export.hookMessage = funcation(isa,sel,imp.old){
    }
   })(exports);
  
  先在/usr/lib/cycript0.9/com/目录下新建“piao”文件夹，再新建pp.cy文件，按照样例将常用命令编写进来：
 (function(exports){
    export.pviews = functions(){
      retunr [UIApp keyWindow] recursiveDescription.toString();
    }
   
     exports.findVC = functions(view){
       var cur_view  = view;
       while(cur_view isKindOfClass:NSClassFromString(@"UIViewController")]!=true){
         cur_view  = [cur_view nextResponder];
       } 
       return cur_view;
 
      exports.printIVars = function(obj){
        var ivars = {};
        for(i in * obj){
          try{
            ivars[i]=(*obj)[i];
          }catch(e){}
        }
        return ivars;
      }
   
   })(exports);
  
    
之后就可以通过@import命令导入并使用：  
  @import com.piao.pp
  pp.views()
  
  脚本文件函数的多少取决平时的积累，希望读者能根据实际情况编写更多的实用脚本函数，并到论坛分享自己的心得。
```

### 5.2.4 iOS 11 使用 Cycript

```
由于Cycript并不支持iOS 11，所以需要借助前面章节提到的bfinject工具来辅助注入开启监听，并在macOS中进行远程连接。
以WhatsApp为例，先通过SSH连接设备，然后执行如下命令：
  
  ./bfinject -P WhattApp -L Ctycript
  
  设备上出现图5-6所示的提示时，记下其中的IP和端口信息，单击“Ok”按钮。之后用macOS版的Cycript工具进行远程连接后操作即可。具体如下：
      ./cycript -r 192.168.199.185:1337
```

5.3 Reveal
----------

```
Reveal是一款强大的UI调试工具，可以调试iOS应用和tvOS应用。
它可以在运行时查看App的界面层级关系，还可以实时修改程序界面，不用重新运行程序就可以看到修改之后的效果，免去了每次修改代码后又重新启动的过程。
逆向工程里面通常用Reveal来快速定位感兴趣的控件，进而找到控制器，再用Cycript进行事件分析。
```

### 5.3.1 越狱环境使用 Reveal

```
越狱环境下只需先装Reveal2Loader插件，再稍微配置一下即可。
```

```
1.安装Reveal2Loader
  在Cydia中搜索“Reveal2Loader”进行安装，安装完成后会在“设置”里面出现“Reveal”选项，如图5-7所示。
```

```
2.将RevealServer.framework导入设备
  在Reveal安装目录的/ShareSupport/iOS-Libraries/子目录下可以找到RevealServer. framework文件，将其导入设备的/Library/Frameworks目录，如下：
  cd /Applications/Reveal.app/Concents/SharedSypport/ios-Libraries
  scp -P 2222 -r ./RevealServer.framework root@localhost:/Library/Frameworks
```

### 5.3.2 iOS 11～iOS 13 使用 Reveal

```
如果在iOS 11～iOS 13系统上使用Reveal2Loader，就会发现无论如何都不会成功，错误日志如下：

 这是因为RevealServer所在目录的权限问题导致其无法加载，Reveal2Loader的作者在GitHub上提供了源码（网址为https://github.com/zidaneno5/Reveal2Loader），下载回来稍作修改，如图5-8所示。
 
 接着在Package目录下新建/usr/lib目录，把RevealServer.framework移动到此，如图5-9所示。
 
 将修改好的安装包重新编译成deb文件（可参考后面的越狱开发相关章节）后安装到设备即可正常使用。这样修改后也是支持其他版本系统的，完全可以替代Cydia里面的Reveal2Loader。
```

### 5.3.3 Reveal 实战

```
打开iOS设备的“设置”→“Reveal”→“Enabled Applications”页面，进入应用选择列表，
选择需要分析的应用，如图5-10所示。

现在打开目标应用，在Reveal中即可看到被分析的应用了，分析效果如图5-11所示。
```

```
Reveal界面中的内容如下所述。
● 左边部分是应用页面的层级结构，可以看出每一层是由哪些控件构成以及自定义控件的名字等，在底部的过滤文本框内输入关键字可以匹配到相应的控件。
● 中间部分是一个可视化视图，可以在这里实时查看视图效果，包括视图框架、缩放、2D视图和3D视图等。
● 右边部分是选中控件的属性、内存地址等参数设置，这些参数都可以实时修改，例如修改UILable的文字及颜色，如图5-12所示。
```

5.4 FLEX
--------

```
FLEX (Flipboard Explorer)是一个iOS应用的内部调试工具。

当它加载时，会向目标程序上方添加一个悬浮的工具栏，通过这个工具栏，可以查看和修改视图的层次结构、动态修改类的属性、动态调用实例和方法、动态查看类和框架以及动态修改UI等。

与其他调试工具不同，FLEX完全在应用程序内部运行，因此不需要连接到LLDB、Xcode或其他远程调试器，也不需要太多编程知识，仅需手动点几下就能查看很多细节，是一款非常强大的分析工具
```

### 5.4.1 越狱环境使用 FLEX

```
越狱环境下使用FLEX来分析第三方应用时，只需要在Cydia中搜索FLEXible插件安装即可。

FLEXible是对FLEX的封装，支持iOS 8～iOS 13。
安装完毕后，进入iOS设备的“设置”→“FLEXible”→“Enabled Applications”页面，会列出所有已安装的应用，根据需要选择待分析的目标即可，如图5-13所示。

重新打开被分析的应用，会发现页面上方出现了FLEX的调试工具栏，如图5-14所示。

其中，
“menu”是工具菜单项，里面包含了App的网络、文件、内存、函数及库文件等信息；\
“views”是当前视图布局的层次结构图，可用来定位函数；
“select”用于选定某个控件；
“move”可以移动所选的控件；
“close”则是关闭工具栏。
```

### 5.4.2 FLEX 实战

```
本节使用FLEX来分析微信的“漂流瓶”功能。
先进入目标页面，使用调试工具栏中的“select”选中该页面的“捡一个”按钮，如图5-15所示。

图中的阴影区域便是已选中的目标，同时调试工具栏上也会根据颜色显示当前选中控件的基本信息，
比如笔者选中的“捡一个”，它是一个BottleButton类型的按钮。

接下来单击调试工具栏中的“views”，进入当前选中区域的视图布局层次结构图，会自动定位到选中的控件，如图5-16所示。

单击BottleButton视图最右方的小按钮，进入详细信息页面。
里面包含了此视图的所有属性和方法，比如视图预览、大小、位置、可视性等。
这时能看到在IVARS里面有一个“toggleFish”的可疑SEL，根据类型和命名来推断，这应该是BottleButton的具体方法实现，如图5-17所示。

带着这个疑问，先通过图5-16找到该按钮对应的ViewController为SandyBeachView-Controller，
单击右边的小按钮进入，然后单击进入“SHORTCUTS”→“View Controller”→“SandyBeachViewController”详情页，

在顶部的过滤器文本框中输入“toggle”关键字，匹配到了4个条目，其中有一个就是“toggleFish”，从名称来看，这些就是“漂流瓶”页面各个按钮的响应方法，如图5-18所示。

FLEX可以很方便地对某个方法进行即时调用，下面利用这个功能来求证以上分析是否正确。
选择“-(void)toggleFish”，进入该方法的详情页，单击右上角的“Call”按钮，如图5-19所示。

回到“漂流瓶”页面，发现已经扔了一个出去，说明分析是正确的，接着就可以用IDA等工具对toggleFish的逻辑进行分析了。

另外，FLEX还可以对带参数的方法进行调用，输入参数可以由开发者控制。
如图5-20所示，调用“-(void)toggleSetting:(id)”方法，传入参数使用nil，单击“Call”按钮之后，就能顺利进入“漂流瓶设置”页面。

本节介绍了FLEX的基本用法，以及如何利用它定位目标方法。
在不同的应用中，目标方法的定位略有不同，但是思路大同小异。
同时，大家应该也领略到了FLEX的便捷，就算没有计算机的配合，也能随时随地对应用进行“把玩”。
```

5.5 Frida
---------

```
Fria是一个跨平台的轻量级Hook框架，支持所有主流操作系统，它可以帮助逆向研究人员对指定的进程进行分析。
它主要提供了精简的Python接口和功能丰富的JS接口，除了使用自身的控制台交互以外，还可以利用Python将JS脚本库注入目标进程。

使用Frida可以获取进程详细信息、拦截和调用指定函数、注入代码、修改参数、从iOS应用程序中dump类和类方法信息等。

Frida源代码托管在GitHub：https://github.com/frida/，感兴趣的读者可以下载阅读。
```

### 5.5.1 Frida 安装

```
Frida需要安装在控制端（macOS端）与被控端（iOS端），两端的版本号最好保持一致，否则可能无法正常工作。
1.被控端（iOS端）打开越狱商店添加软件源https://build.frida.re/，搜索“Frida”进行安装，当前最新版本为12.6.10。
重启手机后能在iOS端看到“frida-server”后台程序，说明安装成功，如下所示：
 ps -ax | grep frida
```

```
2.控制端（macOS端）
控制端可以安装到Windows、macOS、Linux等平台，本书仅以macOS平台为例。
使用sudo权限，利用pip安装或升级frida及frida-tools，如下：
 sudo pip install frida
 sudo pip install frida-tools
 frida --verrsion
 12.6.10
 # 升级
 sudo pip install frida --ungrade
 sudo pip install frida-tools--ingrade
 
 如果执行命令的过程中出现“zsh:command not found:pip”错误，说明本机没有安装pip，通过以下步骤解决：
  brew install wget
  wget https://bootstrap.pypa.ip/get-pip.py
  sudo opython get-pip.py

另外可能会出现以下错误：
解决方法如下：
  sudo pip install six --upgrade --ignore-installed six
```

### 5.5.2 Frida 入门

```
除了主程序frida以外，frida-tools里面还提供了五个实用工具：
frida-discover、
frida-kill、
frida-ls-devices、
frida-ps以及
frida-trace，

它们位于/usr/local/bin/目录下，如图5-21所示。

这些工具都是基于Frida的Python接口实现的，在官网有简单的使用帮助（见https://www.frida.re/docs/home/），想深入了解这些工具的话，可以阅读它们的源码。
```

```
1.获取可用设备列表frida-ls-devices用于获取可用设备列表，在与多个设备进行交互时非常有用，如下所示：
  frida-ls-devices
  
 获取了Id之后，就可以在frida-ps等工具中使用了。
```

```
2.获取设备进程列表
  frida-ps与ps命令的功能类似，用于获取进程列表信息，它的帮助信息如下：
  frida-ps [options]

● -U：连接到USB设备。
● -D：如果有多个USB设备，可以用该选项指定设备的UDID。
● -R/-H：连接到远程frida-server，主要用于远程调试。

● -a：仅显示正在运行的应用。
● -i：显示所有已安装的应用（包括App Store应用和系统应用）。

实际上这份帮助信息中除了“-a”和“-i”以外都是公共的，后面的很多工具都支持这些选项。

frida-ps的具体使用方法如下：
  #连接到USB设备查看进程列表
  frida-ps -U
  #连接到USB设备查看正在运行的应用
  frida-ps -U -a
  #连接到USB设备查看所有安装的应用
  frida-ps -U -a -i
  #指定查看某个设备
  frida-ps -D a666666 -a
```

```
3.结束设备上的某个进程
  frida-kill工具用来结束设备上的某个进程，它的使用规则如下:
  frida-kill -U <PID>/<Name>
  frida-kill -D <DEVICE-ID> <PID>/<NAME>

例如用frida-ps获取了微信的PID为1815，则可以用如下命令结束它：
  frida-kill -U 1815
  frida-kill -U 微信
  frida-kill -D a6666 1815
  frida-kill -D a6666 微信
```

```
4.跟踪函数/方法调用
  frida-trace工具用于跟踪函数或方法的调用，其功能非常强大，读者可以使用“-h”选项来查看帮助信息。
```

```
（1）跟踪函数调用
    ~/Desktop
    frida-trace -U -i compress -i "recv*" -x recvmsg*: -x reacfromwei xin 

   “-i”选项表示包含某个函数，
   “-x”选项表示排除某个函数，它们都支持模糊匹配，可以组合使用。
   
   命令执行后会自动生成基本的JS文件，仅输出一些日志，开发者可以根据需要修改它们。
   上面命令的意思是：
     跟踪名为“compress”的函数和“recv”开头的函数，
     同时排除“recvmsg”开头的函数和名为“recvfrom”的函数，模糊匹配的内容要使用双引号包围。当函数触发后就会输出下面的日志：
     
   以上过程是附加到目标进程再操作，如果需要强制启动进程进行跟踪，可以使用“-f”选项加上应用的BundleID，命令如下：
  frida-trace -U -f com.tencent.xin -i compress -i "recv*" -X"recvmsg*" -x recvfrom
  
  无论采用何种方式进行追踪，在当前目录下都会生成一个“__handlers__”文件夹，里面就是frida-trace自动生成的脚本文件，例如：
  ~/Desktop/__handlers__
  
  frida-trace在启动时不会覆盖已有的脚本文件，所以可以随时修改这些JS文件来添加自己的功能。
```

```
（2）跟踪Objective-C方法调用
frida-trace -U -m  "-[BaseMsgContentLogicController SendNotGameEmotionMessage:errorMsg:]" 微信

“-m”选项表示包含某个方法，
“-M”选项表示排除某个方法，
支持模糊匹配，可以组合使用。触发后的日志如下：
```

```
（3）跟踪导出函数：
  frida-trace -U -I mars -I libxml12.2.dylib -I "MM*" -X MMCommon 微信
  “-I”选项表示包含某个模块，
  “-X”选项表示排除某个模块，
  它们都支持模糊匹配，可以混合使用。如果模块的导出函数很多，命令执行时间将会很长，非特殊情况下不建议对整个模块进行跟踪。
```

```
（4）跟踪导入函数
 frida-trace -U -t CoreDFoundation 微信
 #跟踪主程序的所有导入函数
 frida-trace -U -T 微信
  
  “-t”选项表示包含某个模块，
  “-T”选项表示包含主程序，可以组合使用。
```

```
（5）跟踪偏移地址“-a”选项添加模块内偏移地址的监控，使用方法是“MODULE!OFFSET”，由于macOS终端会把“!”转义，所以需要输入“\!”才能正常使用。
  例如对Aweme模块中偏移地址为0x2A65CCC的函数进行追踪，示例如下：
  ～/Desktop
  frida-trace -U -f com.ss.iphone.ugc.Aweme -a Aweme\!2A65CC
```

```
（6）跟踪调用栈利用Frida可以非常方便地跟踪某个方法的调用栈，只需要在JS里面添加下面的代码片段即可：
 log('\tBacktrace:\n\t'+Thread.backtrace(this.context, BackTracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n\t'));
```

```
5.进入交互模式主程序frida进入交互模式有两种方法：（1）通过应用名或PID附加
 (1)通过应用名或pid附加
  frida -U 微信
  frida -U -p PID
  
（2）直接启动进程
  frida -U -f com.tencent.xin

如果读者在使用“-f”选项启动应用后被强制退出或者不想额外再输入“%resume”，
请加入“-no-pause”选项，它表示不中断应用程序的启动。如下：
  frida -U -f com.tencent.xin --no-pause
进入交互模式后就可以访问目标进程的所有属性、方法等。
```

### 5.5.3 Frida 实战

```
上一小节介绍了Frida最基本的使用方法，大家已经见识到了它的强大，但是Frida真正的强大之处岂止于此？Frida提供的函数（方法）拦截功能堪称一绝，

它能动态地对某个类或方法甚至是C函数进行替换或追踪，达到Hook的目的。
本节用一个经典的实例来讲解Frida的函数拦截技术。

微信是大家日常生活中使用频率很高的沟通工具，其指纹支付功能更是方便快捷。
某一天，笔者将自己的手机越狱后，发现微信团队发来一个通知“你好，由于当前设备已越狱，指纹支付功能已自动停用”，如图5-22所示。同时，位于“支付管理”页面的“指纹支付”开关也消失了。
下面对这一功能进行分析。

由于微信对指纹支付的检测相对温柔，所以给了大家一次使用Frida练手的机会。

大家知道，“越狱”的英文通常为“JailBroken”或“JailBreak”，
因此，使用关键词“jail”对当前进程的所有类及方法进行检索，就能初步筛选出一些符合要求的内容。
而Frida提供的JavaScript API里面，

由ObjC.classes能得到当前应用中所有已经注册的类，对它进行遍历就可以得到符合要求的类名，

再由“ObjC.classes.XXXXClass.$methods”就可以得到XXXXClass类的所有方法，对它遍历就能找到符合要求的方法名。

按照此逻辑编写一段JS代码：
//筛选符合要求的类及方法
if(ObjC.available){
  for(var className in ObjC.classes){//获取所有类
   if(classname.toLowerCase().indexOf('jail')!=-1){
     console.log("[#]ClassName------>"+className)
     
     //获取类的所有方法
     var method = eval('ObjC.classes.'+className+'.$methods'){
       for(var i=0;i<methods.length;i++){
          try{
            if(methods[i].toLowerCase().indexOf('jail')!=-1)
              console.log("[-]"+methods[i])
          }catch(err){
              console.log("[!]Exception:"+err.message)
          }
        }
     }
  }
}
```

```
将这段脚本保存为“WeChat.js”，可以通过下面两种方法来加载：
#1.附加或者启动时使用“-l”选项加载脚本
 frida -U -l Wechat.js 微信
#2.进入交互界面后使用%load手动加载脚本
%load path/to/WeChat.js
```

```
由输出信息可以看到，存在一个JailBreakHelper类，匹配到的方法有6个，
进一步分析发现“+ JailBroken”和“- IsJailBreak”的嫌疑最大。

现在对它们进行拦截以确定最终调用了哪个方法。

Frida提供的JavaScript API里面有一个Interceptor拦截器，将它附加到某个函数后，可以在调用前和调用后分别进行拦截，以获取函数的参数和返回值。

根据帮助文档的指引，编写以下JS脚本：

//打印函数返回值
if(Objc.available){
  try{
    var className = 'JailBreakHelper';
    var methodNames = ['+jailbroken','-lsJailBreak'];
    
    for(var index in methodNames){
      vat methodName = methodNames[index];
      var hook = eval('ObjC.classes.'+className+'["'+methodname+'"]');
      
      Interceptor.attach(hook.implementation,{
        onEnter:function(args){
          console.log("======onEnter========");
          console.log("[*]Class Name:"+className");
          console.log("[*]Method Name:"+methodName);
          console.log("=====================");
        },
        onLevel:function(retval){
          console.log("======onLevel======");
          console.log("[*]Class Name:"+className");
          console.log("[*]Method Name:"+methodName);
          console.log("\t[-] Return Value"+retval);
          console.log("=====================");
        }
    });
  }
    }catch(err){
      console.log("[!]Exception: "+err.message);
    }
}
onEnter和onLeave是两个回调函数，分别处理原始方法调用前和调用后的逻辑。
Frida支持脚本热更新，当JS文件被修改并保存后，能够实时生效，这样免去了重新加载脚本的麻烦。
再次进入“支付管理”页面，让越狱检测逻辑触发，此时日志如下：
```

```
由此可断定“支付管理”页面仅调用了“-IsJailBreak”方法来判断是否越狱。
继续修改JS脚本，将函数返回值改为“NO”，如下：

//修改函数返回值
if(Objc.available){
  try{
    Interceptor.attach(ObjC.classes.JailBreakHelper['-IsjailBreak'].implementation,{
      onLevel:function(retval){
        console.log("return = NO");
        retval.replace(ptr(0x0);
      }
    });
  }catch(err){
    console.log("[!]Exception:"+err.message);
  }
}

脚本生效后，再次进入“支付管理”页面，此时“指纹支付”选项已经显示出来了，如图5-23所示。

将此功能开启后，给好友发送红包进行测试，指纹支付恢复正常。
```

### 5.5.4 Frida 进阶

```
掌握基本的Frida实战技术之后，再学习一些进阶技术对以后的逆向分析会有很大帮助。
Frida的出现就是为了解决烦琐的问题，本节所讲的内容甚至能作为一种定式，直接套用到今后的项目中。1.Python交互Frida提供了Python和JS脚本的交互，下面通过编写一些实用的功能代码来讲解它们的使用方法。
```

#### 1.Python 交互

```
1.Python交互
Frida提供了Python和JS脚本的交互，下面通过编写一些实用的功能代码来讲解它们的使用方法。
（1）获取设备
 DeviceManager类用于管理设备信息，
 可以枚举所有设备，
 也可以根据UDID获取某个设备。
 当然也有快速获取USB设备的方法，具体如下：
 
 if __name__ == '__main__':
   #获取设备管理器
   deviceManager = frida.get_device_manager()
   #枚举所有链接的设备
   print deviceManager.enumerate_devices()
   #根据UDID获取设备
   print deviceManager.get_device("xx666666")
   #获取当前USB连接的设备
   print frida.get_usb_device()
   
   输出如下：
   
   获取设备后，就可以对上面的应用进行操作了。
```

```
（2）附加进程
使用attach()附加目标进程，然后返回一个Session实例，参数支持进程名和PID，
具体如下：
if __name__ == '__main__':
   device = frida.get_usb_device()
   session = device.attch(u"微信")
   print session

输出如下：
  session(pid=2327)
```

```
（3）启动进程
使用spawn()启动进程，此时进程处于挂起状态，需要配合resume()才能唤醒。
这时还可以使用attach()附加上去，具体如下：
 if __name__ == '__main__':
    device = frida.get_usb_device()
    pid = device.spawn("com.tencent.xin")
    #session = device.attach(pid)
    device.resume(pid)
    
device.spawn()允许带参数执行，
例如通过下面的代码传入url参数后将会启动Safari浏览器并打开网站https://www.chinapyg.com：
 pid = device.spawn("com.apple.mobilesafari", url = "https://www.chinapyg.com"
```

```
（4）脱离进程得到Session并操作完毕后，需要使用detach()脱离进程，如下：
   if __name__ == '__main__':
    ...
    session.detach()
```

```
（5）注入JS脚本
一旦得到Session，就可以使用create_script()创建一个脚本对象，
然后调用load()方法将脚本载入，此时Python的使命完成，主要工作就交给JS了。

例如下面的一些实用功能。

● 枚举指定进程的所有模块信息：
  JS脚本可以直接使用一对"""符号嵌套在Python源文件中，
  也可以将JS脚本保存到专门的文件中再读取，上面的代码可以修改为：
  
● 获取应用的沙盒路径：

● 获取指定进程的指定模块信息。
 上面函数的JS脚本都是不需要传递参数的，那么某些功能需要参数时怎么办呢？
 Python与JS又如何交互呢？
 其实Script类可以设置回调函数，用来处理JS端传来的消息；
 而JS端设置recv()的回调来接收Python端的消息。具体看下面的例子：
 
 g_event是保证同步的，Python端使用script.post()将参数发送出去之后就调用g_event.wait()进入等待状态，当JS处理完成后，会将status设为“success”，
 再使用send()发送给Python端设置的on_message()回调，
 最终在payload_message()函数中完成逻辑处理，
 当识别到status为“success”后，调用g_event.set()使主线程继续执行。
 
执行结果如下：上面介绍的几个函数非常有用，可用在第7.2节替代vmmap来获取加载基址。
```

#### 2. 拦截某个类的所有方法

```
2.拦截某个类的所有方法
  如果想对某个类的所有方法进行批量拦截，该如何做呢？
  这个问题Frida团队早就想到了，他们在API里面提供了ApiResolver接口，它能够根据正则表达式获取符合条件的所有方法。
  例如获取微信红包相关的WCRedEnvelopesLogicMgr类的所有方法，JS代码如下：
  var resolver = new ApiResolver('objc');
  resolver.enumerateMatches('*[WCRedEnvelopesLogicMgr *]',{
     onMatch:funtion(match){
       console.log(match['name']+":"+match['address']);
     },
     onComplete:function(){}
  });
   
  运行脚本后，将WCRedEnvelopesLogicMgr类的所有方法及内存地址都打印出来了，如下：
   
   实际上，如果存在子类的话，也将被打印出来，读者可以自行尝试。
   获得所有方法后，就可以使用大家熟悉的Interceptor.attach()对每个方法进行拦截，然后打印参数和返回值了。代码如下：
   
   在拦截器中将调用的方法、传入的参数以及返回值都打印出来，这样在逆向分析函数调用顺序时就一目了然。
   由于接收微信红包的完整监控数据太多，这里就不列出了。
```

#### 3. 拦截 C 函数

```
3.拦截C函数
  在逆向分析时，经常会遇到sub_XXX这样的C函数（也可称为Native函数，后续不再特别指明），如果需要获取它的参数，除了采用动态调试方式以外，还可以用Frida来操作，具体如下：

  其中，“0x23adf6”是C函数的偏移地址，而“args[0].add(8)”相当于ARM指令的“LDRB W0, [X0,#8]”，也就是访问第一个成员变量。
  
  如果模块函数被导出，则可以使用
  Module.findExportByName(moduleName|null,exportName)或
  Module.getExportByName(moduleName|null, exportName)直接获取目标函数进行拦截。
  
  与上面等效的代码如下：
  
  其中，“_ZN4mars3stn13MMTLSCtrlInfo14IsMMTLSEnabledEv”是在IDA中看到的导出函数名称，注意前面的下画线不要丢了。
```

#### 4. 替换原方法

```
4.替换原方法

使用Interceptor.attach()拦截目标后，可以打印参数、修改返回值等，
但无法阻止原始方法的执行。
在进行逆向工程时，有时需要屏蔽某些方法的执行，减少一些不必要的干扰，从而快速定位切入点。
例如某个类存在一个update方法，原型如下：

当需要准确知道是否因为这个方法的执行而导致界面发生了改变时，可以将原方法替换掉进行测试：

如果需要在某个时刻调回原方法怎么办呢？代码如下：

 对于C函数，操作稍微复杂一些，需要使用NativeFunction定义原函数与新函数，然后使用Interceptor.replace()进行替换，具体如下：
```

#### 5.RPC 调用

```
5.RPC调用
Frida提供了RPC（远程过程调用）功能，开发人员可以将封装好的任意函数指定为RPC函数，提供给Python使用。
这个功能在调试过程中非常有用，比如在目标应用中存在一个加密函数，而开发者并不想分析它的实现，只是想调用它来得到结果，则可以将此加密函数导出，利用Python来调用它。
利用rpc.exports={}导出RPC函数，多个函数间用逗号分隔，示例如下：rpc.js文件内容如下：
   
rpc.py文件内容如下：

本例的JS脚本导出了openurl()和sound()两个RPC函数，当目标程序启动后，执行rpc.py脚本将rpc.js文件注入，如果成功就会启动Safari浏览器并打开网站https://www.chinapyg.com，同时播放系统声音。
```

#### 6. 小结

```
6.小结
Frida提供的API接口非常丰富，笔者介绍的一些方法只是冰山一角，仅做抛砖引玉之用。强烈建议大家阅读官方文档https://www.frida.re/docs/javascript-api，自行尝试使用其中的API，以便实现更加强大的功能。
脚本的意义就是简化逆向过程中的烦琐步骤，随着知识的积累，编写的脚本越来越多，逆向过程将会更加顺畅。

另外，在https://codeshare.frida.re/中可以找到很多共享的脚本，使用“$frida-codeshare”将其加载进来即可，读者也可以将自己的脚本传上去以便其他人使用。
```

5.6 本章小结
--------

```
本章分别介绍了iOS逆向工程中非常受欢迎的Cycript、Reveal、FLEX及Frida运行时分析工具，并从多种层面配合多个实例演示了它们的实际应用方法。

在这些工具中，Frida是目前功能最强大的，也是跨平台的，因此笔者用了较大的篇幅来讲解它的一些技巧，可以作为参考随时查看。
本章具有很强的实用性，而且没有任何复杂的操作，更不涉及任何汇编操作，是iOS逆向工程入门的最佳实践。
```