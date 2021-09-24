> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38851536/article/details/117533396?spm=1001.2014.3001.5501)

### 文章目录

*   *   *   [一、前言](#_3)
        *   [二、准备](#_12)
        *   [三、Unidbg 模拟执行](#Unidbg_37)
        *   [四、算法还原](#_627)
        *   [五、尾声](#_1061)

### 一、前言

这是 SO 逆向入门实战教程的第三篇，上篇的重心是 Unidbg 的 Hook 使用，本篇的重点是如何在 Unidbg 中补齐 JAVA 环境以及哈希算法的魔改。

*   侧重**新工具、新思路、新方法**的使用，算法分析的常见路子是 Frida Hook + IDA ，在本系列中，会淡化 Frida 的作用，采用 Unidbg Hook + IDA 的路线。
*   主打入门，但**并不限于入门**，你会在样本里看到有浅有深的魔改加密算法、以及 OLLVM、SO 对抗等内容。
*   对样本的分析仅限于学习和研究，坚决抵制黑灰产。
*   一共十三篇，1-2 天更新一篇。每篇的资料放在文末的百度网盘中。

### 二、准备

![](https://img-blog.csdnimg.cn/20210603195700688.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
sign 方法就是我们的目标方法，参数 1 是字符串，参数 2 是字符串的字节数组。我们设参数 1 是为 “12345”，参数 2 为 “r0ysue”，在 Frida 中主动调用测试返回结果：

```
function callSign(){
    Java.perform(function () {
        var NetCrypto = Java.use("com.izuiyou.network.NetCrypto");
        var JavaString = Java.use("java.lang.String");

        var plainText = "r0ysue";
        var plainTextBytes = JavaString.$new(plainText).getBytes("UTF-8");

        var result = NetCrypto.a("12345", plainTextBytes);
        console.log(result);
    });
}
```

![](https://img-blog.csdnimg.cn/20210603195750738.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
多次变换入参可以验证，输出有如下特征

*   输出：参数 1+"?"+“sign=v2-”+32 位字符串
*   输入不变则输出不变

### 三、Unidbg 模拟执行

需要注意的是，在执行 sign 函数前需要先执行 native_init 函数。  
![](https://img-blog.csdnimg.cn/20210603195845464.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

老规矩，先搭一下架子

```
package com.right;

import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

public class zuiyou extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    zuiyou() {
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.xiaochuankeji.tieba").build(); // 创建模拟器实例
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\zuiyou\\right573.apk")); // 创建Android虚拟机
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\zuiyou\\libnet_crypto.so"), true); // 加载so到虚拟内存
        module = dm.getModule(); //获取本SO模块的句柄

        vm.setJni(this);
        vm.setVerbose(true);
        dm.callJNI_OnLoad(emulator);
    };

    public static void main(String[] args) throws Exception {
        zuiyou test = new zuiyou();
    }
}
```

![](https://img-blog.csdnimg.cn/20210603195859810.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
可以看到，在 JNIOnLoad 中做了函数的动态注册。

此处有个值得一提的问题，如果在加载 so 到虚拟内存的步骤中，参数二设为 false(即不执行 init 相关函数)，会出现有趣的一幕。  
![](https://img-blog.csdnimg.cn/20210603195915217.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

我们发现，输出竟然”乱码 “了，如果参数 2 为 false 即” 乱码 “，true 就” 不乱码“，这是为什么呢？甚至有人在论坛发帖求助类似问题：

[[求助]Unidbg 的 Jnionload 加载出的类是乱码 -¥ 付费问答 - 看雪论坛 - 安全社区 | 安全招聘 | bbs.pediy.com](https://bbs.pediy.com/thread-266837.htm) 。

其实其中的道理并不复杂，甚至可以说很简单——SO 样本做了字符串的混淆或加密，以此来对抗分析人员，但字符串总是要解密的，不然怎么用呢？这个解密一般发生在 Init array 节或者 JNI OnLoad 中，又或者是该字符串使用前的任何一个时机，而本例呢，就发生在 Init array 节中，Shift+F7 快捷键查看节区验证这一点

![](https://img-blog.csdnimg.cn/20210603200008303.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)![](https://img-blog.csdnimg.cn/20210603200030848.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

我们可以看到，Init array 节内有大量函数，解密就发生在其中。当我们使用 Unidbg 模拟执行时，如果加载 SO 时配置为不执行 Init 相关函数，这导致整个 SO 中的字符串都没有被解密，自然输出就是一团” 乱码 “。

由此还可以衍生出一个小话题——如果样本中的字符串被加密了，如何还原？使得分析者可以愉快的用 IDA 静态分析？

*   从内存中 Dump 出解密后的 SO 或者字符串（可以用 Frida/IDA 脚本 / adb 等），将结果回填或者说修复本身 SO。
*   使用 Unicorn 或基于 Unicorn 的模拟执行工具（Unidbg、ExAndroidNativeemu 等）运行 SO，dump 解密后的虚拟内存，回填修复 SO。

言归正传，接下来执行我们的目标函数，如图这两个函数。

![](https://img-blog.csdnimg.cn/20210603200044499.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

首先是 native_init 函数，有过前两篇的基础，就不在此处多费口舌了，看一下更新后的代码

```
package com.right;

import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

public class zuiyou extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    zuiyou() {
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.xiaochuankeji.tieba").build(); // 创建模拟器实例
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\zuiyou\\right573.apk")); // 创建Android虚拟机
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\zuiyou\\libnet_crypto.so"), true); // 加载so到虚拟内存
        module = dm.getModule(); //获取本SO模块的句柄

        vm.setJni(this);
        vm.setVerbose(true);
        dm.callJNI_OnLoad(emulator);
    };

    public void native_init(){
        // 0x4a069
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclass，直接填0，一般用不到。
        module.callFunction(emulator, 0x4a069, list.toArray());
    };

    public static void main(String[] args) throws Exception {
        zuiyou test = new zuiyou();
        test.native_init();
    }
}
```

运行，肉眼可见的报错  
![](https://img-blog.csdnimg.cn/20210603200111589.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

让我们用 Unidbg 的口吻来翻译一下这个报错：  
我在通过 callStaticObjectMethodV 方法调用 JAVA 函数时，遇到一个签名叫做 **com/izuiyou/common/base/BaseApplication->getAppContext()Landroid/content/Context;** 的函数，我不知道怎么处理，你可以立刻到 AbstractJni.java:401 行上面进行查看和处理。

![](https://img-blog.csdnimg.cn/20210603200145644.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)可以看到，一些常见的、系统的 Java 类和方法，Unidbg 作开发者已经做了处理，但不常使用的类库以及自定义 Java 类显然不在此列，所以需要我们像它内置的这些方法一样，把报错的方法补进去。

接下来开始补环境，考虑两个问题

*   怎么补
*   补什么

关于第一点，我们既可以根据报错提示，在 AbstractJni 对应的函数体内，依葫芦画瓢，case "xxx“。

![](https://img-blog.csdnimg.cn/20210603200206135.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
也可以在我们的 zuiyou 类中补，因为 zuiyou 类继承了 AbstractJNI。

关于补法，有两种实践方法都很有道理

*   全部在用户类中补，防止项目迁移或者 Unidbg 更新带来什么问题，这样做代码的移植性比较好。
*   样本的自定义 JAVA 方法在用户类中补，通用的方法在 AbstractJNI 中补，这样做的好处是，之后运行的项目如果调用通用方法，就不用做重复的修补工作。

读者可以自行选择，我这边全部写在用户类中，方便演示， 在 zuiyou 类中重写 callStaticObjectMethodV 方法

```
@Override
    public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "com/izuiyou/common/base/BaseApplication->getAppContext()Landroid/content/Context;":
                System.out.println("TODO");
        }
        return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
    }
```

第二个问题是补什么，从签名中可以看出，返回值是 **Landroid/content/Context;**，即一个 context 对象，那我们传入一个最基本的 context。

```
@Override
    public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "com/izuiyou/common/base/BaseApplication->getAppContext()Landroid/content/Context;":
                return vm.resolveClass("android/content/Context").newObject(null);
        }
        return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
    }
```

这肯定是不够用的，但没办法，只能一步一步来，就好比贵公子需要出去度假，Android 系统可以提供给他一条豪华游轮，但我们的虚拟系统没法给他那么多，我们就先提供一条木船。这条小船和尊贵的客人一起出发，客人会不断去船里索取物资，他要什么，我们再补什么！我们只关注最后贵公子取的东西是什么，这个东西一定要按照豪华游轮的标准去给他，前面的汤汤水水应付完事。

看一下完整代码和运行效果

```
package com.right;

import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

public class zuiyou extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    zuiyou() {
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.xiaochuankeji.tieba").build(); // 创建模拟器实例
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\zuiyou\\right573.apk")); // 创建Android虚拟机
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\zuiyou\\libnet_crypto.so"), true); // 加载so到虚拟内存
        module = dm.getModule(); //获取本SO模块的句柄

        vm.setJni(this);
        vm.setVerbose(true);
        dm.callJNI_OnLoad(emulator);
    };

    public void native_init(){
        // 0x4a069
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclass，直接填0，一般用不到。
        module.callFunction(emulator, 0x4a069, list.toArray());
    };

    @Override
    public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "com/izuiyou/common/base/BaseApplication->getAppContext()Landroid/content/Context;":
                return vm.resolveClass("android/content/Context").newObject(null);
        }
        return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
    }

    public static void main(String[] args) throws Exception {
        zuiyou test = new zuiyou();
        test.native_init();
    }
}
```

![](https://img-blog.csdnimg.cn/20210603200240666.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
似乎一切顺利，接下来执行 sign 方法。字节数组以及字符串类型都是前两节遇到过的，不做赘述。

```
private String callSign(){
        // 准备入参
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclass，直接填0，一般用不到。
        list.add(vm.addLocalObject(new StringObject(vm, "12345")));
        ByteArray plainText = new ByteArray(vm, "r0ysue".getBytes(StandardCharsets.UTF_8));
        list.add(vm.addLocalObject(plainText));
        Number number = module.callFunction(emulator, 0x4a28D, list.toArray())[0];
        return vm.getObject(number.intValue()).getValue().toString();
    };
```

看一下整体代码和运行效果

```
package com.right;

import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

public class zuiyou extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    zuiyou() {
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.xiaochuankeji.tieba").build(); // 创建模拟器实例
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\zuiyou\\right573.apk")); // 创建Android虚拟机
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\zuiyou\\libnet_crypto.so"), true); // 加载so到虚拟内存
        module = dm.getModule(); //获取本SO模块的句柄

        vm.setJni(this);
        vm.setVerbose(true);
        dm.callJNI_OnLoad(emulator);
    };

    public void native_init(){
        // 0x4a069
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclass，直接填0，一般用不到。
        module.callFunction(emulator, 0x4a069, list.toArray());
    };

    private String callSign(){
        // 准备入参
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclass，直接填0，一般用不到。
        list.add(vm.addLocalObject(new StringObject(vm, "12345")));
        ByteArray plainText = new ByteArray(vm, "r0ysue".getBytes(StandardCharsets.UTF_8));
        list.add(vm.addLocalObject(plainText));
        Number number = module.callFunction(emulator, 0x4a28D, list.toArray())[0];
        return vm.getObject(number.intValue()).getValue().toString();
    };

    @Override
    public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "com/izuiyou/common/base/BaseApplication->getAppContext()Landroid/content/Context;":
                return vm.resolveClass("android/content/Context").newObject(null);
        }
        return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
    }

    public static void main(String[] args) throws Exception {
        zuiyou test = new zuiyou();
        test.native_init();
        System.out.println(test.callSign());
    }
}
```

运行结果  
![](https://img-blog.csdnimg.cn/20210603200356511.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
提示调用 Context 的 getClass 方法，找不到，所以报错了。不用怀疑，正如你想的那样，这儿的 Context 就是我们上面传入的 Context。破罐子破摔，先重写 callObjectMethodV，返回一个空的类，看贵公子下一步干什么，我们只需要最后补正确就行。

```
@Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "android/content/Context->getClass()Ljava/lang/Class;":{
                return dvmObject.getObjectType();
            }
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    };
```

完整代码以及运行效果

```
package com.right;

import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

public class zuiyou extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    zuiyou() {
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.xiaochuankeji.tieba").build(); // 创建模拟器实例
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\zuiyou\\right573.apk")); // 创建Android虚拟机
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\zuiyou\\libnet_crypto.so"), true); // 加载so到虚拟内存
        module = dm.getModule(); //获取本SO模块的句柄

        vm.setJni(this);
        vm.setVerbose(true);
        dm.callJNI_OnLoad(emulator);
    };

    public void native_init(){
        // 0x4a069
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclass，直接填0，一般用不到。
        module.callFunction(emulator, 0x4a069, list.toArray());
    };

    private String callSign(){
        // 准备入参
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclass，直接填0，一般用不到。
        list.add(vm.addLocalObject(new StringObject(vm, "12345")));
        ByteArray plainText = new ByteArray(vm, "r0ysue".getBytes(StandardCharsets.UTF_8));
        list.add(vm.addLocalObject(plainText));
        Number number = module.callFunction(emulator, 0x4a28D, list.toArray())[0];
        return vm.getObject(number.intValue()).getValue().toString();
    };

    @Override
    public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "com/izuiyou/common/base/BaseApplication->getAppContext()Landroid/content/Context;":
                return vm.resolveClass("android/content/Context").newObject(null);
        }
        return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
    }

    @Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "android/content/Context->getClass()Ljava/lang/Class;":{
                return dvmObject.getObjectType();
            }
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    };

    public static void main(String[] args) throws Exception {
        zuiyou test = new zuiyou();
        test.native_init();
        System.out.println(test.callSign());
    }
}
```

![](https://img-blog.csdnimg.cn/20210603200426164.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

这次报错，在找这个类的 getSimpleName，getSimpleName 是类名，比如类：com.R0ysue.test.abc，类名就是 abc。

让我们捋一下完整的流程，在 com/izuiyou/common/base/BaseApplication 中调用 getAppContext 方法，获得一个 Context 上下文，然后 getClass 获取它的类，最后查看它的类名。类名就是这一系列操作的最终目的，我们前面几步都只浅浅的补了一下，只能说类型给对了，别的都没给。但只要最后的类名给它返回正确的字符串，就没问题。

使用 Objection 的插件 [Wallbreaker](https://github.com/Simp1er/Wallbreaker) 查看相关类（BaseApplication 的 getAppContext 其结果以及类名）

![](https://img-blog.csdnimg.cn/20210603200457738.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20210603200544548.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
完整类名，cn.xiaochaunkeji.tieba.AppController，getSimpleName 即 AppController

修复后完整代码如下

```
package com.right;

import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;

public class zuiyou extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    zuiyou() {
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.xiaochuankeji.tieba").build(); // 创建模拟器实例
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\zuiyou\\right573.apk")); // 创建Android虚拟机
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\zuiyou\\libnet_crypto.so"), true); // 加载so到虚拟内存
        module = dm.getModule(); //获取本SO模块的句柄

        vm.setJni(this);
        vm.setVerbose(true);
        dm.callJNI_OnLoad(emulator);
    };

    public void native_init(){
        // 0x4a069
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclass，直接填0，一般用不到。
        module.callFunction(emulator, 0x4a069, list.toArray());
    };

    private String callSign(){
        // 准备入参
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclass，直接填0，一般用不到。
        list.add(vm.addLocalObject(new StringObject(vm, "12345")));
        ByteArray plainText = new ByteArray(vm, "r0ysue".getBytes(StandardCharsets.UTF_8));
        list.add(vm.addLocalObject(plainText));
        Number number = module.callFunction(emulator, 0x4a28D, list.toArray())[0];
        return vm.getObject(number.intValue()).getValue().toString();
    };

    @Override
    public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "com/izuiyou/common/base/BaseApplication->getAppContext()Landroid/content/Context;":
                return vm.resolveClass("android/content/Context").newObject(null);
        }
        return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
    }

    @Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "android/content/Context->getClass()Ljava/lang/Class;":{
                return dvmObject.getObjectType();
            }
            case "java/lang/Class->getSimpleName()Ljava/lang/String;":{
                return new StringObject(vm, "AppController");
            }
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    };

    public static void main(String[] args) throws Exception {
        zuiyou test = new zuiyou();
        test.native_init();
        System.out.println(test.callSign());
    }
}
```

继续运行

![](https://img-blog.csdnimg.cn/20210603200615988.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
可以看到，接下来获取了类的路径，这一步是什么意思呢？

实际上，这依然是签名校验的一部分，不管是获取类名，还是此处获取类的文件路径，都是在做校验——校验 SO 是否在本 App 内执行。”补 “+” 修复“循环往复，下面一连补两个签名，返回值都根据实际 APP 情况。

```
@Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "android/content/Context->getClass()Ljava/lang/Class;":{
                return dvmObject.getObjectType();
            }
            case "java/lang/Class->getSimpleName()Ljava/lang/String;":{
                return new StringObject(vm, "AppController");
            }
            case "android/content/Context->getFilesDir()Ljava/io/File;":
            case "java/lang/String->getAbsolutePath()Ljava/lang/String;": {
                return new StringObject(vm, "/data/user/0/cn.xiaochuankeji.tieba/files");
            }
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    };
```

继续运行  
![](https://img-blog.csdnimg.cn/20210603200635875.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
检测是否有调试，如法炮制

```
@Override
    public boolean callStaticBooleanMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature){
            case "android/os/Debug->isDebuggerConnected()Z":{
                return false;
            }

        }
        throw new UnsupportedOperationException(signature);
    }
```

![](https://img-blog.csdnimg.cn/20210603200649928.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
使用 Unidbg 的 API 返回 PID

```
@Override
    public int callStaticIntMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature){
            case "android/os/Process->myPid()I":{
                return emulator.getPid();
            }

        }
        throw new UnsupportedOperationException(signature);
    }
```

继续运行

![](https://img-blog.csdnimg.cn/20210603200703939.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
结果与 Frida 主动调用结果完全一致，大功告成！但是，关于 JNI 环境的补充这一块，想必大家还有很多疑惑，整个过程滞涩感比较重，读者恐怕很难感受到其中的连续感。其实这是补 JNI 环境时都会出现的感觉，个人建议使用 Frida 主动调用 + JNItrace 实现一次完整的 JNI trace。然后依照着 trace 做补环境的工作。但实际使用时，会遇到不少问题。比如 JNItrace 的 attach 模式有问题，spawn 模式容易崩溃，且输出过多难以辨识。所以建议写 Demo 加载 SO，然后使用 JNItrace trace 结果，这是一个妥善的方法，但记得时常需要处理 JNI 层的签名校验，在之后我们完整的展示这个过程（事实上，还挺费事和曲折）

### 四、算法还原

因为返回值总是 32 位长度，且明文不变时输出也不变，很容易让人想到哈希算法，尤其是 MD5 算法。但是，样本经过了一定程度的 OLLVM 混淆，很难自上而下或者自下而上逐个模块分析代码逻辑，所以我们需要借助一下工具，当当当， FIndHash 试一下。

FindHash 需要运行数分钟，因为其原理是对哈希算法中的运算特征进行正则匹配，需要对函数逐个反编译，运行结束后，根据提示运行 Frida 脚本

![](https://img-blog.csdnimg.cn/20210603200800539.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
IDA 快捷键 G 跳转到 65540  
![](https://img-blog.csdnimg.cn/20210603201520156.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
编写对该函数的 Hook，因为不确定三个参数是指针还是数值，所以先全部做为数值处理，作为 long 类型看待，防止整数溢出。

```
public void hook65540(){
        // 加载HookZz
        IHookZz hookZz = HookZz.getInstance(emulator);

        hookZz.wrap(module.base + 0x65540 + 1, new WrapCallback<HookZzArm32RegisterContext>() { // inline wrap导出函数
            @Override
            // 类似于 frida onEnter
            public void preCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                // 类似于Frida args[0]
                System.out.println(ctx.getR0Long());
                System.out.println(ctx.getR1Long());
                System.out.println(ctx.getR2Long());
            };

            @Override
            // 类似于 frida onLeave
            public void postCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
            }
        });
    }

    public static void main(String[] args) throws Exception {
        zuiyou test = new zuiyou();
        test.hook65540();
        test.native_init();
        System.out.println(test.callSign());
    }
```

![](https://img-blog.csdnimg.cn/20210603201601538.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
可以看到，参数 2 应该是数组，参数 1 和 3 则像是地址。

采用如下方式打印地址所指向的内存，其效果类似于 frida 中 hexdump。

```
public void hook65540(){
        // 加载HookZz
        IHookZz hookZz = HookZz.getInstance(emulator);

        hookZz.wrap(module.base + 0x65540 + 1, new WrapCallback<HookZzArm32RegisterContext>() { // inline wrap导出函数
            @Override
            // 类似于 frida onEnter
            public void preCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                // 类似于Frida args[0]
                Inspector.inspect(ctx.getR0Pointer().getByteArray(0, 0x10), "Arg1");
                System.out.println(ctx.getR1Long());
                Inspector.inspect(ctx.getR2Pointer().getByteArray(0, 0x10), "Arg3");
            };

            @Override
            // 类似于 frida onLeave
            public void postCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
            }
        });
    }
```

![](https://img-blog.csdnimg.cn/20210603201618717.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
不要管 “md5=xxx，hex=xxx”，这是 Unidbg 中日志输出的固定格式，千万不要当成某种 hook 的结果。

可以发现，参数 1 就是我们 JAVA 层传入的参数 2，而参数 3，意义未知。事实上，参数 3 大概率是 Buffer，它用于存放运算的结果，这是 C 常用的开发习惯，大家记住就好。而参数 2，长度总是和入参的字符串长度一致，所以就是长度。

在 Frida 中，onEnter 中使用到的 arg，onLeave 中无法获取到，因此我们用 this.xxx = args[n] 的方式保存它，然后在 onLeave 中查看这个 buffer 在函数运行完后的结果。

HookZz 也提供了类似的功能，在执行前，push 保存，在后面再 pop 取出，用法如下

```
public void hook65540(){
        // 加载HookZz
        IHookZz hookZz = HookZz.getInstance(emulator);

        hookZz.wrap(module.base + 0x65540 + 1, new WrapCallback<HookZzArm32RegisterContext>() { // inline wrap导出函数
            @Override
            // 类似于 frida onEnter
            public void preCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                // 类似于Frida args[0]
                Inspector.inspect(ctx.getR0Pointer().getByteArray(0, 0x10), "Arg1");
                System.out.println(ctx.getR1Long());
                Inspector.inspect(ctx.getR2Pointer().getByteArray(0, 0x10), "Arg3");
                // push 
                ctx.push(ctx.getR2Pointer());
            };

            @Override
            // 类似于 frida onLeave
            public void postCall(Emulator<?> emulator, HookZzArm32RegisterContext ctx, HookEntryInfo info) {
                // pop 取出
                Pointer output = ctx.pop();
                Inspector.inspect(output.getByteArray(0, 0x10), "Arg3 after function");
            }
        });
    }
```

![](https://img-blog.csdnimg.cn/2021060320163974.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
Hook 结果验证了我们的说法，参数 1 是输入，参数 2 是长度，参数 3 是 buffer，用于存储结果。

接下来我们就要好好分析这个算法了，它疑似 MD5 算法，按 H 键将这四个数转成十六进制

![](https://img-blog.csdnimg.cn/20210603201652289.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
说它疑似 MD5 主要有两个依据

*   输出结果是 32 位，MD5 恰好也是 32 位长度。
*   有四个 IV，MD5 就有四个 IV

但是呢，它不是标准 MD5，看一下标准 MD5 的四个 IV  
![](https://img-blog.csdnimg.cn/20210603201748275.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
可以发现 IV 不一致，我们也可以在 Cyberchef 中验证是否是标准 MD5 的结果。

![](https://img-blog.csdnimg.cn/20210603201802820.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
结果不一致，那么我们很可能遇到了魔改哈希算法。但不必感到惊慌，不熟悉算法原理的可以看一下 SO 基础课的算法部分，对原理的讲解非常深刻细致，我们这里关注于实战的部分。

哈希算法的魔改，最简单的修改点就是修改 IV，此处似乎采用了这种。如下是一份 python 版本带注释的 MD5 源码，我们对应着修改一下 IV，测试一下结果。

```
import binascii

SV = [0xd76aa478, 0xe8c7b756, 0x242070db, 0xc1bdceee, 0xf57c0faf,
      0x4787c62a, 0xa8304613, 0xfd469501, 0x698098d8, 0x8b44f7af,
      0xffff5bb1, 0x895cd7be, 0x6b901122, 0xfd987193, 0xa679438e,
      0x49b40821, 0xf61e2562, 0xc040b340, 0x265e5a51, 0xe9b6c7aa,
      0xd62f105d, 0x2441453, 0xd8a1e681, 0xe7d3fbc8, 0x21e1cde6,
      0xc33707d6, 0xf4d50d87, 0x455a14ed, 0xa9e3e905, 0xfcefa3f8,
      0x676f02d9, 0x8d2a4c8a, 0xfffa3942, 0x8771f681, 0x6d9d6122,
      0xfde5380c, 0xa4beea44, 0x4bdecfa9, 0xf6bb4b60, 0xbebfbc70,
      0x289b7ec6, 0xeaa127fa, 0xd4ef3085, 0x4881d05, 0xd9d4d039,
      0xe6db99e5, 0x1fa27cf8, 0xc4ac5665, 0xf4292244, 0x432aff97,
      0xab9423a7, 0xfc93a039, 0x655b59c3, 0x8f0ccc92, 0xffeff47d,
      0x85845dd1, 0x6fa87e4f, 0xfe2ce6e0, 0xa3014314, 0x4e0811a1,
      0xf7537e82, 0xbd3af235, 0x2ad7d2bb, 0xeb86d391]


# 根据ascil编码把字符转成对应的二进制
def binvalue(val, bitsize):
    binval = bin(val)[2:] if isinstance(val, int) else bin(ord(val))[2:]
    if len(binval) > bitsize:
        raise ("binary value larger than the expected size")
    while len(binval) < bitsize:
        binval = "0" + binval
    return binval


def string_to_bit_array(text):
    array = list()
    for char in text:
        binval = binvalue(char, 8)
        array.extend([int(x) for x in list(binval)])
    return array


# 循环左移
def leftCircularShift(k, bits):
    bits = bits % 32
    k = k % (2 ** 32)
    upper = (k << bits) % (2 ** 32)
    result = upper | (k >> (32 - (bits)))
    return (result)


# 分块
def blockDivide(block, chunks):
    result = []
    size = len(block) // chunks
    for i in range(0, chunks):
        result.append(int.from_bytes(block[i * size:(i + 1) * size], byteorder="little"))
    return result


# F函数作用于“比特位”上
# if x then y else z
def F(X, Y, Z):
    compute = ((X & Y) | ((~X) & Z))
    return compute


# if z then x else y
def G(X, Y, Z):
    return ((X & Z) | (Y & (~Z)))


# if X = Y then Z else ~Z
def H(X, Y, Z):
    return (X ^ Y ^ Z)


def I(X, Y, Z):
    return (Y ^ (X | (~Z)))


# 四个F函数
def FF(a, b, c, d, M, s, t):
    result = b + leftCircularShift((a + F(b, c, d) + M + t), s)
    return (result)


def GG(a, b, c, d, M, s, t):
    result = b + leftCircularShift((a + G(b, c, d) + M + t), s)
    return (result)


def HH(a, b, c, d, M, s, t):
    result = b + leftCircularShift((a + H(b, c, d) + M + t), s)
    return (result)


def II(a, b, c, d, M, s, t):
    result = b + leftCircularShift((a + I(b, c, d) + M + t), s)
    return (result)


# 数据转换
def fmt8(num):
    bighex = "{0:08x}".format(num)
    binver = binascii.unhexlify(bighex)
    result = "{0:08x}".format(int.from_bytes(binver, byteorder='little'))
    return (result)


# 计算比特长度
def bitlen(bitstring):
    return len(bitstring) * 8


def md5sum(msg):
    # 计算比特长度，如果内容过长，64个比特放不下。就取低64bit。
    msgLen = bitlen(msg) % (2 ** 64)
    # 先填充一个0x80，其实是先填充一个1，后面跟对应个数的0，因为一个明文的编码至少需要8比特，所以直接填充 0b10000000即0x80
    msg = msg + b'\x80'  # 0x80 = 1000 0000
    # 似乎各种编码，即使是一个字母，都至少得1个字节，即8bit才能表示，所以不会出现原文55bit，pad1就满足的情况？可是不对呀，要是二进制文件呢？
    # 填充0到满足要求为止。
    zeroPad = (448 - (msgLen + 8) % 512) % 512
    zeroPad //= 8
    msg = msg + b'\x00' * zeroPad + msgLen.to_bytes(8, byteorder='little')
    # 计算循环轮数，512个为一轮
    msgLen = bitlen(msg)
    iterations = msgLen // 512
    # 初始化变量
    # 算法魔改的第一个点，也是最明显的点
    # A = 0x67452301
    # B = 0xefcdab89
    # C = 0x98badcfe
    # D = 0x10325476

    # 魔改IV
    A = 0x67552301
    B = 0xEDCDAB89
    C = 0x98BADEFE
    D = 0x16325476
    # MD5的主体就是对abcd进行n次的迭代，所以得有个初始值，可以随便选，也可以用默认的魔数，这个改起来毫无风险，所以大家爱魔改它，甚至改这个都不算魔改。
    # main loop
    for i in range(0, iterations):
        a = A
        b = B
        c = C
        d = D
        block = msg[i * 64:(i + 1) * 64]
        # 明文的处理，顺便调整了一下端序
        M = blockDivide(block, 16)
        # Rounds
        a = FF(a, b, c, d, M[0], 7, SV[0])
        d = FF(d, a, b, c, M[1], 12, SV[1])
        c = FF(c, d, a, b, M[2], 17, SV[2])
        b = FF(b, c, d, a, M[3], 22, SV[3])
        a = FF(a, b, c, d, M[4], 7, SV[4])
        d = FF(d, a, b, c, M[5], 12, SV[5])
        c = FF(c, d, a, b, M[6], 17, SV[6])
        b = FF(b, c, d, a, M[7], 22, SV[7])
        a = FF(a, b, c, d, M[8], 7, SV[8])
        d = FF(d, a, b, c, M[9], 12, SV[9])
        c = FF(c, d, a, b, M[10], 17, SV[10])
        b = FF(b, c, d, a, M[11], 22, SV[11])
        a = FF(a, b, c, d, M[12], 7, SV[12])
        d = FF(d, a, b, c, M[13], 12, SV[13])
        c = FF(c, d, a, b, M[14], 17, SV[14])
        b = FF(b, c, d, a, M[15], 22, SV[15])

        a = GG(a, b, c, d, M[1], 5, SV[16])
        d = GG(d, a, b, c, M[6], 9, SV[17])
        c = GG(c, d, a, b, M[11], 14, SV[18])
        b = GG(b, c, d, a, M[0], 20, SV[19])
        a = GG(a, b, c, d, M[5], 5, SV[20])
        d = GG(d, a, b, c, M[10], 9, SV[21])
        c = GG(c, d, a, b, M[15], 14, SV[22])
        b = GG(b, c, d, a, M[4], 20, SV[23])
        a = GG(a, b, c, d, M[9], 5, SV[24])
        d = GG(d, a, b, c, M[14], 9, SV[25])
        c = GG(c, d, a, b, M[3], 14, SV[26])
        b = GG(b, c, d, a, M[8], 20, SV[27])
        a = GG(a, b, c, d, M[13], 5, SV[28])
        d = GG(d, a, b, c, M[2], 9, SV[29])
        c = GG(c, d, a, b, M[7], 14, SV[30])
        b = GG(b, c, d, a, M[12], 20, SV[31])

        a = HH(a, b, c, d, M[5], 4, SV[32])
        d = HH(d, a, b, c, M[8], 11, SV[33])
        c = HH(c, d, a, b, M[11], 16, SV[34])
        b = HH(b, c, d, a, M[14], 23, SV[35])
        a = HH(a, b, c, d, M[1], 4, SV[36])
        d = HH(d, a, b, c, M[4], 11, SV[37])
        c = HH(c, d, a, b, M[7], 16, SV[38])
        b = HH(b, c, d, a, M[10], 23, SV[39])
        a = HH(a, b, c, d, M[13], 4, SV[40])
        d = HH(d, a, b, c, M[0], 11, SV[41])
        c = HH(c, d, a, b, M[3], 16, SV[42])
        b = HH(b, c, d, a, M[6], 23, SV[43])
        a = HH(a, b, c, d, M[9], 4, SV[44])
        d = HH(d, a, b, c, M[12], 11, SV[45])
        c = HH(c, d, a, b, M[15], 16, SV[46])
        b = HH(b, c, d, a, M[2], 23, SV[47])

        a = II(a, b, c, d, M[0], 6, SV[48])
        d = II(d, a, b, c, M[7], 10, SV[49])
        c = II(c, d, a, b, M[14], 15, SV[50])
        b = II(b, c, d, a, M[5], 21, SV[51])
        a = II(a, b, c, d, M[12], 6, SV[52])
        d = II(d, a, b, c, M[3], 10, SV[53])
        c = II(c, d, a, b, M[10], 15, SV[54])
        b = II(b, c, d, a, M[1], 21, SV[55])
        a = II(a, b, c, d, M[8], 6, SV[56])
        d = II(d, a, b, c, M[15], 10, SV[57])
        c = II(c, d, a, b, M[6], 15, SV[58])
        b = II(b, c, d, a, M[13], 21, SV[59])
        a = II(a, b, c, d, M[4], 6, SV[60])
        d = II(d, a, b, c, M[11], 10, SV[61])
        c = II(c, d, a, b, M[2], 15, SV[62])
        b = II(b, c, d, a, M[9], 21, SV[63])
        A = (A + a) % (2 ** 32)
        B = (B + b) % (2 ** 32)
        C = (C + c) % (2 ** 32)
        D = (D + d) % (2 ** 32)
    result = fmt8(A) + fmt8(B) + fmt8(C) + fmt8(D)
    return result


if __name__ == "__main__":
    data = str("r0ysue").encode("UTF-8")
    print("plainText: ", data)
    print("result: ", md5sum(data))
```

![](https://img-blog.csdnimg.cn/20210603201831369.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

结果与样本结果一致，因此可以断定，此处就是魔改且只魔改了 IV 的 MD5 算法。但我并不打算在此处结束这篇文章，我们还可以讨论更多的话题。

1. 如何主动调用一个 Native 函数

在 Frida 中可以使用 NativeFunction API 主动调用

```
function call_65540(base_addr){
    // 函数在内存中的地址
    var real_addr = base_addr.add(0x65541)
    var md5_function = new NativeFunction(real_addr, "int", ["pointer", "int", "pointer"])
    // 参数1 明文字符串的指针
    var input = "r0ysue";
    var arg1 = Memory.allocUtf8String(input);
    // 参数2 明文长度
    var arg2 = input.length;
    // 参数3，存放结果的buffer
    var arg3 = Memory.alloc(16);
    md5_function(arg1, arg2, arg3);
    console.log(hexdump(arg3,{length:0x10}));
}

function callMd5(){
    // 确定SO 的基地址
    var base_addr = Module.findBaseAddress("libnet_crypto.so");
    call_65540(base_addr);
}

// frida -UF -l path\hookright.js
```

![](https://img-blog.csdnimg.cn/20210603201847254.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
在 Unidbg 也是类似的，只不过换一下 API 罢了, 让我们来看一下

```
public void callMd5(){
        List<Object> list = new ArrayList<>(10);

        // arg1
        String input = "r0ysue";
        // malloc memory
        MemoryBlock memoryBlock1 = emulator.getMemory().malloc(16, false);
        // get memory pointer
        UnidbgPointer input_ptr=memoryBlock1.getPointer();
        // write plainText on it
        input_ptr.write(input.getBytes(StandardCharsets.UTF_8));

        // arg2
        int input_length = input.length();

        // arg3 -- buffer
        MemoryBlock memoryBlock2 = emulator.getMemory().malloc(16, false);
        UnidbgPointer output_buffer=memoryBlock2.getPointer();

        // 填入参入
        list.add(input_ptr);
        list.add(input_length);
        list.add(output_buffer);
        // run
        module.callFunction(emulator, 0x65540 + 1, list.toArray());

        // print arg3
        Inspector.inspect(output_buffer.getByteArray(0, 0x10), "output");
    };
```

需要注意，在 Unidbg 中，同样的功能有至少两种实现和写法——Unicorn 的原生方法以及 Unidbg 封装后的方法，在阅读别人代码时需要灵活变通。就好比 **getR0long** 和 **emulator.getBackend().reg_read(ArmConst.UC_ARM_REG_R0)**，它们都是获取寄存器 R0 的数值。

![](https://img-blog.csdnimg.cn/20210603201902203.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
在上面，我们演示了 Unidbg 和 Frida 主动调用单个 Native 函数的代码，千万不要小瞧它，这是很有用的技巧，尤其在 Unidbg 中。举个例子，一个样本较为复杂，其中包含大量 JNI 交互，使用 Unicorn 补环境使得整体跑起来非常麻烦，那我们就可以静态分析出关键函数，只模拟执行关键函数，或者从算法还原的角度上讲，单独执行待分析的函数以便减少干扰也是有用的。

第二个问题

哈希算法的 IV 是一个常见且简单的魔改点，在大量样本中都可以看到，事实上，它对分析者的阻挡程度很小，那么如果样本做了更深层的魔改呢？比如当我们对应着修改完 IV，发现结果依然对不上，那么该怎么分析更深的魔改哈希算法呢？

这就是下一篇的样本和内容喽！

### 五、尾声

凭心而论，在补 JNI 环境那块儿讲的有点含糊，想把此处讲清实在不容易，JNItrace 是补 JNI 环境的利器，但它的实操体验并不顺畅。在额外的文章中，我们把这个问题讲清楚，下一篇是深度魔改哈希算法，敬请期待。

资料链接：https://pan.baidu.com/s/1_ydXiPKgG-zpTYu8xwWG8A  
提取码：bm0b