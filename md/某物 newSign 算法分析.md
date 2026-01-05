> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/vbKEjxipoB5-Y_mCbUmADQ)

0x1. 抓包分析:

通过抓包读者不难发现, 很多的接口查询参数或表单中都有带上一个叫 newSign 这样的参数, 这也是本篇的目标.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSltgy6JticicnXNfjibvkn1psm5XlfUnbTQ5RSXial9KGepT0XcrUVzz7kA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=0)

0x2. 初步分析 newSign:

```
acb1bbfb8e99de6f70eceb9f87564196
```

读者也不难看出, newSign 是 32 位的十六进制, 那么说到这里, 读者也能够联想到一个老生常谈的算法 --MD5 算法 / Hmac 算法. 那么我们要如何去验证我们的猜想呢? 笔者这里的思路是确定参数算法逻辑是在哪一个层面进行的, 对于 Android 而言, 逆向分析无非就是两个层面 Java 层和 native 层. 确定的思路也很简单, 用算法助手开启算法自吐, 如果在日志中能够找到参数, 那么就是在 Java 层的, 否则就是在 native 层的, 同时也可以去 hook 一个 jni 方法 NewStringUTF, 不过如果样本调用 NewStringUTF 过于频繁, 用 fridahook 反而会导致 App 崩溃, 建议是加一些限制条件, 比如在此场景下可以对长度做判断等于 32 才打印堆栈, 当然啊, 笔者这里的观点不一定适用于所有的样本, 有时候不能盲目下结论, 需要去做验证.

0x3. 算法助手自吐:

开启自吐

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwS8RY0ZibaxrW201AZ1jy2G4M4mPmsXVcWG8F3cBf0PX9JjuDzoEllfYQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=1)

在日志中搜索 newSign 发现搜索到了一个结果

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSdfSL3IjQvaiaNlc44bbGpKfodp5TuicZKxqddt4cd7bXXCa9BGPUEvVQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=2)

进入日志, 发现签名的数据是密文

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwS2jzKRqa54AwFU86s8Xa4ldibCXAJPqAlYYtOCCHuiaZHIvmqHFw5y9CA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=3)

读者也不难看出, 这是进行 base64 编码过的数据, 尝试解码发现是乱码, 那么一定做了加密操作, 最后再对原始数据进行编码.

在日志中搜索密文数据, 发现搜索不到结果, 那我们有理由怀疑是不是在 native 层进行的加密.

0x4. 探寻密文的生成:

笔者这里有两个思路, 一个是跟调用堆栈找到 MD5 加密的位置, 分析前后上下文, 找到密文的生成函数, 另一个是直接 hook NewStringUTF 函数看看有没有密文的痕迹.

我选择的是第二者, hook NewStringUTF 脚本如下:

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
                if (dataString.includes("+")) {
                    console.log(dataString);
                    //打印堆栈
                    console.log(Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n') + '\n');
                    console.log(Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()));
                }
              
            }
        });
    }
}
hook_newStr();
```

发现确实有密文的痕迹, 日志如下:

```
dWWoXlbR3K87j2N27Dkv4uOPUnOsh0xrJ5t+atsiQXC+/dIN8Kof9Gm2x1kil7S/ZocOYcJB3TSnGOtLBnBBuEXG0NpaRmZLUgBdMqk7BJ8dEQTRl/XH3g7OkPkBuKaXGHbGD8SiwJ1RBXtXsjSKW+3CGRxpYShEOqtR3LOJkWP7vPPEbf0x7mdDs9WNQuXlz+kPY2Jvb1L1aXuL3kKJKzdmwZvfDSPrhXVR0dDNe321I9SalgZYrrDbmX9xxCAGTa/b6ocFLIySWTom6kb9qQ==
0x7269bcc85c libdewuhelper.so!encode+0x138
0x727f10ed3c base.odex!0x6b5d3c
0x727f10ed3c base.odex!0x6b5d3c

java.lang.Throwable
        at com.shizhuang.duapp.common.helper.ee.DuHelper.encodeByte(Native Method)
        at lte.NCall.IL(Native Method)
        at lte.NCall.IL(Native Method)
        at com.shizhuang.duapp.common.helper.ee.DuHelper.doWork(Unknown Source:18)
        at lte.NCall.IL(Native Method)
        at lte.NCall.IL(Native Method)
        at com.shizhuang.duapp.common.utils.RequestUtils.e(Unknown Source:28)
        at com.shizhuang.duapp.common.utils.RequestUtils.e(Native Method)
        at lte.NCall.IL(Native Method)
        at lte.NCall.IL(Native Method)
        at com.shizhuang.duapp.common.helper.net.interceptor.HttpRequestInterceptor.intercept(Unknown Source:18)
        at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:10)
        at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:1)
        at w9.b.intercept(MergeHostAfterInterceptor.java:130)
        at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:10)
        at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:1)
        at w9.d.intercept(MergeHostInterceptor.java:79)
        at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:10)
        at okhttp3.internal.http.RealInterceptorChain.proceed(RealInterceptorChain.java:1)
        at okhttp3.RealCall.getResponseWithInterceptorChain(RealCall.java:121)
        at okhttp3.RealCall.execute(RealCall.java:31)
        at retrofit2.OkHttpCall.execute(OkHttpCall.java:62)
        at retrofit2.adapter.rxjava2.CallExecuteObservable.subscribeActual(CallExecuteObservable.java:24)
        at jx0.m.subscribe(Observable.java:7)
        at retrofit2.adapter.rxjava2.BodyObservable.subscribeActual(BodyObservable.java:8)
        at jx0.m.subscribe(Observable.java:7)
        at io.reactivex.internal.operators.observable.k1.subscribeActual(ObservableMap.java:10)
        at jx0.m.subscribe(Observable.java:7)
        at io.reactivex.internal.operators.observable.ObservableRetryWhen$RepeatWhenObserver.subscribeNext(ObservableRetryWhen.java:25)
        at io.reactivex.internal.operators.observable.ObservableRetryWhen.subscribeActual(ObservableRetryWhen.java:34)
        at jx0.m.subscribe(Observable.java:7)
        at io.reactivex.internal.operators.observable.ObservableRetryBiPredicate$RetryBiObserver.subscribeNext(ObservableRetryBiPredicate.java:19)
        at io.reactivex.internal.operators.observable.ObservableRetryBiPredicate.subscribeActual(ObservableRetryBiPredicate.java:18)
        at jx0.m.subscribe(Observable.java:7)
        at io.reactivex.internal.operators.observable.ObservableSubscribeOn$a.run(ObservableSubscribeOn.java:7)
        at io.reactivex.internal.schedulers.ScheduledDirectTask.call(ScheduledDirectTask.java:3)
        at io.reactivex.internal.schedulers.ScheduledDirectTask.call(ScheduledDirectTask.java:1)
        at java.util.concurrent.FutureTask.run(FutureTask.java:266)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1167)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:641)
        at java.lang.Thread.run(Thread.java:919)
```

生成逻辑在 libdewuhelper.so 中, 对应的入口方法为

```
com.shizhuang.duapp.common.helper.ee.DuHelper.encodeByte
```

那就 hook 一下这个方法, 看看入参以及返回值.

笔者这里使用的是算法助手进行 hook(基于 xposedhook), 个人觉得比较方便, 读者也可以随其所好, fridahook 也行, xposedhook 也行.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSeqGqia1Jic36fGuacicWicXJicwWfPibFp1AWB4EIIrOrAwJ5zibGibKJm0Dicw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=4)

在 mt 管理器中找到此方法, 复制方法签名粘贴到算法助手即可, 前面的文章有讲过, 这里不做过多的赘述.

重启 app, 算法助手的 hook 才能生效.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSQwlZf3eLiatRZxEfSBTuT7VFAdziaibm50ckhNmRI6xvBs49IqPKEt7GA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=5)

传了两个参数, 一个是一个字节数组类型的, 算法助手贴心的为我们进行了 base64 编码, 第二个是 0101 组成的字符串, 暂时不知道是啥.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSCIbvXiavianB66qtIdfibdicGF94SZlTtoMib7RG0MiaJq9954XibEW1RwrNw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=6)

解码一下第一个参数, 发现是和查询参数相关的东西做了字典排序拼接, keyvaluekey1value1 这种拼接形式.

0x5.Unidbg 模拟执行还原算法.

Unidbg 项目 github:

```
https://github.com/zhkl0228/unidbg
```

读者可自行 download 下来, 笔者用的 unidbg-0.9.8

同时需要用到 idea 集成开发环境以及配置 maven 环境, 配置 maven 环境可以看我写的语雀笔记  

```
https://www.yuque.com/g/nangongxiaotian/kn7n44/ddr6nzn2p89ill7w/collaborator/join?token=e0DdizbRD4xrCbar&source=doc_collaborator# 《Unidbg环境配置记录》
```

这里不做过多的赘述了.

首先, 搭好架子并调用 encodeByte 方法, 代码如下:

```
package com.DeWu;import com.github.unidbg.AndroidEmulator;import com.github.unidbg.Module;import com.github.unidbg.linux.android.AndroidEmulatorBuilder;import com.github.unidbg.linux.android.AndroidResolver;import com.github.unidbg.linux.android.dvm.*;import com.github.unidbg.linux.android.dvm.array.ArrayObject;import com.github.unidbg.linux.android.dvm.array.ByteArray;import com.github.unidbg.linux.android.dvm.wrapper.DvmInteger;import com.github.unidbg.memory.Memory;import java.io.File;public class DwSec extends AbstractJni {    private final AndroidEmulator emulator;    private final VM vm;    private final Module module;    private final DvmClass DuHelper;    public DwSec(){        //创建模拟器实例 (ARM32架构)        emulator = AndroidEmulatorBuilder.for64Bit()  //根据so的架构改                .setProcessName("com.shizhuang.duapp")//根据app包名改                .build();        //获取内存接口        final Memory memory = emulator.getMemory();        //Android 6.0        memory.setLibraryResolver(new AndroidResolver(23));        //创建Android虚拟机        //可选填 new File("apk路径")        vm = emulator.createDalvikVM(new File("unidbg-android/src/test/resources/apks/得物_v5.73.5.apk"));        //设置JNI接口        vm.setJni(this);        //是否打印详细日志        vm.setVerbose(true);        //加载目标SO库        DalvikModule dm = vm.loadLibrary(new File("unidbg-android/src/test/resources/example_binaries/arm64-v8a/libdewuhelper.so"), true);  // 替换为实际SO路径        //调用SO初始化函数        dm.callJNI_OnLoad(emulator);        module = dm.getModule();        //获取目标类        DuHelper = vm.resolveClass("com/shizhuang/duapp/common/helper/ee/DuHelper");    }    public String encodeByte(byte[] bArr,String str){        //下断点        //emulator.attach().addBreakPoint(module,0x2EC4);        //emulator.attach().addBreakPoint(module,0x160C);        emulator.attach().addBreakPoint(module,0x382C);        //调用native方法        DvmObject<?> result = DuHelper.callStaticJniMethodObject(                emulator,                "encodeByte([BLjava/lang/String;)Ljava/lang/String;",                new ByteArray(vm,bArr),                new StringObject(vm,str)        );        //获取并返回结果        return (String) result.getValue();    }    public static void main(String[] args) {        DwSec dwSec = new DwSec();        String newSign = dwSec.encodeByte(                "cipherParamuserNamecountryCode86loginTokenpassword5305d2b87fadce24258ff955a25e06aeplatformandroidtimestamp1759927200108typepwduserNameeb2157d58586e8c59eb06c27be046dad_1uuidb9c3255d079dbcc2v5.73.5".getBytes(),                "010110100010001010010010000011000111001011101010101000101110111010011010101101101010001000101100010110100010001010011010110011001111001011100010101000100100110010110010100010101011110010111100"        );        System.out.println("NewSign ======>"+newSign);    }}
```

值得高兴的是, 这个样本不需要环境就能跑起来生成正确的值.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSiae9BOibiaYN2BibYBAgXib7QQf2WDoxicOfbur7B7eDHp0FGbDmnS5EvkLw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=7)

接下来, 我们需要 unidbg 的下断点动态调试配合 ida 的反汇编的伪 C 静态分析还原算法.

将 app 的 so 通过 adbpull 到本地拖到 ida 反汇编时, ida 貌似出了点岔子

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSUyGZI0UTO4xkLa8nEWoUicff0KHtsqHKIqk8UzeafK5W7yuqkKz3j3g/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=8)

遇到这种情况, 多半是 so 做了加固处理, 磁盘的 so 与内存的 so 不同, 磁盘的 so 是加密的, ida 也无法识别出来, 竟然是加密的, 为了保证能够正常运行, 肯定会在内存中进行解密, 所以我们只需要将解密好的 so 保存下来进行修复即可让 ida 重新识别出来.

so 脱壳的脚本如下, 此脚本来自于 GitHub 某位大佬的开源.

```
function dump_so(so_name) {
    Java.perform(function () {
        var currentApplication = Java.use("android.app.ActivityThread").currentApplication();
        var dir = currentApplication.getApplicationContext().getFilesDir().getPath();
        var libso = Process.getModuleByName(so_name);
        console.log("[name]:", libso.name);
        console.log("[base]:", libso.base);
        console.log("[size]:", ptr(libso.size));
        console.log("[path]:", libso.path);
        var file_path = dir + "/" + libso.name + "_" + libso.base + "_" + ptr(libso.size) + ".so";
        var file_handle = new File(file_path, "wb");
        if (file_handle && file_handle != null) {
            Memory.protect(ptr(libso.base), libso.size, 'rwx');
            var libso_buffer = ptr(libso.base).readByteArray(libso.size);
            file_handle.write(libso_buffer);
            file_handle.flush();
            file_handle.close();
            console.log("[dump]:", file_path);
        }
    });
}
dump_so("libdewuhelper.so");
```

将 dump 下来的 so 通过 adb pull 到本地需要对 so 进行修复才能让 ida 正常识别. 这里使用的 so 修复工具是 sofixer

```
https://github.com/F8LEFT/SoFixer/releases
```

读者可自行下载对应系统的, 需要注意 so 是多少位的就用多少位的可执行程序去执行. 这里的 so 是 64 位的, 笔者是 Windows 系统, 那么就得选择 windows-64 的

修复命令如下

```
E:\Microsoft_Download\SoFixer.exe -s libdewuhelper.so_0x7269529000_0x7000.so -o libdewuhelper_fix.so -m 0x7269529000 -d
```

-m 后面的参数是 so 的基地址

也就是脱壳输出中的 base 这一项

```
[base]: 0x7269529000
```

```
[name]: libdewuhelper.so
[base]: 0x7269529000
[size]: 0x7000
[path]: /data/app/com.shizhuang.duapp-yhu5KhY-aaeA9feBslKT7A==/lib/arm64/libdewuhelper.so
[dump]: /data/user/0/com.shizhuang.duapp/files/libdewuhelper.so_0x7269529000_0x7000.so
```

修复好的 so 拖入 ida 当中就能正确的识别了.

这个样本也是对 so 没有做混淆的处理, 符号信息也是比较全的.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwS8uibFWLSATo3PSduULPJamzACcUWlbGYlgN0V4hia5HWGTTZvq7ndjyg/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=9)

能够发现用到了 aes 和 base64 这两种算法, 这里需要去验证一下.

在 unidbg 的日志当中, 读者不难发现, 有一条这样的日志, 调用 newstringutf 方法将 c 的 str 转成 Java 的 str, 这个也是我们的切入点.

```
JNIEnv->NewStringUTF("dWWoXlbR3K87j2N27Dkv4uOPUnOsh0xrJ5t+atsiQXC+/dIN8Kof9Gm2x1kil7S/PVV3j/r92qWWfiz5AloLqW7WI7w0QmjxB19MuQbYlAbE2Hm4JmzduzRBf6d/RfyIGHbGD8SiwJ1RBXtXsjSKWzHeceZPLF5WuCJnpLiJaWeu6xS3PtnA2yK6hedNfzb0bfUfAvDy60pqsMmUQMbjsQskdFQ+zM87dE9fV1dx7x+1I9SalgZYrrDbmX9xxCAGTa/b6ocFLIySWTom6kb9qQ==") was called from RX@0x4000185c[libdewuhelper.so]0x185c
```

他在 0x185c 偏移处, ida 按下键盘的 g 键跳转地址, 按 tab/f5 转成伪代码

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwS6giaTfhGxb5UYxF9F3FrWrtibicHNfGuS02GGXyCxiaFwjsHxa8aEicicUdA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=10)

NewStringUTF 的第二个参数是我们重点关注的对象, 第一个是 env 的指针不需要重点关注.

v18 来自于上面的方法, 我们可以跟进这个方法

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSfJm4pBdBGSjB2ibLicYNchARGiciartZ9R0LvuIdAU6YrMxHX2ic0cv3oUA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=11)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSQjdKvf4zh9ictpHsAXxbiazQYv99ic0Bib32Jlb2ADo7Ar0HKcPG9tdkgQ/640?wx_fmt=png&from=appmsg#imgIndex=12)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSicgow75I3KWCwSFia01AQUDRicrJSmIGnEg4WtKHVVLLDOQ1Nf3RbSd0Q/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=13)

然后跳转到这里, 按下 f5 转成伪 C 代码

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSPvouOO2ibNJ2amczgLERyUyx2bAGsvdKQaNhD3J7LUbKG8EZhe7FhZQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=14)

读者可以对这个函数下断点, 看看入参是什么.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSbS9CZNVGX2sTk7WYHEBGdX4Fkr9lNE2yZCjvGOd0DRwojKGW8EEwgQ/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=15)

拿到此函数的偏移, 进行下断.

```
void Debugger(){
    //下断点    
    emulator.attach().addBreakPoint(module,0x2EC4);
}
```

```
public static void main(String[] args) {
    DwSec dwSec = new DwSec();
    dwSec.Debugger();
    String newSign = dwSec.encodeByte(            
      "cipherParamuserNamecountryCode86loginTokenpassword5305d2b87fadce24258ff955a25e06aeplatformandroidtimestamp1759927200108typepwduserNameeb2157d58586e8c59eb06c27be046dad_1uuidb9c3255d079dbcc2v5.73.5".getBytes(),
      "010110100010001010010010000011000111001011101010101000101110111010011010101101101010001000101100010110100010001010011010110011001111001011100010101000100100110010110010100010101011110010111100"    
      );
    System.out.println("NewSign ======>"+newSign);
}
```

在执行之前下断点, 封装成一个方法方便管理.

运行, 成功断下来.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSibKvVfdbO2MndfQlGCUibc2EAhcWhr2GALbP3beJnKcSMf22UHpwwdsA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=16)

这里我们如何查看参数呢, 根据调用 arm 汇编约定, 函数的参数会被存在 x0-x7(64 位)/r0-r3 寄存器中, 多余的参数通过栈传递.

可以通过此命令查看函数参数

```
mx0
```

```
mx1
```

在 unidbg 的控制台中输入命令, m 后面可以跟寄存器名称 (x0-x7/r0-r4), 也可以是内存地址.

mx0

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSibJu64s39jib37DlpFjKpcJEm92rekRQfWibibkCrm889icNHPJFX7W3V5Q/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=17)

mx1

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwS26lqANicicWkV2PiaHnNttRBO9fcukmpSdAKHUeeysfnYMU6mBL3JCXFA/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=18)

发现第一个参数是我们的明文, 第二个参数 16 位猜测是 key/iv 相关的, 结合反汇编的函数名称, 应该与 AES_128_ECB_PKCS5Padding_Encrypt 有关系.

可以 aes 加密看看结果是否一致.

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSrZJJf08IHE6CdjrFaQdkJtbGU9yFJ27zYDXSma10aeMv40PiaMCanHw/640?wx_fmt=png&from=appmsg&watermark=1#imgIndex=19)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/e8LMyuEM06NoxPFaH7hiao8cKdt0q7ibwSFQPqFn9F0vJ6eADy56DhTXicC0b9Kxcek0PN3jXce94cHGQicVSpBHIA/640?wx_fmt=png&from=appmsg#imgIndex=20)

发现与结果是一致的, 那么这就是 aes-ecb 加密的了, 现在唯一要解决的是, 这个 key 是如何来的, 经过我的测试, 他和第二个参数有关, 并且第二个参数是固定的, 那么也就意味着 key 也是固定的, 至此全篇分析结束.

感谢各位读者的收听.

下期精彩预告:

libyzwg.so 算法分析

libpdd_secure.so 算法分析

libjdgs.so 算法分析