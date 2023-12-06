> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [cmrodriguez.me](https://cmrodriguez.me/blog/hpandro-1/)

> frida ctf challenge http intercept

In this series of posts I’ll be solving the [hpandro ctf challenges](http://ctf.hpandro.raviramesh.info/). hpAndro created an Android application with multiple vulnerabilities, following the [MSTG](https://github.com/OWASP/owasp-mstg).

This application has a lot of simple challenges, so it is good for beginners or to test some techniques. I’ll try to solve all the challenges using Frida. As a secondary objective I’ll try to use scripts that can work in multiple scenarios and not just to get the flag.

The first challenge to solve is the Http interception. This challenge can be solved with the following strategies:

*   Intercepting the application requests with any web-proxy.
*   Use a lower stack network proxy like wireshark. In this case it is possible because the content is in cleartext.
*   Using static code analysis tools in order to find what the application does, and executing the same request on a browser.
*   Last but not least, using Frida.

As we already explained, we’ll be solving the challenge with Frida. So in order to find what to script with Frida, I’ll check the decompiled application.

So the first thing to find is which component is executing the request that has to be intercepted. This can be done analyzing the Android Manifest, where we can find the following activity:

```
	        <activity android:/>


```

In this case the activity class is descriptive, so we can read that class directly. We can find in the HTTPTrafficActivity class the following code:

```
    if (z) {
        WebView webView4 = (WebView) y(R.id.webviewTask);
        g.c(webView4);
        webView4.loadUrl("http://hpandro.raviramesh.info/http_task.php");
    }


```

It can be seen that the URL is being loaded in a WebView. As it is a GET request, I executed it in a browser. But the server did not return any content on the response.

After reading a bit more what was being done in the code I found the following method:

```
  a.D(a.J((WebView) y(R.id.webviewTask), "webviewTask!!.settings", true, true, false, false, true), WebSettings.LayoutAlgorithm.SINGLE_COLUMN, 2, true, "hpAndro");


```

a.J does not do anything significative for our analysys, but a.D does the following:

```
    public static void D(WebSettings webSettings, WebSettings.LayoutAlgorithm layoutAlgorithm, int i, boolean z, String str) {
        webSettings.setLayoutAlgorithm(layoutAlgorithm);
        webSettings.setCacheMode(i);
        webSettings.setDomStorageEnabled(z);
        webSettings.setUserAgentString(str);
    }


```

which sets up “hpAndro” as the User-Agent. I tested the request changing it:

```
GET /http_task.php HTTP/1.1
Host: hpandro.raviramesh.info
Pragma: no-cache
Cache-Control: no-cache
Upgrade-Insecure-Requests: 1
User-Agent: hpAndro
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
X-Requested-With: com.hpandro.androidsecurity
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Connection: close


```

which returns the desired flag in a header. So in order to solve this challenge we need to see the way to intercept traffic in a WebView.

Whenever I have to create a complex script with Frida I follow 3 steps.

1.  Code a Java application that does whatever I want the Frida script to. I do this in order to discard the complexity of the targeted application and focus on the problem to solve. In this case it was how to intercept the request and the response from a WebView.
    
2.  Translate the Java code in a Frida script. In this case the translation is in the mock application.
    
3.  Adapt the code to the particular application. As the Frida script from step 2 applies to the mock application, I need to find where to hook the script in the targeted application in order to make it work.
    

Step 1
------

It is not that easy to intercept requests from a WebView, as it runs in a different process and executes requests in a browser engine. So I had to find a way to change the behavior of the WebView in order to send the traffic through the Java layer. I found that the Android SDK has a way to configure the way the resources are being loaded by the WebViewClient through the function [shouldInterceptRequest](https://developer.android.com/reference/android/webkit/WebViewClient#shouldInterceptRequest(android.webkit.WebView,%20android.webkit.WebResourceRequest)). This function is used to notify the host application of a resource request and allow the application to return the data. If the return value is null, the WebView will continue to load the resource as usual. Otherwise, the return response and data will be used.

So if we implement this function in the Java layer, the application will be able to intercept the requests sent by the WebView.

**Note: This function works fine for GET parameters, with other verbs with body it does not work.**

So I created an Activity in an application with a WebView, and implemented the shouldInterceptRequest:

a. Configure the WebViewClient for the WebView:

```
        testWebView.setWebViewClient(new WebViewClient() {

            @RequiresApi(api = Build.VERSION_CODES.LOLLIPOP)
            @Override
            public WebResourceResponse shouldInterceptRequest (final WebView view, WebResourceRequest request) {

            	...
                WebResourceResponse webResourceResponse = FirstFragment.shouldInterceptRequest(view, request);
                return webResourceResponse;
                ...


```

b. Implement the function that takes the content of the original request (WebResourceRequest) and executes the request. It is possible, because the WebResourceRequest instance has the URL and headers of the original petition.

```
    @RequiresApi(api = Build.VERSION_CODES.LOLLIPOP)
    private static  WebResourceResponse shouldInterceptRequest (final WebView view, WebResourceRequest request) {

    	//show method
	    System.out.println("Method: " + request.getMethod());
        //show URL and queryString
	    System.out.println("Uri: " + request.getUrl().toString());
        //show all http headers
        Map<String, String> headers = request.getRequestHeaders();
        for (Map.Entry<String,String> entry : headers.entrySet())
            System.out.println("Header: " + entry.getKey() +
                    "= " + entry.getValue());

        // and here, if you want, you can load the page normally
        String htmlContent = "";

        URL url = null;
        try {
        	//execute the request
            url = new URL(request.getUrl().toString());
            HttpURLConnection urlConnection = (HttpURLConnection) url.openConnection();
            urlConnection.setRequestProperty("User-Agent", "hpAndro");
            try {
                InputStream in = new BufferedInputStream(urlConnection.getInputStream());
                //print content of response
                readStream(in);
                Map<String, List<String>> map = urlConnection.getHeaderFields();
                for (Map.Entry<String, List<String>> entry : map.entrySet()) {
                    System.out.println("Key : " + entry.getKey() + " ,Value : " + entry.getValue());
                }
                //return response to the WebView.
                WebResourceResponse webResourceRes;
                webResourceRes = new WebResourceResponse("text/html","utf-8", in);
                return webResourceRes;
                ...


```

As it can be seen in the previous code, we achieve the initial goal of sending the request to the Java layer in order to manipulate it then with Frida.

Step 2
------

We need to translate the previous code to a Frida script. Most of the translation is straightforward if you have done some scripts on your own.

The most complex part of the translation is that the solution needs us to create a new class that inherits from WebViewClient, and that implements the shouldInterceptRequest.

The following Java code creates an anonymous class, that is being used as a parameter in the setWebViewClient:

```
        testWebView.setWebViewClient(new WebViewClient() {


```

In Frida we can’t create anonymous classes, so we have to use the _Java.registerClass_ API:

```
	var WebViewClient = Java.use('android.webkit.WebViewClient');
	
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
			    return webResourceRes;
	    	}
	    }],
	  }
	});


```

The other issue I had creating the scripts is that Frida-client breaks whenever a method returns a _null_ value. In this case I wanted to return null in all non HTML requests (like the favico or any javascript/css request), and avoid creating a WebResourceResponse per each potential kind of resource.

As I couldn’t find a way to make it work I analyzed the resources being requested by the webview and then create different WebResourceResponses based on them. That is the reason you’ll find in a script the following if statement:

```
			    if (!isFavicon) {
			    	...
				    webResourceRes = WebResourceResponse.$new("text/html","utf-8", inputStr);
				} else {
					webResourceRes = WebResourceResponse.$new("image/vnd.microsoft.icon","gzip", inputStr);
				}


```

In order to create a custom script to intercept all HTTP traffic, you’ll need to consider that you’ll need to handle the non HTTP traffic as well, because of the return of null error.

Step 3
------

I needed to find a place to inject the script in the original application. The requirements of the script I wrote is that I need the reference to the WebView in order to set the WebViewClient. Based on the decompiled code:

```
  super.onCreate(bundle);
        setContentView(R.layout.activity_httptraffic);
        ...
        boolean z = true;
        this.i.a(this, new v0.d.a.c.a.d.h.j(this, true));
        a.D(a.J((WebView) y(R.id.webviewTask), "webviewTask!!.settings", true, true, false, false, true), WebSettings.LayoutAlgorithm.SINGLE_COLUMN, 2, true, "hpAndro");
        int i = Build.VERSION.SDK_INT;
        WebView webView = (WebView) y(R.id.webviewTask);
        ...
        if (systemService != null) {
            ...
            if (z) {
                WebView webView4 = (WebView) y(R.id.webviewTask);
                g.c(webView4);
                webView4.loadUrl("http://hpandro.raviramesh.info/http_task.php");
            } else {
                RelativeLayout relativeLayout = (RelativeLayout) y(
                ...
            }
            ((Button) y(R.id.btnCheck)).setOnClickListener(new i(this));
            return;
        }
        throw new NullPointerException("null cannot be cast to non-null type android.net.ConnectivityManager");


```

the function that receives a WebView as a parameter is a.J. So I created the hook on that function:

```
	var WebViewConfigurator = Java.use("v0.a.a.a.a");
	WebViewConfigurator.J.implementation = function (webView, str, z, z2, z3, z4, z5) {
		webView.setWebViewClient(MyWebViewClient.$new());
		return this.J(webView, str, z, z2, z3, z4, z5);
	}


```

The final script can be seen in the following URL: [https://github.com/CesarMRodriguez/lunesdemobile/blob/main/Sesion%201/http_task.js](https://github.com/CesarMRodriguez/lunesdemobile/blob/main/Sesion%201/http_task.js)

Sadly after running the script in the application to see if it worked, I found that the custom class was not being called. After reading and checking the script I found that between a.J and webview.loadUrl, the HttpTrafficActivity class was setting a custom webViewClient, overwriting the custom class I wrote.

```
        a.D(a.J((WebView) y(R.id.webviewTask), "webviewTask!!.settings", true, true, false, false, true), WebSettings.LayoutAlgorithm.SINGLE_COLUMN, 2, true, "hpAndro");
        ...
        webView3.setWebChromeClient(new h());
        g.c(this);
        Object systemService = getSystemService("connectivity");
        if (systemService != null) {
        	...
            if (z) {
                WebView webView4 = (WebView) y(R.id.webviewTask);
                g.c(webView4);
                webView4.loadUrl("http://hpandro.raviramesh.info/http_task.php");


```

So I had to create a new script. In this case the solution was much more simpler than what I wrote initially, because I only had to overwrite the shouldInterceptRequest from that class:

```
Java.perform( function () {

	var WebViewClient = Java.use('v0.d.a.c.a.d.h.g');
	WebViewClient.shouldInterceptRequest.overload('android.webkit.WebView', 'android.webkit.WebResourceRequest').implementation = function (webView, request) {

		...
		if (!isFavicon) {
	    	readNewStream(inputStr);
		    ...
		    
		    webResourceRes = WebResourceResponse.$new("text/html","utf-8", inputStr);
		} else {
			webResourceRes = WebResourceResponse.$new("image/vnd.microsoft.icon","gzip", inputStr);
		}
	    
	    return webResourceRes;
	}
});


```

This second script worked as desired, achieving the goal of solving the challenge with Frida.

In the following link [https://github.com/CesarMRodriguez/lunesdemobile/tree/main/Sesion%201](https://github.com/CesarMRodriguez/lunesdemobile/tree/main/Sesion%201) you can find the following content:

*   test: Application written with Android Studio to practice the script from step 2. There you will find two Fragments. The first fragment has the code used in step 1, and the second one has an empty structure in order to generate the script manually.
*   hpandro.apk: Application with challenges
*   http_task.js: First script shown
*   http_task2.js: Second script shown