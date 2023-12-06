> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.twblogs.net](https://www.twblogs.net/a/5c08abe8bd9eee6fb212be54)

> 严重声明本文的意图只有一个就是通过分析app 学习更多的Xposed 逆向技术，如果有人利用本文知识和技术进行非法操作进行牟利，带来的任何法律责任都将由操作者本人承担，和本文作者无任何关系。

本文的意图只有一个就是通过分析app 学习更多的Xposed 逆向技术，如果有人利用本文知识和技术进行非法操作进行牟利，带来的任何法律责任都将由操作者本人承担，和本文作者无任何关系。最终还是希望大家能够秉着学习的心态阅读此文，也不枉我写此文的目的，希望大家都能不忘初心，归来仍是少年！

*   真正开始学Xposed 的时候，其实并没有想象中那么复杂，原理和相关的API 都很简单。但难的是逆向过程，怎么去Hook 到你要的方法功能（堆栈跟踪、方法轨迹跟踪、源码跟踪等等），在这个过程中你需要去分析很多很多东西，通过猜测调试打印日志去定位。有时候折腾几天甚至几周可能毫无进展，这时可能整个人都不好了，这就是我这段时间的真实写照，这也是我决定写这一系列文章的初心，希望可以帮到和我一样初入android 逆向的朋友，可以给你提供一点思路。
*   这里提下现在很多网上android 逆向的博客和帖子的现状，其一量不是很多；其二有的已经过时了；其三有的过程不是很全，没法参考了。这对很多刚接触这方面技术的人来说是很不友好的，完全没法跟着步骤一步一步来学习。
*   造成上述现状的原因我这里也说下我的看法。其一可能有些人不愿将这种自己辛苦研究出来的成果分享出来，这可以理解，很正常。其二可能怕这种有点带黑科技性质的东西分享出来后被有人用于非法途径，这点我觉得有的多虑了，我们开发者都是高素质的，都是抱着学习态度来看的，至于其它一部分不懂的人我不发布打包的apk 他也不会用，他有有这闲工夫看懂，我感觉他觉悟也挺高的，应该可以顿悟了，顿悟不了的就看他造化了。最后还有可能会怕自己将逆向的过程分享出来后，可能相应正向开发者会针对你这个逆向过程做防护，这点完全是想多了，无论是防护还是破解都是在不断进步改进的，你的一篇逆向技术教程就能让对应厂商针对上也是没谁了。

我的Xposed 逆向学习这一系列的教程都是基于微信的，使用的版本是微信6.6.7，所以我希望你能配置好相应的开发环境，至于环境怎么搭建我这里就不说了，我之前的有的博客是关于搭逆向环境的，你可以去翻出来照着搭起来，搭起来之后能跟着博文一步一步跟下去，同时还可以自己多一些尝试，这样学习起来效率可能会更高，而不仅仅是看一遍。

*   我们这篇的目的就是通过Xposed 逆向查找到微信附近的人打招呼功能。最终我们可以编写一个当打开附近的人页面时自动向查找到的前20 个人打招呼模块。

1.  首先要查找打招呼功能一开始很自然就会想到通过打招呼的UI 界面的发送按钮来定位到其点击事件执行的地方从而定位到功能。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-8b65a86d2af8cceb.png)

*   于是来编写下Xposed 代码将View 的setOnClickListener 方法拦截下来得到其点击的Id 和点击事件所在类。这里要注意下你必须到页面中点了相应的View，才会触发setOnClickListener 方法，我们才能进行hook。

```
import android.app.Application;
import android.content.Context;
import android.util.Log;
import android.view.View;

import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.XC_MethodHook;
import de.robv.android.xposed.XposedHelpers;
import de.robv.android.xposed.callbacks.XC_LoadPackage;

public class Module implements IXposedHookLoadPackage {
    //打印日誌的tag
    public static final String TAG = "***Xposed***";
    //微信包名，用於過濾，只對微信方法的執行過程進行鉤取
    private String weChartPackageName = "com.tencent.mm";
    //鉤取的微信Application的上下文對象，便於後面使用
    private static Context context;

    @Override
    public void handleLoadPackage(final XC_LoadPackage.LoadPackageParam lpparam) throws Throwable {
        if (lpparam.packageName.equals(weChartPackageName)) {
            //鉤取微信Application上下文對象
            XposedHelpers.findAndHookMethod(Application.class, "attach", Context.class, new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    super.afterHookedMethod(param);
                    context = (Context) param.args[0];
                }
            });
            //鉤取設置點擊事件方法得到點擊事件的類名
            XposedHelpers.findAndHookMethod(View.class, "setOnClickListener", View.OnClickListener.class, new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    //設置點擊事件的View對象
                    final View view = (View) param.thisObject;
                    //setOnClickListener方法的參數也就是點擊事件的監聽器，也是我們需要獲取的對象了
                    final View.OnClickListener listener = (View.OnClickListener) param.args[0];
                    //創建個自己的監聽器將默認傳入監聽器參數替換掉，從而在裏面做些我們自己的事件
                    View.OnClickListener newListener = new View.OnClickListener() {
                        @Override
                        public void onClick(View v) {
                            if (listener == null) { //當默認傳入的監聽器爲空時不處理
                                return;
                            } else {
                                try {
                                    listener.onClick(v); //我們自己監聽器裏面依然還是要調用傳入監聽器的點擊方法，不然所有點擊都會失效了
                                    String resourceName = context.getResources().getResourceName(view.getId());
                                    Log.e(TAG, resourceName + "&&&" + view.getId());//打印View的資源名和Id
                                    Log.e(TAG, listener.toString());//點擊監聽器類
                                } catch (Exception e) {
                                    e.printStackTrace();
                                }
                            }
                        }
                    };
                    param.args[0] = newListener;
                }
            });
        }
    }
}
```

*   將 Xposed 模塊運行重啓，打開微信的附近的人隨便選個人點打招呼到發送打招呼消息頁面（也就是上面截圖頁面），隨便寫點東西。這時先把 AndroidStudio 的日誌清理下，因爲所有 View 的點擊事件我們都攔截了，之前我們的點擊也會打印日誌，爲了避免干擾先清除下。之後點擊下 “發送”，可以看到我們需要的信息就打印出來了。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-91cc2d79bab030da.png)
*   打開 ddms 工具來覈對下資源 id 是否能對應上。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-b1c4cbd689f636e5.png)
*   可以看到 com.tencent.mm:id/hg 就是發送的 Id，在我們日誌中打印的資源 Id 也是可以對應上的，說明沒錯我們需要的點擊類就是 com.tencent.mm.ui.s$12 了，通過在微信源碼中搜索類來找到對應的類。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-de2151c7aff48a0a.png)
*   可以看到沒有完全對應的類名，先不管點第一個進到 s 類，在類裏搜索 12。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-ecad0e2fda311f16.png)
*   美元符號 $ 用於表示內部類（由於混淆或反編譯的緣故），如上面 s$12 表示的是 s 類的內部類 12。還有下圖左紅框裏面的帶 $ 數字的類其實都是屬於同一個類的，反編譯出來後被分成多個類了，所以有時我們通過日誌打印定位類或方法去源碼裏對應類搜索不到時可能需要到其內部類裏也去搜索下，一般都是能夠查找到的。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-9431b3da78a88bc1.png)
*   還有上面圖片源碼紅框中的 J 方法一開始要我們自己來表示可能會寫成 com.tencent.mm.plugin.gallery.ui.AlbumPreviewUI$AlbumPreviewUI_3 或 com.tencent.mm.plugin.gallery.ui.AlbumPreviewUI.AlbumPreviewUI_3。其實正確的表示是 com.tencent.mm.plugin.gallery.ui.AlbumPreviewUI$3.J。這裏的內部類 AlbumPreviewUI_3 表示的其實就是 AlbumPreviewUI 的內部類 3（通過類名_3 表示），所以我們需要將其轉成用 $ 表示內部類的方式。在實際遇到類似 hook 內部類方法提示 de.robv.android.xposed.XposedHelpers$ClassNotFoundError: java.lang.ClassNotFoundException 找不到類時，我們可以用 $ 或. 和刪除相同重複類名方法靈活修改類名來嘗試直達鉤取到我們目標類爲止。
*   好了，這裏內部類知識就到這裏了，接下來我們接着上面定位到的點擊監聽內部類 12 往下走。onClick 裏面的 a 方法就是我們下一步目標了，按住 Ctrl，鼠標點擊跳轉到 a 方法去看看。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-9bdcba61d8926c1d.png)
*   寫點 Xposed 代碼打印參數看下, 這裏 a 參數所在 a 類（如下圖）是有很多成員變量的，所以我們通過反射獲取改類的成員變量並將其值也打印出來看下：
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-9f3d1d5c9433c162.png)

```
Class<?> a = XposedHelpers.findClassIfExists("com.tencent.mm.ui.s.a", lpparam.classLoader);
            XposedHelpers.findAndHookMethod("com.tencent.mm.ui.s", lpparam.classLoader, "a", MenuItem.class, a, new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    super.afterHookedMethod(param);
                    Object[] args = param.args;
                    for (int i = 0; i < args.length; i++) {
                        Object arg = args[i];
                        if (arg == null) return;
                        Log.e(TAG, arg.toString() + "---" + i + "---");
                        Field[] declaredFields = arg.getClass().getDeclaredFields();
                        for (int j = 0; j < declaredFields.length; j++) {
                            String declaredFieldName = declaredFields[j].getName();
                            Object objectField = XposedHelpers.getObjectField(arg, declaredFieldName);
                            if (objectField == null) return;
                            Log.e(TAG, declaredFieldName + ":" + objectField.toString());
                        }
                    }
                }
            });
```

![](http://upload-images.jianshu.io/upload_images/4834678-078fba41e8f250b8.png)

*   通過日誌可以看到，除了看到發送兩個字對應了 UI 頁面外，其它好像都沒用了，而且 a 參數爲空都沒打印出來，很不可思議，所以這個思路在我這裏暫時是走不通了，我就先放棄換一種思路了，也有可能是我哪裏弄錯了，你可以接着這個思路去繼續鉤一些方法或參數，有可能是可能得到答案的哦（打斷點處就可能是一個突破口哦）。
*   接下來我還是有點不死心，暫時也沒其它思路，就鉤了下點擊事件的調用，後來打印日誌發現同樣無用，其實後來想想完全沒必要去鉤取，因爲只有在點擊方法裏面執行的纔可能是發送打招呼，之前調用執行點擊方法的方法是不可能是目標的。

```
//鉤取點擊事件的調用信息
            XposedHelpers.findAndHookMethod("com.tencent.mm.ui.s$12", lpparam.classLoader, "onClick", View.class, new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    super.afterHookedMethod(param);
                    Log.e(TAG, "com.tencent.mm.ui.s$12.onClick", new Exception());
                }
            });
```

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

*   全是調用 Android 系統的方法，所以沒用。

2.  接下來我就想到我們發送打招呼消息的內容編輯完成之後，發送之前肯定要獲取下我們輸入的內容的，我就去通過上面用過的 ddms 工具得到輸入框 id 結合 adb shell dumpsys activity top 命令來查看下頁面信息。
    
    ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

*   還是挺幸運的找到了一個 MMEditText，接下來就去頁面對應的源碼（具體類名如上圖最頂上紅框）裏面找這個控件。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-ed37ea83d51ed0ab.png)
*   a 方法有點可疑，hook 下。

```
Class<?> sayHiEditUI = XposedHelpers.findClassIfExists("com.tencent.mm.ui.contact.SayHiEditUI", lpparam.classLoader);
            XposedHelpers.findAndHookMethod("com.tencent.mm.ui.contact.SayHiEditUI", lpparam.classLoader, "a", sayHiEditUI, new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    Log.e(TAG, param.getResult().toString(), new Exception());
                    //at com.tencent.mm.ui.contact.SayHiEditUI$1.onMenuItemClick(SourceFile:93)
                    //at com.tencent.mm.ui.s.a(SourceFile:1147)
                    //at com.tencent.mm.ui.s.a(SourceFile:82)
                    //at com.tencent.mm.ui.s$12.onClick(SourceFile:976)
                    super.afterHookedMethod(param);
                }
            });
```

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

*   可以看到當我們在打招呼頁面輸入框中輸入的內容，在點擊發送之後就會打印在日誌中，也就是 a 方法的返回值。到此還是挺開心的，說明思路對了，可以繼續搞了。除此之外我還打印了調用信息，這也是我們接下來要定位的幾個方法。
*   我這裏先通過採坑方式從後往前進行方法定位，其實一般從前往後纔是正確姿勢，因爲離調用者越近是我們要找目標的概率就越大。從後往前最後一個方法就是 at com.tencent.mm.ui.s$12.onClick(SourceFile:976)，是不是感覺很熟悉，其實就是上面點擊事件了，世界就是這麼小，兜兜轉轉又回到原點了，這個方法我們已經放棄了，先不管了，接下來看 at com.tencent.mm.ui.s.a(SourceFile:82)，還有 at com.tencent.mm.ui.s.a(SourceFile:1147)，s 類裏面 a 方法很多要根據日誌最後的數字去大概定位，不確定的還需要通過 Xposed 鉤取方法去確認下是否有用，這裏位置太多我就大概寫下了，最後發現有些要麼和上面點擊事件已經 hook 過了，沒用；要麼就是基本用處不大，可能有些會起到輔助定位的功能，所以建議學習的時候還是可以去嘗試各種 hook。
*   接下來只剩 at com.tencent.mm.ui.contact.SayHiEditUI$1.onMenuItemClick(SourceFile:93) 了（這個方法一看到感覺可能性還是挺大的，對應着頂部 Menu 的點擊事件），先不急着鉤了，先來看看源碼。
    
    ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)
*   可以看到最終定位到我箭頭所指的兩個地方，這裏還是來踩下坑，也許你找的時候運氣好，一上來就去找對的了，或者直接沒看到坑所在位置，因爲我遇到了我就寫下。我先猜的是下面那個 SayHiEditUI.a() 方法是目標，所以就去寫下面代碼去 hook 了。

```
//這裏是爲了鉤取SayHiEditUI$1類onMenuItemClick方法裏另一個方法用的，最後發現不是該方法，該方法沒用
            XposedHelpers.findAndHookMethod("com.tencent.mm.ui.MMActivity", lpparam.classLoader, "a", Class.class, Intent.class, new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    Log.e(TAG, param.args[0].toString(), new Exception());
                    //沒有日誌輸出，沒執行，排除，鉤取改方法的原因
                    //是因爲在反編譯源碼中按住Ctrl點擊回去的方法
                    //是SayHiEditUI的父類方法MMActivity的a方法
                    super.beforeHookedMethod(param);
                }
            });
            Class<?> l = XposedHelpers.findClassIfExists("com.tencent.mm.ab.l", lpparam.classLoader);
            XposedHelpers.findAndHookMethod("com.tencent.mm.ui.contact.SayHiEditUI", lpparam.classLoader, "a", int.class, int.class, String.class, l, new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    Log.e(TAG, "&&&&&&&&", new Exception());
                    //at com.tencent.mm.ab.o$6.run(SourceFile:499)
                    //at com.tencent.mm.sdk.platformtools.am.run(SourceFile:127)//這個方法hook的話會發現沒用，基本執行任何微信的方法都會調用
                    super.beforeHookedMethod(param);
                }
            });
```

![](http://upload-images.jianshu.io/upload_images/4834678-ae0cff07f160c4d1.png)

*   這裏參數我就沒打印了，你可以去打印出來看下，最後可以發現不是該方法，該方法沒用。
*   現在還剩最後一個方法了，具體 hook 代碼如下：

```
Class<?> l = XposedHelpers.findClassIfExists("com.tencent.mm.ab.l", lpparam.classLoader);
            XposedHelpers.findAndHookMethod("com.tencent.mm.ab.o", lpparam.classLoader, "a", l, int.class, new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    Log.e(TAG, param.getResult().toString() + "&&&&&&&&&&&&&&&&&&" + param.args[1].toString());
                    Log.e(TAG, param.getResult().toString(), new Exception());
                    super.afterHookedMethod(param);
                }
            });
```

*   微信打開之後日誌就一直打印，根本就還沒到打招呼頁面，這時的我是奔潰的，所以暫時先到這裏了，我又回去看了下反編譯源碼。鬼使神差的根據源碼鉤了下 SayHiEditUI$1 的構造方法（對應的這麼完美怎麼可能不是呢）。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-fff568b3ded30775.png)

```
Class<?> sayHiEditUI = XposedHelpers.findClassIfExists("com.tencent.mm.ui.contact.SayHiEditUI", lpparam.classLoader);
            XposedHelpers.findAndHookConstructor("com.tencent.mm.ui.contact.SayHiEditUI$1", lpparam.classLoader, sayHiEditUI, String.class, int.class, String.class, new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    int i = 0;
                    for (Object arg : param.args) {
                        ++i;
                        Log.e(TAG, arg.toString() + "******" + i);
                        if (i == 4) {
                            String arg4 = (String) arg;
                            Log.e(TAG, "***length***" + arg4.length());
                        }
                    }
                    super.beforeHookedMethod(param);
                }
            });
```

*   隨便選個人跳轉到到招呼頁面，日誌就出現了，退出再進，多打印幾個人的日誌看下。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-36eb0d6aae7cd146.png)
*   可以看到參數除了第二個都是固定的，可能第二個就是打招呼對象的標識吧，於是乎我就自己按照打印的參數 hook 打招呼頁面的 onCreate 方法來調用我上面所說的特別穩合的那個方法，看下當我們跳到打招呼頁面時會不會彈出和我們手動點擊發送時的已發送 Toast 並自動返回上一個頁面。

```
XposedHelpers.findAndHookMethod("com.tencent.mm.ui.contact.SayHiEditUI", lpparam.classLoader, "initView", new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    super.afterHookedMethod(param);
                    Log.e(TAG, "----------------------------------開始打招呼------------------------------");
                    Class<?> au = XposedHelpers.findClassIfExists("com.tencent.mm.model.au", lpparam.classLoader);
                    Method DF = au.getMethod("DF");
                    Object o = DF.invoke(null);
                    Class<?> m = XposedHelpers.findClassIfExists("com.tencent.mm.pluginsdk.model.m", lpparam.classLoader);
                    Constructor<?> constructor = m.getConstructor(int.class, List.class, List.class, List.class, String.class, String.class, Map.class, String.class, String.class);
                    List linkedList = new LinkedList();
                    linkedList.add("v1_6833929db1c9a6fb9ead1627c45a0d3f46b397579acbb91b243502100706e4b00a3d5f2c5dda323a63145a6007be80f0@stranger");
                    List linkedList2 = new LinkedList();
                    linkedList2.add(18);
                    List linkedList3 = new LinkedList();
                    linkedList3.add("");
                    Object mVar = constructor.newInstance(2, linkedList, linkedList2, linkedList3, "hi", "", null, null, "");
                    Object bool = XposedHelpers.callMethod(o, "a", mVar, 0);
                    Log.e(TAG, "" + bool.toString());
                }
            });
```

![](http://upload-images.jianshu.io/upload_images/4834678-7fb097565dacab5d.png)

*   當我打開附近的人隨便選擇一個人，打開打向他發送打招呼消息的頁面時，頁面全自動彈出已發送吐司，並自動返回上一個頁面，幸福真的是來的太突然了，成功了。可以看到日誌還打印了打招呼成功的返回值 true，說明打招呼消息發送成功。至於上面我們自己去鉤取對應方法的時候爲什麼會打開微信日誌還沒到打招呼頁面就開始調用該方法了，我這裏猜測是微信封裝的緣故，很多功能都封裝在了這個方法上，參數不同實現的功能也就不同。
*   到此你以爲完成了嗎，其實還沒有。第二個標誌打招呼對象的字符串參數還沒得到呢，光通過日誌打印出來的參數寫死肯定是不行的啊，這樣還是隻能自動給一個人打招呼。要是想換個人打招呼的話得重新選個人運行代碼打印標誌他身份的字符串，替換掉打招呼模塊第二個參數，再次運行 Xposed 模塊，打開微信手動點擊一步一步跳轉到發送打招呼頁面，這樣也太麻煩了吧，有這功夫我還不如手動去點呢。
*   看到上面你可能會很生氣，搞了半天你是來搞笑的吧。哈哈，下面我們就來解決這個問題，首先我們要取很多人的信息，不是一個人信息，肯定就不能在指定一個人的頁面去獲取這個數據了。有附近很多人的信息肯定就想到剛打開附近的人的那個通過列表列出附近的人的信息的那個頁面。所以我們找到那個頁面的源碼先來看下。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-9c8e69c1acff70cc.png)
*   找到展示的 ListView 和 Adapter，打印日誌出來看下，有沒有有用的信息。

```
//打印頁面用於展示附近的人信息的列表和適配器
            XposedHelpers.findAndHookMethod("com.tencent.mm.plugin.nearby.ui.NearbyFriendsUI", lpparam.classLoader, "initView", new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    super.afterHookedMethod(param);
                    Object eMX = XposedHelpers.getObjectField(param.thisObject, "eMX");
                    Log.e(TAG, eMX.toString());
                    Object lBP = XposedHelpers.getObjectField(param.thisObject, "lBP");
                    Log.e(TAG, lBP.toString());
                    //E: com.tencent.mm.plugin.nearby.ui.NearbyFriendsUI$b@6acafd5
                }
            });
```

![](http://upload-images.jianshu.io/upload_images/4834678-29bf8414d85068e7.png)

*   適配器頁面找到，打開源碼看下。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-01ef3c05cdb6ae4a.png)
*   getItem() 一般就是返回列表數據的方法，可以看到列表的數據還是從 NearbyFriendsUI 拿到的，可是由於反編譯緣故很不幸 r 方法找不到了，暫時沒法定位具體數據。但是沒關係，有返回值對象我們直接去 NearbyFriendsUI 源碼搜索 aqp。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-622569cea4211a0f.png)
*   一上來就搜索到兩個，還是兩個集合，很有可能就是我們要找的目標哦，找到對應初始化方法，從源碼可以看到這兩個集合從頭至尾都沒離開過 a 方法。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-408e17505a7d1572.png)
*   hook 下 a 方法，將集合裏對象變量打印出來看下。

```
Class<?> l = XposedHelpers.findClassIfExists("com.tencent.mm.ab.l", lpparam.classLoader);
            XposedHelpers.findAndHookMethod("com.tencent.mm.plugin.nearby.ui.NearbyFriendsUI", lpparam.classLoader, "a", int.class, int.class, String.class, l, new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    super.afterHookedMethod(param);
                    Object thisObject = param.thisObject;
                    List iKk = (List) XposedHelpers.getObjectField(thisObject, "iKk");
                    for (int i = 0; i < iKk.size(); i++) {
                        Object o = iKk.get(i);
                        Field[] declaredFields = o.getClass().getDeclaredFields();
                        for (int j = 0; j < declaredFields.length; j++) {
                            Field declaredField = declaredFields[j];
                            declaredField.setAccessible(true);
                            Object o1 = declaredField.get(o);
                            if (o1 != null)
                                Log.e(TAG, "***iKk***" + declaredField.getName() + ":" + o1.toString());
                        }
                        Log.e(TAG, "----------------------------------------------------------------");
                    }
                    List kJo = (List) XposedHelpers.getObjectField(thisObject, "kJo");
                    if (kJo != null)
                        Log.e(TAG, "kJo數量：" + kJo.size());
                    for (int i = 0; i < kJo.size(); i++) {
                        Object o = kJo.get(i);
                        Field[] declaredFields = o.getClass().getDeclaredFields();
                        for (int j = 0; j < declaredFields.length; j++) {
                            Field declaredField = declaredFields[j];
                            declaredField.setAccessible(true);
                            Object o1 = declaredField.get(o);
                            if (o1 != null)
                                Log.e(TAG, "***kJo***" + declaredField.getName() + ":" + o1.toString());
                        }
                        Log.e(TAG, "----------------------------------------------------------------");
                    }
                }
            });
```

![](http://upload-images.jianshu.io/upload_images/4834678-f0a1968937772e7f.png)  
![](http://upload-images.jianshu.io/upload_images/4834678-64baba3d41372294.png)  
![](http://upload-images.jianshu.io/upload_images/4834678-a0bf6dcfbf2f1689.png)

*   從日誌很明顯可以看到數據都在 iKk 那個集合裏，kJo 裏面是沒有數據的，我們要的標誌打招呼目標的那個字符串就對應 aqp 對象裏的 hbL 字段。好了到此爲止附近的人數據有了，發送打招呼內容方法有了，你就可以在得到數據的 a 方法中將這兩部分合起來實現一次性給很多人打招呼，我這裏就不提供合起來的代碼了，你要認真學過來的話組合代碼是很容易就可以寫出來的，建議給一個人打完招呼後加個延時，模仿下手動操作，防止檢測到異常被封號哦。
*   最後再來延伸下，你可以加個定位功能，修改位置，這樣你就可以想給哪附近的人打招呼就給哪附近的人打招呼了。最後再次聲明下，我們實現這個功能僅僅就是爲了實現下自動化，省點手動操作時間，不要有其他想法哦，不過我最終源碼都沒放出來，你要不懂拿去用處好像也不大，哈哈，我是故意的。