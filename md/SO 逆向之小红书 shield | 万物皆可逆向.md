> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [onejane.github.io](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/#group-77)

> SO 逆向之小红书 shield

安装 charles 证书

安卓 8

plain

```
cd /data/misc/user/0/cacerts-added/
mount -o remount,rw /
mount -o remount,rw /system
chmod 777 *
cp * /etc/security/cacerts/
mount -o remount,ro /
mount -o remount,ro /system
```

避免 SSL Pinning 报错网络错误，也可以通过 frida 过掉 SSLPinning

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224214105777.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224214105777.png)

[image-20220224214105777](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224214105777.png)

jadx 搜索 shield，没什么实际结果，大概率不在 java 层，[frida_hook_libart](https://github.com/lasting-yang/frida_hook_libart) 对 libart 进行 hook，其中 **hook_RegisterNatives.js** 动态注册 jni 函数，**hook_art.js** 对常用 Native 方法 hook

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224215953511.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224215953511.png)

[image-20220224215953511](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224215953511.png)

plain

```
./fs14216arm64 
pyenv local 3.8.2
frida --version
   14.2.16
frida -UF -l hook_art.js -o xhs.log
```

由于在 **NewStringUTF** 函数中处理了这个 shield 签名，添加定位地址，修改 hook_art.js

plain

```
if (addrNewStringUTF != null) {
    Interceptor.attach(addrNewStringUTF, {
        onEnter: function (args) {
            if (args[1] != null) {
                var module = Process.findModuleByAddress(this.returnAddress);
                if (module != null && module.name.indexOf(so_name) == 0) {
                    var string = Memory.readCString(args[1]);
                    // console.log("[NewStringUTF] bytes:" + string, DebugSymbol.fromAddress(this.returnAddress));
                    //////xhs////////
                    if (string.length > 50) {
                        console.log("[NewStringUTF] bytes:" + string);
                        console.log("xhs---------", Thread.backtrace(this.context, Backtracer.FUZZY)
                            .map(DebugSymbol.fromAddress).join("\n"))
                    }
                    //////xhs////////
                }
            }
        },
        onLeave: function (retval) { }
    });

}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224214326592.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224214326592.png)

[image-20220224214326592](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224214326592.png)

IDA 打开 libshield.so,G 跳转到 0x93fa8 地址，看到该位置处于函数 **sub_939D8**

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224221418951.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224221418951.png)

[image-20220224221418951](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224221418951.png)

F5 反汇编，X 查看 **sub_939D8** 方法的调用处

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224221553187.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224221553187.png)

[image-20220224221553187](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224221553187.png)

可知在 Java 层的 **intercept** 方法，参数 **(Lokhttp3/Interceptor”,0x24,”Chain;J)Lokhttp3/Response;**

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224221725263.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224221725263.png)

[image-20220224221725263](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224221725263.png)

搜索 **intercept(**

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224222142075.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224222142075.png)

[image-20220224222142075](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224222142075.png)

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224222423752.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224222423752.png)

[image-20220224222423752](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224222423752.png)

开始 hook

plain

```
// 干掉 Android SSL Pinning
var array_list = Java.use("java.util.ArrayList");
var ApiClient = Java.use('com.android.org.conscrypt.TrustManagerImpl');
ApiClient.checkTrustedRecursive.implementation = function(a1, a2, a3, a4, a5, a6) { 
    var k = array_list.$new();
	return k; 
}

// shield
var shieldCls = Java.use("com.xingin.shield.http.XhsHttpInterceptor");
shieldCls.intercept.overload('okhttp3.Interceptor$Chain', 'long').implementation = function(chain,j){
	var result = this.intercept(chain,j);
	var request = chain.request();
	console.log(request.toString());
	console.log(result.toString());
	return result;
}

// okhttp
var OkHttpClient = Java.use("okhttp3.OkHttpClient");
OkHttpClient.newCall.implementation = function (request) {
    var result = this.newCall(request);
    console.log(request.toString());
    var stack = threadinstance.currentThread().getStackTrace();
    console.log("http >>> Full call stack:" + Where(stack));
    return result;
};
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224222851531.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224222851531.png)

[image-20220224222851531](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220224222851531.png)

这也就意味着 http 的请求和 shield 的生成都在 so 层完成。

[](#Unidbg "Unidbg")Unidbg
--------------------------

打开 libshield.so，搜索 java，没有函数说明都是动态注册进入 JNI_OnLoad

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226104042957.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226104042957.png)

[image-20220226104042957](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226104042957.png)

plain

```
jint JNI_OnLoad(JavaVM *vm, void *reserved)
{
  return sub_1027C(vm);
}
```

### [](#搭建框架 "搭建框架")搭建框架

调用 callJNI_OnLoad

plain

```
public class XhsShield extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;
    private Headers headers;
    private Request request;
    public static String shield;
    private String commonParams;


    public XhsShield(){
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.xhs").build(); // 创建模拟器实例，要模拟32位或者64位，在这里区分
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析
        vm = emulator.createDalvikVM(new File("小红书v6.73.0.apk"));
        vm.setVerbose(true);
        vm.setJni(this);
        DalvikModule dm = vm.loadLibrary(new File("libshield.so"), true);
        module = dm.getModule();
        System.out.println("call JNIOnLoad");
        dm.callJNI_OnLoad(emulator);
    }
    public static void main(String[] args) {
        XhsShield test = new XhsShield();
    }
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226110431367.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226110431367.png)

[image-20220226110431367](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226110431367.png)

### [](#initializeNative "initializeNative")initializeNative

对 jni 调用 java 方法的一些类进行初始化操作

进入 sub_1027C 函数

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226104430970.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226104430970.png)

[image-20220226104430970](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226104430970.png)

发现函数有很多，在 IDA View-A 中 alt+t 搜索上文中的`com.xingin.shield.http.XhsHttpInterceptor`的`initializeNative`方法

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226105749125.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226105749125.png)

[image-20220226105749125](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226105749125.png)

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226105807738.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226105807738.png)

[image-20220226105807738](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226105807738.png)

> .text 程序代码段内存区域  
> .data 已初始化的全局数据、全局常量段内存区域  
> .rodata 资源数据段，#define 定义的常量  
> .bss 程序中未初始化的全局变量的内存区域

直接看. data 数据块中的汇编，initializeNative 地址 0x94288+1，intercept 地址 sub_939D8+1，initialize 地址 sub_937B0+1

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226111556828.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226111556828.png)

[image-20220226111556828](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226111556828.png)

g 跳转到 0x94288+1，查看 initializeNative 函数，n 重命名

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226150010437.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226150010437.png)

[image-20220226150010437](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226150010437.png)

在 XhsHttpInterceptor 类中函数执行顺序，initializeNative>initialize>intercept

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226112824885.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226112824885.png)

[image-20220226112824885](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226112824885.png)

plain

```
public void callinitializeNative(){
    List<Object> list = new ArrayList<>(10);
    list.add(vm.getJNIEnv()); // 第一个参数是env
    list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
    module.callFunction(emulator, 0x94288+1, list.toArray());
}
public static void main(String[] args) {
    XhsShield test = new XhsShield();
    test.callinitializeNative();
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226113553005.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226113553005.png)

[image-20220226113553005](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226113553005.png)

plain

```
@Override
public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
    switch (signature){
        case "java/nio/charset/Charset->defaultCharset()Ljava/nio/charset/Charset;":{
            return vm.resolveClass("java/nio/charset/Charset").newObject(Charset.defaultCharset());
        }
    }
    return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226113820208.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226113820208.png)

[image-20220226113820208](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226113820208.png)

通过 objection 内存搜索，是 PackageInfo 的一个实例变量，该类获取 AndroidManifest.xml 文件的信息

plain

```
frida-ps -Ua
objection -g com.xingin.xhs explore -P ~/.objection/plugins
plugin wallbreaker classdump --fullname android.content.pm.PackageInfo
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226115809498.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226115809498.png)

[image-20220226115809498](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226115809498.png)

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226120639081.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226120639081.png)

[image-20220226120639081](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226120639081.png)

也可以通过 jnitrace 获取 versionCode,`jnitrace -l libshield.so -m spawn com.xingin.xhs --ignore-vm > xhs.log`

> jnitrace 出现 app 闪退或者黑屏解决方案
> 
> 1.  更新版本 pip install jnitrace==v3.0.8 ，frida==12.8.0
>     
> 2.  修改 jnitrace-engine
>     
>     plain
>     
>     ```
>     Interceptor.attach(dlopenRef, {
>         onEnter:function(args){
>             this.path = args[0].readCString();
>         },onLeave:function(retval){
>             if (this.path !== null) {
>                 if (checkLibrary(this.path)) {
>                     trackedLibs.set(retval.toString(), true);
>                 }
>                 else {
>                     libBlacklist.set(retval.toString(), true);
>                 }
>             }
>         }
>     });
>     ```
>     
>     [![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220228190245812.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220228190245812.png)
>     
>     [image-20220228190245812](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220228190245812.png)
>     
>     python -m jnitrace.jnitrace -l libshield.so -b none com.xingin.xhs
>     

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226120720392.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226120720392.png)

[image-20220226120720392](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226120720392.png)

plain

```
@Override
public int getIntField(BaseVM vm, DvmObject<?> dvmObject, String signature) {
    switch (signature){
        case "android/content/pm/PackageInfo->versionCode:I":{
            return 6730157;
        }
    }
    return super.getIntField(vm, dvmObject, signature);
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226120944572.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226120944572.png)

[image-20220226120944572](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226120944572.png)

获取静态变量，frida 一步到位，`console.log(Java.use("com.xingin.shield.http.ContextHolder").sDeviceId.value)`

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121315768.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121315768.png)

[image-20220226121315768](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121315768.png)

plain

```
@Override
public DvmObject<?> getStaticObjectField(BaseVM vm, DvmClass dvmClass, String signature) {
    switch (signature){
        case "com/xingin/shield/http/ContextHolder->sDeviceId:Ljava/lang/String;":{
            return new StringObject(vm, "145b5374-b973-38d1-8299-eb98f9d950ce");
        }
    }
    return super.getStaticObjectField(vm, dvmClass, signature);
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121408631.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121408631.png)

[image-20220226121408631](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121408631.png)

同理 frida 一步到位，`console.log(Java.use("com.xingin.shield.http.ContextHolder").sAppId.value)`, 或者 jnitrace

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121616078.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121616078.png)

[image-20220226121616078](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121616078.png)

plain

```
@Override
public int getStaticIntField(BaseVM vm, DvmClass dvmClass, String signature) {
    switch (signature){
        case "com/xingin/shield/http/ContextHolder->sAppId:I":{
            return -319115519;
        }
    }
    return super.getStaticIntField(vm, dvmClass, signature);
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121711912.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121711912.png)

[image-20220226121711912](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121711912.png)

jnitrace 或者 frida 一步到位

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121744091.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121744091.png)

[image-20220226121744091](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121744091.png)

plain

```
@Override
public boolean getStaticBooleanField(BaseVM vm, DvmClass dvmClass, String signature) {
    switch (signature) {
        case "com/xingin/shield/http/ContextHolder->sExperiment:Z":
            return true;
    }
    return super.getStaticBooleanField(vm, dvmClass, signature);
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121902850.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121902850.png)

[image-20220226121902850](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226121902850.png)

### [](#initialize "initialize")initialize

plain

```
public XhsHttpInterceptor(String str, a<Request> aVar) {
    this.cPtr = initialize(str);
    this.predicate = aVar;
}
public static XhsHttpInterceptor newInstance(String str, a<Request> aVar) {
    return new XhsHttpInterceptor(str, aVar);
}
```

在 XhsHttpInterceptor 的 newInstance 查找用例找到 b 函数，直接传入的字符串为”main”

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226135417525.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226135417525.png)

[image-20220226135417525](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226135417525.png)

plain

```
// 第二个初始化函数 public native long initialize(String str);
public long callinitialize(){
    List<Object> list = new ArrayList<>(10);
    list.add(vm.getJNIEnv()); // 第一个参数是env
    list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
    list.add(vm.addLocalObject(new StringObject(vm, "main")));
    Number number = module.callFunction(emulator, 0x937B0+1, list.toArray())[0];
    return number.longValue();
}
public static void main(String[] args) {
    XhsShield test = new XhsShield();
    test.callinitializeNative();
    long ptr = test.callinitialize();
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226161022449.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226161022449.png)

[image-20220226161022449](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226161022449.png)

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226161050578.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226161050578.png)

[image-20220226161050578](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226161050578.png)

plain

```
@Override
public DvmObject<?> callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
    switch (signature) {
        case "java/nio/charset/Charset->defaultCharset()Ljava/nio/charset/Charset;": {
            return vm.resolveClass("java/nio/charset/Charset").newObject(Charset.defaultCharset());
        }
    }
    return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226141612842.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226141612842.png)

[image-20220226141612842](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226141612842.png)

**Context.getSharedPreferences(String name,int mode)** 获取一个 SharedPreferences 实例，name 文件名称，不需要加后缀. xml，一般这个文件存储在** /data/data//shared_prefs** 下，mode 指读写权限，从 jnitrace 中查看 jstring 为 0x11，jint 为 0，断点调试 jstring 就是”s”，DvmObject 对象是 Unidbg 抽象出来的一个 JNI 交互的对象，给他一个这样的对象都不会报错，这个值后面要用到所以不能 newObject(null)。

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226155632734.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226155632734.png)

[image-20220226155632734](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226155632734.png)

plain

```
@Override
public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
    switch (signature) {
        case "android/content/Context->getSharedPreferences(Ljava/lang/String;I)Landroid/content/SharedPreferences;":
            return vm.resolveClass("android/content/SharedPreferences").newObject(vaList.getObjectArg(0));
    }

    return super.callObjectMethodV(vm, dvmObject, signature, vaList);
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226195613330.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226195613330.png)

[image-20220226195613330](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226195613330.png)

plain

```
// ???为什么main返回空，main_hmac返回s.xml的内容
@Override
public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
    switch (signature) {
        // android.content.SharedPreferences android.content.Context.getSharedPreferences(java.lang.String, int)
        case "android/content/Context->getSharedPreferences(Ljava/lang/String;I)Landroid/content/SharedPreferences;":
            return vm.resolveClass("android/content/SharedPreferences").newObject(vaList.getObjectArg(0));
        case "android/content/SharedPreferences->getString(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;": {
            if(((StringObject) dvmObject.getValue()).getValue().equals("s")){
                System.out.println("getString :"+vaList.getObjectArg(0).getValue());
                if (vaList.getObjectArg(0).getValue().equals("main")) {
                    return new StringObject(vm, "");
                }
                if(vaList.getObjectArg(0).getValue().equals("main_hmac")){
                    return  new StringObject(vm, "hmLDH4NVbddu/qNZWZj80kqBVKWrexc+1w3zCF0FCNQ03x3a9o9RsHcP2e1LwpK0gPbC4nHeU9dU2d0hyOhPElbIBZlNMjj9HRCCNMmb0uRWywu1tq3IvjogrlLosRl5");
                }
            }
		}
    }
    return super.callObjectMethodV(vm, dvmObject, signature, vaList);
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202103259.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202103259.png)

[image-20220226202103259](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202103259.png)

### [](#intercept "intercept")intercept

该函数中完成了请求与响应，所以必然会出现 shield 的生成。

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226140519884.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226140519884.png)

[image-20220226140519884](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226140519884.png)

plain

```
// 目标函数 public Response intercept(Interceptor.Chain chain)
public void callintercept(long ptr){
    List<Object> list = new ArrayList<>(10);
    list.add(vm.getJNIEnv()); // 第一个参数是env
    list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
    DvmObject<?> chain = vm.resolveClass("okhttp3/Interceptor$Chain").newObject(null);
    list.add(vm.addLocalObject(chain));
    list.add(ptr);
    module.callFunction(emulator, 0x939D8 + 1, list.toArray());
}
// jadx中分析调用initialize得到的结果作为参数传入intercept函数中
public static void main(String[] args) {
    XhsShield test = new XhsShield();
    test.callinitializeNative();
    long ptr = test.callinitialize();
    test.callintercept(ptr);
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226200725718.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226200725718.png)

[image-20220226200725718](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226200725718.png)

plain

```
// ??? request哪里来
request = new Request.Builder()
        .url("https://edith.xiaohongshu.com/api/sns/v6/homefeed?oid=homefeed_recommend&cursor_score=&geo=eyJsYXRpdHVkZSI6MC4wMDAwMDAsImxvbmdpdHVkZSI6MC4wMDAwMDB9%0A&trace_id=3eb910d5-a037-3076-92d3-739e2d02e3e0¬e_index=0&refresh_type=1&client_volume=0.00&preview_ad=&loaded_ad=%7B%22ads_id_list%22%3A%5B%5D%7D&personalization=1&pin_note_id=&pin_note_source=&unread_begin_note_id=620cc5700000000021036137&unread_end_note_id=6203bff30000000021039a28&unread_note_count=6")
        .addHeader("xy-common-params", "fid=164554128510caadf802049b8d38a1ca58cbf294b8d2&device_fingerprint=20220222224814dd7db8705f0befea0b24b13732c811a70164bde096a4c7f6&device_fingerprint1=20220222224814dd7db8705f0befea0b24b13732c811a70164bde096a4c7f6&launch_id=1645664833&tz=Asia%2FShanghai&channel=YingYongBao&version)
        .build();
case "okhttp3/Interceptor$Chain->request()Lokhttp3/Request;": {
    return vm.resolveClass("okhttp3/Request").newObject(request);
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226201514098.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226201514098.png)

[image-20220226201514098](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226201514098.png)

plain

```
// ??? 为什么url为request的url
case "okhttp3/Request->url()Lokhttp3/HttpUrl;": {
    Request request = (Request) dvmObject.getValue();
    return vm.resolveClass("okhttp3/HttpUrl").newObject(request.url());
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202124581.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202124581.png)

[image-20220226202124581](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202124581.png)

plain

```
case "com/xingin/shield/http/Base64Helper->decode(Ljava/lang/String;)[B":{
    String input = (String) vaList.getObjectArg(0).getValue();
    byte[] result = Base64.decodeBase64(input);
    return new ByteArray(vm, result);
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202629238.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202629238.png)

[image-20220226202629238](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202629238.png)

plain

```
case "okhttp3/HttpUrl->encodedPath()Ljava/lang/String;": {
    HttpUrl httpUrl = (HttpUrl) dvmObject.getValue();
    return new StringObject(vm, httpUrl.encodedPath());
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202740592.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202740592.png)

[image-20220226202740592](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202740592.png)

plain

```
case "okhttp3/HttpUrl->encodedQuery()Ljava/lang/String;": {
    HttpUrl httpUrl = (HttpUrl) dvmObject.getValue();
    return new StringObject(vm, httpUrl.encodedQuery());
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202824235.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202824235.png)

[image-20220226202824235](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202824235.png)

plain

```
case "okhttp3/Request->body()Lokhttp3/RequestBody;": {
    Request request = (Request) dvmObject.getValue();
    return vm.resolveClass("okhttp3/RequestBody").newObject(request.body());
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202933455.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202933455.png)

[image-20220226202933455](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226202933455.png)

plain

```
case "okhttp3/Request->headers()Lokhttp3/Headers;": {
    Request request = (Request) dvmObject.getValue();
    return vm.resolveClass("okhttp3/Headers").newObject(request.headers());
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203025309.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203025309.png)

[image-20220226203025309](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203025309.png)

plain

```
@Override
public DvmObject<?> newObjectV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
    switch (signature){
        case "okio/Buffer-><init>()V":
            return dvmClass.newObject(new Buffer());
    }
    return super.newObjectV(vm, dvmClass, signature, vaList);
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203137691.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203137691.png)

[image-20220226203137691](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203137691.png)

plain

```
case "okio/Buffer->writeString(Ljava/lang/String;Ljava/nio/charset/Charset;)Lokio/Buffer;": {
    System.out.println("write to my buffer:"+vaList.getObjectArg(0).getValue());
    Buffer buffer = (Buffer) dvmObject.getValue();
    buffer.writeString(vaList.getObjectArg(0).getValue().toString(), (Charset) vaList.getObjectArg(1).getValue());
    return dvmObject;
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203345411.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203345411.png)

[image-20220226203345411](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203345411.png)

plain

```
case "okhttp3/Headers->size()I":
    Headers headers = (Headers) dvmObject.getValue();
    return headers.size();
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203445936.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203445936.png)

[image-20220226203445936](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203445936.png)

plain

```
case "okhttp3/Headers->name(I)Ljava/lang/String;": {
    Headers headers = (Headers) dvmObject.getValue();
    return new StringObject(vm, headers.name(vaList.getIntArg(0)));
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203601107.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203601107.png)

[image-20220226203601107](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203601107.png)

plain

```
case "okhttp3/Headers->value(I)Ljava/lang/String;": {
    Headers headers = (Headers) dvmObject.getValue();
    return new StringObject(vm, headers.value(vaList.getIntArg(0)));
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203637038.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203637038.png)

[image-20220226203637038](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203637038.png)

plain

```
case "okhttp3/RequestBody->writeTo(Lokio/BufferedSink;)V": {
    BufferedSink bufferedSink = (BufferedSink) vaList.getObjectArg(0).getValue();
    RequestBody requestBody = (RequestBody) dvmObject.getValue();
    if(requestBody != null){
        try {
            requestBody.writeTo(bufferedSink);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    return;
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203830675.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203830675.png)

[image-20220226203830675](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203830675.png)

plain

```
case "okio/Buffer->clone()Lokio/Buffer;": {
    Buffer buffer = (Buffer) dvmObject.getValue();
    return vm.resolveClass("okio/Buffer").newObject(buffer.clone());
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203908813.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203908813.png)

[image-20220226203908813](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226203908813.png)

plain

```
case "okio/Buffer->read([B)I":
    Buffer buffer = (Buffer) dvmObject.getValue();
    return buffer.read((byte[]) vaList.getObjectArg(0).getValue());
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204002902.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204002902.png)

[image-20220226204002902](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204002902.png)

plain

```
case "okhttp3/Request->newBuilder()Lokhttp3/Request$Builder;": {
    Request request = (Request) dvmObject.getValue();
    return vm.resolveClass("okhttp3/Request$Builder").newObject(request.newBuilder());
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204058455.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204058455.png)

[image-20220226204058455](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204058455.png)

plain

```
case "okhttp3/Request$Builder->header(Ljava/lang/String;Ljava/lang/String;)Lokhttp3/Request$Builder;": {
    Request.Builder builder = (Request.Builder) dvmObject.getValue();
    builder.header(vaList.getObjectArg(0).getValue().toString(), vaList.getObjectArg(1).getValue().toString());
    if("shield".equals(vaList.getObjectArg(0).getValue().toString())){
        shield = vaList.getObjectArg(1).getValue().toString();
    }
    return dvmObject;
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204204774.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204204774.png)

[image-20220226204204774](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204204774.png)

plain

```
case "okhttp3/Request$Builder->build()Lokhttp3/Request;": {
    Request.Builder builder = (Request.Builder) dvmObject.getValue();
    return vm.resolveClass("okhttp3/Request").newObject(builder.build());
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204316786.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204316786.png)

[image-20220226204316786](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204316786.png)

plain

```
case "okhttp3/Interceptor$Chain->proceed(Lokhttp3/Request;)Lokhttp3/Response;": {
    return vm.resolveClass("okhttp3/Response").newObject(null);
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204342442.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204342442.png)

[image-20220226204342442](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204342442.png)

plain

```
case "okhttp3/Response->code()I":
    return 200;
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204517704.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204517704.png)

[image-20220226204517704](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204517704.png)

[](#Unidbg-server "Unidbg-server")[Unidbg-server](https://github.com/cxapython/unidbg-server.git)
-------------------------------------------------------------------------------------------------

经过一番补环境，终于拿到了 shield，保存在静态变量中，引入 unidbg-server

plain

```
@RequestMapping(value = "xhs", method = {RequestMethod.GET, RequestMethod.POST})
public String xhsShield() {
    String url = "https://edith.xiaohongshu.com/api/sns/v6/homefeed?oid=homefeed_recommend&cursor_score=&geo=eyJsYXRpdHVkZSI6MC4wMDAwMDAsImxvbmdpdHVkZSI6MC4wMDAwMDB9%0A&trace_id=3eb910d5-a037-3076-92d3-739e2d02e3e0¬e_index=0&refresh_type=1&client_volume=0.00&preview_ad=&loaded_ad=%7B%22ads_id_list%22%3A%5B%5D%7D&personalization=1&pin_note_id=&pin_note_source=&unread_begin_note_id=620cc5700000000021036137&unread_end_note_id=6203bff30000000021039a28&unread_note_count=6";
    String commonParams = "fid=164554128510caadf802049b8d38a1ca58cbf294b8d2&device_fingerprint=20220222224814dd7db8705f0befea0b24b13732c811a70164bde096a4c7f6&device_fingerprint1=20220222224814dd7db8705f0befea0b24b13732c811a70164bde096a4c7f6&launch_id=1645664833&tz=Asia%2FShanghai&channel=YingYongBao&version;
    String platformInfo = "platform=android&build=6730157&deviceId=145b5374-b973-38d1-8299-eb98f9d950ce";
    XhsShield xhs = new XhsShield();
    String shield = xhs.getShield();
    Map<String, String> headMap = new HashMap<>();
    headMap.put("xy-common-params",commonParams);
    headMap.put("shield",shield);
    headMap.put("xy-platform-info",platformInfo);
    return httpGet(url,null, headMap);
}
```

[![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204843128.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204843128.png)

[image-20220226204843128](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226204843128.png)

> objection hook 构造函数的方法
> 
> 1.9.6 版本以前 / root/.pyenv/versions/3.8.0/lib/python3.8/site-packages/objection/agent.js 的 9211 行
> 
> [![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226173411982.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226173411982.png)
> 
> [image-20220226173411982](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226173411982.png)
> 
> 1.9.6 版本以后 / root/.pyenv/versions/3.8.0/lib/python3.8/site-packages/objection/agent.js 的 20238 行
> 
> [![](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226173319019.png)](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226173319019.png)
> 
> [image-20220226173319019](https://onejane.github.io/2022/02/24/SO%E9%80%86%E5%90%91%E4%B9%8B%E5%B0%8F%E7%BA%A2%E4%B9%A6shield/image-20220226173319019.png)

版权声明: 本博客所有文章除特别声明外，均采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可协议。转载请注明来自 [万物皆可逆向](http://onejane.github.io/)！