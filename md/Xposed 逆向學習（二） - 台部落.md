> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.twblogs.net](https://www.twblogs.net/a/5c0ca060bd9eee5e40ba7a31)

> 嚴重聲明 本文的意圖只有一個就是通過分析 app 學習更多的 Xposed 逆向技術，如果有人利用本文知識和技術進行非法操作進行牟利，帶來的任何法律責任都將由操作者本人承擔，和本文作者無任何關係。

本文的意圖只有一個就是通過分析 app 學習更多的 Xposed 逆向技術，如果有人利用本文知識和技術進行非法操作進行牟利，帶來的任何法律責任都將由操作者本人承擔，和本文作者無任何關係。最終還是希望大家能夠秉着學習的心態閱讀此文，也不枉我寫此文的目的，希望大家都能不忘初心，歸來仍是少年！

*   本文的最終目的就是逆向查找到微信聊天時發送圖片的方法，會寫的相對簡單，裏面會留有很多值得進一步學習查找的提示，希望抱着學習心態來的你可以按照提示去進一步挖掘學習哦。
*   提示一：可以按照類似打招呼那篇走界面控件和點擊事件結合源碼的方法從選擇圖片點擊發送後一步一步跟着去查找。
*   本文通過的方式是一種倒推參數的方式哦，首先在打招呼的那篇博客裏如下圖我稍微提了一下微信很多功能都可能封裝在了同一個方法裏，參數不同實現的功能也就不同了。打招呼和發圖兩個頁面點開來看下會不會感覺特別的相似，既然這麼相似我就有理由猜測最終走的是同一個方法哦，也就是那個 a 方法了。其實即使頁面沒啥相似你也可以這樣去猜測，做出猜測之後你再去進行合理的的驗證，萬一驗證出來是對也就成功了。這方法怎麼感覺有點像上學時的假設檢驗方法，這應該算學以致用吧，這波聯繫我覺得也是沒誰了。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-25ede7a8f84c47f3.png)
*   好了我們來將 a 方法 hook 住。

```
Class<?> l = XposedHelpers.findClassIfExists("com.tencent.mm.ab.l", classLoader);
        XposedHelpers.findAndHookMethod("com.tencent.mm.ab.o", classLoader, "a", l, int.class, new XC_MethodHook() {
            @Override
            protected void afterHookedMethod(XC_MethodHook.MethodHookParam param) throws Throwable {
                Log.e(TAG, "!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!");
                Log.e(TAG, "調用信息：", new Exception());
                Object[] args = param.args;
                if (args == null) return;
                for (int i = 0; i < args.length; i++) {
                    Object arg = args[i];
                    if (arg != null) {
                        Log.e(TAG, "******************參數" + (i + 1) + ":" + arg.toString());
                        Field[] declaredFields = arg.getClass().getDeclaredFields();
                        if (declaredFields == null) return;
                        for (int j = 0; j < declaredFields.length; j++) {
                            Field declaredField = declaredFields[j];
                            String fieldName = declaredField.getName();
                            Object objectField = XposedHelpers.getObjectField(arg, fieldName);
                            if (objectField != null) {
                                Log.e(TAG, fieldName + ":" + objectField.toString());
                            }
                        }
                    }
                }
                Log.e(TAG, "返回值：" + param.getResult());
                super.afterHookedMethod(param);
            }
        });
```

*   在發圖之前將日誌清空下，因爲這個方法會被很多地方使用，我們只需要發圖時的那次調用，所以避免干擾先清下。發圖後打印的那條日誌如下。
    
    ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)
*   因爲 a 方法有可能就是我們要找的發圖方法，所以我們暫時就先別往前找繼續跟調用方法，我這裏一次性寫了打印這麼多信息的代碼主要是避免萬一有用，所以都打印出來好了。方法有了，我們暫時缺的就是參數了，而這個方法就兩個參數，第一個所屬類從上圖日誌中可以看到，第二個是 int 型，而且值通過多次發圖打印日誌可以知道爲固定值 0。所以下面我就來鉤下第一個參數所屬類的構造方法。

```
Class<?> lClass = XposedHelpers.findClassIfExists("com.tencent.mm.ak.l", classLoader);
        XposedBridge.hookAllConstructors(lClass, new XC_MethodHook() {
            @Override
            protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                super.beforeHookedMethod(param);
                Object[] args = param.args;
                if (args != null) {
                    Log.e("hook參數",
                            "參數長度：" + args.length);
                    for (int i = 0; i < args.length; i++) {
                        if (args[i]!=null)
                        Log.e("hook參數",
                                "參數" + i + ":" + args[i].toString());
                    }
                }
            }
        });
```

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

```
//原圖
        12-09 08:30:06.278 2235-2369/com.tencent.mm E/hook參數: 參數長度：10
        參數0:13
        參數1:wxid_...
        參數2:wxid_...
        參數3:/storage/emulated/0/launcher/ad/471ee36d02f2c69af15b9c70b095cee3.png
        參數4:1
        參數5:com.tencent.mm.ak.i@1a54d39
        參數6:0
        參數8:
        參數9:2130837999
     
        //原圖
        12-09 08:30:23.513 2235-2369/com.tencent.mm E/hook參數: 參數長度：10
        參數0:15
        參數1:wxid_...
        參數2:wxid_...
        參數3:/storage/emulated/0/tencent/MicroMsg/WeiXin/mmexport1544315379366.jpg
        參數4:1
        參數5:com.tencent.mm.ak.i@1a54d39
        參數6:0
        參數8:
        參數9:2130837999
           
        //非原圖     
        12-09 08:31:17.939 2235-2369/com.tencent.mm E/hook參數: 參數長度：10
        參數0:16
        參數1:wxid_...
        參數2:wxid_...
        參數3:/storage/emulated/0/tencent/MicroMsg/WeiXin/mmexport1544315010796.jpg
        參數4:0
        參數5:com.tencent.mm.ak.i@1a54d39
        參數6:0
        參數8:
        參數9:2130837999
          
        //原圖      
        12-09 08:31:38.621 2235-2369/com.tencent.mm E/hook參數: 參數長度：10
        參數0:18
        參數1:wxid_...
        參數2:wxid_...
        參數3:/storage/emulated/0/tencent/MicroMsg/WeiXin/mmexport1544315379366.jpg
        參數4:1
        參數5:com.tencent.mm.ak.i@1a54d39
        參數6:0
        參數8:
        參數9:2130837999
```

*   看到日誌是不是感覺成功了一半，除了我標註出來的那些有規律的可變參數，其它參數要麼爲固定值，要麼爲空。參數 5 通過日誌是看不出什麼的，那我們找到對應類反編譯源碼來看下。
    
    ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)
*   可以看到該類有無參的構造方法，那我們就可以很容易將它構造出來。這樣一來我們發圖的方法就找到了，參數也有了，我們就可以將發圖功能實現出來了。

```
/**
     * @param classLoader
     * @param imgIndex    對應上面日誌中l類構造方法參數0
     * @param sendWxId    參數1
     * @param receiveWxId 參數2
     * @param imgPath     參數3
     * @param isOriginal  參數4,原圖爲1，非原圖爲0
     */
    private void sendImg(ClassLoader classLoader, long imgIndex, String sendWxId, String receiveWxId, String imgPath, int isOriginal) {
        try {
            Log.e(TAG, "開始發送圖片，圖片下標：" + imgIndex);
            //通過公開靜態無參方法DF獲取我們發圖的a方法
            Class<?> au = XposedHelpers.findClassIfExists("com.tencent.mm.model.au", classLoader);
            Method DF = au.getMethod("DF");
            Object o = DF.invoke(null);
            //得到l參數的構造方法
            Class<?> l = XposedHelpers.findClassIfExists("com.tencent.mm.ak.l", classLoader);
            Class<?> f = XposedHelpers.findClassIfExists("com.tencent.mm.ab.f", classLoader);
            Constructor<?> constructor = l.getConstructor(long.class, String.class, String.class, String.class, int.class, f, int.class, String.class, String.class, int.class);
            Class<?> i = XposedHelpers.findClassIfExists("com.tencent.mm.ak.i", classLoader);
            Object lVar = constructor.newInstance(imgIndex, sendWxId, receiveWxId, imgPath, isOriginal, i.newInstance(), 0, null, "", 2130837999);
            Object bool = XposedHelpers.callMethod(o, "a", lVar, 0);
            Log.e(TAG, "發送圖片返回值：" + bool.toString());
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        }
    }
```

*   好了，到此你可能覺得已經完成我們的需求了，可是當你用上面方法去試下發圖時會發現是發不出去的。不知道你發現沒有上面 l 類的構造方法的參數 0 是不好確定的，雖然是自增的，但是是不可控的，你可以用上面代碼自動去發圖，發一張自增 1 或 2，也可以打開微信自己手動去發，發一張也自增 1 或 2，甚至可能該參數規則根本就不是這樣，我們僅僅是根據幾條測試的日誌來猜測的而已。其實參數 4 也是存在同樣問題的。具體微信對於這兩個可變參數的的規則我們通過不斷髮圖測試要測出來是很難得的，如果你測出來了，麻煩留言告訴我下哦。那既然我們猜不到我就來源碼中搜索下看微信自己在哪裏使用了這個構造方法，是怎麼使用的。搜索了下發現沒有使用的地方，但是再看該構造方法的時候發現了一個有意思的東西如下所示。
    
    ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)
*   其實 x 類就是微信自己打印日誌的類，可以看到它在日誌中將發送者 wxid 打印出來了，這就給了我們定位方法提供了一個思路，可以 hook 日誌類 x，將它日誌信息打印出來，在它日誌信息中搜索 wxid，再通過一些固定的字符串比如這裏的 FROM A UI : 去源碼中全局搜索定位到方法。
*   好了，我們繼續，雖然那個構造方法沒用到，但是可以看到 l 類還有很多其它構造方法，只有我們能夠將其構造出來，說不定還是可以將圖片發送出去的。我們就換一個構造方法好了，既然參數 0 遞增不可控，我就找個沒這個參數的構造方法好了，我找了具有 11 個參數的那個構造方法。去源碼中搜索下使用它的地方。
    
    ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)
*   它自己可以這樣調用，我們也可以仿照它的寫法去構造一個了，無法直接看出來的那幾個參數結合上面 10 個參數的那個方法和參照 l 類源碼看下我們可以很容易猜測出來，然後寫出發送圖片方法如下。

```
/**
     * @param classLoader
     * @param sendWxId    參數1
     * @param receiveWxId 參數2
     * @param imgPath     參數3
     */
    private void sendImg(ClassLoader classLoader, String sendWxId, String receiveWxId, String imgPath) {
        try {
            //通過公開靜態無參方法DF獲取我們發圖的a方法
            Class<?> au = XposedHelpers.findClassIfExists("com.tencent.mm.model.au", classLoader);
            Method DF = au.getMethod("DF");
            Object o = DF.invoke(null);
            //得到l參數的構造方法
            Class<?> l = XposedHelpers.findClassIfExists("com.tencent.mm.ak.l", classLoader);
            Class<?> f = XposedHelpers.findClassIfExists("com.tencent.mm.ab.f", classLoader);
            Constructor<?> constructor = l.getConstructor(int.class, String.class, String.class, String.class, int.class, f, int.class, String.class, String.class, boolean.class, int.class);
            Class<?> i = XposedHelpers.findClassIfExists("com.tencent.mm.ak.i", classLoader);
//10參數時構造方法：Object lVar = constructor.newInstance(imgIndex, sendWxId, receiveWxId, imgPath, isOriginal, i.newInstance(), 0, null, "", 2130837999);
            Object lVar = constructor.newInstance(4, sendWxId, receiveWxId, imgPath, 0, null, 0, "", "", true, 2130837999);
            Object bool = XposedHelpers.callMethod(o, "a", lVar, 0);
            Log.e(TAG, "發送圖片返回值：" + bool.toString());
        } catch (NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        }
    }
```

*   測試下，可以發送成功，好了，到此就算完了。當然實際情況可能你不會一上來就看到對的那個構造方法，但是隻要一個一個試過去就是會找到對的那個構造方法的，逆向就是這樣，一步一步採着坑到達終點。其實你要發圖的話需要要發送和接收方 wxid，這也留個提示吧，如果是自動回覆的話接收地方這兩個 id 肯定是可以得到的，如果不是的話可以通過鉤取微信數據庫對象將 id 查詢讀取出來，然後在進行發送。