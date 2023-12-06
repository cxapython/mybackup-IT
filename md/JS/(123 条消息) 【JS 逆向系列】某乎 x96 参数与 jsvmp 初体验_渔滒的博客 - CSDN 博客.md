> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/zjq592767809/article/details/124066638)

### 【JS 逆向系列】某乎 x96 参数与 jsvmp 初体验

*   *   [js 分析](#js_4)
    *   [jsvmp 分析](#jsvmp_39)
    *   [第一种解决方案 - 补环境](#_64)
    *   [第二种解决方案 - 修改操作码](#_81)
    *   [第三种解决方案 - 算法还原](#_727)
    *   [参考文章](#_872)

样品网址：aHR0cHM6Ly93d3cuemhpaHUuY29tLw==

js 分析
-----

在搜索的时候，请求头中会存在一个 x-zse-96 的参数，这个参数是[加密](https://so.csdn.net/so/search?q=%E5%8A%A0%E5%AF%86&spm=1001.2101.3001.7020)的，本篇文章主要分析这个参数是如何生成的

![](https://img-blog.csdnimg.cn/bfec23e5578e4e70bfca4e306bd1eb77.png#pic_center)  
直接在全局中搜索

![](https://img-blog.csdnimg.cn/bbc924f3d16949e699bb7126353ed291.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
发现只有两处地方引用到，那么直接打上断点刷新

![](https://img-blog.csdnimg.cn/b571f5f407434737a5ac3695ce530e55.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
此时发现参数已经生成，前面的【2.0_】是固定值，后面的【y】是前面的【O.signature】

继续往前看【O】，这是由一个自执行函数生成的，在里面也下一个断点

![](https://img-blog.csdnimg.cn/7c68537397d8472a86452edd7860536d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
这里有三个参数，第一个【e】是请求的地址，第二个【t】是请求体，第三个【n】是一个对象，里面又有三个值，其中【zse93】是另一个请求头的值，是一个定值，【dc0】是 cookie 中的【d_c0】的值，还有一个【xZst81】是空的，可以不用管

接着的参数【s】就是过滤掉布尔值为 false 的值然后进行拼接，所以实际【s】就是【{x_zse_93}+{path}+{d_c0}】

然后我们需要的参数【signature】就是【s】经过两个函数生成的，那么首先来看看【f】函数

![](https://img-blog.csdnimg.cn/07c37f1d5f354d99bb70e10adbfbafea.png#pic_center)  
看到返回值是一个长度 32 的字符串，猜测这个函数很有可能是一个 md5 的计算，从在线网站计算一下对比，发现这个就是一个标准的 md5 算法

接着就是【u】函数

![](https://img-blog.csdnimg.cn/c48ffacec2394e4ba928da95745cc84f.png#pic_center)  
这个就是我们需要的结果，跟进去看看

![](https://img-blog.csdnimg.cn/30af1a5ef0414cb6b2d772173b41e762.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
接着在单步调试的时候，发现会在各个函数之间反复横跳。看到这个前面有一个很长的字符串【AxjgB5MAnACoAJwBpAAAABAAIAKcAqgAM*****************】，这种很有可能就是 jsvmp。

jsvmp 分析
--------

jsvmp 就是将 js 源代码首先编译为字节码，得到的这种字节码就变成只有操作码（opcode）和操作数（Operands），这是其中一个前端代码的保护技术。

整体架构流程是服务器端通过对 JavaScript 代码词法分析 -> 语法分析 -> 语法树 -> 生成 AST-> 生成私有指令 -> 生成对应私有解释器，将私有指令加密与私有解释器发送给浏览器，就开始一边解释，一边执行。

![](https://img-blog.csdnimg.cn/img_convert/f90bcdc9ae12c152fe3f4393508e1f80.png#pic_center)  
在客户端中，通过特定的解释器，不断执行一条一条的指令

![](https://img-blog.csdnimg.cn/img_convert/4d5087e6d8b74845116b902c6ace5166.png#pic_center)

既然是 jsvmp，那么肯定会有 vmp 的一些特征

![](https://img-blog.csdnimg.cn/36949fce3b59490ea1eead0983c2ff95.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
这里的【G】就是 vm 初始化，这里有几个参数需要特别注意

<table><thead><tr><th>参数</th><th>映射意义</th></tr></thead><tbody><tr><td>this.r</td><td>全局变量</td></tr><tr><td>this.C</td><td>pc 寄存器</td></tr><tr><td>this.k</td><td>本地变量</td></tr><tr><td>this.f</td><td>调用堆栈</td></tr><tr><td>this.J</td><td>操作指令</td></tr></tbody></table>

上面的仅仅是我个人的理解，不一定对

第一种解决方案 - 补环境
-------------

首先把整段代码扣到本地，在开头处加上 jsdom 的代码

![](https://img-blog.csdnimg.cn/03989586de53460f925a3281e026b427.png#pic_center)

```
const{JSDOM}=require("jsdom");
const dom=new JSDOM("<!DOCTYPE html><p>Hello world</p>");
window=dom.window;
```

然后尝试生成的参数是否与网页一致

![](https://img-blog.csdnimg.cn/1c218ce61a02409a90a9a94b8822ca0b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

测试发现是一模一样的，那么扣代码的方法就完成了

第二种解决方案 - 修改操作码
---------------

那么有没有办法不用补环境呢？那也是有的，那么接着来进一步分析这个 jsvmp，调试一下上面补环境得到的 js

![](https://img-blog.csdnimg.cn/f344b9211dc148f9adc0aa6427f60c0e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
这里的【e】就是那个很长的字符串，调用的【this.D】方法

![](https://img-blog.csdnimg.cn/fb025af6a46f493489191da865c50d84.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
这里是解码这个字符串，得到一段数值数组，赋值给【this.G】和一段字符串数组，赋值给【this.b】

![](https://img-blog.csdnimg.cn/3b8bff902a6f41058c810206e49a805a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
解码完了以后，有一个【this.R】，这个如果两条运行两条指令之间的时间，如果大于 500 毫秒，就认为是在调试，被强制退出了，所以在调试的时候可以把这个 if 判断注释掉

![](https://img-blog.csdnimg.cn/26a01fa7684b4401bf5eb2e5f69d0134.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
【this.C】是 pc 寄存器，从【this.G】不断取出操作码，然后使用【this.e】方法执行这条操作码，跟进去第一条操作码

![](https://img-blog.csdnimg.cn/acb94e95d7b64b6aa55b6f2b9b2bba28.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
因为之前解码那个长字符串的时候，是两个字节得出一个操作码，可以看出一个操作码是一个 short 类型，这里【61440】的二进制是【11110000 00000000】，并且左移 12 位，就是取最高的 4 位。2 的 4 次方是 16，刚好对应着 16 种不同的操作指令

![](https://img-blog.csdnimg.cn/4a7ed6d4c4634c8f8a4eec777eae8d56.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
继续单步第一条的操作码

![](https://img-blog.csdnimg.cn/c9d08e5241c248ac9684fd3bf9773190.png#pic_center)  
这里计算了一个【this.h】的值，多调试几个会发现，除了计算【this.t】是真正的操作类型下面的操作码，其他都是操作数。所以这里得到一个【this.h】的操作数是【7】，

![](https://img-blog.csdnimg.cn/42d2b702b30744d58b0b622eba6788ef.png#pic_center)

计算完真正的操作码和操作数后，调用了对应的【e】方法来执行

![](https://img-blog.csdnimg.cn/7f0cae2bbb1d4e0084407f942cc9f2e5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
这是一个函数初始化，因为之前说过把【r】当作全局变量。所以这里把内部的【r】函数，绑定到全局的第四个值，那么，第一条指令就执行完成了，接着回到循环，执行第二条，第三条指令，一次类推，直到所有的指令的执行完。

那么这里的代码具有一定的共性，那么能不能自己根据这些操作码，来还原出伪代码呢，来试一下。

首先把解码的函数复制过来，并生成四个全局变量

```
const parser = require("@babel/parser");
// 为parser提供模板引擎
const template = require("@babel/template").default;
// 遍历AST
const traverse = require("@babel/traverse").default;
// 操作节点，比如判断节点类型，生成新的节点等
const types = require("@babel/types");
// 将语法树转换为源代码
const generator = require("@babel/generator");
// 操作文件
const fs = require("fs");

var k = function (e) {
    for (var t = 66, n = [], r = 0; r < e.length; r++) {
        var i = 24 ^ e.charCodeAt(r) ^ t;
        n.push(String.fromCharCode(i));
        t = i;
    }

    return n.join("");
};

let vmp_bin = "AxjgB5MAnACoAJwBpAAAABAAIAKcAqgAMAq0AzRJZAZwUpwCqACQACACGAKcBKAAIAOcBagAIAQYAjAUGgKcBqFAuAc5hTSHZAZwqrAIGgA0QJEAJAAYAzAUGgOcCaFANRQ0R2QGcOKwChoANECRACQAsAuQABgDnAmgAJwMgAGcDYwFEAAzBmAGcSqwDhoANECRACQAGAKcD6AAGgKcEKFANEcYApwRoAAxB2AGcXKwEhoANECRACQAGAKcE6AAGgKcFKFANEdkBnGqsBUaADRAkQAkABgCnBagAGAGcdKwFxoANECRACQAGAKcGKAAYAZx+rAZGgA0QJEAJAAYA5waoABgBnIisBsaADRAkQAkABgCnBygABoCnB2hQDRHZAZyWrAeGgA0QJEAJAAYBJwfoAAwFGAGcoawIBoANECRACQAGAOQALAJkAAYBJwfgAlsBnK+sCEaADRAkQAkABgDkACwGpAAGAScH4AJbAZy9rAiGgA0QJEAJACwI5AAGAScH6AAkACcJKgAnCWgAJwmoACcJ4AFnA2MBRAAMw5gBnNasCgaADRAkQAkABgBEio0R5EAJAGwKSAFGACcKqAAEgM0RCQGGAYSATRFZAZzshgAtCs0QCQAGAYSAjRFZAZz1hgAtCw0QCQAEAAgB7AtIAgYAJwqoAASATRBJAkYCRIANEZkBnYqEAgaBxQBOYAoBxQEOYQ0giQKGAmQABgAnC6ABRgBGgo0UhD/MQ8zECALEAgaBxQBOYAoBxQEOYQ0gpEAJAoYARoKNFIQ/zEPkAAgChgLGgkUATmBkgAaAJwuhAUaCjdQFAg5kTSTJAsQCBoHFAE5gCgHFAQ5hDSCkQAkChgBGgo0UhD/MQ+QACAKGAsaCRQCOYGSABoAnC6EBRoKN1AUEDmRNJMkCxgFGgsUPzmPkgAaCJwvhAU0wCQFGAUaCxQGOZISPzZPkQAaCJwvhAU0wCQFGAUaCxQMOZISPzZPkQAaCJwvhAU0wCQFGAUaCxQSOZISPzZPkQAaCJwvhAU0wCQFGAkSAzRBJAlz/B4FUAAAAwUYIAAIBSITFQkTERwABi0GHxITAAAJLwMSGRsXHxMZAAk0Fw8HFh4NAwUABhU1EBceDwAENBcUEAAGNBkTGRcBAAFKAAkvHg4PKz4aEwIAAUsACDIVHB0QEQ4YAAsuAzs7AAoPKToKDgAHMx8SGQUvMQABSAALORoVGCQgERcCAxoACAU3ABEXAgMaAAsFGDcAERcCAxoUCgABSQAGOA8LGBsPAAYYLwsYGw8AAU4ABD8QHAUAAU8ABSkbCQ4BAAFMAAktCh8eDgMHCw8AAU0ADT4TGjQsGQMaFA0FHhkAFz4TGjQsGQMaFA0FHhk1NBkCHgUbGBEPAAFCABg9GgkjIAEmOgUHDQ8eFSU5DggJAwEcAwUAAUMAAUAAAUEADQEtFw0FBwtdWxQTGSAACBwrAxUPBR4ZAAkqGgUDAwMVEQ0ACC4DJD8eAx8RAAQ5GhUYAAFGAAAABjYRExELBAACWhgAAVoAQAg/PTw0NxcQPCQ5C3JZEBs9fkcnDRcUAXZia0Q4EhQgXHojMBY3MWVCNT0uDhMXcGQ7AUFPHigkQUwQFkhaAkEACjkTEQspNBMZPC0ABjkTEQsrLQ==";
let opcode = [];
for (let t = atob(vmp_bin), n = t.charCodeAt(0) << 8 | t.charCodeAt(1), i = 2; i < n + 2; i += 2) {
    opcode.push(t.charCodeAt(i) << 8 | t.charCodeAt(i + 1));
}
opcode.push(0);
let opstr = [];
for (let t = atob(vmp_bin), n = t.charCodeAt(0) << 8 | t.charCodeAt(1), a = n + 2; a < t.length;) {
    var c = t.charCodeAt(a) << 8 | t.charCodeAt(a + 1),
        u = t.slice(a + 2, a + 2 + c);
    opstr.push(k(u));
    a += c + 2;
}

let ast = parser.parse("");
ast.program.body.push(types.variableDeclaration("var", [
    types.variableDeclarator(types.identifier('global_0')),
    types.variableDeclarator(types.identifier('global_1')),
    types.variableDeclarator(types.identifier('global_2')),
    types.variableDeclarator(types.identifier('global_3'))
]));
```

接着后面就是一个大的 switch，针对每条指令，生成一个 js 代码，例如第一条的操作类型是 14

```
case 14:
                let into_code = 4095 & opcode[pc];
                ast.program.body.push(types.functionDeclaration(types.identifier('_0x' + into_code), [], types.blockStatement([])));
                blockstatement.push(types.expressionStatement(types.assignmentExpression("=", types.identifier('global_3'), types.identifier('_0x' + into_code))));
                pc++;
                break;
```

这是一个函数初始化指令，需要生成一个函数，并且赋值给全局变量，如此类推，把所有的指令都还原出来，相当于自己实现了一个解释器

```
const parser = require("@babel/parser");
// 为parser提供模板引擎
const template = require("@babel/template").default;
// 遍历AST
const traverse = require("@babel/traverse").default;
// 操作节点，比如判断节点类型，生成新的节点等
const types = require("@babel/types");
// 将语法树转换为源代码
const generator = require("@babel/generator");
// 操作文件
const fs = require("fs");

var k = function (e) {
    for (var t = 66, n = [], r = 0; r < e.length; r++) {
        var i = 24 ^ e.charCodeAt(r) ^ t;
        n.push(String.fromCharCode(i));
        t = i;
    }

    return n.join("");
};

let vmp_bin = "AxjgB5MAnACoAJwBpAAAABAAIAKcAqgAMAq0AzRJZAZwUpwCqACQACACGAKcBKAAIAOcBagAIAQYAjAUGgKcBqFAuAc5hTSHZAZwqrAIGgA0QJEAJAAYAzAUGgOcCaFANRQ0R2QGcOKwChoANECRACQAsAuQABgDnAmgAJwMgAGcDYwFEAAzBmAGcSqwDhoANECRACQAGAKcD6AAGgKcEKFANEcYApwRoAAxB2AGcXKwEhoANECRACQAGAKcE6AAGgKcFKFANEdkBnGqsBUaADRAkQAkABgCnBagAGAGcdKwFxoANECRACQAGAKcGKAAYAZx+rAZGgA0QJEAJAAYA5waoABgBnIisBsaADRAkQAkABgCnBygABoCnB2hQDRHZAZyWrAeGgA0QJEAJAAYBJwfoAAwFGAGcoawIBoANECRACQAGAOQALAJkAAYBJwfgAlsBnK+sCEaADRAkQAkABgDkACwGpAAGAScH4AJbAZy9rAiGgA0QJEAJACwI5AAGAScH6AAkACcJKgAnCWgAJwmoACcJ4AFnA2MBRAAMw5gBnNasCgaADRAkQAkABgBEio0R5EAJAGwKSAFGACcKqAAEgM0RCQGGAYSATRFZAZzshgAtCs0QCQAGAYSAjRFZAZz1hgAtCw0QCQAEAAgB7AtIAgYAJwqoAASATRBJAkYCRIANEZkBnYqEAgaBxQBOYAoBxQEOYQ0giQKGAmQABgAnC6ABRgBGgo0UhD/MQ8zECALEAgaBxQBOYAoBxQEOYQ0gpEAJAoYARoKNFIQ/zEPkAAgChgLGgkUATmBkgAaAJwuhAUaCjdQFAg5kTSTJAsQCBoHFAE5gCgHFAQ5hDSCkQAkChgBGgo0UhD/MQ+QACAKGAsaCRQCOYGSABoAnC6EBRoKN1AUEDmRNJMkCxgFGgsUPzmPkgAaCJwvhAU0wCQFGAUaCxQGOZISPzZPkQAaCJwvhAU0wCQFGAUaCxQMOZISPzZPkQAaCJwvhAU0wCQFGAUaCxQSOZISPzZPkQAaCJwvhAU0wCQFGAkSAzRBJAlz/B4FUAAAAwUYIAAIBSITFQkTERwABi0GHxITAAAJLwMSGRsXHxMZAAk0Fw8HFh4NAwUABhU1EBceDwAENBcUEAAGNBkTGRcBAAFKAAkvHg4PKz4aEwIAAUsACDIVHB0QEQ4YAAsuAzs7AAoPKToKDgAHMx8SGQUvMQABSAALORoVGCQgERcCAxoACAU3ABEXAgMaAAsFGDcAERcCAxoUCgABSQAGOA8LGBsPAAYYLwsYGw8AAU4ABD8QHAUAAU8ABSkbCQ4BAAFMAAktCh8eDgMHCw8AAU0ADT4TGjQsGQMaFA0FHhkAFz4TGjQsGQMaFA0FHhk1NBkCHgUbGBEPAAFCABg9GgkjIAEmOgUHDQ8eFSU5DggJAwEcAwUAAUMAAUAAAUEADQEtFw0FBwtdWxQTGSAACBwrAxUPBR4ZAAkqGgUDAwMVEQ0ACC4DJD8eAx8RAAQ5GhUYAAFGAAAABjYRExELBAACWhgAAVoAQAg/PTw0NxcQPCQ5C3JZEBs9fkcnDRcUAXZia0Q4EhQgXHojMBY3MWVCNT0uDhMXcGQ7AUFPHigkQUwQFkhaAkEACjkTEQspNBMZPC0ABjkTEQsrLQ==";
let opcode = [];
for (let t = atob(vmp_bin), n = t.charCodeAt(0) << 8 | t.charCodeAt(1), i = 2; i < n + 2; i += 2) {
    opcode.push(t.charCodeAt(i) << 8 | t.charCodeAt(i + 1));
}
opcode.push(0);
let opstr = [];
for (let t = atob(vmp_bin), n = t.charCodeAt(0) << 8 | t.charCodeAt(1), a = n + 2; a < t.length;) {
    var c = t.charCodeAt(a) << 8 | t.charCodeAt(a + 1),
        u = t.slice(a + 2, a + 2 + c);
    opstr.push(k(u));
    a += c + 2;
}

let ast = parser.parse("");
ast.program.body.push(types.variableDeclaration("var", [
    types.variableDeclarator(types.identifier('global_0')),
    types.variableDeclarator(types.identifier('global_1')),
    types.variableDeclarator(types.identifier('global_2')),
    types.variableDeclarator(types.identifier('global_3'))
]));

function get_blockstatement(pc, Local, blockstatement){
    let Stack = [];
    while (opcode[pc] !== 0){
        let t, s, i, h, a, c, n;
        switch ((61440 & opcode[pc]) >> 12) {
            case 0:
                break;
            case 1:
                break;
            case 2:
                break;
            case 3:
                break;
            case 4:
                break;
            case 5:
                break;
            case 6:
                break;
            case 7:
                break;
            case 8:
                break;
            case 9:
                t = (4095 & opcode[pc]) >> 10;
                s = (1023 & opcode[pc]) >> 8;
                i = 1023 & opcode[pc];
                h = 63 & opcode[pc];
                switch (t) {
                    case 0:
                        Stack.push(types.identifier('global_' + s));
                        pc++;
                        break;

                    case 1:
                        break;

                    case 2:
                        break;

                    case 3:
                        Stack.push(types.stringLiteral(opstr[h]));
                        pc++;
                        break
                }
                break;
            case 10:
                t = (4095 & opcode[pc]) >> 10;
                a = (1023 & opcode[pc]) >> 8;
                c = (255 & opcode[pc]) >> 6;
                switch (t) {
                    case 0:
                        break;
                    case 1:
                        s = Stack.pop(), i = Stack.pop();
                        blockstatement.push(
                            types.expressionStatement(
                                types.assignmentExpression(
                                    "=",
                                    types.memberExpression(
                                        types.identifier('global_' + c),
                                        s,
                                        true
                                    ),
                                    i
                                )
                            )
                        );
                        pc++;
                        break;
                    case 2:
                        h = Stack.pop();
                        blockstatement.push(types.expressionStatement(types.assignmentExpression("=", types.identifier('global_' + a), types.callExpression(types.identifier('eval'), [h]))));
                        pc++;
                        break;
                }
                break;
            case 11:
                break;
            case 12:
                break;
            case 13:
                break;
            case 14:
                let into_code = 4095 & opcode[pc];
                ast.program.body.push(types.functionDeclaration(types.identifier('_0x' + into_code), [], types.blockStatement([])));
                blockstatement.push(types.expressionStatement(types.assignmentExpression("=", types.identifier('global_3'), types.identifier('_0x' + into_code))));
                pc++;
                break;
            case 15:
                break;
        }
    }
    return blockstatement
}

ast.program.body.push(types.expressionStatement(types.callExpression(types.functionExpression(null, [], types.blockStatement(get_blockstatement(0, [], []))), [])));

let code = generator.default(ast).code;
console.log(code);
```

运行后可以得到初始化部分的伪代码

```
var global_0, global_1, global_2, global_3;

function _0x7() {}

(function () {
  global_3 = _0x7;
  global_0 = eval("__g");
  global_0["_encrypt"] = global_3;
})();
```

从伪代码中可以看出，确实是把基址是【0x7】的函数，动态绑定到【__g】对象的【_encrypt】属性下面，上面我们生成签名的时候，确实也是调用的这个

继续完善解释器的代码，可以得到下面的伪代码

```
var global_0, global_1, global_2, global_3;

function _0x7(params_0) {
  var local_0 = params_0;
  global_0 = 0;
  var local_2 = global_0;
  global_0 = eval("window");
  global_0 = t(global_0);
  global_1 = "undefined";
  global_1 = global_0 !== global_1;

  if (global_1) {
    global_0 = eval("window");
    local_2 = global_0;
  }

  global_0 = local_2;
  global_0 = global_0["navigator"];
  var local_3 = global_0;
  global_0 = eval("Object");
  var local_4 = global_0;
  global_0 = local_2;
  global_0 = !global_0;
  global_1 = local_2;
  global_1 = global_1["name"];
  global_2 = "nodejs";
  global_2 = global_1 == global_2;
  global_1 = global_0 || global_2;

  if (global_1) {
    global_0 = "\x10";
    global_1 = local_0;
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = local_3;
  global_0 = !global_0;
  global_1 = local_3;
  global_1 = global_1["userAgent"];
  global_1 = !global_1;
  global_1 = global_0 || global_1;

  if (global_1) {
    global_0 = "\x11";
    global_1 = local_0;
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = "headless";
  global_0 = local_3;
  global_0 = global_0["userAgent"];
  global_3 = global_0["toLowerCase"]();
  global_3 = global_3["indexOf"](global_0);
  global_0 = 0;
  global_0 = global_3 >= global_0;

  if (global_0) {
    global_0 = "\x12";
    global_1 = local_0;
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = local_2;
  global_0 = global_0["callPhantom"];
  global_1 = local_2;
  global_1 = global_1["_phantom"];
  global_1 = global_0 || global_1;
  global_0 = local_2;
  global_0 = global_0["__phantomas"];
  global_0 = global_1 || global_0;

  if (global_0) {
    global_0 = "\x13";
    global_1 = local_0;
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = local_2;
  global_0 = global_0["buffer"];
  global_1 = local_2;
  global_1 = global_1["Buffer"];
  global_1 = global_0 || global_1;

  if (global_1) {
    global_0 = "\x14";
    global_1 = local_0;
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = local_2;
  global_0 = global_0["emit"];

  if (global_0) {
    global_0 = "\x15";
    global_1 = local_0;
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = local_2;
  global_0 = global_0["spawn"];

  if (global_0) {
    global_0 = "\x16";
    global_1 = local_0;
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = local_3;
  global_0 = global_0["webdriver"];

  if (global_0) {
    global_0 = "\x17";
    global_1 = local_0;
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = local_2;
  global_0 = global_0["domAutomation"];
  global_1 = local_2;
  global_1 = global_1["domAutomationController"];
  global_1 = global_0 || global_1;

  if (global_1) {
    global_0 = "\x18";
    global_1 = local_0;
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = local_4;
  global_0 = global_0["getOwnPropertyDescriptor"];
  global_0 = !global_0;

  if (global_0) {
    global_0 = "\x19";
    global_1 = local_0;
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = local_3;
  global_0 = "userAgent";
  global_0 = local_4;
  global_3 = global_0["getOwnPropertyDescriptor"](global_0, global_0);

  if (global_3) {
    global_0 = "\x1A";
    global_1 = local_0;
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = local_3;
  global_0 = "webdriver";
  global_0 = local_4;
  global_3 = global_0["getOwnPropertyDescriptor"](global_0, global_0);

  if (global_3) {
    global_0 = "\x1B";
    global_1 = local_0;
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = "[native code]";
  global_0 = local_4;
  global_0 = global_0["getOwnPropertyDescriptor"];
  global_0 = eval("Function");
  global_0 = global_0["prototype"];
  global_0 = global_0["toString"];
  global_3 = global_0["call"](global_0);
  global_3 = global_3["indexOf"](global_0);
  global_0 = 0;
  global_0 = global_3 < global_0;

  if (global_0) {
    global_0 = "\x1C";
    global_1 = local_0;
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = local_1;
  global_1 = 42;
  global_1 = global_0 || global_1;
  var local_1 = global_1;
  global_0 = "";
  var local_5 = global_0;
  global_0 = local_0;
  global_0 = global_0["length"];
  global_1 = 3;
  global_1 = global_0 % global_1;
  var local_6 = global_1;
  global_0 = local_6;
  global_1 = 1;
  global_1 = global_0 == global_1;

  if (global_1) {
    global_0 = local_0;
    global_1 = "\0\0";
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = local_6;
  global_1 = 2;
  global_1 = global_0 == global_1;

  if (global_1) {
    global_0 = local_0;
    global_1 = "\0";
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = 0;
  var local_7 = global_0;
  global_0 = "RuPtXwxpThIZ0qyz_9fYLCOV8B1mMGKs7UnFHgN3iDaWAJE-Qrk2ecSo6bjd4vl5";
  var local_8 = global_0;
  global_0 = local_0;
  global_0 = global_0["length"];
  global_1 = 1;
  global_1 = global_0 - global_1;
  var local_9 = global_1;
  global_0 = local_9;
  global_1 = 0;
  global_1 = global_0 >= global_1;

  for (; global_1;) {
    global_0 = 8;
    global_1 = local_7;
    global_2 = 1;
    global_2 = global_1 + global_2;
    local_7 = global_2;
    global_2 = 4;
    global_2 = global_1 % global_2;
    global_1 = global_0 * global_2;
    var local_10 = global_1;
    global_0 = local_9;
    global_0 = local_0;
    global_3 = global_0["charCodeAt"](global_0);
    global_0 = local_1;
    global_1 = local_10;
    global_1 = global_0 >>> global_1;
    global_0 = 255;
    global_0 = global_1 & global_0;
    global_0 = global_3 ^ global_0;
    var local_11 = global_0;
    global_0 = 8;
    global_1 = local_7;
    global_2 = 1;
    global_2 = global_1 + global_2;
    local_7 = global_2;
    global_2 = 4;
    global_2 = global_1 % global_2;
    global_1 = global_0 * global_2;
    local_10 = global_1;
    global_0 = local_1;
    global_1 = local_10;
    global_1 = global_0 >>> global_1;
    global_0 = 255;
    global_0 = global_1 & global_0;
    local_10 = global_0;
    global_0 = local_11;
    global_1 = local_9;
    global_2 = 1;
    global_2 = global_1 - global_2;
    global_1 = local_0;
    global_3 = global_1["charCodeAt"](global_2);
    global_1 = local_10;
    global_1 = global_3 ^ global_1;
    global_2 = 8;
    global_2 = global_1 << global_2;
    global_1 = global_0 | global_2;
    local_11 = global_1;
    global_0 = 8;
    global_1 = local_7;
    global_2 = 1;
    global_2 = global_1 + global_2;
    local_7 = global_2;
    global_2 = 4;
    global_2 = global_1 % global_2;
    global_1 = global_0 * global_2;
    local_10 = global_1;
    global_0 = local_1;
    global_1 = local_10;
    global_1 = global_0 >>> global_1;
    global_0 = 255;
    global_0 = global_1 & global_0;
    local_10 = global_0;
    global_0 = local_11;
    global_1 = local_9;
    global_2 = 2;
    global_2 = global_1 - global_2;
    global_1 = local_0;
    global_3 = global_1["charCodeAt"](global_2);
    global_1 = local_10;
    global_1 = global_3 ^ global_1;
    global_2 = 16;
    global_2 = global_1 << global_2;
    global_1 = global_0 | global_2;
    local_11 = global_1;
    global_0 = local_5;
    global_1 = local_11;
    global_2 = 63;
    global_2 = global_1 & global_2;
    global_1 = local_8;
    global_3 = global_1["charAt"](global_2);
    global_1 = global_0 + global_3;
    local_5 = global_1;
    global_0 = local_5;
    global_1 = local_11;
    global_2 = 6;
    global_2 = global_1 >>> global_2;
    global_1 = 63;
    global_1 = global_2 & global_1;
    global_1 = local_8;
    global_3 = global_1["charAt"](global_1);
    global_1 = global_0 + global_3;
    local_5 = global_1;
    global_0 = local_5;
    global_1 = local_11;
    global_2 = 12;
    global_2 = global_1 >>> global_2;
    global_1 = 63;
    global_1 = global_2 & global_1;
    global_1 = local_8;
    global_3 = global_1["charAt"](global_1);
    global_1 = global_0 + global_3;
    local_5 = global_1;
    global_0 = local_5;
    global_1 = local_11;
    global_2 = 18;
    global_2 = global_1 >>> global_2;
    global_1 = 63;
    global_1 = global_2 & global_1;
    global_1 = local_8;
    global_3 = global_1["charAt"](global_1);
    global_1 = global_0 + global_3;
    local_5 = global_1;
    global_0 = local_9;
    global_1 = 3;
    global_1 = global_0 - global_1;
    local_9 = global_1;
    global_0 = local_9;
    global_1 = 0;
    global_1 = global_0 >= global_1;
  }

  global_3 = local_5;
  return global_3;
}

(function () {
  global_3 = _0x7;
  global_0 = eval("__g");
  global_0["_encrypt"] = global_3;
})();
```

伪代码是没有办法直接运行的，需要继续优化，才可能可以运行起来。但是在这里也可以看到比较直观的逻辑了。所以能不能运行关系并不大

可以看出，前面一半左右，都是在检测环境，如果检测不通过，就会添加一个字节，导致最后的结果不一样。例如，如果你是使用自动化工具来爬取的，那么很有可能【webdriver】这个属性就为真，那么参数就会被添加一个【\x17】，服务端解码结果发现有一个【\x17】，就知道是自动化工具，而不是正常的浏览器，那么就不返回数据了。

这时就有一个想法了，如果是 js 代码，那么我可以直接把环境检测的代码注释掉，那么就直接不需要环境也可以得出正确的结果，那么在 jsvmp 中，能不能有类似的操作呢？那么就要巧妙的使用到 pc 寄存器了。

![](https://img-blog.csdnimg.cn/1ae3be4685b646b2822471ff7b08def8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
上图这里是最后一个环境检测，离这里最近的字符串是【indexOf】，那么在解密字符串的地方下一个断点

![](https://img-blog.csdnimg.cn/b3c3f944e3454a6f9f7cd5fc83b5c7a8.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
当第二次断下时，进行单步调试

![](https://img-blog.csdnimg.cn/86c3f643831b455ebc0e88b4bb748f57.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
当来到下一个跳转指令的时候，pc 寄存器的指向是从 209 到 214，也就是说 214 的地方才是结算签名的开始，前面的都是环境检测，那么知道真正开始的地方，我们就可以修改 pc 寄存器的值了，当 pc 寄存器的值为 7 的时候，也就是函数的入口，把它直接改成 214

![](https://img-blog.csdnimg.cn/115595916446499987ce393c328c64de.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)  
添加完这个后，就删除前面添加的 jsdom，再次运行，发现结果也是正确的。绝了，这居然真的可以，到这里第二种方法就完成了

第三种解决方案 - 算法还原
--------------

根据前面我们已经知道，只有后半部分的伪代码才是真实的逻辑，那么我们就只看后半部分来还原就好了

```
global_0 = local_1;
  global_1 = 42;
  global_1 = global_0 || global_1;
  var local_1 = global_1;
  global_0 = "";
  var local_5 = global_0;
  global_0 = local_0;
  global_0 = global_0["length"];
  global_1 = 3;
  global_1 = global_0 % global_1;
  var local_6 = global_1;
  global_0 = local_6;
  global_1 = 1;
  global_1 = global_0 == global_1;

  if (global_1) {
    global_0 = local_0;
    global_1 = "\0\0";
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }

  global_0 = local_6;
  global_1 = 2;
  global_1 = global_0 == global_1;

  if (global_1) {
    global_0 = local_0;
    global_1 = "\0";
    global_1 = global_0 + global_1;
    local_0 = global_1;
  }
```

这里大致可以看出是对参数进行填充成 3 的倍数

```
global_0 = 0;
  var local_7 = global_0;
  global_0 = "RuPtXwxpThIZ0qyz_9fYLCOV8B1mMGKs7UnFHgN3iDaWAJE-Qrk2ecSo6bjd4vl5";
  var local_8 = global_0;
  global_0 = local_0;
  global_0 = global_0["length"];
  global_1 = 1;
  global_1 = global_0 - global_1;
  var local_9 = global_1;
  global_0 = local_9;
  global_1 = 0;
  global_1 = global_0 >= global_1;
```

然后初始化了一个长度 64 的字符串，根据前后可以猜测很有可能是一个 base64 的编码算法，只不过不是一个标准的 base64，而是修改了编码表以及一些算法细节

继续往后看，是一个循环，这里就是编码的开始

```
global_0 = 8;
    global_1 = local_7;
    global_2 = 1;
    global_2 = global_1 + global_2;
    local_7 = global_2;
    global_2 = 4;
    global_2 = global_1 % global_2;
    global_1 = global_0 * global_2;
    var local_10 = global_1;
    global_0 = local_9;
    global_0 = local_0;
    global_3 = global_0["charCodeAt"](global_0);
```

这里是从字符串后后面开始取值，正常的 base64 编码都是从前面开始的，但是这里是从后面开始的，这是一个魔改点，base64 编码是三个字节一个单位，所以 python 还原的话就要从后面开始三个字节一组反转

```
for i in range(len(in_put) - 1, 0, -3): 
    b64_in += in_put[i - 2: i + 1]
```

接着继续看后面的操作

```
global_0 = local_1;
 global_1 = local_10;
 global_1 = global_0 >>> global_1;
 global_0 = 255;
 global_0 = global_1 & global_0;
 global_0 = global_3 ^ global_0;
 var local_11 = global_0;
 global_0 = 8;
 global_1 = local_7;
 global_2 = 1;
 global_2 = global_1 + global_2;
 local_7 = global_2;
 global_2 = 4;
 global_2 = global_1 % global_2;
 global_1 = global_0 * global_2;
 local_10 = global_1;
 global_0 = local_1;
 global_1 = local_10;
 global_1 = global_0 >>> global_1;
 global_0 = 255;
 global_0 = global_1 & global_0;
 local_10 = global_0;
 global_0 = local_11;
 global_1 = local_9;
 global_2 = 1;
 global_2 = global_1 - global_2;
 global_1 = local_0;
```

这里的实际逻辑经过优化后，就是 12 个字节为一组。然后 2，4，6 的位置与 42 进行异或，这里是第二个魔改点

```
for i inrange(0, len(b64_in), 12): 
    b64_in[i + 2], b64_in[i + 4], b64_in[i + 6] = b64_in[i + 2] ^ 42, b64_in[i + 4] ^ 42, b64_in[i + 6] ^ 42
```

接着就是 base64 编码，但是编码的端序不相同，所以每一组的端序都要修改，这是第三个魔改点，完整的 python 代码如下

```
def x96_b64encode(in_put: str) -> str:
        in_put = in_put.encode('utf-8')
        while len(in_put) % 3 != 0:
            in_put += bytes([0])

        table1 = list('RuPtXwxpThIZ0qyz_9fYLCOV8B1mMGKs7UnFHgN3iDaWAJE-Qrk2ecSo6bjd4vl5')
        table2 = list('ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/')
        table3 = {table2[v]: table1[v] for v in range(len(table1))}

        b64_in = bytearray()
        for i in range(len(in_put) - 1, 0, -3):
            b64_in += in_put[i - 2: i + 1]
        for i in range(0, len(b64_in), 12):
            b64_in[i + 2], b64_in[i + 4], b64_in[i + 6] = b64_in[i + 2] ^ 42, b64_in[i + 4] ^ 42, b64_in[i + 6] ^ 42

        b64_out = ''.join(list(map(lambda n: table3[n], list(base64.b64encode(b64_in).decode()))))
        return ''.join([b64_out[i: i + 4][::-1] for i in range(0, len(b64_out), 4)])
```

测试这个函数，与浏览器生成的值是一样的，所以到这里，第三种方法也解决了。到此为止，对 jsvmp 也有了初步的了解。

参考文章
----

1.[H5 应用加固防破解 - JS 虚拟机保护方案](https://blog.ntan520.com/article/11179)  
2. [某网站视频加密的 wasm 略谈（二）](https://www.52pojie.cn/thread-1535897-1-1.html)

更多内容欢迎加入我的星球

![](https://img-blog.csdnimg.cn/3ebf6f7a7b3047c2ac8d937178ed6bc2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5riU5ruS,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)