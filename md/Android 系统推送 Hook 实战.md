> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.mythsman.com](https://blog.mythsman.com/post/60d983618d978138ba12ed16/)

> 背景随着数据生产功能的逐渐稳定，工作重点开始从保证数据总量转移到了保证数据实效性上来了。

随着数据生产功能的逐渐稳定，工作重点开始从**保证数据总量**转移到了**保证数据实效性**上来了。传统的定时爬虫能比较轻松的把数据总量做起来，但是对于很多热点数据，却很难做到实时获取。

考虑到大部分产品、尤其是新闻资讯类的产品，都会对热点数据做推送拉活，如果能拦截到这些数据，那么我们就能应当将数据实效性提升一个档次。

这次我们就主要尝试拦截下小米手机的系统通道的推送数据。当然，对于各个 App 自身的通道，也可以用类似的方法做到拦截，不过那就需要对各个 App 再做好保活。

![](https://blog.mythsman.com/content/images/2021/06/image-33.png)

以小米官方的文档为例，整体推送流程大致分四步：

1.  应用客户端在启动时向 MiPush SDK 中注册当前设备，并获得对应的唯一标识 regId。
2.  应用服务端告诉小米统一推送服务，他需要向某个指定账号、指定类型、或指定设备推送消息。
3.  小米统一的服务端通过与手机上的 MiPush SDK 的长连接，向手机推送数据，并展示在通知栏中。
4.  应用客户端实现一个自定义 PushMessageReceiver 做好数据解析的准备，等用户点击通知栏后就可以将数据传过去。

牵涉到的大概有五方的代码：

1.  手机内部的 MIUI 框架部分代码。
2.  应用客户端代码。
3.  应用客户端引用的小米的 MiPush SDK 代码。
4.  小米 PushServer 的代码。
5.  应用服务端的代码。

在这里当中，我们其实只需要关心 1、3 两部分的代码即可。

简单翻阅了网络上已知的策略，获得通知栏的推送数据一般有如下思路：

1.  通过 Appium 等自动化测试工具，直接获取通知栏的元素中的消息文本。
2.  自己写一个 App ，实现 NotificationListenerService 方法，监听所有过来的消息。
3.  通过 Frida 等 hook 工具，直接拦截 MiPush 推过来的原始数据，自行做解析和解密。

虽然我并没有做尝试，但是前两种方式似乎最多只能获得推送的标题和文本等基本信息，无法获得更详细的原始信息（尤其是跳转 scheme 、参数列表等）。因此我们主要尝试第三种思路。

下面就以手头的 Redmi6 设备进行尝试。

定位应用
----

既然我们希望获取通知栏的数据，那我们首先应当定位到通知栏的相关代码。拉出通知栏，dumpsys 一下：

```
$ adb shell dumpsys activity top|grep ACTIVITY
ACTIVITY com.android.systemui/.recents.RecentsActivity c2efb6 pid=30467
ACTIVITY com.ss.android.ugc.aweme/.splash.SplashActivity f9d8beb pid=7270
ACTIVITY com.miui.home/.launcher.Launcher 119c6 pid=1950
```

看起来 com.android.systemui 和 com.miui.home 这两个包比较可疑，我们再用 Appium check 一下。

![](https://blog.mythsman.com/content/images/2021/06/image-34.png)

这样就肯定是 com.android.systemui 没跑了。

获取代码
----

定位到包名，那就去手机上把包搞下来就行：

```
$ adb shell pm list packages -f |grep com.android.systemui
package:/vendor/overlay/SysuiDarkTheme/SysuiDarkThemeOverlay.apk=com.android.systemui.theme.dark
package:/system/priv-app/MiuiSystemUI/MiuiSystemUI.apk=com.android.systemui
 
$ adb pull /system/priv-app/MiuiSystemUI/MiuiSystemUI.apk
/system/priv-app/MiuiSystemUI/MiuiSystemUI.apk: 1 file pulled, 0 skipped. 10.5 MB/s (4901017 bytes in 0.446s)
 
$ ls -la MiuiSystemUI.apk
-rw-r--r--  1 myths  staff  4901017 Jun 29 15:42 MiuiSystemUI.apk
```

然后拿 jadx 打开看看：

![](https://blog.mythsman.com/content/images/2021/06/image-35.png)

发现不对劲，怎么一行代码也没有，都是些没用的资源文件。。。在尝试了诸如 jeb、androidkiller、apktool 等反编译软件均失败后，我陷入了沉思。

沉思过后，决定换一部更高级的 Redmi9 手机试试，结果发现 Redmi9 手机的 MiuiSystemUI.apk 竟然是有代码的，估计是 Redmi6 用了什么古老的科技把代码隐藏掉或者下沉了啥的，但是代码一定是有的，否则哪来的 `com.android.systemui/.recents.RecentsActivity` 这个 activity。

于是我暴力 grep 了 /system 目录下的文件，果然给我找到了线索：

```
$ adb shell su -c "grep RecentsActivity -r /system"
grep: /system/framework/miuisdk.jar: No such file or directory
grep: /system/framework/miuisystem.jar: No such file or directory
Binary file /system/framework/oat/arm/services.vdex matches
Binary file /system/priv-app/MiuiSystemUI/oat/arm/MiuiSystemUI.vdex matches
```

发现了一个奇怪的 `MiuiSystemUI.vdex` 文件，简单查了下，应当是为了优化系统 [OTA](https://source.android.google.cn/devices/tech/ota?hl=zh-cn) 时 dex2oat 过程的类似预编译的东西，本质还是代码片段。

于是用 [vdexExtractor](https://github.com/anestisb/vdexExtractor) 转成 MiuiSystemUI.dex  :

```
$ ./vdexExtractor -i  ../../Jadx/MiuiSystemUI.vdex  -o output
[INFO] Processing 1 file(s) from ../../Jadx/MiuiSystemUI.vdex
[INFO] 1 out of 1 Vdex files have been processed
[INFO] 1 Dex files have been extracted in total
[INFO] Extracted Dex files are available in 'output'
```

再用 jadx 打开这个 MiuiSystemUI.dex ：

![](https://blog.mythsman.com/content/images/2021/06/image-36.png)

这下就舒服了。

代码结构
----

对源码经过简单阅读，再辅以 [objection](https://github.com/sensepost/objection) 和 frida 的 Java.choose() 方法动态调试后发现存储通知栏消息的地方主要是在 `com.android.systemui.statusbar.phone.NotificationGroupManager` 。这个类在系统中似乎还是个单例。

找到这个类之后，四下观察就可以整理出大致如下的类图：

![](https://blog.mythsman.com/content/images/2021/06/image-37.png)

这里每个 NotificationData.Entry 就是每一条推送记录，不断深入看下去就会发现真正的数据还是指向了 StatusBarNotification 这个类。而这个类的源码只能去对应的 Android SDK 里去看了。

于是打开了对应版本（api 27）的 SDK，继续整理了一下核心数据涉及到的类：

![](https://blog.mythsman.com/content/images/2021/06/image-38.png)

数据解析
----

跟踪上面涉及到的相关对象，根据所需要的数据做相关解析和拼接即可：

```
Java.perform(function () {

    let MurmurHash3 = {
        mul32: function (m, n) {
            let nlo = n & 0xffff;
            let nhi = n - nlo;
            return ((nhi * m | 0) + (nlo * m | 0)) | 0;
        },
        hashString: function (data, len, seed) {
            let c1 = 0xcc9e2d51, c2 = 0x1b873593;

            let h1 = seed;
            let k1 = '';
            let roundedEnd = len & ~0x1;

            for (let i = 0; i < roundedEnd; i += 2) {
                let k1 = data.charCodeAt(i) | (data.charCodeAt(i + 1) << 16);

                k1 = this.mul32(k1, c1);
                k1 = ((k1 & 0x1ffff) << 15) | (k1 >>> 17);
                k1 = this.mul32(k1, c2);

                h1 ^= k1;
                h1 = ((h1 & 0x7ffff) << 13) | (h1 >>> 19);
                h1 = (h1 * 5 + 0xe6546b64) | 0;
            }

            if ((len % 2) === 1) {
                k1 = data.charCodeAt(roundedEnd);
                k1 = this.mul32(k1, c1);
                k1 = ((k1 & 0x1ffff) << 15) | (k1 >>> 17);
                k1 = this.mul32(k1, c2);
                h1 ^= k1;
            }

            h1 ^= (len << 1);

            h1 ^= h1 >>> 16;
            h1 = this.mul32(h1, 0x85ebca6b);
            h1 ^= h1 >>> 13;
            h1 = this.mul32(h1, 0xc2b2ae35);
            h1 ^= h1 >>> 16;

            return h1;
        }
    };

    let NotificationGroup = Java.use('com.android.systemui.statusbar.phone.NotificationGroupManager$NotificationGroup')
    let Entry = Java.use('com.android.systemui.statusbar.NotificationData$Entry')
    let Base64 = Java.use('android.util.Base64');

    function processInfo(statusbar, folded) {
        let mmap = statusbar.notification.value.extras.value.mMap.value;

        if (statusbar.notification.value.contentIntent.value) {
            let intent = statusbar.notification.value.contentIntent.value.getIntent();
            if (intent.mComponent.value) {
                let txt = mmap.get("android.text").toString()
                let id = MurmurHash3.hashString(txt, txt.length, 0).toString().replace("-", "")
                let msg = {
                    'type': 'mi_push',
                    'data': {
                        "folded": folded,
                        "id": id,
                        "intent": intent.getDataString() === null ? "" : intent.getDataString(),
                        "title": mmap.get("android.title").toString(),
                        "text": mmap.get("android.text").toString(),
                        "package": intent.mComponent.value.mPackage.value,
                        "class": intent.mComponent.value.mClass.value,
                        "time": statusbar.postTime.value
                    },
                    'id': id
                };
                let bundle = intent.getExtras()
                if (bundle) {
                    let miPushByte = bundle.get("mipush_payload")
                    if (miPushByte) {
                        let arr = Java.array('byte', miPushByte)
                        msg['data']['mipush'] = Base64.encodeToString(arr, 0).replaceAll("\n", "");
                    }

                }
                send(JSON.stringify(msg));
            } else {
                console.log("contentIntent is null " + mmap)
            }
        } else {
            console.log("intent.mComponent.value not exist " + mmap)
        }
    }


    function processManager(manager) {
        try {
            let entrys = manager.mGroupMap.value.keySet()
            let it = entrys.iterator();
            while (it.hasNext()) {
                let key = it.next();
                let data = manager.mGroupMap.value.get(key)
                let nData = Java.cast(data, NotificationGroup)
                let unfoldIterator = nData.unfoldChildren.value.iterator()
                let foldIterator = nData.foldChildren.value.iterator()

                while (unfoldIterator.hasNext()) {
                    let entry = Java.cast(unfoldIterator.next(), Entry);
                    let row = entry.row.value
                    let statusbar = row.mExpandedNotification.value
                    processInfo(statusbar, false)

                }
                while (foldIterator.hasNext()) {
                    let entry = Java.cast(foldIterator.next(), Entry);
                    let row = entry.row.value
                    let statusbar = row.mExpandedNotification.value
                    processInfo(statusbar, true)
                }
            }
        } catch (e) {
            console.log(e.stack)
        }
    }
    Java.choose("com.android.systemui.statusbar.phone.NotificationGroupManager", {
        onMatch: function (instance) {
            processManager(instance)
        },
        onComplete: function () {
        }
    });
});
```

需要注意的是，这里的代码和 Android 版本、MIUI 版本都有关，不同种的设备之间大概率是不能兼容的。在我的手机上，这样解析出来数据如下：

```
{
    "package": "com.smile.gifmaker",
    "mipush": "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bGxiYWNrLnBhcmFtAAAAMDE0M0pyQnQ0R2JKVzAwNjM3KjFFTU5QV2pjZHA1dW5zZXQ4MncwTVdoNlJlZG1pNgAAAA1fX3RhcmdldF9uYW1lAAAAQE5UTWpMRk1ZNnRQWkpONFcrdytvZndvTUZOOWxhZDNublFUOWJPUEEyeUNJSDlrSml2bWM5V1Jmb0R0ZWRXOWoAAAAFZmVfdHMAAAANMTYyNDkzMjg4MDk2MgAAAA1jYWxsYmFjay50eXBlAAAAAjE3AAAAF25vdGlmaWNhdGlvbl9zdHlsZV90eXBlAAAAATEAAAAIY2FsbGJhY2sAAAA+aHR0cHM6Ly9wdXNoLmtzYXBpc3J2LmNvbS9yZXN0L2luZnJhL3B1c2gvcHJvdmlkZXIvbXQvY2FsbGJhY2sAAAAGX19tX3RzAAAADTE2MjQ5MzI4ODEzNDQNAAsLCwAAAAEAAAAKc2NvcmVfaW5mbwAABIl7InNlcnZlcl9zY29yZSI6MSwiZ3JvdXBfaW50ZXJ2YWwiOjcyMDAwMDAsImV4dHJhX2luZm8iOiJbe1widlwiOjAsXCJiXCI6LTAuOTI4NjcyMzk3NDQxOTE1NSxcIndcIjpbMTQuMDQyMzAwNjYwODQ4MDQyLC0wLjEwNDgyMTA5MTUxOTE1MzA5LDAsMCwwXSxcImVsXCI6W1szMCwwXSxbMCwwXV0sXCJjbFwiOltbMCwwXV0sXCJjZ1wiOltbMCwwXV0sXCJwa2dcIjp7XCJjb20udGVuY2VudC5tbVwiOjksXCJjb20udGVuY2VudC5tb2JpbGVxcVwiOjksXCJjb20uYW5kcm9pZC5tbXNcIjo5LFwiY29tLmFuZHJvaWQuY29udGFjdHNcIjo5LFwiY29tLmFuZHJvaWQucHJvdmlkZXJzLmNvbnRhY3RzXCI6OSxcImNvbS5hbmRyb2lkLmNhbGVuZGFyXCI6OSxcImNvbS5hbmRyb2lkLnByb3ZpZGVycy5jYWxlbmRhclwiOjksXCJjb20uZ29vZ2xlLmFuZHJvaWQuY2FsZW5kYXJcIjo5LFwiY29tLndoYXRzYXBwXCI6OSxcImNvbS5mYWNlYm9vay5vcmNhXCI6OSxcImNvbS5nb29nbGUuYW5kcm9pZC5nbVwiOjksXCJjb20uYW5kcm9pZC5kZXNrY2xvY2tcIjo5LFwiY29tLmFuZHJvaWQucGhvbmVcIjo5LFwiY29tLmFuZHJvaWQuc3RrXCI6OSxcImNvbS5hbmRyb2lkLmNlbGxicm9hZGNhc3RyZWNlaXZlclwiOjksXCJjb20uYW5kcm9pZC5pbmNhbGx1aVwiOjksXCJjb20uYW5kcm9pZC5zZXJ2ZXIudGVsZWNvbVwiOjksXCJjb20uYW5kcm9pZC5lbWFpbFwiOjksXCJjb20ueGlhb21pLmNoYW5uZWxcIjo5LFwiY29tLmFuZHJvaWQudXBkYXRlclwiOjksXCJjb20uYW5kcm9pZC5zZXR0aW5nc1wiOjksXCJjb20ubWl1aS5wbGF5ZXJcIjo5LFwiY29tLm1pdWkuYnVncmVwb3J0XCI6OSxcImNvbS5lZy5hbmRyb2lkLkFsaXBheUdwaG9uZVwiOjksXCJjb20ueGlhb21pLm1hcmtldFwiOjksXCJjb20uYW5kcm9pZC5wcm92aWRlcnMuZG93bmxvYWRzXCI6OSxcImNvbS5hbGliYWJhLmFuZHJvaWQucmltZXQuYWxpZGluZ3RhbGtcIjo5LFwiY29tLmFsaWJhYmEuYW5kcm9pZC5yaW1ldFwiOjksXCJjb20udGVuY2VudC50aW1cIjo5LFwiY29tLm1pdWkuaG9tZVwiOjl9LFwidGhcIjowLjUsXCJuXCI6M31dIiwidGhyZXNob2xkIjowLjY3LCJyYXdfc2NvcmUiOjAuNjQ3Mzc2NTY0OTk5MDQ1NSwic29ydF9kZWxheSI6MTAwMDAsInNlcnZlcl9zdHJhdGVneSI6ImxyIn0CAAwAAAA=",
    "text": "你看过的娟姐（手机摄影）发布了新作品：夏天的蚊子好多啊～太讨厌了，整个大蚊香把他们熏晕吧！ #手机摄影 #创作灵感 #创意",
    "title": "快手",
    "folded": false,
    "id": "scm59254624932880962jS",
    "time": "1624932881487",
    "class": "com.xiaomi.mipush.sdk.PushMessageHandler"
}
```

可以发现，在 intent 的 bundle 里有一个 key 为 "mipush_payload" 的加密数据（我这里展示成了 base64），这里存的就是加密后的推送数据详情，我们还需要继续做处理。

数据解密
----

### 入口

数据解密的逻辑应当是放在 MiPush SDK 中，我们就只能去[小米开发者中心](https://admin.xmpush.xiaomi.com/zh_CN/mipush/downpage)把源码下下来看看。小米这里的处理比较让人蛋疼，一定要注册小米开发者然后才能下载代码。求爹爹告奶奶终于整到了 SDK，下面就以 MiPush_SDK_Client_4_0_2.jar 这个版本来看。

很容易想到的入手点就是 "mipush_payload" 这个字符串，先简单搜索一番试试：

![](https://blog.mythsman.com/content/images/2021/06/image-39.png)

涉及到的代码不多，挺好。考虑到既然我们想要的是解密，那应当是更关心 getByteArrayExtra 方法。进去看一看：

![](https://blog.mythsman.com/content/images/2021/06/image-40.png)

这里对 action 的值有几个分支判断，我们显然是希望 MESSAGE_ARRIVED 这个事件，那解析代码应当就是这个没跑了。

### 坑点

入口都找到了，剩下就是无聊的人肉跟踪了。不过小米做了字节码级别的混淆，导致在同一个类中有很多类型不同但是名字相同的变量（不符合 java 语法、但是符合 jvm 规范）。这样就使 jadx 无法正常反编译出 java 代码，只能对变量名重命名展示。例如上面的 `b.m36a(this.f43a)` 这个方法，他的函数声明的地方有 jadx 的注释:

![](https://blog.mythsman.com/content/images/2021/06/image-41.png)

为了方便人眼看，这样的操作还是很有帮助的。但是由于在写代码的时候还是需要用真正的变量名，因此在搞清逻辑之后，我们还是只能用 IDEA 去看真正的代码。上面的代码在 IDEA 中就是这样：

![](https://blog.mythsman.com/content/images/2021/06/image-42.png)

不得不说这种代码混淆是真的恶心，在 IDEA 中我还看到了这种槽点十足、普通方法根本无法调用的代码：

![](https://blog.mythsman.com/content/images/2021/06/image-43.png)

上面的一系列操作导致的结果就是我们不得不一律采用反射来调用这些函数。

### 密钥

在经过一系列斗争后，发现这些数据其实是采用了 AES 对称加密，加解密用的 key 是从名为 mipush 的 sharedpreferences 中的 regSec 中获得的：

```
//com.xiaomi.push.i
private static Cipher a(byte[] bArr, int i) {
    SecretKeySpec secretKeySpec = new SecretKeySpec(bArr, "AES");
    IvParameterSpec ivParameterSpec = new IvParameterSpec(a);
    Cipher instance = Cipher.getInstance("AES/CBC/PKCS5Padding");
    instance.init(i, secretKeySpec, ivParameterSpec);
    return instance;
}
 
//com.xiaomi.mipush.sdk.b
public static SharedPreferences a(Context context) {
    return context.getSharedPreferences("mipush", 0);
}
 
//com.xiaomi.mipush.sdk.b
public static a a(Context context, String str) {
    try {
        JSONObject jSONObject = new JSONObject(str);
        a aVar = new a(context);
        aVar.f62a = jSONObject.getString("appId");
        aVar.b = jSONObject.getString("appToken");
        aVar.c = jSONObject.getString("regId");
        aVar.d = jSONObject.getString("regSec");//这个！
        aVar.f = jSONObject.getString("devId");
        aVar.e = jSONObject.getString("vName");
        aVar.f63a = jSONObject.getBoolean("valid");
        aVar.f64b = jSONObject.getBoolean("paused");
        aVar.a = jSONObject.getInt("envType");
        aVar.g = jSONObject.getString("regResource");
        return aVar;
    } catch (Throwable th) {
        com.xiaomi.channel.commonutils.logger.b.a(th);
        return null;
    }
}
```

密钥的位置在对应 app 内部存储目录的 shared_prefs 文件夹下。以 gifmaker 为例：

```
$ adb shell su -c 'cat /data/data/com.smile.gifmaker/shared_prefs/mipush.xml'
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string >951ECBAB1F2EB405ABF912BD65FC703FF43969CE</string>
    <boolean  />
    <string >oGZD6g</string>
    <string >8.2.31.17191</string>
    <string >,sdm580566248810824873c,sdm58782624881295100Zr</string>
    <int  />
    <string >5431713053534</string>
    <string >2882303761517130534</string>
    <string >NTMjLFMY6tPZJN4W+w+ofwoMFN9lad3nnQT9bOPA2yCIH9kJivmc9WRfoDtedW9j</string>
    <string >AMjTZmbZx2LS1X7dp0AAAA==</string>
    <string >China</string>
</map>
```

这里的密钥就是 AMjTZmbZx2LS1X7dp0AAAA== 。

需要注意的是，这个密钥是在 app 初次启动时生成的，不通设备之间不一样，相同设备的不同次安装也不一样。

### 实现

最后把相关代码拼接下，在服务端实现一个服务即可。代码参考如下：

```
public class PushMessage implements Serializable {
    private static final long serialVersionUID = 4745289224411613134L;
 
    private String id;
    private Long channelId;
    private String userId;
    private String server;
    private String resource;
    private String appId;
    private String packageName;
    private String payload;
    private Long createAt;
    private Long ttl;
    private Boolean isOnline;
    private Boolean needsAck;
    
    //some getter and setter
}
 
public PushMessage decrypt(String secKey, String payload) {
    try {
        byte[] data = Base64.getDecoder().decode(payload);
        byte[] key = Base64.getDecoder().decode(secKey);
 
        im im = new im();
 
        ja.a(im, data);
 
        byte[] buffer = null;
        for (Method declaredMethod : im.getClass().getDeclaredMethods()) {
            if (declaredMethod.getReturnType().equals(byte[].class)) {
                try {
                    buffer = (byte[]) declaredMethod.invoke(im);
                } catch (IllegalAccessException | InvocationTargetException e) {
                    throw new RuntimeException(e);
                }
            }
        }
        if (buffer == null) {
            throw new RuntimeException("buffer is null");
        }
 
 
        byte[] decrypted = i.a(key, buffer);
        it a3 = null;
        for (Method declaredMethod : com.xiaomi.mipush.sdk.ai.class.getDeclaredMethods()) {
            if (declaredMethod.getName().equals("a")
                    && declaredMethod.getReturnType().equals(jb.class)
                    && declaredMethod.getModifiers() == (Modifier.PRIVATE | Modifier.STATIC)) {
                declaredMethod.setAccessible(true);
                try {
                    a3 = (it) declaredMethod.invoke(com.xiaomi.mipush.sdk.ai.class, hq.a(5), true);
                } catch (IllegalAccessException | InvocationTargetException e) {
                    throw new RuntimeException("reflect invoke failed", e);
                }
            }
        }
 
        if (a3 == null) {
            throw new RuntimeException("a3 not found");
        }
        new jf(new com.xiaomi.push.js.a(true, true, data.length)).a(a3, decrypted);
 
        PushMessage pushMessage = convertToMessage(a3);
        logger.info("decrypt for {} \ndata: {}\nresult: {}", secKey, payload, JSONObject.toJSONString(pushMessage));
        return pushMessage;
    } catch (DecryptFailedException e) {
        logger.error("decrypt for {} failed\ndata: {}", secKey, payload, e);
        throw e;
    } catch (Throwable e) {
        RuntimeException ex = new RuntimeException("decode failed,", e);
        logger.error("decrypt for {} failed\ndata: {}", secKey, payload, ex);
        throw ex;
    }
}
 
private PushMessage convertToMessage(it result) throws IllegalAccessException, ClassNotFoundException {
    PushMessage pushMessage = new PushMessage();
 
    Field idField = findFieldByNameAndType(it.class, "b", String.class);
    pushMessage.setId((String) idField.get(result));
 
    Field needsAckField = findFieldByNameAndType(it.class, "a", boolean.class);
    pushMessage.setNeedsAck((boolean) needsAckField.get(result));
 
    Field appIdField = findFieldByNameAndType(it.class, "c", String.class);
    pushMessage.setAppId((String) appIdField.get(result));
 
    Field packageNameField = findFieldByNameAndType(it.class, "d", String.class);
    pushMessage.setPackageName((String) packageNameField.get(result));
 
    Class<?> ifClass = Class.forName("com.xiaomi.push.if");
    Field ifField = findFieldByNameAndType(it.class, "a", ifClass);
    Object target = ifClass.cast(ifField.get(result));
 
    Field targetField = findFieldByNameAndType(ifClass, "a", long.class);
    pushMessage.setChannelId((long) targetField.get(target));
 
    Field userIdField = findFieldByNameAndType(ifClass, "a", String.class);
    pushMessage.setUserId((String) userIdField.get(target));
 
    Field serverField = findFieldByNameAndType(ifClass, "b", String.class);
    pushMessage.setServer((String) serverField.get(target));
 
    Field resourceField = findFieldByNameAndType(ifClass, "c", String.class);
    pushMessage.setResource((String) resourceField.get(target));
 
    Field icField = findFieldByNameAndType(it.class, "a", ic.class);
    ic pmsg = (ic) icField.get(result);
 
    Field payloadField = findFieldByNameAndType(ic.class, "c", String.class);
    pushMessage.setPayload((String) payloadField.get(pmsg));
 
    Field createAtField = findFieldByNameAndType(ic.class, "a", long.class);
    pushMessage.setCreateAt((long) createAtField.get(pmsg));
 
    Field ttlField = findFieldByNameAndType(ic.class, "b", long.class);
    pushMessage.setTtl((long) ttlField.get(pmsg));
 
    Field onlineField = findFieldByNameAndType(ic.class, "a", boolean.class);
    pushMessage.setOnline((boolean) onlineField.get(pmsg));
 
    return pushMessage;
}
 
private Field findFieldByNameAndType(Class<?> clazz, String name, Class<?> returnType) {
    for (Field field : clazz.getDeclaredFields()) {
        if (field.getName().equals(name) && field.getType().equals(returnType)) {
            return field;
        }
    }
    throw new FieldNotFoundException(String.format("cannot find name->%s returnType->%s for class->%s", name, returnType, clazz));
}
```

mipush 的内部实现依赖了 android 相关的类，虽然在解密的过程中并没有用到，但是如果类不存在还是会报错的。因此还需要加两个 mock 类，空实现即可。

```
package android.content;
 
public class Context {
}
```

```
package android.text;
 
public class TextUtils {
}
```

由于 java 编译器的关系，在运行时需要加上 `-noverify` 的参数，否则会报 `java.lang.VerifyError` 的错。（参考 [sf](https://stackoverflow.com/questions/15122890/java-lang-verifyerror-expecting-a-stackmap-frame-at-branch-target-jdk-1-7) ）

### 运行

最后，我们把本次从 gifmaker 拿到的推送解析一把：

```
{
    "appId": "2882303761517130534",
    "channelId": 5,
    "createAt": 1624932880962,
    "id": "scm59254624932880962jS",
    "needsAck": true,
    "online": true,
    "packageName": "com.smile.gifmaker",
    "payload": "{\"click_payload\":\"true\",\"push_notification\":\"{}\",\"infra_tag\":\"cdp\",\"onlyInBar\":\"false\",\"dndModeIsOn\":\"false\",\"push_msg_id\":\"0000016249328800885970645000158\",\"id\":\"0000016249328800885970645000158\",\"push_back\":\"143JrBt4GbJW00637*1EMNPWjcdp5unset82w0MWh6Redmi6oANDROID_021f5a0bbeeb1eb1\",\"server_key\":\"{\\\"business\\\":\\\"RECO_PUSH\\\",\\\"object_type\\\":\\\"OBJECT_PHOTO\\\",\\\"item_id\\\":\\\"0\\\",\\\"ks_order_id\\\":\\\"empty\\\",\\\"object_id\\\":\\\"5205879746946008323\\\",\\\"sender_id\\\":\\\"0\\\",\\\"push_type\\\":\\\"PUSH_PHOTO\\\",\\\"infra_tag\\\":\\\"cdp\\\",\\\"event_type\\\":\\\"EVENT_CONSUME_AUTHOR_PHOTO_PUSH\\\",\\\"time_ms\\\":\\\"1624932880244\\\",\\\"business_type\\\":\\\"KUAISHOU\\\",\\\"badge\\\":0}\",\"title\":\"快手\",\"body\":\"你看过的娟姐（手机摄影）发布了新作品：夏天的蚊子好多啊～太讨厌了，整个大蚊香把他们熏晕吧！ #手机摄影 #创作灵感 #创意\",\"uri\":\"kwai:\\/\\/work\\/5205879746946008323?userId=2331994191&exp_tag=1_a\\/0_ps\"}",
    "resource": "anSQRSEH",
    "server": "xiaomi.com",
    "ttl": 86401,
    "userId": "70084876406"
}
```

其中的 payload 再展开看下：

```
{
    "click_payload": "true",
    "push_notification": "{}",
    "infra_tag": "cdp",
    "onlyInBar": "false",
    "dndModeIsOn": "false",
    "push_msg_id": "0000016249328800885970645000158",
    "id": "0000016249328800885970645000158",
    "push_back": "143JrBt4GbJW00637*1EMNPWjcdp5unset82w0MWh6Redmi6oANDROID_021f5a0bbeeb1eb1",
    "server_key": "{\"business\":\"RECO_PUSH\",\"object_type\":\"OBJECT_PHOTO\",\"item_id\":\"0\",\"ks_order_id\":\"empty\",\"object_id\":\"5205879746946008323\",\"sender_id\":\"0\",\"push_type\":\"PUSH_PHOTO\",\"infra_tag\":\"cdp\",\"event_type\":\"EVENT_CONSUME_AUTHOR_PHOTO_PUSH\",\"time_ms\":\"1624932880244\",\"business_type\":\"KUAISHOU\",\"badge\":0}",
    "title": "快手",
    "body": "你看过的娟姐（手机摄影）发布了新作品：夏天的蚊子好多啊～太讨厌了，整个大蚊香把他们熏晕吧！ #手机摄影 #创作灵感 #创意",
    "uri": "kwai://work/5205879746946008323?userId=2331994191&exp_tag=1_a/0_ps"
}
```

信息非常全，还带了跳转用的 scheme，通过`adb shell am start -a android.intent.action.VIEW -d kwai://work/5205879746946008323?userId=2331994191&exp_tag=1_a/0_ps` 就能直接跳转到对应页面了。

通知栏清理
-----

上面的脚本仅仅涉及了获取数据，并没有涉及清理通知栏。显然，如果一直不清理，不仅要考虑数据去重，也会影响拉取效率。实际用起来发现，如果一直不清理，也容易 hook 不动。。。

要清理也很简单，在每次拉取消息结束后，调用一下命令即可：

```
$ adb shell su -c 'service call notification 1'
```

实时监听
----

不要忘了，我们需要的是实时监听，因此还需要监听下`NotificationGroupManager` 的触发器。

```
let NotificationGroupManager = Java.use('com.android.systemui.statusbar.phone.NotificationGroupManager')
let onEntryAdded = NotificationGroupManager.onEntryAdded;
onEntryAdded.implementation = function (entry) {
    let res = onEntryAdded.call(this, entry);
    processManager(this)
    return res;
}
```

不过，从实践中看，想通过直接 hook 一次这个 onEntryAdded 事件就能一劳永逸还是想太多了。Frida 进程或者是桌面进程总会有各种理由挂掉，因此我们在生产中也并没有依赖这个，而是通过外部定时任务来触发。

定时拉活
----

由于小米厂家自身的通道（可能）比较贵，因此各大应用几乎都有很大一部分走的是自己的长链接。与小米自身的系统通道不同，这些长链接都是需要 App 在后台运行才能保证的。因此我们也需要定期把那些重要的 App 进行强制拉活，这样我们能收到的 push 才能更快、更多。不过好消息是，应用自身通道的推送数据是不用走 mipush 加密那一套东西，所以搞起来更简单～

最后反手夸一夸腾讯，看起来各大厂家对热点事件的推送中，腾讯爸爸还是最及时的，运营同学们辛苦了。

[小米推送产品说明](https://dev.mi.com/console/doc/detail?pId=863)

[Android 8.0 VDEX 机制简介](https://wwm0609.github.io/2017/12/21/android-vdex/)

[逆向 settings 实现监控 app 通知](https://pitechan.com/%E9%80%86%E5%90%91Settings%E5%AE%9E%E7%8E%B0%E7%9B%91%E6%8E%A7App%E9%80%9A%E7%9F%A5/)