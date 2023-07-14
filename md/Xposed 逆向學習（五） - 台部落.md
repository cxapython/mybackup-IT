> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.twblogs.net](https://www.twblogs.net/a/5c187e09bd9eee5e41845b95)

> 嚴重聲明 本文的意圖只有一個就是通過分析 app 學習更多的 Xposed 逆向技術，如果有人利用本文知識和技術進行非法操作進行牟利，帶來的任何法律責任都將由操作者本人承擔，和本文作者無任何關係。

本文的意圖只有一個就是通過分析 app 學習更多的 Xposed 逆向技術，如果有人利用本文知識和技術進行非法操作進行牟利，帶來的任何法律責任都將由操作者本人承擔，和本文作者無任何關係。最終還是希望大家能夠秉着學習的心態閱讀此文，也不枉我寫此文的目的，希望大家都能不忘初心，歸來仍是少年！

*   本篇逆向的最終目的就是查找到微信下載保存聊天時原圖的方法，實現當接收到圖片時自動下載原圖到我們指定的路徑中。廢話不多說了，直接進入正題。我們直接通過 ddms 工具來錄製下聊天圖片下載保存的方法執行軌跡。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-04b2efa320056fc4.png)
*   軌跡錄製出來我們就要去定位具體的方法了，這時我就想它既然是下載我就直接來搜索下 download 好了。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-9ef5ffe3363945e4.png)
*   運氣還挺好，搜索到了，好了我直接來 hook 下確認這個方法在下載保存原圖時是否執行了。通過源碼可以看到 startC2CDownload 方法是個 native 方法，但是我們還是可以通過一樣的方法去 hook 它。

```
Class<?> cdn = XposedHelpers.findClassIfExists("com.tencent.mars.cdn.CdnLogic", classLoader);
        Class<?> request = XposedHelpers.findClassIfExists("com.tencent.mars.cdn.CdnLogic$C2CDownloadRequest", classLoader);
        //聊天消息大圖
        XposedHelpers.findAndHookMethod(cdn, "startC2CDownload", request, new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                super.afterHookedMethod(param);
                //因爲錄製了方法軌跡，所以這裏調用信息就不需要打印了，調用信息可以直接從方法軌跡看出來
                //Log.e(TAG, "*********startC2CDownload***********", new Exception());
                Object arg = param.args[0];
                if (arg != null) {
                    Field[] declaredFields = arg.getClass().getDeclaredFields();
                    for (int i = 0; i < declaredFields.length; i++) {
                        Field declaredField = declaredFields[i];
                        String name = declaredField.getName();
                        Object objectField = XposedHelpers.getObjectField(arg, name);
                        if (objectField != null) {
                            Log.e(TAG, name + "：" + objectField.toString());
                        }
                    }
                    Log.e(TAG, "-------------------------------------");
                }
            }
        });
```

![](http://upload-images.jianshu.io/upload_images/4834678-c897fc01d0bf0eb8.png)

*   從日誌可以看出執行了，CdnLogic$C2CDownloadRequest 類對象的參數屬性打印出來很多，我一開始看到有用的就是我框起來的圖片保存路徑。好了通過 hook 該方法證明我們的猜測應該是對的。既然猜測可能是對的，我們就繼續往前找調用該方法的地方，看下能把這個方法的參數構造出來不，如果可以的話，那我們的需求就實現了。從方法軌跡可以看出調用改下載方法的是個 bH 方法。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-35212e5b98187a2b.png)
*   從源碼找到 bH 方法來看下。主要看調用下載方法的地方。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-a233b1747e9b5917.png)
*   好了從源碼點擊跳轉找不到要構造參數的來源，我們可能會想從 ddms 方法軌跡裏去看能找到不，其實是不行的，方法軌跡只能看到調用執行的方法和參數的類型，至於參數調用信息和怎麼得到的來源是無從知道的。我們要知道這個參數的來源這裏提供一個思路就是 hook 那個參數的構造方法，因爲我們前幾篇就用過這種方式，這裏就只提示一下了。我們這裏用另外一個快速的方法，從引包信息找到對應類。我們看下 bH 方法所在類 c 裏面有沒有引入構造參數的類 b 的信息。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-6a0518624629ff19.png)
*   從所有引包信息可以看出只有一個引包信息是引入 b 類的，所以這個很可能就是構造參數的那個 b 類。按引入的包結構從源碼裏找到那個類，並找到構造參數的 a 方法看下。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-15f7c392aaa5c693.png)
*   可以看到這個類的 a 方法就是我們要找的在 bH 方法裏面用於構造下載參數的方法。可以看到是個 static 方法，通過我們之前這麼多篇的經驗來說，一般找到 static 方法基本就離我們的目標不遠了，所以進一步輔證了我們思路是對的。從 a 方法可以看到我們的下載請求的 c2CDownloadRequest 對象是屬性都是從傳過來的 iVar 的屬性賦值得來的，所以我們我們的下一個目標就是 a 方法的參數 iVar 了，點擊跳轉到它的所屬類 i 去看下。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-45199f84222c5d36.png)
*   可以看到 i 類裏面很多變量，沒有看到有參構造方法，所以我們來 hook 下它所有構造方法，並將它調用信息和變量打印出來看下。

```
Class<?> i = XposedHelpers.findClassIfExists("com.tencent.mm.modelcdntran.i", classLoader);
        XposedBridge.hookAllConstructors(i, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                super.beforeHookedMethod(param);
                Log.e(TAG, "i構造方法調用信息：", new Exception());
                Object[] args = param.args;
                if (args != null) {
                    Log.e(TAG, "參數長度：" + args.length, new Exception());
                    for (int i = 0; i < args.length; i++) {
                        if (args[i] != null) {
                            Log.e(TAG, "參數" + i + ":" + args[i].toString());
                        }
                    }
                } else {
                    Log.e(TAG, "沒有有參構造方法");
                }
                Object thisObject = param.thisObject;
                Field[] declaredFields = thisObject.getClass().getDeclaredFields();
                for (int j = 0; j < declaredFields.length; j++) {
                    String name = declaredFields[j].getName();
                    Log.e(TAG, name);
                    Object objectField = XposedHelpers.getObjectField(thisObject, name);
                    if (objectField != null) {
                        Log.e(TAG, "：" + objectField.toString());
                    }
                }
            }
        });
```

![](http://upload-images.jianshu.io/upload_images/4834678-ec1f7ab1eead7211.png)  
![](http://upload-images.jianshu.io/upload_images/4834678-b4adb5660839872d.png)

*   變量實在是太多了，所以變量日誌我只截圖了部分，從日誌可以看出變量的信息並沒有有用信息，再說這麼多的變量要我們都找到並構造出來也不現實，所以我們從調用信息入手，找到調用信息，看源碼是怎麼將 i 對象構造出來的，哪些變量需要賦值。找到我上面框起來的調用信息 k 類 a 方法中構造 i 對象的地方。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-64e0d59e0cb94725.png)
*   可以看到比起 i 類的所有變量來說這裏賦值了的變量少了很多了，雖然知道怎麼構造了但是我們還是無法去構造，因爲我們完全就沒見過下載時 i 類參數具體是什麼樣，我們最起碼的打印出來過一次才能照葫蘆畫瓢將這個參數構造出來啊，沒辦法我又繼續看之前的源碼。前面 bH 方法是調用 startC2CDownload 下載方法的源頭，這裏面也有 i 類的 iVar 參數，我們剛剛好像忽略了它裏面的這個參數是怎麼得來的就直接去跟 i 的構造方法了，現在回過來看下 bH 的 iVar 參數是怎麼得來的。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-03a82ba5f9478b9f.png)  
    ![](http://upload-images.jianshu.io/upload_images/4834678-872ec20a50f2eda2.png)  
    ![](http://upload-images.jianshu.io/upload_images/4834678-713641accc9f39d8.png)
*   可以看到這個參數是從 c 類裏 dOV 這個 Map 對象裏取出來的，那麼我們來 hook 下。

```
XposedHelpers.findAndHookMethod("com.tencent.mm.modelcdntran.c", classLoader, "bH", boolean.class, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                super.beforeHookedMethod(param);
                Object thisObject = param.thisObject;
                Field dOV = thisObject.getClass().getDeclaredField("dOV");
                Map<String, Object> map = (Map) dOV.get(thisObject);
                for (Object v : map.values()) {
                    Field[] declaredFields = v.getClass().getDeclaredFields();
                    for (int i = 0; i < declaredFields.length; i++) {
                        String name = declaredFields[i].getName();
                        Object objectField = XposedHelpers.getObjectField(v, name);
                        if (objectField != null) {
                            Log.e(TAG, name + "：" + objectField);
                        }
                    }
                    Log.e(TAG, "********************************");
                }
            }
        });
```

*   日誌打印如下，這裏注意微信在接收到圖片的時候也會調用該方法去緩存下載縮略圖，所以打印的日誌會有些是縮略圖的，而我們要的是原圖的日誌，所以在點擊下載原圖前最好清理下日誌，這樣得到的就是下載原圖調用該方法的那次日誌了。

```
12-17 17:15:33.629 1284-1306/? E/***Xposed***: allow_mobile_net_download：false
    ceW：false
    dPV：com.tencent.mm.ak.k$2@2b2bcc9f
    dPW：
    dPX：0
    dPY：0
    dQa：true
    dQb：false
    dQc：false
    dQd：
    dQf：false
    dQg：0
    dQh：1
    field_advideoflag：0
    field_aesKey：d0529c835f6487a55bfbc6e683d2a183
    field_appType：0
    field_arg：0
    field_autostart：false
    field_bzScene：0
    field_chattype：0
    field_enable_hitcheck：true
12-17 17:15:33.630 1284-1306/? E/***Xposed***: field_fake_bigfile_signature：
    field_fake_bigfile_signature_aeskey：
    field_fileId：3053020100044c304a0201000204bc33e98c02032f56c10204e8e5e77302045c17691a0425617570696d675f366131353162356437356336373636345f313534353033383130353934330204010438010201000400
    field_fileType：1
    field_filemd5：
    field_force_aeskeycdn：false
    field_fullpath：/storage/emulated/0/tencent/MicroMsg/3695b06082dc3da729994b873bcaadf8/image2/f3/f2/f3f2ad94f24ffb11bde03b9f0f79ada8.temp
    field_isColdSnsData：false
    field_isSilentTask：false
    field_isStreamMedia：false
    field_largesvideo：0
    field_lastProgressCallbackTime：0
    field_limitrate：0
    field_mediaId：adownimg_162cdf33a01e5ed4_1545038109_6
    field_midFileLength：0
    field_midimgpath：
    field_needCompressImage：false
    field_needStorage：false
    field_onlycheckexist：false
    field_preloadRatio：30
    field_priority：2
    field_requestVideoFormat：1
    field_sendmsg_viacdn：false
    field_signalQuality：
    field_smallVideoFlag：0
    field_snsScene：
    field_startTime：0
    field_svr_signature：
    field_talker：
    field_thumbpath：
    field_totalLen：2327743
    field_trysafecdn：false
    field_videoFileId：
    field_videosource：0
    field_wxmsgparam：
    initialDownloadLength：-1
    initialDownloadOffset：-1
    is_resume_task：false
    ********************************
```

*   好了有了對照我們在結合微信自己構造 i 參數的地方來分析下。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-55b1979154c0c2eb.png)  
    ![](http://upload-images.jianshu.io/upload_images/4834678-8b0b2f29f469560f.png)
*   好了簡化下上面打印 i 類參數變量的 hook 方法，只打印出我們從源碼中分析出來需要賦值的那些變量的值。

```
XposedHelpers.findAndHookMethod("com.tencent.mm.modelcdntran.c", classLoader, "bH", boolean.class, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                super.beforeHookedMethod(param);
                Object thisObject = param.thisObject;
                Field dOV = thisObject.getClass().getDeclaredField("dOV");
                Map<String, Object> map = (Map) dOV.get(thisObject);
                for (Object v : map.values()) {
                    Field[] declaredFields = v.getClass().getDeclaredFields();
                    for (int i = 0; i < declaredFields.length; i++) {
                        String name = declaredFields[i].getName();
                        Object objectField = XposedHelpers.getObjectField(v, name);
                        if (objectField != null) {
                            if (name.equals("field_mediaId") || name.equals("field_fullpath") || name.equals("field_fileType")
                                    || name.equals("field_totalLen") || name.equals("field_aesKey") || name.equals("field_fileId")
                                    || name.equals("field_priority") || name.equals("field_chattype") || name.equals("field_autostart"))
                                Log.e(TAG, name + "：" + objectField);
                        }
                    }
                    Log.e(TAG, "********************************");
                }
            }
        });
```

*   這個我就不單獨運行打印一遍日誌了，結果肯定是和上面一樣的，只是刪減了一些對我們無用的變量罷了，避免變量太多不便於我們後面的分析。好了，到現在我們要解決的問題就是要對我們挑出來的 9 個變量進行賦值了。通過日誌我們也沒法直接看出這個變量值有啥特點除了那個 field_fullpath 圖片路徑可能是下載圖片保存的路徑。那麼其它參數要從哪裏得來，我們首先想到的肯定是在包含在接收到圖片的地方，接收消息的方法我們在 [Xposed 逆向學習（四）](https://www.jianshu.com/p/577a05ad4a97)  
    已經找到了，那麼這裏我就直接寫出 hook 代碼了，不再重複了。這裏建議接收消息和上面打印參數 i 的 hook 方法一起運行，便於進行數據對比。

```
//hook參數i對應變量的值
        XposedHelpers.findAndHookMethod("com.tencent.mm.modelcdntran.c", classLoader, "bH", boolean.class, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                super.beforeHookedMethod(param);
                Object thisObject = param.thisObject;
                Field dOV = thisObject.getClass().getDeclaredField("dOV");
                Map<String, Object> map = (Map) dOV.get(thisObject);
                for (Object v : map.values()) {
                    Field[] declaredFields = v.getClass().getDeclaredFields();
                    for (int i = 0; i < declaredFields.length; i++) {
                        String name = declaredFields[i].getName();
                        Object objectField = XposedHelpers.getObjectField(v, name);
                        if (objectField != null) {
                            if (name.equals("field_mediaId") || name.equals("field_fullpath") || name.equals("field_fileType")
                                    || name.equals("field_totalLen") || name.equals("field_aesKey") || name.equals("field_fileId")
                                    || name.equals("field_priority") || name.equals("field_chattype") || name.equals("field_autostart"))
                                Log.e(TAG, name + "：" + objectField);
                        }
                    }
                    Log.e(TAG, "********************************");
                }
            }
        });

        //hook接收微信圖片消息
        XposedHelpers.findAndHookMethod("com.tencent.wcdb.database.SQLiteDatabase", classLoader, "insertWithOnConflict",
                String.class, String.class, ContentValues.class, int.class,
                new XC_MethodHook() {
                    @Override
                    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                        super.afterHookedMethod(param);
                        Object[] args = param.args;
                        if (args == null) return;
                        for (int i = 0; i < args.length; i++) {
                            if (args[i] == null) break;
                            if (i == 2) {
                                ContentValues contentValues = (ContentValues) param.args[i];
                                //挑選出接收圖片類消息
                                if (contentValues.getAsInteger("isSend") == 0 && contentValues.getAsInteger("type") == 3) {
                                    Set<String> keys = contentValues.keySet();
                                    for (String key : keys) {
                                        Log.e(TAG, key + "：" + contentValues.get(key));
                                    }
                                }
                            }
                        }
                    }
                });
```

*   日誌輸出如下

```
E: field_aesKey：76530c123634de7386045e781c67db5b
    field_autostart：false
    field_chattype：0
    field_fileId：3053020100044c304a0201000204bc33e98c02032f56c102049de5e77302045c178dc30425617570696d675f623430363432333236326230656532345f313534353034373439303836300204010438010201000400
    field_fileType：1
    field_fullpath：/storage/emulated/0/tencent/MicroMsg/cbb75469e9efeaaba9e994a45d60077b/image2/43/46/4346b8eab3ddd7623fef47477bf20477.temp
    field_mediaId：adownimg_b58ce21b0bdbd619_1545047493_3
    field_priority：2
E: field_totalLen：3246888
    ********************************

E: bizClientMsgId：
    msgId：3
    msgSvrId：151458135540260912
    talker：wxid_zkhz3yple5do21
    content：<?xml version="1.0"?>
    <msg>
        <img aeskey="76530c123634de7386045e781c67db5b" encryver="0" cdnthumbaeskey="76530c123634de7386045e781c67db5b" cdnthumburl="3053020100044c304a0201000204bc33e98c02032f56c102049de5e77302045c178dc30425617570696d675f623430363432333236326230656532345f313534353034373439303836300204010438010201000400" cdnthumblength="6035" cdnthumbheight="120" cdnthumbwidth="90" cdnmidheight="0" cdnmidwidth="0" cdnhdheight="0" cdnhdwidth="0" cdnmidimgurl="3053020100044c304a0201000204bc33e98c02032f56c102049de5e77302045c178dc30425617570696d675f623430363432333236326230656532345f313534353034373439303836300204010438010201000400" length="85925" cdnbigimgurl="3053020100044c304a0201000204bc33e98c02032f56c102049de5e77302045c178dc30425617570696d675f623430363432333236326230656532345f313534353034373439303836300204010438010201000400" hdlength="3246888" md5="73afc038d01d0c11a6b9d406cdc80fdf" />
    </msg>
    flag：0
    status：3
    msgSeq：680033779
    imgPath：THUMBNAIL_DIRPATH://th_4346b8eab3ddd7623fef47477bf20477
    createTime：1545047492000
    lvbuffer：[B@37d3dd01
    isSend：0
    type：3
    bizChatId：-1
    talkerId：27
```

![](http://upload-images.jianshu.io/upload_images/4834678-b0a3dbc2b1fa7069.png)

```
XposedHelpers.findAndHookMethod("com.tencent.mm.modelcdntran.d", classLoader, "a", String.class, long.class, String.class, String.class, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                super.beforeHookedMethod(param);
                for (Object arg : param.args) {
                    if (arg != null) {
                        Log.e(TAG, arg.toString());
                    }
                }
            }
        });
```

![](http://upload-images.jianshu.io/upload_images/4834678-a48b9006e5cb2dd8.png)

*   好了，齊活，現在所有參數都有了，我們就可以來編寫我們的目標代碼了。

```
//hook接收微信圖片消息
        XposedHelpers.findAndHookMethod("com.tencent.wcdb.database.SQLiteDatabase", classLoader, "insertWithOnConflict",
                String.class, String.class, ContentValues.class, int.class,
                new XC_MethodHook() {
                    @Override
                    protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                        super.afterHookedMethod(param);
                        Object[] args = param.args;
                        if (args == null) return;
                        for (int i = 0; i < args.length; i++) {
                            if (args[i] == null) break;
                            if (i == 2) {
                                ContentValues contentValues = (ContentValues) param.args[i];
                                if (contentValues.get("isSend") == null || contentValues.get("type") == null) {
                                    return;
                                }
                                //挑選出接收圖片類消息
                                if (contentValues.getAsInteger("isSend") == 0 && contentValues.getAsInteger("type") == 3) {
                                    Set<String> keys = contentValues.keySet();
                                    for (String key : keys) {
                                        Log.e(TAG, key + "：" + contentValues.get(key));
                                    }
                                    Log.e(TAG, "---------------------以上是接收到圖片消息數據-------------------------");
                                    String talker = contentValues.getAsString("talker");
                                    String msgId = contentValues.getAsString("msgId");
                                    String content = contentValues.getAsString("content"); //xml數據
                                    //得到 DocumentBuilderFactory 對象
                                    DocumentBuilderFactory builderFactory = DocumentBuilderFactory.newInstance();
                                    //得到DocumentBuilder對象
                                    DocumentBuilder builder = builderFactory.newDocumentBuilder();
                                    //建立Document存放整個xml的Document對象數據
                                    ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(content.getBytes());
                                    Document document = builder.parse(byteArrayInputStream);
                                    //得到xml數據的"根節點"（msg）
                                    Element element = document.getDocumentElement();
                                    //獲取根節點的所有img的節點
                                    NodeList list = element.getElementsByTagName("img");
                                    //獲取img節點
                                    Element img = (Element) list.item(0);
                                    //獲取img的屬性
                                    String aeskey = img.getAttribute("aeskey");
                                    String cdnbigimgurl = img.getAttribute("cdnbigimgurl");
                                    int hdlength = Integer.parseInt(img.getAttribute("hdlength"));
                                    Log.e(TAG, aeskey + "******" + cdnbigimgurl + "******" + hdlength);
                                    //構造用於得到下載請求對象的iVar參數
                                    Class<?> classI = XposedHelpers.findClassIfExists("com.tencent.mm.modelcdntran.i", classLoader);
                                    Object iVar = XposedHelpers.newInstance(classI);
                                    //獲得field_mediaId變量值
                                    Class<?> classD = XposedHelpers.findClassIfExists("com.tencent.mm.modelcdntran.d", classLoader);
                                    long timestamp = System.currentTimeMillis();
                                    String mediaId = (String) XposedHelpers.callStaticMethod(classD, "a", "downimg", timestamp / 1000, talker, msgId);
                                    XposedHelpers.setObjectField(iVar, "field_mediaId", mediaId);
                                    //圖片保存路徑
                                    File saveDir = new File(Environment.getExternalStorageDirectory() + File.separator + "wechat");
                                    if (!saveDir.exists()) {
                                        saveDir.mkdir();
                                    }
                                    File saveFile = new File(saveDir.getAbsolutePath(), msgId + "_" + talker + ".jpg");
                                    String savePath = saveFile.getAbsolutePath();
                                    XposedHelpers.setObjectField(iVar, "field_fullpath", savePath);
                                    XposedHelpers.setIntField(iVar, "field_fileType", 1);
                                    XposedHelpers.setIntField(iVar, "field_totalLen", hdlength);
                                    XposedHelpers.setObjectField(iVar, "field_aesKey", aeskey);
                                    XposedHelpers.setObjectField(iVar, "field_fileId", cdnbigimgurl);
                                    XposedHelpers.setIntField(iVar, "field_priority", 2);
                                    XposedHelpers.setIntField(iVar, "field_chattype", 0);
                                    XposedHelpers.setBooleanField(iVar, "field_autostart", false);
                                    //獲得圖片下載請求對象
                                    Class<?> classB = XposedHelpers.findClassIfExists("com.tencent.mm.modelcdntran.b", classLoader);
                                    Object request = XposedHelpers.callStaticMethod(classB, "a", iVar);
                                    //進行圖片下載
                                    Class<?> cdn = XposedHelpers.findClassIfExists("com.tencent.mars.cdn.CdnLogic", classLoader);
                                    int re = (int) XposedHelpers.callStaticMethod(cdn, "startC2CDownload", request);
                                    Log.e(TAG, "保存聊天圖片返回值：" + re);//當爲0時表示成功
                                }
                            }
                        }
                    }
                });
```

*   到此我們接收到聊天圖片自動保存原圖到指定文件夾的功能就實現，同時我們逆向系列教材也暫時告一段落了，相信聰明的你如果這 5 篇 Xposed 逆向學習都一路跟着實現了一遍的話，Xposed 逆向應該也算基本入門了。

這也算是說在這個系列最後的話吧，記得我在開始寫逆向工具那篇的時候說過逆向的世界很容易迷失，很多逆向開發者可能因爲被生活所迫已經丟掉了最初那顆樂於分享和交流的心了，基於種種原因我就決定來寫下這一個系列，不僅讓自己從中學到東西，而且也希望可以幫助到初入 Xposed 逆向的你。最後，真心感謝那些素未謀面就樂於幫助我的那些小夥伴，希望我們能一直守着初心，歸來仍是少年！