> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_44675969/article/details/123532633)

JS 逆向之浏览器补环境 (一)
================

简介
--

今天分享是是浏览器环境检测以及 [node](https://so.csdn.net/so/search?q=node&spm=1001.2101.3001.7020) js 补环境

![](https://img-blog.csdnimg.cn/img_convert/f68b435cd903fa8458924c5810f570ea.png)

我们点击检测设备信息看发出的请求，可以看到是 sign 和 data

![](https://img-blog.csdnimg.cn/img_convert/810f5097454f52a491ba7290c44aa4f0.png)

这个 data 看起来像 [base64](https://so.csdn.net/so/search?q=base64&spm=1001.2101.3001.7020), 我们试一下，发现不是

![](https://img-blog.csdnimg.cn/img_convert/36e68ef3372ae9054df1a3174ccdfab2.png)

分析[加密](https://so.csdn.net/so/search?q=%E5%8A%A0%E5%AF%86&spm=1001.2101.3001.7020)位置
------------------------------------------------------------------------------------

![](https://img-blog.csdnimg.cn/img_convert/82eb7f8b1e7b25e341da3f2971fa27e7.png)

可以看到经过混淆了

![](https://img-blog.csdnimg.cn/img_convert/53eb70e1dff5e68383ce1bafece8afb9.png)

我们选中看一下

![](https://img-blog.csdnimg.cn/img_convert/5377fe2fd2458213208af8e76ddbde29.png)

![](https://img-blog.csdnimg.cn/img_convert/d560d9dd5f783010c1a9db61c2fbe391.png)

那我们接下来就知道了，要逆向的两个参数变量

```
_0x392f2d
_0xd9e63f
```

我们直接把整个文件拷贝到 webstrom 里，看一下这两个变量在哪里定义的，可以看到在这

![](https://img-blog.csdnimg.cn/img_convert/9675147615aaf88a2486397cbd24c93f.png)

由于我们只需要这两个变量，所以下面的代码可以删除了，变成这样

![](https://img-blog.csdnimg.cn/img_convert/8ccdf9eb7727379cff78c16d4e34e5ab.png)

追踪检测
----

然后引入 proxy 检测对象变化的代码

**proxy.js**

```
let _window = {
    
};
let _stringify = JSON.stringify;
JSON.stringify = function (Object) {
    // ?? 的意思是，如果 ?? 左边的值是 null 或者 undefined，那么就返回右边的值。
    if ((Object?.value ?? Object) === global) {
        return "global";
    }
    return _stringify(Object);
};

function getMethodHandler(WatchName) {
    let methodhandler = {
        apply(target, thisArg, argArray) {
            let result = Reflect.apply(target, thisArg, argArray);
            console.log(`[${WatchName}] apply function name is [${target.name}], argArray is [${argArray}], result is [${result}].`);
            return result;
        },
        construct(target, argArray, newTarget) {
            let result = Reflect.construct(target, argArray, newTarget);
            console.log(`[${WatchName}] construct function name is [${target.name}], argArray is [${argArray}], result is [${JSON.stringify(result)}].`);
            return result;
        }
    };
    return methodhandler;
}

function getObjHandler(WatchName) {
    let handler = {
        get(target, propKey, receiver) {
            let result = Reflect.get(target, propKey, receiver);
            if (result instanceof Object) {
                if (typeof result === "function") {
                    console.log(`[${WatchName}] getting propKey is [${propKey}] , it is function`);
                    //return new Proxy(result,getMethodHandler(WatchName))
                } else {
                    console.log(`[${WatchName}] getting propKey is [${propKey}], result is [${JSON.stringify(result)}]`);
                }
                return new Proxy(result, getObjHandler(`${WatchName}.${propKey}`));
            }
            console.log(`[${WatchName}] getting propKey is [${propKey}], result is [${result}]`);
            return result;
        },
        set(target, propKey, value, receiver) {
            if (value instanceof Object) {
                console.log(`[${WatchName}] setting propKey is [${propKey}], value is [${JSON.stringify(value)}]`);
            } else {
                console.log(`[${WatchName}] setting propKey is [${propKey}], value is [${value}]`);
            }
            return Reflect.set(target, propKey, value, receiver);
        },
        has(target, propKey) {
            let result = Reflect.has(target, propKey);
            console.log(`[${WatchName}] has propKey [${propKey}], result is [${result}]`);
            return result;
        },
        deleteProperty(target, propKey) {
            let result = Reflect.deleteProperty(target, propKey);
            console.log(`[${WatchName}] delete propKey [${propKey}], result is [${result}]`);
            return result;
        },
        getOwnPropertyDescriptor(target, propKey) {
            let result = Reflect.getOwnPropertyDescriptor(target, propKey);
            console.log(`[${WatchName}] getOwnPropertyDescriptor  propKey [${propKey}] result is [${JSON.stringify(result)}]`);
            return result;
        },
        defineProperty(target, propKey, attributes) {
            let result = Reflect.defineProperty(target, propKey, attributes);
            console.log(`[${WatchName}] defineProperty propKey [${propKey}] attributes is [${JSON.stringify(attributes)}], result is [${result}]`);
            return result;
        },
        getPrototypeOf(target) {
            let result = Reflect.getPrototypeOf(target);
            console.log(`[${WatchName}] getPrototypeOf result is [${JSON.stringify(result)}]`);
            return result;
        },
        setPrototypeOf(target, proto) {
            console.log(`[${WatchName}] setPrototypeOf proto is [${JSON.stringify(proto)}]`);
            return Reflect.setPrototypeOf(target, proto);
        },
        preventExtensions(target) {
            console.log(`[${WatchName}] preventExtensions`);
            return Reflect.preventExtensions(target);
        },
        isExtensible(target) {
            let result = Reflect.isExtensible(target);
            console.log(`[${WatchName}] isExtensible, result is [${result}]`);
            return result;
        },
        ownKeys(target) {
            let result = Reflect.ownKeys(target);
            console.log(`[${WatchName}] invoke ownKeys, result is [${JSON.stringify(result)}]`);
            return result;
        },
        apply(target, thisArg, argArray) {
            let result = Reflect.apply(target, thisArg, argArray);
            console.log(`[${WatchName}] apply function name is [${target.name}], argArray is [${argArray}], result is [${result}].`);
            return result;
        },
        construct(target, argArray, newTarget) {
            let result = Reflect.construct(target, argArray, newTarget);
            console.log(`[${WatchName}] construct function name is [${target.name}], argArray is [${argArray}], result is [${JSON.stringify(result)}].`);
            return result;
        }
    };
    return handler;
}
_window = Object.assign(global, _window);
const window = new Proxy(Object.create(_window), getObjHandler("window"));
module.exports = {
    window
};
```

在拷贝到 webstrom 里的代码进行导入

```
const {window} = require("../tools/proxy");
```

然后我们运行之前拷贝到 webstrom 里的代码

![](https://img-blog.csdnimg.cn/img_convert/1b133c0aa8a9350c7a551681f09acd54.png)

发现在 window 上需要定义一个 XMLHttpRequest，那是定义在 window 上呢，还是定义在它的原型链上呢，我们可以浏览器里看一下

第一个是检查对象自身的属性，第二是检查所有属性。因此我们需要定义在它的原型上

![](https://img-blog.csdnimg.cn/img_convert/9c2cd4844f82ebc8b16310eccaec016c.png)

修改追踪的代码

![](https://img-blog.csdnimg.cn/img_convert/e609ae756d7c1fe9e418423cc3c0d052.png)

然后我们打印看一下，发现也没有问题

![](https://img-blog.csdnimg.cn/img_convert/c78bd42f50cfcf00b4c462b0d47891e0.png)

我们再运行以下之前的代码，发现需要 navigator

![](https://img-blog.csdnimg.cn/img_convert/fbbc907e0ddcc64523b2dbe5bd408d90.png)

那我们也补上

```
let _window = {
    XMLHttpRequest: function (){},
};
let _navigator = {

};
_window = Object.assign(global, _window);
const window = new Proxy(Object.create(_window), getObjHandler("window"));
const navigator = new Proxy(Object.create(_navigator), getObjHandler("navigator"));
module.exports = {
    window,
    navigator,
};
```

![](https://img-blog.csdnimg.cn/img_convert/18261d472e3a6c7b6e166ddc6e964f9a.png)

然后又要这个，同理

![](https://img-blog.csdnimg.cn/img_convert/d683e58268607fb31d8a7a5a1412bc08.png)

![](https://img-blog.csdnimg.cn/img_convert/ae96d986e1c5be1811ba6c948d53a93a.png)

![](https://img-blog.csdnimg.cn/img_convert/9214f6f73ab3ca018f23d9aeb4945d07.png)

发现没有问题了，也拿到了 sign 和 data

![](https://img-blog.csdnimg.cn/img_convert/1989ea2d002db6637251e252b82293f2.png)

python 调用
---------

然后我们通过 python 试一下

![](https://img-blog.csdnimg.cn/img_convert/f6bde9f0b57ed604e43af731f010936b.png)

然后到在线工具里转一下

https://tool.lu/curl/

![](https://img-blog.csdnimg.cn/img_convert/c9cf8de978c4e97205c5e9e5579ff7e6.png)

发现直接被检测到了，就是我们的环境检测没有通过

![](https://img-blog.csdnimg.cn/img_convert/722c3c35bd59c27d3705cf36c7004ac8.png)

补环境
---

我们可以看到有一堆 undefined，这些都是我们需要补的环境

![](https://img-blog.csdnimg.cn/img_convert/f47a0d3a6645f37c2742e6243ed40712.png)

然后我们要看一下这个值在浏览器里是怎么样的，并且是定义在对象上还是原型上，比如 navigator 来说，发现是定义在原型上的。

![](https://img-blog.csdnimg.cn/img_convert/ab4fb29c81fd383b6ade63caf4247fa4.png)

然后就照着补就行了

注意这个，getOwnPropertyDescriptor 是拿的自身属性上的

![](https://img-blog.csdnimg.cn/img_convert/e3fcec20dab2549f5be81f09a537974a.png)

最后补的结果

```
let _window = {
    XMLHttpRequest: function (){},
    sessionStorage: function (){},
    localStorage: function (){},
};
let _navigator = {
    userAgent: "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.51 Safari/537.36 Edg/99.0.1150.39",
    appCodeName: "Mozilla",
    plugins: {
        length: 0
    },
};
let _screen = {
    colorDepth: 24,
    height: 900,
    width: 1440,
};
_window = Object.assign(global, _window);
const window = new Proxy(Object.create(_window), getObjHandler("window"));
const navigator = new Proxy(Object.create(_navigator), getObjHandler("navigator"));
const screen = new Proxy(Object.create(_screen), getObjHandler("screen"));
module.exports = {
    window,
    navigator,
    screen,
};
```

再跑一次，拿着 data 和 sign 去请求，通过！

![](https://img-blog.csdnimg.cn/img_convert/dde9c74145ed46beb09b837b8a95270f.png)

另一种操作 (不推荐)
-----------

使用 JSDOM 自带的浏览器环境

安装

```
npm install jsdom
```

使用

```
const jsdom = require("jsdom");
const {JSDOM} = jsdom;


const {window} = new JSDOM(``);
const navigator = window.navigator;
const screen = window.screen;
// 这个是缺啥补啥 基本都齐了
window.XMLHttpRequest = function XMLHttpRequest() {
};
module.exports = {
    window,
    navigator,
    screen
};
```

导入

![](https://img-blog.csdnimg.cn/img_convert/533b7cf293d0c11a6f4e8a4bef16f5f7.png)

但是会出现各式各样的问题，不推荐

![](https://img-blog.csdnimg.cn/img_convert/9146feda16ec4da71b6c5eada9620aaa.png)