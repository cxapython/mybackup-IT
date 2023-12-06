> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/xiaoweigege/p/14999469.html)

Frida Hook So 一些操作说明[#](#frida-hook-so-一些操作说明)
----------------------------------------------

### Native 方法第一个参数是 JNIEnv *env 如何在 Frida 中获取 JNIEnv 对象呢?[#](#native方法第一个参数是-jnienv-env-如何在frida中获取-jnienv-对象呢)

```
Java.vm.getEnv();
```

### 如何将 string 类型转换 jstring 类型呢?[#](#如何将string类型转换jstring类型呢)

```
let jstring = Java.vm.getEnv().newStringUtf(str);
```

### 如何将 jstring 类型转 string 类型呢？[#](#如何将jstring类型转string类型呢？)

```
aes_value = Java.vm.getEnv().getStringUtfChars(result, null).readCString()
```

### Hook So 导出函数[#](#hook-so-导出函数)

```
let method1_addr = Module.findExportByName('libxiaowei.so', 'Java_com_example_xiaoweiso_MainActivity_method01');
```

### Hook So 非导出函数[#](#hook-so-非导出函数)

```
let so_addr = Module.findBaseAddress('libxiaowei.so');
// 需要去so中找到非导出函数的地址
let encrypt_addr = so_addr.add(0x42B0);
```

### 如何在 so 中定义一个字符串[#](#如何在so中定义一个字符串)

```
let cstring = Memory.allocUtf8String("xiaoweigege");
```

### 如何将 c 中的字符串转成 js string?[#](#如何将c中的字符串转成js-string)

```
ptr(result).readCString()
```

### 将函数地址定义成一个函数能在 js 中进行调用[#](#将函数地址定义成一个函数能在js中进行调用)

```
let so_addr = Module.findBaseAddress('libxiaowei.so');
let encrypt_addr = so_addr.add(0x42B0);
let encrypt_fun = new NativeFunction(encrypt_addr, 'pointer', ['pointer']);
let cstring = Memory.allocUtf8String(str);
let result = encrypt_fun(cstring);
```

### 在任意 apk 中加载 so 文件[#](#在任意apk中加载so文件)

```
var load_model = Module.load('/data/local/tmp/libxiaowei.so');
```

这样操作就可以在任意地方调用 so 中的方法，那么我们就可以脱离 apk 比如 hook 系统中 设置 来调用我们的 so 文件方法，达到脱离本身 APK

使用示例[#](#使用示例)
--------------

```
var load_model = Module.load('/data/local/tmp/libxiaowei.so');

function hook_method1(str) {

    let method1_addr = Module.findExportByName('libxiaowei.so', 'Java_com_example_xiaoweiso_MainActivity_method01');

    let method1_fun = new NativeFunction(method1_addr, 'pointer', ['pointer', 'pointer', 'pointer']);
    let aes_value = null


    Java.perform(function () {
        // Java.vm.getEnv() JNIEnv 对象获取

        let jstring = Java.vm.getEnv().newStringUtf(str);
        let result = method1_fun(Java.vm.getEnv(), jstring, jstring);

        aes_value = Java.vm.getEnv().getStringUtfChars(result, null).readCString()
    })
    return aes_value;


}

function hook_method2(str) {

    let method1_addr = Module.findExportByName('libxiaowei.so', 'Java_com_example_xiaoweiso_MainActivity_method02');

    let method1_fun = new NativeFunction(method1_addr, 'pointer', ['pointer', 'pointer', 'pointer']);
    let aes_value = null


    Java.perform(function () {
        let jstring = Java.vm.getEnv().newStringUtf(str);
        let result = method1_fun(Java.vm.getEnv(), jstring, jstring);

        aes_value = Java.vm.getEnv().getStringUtfChars(result, null).readCString()
    })
    return aes_value;


}


function hook_encrypt(str) {
    let so_addr = Module.findBaseAddress('libxiaowei.so');
    let encrypt_addr = so_addr.add(0x42B0);
    let encrypt_fun = new NativeFunction(encrypt_addr, 'pointer', ['pointer']);
    let cstring = Memory.allocUtf8String(str);
    let result = encrypt_fun(cstring);

    console.log(ptr(result).readCString())

}

function hook_decrypt(str) {

    let so_addr = Module.findBaseAddress('libxiaowei.so');
    let encrypt_addr = so_addr.add(0x4538);
    let encrypt_fun = new NativeFunction(encrypt_addr, 'pointer', ['pointer']);
    let cstring = Memory.allocUtf8String(str);
    let result = encrypt_fun(cstring);

    console.log(ptr(result).readCString())
}


function main() {
    let value = hook_method1('xiaoweigege')
    hook_method2(value)
}


setImmediate(main)
```