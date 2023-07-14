> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.twblogs.net](https://www.twblogs.net/a/5c133865bd9eee5e4183dbd8)

> 嚴重聲明 本文的意圖只有一個就是通過分析 app 學習更多的 Xposed 逆向技術，如果有人利用本文知識和技術進行非法操作進行牟利，帶來的任何法律責任都將由操作者本人承擔，和本文作者無任何關係。

本文的意圖只有一個就是通過分析 app 學習更多的 Xposed 逆向技術，如果有人利用本文知識和技術進行非法操作進行牟利，帶來的任何法律責任都將由操作者本人承擔，和本文作者無任何關係。最終還是希望大家能夠秉着學習的心態閱讀此文，也不枉我寫此文的目的，希望大家都能不忘初心，歸來仍是少年！

*   本篇還是繼續我們的逆向學習系列，最終要實現的需求就是可以自由控制微信擲骰子停下來的點數。
*   首先我們要在擲骰子之前加個用於選擇點數的對話框，因爲每次擲骰子之前要點擊發送表情的那個圖標（如下圖），所以我們就加到這裏好了。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-a223b3e287851c7d.png)
*   要在點擊這裏之後彈出一個我們新加用於選擇點數的對話框，我們就得先找到這個圖標的點擊事件執行的方法。下面我用 ddms 工具來錄製下點擊圖標執行的方法軌跡，通過搜索 click 來找到點擊方法。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-91b005b1e4a56c96.png)
*   可以看到通過 ddms 工具來找點擊方法還是很方便的，一下就找到了。下面我們就來 hook 該方法，在該方法執行前彈出一個選點數對話框。這裏要注意創建一個對話框需要一個 Context，這個上下文對象不能是 Application 的上下文對象，只能是當前頁面的上下文對象。所以我看下點擊事件所在類源碼看下里面有沒有相應的 Context 可以供我們使用。要搜索找到點擊事件的類，需要用到我們 [Xposed 逆向學習（一）](https://www.jianshu.com/p/2d5f8e98d9f6)中關於內部類的知識點，這裏就不再重複了，不清楚的可以回去看，從下面源碼可以看出點擊事件所在類 ChatFooter_6 是 ChatFooter 的內部類。ChatFooter_6 裏是沒有看到 Context 對象的，但是它通過構造方法傳入了一個 ChatFooter 對象，並在類裏持有一個 ChatFooter 對象的變量 qMv。  
    ![](http://upload-images.jianshu.io/upload_images/4834678-e583686f378c2a78.png)
*   既然點擊事件類裏面沒有，但是它裏面有一個 ChatFooter 對象的變量，所以我們可以來看下 ChatFooter 類裏面有沒有 Context 對象，如果有的話，我們也是可以獲取得到一個上下文對象來供我們創建對話框使用的。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-092a46c4a983aa3b.png)
*   通過截圖可以看到 ChatFooter 類裏面是有我們所需要的上下文對象的，這樣一來我們就可以 hook 住點擊事件，並在點擊事件方法前彈出我們選點數的對話框了，這裏注意骰子只有六個面，但是我弄了 7 個選項，目的是爲了當你選擇了 0 的時候停下來的點數還是微信隨機產生的，除此之外我把對話框的除了通過選擇選項來取消之外的其他取消比如點擊對話框外讓對話框取消消失的途徑都關閉了，目的就是讓你必須按我們邏輯流程來選擇一項。

```
XposedHelpers.findAndHookMethod("com.tencent.mm.pluginsdk.ui.chat.ChatFooter$6", classLoader, "onClick", View.class, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                super.beforeHookedMethod(param);
                Object thisObject = param.thisObject;
                Object qMv = XposedHelpers.getObjectField(thisObject, "qMv");
                final Context currentContext = (Context) XposedHelpers.getObjectField(qMv, "context");
                //Toast.makeText(currentContext, "點擊了圖標", Toast.LENGTH_SHORT).show();
                new AlertDialog.Builder(currentContext)
                        .setTitle("想要擲幾點？")
                        .setSingleChoiceItems(new String[]{"0", "1", "2", "3", "4", "5", "6"}, 0,
                                new DialogInterface.OnClickListener() {
                                    @Override
                                    public void onClick(DialogInterface dialog, int which) {
                                        index = which + "";
                                        Toast.makeText(currentContext, index, Toast.LENGTH_SHORT).show();
                                        dialog.dismiss();
                                    }
                                })
                        //.setNegativeButton("取消", null)
                        .setCancelable(false)
                        .show();
            }
        });
```

![](http://upload-images.jianshu.io/upload_images/4834678-31487d6ffb805885.png)

*   好了，對話框加好了，接下來我們就來找我們的控制骰子點數的方法了。還是用我們 ddms 感覺來錄製下骰子投擲時執行的方法軌跡，從方法軌跡一步一步跟看能跟到我們需要的方法不。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-5588ef9372f12d02.png)  
    ![](http://upload-images.jianshu.io/upload_images/4834678-14a0ee223ff4d5ff.png)
*   通過方法軌跡和源碼我們可以看到 switch 走的是 25 所在分支。25 分支一開始執行軌跡裏的 self 是一個 if 判斷，按我們經驗來說這個判斷一般都是異常判斷，這裏是我們想要方法的可能性很小，所以我們繼續往下跟 a 方法。hook 參數來看下。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-b7435c7b184cae66.png)

```
//第一個參數是個從源碼可以看出是個GridView，所以這個參數意義對我們來說作用不大，不打印
        Class<?> smileyGrid = XposedHelpers.findClassIfExists("com.tencent.mm.view.SmileyGrid", classLoader);
        //第二個參數，裏面很多變量，而且從源碼（如上圖）可以看出它還有個toString方法，
        //所以我們就不通過反射拿到變量名去打印變量了，直接用toString了
        Class<?> emojiInfo = XposedHelpers.findClassIfExists("com.tencent.mm.storage.emotion.EmojiInfo", classLoader);
        XposedHelpers.findAndHookMethod("com.tencent.mm.view.SmileyGrid", classLoader, "a", smileyGrid, emojiInfo, new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                super.afterHookedMethod(param);
                //因爲我們錄製了方法軌跡，從方法軌跡中看到的方法再去打印調用信息意義就不大了。
                //基本調用信息都能從方法軌跡裏看到。
                //Log.e(TAG, "SmileyGrid調用信息：", new Exception());
                if (param.args[1] != null)
                    Log.e(TAG, "emojiInfo：" + param.args[1].toString());
            }
        });
```

![](http://upload-images.jianshu.io/upload_images/4834678-90da09dcb105b713.png)

*   看下日誌好像並沒有啥有用的信息，那就先不管了。繼續來看下方法軌跡，通過點擊 a 方法進入到下一個執行的方法也就是 a 方法了，並打開 a 方法所在源碼來看下。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-5f2016a1c00062ab.png)  
    ![](http://upload-images.jianshu.io/upload_images/4834678-290de68287ee06bd.png)
*   通過軌跡下一步執行的 l 方法可以定位到執行分支。通過分析執行分支源碼可以得出第一個打斷點處就是 self 執行的地方，第二處就是調用下一個方法 l 方法執行的地方。

### 方法一

*   那麼我們就先拿第一個斷點作爲我們下一步目標通過看源碼跟蹤斷點所在位置執行軌跡。

```
EmojiInfo c = ((c) g.n(c.class)).getProvider().c(emojiInfo);
```

*   通過這行源碼可以看出我們最後就是要通過 c 方法得到一個 EmojiInfo 對象的返回值。按住 Ctrl，點擊 c 方法，跳到 c 方法。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-118c2f5887c2c5ba.png)
*   通過源碼可以看到 c 方法所在類是個接口，搜索下該接口實現類。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-2637601c6bde89f2.png)
*   由最開始斷點處源碼分析知道我們最終目的是要拿到一個 EmojiInfo 對象返回值，從上面源碼可以看到 c 方法返回的是 getEmojiMgr() 方法返回的對象的 c 方法的返回值，所以我們繼續點擊 getEmojiMgr 跳過去看下。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-6dc399caf1f7eeae.png)  
    ![](http://upload-images.jianshu.io/upload_images/4834678-6204377b7d2903c1.png)
*   可以看到 d 也是個接口同時還繼承了 e 接口，所以最終 e 接口的實現類可以順理成章的去調用 e 接口子類接口實現類的 c 方法去得到相同的 EmojiInfo 對象返回值。我們繼續來搜索下 d 接口的實現類。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-35d11d871cb07907.png)
*   可以看到就一個實現類，找到實現類裏 c 方法看下，我們的目的是 emojiInfo，所以我們找與之相關的執行地方去看，從第一處我們打斷點處開始，點擊 eF 方法跳轉過去看下。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-9bbbdcb490653430.png)
*   返回值是個 int，而且通過 Random 產生了一個隨機數，看到這裏你肯定是會有點想法的，很可能這個方法就是我們要找的那個方法。hook 一下返回值，驗證下我們的猜測。

```
XposedHelpers.findAndHookMethod("com.tencent.mm.sdk.platformtools.bi", classLoader, "eF", int.class, int.class, new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                super.afterHookedMethod(param);
                Log.e(TAG, "返回值" + param.getResult());
            }
        });
```

*   擲幾次骰子來對比下返回值和最終停下來的點數看下有沒有對應關係。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-740fdec2e9b7c328.png)  
    ![](http://upload-images.jianshu.io/upload_images/4834678-d906c48d0030751d.png)
*   看到這裏結果應該很明顯了吧。最終的點數就是這個方法返回值加一。好了，到此決定點數的方法就找到了，但是你會不會感覺這樣找起來其實還是挺困難的，得有一定的分析源碼的和猜測敏銳度的能力。所以我們下面就繼續來講一個簡單的定位到目標方法的思路。

### 方法二

*   這個思路對應的就是上面截圖（SmileyGrid 類的 a 方法）的第二個斷點處，對應下面截圖的 l（從方法軌跡可以看到該方法完整類名路徑）方法，它的參數也是 EmojiInfo 對象。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-816a003cc7574ac5.png)
*   hook 下 EmojiInfo 類型的參數。

```
Class<?> emojiInfo = XposedHelpers.findClassIfExists("com.tencent.mm.storage.emotion.EmojiInfo", classLoader);
        XposedHelpers.findAndHookMethod("com.tencent.mm.ui.chatting.w", classLoader, "l", emojiInfo, new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                super.afterHookedMethod(param);
                if (param.args[0] != null)
                    Log.e(TAG, "emojiInfo：" + param.args[0].toString());
            }
        });
```

![](http://upload-images.jianshu.io/upload_images/4834678-f72880ce77217f1f.png)  
![](http://upload-images.jianshu.io/upload_images/4834678-b50f14bc8fa3ba31.png)  
![](http://upload-images.jianshu.io/upload_images/4834678-7ecb48ba8baee4a0.png)

*   這樣一對比結果應該就更明顯了吧，dice_2.png 對應停下來的點數是 2，dice_1.png 對應點數 1。既然結果與 EmojiInfo 類的參數有對應關係，那麼我們就來 hook 下 EmojiInfo 類的構造方法，來根據它的調用信息來追蹤到目標方法。

```
XposedHelpers.findAndHookConstructor("com.tencent.mm.storage.emotion.EmojiInfo", classLoader, new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                super.afterHookedMethod(param);
                Log.e(TAG, "EmojiInfo構造方法調用信息：", new Exception());
            }
        });
```

![](http://upload-images.jianshu.io/upload_images/4834678-79404637b95d9f93.png)

*   好了直接就可以找到上面方法一對應的倒數第二步的 com.tencent.mm.plugin.emoji.e.g.c 方法了，接下來思路和方法一就完全一樣了，這裏就略去了。
*   從兩個方法可以看出 EmojiInfo 對象一直貫穿着我們跟蹤的整條線，所以這裏給了我們一個思路，當我們發現某個對象有用時就可以一直圍繞着它去找我們的目標方法。
*   這樣一來我們就可以結合起最開始的對話框選擇一個點數傳入到下面方法，將 eF 的返回值修改爲我們選擇的點數，這樣我們就可以隨意控制骰子停下來的點數了。

```
/**
     * @param classLoader
     * @param number      想要骰子停下來的點數
     */
    private void modifyDotNumber(ClassLoader classLoader, final int number) {
        XposedHelpers.findAndHookMethod("com.tencent.mm.sdk.platformtools.bi", classLoader, "eF", int.class, int.class, new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                super.afterHookedMethod(param);
                if (number > 0 && number < 7) { //進行下判斷，傳入的參數必須有骰子對應面的點數時我們才修改
                    param.setResult(number - 1);//這裏要注意下對應關係，返回值對應骰子點數減一
                }
            }
        });
    }
```

*   實際上你這樣寫一測試會發現有問題，由於 number 是 final 的緣故會導致修改的點數一直都是第一次你選擇的那個點數。所以我們修改下，把選擇要修改的點數抽取成一個靜態全局變量放外邊去。

```
//選擇的想要投擲的點數
    private static int dot;
    private void hook(final ClassLoader classLoader) {
        XposedHelpers.findAndHookMethod("com.tencent.mm.pluginsdk.ui.chat.ChatFooter$6", classLoader, "onClick", View.class, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                super.beforeHookedMethod(param);
                Object thisObject = param.thisObject;
                Object qMv = XposedHelpers.getObjectField(thisObject, "qMv");
                final Context currentContext = (Context) XposedHelpers.getObjectField(qMv, "context");
                new AlertDialog.Builder(currentContext)
                        .setTitle("想要擲幾點？")
                        .setSingleChoiceItems(new String[]{"0", "1", "2", "3", "4", "5", "6"}, 0,
                                new DialogInterface.OnClickListener() {
                                    @Override
                                    public void onClick(DialogInterface dialog, int which) {
                                        dot = which;
                                        dialog.dismiss();
                                    }
                                })
                        .setCancelable(false)
                        .show();
            }
        });
        XposedHelpers.findAndHookMethod("com.tencent.mm.sdk.platformtools.bi", classLoader, "eF", int.class, int.class, new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                super.afterHookedMethod(param);
                if (dot > 0 && dot < 7) { //進行下判斷，傳入的參數必須有骰子對應面的點數時我們才修改
                    param.setResult(dot - 1);//這裏要注意下對應關係，返回值對應骰子點數減一
                }
            }
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                super.beforeHookedMethod(param);
                if (dot > 0 && dot < 7) { //進行下判斷，傳入的參數必須有骰子對應面的點數時我們才修改
                    param.setResult(dot - 1);//這裏要注意下對應關係，返回值對應骰子點數減一
                }
            }
        });
    }
```

*   這樣修改後就沒問題了，每次骰子停下來的點數就是我們選擇的點數了，這樣就可以和朋友去愉快的玩耍下了。最後附個讓停下來點數從 1-6 的成果圖。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-4f5e8a3d119e3b21.png)