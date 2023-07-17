> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.twblogs.net](https://www.twblogs.net/a/5c133865bd9eee5e4183dbd8)

> 严重声明本文的意图只有一个就是通过分析app 学习更多的Xposed 逆向技术，如果有人利用本文知识和技术进行非法操作进行牟利，带来的任何法律责任都将由操作者本人承担，和本文作者无任何关系。

本文的意图只有一个就是通过分析app 学习更多的Xposed 逆向技术，如果有人利用本文知识和技术进行非法操作进行牟利，带来的任何法律责任都将由操作者本人承担，和本文作者无任何关系。最终还是希望大家能够秉着学习的心态阅读此文，也不枉我写此文的目的，希望大家都能不忘初心，归来仍是少年！

*   本篇还是继续我们的逆向学习系列，最终要实现的需求就是可以自由控制微信掷骰子停下来的点数。
*   首先我们要在掷骰子之前加个用于选择点数的对话框，因为每次掷骰子之前要点击发送表情的那个图标（如下图），所以我们就加到这里好了。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-a223b3e287851c7d.png)
*   要在点击这里之后弹出一个我们新加用于选择点数的对话框，我们就得先找到这个图标的点击事件执行的方法。下面我用ddms 工具来录制下点击图标执行的方法轨迹，通过搜索click 来找到点击方法。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-91b005b1e4a56c96.png)
*   可以看到通过ddms 工具来找点击方法还是很方便的，一下就找到了。下面我们就来hook 该方法，在该方法执行前弹出一个选点数对话框。这里要注意创建一个对话框需要一个Context，这个上下文对象不能是Application 的上下文对象，只能是当前页面的上下文对象。所以我看下点击事件所在类源码看下里面有没有相应的Context 可以供我们使用。要搜索找到点击事件的类，需要用到我们[Xposed 逆向学习（一）](https://www.jianshu.com/p/2d5f8e98d9f6)中关于内部类的知识点，这里就不再重复了，不清楚的可以回去看，从下面源码可以看出点击事件所在类ChatFooter_6 是ChatFooter 的内部类。ChatFooter_6 里是没有看到Context 对象的，但是它通过构造方法传入了一个ChatFooter 对象，并在类里持有一个ChatFooter 对象的变量qMv。  
    ![](http://upload-images.jianshu.io/upload_images/4834678-e583686f378c2a78.png)
*   既然点击事件类里面没有，但是它里面有一个ChatFooter 对象的变量，所以我们可以来看下ChatFooter 类里面有没有Context 对象，如果有的话，我们也是可以获取得到一个上下文对象来供我们创建对话框使用的。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-092a46c4a983aa3b.png)
*   通过截图可以看到ChatFooter 类里面是有我们所需要的上下文对象的，这样一来我们就可以hook 住点击事件，并在点击事件方法前弹出我们选点数的对话框了，这里注意骰子只有六个面，但是我弄了7 个选项，目的是为了当你选择了0 的时候停下来的点数还是微信随机产生的，除此之外我把对话框的除了通过选择选项来取消之外的其他取消比如点击对话框外让对话框取消消失的途径都关闭了，目的就是让你必须按我们逻辑流程来选择一项。

```
XposedHelpers.findAndHookMethod( "com.tencent.mm.pluginsdk.ui.chat.ChatFooter$6" , classLoader, "onClick" , View.class, new XC_MethodHook() {
             @Override 
            protected  void  beforeHookedMethod (MethodHookParam param)  throws Throwable {
                 super .beforeHookedMethod(param);
                Object thisObject = param.thisObject;
                Object qMv = XposedHelpers.getObjectField(thisObject, "qMv" );
                 final Context currentContext = (Context) XposedHelpers.getObjectField(qMv, "context" );
                 //Toast.makeText(currentContext, "点击了图标", Toast.LENGTH_SHORT).show(); 
                new AlertDialog.Builder(currentContext)
                        .setTitle( "想要掷几点？" )
                        .setSingleChoiceItems( new String[]{ "0" , "1" , "2" , "3" , "4" , "5" , "6" }, 0 ,
                                 new DialogInterface.OnClickListener() {
                                     @Override 
                                    public  void  onClick (DialogInterface dialog, int which)  {
                                        index = which + "" ;
                                        Toast.makeText(currentContext, index, Toast.LENGTH_SHORT).show();
                                        dialog.dismiss();
                                    }
                                })
                        //.setNegativeButton("取消", null) 
                        .setCancelable( false )
                        .show();
            }
        });
```

![](http://upload-images.jianshu.io/upload_images/4834678-31487d6ffb805885.png)

*   好了，对话框加好了，接下来我们就来找我们的控制骰子点数的方法了。还是用我们ddms 感觉来录制下骰子投掷时执行的方法轨迹，从方法轨迹一步一步跟看能跟到我们需要的方法不。
    
    ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)  
    ![](http://upload-images.jianshu.io/upload_images/4834678-14a0ee223ff4d5ff.png)
*   通过方法轨迹和源码我们可以看到switch 走的是25 所在分支。25 分支一开始执行轨迹里的self 是一个if 判断，按我们经验来说这个判断一般都是异常判断，这里是我们想要方法的可能性很小，所以我们继续往下跟a 方法。hook 参数来看下。
    
    ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

```
//第一个参数是个从源码可以看出是个GridView，所以这个参数意义对我们来说作用不大，不打印
        Class<?> smileyGrid = XposedHelpers.findClassIfExists( "com.tencent.mm.view.SmileyGrid" , classLoader);
         //第二个参数，里面很多变量，而且从源码（如上图）可以看出它还有个toString方法，
        //所以我们就不通过反射拿到变量名去打印变量了，直接用toString了
        Class<?> emojiInfo = XposedHelpers.findClassIfExists( "com.tencent.mm.storage.emotion.EmojiInfo" , classLoader);
        XposedHelpers.findAndHookMethod( "com.tencent.mm.view.SmileyGrid" , classLoader, "a" , smileyGrid, emojiInfo, new XC_MethodHook() {
             @Override 
            protected  void  afterHookedMethod (MethodHookParam param)  throws Throwable {
                 super .afterHookedMethod(param);
                 //因为我们录制了方法轨迹，从方法轨迹中看到的方法再去打印调用信息意义就不大了。
                //基本调用信息都能从方法轨迹里看到。
                //Log.e(TAG, "SmileyGrid调用信息：", new Exception()); 
                if (param.args[ 1 ] != null )
                    Log.e(TAG, "emojiInfo：" + param.args[ 1 ].toString());
            }
        });
```

![](http://upload-images.jianshu.io/upload_images/4834678-90da09dcb105b713.png)

*   看下日志好像并没有啥有用的信息，那就先不管了。继续来看下方法轨迹，通过点击a 方法进入到下一个执行的方法也就是a 方法了，并打开a 方法所在源码来看下。
    
    ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)  
    ![](http://upload-images.jianshu.io/upload_images/4834678-290de68287ee06bd.png)
*   通过轨迹下一步执行的l 方法可以定位到执行分支。通过分析执行分支源码可以得出第一个打断点处就是self 执行的地方，第二处就是调用下一个方法l 方法执行的地方。

### 方法一

*   那么我们就先拿第一个断点作为我们下一步目标通过看源码跟踪断点所在位置执行轨迹。

```
EmojiInfo c = ((c) gn(c.class)).getProvider().c(emojiInfo);
```

*   通过这行源码可以看出我们最后就是要通过c 方法得到一个EmojiInfo 对象的返回值。按住Ctrl，点击c 方法，跳到c 方法。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-118c2f5887c2c5ba.png)
*   通过源码可以看到c 方法所在类是个接口，搜索下该接口实现类。
    
    ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)
*   由最开始断点处源码分析知道我们最终目的是要拿到一个EmojiInfo 对象返回值，从上面源码可以看到c 方法返回的是getEmojiMgr() 方法返回的对象的c 方法的返回值，所以我们继续点击getEmojiMgr 跳过去看下。
    
    ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)  
    ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)
*   可以看到d 也是个接口同时还继承了e 接口，所以最终e 接口的实现类可以顺理成章的去调用e 接口子类接口实现类的c 方法去得到相同的EmojiInfo 对象返回值。我们继续来搜索下d 接口的实现类。
    
    ![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)
*   可以看到就一个实现类，找到实现类里c 方法看下，我们的目的是emojiInfo，所以我们找与之相关的执行地方去看，从第一处我们打断点处开始，点击eF 方法跳转过去看下。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-9bbbdcb490653430.png)
*   返回值是个int，而且通过Random 产生了一个随机数，看到这里你肯定是会有点想法的，很可能这个方法就是我们要找的那个方法。hook 一下返回值，验证下我们的猜测。

```
XposedHelpers.findAndHookMethod( "com.tencent.mm.sdk.platformtools.bi" , classLoader, "eF" , int .class, int .class, new XC_MethodHook() {
             @Override 
            protected  void  afterHookedMethod (MethodHookParam param)  throws Throwable {
                 super .afterHookedMethod(param);
                Log.e(TAG, "返回值" + param.getResult());
            }
        });
```

*   掷几次骰子来对比下返回值和最终停下来的点数看下有没有对应关系。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-740fdec2e9b7c328.png)  
    ![](http://upload-images.jianshu.io/upload_images/4834678-d906c48d0030751d.png)
*   看到这里结果应该很明显了吧。最终的点数就是这个方法返回值加一。好了，到此决定点数的方法就找到了，但是你会不会感觉这样找起来其实还是挺困难的，得有一定的分析源码的和猜测敏锐度的能力。所以我们下面就继续来讲一个简单的定位到目标方法的思路。

### 方法二

*   这个思路对应的就是上面截图（SmileyGrid 类的a 方法）的第二个断点处，对应下面截图的l（从方法轨迹可以看到该方法完整类名路径）方法，它的参数也是EmojiInfo 对象。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-816a003cc7574ac5.png)
*   hook 下EmojiInfo 类型的参数。

```
Class<?> emojiInfo = XposedHelpers.findClassIfExists( "com.tencent.mm.storage.emotion.EmojiInfo" , classLoader);
        XposedHelpers.findAndHookMethod( "com.tencent.mm.ui.chatting.w" , classLoader, "l" , emojiInfo, new XC_MethodHook() {
             @Override 
            protected  void  afterHookedMethod (MethodHookParam param)  throws Throwable {
                 super .afterHookedMethod(param);
                 if (param.args[ 0 ] != null )
                    Log.e(TAG, "emojiInfo：" + param.args[ 0 ].toString());
            }
        });
```

![](http://upload-images.jianshu.io/upload_images/4834678-f72880ce77217f1f.png)  
![](http://upload-images.jianshu.io/upload_images/4834678-b50f14bc8fa3ba31.png)  
![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

*   这样一对比结果应该就更明显了吧，dice_2.png 对应停下来的点数是2，dice_1.png 对应点数1。既然结果与EmojiInfo 类的参数有对应关系，那么我们就来hook 下EmojiInfo 类的构造方法，来根据它的调用信息来追踪到目标方法。

```
XposedHelpers.findAndHookConstructor( "com.tencent.mm.storage.emotion.EmojiInfo" , classLoader, new XC_MethodHook() {
             @Override 
            protected  void  afterHookedMethod (MethodHookParam param)  throws Throwable {
                 super .afterHookedMethod(param);
                Log.e(TAG, "EmojiInfo构造方法调用信息：" , new Exception());
            }
        });
```

![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)

*   好了直接就可以找到上面方法一对应的倒数第二步的com.tencent.mm.plugin.emoji.egc 方法了，接下来思路和方法一就完全一样了，这里就略去了。
*   从两个方法可以看出EmojiInfo 对象一直贯穿着我们跟踪的整条线，所以这里给了我们一个思路，当我们发现某个对象有用时就可以一直围绕着它去找我们的目标方法。
*   这样一来我们就可以结合起最开始的对话框选择一个点数传入到下面方法，将eF 的返回值修改为我们选择的点数，这样我们就可以随意控制骰子停下来的点数了。

```
/**
     * @param classLoader
     * @param number 想要骰子停下来的点数
     */ 
    private  void  modifyDotNumber (ClassLoader classLoader, final  int number)  {
        XposedHelpers.findAndHookMethod( "com.tencent.mm.sdk.platformtools.bi" , classLoader, "eF" , int .class, int .class, new XC_MethodHook() {
             @Override 
            protected  void  afterHookedMethod (MethodHookParam param)  throws Throwable {
                 super .afterHookedMethod(param);
                 if (number > 0 && number < 7 ) { //进行下判断，传入的参数必须有骰子对应面的点数时我们才修改
                    param.setResult(number - 1 ); //这里要注意下对应关系，返回值对应骰子点数减一
                }
            }
        });
    }
```

*   实际上你这样写一测试会发现有问题，由于number 是final 的缘故会导致修改的点数一直都是第一次你选择的那个点数。所以我们修改下，把选择要修改的点数抽取成一个静态全局变量放外边去。

```
//选择的想要投掷的点数
    private  static  int dot;
     private  void  hook ( final ClassLoader classLoader)  {
        XposedHelpers.findAndHookMethod( "com.tencent.mm.pluginsdk.ui.chat.ChatFooter$6" , classLoader, "onClick" , View.class, new XC_MethodHook() {
             @Override 
            protected  void  beforeHookedMethod (MethodHookParam param)  throws Throwable {
                 super .beforeHookedMethod(param);
                Object thisObject = param.thisObject;
                Object qMv = XposedHelpers.getObjectField(thisObject, "qMv" );
                 final Context currentContext = (Context) XposedHelpers.getObjectField(qMv, "context" );
                 new AlertDialog.Builder(currentContext)
                        .setTitle( "想要掷几点？" )
                        .setSingleChoiceItems( new String[]{ "0" , "1" , "2" , "3" , "4" , "5" , "6" }, 0 ,
                                 new DialogInterface.OnClickListener() {
                                     @Override 
                                    public  void  onClick (DialogInterface dialog, int which)  {
                                        dot = which;
                                        dialog.dismiss();
                                    }
                                })
                        .setCancelable( false )
                        .show();
            }
        });
        XposedHelpers.findAndHookMethod( "com.tencent.mm.sdk.platformtools.bi" , classLoader, "eF" , int .class, int .class, new XC_MethodHook() {
             @Override 
            protected  void  afterHookedMethod (MethodHookParam param)  throws Throwable {
                 super .afterHookedMethod(param);
                 if (dot > 0 && dot < 7 ) { //进行下判断，传入的参数必须有骰子对应面的点数时我们才修改
                    param.setResult(dot - 1 ); //这里要注意下对应关系，返回值对应骰子点数减一
                }
            }
            @Override 
            protected  void  beforeHookedMethod (MethodHookParam param)  throws Throwable {
                 super .beforeHookedMethod(param);
                 if (dot > 0 && dot < 7 ) { //进行下判断，传入的参数必须有骰子对应面的点数时我们才修改
                    param.setResult(dot - 1 ); //这里要注意下对应关系，返回值对应骰子点数减一
                }
            }
        });
    }
```

*   这样修改后就没问题了，每次骰子停下来的点数就是我们选择的点数了，这样就可以和朋友去愉快的玩耍下了。最后附个让停下来点数从1-6 的成果图。
    
    ![](http://upload-images.jianshu.io/upload_images/4834678-4f5e8a3d119e3b21.png)