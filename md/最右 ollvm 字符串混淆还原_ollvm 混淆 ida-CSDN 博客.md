> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/wang_624/article/details/115066838)

某 apk so 层 ollvm 字符串混淆
----------------------

> 本次逆向分析用到的工具：[adb](https://so.csdn.net/so/search?q=adb&spm=1001.2101.3001.7020)、ida、010Editor、ddms。

这次主要对某右的`libnet_crypto.so`分析, 主要工作是分析`ollvm`混淆的字符串被处理加密。

先检测一下设备是否正常

`adb devices`

![](https://img-blog.csdnimg.cn/20210322104004674.png)

然后解压 apk 文件，提取出 lib 目录下的`libnet_crypto.so`文件

![](https://img-blog.csdnimg.cn/20210322104014354.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

`armeabi-v7a`对应的是 32 位的 ARM 设备，调试使用`IDA`，不要用`IDA64`

`so`文件挺大的，编译需要等待一会，看一下字符串，基本都是混淆加密的  
![](https://img-blog.csdnimg.cn/20210322104024675.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

接下来，分析一下`JNI_onload`

先看一下`JNI_onload`流程，不是很复杂  
![](https://img-blog.csdnimg.cn/20210322104044462.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

在`F5`或者`空格`查看伪代码，代码没什么混淆

![](https://img-blog.csdnimg.cn/20210322104057306.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

修改参数, 静态分析方法

右键选择`set lvar type`或`y`修改参数类型  
![](https://img-blog.csdnimg.cn/20210322104113519.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)  
在下面`GetEnv`处 选择`Force call type`识别一下类型![](https://img-blog.csdnimg.cn/20210322104158221.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

`v4`是`JNIenv *` 也修改一下

![](https://img-blog.csdnimg.cn/2021032210421949.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

下面 `FindClass`和`Registernatives`也一下，识别一下类型

点进去查看，没有`native`方法的对应关系，第一个参数是`java`层`native`的方法名称，第二个参数是`java`层`native`的签名信息，第三个参数才是对应的`c\c++`的参数

![](https://img-blog.csdnimg.cn/202103221042316.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

根据上面静态分析遇到的问题，接下来的想法是恢复被 ollvm 加密混淆的字符串信息

打开`010Editor`，将`so`文件和`elf`头文件拖进去

![](https://img-blog.csdnimg.cn/20210322104241673.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

主要分析数据区域

第一个是程序头，不是关注的，第二个和第三个是关注重点，第二个是只读的，那么根据 so 文件对字符串进行加解密可以判断`ida`编译出的混淆字符串属于第三个区域  
![](https://img-blog.csdnimg.cn/20210322104254329.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

这部分区域是被加密处理的

![](https://img-blog.csdnimg.cn/20210322104302697.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

那对于这个`so`的还原，就需要`dump`出解密后的数据，将数据复制到原加密数据位置，完成对字符串解密。

`dump`的位置可能在`JNI_onload` 、 `init` 、 `init_aray`

`ida` 附加一下进程，看一下`so`文件是否被加载

先进行一下`android`端的配置

找到`ida`的`android_server`  
![](https://img-blog.csdnimg.cn/20210322104313621.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

将`ida`的`android_server`发送到`android`

![](https://img-blog.csdnimg.cn/20210322104324508.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

执行`android_server`

转发端口  
![](https://img-blog.csdnimg.cn/20210322104332751.png)

附加端口，这里先要启动 apk

`Debugger`-> `Attach` -> `Remote ARmlinux\Android debugger`

![](https://img-blog.csdnimg.cn/20210322104342143.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

在`Modules`发现`so`文件已经被加载进去了  
![](https://img-blog.csdnimg.cn/20210322104357774.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

那么直接搜索`dump`内存中解密的字符串数据  
![](https://img-blog.csdnimg.cn/20210322104413629.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

`ctrl+s`搜 `crypto`

`File`->`script commond`

然后输入命令，替换初始地址和结束地址

![](https://img-blog.csdnimg.cn/20210322104432615.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

使用`010Editor`打开，里面的字符串都是解密的

![](https://img-blog.csdnimg.cn/20210322104444650.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

这是加密的数据，可以对比一下，前六行基本是一致的都是地址。最后一行都是 00，这里是相对地址

![](https://img-blog.csdnimg.cn/20210322104455806.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

这个是解密的数据，前六行都是 04 这里是绝对地址。  
![](https://img-blog.csdnimg.cn/20210322104503348.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

复制解密后的数据，到这里为止

![](https://img-blog.csdnimg.cn/a72beb44525a436ea50896dc736a3ecc.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAU1QwbmV3,size_20,color_FFFFFF,t_70,g_se,x_16)

在鼠标放到如图位置，`ctrl+v`搞定

![](https://img-blog.csdnimg.cn/20210322104528156.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

看一下修复后的`so`文件

看一下`Findclass`, 之前混淆的现在已经解密了

![](https://img-blog.csdnimg.cn/2021032210454131.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)  
同样`RegisterNatives`也是，之前不知道方法的类型和参数，现在都已经解密出来了![](https://img-blog.csdnimg.cn/20210322104557948.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)

### 总结

这种方法仅适用在内存中解密状态的情况，如果是在使用时解密，解密后在清除，需要使用动态调试。

欢迎关注我的公众号 了解最新文章

![](https://img-blog.csdnimg.cn/20210322104704303.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdfNjI0,size_16,color_FFFFFF,t_70)