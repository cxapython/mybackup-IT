> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/6b4a80654d4e)

Xposed 是一个很强大的 Android 平台上的 HOOK 工具，而且作者为了方便开发者使用开发了一个 APP（Xposed Installer，下文称为 Installer） 来使用开发者自己开发的模块。开发者安装自己的模块后需要在 Installer 中勾选自己的模块然后重启手机自己的模块才会起作用。但是这样有点不利于开发者测试，每次都要点开 Installer 操作几下尤其是还要重启就显得有点麻烦了。

读过 Xposed 的源码后会发现仅通过更改 **XposedBridge.jar** 的源码就可以更简便一些：  
**1. 不需重启手机  
2. 不需操作 Installer 这个 App，且不用安装 hook 模块，只需 push 到手机即可**

_首先需要下载源码，[rovo89](https://github.com/rovo89) 链接里面有 Xposed 所有源码，我们只需要下载 XposedBridge 就好。原版的 Xposed 并不适用于三星手机，[wanam](https://github.com/wanam) 改过的 Xposed 可以用，所以如果是三星手机的话就下载 wanam 的。_

1. Xposed 原理简介
==============

现在安装 Xposed 比较方便，因为 Xposed 作者开发了一个 Xposed Installer App，下载后按照提示傻瓜式安装（前提是 root 手机）。其实它的安装过程是这个样子的：首先探测手机型号，然后按照手机版本下载不同的刷机包，最后把 Xposed 刷机包刷入手机重启就好。[刷机包下载](https://dl-xda.xposed.info/framework/) 里面有所有版本的刷机包。  
刷机包解压打开里面的问件构成是这个样子的：

```
META-INF/    里面有文件配置脚本 flash-script.sh 配置各个文件安装位置。
system/bin/   替换zygote进程等文件
system/framework/XposedBridge.jar jar包位置
system/lib system/lib64 一些so文件所在位置
xposed.prop xposed版本说明文件
```

所以安装 Xposed 的过程就上把上面这些文件放到手机里相同文件路径下。  
通过查看文件安装脚本发现：  
**system/bin / 下面的文件替换了 app_process 等文件，app_process 就是 zygote 进程文件。所以 Xposed 通过替换 zygote 进程实现了控制手机上所有 app 进程。因为所有 app 进程都是由 Zygote fork 出来的。**  
Xposed 的基本原理是修改了 ART/Davilk 虚拟机，将需要 hook 的函数注册为 Native 层函数。当执行到这一函数是虚拟机会优先执行 Native 层函数，然后再去执行 Java 层函数，这样完成函数的 hook。如下图：  

![](http://upload-images.jianshu.io/upload_images/13364326-d5d0d8280a6c6cfa.png) Xposed HOOK 原理

通过读 Xposed 源码发现其启动过程：

1.  手机启动时 init 进程会启动 zygote 这个进程。由于 zygote 进程文件 app_process 已被替换，所以启动的时 Xposed 版的 zygote 进程。
2.  Xposed_zygote 进程启动后会初始化一些 so 文件（system/lib system/lib64），然后进入 XposedBridge.jar 中的 XposedBridge.main 中初始化 jar 包完成对一些关键 Android 系统函数的 hook。
3.  Hook 则是利用修改过的虚拟机将函数注册为 native 函数。
4.  然后再返回 zygote 中完成原本 zygote 需要做的工作。  
    这只是在宏观层面稍微介绍了下 Xposed，要想详细了解需要读它的源码了。下面两篇写的挺好，要想深入理解的可以看看。  
    [Android Hook 框架 Xposed 原理与源代码分析](https://blog.csdn.net/wxyyxc1992/article/details/17320911)  
    [深入理解 Android 之 Xposed 详解](https://blog.csdn.net/innost/article/details/50461783)

2. Xposed 精简化
=============

上面稍微介绍了下它的原理，下面就介绍如何精简化 Xposed。下面只修改了 XposedBridge.jar 包中的 XposedBridge.java 这个文件，修改完重新 Build apk 然后把 apk 重命名为 XposedBridge.jar 然后替换刷机包中的 jar 包，刷入手机即可。

2.1 取消重启手机
----------

看下 XposedBridge.jar 的源码  
代码文件 de.robv.android.xposed.XposedBridge.java

```
protected static void main(String[] args) {
        // Initialize the Xposed framework and modules
        try {
            if (!hadInitErrors()) {
                initXResources();

                SELinuxHelper.initOnce();
                SELinuxHelper.initForProcess(null);

                runtime = getRuntime();
                XPOSED_BRIDGE_VERSION = getXposedVersion();

                if (isZygote) {
                    XposedInit.hookResources();
                    XposedInit.initForZygote();
                }
               //修改时需注释下面这行代码
                XposedInit.loadModules();//*********load hook 模块*******************
            } else {
                Log.e(TAG, "Not initializing Xposed because of previous errors");
            }
        } catch (Throwable t) {
            Log.e(TAG, "Errors during Xposed initialization", t);
            disableHooks = true;
        }

        // Call the original startup code
        if (isZygote) {  //****代码修改位置****           
            ZygoteInit.main(args);
        } else {
            RuntimeInit.main(args);
        }
    }
```

注意上面的 XposedInit.loadModules() 这个函数，这个函数的作用就是 load hook 模块到进程中。  
因为 zygote 启动时先跑到 java 层 XposeBridge.main 中，在 main 里面有一步操作是将 hook 模块 load 进来，模块加载到 zygote 进程中，zygote fork 所有的 app 进程里面也有这个 hook 模块，所以这个模块可以 hook 任意 app。（编写 hook 模块的第一步就是判断当前的进程名字，如果是要 hook 的进程就 hook，不是则返回）。  
所以修改模块后，**要将模块重新 load zygote 里面必须重启 zygote，要想 zygote 重启就要重启手机了。**  
所以修改的逻辑是不把模块 load 到 zygote 里面，而是 load 到自己想要 hook 的进程里面，这样修改模块后只需重启该进程即可。  
在上面代码的**代码修改位置**添加如下代码, 并将**上面 XposedInit.loadModules() 注释掉即可**。

```
if (isZygote) {
            XposedHelpers.findAndHookMethod("com.android.internal.os.ZygoteConnection", BOOTCLASSLOADER, "handleChildProc",
                    "com.android.internal.os.ZygoteConnection.Arguments",FileDescriptor[].class,FileDescriptor.class,
                    PrintStream.class,new XC_MethodHook() {

                        @Override
                        protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                            // TODO Auto-generated method stub
                            super.afterHookedMethod(param);
                            String processName = (String) XposedHelpers.getObjectField(param.args[0], "niceName");
                            String coperationAppName = "指定进程名称如：com.android.settings";
                            if(processName != null){
                                if(processName.startsWith(coperationAppName)){
                                    log("--------Begin Load Module-------");
                                    XposedInit.loadModules()
                                }
                            }
                        }

                    });
            ZygoteInit.main(args);
        } else {
            RuntimeInit.main(args);
        }
```

2.2 取消操作 Installer APP
----------------------

通过读 Install App 的源码发现其实勾选 hook 模块其实 app 就是把模块的 apk 位置写到一个文件里，等 load 模块时会读取这个文件，从这个文件中的 apk 路径下把 apk load 到进程中。  
看下 loadmodules 的源码

```
XposedInit.java
/**
     * Try to load all modules defined in <code>BASE_DIR/conf/modules.list</code>
     */
    /*package*/ static void loadModules() throws IOException {
        final String filename = BASE_DIR + "conf/modules.list";
        BaseService service = SELinuxHelper.getAppDataFileService();
        if (!service.checkFileExists(filename)) {
            Log.e(TAG, "Cannot load any modules because " + filename + " was not found");
            return;
        }

        ClassLoader topClassLoader = XposedBridge.BOOTCLASSLOADER;
        ClassLoader parent;
        while ((parent = topClassLoader.getParent()) != null) {
            topClassLoader = parent;
        }

        InputStream stream = service.getFileInputStream(filename);
        BufferedReader apks = new BufferedReader(new InputStreamReader(stream));
        String apk;
        while ((apk = apks.readLine()) != null) {
            loadModule(apk, topClassLoader);
        }
        apks.close();
    }
```

apk 配置文件就是 installer app 文件路径下的 conf/modules.list 这个文件 data/data/de.robv.android.xposed.installer/conf/modules.list  
或者 data/user_de/0/de.robv.android.xposed.installer/conf/modules.list（不同手机版本位置不同）  
所以我们改下，直接给他个确定的 apk 路径及名称。就定为 / data/local/tmp/module.apk。  
所以再把之前的代码改成如下的样子就好了

```
if (isZygote) {
            XposedHelpers.findAndHookMethod("com.android.internal.os.ZygoteConnection", BOOTCLASSLOADER, "handleChildProc",
                    "com.android.internal.os.ZygoteConnection.Arguments",FileDescriptor[].class,FileDescriptor.class,
                    PrintStream.class,new XC_MethodHook() {

                        @Override
                        protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                            // TODO Auto-generated method stub
                            super.afterHookedMethod(param);
                            String processName = (String) XposedHelpers.getObjectField(param.args[0], "niceName");
                            String coperationAppName = "指定进程名称如：com.android.settings";
                            if(processName != null){
                                if(processName.startsWith(coperationAppName)){
                                    log("--------Begin Load Module-------");
                                    String path = "/data/local/tmp/module.apk";
                                    //注意由loadModules换成了loadModule，记得改
                                    XposedInit.loadModule(path, BOOTCLASSLOADER);

                                }
                            }
                        }

                    });
            ZygoteInit.main(args);
        } else {
            RuntimeInit.main(args);
        }
```

**最后， 同学点个赞吧！！！ 加个关注好么**  
所以每次修改模块后直接修改名字为 module.apk 然后 push 到 / data/local/tmp / 下，然后重启 app 就好。