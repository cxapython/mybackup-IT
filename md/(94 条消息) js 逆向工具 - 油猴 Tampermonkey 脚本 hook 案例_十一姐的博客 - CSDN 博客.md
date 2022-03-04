> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/weixin_43411585/article/details/109798452?spm=1001.2014.3001.5502)

### 目录

*   *   *   *   [一、油猴下载与安装](#_1)
            *   [二、油猴脚本免费使用网站](#_5)
            *   [三、油猴脚本编写介绍](#_10)
            *   *   [1、添加新脚本](#1_11)
                *   [2、油猴脚本注释内容解释](#2_15)
                *   [3、编写油猴脚本的基本步骤](#3_33)
                *   [4、油猴脚本调试测试](#4_36)
            *   [四、hook 之 js 逆向案例](#hookjs_40)
            *   *   [1、hook 之 window 属性案例](#1hookwindow_45)
                *   [2、hook 之 cookie 生成案例](#2hookcookie_87)
            *   [五、hook 的脚本合集](#hook_121)
            *   *   [1、hook 之 cookie](#1hookcookie_122)
                *   [2、hook 之 window 属性](#2hookwindow_170)
                *   [3、hook 之 eval](#3hookeval_211)
                *   [4、hook 之 setInterval](#4hooksetInterval_231)
                *   [5、hook 之 console](#5hookconsole_243)
                *   [5、hook 函数中 debugger](#5hookdebugger_256)
                *   [6、hook 之 split](#6hooksplit_267)
                *   [7、hook 之覆盖原函数总结](#7hook_281)

#### 一、油猴下载与安装

*   油猴：Tampermonkey 是一款免费的浏览器扩展和最为流行的用户脚本管理器， 可以通过不同的 “脚本” 实现数十甚至上百个功能，比如视频去水印，去广告，js 逆向定位加密参数生成位置等
*   下载与安装：[下载. crx 插件](https://www.chrome666.com/chrome-extension/tampermonkey.html) ，安装：谷歌 > 右上角 > 更多工具 > 扩展程序，直接将 crx 文件拖入如下图页面  
    ![](https://img-blog.csdnimg.cn/20201119072614274.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQxMTU4NQ==,size_16,color_FFFFFF,t_70#pic_center)

#### 二、油猴脚本免费使用网站

*   [Userscript.Zone Search](https://www.userscript.zone/?utm_source=tm.net&utm_medium=scripts&utm_campaign=0)：是一个新网站，允许通过输入合适的 URL 或域来搜索用户脚本
*   [Greasy Fork](https://greasyfork.org/zh-CN)：最受欢迎的后起之秀，提供用户脚本的网站，可实现去掉视频播放广告，去水印等多种功能，可以直接安装使用，储存库中有大量的脚本资源
*   [OpenUserJS](https://openuserjs.org/)：继 GreasyFork 之后开始创办，在其储存库中也拥有大量的脚本资源
*   [serscripts Mirror](https://userscripts-mirror.org/)

#### 三、油猴[脚本编写](https://so.csdn.net/so/search?q=%E8%84%9A%E6%9C%AC%E7%BC%96%E5%86%99&spm=1001.2101.3001.7020)介绍

##### 1、添加新脚本

*   右上角打开油猴 > 添加新脚本 > 即为如下的编辑脚本页面  
    ![](https://img-blog.csdnimg.cn/1c15c1785de942de82303d9802fabaa4.png)  
    ![](https://img-blog.csdnimg.cn/20201120074657320.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQxMTU4NQ==,size_16,color_FFFFFF,t_70#pic_center)

##### 2、油猴脚本注释内容解释

*   `@match`这个注释的内容最为重要，它决定了你的脚本应用到哪个具体的网页，还是应用到所有的网页
*   `@run-at`确定了脚本的注入时机，在 js 逆向中也很重要
*   [油猴脚本所有注释介绍详情](https://www.tampermonkey.net/documentation.php)

<table><thead><tr><th align="left">属性名</th><th align="left">作用</th></tr></thead><tbody><tr><td align="left">@name</td><td align="left">油猴脚本的名字</td></tr><tr><td align="left">@namespace</td><td align="left">命名空间，用来区分相同名称的脚本，一般写成作者名字或者网址就可以了</td></tr><tr><td align="left">@version</td><td align="left">脚本版本，油猴脚本的更新会读取这个版本号</td></tr><tr><td align="left">@description</td><td align="left">描述，用来告诉用户这个脚本是干什么用的</td></tr><tr><td align="left">@author</td><td align="left">作者名字</td></tr><tr><td align="left"><code>@match</code></td><td align="left">只有匹配的网址才会执行对应的脚本，例如<code>*、http://*、http://www.baidu.com/*</code>等</td></tr><tr><td align="left">@grant</td><td align="left">指定脚本运行所需权限，如果脚本拥有相应的权限，就可以调用油猴扩展提供的 API 与浏览器进行交互。如果设置为 none 的话，则不使用沙箱环境，脚本会直接运行在网页的环境中，这时候无法使用大部分油猴扩展的 API。如果不指定的话，油猴会默认添加几个最常用的 API</td></tr><tr><td align="left">@require</td><td align="left">如果脚本依赖其他 js 库的话，可以使用 require 指令，在运行脚本之前先加载其他库，常见用法是加载 jquery，导库，和 node 差不多，相当于导入外部的脚本</td></tr><tr><td align="left"><code>@run-at</code></td><td align="left">脚本注入时机，这个比较重要，有时候是能不能 hook 到的关键，<code>document-start</code>：网页开始时；<code>document-body</code>：body 出现时；<code>document-end</code>：载入时或者之后执行；<code>document-idle</code>：载入完成后执行，默认选项</td></tr><tr><td align="left">@connect</td><td align="left">当用户使用 GM_xmlhttpRequest 请求远程数据的时候，需要使用 connect 指定允许访问的域名，支持域名、子域名、IP 地址以及 * 通配符</td></tr><tr><td align="left">@updateURL</td><td align="left">脚本更新网址，当油猴扩展检查更新的时候，会尝试从这个网址下载脚本，然后比对版本号确认是否更新</td></tr></tbody></table>

##### 3、编写油猴脚本的基本步骤

*   编写到`// Your code here`那里即可  
    ![](https://img-blog.csdnimg.cn/20201120080550689.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQxMTU4NQ==,size_16,color_FFFFFF,t_70#pic_center)

##### 4、油猴脚本调试测试

*   利用浏览器的调试功能，在脚本需要调试的地方添加`debugger;`语句，在相应网页刷新后，即可利用 F12 开发者工具进行单步调试、监视变量等操作了  
    ![](https://img-blog.csdnimg.cn/20201120080307891.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQxMTU4NQ==,size_16,color_FFFFFF,t_70#pic_center)

#### 四、hook 之 js 逆向案例

*   可用于：定位 window 属性加密参数，定位 cookie 生成位置、修改原函数等
*   `借助油猴的hook`：你可以将 hook 的脚本放到油猴里面执行
*   `直接用谷歌控制台console界面进行hook`：hook 时机最好选择你看见的第一个 js 文件的第一行的时候下断点，然后在控制台 console 界面就注入
*   [hook 案例推荐](https://www.52pojie.cn/forum.php?mod=viewthread&tid=1519457&extra=page%3D1%26filter%3Dtypeid%26typeid%3D378)

##### 1、hook 之 window 属性案例

*   （1）[目标网站](https://flight.qunar.com/)，目标定位加密参数 pre：  
    ![](https://img-blog.csdnimg.cn/20201119081356760.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQxMTU4NQ==,size_16,color_FFFFFF,t_70#pic_center)
*   （2）全局搜索’pre’参数，发现`pre的值就是window._pt_的值`，但是直接调试需要耗一些时间才能定位到加密位置，通过油猴 hook 可以轻松定位到加密位置  
    ![](https://img-blog.csdnimg.cn/20201119081807728.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQxMTU4NQ==,size_16,color_FFFFFF,t_70#pic_center)
*   （3）首先，添加油猴脚本如下，保存该脚本并启用，然后切换到目标网页：
    *   注意 hook 的域名为`@match https://flight.qunar.com/*`
    *   hook 的注入时机为文档开始时`@run-at document-start`
    *   [Object.defineProperty()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty)：替换一个对象的属性，并返回此对象，属性里面可能存的是方法，也可能存的就是一个值（getter，setter）
    *   而该脚本的 hook 点就是_pt_属性的 get/set 方法，`在set处即_pt_属性被赋值时，debugger住`
        
        ```
        // ==UserScript==
        // @name         定位pre加密参数
        // @namespace    http://tampermonkey.net/
        // @version      0.1
        // @description  try to take over the world!
        // @author       Shirmay1
        // @run-at       document-start
        // @match        https://flight.qunar.com/*
        // @grant        none
        // ==/UserScript==
        
        (function() {
            'use strict';
            var pre = "";
            Object.defineProperty(window, '_pt_', {
                get: function() {
                    console.log('Getting window.属性');
                    return pre
                },
                set: function(val) {
                    console.log('Setting window.属性', val);
                    debugger ;
                    pre = val;
                }
            })
        })();
        ```
        
*   （4）接着，F12 打开谷歌开发者工具，然后点击目标网站搜索按钮，油猴脚本随之执行，即可轻松定位到该加密参数位置，如下图  
    ![](https://img-blog.csdnimg.cn/20201119082145422.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQxMTU4NQ==,size_16,color_FFFFFF,t_70#pic_center)
*   （5）最后，通过点击右侧调用栈回溯前一个函数，即可精准定位到`window._pt_的值也就是pre的值`  
    ![](https://img-blog.csdnimg.cn/20201119082426469.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQxMTU4NQ==,size_16,color_FFFFFF,t_70#pic_center)

##### 2、hook 之 cookie 生成案例

*   （1）[目标网站](https://hotels.ctrip.com/hotel/shanghai2/star5#ctm_ref=ctr_hp_sb_lst)，定位 _bfa 这个 cookie 参数生成位置  
    ![](https://img-blog.csdnimg.cn/20201124081655270.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzQxMTU4NQ==,size_16,color_FFFFFF,t_70#pic_center)
*   （2）油猴脚本如下：`test()`检查字符串是否与给出的正则表达式模式相匹配，如果是则返回 true，否则就返回 false，该脚本匹配_bfa 的相关字符串
    
    ```
    // ==UserScript==
    // @name         定位携程cookie
    // @namespace    http://tampermonkey.net/
    // @version      0.1
    // @description  try to take over the world!
    // @author       Shirmay1
    // @match        https://hotels.ctrip.com/*
    // @run-at       document-start
    // @grant        none
    // ==/UserScript==
    
    (function() {
        'use strict';
        var _cookie = document.cookie;; // hook cookie
        Object.defineProperty(document, 'cookie', {
            set: function(val) {
                console.log('cookie set->', val);
                var pi = new RegExp('^_bfa.*');
                if (pi.test(val)) {debugger;}
                _cookie = val;
                return val;
            },
            get: function() {
            	console.log('cookie get->', _cookie);
                return _cookie;
            }
       });
    })();
    ```
    

#### 五、hook 的脚本合集

##### 1、hook 之 cookie

*   ① 简单的 console 界面直接 hook，可如下
    
    ```
    Object.defineProperty(document, 'cookie', {
            set: function(val) {
                debugger ;
            }
        })
    ```
    
*   ② 指定某个 cookie 简单的 hook，如 hook 这个 cookie RM4hZBv0dDon443M
    
    ```
    Object.defineProperty(document, 'cookie', {
        set: function(val) {
            if (val.indexOf('RM4hZBv0dDon443M') != -1){
                debugger ;
            }
        }
    })
    ```
    
*   ③ 油猴脚本可如下设置： `hook所有cookie，注意修改@match 所匹配的网址，否则可能hook不到`
    
    ```
    // ==UserScript==
    // @name         定位cookie
    // @namespace    http://tampermonkey.net/
    // @version      0.1
    // @description  try to take over the world!
    // @author       Shirmay1
    // @match        http://entp.yjj.gxzf.gov.cn/appnet/appEntpList.action?entpType=004
    // @run-at       document-start
    // @grant        none
    // ==/UserScript==
    
    (function() {
       'use strict';
        var _cookie = ""; // hook cookie
        Object.defineProperty(document, 'cookie', {
            set: function(val) {
                console.log('cookie set->', new Date().getTime(), val);
                debugger;
                _cookie = val;
                return val;
            },
            get: function() {
                return _cookie;
            }
       });
    })()
    ```
    

##### 2、hook 之 window 属性

*   hook 对象属性通用 demo 逻辑  
    ![](https://img-blog.csdnimg.cn/b8cfd6b2319e4561bf62587c5043b3da.png)
    
*   ① 简单的 console 界面直接 hook，window._$ss
    
    ```
    Object.defineProperty(window, '_$ss', {
        set: function(val) {
            debugger ;
        }
    })
    ```
    
*   ② 油猴脚本 hook，window._pt_
    
    ```
    // ==UserScript==
    // @name         定位pre加密参数
    // @namespace    http://tampermonkey.net/
    // @version      0.1
    // @description  try to take over the world!
    // @author       Shirmay1
    // @run-at       document-start
    // @match        https://flight.qunar.com/*
    // @grant        none
    // ==/UserScript==
    
    (function() {
        'use strict';
        var pre = "";
        Object.defineProperty(window, '_pt_', {
            get: function() {
                console.log('Getting window.属性');
                return pre
            },
            set: function(val) {
                console.log('Setting window.属性', val);
                debugger ;
                pre = val;
            }
        })
    })();
    ```
    

##### 3、hook 之 eval

*   当检测函数的 toString 方法和原型链上的 toString 方法
    
    ```
    var _eval = eval
    eval = function(arg) {
        debugger;
        return _eval(arg)
    }
    eval.toString = function() {
        return "function eval() { [native code] }"
    }
    eval.length = 1;
    var _old = Function.prototype.toString.call
    Function.prototype.toString.call = function(arg) {
        if (arg == eval)
            return "function eval() { [native code] }"
        return _old(arg);
    
    }
    ```
    

##### 4、hook 之 setInterval

*   置空定时器或者过掉定时器的 debugger  
    ![](https://img-blog.csdnimg.cn/fd1fc459acec4399be9a93a5f5a8e608.png)
    
    ```
    var _setInterval=setInterval;
    setInterval=function(a,b){
    	if(a.toString().indexOf("debugger")!=-1{
    		return 
    	}
    	_setInterval(a,b);
    }
    ```
    

##### 5、hook 之 console

*   置空 console 一直打印的情况  
    ![](https://img-blog.csdnimg.cn/c4de2b3862a94031b3deda7c916fe673.png?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5Y2B5LiAKFNocmltYXkxKQ==,size_10,color_FFFFFF,t_70,g_se,x_16)
    
    ```
    console._log=console.log
    console.log=function(a){
        if(a==="世上无难事，只要肯放弃"){
            return
        }
        console._log(a)
    }
    ```
    

##### 5、hook 函数中 debugger

*   修改传参为 debugger 的函数，假设 A 函数有参数 debugger 传入  
    ![](https://img-blog.csdnimg.cn/ed4eab32f04d422a95790462b0dbeaa5.png)
    
    ```
    Function.prototype.constructor = function(param){
        if(param!="debugger"){
            return A(param)  
        }
        return function(){}
    }
    ```
    

##### 6、hook 之 split

*   hook a.split  
    ![](https://img-blog.csdnimg.cn/b7786f78ab15469f939769d5c62d63e5.png)
    
    ```
    String.prototype.split_bk = String.prototype.split;
    String.prototype.split = function(val){
        str =  this.toString();
        debugger;
        return str.split_bk(val)
    }
    a = 'dsdfasdf sdsasd'
    a.split(' ')
    ```
    

##### 7、hook 之覆盖原函数总结

*   通用的函数 hook 覆盖 demo 逻辑  
    ![](https://img-blog.csdnimg.cn/938a49b81b3e465baa4351a2b32bd87c.png)
*   （1）小案例，直接修改原函数
    
    ```
    function _before(){console.log("我是原函数执行中")}
    var temp_before = _before;  // 原函数临时存储
    _before = function(){console.log("我是原函数但被修改了")}
    _before()  // 原函数被修改
    ```
    
*   （2）修改 XMLHttpRequest.prototype.send，hook send  
    ![](https://img-blog.csdnimg.cn/18a97cd4a03b470e91eb7337c4781289.png)
*   （3）hook 原型对象返回字符串修改
    
    ```
    Function.prototype.toString=function(){
    	return "function test(x,y){z=x+y;return z;}";
    }
    ```