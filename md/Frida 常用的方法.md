
Frida 常用的方法
--------

安装运行
----

1.  电脑端安装  
    sudo pip3 install frida pip3 install frida-tools  
    
2.  下载对应版本 server [https://github.com/frida/frida/releases](https://link.zhihu.com/?target=https%3A//github.com/frida/frida/releases)  
    

![](https://pic1.zhimg.com/v2-13c4d4c9a690a310af2f3f3ddb953348_r.jpg)

```
adb push frida-server-12.11.12-android-arm64 /data/local/tmp

```

1.  运行 frida-server  
    adb shell cd /data/local/tmp chmod 755 ./frida-server-12.11.12-android-arm64 ./frida-server-12.11.12-android-arm64 &  
    
2.  转发端口：  
    adb forward tcp:27042 tcp:27043  
    
3.  验证 (列出正在运行的包名)：  
    frida-ps -U  
    

常用工具函数
------

### 1. 打印某个类的所有成员变量

```
function dumpAllFieldValue(obj) {
    if (obj === null) {
        return;
    }

    console.log("Dump all fields value for  " + obj.getClass() + " :");

    var cls = obj.getClass();

    while (cls !== null && !cls.equals(Java.use("java.lang.Object").class)) {
        var fields = cls.getDeclaredFields();
        if (fields === null || fields.length === 0) {
            cls = cls.getSuperclass();
            continue;
        }

        if (!cls.equals(obj.getClass())) {
            console.log("Dump super class  " + cls.getName() + " fields:");
        }

        for (var i = 0; i < fields.length; i++) {
            var field = fields[i];
            field.setAccessible(true);
            var name = field.getName();
            var value = field.get(obj);
            var type = field.getType();
            console.log(type + " " + name + "=" + value);
        }

        cls = cls.getSuperclass();
    }
}

```

### 2. 获取成员变量的值

```
function getFieldValue(obj, fieldName) {
    var cls = obj.getClass();
    var field = cls.getDeclaredField(fieldName);
    field.setAccessible(true);
    var name = field.getName();
    var value = field.get(obj);
    // console.log("field: " + field + "\tname:" + name + "\tvalue:" + value);
    return value;
}

```

### 3. 打印调用堆栈

```
function printStack() {
    Java.perform(function() {
        var Exception = Java.use("java.lang.Exception");
        var ins = Exception.$new("Exception");
        var straces = ins.getStackTrace();
        if (straces != undefined && straces != null) {
            var strace = straces.toString();
            var replaceStr = strace.replace(/,/g, "\r\n");
            console.log(
                "============================= Stack start ======================="
            );
            console.log(replaceStr);
            console.log(
                "============================= Stack end =======================\r\n"
            );
            Exception.$dispose();
        }
    });
}

```

堆栈调用顺序为自下而上

java 层 hook
-----------

### 1. hook 模板

```
# -*- coding: utf-8 -*-
import frida
import sys

hook_code = """
Java.perform(function(){

    var utils = Java.use("类名路径");
    utils.方法名.implementation = function(a, b){

        return retval;
    }
});
"""
process = frida.get_usb_device().attach('包名')
script = process.create_script(hook_code)
script.load()
sys.stdin.read()

```

### 2. hook 方法（非重载方法不用写方法类型）

```
var utils = Java.use("类名路径");
    utils.方法名.implementation = function(a, b){

    a = 123;
    b = 456;

    var retval = this.方法名(a, b);
    console.log(a, b, retval);

    return retval;
}

```

### 3. hook 重载方法

```
var utils = Java.use("类名路径");
utils.方法名.overload("方法类型").implementation = function(a, b){

    a = 123;
    b = 456;

    var retval = this.方法名(a, b);
    console.log(a, b, retval);

    return retval;
}

```

### 4. hook 所有重载方法

```
var utils = Java.use("类名路径");
    //console.log(utils.方法名.overloads.length);
    for(var i = 0; i < utils.方法名.overloads.length; i++){
        utils.方法名.overloads[i].implementation = function(){
            //console.log(JSON.stringify(arguments));

            if(arguments.length == 0){
                return "调用了没有参数的";
            }else if(arguments.length == 1){
                if(JSON.stringify(arguments).indexOf("Money") != -1){
                    return "调用了Money参数的";
                }else{
                    return "调用了int参数的";
                }
            }

            arguments[0] = 1000;
            return this.方法名.apply(this, arguments);
        }
    }

```

### 5. hook 构造方法

```
Java.perform(function hookTest4(){
    var money = Java.use("类名路径");
    money.$init.overload('重载的参数1', '重载的参数2').implementation = function(str, num){
        console.log(str, num);
        str = "欧元";
        num = 2000;
        this.$init(str, num);
    }
});

```

### 6. 对象实例化

```
Java.perform(function hookTest2(){
    var utils = Java.use("utils类路径");
    var money = Java.use("money类路径");
    utils.方法名.overload('重载参数').implementation = function(a){
        a = 888;
        var retval = this.方法名(money.$new("日元", 100000));//对象实例化
        console.log(a, retval);
        return retval;
    }
});

```

### 7. 修改类的字段

```
Java.perform(function(){
    //静态字段的修改
    var money = Java.use("money类路径");
    //console.log(JSON.stringify(money.字段名));
    money.字段名.value = "xxxxx";
    console.log(money.flag.value);

    //非静态字段的修改
    Java.choose("money类路径", {
        onMatch: function(obj){
            obj._name.value = "ouyuan"; //字段名与函数名相同，前面加个下划线
            obj.name.value = "ouyuan"; //字段名与函数名不同
        },
        onComplete: function(){
        }
    });
});

```

也可以通过 java 的反射方式修改

```
function setFieldValue(obj, fieldName, fieldValue) {
    var cls = obj.getClass();
    var field = cls.getDeclaredField(fieldName);
    field.setAccessible(true);
    field.set(obj, fieldValue);
}

```

### 8. hook 内部类与匿名类

```
hook_code = """
Java.perform(function hookTest6(){
    Java.perform(function(){
        var innerClass = Java.use("money类路径$内部类名");
        console.log(innerClass);
        innerClass.$init.implementation = function(a, b){
            a = "nb";
            b = 888;
            return this.$init(a, b);
        }

        var xxx = Java.use("xxx类路径$smali中查看匿名类数字编号");
        console.log(xxx);
        xxx.getInfo.implementation = function(){
            return "匿名类被Hook了"
        }
    });
});
"""

```

### 9. hook 类的所有方法

```
hook_code = """
Java.perform(function hookTest8(){
    Java.perform(function(){
        var md5 = Java.use("md5类路径");
        var methods = md5.class.getDeclaredMethods();
        for(var j = 0; j < methods.length; j++){
            var methodName = methods[j].getName();
            console.log(methodName);

            for(var k = 0; k < md5[methodName].overloads.length; k++){

                md5[methodName].overloads[k].implementation = function(){
                    for(var i = 0; i < arguments.length; i++){
                        console.log(arguments[i]);
                    }
                    return this[methodName].apply(this, arguments);
                }
            }
        }
    });
});
"""

```

### 10. Hook 动态加载的 dex(Android 7 以上)

```
hook_code = """
Java.perform(function () {
    Java.enumerateClassLoaders({
        onMatch: function (loader) {
            try {
                if (loader.loadClass("com.xiaojianbang.app.Dynamic")) {
                    Java.classFactory.loader = loader;
                    var Dynamic = Java.use("com.xiaojianbang.app.Dynamic");
                    console.log(Dynamic);
                    Dynamic.sayHello.implementation = function () {
                        return "9999999";
                    }
                }
            } catch (error) {
            }
        },
        onComplete: function () {
        }
    });
});
"""

```

### 11. Java 特殊类型的遍历与修改（Map 举例）

```
hook_code = """
Java.perform(function () {
    var ShufferMap = Java.use("com.xiaojianbang.app.ShufferMap");
    console.log(ShufferMap);
    ShufferMap.show.implementation = function (map) {
        console.log(JSON.stringify(map));
        //Java map的遍历
        var key = map.keySet();
        var it = key.iterator();
        var result = "";
        while(it.hasNext()){
            var keystr = it.next();
            var valuestr = map.get(keystr);
            result += valuestr;
        }
        console.log(result);
        // return result;

        //Java map的修改
        map.put("pass", "zygx8");
        map.put("guanwang", "www.zygx8.com");

        var retval = this.show(map);
        console.log(retval);
        return retval;
    }
});
"""

```

### 12. 打印 HashMap

```
console.log(JSON.stringify(arguments))
var Map = Java.use('java.util.HashMap');
var args_map = Java.cast(arguments[1], Map);
console.log(args_map.toString());

```

### 13. Java 层主动调用

```
hook_code = """
Java.perform(function () {
    //静态方法的主动调用
    var rsa = Java.use("com.xiaojianbang.app.RSA");
    var str = Java.use("java.lang.String");
    var base64 = Java.use("android.util.Base64");
    var bytes = str.$new("xiaojianbang").getBytes();
    console.log(JSON.stringify(bytes));
    var retval = rsa.encrypt(bytes);
    var result = base64.encodeToString(retval, 0);
    console.log(result);
    //非静态方法的主动调用1 (新建一个对象去调用)
    var res = Java.use("com.xiaojianbang.app.Money").$new("日元", 300000).getInfo();
    console.log(res);
    var utils = Java.use("com.xiaojianbang.app.Utils");
    res = utils.$new().myPrint(["xiaojianbang", "is very good", " ", "zygx8", "is very good"]);
    console.log(res);
    //非静态方法的主动调用2 (获取已有的对象调用)
    Java.choose("com.xiaojianbang.app.Money", {
        onMatch: function (obj) {
            if (obj._name.value == "美元") {
                res = obj.getInfo();
                console.log(res);
            }
        },
        onComplete: function () {
        }
    });
});
"""

```

### 14. 删除对象引用

```
$.dispose

```

### 15. 获取参数类型

```
xxx.class.getType()

```

### 16. 用 frida 注入 dex 文件

```
hook_code = """
Java.perform(function () {
    Java.openClassFile("/data/local/tmp/xiaojianbang.dex").load();
    var xiaojianbang = Java.use("com.xiaojianbang.test.xiaojianbang");

    var ShufferMap = Java.use("com.xiaojianbang.app.ShufferMap");
    ShufferMap.show.implementation = function (map) {
        var retval = xiaojianbang.sayHello(map);
        console.log(retval);
        return retval;
    }

});
"""

```

### 17. 端口检测解决方案

```
./data/local/tmp/frida_server_arm64 -l 127.0.0.1:9999

adb forward tcp:9999 tcp:9999

```

### 18. frida 启动前注入

frida -H 127.0.0.1:9999 -f com.xjb.cpp -l hook.js --no-pause

so 层 hook
---------

### 1. 枚举导入导出表 (ELF 即 so 文件)

### 1. 枚举导出表

```
var imports = Module.enumerateImports("libxiaojianbang.so");
for(var i = 0; i < imports.length; i++){
    if(imports[i].name == "strncat"){
        console.log(JSON.stringify(imports[i]));
        console.log(imports[i].address);
    }
}

```

### 2. 枚举导出表

```
var exports = Module.enumerateExports("libxiaojianbang.so");
for(var i = 0; i < exports.length; i++){
    //if(exports[i].name == "strncat"){
        console.log(JSON.stringify(exports[i]));
    //}
}

```

### 2. hook 导出函数

```
hook_code = """
Java.perform(function hookTest2() {
    var helloAddr = Module.findExportByName("libxiaojianbang.so", "Java_com_xiaojianbang_app_NativeHelper_add");
    console.log(helloAddr);
    if (helloAddr != null) {
        Interceptor.attach(helloAddr, {
            onEnter: function (args) {
                //args参数数组
                console.log(args[0]);
                console.log(args[1]);
                console.log(args[2]);
                console.log(args[3]);
                console.log(args[4].toInt32());
            },
            onLeave: function (retval) {
                //retval函数返回值
                console.log(retval);
                console.log("retval", retval.toInt32());
            }
        });
    }
});
"""

```

### 3. 函数地址计算

```
function hookTest14(){
    var soAddr = Module.findBaseAddress("libxiaojianbang.so");
    console.log(soAddr);
    var funcAddr = soAddr.add(0x23F4);
    console.log(funcAddr);
}

```

### 4. Hook 未导出函数

```
function hookTest14(){
    var soAddr = Module.findBaseAddress("libxiaojianbang.so");
    console.log(soAddr);
    var funcAddr = soAddr.add(0x23F4);
    console.log(funcAddr);

    if(funcAddr != null){
        Interceptor.attach(funcAddr,{
            onEnter: function(args){

            },
            onLeave: function(retval){
                console.log(hexdump(retval));
            }
        });
     }
}

```

### 5. 获取指针参数返回值

```
function hookTest5(){
    var soAddr = Module.findBaseAddress("libxiaojianbang.so");
    console.log(soAddr);
    var sub_930 = soAddr.add(0x930); //函数地址计算 thumb+1 ARM不加
    console.log(sub_930);

     var sub_208C = soAddr.add(0x208C); //函数地址计算 thumb+1 ARM不加
     console.log(sub_208C);
     if(sub_208C != null){
        Interceptor.attach(sub_208C,{
            onEnter: function(args){
                this.args1 = args[1];
            },
            onLeave: function(retval){
                console.log(hexdump(this.args1));
            }
        });
     }
}

```

### 6. Hook_dlopen

```
function hookTest6(){
    var dlopen = Module.findExportByName(null, "dlopen");
    console.log(dlopen);
    if(dlopen != null){
        Interceptor.attach(dlopen,{
            onEnter: function(args){
                var soName = args[0].readCString();
                console.log(soName);
                if(soName.indexOf("libxiaojianbang.so") != -1){
                    this.hook = true;
                }
            },
            onLeave: function(retval){
                if(this.hook) { hookTest5() };
            }
        });
    }

    var android_dlopen_ext = Module.findExportByName(null, "android_dlopen_ext");
    console.log(android_dlopen_ext);
    if(android_dlopen_ext != null){
        Interceptor.attach(android_dlopen_ext,{
            onEnter: function(args){
                var soName = args[0].readCString();
                console.log(soName);
                if(soName.indexOf("libxiaojianbang.so") != -1){
                    this.hook = true;
                }
            },
            onLeave: function(retval){
                if(this.hook) { hookTest5() };
            }
        });
    }

}

```

### 7. 内存读写

```
function hookTest7(){
    var soAddr = Module.findBaseAddress("libxiaojianbang.so");
    console.log(soAddr);
    if(soAddr != null){
        //console.log(soAddr.add(0x2C00).readCString());
        //console.log(hexdump(soAddr.add(0x2C00)));  //读取指定地址的字符串

        //var strByte = soAddr.add(0x2C00).readByteArray(16); //读内存
        //console.log(strByte);

        //soAddr.add(0x2C00).writeByteArray(stringToBytes("xiaojianbang")); //写内存
        //console.log(hexdump(soAddr.add(0x2C00)));  //dump指定内存

        //var bytes = Module.readByteArray(soAddr.add(0x2C00), 16);
        //console.log(bytes);

    }
}

```

### 8. 主动调用 JNI 函数

```
function hookTest8(){
    var funcAddr = Module.findExportByName("libxiaojianbang.so", "Java_com_xiaojianbang_app_NativeHelper_helloFromC");
    console.log(funcAddr);
    if(funcAddr != null){
        Interceptor.attach(funcAddr,{
            onEnter: function(args){

            },
            onLeave: function(retval){
                var env = Java.vm.tryGetEnv();
                var jstr = env.newStringUtf("www.zygx8.com");  //主动调用jni函数 cstr转jstr
                retval.replace(jstr);
                var cstr = env.getStringUtfChars(jstr); //主动调用 jstr转cstr
                console.log(cstr.readCString());
                console.log(hexdump(cstr));
            }
        });
    }
}

```

### 9. jni 函数 Hook(计算地址方式)

```
function hookTest9(){
    Java.perform(function(){
        //console.log(JSON.stringify(Java.vm.tryGetEnv()));
        var envAddr = ptr(Java.vm.tryGetEnv().handle).readPointer();
        var newStringUtfAddr = envAddr.add(0x538).readPointer();
        var registerNativesAddr = envAddr.add(1720).readPointer();
        console.log("newStringUtfAddr", newStringUtfAddr);
        console.log("registerNativesAddr", registerNativesAddr)
        if(newStringUtfAddr != null){
            Interceptor.attach(newStringUtfAddr,{
                onEnter: function(args){
                    console.log(args[1].readCString());
                    //args[1] = "xiaojianbang is very good!";
                },
                onLeave: function(retval){

                }
            });
        }
        if(registerNativesAddr != null){     //Hook registerNatives获取动态注册的函数地址
            Interceptor.attach(registerNativesAddr,{
                onEnter: function(args){
                    console.log(args[2].readPointer().readCString());
                    console.log(args[2].add(Process.pointerSize).readPointer().readCString());
                    console.log(args[2].add(Process.pointerSize * 2).readPointer());
                    console.log(hexdump(args[2]));
                    console.log("sub_289C", Module.findBaseAddress("libxiaojianbang.so").add(0x289C));
                },
                onLeave: function(retval){

                }
            });
        }

    });
}

```

### 10. jni 函数 Hook(libart.so)

```
function hookTest10(){
    var artSym = Module.enumerateSymbols("libart.so");
    var NewStringUTFAddr = null;
    for(var i = 0; i < artSym.length; i++){
        if(artSym[i].name.indexOf("CheckJNI") == -1 && artSym[i].name.indexOf("NewStringUTF") != -1){
            console.log(JSON.stringify(artSym[i]));
            NewStringUTFAddr = artSym[i].address;
        }
    };

    if(NewStringUTFAddr != null){
        Interceptor.attach(NewStringUTFAddr,{
            onEnter: function(args){
                console.log(args[1].readCString());
            },
            onLeave: function(retval){

            }
        });
    }

}

```

### 11. so 层函数主动调用

```
function hookTest11(){
    Java.perform(function(){
        var funcAddr = Module.findBaseAddress("libxiaojianbang.so").add(0x23F4);
        var func = new NativeFunction(funcAddr, "pointer", ['pointer', 'pointer']);
        var env = Java.vm.tryGetEnv();
        console.log("env: ", JSON.stringify(env));
        if(env != null){
            var jstr = env.newStringUtf("xiaojianbang is very good!!!");
            //console.log("jstr: ", hexdump(jstr));
            var cstr = func(env, jstr);
            console.log(cstr.readCString());
            console.log(hexdump(cstr));
        }
    });
}

```

### 12. frida 读写文件

```
//frida API 读写文件
function hookTest12(){
    var ios = new File("/sdcard/xiaojianbang.txt", "w");
    ios.write("xiaojianbang is very good!!!\n");
    ios.flush();
    ios.close();
}
//Hook libc 读写文件
function hookTest13() {

    var addr_fopen = Module.findExportByName("libc.so", "fopen");
    var addr_fputs = Module.findExportByName("libc.so", "fputs");
    var addr_fclose = Module.findExportByName("libc.so", "fclose");

    console.log("addr_fopen:", addr_fopen, "addr_fputs:", addr_fputs, "addr_fclose:", addr_fclose);
    var fopen = new NativeFunction(addr_fopen, "pointer", ["pointer", "pointer"]);
    var fputs = new NativeFunction(addr_fputs, "int", ["pointer", "pointer"]);
    var fclose = new NativeFunction(addr_fclose, "int", ["pointer"]);

    var filename = Memory.allocUtf8String("/sdcard/xiaojianbang.txt");
    var open_mode = Memory.allocUtf8String("w");
    var file = fopen(filename, open_mode);
    console.log("fopen:", file);

    var buffer = Memory.allocUtf8String("zygxb\n");
    var retval = fputs(buffer, file);
    console.log("fputs:", retval);

    fclose(file);

}

```

RPC
---

```
import frida
import sys

rdev = frida.get_usb_device()
session = rdev.attach("com.yuanrenxue.onlinejudge2020")  # 包名

js_code = """
rpc.exports = {
    getsign: function(i) {
        Java.perform(function() {
            console.log("get_sign");
            var my_class1 = Java.use("com.yuanrenxue.onlinejudge2020.OnlineJudgeApp");
            var reslut = my_class1.getSign1(i);
            console.log(reslut);
            send({ "sign": reslut, "num": i })
            return reslut;
        });
    },
};

"""

script = session.create_script(js_code)


def on_message(message, data):
    sign = message.get("payload").get("sign")
    num = message.get("payload").get("num")


script.on("message", on_message)
script.load()

script.exports.getsign(1) # 调用的函数

sys.stdin.read()

```

通过 wifiadb 实现群控
---------------

```
import frida
import os
import time

app = "com.xxx.xxx";

device_ids = [];
devices = frida.enumerate_devices();
for device in devices :
    # print(device)
    ## 枚举所有通过wifiadb 连的机器
    if device.id.find(":") > 0:
        #print(device.id)
        device_ids.append(device.id.replace("5555", "9999"))

for id in device_ids :
    print(id)
    device = frida.get_device_manager().add_remote_device(id)
    print(device)
    pid = device.spawn([app])
    print(pid)
    device.resume(pid)
    time.sleep(1)
    session = device.attach(pid)
    with open("load_hook.js") as f:
        script = session.create_script(f.read())
        script.load()
input()

```

常用命令
----

### 1. 列出正在运行的进程：

```
frida-ps -U

```

### 2. 列出安装的程序

```
frida-ps -Uai

```

### 3. 列出运行中的程序（查看包名很方便）

```
frida-ps -Ua

```

### 4. 连接 frida 到一个指定的设备上

```
frida-ps -D 设备id

```

另外还有四个分别是： frida-trace, frida-discover, frida-ls-devices, frida-kill

objection 使用
------------

Frida 只是提供了各种 API 供我们调用，在此基础之上可以实现具体的功能，比如禁用证书绑定之类的脚本，就是使用 Frida 的各种 API 来组合编写而成。于是有大佬将各种常见、常用的功能整合进一个工具，供我们直接在命令行中使用，这个工具便是 objection。

安装
--

```
pip3 install objection

```

使用
--

### 1. 连接 app

```
objection -g 包名 explore

```

### 2. Memory 指令

```
memory list modules  // 查看内存中加载的库
memory list exports libssl.so  // 查看库的导出函数
memory list exports libart.so --json /root/libart.json //将结果保存到json文件中
memory search --string --offsets-only //搜索内存
memory search "64 65 78 0a 30 35 00"

```

### 3. root

```
android root disable //尝试关闭app的root检测
android root simulate //尝试模拟root环境

```

### 4. activities

```
android hooking list activities // 可以列出app具有的所有avtivity

```

### 5. 内存漫游

```
//列出内存中所有的类
android hooking list classes

//在内存中所有已加载的类中搜索包含特定关键词的类
android hooking search classes [search_name]

//在内存中所有已加载的方法中搜索包含特定关键词的方法
android hooking search methods [search_name]

//直接生成hook代码
android hooking generate simple [class_name]

// 查看类的全部广法
android hooking list class_methods [class_name]

```

### 6. hook 方式

hook 指定方法, 如果有重载会 hook 所有重载, 如果有疑问可以看 --dump-args : 打印参数 --dump-backtrace : 打印调用栈 --dump-return : 打印返回值

```
// 查看方法的参数、返回值和调用栈
android hooking watch class_method com.xxx.xxx.methodName --dump-args --dump-backtrace --dump-return

//获取全部tostring的返回来值
android hooking watch class_method java.lang.StringBuilder.toString --dump-return

//弹窗
android hooking watch class_method android.app.Dialog.show --dump-args --dump-backtrace --dump-return

//hook指定类, 会打印该类下的所有调用 
android hooking watch class com.xxx.xxx

//设置返回值(只支持bool类型) 
android hooking set return_value com.xxx.xxx.methodName false

```

### 7. 关闭 app 的 ssl 校验

android sslpinning disable

### 8. Spawn 方式 Hook

从 Objection 的使用操作中我们可以发现，Obejction 采用 Attach 附加模式进行 Hook，这可能会让我们错过较早的 Hook 时机，可以通过如下的代码启动 Objection，引号中的 objection 命令会在启动时就注入 App。

```
objection -g packageName explore --startup-command 'android hooking watch xxx'

```

ARIDA
-----

管理 PRC 脚本，自动生成 http 接口的工具

### 1. 安装

下载

```
git clone git@github.com:lateautumn4lin/arida.git

```

使用 conda 安装

```
conda create -n arida python==3.8
conda install --yes --file requirements.txt

```

使用 pip 安装

```
virtualenv venv
source venv/bin/activate

// 下载pip安装格式的requirements.txt https://github.com/Boris-code/arida/blob/master/requirements.txt

pip install -r requirements.txt

```

### 2. 运行

```
uvicorn main:app --reload

watch 127.0.0.1:8000/docs

```

![](https://pic4.zhimg.com/v2-49fab27275c50ec20c25b1c477646e2b_r.jpg)

### 3. 开发

Config 文件中写入自己的 App 信息

apps 目录写开发相应的 Frida-Js 脚本，可参考其他两个文件

参考：
---

1.  Objection 的安装和简单使用：[https://www.cnblogs.com/qiaorui/p/13455420.html](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/qiaorui/p/13455420.html)  
    
2.  Firda API 示例：[https://juejin.cn/post/6844904127705661448#heading-9](https://link.zhihu.com/?target=https%3A//juejin.cn/post/6844904127705661448%23heading-9)  
    
3.  ARIDA: [https://github.com/lateautumn4lin/arida](https://link.zhihu.com/?target=https%3A//github.com/lateautumn4lin/arida)

了解更多
----

欢迎加入知识星球 [https://t.zsxq.com/eEmAeae](https://link.zhihu.com/?target=https%3A//t.zsxq.com/eEmAeae)

![](https://pic2.zhimg.com/v2-ab533499f9975a095c6ceaba326d3f5d_b.jpg)

本星球专注于爬虫技术分享，通过一些案例详细讲解爬虫中遇到的问题以及解决手段。涉及的知识包括但不限于 爬虫框架刨析、js 逆向、中间人、selenium 、pyppeteer、Android 逆向！期待您的加入，和我们一起探讨爬虫技术，拓展爬虫思维！

QQ 群：750614606

![](https://pic1.zhimg.com/v2-4f8128fd816928f34935e8540e247af0_b.jpg)
