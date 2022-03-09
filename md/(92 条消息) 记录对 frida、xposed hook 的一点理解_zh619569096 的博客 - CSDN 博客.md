> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/zh619569096/article/details/119887526)

### 记录对 frida、xposed hook 的一点理解

  
先说结论：hook 就类似于在自己项目里导入了一个第三方 jar，一般情况下不需要去关注 app 内的方法是怎么实现的，只需要确定需要传递的参数即可。使用方式和调用 api 类似。

由于数据抓取问题接触了安卓逆向，先后使用了 frida 与 xposed，但之前在使用上一直存在误区，例如下面这段代码

```
Java.perform(function() {
		var SecurityGuardManager = Java.use("com.alibaba.wireless.security.open.SecurityGuardManager");
		var MtopUtils = Java.use("mtopsdk.common.util.MtopUtils");
		var Map = Java.use("java.util.HashMap");
		var Md5 = Java.use("mtopsdk.security.util.SecurityUtils");
		var SecurityGuardParamContext = Java.use("com.alibaba.wireless.security.open.SecurityGuardParamContext");
		/**
		省略一部分代码
		**/
		var input = hashMap.get("utdid") + "&" + hashMap.get("uid") + "&&" + hashMap.get("appKey") + "&" + hashMap.get("data") + "&" + hashMap.get("t") + "&" + hashMap.get("api") + "&" + hashMap.get("v") + "&" + hashMap.get("sid") + "&" + hashMap.get("ttid") + "&" + hashMap.get("deviceId") + "&" + hashMap.get("lat") + "&" + hashMap.get("lng") + "&" + hashMap.get("x-features");
		var inputMap = Map.$new();
		inputMap.put("INPUT", input);
		var securityGuardParamContext = SecurityGuardParamContext.$new();
		securityGuardParamContext.appKey.value = hashMap.get("appKey");
		securityGuardParamContext.requestType.value = 7;
		securityGuardParamContext.paramMap.value = inputMap;
		var sign = SecurityGuardManager.getInstance(MtopUtils.getContext()).getSecureSignatureComp().signRequest(securityGuardParamContext, "");
		console.log("sign:" + sign);
	});
```

这段代码是想实现主动调用 frida 获取加密参数 sign，运行效果没有问题可以获取 sign，但是代码实现属于脱裤子放屁多此一举。。。。。尤其是后来想按照这个逻辑编写 xposed 插件时，更是难以实现。

特此记录一下，引以为戒

### 记录 xposed 插件间通信方式

  
需要通过遍历 ID 的方式抓取某 APP 数据，最初的想法是为插件写个界面，在界面里设置要遍历的 ID 范围，通过一个按钮控制生成 sign 方法的启停，sign 生成后发送 http 通过一个 web 服务将 sign 存入 redis，然后服务器上的爬虫轮询 redis 进行数据抓取。  
实际编写时遇到问题，即 Activity 与 hook 方法数据不能互通。原因是因为 Activity 与 hook 就不在一个进程内，虽然 Activity 与 hook 方法在同一个 app 内，所以需要将 Activity 与 hook 看作两个不同的 app，数据想要互通就需要考虑跨进程通讯。  
最后是通过广播实现的数据交互，当然还有其他解决方式，但对于一个 android 小白来说，这算是最简单的方法了。  
**再补充一点，同一个 handleLoadPackage 方法里不管你 hook 多少接口，只要 hook 的应用不同数据变不能共享，想共享数据就必须跨进程，相应的如果 hook 的应用相同，数据便是共享的。**

下面来说下具体的实现  
1. 首先是在 Activity 的按钮点击监听事件中，发送广播

```
// 首先是在Activity的按钮点击监听事件中，发送广播
        loginButton.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
							 // 要广播的参数
                JSONObject jsonObject = new JSONObject();
                jsonObject.put("startId", usernameEditText.getText().toString());
                jsonObject.put("endId", passwordEditText.getText().toString());
                jsonObject.put("status", loginButton.getText().equals("启动") ? 1 : 2);

                // 发送广播
                Intent i = new Intent("koubei_intent_send");
                i.putExtra("json", jsonObject.toJSONString());
                sendBroadcast(i, null);
                // 修改按钮文字，输出提示信息
                if(loginButton.getText().equals("启动")) {
                    updateUiWithUser("任务已启动");
                    loginButton.setText("停止");
                } else {
                    updateUiWithUser("任务已停止");
                    loginButton.setText("启动");
                }
            }
        });
```

2. 在 handleLoadPackage 里需要 hook 两个接口，一个是 Application 的 onCreate 方法并在此处动态注册自己的广播接收者（**最好通过 processName 名称过滤一下，不然有可能多次注册广播接收者**）。  
另一个就是 hook sign 的生成方法了。

```
/**
     * 实现广播接收
     * */
    private final BroadcastReceiver myReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            if (intent.getAction().equals("koubei_intent_send")) {
                JSONObject jsonObject = JSONObject.parseObject(intent.getStringExtra("json"));
                startId = jsonObject.getInteger("startId");
                endId = jsonObject.getInteger("endId");
                status = jsonObject.getInteger("status");
                XposedBridge.log("=================>接收广播【json】:" + intent.getStringExtra("json"));
                if(status == 1) {
                    new Thread(() -> {
                    	// 具体的sign参数生成方法，这里省略了。。。
                        createSign();
                    }).start();
                }
            }
        }
    };
    
    // 实现目标接口hook
	@Override
    public void handleLoadPackage(LoadPackageParam loadPackageParam) throws Throwable {
        
        if(loadPackageParam.packageName.equals("com.taobao.mobile.dipei")) {
            /**
             * hook Application并注册广播接收者
             */
            XposedHelpers.findAndHookMethod(Application.class, "onCreate", new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    super.afterHookedMethod(param);
                    application = (Application) param.thisObject;

                    // 过滤一下进程名称，不然广播接收会被多次注册，
                    if(intentFilter == null && loadPackageParam.processName.equals("com.taobao.mobile.dipei")) {
                        XposedBridge.log("=================>【" + loadPackageParam.processName + "】注册广播接收者");
                        intentFilter = new IntentFilter();
                        intentFilter.addAction("koubei_intent_send");//要接收的广播
                        application.registerReceiver(myReceiver, intentFilter);//注册接收者
                    }
                }
            });

            /**
             * hook sign参数生成接口
             */
            XposedHelpers.findAndHookMethod(
                 // 获取生成sign的method及object，将其赋值给当前类中的类变量 其他方便便可以调用到了。
            );
        }
    }
		
	// 定义具体的sign生成方法
    private void createSign() {
		// 生成sign，通过okhttp发送给web服务器进行保存，爬虫轮询抓取。
		// 这样做的好处就是不用限制网络，只要有网就能运行，不过这并不是唯一的解决方式，像内网穿透之类的也可以	
	}
```