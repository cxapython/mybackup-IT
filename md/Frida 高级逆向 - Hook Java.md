> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/xiaoweigege/p/14868003.html)

Frida Hook Java 层

Frida 两种启动方式的区别[#](#frida两种启动方式的区别)
-----------------------------------

1.  `span` 模式：frida 重新打开一个进程  
    `frida -U -f 包名 -l js路径 --no-pause`
2.  `attch` 模式: 附加在当前打开的进程  
    `frida -U -l js路径 --no-pause`  
    区别就是一个带 `-f` 一个不带

### 命令行参数详解[#](#命令行参数详解)

```
--version             显示版本号
  -h, --help            查看帮助
  -D ID, --device=ID    用给定的ID连接到设备
  -U, --usb             连接 USB 设备
  -R, --remote          连接远程设备
  -H HOST, --host=HOST  连接远程的设备 地址
  -f FILE, --file=FILE  spawn FILE 模式
  -F, --attach-frontmost
                        attach to frontmost application 附加到最前端的应用程序
  -n NAME, --attach-name=NAME
                        attach to NAME 附加的名称
  -p PID, --attach-pid=PID
                        attach to PID 附加的进程ID
  --stdio=inherit|pipe  stdio behavior when spawning (defaults to “inherit”)
  --runtime=duk|v8      script runtime to use (defaults to “duk”) 指定js解释器 duk 或者 v8
  --debug               enable the Node.js compatible script debugger 
  -l SCRIPT, --load=SCRIPT 加载脚本文件
                        load SCRIPT
  -P PARAMETERS_JSON, --parameters=PARAMETERS_JSON
                        Parameters as JSON, same as Gadget
  -C CMODULE, --cmodule=CMODULE
                        load CMODULE
  -c CODESHARE_URI, --codeshare=CODESHARE_URI
                        load CODESHARE_URI
  -e CODE, --eval=CODE  evaluate CODE
  -q                    quiet mode (no prompt) and quit after -l and -e
  --no-pause            automatically start main thread after startup 启动后自动启动主线程
  -o LOGFILE, --output=LOGFILE 日志输出文件
                        output to log file
  --exit-on-error       exit with code 1 after encountering any exception in
                        the SCRIPT 异常退出脚本
```

Hook 一个 Java 层函数[#](#hook一个java层函数)
-----------------------------------

*   Java.use  
    功能:  
    动态为 className 生成一个 JavaScript 包装器；可以通过调用 $new() 来实例化对象来调用构造函数  
    参数: className
*   implementaion  
    功能: 方法重新实现的属性
*   overloads  
    功能: 重载函数指定类型方法

示例:

```
function hook_java() {
    // 必须写在 Java 虚拟机中 
    Java.perform(function() {

        const login = Java.use('com.example.androiddemo.Activity.LoginActivity');
        login.a.overload('java.lang.String', 'java.lang.String').implementation = function(a1, a2) {
            let result = this.a(a1, a2);
            console.log(a1, a2, result);
            return result
        }
    })

}
```

修改一个函数返回值或参数[#](#修改一个函数返回值或参数)
------------------------------

示例:

```
function hook_java() {
    // 必须写在 Java 虚拟机中 
    Java.perform(function() {

        const login = Java.use('com.example.androiddemo.Activity.LoginActivity');
        const string = Java.use('java.lang.String');
        login.a.overload('java.lang.String', 'java.lang.String').implementation = function(a1, a2) {
            // string int bool 这3种类型 frida 会直接帮我们转换成 java 类型的
            var result = '这是我修改返回的'

            // 我们自己进行类型转换
            var result = string.$new("我们自己去转换的java类型的")

            return result
        }
    })

}
```

调用静态函数和非静态函数[#](#调用静态函数和非静态函数)
------------------------------

静态函数和静态方法可以使用 `Java.use` 出来的函数直接调用。`Java.use` 相当于 new 了一个新的对象  
非静态函数和静态方法, 有两种使用情况，一种是 `Java.use` 出来的对象，自己进行实例化 `obj.$new()` 这种方法需要知道构造参数，比较麻烦。  
那么可以使用 `Java.choose` 该方法是从内存中找到实例化好的类进行调用

示例:

```
function hook_java() {
    // 必须写在 Java 虚拟机中 
    Java.perform(function() {

        // 调用静态方法
        var frida_activate = Java.use('com.example.Frida');
        frida_activate.setStatic_bool)var()

        // 调用非静态方法
        Java.choose('com.example.androiddemo.Activity.FridaActivity2', {
            // 匹配到的情况下,进行调用
            // 注意： 内存中可以有多个实例化好的类?
            onMatch: function (instance) {
                // 调用方法
                instance.setBool_var()
                // 设置变量值
                instance.static_bool_var.value = true;
            },
            onComplete: function () {

            }
        })

})

}
```

设置成员变量[#](#设置成员变量)
------------------

设置和函数名相同的成员变量  
属性和函数名重名了, 需要在属性前面 加一个 下划线 `_`

示例:

```
function hook_java() {
    // 必须写在 Java 虚拟机中 
    Java.choose('com.example.androiddemo.Activity.FridaActivity3', {
            onMatch: function (instance) {
                // instance.static_bool_var.value = true;
                instance.bool_var.value = true;
                // TODO 关注一下这个属性，他跟函数名重名了，需要加一个  _  下划线
                instance._same_name_bool_var.value = true;
                console.log(instance.bool_var.value, instance._same_name_bool_var.value)
            },
            onComplete: function () {

            }
        })

}
```

Hook 内部类，枚举类的函数[#](#hook内部类，枚举类的函数)
-----------------------------------

示例:

```
function call_frida4() {
    // hook 类的多个函数, 内部类
    Java.perform(function () {
        // 这是hook类的 内部类
        const FridaActivity4 = Java.use('com.example.androiddemo.Activity.FridaActivity4$InnerClasses');
        // getDeclaredMethods 获取当前类的方法   getMethods 获取当前类以及继承类的所有方法
        const methods = FridaActivity4.class.getDeclaredMethods();
        for (let method of methods) {
            // console.log(method)
            let method_str = method.toString();
            let mtdsplit = method_str.split('.')
            let meds = mtdsplit[mtdsplit.length - 1].replace('()', '');
            console.log(meds)
            FridaActivity4[meds].implementation = function () {
                return true
            }
        }


    })
}
```

查找接口，Hook 动态加载 dex[#](#查找接口，hook动态加载dex)
----------------------------------------

示例:

```
function call_frida5() {

    // hook 动态加载的dex 类, 以及查看类的类名

    Java.perform(function () {

        // 1. 尝试 hook 一下 接口类
        // 会提示 Error: java.lang.ClassNotFoundException:
        // Java.use('com.example.androiddemo.Dynamic.AbstractC0000CheckInterface');

        // 2. 查看动态加载的类名
        const FridaActivity5 = Java.use('com.example.androiddemo.Activity.FridaActivity5');
        FridaActivity5.getDynamicDexCheck.implementation = function () {

            let result = this.getDynamicDexCheck();
            // 查看当前返回值的 类名
            console.log(result.$className)
            return result

        }

        // 这个时候还是 没有找到类 需要将 类 loader 进来
        // Java.use('com.example.androiddemo.Dynamic.DynamicCheck').check.implementation = function (){
        //     return true
        // }

        // 3. hook 动态加载的 dex
        // 枚举类加载
        Java.enumerateClassLoaders({
            onMatch: function (loader) {
                console.log(loader)
                try {
                    if (loader.findClass('com.example.androiddemo.Dynamic.DynamicCheck')) {
                        console.log(loader);
                        // 切换classloader
                        Java.classFactory.loader = loader
                    }
                } catch (e) {
                    // console.log(e)

                }


            },
            onComplete: function () {

            }
        })
        // 这个时候再来 hook 这个类
        Java.use('com.example.androiddemo.Dynamic.DynamicCheck').check.implementation = function () {
            return true
        }


    })
}
```

枚举 class[#](#枚举class)
---------------------

示例:

```
function call_frida6() {

    //  hook 多个 class
    // 发现一个关键点 就是 当一个类 没有执行的时候, frida 枚举是枚举不出来的。
    Java.perform(function () {
        // 遍历当前 loader 的 所有类
        Java.enumerateLoadedClasses({
            onMatch: function (name, handle) {
                // console.log(name)
                if (name.indexOf('com.example.androiddemo.Activity.Frida6') >= 0) {
                    console.log(name);
                    Java.use(name).check.implementation = function () {
                        return true;
                    }

                }
            },
            onComplete: function () {

            }
        })


    })
}
```

Hook 构造函数[#](#hook-构造函数)
------------------------

$init 表示构造函数

示例:

```
function call_frida6() {

    Java.perform(function () {
        // $init 表示构造函数
       Java.use('com.example.androiddemo.Dynamic.DynamicCheck').$init.implementation = function (a,b) {
            return this.$init(a,b)
        }


    })
}
```

打印调用栈[#](#打印调用栈)
----------------

示例:

```
function show_stack() {
    Java.perform(function () {
        console.log(Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()));

    })
}
```

加载本地 DEX[#](#加载本地dex)
---------------------

```
// 打开 dex 文件
var dex = Java.openClassFile('/data/local/tmp/ddex.dex')
Java.perform(function () {
        // 加载 dex
        dex.load()
        // 进行hook
        Java.use('com.tlamb96.kgbmessenger.LoginActivity').j.implementation = function () {
            return true
        }

})
```