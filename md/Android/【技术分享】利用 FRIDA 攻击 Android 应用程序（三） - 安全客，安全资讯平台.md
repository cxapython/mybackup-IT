> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/85996)

> Frida 是一个动态代码插桩框架。本篇文章主要介绍如何使用 Frida 解决 OWASP Android crackme。

[![](https://p4.ssl.qhimg.com/t017f286902b6434321.jpg)](https://p4.ssl.qhimg.com/t017f286902b6434321.jpg)

翻译：[**houjingyi233**](http://bobao.360.cn/member/contribute?uid=2802394681)

预估稿费：200RMB

投稿方式：发送邮件至 linwei#360.cn，或登陆网页版在线投稿

传送门

[【技术分享】利用 FRIDA 攻击 Android 应用程序（一）](http://bobao.360.cn/learning/detail/3641.html)

[**【技术分享】利用 FRIDA 攻击 Android 应用程序（二）**](http://bobao.360.cn/learning/detail/3634.html)

**前言**

在我的有关 frida 的第二篇博客发布不久之后，[@muellerberndt](https://twitter.com/muellerberndt) 决定发布另一个 OWASP Android crackme，我很想知道是否可以再次用 frida 解决。如果你想跟着我做一遍，你需要下面的工具。

[**OWASP Uncrackable Level2 APK**](https://github.com/OWASP/owasp-mstg/blob/master/OMTG-Files/02_Crackmes/01_Android/Level_02/UnCrackable-Level2.apk)

[**Android SDK 和模拟器**](https://developer.android.com/studio/index.html) **(我使用的是 Android 7.1 x64 镜像)**

[**frida**](https://www.codemetrix.net/hacking-android-apps-with-frida-3/frida.re)**(和** [**frida-server**](https://github.com/frida/frida/releases)**)** 

[**bytecodeviewer**](https://bytecodeviewer.com/)

[**radare2**](https://radare.org/)**(或您选择的其他一些反汇编工具)**

[**apktool**](https://ibotpeaches.github.io/Apktool/)

如果您需要知道如何安装 Frida，请查看 Frida 文档。对于 Frida 的使用，您还可以检查本教程的第 [I](https://www.codemetrix.net/hacking-android-apps-with-frida-1/) 部分。我假设你在继续之前拥有上面的工具，并且基本熟悉 Frida。另外，确保 Frida 可以连接到您的设备 / 模拟器 (例如使用 frida-ps -U)。我将向您展示各种方法来克服具体的问题，如果您只是寻找一个快速的解决方案，请在本教程末尾查看 Frida 脚本。注意：如果使用 frida 遇到了

```
Error: access violation accessing 0xebad8082
```

或者类似的错误，从模拟器中擦除用户数据，重新启动并重新安装该 apk 可能有助于解决问题。做好可能需要多次尝试的准备。该应用程序可能会崩溃，模拟器可能会重新启动，一切可能会搞砸，但是最终我们会成功的。

**第一次尝试**

和 UnCrackable1 一样，当在仿真器中运行它时，它会检测到它是在 root 设备上运行的。 

[![](https://p4.ssl.qhimg.com/t011298204f2fe5f7d6.png)](https://p4.ssl.qhimg.com/t011298204f2fe5f7d6.png)

我们可以尝试像前面一样 hook OnClickListener。但首先我们来看看我们是否可以连接 frida 开始 tampering。   

[![](https://p2.ssl.qhimg.com/t01ad90be6202963626.png)](https://p2.ssl.qhimg.com/t01ad90be6202963626.png)

有两个名称相同的进程，我们可以通过 frida-ps -U 验证一下。 

[![](https://p2.ssl.qhimg.com/t019035f3ca4c124cb1.png)](https://p2.ssl.qhimg.com/t019035f3ca4c124cb1.png)

我们来试试将 frida 注入父进程。 

[![](https://p1.ssl.qhimg.com/t01c27eb1a314c48430.png)](https://p1.ssl.qhimg.com/t01c27eb1a314c48430.png)

这里发生了什么？我们来看看应用程序吧。解压缩 apk 并反编译 classes.dex。  

```
package sg.vantagepoint.uncrackable2;
 
import android.app.AlertDialog;
import android.content.Context;
import android.content.DialogInterface;
import android.os.AsyncTask;
import android.os.Bundle;
import android.support.v7.app.c;
import android.text.Editable;
import android.view.View;
import android.widget.EditText;
import sg.vantagepoint.a.a;
import sg.vantagepoint.a.b;
import sg.vantagepoint.uncrackable2.CodeCheck;
import sg.vantagepoint.uncrackable2.MainActivity;
 
public class MainActivity
extends c {
    private CodeCheck m;
 
    static {
        System.loadLibrary("foo"); //[1]
    }
 
    private void a(String string) {
        AlertDialog Dialog = new AlertDialog.Builder((Context)this).create();
        Dialog.setTitle((CharSequence)string);
        Dialog.setMessage((CharSequence)"This in unacceptable. The app is now going to exit.");
        Dialog.setButton(-3, (CharSequence)"OK", (DialogInterface.OnClickListener)new /* Unavailable Anonymous Inner Class!! */);
        Dialog.setCancelable(false);
        Dialog.show();
    }
 
    static /* synthetic */ void a(MainActivity mainActivity, String string) {
        mainActivity.a(string);
    }
 
    private native void init(); //[2]
 
    protected void onCreate(Bundle bundle) {
        this.init(); //[3]
        if (b.a() || b.b() || b.c()) {
            this.a("Root detected!");
        }
        if (a.a((Context)this.getApplicationContext())) {
            this.a("App is debuggable!");
        }
        new /* Unavailable Anonymous Inner Class!! */.execute((Object[])new Void[]{null, null, null});
        this.m = new CodeCheck();
        super.onCreate(bundle);
        this.setContentView(2130968603);
    }
 
    public void verify(View view) {
        String string = ((EditText)this.findViewById(2131427422)).getText().toString();
        AlertDialog Dialog = new AlertDialog.Builder((Context)this).create();
        if (this.m.a(string)) {
            Dialog.setTitle((CharSequence)"Success!");
            Dialog.setMessage((CharSequence)"This is the correct secret.");
        } else {
            Dialog.setTitle((CharSequence)"Nope...");
            Dialog.setMessage((CharSequence)"That's not it. Try again.");
        }
        Dialog.setButton(-3, (CharSequence)"OK", (DialogInterface.OnClickListener)new /* Unavailable Anonymous Inner Class!! */);
        Dialog.show();
    }
}
```

我们注意到程序加载了 foo 库 ([1])。在 onCreate 方法的第一行程序调用了 this.init()([3])，它被声明成一个 native 方法 ([2])，所以它可能是 foo 的一部分。现在我们来看看 foo 库。使用 radare2 打开它并分析，列出它的导出函数。   

[![](https://p0.ssl.qhimg.com/t01c2188984cd199abe.png)](https://p0.ssl.qhimg.com/t01c2188984cd199abe.png)

该库导出两个有趣的功能：Java_sg_vantagepoint_uncrackable2_MainActivity_init 和 Java_sg_vantagepoint_uncrackable2_CodeCheck_bar。我们来看看 Java_sg_vantagepoint_uncrackable2_MainActivity_init。   

[![](https://p5.ssl.qhimg.com/t0100429827c6be15df.png)](https://p5.ssl.qhimg.com/t0100429827c6be15df.png)

这是一个很短的函数。  

[![](https://p1.ssl.qhimg.com/t01f7400b7960b1a921.png)](https://p1.ssl.qhimg.com/t01f7400b7960b1a921.png)

 它调用了 sub.fork_820，这里面有更多的内容。

[![](https://p1.ssl.qhimg.com/t01a66c2caa1a288f5d.png)](https://p1.ssl.qhimg.com/t01a66c2caa1a288f5d.png)

这个函数中调用了 fork、pthread_create、getppid、ptrace 和 waitpid 等函数。这是一个基本的反调试技术，附加调试进程被阻止，因为已经有其他进程作为调试器连接。

**对抗反调试方案一：frida**

我们可以让 frida 为我们生成一个进程而不是将它注入到运行中的进程中。  

[![](https://p3.ssl.qhimg.com/t019297f7d30b4902d5.png)](https://p3.ssl.qhimg.com/t019297f7d30b4902d5.png)

[![](https://p4.ssl.qhimg.com/t01c0b53fb40a439aa1.png)](https://p4.ssl.qhimg.com/t01c0b53fb40a439aa1.png)

frida 注入到 Zygote 中，生成我们的进程并且等待输入，这个过程可能比较漫长。  

**对抗反调试方案二：patch**

我们可以通过 apktool 实现 patch。

[![](https://p5.ssl.qhimg.com/t01936ff49f72cf02bc.png)](https://p5.ssl.qhimg.com/t01936ff49f72cf02bc.png)

(我通过 - r 选项跳过了资源提取，因为在回编译 apk 的时候它可能会导致问题，反正我们这里不需要资源文件。) 看一下 smali/sg/vantagepoint/uncrackable2/MainActivity.smali 中的 smali 代码。你可以在第 82 行找到 init 的调用并注释掉它。

[![](https://p2.ssl.qhimg.com/t01ec6827464ed27337.png)](https://p2.ssl.qhimg.com/t01ec6827464ed27337.png)

回编译 apk(忽略错误)。

[![](https://p5.ssl.qhimg.com/t017a9eb2e19c01b432.png)](https://p5.ssl.qhimg.com/t017a9eb2e19c01b432.png)

对齐。

[![](https://p1.ssl.qhimg.com/t011125b77a2133c44d.png)](https://p1.ssl.qhimg.com/t011125b77a2133c44d.png)

签名 (注意：您需要有一个密钥和密钥库)。

[![](https://p3.ssl.qhimg.com/t010bfa0913789a0c7b.png)](https://p3.ssl.qhimg.com/t010bfa0913789a0c7b.png)

你可以在 [OWASP Mobile Security Testing Guide](https://github.com/OWASP/owasp-mstg/blob/master/Document/0x05c-Reverse-Engineering-and-Tampering.md#repackaging) 中找到更详细的描述。卸载原来的 apk 并安装我们 patch 过的 apk。

[![](https://p1.ssl.qhimg.com/t01d5c5540084a521a6.png)](https://p1.ssl.qhimg.com/t01d5c5540084a521a6.png)

重新启动应用程序。运行 frida-ps，现在只有一个进程了。

[![](https://p1.ssl.qhimg.com/t0187987140c284d3cb.png)](https://p1.ssl.qhimg.com/t0187987140c284d3cb.png)

frida 进行连接也没什么问题。

[![](https://p5.ssl.qhimg.com/t01ead21312c0bd7fd7.png)](https://p5.ssl.qhimg.com/t01ead21312c0bd7fd7.png)

这比在 frida 中增加 - r 选项更为繁琐，但也更普遍。如前所述，当我们使用 patch 过的版本 (我会告诉你如何解决这个问题，所以不要把它删了) 不能轻易地提取需要的字符串。现在我们继续使用原来的 apk。确保安装的是原始的 apk。  

**继续尝试**

在我们摆脱反调试之后来看看如何继续进行下去。一旦按了 OK 按钮，应用程序就会在模拟器中运行时进行 root 检测并退出。我们可以 patch 掉这个行为，也可以用 frida 来解决这个问题。由于 OnClickListener 实现调用，我们可以 hook System.exit 函数使其不产生作用。

```
setImmediate(function() {
    console.log("[*] Starting script");
    Java.perform(function() {
        exitClass = Java.use("java.lang.System");
        exitClass.exit.implementation = function() {
            console.log("[*] System.exit called");
        }
        console.log("[*] Hooking calls to System.exit");
    });
});
```

再次关闭任何正在运行的 UnCrackable2 实例，并再次在 frida 的帮助下启动它。   

[![](https://p5.ssl.qhimg.com/t018e0625ed547fea75.png)](https://p5.ssl.qhimg.com/t018e0625ed547fea75.png)

等到 app 启动，frida 在控制台中显示 Hooking calls… 然后按 OK。你应该得到这样的信息。   

[![](https://p0.ssl.qhimg.com/t01565cb87ac46459df.png)](https://p0.ssl.qhimg.com/t01565cb87ac46459df.png)

该应用程序不再退出，我们可以输入一个字符串。  

[![](https://p4.ssl.qhimg.com/t01e562f545d90d8067.png)](https://p4.ssl.qhimg.com/t01e562f545d90d8067.png)

但是我们应该在这里输入什么呢？看看 MainActivity。

```
this.m = new CodeCheck();
 
[...]
 
//in method: public void verify
if (this.m.a(string)) {
            Dialog.setTitle((CharSequence)"Success!");
            Dialog.setMessage((CharSequence)"This is the correct secret.");
}
```

这是 CodeCheck 类。

```
package sg.vantagepoint.uncrackable2;
 
public class CodeCheck {
    private native boolean bar(byte[] var1);
 
    public boolean a(String string) {
        return this.bar(string.getBytes()); //Call to a native function
    }
}
```

我们注意到输入的字符串被传递给了一个 native 方法 bar。同样，我们在 libfoo.so 中找到了这个函数。使用 radare2 寻找这个函数的地址并反汇编它。 

[![](https://p2.ssl.qhimg.com/t017f9dd77ba4f7755e.png)](https://p2.ssl.qhimg.com/t017f9dd77ba4f7755e.png)

反汇编代码中有一些字符串比较操作，有一个有趣的明文字符串 Thanks for all t。输入这个字符串，但是它不起作用。看看地址 0x000010d8 处的反汇编代码。 

[![](https://p4.ssl.qhimg.com/t01d436d1584289972e.png)](https://p4.ssl.qhimg.com/t01d436d1584289972e.png)

这里有一个 eax 和 0x17 的比较，如果不相同的话 strncmp 函数不会被调用。我们同时注意到 0x17 是 strncmp 的一个参数。 

[![](https://p4.ssl.qhimg.com/t013eff11ddb2139fe0.png)](https://p4.ssl.qhimg.com/t013eff11ddb2139fe0.png)

464 位的 linux 中函数的前 6 个参数通过寄存器传递，前 3 个寄存器分别是 RDI、 RSI 和 RDX。所以 strncmp 函数将比较 0x17=23 个字符。可以推断，输入的字符串的长度应该是 23。让我们尝试 hook strncmp，并打印出它的参数。如果你这样做，你会发现 strncmp 被调用了很多次，我们需要进一步限制输出。  

```
var strncmp = undefined;
imports = Module.enumerateImportsSync("libfoo.so");
 
for(i = 0; i < imports.length; i++) {
if(imports[i].name == "strncmp") {
        strncmp = imports[i].address;
        break;
    }
 
}
 
Interceptor.attach(strncmp, {
            onEnter: function (args) {
               if(args[2].toInt32() == 23 && Memory.readUtf8String(args[0],23) == "01234567890123456789012") {
                    console.log("[*] Secret string at " + args[1] + ": " + Memory.readUtf8String(args[1],23));
               }
            }
});
```

1. 该脚本调用 Module.enumerateImportsSync 以从 libfoo.so 中获取有关导入信息的对象数组。我们遍历这个数组，直到找到 strncmp 并检索其地址。然后我们将 interceptor 附加到它。   
2.Java 中的字符串不会以 null 结束。当 strncmp 使用 frida 的 Memory.readUtf8String 方法访问字符串指针的内存位置并且不提供长度时，frida 会期待一个结束符，或者输出一些垃圾内存。它不知道字符串在哪里结束。如果我们指定要读取的字符数量作为第二个参数就解决了这个问题。   
3. 如果我们没有限制 strncmp 参数的条件将得到很多输出。限制条件为第三个参数 size_t 为 23。   
我怎么如何知道 args[0] 是我们的输入，args[1] 是我们寻找的字符串呢？我不知道，我只是测试，将大量的输出 dump 到屏幕以找到我的输入。如果你不想跳过这部分，可以删除上面脚本中的 if 语句，并使用 frida 的 hexdump 输出。

```
buf = Memory.readByteArray(args[0],32);
console.log(hexdump(buf, {
     offset: 0,
     length: 32,
     header: true,
     ansi: true
}));
 
buf = Memory.readByteArray(args[1],32);
console.log(hexdump(buf, {
    offset: 0,
    length: 32,
    header: true,
   ansi: true
}));
```

以下是完整版的脚本，可以更好地输出参数。

```
setImmediate(function() {
    Java.perform(function() {
        console.log("[*] Hooking calls to System.exit");
        exitClass = Java.use("java.lang.System");
        exitClass.exit.implementation = function() {
            console.log("[*] System.exit called");
        }
 
        var strncmp = undefined;
        imports = Module.enumerateImportsSync("libfoo.so");
 
        for(i = 0; i < imports.length; i++) {
        if(imports[i].name == "strncmp") {
                strncmp = imports[i].address;
                break;
            }
 
        }
 
        Interceptor.attach(strncmp, {
            onEnter: function (args) {
               if(args[2].toInt32() == 23 && Memory.readUtf8String(args[0],23) == "01234567890123456789012") {
                    console.log("[*] Secret string at " + args[1] + ": " + Memory.readUtf8String(args[1],23));
                }
             },
        });
        console.log("[*] Intercepting strncmp");
    });
});
```

现在启动 frida 加载这个脚本。 

[![](https://p1.ssl.qhimg.com/t01b756d7e56b1d1e57.png)](https://p1.ssl.qhimg.com/t01b756d7e56b1d1e57.png)

输入字符串并且按下 VERIFY。   

[![](https://p4.ssl.qhimg.com/t0185931bc4fa6c71ac.png)](https://p4.ssl.qhimg.com/t0185931bc4fa6c71ac.png)

在控制台会看到下面的结果。

[![](https://p0.ssl.qhimg.com/t011c4c8e1c074515b5.png)](https://p0.ssl.qhimg.com/t011c4c8e1c074515b5.png)

我们找到了正确的字符串 Thanks for all the fish。   

[![](https://p3.ssl.qhimg.com/t01ef42947ee7479013.png)](https://p3.ssl.qhimg.com/t01ef42947ee7479013.png)

**使用 patch 过的 apk**

当我们使用 patch 过的 apk 时可能不会得到需要的字符串。libfoo 库中的 init 函数包含一些初始化逻辑，阻止应用程序根据我们的输入检查或解码字符串。如果我们再看看 init 函数的反汇编代码会看到有趣的一行。 

[![](https://p2.ssl.qhimg.com/t012e35ebfc5411af3e.png)](https://p2.ssl.qhimg.com/t012e35ebfc5411af3e.png)

相同的变量会在 libfoo 库的 bar 函数中检查，如果没有设置，那么代码会跳过 strncmp。 

[![](https://p1.ssl.qhimg.com/t01f819173da79c41e5.png)](https://p1.ssl.qhimg.com/t01f819173da79c41e5.png)

它可能是一个 boolean 类型的变量，当 init 函数运行时被设置。如果我们想要让 patch 过的 apk 调用 strncmp 函数就需要设置这个变量或者至少阻止它跳过 strncmp 的调用。我们可以再 patch 一次，但是既然这是 frida 教程，我们可以使用它动态改变内存。下面是可供 patch 过的 apk 使用的完整的脚本。  

```
setImmediate(function() {
    Java.perform(function() {
        console.log("[*] Hooking calls to System.exit");
        exitClass = Java.use("java.lang.System");
        exitClass.exit.implementation = function() {
            console.log("[*] System.exit called");
        }
 
        var strncmp = undefined;
        imports = Module.enumerateImportsSync("libfoo.so");
 
        for(i = 0; i < imports.length; i++) {
            if(imports[i].name == "strncmp") {
                strncmp = imports[i].address;
                break;
            }
 
        }
 
        //Get base address of library
        var libfoo = Module.findBaseAddress("libfoo.so");
 
        //Calculate address of variable
        var initialized = libfoo.add(ptr("0x400C"));
 
        //Write 1 to the variable
        Memory.writeInt(initialized,1);
 
        Interceptor.attach(strncmp, {
            onEnter: function (args) {
               if(args[2].toInt32() == 23 && Memory.readUtf8String(args[0],23) == "01234567890123456789012") {
                    console.log("[*] Secret string at " + args[1] + ": " + Memory.readUtf8String(args[1],23));
                }
             },
        });
        console.log("[*] Intercepting strncmp");
    });
});
```