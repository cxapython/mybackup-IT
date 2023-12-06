> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/xiaoweigege/p/14954648.html)

JavaScript 常用的 Hook 脚本
======================

本文 Hook 脚本 来自 `包子`

页面最早加载代码 Hook 时机[#](#页面最早加载代码hook时机)
------------------------------------

1.  在 source 里 用 dom 事件断点的 script 断点
2.  然后刷新网页，就会断在第一个 js 标签，这时候就可以注入代码进行 hook

监听 键盘 与 鼠标 事件[#](#监听-键盘-与-鼠标-事件)
--------------------------------

```
// 判断是否按下F12  onkeydown事件
/*
提示： 与 onkeydown 事件相关联的事件触发次序:
onkeydown
onkeypress
onkeyup
*/

// F12的键码为 123，可以直接全局搜索 keyCode == 123, == 123 ,keyCode
document.onkeydown = function() {
    if (window.event && window.event.keyCode == 123) {
        // 改变键码
        event.keyCode = 0;
        event.returnValue = false;
        // 监听到F12被按下直接关闭窗口
        window.close();
        window.location = "about:blank";
    }
}
;
// 监听鼠标右键是否被按下方法 1， oncontextmenu事件
document.oncontextmenu = function () { return false; };

// 监听鼠标右键是否被按下方法 2，onmousedown事件
document.onmousedown = function(evt){
    // button属性是2 就代表是鼠标右键 
    if(evt.button == 2){
        alert('监听到鼠标右键被按下')
        evt.preventDefault() // 该方法将通知 Web 浏览器不要执行与事件关联的默认动作
        return false;
    }
}

// 监听用户工具栏调起开发者工具，判断浏览器的可视高度和宽度是否有改变，有改变则处理，
// 判断是否开了开发者工具不太合理。
var h = window.innerHeight, w = window.innerWidth;
window.onresize = function(){
    alert('改变了窗口高度')
}

// hook代码
(function() {
    //严谨模式 检查所有错误
    'use strict';
    // hook 鼠标选择
    Object.defineProperty(document, 'onselectstart', {
		set: function(val) {
			console.log('Hook捕获到选中设置->', val);
			return val;
		}
      });
	// hook 鼠标右键
	Object.defineProperty(document,'oncontextmenu',{
		set:function(evt){
			console.log("检测到右键点击");
			return evt
		}
	});
})();
```

webpack hook 半自动扣[#](#webpack-hook-半自动扣)
----------------------------------------

```
//在加载器后面下断点  执行下面代码
// 这里的f 替换成需要导出的函数名
window.zhiyuan = f;
window.wbpk_ = "";
window.isz = false;
f = function(r){
	if(window.isz)
	{
        // e[r]里的e 是加载器里的call那里
		window.wbpk_ = window.wbpk_ + r.toString()+":"+(e[r]+"")+ ",";
	}
	return window.zhiyuan(r);
}

//在你要的方法加载前下断点 执行 window.isz=true
//在你要的方法运行后代码处下断点  执行 window.wbpk_  拿到所有代码  注意后面有个逗号

function o(t) {

    if (n[t])
        return n[t].exports;
    var i = n[t] = {
        i: t,
        l: !1,
        exports: {}
    };
    console.log("被调用的 >>> ", e[t].toString());
    //  这里进行拼接，bb变量需要在全局定义一下 
    // t 是模块名， e[t] 是模块对应的函数， 也就是key：value形式
    bb += `"${t}":${e[t].toString()},`
    return e[t].call(i.exports, i, i.exports, o),
    i.l = !0,
    i.exports
}
bz = o;
```

如果只是调用模块，不用模块里面的方法, 那么直接获取调用模块的时候所有加载过的模块，进行拼接

document 下的 createElement() 方法的 hook, 查看创建了什么元素[#](#document下的createelement方法的hook查看创建了什么元素)
--------------------------------------------------------------------------------------------

```
(function() {
    'use strict'
   var _createElement = document.createElement.bind(document);
   document.createElement = function(elm){
   // 这里做判断 是否创建了script这个元素    
   if(elm == 'body'){
        debugger;
   }
    return _createElement(elm);
}
})();
```

之前我不知道我用的是 `var _createElement = document.createElement` 导致一直报错 `Uncaught TypeError: Illegal invocation`  
原来是需要绑定一下对象 `var _createElement = document.createElement.bind(document);`

headers hook 当 header 中包含 Authorization 时，则插入断点[#](#headers-hook--当header中包含authorization时，则插入断点)
-------------------------------------------------------------------------------------------------

```
var code = function(){
var org = window.XMLHttpRequest.prototype.setRequestHeader;
window.XMLHttpRequest.prototype.setRequestHeader = function(key,value){
    if(key=='Authorization'){
        debugger;
    }
    return org.apply(this,arguments);
}
}
var script = document.createElement('script');
script.textContent = '(' + code + ')()';
(document.head||document.documentElement).appendChild(script);
script.parentNode.removeChild(script);
```

请求 hook 当请求的 url 里包含 MmEwMD 时，则插入断点[#](#请求hook--当请求的url里包含mmewmd时，则插入断点)
------------------------------------------------------------------------

```
var code = function(){
var open = window.XMLHttpRequest.prototype.open;
window.XMLHttpRequest.prototype.open = function (method, url, async){
    if (url.indexOf("MmEwMD")>-1){
        debugger;
    }
    return open.apply(this, arguments);
};
}
var script = document.createElement('script');
script.textContent = '(' + code + ')()';
(document.head||document.documentElement).appendChild(script);
script.parentNode.removeChild(script);
```

docuemnt.getElementById 以及 value 属性的 hook[#](#docuemntgetelementbyid-以及value属性的hook)
------------------------------------------------------------------------------------

```
// docuemnt.getElementById 以及value属性的hook,可以参考完成innerHTML的hook
document.getElementById = function(id) {
    var value = document.querySelector('#' + id).value;
    console.log('DOM操作 id: ', id)
    try {

        Object.defineProperty(document.querySelector('#'+ id), 'value', {
            get: function() {
                console.log('getting -', id, 'value -', value);
                return value;
            },
            set: function(val) {
                console.log('setting -', id, 'value -', val)
                value = val;
            }
        })
    } catch (e) {
        console.log('---------华丽的分割线--------')
    }
    return document.querySelector('#' + id);
}
```

过 debugger 阿布牛逼[#](#过debugger-阿布牛逼)
-----------------------------------

```
function Closure(injectFunction) {
    return function() {
        if (!arguments.length)
            return injectFunction.apply(this, arguments)
        arguments[arguments.length - 1] = arguments[arguments.length - 1].replace(/debugger/g, "");
        return injectFunction.apply(this, arguments)
    }
}

var oldFunctionConstructor = window.Function.prototype.constructor;
window.Function.prototype.constructor = Closure(oldFunctionConstructor)
//fix native function
window.Function.prototype.constructor.toString = oldFunctionConstructor.toString.bind(oldFunctionConstructor);

var oldFunction = Function;
window.Function = Closure(oldFunction)
//fix native function
window.Function.toString = oldFunction.toString.bind(oldFunction);

var oldEval = eval;
window.eval = Closure(oldEval)
//fix native function
window.eval.toString = oldEval.toString.bind(oldEval);

// hook GeneratorFunction
var oldGeneratorFunctionConstructor = Object.getPrototypeOf(function*() {}).constructor
var newGeneratorFunctionConstructor = Closure(oldGeneratorFunctionConstructor)
newGeneratorFunctionConstructor.toString = oldGeneratorFunctionConstructor.toString.bind(oldGeneratorFunctionConstructor);
Object.defineProperty(oldGeneratorFunctionConstructor.prototype, "constructor", {
    value: newGeneratorFunctionConstructor,
    writable: false,
    configurable: true
})

// hook Async Function
var oldAsyncFunctionConstructor = Object.getPrototypeOf(async function() {}).constructor
var newAsyncFunctionConstructor = Closure(oldAsyncFunctionConstructor)
newAsyncFunctionConstructor.toString = oldAsyncFunctionConstructor.toString.bind(oldAsyncFunctionConstructor);
Object.defineProperty(oldAsyncFunctionConstructor.prototype, "constructor", {
    value: newAsyncFunctionConstructor,
    writable: false,
    configurable: true
})

// hook dom
var oldSetAttribute = window.Element.prototype.setAttribute;
window.Element.prototype.setAttribute = function(name, value) {
    if (typeof value == "string")
        value = value.replace(/debugger/g, "")
    // 向上调用
    oldSetAttribute.call(this, name, value)
}
;
var oldContentWindow = Object.getOwnPropertyDescriptor(HTMLIFrameElement.prototype, "contentWindow").get
Object.defineProperty(window.HTMLIFrameElement.prototype, "contentWindow", {
    get() {
        var newV = oldContentWindow.call(this)
        if (!newV.inject) {
            newV.inject = true;
            core.call(newV, globalConfig, newV);
        }
        return newV
    }
})
```

过 debugger—1 constructor 构造器构造出来的[#](#过debugger1---constructor-构造器构造出来的)
------------------------------------------------------------------------

```
var _constructor = constructor;
Function.prototype.constructor = function(s) {
    if (s == "debugger") {
        console.log(s);
        return null;
    }
    return _constructor(s);
}
```

过 debugger—2 eval 的[#](#过debugger2--eval的)
------------------------------------------

```
(function() {
    'use strict';
    var eval_ = window.eval;
    window.eval = function(x) {
        eval_(x.replace("debugger;", "  ; "));
    }
    ;
    window.eval.toString = eval_.toString;
}
)();
```

JSON HOOK[#](#json-hook)
------------------------

```
var my_stringify = JSON.stringify;
JSON.stringify = function (params) {
    //这里可以添加其他逻辑比如 debugger
    console.log("json_stringify params:",params);
    return my_stringify(params);
};

var my_parse = JSON.parse;
JSON.parse = function (params) {
    //这里可以添加其他逻辑比如 debugger
    console.log("json_parse params:",params);
    return my_parse(params);
};
```

对象属性 hook 属性自定义[#](#对象属性hook-属性自定义)
-----------------------------------

```
(function(){
    // 严格模式，检查所有错误
    'use strict'
    // document 为要hook的对象 ,属性是cookie
    Object.defineProperty(document,'cookie',{
        // hook set方法也就是赋值的方法，get就是获取的方法
        set: function(val){
            // 这样就可以快速给下面这个代码行下断点，从而快速定位设置cookie的代码
            debugger;  // 在此处自动断下
            console.log('Hook捕获到set-cookie ->',val);
            return val;
        }
    })
})();
```

cookies - 1 （不是万能的 有些时候 hook 不到 自己插入 debugger）[#](#cookies---1-（不是万能的-有些时候hook不到-自己插入debugger）)
-----------------------------------------------------------------------------------------------

```
var cookie_cache = document.cookie;

Object.defineProperty(document, 'cookie', {
    get: function() {
        console.log('Getting cookie');
        return cookie_cache;
    },
    set: function(val) {
        console.log("Seting cookie",val);
        var cookie = val.split(";")[0];
        var ncookie = cookie.split("=");
        var flag = false;
        var cache = cookie_cache.split("; ");
        cache = cache.map(function(a){
            if (a.split("=")[0] === ncookie[0]){
                flag = true;
                return cookie;
            }
            return a;
        })
    }
})
```

cookies - 2[#](#cookies---2)
----------------------------

```
var code = function(){
    var org = document.cookie.__lookupSetter__('cookie');
    document.__defineSetter__("cookie",function(cookie){
        if(cookie.indexOf('TSdc75a61a')>-1){
            debugger;
        }
        org = cookie;
    });
    document.__defineGetter__("cookie",function(){return org;});
}
var script = document.createElement('script');
script.textContent = '(' + code + ')()';
(document.head||document.documentElement).appendChild(script);
script.parentNode.removeChild(script);

// 当cookie中匹配到了 TSdc75a61a， 则插入断点。
```

window attr[#](#window-attr)
----------------------------

```
// 定义hook属性
var window_flag_1 = "_t";
var window_flag_2 = "ccc";

var key_value_map = {};
var window_value = window[window_flag_1];

// hook
Object.defineProperty(window, window_flag_1, {
    get: function(){
        console.log("Getting",window,window_flag_1,"=",window_value);
        //debugger
        return window_value
    },
    set: function(val) {
        console.log("Setting",window, window_flag_1, "=",val);
        //debugger
        window_value = val;
        key_value_map[window[window_flag_1]] = window_flag_1;
        set_obj_attr(window[window_flag_1],window_flag_2);
    },

});

function set_obj_attr(obj,attr){
    var obj_attr_value = obj[attr];
    Object.defineProperty(obj,attr, {
        get: function() {
            console.log("Getting", key_value_map[obj],attr, "=", obj_attr_value);
            //debugger
            return obj_attr_value;
        },
        set: function(val){
            console.log("Setting", key_value_map[obj], attr, "=", val);
            //debugger
            obj_attr_value = val;
        },
    });
}
```

eval/Function[#](#evalfunction)
-------------------------------

```
window.__cr_eval = window.eval;
var myeval = function(src) {
    // src就是eval运行后 最终返回的值
    console.log(src);
    console.log("========= eval end ===========");
    return window.__cr_eval;
}

var _myeval = myeval.bind(null);
_myeval.toString = window.__cr_eval.toString;
Object.defineProperty(window, 'eval',{value: _myeval});

window._cr_fun = window.Function
var myfun = function(){
    var args = Array.prototype.slice.call(arguments, 0, -1).join(","), src = arguments[arguments.lenght -1];
    console.log(src);
    console.log("======== Function end =============");
    return window._cr_fun.apply(this, arguments)
}

myfun.toString = function() {return window._cr_fun + ""} //小花招，这里防止代码里检测原生函数
Object.defineProperty(window, "Function",{value: myfun})
```

eval 取返回值[#](#eval-取返回值)
------------------------

```
_eval = eval;
eval = (res)=>{
    res1 = res // 返回值
    return _eval(res)
}

eval(xxxxxxxxx)
```

eval proxy 代理 [https://segmentfault.com/a/1190000025154230](https://segmentfault.com/a/1190000025154230)[#](#eval-proxy代理---httpssegmentfaultcoma1190000025154230)
------------------------------------------------------------------------------------------------------------------------------------------------------------------

```
// 代理eval
eval = new Proxy(eval,{
    // 如果代理的是函数 查看调用 就用apply属性
    // 第二个参数是prop 这里用不上 因为是属性，eval只是个函数 所以prop为undefind 这里设置了下划线 ——
    apply: (target,_,arg)=>{
        // target 是被代理的函数或对象名称，当前是[Function: eval]
        // arg是传进来的参数,返回的是个列表
        console.log(arg[0])
    }
})

//  eval执行的时候就会被代理拦截
// 传入的如果是字符串 那么只会返回字符串，这里是匿名函数 直接执行 return了内容
eval(
    (function(){return "我是包子 自己执行了"})()
)

// 结果 ： 我是包子 自己执行了
```

websocket hook[#](#websocket-hook)
----------------------------------

```
// 1、webcoket 一般都是json数据格式传输，那么发生之前需要JSON.stringify  
var my_stringify = JSON.stringify;
JSON.stringify = function (params) {
    //这里可以添加其他逻辑比如 debugger
    console.log("json_stringify params:",params);
    return my_stringify(params);
};

var my_parse = JSON.parse;
JSON.parse = function (params) {
    //这里可以添加其他逻辑比如 debugger
    console.log("json_parse params:",params);
    return my_parse(params);
};

// 2  webScoket 绑定在windows对象，上，根据浏览器的不同，websokcet名字可能不一样 
//chrome window.WebSocket  firfox window.MozWebSocket;
window._WebSocket = window.WebSocket;

// hook send
window._WebSocket.prototype.send = function (data) {
    console.info("Hook WebSocket", data);
    return this.send(data)
}

Object.defineProperty(window, "WebSocket",{value: WebSocket})
```

hook 正则 —— 1[#](#hook-正则--1)
----------------------------

```
(function () {
    var _RegExp = RegExp;
    RegExp = function (pattern, modifiers) {
        console.log("Some codes are setting regexp");
        debugger;
        if (modifiers) {
            return _RegExp(pattern, modifiers);
        } else {
            return _RegExp(pattern);
        }
    };
    RegExp.toString = function () {
        return "function setInterval() { [native code] }"
    };
})();
```

hook 正则 2 加在 sojson 头部过字符串格式化检测[#](#hook-正则-2-加在sojson头部过字符串格式化检测)
------------------------------------------------------------------

```
(function() {
    var _RegExp = RegExp;
    RegExp = function(pattern, modifiers) {
        if (pattern == decodeURIComponent("%5Cw%2B%20*%5C(%5C)%20*%7B%5Cw%2B%20*%5B'%7C%22%5D.%2B%5B'%7C%22%5D%3B%3F%20*%7D") || pattern == decodeURIComponent("function%20*%5C(%20*%5C)") || pattern == decodeURIComponent("%5C%2B%5C%2B%20*(%3F%3A_0x(%3F%3A%5Ba-f0-9%5D)%7B4%2C6%7D%7C(%3F%3A%5Cb%7C%5Cd)%5Ba-z0-9%5D%7B1%2C4%7D(%3F%3A%5Cb%7C%5Cd))") || pattern == decodeURIComponent("(%5C%5C%5Bx%7Cu%5D(%5Cw)%7B2%2C4%7D)%2B")) {
            pattern = '.*?';
            console.log("发现sojson检测特征，已帮您处理。")
        }
        if (modifiers) {
            console.log("疑似最后一个检测...已帮您处理。")
            console.log("已通过全部检测，请手动处理debugger后尽情调试吧！")
            return _RegExp(pattern, modifiers);
        } else {
            return _RegExp(pattern);
        }
    }
    ;
    RegExp.toString = function() {
        return _RegExp.toString();
    }
    ;
}
)();
```

hook canvas (定位图片生成的地方)[#](#hook-canvas-定位图片生成的地方)
--------------------------------------------------

```
(function() {
    'use strict';
    let create_element = document.createElement.bind(doument);

    document.createElement = function (_element) {
        console.log("create_element:",_element);
        if (_element === "canvas") {
            debugger;
        }
        return create_element(_element);
    }
})();
```

setInterval 定时器[#](#setinterval-定时器)
------------------------------------

```
(function() {
    setInterval_ = setInterval;
    console.log("原函数已被重命名为setInterval_")
    setInterval = function() {}
    ;
    setInterval.toString = function() {
        console.log("有函数正在检测setInterval是否被hook");
        return setInterval_.toString();
    }
    ;
}
)();
```

setInterval 循环清除定时器[#](#setinterval-循环清除定时器)
--------------------------------------------

```
for(var i = 0; i < 9999999; i++) window.clearInterval(i)
```

console.log 检测例子 （不让你输出调试）[#](#consolelog-检测例子-（不让你输出调试）)
---------------------------------------------------------

```
var oldConsole = ["debug", "error", "info", "log", "warn", "dir", "dirxml", "table", "trace", "group", "groupCollapsed", "groupEnd", "clear", "count", "countReset", "assert", "profile", "profileEnd", "time", "timeLog", "timeEnd", "timeStamp", "context", "memory"].map(key=>{
    var old = console[key];
    console[key] = function() {}
    ;
    console[key].toString = old.toString.bind(old)
    return old;
}
)
```

检测函数是否被 hook 例子[#](#检测函数是否被hook例子)
----------------------------------

```
if (window.eval == 'native code') {
    console.log('发现eval函数被hook了 开始死循环');
}
```