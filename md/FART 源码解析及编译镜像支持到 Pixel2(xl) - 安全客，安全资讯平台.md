> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/201896)

> 先看结论吧，最近用 android-8.1.0_r1 这个版本来编译了 FART，这个版本支持的设备比较多，大家常用的应该都有了。

[![](https://p5.ssl.qhimg.com/t0132b2f7adba02f3d0.jpg)](https://p5.ssl.qhimg.com/t0132b2f7adba02f3d0.jpg)

镜像编译
----

先看结论吧，最近用`android-8.1.0_r1`这个版本来编译了`FART`，这个版本支持的设备比较多，大家常用的应该都有了。

<table><thead><tr><th>Codename</th><th>aosp version</th><th>android version</th><th>supported devices</th></tr></thead><tbody><tr><td>OPM1.171019.011</td><td>android-8.1.0_r1</td><td>Oreo</td><td>Pixel 2 XL, Pixel 2, Pixel XL, Pixel, Pixel C, Nexus 6P, Nexus 5X</td></tr></tbody></table>

[![](https://p2.ssl.qhimg.com/t01a455f8a1b44c70ad.png)](https://p2.ssl.qhimg.com/t01a455f8a1b44c70ad.png)

也传到网盘里去了，网盘链接在我 github 主页最下方：

*   [https://github.com/r0ysue/AndroidSecurityStudy](https://github.com/r0ysue/AndroidSecurityStudy)

源码解析
----

分析方法为源码比对，如下图所示：

[![](https://p0.ssl.qhimg.com/t0109117d48fad69f17.jpg)](https://p0.ssl.qhimg.com/t0109117d48fad69f17.jpg)

源码分析常用在漏洞分析的情况下，只要知道补丁打在了哪里，就可以研究漏洞出在哪里，知道漏洞成因后写出利用代码，批量攻击没有打补丁的机器；虽然现在补丁发布后`90`天才允许发布漏洞细节报告，但是只要分析能力够强，可以直接逆补丁即可，获取漏洞成因和细节，写出`1day`的利用，（然后被不法分子使用，在网上扫肉鸡）。

比如这位大佬在源码还没有开源的情况下，仅通过逆向分析`system.img`镜像和`libart.so`就还原出源码并应用到安卓`9`上去，是真的大佬，原贴链接：[FART：ART 环境下基于主动调用的自动化脱壳方案 [android 脱壳源码公开，基于 android-9.0.0_r36]](https://bbs.pediy.com/thread-257101.htm) 。当然以下的分析其实这位大佬和寒冰大佬已经讲得披露得非常清楚了，也没啥新的东西，只是再康康源码，见下图源码结构。

[![](https://p0.ssl.qhimg.com/t01290ef72c63222ea4.jpg)](https://p0.ssl.qhimg.com/t01290ef72c63222ea4.jpg)

虽然俺这里源码改好了，但是毕竟是人家的代码，等人家自己公开哈。

### 第一组件：脱壳

在`FART`系列文章中，三篇中有两篇是讲第一组件也就是脱壳，在寒冰大佬的第一篇中使用的脱壳方法还是通过`ClassLoader`脱壳的方式，选择合适的时机点获取到应用解密后的`dex`文件最终依附的`Classloader`，进而通过`java`的反射机制最终获取到对应的`DexFile`的结构体，并完成`dex`的`dump`。

**不优雅且效率低的`Classloader`时代**

对于获取`Classloader`的时机点的选择。在`App`启动流程以及`App`加壳原理和执行流程的过程中，可以看到，`App`中的`Application`类中的`attachBaseContext`和`onCreate`函数是`app`中最先执行的方法。壳都是通过替换`App`的`Application`类并自己实现这两个函数，并在这两个函数中实现`dex`的解密加载，`hook`系统中`Class`和`method`加载执行流程中的关键函数，最后通过反射完成关键变量如最终的`Classloader`，`Application`等的替换从而完成执行权的交付。

因此，可以选在任意一个在`Application`的`onCreate`函数执行之后才开始被调用的任意一个函数中。众所周知，对于一个正常的应用来说，最终都要由一个个的`Activity`来展示应用的界面并和用户完成交互，那么我们就可以选择在`ActivityThread`中的`performLaunchActivity`函数作为时机，来获取最终的应用的`Classloader`。选择该函数还有一个好处在于该函数和应用的最终的`application`同在`ActivityThread`类中，可以很方便获取到该类的成员。

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ......

Activity activity = null;
        try {
            java.lang.ClassLoader cl = r.packageInfo.getClassLoader();
//下面通过application的getClassLoader()获取最终的Classloader，并开启线程，在新线程中完成内存中的dex的dump以及主动调用过程，由于该过程相对耗时，为了防止应用出现ANR，从而开启新线程，在新线程中进行，主要的工作都在getDexFilesByClassLoader_23
            //addstart
            packagename=r.packageInfo.getPackageName();
            //mInitialApplication
            //final java.lang.ClassLoader finalcl=cl
            if(mInitialApplication!=null){
            final java.lang.ClassLoader finalcl=mInitialApplication.getClassLoader();
            new Thread(new Runnable() {
            @Override
            public void run() {              
                                getDexFilesByClassLoader_23(finalcl);
                            }
                        }).start();

                }

            //addend
          }
}
```

`getDexFilesByClassLoader_23()`函数的主要流程就是通过一系列的反射，最终获取到当前`Classloader`中的`mCookie`，即`Native`层中的`DexFile`。为了在`C/C++`中完成对`dex`的`dump`操作。这里我们在`framework`层的`DexFile`类中添加两个`Native`函数供调用：

在文件`libcore/dalvik/src/main/java/dalvik/system/DexFile.java`中

```
private static native void dumpDexFile(String dexfilepath,Object cookie);
private static native void dumpMethodCode(String eachclassname, String methodname,Object cookie, Object method);
```

以上流程总结一下就是：

1.  获取最终`dex`依附的`ClassLoader`
2.  通过反射获取到`PathList`对象、`Element`对象、`mCookie`对象
3.  在`native`层通过`mCookie`完成`dex`的`dump`

具体的脱壳代码就不贴了，因为`FART`最终采用的完全不是这个方法。

我们分析最终的`libcore/dalvik/src/main/java/dalvik/system/DexFile.java`文件也发现，移除掉了`dumpDexFile`函数，只剩下`dumpMethodCode`函数。

```
//add
private static native void dumpMethodCode(Object m);
//add
```

在最终的`ActivityThread.java`的`performLaunchActivity`函数中也没有出现`getDexFilesByClassLoader_23()`函数，可见被整体移除了。

**直接内存中获取`DexFile`对象脱壳**

具体原因在大佬的第二、三篇中解释的非常清楚。大佬的第二篇就对第一组件提出了改进，[《ART 下几个通用简单高效的 dump 内存中 dex 方法》](https://bbs.pediy.com/thread-254028.htm)，在对`ART`虚拟机的类加载执行流程以及`ArtMethod`类的生命周期进行完整分析之后，选择了通过运行过程中`ArtMethod`来使用`GetDexFile()`函数从而获取到`DexFile`对象引用进而达成`dex`的`dump`这种内存型脱壳的技术。

最终实现的代码如下，修改的文件为`art_method.cc`，直接看注释：

```
extern "C" void dumpdexfilebyExecute(ArtMethod* artmethod)  REQUIRES_SHARED(Locks::mutator_lock_) {
            //为保存dex的名称开辟空间
            char *dexfilepath=(char*)malloc(sizeof(char)*1000);    
            if(dexfilepath==nullptr)
            {
                LOG(ERROR)<< "ArtMethod::dumpdexfilebyArtMethod,methodname:"<<artmethod->PrettyMethod().c_str()<<"malloc 1000 byte failed";
                return;
            }
            int result=0;
            int fcmdline =-1;
            char szCmdline[64]= {0};
            char szProcName[256] = {0};
            int procid = getpid();
            sprintf(szCmdline,"/proc/%d/cmdline", procid);
            //根据进程号得到进程的命令行参数
            fcmdline = open(szCmdline, O_RDONLY,0644);
            if(fcmdline >0)
            {
                result=read(fcmdline, szProcName,256);
                if(result<0)
                {
                    LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,open cmdline file error";
                    }
                close(fcmdline);

            }
            if(szProcName[0])
            {

                      //获取当前`DexFile`对象
                      const DexFile* dex_file = artmethod->GetDexFile();
                      //当前DexFile的起始地址
                      const uint8_t* begin_=dex_file->Begin();  // Start of data.
                      //当前DexFile的长度
                      size_t size_=dex_file->Size();  // Length of data.

                      //保存地址置零
                      memset(dexfilepath,0,1000);
                      int size_int_=(int)size_;
                      //构造保存地址
                      memset(dexfilepath,0,1000);
                      sprintf(dexfilepath,"%s","/sdcard/fart");
                      mkdir(dexfilepath,0777);

                      memset(dexfilepath,0,1000);
                      sprintf(dexfilepath,"/sdcard/fart/%s",szProcName);
                      mkdir(dexfilepath,0777);
                      //与进程名、大小一起，构建文件名
                      memset(dexfilepath,0,1000);
                      sprintf(dexfilepath,"/sdcard/fart/%s/%d_dexfile_execute.dex",szProcName,size_int_);
                      //打开文件，转储内容
                      int dexfilefp=open(dexfilepath,O_RDONLY,0666);
                      if(dexfilefp>0){
                          close(dexfilefp);
                          dexfilefp=0;
                          }else{
                                      int fp=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
                                      if(fp>0)
                                      {
                                          //直接写内容
                                          result=write(fp,(void*)begin_,size_);
                                          if(result<0)
                                          {
                                              LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,open dexfilepath error";
                                              }
                                          fsync(fp); 
                                          //dex转储完成
                                          close(fp);  
                                          memset(dexfilepath,0,1000);
                                          //构造写类名的地址
                                          sprintf(dexfilepath,"/sdcard/fart/%s/%d_classlist_execute.txt",szProcName,size_int_);
                                          int classlistfile=open(dexfilepath,O_CREAT|O_APPEND|O_RDWR,0666);
                                            if(classlistfile>0)
                                            {
                                                for (size_t ii= 0; ii< dex_file->NumClassDefs(); ++ii) 
                                                {
                                                    const DexFile::ClassDef& class_def = dex_file->GetClassDef(ii);
                                                    const char* descriptor = dex_file->GetClassDescriptor(class_def);
                                                    //写入GetClassDescriptor
                                                    result=write(classlistfile,(void*)descriptor,strlen(descriptor));
                                                    if(result<0)
                                                    {
                                                        LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write classlistfile file error";

                                                        }
                                                    const char* temp="n";
                                                    //写入一个换行符
                                                    result=write(classlistfile,(void*)temp,1);
                                                    if(result<0)
                                                    {
                                                        LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,write classlistfile file error";

                                                        }
                                                    }
                                                  fsync(classlistfile); 
                                                  //写入完成
                                                  close(classlistfile); 

                                                }
                                          }


                                      }


            }

            if(dexfilepath!=nullptr)
            {
                free(dexfilepath);
                dexfilepath=nullptr;
            }

}
```

总结一下脱壳流程就是：

1.  通过`ArtMethod`对象的`getDexFile()`获取所属的`DexFile`对象
2.  通过获得的`DexFile`对象完成`dex`的`dump`

**进一步优化找到完美脱壳点**

然后到了第三篇中，原来找到的`LoadClass->LoadClassMembers->LinkCode`脱壳点，再次被无情的抛弃，寒冰大佬对当前安卓`App`脱壳的本质进行了深入的总结，以及引申开，介绍如何快速发现`Art`虚拟机中隐藏的海量脱壳点，具体内容见[《安卓 APP 脱壳的本质以及如何快速发现 ART 下的脱壳点》](https://bbs.pediy.com/thread-254555.htm)。

最终在`interpreter.cc`中添加的就约等于两句话，在文件头声明一下，在`Execute`函数中执行调用一下，将`dumpdexfilebyExecute`函数跑起来就行了。

```
//add
namespace art {
    extern "C" void dumpdexfilebyExecute(ArtMethod* artmethod);
//add

static inline JValue Execute(
    Thread* self,
    const DexFile::CodeItem* code_item,
    ShadowFrame& shadow_frame,
    JValue result_register,
    bool stay_in_interpreter = false) REQUIRES_SHARED(Locks::mutator_lock_) {

    //add
    if(strstr(shadow_frame.GetMethod()->PrettyMethod().c_str(),"<clinit>"))
    {
        dumpdexfilebyExecute(shadow_frame.GetMethod());
        }
    //add


  DCHECK(!shadow_frame.GetMethod()->IsAbstract());
  DCHECK(!shadow_frame.GetMethod()->IsNative());
```

这部分代码最少，确实最考验功底的地方。在这里脱壳，实践下来可以脱市面上绝大多数壳，很多抽取型的壳在这个点上也方法体也恢复进了`dex`连壳带方法完整的脱了下来，因为现在很多壳通过阻断`dex2oat`的编译过程，导致了不只是类的初始化函数在解释模式下执行，也让类中的其他函数也运行在解释模式下的原因。

这就好比修电脑，换个电容五毛钱，但是你要付五十块一样的。几行代码简单，找到这个点不简单，能脱几乎所有壳，壳还没办法绕过，这个值钱。

### 第二组件：转存函数体

在`FART`系列文章中，三篇中只有第一篇介绍了`FART`的针对函数抽取型壳而设计的主动调用方法、转储方法体组件，对设计的细节和代码的介绍非常少，在这里我们结合代码来看一下具体的流程。

**从`ActivityThread`中开始**

故事的开始，起源于`ActivityThread.java`。引用大佬文章中的图：

[![](https://p1.ssl.qhimg.com/t0121f8c06f4dea831f.png)](https://p1.ssl.qhimg.com/t0121f8c06f4dea831f.png)

通过`Zygote`进程到最终进入到`app`进程世界，我们可以看到`ActivityThread.main()`是进入`App`世界的大门，对于`ActivityThread`这个类，其中的`sCurrentActivityThread`静态变量用于全局保存创建的`ActivityThread`实例，同时还提供了`public static ActivityThread currentActivityThread()`静态函数用于获取当前虚拟机创建的`ActivityThread`实例。`ActivityThread.main()`函数是`java`中的入口`main`函数, 这里会启动主消息循环，并创建`ActivityThread`实例，之后调用`thread.attach(false)`完成一系列初始化准备工作，并完成全局静态变量`sCurrentActivityThread`的初始化。之后主线程进入消息循环，等待接收来自系统的消息。当收到系统发送来的`bindapplication`的进程间调用时，调用函数`handlebindapplication`来处理该请求。

在`handleBindApplication`函数中第一次进入了`app`的代码世界，该函数功能是启动一个`application`，并把系统收集的`apk`组件等相关信息绑定到`application`里，在创建完`application`对象后，接着调用了`application`的`attachBaseContext`方法，之后调用了`application`的`onCreate`函数，而作者添加的`fartthread()`这个函数体，就位于`handleBindApplication`之前，当然具体的调用，还是在上文的`performLaunchActivity`之中。

```
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        // System.out.println("##### [" + System.currentTimeMillis() + "] ActivityThread.performLaunchActivity(" + r + ")");
        Log.e("ActivityThread","go into performLaunchActivity");
        ...
        //add
        fartthread()
        //add
        ...
```

`fartthread()`函数的根本目和最终目标还是通过一系列的反射，最终获取到当前`Classloader`中的`mCookie`，即`Native`层中的`DexFile`，这一点体现在下一小节的`DexFile_dumpMethodCode`函数之中，完成主动调用链的构造。

`fartthread()`代码非常简单，进入`ActivityThread`开个新线程之后，先睡个一分钟，再进入`fart()`函数。

```
public static void fartthread() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            // TODO Auto-generated method stub
            try {
                Log.e("ActivityThread", "start sleep......");
                Thread.sleep(1 * 60 * 1000);
            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
            Log.e("ActivityThread", "sleep over and start fart");
            fart();
            Log.e("ActivityThread", "fart run over");

        }
    }).start();
}
```

在`fart()`函数中获取当前类的`Classloader`和父类的`Classloader`，并且不断向上追溯，只要没有到`java.lang.BootClassLoader`，就持续获取`Classloader`，并且使用`fartwithClassloader()`来进入这些`Classloader`中去。

```
public static void fart() {
    ClassLoader appClassloader = getClassloader();  //见下文getClassloader的分析
    ClassLoader tmpClassloader=appClassloader;
    ClassLoader parentClassloader=appClassloader.getParent();
    if(appClassloader.toString().indexOf("java.lang.BootClassLoader")==-1)
    {
        fartwithClassloader(appClassloader);
    }
    while(parentClassloader!=null){
        if(parentClassloader.toString().indexOf("java.lang.BootClassLoader")==-1)
        {
            fartwithClassloader(parentClassloader);
        }
        tmpClassloader=parentClassloader;
        parentClassloader=parentClassloader.getParent();
    }
}
```

接下来看`fartwithClassloader`到底做了什么：

```
public static void fartwithClassloader(ClassLoader appClassloader) {
    //进行一系列的准备工作
    List<Object> dexFilesArray = new ArrayList<Object>();
    Field pathList_Field = (Field) getClassField(appClassloader, "dalvik.system.BaseDexClassLoader", "pathList");
    Object pathList_object = getFieldOjbect("dalvik.system.BaseDexClassLoader", appClassloader, "pathList");
    Object[] ElementsArray = (Object[]) getFieldOjbect("dalvik.system.DexPathList", pathList_object, "dexElements");
    Field dexFile_fileField = null;
    //通过当前的`appClassloader`得到dex文件的实例
    try {
        dexFile_fileField = (Field) getClassField(appClassloader, "dalvik.system.DexPathList$Element", "dexFile");
    } catch (Exception e) {
            e.printStackTrace();
        } catch (Error e) {
            e.printStackTrace();
        }
    Class DexFileClazz = null;
    try {
        DexFileClazz = appClassloader.loadClass("dalvik.system.DexFile");
    } catch (Exception e) {
            e.printStackTrace();
        } catch (Error e) {
            e.printStackTrace();
        }
    Method getClassNameList_method = null;
    Method defineClass_method = null;
    Method dumpDexFile_method = null;
    Method dumpMethodCode_method = null;

    //开始分配不同的工作任务，当然dumpDexFile那个已经被废弃了
    for (Method field : DexFileClazz.getDeclaredMethods()) {
        if (field.getName().equals("getClassNameList")) {
            getClassNameList_method = field;
            getClassNameList_method.setAccessible(true);
        }
        if (field.getName().equals("defineClassNative")) {
            defineClass_method = field;
            defineClass_method.setAccessible(true);
        }
        if (field.getName().equals("dumpDexFile")) {
            dumpDexFile_method = field;
            dumpDexFile_method.setAccessible(true);
        }
        if (field.getName().equals("dumpMethodCodmpMethodCode")) {
            dumpMethodCode_method = field;
            dumpMethodCode_method.setAccessible(true);
        }
    }

    //获取mCookie
    Field mCookiefield = getClassField(appClassloader, "dalvik.system.DexFile", "mCookie");
    Log.v("ActivityThread->methods", "dalvik.system.DexPathList.ElementsArray.length:" + ElementsArray.length);//5个
    for (int j = 0; j < ElementsArray.length; j++) {
        Object element = ElementsArray[j];
        Object dexfile = null;
        try {
            dexfile = (Object) dexFile_fileField.get(element);
        } catch (Exception e) {
            e.printStackTrace();
        } catch (Error e) {
            e.printStackTrace();
        }
        if (dexfile == null) {
            Log.e("ActivityThread", "dexfile is null");
            continue;
        }
        if (dexfile != null) {
            dexFilesArray.add(dexfile);
            Object mcookie = getClassFieldObject(appClassloader, "dalvik.system.DexFile", dexfile, "mCookie");
            if (mcookie == null) {
                Object mInternalCookie = getClassFieldObject(appClassloader, "dalvik.system.DexFile", dexfile, "mInternalCookie");
                if(mInternalCookie!=null)
                {
                    mcookie=mInternalCookie;
                }else{
                        Log.v("ActivityThread->err", "get mInternalCookie is null");
                        continue;
                        }

            }
            ////调用DexFile类的getClassNameList获取dex中所有类名
            String[] classnames = null;
            try {
                classnames = (String[]) getClassNameList_method.invoke(dexfile, mcookie);
            } catch (Exception e) {
                e.printStackTrace();
                continue;
            } catch (Error e) {
                e.printStackTrace();
                continue;
            }
            if (classnames != null) {
                for (String eachclassname : classnames) {
                    loadClassAndInvoke(appClassloader, eachclassname, dumpMethodCode_method);
                }
            }

        }
    }
    return;
}
```

获取当前进程的`Classloader`代码如下，具体流程为

1.  通过反射调用`ActivityThread`类的静态函数`currentActivityThread`获取当前的`ActivityThread`对象
2.  然后获取`ActivityThread`对象的`mBoundApplication`成员变量；
3.  之后获取`mBoundApplication`对象的`info`成员变量，他是个`LoadedApk`类型；
4.  最终获取`info`对象的`mApplication`成员变量，他的类型是`Application`；
5.  最后通过调用`Application.getClassLoader`得到当前进程的`Classloader`。

```
public static ClassLoader getClassloader() {
    ClassLoader resultClassloader = null;
    Object currentActivityThread = invokeStaticMethod(
            "android.app.ActivityThread", "currentActivityThread",
            new Class[]{}, new Object[]{});
    Object mBoundApplication = getFieldOjbect(
            "android.app.ActivityThread", currentActivityThread,
            "mBoundApplication");
    Application mInitialApplication = (Application) getFieldOjbect("android.app.ActivityThread",
            currentActivityThread, "mInitialApplication");
    Object loadedApkInfo = getFieldOjbect(
            "android.app.ActivityThread$AppBindData",
            mBoundApplication, "info");
    Application mApplication = (Application) getFieldOjbect("android.app.LoadedApk", loadedApkInfo, "mApplication");
    resultClassloader = mApplication.getClassLoader();
    return resultClassloader;
}
```

一系列辅助的工具类：

```
//add
public static Field getClassField(ClassLoader classloader, String class_name,
                                    String filedName) {
    try {
        Class obj_class = classloader.loadClass(class_name);//Class.forName(class_name);
        Field field = obj_class.getDeclaredField(filedName);
        field.setAccessible(true);
        return field;
    } catch (SecurityException e) {
        e.printStackTrace();
    } catch (NoSuchFieldException e) {
        e.printStackTrace();
    } catch (IllegalArgumentException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
    return null;

}

public static Object getClassFieldObject(ClassLoader classloader, String class_name, Object obj,
                                            String filedName) {

    try {
        Class obj_class = classloader.loadClass(class_name);//Class.forName(class_name);
        Field field = obj_class.getDeclaredField(filedName);
        field.setAccessible(true);
        Object result = null;
        result = field.get(obj);
        return result;
    } catch (SecurityException e) {
        e.printStackTrace();
    } catch (NoSuchFieldException e) {
        e.printStackTrace();
    } catch (IllegalArgumentException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    }
    return null;

}

public static Object getFieldOjbect(String class_name, Object obj,
                                    String filedName) {
    try {
        Class obj_class = Class.forName(class_name);
        Field field = obj_class.getDeclaredField(filedName);
        field.setAccessible(true);
        return field.get(obj);
    } catch (SecurityException e) {
        e.printStackTrace();
    } catch (NoSuchFieldException e) {
        e.printStackTrace();
    } catch (IllegalArgumentException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    } catch (NullPointerException e) {
        e.printStackTrace();
    }
    return null;

}

//辅助getClassloader函数中通过反射获取当前的`ActivityThread`对象
public static Object invokeStaticMethod(String class_name,
                                        String method_name, Class[] pareTyple, Object[] pareVaules) {

    try {
        Class obj_class = Class.forName(class_name);
        Method method = obj_class.getMethod(method_name, pareTyple);
        return method.invoke(null, pareVaules);
    } catch (SecurityException e) {
        e.printStackTrace();
    } catch (IllegalArgumentException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    } catch (NoSuchMethodException e) {
        e.printStackTrace();
    } catch (InvocationTargetException e) {
        e.printStackTrace();
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    }
    return null;

}
```

**主动调用模块设计**

`FART`中的主动调用模块的设计，参考了`jni`桥的源码设计，`jni`提供了一系列`java`层函数与`Native`层函数交互的接口。当需要在`Native`层中的`c/c++`函数中调用位于`java`层的函数时，需要先获取到该函数的`jmethodid`然后再通过诸如`jni`中提供的`call`开头的一系列函数来完成对`java`层中函数的调用。

作者以`jni`中的`CallObjectMethod`函数为例，阐述该函数如何将参数转化成指针、进而层层深入、最终完成调用的，是`ArtMethod`类中的`Invoke`函数完成对`java`层中的函数的调用。

于是，作者构造出自己的`invoke`函数，在该函数中再调用`ArtMethod`的`Invoke`方法从而完成主动调用，并在`ArtMethod`的`Invoke`函数中首先进行判断，当发现是我们自己的主动调用时就进行方法体的`dump`并直接返回，从而完成对壳的欺骗，达到方法体的`dump`。

具体体现在代码上，主动调用就体现到这里了：`appClassloader.loadClass(eachclassname);`

*   加壳程序`hook`了加载类的方法，当真正执行时加载类的时候会进行还原，这个加载类相当于隐式加载。
*   我们这里`loadClass`是显示加载所有的类，这时候类的方法已经被还原。
*   `loadClassAndInvoke`，首先通过`loadClass`来主动加载所有类，然后调用`dumpMethodCode`来进行脱壳，参数为`Method`或者`Constructor`对象。

```
public static void loadClassAndInvoke(ClassLoader appClassloader, String eachclassname, Method dumpMethodCode_method) {
    Class resultclass = null;
    try {
        ////主动加载dex中的所有类，此时Method数据已解密
        resultclass = appClassloader.loadClass(eachclassname);
    } catch (Exception e) {
        e.printStackTrace();
        return;
    } catch (Error e) {
        e.printStackTrace();
        return;
    }
    if (resultclass != null) {
        try {
            Constructor<?> cons[] = resultclass.getDeclaredConstructors();
            for (Constructor<?> constructor : cons) {
                if (dumpMethodCode_method != null) {
                    try {
                        ////调用DexFile中dumpMethodCode方法，参数为Constructor对象
                        dumpMethodCode_method.invoke(null, constructor);
                    } catch (Exception e) {
                        e.printStackTrace();
                        continue;
                    } catch (Error e) {
                        e.printStackTrace();
                        continue;
                    }
                } else {
                    Log.e("ActivityThread", "dumpMethodCode_method is null ");
                }

            }
        } catch (Exception e) {
            e.printStackTrace();
        } catch (Error e) {
            e.printStackTrace();
        }
        try {
            Method[] methods = resultclass.getDeclaredMethods();
            if (methods != null) {
                ////调用DexFile中dumpMethodCode方法，参数为Method对象
                for (Method m : methods) {
                    if (dumpMethodCode_method != null) {
                        try {
                            dumpMethodCode_method.invoke(null, m);
                            } catch (Exception e) {
                            e.printStackTrace();
                            continue;
                        } catch (Error e) {
                            e.printStackTrace();
                            continue;
                        }
                    } else {
                        Log.e("ActivityThread", "dumpMethodCode_method is null ");
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        } catch (Error e) {
            e.printStackTrace();
        }
    }
}

//add
```

**方法体`dump`模块**

接下来则是在`art_method`中的`dumpMethodCode`部分，首先在`framework`层的`DexFile`类中添加个`Native`函数供调用，在文件`libcore/dalvik/src/main/java/dalvik/system/DexFile.java`中

```
private static native void dumpMethodCode(String eachclassname, String methodname,Object cookie, Object method);
```

下面开始代码部分。在`/art/runtime/native/dalvik_system_DexFile.cc`的另一个`Native`函数`DexFile_dumpMethodCode`中添加如下代码：

```
//addfunction
static void DexFile_dumpMethodCode(JNIEnv* env, jclass,jobject method) {
  if(method!=nullptr)
  {
          ArtMethod* proxy_method = jobject2ArtMethod(env, method);
          myfartInvoke(proxy_method);
      }     

  return;
}
//addfunction
```

可以看到代码非常简洁，首先是对`Java`层传来的`Method`结构体进行了类型转换，转成`Native`层的`ArtMethod`对象，接下来就是调用`ArtMethod`类中`myfartInvoke`实现虚拟调用，并完成方法体的`dump`。下面看`ArtMethod.cc`中添加的函数`myfartInvoke`的实现主动调用的代码部分，具体修改的文件是`art_method.cc`：

```
extern "C" void myfartInvoke(ArtMethod* artmethod)  REQUIRES_SHARED(Locks::mutator_lock_) {
    JValue *result=nullptr;
    Thread *self=nullptr;
    uint32_t temp=6;
    uint32_t* args=&temp;
    uint32_t args_size=6;
    artmethod->Invoke(self, args, args_size, result, "fart");
}
```

这里代码依然很简洁，只是对`ArtMethod`类中的`Invoke`的一个调用包装，不同的是在参数方面，我们直接给`Thread*`传递了一个`nullptr`，作为对主动调用过来的标识。下面看`ArtMethod`类中的`Invoke`函数：

```
...
void ArtMethod::Invoke(Thread* self, uint32_t* args, uint32_t args_size, JValue* result,
                       const char* shorty) {

        if (self== nullptr) {
        dumpArtMethod(this);
        return;
        }
  if (UNLIKELY(__builtin_frame_address(0) < self->GetStackEnd())) {
    ThrowStackOverflowError(self);
    return;
  }
  ...
```

该函数只是在最开头添加了对`Thread*`参数的判断，当发现该参数为`nullptr`时，即表示是我们自己构造的主动调用链到达，则此时调用`dumpArtMethod()`函数完成对该`ArtMethod`的`CodeItem`的`dump`，这部分代码和`fupk3`一样直接采用`dexhunter`里的，这里不再赘述。

```
extern "C" void dumpArtMethod(ArtMethod* artmethod)  REQUIRES_SHARED(Locks::mutator_lock_) {
            char *dexfilepath=(char*)malloc(sizeof(char)*1000);    
            if(dexfilepath==nullptr)
            {
                LOG(ERROR) << "ArtMethod::dumpArtMethodinvoked,methodname:"<<artmethod->PrettyMethod().c_str()<<"malloc 1000 byte failed";
                return;
            }
            int result=0;
            int fcmdline =-1;
            char szCmdline[64]= {0};
            char szProcName[256] = {0};
            int procid = getpid();
            sprintf(szCmdline,"/proc/%d/cmdline", procid);
            fcmdline = open(szCmdline, O_RDONLY,0644);
            if(fcmdline >0)
            {
                result=read(fcmdline, szProcName,256);
                if(result<0)
                {
                    LOG(ERROR) << "ArtMethod::dumpdexfilebyArtMethod,open cmdline file file error";                                    
                }
                close(fcmdline);
            }

            if(szProcName[0])
            {
                      const DexFile* dex_file = artmethod->GetDexFile();
                      const uint8_t* begin_=dex_file->Begin();  // Start of data.
                      size_t size_=dex_file->Size();  // Length of data.

                      memset(dexfilepath,0,1000);
                      int size_int_=(int)size_;
            ...
            ...

}
```

到这里，我们就完成了内存中`DexFile`结构体中的`dex`的整体`dump`以及主动调用完成对每一个类中的函数体的`dump`，下面就是修复被抽取的函数部分。

### 第三组件：函数体填充

壳在完成对内存中加载的`dex`的解密后，该`dex`的索引区即`stringid`，`typeid`，`methodid`，`classdef`和对应的`data`区中的`string`列表并未加密。

而对于`classdef`中类函数的`CodeItem`部分可能被加密存储或者直接指向内存中另一块区域。这里我们只需要使用`dump`下来的`method`的`CodeItem`来解析对应的被抽取的方法即可，这里大佬提供了一个用`python`实现的修复脚本，该脚本的`decode`部分与`art_method.cc`中的`encode`部分是相对应的，下面是`encode`部分的代码：

```
//add
uint8_t* codeitem_end(const uint8_t **pData)
{
    uint32_t num_of_list = DecodeUnsignedLeb128(pData);
    for (;num_of_list>0;num_of_list--) {
        int32_t num_of_handlers=DecodeSignedLeb128(pData);
        int num=num_of_handlers;
        if (num_of_handlers<=0) {
            num=-num_of_handlers;
        }
        for (; num > 0; num--) {
            DecodeUnsignedLeb128(pData);
            DecodeUnsignedLeb128(pData);
        }
        if (num_of_handlers<=0) {
            DecodeUnsignedLeb128(pData);
        }
    }
    return (uint8_t*)(*pData);
}

extern "C" char *base64_encode(char *str,long str_len,long* outlen){
    long len;   
    char *res;  
    int i,j;  
    const char *base64_table="ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";  
    if(str_len % 3 == 0)  
        len=str_len/3*4;  
    else  
        len=(str_len/3+1)*4;  

    res=(char*)malloc(sizeof(char)*(len+1));  
    res[len]='';  
    *outlen=len;
    for(i=0,j=0;i<len-2;j+=3,i+=4)  
    {  
        res[i]=base64_table[str[j]>>2];  
        res[i+1]=base64_table[(str[j]&0x3)<<4 | (str[j+1]>>4)]; 
        res[i+2]=base64_table[(str[j+1]&0xf)<<2 | (str[j+2]>>6)]; 
        res[i+3]=base64_table[str[j+2]&0x3f]; 
    }  

    switch(str_len % 3)  
    {  
        case 1:  
            res[i-2]='=';  
            res[i-1]='=';  
            break;  
        case 2:  
            res[i-1]='=';  
            break;  
    }  

    return res;  
    }

//addend
```

至于`fart.py`的那两千多行代码，主要就是看那几个类即可，无非是各种`parser`的叠加和组合。

[![](https://p3.ssl.qhimg.com/t01f52530ad028cb93f.png)](https://p3.ssl.qhimg.com/t01f52530ad028cb93f.png)

感觉不一定要用`py`写，如果用 [`dexlib2`](https://github.com/JesusFreke/smali/tree/master/dexlib2)之类的库可以更加方便，甚至直接合成`dex`，有心人可以玩一玩。

虽然大家没有`8.1.0`的源码，但是其实大佬公开的`6.0`的源码原理也是一模一样的，欢迎大家使用`FART`脱壳机，并且跟我们多多交流哈。