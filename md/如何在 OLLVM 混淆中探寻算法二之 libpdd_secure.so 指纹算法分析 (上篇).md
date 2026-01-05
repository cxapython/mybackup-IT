> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/OCNRA_T-EzMcDscQPBH2RQ)

0x1 - 前言:

这是 <如何在 OLLVM 混淆中探寻算法> 系列的第二篇文章, 相信读者通过前面文章的学习, 已经对 unidbg 有了初步的认识, 这篇是 unidbg 的一个进阶, 样本涉及的依赖相对前面文章的样本来说比较多, 主要是 jni 调用与文件访问的问题, 笔者将拆分成两篇来讲述, 一篇是 unidbg 补环境, 一篇是算法分析. 国内有难度的样本有四个, 分别是 libmetasec_ml.so,libsgmain.so,libmtguard.so,libtiny.so, 还有一个 unidbg 补环境的高峰 -- 同盾样本, 本篇的样本在这五个样本看来, 难度上算简单的.

0x2 - 样本信息:

链接直达:

```
aHR0cHM6Ly93d3cud2FuZG91amlhLmNvbS9hcHBzLzY2NDk1OTEvaGlzdG9yeV92NzgwMDA=
```

version:

```
7.80.0
```

0x3 - 抓包分析与入口定位:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLa787Axug15iayvoz1tzeGfZ9PVvaibriad9DSEibWfs6zrgfXdLPffR8EQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=0)

每个接口的头部都会带上 anti_token 这个参数, 且是 2af 为前缀的, 这个就是我们的目标了, 至于定位入口, 关键字搜索 / hook newstringutf 都能定位到.

笔者就不带着大家去跟一遍了, 看过我前面的文章, 脚本拿过来改一改就能定位了, 贴一下最终的入口.

```
    package com.xunmeng.pinduoduo.secure;
    import android.content.Context;
    /* compiled from: Pdd */
    /* loaded from: classes.dex */
    public class DeviceNative {
        protected static native String info2(Context context, long j);

        protected static native String info3(Context context, long j, String str);
    }
```

这个 info2 方法就是最终的入口位置了, 他所属的 so 是 libpdd_secure.so, 他的入参是一个 content 和 long 类型, long 类型的参数是时间戳, 可以使用我这篇文章的脚本, 把 native 方法的信息枚举出来.

> 文章跳转
> 
> Fashion 哥，公众号：Fashion 哥的爬虫历险记[安卓 Native 层逆向 - 只通过一个 native 方法名枚举出 native 方法的所有信息](https://mp.weixin.qq.com/s/meqLKz8BoFTxbAfx97pDNg)

0x4-Unidbg 搭架子补环境

先搭个架子:

```
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;
import java.io.File;


public class Pdd_Sec extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    private final DvmClass DeviceNative;

    public Pdd_Sec(){
        //创建模拟器实例 (ARM64架构)
        emulator = AndroidEmulatorBuilder.for64Bit()
                .setProcessName("com.xunmeng.pinduoduo")
                .build();
        //获取内存接口
        final Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));  // Android 6.0
        //创建Android虚拟机
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/ApkResFile/Apks/pdd_7.80.0.apk"));
        //设置JNI接口
        vm.setJni(this);
        vm.setVerbose(true);  // 打印详细日志
        //加载目标SO库
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android/src/test/ApkResFile/SoFile/arm64/libpdd_secure.so"), true);  // 替换为实际SO路径
        dm.callJNI_OnLoad(emulator);  // 调用SO初始化函数
        module = dm.getModule();
        //获取目标类
        DeviceNative = vm.resolveClass("com/xunmeng/pinduoduo/secure/DeviceNative");
    }

    public static void main(String[] args){
        Pdd_Sec pdd_sec_test = new Pdd_Sec();
    }
}
```

读者也不必着急复制粘贴, 文章末尾会贴一份补好的环境代码.

0x01 - 补 Jni 调用

右键运行, 发现报了个错

```
com/xunmeng/core/log/Logger->i(Ljava/lang/String;Ljava/lang/String;)V
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tL7UH6ziaG3XbpJ7ujFs96lmoyqs3Yu84ma7LOywcRCqlbuxJ0Z8WGULA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=1)

对于此方法, 他的返回值是 void, 我们直接返回即可.

```
@Override
public void callStaticVoidMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
    switch (signature){
        case "com/xunmeng/core/log/Logger->i(Ljava/lang/String;Ljava/lang/String;)V":
            return;
    }
    super.callStaticVoidMethodV(vm,dvmClass,signature,vaList);
}
```

补完发现没有其他报错了, 红色的日志是样本自己调用的打印方法, 我们可以不管.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tL8qQsicKnSt4ibufgu2E6yuibV4gIY5kAOAxsbfSNplBMPf7KiaP9tIKJfA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=2)

接下来是主动调用目标函数, 这里有两种方法一种是通过符号去调用, 另一种是通过地址去调用, 笔者比较习惯符号调用.

```
//主动调用
public String callInfo2(long jParam) {
    //调用静态native方法
    DvmObject<?> result = DeviceNative.callStaticJniMethodObject(
            emulator,
            "info2(Landroid/content/Context;J)Ljava/lang/String;",
            vm.resolveClass("android/content/Context").newObject(null),
            jParam
    );
    //获取并返回结果
    return (String) result.getValue();
}
```

info2 方法的参数有两个一个是 android/content/Context, 一个是 long,

对于 java 的八大基本数据类型 (byte,char,int,short,long,double,float) 等不用做处理, 可以直接原封不动的传递, 其他的要做包装, 像这样进行一层包装.

```
vm.resolveClass("类型").newObject(null)
```

在 main 进行调用

```
public static void main(String[] args){
    Pdd_Sec pdd_sec_test = new Pdd_Sec();
    String antitoken = pdd_sec_test.callInfo2(1759120327158L);
    System.out.println("Anti_token =======>" + antitoken);
}
```

运行

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLzFSnzDa9iav5BkicMFIby7es1dzwqKgXTic3OL67xTdadx9EPQib2zrz1A/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=3)

报了这个错误

```
android/content/Context->getSystemService(Ljava/lang/String;)Ljava/lang/Object;
```

此方法是 Android 框架中用于获取系统级服务的方法

谷歌官网文档

```
https://developer.android.com/reference/android/content/Context#getSystemService(java.lang.String)
```

如果对方法不是很了解的可以看看谷歌的开发文档

```
https://developer.android.com/reference/
```

可以这样补

```
@Override
public DvmObject<?> callObjectMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {
    switch (signature) {
        case "android/content/Context->getSystemService(Ljava/lang/String;)Ljava/lang/Object;":
            // 获取服务名称
            String serviceName = varArg.getObjectArg(0).getValue().toString();
            System.out.println("请求系统服务: " + serviceName);
            return null;
    }
    return super.callObjectMethod(vm,dvmObject,signature,varArg);
}
```

打印看看是获取了哪些系统服务

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLebOEyj9LoPKhJf7JUlGHeE3rkXotMibo9ib7kq1dsgOw89JibEb7SvAibA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=4)

只获取了一个, 可以写个判断, 然后返回

```
@Override
public DvmObject<?> callObjectMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {
    switch (signature) {
        case "android/content/Context->getSystemService(Ljava/lang/String;)Ljava/lang/Object;":
            // 获取服务名称
            String serviceName = varArg.getObjectArg(0).getValue().toString();
            System.out.println("请求系统服务: " + serviceName);
            // 根据服务名称返回不同的对象
            if (serviceName.equals("phone")) {
                // 返回TelephonyManager对象
                return vm.resolveClass("android/telephony/TelephonyManager").newObject(null);
            }
    }
    return super.callObjectMethod(vm,dvmObject,signature,varArg);
}
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLVmJQgxU2QjbZ0LibD9gGngGx0Piaqt4T1ucxyIhRLnQWYkgvuo89TIpw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=5)

这个需要 frida 去主动调用一下拿到结果, 返回结果是个 string 类型, 要用 unidbg 的 api 封装一下.

```
@Override
public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
    switch (signature) {
        case "com/xunmeng/pinduoduo/secure/EU->gad()Ljava/lang/String;":
            return new StringObject(vm,"49b8d5557d5bafd9");
    }
    return super.callStaticObjectMethodV(vm,dvmClass,signature,vaList);
}
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLxicBvfvKcau7eUa7gkvNJpskib2x0xNb4qErjh5ialJy6KLyA0FicrZSDQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=6)

同样的, 用 frida 主动调用获取结果, 返回时 int, 基本数据类型, 就不用封装.

```
@Override
public int callIntMethod(BaseVM vm, DvmObject<?> dvmObject, DvmMethod dvmMethod, VarArg varArg) {
    if("android/telephony/TelephonyManager->getSimState()I".equals(dvmMethod.toString())){
        return 1;
    }
    return super.callIntMethod(vm, dvmObject, dvmMethod, varArg);
}
```

接着是这个, 主动调用获取结果

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tL07aSibZ4fr3QdLjto8IebCnJtWpkD7cmicZJibkFiaXb5HlTFX3v1gicfsg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=7)

```
case "android/telephony/TelephonyManager->getSimOperatorName()Ljava/lang/String;":
    return new StringObject(vm, "");
```

接着是这个, 主动调用获取结果

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLntOYZcqYdNa7VQtJAIGLbcMqU4VYkcib7oKW2m9vNQZOdGypIbgmCyQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=8)

```
case "android/telephony/TelephonyManager->getSimCountryIso()Ljava/lang/String;":
    return new StringObject(vm, "");
```

接着是这个

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tL8picsFNALnrq7lFuMQWTQ6X6ztryHMOMzWvqibQibB3yRqbr1C7iatskhg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=9)

```
if("android/telephony/TelephonyManager->getNetworkType()I".equals(dvmMethod.toString())){
    return 0;
}
```

下一个是

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLKHgmIEvS8sjpqGzdAXia6QBHYhvERrSETutr85aOdxrBXYibUNat46ag/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=10)

```
case "android/telephony/TelephonyManager->getNetworkOperator()Ljava/lang/String;":
    return new StringObject(vm, "");
```

下一个是

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLqGl5sPGxlpVia4muTDoa8VOE9MVA0Tghg8ol3UtvNBhkLx8pX6PyAiaQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=11)

```
case "android/telephony/TelephonyManager->getNetworkOperatorName()Ljava/lang/String;":
    return new StringObject(vm, "");
```

下一个接着

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLRkH99nP0pmmBFuz2KO2LgMDoQv3oiaVICqicwK72FDK970vQ8UuX2dzA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=12)

```
case "android/telephony/TelephonyManager->getNetworkCountryIso()Ljava/lang/String;":
    return new StringObject(vm, "cn");
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLhD87yOCNQrGy1ShT4FDialZPUicBrlYQZBXeBHvEcstaETFR7ZWcS4JA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=13)

```
if("android/telephony/TelephonyManager->getDataState()I".equals(dvmMethod.toString())){
    return 0;
}
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLWCgxePJ4lKxjS1ia2DlcU7IuicABoVeBSsSzCvvKbpjs8wSHFiaZz4uSg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=14)

```
if("android/telephony/TelephonyManager->getDataActivity()I".equals(dvmMethod.toString())){
    return 0;
}
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLiaybgPN4BgyccNzpQFweTNsWZr582P5SHibSsWrYyKFXsWb8svy8XnUA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=15)

这个方法是判断是否处于调试连接, 给他返回 false

```
@Override
public boolean callStaticBooleanMethod(BaseVM vm, DvmClass dvmClass, String signature, VarArg varArg) {
    switch (signature){
        case "android/os/Debug->isDebuggerConnected()Z":
            return false;
    }
    return super.callStaticBooleanMethod(vm,dvmClass,signature,varArg);
}
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLvtDBcDAco6MbnhT8CgibaKicsQPorBQwHcJBaAdSQvIp0sbL37QHGwDg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=16)

对于这种方法, 不会补怎么办, 看 unidbg 的示例代码, 看 unidbg 如何写的, 我们也跟着写.

点击 (AbstractJni.java:753) 跳转过去

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLlstTqZ0S3vIX1TMDL7jkcknm3DwYcsiasYg5tCJTaFib3WBS3BW4PgEg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=17)

unidbg 提供了两个示例, 我们也跟着改一改

```
@Override
public DvmObject<?> newObject(BaseVM vm, DvmClass dvmClass, String signature, VarArg varArg) {
    switch (signature) {
        case "java/lang/Throwable-><init>()V":
            return vm.resolveClass("java/lang/Throwable").newObject(new Throwable());
    }
    return super.newObject(vm, dvmClass, signature, varArg);
}
```

接着继续

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLfCGeI7nicDShEtjvmylNPOiaInor7DXsibLKnSlFwOLzj3teLA1xCMPEQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=18)

他需要的返回值是

```
[Ljava/lang/StackTraceElement
```

我们可以用 unidbg 提供的 api 嵌套返回.

```
new ArrayObject(vm.resolveClass("java/lang/StackTraceElement").newObject(null),...,...)
```

代码如下

```
case "java/lang/Throwable->getStackTrace()[Ljava/lang/StackTraceElement;":
    return new ArrayObject(
            vm.resolveClass("java/lang/StackTraceElement").newObject(null),
            vm.resolveClass("java/lang/StackTraceElement").newObject(null),
            vm.resolveClass("java/lang/StackTraceElement").newObject(null)
    );
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tL2DkARzHibgKvn9F2udpAqYOQ7XibxhuosyKFNJcgto6MFV6l8QRa0TbQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=19)

```
case "java/lang/StackTraceElement->getClassName()Ljava/lang/String;":
    return new StringObject(vm,"");
```

下面是一个 replaceall 方法

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tL8sC5tt5piczHg4iajibvAggtgLNm0Yl6BZMn2RHSKTmm3RvQGqoCSKlibw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=20)

可以点进去看看 unidbg 如何实现的, 改一改代码即可

```
@Override
public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
    switch (signature) {
        case "java/lang/String->replaceAll(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;":
            // 获取原始字符串
            String original = (String) dvmObject.getValue();
            // 获取正则表达式参数
            DvmObject<?> regexObj = vaList.getObjectArg(0);
            String regex = (String) regexObj.getValue();
            // 获取替换字符串参数
            DvmObject<?> replacementObj = vaList.getObjectArg(1);
            String replacement = (String) replacementObj.getValue();
            // 执行正则替换
            String result = original.replaceAll(regex, replacement);
            return new StringObject(vm, result);
    }
    return super.callObjectMethodV(vm,dvmObject,signature,vaList);
}
```

当补到这个方法时, 离成功已经很接近了, 同时也要注意, 这个 jni 调用也拖带出了第一个算法 gzip

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLF7a2iat4iccTNU2UWibzZev5GVQSYSERtmnr65ZDeksoShXe6le76U2Aw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=21)

```
case "java/io/ByteArrayOutputStream-><init>()V":
    ByteArrayOutputStream OutputMemoryBuffer = new ByteArrayOutputStream();
    return vm.resolveClass("java/io/ByteArrayOutputStream").newObject(OutputMemoryBuffer);
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tL0cbT2sLHKw5ztqYx3Mx8ibMzCevWP56D4auy2AfBBFCC2uwlIJODUeQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=22)

```
case "java/util/zip/GZIPOutputStream-><init>(Ljava/io/OutputStream;)V":
    try{
        ByteArrayOutputStream MemoryBuffer = (ByteArrayOutputStream) varArg.getObjectArg(0).getValue();
        GZIPOutputStream GzipCompressiveFlow = new GZIPOutputStream(MemoryBuffer);
        return vm.resolveClass("java/util/zip/GZIPOutputStream").newObject(GzipCompressiveFlow);
    }catch (Exception e){}
```

注意补这个方法要外层套上 try-catch

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLia9GNfGPNcKBdFrdNGC7OpSMpsR4xTVhicszicEpMBNQ5wjJ78VOstAVA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=23)

对于此方法, 我们也可以看看 unidbg 内部时如何实现的, 然后照猫画虎

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLGLGjHDqRO84CYoXLiaseRibvLASG19qMUa9mMiaJpbhlialKePkzibPg9fg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=24)

```
/**
    * 将 byte 数组转换为 Hex 字符串（大写）
    * @param data byte 数组
    * @return Hex 编码的字符串（大写）
    */
public static String ByteArrayToHex(byte[] data) {
    StringBuilder hexString = new StringBuilder();
    for (byte b : data) {
        String hex = Integer.toHexString(b & 0xFF);
        if (hex.length() == 1) {
            hexString.append('0');
        }
        hexString.append(hex);
    }
    return hexString.toString().toLowerCase();
}

@Override
public void callVoidMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {
    switch (signature) {
        case "java/util/zip/GZIPOutputStream->write([B)V":
            //System.out.println("write ====> "+ new String((byte[])varArg.getObjectArg(0).getValue() , StandardCharsets.UTF_8));
            byte[] rawData = (byte[])varArg.getObjectArg(0).getValue();
            System.out.println("write ====> "+ ByteArrayToHex((byte[])varArg.getObjectArg(0).getValue()));
            GZIPOutputStream GzipCompressiveFlow = (GZIPOutputStream)dvmObject.getValue();
            try{
                GzipCompressiveFlow.write(rawData);
            }catch (Exception e){}
            return;
    }
    super.callVoidMethod(vm,dvmObject,signature,varArg);
}
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLI3wwbxZ823KXSjgJQr3qMvicvxR0SjiaPSPfWoaoGdRVysY6nqq872DA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=25)

```
case "java/util/zip/GZIPOutputStream->finish()V":
    GZIPOutputStream GzipCompressiveFlow1 = (GZIPOutputStream)dvmObject.getValue();
    try{
        GzipCompressiveFlow1.finish();
    }catch(Exception e){}
    return;
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLQ1E07p7gDh1zNGmibZXmJooOZBqPb7LY8qEVWYpAD055yQN81clyeBw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=26)

```
case "java/io/ByteArrayOutputStream->toByteArray()[B":
    ByteArrayOutputStream MemoryBuffer = (ByteArrayOutputStream)dvmObject.getValue();
    byte[] compressedData = MemoryBuffer.toByteArray();
    return new ByteArray(
            vm,
            compressedData
    );
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLccbBxicgNZxpGibg2TWVA7YTibzLLr2tLVmLgjXa6ctM8M6kloQ3V8e1Q/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=27)

```
case "java/util/zip/GZIPOutputStream->close()V":
    GZIPOutputStream GzipCompressiveFlow2 = (GZIPOutputStream)dvmObject.getValue();
    try{
        GzipCompressiveFlow2.close();
    }catch(Exception e){}
    return;
```

补完这个方法过后, 成功的出值了.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLQgPmeM5ccduIjhibrzuFLYpeaJbJUicmtY4c8JZxEic6ic8HSPNBdqWs6A/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=28)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLkKZotS37kYLtia2YDat5wictCxsYPE4TlZB44ANibcPkwicF2qOzhr2ILQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=29)

整体下来是这么多.

0x02 - 补文件访问

接下来是文件访问, 这也是 unidbg 补环境最容易漏掉的问题.

unidbg 开启文件访问记录

implements IOResolver<AndroidFileIO>

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLQGBwtw91H3JvYIxufEXBS5fRtOo6PCibRZw0AdSeXxzSREkBrLYcAOw/640?wx_fmt=png&from=appmsg#imgIndex=30)

添加 IOResolver

```
emulator.getSyscallHandler().addIOResolver(this);
```

实现 resolve 方法

```
@Override
public FileResult<AndroidFileIO> resolve(Emulator<AndroidFileIO> emulator, String pathname, int oflags) {
    System.out.println("file open:"+pathname);
    return null;
}
```

运行后会发现访问了 5 个文件

```
file open:/dev/__properties__
file open:/proc/stat
file open:/system/build.prop
file open:/proc/version
file open:/proc/5620/status
```

笔者分别讲一下这几个文件是啥

```
/dev/__properties__
```

这个是一个文件夹, 是 Android 属性系统的设备文件, 用于系统属性存储和通信, 当应用调用`System.setProperty()/系统服务更新属性/属性服务自动更新时,会发生变化.`

```
/proc/stat
```

`这是一个文件,包含CPU和系统统计信息,每次CPU时间片分配时/系统中断发生时/进程创建/销毁时,几乎实时高频变化.`

```
/system/build.prop
```

`这是一个文件,包含系统构建时的配置属性,正常情况下不会变化,当系统OTA更新时/刷机时/手动修改时.`

```
/proc/version
```

`这是一个文件,显示Linux内核版本,编译时间,编译器版本等信息,仅在内核重新编译或系统更新时变化.`

```
/proc/5620/status
```

`这是一个文件,特定进程的详细状态信息,当进程内存使用变化时/进程状态改变时/线程数变化时发生变化.`

`我们可以分别将这几个文件从手机上面pull到本地,直接返回.`

```
@Override
public FileResult<AndroidFileIO> resolve(Emulator<AndroidFileIO> emulator, String pathname, int oflags) {
    System.out.println("file open:"+pathname);
    String statusPath = String.format("/proc/%d/status", emulator.getPid());
    switch (pathname) {
        case "/system/build.prop":
            return FileResult.<AndroidFileIO>success(
                    new SimpleFileIO(
                            oflags,
                            new File("unidbg-android/src/test/ApkResFile/pdd_files/build.prop"),
                            pathname
                    )
            );
        case "/proc/version":
            return FileResult.<AndroidFileIO>success(
                    new SimpleFileIO(
                            oflags,
                            new File("unidbg-android/src/test/ApkResFile/pdd_files/version"),
                            pathname
                    )
            );
        case "/proc/stat":
            return FileResult.<AndroidFileIO>success(
                    new SimpleFileIO(
                            oflags,
                            new File("unidbg-android/src/test/ApkResFile/pdd_files/stat"),
                            pathname
                    )
            );
        case "/dev/__properties__":
            return null;
    }
    if(statusPath.equals(pathname)){
        return FileResult.<AndroidFileIO>success(
                new SimpleFileIO(
                        oflags,
                        new File("unidbg-android/src/test/ApkResFile/pdd_files/status"),
                        pathname
                )
        );
    }
    return null;
}
```

`new File位置改成你们本地pull文件的路径.`

`最终的补环境代码笔者也贴出来供大家参考.`

```
package com.pdd_anti_token;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.Module;
import com.github.unidbg.file.FileResult;
import com.github.unidbg.file.IOResolver;
import com.github.unidbg.file.linux.AndroidFileIO;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ArrayObject;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.linux.file.SimpleFileIO;
import com.github.unidbg.memory.Memory;
import java.io.ByteArrayOutputStream;
import java.io.File;
import java.util.zip.GZIPOutputStream;


public class Pdd_Sec extends AbstractJni implements IOResolver<AndroidFileIO> {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    private final DvmClass DeviceNative;

    @Override
    public FileResult<AndroidFileIO> resolve(Emulator<AndroidFileIO> emulator, String pathname, int oflags) {
        System.out.println("file open:"+pathname);
        String statusPath = String.format("/proc/%d/status", emulator.getPid());
        switch (pathname) {
            case "/system/build.prop":
                return FileResult.<AndroidFileIO>success(
                        new SimpleFileIO(
                                oflags,
                                new File("unidbg-android/src/test/ApkResFile/pdd_files/build.prop"),
                                pathname
                        )
                );
            case "/proc/version":
                return FileResult.<AndroidFileIO>success(
                        new SimpleFileIO(
                                oflags,
                                new File("unidbg-android/src/test/ApkResFile/pdd_files/version"),
                                pathname
                        )
                );
            case "/proc/stat":
                return FileResult.<AndroidFileIO>success(
                        new SimpleFileIO(
                                oflags,
                                new File("unidbg-android/src/test/ApkResFile/pdd_files/stat"),
                                pathname
                        )
                );
            case "/dev/__properties__":
                return null;
        }
        if(statusPath.equals(pathname)){
            return FileResult.<AndroidFileIO>success(
                    new SimpleFileIO(
                            oflags,
                            new File("unidbg-android/src/test/ApkResFile/pdd_files/status"),
                            pathname
                    )
            );
        }
        return null;
    }

    public Pdd_Sec(){
        //创建模拟器实例 (ARM32架构)
        emulator = AndroidEmulatorBuilder.for64Bit()
                .setProcessName("com.xunmeng.pinduoduo")
                .build();
        //获取内存接口
        final Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));  // Android 6.0
        //创建Android虚拟机
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/ApkResFile/Apks/pdd_7.80.0.apk"));
        //设置JNI接口
        vm.setJni(this);
        vm.setVerbose(true);  // 打印详细日志
        emulator.getSyscallHandler().addIOResolver(this);
        //加载目标SO库
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android/src/test/ApkResFile/SoFile/arm64/libpdd_secure.so"), true);  // 替换为实际SO路径
        dm.callJNI_OnLoad(emulator);  // 调用SO初始化函数
        module = dm.getModule();
        //获取目标类
        DeviceNative = vm.resolveClass("com/xunmeng/pinduoduo/secure/DeviceNative");
    }

    @Override
    public void callStaticVoidMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature){
            case "com/xunmeng/core/log/Logger->i(Ljava/lang/String;Ljava/lang/String;)V":
                return;
        }
        super.callStaticVoidMethodV(vm,dvmClass,signature,vaList);
    }

    @Override
    public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        switch (signature) {
            case "com/xunmeng/pinduoduo/secure/EU->gad()Ljava/lang/String;":
                return new StringObject(vm,"49b8d5557d5bafd9");
        }
        return super.callStaticObjectMethodV(vm,dvmClass,signature,vaList);
    }

    @Override
    public boolean callStaticBooleanMethod(BaseVM vm, DvmClass dvmClass, String signature, VarArg varArg) {
        switch (signature){
            case "android/os/Debug->isDebuggerConnected()Z":
                return false;
        }
        return super.callStaticBooleanMethod(vm,dvmClass,signature,varArg);
    }

    @Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "java/lang/String->replaceAll(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;":
                // 获取原始字符串
                String original = (String) dvmObject.getValue();
                // 获取正则表达式参数
                DvmObject<?> regexObj = vaList.getObjectArg(0);
                String regex = (String) regexObj.getValue();
                // 获取替换字符串参数
                DvmObject<?> replacementObj = vaList.getObjectArg(1);
                String replacement = (String) replacementObj.getValue();
                // 执行正则替换
                String result = original.replaceAll(regex, replacement);
                return new StringObject(vm, result);
        }
        return super.callObjectMethodV(vm,dvmObject,signature,vaList);
    }

    @Override
    public int callIntMethod(BaseVM vm, DvmObject<?> dvmObject, DvmMethod dvmMethod, VarArg varArg) {
        if("android/telephony/TelephonyManager->getSimState()I".equals(dvmMethod.toString())){
            return 1;
        }
        if("android/telephony/TelephonyManager->getNetworkType()I".equals(dvmMethod.toString())){
            return 0;
        }
        if("android/telephony/TelephonyManager->getDataState()I".equals(dvmMethod.toString())){
            return 0;
        }
        if("android/telephony/TelephonyManager->getDataActivity()I".equals(dvmMethod.toString())){
            return 0;
        }
        return super.callIntMethod(vm, dvmObject, dvmMethod, varArg);
    }

    @Override
    public DvmObject<?> callObjectMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {
        switch (signature) {
            case "android/content/Context->getSystemService(Ljava/lang/String;)Ljava/lang/Object;":
                // 获取服务名称
                String serviceName = varArg.getObjectArg(0).getValue().toString();
                System.out.println("请求系统服务: " + serviceName);
                // 根据服务名称返回不同的对象
                if (serviceName.equals("phone")) {
                    // 返回TelephonyManager对象
                    return vm.resolveClass("android/telephony/TelephonyManager").newObject(null);
                }
            case "android/telephony/TelephonyManager->getSimOperatorName()Ljava/lang/String;":
                return new StringObject(vm, "");
            case "android/telephony/TelephonyManager->getSimCountryIso()Ljava/lang/String;":
                return new StringObject(vm, "");
            case "android/telephony/TelephonyManager->getNetworkOperator()Ljava/lang/String;":
                return new StringObject(vm, "");
            case "android/telephony/TelephonyManager->getNetworkOperatorName()Ljava/lang/String;":
                return new StringObject(vm, "");
            case "android/telephony/TelephonyManager->getNetworkCountryIso()Ljava/lang/String;":
                return new StringObject(vm, "cn");
            case "java/lang/Throwable->getStackTrace()[Ljava/lang/StackTraceElement;":
                return new ArrayObject(
                        vm.resolveClass("java/lang/StackTraceElement").newObject(null),
                        vm.resolveClass("java/lang/StackTraceElement").newObject(null),
                        vm.resolveClass("java/lang/StackTraceElement").newObject(null)
                );
            case "java/lang/StackTraceElement->getClassName()Ljava/lang/String;":
                return new StringObject(vm,"");
            case "java/io/ByteArrayOutputStream->toByteArray()[B":
                ByteArrayOutputStream MemoryBuffer = (ByteArrayOutputStream)dvmObject.getValue();
                byte[] compressedData = MemoryBuffer.toByteArray();
                return new ByteArray(
                        vm,
                        compressedData
                );
        }
        return super.callObjectMethod(vm,dvmObject,signature,varArg);
    }

    @Override
    public DvmObject<?> newObject(BaseVM vm, DvmClass dvmClass, String signature, VarArg varArg) {
        switch (signature) {
            case "java/lang/Throwable-><init>()V":
                return vm.resolveClass("java/lang/Throwable").newObject(new Throwable());
            case "java/io/ByteArrayOutputStream-><init>()V":
                ByteArrayOutputStream OutputMemoryBuffer = new ByteArrayOutputStream();
                return vm.resolveClass("java/io/ByteArrayOutputStream").newObject(OutputMemoryBuffer);
            case "java/util/zip/GZIPOutputStream-><init>(Ljava/io/OutputStream;)V":
                try{
                    ByteArrayOutputStream MemoryBuffer = (ByteArrayOutputStream) varArg.getObjectArg(0).getValue();
                    GZIPOutputStream GzipCompressiveFlow = new GZIPOutputStream(MemoryBuffer);
                    return vm.resolveClass("java/util/zip/GZIPOutputStream").newObject(GzipCompressiveFlow);
                }catch (Exception e){}
        }
        return super.newObject(vm, dvmClass, signature, varArg);
    }

    @Override
    public void callVoidMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {
        switch (signature) {
            case "java/util/zip/GZIPOutputStream->write([B)V":
                //System.out.println("write ====> "+ new String((byte[])varArg.getObjectArg(0).getValue() , StandardCharsets.UTF_8));
                byte[] rawData = (byte[])varArg.getObjectArg(0).getValue();
                System.out.println("write ====> "+ ByteArrayToHex((byte[])varArg.getObjectArg(0).getValue()));
                GZIPOutputStream GzipCompressiveFlow = (GZIPOutputStream)dvmObject.getValue();
                try{
                    GzipCompressiveFlow.write(rawData);
                }catch (Exception e){}
                return;
            case "java/util/zip/GZIPOutputStream->finish()V":
                GZIPOutputStream GzipCompressiveFlow1 = (GZIPOutputStream)dvmObject.getValue();
                try{
                    GzipCompressiveFlow1.finish();
                }catch(Exception e){}
                return;
            case "java/util/zip/GZIPOutputStream->close()V":
                GZIPOutputStream GzipCompressiveFlow2 = (GZIPOutputStream)dvmObject.getValue();
                try{
                    GzipCompressiveFlow2.close();
                }catch(Exception e){}
                return;
        }
        super.callVoidMethod(vm,dvmObject,signature,varArg);
    }

    /**
     * 将 byte 数组转换为 Hex 字符串（大写）
     * @param data byte 数组
     * @return Hex 编码的字符串（大写）
     */
    public static String ByteArrayToHex(byte[] data) {
        StringBuilder hexString = new StringBuilder();
        for (byte b : data) {
            String hex = Integer.toHexString(b & 0xFF);
            if (hex.length() == 1) {
                hexString.append('0');
            }
            hexString.append(hex);
        }
        return hexString.toString().toLowerCase();
    }

    //主动调用
    public String callInfo2(long jParam) {
        //调用静态native方法
        DvmObject<?> result = DeviceNative.callStaticJniMethodObject(
                emulator,
                "info2(Landroid/content/Context;J)Ljava/lang/String;",
                vm.resolveClass("android/content/Context").newObject(null),
                jParam
        );
        //获取并返回结果
        return (String) result.getValue();
    }

    //入口
    public static void main(String[] args){
        Pdd_Sec pdd_sec_test = new Pdd_Sec();
        String antitoken = pdd_sec_test.callInfo2(1759120327158L);
        System.out.println("Anti_token =======>" + antitoken);
    }
}
```

至此补环境就告一段落了, 笔者最后还想提一下一个点, 抛砖引玉, 为下篇的算法分析做铺垫.

在我们补的这个方法当中,

```
java/util/zip/GZIPOutputStream->write([B)V
```

他传入了一个参数, 我们打印了出来

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLgcojibgJj7BPASYPiaD5WQT6rcriaGCvicG1hAgTricBpSsib8dgZeJGYFpg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=31)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLTpfqYNCMLdMB9SGdPFticL3sJMcetZUKqFTPujgNeGSbKc8OHJZTUSg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=32)

那么这个数据是什么呢, 我们放到 cyberchef 反 hex 一下

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OmYIPUgIvD82BZ89HLg9tLEU4JMm8mADvYapM6mj3wtbyoxps6cdFny6oQNfKFaiasib5XMSXOiaunQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=33)

是这样的, 其实这是算法的入参 --- 指纹数据, 只不过他做了点自定义的排序, 把指纹分组进行排序.

```
Ljava/io/ByteArrayOutputStream;-><init>()V,
Ljava/util/zip/GZIPOutputStream;-><init>(Ljava/io/OutputStream;)V,
Ljava/util/zip/GZIPOutputStream;->write([B)V,
Ljava/util/zip/GZIPOutputStream;->finish()V,
Ljava/io/ByteArrayOutputStream;->toByteArray()[B,
Ljava/util/zip/GZIPOutputStream;->close()V
```

我们刚刚补的这几个方法其实是实现一个 gzip 压缩算法

```
import java.io.*;
import java.nio.charset.StandardCharsets;
import java.util.Base64;
import java.util.zip.GZIPInputStream;
import java.util.zip.GZIPOutputStream;

public class GzipCompress {

    /**
     * 将 byte 数组转换为 Base64 字符串
     * @param data byte 数组
     * @return Base64 编码的字符串
     */
    public static String ByteArrayToBase64(byte[] data) {
        return Base64.getEncoder().encodeToString(data);
    }

    /**
     * 将 byte 数组转换为 Hex 字符串（大写）
     * @param data byte 数组
     * @return Hex 编码的字符串（大写）
     */
    public static String ByteArrayToHex(byte[] data) {
        StringBuilder hexString = new StringBuilder();
        for (byte b : data) {
            String hex = Integer.toHexString(b & 0xFF);
            if (hex.length() == 1) {
                hexString.append('0');
            }
            hexString.append(hex);
        }
        return hexString.toString().toLowerCase();
    }

    // Hex字符串转字节数组
    public static byte[] hexStringToByteArray(String hex) {
        int len = hex.length();
        byte[] data = new byte[len / 2];
        for (int i = 0; i < len; i += 2) {
            data[i / 2] = (byte) ((Character.digit(hex.charAt(i), 16) << 4)
                    + Character.digit(hex.charAt(i+1), 16));
        }
        return data;
    }

    // GZIP解压方法
    public static String decompressGzip(byte[] compressedData) throws IOException {
        ByteArrayInputStream bis = new ByteArrayInputStream(compressedData);
        GZIPInputStream gzipStream = new GZIPInputStream(bis);
        BufferedReader reader = new BufferedReader(new InputStreamReader(gzipStream, StandardCharsets.UTF_8));
        StringBuilder result = new StringBuilder();
        String line;
        while ((line = reader.readLine()) != null) {
            result.append(line);
        }
        reader.close();
        gzipStream.close();
        bis.close();
        return result.toString();
    }

    // GZIP解压（二进制版本）
    public static byte[] decompressGzipBinary(byte[] compressedData) throws IOException {
        ByteArrayInputStream bis = new ByteArrayInputStream(compressedData);
        GZIPInputStream gzipStream = new GZIPInputStream(bis);
        ByteArrayOutputStream bos = new ByteArrayOutputStream();

        byte[] buffer = new byte[1024];
        int len;
        while ((len = gzipStream.read(buffer)) > 0) {
            bos.write(buffer, 0, len);
        }
        bos.close();
        gzipStream.close();
        bis.close();
        return bos.toByteArray();
    }

    public static void main(String[] args) throws IOException {
            // 创建内存缓冲区
            ByteArrayOutputStream MemoryBuffer = new ByteArrayOutputStream();
            // 创建GZIP压缩流(连接至内存缓冲区)
            GZIPOutputStream GzipCompressiveFlow = new GZIPOutputStream(MemoryBuffer);
            // 写入原始数据
            //byte[] rawData = "HelloWorld".getBytes();
            byte[] rawData = hexStringToByteArray("0100000001d9000b010108326530303661353400090102065869616f6d69000a010307706c6174696e61000c0104094d492038204c69746500090105065869616f6d69000901060673646d363630001c010719514b51312e3139303931302e30303220746573742d6b6579730036010833514b51312e3139303931302e3030322f5631322e302e322e302e514454434e584d3a757365722f72656c656173652d6b657973000501090231300004010a011d0007010b04604a280c0007010c04495c07800003010d000003010e000003010f00000301100000030111000004011201050009011306545255452d48000501140274680003011500000a011607554e4b4e4f574e0003011700000301180000030119000005011a02636e0004011b01000004011c01000064011d61342e342e3139322d706572662d67326263633339336275696c6465724063352d6d6975692d6f74612d62643134362e626a342e392e78323031353031323370726572656c6561736523315468754d6172313132323a33353a343943535432303231000b011e0849b8d5557d5bafd90004011f0100000b0120080000019a116b480c000b0121087405eaab3a3b5fbf00230122203931393539376636323363373461636262376164653865313130336337393334");
            GzipCompressiveFlow.write(rawData);
            // 完成压缩（可选，close会自动调用）
            GzipCompressiveFlow.finish();
            // 关闭流（确保资源释放）
            GzipCompressiveFlow.close();
            // 6. 获取压缩结果
            byte[] compressedData = MemoryBuffer.toByteArray();
            System.out.println("Gzip Result(Basee64) ======>"+ ByteArrayToBase64(compressedData));
            System.out.println("Gzip Result(Hex) ======>"+ ByteArrayToHex(compressedData));

            // 解压
            System.out.println(decompressGzip(hexStringToByteArray("1f8b08000000000000008d50cb6ed340149d4993266d682185f27ea4741bdb337766fc884022808a105528905209b1b167c669c0b1233b2920b1e017588184f801d8b0e07bd8f111b0624c5a21c486cdd1bd67eebde7ccc108a1ca57d4c278350880d2adadde1342fcde63b4842b8bc32c1b261a35f0422d4ec2b146cbb85adf19bdd4499b9b81dae29da381c5c381755c5f1becd09e0d400911362160df04e4e046e75fda09980f8c89eeacd0b993eb448785b69eeb5705aae1a50a65a88a97f106aae36655be155f4c71ac7af769fd0d5ac02bc8c06a09c74b3851420b9985358c4d79b2ec4f95b08e8cebd3f5ddfebdfefdbdbe21ce94ecd912ce21a373be2253b3760197cb1731c2037ce901b729b7c173ada1ab6310c278963ab4c228202e0440c254e5d94859d16c94a81b611487f9d80242b8450463bdf9ab073e77817722f32b95a539a72ef5592493301d1ee8bc186529059bd8627f3a9d145dc739bc6acf632fb2592eb52db3b133cdb244ee87a3d4499283b135c9b3675a4e65c0840a02e5c9180875c10f18014e4179c47739550214c49ed69dededdb7321e7b7df289b3a452e8fe4e637ff4864b3e95f32a651ff2bb549f7b47aa4279413bf4b449704bb835b400050135f6ed81faf7da2e8fb0713f415137713b71b08e1f73f5b9f7f9866a3f170e5dbebe0dd8beb68135f6d534e998410040d622e81458232ae6418521573e67abf0007f5c1e7ba020000")));
            System.out.println(ByteArrayToHex(decompressGzipBinary(hexStringToByteArray("1f8b08000000000000005d513d53db4010dd35b625639310020402244a68d248dc9d3e905c65063283019b7162279e8102e97c0e4a6c30923c9314cce477a4cb2f489b923a1d05053f858ed5903414f7e676f7bdb7bbb30800780355445d28c6bcd075a08285722f0ecf46314ce394361e86597c1a420d8b9566c3f08dfd3853442afd2755b05c4efb23cf63b08ada727bafcd2d1eb080338b3161642acdccafea7b0a1eeaf683eac6472e2c66e5afbdddd96af59af549aa928d440d5598aa7b5d092b05cea088d3b8061a568bc7bb6f6af4a9151b47da0f98c219207894c3e31c667378022498c3128df7b4dc79df7d67ee90d17c213ba1e202d06a8b5ab7b5d73af8d4a2c4b35cb294c33210ed79419e927c05739355c23eae858ee5d0e4c21cab64607e1691947660479378d857c95be99aa378129b675968467dee7856f4c5b102eb9b60dc655cd8e344fd5b6a9d774e26cd30e15c88baedd69d60eb434730c1e90c2ff4c69febeec5e1ef1b6afc921a57d1d0e9443f9b7f7ff52878a59fdf5e5da23f770bebf8daf0a41cd8e41e487bd3713ddf1f4481ebfbbe9479e8b97700378f22df010000"))));
    }
}
```

具体可以看看这个 Java 实现的 gzip 示例, 与我们补环境的方法一一对应, 那么 gzip 压缩算法肯定有参与到算法运算当中, 本篇做一个开端, 读者敬请期待下篇的算法分析, 提前剧透一下涉及到了两个算法 base64 和 aes, 均为标准的.

下期预告:

<如何在 OLLVM 混淆中探寻算法二> 之 libpdd_secure.so 指纹算法分析 (下篇)