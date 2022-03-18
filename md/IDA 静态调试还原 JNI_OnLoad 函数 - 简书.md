> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/c423e5fefd5e)

1. 找到 JNI_OnLoad 函数
===================

![](http://upload-images.jianshu.io/upload_images/3298299-0d57ea45362e0ea2.png) 图片. png

![](http://upload-images.jianshu.io/upload_images/3298299-1665923859e4b1bf.png) 图片. png

2. 识别成函数
========

按 p

![](http://upload-images.jianshu.io/upload_images/3298299-f70952f31fd59aa5.png) 图片. png

![](http://upload-images.jianshu.io/upload_images/3298299-742607107bf1988e.png) 图片. png

再按 F5

![](http://upload-images.jianshu.io/upload_images/3298299-4bb27055913ff482.png) 图片. png

![](http://upload-images.jianshu.io/upload_images/3298299-12ab2e1cbca79d0f.png) 图片. png

注：很多时候直接按 F5 就能直接识别出函数

3. 改参数
======

在 Cpp 文件中一个完整 JNI_OnLoad 函数是这样的

![](http://upload-images.jianshu.io/upload_images/3298299-af36a8b360c9fde1.png) 图片. png

只需要把关键地方的函数还原就行

1. 改 JNI_OnLoa 函数的参数
--------------------

![](http://upload-images.jianshu.io/upload_images/3298299-f4abe1b1f4e888a4.png) 图片. png

改成

![](http://upload-images.jianshu.io/upload_images/3298299-073ee776f855ad01.png) 图片. png

按 xx.cpp 中的 JNI_OnLoad 函数模型来改

![](http://upload-images.jianshu.io/upload_images/3298299-081a2a6ac62030ac.png) 图片. png

2. 改 GetEnv 函数
--------------

参照原函数来改

![](http://upload-images.jianshu.io/upload_images/3298299-b15b328d383b3c6d.png) 图片. png

第一个参数是 vm，在 cpp 代码里是默认参数不显示  
第二个参数是 JNIEnv 类型  
第三个参数是 JNI 版本，这个不用改  
在 IDA 里是这样显示

![](http://upload-images.jianshu.io/upload_images/3298299-a1eb017f5dfb3795.png) 图片. png

先获取类型

![](http://upload-images.jianshu.io/upload_images/3298299-d77738d231c24f45.png) 图片. png

![](http://upload-images.jianshu.io/upload_images/3298299-b8639a4083efa950.png) 图片. png

改二个参数类型

![](http://upload-images.jianshu.io/upload_images/3298299-ac6b4b0465e04e56.png) 图片. png

![](http://upload-images.jianshu.io/upload_images/3298299-08ff3b04a230ccda.png) 图片. png

改完后

![](http://upload-images.jianshu.io/upload_images/3298299-e6633bad79a45ef8.png) 图片. png

这里把 v62 赋值 v5, 接收的 v5 类型也改成 JNIEnv *

识别成功后, FindClass 和 RegisterNatives 函数也出来了

![](http://upload-images.jianshu.io/upload_images/3298299-edf5d6dcc59212ee.png) 图片. png

3. 改 FindClass
--------------

![](http://upload-images.jianshu.io/upload_images/3298299-0a55a0be40efd334.png) 图片. png

4. 改 RegisterNatives
--------------------

![](http://upload-images.jianshu.io/upload_images/3298299-1802a1fb2fb380d3.png) 图片. png

这个参数

![](http://upload-images.jianshu.io/upload_images/3298299-d4e8f4dcf6d098e9.png) 图片. png

就是 java 方法与 C 的函数对应表

![](http://upload-images.jianshu.io/upload_images/3298299-432c3f298c9b00a2.png) 图片. png

4. 找到动态注册表
==========

点击 g_methods 函数所在的地址，也就是上一步在 IDA 里识别到的第三个参数 off_A9094

![](http://upload-images.jianshu.io/upload_images/3298299-fdb579f5b39f959b.png) 图片. png

动态注册表就在这里

![](http://upload-images.jianshu.io/upload_images/3298299-68cb48b2d8f26015.png) 图片. png

从上到下依次是 函数名，函数参数和返回值类型，函数所在内存地址

![](http://upload-images.jianshu.io/upload_images/3298299-9d3756a9a70282dd.png) 图片. png

点击函数地址

![](http://upload-images.jianshu.io/upload_images/3298299-acf5146bf6032b3c.png) 图片. png

进入到对应的函数

![](http://upload-images.jianshu.io/upload_images/3298299-3c8af7402870a10e.png) 图片. png

点 F5 识别成函数

![](http://upload-images.jianshu.io/upload_images/3298299-329d7c45f00e6b1c.png) 图片. png