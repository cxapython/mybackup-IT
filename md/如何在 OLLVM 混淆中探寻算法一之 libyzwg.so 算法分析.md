> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/ABZ6Lqs_p06HrthGh11lHg?)

0x1 - 前言:

随着移动安全攻防的激进, 裸奔的 so 出场率越来越少, 取而代之的是基于 OLLVM 框架魔改的混淆的 so. 标题为了避显, 延长文章存活时间, 此篇以分析算法为主. 文章内容仅供学习, 切勿用于违法犯罪.

0x2 - 样本信息:

直达地址:

```
aHR0cHM6Ly93d3cud2FuZG91amlhLmNvbS9hcHBzLzYyMDIyMjIvaGlzdG9yeV92MTMxNDExMA==
```

版本信息:

```
v13.141
```

0x3 - 抓包分析:

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQjGW74R7y2cIJxXVuR7QE4EwibQ0o1qiavPRstNuxUQUWdYzKyJicR8rSQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=0)

读者不难发现, 部分的接口的表单中以及响应数据都做了对应的反爬操作, 表单中对请求参数做了加密以及签名, 响应数据也做了加密处理.

0x4 - 加密参数初步分析:

sig:

多次抓包发现,"V3.0" 是一个固定的前缀, 猜测是算法版本, 后面拼接 32 位的 16 进制.

sp:

从密文中分析不出有用的线索.

响应数据加密:

从密文中分析不出有用的线索.

0x5 - 定位入口:

跑一遍算法自吐, 发现并没有实质的线索, 那么加密逻辑可能在 native 层实现.

笔者通过 hook NewStringUTF 来快速定位, 当然也可以选择反编译 apk 做关键字的检索, 或者 hook hashmap 这种常见的数据类型, 定位的方法很多, 这里不做过多的赘述了.

```
function hook_newStr() {
    var symbols = Module.enumerateSymbolsSync("libart.so");
    var addrNewStringUTF = null;
    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i];
        if (symbol.name.indexOf("NewStringUTF") >= 0 && symbol.name.indexOf("CheckJNI") < 0) {
            addrNewStringUTF = symbol.address;
        }
    }
    if (addrNewStringUTF != null) {
        Interceptor.attach(addrNewStringUTF, {
            onEnter: function (args) {
                var c_string = args[1];
                var dataString = c_string.readCString();
                //筛选结果
                if (dataString.includes("V3.0")) {
                    console.log(dataString);
                    //打印堆栈
                    console.log(Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n') + '\n');
                    console.log(Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()));
                }
                if (dataString.includes("zwp")) {
                    console.log(dataString);
                    //打印堆栈
                    console.log(Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n') + '\n');
                    console.log(Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()));
                }
            }
        });
    }
}
```

相信看了我之前写的文章的读者们, 定位这一步应该不成问题, 笔者这里就不带着去定位了, 把文章的篇幅留给算法分析这一块, 笔者贴一下对应的 native 方法信息.

<table><tbody><tr><td data-colwidth="187" width="187"><section><br></section></td><td data-colwidth="187" width="187"><p>sig</p></td><td data-colwidth="187" width="187"><p>sp</p></td><td data-colwidth="253" width="253"><p>响应解密</p></td></tr><tr><td data-colwidth="187" width="187"><p>类</p></td><td data-colwidth="187" width="187"><p>com.twl.signer.YZWG</p></td><td data-colwidth="187" width="187"><p>com.twl.signer.YZWG</p></td><td data-colwidth="253" width="253"><p>com.twl.signer.YZWG</p></td></tr><tr><td data-colwidth="187" width="187"><p>native 方法</p></td><td data-colwidth="187" width="187"><p>private static native byte[] nativeSignature(byte[] bArr, String str);</p></td><td data-colwidth="187" width="187"><p>private static native String nativeEncodeRequest(byte[] bArr, String str);</p></td><td data-colwidth="253" width="253"><p>private static native nativeDecodeContent([BLjava/lang/String;III)[B</p></td></tr><tr><td data-colwidth="187" width="187"><p>在 yzwg.so 偏移</p></td><td data-colwidth="187" width="187"><p>0x21864</p></td><td data-colwidth="187" width="187"><p>0x209a4</p></td><td data-colwidth="253" width="253"><p>0x24dc8</p></td></tr><tr><td data-colwidth="187" width="187"><p>参数解释</p></td><td data-colwidth="187" width="187"><p>bArr 是表单参数进行 url 编码结果, str 是 null</p></td><td data-colwidth="187" width="187"><p>bArr 是表单参数进行 url 编码结果, str 是 null</p></td><td data-colwidth="253"><section>参数 1 为加密的响应数据字节数组, 参数二为 null, 参数三 - 五为 0,1,0</section></td></tr></tbody></table>

0x6-Unidbg 搭架子与补环境:

```
package com.bosszp;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.array.ArrayObject;
import com.github.unidbg.memory.Memory;
import java.io.*;
import java.nio.charset.StandardCharsets;

public class Boss_Signer extends AbstractJni{
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    private final DvmClass YZWG;

    //初始化虚拟机
    public Boss_Signer() {
        //创建模拟器实例 (ARM64架构)
        emulator = AndroidEmulatorBuilder.for64Bit()
                .setProcessName("com.xunmeng.pinduoduo")
                .build();
        //获取内存接口
        final Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));  // Android 6.0
        //创建Android虚拟机
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/apks/BOSS直聘_v13.141.apk"));  // 替换为实际APK路径
        //设置JNI接口
        vm.setJni(this);
        vm.setVerbose(true);  // 打印详细日志
        //加载目标SO库
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android/src/test/resources/example_binaries/arm64-v8a/libyzwg.so"), true);  // 替换为实际SO路径
        dm.callJNI_OnLoad(emulator);  // 调用SO初始化函数
        module = dm.getModule();
        //获取目标类
        YZWG = vm.resolveClass("com/twl/signer/YZWG");
    }
    //入口
    public static void main(String[] args) {
        Boss_Signer signer = new Boss_Signer();
    }
}
```

运行一下, 发现没有那么幸运, 这回需要补一下环境才能正常运行.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQNCviaB8DUOe8s6l3HefUVgVQEbOpwUrTfP4fmycBNrqKeMl1EgZhGtA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=1)

鼠标点击 AbstractJni.java:103 蓝色字体进入 AbstractJni 文件内.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQKhkVwWb8gSulXnibOmXnKz8gFcBZ8CfnpWbX0xYpcibqtEq744UsicAjA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=2)

将鼠标对应位置的这一块方法给扣到我们自己写的架子中.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQiaSL3A8Jr20YqYVsKBIxgqicY2xU8VnxicMpgGdA7TejcRQLHoiaoiboRrA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=3)

然后做一点小小的改造.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQOvQzf6ciaOEqK6VjMLnR9YbHSaMibibgxiabJQmqXIoG6uCcWiaOBwcC7icQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=4)

需要注意, 我们自己写的类需要继承自 AbstractJni. 不然无法重写 getStaticObjectField.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQhba045rLfbqk29O9ltOvjbNaMDuqQj48PMWwnNX4OiajwzZFptHKcQw/640?wx_fmt=png&from=appmsg#imgIndex=5)

对于此方法签名, 他需要返回这种类型的静态字段 Landroid/content/Context;

```
com/twl/signer/YZWG->gContext:Landroid/content/Context;
```

我们就给他返回一个这个类型的.

```
@Override
public DvmObject<?> getStaticObjectField(BaseVM vm, DvmClass dvmClass, String signature) {
    switch (signature) {
        case "com/twl/signer/YZWG->gContext:Landroid/content/Context;":
            return vm.resolveClass("android/content/Context").newObject(null);
    }
    return super.getStaticObjectField(vm,dvmClass,signature);
}
```

接着运行, 又报错

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQVamBDBvwFicAxrkS4swWGnnmIyMWkV5kuQbicicpdzvrjv5nicGHhX0iaTw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=6)

此方法是**根据 UID（用户 ID）获取对应的应用程序包名数组.**

可以这样补

```
@Override
public DvmObject<?> callObjectMethod(BaseVM vm, DvmObject<?> dvmObject, String signature,VarArg varArg) {
    System.out.println("[callObjectMethod call]:::::signature=====>"+signature);
    switch (signature) {
        case "android/content/pm/PackageManager->getPackagesForUid(I)[Ljava/lang/String;":
            int uid = varArg.getIntArg(0);
            //System.err.println("uid:"+uid);
            return new ArrayObject(new StringObject(vm, vm.getPackageName()));
    }
    return super.callObjectMethod(vm,dvmObject,signature,varArg);
}
```

接着继续运行, 又报错了

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQZPavb7Jt0QdbJdG7iaV3tAWiadp1SkAhkrP0OJalGbbcJ6AJ3iaIOMepQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=7)

这个可以这样补

```
@Override
public int callIntMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {
    switch (signature) {
        case "java/lang/String->hashCode()I":
            System.out.println("java/lang/String->hashCode()I enter");
            String str = dvmObject.getValue().toString();
            return str.hashCode();
    }
    return super.callIntMethod(vm,dvmObject,signature,varArg);
}
```

补完发现没有别的报错了, 接下来我们调用下目标方法.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQSAvSuvK8PtZkCLIXOciaibibswQ0VGicKu9o3aVkBtYffyd4icIFfu2tm0Q/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=8)

```
//主动调用方法
//sp
public String nativeEncodeRequest(byte[] bArr, String str){
    //调用静态native方法
    DvmObject<?> result = YZWG.callStaticJniMethodObject(
            emulator,
            "nativeEncodeRequest([BLjava/lang/String;)Ljava/lang/String;",
            bArr,
            str
    );
    //获取并返回结果
    return (String) result.getValue();
}

//sig
public byte[] nativeSignature(byte[] bArr, String str){
    //调用静态native方法
    DvmObject<?> result = YZWG.callStaticJniMethodObject(
            emulator,
            "nativeSignature([BLjava/lang/String;)[B",
            bArr,
            str
    );
    //获取并返回结果
    return (byte[]) result.getValue();
}

//decode content
public byte[] nativeDecodeContent(byte[] bArr, String str, int i11, int i12, int i13){
    //调用静态native方法
    DvmObject<?> result = YZWG.callStaticJniMethodObject(
            emulator,
            "nativeDecodeContent([BLjava/lang/String;III)[B",
            bArr,
            str,
            i11,
            i12,
            i13
    );
    //获取并返回结果
    return (byte[]) result.getValue();
}
```

unidbg 结果需要和真机对比, 看看是否一致.

完整 unidbg 代码如下

```
package com.bosszp;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.android.dvm.api.SystemService;
import com.github.unidbg.linux.android.dvm.array.ArrayObject;
import com.github.unidbg.memory.Memory;
import java.io.*;
import java.nio.charset.StandardCharsets;

public class Boss_Signer extends AbstractJni{

    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    private final DvmClass YZWG;

    //初始化虚拟机
    public Boss_Signer() {
        //创建模拟器实例 (ARM64架构)
        emulator = AndroidEmulatorBuilder.for64Bit()
                .setProcessName("com.xunmeng.pinduoduo")
                .build();
        //获取内存接口
        final Memory memory = emulator.getMemory();
        memory.setLibraryResolver(new AndroidResolver(23));  // Android 6.0
        //创建Android虚拟机
        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/apks/BOSS直聘_v13.141.apk"));  //替换为实际APK路径
        //设置JNI接口
        vm.setJni(this);
        vm.setVerbose(true);  // 打印详细日志
        //加载目标SO库
        DalvikModule dm = vm.loadLibrary(new File("unidbg-android/src/test/resources/example_binaries/arm64-v8a/libyzwg.so"), true); // 替换为实际SO路径
        dm.callJNI_OnLoad(emulator);  // 调用SO初始化函数
        module = dm.getModule();
        //获取目标类
        YZWG = vm.resolveClass("com/twl/signer/YZWG");
    }

    //主动调用方法
    //sp
    public String nativeEncodeRequest(byte[] bArr, String str){
        //调用静态native方法
        DvmObject<?> result = YZWG.callStaticJniMethodObject(
                emulator,
                "nativeEncodeRequest([BLjava/lang/String;)Ljava/lang/String;",
                bArr,
                str
        );
        //获取并返回结果
        return (String) result.getValue();
    }

    //sig
    public byte[] nativeSignature(byte[] bArr, String str){
        //调用静态native方法
        DvmObject<?> result = YZWG.callStaticJniMethodObject(
                emulator,
                "nativeSignature([BLjava/lang/String;)[B",
                bArr,
                str
        );
        //获取并返回结果
        return (byte[]) result.getValue();
    }

    //decode content
    public byte[] nativeDecodeContent(byte[] bArr, String str, int i11, int i12, int i13){
        //调用静态native方法
        DvmObject<?> result = YZWG.callStaticJniMethodObject(
                emulator,
                "nativeDecodeContent([BLjava/lang/String;III)[B",
                bArr,
                str,
                i11,
                i12,
                i13
        );
        //获取并返回结果
        return (byte[]) result.getValue();
    }

    //补环境部分
    @Override
    public DvmObject<?> getStaticObjectField(BaseVM vm, DvmClass dvmClass, String signature) {
        System.out.println("[getStaticObjectField call]::::::signature======>"+signature);
        switch (signature) {
            case "com/twl/signer/YZWG->gContext:Landroid/content/Context;":
                return vm.resolveClass("android/content/Context").newObject(null);
        }
        return super.getStaticObjectField(vm,dvmClass,signature);
    }

    @Override
    public DvmObject<?> callObjectMethod(BaseVM vm, DvmObject<?> dvmObject, String signature,VarArg varArg) {
        System.out.println("[callObjectMethod call]:::::signature=====>"+signature);
        switch (signature) {
            case "android/content/pm/PackageManager->getPackagesForUid(I)[Ljava/lang/String;":
                int uid = varArg.getIntArg(0);
                //System.err.println("uid:"+uid);
                return new ArrayObject(new StringObject(vm, vm.getPackageName()));
        }
        return super.callObjectMethod(vm,dvmObject,signature,varArg);
    }

    @Override
    public int callIntMethod(BaseVM vm, DvmObject<?> dvmObject, String signature, VarArg varArg) {
        switch (signature) {
            case "java/lang/String->hashCode()I":
                System.out.println("java/lang/String->hashCode()I enter");
                String str = dvmObject.getValue().toString();
                return str.hashCode();
        }
        return super.callIntMethod(vm,dvmObject,signature,varArg);
    }


    public static void main(String[] args) {
        Boss_Signer signer = new Boss_Signer();

        String sp = signer.nativeEncodeRequest("account=vGMZRH2Mc2yGuEY%3D&client_info=%7B%22version%22%3A%2210%22%2C%22os%22%3A%22Android%22%2C%22start_time%22%3A%221756205896294%22%2C%22resume_time%22%3A%221756205896294%22%2C%22channel%22%3A%2228%22%2C%22model%22%3A%22google%7C%7CPixel+3%22%2C%22dzt%22%3A0%2C%22loc_per%22%3A0%2C%22uniqid%22%3A%223e818bd6-0d55-45de-8aca-44216661d569%22%2C%22oaid%22%3A%225752bf62-1e11-4a54-979a-c2786373ab6c%22%2C%22oaid_honor%22%3A%225752bf62-1e11-4a54-979a-c2786373ab6c%22%2C%22did%22%3A%22DUuJwkteNiOzCkxbSM3SoAheX_4j-JxcaP7dRFV1SndrdGVOaU96Q2t4YlNNM1NvQWhlWF80ai1KeGNhUDdkc2h1%22%2C%22tinker_id%22%3A%22Prod-arm64-v8a-release-13.141.1314110_0812-10-06-09%22%2C%22is_bg_req%22%3A0%2C%22network%22%3A%22wifi%22%2C%22operator%22%3A%22UNKNOWN%22%2C%22abi%22%3A1%7D&curidentity=0&identityType=-1&isWxLogin=false&phoneCode=2598®ionCode=%2B86&req_time=1756207770991&uniqid=3e818bd6-0d55-45de-8aca-44216661d569&v=13.141".getBytes(),null);
        byte[] sig = signer.nativeSignature("/api/zppassport/user/codeLoginaccount=vGMZRH2Mc2yGuEY%3D&client_info=%7B%22version%22%3A%2210%22%2C%22os%22%3A%22Android%22%2C%22start_time%22%3A%221756205896294%22%2C%22resume_time%22%3A%221756205896294%22%2C%22channel%22%3A%2228%22%2C%22model%22%3A%22google%7C%7CPixel+3%22%2C%22dzt%22%3A0%2C%22loc_per%22%3A0%2C%22uniqid%22%3A%223e818bd6-0d55-45de-8aca-44216661d569%22%2C%22oaid%22%3A%225752bf62-1e11-4a54-979a-c2786373ab6c%22%2C%22oaid_honor%22%3A%225752bf62-1e11-4a54-979a-c2786373ab6c%22%2C%22did%22%3A%22DUuJwkteNiOzCkxbSM3SoAheX_4j-JxcaP7dRFV1SndrdGVOaU96Q2t4YlNNM1NvQWhlWF80ai1KeGNhUDdkc2h1%22%2C%22tinker_id%22%3A%22Prod-arm64-v8a-release-13.141.1314110_0812-10-06-09%22%2C%22is_bg_req%22%3A0%2C%22network%22%3A%22wifi%22%2C%22operator%22%3A%22UNKNOWN%22%2C%22abi%22%3A1%7D&curidentity=0&identityType=-1&isWxLogin=false&phoneCode=2369®ionCode=%2B86&req_time=1756207335188&uniqid=3e818bd6-0d55-45de-8aca-44216661d569&v=13.141".getBytes(),null);
        byte[] decode_content = signer.nativeDecodeContent(
                new byte[]{-10, 114, 76, 25, 44, -37, 100, 101, -128, -89, 82, -77, -46, -79, -105, 125, -105, -32, -97, 119, 58, 83, -13, -111, -30, -102, -103, -4, -64, 116, 71, 21, -99, 2, -95, 113, -79, 114, 8, 37, 105, 120},
                null,
                0,
                1,
                0
        );

        System.out.println("sig======>"+new String(sig, StandardCharsets.UTF_8));
        System.out.println("sp======>"+sp);
        System.out.println("decode_content======>"+new String(decode_content));
    }

}
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQtQoO0xFo9CC5qa8CSdibRAA2q0SdUbHxsmaJ9tcAxTYrgp9RstbGEVA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=9)

对比真机

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQHne58Tia8Uia2oJvjHzgRA86OZiban5fictvu1zYibdeZCfMR8H449WDicZA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=10)

结果一致, 没有问题, 响应解密出来也正常, 不是乱码之类的东西.

0x7-Ida 静态分析 + Unidbg 下断点还原算法:

0x01-sig(加盐 md5 算法):

由于 nativeSignature 方法返回值是字节数组, 可以以 JNIEnv->SetByteArrayRegion 为切入点, ida 按 g 跳转此偏移 f5 看看上下文.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQk6yI6Av1t3ibHZ1A7E3fFSuV9u7qfS2UkyebgVHPjR0mU2XFYwXkCmg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=11)

通过分析可得, v28 是 v27 的长度, 而 v29 是 v27, 那么研究对象就是这个 v27 是怎么来的, 他是通过一个 sub_1C38C 方法调用所返回的, 我们可以在这个下断点在这个方法.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQByOMgeJf2Vhzric6oPib4rzHyf3X0ArsFjvqWL8JxLVn0ITRODuib2yew/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=12)

```
void sigDebug(){
    emulator.attach().addBreakPoint(module,0x1C38C);
}
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQa5zoIDcdQM0l4icAerIERCL3VabY9c6LBFEGlsAic2S6k312pexzvyibw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=13)

可以写出方法的形式方便管理.

这里 ida 识别的有点问题, 应该是两个参数的, ida 识别成了一个, 鼠标放在这个位置, 按一下 f5,ida 即可重写识别.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQyqZo8kpR7V16JudRSSKhBvawibMXg9NLTvEVqaFQV8Ekmw4HwBicpuRg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=14)

运行我们搭好的架子, 成功断下

mx0 查看寄存器值, 也就是我们第一个参数

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQn4sXOzBqG3ASvApiabEGWFcfRz5a4Chok6rwGC2gNY9DTgOibUe3Tvmg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=15)

第二个参数是一个数值, 是 mx0 这个值的长度, 不是地址, 不能通过 mx1 这样查看, 不然会报错.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQpgcicY5nZNcNx01bsPbsM6ichUgqm6qib1xvVU08f1ibTajnzibgYtibIMFw/640?wx_fmt=png&from=appmsg#imgIndex=16)

通过这样的命令即可查看第一个参数完整的值了

```
mx0 0x3c9
```

拿到 cyberchef 里面 from hexdump 一下

可以发现前面是我们传入的签名数据, 后面的 32 位不知道是啥

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQnt7VDaIvTcevnt4VUxbWGraS9gbabGDMAibMKUpujhFTpfcicePnh5og/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=17)

```
a308f3628b3f39f7d35cdebeb6920e21
```

我们之前也有做过猜想, sig 的值有可能是 md5 加密某个数据, 这里我们也来验证一下.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQmHFRztWLV9kWLfeWghU0QdbLLXab96NOXMcCrSMLMmBkDwKibxXImQg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=18)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQkjP95PsNhEYhv5kNAxR12Zls3aah1rdNhRbjPTo373OZib2ib9l9AsJA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=19)

发现是能够对的上的, 现在我们未知的是这个 32 位的数据是从哪来的, 是固定的还是动态的呢? 我们通过改变签名数据发现, 这个值不会变化, 而且代入到我们真机 hook 的数据中也能签名一致, 那么我们姑且相信他是固定的, 读者也可以自行研究研究.

sig 的算法是 "V3.0" 的前缀拼接 md5 加密 urlpath + 请求参数, 至此 sig 算法分析结束.

0x02-sp(标准 base64 + 标准 rc4 + 魔改 lz4 压缩):

sp 是调用的 nativeEncodeRequest 方法生成结果, 返回值是字符串, 调用 NewStringUTF 转换 java 和 c 层数据, 这个可以是我们的切入点, 上篇文章也有讲过, 这篇我们换一个思路.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQfsZAwPa01jkEicib3Eq2licoXXHg8oKH4HKItIdgw6rwIJgCzKwWIQc9w/640?wx_fmt=png&from=appmsg#imgIndex=20)

通过搜索 base64 编码表来定位,

打开 Strings 这一栏

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQQqLCzP8RQx16R8n7EiaWcyLYicuos1IhJe2woPpekwDicLRQdt5AhUwaA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=21)

搜索一下 base64 的码表

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQ2OFqrrpribsaKlu1kL9EqbVg8NjLSlAncb8QdAVslJnj1bd7J8vJBeA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=22)

发现确实有, 点击跳转过去, 鼠标放在上面按 x 查找交叉引用

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQnMtFBZokCkGDtPDicTOssKiaS8UxtCKm0ZsqNd93nUyibVmKiaMbZjQH9w/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=23)

引用的位置都指向 sub_29E90 这个函数, f5 反编译看看

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQZjSgMOgSV2bFttBDVtibfYCL8qzBeglNfJiczL2taveSUIRd3RiboIBbw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=24)

发现代码中有很多 if-else,cfg 流程图中块与块之间的联系变的很复杂, 静态分析很难看出来块与块之间的联系, 像这种是做了 ollvm 的混淆处理的, 抗 ida 的静态分析, 既然他用到了 base64 编码表, 那么这个函数有没有可能是实现 base64 编码逻辑的呢?

我们可以在此函数下个断点看看, 如果断住了, 是不是就说明 sp 参数有用到这个算法呢?

```
emulator.attach().addBreakPoint(module,0x29E90);
```

```
sub_29E90(_BYTE *a1, __int64 a2, int a3)
```

这个函数有三个参数, 我们看看分别是什么

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQ48dgYGy44LU2g69115VfQOfmJsLBQAg494STRdiaEBJffB8GA91ADMA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=25)

x0,x1 是内存地址, 而 x2 是 x1 这块内存地址的占用大小.

通过命令查看发现, 这块内存地址什么都没有, 这其实是个内存缓冲区, 函数离开后会将结果写入到此处

```
mx0
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQ0SyWia1yU2F3tr4ZgzVwqEXhhK79fglUkKPJEZNFCKynqX7VQktHpVw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=26)

```
mx1 0x2b1
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQorUd0DoGia4QGqPodEgiaNJt8kacuIZa366lfq1xiajy3nicKMV1BqXelA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=27)

x1 这块内存地址看着不像我们的明文, 对其进行标准 base64 编码看看与加密结果一致不一致.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQfibSmzmVicZbJvnamCTfFqZ7ezt92xbPUkGDVu3iazl1nH8qDsakQ8ZKQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=28)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQ25Z2pJTBwwOqkwdfND6XHnBM467VvIDtPSnqQVhhMuDibWjg1odLC1g/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=29)

对比之后, 眼光好一点的读者, 可以发现, base64 码表做了一点小的改动,

/ ---> _

+ --->-

= ---> ~

我们也可以通过 cyberchef 将码表改成下面这个

```
A-Za-z0-9-_~
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQU2UnzBhicJoc1u3WLgSfnxJjFGibreCbdZHgKrNFOPw7sOuDzgj7yW3w/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=30)

copy 一下搜索发现确实是标准的只不过改了下码表

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQ0VlcIlfUHjXsOnb68keYK3OzdsdxSB7zycWribumJN4M1KTorbCvVMg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=31)

那么我们现在的研究对象是这个编码的密文是哪来的,

鼠标放在这个函数上面, 按住 x 查找交叉引用

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQIDlAb3jC68mefAROOKp1ZUMolEPaozR2aTKXibzqes7elZUKHPMqIzQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=32)

发现有多个引用位置, 换一个方法, 可以通过 unidbg 打印调用堆栈, 这个比较快一些.

还是在我们刚刚的断点处, 输入下面命令, 即可打印调用堆栈

```
bt
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQ5gQ2jBwOv73gS06FH1KteXVlCNiciaW4Qh3M09rXyBeOqcPP0bYibG3nw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=33)

```
[0x040000000][0x040020ff4][libyzwg.so][0x020ff4]
```

ida 跳转到这个偏移位置

```
0x020ff4
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQ6O5qa2wywQqd6TOHOiajyOU2KHE4zIJG9eRR4le5PSFbLqXtqKq1DJQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=34)

第二个参数是我们需要研究的对象, 也就是这个 ptr, 他来自于 sub_2E91C 这个函数, ptr 应该是内存的缓冲区, 通过这个函数执行后写入数据到这块内存的.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQD926ia0IX7Fbxnv6RRcTXZIzIfK2Bhx8TRJA4V4P91pMBtO7o5NLMTA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=35)

对 sub_2E91C 下断点看看

```
emulator.attach().addBreakPoint(module,0x2E91C);
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQUGtOSr0XlReibdL5Wd7Cx70NHkK9LvLZBhvKpPXNYJcHHc7ex8pTtBQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=36)

sub_2E91C 函数伪 C 代码

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQjODIc0j3dibxxkjGMHGG8r1vKhDaZX5uIicF3ENCTNicIIicSOgYcB9nicQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=37)

第一个参数

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQIiaVxfL8jNyBGI3jUefWtwD2WpciasH7oPw8GagNGDJOflgsRMM7qMng/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=38)

第二个参数

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQkSic3HLDv4oN2pncGvIq2B1LwuJYshpPND7yxIDNaibXRkCgVxSiaZl9w/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=39)

第三个跟第二个相同, 第四个是数据长度.

第二个参数看起来有点像我们的明文, 但是不完全是.

通过分析 sub_2E91C 函数的伪 C 代码, 我们发现这是一个 rc4 算法.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQM3PD5ic2icej2iaAKHhwP9s0osjNNdBrP0U7ctQ4QkThGWOPpGeTwsZyw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=40)

这个函数是 rc4 主要的加密 / 解密函数, 但是还有 s 盒的初始化算法, 在他的上一个函数.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQJzP6duK8WDY04Wf1aic8s4YUIyU1130Q9djBPOJ3sPqWMbCYm7OySeA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=41)

v62 就是对应的 s 盒.

s 对应的是 key.

v53 是 key 的长度.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQ5KOwiaEMciath1VCtn60Gy5aZo8eYRBnTnlgTwHZftsLns2y77H2YPZw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=42)

下个断点看看 key 是什么.

```
emulator.attach().addBreakPoint(module,0x2E680);
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQtxw5mN7gEiaPDDxKhpjib2am6pYRwDOI55D6LOmib8CQ0ZIA8ZDhDLu5A/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=43)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQVpQm5vSryAiaOGjP2zZVWpWgWl0mRJN1ABcFriacevibSyn2qZOaz2jcg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=44)

发现其实是我们之前分析的 sig 的盐值.

拿到网站上面测试一下, 看看是否是标准的, 验证我们的猜想.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQH6HiadJUye5Dot2P9LibfpD7s8cX6Trf7icGlapDtTLv6jLtKM67PfCbQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=45)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQ6Yl9FicUaAcmIPdFJhMpALqwzMXyYdNyuQpw45PKeoyAabLhYVHAl7A/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=46)

确实就是标准的, 现在离成功又近了一步.

我们的研究对象现在是这个跟明文很像的东西是个啥.

通过查找交叉引用

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQfE4OLg7RAiccgJ6uAJ6icLllX1w9ibRsnGA5NGFghicL40UAibaK9ELxoBQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=47)

发现来自上面那个函数.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQcCxR2CwcwtmQXDNWgVB5niclPtEeHlicWCAdgsiaAKl7hRhWI5AvVichhg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=48)

那就下个断点看看情况.

```
emulator.attach().addBreakPoint(module,0x1D444);
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQT5NagfBIBHxCVZgzBT7hkpBZUWYREOcsvicaXUg9ibxnjNib6RW1EEhPw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=49)

第一个参数是我们的明文数据, 第三个是明文数据的长度

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQutP6qz31Dq6sC0llzpLBibUt3md4Ktc1ypA2F84sUgk0DicHbyicefB3Q/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=50)

第二个是一个内存的缓冲区

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQfrGCHebO2icgIh9WC9J8DNsvRlpjyOxJaibFkOHweLVQG4vFDGLAm16w/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=51)

sub_1D444 函数伪 C 代码

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQdYoKsRoauf79HxANXBtL8xShnibich4QPomJL2mUYlLWsOhHozibvbqFw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=52)

根据频繁出现的 lz4 字眼, 猜测可能跟这个压缩算法有关.

通过互联网搜索 LZ4_compress_limitedOutput 关键字, 发现了一个与 lz4 相关的开源项目.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQCqmo7mOvQQTDyNB3lAzibTEltYqHGUU2OO1A5auOANN4ricicQFKI8LMQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=53)

```
https://github.com/lz4/lz4
```

也许这个样本使用了开源的框架. 但是他的压缩结果显然不符合标准的 lz4.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQWuzibbNKGzsleqQWRTDx8m8HFlTicXjGaJan3nrIwbK4hJQA0hGSjeicw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=54)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQeBNHahsVQREbLjAXMbQ0aTXGC9DGgfwF0joTXp7BTZdmy5pNPqyPMw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=55)

仔细分析伪 C 的这部分

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQgkdlFfBB4sCDpGyycntTibCsvDxiaQgSQSicHjJyyHRwcnx5XVHb5ib6dQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=56)

发现他多分配了 24 个字节

sub_1D444 函数内部

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQeYtq4u4gsTXvegBRQ7gFhLKqWpCtmib0bckX3mNMtPYBOiaN0mSWku6Q/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=57)

也有对这 24 字节的写入相关的代码.

或许就改动了压缩的头部魔数.

python 实现 lz4 压缩

```
import lz4.frame
import lz4.block
def lz4_compress_fast_extstate_python(data, acceleration=1):
    # 使用帧压缩（推荐，包含完整的帧头信息）
    compressed = lz4.frame.compress(
        data,
        compression_level=acceleration,  # 1=最快，9=最高压缩率
        block_linked=True,               # 使用链式块
        content_checksum=False,          # 校验和
        block_checksum=False
    )
    return compressed
def lz4_decompress_python(compressed_data):
    """解压函数"""
    return lz4.frame.decompress(compressed_data)
# 测试
if __name__ == "__main__":
    test_data = b"account=vGMZRH2Mc2yGuEY%3D&client_info=%7B%22version%22%3A%2210%22%2C%22os%22%3A%22Android%22%2C%22start_time%22%3A%221756205896294%22%2C%22resume_time%22%3A%221756205896294%22%2C%22channel%22%3A%2228%22%2C%22model%22%3A%22google%7C%7CPixel+3%22%2C%22dzt%22%3A0%2C%22loc_per%22%3A0%2C%22uniqid%22%3A%223e818bd6-0d55-45de-8aca-44216661d569%22%2C%22oaid%22%3A%225752bf62-1e11-4a54-979a-c2786373ab6c%22%2C%22oaid_honor%22%3A%225752bf62-1e11-4a54-979a-c2786373ab6c%22%2C%22did%22%3A%22DUuJwkteNiOzCkxbSM3SoAheX_4j-JxcaP7dRFV1SndrdGVOaU96Q2t4YlNNM1NvQWhlWF80ai1KeGNhUDdkc2h1%22%2C%22tinker_id%22%3A%22Prod-arm64-v8a-release-13.141.1314110_0812-10-06-09%22%2C%22is_bg_req%22%3A0%2C%22network%22%3A%22wifi%22%2C%22operator%22%3A%22UNKNOWN%22%2C%22abi%22%3A1%7D&curidentity=0&identityType=-1&isWxLogin=false&phoneCode=2598®ionCode=%2B86&req_time=1756207770991&uniqid=3e818bd6-0d55-45de-8aca-44216661d569&v=13.141"
    # 压缩
    compressed = lz4_compress_fast_extstate_python(test_data, acceleration=1)
    print(compressed.hex())
```

我们可以用 python 压缩数据得到一份和 unidbg 的做个对比.

通过两组数据对比, 不难发现, 这个样本魔改的 lz4 算法的魔改点是头部的魔数, 也就是那 24 字节, 主要的压缩逻辑没有做魔改.

通过对伪 C 的分析, 我们确定了这 24 个字节的来源

```
1-8字节: 42 5a 50 42 6c 6f 63 6b 固定 (8字节)
9-16字节:00000000 (固定 4字节) + lz4算法压缩长度(不包含头和尾巴00)(4字节)
17-24字节:数据长度(4字节) + (数据长度 ^ lz4算法计算的结果)(4字节)
```

这个 v9 是压缩后的长度, 通过 python 压缩的结果长度, 去掉标准的头以及后面填充的 00 之后的长度就是压缩后的长度.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQ7XByVtaEIY4UKcadQqliaSftphibicQbaJpKergO7J9mYhkIiaib72icF97w/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=58)

至此 sp 算法分析结束.

0x03 - 响应数据的解密算法 (rc4)

貌似无法根据已有的线索去推断, 直接 trace(踹死) 一份汇编执行流.

```
//trace工具
void traceCode() {
    String traceFile = "unidbg-android/src/test/java/com/bosszp/sp_traceCode.log";// 输出的路径
    PrintStream traceStream = null;// 打印流
    try {
        traceStream = new PrintStream(new FileOutputStream(traceFile), true);
    } catch (FileNotFoundException e) {
        throw new RuntimeException(e);
    }
    // traceCode 对代码进行监控
    emulator.traceCode(module.base, module.base + module.size).setRedirect(traceStream);
}
```

在执行之前进行 trace

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQicxq6v115fpftOO8gVhpTjI52icQiaT6ZASfYE4fYBtcsxzpDHPBeyTjQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=59)

将 trace 的日志丢到 010editor 里面分析.

nativeDecodeContent 最终返回的是 byte[] 类型我们把它转成了字符串, 这里我们需要再把字符串转成 hex 在 010editor 里面进行检索.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQYTqAs73qfLOYg43uAjB6WchkZIUSyWmIGGrA4Z13XzBD0KGmwqgk5A/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=60)

```
00000000  7b 22 63 6f 64 65 22 3a 30 2c 22 6d 65 73 73 61  |{"code":0,"messa|
00000010  67 65 22 3a 22 53 75 63 63 65 73 73 22 2c 22 7a  |ge":"Success","z|
00000020  70 44 61 74 61 22 3a 7b 7d 7d                    |pData":{}}|
```

根据现有的信息, 我们总结一下

```
因:
[-10, 114, 76, 25, 44, -37, 100, 101, -128, -89, 82, -77, -46, -79, -105, 125, -105, -32, -97, 119, 58, 83, -13, -111, -30, -102, -103, -4, -64, 116, 71, 21, -99, 2, -95, 113, -79, 114, 8, 37, 105, 120]
转16进制:
F6 72 4C 19 2C DB 64 65 80 A7 52 B3 D2 B1 97 7D 97 E0 9F 77 3A 53 F3 91 E2 9A 99 FC C0 74 47 15 9D 02 A1 71 B1 72 08 25 69 78

果:
[123, 34, 99, 111, 100, 101, 34, 58, 48, 44, 34, 109, 101, 115, 115, 97, 103, 101, 34, 58, 34, 83, 117, 99, 99, 101, 115, 115, 34, 44, 34, 122, 112, 68, 97, 116, 97, 34, 58, 123, 125, 125]
转16进制:
7b 22 63 6f 64 65 22 3a 30 2c 22 6d 65 73 73 61 67 65 22 3a 22 53 75 63 63 65 73 73 22 2c 22 7a 70 44 61 74 61 22 3a 7b 7d 7d
```

在汇编执行流中按单个字节搜索, 然后定位由果溯因, 数据溯源.

搜索的结果有 156 个

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQ2iccskngIBaxQ39FTcaZXVCkFRcstiagQCxx7uKCUmmWwYpOrWRHBBFQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=61)

定位的话, 要花点时间, 一个一个分析下断点, 笔者定位到的最终位置是第 151 个

```
[12:44:45 923][libyzwg.so 0x02e9bc] [4d682f38] 0x4002e9bc: "strb w13, [x2, x15]" w13=0x7b x2=0x403d4000 x15=0x0 => w13=0x7b
```

010editor 里面搜索这个偏移

```
0x02e9bc
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQasMoicIMwmXABR8xHpfDypKUaKjPl9wg4c74YTLZTVFzZ2SibX4UJUqQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=62)

与结果是对的上的, 那就是这个位置了, ida 跳转到此处看看上下文.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQU0Mf5xf989HKA8mRMPibYicficmfFWZ9fgWufMeeElBx036mfpqRiaWKcA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=63)

可以发现是我们刚刚分析过的 rc4, 用原来的 key 解密一下看看能否得到结果

```
a308f3628b3f39f7d35cdebeb6920e21
```

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQwKd5nbBK1oUGQgZ9Y1gUO5fibGj3KBteZiapd89GQN52Ss3icmk4ls4mw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=64)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQd9pJ4D7zWXLvT52siamTOjF6C0mOkpnJsWVDzsDSsbibqQbLPIN2DD2w/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=65)

发现能够正常的解密

至此 libyzwg.so 的算法就分析完了.

这里额外提几个点, 经过我的测试, 新版本的算法差别不大, 几乎没有改变.

新版本把 64 位的 so 换成了 32 位, 算法没变, key 也没变, 同一个数据, 修改 apk 路径和 so 路径的两个 unidbg 得到的结果一致.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06OiccicZpI1cibRo60N0UZtVicQicnbIfzKadbszDicMs4X8iaKnkoyV7CExjWmTTecCLGb7pGBBmzTGvRAA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=66)

另外就是响应解密那个部分, 有些接口, 返回的数据过大会进行 lz4 压缩在 rc4 加密, lz4 算法跟前面分析的差不多, 知道压缩就能逆推解压的算法.

0x8 - 结语:

感谢各位读者的收听, 如文章有分析不妥或错误的地方, 欢迎各位师傅斧正, 笔者本着是想一口气写完三个样本的分析, 但是考虑到篇幅可能过长, 所以循序渐进的来讲解案例, 逐帧学习.

下期精彩:

libpdd_secure.so 算法分析

libjdgs.so 算法分析