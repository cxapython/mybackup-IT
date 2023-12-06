> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-229657.htm)

> [原创] 进阶 Frida--Android 逆向之动态加载 dex Hook（三）（下篇）

上篇花了很多篇幅讲了 Robust 的原理，并以做题的思路去求解了这个示例 ctf，其实这是一种思路的启示，当我们在不知道怎么 hook 动态加载的 dex，jar 时候，找找是否存在能够操作动态加载出来的类的方法。当然这不是重点，这篇我会重点给大家分享如何使用 frida 去 hook DexclassLoader，怎么用反射直接调用类的方法，达到跟 hook 一般类一样的效果。

 

目录

*            [文章涉及内容及使用到的工具](#文章涉及内容及使用到的工具)
*                    0x00 使用到的工具
*                    0x01 涉及知识点
*            [代码分析与构造](#代码分析与构造)
*                    0x00 Frida Spawn 的使用
*                    0x01 DexClassLoader 动态加载机制
*                    0x02 Frida 方法重载（overload）
*                    0x03 Frida 类型转换（Java.cast）
*                    0x04 Frida 创建任意类型数组（Java.array）
*                    0x05 最终代码
*            [总结](#总结)

文章涉及内容及使用到的工具
-------------

### 0x00 使用到的工具

*   ADT（Android Developer Tools）
*   Jadx-gui
*   JEB
*   frida
*   apktool
*   android 源码包
*   天天模拟器（genymotion，实体机等亦可）

### 0x01 涉及知识点

*   Java 泛型
*   Java 反射机制
*   DexClassLoader 动态加载机制
*   Frida 基本操作
*   Frida 创建任意类型数组（Java.array）
*   Frida 类型转换（Java.cast）
*   Frida 方法重载（overload）
*   Frida Spawn

代码分析与构造
-------

### 0x00 Frida Spawn 的使用

经过上篇文章我们对 Robust 的原理学习和对 app 的分析，我们知道 Robust 的其实就是在正常的使用`DexClassLoader`去动态加载文件，随后通过`反射`的方式去调用类方法或成员变量。

 

同时在上篇文章中，我们也知道 Robust 调用 DexClassLoader 的类是在`PatchExecutor`中，而调用 PatchExecutor 类是在一个叫`runRobust`的方法中，这个方法就在`MainActivity`中，并且在`onCreate`方法中调用。  
![](https://bbs.pediy.com/upload/attach/201807/811277_9SM4VE5P7XUE4WS.png)

 

现在我们明白了一点是，app 动态加载 dex 的地方是在 onCreate 中，也就是说 app 一启动就执行了动态加载，并不是在我们点击按钮的时候。所以这个地方，我们要执行 py 脚本的话，需要在 app 执行 onCreate 方法之前。frida 有一个功能可以为我们生成一个进程而不是将它注入到运行中的进程中，它注入到 Zygote 中，生成我们的进程并且等待输入。

 

我们可以通过`-f`参数选项来实现。

```
frida -U -f app完整名
```

![](https://bbs.pediy.com/upload/attach/201807/811277_42H7CXEWCHXM7QJ.gif)  
从上面可以看到，通过`-f`参数，frida 会 Spawned 这个应用，在这个时候启动 python 脚本，再执行`%resume`命令, 我们就可以在 app 执行 onCreate 方法前完成脚本的启动，这时候就能 hook 住 onCreate 中执行的一些方法。

### 0x01 DexClassLoader 动态加载机制

好，我们已经知道怎么 hook 住 onCreate 中执行的方法了，现在我们就来试试，第一个目标是能够获取到动态加载的 dex 中的类。在这之前我们来看看，直接去 hook 动态加载的类会出现什么情况。  
测试 js 代码如下，我们尝试获取 dex 中的`MainActivityPatch`类。

```
Java.perform(function(){
        console.log("test");
        Java.use("cn.chaitin.geektan.crackme.MainActivityPatch");
        console.log("test over");
 
});
```

完整代码：（后面的代码就只贴 js_code = 中的 javascript 代码了，因为这里面只有 js_code 的代码变化了）

```
# -*- coding: UTF-8 -*-
import frida,sys
 
def on_message(message, data):
    if message['type'] == 'send':
        print("[*] {0}".format(message['payload']))
    else:
        print(message)
 
 
js_code = '''
    Java.perform(function(){
        console.log("test");
        Java.use("cn.chaitin.geektan.crackme.MainActivityPatch");
        console.log("test over");
 
});
'''
 
 
session = frida.get_usb_device().attach("cn.chaitin.geektan.crackme")
script = session.create_script(js_code)
script.on('message',on_message)
script.load()
sys.stdin.read()
```

![](https://bbs.pediy.com/upload/attach/201807/811277_VFGG5639MXG2B4S.gif)

 

可以清晰的看到错误信息，未找到类。

```
java.lang.ClassNotFoundException: Didn\'t find class "cn.chaitin.geektan.crackme.MainActivityPatch
```

果不其然，这样去直接 hook 类肯定是不行的，但我们知道只要是从外部资源文件中动态加载 dex，一般都是采用 DexClassLoader 动态加载的。学习过 Java 的同学应该知道，DexClassLoader 动态加载的主要方法就是`loadClass()`。我们从 Java 源码上去分析一下，这里给大家一个 java 开发文档的查看地址：[访问](https://www.androidos.net.cn/android/8.0.0_r4/xref/libcore/dalvik/src/main/java/dalvik/system/DexClassLoader.java)

 

我们看到 DexClassLoader 的构造函数有 4 个参数，这里没有 loadClass()，我们继续查看它的父类 BaseDexClassLoader。

```
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String librarySearchPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), librarySearchPath, parent);
    }
}
```

同样 BaseDexClassLoader 也没有 loadClass()，最终在它的父类`ClassLoader`中找到了 loadClass() 方法。  
![](https://bbs.pediy.com/upload/attach/201807/811277_JRV9TXD7T82PH64.png)

 

可以看到 DexClassLoader 加载的逻辑其实就是 ClassLoader 中的 loadClass()，它的机制简单的了解到这里，现在我们来试试，通过这样的方式能不能 hook 我们想要的类。

 

我们先 hook DexClassLoader 的构造函数，看看传递进的参数值是什么。

```
Java.perform(function(){
        //创建一个DexClassLoader的wapper
        var dexclassLoader = Java.use("dalvik.system.DexClassLoader");
        //hook 它的构造函数$init，我们将它的四个参数打印出来看看。
        dexclassLoader.$init.implementation = function(dexPath,optimizedDirectory,librarySearchPath,parent){
             console.log("dexPath:"+dexPath);
            console.log("optimizedDirectory:"+optimizedDirectory);
             console.log("librarySearchPath:"+librarySearchPath);
            console.log("parent:"+parent);
            //不破换它原本的逻辑，我们调用它原本的构造函数。
          this.$init(dexPath,optimizedDirectory,librarySearchPath,parent);
        }
        console.log("down!");
 
});
```

执行看看：

 

我们获取到了构造函数的参数，简单看一下。  
![](https://bbs.pediy.com/upload/attach/201807/811277_KPXPJTC7VCDE2RU.gif)

```
dexPath:/data/data/cn.chaitin.geektan.crackme/cache/GeekTan/patch_temp.jar
optimizedDirectory:/data/data/cn.chaitin.geektan.crackme/cache
librarySearchPath:null
parent:dalvik.system.PathClassLoader[DexPathList[[zip file "/data/app/cn.chaitin.geektan.crackme-2.apk"],nativeLibraryDirectories=[/data/app-lib/cn.chaitin.geektan.crackme-2, /system/lib, /system/lib/arm]]]
```

### 0x02 Frida 方法重载（overload）

接下来，就来尝试一下获取动态加载的类。

```
Java.perform(function(){
        var dexclassLoader = Java.use("dalvik.system.DexClassLoader");
        var hookClass = undefined；
        //hook loadClass方法
        dexclassLoader.loadClass.implementation = function(name){
        /*因为loadClass可能会加载很多类，所以我们得定义个hookname变量，
        这样有针对的获取我们想要的类*/
            var hookname = "cn.chaitin.geektan.crackme.MainActivityPatch";
            var result = this.loadClass(name,false);
            if(name === hookname){
                hookClass = result;
                console.log(hookClass);
                return result;
            }
            return result;
        }
});
```

执行看看结果。  
![](https://bbs.pediy.com/upload/attach/201807/811277_Q4HQT66N83ETYA9.gif)  
可以看到 frida 报错了，从报错信息我们可以看到 loadClass() 有 2 个重载方法，我们这里需要通过`overload`指定我们需要 Hook 的重载方法才行，如果你不知道该用哪个重载方法，可以先让 frida 报错，然后它会把所有的重载方法抛出在错误信息中，像下面一样，这是个小技巧。

```
{'type': 'error', 'description': "Error: loadClass(): has more than one overload, use
.overload() to choose from:.overload('java.lang.String')
.overload('java.lang.String', 'boolean')"....} 
```

我们在来看看 ClassLoader 类中的两个 loadClass 重载方法。  
loadClass(String name)；

```
public Class loadClass(String name) throws ClassNotFoundException {
        return loadClass(name, false);
    }
```

loadClass(String name, boolean resolve)；

```
protected Class loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
            // First, check if the class has already been loaded
            Class c = findLoadedClass(name);
            if (c == null) {
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }
 
                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    c = findClass(name);
                }
            }
            return c;
    }
```

可以看到真正执行 loadClass 的方法是`loadClass(String name, boolean resolve)；`，而`loadClass(String name)；`只是简单的调用它。那我们 hook 哪个呢？当然是选择第一个，因为我们 hook 第一个，然后在其调用第二个重载方法，这不就是它原本的逻辑吗，这样理解更容易一些。下面，我们来重新构造一下 js 代码。

```
  Java.perform(function(){
        var dexclassLoader = Java.use("dalvik.system.DexClassLoader");
        var hookClass = undefined;
        var ClassUse = Java.use("java.lang.Class");
 
        dexclassLoader.loadClass.overload('java.lang.String').implementation = function(name){
           //定义一个String变量，指定我们需要的类
            var hookname = "cn.chaitin.geektan.crackme.MainActivityPatch";
            //直接调用第二个重载方法，跟原本的逻辑相同。
            var result = this.loadClass(name,false);
            //如果loadClass的name参数和我们想要hook的类名相同
            if(name === hookname){
                //则拿到它的值
                hookClass = result;
                //打印hookClass变量的值
                console.log(hookClass);
                send(hookClass);
                return result;
            }
            return result;
        }
});
```

执行看看：  
![](https://bbs.pediy.com/upload/attach/201807/811277_Y9AW49VQA87B2TR.gif)

 

可以看到打印出的 hookClass 变量值即为我们想要 hook 的类，说明我们已经通过 hook loadClass 拿到了 MainActivityPatch 类。

```
class cn.chaitin.geektan.crackme.MainActivityPatch
[*] {'$handle': '0x1d2006ee', '$weakRef': 693}
```

### 0x03 Frida 类型转换（Java.cast）

那接下来怎么调用类方法呢，这里我们是不能直接通过 (.) 运算符直接调用方法的，可以看到 loadClass()返回类型的是一个泛型，其中？代表任何类型，因为 loadClass(）也不知道要加载的类的类型，所以返回类型就采用`Class<?>`代表所有类型的类，所以最后返回的是一个类型为指向`MainActivityPatch`的 Class 对象

```
protected Class loadClass(String name, boolean resolve)；
```

理论是这样的，但实际上却不是，我们还需要进行类型转换。这里 Frida 提供的一个方法处理泛型的方法`Java.cast`。  
构造代码如下：

```
    Java.perform(function(){
        var hookClass = undefined;
        var ClassUse = Java.use("java.lang.Class");
        var dexclassLoader = Java.use("dalvik.system.DexClassLoader");
        var constructorclass = Java.use("java.lang.reflect.Constructor");
        var objectclass= Java.use("java.lang.Object");
        dexclassLoader.loadClass.overload('java.lang.String').implementation = function(name){
            var hookname = "cn.chaitin.geektan.crackme.MainActivityPatch";
            var result = this.loadClass(name,false);
 
            if(name == hookname){
                var hookClass = result;
                console.log("------------------------------CAST--------------------------------")
                //类型转换
                var hookClassCast = Java.cast(hookClass,ClassUse);
                //调用getMethods()获取类下的所有方法
                var methods = hookClassCast.getMethods();
                console.log(methods);
                console.log("-----------------------------NOT CAST-----------------------------")
                //未进行类型转换，看看能否调用getMethods()方法
                var methodtest = hookClass.getMethods();
                console.log(methodtest);
                console.log("---------------------OVER------------------------")
                return result;
 
            }
            return result;
        }
 
 
});
```

执行看看结果：  
![](https://bbs.pediy.com/upload/attach/201807/811277_9ETRXTKCV9ENTHN.gif)  
可以清晰的看到，cast 后才能调用 getMethods()，没有 cast 则会报未定义不能调用的错误。  
![](https://bbs.pediy.com/upload/attach/201807/811277_FEK6745SS43JQFV.png)

### 0x04 Frida 创建任意类型数组（Java.array）

现在我们得到了一个类型为`MainActivityPatch`的 Class 对象，我们接下来就来看看怎么调用`Joseph`方法。在这之前，你需要对反射的用法有一定的了解。至于怎么用，就针对实际情况选取你认为最好的办法就行了。  
当然在我多次踩坑之后，比如：

*   在 Class 的 getMethod 方法中，怎么用 js 构造`int.class`,`float.class`，以及构造 Integer.TYPE 数组出现莫名错误。
*   怎么利用 Frida 函数构造任意类型的数组。
*   无参构造函数调用 newInstance()，跟有参构造函数调用 newInstance() 的问题。
*   ....

我认为还是有必要给大家提供一种`比较通用`的方法。  
1. 利用 getDeclaredConstructor() 获取具有指定参数列表构造函数的 Constructor。

```
public Constructor getDeclaredConstructor(Class... parameterTypes)
     throws NoSuchMethodException, SecurityException {
     return getConstructor0(parameterTypes, Member.DECLARED);
 } 
```

可以看到，参数是一个`Class<?>...`，也就是说这是一个`[Ljava.lang.Object;`类型的数组。我们现在要得到`MainActivityPatch`构造函数对象，从代码中可以知道参数是 Object 类型。  
![](https://bbs.pediy.com/upload/attach/201807/811277_BHK3N6DFDVSS7RB.png)  
我们怎么来构造并传入这个数组呢。

```
\\利用java.array的标准写法
var objectclass= Java.use("java.lang.Object");
var ConstructorParam =Java.array('Ljava.lang.Object;',[objectclass.class]);
var a = hookClassCast.getDeclaredConstructor(ConstructorParam);
 
\\偷懒写法
var a = hookClassCast.getDeclaredConstructor([objectclass.class]);
```

这里要特别提出的是 java.array() 的用法格式

```
Java.array('type',[value1,value2,....]);
```

支持什么 type，大家可以参看`frida-java`的[源码](https://github.com/frida/frida-java)。在`class-factory.js`中就可以找到了。基本类型如下：

```
1. Z -- boolean
2. B -- byte
3. C -- char
4. S -- short
5. I --  int
6. J -- long
7. F -- float
8. D  -- double
9. V -- void
```

话题回过来，我们 getDeclaredConstructor() 得到了构造函数 Constructor，我们现在要将它实例化，再来看看 MainActivityPatch 的构造函数，传递一个 object 对象，并将它强制转换成 MainActivity 类型。  
那我们实例化的参数就是 MainActivity 对象了。

```
public class MainActivityPatch {
    MainActivity originClass;
    //构造函数
    public MainActivityPatch(Object obj) {
        this.originClass = (MainActivity) obj;
    }
```

代码如下：

```
Java.perform(function(){
        var hookClass = undefined;
        var ClassUse = Java.use("java.lang.Class");
        var objectclass= Java.use("java.lang.Object");
        var dexclassLoader = Java.use("dalvik.system.DexClassLoader");
        var orininclass = Java.use("cn.chaitin.geektan.crackme.MainActivity");
        var Integerclass = Java.use("java.lang.Integer");
        //实例化MainActivity对象
        var mainAc = orininclass.$new();
 
 
        dexclassLoader.loadClass.overload('java.lang.String').implementation = function(name){
            var hookname = "cn.chaitin.geektan.crackme.MainActivityPatch";
            var result = this.loadClass(name,false);
 
            if(name == hookname){
                var hookClass = result;
                var hookClassCast = Java.cast(hookClass,ClassUse);
                console.log("-----------------------------BEGIN-------------------------------------");
                //获取构造器
                var ConstructorParam =Java.array('Ljava.lang.Object;',[objectclass.class]);
                var Constructor = hookClassCast.getDeclaredConstructor(ConstructorParam);
                console.log("Constructor:"+Constructor);
                console.log("orinin:"+mainAc);
                //实例化，newInstance的参数也是Ljava.lang.Object;
                var instance = Constructor.newInstance([mainAc]);
                console.log("patchAc:"+instance);
                send(instance);
console.log("--------------------------------------------------------------------");
                return result;
 
            }
            return result;
        }
});
```

看看结果：  
![](https://bbs.pediy.com/upload/attach/201807/811277_G6ENHAYADVA363K.gif)  
可以看到，得到的结果为：

```
Constructor:public cn.chaitin.geektan.crackme.MainActivityPatch(java.lang.Object)
orinin:cn.chaitin.geektan.crackme.MainActivity@4a7ebc0c
MainActivityPatchInstance:cn.chaitin.geektan.crackme.MainActivityPatch@4a7f9d74
```

2. 我们得到了实例化的类后，第二步就是利用 getDeclaredMethods()，获取本类中的所有方法，包括私有的 (private、protected、默认以及 public) 的方法，并通过数组下标的方式获取我们想要的方法。

```
public Method[] getDeclaredMethods() throws SecurityException {
       Method[] result = getDeclaredMethodsUnchecked(false);
       for (Method m : result) {
           // Throw NoClassDefFoundError if types cannot be resolved.
           m.getReturnType();
           m.getParameterTypes();
       }
       return result;
   }
```

从 getDeclaredMethods()，我们知道它返回的是一个 Method 数组。

```
var func = hookClassCast.getDeclaredMethods();
console.log(func);
//直接通过下标获取我们要调用的方法
console.log(func[0]);
```

看看一个完整的示例，和上面的一样，仅仅调用了 getDeclaredMethods() 方法。

```
Java.perform(function(){
        var hookClass = undefined;
        var ClassUse = Java.use("java.lang.Class");
        var objectclass= Java.use("java.lang.Object");
        var dexclassLoader = Java.use("dalvik.system.DexClassLoader");
        var orininclass = Java.use("cn.chaitin.geektan.crackme.MainActivity");
        var Integerclass = Java.use("java.lang.Integer");
        //实例化MainActivity对象
        var mainAc = orininclass.$new();
 
 
        dexclassLoader.loadClass.overload('java.lang.String').implementation = function(name){
            var hookname = "cn.chaitin.geektan.crackme.MainActivityPatch";
            var result = this.loadClass(name,false);
 
            if(name == hookname){
                var hookClass = result;
                var hookClassCast = Java.cast(hookClass,ClassUse);
                console.log("-----------------------------BEGIN-------------------------------------");
                //获取构造器
                var ConstructorParam =Java.array('Ljava.lang.Object;',[objectclass.class]);
                var Constructor = hookClassCast.getDeclaredConstructor(ConstructorParam);
                console.log("Constructor:"+Constructor);
                console.log("orinin:"+mainAc);
                //实例化，newInstance的参数也是Ljava.lang.Object;
                var instance = Constructor.newInstance([mainAc]);
                console.log("MainActivityPatchInstance:"+instance);
                send(instance);
                console.log("----------------------------Methods---------------------------------");
                var func = hookClassCast.getDeclaredMethods();
                console.log(func);
                console.log("--------------------------Need Method---------------------------------");
                console.log(func[0]);
                var f = func[0];
                console.log("---------------------------- OVER---------------------------------");
                return result;
 
            }
            return result;
        }
});
```

执行看看。  
![](https://bbs.pediy.com/upload/attach/201807/811277_2GXUKDM6HD46Z89.gif)

 

可以看到，我们已经拿到了我们想要的方法。  
![](https://bbs.pediy.com/upload/attach/201807/811277_F9W6UBBK4YWVA2P.png)

 

3. 接下来就是调用 Method.invoke() 去执行方法了，invoke 方法的第一个参数是执行这个方法的对象实例，第二个参数是带入的实际值数组，返回值是 Object，也既是该方法执行后的返回值。

```
public native Object invoke(Object obj, Object... args)
           throws IllegalAccessException, IllegalArgumentException, InvocationTargetException;
```

那现在就有一个问题，第二个值是一个数据，我们需要带入实际值的数组，那这么来构造数组呢，很简单刚刚我们已经学习了 Frida 中 Java.array 的用法。现在我们就来构造 2 个实际值的 Integer 数组。

```
var Integerclass = Java.use("java.lang.Integer");
var num1 = Integerclass.$new(5);
var num2 = Integerclass.$new(6);
var numArr1 = Java.array('Ljava.lang.Object;',[num1,num2]);
var num3 = Integerclass.$new(7);
var num4 = Integerclass.$new(8);
var numArr2 = Java.array('Ljava.lang.Object;',[num3,num4]);
```

接下来我们就可以愉快的调用 Joseph 方法了。

### 0x05 最终代码

来看看最终代码，这里就不在写注释了，大家应该都能看懂了。  
最终代码脚本：[下载](https://github.com/ghostmaze/Android-Reverse/blob/master/Hello%20Baby%20Dex/dex3.py)

```
Java.perform(function(){
        var hookClass = undefined;
        var ClassUse = Java.use("java.lang.Class");
        var objectclass= Java.use("java.lang.Object");
        var dexclassLoader = Java.use("dalvik.system.DexClassLoader");
        var orininclass = Java.use("cn.chaitin.geektan.crackme.MainActivity");
        var Integerclass = Java.use("java.lang.Integer");
        var mainAc = orininclass.$new();
 
 
        dexclassLoader.loadClass.overload('java.lang.String').implementation = function(name){
            var hookname = "cn.chaitin.geektan.crackme.MainActivityPatch";
            var result = this.loadClass(name,false);
            if(name == hookname){
                var hookClass = result;
                var hookClassCast = Java.cast(hookClass,ClassUse);
                console.log("-----------------------------GET Constructor-------------------------------------");
                var ConstructorParam =Java.array('Ljava.lang.Object;',[objectclass.class]);
                var Constructor = hookClassCast.getDeclaredConstructor(ConstructorParam);
                console.log("Constructor:"+Constructor);
                console.log("orinin:"+mainAc);
                var instance = Constructor.newInstance([mainAc]);
                console.log("patchAc:"+instance);
                send(instance);
 
                console.log("-----------------------------GET Methods----------------------------");
                var func = hookClassCast.getDeclaredMethods();
                console.log(func);
                console.log("--------------------------GET Joseph Function---------------------------");
                console.log(func[0]);
                var f = func[0];
                var num1 = Integerclass.$new(5);
                var num2 = Integerclass.$new(6);
                var numArr1 = Java.array('Ljava.lang.Object;',[num1,num2]);
                var num3 = Integerclass.$new(7);
                var num4 = Integerclass.$new(8);
                var numArr2 = Java.array('Ljava.lang.Object;',[num3,num4]);
                console.log("-----------------------------GET Array------------------------------");
                console.log(numArr1);
                console.log(numArr2);
                var rtn1 = f.invoke(instance,numArr1);
                var rtn2 = f.invoke(instance,numArr2);
                console.log("--------------------------------FLAG---------------------------------");
                console.log("DDCTF{"+rtn1+rtn2+"}");
                console.log("--------------------------------OVER--------------------------------");
                return result;
 
            }
            return result;
        }
});
```

执行后发现成功 ：D  
![](https://bbs.pediy.com/upload/attach/201807/811277_H3F66QKXWTUSKZ5.gif)  
得到最终答案：  
![](https://bbs.pediy.com/upload/attach/201807/811277_Q8Z3YAUUHSDS4N2.png)

总结
--

通过对这个例子的一个完整过程的学习，可以说算是掌握了 Frida java 中一些比较高级的用法

*   java.array 任意数据的构造
*   java.cast 类型转换
*   overload 方法重载
*   spawn

同时也对 frida 怎么 hook 动态加载 dex 的方法有了一个清晰的思路，当然 frida 学习路还很长......

[【公告】欢迎大家踊跃尝试高研班 11 月试题，挑战自己的极限！](https://bbs.pediy.com/thread-270220.htm)

最后于 2018-7-8 14:36 被 ghostmazeW 编辑 ，原因：