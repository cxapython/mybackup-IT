> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-260540.htm)

> [原创] 使用 Frida 简单实现函数粒度脱壳

前言  

=====

    本题来自于 2W 班的第一题，完成某 APP 的脱壳。题目中的 APP 实现了自定义 ClassLoader 导致默认版本 Fart 无法正常脱壳，需要自己定制。这里尝试使用 Frida 进行脱壳，脚本完全模仿默认版本的 Fart 运行流程进行编写，当然很多函数 Frida 完全没修改源码来的直接方便。这里也是为了熟悉下 Frida 所以进行的尝试，很多函数也都是直接用 Frida 编码实现，比如解析 Dex 中类，计算 Dex 函数代码长度等，相对于 hanbing 老师的 Frida 脱壳麻烦很多。水平太差只能用笨方法了。

    实现过程中，加深以下几点知识点的理解

    1. 了解 Fart，尝试解决一些自定义问题  

    2.Frida 遍历 Dex 类，类方法，类函数

    3.Frida 主动调用指定函数

    4. 自定义 ClassLoader 对脱壳的影响

解题步骤  

**一、解题思路**

    首先直接使用 fart 是肯定不行了，就不重复写了。脱下来的 Dex 大多都是抽取的，除了一些被动调用的函数能顺便 Dump 下来。由于编译源码比较麻烦，所以这里使用 Frida 脚本来实现。

**二、查看一些被动调用还原的代码**

    用的 yang 大佬的 [dump 脚本](https://github.com/lasting-yang/frida_dump/blob/master/dump_dex.js)，Dump 下 Dex 后，发现是自定义 ClassLoader 导致 Fart 无法正常运行。  

![](https://bbs.pediy.com/upload/attach/202005/571058_PWN7KUC8PU3HYHK.jpg)

**三、编写 Frida 脱壳脚本**

   需要解决的问题：

        Frida 遍历 ClassLoader, 类 ，类函数，并依次调用

        Hook 函数运行流程中某一处，获取当时 dex 中函数的代码并保存

**四、遍历类并遍历函数调用**

    1. 枚举 ClassLoader 类代码

```
function hook_java(){
    Java.perform(function(){
        console.log("---------------Java.enumerateClassLoaders");
        Java.enumerateClassLoaders({
            onMatch: function(cl){
                fartwithClassloader(cl);
            },
            onComplete: function(){
            }
        });
    });
}
 
function fartwithClassloader(cl){
    Java.perform(function(){    
        var clstr = cl.$className.toString();
        if(clstr.indexOf("java.lang.BootClassLoader") >= 0 ){
            return
        }
        console.log("  |------------",cl.$className);
 
        var class_BaseDexClassLoader = Java.use("dalvik.system.BaseDexClassLoader");
        var pathcl = Java.cast(cl, class_BaseDexClassLoader);
        console.log(".pathList",pathcl.pathList.value);
 
        var class_DexPathList = Java.use("dalvik.system.DexPathList");
        var dexPathList = Java.cast(pathcl.pathList.value, class_DexPathList);
        console.log(".dexElements:",dexPathList.dexElements.value.length);
 
        var class_DexFile = Java.use("dalvik.system.DexFile");
        var class_DexPathList_Element = Java.use("dalvik.system.DexPathList$Element");
        for(var i=0;i -1){
                        loadClassAndInvoke(cl, classNameArr[i]);
                    }
                }
                console.log("     |------------Enumerate ClassName End");
            }
        }
    });
} 
```

    根据获取 ClassLoader 继承链，可以找到 dalvik.system.DexPathList$Element 类，根据此类即可获取 dexFile 字段枚举所有类。此处主要是 Java.cast 的使用，具体参考 ClassLoader 的源码。

    2. 获取类函数

```
var classResult = Java.use(className).class;
if(!classResult) return;
var methodArr = classResult.getDeclaredConstructors();
methodArr = methodArr.concat(classResult.getDeclaredMethods());
```

很容易就可以获取构造函数和普通函数列表

    3. 调用类函数  

    实现了 2 种方法，第一种通过 Java 层 java.lang.reflect.Method 的函数 public native Object invoke(Object obj, Object... args)

```
var argsTypes = methodArr[i].getParameterTypes();
var args = []
// int类型
var class_int = Java.use("java.lang.Integer");
args[0] = class_int.$new(0x1);
 
// String类型
var class_String = Java.use("java.lang.String");
args[0] = class_String.$new("TEST");
 
// 例:android.os.Bundle类型，OnCreate
var class_Bundle = Java.use("android.os.Bundle");
args[0] = class_Bundle.$new();
// 参数列表
var arr = Java.array("Ljava.lang.Object;",args);
methodArr[i].setAccessible(true)
console.log("invoke result:",methodArr[i].invoke(null,arr));
 
// 非静态需要传第一个参数
// var class_MainActivity = Java.use("com.aipao.hanmoveschool.activity.MainActivity");
// class_MainActivity.$new();
// Java.choose("com.aipao.hanmoveschool.activity.MainActivity",{
//     onMatch: function(ins){
//         try {
//             console.log(methodArr[i].invoke(ins,arr)); //.overload('java.lang.Object', '[Ljava.lang.Object;')
//         } catch (error) {
//             console.log("Java.choose:[",methodArr[i].toString(),']',error);
//         }
//     },
//     onComplete: function(){
//     }
// });
```

    这种调用方式非常繁琐，每个类型都要创建对应类的对象，如果是构造参数不是空的就麻烦死了。

    好处就是如果参数正常可以保证函数正常运行。  

    最初的时候就是想像 fart 一样直接调用 ArtMethod::Invoke，但是当时很多参数不知道怎么传送。

    后面是第一种方式太复杂，很多函数基本上无法调用，所以找到了第二种方式。代码如下

```
var invokeSize = Memory.alloc(0x10).writeU32(6);
var invokeStr = Memory.alloc(0x100).writeUtf8String("fart");
var allocPrettyMethod = Memory.alloc(0x100);
var allocPrettyMethodInit = []
ArtMethod_invoke_replace(ptr(methodArr[i].getArtMethod()), ptr(0), ptr(0), 6, invokeSize, invokeStr);
```

    直接使用函数 getArtMethod() 获取到 ArtMethod 的指针。这里虽然在 ArtMethod::invoke 运行时会报错，但是可以进入到 invoke 方法，获取当时的函数代码  

**五、HOOK art_method.cc 文件中的 ArtMethod::Invoke，根据参数 Dump 函数**

    1.Hook 代码使用 lasting-yang 大佬的[代码](https://github.com/lasting-yang/frida_hook_libart/blob/master/hook_artmethod.js)，主要是使用 PrettyMethod 打印出函数名，好做个过滤

    2. 具体 DumpCode 的代码会有一些 BUG, 只解决影响 Dump 的，也有些还没解决的就跳过 Dump 

```
var dex_code_item_offset_ = args[0].add(sizeU32*2).readU32();
var dex_method_index_ = args[0].add(sizeU32*3).readU32();
if(dex_code_item_offset_ <= 0){
    //com.aipao.hanmoveschool.activity.StepDetector$OnSensorChangeList
    console.log("dex_code_item_offset_ error:",dex_code_item_offset_);
    return;
}
// console.log("dex_code_item_offset_:",dex_code_item_offset_.toString
// console.log("dex_method_index_:",dex_method_index_.toString(16));
if(DexBase){
    var addrCodeOffset = DexBase.add(dex_code_item_offset_);
    // console.log("addrCodeOffset:",hexdump(addrCodeOffset));
    var tries_size = addrCodeOffset.add(sizeShort*3).readU16(); 
    var insns_size = addrCodeOffset.add(sizeU32*3).readU16(); 
    if(tries_size > 256){
        console.log("tries_size:",tries_size.toString(16));
        console.log("insns_size:",insns_size.toString(16));
        return;
    }
    // console.log("tries_size:",tries_size.toString(16));
    // console.log("insns_size:",insns_size.toString(16));
    var codeLen = 16 + insns_size*2;
    if(tries_size > 0){
        var addrTryStart = addrCodeOffset.add(codeLen);
        // if(addrTryStart.readU16() == 0){ //padding
        //     addrTryStart = addrTryStart.add(0x2);
        // }
        if(codeLen %4 != 0){ //padding
            addrTryStart = addrTryStart.add(0x2);
        }
        // console.log("addrTryStart:",hexdump(addrTryStart));
        var addrTryEnd = addrTryStart.add(sizePointer*tries_size);
        var addrCodeEnd = CodeItemEnd(addrTryEnd);
        codeLen = addrCodeEnd - addrCodeOffset;
    }
    var allins = "";
    for(var i=0;i
```

    主要是如何计算 codeLen, 如果有 try 的函数就复杂很多。

    除了计算 codeLen，还有些函数的 code_item_offset 异常，比如代码中就有判断 offset 是 0 的，直接就是 dex 文件头了，应该是在哪里有还原吧。

    对于 tries_size,insns_size 异常并没有去一个个函数去查看什么问题。直接选择跳过。

    3.DexBase 的获取

    比较偷懒，直接使用网上随便找的 DumpDex 的 Frida 代码，Dump 下抽取后的 Dex 后，直接判断下长度。对于多 dex 没考虑。  

```
Interceptor.attach(addr_ClassLinker_DefineClass, {
        onEnter: function(args){
            if(DexBase) {
                //找到就不运行下面了
                return;
            }
            console.log("addr_ClassLinker_DefineClass:",DexBase);
            var dex_file = args[5];
            var base = ptr(dex_file).add(Process.pointerSize).readPointer();
            var size = ptr(dex_file).add(Process.pointerSize *2).readUInt();
            console.log("base:",base,"\tsize:",size);
            if(size > 0x3b0000 && size < 0x3f0000){
                DexBase = base;
            }
        },
        onLeave: function(retval) {
 
        }
    });
```

    Dex 长度是 0x3be578，取了个范围，ArtMethod::Invoke 运行的时候就会获取 DexBase。要注意的就是 Dump 前要触发 ClassLinker::DefineClass，一般是切换下界面，点点按键就有新的类创建触发了。

    4. 关于 ArtMethod::Invoke 不能 hook 到很多函数

    由于对 Fart 流程没理解，所以耽误了不少时间。问了 hanbingle 大佬后才知道这里只是通过反射运行的函数才能 HOOK 到。

    另外我使用 replace 比 attach hook 到的更少了，一直不知道什么问题。但是使用 replace 如果不调用 ArtMethod::Invoke 原始函数也不会触发程序填充函数，所以也就还是只用 attach 了。

**六、使用 Frida 脱壳脚本**

 上方的 Frida 编写时是对应另外一个 APK 进行编写的，所以到了本题也有一些修改，很不方便的一点就是 Dex 在内存中位置的取值是写死的，具体可以查看上传的代码。

  1. 由于自己写的 Frida 脚本就是按着 fart 的思路来写的，所以也会在自定义 ClassLoader 这里出错。![](https://bbs.pediy.com/upload/attach/202005/571058_AC3EK8X9SY5QUPX.jpg)

    Error: Cast from 'com.bytedance.frameworks.plugin.core.DelegateClassLoader' to 'dalvik.system.BaseDexClassLoader' isn't possible

    错误是由于 DelegateClassLoader 直接继承至 ClassLoader, 不能转换为 BaseDexClassLoader，也无法枚举出所有 ClassName

    2. 这时候虽然枚举不出来类，但是 Java.use("com.sup.android.superb.SplashActivity") 是正常的。那么可以直接不枚举 Class, 直接指定一个类名，然后枚举它的函数主动调用，Dump 下对应 Code。

```
function hook_java(){
    Java.perform(function(){
        loadClassAndInvoke("com.sup.android.superb.SplashActivity");
    });
}
 
function loadClassAndInvoke(className) {
    Java.perform(function(){
        try {
            var classResult = Java.use(className).class;
            if(!classResult) return;
 
            var methodArr = classResult.getDeclaredConstructors();
            methodArr = methodArr.concat(classResult.getDeclaredMethods());
 
            console.log(className,"\t",methodArr.length);
            for(var i=0;i -1){
                    if(methodName in dumpMethodName){
                        continue;
                    }
                    console.log("methodName:",methodName);
                    // c++层调用
                    if(ArtMethod_invoke_replace){
                        //每次都会报错,但是我还没找到更方便的
                        try{
                            dumpMethodName.push(methodName);
                            // console.log("getArtMethod:", hexdump(ptr(methodArr[i].getArtMethod())));
                            ArtMethod_invoke_replace(ptr(methodArr[i].getArtMethod()), ptr(0), ptr(0), 6, invokeSize, invokeStr);
                        } catch(error){
                            // console.log("ArtMethod_invoke error:[",className,"]",error);
                        }
                    }
                }
            }
             
        } catch (error) { 
            console.log("loadClassAndInvoke error:[",className,"]",error);
        }
    });
} 
```

    这时候 Dump 是成功的，还原到 Dex 文件，这个类就修复了。

![](https://bbs.pediy.com/upload/attach/202005/571058_RUMJB8KB8Q2VZ43.jpg)    

    3. 那么现在问题就是如何枚举 Dex 的 ClassName。其实这里可以直接使用 Fart 的 8958236_classlist_execute.txt 文件即可。但是还是想试试能不能直接通过 ClassLoader 枚举出来类。  

**七、解决枚举 Dex 类**

    1. 这时候查看 8958236_classlist_execute.txt, 发现里面其实是有我们需要枚举的类，现在就是看这个怎么枚举来的。

 ![](https://bbs.pediy.com/upload/attach/202005/571058_36GZ5DY6KR6GXFX.jpg)

    2.8958236_classlist_execute.txt 来源，他其实是通过解析 Dex 文件得来的。具体可以查看 Fart 源码的 dumpdexfilebyExecute 方法。那么得出结论，Fart 虽然枚举出来了这些类，但其实也不是通过 ClassLoader 枚举，没有参考价值。

![](https://bbs.pediy.com/upload/attach/202005/571058_HQFVMZMURBKGQZ4.jpg)

    3. 先看看普通的 ClassLoader 枚举类的方式      -> 这里代表继承自

    PathClassLoader->dalvik.system.BaseDexClassLoader

    dalvik.system.BaseDexClassLoader.pathList->dalvik.system.DexPathList

    pathList.dexElements->dalvik.system.DexPathList$Element

    dexElements.dexFile->dalvik.system.DexFile

    dexFile.getClassNameList

    那其实也就是获取到对应 DexFile 对象然后调用 getClassNameList 方法，看下 getClassNameList 方法好像也就是解析 Dex 文件，也不能参考。

    4. 查看 ClassLoader.java 源码可以看到一些与 java.lang.Package 类相关的字段和函数。而 Package 也并不是 Dex 相关。

    同时 ClassLoader 类也有字段 private transient long classTable; 看着比较像，但是 Frida 得出值为 0.

    5. 再次查看 com.bytedance.frameworks.plugin.core.DelegateClassLoader 类，发现有个字段名叫 pathClassLoader。尝试枚举后发现其实 pathClassLoader 字段对应的 DexFile 只能枚举出 100 多个类，和 6000 多差的太远。

    6. 看了一圈，决定这里也通过自己解析 DexFile 文件来实现枚举 Class  

```
DexBase = base;
DexSize = size;
// console.log("DexBase:",hexdump(base));
var string_ids_size = DexBase.add(0x38).readU32();
var string_ids_off = DexBase.add(0x3c).readU32();
console.log("uint string_ids_size:",string_ids_size); //.toString(1
console.log("uint string_ids_off:",string_ids_off);
var type_ids_size = ptr(DexBase).add(0x40).readU32();
var type_ids_off = ptr(DexBase).add(0x44).readU32();
console.log("uint type_ids_size:",type_ids_size);
console.log("uint type_ids_off:",type_ids_off);
                 
var class_idx = ptr(DexBase).add(0x60).readU32();
var class_defs_off = ptr(DexBase).add(0x64).readU32();
console.log("uint class_idx:",class_idx);
console.log("uint class_defs_off:",class_defs_off);
// var offsetStrEnd = DexBase.add(type_ids_off);
// console.log("offsetStrEnd:",offsetStrEnd);
for(var i=0; i size){
        console.log("offsetStr > size:",offsetStr,">",size);
        break;
    }
    var addrStr =  DexBase.add(offsetStr);
    // console.log("addrStr:", hexdump(addrStr));
    // console.log("addrStr.readU32:",);
    var classNameLen =  addrStr.readU8();
    if(classNameLen > 0x7f){
        //这里类名都没超过0x7F
        console.log("ClassName Len > 0x7f:",addrStr);
        var lebdtl = DecodeUnsignedLeb128(addrStr);
        addrStr = addrStr.add(lebdtl[1]);
    }else{
        addrStr = addrStr.add(1);
    }
    // console.log("addrStr:",addrStr);
    // 读utf16有错误
    // var str = addrStr.readUtf16String();
    var str = addrStr.readUtf8String();
    // console.log(i,":", str);
    // console.log(hexdump(addrStr));
    // break;
    str = str.replace(/L([^;]+);/,"$1").replace(/\//g,'.');
    classArr.push(str);
}
console.log("classArr.length:",classArr.length); 
```

    枚举出 6895 个类，枚举类问题解决。  

**八、脱壳操作**

    1. 修改脚本，直接根据指定 DexFile 文件枚举出的类列表依次主动调用。具体操作和那个作业类似。

```
function hook_java(){
    console.log("--------------------Start Invoke:",new Date().getTime());
    for(var i=0; i= 0){
            console.log("class:",classArr[i]);
            loadClassAndInvoke(classArr[i]);
        }
    }
    console.log("--------------------End Invoke:",new Date().getTime());
    dump_dex("fixed.dex");
} 
```

    2.Dump 包含 com.sup.android 字符串的类，共 2926 个函数体，修复后查看 Dex, 可以看到 com.sup.android 下的一些类函数都还原了。

![](https://bbs.pediy.com/upload/attach/202005/571058_YXDSRGHCZ5TPDZH.jpg)

    3. 直接 Dump 修复整个 Dex 的所有函数，这里直接把过滤字符置空即可  

![](https://bbs.pediy.com/upload/attach/202005/571058_SFEJE5UA3XSZ6MA.jpg)

    程序运行了大概 20 多分钟才结束，非常慢，Dump 出的 Bin 文件 40 多 M

![](https://bbs.pediy.com/upload/attach/202005/571058_WWVW7Q4TTY98VVX.jpg)

    共 Dump 下 14 万方法，修复后查看 Dex 文件

![](https://bbs.pediy.com/upload/attach/202005/571058_P8ZTUMZNTCFWRVQ.jpg)

    对比文件修改的地方非常多。

![](https://bbs.pediy.com/upload/attach/202005/571058_35WNPD4S6HPXXQV.jpg)

    大多数函数也已经修复了。现在问题就是一次运行太慢了。

**九、优化整体脱壳速度**

    1. 根据之前被动调用脱下来的函数可以得出结论，函数被修复后就一直保存在 Dex 文件中了。那么可以直接获取所有类主动调用，过程中不 Dump 下每个函数，而是等全部类主动调用完后 Dump 下当时内存的 Dex 文件。  

    2. 不进行 hook 或者直接 return 都可，这里还是留着，直接 return.  

![](https://bbs.pediy.com/upload/attach/202005/571058_Z6W2CENQKP6WQ3J.jpg)

    **这样时间大概只有 3-4 分钟，快了一些。**

    3. 再最初获取到 Dex 文件的时候 Dump 一次保存问 init.dex。另外在主动调用完之后再保存一份 fixed.dex。

![](https://bbs.pediy.com/upload/attach/202005/571058_ZVYFB929V9DFRNU.jpg)

    fixed.dex 中相对于 init.dex 也填充了很多函数体。  

    4. 对比整体 Dex 和函数粒度修复的 Dex  

![](https://bbs.pediy.com/upload/attach/202005/571058_9R9J34CU5G98D7Q.jpg)

    整体 Dump 的比函数粒度修复的多了一点, 应该是函数粒度有些运行 BUG

    那么像这种填充函数体后可以直接 Dump 的还是直接 Dump 整体 Dex 更快也更稳定。  

**十、总结**

    1. 本题特别之处就是自定义 ClassLoader 导致不能通过 ClassLoader 枚举出类，直接解析 Dex 文件也方便解决

    2. 自己写的这个脚本，其实和网上整体 DexDump 就多了一个主动调用，只是可以单个函数调试，查看某个函数如何填充的。便于个人理解，实际作用倒也不大。

    3. 示例程序没有禁止 Frida，方便很多。  

APK 超过 8M 无法直接上传 [https://pan.baidu.com/s/1GgAa5EF5rtZ5HX9vguSfnA](https://pan.baidu.com/s/1GgAa5EF5rtZ5HX9vguSfnA) s0oe 

附件第一个 js 是另外一个测试 app，第二个 js 对应网盘 apk

[【公告】【iPhone 13 大奖等你拿】看雪. 众安 2021 KCTF 秋季赛 防守篇 - 征题倒计时（11 月 14 日截止）！](https://bbs.pediy.com/thread-269228.htm)

最后于 2020-7-13 15:19 被无造编辑 ，原因：

上传的附件：

*   [dexdump.js](javascript:void(0)) （18.25kb，226 次下载）
*   [dexdump_com_sup_android.js](javascript:void(0)) （18.90kb，142 次下载）