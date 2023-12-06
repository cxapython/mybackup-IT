> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/xiaoweigege/p/14876089.html)

Frida Hook Native

Frida Hook Java Jni[#](#frida-hook-java-jni)
--------------------------------------------

demo:

```
function hook_java() {

    Java.perform(function () {

        const myapp = Java.use('com.gdufs.xman.MyApp');
        myapp.m.value = 1;
        console.log('m=', myapp.m.value);
        myapp.saveSN.implementation = function (s) {
            let result = this.saveSN(s);
            console.log('myapp.saveSN:', s, result);
            return result;
        }
    })

}
```

获取模块基址，Hook 导出函数[#](#获取模块基址，hook导出函数)
-------------------------------------

demo:

```
function hook_native() {

    // 获取模块基址
    const my_jni = Module.findBaseAddress('libmyjni.so');

    if (!my_jni) return;

    // 查找 so 文件的 导出函数 地址
    const addr_n2 = Module.findExportByName('libmyjni.so', 'n2');
    console.log('my_jni: ', my_jni, 'addr_n2:', addr_n2);
    // 对函数地址进行Hook
    Interceptor.attach(addr_n2, {
        onEnter: function (arg) {
            console.log('addr_n2 onEnter :', arg[0], ptr(arg[1]).readCString(), ptr(arg[2]).readCString())
        },
        onLeave: function (retval) {

        }
    })

}
```

枚举模块的符号 Hook libart 的一些函数[#](#枚举模块的符号---hook-libart的一些函数)
---------------------------------------------------------

demo:

```
function hook_art() {

    // 查找模块
    const lib_art = Process.findModuleByName('libart.so');

    // 枚举模块的符号
    const symbols = lib_art.enumerateSymbols();

    for (let symbol of symbols) {

        var name = symbol.name;

        if (name.indexOf("art") >= 0) {
            if ((name.indexOf("CheckJNI") == -1) && (name.indexOf("JNI") >= 0)) {
                if (name.indexOf("GetStringUTFChars") >= 0) {
                    console.log('开始 HOOK libart ', symbol.name);
                    Interceptor.attach(symbol.address, {
                        onEnter: function (arg) {
                            // 打印调用栈
                            // console.log('GetStringUTFChars called from:\n' + Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n') + '\n');
                        },
                        onLeave: function (retval) {
                            console.log('onLeave GetStringUTFChars:', ptr(retval).readCString())
                        }
                    })
                } else if (name.indexOf("FindClass") >= 0) {
                    console.log('开始 HOOK libart ', symbol.name);
                    Interceptor.attach(symbol.address, {
                        onEnter: function (arg) {
                            console.log('onEnter FindClass:', ptr(arg[1]).readCString())
                        },
                        onLeave: function (retval) {
                            // console.log('onLeave FindClass:', ptr(retval).readCString())
                        }
                
                    })
                } else if (name.indexOf("GetStaticFieldID") >= 0) {
                    console.log('开始 HOOK libart ', symbol.name);
                    Interceptor.attach(symbol.address, {
                        onEnter: function (arg) {
                            console.log('onEnter GetStaticFieldID:', ptr(arg[2]).readCString(), ptr(arg[3]).readCString())
                        },
                        onLeave: function (retval) {
                            // console.log('onLeave GetStaticFieldID:', ptr(retval).readCString())
                        }
                    })
                } else if (name.indexOf("SetStaticIntField") >= 0) {
                    console.log('开始 HOOK libart ', symbol.name);
                    Interceptor.attach(symbol.address, {
                        onEnter: function (arg) {
                            console.log('onEnter SetStaticIntField:', arg[3])
                        },
                        onLeave: function (retval) {
                            // console.log('onLeave SetStaticIntField:', ptr(retval).readCString())
                        }
                    })
                } 
 
            }
        }


    }


}
```

打印调用栈[#](#打印调用栈)
----------------

```
console.log('GetStringUTFChars called from:\n' + Thread.backtrace(this.context, Backtracer.ACCURATE).map(DebugSymbol.fromAddress).join('\n') + '\n');
```

Hook libc 的函数[#](#hook-libc的函数)
-------------------------------

demo:

```
function hook_libc() {

    const strcmp = Module.findExportByName('libc.so', 'strcmp');
    console.log('strcmp: ', strcmp);
    Interceptor.attach(strcmp, {
        onEnter: function (arg) {
            let str2 = ptr(arg[1]).readCString();
            if (str2 === 'EoPAoY62@ElRD') {
                console.log('strcmp:', ptr(arg[0]).readCString(), str2);
            }

        },
        onLeave: function (retval) {
        }
    })

}
```

Frida 的 File api 写文件[#](#frida-的-file-api-写文件)
----------------------------------------------

```
function write_data() {
    const file = new File('/sdcard/reg.dat', 'w');
    file.write('EoPAoY62@ElRD');
    file.flush();
    file.close()

}
```

把 C 函数定义为 NativaFunction 来写文件[#](#把c函数定义为nativafunction来写文件)
------------------------------------------------------------

demo:

```
// hook libc.so 的方式来写文件
function write_data_native() {
    // 读取lic的导出函数地址
    const addr_fopen = Module.findExportByName('libc.so', 'fopen');
    const addr_fputs = Module.findExportByName('libc.so', 'fputs');
    const addr_fclose = Module.findExportByName('libc.so', 'fclose');

    console.log('fopen:', addr_fopen, 'fputs', addr_fputs, 'fclose', addr_fclose);
    // 构建函数
    const fopen = new NativeFunction(addr_fopen, 'pointer', ['pointer', 'pointer']);
    const fputs = new NativeFunction(addr_fputs, 'int', ['pointer', 'pointer']);
    const fclose = new NativeFunction(addr_fclose, 'int', ['pointer']);

    // 申请内存空间
    let file_name = Memory.allocUtf8String('/sdcard/reg.dat');
    let model = Memory.allocUtf8String('w+');
    let data = Memory.allocUtf8String('EoPAoY62@ElRD');
    let file = fopen(file_name, model);
    let ret = fputs(data, file);
    console.log('fputs ret: ', ret);
    fclose(file);

}
```