> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38851536/article/details/118000259)

### 一、前言

一个较好的综合性样本。本篇抛砖引玉分析一下。

### 二、准备

这是我们的目标方法

![](https://img-blog.csdnimg.cn/20210617192735110.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

参数 1 是 203

参数 2 是一个[对象数组](https://so.csdn.net/so/search?from=pc_blog_highlight&q=%E5%AF%B9%E8%B1%A1%E6%95%B0%E7%BB%84)

*   9b69f861-e054-4bc4-9daf-d36ae205ed3e (String)
*   GET /aggroup/homepage/display __xxxxx（byte 数组形式）
*   2 （int 包装类）

### 三、Unidbg 模拟执行

首先搭一个架子

```
package com.lession10;

import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.api.PackageInfo;
import com.github.unidbg.linux.android.dvm.api.Signature;
import com.github.unidbg.linux.android.dvm.array.ArrayObject;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.linux.android.dvm.wrapper.DvmInteger;
import com.github.unidbg.memory.Memory;

import java.io.File;

public class NBridge extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;


    NBridge(){
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.meituan").build();
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析

        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\lession7\\mt.apk")); // 创建Android虚拟机
        vm.setVerbose(true); // 设置是否打印Jni调用细节
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\lession7\\libmtguard.so"), true);

        module = dm.getModule(); //

        vm.setJni(this);
        dm.callJNI_OnLoad(emulator);
    }

    public static void main(String[] args) {
        NBridge test = new NBridge();
    }
}
```

![](https://img-blog.csdnimg.cn/20210617192752197.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

从日志看，主要做了函数的动态绑定，接下来执行目标方法

```
package com.lession7;

import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.api.PackageInfo;
import com.github.unidbg.linux.android.dvm.api.Signature;
import com.github.unidbg.linux.android.dvm.array.ArrayObject;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.linux.android.dvm.wrapper.DvmInteger;
import com.github.unidbg.memory.Memory;

import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

public class NBridge extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;


    NBridge(){
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.meituan").build();
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析

        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\lession7\\mt.apk")); // 创建Android虚拟机
        vm.setVerbose(true); // 设置是否打印Jni调用细节
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\lession7\\libmtguard.so"), true);

        module = dm.getModule(); //

        vm.setJni(this);
        dm.callJNI_OnLoad(emulator);
    }

    public static void main(String[] args) {
        NBridge test = new NBridge();
        test.main203();
    }

    public String main203(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
        list.add(203);
        StringObject input2_1 = new StringObject(vm, "9b69f861-e054-4bc4-9daf-d36ae205ed3e");
        ByteArray input2_2 = new ByteArray(vm, "GET /aggroup/homepage/display __r0ysue".getBytes(StandardCharsets.UTF_8));
        DvmInteger input2_3 = DvmInteger.valueOf(vm, 2);
        vm.addLocalObject(input2_1);
        vm.addLocalObject(input2_2);
        vm.addLocalObject(input2_3);
        // 完整的参数2
        list.add(vm.addLocalObject(new ArrayObject(input2_1, input2_2, input2_3)));
        Number number = module.callFunction(emulator, 0x5a38d, list.toArray())[0];
        return vm.getObject(number.intValue()).getValue().toString();
    };
}
```

这边的注意点有 2

1. 参数如何构造

2. 我没有用 Unidbg 封装的方式去 call，其实那样可以简单一些，但是！地址！yyds！

![](https://img-blog.csdnimg.cn/20210617192805938.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

非常快的就报错退出了

这个时候使用 JNItrace + Frida Call 的方式看一下真实样本中的 JNI 执行流。

```
/* TID 12169 */
 134794 ms [+] JNIEnv->GetObjectArrayElement
 134794 ms |- JNIEnv*          : 0xe385fdc0
 134794 ms |- jobjectArray     : 0xd1c48a40
 134794 ms |- jsize            : 0
 134794 ms |= jobject          : 0x11

 134794 ms ----------------------------Backtrace----------------------------
 134794 ms |-> 0xc64dfeab: libmtguard.so!0x5beab (libmtguard.so:0xc6484000)


           /* TID 12169 */
 135008 ms [+] JNIEnv->GetObjectArrayElement
 135008 ms |- JNIEnv*          : 0xe385fdc0
 135008 ms |- jobjectArray     : 0xd1c48a40
 135008 ms |- jsize            : 1
 135008 ms |= jobject          : 0x21

 135008 ms ----------------------------Backtrace----------------------------
 135008 ms |-> 0xc64df07b: libmtguard.so!0x5b07b (libmtguard.so:0xc6484000)

           /* TID 12169 */
 135223 ms [+] JNIEnv->GetObjectArrayElement
 135223 ms |- JNIEnv*          : 0xe385fdc0
 135223 ms |- jobjectArray     : 0xd1c48a40
 135223 ms |- jsize            : 2
 135223 ms |= jobject          : 0x35

 135223 ms ----------------------------Backtrace----------------------------
 135223 ms |-> 0xc64df555: libmtguard.so!0x5b555 (libmtguard.so:0xc6484000)

           /* TID 12169 */
 135434 ms [+] JNIEnv->GetObjectClass
 135434 ms |- JNIEnv*          : 0xe385fdc0
 135434 ms |- jobject          : 0x35
 135434 ms |= jclass           : 0x49

 135434 ms ----------------------------Backtrace----------------------------
 135434 ms |-> 0xc64df24d: libmtguard.so!0x5b24d (libmtguard.so:0xc6484000)

           /* TID 12169 */
 135602 ms [+] JNIEnv->GetMethodID
 135602 ms |- JNIEnv*          : 0xe385fdc0
 135602 ms |- jclass           : 0x49
 135602 ms |- char*            : 0xc6569650
 135602 ms |:     intValue
 135602 ms |- char*            : 0xc6569659
 135602 ms |:     ()I
 135602 ms |= jmethodID        : 0x708253b0    { intValue()I }

 135602 ms ----------------------------Backtrace----------------------------
 135602 ms |-> 0xc64df321: libmtguard.so!0x5b321 (libmtguard.so:0xc6484000)

           /* TID 12169 */
 135700 ms [+] JNIEnv->ExceptionCheck
 135700 ms |- JNIEnv*          : 0xe385fdc0
 135700 ms |= jboolean         : 0    { false }

 135700 ms ----------------------------Backtrace----------------------------
 135700 ms |-> 0xc64efa1f: libmtguard.so!0x6ba1f (libmtguard.so:0xc6484000)


           /* TID 12169 */
 135797 ms [+] JNIEnv->CallIntMethodV
 135797 ms |- JNIEnv*          : 0xe385fdc0
 135797 ms |- jobject          : 0x35
 135797 ms |- jmethodID        : 0x708253b0    { intValue()I }
 135797 ms |- va_list          : 0xd1c489bc
 135797 ms |= jint             : 2

 135797 ms ----------------------------Backtrace----------------------------
 135797 ms |-> 0xc64e0369: libmtguard.so!0x5c369 (libmtguard.so:0xc6484000)

           /* TID 12169 */
 135991 ms [+] JNIEnv->GetStringUTFChars
 135991 ms |- JNIEnv*          : 0xe385fdc0
 135991 ms |- jstring          : 0x11
 135991 ms |- jboolean*        : 0x0
 135991 ms |= char*            : 0xc804d4b8

 135991 ms ----------------------------Backtrace----------------------------
 135991 ms |-> 0xc64eb2c1: libmtguard.so!0x672c1 (libmtguard.so:0xc6484000)

           /* TID 12169 */
 136088 ms [+] JNIEnv->ReleaseStringUTFChars
 136088 ms |- JNIEnv*          : 0xe385fdc0
 136088 ms |- jstring          : 0xc804d4b8
 136088 ms |- char*            : 0xc804d4b8
 136088 ms |:     9b69f861-e054-4bc4-9daf-d36ae205ed3e

 136088 ms ----------------------------Backtrace----------------------------
 136088 ms |-> 0xc64e8f07: libmtguard.so!0x64f07 (libmtguard.so:0xc6484000)

           /* TID 12169 */
 136179 ms [+] JNIEnv->GetArrayLength
 136179 ms |- JNIEnv*          : 0xe385fdc0
 136179 ms |- jarray           : 0x21
 136179 ms |= jsize            : 754

 136179 ms ----------------------------Backtrace----------------------------
 136179 ms |-> 0xc64df6ef: libmtguard.so!0x5b6ef (libmtguard.so:0xc6484000)


           /* TID 12169 */
 136369 ms [+] JNIEnv->GetByteArrayRegion
 136369 ms |- JNIEnv*          : 0xe385fdc0
 136369 ms |- jbyteArray       : 0x21
 136369 ms |- jsize            : 0
 136369 ms |- jsize            : 754
 136369 ms |- jbyte*           : 0xc8037700
 136369 ms |:     0000000: 47 45 54 20 2F 61 67 67  72 6F 75 70 2F 68 6F 6D  GET /aggroup/hom
 136369 ms |:     0000010: 65 70 61 67 65 2F 64 69  73 70 6C 61 79 20 5F 5F  epage/display __
 136369 ms |:     0000020: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx


 136369 ms ----------------------------Backtrace----------------------------
 136369 ms |-> 0xc64df12b: libmtguard.so!0x5b12b (libmtguard.so:0xc6484000)


           /* TID 12169 */
 136479 ms [+] JNIEnv->ExceptionCheck
 136479 ms |- JNIEnv*          : 0xe385fdc0
 136479 ms |= jboolean         : 0    { false }

 136479 ms ----------------------------Backtrace----------------------------
 136479 ms |-> 0xc64efa1f: libmtguard.so!0x6ba1f (libmtguard.so:0xc6484000)


           /* TID 12169 */
 136584 ms [+] JNIEnv->FindClass
 136584 ms |- JNIEnv*          : 0xe385fdc0
 136584 ms |- char*            : 0xc656a2b0
 136584 ms |:     java/lang/String
 136584 ms |= jclass           : 0x55    { java/lang/String }

 136584 ms ----------------------------Backtrace----------------------------
 136584 ms |-> 0xc64eb013: libmtguard.so!0x67013 (libmtguard.so:0xc6484000)


           /* TID 12169 */
 136696 ms [+] JNIEnv->ExceptionCheck
 136696 ms |- JNIEnv*          : 0xe385fdc0
 136696 ms |= jboolean         : 0    { false }

 136696 ms ----------------------------Backtrace----------------------------
 136696 ms |-> 0xc64efa1f: libmtguard.so!0x6ba1f (libmtguard.so:0xc6484000)


           /* TID 12169 */
 136811 ms [+] JNIEnv->GetMethodID
 136811 ms |- JNIEnv*          : 0xe385fdc0
 136811 ms |- jclass           : 0x55    { java/lang/String }
 136811 ms |- char*            : 0xc656a2c1
 136811 ms |:     <init>
 136811 ms |- char*            : 0xc656a2c8
 136811 ms |:     ([BLjava/lang/String;)V
 136811 ms |= jmethodID        : 0x708504a4    { <init>([BLjava/lang/String;)V }

 136811 ms ----------------------------Backtrace----------------------------
 136811 ms |-> 0xc64eb097: libmtguard.so!0x67097 (libmtguard.so:0xc6484000)


           /* TID 12169 */
 137033 ms [+] JNIEnv->NewByteArray
 137033 ms |- JNIEnv*          : 0xe385fdc0
 137033 ms |- jsize            : 506
 137033 ms |= jbyteArray       : 0x65

 137033 ms ----------------------------Backtrace----------------------------
 137033 ms |-> 0xc64eb0c5: libmtguard.so!0x670c5 (libmtguard.so:0xc6484000)


           /* TID 12169 */
 137222 ms [+] JNIEnv->SetByteArrayRegion
 137222 ms |- JNIEnv*          : 0xe385fdc0
 137222 ms |- jbyteArray       : 0x65
 137222 ms |- jsize            : 0
 137222 ms |- jsize            : 506
 137222 ms |- jbyte*           : 0xc5a1620c
 137222 ms |:     0000000: 7B 22 61 30 22 3A 22 31  2E 35 22 2C 22 61 31 22  {"a0":"1.5","a1"
 137222 ms |:     0000010: 3A 22 39 62 36 39 66 38  36 31 2D 65 30 35 34 2D  :"9b69f861-e054-
 137222 ms |:     0000020: 34 62 63 34 2D 39 64 61  66 2D 64 33 36 61 65 32  4bc4-9daf-d36ae2
 137222 ms |:     0000030: 30 35 65 64 33 65 22 2C  22 61 32 22 3A 22 34 36  05ed3e","a2":"46
 137222 ms |:     0000040: 30 65 32 30 31 35 30 66  64 37 33 32 36 39 33 66  0e20150fd732693f
 137222 ms |:     0000050: 37 31 31 39 66 39 31 64  61 37 36 35 66 39 35 39  7119f91da765f959
 137222 ms |:     0000060: 65 31 36 66 30 66 22 2C  22 61 33 22 3A 32 2C 22  e16f0f","a3":2,"
 137222 ms |:     0000070: 61 34 22 3A 31 36 31 36  36 35 31 39 34 34 2C 22  a4":1616651944,"
 137222 ms |:     0000080: 61 35 22 3A 22 76 49 79  66 63 38 7A 54 58 51 41  a5":"vIyfc8zTXQA
 137222 ms |:     0000090: 6F 53 58 6B 78 4E 6B 4E  38 74 4E 44 69 36 48 73  oSXkxNkN8tNDi6Hs
 137222 ms |:     00000A0: 76 4C 70 37 5A 6C 6E 70  44 43 38 47 49 77 50 61  vLp7ZlnpDC8GIwPa
 137222 ms |:     00000B0: 4C 72 55 4C 2B 79 71 76  54 74 2B 39 51 76 30 35  LrUL+yqvTt+9Qv05
 137222 ms |:     00000C0: 36 54 33 6D 44 36 52 6B  36 6A 4F 42 42 66 42 37  6T3mD6Rk6jOBBfB7
 137222 ms |:     00000D0: 65 5A 6E 6E 54 71 42 35  56 48 6F 4E 37 4F 33 36  eZnnTqB5VHoN7O36
 137222 ms |:     00000E0: 41 62 73 6B 32 4C 48 42  30 43 4E 42 55 35 34 2B  Absk2LHB0CNBU54+
 137222 ms |:     00000F0: 63 32 63 51 33 6D 50 7A  56 59 70 59 4D 2B 61 5A  c2cQ3mPzVYpYM+aZ
 137222 ms |:     0000100: 42 69 64 4B 72 74 31 79  46 6C 66 2F 42 51 72 68  BidKrt1yFlf/BQrh
 137222 ms |:     0000110: 74 79 4F 66 77 46 65 67  47 33 53 75 4A 4D 35 52  tyOfwFegG3SuJM5R
 137222 ms |:     0000120: 56 4A 6C 71 52 43 4D 53  67 77 67 6D 4C 44 5A 4F  VJlqRCMSgwgmLDZO
 137222 ms |:     0000130: 66 32 7A 32 2F 38 6D 69  72 4B 56 4D 48 66 71 73  f2z2/8mirKVMHfqs
 137222 ms |:     0000140: 68 6C 2B 35 74 52 50 79  7A 46 55 67 30 75 4B 76  hl+5tRPyzFUg0uKv
 137222 ms |:     0000150: 36 50 55 2B 78 54 35 72  43 59 71 55 2B 62 4D 4D  6PU+xT5rCYqU+bMM
 137222 ms |:     0000160: 48 73 70 47 52 55 43 71  44 64 65 79 41 74 6E 6F  HspGRUCqDdeyAtno
 137222 ms |:     0000170: 66 71 4F 4F 55 74 62 4B  31 73 72 4C 36 48 6C 46  fqOOUtbK1srL6HlF
 137222 ms |:     0000180: 31 37 6D 55 67 2F 2F 44  7A 4F 2F 30 6F 51 49 4F  17mUg//DzO/0oQIO
 137222 ms |:     0000190: 35 33 38 6A 49 6B 30 55  67 32 32 34 48 38 6B 37  538jIk0Ug224H8k7
 137222 ms |:     00001A0: 39 57 6F 6A 39 43 75 62  4A 30 52 6B 65 36 54 6C  9Woj9CubJ0Rke6Tl
 137222 ms |:     00001B0: 5A 2F 50 70 72 56 4D 66  71 37 72 4F 34 34 57 64  Z/PprVMfq7rO44Wd
 137222 ms |:     00001C0: 30 22 2C 22 61 36 22 3A  30 2C 22 64 31 22 3A 22  0","a6":0,"d1":"
 137222 ms |:     00001D0: 38 66 65 62 35 34 33 63  30 30 65 36 63 35 64 64  8feb543c00e6c5dd
 137222 ms |:     00001E0: 32 32 33 33 37 39 65 33  31 61 39 63 30 61 34 39  223379e31a9c0a49
 137222 ms |:     00001F0: 31 63 37 36 35 36 62 61  22 7D                    1c7656ba"}

 137222 ms ----------------------------Backtrace----------------------------
 137222 ms |-> 0xc64eb107: libmtguard.so!0x67107 (libmtguard.so:0xc6484000)
```

可以发现，我们在参数还没完全转换完的情况下，Unidbg 就退出了。

这种情况下，可能的原因有很多，但可能性较大的是两个

*   上下文环境缺失

*   样本使用某种手段检测或反制了 Unidbg

先看一下是否是上下文的问题，假设是上下文缺失，通俗的讲就是在 SO 加载后到我们的 main 函数调用前的这段时间里，样本需要调用一些函数对 SO 进行初始化，而我们没有注意也没做这个事，这导致了 Unidbg 无法顺利运行。

既然提出了假设，就需要去验证。

使用 Frida 主动调用 main，可以顺利得到结果，那如果换个时机呢？

我们在目标 SO 的 JNIOnLoad 刚执行完时再尝试一下 call，如果存在初始化函数，这个时机点样本的初始化函数应该也还没来得及运行，call 应该是没有结果的。

为了实现在这个时机点 hook 和 call，我们还需要借助一下 android_dlopen_ext 和 Frida 的 spawn 模式。

```
var android_dlopen_ext = Module.findExportByName(null, "android_dlopen_ext");
if (android_dlopen_ext != null) {
    Interceptor.attach(android_dlopen_ext, {
        onEnter: function (args) {
            this.hook = false;
            var soName = args[0].readCString();
            if (soName.indexOf("libmtguard.so") !== -1) {
                this.hook = true;
            }
        },
        onLeave: function (retval) {
            if (this.hook) {
                var jniOnload = Module.findExportByName("libmtguard.so", "JNI_OnLoad");
                Interceptor.attach(jniOnload, {
                    onEnter: function (args) {
                        console.log("Enter Mtguard JNI OnLoad");
                    },
                    onLeave: function (retval) {
                        console.log("After Mtguard JNI OnLoad");
                        call_mtgsig();
                    }
                });
            }
        }
    });
}

function call_mtgsig() {
    Java.perform(function () {
        var jinteger = Java.use("java.lang.Integer")
        var jstring = Java.use("java.lang.String");
        var NBridge = Java.use("com.meituan.android.common.mtguard.NBridge")
        var objArr = [jstring.$new("9b69f861-e054-4bc4-9daf-d36ae205ed3e"), jstring.$new("GET /aggroup/homepage/display __reqTraceID=5ca01019-fafc-4f92-a80e-82ce1389aab7&abStrategy=d&allowToTrack=1&ci=1&cityId=1&clearTimeStamp=-1&client).getBytes(), jinteger.valueOf(2)]
        var result = NBridge.main(203, objArr);
        console.log("result:"+result);
    })
}
```

```
[Nexus 5X::com.sankuai.meituan]-> %resume
[Nexus 5X::com.sankuai.meituan]-> 
Enter Mtguard JNI OnLoad
After Mtguard JNI OnLoad
result:null
```

返回确实是 null，这一定程度上验证了我们的猜想——上下文环境缺失

但这并不是绝对的原因，也可能是别的原因导致的，逻辑链并没有完全导向我们的猜想。但是相对来说，上下文缺失有重大嫌疑。

在分析和研究商用 APP 而不是 demo 时，我们需要意识到三点

*   构建完全严丝合缝的分析逻辑是很难的，需要猜测和试错
*   构建完全严丝合缝的分析逻辑是很难的，需要猜测和试错
*   构建完全严丝合缝的分析逻辑是很难的，需要猜测和试错

我们初步确认存在某种初始化操作，接下来尝试去找它。先试一下偷懒的办法，如果不成再用 Jadx 把相关的类仔细看一遍。

能做初始化验证，并导致单独执行 main 函数无法返回结果的，大概率是 native 函数。以下是 Unibdg 日志中显示的 Libmtguard 中注册的 native 函数。

```
JNIEnv->FindClass(com/meituan/android/common/mtguard/NBridge) was called from RX@0x400039f9[libmtguard.so]0x39f9
JNIEnv->RegisterNatives(com/meituan/android/common/mtguard/NBridge, RW@0x400e1004[libmtguard.so]0xe1004, 1) was called from RX@0x40003947[libmtguard.so]0x3947
RegisterNative(com/meituan/android/common/mtguard/NBridge, main(I[Ljava/lang/Object;)[Ljava/lang/Object;, RX@0x4005a38d[libmtguard.so]0x5a38d)
JNIEnv->FindClass(com/meituan/android/common/mtguard/NBridge$SIUACollector) was called from RX@0x40003a99[libmtguard.so]0x3a99
JNIEnv->RegisterNatives(com/meituan/android/common/mtguard/NBridge$SIUACollector, RW@0x4031305c, 10) was called from RX@0x40003acf[libmtguard.so]0x3acf
RegisterNative(com/meituan/android/common/mtguard/NBridge$SIUACollector, getHWProperty()Ljava/lang/String;, RX@0x40008ea9[libmtguard.so]0x8ea9)
RegisterNative(com/meituan/android/common/mtguard/NBridge$SIUACollector, getEnvironmentInfoExtra()Ljava/lang/String;, RX@0x4000567d[libmtguard.so]0x567d)
RegisterNative(com/meituan/android/common/mtguard/NBridge$SIUACollector, getEnvironmentInfo()Ljava/lang/String;, RX@0x40005379[libmtguard.so]0x5379)
RegisterNative(com/meituan/android/common/mtguard/NBridge$SIUACollector, getHWStatus()Ljava/lang/String;, RX@0x40018e05[libmtguard.so]0x18e05)
RegisterNative(com/meituan/android/common/mtguard/NBridge$SIUACollector, getHWEquipmentInfo()Ljava/lang/String;, RX@0x40026dcd[libmtguard.so]0x26dcd)
RegisterNative(com/meituan/android/common/mtguard/NBridge$SIUACollector, getExternalEquipmentInfo()Ljava/lang/String;, RX@0x40033971[libmtguard.so]0x33971)
RegisterNative(com/meituan/android/common/mtguard/NBridge$SIUACollector, getUserAction()Ljava/lang/String;, RX@0x4003e4a9[libmtguard.so]0x3e4a9)
RegisterNative(com/meituan/android/common/mtguard/NBridge$SIUACollector, getPlatformInfo()Ljava/lang/String;, RX@0x400447f1[libmtguard.so]0x447f1)
RegisterNative(com/meituan/android/common/mtguard/NBridge$SIUACollector, getLocationInfo()Ljava/lang/String;, RX@0x4004dc3d[libmtguard.so]0x4dc3d)
RegisterNative(com/meituan/android/common/mtguard/NBridge$SIUACollector, startCollection()Ljava/lang/String;, RX@0x40058e75[libmtguard.so]0x58e75)
JNIEnv->FindClass(com/meituan/android/common/dfingerprint/v3/DFPTest) was called from RX@0x4006b999[libmtguard.so]0x6b999
JNIEnv->RegisterNatives(com/meituan/android/common/dfingerprint/v3/DFPTest, RW@0x400e6650[libmtguard.so]0xe6650, 1) was called from RX@0x4006b9b7[libmtguard.so]0x6b9b7
RegisterNative(com/meituan/android/common/dfingerprint/v3/DFPTest, interface0(I[Ljava/lang/Object;)Ljava/lang/String;, RX@0x4006b82d[libmtguard.so]0x6b82d)
```

这里面所有函数都有嫌疑，甚至是 main 函数自己，甚至应该说，main 函数的嫌疑最大。因为我们的参数 1 是 203，可是参数 1 的潜在选择可不止这一种，或许其中某一个数作为参数 1 时，就充当着”激活函数 “或者叫” 初始化函数“。

![](https://img-blog.csdnimg.cn/20210617192900107.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

所以，我们需要在 JNIOnLoad 结束之后的这个时机做更多事

1. 对所有动态注册的函数，在 JAVA 层进行 HOOK，看一下到底是哪些函数，在 SO 刚加载进来就开始执行。

2.Call main203，在每一个 Hook 触发的时候 call main203

简而言之，我们想知道在谁之后，call main 就可以顺利执行，在这之前的所有调用就是初始化函数。

```
var android_dlopen_ext = Module.findExportByName(null, "android_dlopen_ext");
if (android_dlopen_ext != null) {
    Interceptor.attach(android_dlopen_ext, {
        onEnter: function (args) {
            this.hook = false;
            var soName = args[0].readCString();
            if (soName.indexOf("libmtguard.so") !== -1) {
                this.hook = true;
            }
        },
        onLeave: function (retval) {
            if (this.hook) {
                var jniOnload = Module.findExportByName("libmtguard.so", "JNI_OnLoad");
                Interceptor.attach(jniOnload, {
                    onEnter: function (args) {
                        console.log("Enter Mtguard JNI OnLoad");
                    },
                    onLeave: function (retval) {
                        console.log("After Mtguard JNI OnLoad");
                        hook_mtso();
                    }
                });
            }
        }
    });
}


function hook_mtso() {
    Java.perform(function () {
        Java.use("com.meituan.android.common.mtguard.NBridge").main.implementation = function(arg1, arg2) {
            console.log("call com/meituan/android/common/mtguard/NBridge, main(I[Ljava/lang/Object;)[Ljava/lang/Object;");
            call_mtgsig();
            return this.main(arg1, arg2);
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getHWProperty.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getHWProperty()Ljava/lang/String;");
            call_mtgsig();
            return this.getHWProperty();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getEnvironmentInfo.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getEnvironmentInfo()Ljava/lang/String;");
            call_mtgsig();
            return this.getEnvironmentInfo();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getEnvironmentInfoExtra.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getEnvironmentInfoExtra()Ljava/lang/String;");
            call_mtgsig();
            return this.getEnvironmentInfoExtra();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getHWStatus.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getHWStatus()Ljava/lang/String;");
            call_mtgsig();
            return this.getHWStatus();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getHWEquipmentInfo.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getHWEquipmentInfo()Ljava/lang/String;");
            call_mtgsig();
            return this.getHWEquipmentInfo();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getUserAction.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getUserAction()Ljava/lang/String;");
            call_mtgsig();
            return this.getUserAction();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getPlatformInfo.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getPlatformInfo()Ljava/lang/String;");
            call_mtgsig();
            return this.getPlatformInfo();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getLocationInfo.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getLocationInfo()Ljava/lang/String;");
            call_mtgsig();
            return this.getLocationInfo();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").startCollection.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, startCollection()Ljava/lang/String;");
            call_mtgsig();
            return this.startCollection();
        }

        Java.use("com.meituan.android.common.mtguard.NBridge$SIUACollector").getExternalEquipmentInfo.implementation = function() {
            console.log("com/meituan/android/common/mtguard/NBridge$SIUACollector, getExternalEquipmentInfo()Ljava/lang/String;");
            call_mtgsig();
            return this.getExternalEquipmentInfo();
        }
    })
}


function call_mtgsig() {
    Java.perform(function () {
        var jinteger = Java.use("java.lang.Integer")
        var jstring = Java.use("java.lang.String");
        var NBridge = Java.use("com.meituan.android.common.mtguard.NBridge")
        var objArr = [jstring.$new("9b69f861-e054-4bc4-9daf-d36ae205ed3e"), jstring.$new("GET /aggroup/homepage/display __reqTraceID=5ca01019-fafc-4f92-a80e-82ce1389aab7&abStrategy=d&allowToTrack=1&ci=1&cityId=1&clearTimeStamp=-1&client).getBytes(), jinteger.valueOf(2)]
        var result = NBridge.main(203, objArr);
        console.log("result:"+result);
    })
}


// frida -U -f com.sankuai.meituan -l C:\Users\pr0214\Desktop\DTA\Unidbg学习指南\Unidbg进阶篇\Unidbg的五个大实例\mtguard\new\mt10\lession10\testOnJniOnLoad.js
```

```
[Nexus 5X::com.sankuai.meituan]-> Enter Mtguard JNI OnLoad
After Mtguard JNI OnLoad
call com/meituan/android/common/mtguard/NBridge, main(I[Ljava/lang/Object;)[Ljava/lang/Object;
result:null
call com/meituan/android/common/mtguard/NBridge, main(I[Ljava/lang/Object;)[Ljava/lang/Object;
result:{"a0":"1.5","a1":"9b69f861-e054-4bc4-9daf-d36ae205ed3e","a2":"460e20150fd732693f7119f91da765f959e16f0f","a3":2,"a4":1617023178,"a5":"fSdm2rcaszWJ/GEN2Q2AjRHumgC7NCXqDt60WlBeuz9u9L+0DyV1Xew5wjn3DEvtF4rSTrOjHAh6ARJ8+DLhP41U3Ayxks9ZTaNMpzMzrkop/PDYkaOJOxpwc9oA9ebwEKzwtnjP0R0S3ZXqtKLrclykX7MLWs9bpWAqTxpPiTAnf1pUYKfH0rn2xzSXryofL1y9ostXEKzV+dveF5zZVpdsiJHiAY1yNp/MJLamFZTirM2/cj9uBC2JisxcGISPjqf+VYbdKgoQLOs6+GJtyA/h6S3xLkU0uoIKHplutHcInHpXViC/tYlzpnO+p+02DUb8qCtnh/sz7/sDHETYT6h4JF5da2I3EC7ypdNj3VP4UK0E9ekKBA==","a6":0,"d1":"0a33910a22d3002db06f8ad2b4977c8379dd3f0a"}
call com/meituan/android/common/mtguard/NBridge, main(I[Ljava/lang/Object;)[Ljava/lang/Object;
com/meituan/android/common/mtguard/NBridge$SIUACollector, startCollection()Ljava/lang/String;
call com/meituan/android/common/mtguard/NBridge, main(I[Ljava/lang/Object;)[Ljava/lang/Object;
result:{"a0":"1.5","a1":"9b69f861-e054-4bc4-9daf-d36ae205ed3e","a2":"460e20150fd732693f7119f91da765f959e16f0f","a3":2,"a4":1617023179,"a5":"FHyKQ0QJaH7NvwkVuDYTKIKJ+XlahtybcECQh4CTRgOlENk40NTsFHH/T429PgP2h6OgSL2lzNONPFbbALchHi3E7eGhWwgwt8PjECE/S5BkwbBk+/axHZdabfdOJhThxTLLeyJ8198buPO2HZ1ojU8RM/Wdh+TThMyahbgbk6jXH+h+tDFqEaecBQmjCN3wax1DcuEToPF2peHnmcyMQQT82m5OSVgzrMzKGol6OlJfnEVXufs789yqrRu5ZaXl+caF0Z8NYPOdgNgDitQ+uuRsFQnM6qGdQW8SdaenPn+6K4Pw9bApk5+obL4=","a6":0,"d1":"086c225497d75442f882e23634d6350e101c5f8d"}
```

可以发现，只有一次 result:null，后面就可以顺利 call 出结果。

这说明两点

1.SO 加载后首先执行 main 函数

2. 第一个 main 函数是初始化函数。

更详细的打印 main 函数，显示其参数和返回值。我们发现第一个 main 函数参数 1 是 111，我们称之为 main111

![](https://img-blog.csdnimg.cn/20210617192916330.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
参数 2 就是一个对象数组，长度为 1，没指定，也就是对象数组里一个空对象。

```
package com.lession7;

import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.api.PackageInfo;
import com.github.unidbg.linux.android.dvm.api.Signature;
import com.github.unidbg.linux.android.dvm.array.ArrayObject;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.linux.android.dvm.wrapper.DvmInteger;
import com.github.unidbg.memory.Memory;

import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

public class NBridge extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;


    NBridge(){
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.meituan").build();
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析

        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\lession7\\mt.apk")); // 创建Android虚拟机
        vm.setVerbose(true); // 设置是否打印Jni调用细节
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\lession7\\libmtguard.so"), true);

        module = dm.getModule(); //

        vm.setJni(this);
        dm.callJNI_OnLoad(emulator);
    }

    public static void main(String[] args) {
        NBridge test = new NBridge();
        test.main111();
        test.main203();
    }

    public String main203(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
        list.add(203);
        StringObject input2_1 = new StringObject(vm, "9b69f861-e054-4bc4-9daf-d36ae205ed3e");
        ByteArray input2_2 = new ByteArray(vm, "GET /aggroup/homepage/display __r0ysue".getBytes(StandardCharsets.UTF_8));
        DvmInteger input2_3 = DvmInteger.valueOf(vm, 2);
        vm.addLocalObject(input2_1);
        vm.addLocalObject(input2_2);
        vm.addLocalObject(input2_3);
        // 完整的参数2
        list.add(vm.addLocalObject(new ArrayObject(input2_1, input2_2, input2_3)));
        Number number = module.callFunction(emulator, 0x5a38d, list.toArray())[0];
        return vm.getObject(number.intValue()).getValue().toString();
    };

    public void main111(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
        list.add(111);
        DvmObject<?> obj = vm.resolveClass("java/lang/object").newObject(null);
        vm.addLocalObject(obj);
        ArrayObject myobject = new ArrayObject(obj);
        vm.addLocalObject(myobject);
        list.add(vm.addLocalObject(myobject));
        module.callFunction(emulator, 0x5a38d, list.toArray());
    };
}
```

![](https://img-blog.csdnimg.cn/20210617193002347.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

产生了一些报错，JAVA 环境缺失，中规中矩补就是了。

有人会比较困扰怎么得到正确的值，其实很简单，使用 JNItrace trace 三分钟，保存在本地，直接搜索，需要用的基本都在里面。不熟悉具体流程的可以来报课鸭，直播互动教学。

![](https://img-blog.csdnimg.cn/20210617193015122.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

通过这种方式补全 JAVA 环境的缺失

```
@Override
    public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "com/meituan/android/common/mtguard/NBridge->getPicName()Ljava/lang/String;":{
                return new StringObject(vm, "ms_com.sankuai.meituan");
            }
            case "com/meituan/android/common/mtguard/NBridge->getSecName()Ljava/lang/String;":{
                return new StringObject(vm, "ppd_com.sankuai.meituan.xbt");
            }
            case "com/meituan/android/common/mtguard/NBridge->getAppContext()Landroid/content/Context;":{
                return vm.resolveClass("android/content/Context").newObject(null);
            }
            case "com/meituan/android/common/mtguard/NBridge->getMtgVN()Ljava/lang/String;": {
                return new StringObject(vm, "4.4.7.3");
            }
        }
        return super.callStaticObjectMethodV(vm, dvmClass, signature,vaList);
    }

    @Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "android/content/Context->getPackageCodePath()Ljava/lang/String;":{
                return new StringObject(vm, "/data/app/com.sankuai.meituan-TEfTAIBttUmUzuVbwRK1DQ==/base.apk");
            }
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }

    @Override
    public int getIntField(BaseVM vm, DvmObject<?> dvmObject, String signature) {
        switch (signature){
            case "android/content/pm/PackageInfo->versionCode:I":{
                return 1100090405;
            }
        };
        return super.getIntField(vm, dvmObject, signature);
    }

    @Override
    public DvmObject<?> newObjectV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "java/lang/Integer-><init>(I)V":
                int input = vaList.getInt(0);
                return new DvmInteger(vm, input);
        }
        return super.newObjectV(vm, dvmClass, signature, vaList);
    }
```

运行效果

![](https://img-blog.csdnimg.cn/20210617193025618.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

前面的实例中，我们主要补的是 JAVA 环境，这里补的是文件访问。

文件访问的情况各种各样，比如从 app 的某个 xml 文件中读取 key，读取某个资源文件的图片做运算，读取 proc/self 目录下的文件反调试等等。

可是我们哪来的手机文件目录，哪来的系统路径呢？我们只是一个虚拟的模拟器罢了。所以 Unidbg 对文件访问的相关 API 进行了重定向。

当样本做文件访问时，Unidbg 重定向到本机的某个位置，进入 src/main/java/com/github/unidbg/file/BaseFileSystem.java

打印一下路径

![](https://img-blog.csdnimg.cn/20210617193036714.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20210617193046730.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

接下来我们按照要求，在 data 目录下新建对应文件夹，并把我们的 apk 复制进去，改名成 base.apk。

![](https://img-blog.csdnimg.cn/20210617193057668.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

这个文件访问就补好了，再运行代码时，就已经跨过这个坑，进行下一个流程了。

除此之外，也可以通过代码的方式进行操作

我们的类实现文件重定向的接口即可，只需要三个步骤，如下：

```
// 1
public class NBridge extends AbstractJni implements IOResolver {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;


    NBridge(){
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.meituan").build();
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析

        vm = emulator.createDalvikVM(new File("C:\\Users\\pr0214\\Desktop\\DTA\\unidbg\\versions\\unidbg-2021-5-17\\unidbg-master\\unidbg-android\\src\\test\\java\\com\\lession7\\mt.apk")); // 创建Android虚拟机
        // 2
        emulator.getSyscallHandler().addIOResolver(this);
        vm.setVerbose(true); // 设置是否打印Jni调用细节
        DalvikModule dm = vm.loadLibrary(new File("C:\\Users\\pr0214\\Desktop\\DTA\\unidbg\\versions\\unidbg-2021-5-17\\unidbg-master\\unidbg-android\\src\\test\\java\\com\\lession7\\libmtguard.so"), true);

        module = dm.getModule(); //

        vm.setJni(this);
        dm.callJNI_OnLoad(emulator);
    }

    // 3
    @Override
    public FileResult resolve(Emulator emulator, String pathname, int oflags) {
        if (("/data/app/com.sankuai.meituan-TEfTAIBttUmUzuVbwRK1DQ==/base.apk").equals(pathname)) {
            // 填入想要重定位的文件
            return FileResult.success(new SimpleFileIO(oflags, new File("unidbg-android\\src\\test\\java\\com\\lession10\\mt.apk"), pathname));
        }
        return null;
    }
```

两种方法各有其最佳的使用场景，如果两种都设置，代码方式设置的优先级更高。

继续往下补 JAVA 环境

![](https://img-blog.csdnimg.cn/20210617193109324.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20210617193121381.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

好像结果出来了，再修改一下输出函数，打印一下。

```
public void main203(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
        list.add(203);
        StringObject input2_1 = new StringObject(vm, "9b69f861-e054-4bc4-9daf-d36ae205ed3e");
        ByteArray input2_2 = new ByteArray(vm, "GET /aggroup/homepage/display __r0ysue".getBytes(StandardCharsets.UTF_8));
        DvmInteger input2_3 = DvmInteger.valueOf(vm, 2);
        vm.addLocalObject(input2_1);
        vm.addLocalObject(input2_2);
        vm.addLocalObject(input2_3);
        // 完整的参数2
        list.add(vm.addLocalObject(new ArrayObject(input2_1, input2_2, input2_3)));
        Number number = module.callFunction(emulator, 0x5a38d, list.toArray())[0];
        StringObject result = (StringObject) ((DvmObject[])((ArrayObject)vm.getObject(number.intValue())).getValue())[0];
        System.out.println(result.getValue());
    };
```

![](https://img-blog.csdnimg.cn/20210617193136224.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

大功告成。

### 四、尾声

这是上下文缺失和补充的一个例子  
资源链接：https://pan.baidu.com/s/1e-u8hbHqCFnS9P0vVM8GbQ  
提取码：igcw
