> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/wxyyxc1992/article/details/17320911)

自己重新做了文章，希望能更好的交流，若有任何冒犯请及时与我联系并致诚挚歉意
=====================================

如果你是 IT 新人，可以看看笔者的我的编程之路，详细的知识点整理：https://segmentfault.com/a/1190000004612590  

================================================================================

1 Introduction
==============

1.1 概述
------

Xposed 是 GitHUB 上 rovo89 大大设计的一个针对 Android 平台的动态劫持项目，通过替换 /system/bin/app_process 程序控制 zygote 进程，使得 app_process 在启动过程中会加载 XposedBridge.jar 这个 jar 包，从而完成对 Zygote 进程及其创建的 Dalvik 虚拟机的劫持。与采取传统的 Inhook 方式（详见 Dynamic Dalvik Instrumentation 分析这篇本章 ）相比，Xposed 在开机的时候完成对所有的 Hook Function 的劫持，在原 Function 执行的前后加上自定义代码。

Xposed 框架的基本运行环境如下：

<table border="1" cellpadding="1" cellspacing="1" width="800"><tbody><tr><td><p>Configuration</p></td><td><p>RequireMent</p></td></tr><tr><td><p>Root&nbsp;Access</p></td><td><p>因为 Xposed 工作原理是在 /system/bin 目录下替换文件，在 install 的时候需要 root 权限，但是运行时不需要 root 权限。</p></td></tr><tr><td><p>版本要求</p></td><td><p>需要在 Android&nbsp;4.0 以上版本的机器中</p></td></tr></tbody></table>  

### 1.1.1 GitHub 上 Xposed 资源梳理

*   XposedBridge.jar：XposedBridge.jar 是 Xposed 提供的 jar 文件，负责在 Native 层与 FrameWork 层进行交互。/system/bin/app_process 进程启动过程中会加载该 jar 包，其它的 Modules 的开发与运行都是基于该 jar 包的。注意：XposedBridge.jar 文件本质上是由 XposedBridge 生成的 APK 文件更名而来，有 图为证：install.sh![](https://img-blog.csdn.net/20131222210248171?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvd3h5eXhjMTk5Mg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
*    Xposed：Xposed 的 C++ 部分，主要是用来替换 /system/bin/app_process，并为 XposedBridge 提供 JNI 方法。
*    XposedInstaller：Xposed 的安装包，负责配置 Xposed 工作的环境并且提供对基于 Xposed 框架的 Modules 的管理。在安装 XposedInstaller 之后，app_process 与 XposedBridge.jar 放置在了 /data/data/de.robv.android.xposed.installer。
*    XposedMods：使用 Xposed 开发的一些 Modules，其中 AppSettings 是一个可以进行权限动态管理的应用

1.2 Mechanism：原理
----------------

### 1.2.1 Zygote

在 Android 系统中，应用程序进程都是由 Zygote 进程孵化出来的，而 Zygote 进程是由 Init 进程启动的。Zygote 进程在启动时会创建一个 Dalvik 虚拟机实例，每当它孵化一个新的应用程序进程时，都会将这个 Dalvik 虚拟机实例复制到新的应用程序进程里面去，从而使得每一个应用程序进程都有一个独立的 Dalvik 虚拟机实例。这也是 Xposed 选择替换 app_process 的原因。

Zygote 进程在启动的过程中，除了会创建一个 Dalvik 虚拟机实例之外，还会将 Java 运行时库加载到进程中来，以及注册一些 Android 核心类的 JNI 方法来前面创建的 Dalvik 虚拟机实例中去。注意，一个应用程序进程被 Zygote 进程孵化出来的时候，不仅会获得 Zygote 进程中的 Dalvik 虚拟机实例拷贝，还会与 Zygote 一起共享 Java 运行时库。这也就是可以将 XposedBridge 这个 jar 包加载到每一个 Android 应用程序中的原因。XposedBridge 有一个私有的 Native（JNI）方法 hookMethodNative，这个方法也在 app_process 中使用。这个函数提供一个方法对象利用 Java 的 Reflection 机制来对内置方法覆写。具体的实现可以看下文的 Xposed 源代码分析。

### 1.2.2 Hook/Replace

Xposed 框架中真正起作用的是对方法的 hook。在 Repackage 技术中，如果要对 APK 做修改，则需要修改 Smali 代码中的指令。而另一种动态修改指令的技术需要在程序运行时基于匹配搜索来替换 smali 代码，但因为方法声明的多样性与复杂性，这种方法也比较复杂。

在 Android 系统启动的时候，zygote 进程加载 XposedBridge 将所有需要替换的 Method 通过 JNI 方法 hookMethodNative 指向 Native 方法 xposedCallHandler，xposedCallHandler 在转入 handleHookedMethod 这个 Java 方法执行用户规定的 Hook Func。

XposedBridge 这个 jar 包含有一个私有的本地方法：hookMethodNative，该方法在附加的 app_process 程序中也得到了实现。它将一个方法对象作为输入参数（你可以使用 Java 的反射机制来获取这个方法）并且改变 Dalvik 虚拟机中对于该方法的定义。它将该方法的类型改变为 native 并且将这个方法的实现链接到它的本地的通用类的方法。换言之，当调用那个被 hook 的方法时候，通用的类方法会被调用而不会对调用者有任何的影响。在 hookMethodNative 的实现中，会调用 XposedBridge 中的 handleHookedMethod 这个方法来传递参数。handleHookedMethod 这个方法类似于一个统一调度的 Dispatch 例程，其对应的底层的 C++ 函数是 xposedCallHandler。而 handleHookedMethod 实现里面会根据一个全局结构 hookedMethodCallbacks 来选择相应的 hook 函数，并调用他们的 before, after 函数。

当多模块同时 Hook 一个方法的时候，Xposed 会自动根据 Module 的优先级来排序，调用顺序如下：

A.before -> B.before -> original method -> B.after -> A.after

2 源代码分析
=======

本部分参考了看雪论坛某大神的文章，对于上一篇文章中代码分析部分几乎照搬了他的文章深表歉意。最近一直在 reading and fucking the code，个人感觉流程方式叙述函数调用虽然调理比较明晰，但是还是较复杂，本文中采取面向函数 / 对象的介绍方式，在函数的介绍顺序上采取流程式。如果对于某个函数 / 方法有疑问的直接 Ctrl+F 在本文内搜索。

2.1 Application：XposedInstaller
-------------------------------

XposedInstaller 负责对 Xposed 运行环境的安装配置与 Module 的管理，无论对于开发者还是普通用户而言，都是接触到的第一个应用。另一方面，如果我们需要抛弃 Xposed 本来的 Module 管理机制，改编为我们自己的应用，也需要了解 XposedInstaller 的基本流程。

### 2.1.1 InstallerFragment

private String install();                         @function 执行 app_process 的替换与 XposedBridge.jar 的写入操作。  

2.2 Cpp:app_main.cpp 
---------------------

正如第一部分所言，Xposed 会在 Android 系统启动的时候加载 XposedBridge.jar 并进行 Function 的替换操作。我们首先从 CPP 模块的 Zygote 进程的源代码 app_main.cpp 谈起。类似 AOSP 中的 frameworks/base/cmds/app_process/app_main.cpp，即 /system/bin/app_process 这个 zygote 真实身份的应用程序的源代码。关于 zygote 进程的分析可以参照 Android:AOSP&Core 中的 Zygote 进程详解。

### 2.2.1 int main(int argc, char* const argv[])

Zygote 进程首先从 Main 函数开始执行。

```
int main(int argc, char* const argv[])
{
	...
	/**
	@function 该函数对于SDK>=18的会获取到atrace_set_tracing_enabled的函数指针，获取到的指针会在Zygote初始化过程中调用，函数定义见代码段下方
	*/
	initTypePointers();
	...
	/**
	@function 该函数主要获取一些属性值譬如SDK版本，设备厂商，设备型号等信息并且打印到Log文件中
	@description xposedInfo函数定义在xposed.cpp中
	*/
	xposedInfo();
	//变量标志是否需要加载Xposed框架
    keepLoadingXposed =
	
	/**
	@function 该函数主要分析Xposed框架是否被禁用了。
	@description 该函数定义在了xposed.cpp中
	*/
	!isXposedDisabled() 
	&& 
	!xposedShouldIgnoreCommand(className, argc, argv) 
	&& 
	/**
	@function 将XposedBridge所在路径加入到系统环境变量CLASSPATH中
	@para zygote bool类型，描述是否需要重启zygote，参数来源于启动指令“--zygote”参数
	@description 该函数定义在了xposed.cpp中
	*/
	addXposedToClasspath(zygote);
	
	/**
	@annotation 1
	*/
    if (zygote) {
        runtime.start(keepLoadingXposed ? XPOSED_CLASS_DOTS : "com.android.internal.os.ZygoteInit",
                startSystemServer ? "start-system-server" : "");
    } else if (className) {
        // Remainder of args get passed to startup class main()
        runtime.mClassName = className;
        runtime.mArgC = argc - i;
        runtime.mArgV = argv + i;
        runtime.start(keepLoadingXposed ? XPOSED_CLASS_DOTS : "com.android.internal.os.RuntimeInit",
                application ? "application" : "tool");
    } 
	else 
	{
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
}
```

annotation 1

在 annotation 1 之后，开始执行 Dalvik 启动的操作。一般情况下 keepLoadingXposed 值为 true，以启动 Zygote 为例 (zygote==true)，分析接下来的代码。如果 zygote 为 false，即是启动普通 Application 所属 Dalvik 的情景。

这一行代码是根据 keepLoadingXposed 的值来判断是加载 Xposed 框架还是正常的 ZygoteInit 类。keepLoadingXposed 值为 true, 则会加载 XPOSED_CLASS_DOTS 类，XPOSED_CLASS_DOTS 值为 de.robv.android.xposed.XposedBridge，即 XposedBridge 类。runtime 是 AppRuntime 的实例，AppRuntime 继承自 AndroidRuntime。

```
...
AppRuntime runtime;
...
/**
@function AppRuntime类的方法，用于启动Dalvik虚拟机。
@description 定义于app_main.cpp中
*/
runtime.start(keepLoadingXposed ? XPOSED_CLASS_DOTS : "com.android.internal.os.RuntimeInit",
　　                application ? "application" : "tool");
```

### 2.2.2 void initTypePointers()

```
/**
@function 该函数对于SDK>=18的会获取到atrace_set_tracing_enabled的函数指针，获取到的指针会在Zygote初始化过程中调用
*/
void initTypePointers()
{
    char sdk[PROPERTY_VALUE_MAX];
    const char *error;
    property_get("ro.build.version.sdk", sdk, "0");
    RUNNING_PLATFORM_SDK_VERSION = atoi(sdk);
    dlerror();
    if (RUNNING_PLATFORM_SDK_VERSION >= 18) {
        *(void **) (&PTR_atrace_set_tracing_enabled) = dlsym(RTLD_DEFAULT, "atrace_set_tracing_enabled");
        if ((error = dlerror()) != NULL) {
            ALOGE("Could not find address for function atrace_set_tracing_enabled: %s", error);
        }
    }
}
```

### 2.2.3 AppRuntime 

```
......
static AndroidRuntime* gCurRuntime = NULL;
......
AndroidRuntime::AndroidRuntime()
{
	......
	assert(gCurRuntime == NULL);        // one per process
	gCurRuntime = this;
}
```

#### 2.2.3.1 void AndroidRuntime::start(const char* className, const bool startSystemServer)

AppRuntime 继承自 AndroidRuntime，其自身没有 override start 方法，则在 AppRuntime 中调用的还是其父类的 start 方法。AndroidRuntime::start(const char* className, const char* options) 函数完成 Dalvik 虚拟机的初始化和启动以及运行参数 className 指定的类中的 main 方法。当启动完虚拟机后，会调用 onVmCreated(JNIEnv* env) 函数。该函数在 AppRuntime 类中被覆盖。因此直接看 AppRuntime::onVmCreated(JNIEnv* env)。

```
void AndroidRuntime::start(const char* className, const bool startSystemServer)
{
	......
	char* slashClassName = NULL;
	char* cp;
	JNIEnv* env;
	......
	/* start the virtual machine */
	if (startVm(&mJavaVM, &env) != 0)
		goto bail;
	/*
	* Register android functions.
	*/
	if (startReg(env) < 0) {
		LOGE("Unable to register all android natives\n");
		goto bail;
	}
	/*
	* We want to call main() with a String array with arguments in it.
	* At present we only have one argument, the class name.  Create an
	* array to hold it.
	*/
	jclass stringClass;
	jobjectArray strArray;
	jstring classNameStr;
	jstring startSystemServerStr;
	stringClass = env->FindClass("java/lang/String");
	assert(stringClass != NULL);
	strArray = env->NewObjectArray(2, stringClass, NULL);
	assert(strArray != NULL);
	classNameStr = env->NewStringUTF(className);
	assert(classNameStr != NULL);
	env->SetObjectArrayElement(strArray, 0, classNameStr);
	startSystemServerStr = env->NewStringUTF(startSystemServer ?
		"true" : "false");
	env->SetObjectArrayElement(strArray, 1, startSystemServerStr);
	/*
	* Start VM.  This thread becomes the main thread of the VM, and will
	* not return until the VM exits.
	*/
	jclass startClass;
	jmethodID startMeth;
	slashClassName = strdup(className);
	for (cp = slashClassName; *cp != '\0'; cp++)
		if (*cp == '.')
			*cp = '/';
	startClass = env->FindClass(slashClassName);
	if (startClass == NULL) {
		......
	} else {
		startMeth = env->GetStaticMethodID(startClass, "main",
			"([Ljava/lang/String;)V");
		if (startMeth == NULL) {
			......
		} else {
			env->CallStaticVoidMethod(startClass, startMeth, strArray);
			......
		}
	}
	......
}
```

如果对 Zygote 启动过程熟悉的话，对后续 XposedBridge 的 main 函数是如何被调用的应该会很清楚。AndroidRuntime.cpp 的 start(const char* className, const char* options) 函数完成环境变量的设置，Dalvik 虚拟机的初始化和启动，同时 Xposed 在 onVmCreated(JNIEnv* env) 中完成了自身 JNI 方法的注册。此后 start() 函数会注册 Android 系统的 JNI 方法，调用传入的 className 指定类的 main 方法，进入 Java 世界。

由于此时参数 className 为 de.robv.android.xposed.XposedBridge，因此会调用 XposedBridge 类的 main 方法。

#### 2.2.3.2 virtual void onVmCreated(JNIEnv* env)

```
virtual void onVmCreated(JNIEnv* env)
　　{
　　    /**
　　    @function 用于加载XposedBridge.jar这个类，本函数只有这个函数是增加的
　　    @para JNI环境指针
　　    @para mClassName runtime.mClassName = className这句指定
　　    @description 定义于xposed.cpp中
　　    */
        keepLoadingXposed = xposedOnVmCreated(env, mClassName);
        if (mClassName == NULL) {
            return; // Zygote. Nothing to do here.
        }
        char* slashClassName = toSlashClassName(mClassName);
        mClass = env->FindClass(slashClassName);
        if (mClass == NULL) {
            ALOGE("ERROR: could not find class '%s'\n", mClassName);
        }
        free(slashClassName);
        mClass = reinterpret_cast<jclass>(env->NewGlobalRef(mClass));
    }
```

2.3 Cpp:xposed.cpp
------------------

### 2.3.1 isXposedDisabled

这个函数负责来判断 Xposed 框架是否为启用，如果不小心 Module 写错了，可以通过 shell 手动创建文件来防止手机鸟掉。

```
/**
@function 判断Xposed框架是否被激活 
*/
bool isXposedDisabled() {
    //该函数通过读取/data/data/de.robv.android.xposed.installer/conf/disabled文件来判断Xposed框架是否被禁用，如果该文件存在，则表示禁用Xposed。
    if (access(XPOSED_LOAD_BLOCKER, F_OK) == 0) {
        ALOGE("found %s, not loading Xposed\n", XPOSED_LOAD_BLOCKER);
        return true;
    }
    return false;
}
```

### 2.3.2 xposedShouldIgnoreCommand

这个函数写的还是极好的，可以看看来增加点见识。

```
/**
@function 为了避免Superuser类似工具滥用Xposed的log文件，此函数会判断是否是SuperUser等工具的启动请求。
*/
bool xposedShouldIgnoreCommand(const char* className, int argc, const char* const argv[]) {
    if (className == NULL || argc < 4 || strcmp(className, "com.android.commands.am.Am") != 0)
        return false;
    if (strcmp(argv[2], "broadcast") != 0 && strcmp(argv[2], "start") != 0)
        return false;
    bool mightBeSuperuser = false;
    for (int i = 3; i < argc; i++) {
        if (strcmp(argv[i], "com.noshufou.android.su.RESULT") == 0
         || strcmp(argv[i], "eu.chainfire.supersu.NativeAccess") == 0)
            return true;
        if (mightBeSuperuser && strcmp(argv[i], "--user") == 0)
            return true;
        char* lastComponent = strrchr(argv[i], '.');
        if (!lastComponent)
            continue;
        if (strcmp(lastComponent, ".RequestActivity") == 0
         || strcmp(lastComponent, ".NotifyActivity") == 0
         || strcmp(lastComponent, ".SuReceiver") == 0)
            mightBeSuperuser = true;
    }
    return false;
　　}
```

### 2.3.3 addXposedToClasspath

若有新版本的 XposedBridge，重命名为 XposedBridge.jar 并返回 false; 判断 XposedBridge.jar 文件是否存在，若不存在，返回 false，否则将 XposedBridge.jar 所在路径添加到 CLASSPATH 环境变量中，返回 true。

```
/**
@function 将XposedBridge.jar添加到CLASSPATH
*/
bool addXposedToClasspath(bool zygote) {
    ALOGI("-----------------\n");
    // do we have a new version and are (re)starting zygote? Then load it!
    if (zygote && access(XPOSED_JAR_NEWVERSION, R_OK) == 0) {
        ALOGI("Found new Xposed jar version, activating it\n");
        if (rename(XPOSED_JAR_NEWVERSION, XPOSED_JAR) != 0) {
            ALOGE("could not move %s to %s\n", XPOSED_JAR_NEWVERSION, XPOSED_JAR);
            return false;
        }
    }
    if (access(XPOSED_JAR, R_OK) == 0) {
        char* oldClassPath = getenv("CLASSPATH");
        if (oldClassPath == NULL) {
            setenv("CLASSPATH", XPOSED_JAR, 1);
        } else {
            char classPath[4096];
            sprintf(classPath, "%s:%s", XPOSED_JAR, oldClassPath);
            setenv("CLASSPATH", classPath, 1);
        }
        ALOGI("Added Xposed (%s) to CLASSPATH.\n", XPOSED_JAR);
        return true;
    } else {
        ALOGE("ERROR: could not access Xposed jar '%s'\n", XPOSED_JAR);
        return false;
    }
}
```

### 2.3.4 bool xposedOnVmCreated(JNIEnv* env, const char* className)

```
bool xposedOnVmCreated(JNIEnv* env, const char* className) {
    if (!keepLoadingXposed)
        return false;
    //将要启动的Class的Name赋值给startClassName    
    startClassName = className;
	/**
	@function 根据JIT是否存在对部分结构体中的成员偏移进行初始化。
	@description 定义在xposed.cpp中。关于JIT的介绍详见我的关于Dalvik虚拟机的介绍
	*/
    xposedInitMemberOffsets();
 
    //下面开始对部分访问检查设置为true
    patchReturnTrue((void*) &dvmCheckClassAccess);
    patchReturnTrue((void*) &dvmCheckFieldAccess);
    patchReturnTrue((void*) &dvmInSamePackage);
    if (access(XPOSED_DIR "conf/do_not_hook_dvmCheckMethodAccess", F_OK) != 0)
        patchReturnTrue((void*) &dvmCheckMethodAccess);
	//针对MIUI操作系统移除android.content.res.MiuiResources类的final修饰符
    jclass miuiResourcesClass = env->FindClass(MIUI_RESOURCES_CLASS);
    if (miuiResourcesClass != NULL) {
        ClassObject* clazz = (ClassObject*)dvmDecodeIndirectRef(dvmThreadSelf(), miuiResourcesClass);
        if (dvmIsFinalClass(clazz)) {
            ALOGD("Removing final flag for class '%s'", MIUI_RESOURCES_CLASS);
            clazz->accessFlags &= ~ACC_FINAL;
        }
    }
    env->ExceptionClear();
	//开始获取XposedBridge类并new一个新的全局引用
    xposedClass = env->FindClass(XPOSED_CLASS);
    xposedClass = reinterpret_cast<jclass>(env->NewGlobalRef(xposedClass));
    
    if (xposedClass == NULL) {
        ALOGE("Error while loading Xposed class '%s':\n", XPOSED_CLASS);
        dvmLogExceptionStackTrace();
        env->ExceptionClear();
        return false;
    }
    
    ALOGI("Found Xposed class '%s', now initializing\n", XPOSED_CLASS);
	/**
	@function 开始注册XposedBridge中JNI函数
	*/
    register_de_robv_android_xposed_XposedBridge(env);
    register_android_content_res_XResources(env);
    return true;
}
```

### 2.3.5 JNI 函数注册区域

```
static const JNINativeMethod xposedMethods[] = {
    {"getStartClassName", "()Ljava/lang/String;", (void*)de_robv_android_xposed_XposedBridge_getStartClassName},
    {"initNative", "()Z", (void*)de_robv_android_xposed_XposedBridge_initNative},
    {"hookMethodNative", "(Ljava/lang/reflect/Member;Ljava/lang/Class;ILjava/lang/Object;)V", (void*)de_robv_android_xposed_XposedBridge_hookMethodNative},
};
static jobject de_robv_android_xposed_XposedBridge_getStartClassName(JNIEnv* env, jclass clazz) {
    return env->NewStringUTF(startClassName);
}
static int register_de_robv_android_xposed_XposedBridge(JNIEnv* env) {
    return env->RegisterNatives(xposedClass, xposedMethods, NELEM(xposedMethods));
}
static const JNINativeMethod xresourcesMethods[] = {
    {"rewriteXmlReferencesNative", "(ILandroid/content/res/XResources;Landroid/content/res/Resources;)V", (void*)android_content_res_XResources_rewriteXmlReferencesNative},
};
static int register_android_content_res_XResources(JNIEnv* env) {
    return env->RegisterNatives(xresourcesClass, xresourcesMethods, NELEM(xresourcesMethods));
}
```

### 2.3.6 initNative

该函数主要完成对 XposedBridge 类中函数的引用，这样可以实现在 Native 层对 Java 层函数的调用。譬如获取 XposedBridge 类中的 handlHookedMethod 函数的 method id，同时赋值给全局变量 xposedHandleHookedMethod。另外，initNative 函数还会获取 android.content.res.XResources 类中的方法，完成对资源文件的处理；调用 register_android_content_res_XResources 注册 rewriteXmlReferencesNative 这个 JNI 方法。

```
static jboolean de_robv_android_xposed_XposedBridge_initNative(JNIEnv* env, jclass clazz) {
	...
    xposedHandleHookedMethod = (Method*) env->GetStaticMethodID(xposedClass, "handleHookedMethod",
        "(Ljava/lang/reflect/Member;ILjava/lang/Object;Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;");
	...
    xresourcesClass = env->FindClass(XRESOURCES_CLASS);
    xresourcesClass = reinterpret_cast<jclass>(env->NewGlobalRef(xresourcesClass));
	...
    if (register_android_content_res_XResources(env) != JNI_OK) {
        ALOGE("Could not register natives for '%s'\n", XRESOURCES_CLASS);
        return false;
    }
    xresourcesTranslateResId = env->GetStaticMethodID(xresourcesClass, "translateResId",
        "(ILandroid/content/res/XResources;Landroid/content/res/Resources;)I");
	...
    xresourcesTranslateAttrId = env->GetStaticMethodID(xresourcesClass, "translateAttrId",
        "(Ljava/lang/String;Landroid/content/res/XResources;)I");
    ...
    return true;
}
```

### 2.3.7 hookMethodNative

```
/**
@function hookMethodNative 将输入的Class中的Method方法的nativeFunc替换为xposedCallHandler
@para declaredClassIndirect 类对象
@para slot Method在类中的偏移位置
*/
static void de_robv_android_xposed_XposedBridge_hookMethodNative(JNIEnv* env, jclass clazz, jobject declaredClassIndirect, jint slot) {
    // Usage errors?
    if (declaredClassIndirect == NULL) {
        dvmThrowIllegalArgumentException("declaredClass must not be null");
        return;
    }
    
    // Find the internal representation of the method
    ClassObject* declaredClass = (ClassObject*) dvmDecodeIndirectRef(dvmThreadSelf(), declaredClassIndirect);
    Method* method = dvmSlotToMethod(declaredClass, slot);
    if (method == NULL) {
        dvmThrowNoSuchMethodError("could not get internal representation for method");
        return;
    }
    
    if (findXposedOriginalMethod(method) != xposedOriginalMethods.end()) {
        // already hooked
        return;
    }
    
    // Save a copy of the original method
    xposedOriginalMethods.push_front(*((MethodXposedExt*)method));
    // Replace method with our own code
    SET_METHOD_FLAG(method, ACC_NATIVE);
    method->nativeFunc = &xposedCallHandler;
    method->registersSize = method->insSize;
    method->outsSize = 0;
    if (PTR_gDvmJit != NULL) {
        // reset JIT cache
        MEMBER_VAL(PTR_gDvmJit, DvmJitGlobals, codeCacheFull) = true;
    }
}
```

### 2.3.8 xposedCallHandler

```
static void xposedCallHandler(const u4* args, JValue* pResult, const Method* method, ::Thread* self) {
    XposedOriginalMethodsIt original = findXposedOriginalMethod(method);
    if (original == xposedOriginalMethods.end()) {
        dvmThrowNoSuchMethodError("could not find Xposed original method - how did you even get here?");
        return;
    }
    
    ThreadStatus oldThreadStatus = MEMBER_VAL(self, Thread, status);
    JNIEnv* env = MEMBER_VAL(self, Thread, jniEnv);
    
    // get java.lang.reflect.Method object for original method
    jobject originalReflected = env->ToReflectedMethod(
        (jclass)xposedAddLocalReference(self, original->clazz),
        (jmethodID)method,
        true);
  
    // convert/box arguments
    const char* desc = &method->shorty[1]; // [0] is the return type.
    Object* thisObject = NULL;
    size_t srcIndex = 0;
    size_t dstIndex = 0;
    
    // for non-static methods determine the "this" pointer
    if (!dvmIsStaticMethod(&(*original))) {
        thisObject = (Object*) xposedAddLocalReference(self, (Object*)args[0]);
        srcIndex++;
    }
    
    jclass objectClass = env->FindClass("java/lang/Object");
    jobjectArray argsArray = env->NewObjectArray(strlen(method->shorty) - 1, objectClass, NULL);
    
    while (*desc != '\0') {
        char descChar = *(desc++);
        JValue value;
        Object* obj;
 
        switch (descChar) {
        case 'Z':
        case 'C':
        case 'F':
        case 'B':
        case 'S':
        case 'I':
            value.i = args[srcIndex++];
            obj = (Object*) dvmBoxPrimitive(value, dvmFindPrimitiveClass(descChar));
            dvmReleaseTrackedAlloc(obj, NULL);
            break;
        case 'D':
        case 'J':
            value.j = dvmGetArgLong(args, srcIndex);
            srcIndex += 2;
            obj = (Object*) dvmBoxPrimitive(value, dvmFindPrimitiveClass(descChar));
            dvmReleaseTrackedAlloc(obj, NULL);
            break;
        case '[':
        case 'L':
            obj  = (Object*) args[srcIndex++];
            break;
        default:
            ALOGE("Unknown method signature description character: %c\n", descChar);
            obj = NULL;
            srcIndex++;
        }
        env->SetObjectArrayElement(argsArray, dstIndex++, xposedAddLocalReference(self, obj));
    }
    
    /**
	@function 调用Java中方法
	@para xposedClass 
	@para xposedHandleHookedMethod     
	xposedHandleHookedMethod = env->GetStaticMethodID(xposedClass, "handleHookedMethod",
        "(Ljava/lang/reflect/Member;Ljava/lang/Object;[Ljava/lang/Object;)Ljava/lang/Object;");
	*/
    jobject resultRef = env->CallStaticObjectMethod(
        xposedClass, xposedHandleHookedMethod, originalReflected, thisObject, argsArray);
        
    // exceptions are thrown to the caller
    if (env->ExceptionCheck()) {
        dvmChangeStatus(self, oldThreadStatus);
        return;
    }
    
    // return result with proper type
    Object* result = dvmDecodeIndirectRef(self, resultRef);
    ClassObject* returnType = dvmGetBoxedReturnType(method);
    if (returnType->primitiveType == PRIM_VOID) {
        // ignored
    } else if (result == NULL) {
        if (dvmIsPrimitiveClass(returnType)) {
            dvmThrowNullPointerException("null result when primitive expected");
        }
        pResult->l = NULL;
    } else {
        if (!dvmUnboxPrimitive(result, returnType, pResult)) {
            dvmThrowClassCastException(result->clazz, returnType);
        }
    }
    
    // set the thread status back to running. must be done after the last env->...()
    dvmChangeStatus(self, oldThreadStatus);
}
```

2.4 Java:XposedBridge
---------------------

注意，以下介绍的 XposedBridge 中所调用的方法都是在

### 2.4.1 private static void main(String[] args)

由 Zygote 跳入 XposedBridge 执行的第一个函数。

```
private static void main(String[] args) {
		/**
		@function Native方法，用于获取当前启动的类名
		*/
		String startClassName = getStartClassName();
 
		// 初始化Xposed框架与模块
		try {
			// 初始化Log文件
			try {
				File logFile = new File("/data/xposed/debug.log");
				if (startClassName == null && logFile.length() > MAX_LOGFILE_SIZE)
					logFile.renameTo(new File("/data/xposed/debug.log.old"));
				logWriter = new PrintWriter(new FileWriter(logFile, true));
				logFile.setReadable(true, false);
				logFile.setWritable(true, false);
			} catch (IOException ignored) {}
			
			String date = DateFormat.getDateTimeInstance().format(new Date());
			log("-----------------\n" + date + " UTC\n"
					+ "Loading Xposed (for " + (startClassName == null ? "Zygote" : startClassName) + ")...");
			/**
			@function 负责获取XposedBridge中Java函数的引用
			@description Native函数，定义于xposed.cpp中
			*/
			if (initNative()) {
				if (startClassName == null) {
				/**
				@function 如果是启动Zygote进程，则执行initXbridgeZygote函数
                @description 定义在XposedBridge类中
				*/
					initXbridgeZygote();
				}
				/**
				@function 负责加载所有模块
                @description 定义在XposedBridge类中
				*/
				loadModules(startClassName);
			} else {
				log("Errors during native Xposed initialization");
			}
		} catch (Throwable t) {
			log("Errors during Xposed initialization");
			log(t);
		}
		
		// 调用原来的启动参数
		if (startClassName == null)
		/**
		@description com.android.internal.os.ZygoteInit
		*/
			ZygoteInit.main(args);
		else
			RuntimeInit.main(args);
　　	}
```

### 2.4.2 void initXbridgeZygote():Hook 系统关键函数

initXbridgeZygote 完成对一些函数的 hook 操作，主要是调用 XposedHelpers 类中的 findAndHookMethod 完成。

```
private static void initXbridgeZygote() throws Exception {
		final HashSet<String> loadedPackagesInProcess = new HashSet<String>(1);
		
		/**
		@function 执行Hook替换操作
		@para ActivityThread.class 需要hook的函数所在的类；
		@para "handleBindApplication" 需要hook的函数名
		@para  "android.app.ActivityThread.AppBindData" 不定参数
		@description 定义在XposedHelper中
		*/
		findAndHookMethod(ActivityThread.class, "handleBindApplication", "android.app.ActivityThread.AppBindData", new XC_MethodHook() {
			protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
				ActivityThread activityThread = (ActivityThread) param.thisObject;
				ApplicationInfo appInfo = (ApplicationInfo) getObjectField(param.args[0], "appInfo");
				ComponentName instrumentationName = (ComponentName) getObjectField(param.args[0], "instrumentationName");
				if (instrumentationName != null) {
					XposedBridge.log("Instrumentation detected, disabling framework for " + appInfo.packageName);
					disableHooks = true;
					return;
				}
				CompatibilityInfo compatInfo = (CompatibilityInfo) getObjectField(param.args[0], "compatInfo");
				if (appInfo.sourceDir == null)
					return;
				
				setObjectField(activityThread, "mBoundApplication", param.args[0]);
				loadedPackagesInProcess.add(appInfo.packageName);
				LoadedApk loadedApk = activityThread.getPackageInfoNoCheck(appInfo, compatInfo);
				XResources.setPackageNameForResDir(appInfo.packageName, loadedApk.getResDir());
				
				LoadPackageParam lpparam = new LoadPackageParam(loadedPackageCallbacks);
				lpparam.packageName = appInfo.packageName;
				lpparam.processName = (String) getObjectField(param.args[0], "processName");
				lpparam.classLoader = loadedApk.getClassLoader();
				lpparam.appInfo = appInfo;
				lpparam.isFirstApplication = true;
				XC_LoadPackage.callAll(lpparam);
			}
		});
		...
	}
```

### 2.4.3 loadModules(String startClassName)：开始 Hook Module 中自定义函数

```
/**
@function 用于加载所有Xposed模块中定义的Hook操作
@para startClassName 当前打开的Class Name
*/
private static void loadModules(String startClassName) throws IOException {
	BufferedReader apks = new BufferedReader(new FileReader(BASE_DIR + "conf/modules.list"));
	String apk;
	while ((apk = apks.readLine()) != null) {
		loadModule(apk, startClassName);
	}
	apks.close();
}
/**
@function 从各个模块定义的xposed_init文件中进行目标函数的Hook
@para apk 模块名称，即所属Application
@para startClassName 当前打开的Class Name
*/
private static void loadModule(String apk, String startClassName) {
		log("Loading modules from " + apk);
		if (!new File(apk).exists()) {
			log("  File does not exist");
			return;
		}
		ClassLoader mcl = new PathClassLoader(apk, BOOTCLASSLOADER);
		
		InputStream is = mcl.getResourceAsStream("assets/xposed_init");
		if (is == null) {
			log("assets/xposed_init not found in the APK");
			return;
		}
		BufferedReader moduleClassesReader = new BufferedReader(new InputStreamReader(is));
		try {
			String moduleClassName;
			while ((moduleClassName = moduleClassesReader.readLine()) != null) {
				moduleClassName = moduleClassName.trim();
				if (moduleClassName.isEmpty() || moduleClassName.startsWith("#"))
					continue;
				try {
					log("  Loading class " + moduleClassName);
					Class<?> moduleClass = mcl.loadClass(moduleClassName);
					if (!IXposedMod.class.isAssignableFrom(moduleClass)) {
						log("This class doesn't implement any sub-interface of IXposedMod, skipping it");
						continue;
					}
					/*
						@description 在Zygote启动之前执行自定义的ZygoteInit函数等自定义的Module指令
						@annotation 1
					*/
					final Object moduleInstance = moduleClass.newInstance();
					if (startClassName == null) {
						if (moduleInstance instanceof IXposedHookZygoteInit) {
							IXposedHookZygoteInit.StartupParam param = new IXposedHookZygoteInit.StartupParam();
							param.modulePath = apk;
							((IXposedHookZygoteInit) moduleInstance).initZygote(param);
						}
						if (moduleInstance instanceof IXposedHookLoadPackage)
							hookLoadPackage(new IXposedHookLoadPackage.Wrapper((IXposedHookLoadPackage) moduleInstance));
						if (moduleInstance instanceof IXposedHookInitPackageResources)
							hookInitPackageResources(new IXposedHookInitPackageResources.Wrapper((IXposedHookInitPackageResources) moduleInstance));
					} else {
						if (moduleInstance instanceof IXposedHookCmdInit) {
							IXposedHookCmdInit.StartupParam param = new IXposedHookCmdInit.StartupParam();
							param.modulePath = apk;
							param.startClassName = startClassName;
							((IXposedHookCmdInit) moduleInstance).initCmdApp(param);
						}
					}
				} catch (Throwable t) {
					log(t);
				}
			}
		} catch (IOException e) {
			log(e);
		} finally {
			try {
				is.close();
			} catch (IOException ignored) {
			}
		}
	}
```

annotation 1

以下代码段主要将 module 中定义的类使用 instanceof 操作符确定其所属父类并依次执行操作，Xposed 提供的接口类主要分为：

<table><tbody><tr><td><p>IXposedHookZygoteInit</p></td><td><p>该类主要提供 ZygoteInit 接口函数，用于在 Zygote 进程启动之前执行相关代码。</p></td></tr><tr><td><p>IXposedHookLoadPackage</p></td><td><p>主要的 Hook 操作类。</p></td></tr><tr><td><p>IXposedHookInitPackageResources</p></td><td><p>提供资源 Hook 相关所需要的函数。</p></td></tr><tr><td><p>IXposedHookCmdInit</p></td><td><p>Hook 并处理启动新的 Application&nbsp;Dalvik 虚拟机时所需要的参数。</p></td></tr></tbody></table>

### 2.4.4 hookMethod

XposedBridge 类的静态方法 hookMethod 实现对函数的 hook 和回调函数的注册。

```
/**
 * @function Hook指定方法并设置前后回调函数
 * @param 方法名
 * @param 回调函数组 
 */
public static XC_MethodHook.Unhook hookMethod(Member hookMethod, XC_MethodHook callback) {
	if (!(hookMethod instanceof Method) && !(hookMethod instanceof Constructor<?>)) {
		throw new IllegalArgumentException("only methods and constructors can be hooked");
	}
	
	boolean newMethod = false;
	TreeSet<XC_MethodHook> callbacks;
	/*	
	HookedMethodCallbacks是一个hashMap的实例，存储每个需要hook的method的回调函数。	
	首先查看hookedMethodCallbacks中是否有hookMethod对应的callbacks的集合，
	如果没有，则创建一个TreeSet，将该callbacks加入到hookedMethodCallbacks中，同时将newMethod标志设为true。
	接下来将传入的callback添加到callbacks集合中。
	*/
	synchronized (hookedMethodCallbacks) {
		callbacks = hookedMethodCallbacks.get(hookMethod);
		if (callbacks == null) {
			callbacks = new TreeSet<XC_MethodHook>();
			hookedMethodCallbacks.put(hookMethod, callbacks);
			newMethod = true;
		}
	}
	synchronized (callbacks) {
		callbacks.add(callback);
	}
	
	if (newMethod) {
		Class<?> declaringClass = hookMethod.getDeclaringClass();
		int slot = (int) getIntField(hookMethod, "slot");
		/**
		@function 调用Native方法hookMethodNative，通知Native层declaringClass中的某个method是被hook的
		@para declaringClass 要hook的目标类
		@para slot Method编号
		@description Native方法，定义在了xposed.cpp中
		*/
		hookMethodNative(declaringClass, slot);
	}
	
	return callback.new Unhook(hookMethod);
}
```

### 2.4.5 handleHookedMethod

handleHookedMethod 将被 hook 的代码又交还给 java 层实现。

```
private static Object handleHookedMethod(Member method, Object thisObject, Object[] args) throws Throwable {
		if (disableHooks) {
			try {
				return invokeOriginalMethod(method, thisObject, args);
			} catch (InvocationTargetException e) {
				throw e.getCause();
			}
		}
```

首先判断 hook 是否被禁用，若是，则直接调用 invokeOriginalMethod 函数，完成对原始函数的执行。关于如何执行原始函数的，可以继续跟踪下去分析。

```
TreeSet<XC_MethodHook> callbacks;
synchronized (hookedMethodCallbacks) {
	callbacks = hookedMethodCallbacks.get(method);
}
if (callbacks == null || callbacks.isEmpty()) {
	try {
		return invokeOriginalMethod(method, thisObject, args);
	} catch (InvocationTargetException e) {
		throw e.getCause();
	}
}
synchronized (callbacks) {
	callbacks = ((TreeSet<XC_MethodHook>) callbacks.clone());
　　}
```

根据 method 值，从 hookedMethodCallbacks 中获取对应的 callback 信息。hookedMethodCallbacks 的分析可以参考之前对 hookMethod 的分析。callbacks 中存储了所有对该 method 进行 hook 的 beforeHookedMethod 和 afterHookedMethod。接着从 callbacks 中获取 beforeHookedMethod 和 afterHookedMethod 的迭代器。

```
Iterator<XC_MethodHook> before = callbacks.iterator();
Iterator<XC_MethodHook> after  = callbacks.descendingIterator();
// call "before method" callbacks
while (before.hasNext()) {
	try {
		before.next().beforeHookedMethod(param);
	} catch (Throwable t) {
		XposedBridge.log(t);
		
		// reset result (ignoring what the unexpectedly exiting callback did)
		param.setResult(null);
		param.returnEarly = false;
		continue;
	}
	
	if (param.returnEarly) {
		// skip remaining "before" callbacks and corresponding "after" callbacks
		while (before.hasNext() && after.hasNext()) {
			before.next();
			after.next();
		}
		break;
	}
}
// call original method if not requested otherwise
if (!param.returnEarly) {
	try {
		param.setResult(invokeOriginalMethod(method, param.thisObject, param.args));
	} catch (InvocationTargetException e) {
		param.setThrowable(e.getCause());
	}
}
// call "after method" callbacks
while (after.hasNext()) {
	Object lastResult =  param.getResult();
	Throwable lastThrowable = param.getThrowable();
	
	try {
		after.next().afterHookedMethod(param);
	} catch (Throwable t) {
		XposedBridge.log(t);
		
		// reset to last result (ignoring what the unexpectedly exiting callback did)
		if (lastThrowable == null)
			param.setResult(lastResult);
		else
			param.setThrowable(lastThrowable);
	}
}
// return
if (param.hasThrowable())
	throw param.getThrowable();
else
　　	return param.getResult();
```

通过以上的分析，基本能够弄清楚 Xposed 框架实现 hook 的原理。Xposed 将需要 hook 的函数替换成 Native 方法 xposedCallHandler，这样 Dalvik 在执行被 hook 的函数时，就会直接调用 xposedCallHandler，xposedCallHandler 再调用 XposedBridge 类的 handleHookedMethod 完成注册的 beforeHookedMethod 以及 afterHookedMethod 的调用，这两类回调函数之间，会调用原始函数，完成正常的功能。

2.5 Java：Class XposedHelper
---------------------------

### 2.5.1 findAndHookMethod

```
/**
@function 根据输入的类名与方法名进行Hook操作。
@para className 输入的类的名称
@para classLoader 当前上下文中的类加载器
@para methodName 方法名称
@para parameterTypesAndCallback 不定参数组，包括方法参数与回调函数
*/
public static XC_MethodHook.Unhook findAndHookMethod(String className, ClassLoader classLoader, String methodName, Object... parameterTypesAndCallback) {
	return findAndHookMethod(findClass(className, classLoader), methodName, parameterTypesAndCallback);
}
public static XC_MethodHook.Unhook findAndHookMethod(Class<?> clazz, String methodName, Object... parameterTypesAndCallback) {
	/**
	@function 判断不定参数组的最后一个参数是否为XC_MethodHook
	*/
	if (parameterTypesAndCallback.length == 0 || !(parameterTypesAndCallback[parameterTypesAndCallback.length-1] instanceof XC_MethodHook))
		throw new IllegalArgumentException("no callback defined");
	
	XC_MethodHook callback = (XC_MethodHook) parameterTypesAndCallback[parameterTypesAndCallback.length-1];
	
	/**
	@function 在一个类中查找到指定方法并将该方法设置为Accessible
	@para clazz 类对象
	@para methodName 方法名称
	@para parameterTypesAndCallback 方法参数
	@description 该类定义在了XposedHelper中
	@exception 如果没有在指定的类中查找到指定方法，则抛出NoSuchMethodError这个异常
	*/
	Method m = findMethodExact(clazz, methodName, parameterTypesAndCallback);
	
	/**
	@function 根据方法对象通知Native函数hookMethodNative执行Hook操作
	@para method 要Hook的方法对象
	@callback 回调函数组
	@description 定义在XposedBridge中
	*/
	return XposedBridge.hookMethod(m, callback);
}
```

### 2.5.2 findMethodExact

```
/**
@function findMethodExact(Class<?> clazz, String methodName, Class<?>... parameterTypes)的重载函数
*/
public static Method findMethodExact(Class<?> clazz, String methodName, Object... parameterTypes) {
	Class<?>[] parameterClasses = null;
	for (int i = parameterTypes.length - 1; i >= 0; i--) {
		Object type = parameterTypes[i];
		if (type == null)
			throw new ClassNotFoundError("parameter type must not be null", null);
		
		// ignore trailing callback
		if (type instanceof XC_MethodHook)
			continue;
		
		if (parameterClasses == null)
			parameterClasses = new Class<?>[i+1];
		
		if (type instanceof Class)
			parameterClasses[i] = (Class<?>) type;
		else if (type instanceof String)
			parameterClasses[i] = findClass((String) type, clazz.getClassLoader());
		else
			throw new ClassNotFoundError("parameter type must either be specified as Class or String", null);
	}
	
	// if there are no arguments for the method
	if (parameterClasses == null)
		parameterClasses = new Class<?>[0];
	
	return findMethodExact(clazz, methodName, parameterClasses);
}
/**
@function 在一个类中查找到指定方法并将该方法设置为Accessible
@para clazz 类对象
@para methodName 方法名称
@para parameterTypesAndCallback 方法参数
@description 该类定义在了XposedHelper中
@exception 如果没有在指定的类中查找到指定方法，则抛出NoSuchMethodError这个异常
*/
public static Method findMethodExact(Class<?> clazz, String methodName, Class<?>... parameterTypes) {
	StringBuilder sb = new StringBuilder(clazz.getName());
	sb.append('#');
	sb.append(methodName);
	sb.append(getParametersString(parameterTypes));
	sb.append("#exact");
	String fullMethodName = sb.toString();
	
	/**
	@function 判断当前Method是否已经被Hook
	*/
	if (methodCache.containsKey(fullMethodName)) {
		Method method = methodCache.get(fullMethodName);
		if (method == null)
			throw new NoSuchMethodError(fullMethodName);
		return method;
	}
	
	try {
		Method method = clazz.getDeclaredMethod(methodName, parameterTypes);
		method.setAccessible(true);
        /**
     	@annotation 1
	   */
 
		methodCache.put(fullMethodName, method);
		return method;
	} catch (NoSuchMethodException e) {
		methodCache.put(fullMethodName, null);
		throw new NoSuchMethodError(fullMethodName);
	}
}
```

annotation 1

在 findMethodExact(Class<?> clazz, String methodName, Class<?>... parameterTypes) 函数中，首先将类名、方法名以及参数信息构建成一个键值，以该键值从 methodCache 中查找是否存在 Method 实例，methodCache 相当于缓存了对应的 Method 实例。如果没有找到，会调用 Class 类的 getDeclaredMethod(String name,Class<?>... parameterTypes) 方法获取 Method 实例，同时将该 Method 设置为可访问，加入到 methodCache 中。

2.5 Java：Class XC_MethodHook
----------------------------

XC_MethodHook 类中的 beforeHookedMethod 函数会在被 hook 的函数调用之前调用，而 afterHookedMethod 函数会在被 hook 的函数调用之后调用。这两个函数的方法体为空，需要在实例化 XC_MethodHook 时根据情况填充方法体。XC_MethodHook 的内部类 MethodHookParam 保存了相应的信息，如调用方法的参数，this 对象，函数的返回值等。

```
public abstract class XC_MethodHook extends XCallback {
	public XC_MethodHook() {
		super();
	}
	public XC_MethodHook(int priority) {
		super(priority);
	}
	
	/**
	 * Called before the invocation of the method.
	 * <p>Can use {@link MethodHookParam#setResult(Object)} and {@link MethodHookParam#setThrowable(Throwable)}
	 * to prevent the original method from being called.
	 */
	protected void beforeHookedMethod(MethodHookParam param) throws Throwable {}
	
	/**
	 * Called after the invocation of the method.
	 * <p>Can use {@link MethodHookParam#setResult(Object)} and {@link MethodHookParam#setThrowable(Throwable)}
	 * to modify the return value of the original method.
	 */
	protected void afterHookedMethod(MethodHookParam param) throws Throwable  {}
	
	
	public static class MethodHookParam extends XCallback.Param {
		/** Description of the hooked method */
		public Member method;
		/** The <code>this</code> reference for an instance method, or null for static methods */
		public Object thisObject;
		/** Arguments to the method call */
		public Object[] args;
		
		private Object result = null;
		private Throwable throwable = null;
		/* package */ boolean returnEarly = false;
		
		/** Returns the result of the method call */
		public Object getResult() {
			return result;
		}
		
		/**
		 * Modify the result of the method call. In a "before-method-call"
		 * hook, prevents the call to the original method.
		 * You still need to "return" from the hook handler if required.
		 */
		public void setResult(Object result) {
			this.result = result;
			this.throwable = null;
			this.returnEarly = true;
		}
		
		/** Returns the <code>Throwable</code> thrown by the method, or null */
		public Throwable getThrowable() {
			return throwable;
		}
		
		/** Returns true if an exception was thrown by the method */
		public boolean hasThrowable() {
			return throwable != null;
		}
		
		/**
		 * Modify the exception thrown of the method call. In a "before-method-call"
		 * hook, prevents the call to the original method.
		 * You still need to "return" from the hook handler if required.
		 */
		public void setThrowable(Throwable throwable) {
			this.throwable = throwable;
			this.result = null;
			this.returnEarly = true;
		}
		
		/** Returns the result of the method call, or throws the Throwable caused by it */
		public Object getResultOrThrowable() throws Throwable {
			if (throwable != null)
				throw throwable;
			return result;
		}
	}
	public class Unhook implements IXUnhook {
		private final Member hookMethod;
		public Unhook(Member hookMethod) {
			this.hookMethod = hookMethod;
		}
		
		public Member getHookedMethod() {
			return hookMethod;
		}
		
		public XC_MethodHook getCallback() {
			return XC_MethodHook.this;
		}
		@Override
		public void unhook() {
			XposedBridge.unhookMethod(hookMethod, XC_MethodHook.this);
		}
	}
}
```