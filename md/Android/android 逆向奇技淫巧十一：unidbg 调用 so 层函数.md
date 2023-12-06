> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/theseventhson/p/14880119.html)

　　做脱机协议，首先要找到关键的加密代码，然而这些代码一般都在 so 里面，因为逆向 c/c++ 的难度远比 java 大多了！找到关键代码后，一般情况下是逐行分析，然后自己写代码复现整个加密过程。但是，有些非标准的加密算法是由一个团队实现的，整个过程非常复杂。逆向人员再去逐行分析和复现，有点 “不划算”！怎么才能直接调用 so 里面的这些关键代码了？可以通过前面的介绍的 frida hook，也可以通过今天介绍的这个 so 的模拟框架 --unidbg！官方的功能介绍如下：

*   Emulation of the JNI Invocation API so JNI_OnLoad can be called.
*   Support JavaVM, JNIEnv.
*   Emulation of syscalls instruction.
*   Support ARM32 and ARM64.
*   Inline hook, thanks to [Dobby](https://github.com/jmpews/Dobby).
*   Android import hook, thanks to [xHook](https://github.com/iqiyi/xHook).
*   iOS [fishhook](https://github.com/facebook/fishhook) and substrate and [whale](https://github.com/asLody/whale) hook.
*   [unicorn](https://github.com/zhkl0228/unicorn) backend support simple console debugger, gdb stub, instruction trace, memory read/write trace.
*   Support iOS objc and swift runtime.
*   Support [dynarmic](https://github.com/MerryMage/dynarmic) fast backend.
*   Support Apple M1 hypervisor, the fastest ARM64 backend.
*   Support Linux KVM backend with Raspberry Pi B4.

　　看着很多，有点唬人，实际并不复杂，以本文分享的为例：我们平时开发 android app，在 Android studio 配置好各种环境和参数后是能直接在 java 层调用 so 层函数的。那么**在 unidbg，也能实现同样的功能：即调用 so 层的函数**！这也是 unidbg 最核心的功能之一了！具体该怎么操作了？ 步骤一：当然是先去 unidbg 的官网下载 unidbg 的框架啦，然后用 intelij 打开，里面能看到作者已经写好的各种 java 的测试工程代码，如下：

　　![](https://img2020.cnblogs.com/blog/2052730/202106/2052730-20210613154916859-661230114.png)

 　　这些测试的 demo 代码已经说明了 unidbg 的接口和核心功能了。今天就用 kanxue 提供的 so 来说明核心 api 和调用流程！

　　1、选择执行引擎：如果明确使用了以下代码，那么 unidbg 使用 dynarmic 引擎，否则默认使用 unicorn 引擎！

```
static {
        DynarmicLoader.useDynarmic();
    }
```

　　2、创建虚拟机 / 模拟器，并执行虚拟机的类型是 art 还是 dailvik：

```
AndroidARMEmulator emulator= new AndroidARMEmulator("com.sun.jna",null,null);
        final Memory memory = emulator.getMemory();

        VM vm = emulator.createDalvikVM();
        vm.setVerbose(true);//这里如果是true，后续调用jni_onload的时候就能打印日志
```

　　3、指定 SDK 的版本，这里用 23 版本：

```
LibraryResolver resolver = new AndroidResolver(23);
        memory.setLibraryResolver(resolver);
```

　　4、开始加载 so 库了：

```
Module unicorn08module=emulator.loadLibrary(new File("D:\\xxxx\\unidbg-master\\unidbg-android\\src\\test\\java\\com\\unicorncourse08\\unicorn08.so"));
```

　　5、调用 so 层的导出函数：这两个都是导出函数，直接用符号名就行了；

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
Number result=unicorn08module.callFunction(emulator,"_Z3addii",1,2)[0];//导出函数直接用符号名就行了
        System.out.println("_Z3addii result:"+result.intValue());

        //_Z7add_sixiiiiii
        result=unicorn08module.callFunction(emulator,"_Z7add_sixiiiiii",1,2,3,4,5,6)[0];
        System.out.println("_Z7add_sixiiiiii result："+result.intValue());
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

　　这个是打印的结果：

```
_Z3addii result:3
_Z7add_sixiiiiii result：21
```

　　6、这两个都是对字符串做操作的，so 层仅仅求了传入字符串的长度：

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
MemoryBlock block1=memory.malloc(10,true);
        UnidbgPointer str1_ptr=block1.getPointer();
        str1_ptr.write("hello".getBytes());
        String content=ARM.readCString(emulator.getBackend(),str1_ptr.peer);
        System.out.println("_Z15getstringlengthPKc:"+str1_ptr.toString()+"---"+content);
        result=unicorn08module.callFunction(emulator,"_Z15getstringlengthPKc",new PointerNumber(str1_ptr))[0];
        Number r0value=emulator.getBackend().reg_read(ArmConst.UC_ARM_REG_R0);
        System.out.println("_Z15getstringlengthPKc result:"+result.intValue()+"----"+r0value);

        MemoryBlock block2=memory.malloc(10,true);
        UnidbgPointer str2_ptr=block2.getPointer();
        str2_ptr.write("666".getBytes());
        String content2=ARM.readCString(emulator.getBackend(),str2_ptr.peer);
        System.out.println("_Z16getstringlength2PKcS0_:"+str2_ptr.toString()+"---"+content2);
        result=unicorn08module.callFunction(emulator,"_Z16getstringlength2PKcS0_",new PointerNumber(str1_ptr),new PointerNumber(str2_ptr))[0];
        r0value=emulator.getBackend().reg_read(ArmConst.UC_ARM_REG_R0);
        System.out.println("_Z16getstringlength2PKcS0_ result:"+result.intValue()+"----"+r0value);
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

　　打印结果：

```
_Z15getstringlengthPKc:RW@0x4016b000---hello
_Z15getstringlengthPKc result:5----5
_Z16getstringlength2PKcS0_:RW@0x4016c000---666
_Z16getstringlength2PKcS0_ result:8----8
```

　　7、这核心的核心：直接调用 jni_onload

```
vm.callJNI_OnLoad(emulator,unicorn08module);
```

　　打印结果：这里可以看到分别在 so 的 0x8c7、0xccb 调用了 FindClass 和 RegisterNative 方法，然后注册 MainActivity 这个类的 stringFromJNI2 方法，该方法和 so 层中 0xb35 的方法是映射的！

```
JNIEnv->FindClass(com/example/unicorncourse08/MainActivity) was called from RX@0x40000c87[libnative-lib.so]0xc87
JNIEnv->RegisterNatives(com/example/unicorncourse08/MainActivity, unidbg@0xbffff778, 1) was called from RX@0x40000ccb[libnative-lib.so]0xccb
RegisterNative(com/example/unicorncourse08/MainActivity, stringFromJNI2(Ljava/lang/String;)Ljava/lang/String;, RX@0x40000b35[libnative-lib.so]0xb35)
```

　　去 so 的 0xc87 和 0xccb 查看，果然是 FindClass 和 RegisterNative 方法，unidbg 诚不我欺！ 作为逆向，其实最重要的还是最后那个打印结果：java 层的 stringFromJNI2 方法就是和 so 层的这个方法是映射的：

 ![](https://img2020.cnblogs.com/blog/2052730/202106/2052730-20210613163941460-1059325062.png)

 　　进入里面调用的各个函数仔细分析，发现这个函数还是比较简单：先接受传入的 string，再打印出来！由于这个函数并未导出，但是和 java 层的函数做了映射，所以这里也可以直接通过 java 层的名字来直接调用，代码如下：

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
//调用jni函数，对于动态注册的jni函数必须在完成地址的绑定才能调用
        System.out.println("stringFromJNI1-------------------------");
        DvmClass MainActivity_dvmclass=vm.resolveClass("com/example/unicorncourse08/MainActivity");//先把类找到，这里的原理很像反射
        DvmObject resultobj=MainActivity_dvmclass.callStaticJniMethodObject(emulator,"stringFromJNI1(Ljava/lang/String;)Ljava/lang/String;","helloworld");//再通过类去调用里面的函数
        System.out.println("resultobj:"+resultobj);
        resultobj=MainActivity_dvmclass.callStaticJniMethodObject(emulator,"stringFromJNI1(Ljava/lang/String;)Ljava/lang/String;",new StringObject(vm, "hellokanxue"));
        System.out.println("resultobj:"+resultobj);
        System.out.println("stringFromJNI1-------------------------");

        //动态注册的jni函数stringFromJNI2
        resultobj=MainActivity_dvmclass.callStaticJniMethodObject(emulator,"stringFromJNI2(Ljava/lang/String;)Ljava/lang/String;",new StringObject(vm, "hellostringFromJNI2"));
        System.out.println("resultobj:"+resultobj);
        System.out.println("stringFromJNI2-------------------------");


        DvmObject mainactivity=MainActivity_dvmclass.newObject(null);
        mainactivity.callJniMethodObject(emulator,"stringFromJNI2(Ljava/lang/String;)Ljava/lang/String;",new StringObject(vm, "hellostringFromJNI2"));
        System.out.println("resultobj:"+resultobj);
        System.out.println("stringFromJNI2-------------------------");
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

　　打印的结果如下：这里面除了我们显式调用 println 打印的日志，还有 unidbg 这个框架自身打印的日志（主要是 JNIenv 这个类的函数调用）：

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
stringFromJNI1-------------------------
Find native function Java_com_example_unicorncourse08_MainActivity_stringFromJNI1(Ljava/lang/String;)Ljava/lang/String; => RX@0x40000a71[libnative-lib.so]0xa71
JNIEnv->GetStringUtfChars("helloworld") was called from RX@0x40000b03[libnative-lib.so]0xb03
JNIEnv->NewStringUTF("helloworld") was called from RX@0x40000b2f[libnative-lib.so]0xb2f
resultobj:"helloworld"
Find native function Java_com_example_unicorncourse08_MainActivity_stringFromJNI1(Ljava/lang/String;)Ljava/lang/String; => RX@0x40000a71[libnative-lib.so]0xa71
JNIEnv->GetStringUtfChars("hellokanxue") was called from RX@0x40000b03[libnative-lib.so]0xb03
[main]I/stringFromJNI1: content:helloworld,length:10
[main]I/stringFromJNI1: content:hellokanxue,length:11
[main]I/stringFromJNI2: content:hellostringFromJNI2,length:19
[main]I/stringFromJNI2: content:hellostringFromJNI2,length:19
JNIEnv->NewStringUTF("hellokanxue") was called from RX@0x40000b2f[libnative-lib.so]0xb2f
resultobj:"hellokanxue"
stringFromJNI1-------------------------
Find native function Java_com_example_unicorncourse08_MainActivity_stringFromJNI2(Ljava/lang/String;)Ljava/lang/String; => RX@0x40000b35[libnative-lib.so]0xb35
JNIEnv->GetStringUtfChars("hellostringFromJNI2") was called from RX@0x40000b03[libnative-lib.so]0xb03
JNIEnv->NewStringUTF("hellostringFromJNI2") was called from RX@0x40000b2f[libnative-lib.so]0xb2f
resultobj:"hellostringFromJNI2"
stringFromJNI2-------------------------
Find native function Java_com_example_unicorncourse08_MainActivity_stringFromJNI2(Ljava/lang/String;)Ljava/lang/String; => RX@0x40000b35[libnative-lib.so]0xb35
JNIEnv->GetStringUtfChars("hellostringFromJNI2") was called from RX@0x40000b03[libnative-lib.so]0xb03
JNIEnv->NewStringUTF("hellostringFromJNI2") was called from RX@0x40000b2f[libnative-lib.so]0xb2f
resultobj:"hellostringFromJNI2"
stringFromJNI2-------------------------
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

 　　完整的代码：

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
package com.unicorncourse08;

import com.github.unidbg.LibraryResolver;
import com.github.unidbg.Module;
import com.github.unidbg.PointerNumber;
import com.github.unidbg.arm.ARM;
import com.github.unidbg.arm.backend.dynarmic.DynarmicLoader;
import com.github.unidbg.linux.android.AndroidARMEmulator;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.DvmClass;
import com.github.unidbg.linux.android.dvm.DvmObject;
import com.github.unidbg.linux.android.dvm.StringObject;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.memory.MemoryBlock;
import com.github.unidbg.pointer.UnidbgPointer;
import unicorn.ArmConst;
import java.io.File;

public class MainActivity {
    static {
        DynarmicLoader.useDynarmic();
    }
    public static void main(String[] args) {
        AndroidARMEmulator emulator= new AndroidARMEmulator("com.sun.jna",null,null);
        final Memory memory = emulator.getMemory();

        VM vm = emulator.createDalvikVM();
        vm.setVerbose(true);

        LibraryResolver resolver = new AndroidResolver(23);
        memory.setLibraryResolver(resolver);
        Module unicorn08module=emulator.loadLibrary(new File("D:\\xxxx\\unidbg-master\\unidbg-android\\src\\test\\java\\com\\unicorncourse08\\unicorn08.so"));
        Number result=unicorn08module.callFunction(emulator,"_Z3addii",1,2)[0];//导出函数直接用符号名就行了
        System.out.println("_Z3addii result:"+result.intValue());

        //_Z7add_sixiiiiii
        result=unicorn08module.callFunction(emulator,"_Z7add_sixiiiiii",1,2,3,4,5,6)[0];
        System.out.println("_Z7add_sixiiiiii result："+result.intValue());

        //_Z15getstringlengthPKc
        MemoryBlock block1=memory.malloc(10,true);
        UnidbgPointer str1_ptr=block1.getPointer();
        str1_ptr.write("hello".getBytes());
        String content=ARM.readCString(emulator.getBackend(),str1_ptr.peer);
        System.out.println("_Z15getstringlengthPKc:"+str1_ptr.toString()+"---"+content);
        result=unicorn08module.callFunction(emulator,"_Z15getstringlengthPKc",new PointerNumber(str1_ptr))[0];
        Number r0value=emulator.getBackend().reg_read(ArmConst.UC_ARM_REG_R0);
        System.out.println("_Z15getstringlengthPKc result:"+result.intValue()+"----"+r0value);

        MemoryBlock block2=memory.malloc(10,true);
        UnidbgPointer str2_ptr=block2.getPointer();
        str2_ptr.write("666".getBytes());
        String content2=ARM.readCString(emulator.getBackend(),str2_ptr.peer);
        System.out.println("_Z16getstringlength2PKcS0_:"+str2_ptr.toString()+"---"+content2);
        result=unicorn08module.callFunction(emulator,"_Z16getstringlength2PKcS0_",new PointerNumber(str1_ptr),new PointerNumber(str2_ptr))[0];
        r0value=emulator.getBackend().reg_read(ArmConst.UC_ARM_REG_R0);
        System.out.println("_Z16getstringlength2PKcS0_ result:"+result.intValue()+"----"+r0value);


        //调用jni_OnLoad函数
        vm.callJNI_OnLoad(emulator,unicorn08module);

        //调用jni函数，对于动态注册的jni函数必须在完成地址的绑定才能调用
        System.out.println("stringFromJNI1-------------------------");
        DvmClass MainActivity_dvmclass=vm.resolveClass("com/example/unicorncourse08/MainActivity");//先把类找到，这里的原理很像反射
        DvmObject resultobj=MainActivity_dvmclass.callStaticJniMethodObject(emulator,"stringFromJNI1(Ljava/lang/String;)Ljava/lang/String;","helloworld");//再通过类去调用里面的函数
        System.out.println("resultobj:"+resultobj);
        resultobj=MainActivity_dvmclass.callStaticJniMethodObject(emulator,"stringFromJNI1(Ljava/lang/String;)Ljava/lang/String;",new StringObject(vm, "hellokanxue"));
        System.out.println("resultobj:"+resultobj);
        System.out.println("stringFromJNI1-------------------------");

        //动态注册的jni函数stringFromJNI2
        resultobj=MainActivity_dvmclass.callStaticJniMethodObject(emulator,"stringFromJNI2(Ljava/lang/String;)Ljava/lang/String;",new StringObject(vm, "hellostringFromJNI2"));
        System.out.println("resultobj:"+resultobj);
        System.out.println("stringFromJNI2-------------------------");


        DvmObject mainactivity=MainActivity_dvmclass.newObject(null);
        mainactivity.callJniMethodObject(emulator,"stringFromJNI2(Ljava/lang/String;)Ljava/lang/String;",new StringObject(vm, "hellostringFromJNI2"));
        System.out.println("resultobj:"+resultobj);
        System.out.println("stringFromJNI2-------------------------");

    }
}
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

 　　总结一下，上述 API 包括了 3 种 so 函数的调用方法：普通的 so 方法、jni_onload 调用、jni 函数调用！上面举例的这些内容相对简单，并不涉及到 so 层调用 java 层的函数。如果遇到 so 层函数调用 java 层函数怎么办么？我们如果自己在 apk 写 java 代码调用 so 层函数，**遇到 so 通过反射调用 java 层函数时，需要自己补上 java 层对应的类、方法和变量，因为这些需要执行的代码是绕不过去的！**unidbg 是这么样的么？ 答案是肯定的！比如下面的这个 so 层的方法，会在 jni_onload 中被调用；这里需要获取 java 层普通变量、static 变量后打印出来；也会获取 java 层的普通方法然后调用，这该怎么办了？

 　　![](https://img2020.cnblogs.com/blog/2052730/202106/2052730-20210613211815383-1064330698.png)

 　　上面说了：so 层调用 java 层的代码肯定是要补齐的（如果直接简单粗暴改 so 层代码，可能导致别人原来的逻辑错误）！ 这里该怎么实操了？ 大概的思路是：

*   　自己补上缺失的方法（当然 java 层的方法可以用 jadx、jeb 等反编译得到，不用自己反编译 smali），这里缺失了 base64 方法，需要补上！
*      自己补上缺失的变量，方法同上！
*      重写 getStaticObjectField、getObjectField、callObjectMethodV 等方法，然后检测传入的参数。一旦发现使用 / 调用的是 java 层变量、方法，用自己补上的哪些代码替换（原理像不像平时常用的 hook？）

 　　说了那么多，完整的 demo 代码如下：

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
package com.unicorncourse08;

import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidARMEmulator;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.jni.ProxyClassFactory;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.virtualmodule.android.AndroidModule;
import com.github.unidbg.virtualmodule.android.JniGraphics;
import org.apache.commons.codec.binary.Base64;
import sun.applet.Main;

import java.io.File;
import java.lang.reflect.Field;

public class MainActivitymethod1 extends AbstractJni {

    private static DvmClass MainActivityClass;

    @Override
    /*
    * staticcontent是java层的静态变量；getStaticObjectField，一旦检测到so层引用这个变量，那么自己返回这个变量的值
    * */
    public DvmObject<?> getStaticObjectField(BaseVM vm, DvmClass dvmClass, String signature) {
        System.out.println("getStaticObjectField->"+signature);
        if(signature.equals("com/example/testjni/MainActivity->staticcontent:Ljava/lang/String;")){
            return new StringObject(vm,"staticcontent");//源码 public static string staticcontent = "staticcontent"
        }
        return super.getStaticObjectField(vm, dvmClass, signature);
    }

    @Override
    /*
    * objcontent是java层的变量；这里重写getObjectField方法，一旦检测到so层引用这个变量，那么自己返回这个变量的值
    * */
    public DvmObject<?> getObjectField(BaseVM vm, DvmObject<?> dvmObject, String signature) {
        System.out.println("getObjectField->"+signature);
        if(signature.equals("com/example/testjni/MainActivity->objcontent:Ljava/lang/String;")){
            return new StringObject(vm,"objcontent");//public string objcontent
        }
        return super.getObjectField(vm, dvmObject, signature);
    }
    /*
    * java层的方法，这里需要复现，否则不知道怎么执行
    * */
    public String base64(String arg3) {
        String result=Base64.encodeBase64String(arg3.getBytes());
        return result;
    }

    @Override
    /*
    * base64是java层的方法，这里重写callObjectMethodV方法：一旦发现调用的是java层的base64方法，这里就用自己复现的base64方法替换
    * */
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        System.out.println("callObjectMethodV->"+signature);
        if(signature.equals("com/example/testjni/MainActivity->base64(Ljava/lang/String;)Ljava/lang/String;")){
            DvmObject dvmobj=vaList.getObjectArg(0);
            String arg= (String) dvmobj.getValue();
            String result=base64(arg);
            return new StringObject(vm,result);
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }

    public static void main(String[] args) {
        MainActivitymethod1 mainActivitymethod1=new MainActivitymethod1();
        AndroidARMEmulator emulator = new AndroidARMEmulator("org.telegram.messenger",null,null);
        final Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));
        VM vm = emulator.createDalvikVM();
        vm.setVerbose(true);
        vm.setJni(mainActivitymethod1);
        DalvikModule dm = vm.loadLibrary(new File("D:\\xxxx\\unidbg-master\\unidbg-android\\src\\test\\java\\com\\unicorncourse08\\calljava.so"), true);
        dm.callJNI_OnLoad(emulator);
        MainActivityClass=vm.resolveClass("com/example/testjni/MainActivity");
        DvmObject obj=MainActivityClass.newObject(null);
        obj.callJniMethodObject(emulator,"base64byjni(Ljava/lang/String;)Ljava/lang/String;","callbase64byjni");
　　}
}
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

　　整个逻辑其实并不复杂，从 main 函数开始：创建模拟器、创建虚拟机、加载 so、调用 so 层函数！打印的结果如下：

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
JNIEnv->FindClass(com/example/testjni/MainActivity) was called from RX@0x400009df[libnative-lib.so]0x9df
JNIEnv->RegisterNatives(com/example/testjni/MainActivity, unidbg@0xbffff768, 1) was called from RX@0x40000f19[libnative-lib.so]0xf19
RegisterNative(com/example/testjni/MainActivity, stringFromJNI(Ljava/lang/String;)Ljava/lang/String;, RX@0x40000cb1[libnative-lib.so]0xcb1)

Find native function Java_com_example_testjni_MainActivity_base64byjni(Ljava/lang/String;)Ljava/lang/String; => RX@0x4000088d[libnative-lib.so]0x88d
JNIEnv->FindClass(com/example/testjni/MainActivity) was called from RX@0x400009df[libnative-lib.so]0x9df
getStaticObjectField->com/example/testjni/MainActivity->staticcontent:Ljava/lang/String;

JNIEnv->GetStaticObjectField(class com/example/testjni/MainActivity, staticcontent Ljava/lang/String; => "staticcontent") was called from RX@0x40000aa5[libnative-lib.so]0xaa5
JNIEnv->GetStringUtfChars("staticcontent") was called from RX@0x40000adb[libnative-lib.so]0xadb

[main]I/stringFromJNI: staticcontent:staticcontent
[main]I/stringFromJNI: objcontent:objcontent

getObjectField->com/example/testjni/MainActivity->objcontent:Ljava/lang/String;

JNIEnv->GetObjectField(com.example.testjni.MainActivity@7b3300e5, objcontent Ljava/lang/String; => "objcontent") was called from RX@0x40000b11[libnative-lib.so]0xb11
JNIEnv->GetStringUtfChars("objcontent") was called from RX@0x40000adb[libnative-lib.so]0xadb
JNIEnv->GetMethodID(com/example/testjni/MainActivity.base64(Ljava/lang/String;)Ljava/lang/String;) was called from RX@0x40000b55[libnative-lib.so]0xb55

callObjectMethodV->com/example/testjni/MainActivity->base64(Ljava/lang/String;)Ljava/lang/String;

JNIEnv->CallObjectMethodV(com.example.testjni.MainActivity@7b3300e5, base64("callbase64byjni") => "Y2FsbGJhc2U2NGJ5am5p") was called from RX@0x40000bb1[libnative-lib.so]0xbb1
JNIEnv->GetStringUtfChars("Y2FsbGJhc2U2NGJ5am5p") was called from RX@0x40000adb[libnative-lib.so]0xadb
JNIEnv->NewStringUTF("Y2FsbGJhc2U2NGJ5am5p") was called from RX@0x40000c05[libnative-lib.so]0xc05


[main]I/stringFromJNI: base64result:Y2FsbGJhc2U2NGJ5am5p
```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

参考：

1、https://github.com/zhkl0228/unidbg  unidbg 官网