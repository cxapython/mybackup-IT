> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/welggy/article/details/121744948)

篇幅有限
====

完整内容及源码关注公众号：ReverseCode，发送 **冲**

起因
==

最近对腾讯视频下手了，因为公众号中的视频来源都是腾讯视频，也就是说通过 2 分钟阅读本文，腾讯视频下整站视频都可以下载下来，也是大多数在线解析 vip 等视频网站底层实现逻辑，接下来就是建站接广告接流量接法院传单。。。。

分析
==

通过点击播放抓包发现不停的请求`http://btrace.video.qq.com/kvcollect`，点击发现没有发现，不过这些请求都是 m3u8 格式片段，我们大胆猜测这是 m3u8，将视频切成无数 ts 小段，一段一段加载播放，可以根据网络状况自动切换视频清晰度，保障流畅播放。那么这种播放方式前端一定有请求索引文件保存 ts 文件，保存了网络 url 链接，这些链接顺序播放就完成了整个视频的播放。而我们数据抓取只需要关注这个文件，m3u8 转为 mp4 格式，具体转换代码见 [https://github.com/OneJane/datautil](https://github.com/OneJane/datautil)

废话一堆，开始寻找 m3u8 请求，通过过滤筛选找到该请求后,`http://58.216.106.14/omts.tc.qq.com/AjzDO1DrTOFhSuI4sQa-qkOmtUv8yq9UrejaLeSKpF2M/uwMROfz2r57BIaQXGdGnC2deOm7WRbkbfdWCxMUsemsF2Gfz/svp_50001/NO3RnkZVfa4hKoQCijd2VpXELo5sw-cgX_CMZmI7XeU8ZfKGTirnxL1xJGXvDq2mliBQiL2MqcB6egr3lk7nk3wyBP18yb1lGlcVFNCkQ82kNml8GgGg4BbokC6yjxDcIJIVugQ8OkIG6GOCUijW9a3QpotWcmbZTrCI5kfpcYxax9isGSZL7Q/szg_3772_50001_0bc3hqaciaaameaajrug6rqvcpgdeq6aajca.f304110.ts.m3u8?ver=4`

![](https://img-blog.csdnimg.cn/2021120613322095.png)

这个请求除去域名外所有参数都加密了，随便找一段加密参数随便搜搜，找到了目标请求`http://vd.l.qq.com/proxyhttp`

![](https://img-blog.csdnimg.cn/20211206133220753.png)

通过 json 格式化`vinfo`参数可以判断`url+pt`即可拿到我们想要的 m3u8 文件

![](https://img-blog.csdnimg.cn/20211206133221394.png)

好戏开场了
-----

![](https://img-blog.csdnimg.cn/20211206133221980.png)

参数做减法，去除`adparam`和`vinfoparam`中的`logintoken`和`ehost`，重点关注`guid`，`flowid`，`cKey`，`tm`就是时间戳，`vid`就是请求中的视频 id

```
spsrt=1
charge=0
defaultfmt=auto
otype=ojson
guid=a66e56d20401a21c8a35e92ad94eebde
flowid=82a7dcecd17cede6b9dc02a39571c4bc_70201
platform=70201
sdtfrom=v1104
defnpayver=0
appVer=3.4.40
host=v.qq.com
refer=v.qq.com
sphttps=0
tm=1637204401
spwm=4
vid=s3306hychob
defn=
fhdswitch=0
show1080p=0
isHLS=1
dtype=3
sphls=1
spgzip=
dlver=
hdcp=0
spau=1
spaudio=15
defsrc=1
encryptVer=9.1
cKey=doItDoTdYRR79ZEItZs_lpJX5WFNi2CdS8kE1h7qVaqtHEZQ1c_X6m2O8hQenWPBG5hnGM2UODs52vPBr7VR-rE3OCFTLlH3-xN1QMZmGWCleJdQ62v6N6dvhRBy86U5pyEtRx0KHILNluNDEH6IC8EOljLQ2VfW2sTdospNPlD9535CNT9iSo3cLRH93ogtX_OJeYNVWrDYS8btjkFpGl3F3IxmISJc_8dRIBruTik-e4rt0isxZAXexKqWDJGxu2qxHq-QxHER_ek2fB1T6ywJriVO0ksOGo7_XQLdE-FshP9ARvdxQlEJPKWtziEF2xwGBgYGBgY0KhFT
```

打上 xhr 断点`vd.l.qq.com/proxyhttp`，在调用栈中`r.requestPostCgi`中参数已经生成，继续向上追溯

![](https://img-blog.csdnimg.cn/20211206133222708.png)

追溯到`requestNewGetinfoImpl`，该方法中`p.requestPostCgi`发起请求参数已经封装完成，手动调用`this.getInfoConfig("vinfoad", i)`时 cKey 及`guid`，`flowid`都已经加密完成。

![](https://img-blog.csdnimg.cn/20211206133223467.png)

进入`getInfoConfig`

![](https://img-blog.csdnimg.cn/20211206133224149.png)

在如下这段代码中 guid 不存在时，`guid: this.context.dataset.guid`将通过`l.getUserId(this.context.config)`

```
this.context.dataset.guid || (this.context.config.guid = l.getUserId(this.context.config),
this.context.dataset.guid = this.context.config.guid);
```

由于`l = o(436)`，进入`436: function(e, t, o)`找到了`getUserId`，当从本地或者内存中取不到时调用`d.createGUID()`，也就是随机 32 位字符串。

```
getUserId: function(e) {
    var t = e.guid || d.getData(i.localStorageKey.userId);
    return e.guid || !d.browser.pcqqlive && !d.browser.macqqlive || (t = (t = d.getPcClientGuid()) || d.getData(i.localStorageKey.userId)),
    t || (t = d.createGUID(),
    d.setData(i.localStorageKey.userId, t)),
    t
},
createGUID: function(e) {
    e = e || 32;
    for (var t = "", o = 1; o <= e; o++) {
        t += Math.floor(16 * Math.random()).toString(16)
    }
    return t
},
```

至于`flowid`跟踪代码得知是通过随机 32 位字符串_70201

```
updatePlayerId: function() {
    this.context.dataset.playerId = l.createGUID(),
    this.context.dataset.flowid = this.context.dataset.playerId + "_" + this.context.dataset.platform
},
```

在执行`m(s)`前 s 中没有`cKey`的值，执行后`cKey`生成

![](https://img-blog.csdnimg.cn/20211206133224816.png)

cKey 逆向
-------

可以将整个 js 拷贝出来放到 vs code 中分析。

![](https://img-blog.csdnimg.cn/20211206133225389.png)

修改如下

```
var createGUID = function(e) {
    e = e || 32;
    for (var t = "", o = 1; o <= e; o++) {
        t += Math.floor(16 * Math.random()).toString(16)
    }
    return t
}
// n函数存在走 (e.encryptVer = "9.1",n(e.platform, e.appVer, e.vids || e.vid, "", e.guid, e.tm))
function m(e) {
    // var t = n ? (e.encryptVer = "9.1",
    // n(e.platform, e.appVer, e.vids || e.vid, "", e.guid, e.tm)) : (e.encryptVer = "8.1",
    // a(e.vids || e.vid, e.tm, e.appVer, e.guid, e.platform));
    var t = n("70201", "3.4.40", "s3306hychob", "", createGUID(), Date.parse(new Date()).toString().substr(0,10))
    return t
}
```

进入 n 函数

```
n = r.cwrap("getckey", "string", ["number", "string", "string", "string", "string", "number"]),
h.cwrap = function(e, t, o, i) {
    return function() {
        return u(e, t, o, arguments)
    }
}
```

![](https://img-blog.csdnimg.cn/20211206133225968.png)

进入 u 函数，其中 w 函数就是判断存在否则异常报错，Ge 根据值是否存在校验跑错，没有操作逻辑。

```
// function w(e, t) {
//     e || Ge("Assertion failed: " + t)
// }
// function Ge(t) {
//     h.onAbort && h.onAbort(t),
//     t = void 0 !== t ? (r(t),
//     y(t),
//     JSON.stringify(t)) : "",
//     v = !0,
//     0;
//     var o = "abort(" + t + ") at " + P();
//     throw $e && $e.forEach(function(e) {
//         o = e(o, t)
//     }),
//     o
//     return t
// }
function w(e, t) {
    return e
}
function u(e, t, o, i, n) {
    var r, a, s = (w(a = h["_" + (r = e)], "Cannot call unknown function " + r + ", make sure it is exported"),
    a), c = [], d = 0;
    if (w("array" !== t, 'Return type should not be "array".'),
    i)
        for (var l = 0; l < i.length; l++) {
            var u = x[o[l]];
            u ? (0 === d && (d = je()),
            c[l] = u(i[l])) : c[l] = i[l]
        }
    var f, p = s.apply(null, c);
    return f = p,
    p = "string" === t ? T(f) : "boolean" === t ? Boolean(f) : f,
    0 !== d && He(d),
    p
}
```

![](https://img-blog.csdnimg.cn/20211206133226609.png)

打印出`h["_" + (r = e)]`, 进入后调用了 wasm 中的函数，即 0005098e 中的函数，截止到目前为止如果不手动新建 wasm 对象，逆向 cKey 将无法继续进行下去

![](https://img-blog.csdnimg.cn/20211206133227255.png)

wasm
----

搜索 wasm 请求，下载 wasm 文件，该文件在 web 环境中作为体积小且加载快的二进制格式指令集合，我们不关心底层编译实现，直接通过 api 调用完成逆向分析。

![](https://img-blog.csdnimg.cn/20211206133227855.png)

```
const fs = require('fs');
var wasm_data = fs.readFileSync('./ckey.wasm')
var buffer = new Uint8Array(wasm_data);
var wasmobject = new WebAssembly.Instance(new WebAssembly.Module(buffer));
```

报错：`WebAssembly.Instance(): Imports argument must be present and must be an object`

```
var wasm_env = {
};
var wasmobject = new WebAssembly.Instance(new WebAssembly.Module(buffer), wasm_env);
```

报错：`Import #0 module="env" error: module is not an object or function`

重点关注`445: function(Ke, e, t)`的`return WebAssembly.instantiate(e, c)`

```
var s, c = {
    global: null,
    env: null,
    asm2wasm: g,
    parent: h
};
var g = {
    "f64-rem": function(e, t) {
        return e % t
    },
    debugger: function() {}
};
```

![](https://img-blog.csdnimg.cn/20211206133228471.png)

```
// 由于h的实现太过复杂，目前只用{}替代
var wasm_env = {
    global: {},
    env: {},
    asm2wasm: {
        "f64-rem": function(e, t) {
            return e % t
        },
        debugger: function() {}
    },
    parent: {}
};
```

报错：`Import #0 module="env" function="enlargeMemory" error: function import requires a callable`。在 h.asmLibraryArg 中查看环境变量信息，由于 Ge 函数只校验参数抛错，所以直接用空函数代替

```
enlargeMemory: K
function K() {
    G()
}
function G() {
    Ge("Cannot enlarge memory arrays. Either (1) compile with  -s TOTAL_MEMORY=X  with X higher than the current value " + Q + ", (2) compile with  -s ALLOW_MEMORY_GROWTH=1  which allows increasing the size at runtime, or (3) if you want malloc to return NULL (0) instead of this abort, compile with  -s ABORTING_MALLOC=0 ")
}
```

添加环境变量中的 env 参数 enlargeMemory

```
var fun_ = function () { };
var wasm_env = {
    global: {},
    env: {
        enlargeMemory: fun_,
    },
    asm2wasm: {
        "f64-rem": function(e, t) {
            return e % t
        },
        debugger: function() {}
    },
    parent: {}
};
```

报错：`Import #1 module="env" function="getTotalMemory" error: function import requires a callable`, 以上同理，以空函数代替

```
var wasm_env = {
    global: {},
    env: {
        abort: fun_,
        assert: fun_,
        enlargeMemory: fun_,
        abortOnCannotGrowMemory: fun_,
        abortStackOverflow: fun_,
        nullFunc_ii: fun_,
        nullFunc_iiii: fun_,
        nullFunc_v: fun_,
        nullFunc_vi: fun_,
        nullFunc_viiii: fun_,
        nullFunc_viiiii: fun_,
        nullFunc_viiiiii: fun_,
        invoke_ii: fun_,
        invoke_iiii: fun_,
        invoke_v: fun_,
        invoke_vi: fun_,
        invoke_viiii: fun_,
        invoke_viiiii: fun_,
        invoke_viiiiii: fun_,
        __ZSt18uncaught_exceptionv: fun_,
        ___cxa_find_matching_catch: fun_,
        ___gxx_personality_v0: fun_,
        ___lock: fun_,
        ___resumeException: fun_,
        ___setErrNo: fun_,
        ___syscall140: fun_,
        ___syscall146: fun_,
        ___syscall54: fun_,
        ___syscall6: fun_,
        ___unlock: fun_,
        _abort: fun_,
        _emscripten_memcpy_big: fun_,
        flush_NO_FILESYSTEM: fun_,
    },
    asm2wasm: {
        "f64-rem": function(e, t) {
            return e % t
        },
        debugger: function() {}
    },
    parent: {}
};
```

报错：`Import #1 module="env" function="getTotalMemory" error: function import requires a callable`

![](https://img-blog.csdnimg.cn/2021120613322938.png)

```
var Q = 16777216
getTotalMemory: function () { return Q },
```

报错：`Import #20 module="env" function="_get_unicode_str" error: function import requires a callable`

![](https://img-blog.csdnimg.cn/20211206133229611.png)

```
_get_unicode_str: function () {  
    function a(e) {
        return e ? 48 < e.length ? e.substr(0, 48) : e : ""
    }
    var e = function () {
        var e = document.URL
            , t = window.navigator.userAgent.toLowerCase()
            , o = "";
        0 < document.referrer.length && (o = document.referrer);
        try {
            0 == o.length && 0 < opener.location.href.length && (o = opener.location.href)
        } catch (e) { }
        var i = window.navigator.appCodeName
            , n = window.navigator.appName
            , r = window.navigator.platform
            , e = a(e)
            , o = a(o);
        return e + "|" + (t = a(t)) + "|" + o + "|" + i + "|" + n + "|" + r
    }()
        , t = q(e) + 1, o =Ve(t);// 5250872; //_malloc(t);
    console.log('---',t, o)
    return S(e, o, t + 1),
        o
},
```

报错：`Import #21 module="env" function="memoryBase" error: global import must be a number or WebAssembly.Global object`

报错：`Import #22 module="env" function="tableBase" error: global import must be a number or WebAssembly.Global object`

![](https://img-blog.csdnimg.cn/20211206133230256.png)

```
memoryBase: 1024,
tableBase: 0,
```

报错：`Import #23 module="env" function="DYNAMICTOP_PTR" error: global import must be a number or WebAssembly.Global object`

报错：`Import #24 module="env" function="tempDoublePtr" error: global import must be a number or WebAssembly.Global object`

报错：`Import #25 module="env" function="STACKTOP" error: global import must be a number or WebAssembly.Global object`

报错：`Import #26 module="env" function="STACK_MAX" error: global import must be a number or WebAssembly.Global object`

![](https://img-blog.csdnimg.cn/20211206133230836.png)

```
DYNAMICTOP_PTR: 7968,
tempDoublePtr: 7952,
tempDoublePtr: 7952,
STACK_MAX: 5250864,
```

报错：`Import #27 module="global" function="NaN" error: global import must be a number or WebAssembly.Global object`

![](https://img-blog.csdnimg.cn/20211206133231416.png)

```
global: {
    NaN: NaN,
    Infinity: 1 / 0
}
```

报错：`Import #29 module="env" function="memory" error: memory import must be a WebAssembly.Memory object`

![](https://img-blog.csdnimg.cn/20211206133231967.png)

搜索`WebAssembly.Memory`

![](https://img-blog.csdnimg.cn/20211206133232546.png)

```
var Q = 16777216, j = 65536;
var wasmMemory = new WebAssembly.Memory({  
    initial: Q / j,
    maximum: Q / j
})
```

报错：`Import #30 module="env" function="table" error: table import requires a WebAssembly.Table`

![](https://img-blog.csdnimg.cn/20211206133233136.png)

```
table: new WebAssembly.Table({
    initial: 99,
    maximum: 99,
    element: "anyfunc"
}),
```

初始化好 wasm 后，开始处理`function u(e, t, o, i, n)`

![](https://img-blog.csdnimg.cn/20211206133233702.png)

```
// h["_" + (r = e)] = wasm._getckey
// je = wasm.stackSave
// He = wasm.stackRestore
// Fe = wasm.stackAlloc
// Ve = wasm._malloc  修改_get_unicode_str中的Ve
function _getckey() {
    return wasmobject.exports._getckey.apply(null, arguments)
}
function stackSave() {   
    return wasmobject.exports.stackSave.apply(null, arguments)
}
function stackRestore() {
    return wasmobject.exports.stackRestore.apply(null, arguments)
}
function stackAlloc() {   
    return wasmobject.exports.stackAlloc.apply(null, arguments)
}
function _malloc() {   
    return wasmobject.exports._malloc.apply(null, arguments)
}


// 函数引用完成n函数 
function S(e, t, o) {  // o(a, b, c)
    return w("number" == typeof o, "stringToUTF8(str, outPtr, maxBytesToWrite) is missing the third parameter that specifies the length of the output buffer!"),
        E(e, C, t, o)
}
function T(e, t) {
    if (0 === t || !e)
        return "";
    for (var o, i = 0, n = 0; w(e + n < Q),
    i |= o = C[e + n >> 0],
    (0 != o || t) && (n++,
    !t || n != t); )
        ;
    t = t || n;
    var r = "";
    if (i < 128) {
        for (var a; 0 < t; )
            a = String.fromCharCode.apply(String, C.subarray(e, e + Math.min(t, 1024))),
            r = r ? r + a : a,
            e += 1024,
            t -= 1024;
        return r
    }
    return _(C, e)
}
var l = {
    stackSave: function() {
        stackSave()
    },
    stackRestore: function() {
        stackRestore()
    },
    arrayToC: function(e) {
        var t, o, i = stackAlloc(e.length); // Fe(e.length);
        return o = i,
        w(0 <= (t = e).length, "writeArrayToMemory array must have a length (should be an array or typed array)"),
        R.set(t, o),
        i
    },
    stringToC: function(e) {
        var t, o = 0;
        return null != e && 0 !== e && (t = 1 + (e.length << 2),
        S(e, o = stackAlloc(t), t)), //Fe(e.length);
        o
    }
}
var x = {
    string: l.stringToC,
    array: l.arrayToC
}
function n(...args) {
    var e = "getckey"
    var t = "string"
    var o = ["number", "string", "string", "string", "string", "number"]
    var i = args
    var n = undefined
    var r, a, s = (w(a = _getckey, "Cannot call unknown function " + r + ", make sure it is exported"),
    a), c = [], d = 0;
    // var r = "getckey", a = _getckey, s = _getckey, c = [], d = 0;
    // if (w("array" !== t, 'Return type should not be "array".'),
    // i)
    for (var l = 0; l < i.length; l++) {
        var u = x[o[l]];
        console.log("uuuu",u)
        u ? (0 === d && (d = stackSave()),  // je()
            c[l] = u(i[l])) : c[l] = i[l]
    }
    var f, p = s.apply(null, c);
    return f = p,
        p = "string" === t ? T(f) : "boolean" === t ? Boolean(f) : f,
        0 !== d && stackRestore(d), // He(d)
        p
}
```

报错：`TypeError: Cannot set property '7984' of undefined` 说明在内存操作的时候有部分变量我们没有注意到，回到`445: function(Ke, e, t)`中，抽出部分值操作

```
function X() {
    R = new Int8Array(k),
        O = new Int16Array(k),
        I = new Int32Array(k),
        C = new Uint8Array(k),
        M = new Uint32Array(k)
}
X()
```

报错：`document is not defined`，`window is not defined`，`Cannot read property 'userAgent' of undefined`...

```
var document = {
    URL: "",
    referrer: ""
}
var window = {
    document: document,
    navigator: {
        userAgent: "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/72.0.3626.121 Safari/537.36",
        appCodeName: "Mozilla",
        appName: "Netscape",
        platform: "Win32"
    },
};
```

node ckey_blog.js

![](https://img-blog.csdnimg.cn/20211206133234302.png)

主动调用
====

由于公众号和正常腾讯视频中的部分参数不一致，修改`function m(e)`

```
function getCKey(plateform,appVer,vid) {
    // var t = n ? (e.encryptVer = "9.1",
    // n(e.platform, e.appVer, e.vids || e.vid, "", e.guid, e.tm)) : (e.encryptVer = "8.1",
    // a(e.vids || e.vid, e.tm, e.appVer, e.guid, e.platform));
    var t = n(plateform, appVer, vid, "", createGUID(), Date.parse(new Date()).toString().substr(0, 10))
    return t
}
```

ckey.py

```
import execjs
import re
import requests
import json

from m3u8 import M3u8Download

with open("ckey_blog.js", "r",encoding="utf-8") as f:
    js_code = f.read()

# target_url = "http://v.qq.com/txp/iframe/player.html?origin=https%3A%2F%2Fmp.weixin.qq.com&chid=17&vid=s3306hychob&autoplay=false&full=true&show1080p=false&isDebugIframe=false"
target_url = "https://v.qq.com/x/cover/bzfkv5se8qaqel2/j002024w2wg.html"
vinfoparam = "spsrt=1&charge=0&defaultfmt=auto&otype=ojson&guid={}&flowid={}&platform={}&sdtfrom={}&defnpayver=0&appVer={}&host=v.qq.com&sphttps=0&tm=1637237951&spwm=4&vid={}&defn=&fhdswitch=0&show1080p=0&isHLS=1&dtype=3&sphls=1&spgzip=&dlver=&hdcp=0&spau=1&spaudio=15&defsrc=1&encryptVer=9.1&cKey={}"
data = {}
data["buid"] = "vinfoad"
guid = execjs.compile(js_code).call('createGUID')
# 区分腾讯视频还是公众号视频
if "mp.weixin.qq.com" in target_url:
    vid = re.compile(r"&vid=(.*?)&").findall(target_url)[0] # ?非贪婪
    plateform = "70201"
    flowid = execjs.compile(js_code).call('createGUID') + "_" + plateform
    sdtfrom = "v1104"
    appVer = "3.4.40"
    ckey = execjs.compile(js_code).call('getCKey',plateform,appVer,vid)
else:
    vid = target_url.split("/")[-1].split(".")[0]
    plateform = "10201"
    flowid = execjs.compile(js_code).call('createGUID') + "_" + plateform
    sdtfrom = "v1010"
    appVer = "3.5.57"
    ckey = execjs.compile(js_code).call('getCKey', plateform, appVer, vid)

data["vinfoparam"] = vinfoparam.format(guid,flowid,plateform,sdtfrom,appVer,vid,ckey)
result = requests.post('http://vd.l.qq.com/proxyhttp', data=json.dumps(data)).json()
# print(result)
if result.get("errCode") == 0:
    url_data = json.loads(result.get("vinfo"))["vl"]["vi"][0]["ul"]["ui"][0]
    url = url_data["url"]+url_data["hls"]["pt"]
    print(url)
    M3u8Download(url,
                 "test1",
                 max_workers=64,
                 num_retries=10,
                 )
```

![](https://img-blog.csdnimg.cn/20211206133235146.png)

总结
==

针对 wasm 二进制方式加密的 js 逆向，类似安卓的 so 逆向，可以选择硬肛分析汇编代码，当然也可以选择 Unidbg 主动调用，本文利用 js 的`WebAssembly`实例化 wasm 并完成调用分析 cKey，完成腾讯系视频的下载，至于会员视频分析`logintoken`参数，下次一定，下次一定。。。

> 本文由博客群发一文多发等运营工具平台 [OpenWrite](https://openwrite.cn?from=article_bottom) 发布