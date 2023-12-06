> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [cmrodriguez.me](https://cmrodriguez.me/blog/hpandro-4/)

> frida ctf challenge websocket

In this series of posts I’ll be solving some persistence challenges from [hpandro ctf challenges](http://ctf.hpandro.raviramesh.info/). hpAndro created an Android application with multiple vulnerabilities, following the [MSTG](https://github.com/OWASP/owasp-mstg).

**NOTE: the challenge is solved with an older version of the application. You can find the copy of the apk here: [https://github.com/CesarMRodriguez/lunesdemobile/blob/main/Sesion%206/hpandro.apk](https://github.com/CesarMRodriguez/lunesdemobile/blob/main/Sesion%206/hpandro.apk)**

We have two different challenges to solve:

*   Intercepting WebSockets.
*   Intercepting Secure WebSockets.

Both exercises have the same behavior, so I’ll focus on the first one. The challenge here is to intercept the flag that is being sent over websockets. I start by searching the AndroidManifest.xml to find the Activity that could be called in order to solve the task:

```
<activity android:/>


```

by checking the com.hpandro.androidsecurity.ui.activity.task.websocket.WebSocketSecureActivity class. The first method to see is **onCreate** where I found a code similar to the one used in the HTTP and HTTPS challenges. Check [hpandro-1](https://cmrodriguez.me/blog/hpandro-1/) for more details on them. It basically configures a Webview and calls the following URL: [http://hpandro.raviramesh.info/ws_task.php](http://hpandro.raviramesh.info/ws_task.php)

```
//configure WebView and WebSettings
a.D(a.J((WebView) y(R.id.webviewTask), "webviewTask!!.settings", true, true, false, false, true), WebSettings.LayoutAlgorithm.SINGLE_COLUMN, 2, true, "hpAndro");
...
WebView webView2 = (WebView) y(R.id.webviewTask);
g.c(webView2);
//set WebviewClient
webView2.setWebViewClient(new b(this));
...
Object systemService = getSystemService("connectivity");
if (systemService != null) {
    ...
    if (z) {
        WebView webView4 = (WebView) y(R.id.webviewTask);
        g.c(webView4);
        //loads the url of the challenge
        webView4.loadUrl("http://hpandro.raviramesh.info/ws_task.php");
        ...
    } else {


```

Whenever I went to the URL in a browser, I found the following content in javascript:

```
eval([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]][([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]][([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]((!![]+[])[+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+([][[]]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+!+[]]+([]+[])[(![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(!![]+[])[+[]]+([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]()[+!+[]+[!+[]+!+[]]]+(([][[]]+[])[!+[]+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]]+[!+[]+!+[]+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+([][[]]+[])[+!+[]]+([][[]]+[])[!+[]+!+[]]+(!![]+[])[+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]]+[+[]]+(!![]+[])[+[]]+[!+[]+!+[]+!+[]+!+[]]+[!+[]+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]]+[+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+([][[]]+[])[+[]]+(!![]+[])[+!+[]]+(!![]+[])[+[]]+[!+[]+!+[]+!+[]+!+[]]+[+[]]+(!![]+[])[+[]]+[+!+[]]+[+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+(!![]+[])[+[]]+[!+[]+!+[]+!+[]+!+[]]+[+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+[!+[]+!+[]+!+[]+!+[]]+[+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]]+[+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+[+[]]+(![]+[])[+!+[]]+([][[]]+[])[+!+[]]+([][[]]+[])[!+[]+!+[]]+(!![]+[])[+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(+(+!+[]+[+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+[!+[]+!+[]]+[+[]])+[])[+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+[!+[]+!+[]+!+[]+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]]+[!+[]+!+[]+!+[]+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]]+[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]]+([][[]]+[])[+!+[]]+(!![]+[])[+[]]+[+!+[]]+[+[]]+[+!+[]]+(!![]+[])[+[]]+[+!+[]]+[+!+[]]+[+!+[]]+([][[]]+[])[!+[]+!+[]]+[+[]]+(![]+[])[!+[]+!+[]+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]]+[!+[]+!+[]+!+[]]+[!+[]+!+[]+!+[]+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]]+[!+[]+!+[]]+([][[]]+[])[+!+[]]+(!![]+[])[+[]]+[+!+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]]+[+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+[!+[]+!+[]]+[+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]]+[+!+[]]+([][[]]+[])[+!+[]]+(!![]+[])[+[]]+[+!+[]]+[+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+(![]+[])[!+[]+!+[]]+(!![]+[])[+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+(![]+[])[+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+[!+[]+!+[]]+(!![]+[])[+[]]+[+!+[]]+[+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+[+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+(!![]+[])[+[]]+[+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]+!+[]+!+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]]+(!![]+[])[+[]]+[!+[]+!+[]+!+[]+!+[]]+[!+[]+!+[]]+(!![]+[])[+[]]+[!+[]+!+[]+!+[]+!+[]+!+[]]+[+!+[]])[(![]+[])[!+[]+!+[]+!+[]]+(+(!+[]+!+[]+[+!+[]]+[+!+[]]))[(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([]+[])[([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]][([][[]]+[])[+!+[]]+(![]+[])[+!+[]]+((+[])[([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]+[])[+!+[]+[+!+[]]]+(!![]+[])[!+[]+!+[]+!+[]]]](!+[]+!+[]+!+[]+[+!+[]])[+!+[]]+(![]+[])[!+[]+!+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(!![]+[])[+[]]]((!![]+[])[+[]])[([][(!![]+[])[!+[]+!+[]+!+[]]+([][[]]+[])[+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([![]]+[][[]])[+!+[]+[+[]]]+(!![]+[])[!+[]+!+[]+!+[]]+(![]+[])[!+[]+!+[]+!+[]]]()+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([![]]+[][[]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]](([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]][([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]((!![]+[])[+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+([][[]]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+!+[]]+(![]+[+[]])[([![]]+[][[]])[+!+[]+[+[]]]+(!![]+[])[+[]]+(![]+[])[+!+[]]+(![]+[])[!+[]+!+[]]+([![]]+[][[]])[+!+[]+[+[]]]+([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(![]+[])[!+[]+!+[]+!+[]]]()[+!+[]+[+[]]]+![]+(![]+[+[]])[([![]]+[][[]])[+!+[]+[+[]]]+(!![]+[])[+[]]+(![]+[])[+!+[]]+(![]+[])[!+[]+!+[]]+([![]]+[][[]])[+!+[]+[+[]]]+([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(![]+[])[!+[]+!+[]+!+[]]]()[+!+[]+[+[]]])()[([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]((![]+[+[]])[([![]]+[][[]])[+!+[]+[+[]]]+(!![]+[])[+[]]+(![]+[])[+!+[]]+(![]+[])[!+[]+!+[]]+([![]]+[][[]])[+!+[]+[+[]]]+([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(![]+[])[!+[]+!+[]+!+[]]]()[+!+[]+[+[]]])+[])[+!+[]])+([]+[])[(![]+[])[+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(!![]+[])[+[]]+([][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+(![]+[])[!+[]+!+[]]+(!![]+[][(![]+[])[+[]]+(![]+[])[!+[]+!+[]]+(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]()[+!+[]+[!+[]+!+[]]])())());


```

After looking for some information about what could be that. I found that the code is done with a liberary called JSFuck (inspired on the [brainfuck](https://es.wikipedia.org/wiki/Brainfuck) language). In order to see what was that code, I had to find a JSFuck decoder: [https://enkhee-osiris.github.io/Decoder-JSFuck/](https://enkhee-osiris.github.io/Decoder-JSFuck/)

After using that page I could decode the content, which was the following one:

```
doSend("Your Flag is hpandro{ws.tTSrUnAId0s9S4ZnM8Yz1QnFlrwfzGpW}")


```

So that is the flag needed. But I wanted to solve the issue with Frida, so understanding the content of what the challenge did I had an idea of what to do. This challenge won’t be as easy as the ones related to http and https. In those examples we solved the challenges by overriding the shouldInterceptRequest function of the WebViewClient. But websocket messages aren’t intercepted by that function, and they go directly through the browser engine.

I could still change GET messages in order to add some payload, so the strategy to follow is:

a- Intercept WebViewClient.shouldOverrideUrl. b- add a javascript payload that changes the execution in order to capture the message. c- Send the message to a JavascriptInterface. + create or use a class wih JavascriptInterface + configure the webview o include the class with javascriptInterface d- Hook the method from the JavascriptInterface with Frida.

The following image shows the solution proposed:

![](https://cmrodriguez.me/images/featured-post/hpandro-4-websockets.png)

### Step a - Intercept WebViewClient

In order to achieve this goal, we can use the same strategy that we did to solve Http and Https traffic interception. In this case the class to inject is “v0.d.a.c.a.d.j.b” due to the following line:

```
webView2.setWebViewClient(new b(this));


```

The only thing to have in mind is that the script must add a “script” tag that adds the payload that we will write. The fsunction that does this is the following one:

```
function readNewStream(inputStream) {

    var BufferedReader = Java.use('java.io.BufferedReader'); 
    var InputStreamReader = Java.use('java.io.InputStreamReader');
    var inputStreamReader = InputStreamReader.$new(inputStream);
    
    var r = BufferedReader.$new(inputStreamReader);
    
    var StringBuilder = Java.use('java.lang.StringBuilder');
    var total = StringBuilder.$new();
    var String = Java.use('java.lang.String');
    while (true) {
        var line = r.readLine();
        if (line == null) {
            break;
        } else {
            total.append(line).append(String.$new('\n'));
        }
    }
    return "<script>...</script>" + total.toString();
}


```

### Step b - create javascript payload

In order to create the javascript payload I wanted to play around with the page in the **chrome://inspect** tab ofth WebView so I could check if the redirection was going to work or not.

The first problem I faced related to the old version of the application is that the developers of the app changed the endpoints of the application to work just with https, and this version of the app pointed to the http endpoint. Depending on the version of the Android OS, the application would show a button to reload the application with https. This would ruin the payload generated in step a. SO in order to avoid this situation I created the following script that could be loaded in the beginning of the application or whenever you launch the websocket task.

```
var WebView = Java.use("android.webkit.WebView");
WebView.loadUrl.overload("java.lang.String").implementation = function (theUrl) {
    if (theUrl.indexOf("ws_task.php") > 0) {
        this.loadUrl(String.$new("https://hpandro.raviramesh.info/ws_task.php") );
    } else {
        this.loadUrl(theUrl);
    }
}


```

Whenever I opened the application with the inpect, I found the following error:

![](https://cmrodriguez.me/images/featured-post/hpandro-4-websocket-error.png)

This error occurs because the application is loaded over https, and the websocket message is sent over ws (plaintext). In order to avoid this bug in Android you should configure the WenSettings to set the MixedContentMode available. This is done in Java in the following way (**do not use this in an actual application**):

```
webView.getSettings().setMixedContentMode(WebSettings.MIXED_CONTENT_ALWAYS_ALLOW);


```

In order to do this with Frida, I needed to find a place where to set this script. I needed a function that received the WebView as a parameter or a function in an object that had the WebView as an attribute. I ended up overwriting the following method _v0.a.a.a.a.J_, which is being called in the onCreate method in the WebSocketActivity:

```
a.D(a.J((WebView) y(R.id.webviewTask), "webviewTask!!.settings", true, true, false, false, true), WebSettings.LayoutAlgorithm.SINGLE_COLUMN, 2, true, "hpAndro");


```

and has the WebView as a parameter:

```
public static WebSettings J(WebView webView, String str, boolean z, boolean z2, boolean z3, boolean z4, boolean z5) {
    g.c(webView);
    WebSettings settings = webView.getSettings();
    g.d(settings, str);
    settings.setJavaScriptEnabled(z);
    settings.setLoadWithOverviewMode(z2);
    settings.setUseWideViewPort(z3);
    settings.setSupportZoom(z4);
    settings.setBuiltInZoomControls(z5);
    return settings;
}


```

The script created is the following one:

```
var a = Java.use('v0.a.a.a.a');
a.J.implementation = function (webView, str, z, z2, z3, z4, z5) {   
    console.log("entra aca");
    webView.getSettings().setMixedContentMode(0);
    return this.J(webView, str, z, z2, z3, z4, z5);
}


```

There are multiple ways to generate the payload. The final form requires the definition of the step “c”. I decided to change the send method from the WebSocket class. this is due to the fact that in order to send messages the content must go over this method.

```
//first we set in the prototype of the WebSocket the original send method.
WebSocket.prototype.send2 = WebSocket.prototype.send; 

//then we change the send method to call the JavascriptInterface, and then we call the original send method (after this it is send2)
WebSocket.prototype.send = function (msg) { JInterface.showMessage(msg); this.send2(msg); } 


```

The second step of calling the original send method is created in order to not crash the application, so this method can be used in every application.

### Step c - create JavascriptInterface class

In order to send the content of the message to Java we need to user or implement a [JavascriptInterface](https://developer.android.com/guide/webapps/webview#UsingJavaScript) class.

The first thing I tried to do was creating a class dynamically with Frida (as we did when we tried to solve the http challenge). As an example this is an abstract of the previous script:

```
var MyWebViewClient = Java.registerClass({
      name: 'com.example.MyWebViewClient',
      superClass: WebViewClient,
      methods: {
        $init() {
          console.log('Constructor called');
        },
        shouldInterceptRequest: [{
          returnType: 'android.webkit.WebResourceResponse',
          argumentTypes: ['android.webkit.WebView', 'android.webkit.WebResourceRequest'],
          implementation(webView, request) {

                ...
            }
        }],
      }
    });


```

It won’t work in this scenario because there is no way (or not an easy available way) to add annotations to a class or method in Frida. I also tried looking for a way to do that with reflection in Java, but it wasn’t possible either.

So I had to go for an alternative. I tried to use a class that already has a @JavascriptInterface in the application.

In this application I had the following methods:

-> v0.f.i4

```
@JavascriptInterface
public void postMessage(String str) {
    try {
        h2.k kVar = h2.k.DEBUG;
        h2.a(kVar, "OSJavaScriptInterface:postMessage: " + str, null);
        JSONObject jSONObject = new JSONObject(str);
        String string = jSONObject.getString("type");
        if (string.equals("rendering_complete")) {
            b(jSONObject);
        } else if (string.equals("action_taken") && !i4.this.b.i) {
            a(jSONObject);
        }
    } catch (JSONException e) {
        e.printStackTrace();
    }
}          


```

-> v0.c.b.b.g.a.fq

```
@JavascriptInterface
public final String getClickSignals(String str) {
    String str2;
    if (TextUtils.isEmpty(str)) {
        str2 = "Click string is empty, not proceeding.";
    } else {
        gv1 d = this.b.d();
        if (d == null) {
            str2 = "Signal utils is empty, ignoring.";
        } else {
            rl1 rl1 = d.b;
            if (rl1 == null) {
                str2 = "Signals object is empty, ignoring.";
            } else if (this.b.getContext() != null) {
                return rl1.g(this.b.getContext(), str, this.b.getView(), this.b.a());
            } else {
                str2 = "Context is null, ignoring.";
            }
        }
    }
    a.e(str2);
    return "";
}

@JavascriptInterface
public final void notify(String str) {
    if (TextUtils.isEmpty(str)) {
        k.R2("URL is empty, ignoring message");
    } else {
        c1.i.post(new hq(this, str));
    }
}


```

So here the most convenient class was “v0.c.b.b.g.a.fq”, because it has a the notify Method which receives an String.

The final script with this concept is the following one {{websocket.js}}, but when I run it I get the following error:

```
[Twitch::com.hpandro.androidsecurity]-> "<instance: v0.c.b.b.g.a.fq>"
Process crashed: Bad access due to invalid address

***
*** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
Build fingerprint: 'Android/vbox86p/vbox86p:9/PI/138:userdebug/test-keys'
Revision: '0'
ABI: 'x86'
pid: 17009, tid: 17135, name: JavaBridge  >>> com.hpandro.androidsecurity <<<
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x10
Cause: null pointer dereference
    eax 00000000  ebx e5e7a85c  ecx e2b80b00  edx 00000001
    edi e9af5d9c  esi e2b80b00
    ebp bcd3c608  esp bcd3c520  eip e5d7f707

backtrace:
    #00 pc 005b1707  /system/lib/libart.so (offset 0x28a000) (artQuickGenericJniTrampoline+103)
    #01 pc 005f6b8f  /system/lib/libart.so (offset 0x5ea000) (art_quick_generic_jni_trampoline+63)
    #02 pc 005f0b82  /system/lib/libart.so (offset 0x28a000) (art_quick_invoke_stub+338)
    #03 pc 000a30ce  /system/lib/libart.so (offset 0x97000) (art::ArtMethod::Invoke(art::Thread*, unsigned int*, unsigned int, art::JValue*, char const*)+222)
    #04 pc 004d3349  /system/lib/libart.so (offset 0x28a000) (art::(anonymous namespace)::InvokeWithArgArray(art::ScopedObjectAccessAlreadyRunnable const&, art::ArtMethod*, art::(anonymous namespace)::ArgArray*, art::JValue*, char const*)+89)
    #05 pc 004d45f7  /system/lib/libart.so (offset 0x28a000) (art::InvokeVirtualOrInterfaceWithJValues(art::ScopedObjectAccessAlreadyRunnable const&, _jobject*, _jmethodID*, jvalue*)+471)
    #06 pc 00381e1f  /system/lib/libart.so (offset 0x28a000) (art::JNI::CallVoidMethodA(_JNIEnv*, _jobject*, _jmethodID*, jvalue*)+959)
    #07 pc 02ecb568  /data/app/com.android.chrome-rZxYnE7qcLsVDKktRW3EUw==/base.apk (offset 0xbdc000)
***
[Twitch::com.hpandro.androidsecurity]->


```

I then tested the same scenario in another custom application and it didn’t work either. I suppose the bug is related to the fact that the browser engine runs in another process, so there can be an error whenever they try to call a changed method in the android application injected with Frida.

The last option here is to upload a class that would set the JavascriptInterface so the message is sent to Java. This can be done with the following Frida API:

Java.openClassFile("…");

which receives the path of a jar, class or dex file. So I needed to create a valid dex file with the class that has the JavaInterface.

I created a project on Android Studio, and coded the following class:

```
package com.hpandro.example;

import android.webkit.JavascriptInterface;
import android.webkit.WebView;

public class JInterfaceInternal {
    /** Show a toast from the web page */
    @JavascriptInterface
    public void showMessage(String toast) {
        innerFunction(toast);
    }

    public void innerFunction(String content) {
        System.out.println("Injected class!! " + content);
    }

    public static void setJavaInterface(WebView webView) {
        webView.addJavascriptInterface(new JInterfaceInternal(),"JInterface");
    }
}


```

Then I built it, generating an apk. I unzipped it to get the classes.dex file. After this I had to convert it to jar with the dex2jar tool, in order to remove potential classes that would crash the process of hpAndro. I decided to do it with the jar and not directly with the dex file, because it is easier to manipulate as it is a zip file with .class files per each class. The next step was to upload it to any folder in the application, and then I run:

```
Java.perform( function () {

    Java.openClassFile("/data/data/com.hpandro.androidsecurity/files/injectablefinal.dex").load();
});


```

Note: the openClassFile does not work well with early instrumentation. Note2: in order to solve this in one shot, the dex file should be downloaded with Frida to a folder, and then injected automatically with a setTimeout or by overriding some method. I did not do it this way in order to remove complexity of the final solution and focus on the important parts.

Step d - Hook the method from the JavascriptInterface with Frida.
-----------------------------------------------------------------

Once the class is injected, we have to run:

```
//this is the method hooked from Frida
Java.perform( function() {
   var JInternal = Java.use("com.hpandro.example.JInterfaceInternal"); 
   JInternal.showMessage.implementation = function (message) {
        console.log(message);
   }
});

//script that changes the behavior of the WebView
Java.perform( function () {

    var newClass = Java.use("com.hpandro.example.JInterfaceInternal");
    
    var a = Java.use('v0.a.a.a.a');
    a.J.implementation = function (webView, str, z, z2, z3, z4, z5) {   
        newClass.setJavaInterface(webView);
        webView.getSettings().setMixedContentMode(0);
        return this.J(webView, str, z, z2, z3, z4, z5);
    }
        
    var WebViewClient = Java.use('v0.d.a.c.a.d.j.b');
    WebViewClient.shouldInterceptRequest.overload('android.webkit.WebView', 'android.webkit.WebResourceRequest').implementation = function (webView, request) {

        var WebResourceResponse = Java.use('android.webkit.WebResourceResponse');
        var isFavicon = request.getUrl().toString().search("favicon.ico") > 0 || request.getUrl().toString().search("mytheme.min.css") > 0;
        
        var URL = Java.use('java.net.URL');
        var url = URL.$new(request.getUrl().toString());
        
        var HttpURLConnection = Java.use('java.net.HttpURLConnection');
        var urlConnection = Java.cast(url.openConnection(),HttpURLConnection);
        
        var String = Java.use('java.lang.String');
        urlConnection.setRequestProperty(String.$new("User-Agent"), String.$new("hpAndro"));
        
        var BufferedInputStream = Java.use('java.io.BufferedInputStream');
        var inputStr = BufferedInputStream.$new(urlConnection.getInputStream());
        var webResourceRes = null;
        if (!isFavicon) {
            var strToPrint = readNewStream(inputStr);
            //convert to inputStr
            var resultado = String.$new(strToPrint);
            var ByteArrayInputStream = Java.use("java.io.ByteArrayInputStream");
            var byteArrayInputStream = ByteArrayInputStream.$new(resultado.getBytes());
            var InputStream = Java.use("java.io.InputStream");
            var newInputStream = Java.cast(byteArrayInputStream,InputStream);            
            webResourceRes = WebResourceResponse.$new("text/html","utf-8", newInputStream);
        } else {
            webResourceRes = WebResourceResponse.$new("image/vnd.microsoft.icon","gzip", inputStr);
        }
        return webResourceRes;
    }
    
});


```

This ended up showing the message in the console, but sometimes it crashed (I still do not know the reason if this behavior), so it wasn’t reliable. I wanted to find a more stable way to do it.

One of the ways that I found to do this was sending the content in an out of band way. There were lots of alternatives, but I decided to use sockets. So in this case in order to make this work, I needed two components. The first one was a Java class that sends the flag over a socket. This was done in the following way:

I added a method in the JInterfaceInternal class:

```
public class JInterfaceInternal {

    @JavascriptInterface
    public void sendMessage(String toast) {
        MessageSender messageSender = new MessageSender(toast);
    }


```

and then created the MessageSender class that opens a socket and sends a message over there:

```
public class MessageSender {


    protected static String SERVER_IP = "127.0.0.1";
    protected static int SERVER_PORT = 23389;
    private String messageToSend = null;
    private PrintWriter output;
    private BufferedReader input;
    public MessageSender(String messageToSend) {

        this.messageToSend = messageToSend;
        Thread thread1 = new Thread(new Thread1());
        thread1.start();
    }

    class Thread1 implements Runnable {
        public void run() {
            Socket socket;
            try {
                socket = new Socket(MessageSender.SERVER_IP, MessageSender.SERVER_PORT);
                output = new PrintWriter(socket.getOutputStream());
                input = new BufferedReader(new InputStreamReader(socket.getInputStream()));
                new Thread(new Thread3(messageToSend)).start();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    class Thread3 implements Runnable {
        private String message;
        Thread3(String message) {
            this.message = message;
        }
        @Override
        public void run() {
            output.write(message);
            output.flush();
        }
    }
}


```

I also needed to create the message receiver. I decided to create it on Frida directly, and this is the result:

```
Java.perform( function() {

    var Runnable = Java.use("java.lang.Runnable"); 
    
    var Thread1 = Java.registerClass({
      name: 'com.example.Thread1',
      implements: [Runnable],
      methods: {
        run() {
            var ServerSocket = Java.use("java.net.ServerSocket");
            var Socket = Java.use("java.net.Socket");
            var BufferedReader = Java.use("java.io.BufferedReader");
            var InputStreamReader = Java.use("java.io.InputStreamReader");
            var serverSocket = ServerSocket.$new(23389);
            while (true) {
                var socket = serverSocket.accept();
                input = BufferedReader.$new(InputStreamReader.$new(socket.getInputStream()));
                var message = input.readLine();
                if (message != null) {
                    console.log(message);
                }
            }
        }
      }
    });


    var Thread = Java.use("java.lang.Thread");

    var MessageReceiver2 = Java.registerClass({
      name: 'com.example.MessageReceiver2',
      methods: {
        $init() {
          var thread1 = Thread.$new(Thread1.$new());
          thread1.start();
        }
      }
    });

    MessageReceiver2.$new();
});


```

And after running the application this way, I could get the flag in a more reliable way.

In the following link [https://github.com/CesarMRodriguez/lunesdemobile/tree/main/Sesion%206](https://github.com/CesarMRodriguez/lunesdemobile/tree/main/Sesion%206) you can find the application we used for this post and the scripts used to solve the challenge.