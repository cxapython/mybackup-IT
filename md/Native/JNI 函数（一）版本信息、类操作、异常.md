> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/Reverse-xiaoyu/p/14129052.html)

JNI 函数（一）版本信息、类操作、异常
====================

一、版本信息
------

 **GetVersion 返回本地方法接口的版本。**

**函数原型：jint GetVersion(JNIEnv* env);**

　　**参数**  
　　　　env：JNI 接口指针。

　　**返回值：**  
　　　　高 16 位返回主版本号，低 16 位返回次版本号。  
　　　　在 JDK1.1 中，GetVersion() 返回 0x00010001。

```
1 #define JNI_VERSION_1_1 0x00010001
2 #define JNI_VERSION_1_2 0x00010002
3 #define JNI_VERSION_1_4 0x00010004
4 #define JNI_VERSION_1_6 0x00010006
```

二、类操作
-----

###  **1、定义类：加载类**

**函数原型：jclass DefineClass(JNIEnv *env, const char* name, jobject loader, const jbyte *buf,  jsize bufLen)；**

　　这个函数，主要是从包含数据的 buffer 中加载类，该 buffer 包含类调用时未被虚拟机所引用的原始类数据。

　　**参数：**

　　　　env：JNI 接口指针

　　　　name：所定义的类名或者接口名，该字符串有 modefied UTF-8 编码

　　　　loader：指派给定义的类加载器

　　　　buf：包含 .class 文件数据的 buffer

　　　　bufLen：buffer 长度

　　**返回：**

　　　　Java 类对象，当错误出现时返回 NULL

　　**可能抛出的异常：**

　　　　如果没有指定这个 Java 类的，则会抛出 ClassFormatError

　　　　如果是一个类 / 接口是它自己的一个父类 / 父接口，则会抛出 ClassCircularityError

　　　　如果内存不足，则会抛出 OutOfMemoryError

　　　　如果想尝试在 Java 包中定义一个类，则会抛出 SecurityException

### 　　2、查找类

**函数原型：jclass FindClass(JNIEnv *env, const char *name);**

　　这里面有两种情况一个是 JDK release1.1，另外一种是 JDK release 1.2。

　　从 JDK release 1.1，该函数加载一个本地定义类，它搜索 CLASSPATH 环境变量里的目录及 zip 文件查找特定名字的类。

　　自从 Java 2 release 1.2，Java 安全模型允许非系统类加载跟调用本地方法。FindClass 定义与当前本地方法关联的类加载，也就是声明本地方法的类的类加载类。如果本地方法属于系统类，则不会涉及类加载器；否则，将调用适当的类加载来加载和链接指定的类。从 Java 2 SDK1.2 版本开始，通过调用接口调用 FindClass 时，没有当前的本机方法或关联的的类加载器。在这种情况下，在这种情况下，使用 **ClassLoader.getSystemClassLoader** 的结果。这是虚拟机为应用程序创建的类加载器，并且能够找到 java.class.path 属性列出的类。

　　**参数：**

　　　　env：JNI 接口指针

　　　　name：一个完全限定的类名，即包含 “包名”+“/” + 类名。举个例子：如 java.lang.String，该参数为 java/lang/String；如果类名以 **[** 开头，将返回一个数组类。比如数组类的签名为 java.lang.Object[]，该参数应该为 "[Ljava/lang/Object"

　　**返回：**

　　　　返回对应完全限定类对象，当找不到类时，返回 NULL

　　**可能抛出的异常：**

　　　　如果没有指定这个 Java 类的，则会抛出 ClassFormatError

　　　　如果是一个类 / 接口是它自己的一个父类 / 父接口，则会抛出 ClassCircularityError

　　　　如果没有找到该类 / 接口的定义，则抛出 NoClassDefFoundError

　　　　如果内存不足，则会抛出 OutOfMemoryError

### 　　3、查找父类

**函数原型：jclass GetSuperclass(JNIEnv *env, jclass clazz);**

　　如果 clazz 不是 Object 类，则此函数将返回表示该 clazz 的父类的 Class 对象，如果该类是 Object，或者 clazz 代表接口，则此函数返回 NULL。

　　**参数：**

　　　　env：JNI 接口指针

　　　　clazz：Java 的 Class 类

　　**返回：**

　　　　如果 clazz 有父类则返回其父类，如果没有其父类则返回 NULL

### 　　4、安全转换

**函数原型：jboolean IsAssignableFrom(JNIEnv *env, jclass clazz1, jclass clazz2);.**

　　判断 clazz1 的对象是否可以安全地转化为 clazz2 的对象

　　**参数：**

　　　　env：JNI 接口指针

　　　　clazz1：Java 的 Class 类，即需要被转化的类

　　　　clazz2：Java 的 Class 类，即需要转化为目标的类

　　**返回：**

　　　　如果满足以下任一条件，则返回 JNI_TRUE：

　　　　如果 clazz1 和 clazz2 是同一个 Java 类。

　　　　如果 clazz1 是 clazz2 的子类

　　　　如果 clazz1 是 clazz2 接口的实现类

三、异常
----

### 　　1、抛出异常

**函数原型：jint Throw(JNIEnv *env, jthrowable obj);**

　　传入一个 jthrowable 对象，并且在 JNI 并将其抛起

　　**参数：**

　　　　env：JNI 接口指针

　　　　jthrowable：一个 Java 的 java.lang.Throwable 对象

　　**返回：**

　　　　成功返回 0，失败返回一个负数。

　　**可能抛出的异常：**

　　抛出一个 java.lang.Throwable 对象

### 　　2、构造一个新的异常并抛出

**函数原型：jint ThrowNew(JNIEnv *env, jclass clazz, const char* message);**

　　传入一个 message，并用其构造一个异常并且抛出。

　　**参数：**

　　　　env：JNI 接口指针

　　　　jthrowable：一个 Java 的 java.lang.Throwable 对象

　　　　message：用于构造一个 java.lang.Throwable 对象的消息，该字符串用 modified UTF-8 编码

　　**返回：**

　　　　如果成功返回 0，失败返回一个负数

　　**可能抛出的异常：**

　　　　抛出一个新构造的 java.lang.Throwable 对象

### 　　3、检查是否发生异常，并抛出异常

**函数原型：jthrowable ExceptionOccurred(JNIEnv *env);**

　　检测是否发生了异常，如果发生了，则返回该异常的引用 (再调用 ExceptionClear() 函数前，或者 Java 处理异常前)，如果没有发生异常，则返回 NULL。

　　参数：

　　　　env：JNI 接口指针

　　返回：

　　　　jthrowable 的异常引用或者 NULL

### 　　4、打印异常的堆栈信息

**函数原型：void ExceptionDescribe(JNIEnv *env)**

　　打印这个异常的堆栈信息

　　**参数：**

　　　　env：JNI 接口指针

### 　　5、清除异常的堆栈信息

**函数原型：void ExceptionClear(JNIEnv *env);**

　　清除正在抛出的异常，如果当前没有异常被抛出，这个函数不起作用

　　**参数：**

　　　　env：JNI 接口指针

### 　　6、致命异常

**函数原型：void FatalError(JNIEnv *env, const char* msg);**

　　致命异常，用于输出一个异常信息，并终止当前 VM 实例，即退出程序。

　　**参数：**

　　　　env：JNI 接口指针

　　　　msg：异常的错误信息，该字符串用 modified UTF-8 编码

### 　　7、仅仅检查是否发生异常

**函数原型：jboolean ExceptionCheck(JNIEnv *env);**

　　检查是否已经发生了异常，如果已经发生了异常，则返回 JNI_TRUE，否则返回 JNI_FALSE

　　参数：

　　　　env：JNI 接口指针

　　返回：

　　　　如果已经发生异常，返回 JNI_TRUE，如果没有发生异常则返回 JNI_FALSE