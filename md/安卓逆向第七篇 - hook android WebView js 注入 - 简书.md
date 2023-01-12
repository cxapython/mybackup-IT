> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/96a45024e687)

github
======

[https://github.com/mengmugai/webviewdome](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fmengmugai%2Fwebviewdome)

前言
==

如果某 app 使用了 webview 来加载网页。而一些登录状态或者加密参数等各类问题导致不能直接在 pc 的浏览器打开或抓取数据  
方案 1：[关于只能微信客户端打开链接的爬取调试](https://www.jianshu.com/p/59cc3a3b5399)  
上述方案当然可以解决一部分问题。  
话说技多不压身  
来看方案 2

hook webview 注入 js 获取数据
=======================

#### hook 时机

WebView.class 里面有很多方法，主要看以下几个

*   OnPageStarted 这个是页面开始加载时，不太适合。
*   OnPageFinished 中注入，这个是页面加载完毕时，这个可以。
*   onProgressChanged 中注入，在进度变化时注入，这个也不错。  
    为什么是进度大于 25% 才进行注入，因为从测试看来只有进度大于这个数字页面才真正得到框架刷新加载，保证 100% 注入成功

```
public void onProgressChanged(WebView view, int newProgress) {
     
        if (newProgress > 20) {
          注入js！
        }
        super.onProgressChanged(view, newProgress);
    }
```

上述内容是前人表述的 我这里依旧使用 OnPageFinished

### 注入 js

先贴几个文章，看不看都可。  
[https://github.com/AlienwareHe/awesome-reverse/blob/main/android/webview-js-hook.md](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FAlienwareHe%2Fawesome-reverse%2Fblob%2Fmain%2Fandroid%2Fwebview-js-hook.md)  
[https://github.com/AlienwareHe/awesome-reverse/blob/main/android/crack-webview.md](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FAlienwareHe%2Fawesome-reverse%2Fblob%2Fmain%2Fandroid%2Fcrack-webview.md)  
大概就是 java 层可以跟 web 做交互。

##### 主要是一下几点

1、loadUrl 这个可以加载 url 也可以加载 js 脚本，执行的结果不能返回  
2、evaluateJavascript 执行 js 脚本，安卓 4.4 之后才能用，可以直接返回结果  
3、addJavascriptInterface 可以让 js 调用 java 的方法，变相能解决 1 返回结果的问题

实战
==

#### 测试案例

[https://github.com/xuexiangjys/XUI](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fxuexiangjys%2FXUI) 里的 1.18 版本  
本人开头 git 里面也有 xuidemo.apk

#### java hook

![](http://upload-images.jianshu.io/upload_images/14134264-eeff483be948f8b7.png) image.png

```
XposedBridge.hookAllMethods(
        XposedHelpers.findClass("com.xuexiang.xuidemo.base.webview.XPageWebViewFragment$4",xcalssloader),
        "onPageFinished",
        new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                super.afterHookedMethod(param);
                Log.e(TAG, "call on onPageFinished  param【0】:  "+ param.args[0].getClass().getName());
                Log.e(TAG, "call on url "+ param.args[1]);
                try {
                    String jscode = "javascript: 你的代码" ;
                    XposedHelpers.callMethod(param.args[0],
                            "loadUrl",
                            jscode);
                    或者
                                // 第三个参数也是你的代码
                        XposedHelpers.callMethod(webView, "evaluateJavascript", "(function(){ return window.document.getElementsByTagName('html')[0].innerHTML})();", new ValueCallback<String>() {
                        @Override
                        public void onReceiveValue(java.lang.String value) {

                     
                            value = value.replace("\\u003C", "<");
                            value = value.replace("\\"", "");

                            int maxLogSize = 1000;
                            for (int i = 0; i <= value.length() / maxLogSize; i++) {
                                int start = i * maxLogSize;
                                int end = (i + 1) * maxLogSize;
                                end = end > value.length() ? value.length() : end;

                                Log.e(TAG, "[onReceiveValue] html:" + value.substring(start, end));
                            }
                        }
                    });
                } catch (Throwable e) {
                    Log.e(TAG,"调用loadUrl error "+e.getMessage());
                }

            }
        }
);
```

上面代码第二个方案可以直接看到加载的 html

如果使用了 loadUrl  
需要通过 addJavascriptInterface 加入一个对象

```
#1
XposedHelpers.callMethod(thiz, "addJavascriptInterface", XposedJavaScriptLocalObj, "Xposed");

#2
    static final class XposedJavaScriptLocalObj {
        @JavascriptInterface
        public void getSource(String html) {
            Log.i(TAG, "WebViewHelper替换html成功  ");
            Log.i(TAG, html);
            //替换最新更新的html内容
            thizHtml = html;
        }

        @JavascriptInterface
        public void getAjax(String json) {
            Log.i(TAG, "返回json成功");
            Log.i(TAG, json);
        }

    }
#3 
就可以注入下面这个js传回来
javascript:window.Xposed.getSource(document.getElementsByTagName('html')[0].innerHTML);
```

为了理的更清晰 画了个图

![](http://upload-images.jianshu.io/upload_images/14134264-654a84c6073ca27f.png) image.png

这样就能拿到 webview 里的 html 了。  
那如果是 JSON 的怎么办呢？

注入 hook json
============

```
var realXhr = "RealXMLHttpRequest";

function hookAjax(proxy) {
    window[realXhr] = window[realXhr] || XMLHttpRequest;
    XMLHttpRequest = function () {
        var xhr = new window[realXhr];
        for (var attr in xhr) {
            var type = "";
            try {
                type = typeof xhr[attr]
            } catch (e) {
            }
            if (type === "function") {
                this[attr] = hookFunction(attr);
            } else {
                Object.defineProperty(this, attr, {
                    get: getterFactory(attr),
                    set: setterFactory(attr),
                    enumerable: true
                })
            }
        }
        this.xhr = xhr;
    };
    function getterFactory(attr) {
        return function () {
            var v = this.hasOwnProperty(attr + "_") ? this[attr + "_"] : this.xhr[attr];
            var attrGetterHook = (proxy[attr] || {})["getter"];
            return attrGetterHook && attrGetterHook(v, this) || v;
         };
    };
    function setterFactory(attr) {
        return function (v) {
            var xhr = this.xhr;
            var that = this;
            var hook = proxy[attr];
            if (typeof hook === "function") {
                xhr[attr] = function () {
                    proxy[attr](that) || v.apply(xhr, arguments);
                };
            } else {
                var attrSetterHook = (hook || {})["setter"];
                v = attrSetterHook && attrSetterHook(v, that) || v;
                try {
                    xhr[attr] = v;
                } catch (e) {
                    this[attr + "_"] = v;
                }
            }
        };
    }
    function hookFunction(fun) {
        return function () {
            var args = [].slice.call(arguments);
            if (proxy[fun] && proxy[fun].call(this, args, this.xhr)) {
                return;
            }
            return this.xhr[fun].apply(this.xhr, args);
        }
    }
    return window[realXhr];
};


function tryParseJson2(v,xhr){
    var contentType=xhr.getResponseHeader("content-type")||"";
    if(xhr.responseURL.startsWith("https://movie.douban.com/j/search_subjects")){
       window.XposedAppiumObj.getAjax(v);
    }
    return v;
}


hookAjax({
    responseText: {
        getter: tryParseJson2
    },
    response: {
        getter: tryParseJson2
    }
});
```

也是通过 loadurl 把上述 js 注入进去 最后会走 tryParseJson2 ，他最后走了刚才绑定的类里的第二个方法 getAjax

![](http://upload-images.jianshu.io/upload_images/14134264-92c917d682c793a3.png) image.png

这里选择的是豆瓣的热门电影页，原 app 是打开百度 我通过 xposed 给换了 点击继续加载日志中就会增加 详细代码可看 github

### 实现点击

可以看[珍惜的 xposedappium 里的自动点击里的这个文件](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fw296488320%2FXposedAppium%2Fblob%2Fbdacf77aca0a7e5b65dc2a723bd8e290d798108c%2FXposedAppiumLib%2Fsrc%2Fmain%2Fjava%2Fcom%2Fzhenxi%2FSuperappium%2FWebViewHelper.java%23L109)  
代码为打开百度输入 1，点击百度一下。  
WebViewHelper 为引入👆🏻这个链接的文件

```
String jsCode = "document.getElementsByTagName('html')[0].innerHTML";
      

    WebViewHelperzx wvhzx = new WebViewHelper((WebView) param.args[0]);

    wvhzx.executeJsCode("document.getElementById(\"index-kw\").value=1");
    wvhzx.clickByXpath("//button[@id=\"index-bn\"]");
    wvhzx.executeJsCode("jsCode",new ValueCallback<String>() {
        @Override
        public void onReceiveValue(String s) {
            thizHtml = s;

        }
    });
```

结束语
===

xuidome.app 这个浏览器页在 ” 扩展 “-》”web 浏览器 “-》直接显示调用  
首次打开会有注册页， 那个没啥用 是个伪界面 手机号和验证码瞎填都行

————————————————————————————————————————-  
2022,4,12 更新  
[https://github.com/WankkoRee/EnableWebViewDebugging](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2FWankkoRee%2FEnableWebViewDebugging)  
这个有各个主流 app 的 webview