> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.seeflower.dev](https://blog.seeflower.dev/archives/79/)

> 尝试使用 frida 追踪 APP 执行的每一条 smali，找到 hook 点之后需要调用 libart.so 中的方法进行转换这样可以得到可阅读的字符串内容，起初是想 frida 通过调用获取，但是感觉麻烦，而且...

尝试使用 frida 追踪 APP 执行的每一条 smali，找到 hook 点之后需要调用 libart.so 中的方法进行转换

这样可以得到可阅读的字符串内容，起初是想 frida 通过调用获取，但是感觉麻烦，而且性能上有一些损失

用过 [frida_fart](https://github.com/hanbinglengyue/FART) 的人都知道寒冰老师是自己写了一个 so，即`libext(64).so`

加上是在高版本系统上，然后就想自己写个 native 库，然后 frida 传指针进行调用

然后就进入了一些列踩坑之旅... 最终当然是没成

先把相关结论写在前面

*   安卓**高版本 (N 之后？)** 上只有部分系统 so 库可以直接被调用，用 frida 载入的 so 如果要用 libart.so 中的导出函数是办不到的，除非修改`public.libraries.txt`

可以查看手机 / 系统镜像中的`public.libraries.txt`，Pixel 4 Android 11 的信息如下

```
flame:/ # cat /system/etc/public.libraries.txt# See https://android.googlesource.com/platform/ndk/+/master/docs/PlatformApis.mdlibandroid.solibaaudio.solibamidi.solibbinder_ndk.solibc.solibcamera2ndk.solibdl.solibEGL.solibGLESv1_CM.solibGLESv2.solibGLESv3.solibicui18n.solibicuuc.solibjnigraphics.soliblog.solibmediandk.solibm.solibnativewindow.solibneuralnetworks.so nopreloadlibOpenMAXAL.solibOpenSLES.solibRS.solibstdc++.solibsync.solibvulkan.solibwebviewchromium_plat_support.solibz.so
```

或者查看 ndk 中的`toolchains`，例如`android-ndk-r20b`

```
$ ls -al ./toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/aarch64-linux-android/29total 18584drwxr-xr-x  2 kali kali     4096 Oct 17  2019 .drwxr-xr-x 10 kali kali     4096 Oct 17  2019 ..-rw-r--r--  1 kali kali     3416 Oct 17  2019 crtbegin_dynamic.o-rw-r--r--  1 kali kali     3304 Oct 17  2019 crtbegin_so.o-rw-r--r--  1 kali kali     3416 Oct 17  2019 crtbegin_static.o-rw-r--r--  1 kali kali     1128 Oct 17  2019 crtend_android.o-rw-r--r--  1 kali kali      664 Oct 17  2019 crtend_so.o-rwxr-xr-x  1 kali kali    22120 Oct 17  2019 libaaudio.so-rwxr-xr-x  1 kali kali    13624 Oct 17  2019 libamidi.so-rwxr-xr-x  1 kali kali    80760 Oct 17  2019 libandroid.so-rwxr-xr-x  1 kali kali    31584 Oct 17  2019 libbinder_ndk.so-rw-r--r--  1 kali kali       28 Oct 17  2019 libc++.a-rw-r--r--  1 kali kali 15017040 Oct 17  2019 libc.a-rwxr-xr-x  1 kali kali    28384 Oct 17  2019 libcamera2ndk.so-rw-r--r--  1 kali kali    27538 Oct 17  2019 libcompiler_rt-extras.a-rw-r--r--  1 kali kali       19 Oct 17  2019 libc++.so-rwxr-xr-x  1 kali kali   290520 Oct 17  2019 libc.so-rw-r--r--  1 kali kali     5600 Oct 17  2019 libdl.a-rwxr-xr-x  1 kali kali    13176 Oct 17  2019 libdl.so-rwxr-xr-x  1 kali kali    28096 Oct 17  2019 libEGL.so-rwxr-xr-x  1 kali kali    71352 Oct 17  2019 libGLESv1_CM.so-rwxr-xr-x  1 kali kali    55192 Oct 17  2019 libGLESv2.so-rwxr-xr-x  1 kali kali   106096 Oct 17  2019 libGLESv3.so-rwxr-xr-x  1 kali kali    11600 Oct 17  2019 libjnigraphics.so-rwxr-xr-x  1 kali kali    12456 Oct 17  2019 liblog.so-rw-r--r--  1 kali kali  2158918 Oct 17  2019 libm.a-rwxr-xr-x  1 kali kali    73792 Oct 17  2019 libmediandk.so-rwxr-xr-x  1 kali kali    63200 Oct 17  2019 libm.so-rwxr-xr-x  1 kali kali    15288 Oct 17  2019 libnativewindow.so-rwxr-xr-x  1 kali kali    20304 Oct 17  2019 libneuralnetworks.so-rwxr-xr-x  1 kali kali    16520 Oct 17  2019 libOpenMAXAL.so-rwxr-xr-x  1 kali kali    17920 Oct 17  2019 libOpenSLES.so-rw-r--r--  1 kali kali    25296 Oct 17  2019 libstdc++.a-rwxr-xr-x  1 kali kali    13408 Oct 17  2019 libstdc++.so-rwxr-xr-x  1 kali kali    11704 Oct 17  2019 libsync.so-rwxr-xr-x  1 kali kali    53216 Oct 17  2019 libvulkan.so-rw-r--r--  1 kali kali   621802 Oct 17  2019 libz.a-rwxr-xr-x  1 kali kali    30120 Oct 17  2019 libz.so
```

也就是说自行编译 so 的时候，只能添加 ndk 中这些 so，即系统 so

如果通过指定预编译 so 来设置依赖不在列表中的系统 so，到时候载入时会提示`dlopen failed: cannot locate symbol`

那就没有办法了吗？当然是有的，可以通过偏移的方式去调用，这样 so 就不用依赖系统库了

如果用偏移的方案当然 frida 也就可以完成了，但是类型转换可能还是有些麻烦

所以最佳方案还是编译 so，然后 frida 传指针，这样兼顾灵活性和性能

相关参考

*   [https://blog.csdn.net/hanhan1016/article/details/111908771](https://blog.csdn.net/hanhan1016/article/details/111908771)

冰冰老师一开始就说用偏移，但是我还是头铁踩坑...

[![](https://blog.seeflower.dev/images/Snipaste_2021-11-21_13-21-51.png)](https://blog.seeflower.dev/images/Snipaste_2021-11-21_13-21-51.png)

大佬的经验都是在踩坑过的基础上得出的，所以大家不要像我这样去踩坑，浪费时间... 又被群友拉开差距...

偏移方案还在实践中，这里对踩坑做下记录

记录
--

因为要对关键函数位置定位，而关键函数又是 inline 函数，所以最好是编译下系统然后把`libart.so`拿出来和手机上的对比下

例如这样插入一些代码

[![](https://blog.seeflower.dev/images/20211121135104.png)](https://blog.seeflower.dev/images/20211121135104.png)

当然也可以直接和源代码进行对比分析，只是这样比较痛苦而且有可能不准确

当然我已经这样尝试过，只是 hook 代码组合有些问题，所以最开始没成

用的是 [android-11.0.0_r21](http://aospxref.com/android-11.0.0_r21) 做的对比

[![](https://blog.seeflower.dev/images/FgJpPkxFxqgSaRpy7eqgTZD3Q3kL.png)](https://blog.seeflower.dev/images/FgJpPkxFxqgSaRpy7eqgTZD3Q3kL.png)

[![](https://blog.seeflower.dev/images/Frh1KOnniFlGvD-yOByS3pENdkj0.png)](https://blog.seeflower.dev/images/Frh1KOnniFlGvD-yOByS3pENdkj0.png)

然后我就开始编译`android-11.0.0_r21`了

官方[环境指南](https://source.android.google.cn/setup/build/initializing)

```
apt-get install gnupg flex bison build-essential curl zlib1g-dev gcc-multilib g++-multilib libc6-dev-i386 lib32ncurses-dev x11proto-core-dev libx11-dev lib32z1-dev libgl1-mesa-dev libxml2-utils xsltproc fontconfig
```

*   对于 kali，需要把 lib32ncurses5-dev 修改为 lib32ncurses-dev

参考`AOSP(Android) 镜像使用帮助`

*   [https://lug.ustc.edu.cn/wiki/mirrors/help/aosp/](https://lug.ustc.edu.cn/wiki/mirrors/help/aosp/)

配置好 repo，然后同步代码

```
proxychains repo init -u git://mirrors.ustc.edu.cn/aosp/platform/manifest -b android-11.0.0_r21 --depth=1repo sync -c -j8
```

直到出现

```
repo sync has finished successfully.
```

这里记录一点，就是 kali 2021 中，由于用户是普通用户，在使用 sudo 挂载分区后，普通用户是不能直接创建文件和文件夹的

所以这个时候得`chmod -R 777 挂载点`，然后才能正常同步源代码

再记录一点，起初我尝试把新加的分区合并到`/dev/sda1`，重新划分 extend 磁盘和 swap 分区，但是虚拟机下系统启动就会贼慢，即使通过修改`/etc/fstab`中`uuid`的内容，依然是这样，所以最后还是普通挂载使用

然后编译系统

```
export LC_ALL=C && . build/envsetup.shlunchm
```

*   `lunch`选的是`aosp_flame_userdebug`
*   没错，不需要设置 java 环境，kali 自带 java11，高版本也能直接编译 pass（至于编译时具体用的是不是系统的 java 就不知道了
*   请记得准备驱动
*   python 优先级似乎也不用管，可能是 kali 2021.3 的原因
*   编译建议至少 16G 内存起

**libart.so 路径**

*   aosp11/out/target/product/flame/apex/com.android.art.debug/lib64/libart.so
*   aosp11/out/target/product/flame/apex/com.android.art.debug/lib/libart.so

然后才是本文正题，自定义调用系统库，并编译

自定义代码主要内容

```
const char* PrettyMethod(const ShadowFrame& shadow_frame) {    std::ostringstream oss;    oss << shadow_frame.GetMethod()->PrettyMethod() << "\n";    return oss.str().c_str();}
```

一开始我尝试在原有`Android.bp`中修改，添加相关内容，最终均未成功，有以下几种情况

*   编译没问题，但是没有`libsfxhelp.so`产物
*   编译异常，显示为一些看不懂的配置错误

查阅了一些资料，发现 aosp 这里面的处理流程比较复杂，加上不了解相关知识，于是放弃随系统编译生成自定义库

改为冰冰老师的建议，将代码单独提取出来，使用 Android Studio 编译

不过在提取代码的过程中发现相关头文件层级关系复杂，改动很麻烦

折腾了半个多小时，放弃单独提取代码，改为在原系统代码文件结构上加上自己的代码，单独配置 ndk 设置进行编译

于是我的代码放在

*   aosp11/art/runtime/interpreter/sfxhelp.cc

内容如下

```
#include <iostream>#include <sstream> #include "art_method-inl.h"#include "shadow_frame-inl.h" extern "C" const char* SFXTraceExecution(const art::ShadowFrame& shadow_frame, const art::Instruction* inst) {    std::ostringstream oss;    oss << shadow_frame.GetMethod()->PrettyMethod()        << inst->DumpString(shadow_frame.GetMethod()->GetDexFile()) << "\n";    return oss.str().c_str();} extern "C" const char* PrettyMethod(const art::ShadowFrame& shadow_frame) {    std::ostringstream oss;    oss << shadow_frame.GetMethod()->PrettyMethod() << "\n";    return oss.str().c_str();} extern "C" const char* DumpString(const art::ShadowFrame& shadow_frame, const art::Instruction* inst) {    std::ostringstream oss;    oss << inst->DumpString(shadow_frame.GetMethod()->GetDexFile()) << "\n";    return oss.str().c_str();}
```

然后编译命令是

```
cd /home/kali/aosp11/home/kali/Desktop/android-ndk-r23b/ndk-build NDK_PROJECT_PATH=. NDK_APPLICATION_MK=Application.mk APP_BUILD_SCRIPT=Android.mk
```

`Application.mk`内容如下

```
APP_ABI := allAPP_STL := c++_staticAPP_PLATFORM := android-21
```

`Android.mk`内容如下

```
LOCAL_PATH := $(call my-dir) include $(CLEAR_VARS)LOCAL_MODULE := sfxhelp#$(warning "the value of LOCAL_PATH is $(LOCAL_PATH)")  # LOCAL_C_INCLUDES := $(LOCAL_PATH)/includeLOCAL_C_INCLUDES += $(LOCAL_PATH)/art/runtimeLOCAL_C_INCLUDES += $(LOCAL_PATH)/art/libartbaseLOCAL_C_INCLUDES += $(LOCAL_PATH)/art/libdexfileLOCAL_C_INCLUDES += $(LOCAL_PATH)/bionic/libc/platformLOCAL_C_INCLUDES += $(LOCAL_PATH)/system/core/base/includeLOCAL_SRC_FILES += $(LOCAL_PATH)/art/runtime/interpreter/sfxhelp.cc #$(warning "lc $(LOCAL_SRC_FILES)") LOCAL_CFLAGS += -DART_STACK_OVERFLOW_GAP_arm=8192LOCAL_CFLAGS += -DART_STACK_OVERFLOW_GAP_arm64=16384LOCAL_CFLAGS += -DART_STACK_OVERFLOW_GAP_x86=16384LOCAL_CFLAGS += -DART_STACK_OVERFLOW_GAP_x86_64=20480LOCAL_CFLAGS += -DART_DEFAULT_GC_TYPE_IS_CMSLOCAL_CFLAGS += -DIMT_SIZE=43 LOCAL_CPPFLAGS += -std=c++17LOCAL_LDLIBS += -ldl -Wl,--unresolved-symbols=ignore-all include $(BUILD_SHARED_LIBRARY)
```

编译产物路径如下，即`Application.mk`所在目录下的`libs`文件夹内

*   /home/kali/aosp11/libs/arm64-v8a/libsfxhelp.so

上面是能正确编译出结果的配置，下面是一些踩坑记录

**踩坑点**

```
error: no template named 'enable_if_t' in namespace 'std'; did you mean 'enable_if'
```

[![](https://blog.seeflower.dev/images/20211121141652.png)](https://blog.seeflower.dev/images/20211121141652.png)

网上说要用`c++11`以上就不会有这个问题

*   [https://stackoverflow.com/questions/64281680/error-no-template-named-enable-if-t-in-namespace-std-did-you-mean-enable](https://stackoverflow.com/questions/64281680/error-no-template-named-enable-if-t-in-namespace-std-did-you-mean-enable)

改成`c++11`后发现还是有，下面这样的报错

```
error: no template named 'is_base_of_v' in namespace 'std'; did you mean 'is_base_of'
```

最终测试了几遍，发现得`c++17`

然后遇到下面这种未定义的错误

```
error: use of undeclared identifier 'ART_STACK_OVERFLOW_GAP_arm'
```

[![](https://blog.seeflower.dev/images/20211121142135.png)](https://blog.seeflower.dev/images/20211121142135.png)

最终找到这些主要是在

*   aosp11/art/build/art.go 中配置的

[![](https://blog.seeflower.dev/images/Snipaste_2021-11-21_14-25-02.png)](https://blog.seeflower.dev/images/Snipaste_2021-11-21_14-25-02.png)

然后这样`LOCAL_CFLAGS += -DART_STACK_OVERFLOW_GAP_arm=8192`设置即可

然后新的错误是

```
ld: error: undefined symbol: art::ArtMethod::PrettyMethod(bool)
```

[![](https://blog.seeflower.dev/images/20211121142544.png)](https://blog.seeflower.dev/images/20211121142544.png)

这主要是最开始我没有引入头文件导致的（好像是，记不太清楚了

```
#include "art_method-inl.h"
```

然后编译出产物了

[![](https://blog.seeflower.dev/images/20211121142853.png)](https://blog.seeflower.dev/images/20211121142853.png)

接着尝试使用 frida 载入 so，提示找不到对应的符号...

[![](https://blog.seeflower.dev/images/20211121142946.png)](https://blog.seeflower.dev/images/20211121142946.png)

这是为什么呢？原因是，虽然编译成功，但是调用的符号本身相关的代码没有被包含进来

即自定义 so 应当依赖 libart.so，这样才能找到符号，我尝试了两种方案

一种是通过这样的命令让编译 so 依赖系统库

```
LOCAL_LDLIBS += -lart
```

但是最开始提到，ndk 下

*   `toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/aarch64-linux-android/29`

这样的目录下是没有 libart.so 的，所以我把`libart.so`给每一个上面这样的目录都放一个 libart.so

然后上面的`LOCAL_LDLIBS += -lart`就能正常工作了

这样编译出的 so 就会依赖 libart.so

另一个方案是参考下面的文章

*   [https://blog.csdn.net/shulianghan/article/details/104288881](https://blog.csdn.net/shulianghan/article/details/104288881)

即添加这样一段配置，就可以向自定义 so 中添加要依赖的 so，libart.so 文件需要放在配置同一级目录

```
include $(CLEAR_VARS)LOCAL_MODULE := artLOCAL_SRC_FILES := libart.soinclude $(PREBUILT_SHARED_LIBRARY) ... LOCAL_SHARED_LIBRARIES := art ...
```

不过这样编译出来，实际上和前面的结果是一样的

而且即使这样，frida 也是不能使用的，会提示打开失败，这就是文章开头说到的问题，高版本不支持直接用系统库

[![](https://blog.seeflower.dev/images/20211121144219.png)](https://blog.seeflower.dev/images/20211121144219.png)

后来我又想，感觉直接把编译的 libart.so 也用 frida 载入好了，但是新的问题又有了，需要更多的依赖库...

[![](https://blog.seeflower.dev/images/20211121143634.png)](https://blog.seeflower.dev/images/20211121143634.png)

果然我还是 **too young too simple**

*   不听老人言，吃亏在眼前

[![](https://blog.seeflower.dev/images/Snipaste_2021-11-21_14-44-55.png)](https://blog.seeflower.dev/images/Snipaste_2021-11-21_14-44-55.png)

*   [https://blog.seeflower.dev/archives/12/](https://blog.seeflower.dev/archives/12/)
*   [https://blog.seeflower.dev/archives/13/](https://blog.seeflower.dev/archives/13/)
*   [https://www.jianshu.com/p/9aab51f4cd6f](https://www.jianshu.com/p/9aab51f4cd6f)
*   [https://www.jianshu.com/p/f69d1c381182](https://www.jianshu.com/p/f69d1c381182)
*   [https://lug.ustc.edu.cn/wiki/mirrors/help/aosp/](https://lug.ustc.edu.cn/wiki/mirrors/help/aosp/)
*   [https://blog.csdn.net/shulianghan/article/details/104288881](https://blog.csdn.net/shulianghan/article/details/104288881)
*   [https://blog.csdn.net/hanhan1016/article/details/111908771](https://blog.csdn.net/hanhan1016/article/details/111908771)
*   [https://stackoverflow.com/questions/50350946/android-ndk-module-depends-on-undefined-modules-log](https://stackoverflow.com/questions/50350946/android-ndk-module-depends-on-undefined-modules-log)