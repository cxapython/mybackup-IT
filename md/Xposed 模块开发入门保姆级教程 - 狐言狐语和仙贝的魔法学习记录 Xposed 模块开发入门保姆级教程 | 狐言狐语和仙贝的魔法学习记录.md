> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.ketal.icu](https://blog.ketal.icu/cn/Xposed%E6%A8%A1%E5%9D%97%E5%BC%80%E5%8F%91%E5%85%A5%E9%97%A8%E4%BF%9D%E5%A7%86%E7%BA%A7%E6%95%99%E7%A8%8B/)

> Xposed 模块开发初学者的指南

介绍
--

介于网络上的 Xposed 开发教程过于破烂，所以狐狸狸决定自己写一篇教程来帮助各位想要开发 / 正在进行开发 Xposed 模块的开发者。

开始之前
----

在开始之前，你需要准备好：

*   一台可以安装 Xposed 框架的手机（推荐 LSPosed、Android 10+）
*   一台可以编写代码并且装有 jdk 的电脑
*   一个名叫 [Android Studio](https://developer.android.com/studio) 的软件（当然你用 [IDEA](https://www.jetbrains.com/zh-cn/idea/download/) 也没问题就是了）
*   一个反编译软件，如：[JADX](https://github.com/skylot/jadx)
*   一个可以查看布局的 App，如：开发者助手

其次，本文假定你已经学会以下内容：

*   会 Java/Kotlin 其中一种语言（强烈推荐 Kotlin，对模块开发特别友好，为此我专门写了一个 kotlin 的 [Xposed 模块开发用库](https://github.com/KyuubiRan/EzXHelper)来使用，能帮助开发者省下 30%~50% 甚至更多的代码量，注重模块本身逻辑的编写！不过本期教程不会使用就是了）
*   Java 的反射
*   Android 的基础套件（如 Context、View 等，其实这两个就足够了，已经能干很多了）

准备工作
----

### 创建项目 & 引入依赖

首先，我们打开 Android Studio 创建一个空项目，不需要任何 Activity，语言凭个人喜好创建，反正我两种都会讲。  
然后，我们需要引入 Xposed 的库，不过它并没有上传到 MavenCentral 上，所以我们需要在`settings.gradle`里修改一下 (gradle 7.0+)

```
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        mavenCentral()
        maven { url 'https://api.xposed.info/' }  // 添加这一行即可
    }
}


```

之后，进入我们 app 目录下的 build.gradle 引入 xposed 的依赖，当然你也可以移除所有依赖来让安装包变小

```
dependencies {
    compileOnly 'de.robv.android.xposed:api:82' 
    // compileOnly 'de.robv.android.xposed:api:82:sources' // 不要导入源码，这会导致idea无法索引文件，从而让语法提示失效
}


```

如果你移除了所有依赖只保留了 Xposed，你就会发现，你的项目不能 build，会直接报错！没关系，我们修复一下：

*   移除`src/res/values/themes.xml`里面的主题 (注意还有个夜间模式，在 values-night 文件夹下)
*   移除`AndroidManifest.xml`文件里`<application ... />`中的`android:theme="xxx"`那一行 移除之后就能 build 了。我们继续，我们需要创建一个模块作用域文件，在`values`目录下创建一个名叫`arrays`的资源文件，它的内容如下：

```
<resources>
    <string-array  >
        <!-- 这里填写模块的作用域应用的包名，可以填多个。 -->
        <item>me.kyuubiran.xposedapp</item> 
    </string-array>
</resources>


```

最后，我们在 Run 那里编辑一下启动配置，勾选`Always install with package manager`并且将`Launch Options`改成`Nothing`

### 声明模块

做完上一步之后，我们需要声明它是一个 Xposed 模块，好让框架发现它，这样我们才能激活模块。  
回到`AndroidManifest.xml`文件里，我们将`<application ... />`改成以下形式（注意，是改成！就是把结尾的`/>`换成`> </application>`）

```
<application ... > 
        <!-- 是否为Xposed模块 -->
        <meta-data
            android:
            android:value="true"/>
        <!-- 模块的简介（在框架中显示） -->
        <meta-data
            android:
            android:value="我是Xposed模块简介" />
        <!-- 模块最低支持的Api版本 一般填54即可 -->
        <meta-data 
            android:     
            android:value="54"/>
        <!-- 模块作用域 -->
        <meta-data
            android:
            android:resource="@array/xposedscope"/>
</appication>


```

然后在`src/main`目录下创建一个文件夹名叫`assets`，并且创建一个文件叫`xposed_init`，**注意，它没有后缀名！！**。  
接着我们需要创建一个入口类，名叫`MainHook`（或者随便你想取什么名字都行），创建好后回到我们的`xposed_init`里并用文本文件的方式打开它，输入我们刚刚创建的类的完整路径。如：`me.kyuubiran.xposedtutorial.MainHook`，同时**注意大小写**。  
到这里，我们声明模块的部分就结束了！怎么样，接下去就到了我们激动人心的编写模块部分了！

模块编写
----

### MainHook

来到我们的`MainHook`，首先我们需要实现以下 Xposed 的 IXposedHookLoadPackage 接口，以便执行 Hook 操作。具体操作如下  
`Java:`

```
package me.kyuubiran.xposedtutorial;

import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.callbacks.XC_LoadPackage;

public class MainHook implements IXposedHookLoadPackage {
    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) throws Throwable {
        // 过滤不必要的应用
        if (!lpparam.packageName.equals("me.kyuubiran.xposedapp")) return;
        // 执行Hook
        hook(lpparam);
    }

    private void hook(XC_LoadPackage.LoadPackageParam lpparam) {
        // 具体流程
    }
}


```

`Kotlin:`

```
package me.kyuubiran.xposedtutorial

import de.robv.android.xposed.IXposedHookLoadPackage
import de.robv.android.xposed.callbacks.XC_LoadPackage

class MainHook : IXposedHookLoadPackage {
    override fun handleLoadPackage(lpparam: XC_LoadPackage.LoadPackageParam) {
        // 过滤不必要的应用
        if (lpparam.packageName != "me.kyuubiran.xposedapp") return
        // 执行Hook
        hook(lpparam)
    }

    private fun hook(lpparam: XC_LoadPackage.LoadPackageParam) {
        // 具体流程
    }
}


```

到这里，我们的准备工作已经完成，安装模块并在框架中激活它！

![](https://blog.ketal.icu/assets/img/xposed_tutorial/%E5%90%AF%E7%94%A8%E6%A8%A1%E5%9D%97.png)

### 反编译

终于到了我们反编译 apk 找 hook 点的时候了！我这里有一份[猜拳游戏](https://github.com/KyuubiRan/Xposed-guess-app/releases)的 apk，今天我我们要做的事情就是成为**赌圣**（  
我们先打开 jadx-gui，选择我们的`guess.apk`，等他加载完，说明反编译就完成了。  
接下去，我们打开应用并且查看它当前布局是什么（我这里使用的是开发者助手专业版）

![](https://blog.ketal.icu/assets/img/xposed_tutorial/%E5%B8%83%E5%B1%80%E6%9F%A5%E7%9C%8B.png)

如图所示，我们现在处于`me.kyuubiran.xposedapp.MainActivity`，接着回到我们的 jadx 中，依次展开并找到这个 Activity，就像这样：

![](https://blog.ketal.icu/assets/img/xposed_tutorial/jadx%E5%8F%8D%E7%BC%96%E8%AF%91.png)

### Hook activity

我们最基础的 hook 方式，就是使用 Xposed 自带的`XposedHelpers.findAndHookMethod`，使用方法如下：  
`Java:`

```
private void hook(XC_LoadPackage.LoadPackageParam lpparam) {
    // 它有两个重载，区别是一个是填Class，一个是填ClassName以及ClassLoader
    // 第一种 填ClassName
    XC_MethodHook.Unhook unhook = XposedHelpers.findAndHookMethod("me.kyuubiran.xposedapp.MainActivity",    // className
            lpparam.classLoader,    // classLoader 使用lpparam.classLoader
            "onCreate",             // 要hook的方法
            Bundle.class,           // 要hook的方法的参数表，如果有多个就用逗号隔开 
            new XC_MethodHook() {   // 最后一个填hook的回调
                @Override
                protected void beforeHookedMethod(MethodHookParam param) {} // Hook方法执行前  
                @Override
                protected void afterHookedMethod(MethodHookParam param) {} // Hook方法执行后
            });
    // 它返回一个unhook 在你不需要继续hook的时候可以调用它来取消Hook
    unhook.unhook();    // 取消空的Hook 

    // 第二种方式 填Class
    // 首先你得加载它的类 我们使用XposedHelpers.findClass即可 参数有两个 一个是类名 一个是类加载器
    Class<?> clazz = XposedHelpers.findClass("me.kyuubiran.xposedapp.MainActivity", lpparam.classLoader);
    XposedHelpers.findAndHookMethod(clazz, "onCreate", Bundle.class, new XC_MethodHook() {
        @Override
        protected void afterHookedMethod(MethodHookParam param){
            // 由于我们需要在Activity创建之后再弹出Toast，所以我们Hook方法执行之后
            Toast.makeText((Activity) param.thisObject, "模块加载成功！", Toast.LENGTH_SHORT).show();
        }
    });
}


```

`Kotlin:`

```
private fun hook(lpparam: XC_LoadPackage.LoadPackageParam) {
    // 它有两个重载，区别是一个是填Class，一个是填ClassName以及ClassLoader
    // 第一种 填ClassName
    val unhook = XposedHelpers.findAndHookMethod("me.kyuubiran.xposedapp.MainActivity",   // className
        lpparam.classLoader,        // classLoader 使用lpparam.classLoader
        "onCreate",                 // 要hook的方法
        Bundle::class.java          // 要hook的方法的参数表，如果有多个就用逗号隔开 
        object : XC_MethodHook() {  // 最后一个填hook的回调 
            override fun beforeHookedMethod(param: MethodHookParam) {} // Hook方法执行前
            override fun afterHookedMethod(param: MethodHookParam) {} // Hook方法执行后
        })
    // 它返回一个unhook 在你不需要继续hook的时候可以调用它来取消Hook
    unhook.unhook()    // 取消空的Hook
        
    // 第二种方式 填Class
    // 首先你得加载它的类 我们使用XposedHelpers.findClass即可 参数有两个 一个是类名 一个是类加载器
    val clazz = XposedHelpers.findClass("me.kyuubiran.xposedapp.MainActivity", lpparam.classLoader)
    // 相当于合并了第一、第二两个参数，所以它的使用方式如下：
    XposedHelpers.findAndHookMethod(clazz,"onCreate", Bundle::class.java, object : XC_MethodHook() {
        override fun afterHookedMethod(param: MethodHookParam) {
          // 由于我们需要在Activity创建之后再弹出Toast，所以我们Hook方法执行之后
          Toast.makeText(param.thisObject as Activity, "模块加载成功！", Toast.LENGTH_SHORT).show()
        }
    })
}


```

其中，`param.thisObject`代表调用这个方法的对象（如果是静态方法的话可能是为 null），我们 hook 的是 Activity 的`onCreate`方法，调用它的自然是这个 Activity 啦，所以我们把`param.thisObject`强制转换为 Activity 类型就能丢给 Toast 来 makeText 啦！  
安装 Xposed 模块并且勾选它，如果弹出 Toast 了，说明你的模块已经生效了！

![](https://blog.ketal.icu/assets/img/xposed_tutorial/%E6%BF%80%E6%B4%BBToast.png)

### Hook 其他类

终于到了我们成为赌圣的时刻！我们需要先分析一下代码，可以看到`MainActivity`里面有个`confirm`方法，它里面包含了判断我们胜负的逻辑。

![](https://blog.ketal.icu/assets/img/xposed_tutorial/%E7%A1%AE%E8%AE%A4%E6%8C%89%E9%92%AE.png)

接着看，它在里面 new 了一个 Guess 对象，我们`CTRL+鼠标左键`点开 Guess 类看看。

![](https://blog.ketal.icu/assets/img/xposed_tutorial/Guess%E7%B1%BB.png)

可以看到，里面有一个`isDraw`和`isWin`来判断我们的胜负和平局，接下去我们就需要 hook 这两个方法！ `Java:`

```
private void hook(XC_LoadPackage.LoadPackageParam lpparam) {
    XposedHelpers.findAndHookMethod("me.kyuubiran.xposedapp.Guess",
            lpparam.classLoader,
            "isDraw",
            int.class,    // 如果参数是宿主的类，你可以使用findClass来加载那个类或是填写那个类的完整名称！
            new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) {
                    // 将返回值设置为false 表示我们不是平局
                    param.setResult(false);
                }
            });
    XposedHelpers.findAndHookMethod("me.kyuubiran.xposedapp.Guess",
            lpparam.classLoader,
            "isWin",
            int.class,
            boolean.class,
            new XC_MethodReplacement() {   // 另一种回调，将方法直接替换成你要执行的内容（其实就是包装过的beforeHook，基本用不到）
                @Override
                protected Object replaceHookedMethod(MethodHookParam param) {
                    return true;
                }
            });
}


```

`Kotlin:`

```
private fun hook(lpparam: XC_LoadPackage.LoadPackageParam) {
    XposedHelpers.findAndHookMethod(
        "me.kyuubiran.xposedapp.Guess",
        lpparam.classLoader,
        "isDraw",
        Int::class.java, // 如果参数是宿主的类，你可以使用findClass来加载那个类或是填写那个类的完整名称！
        object : XC_MethodHook() {
            override fun beforeHookedMethod(param: MethodHookParam) {
                // 将返回值设置为false 表示我们不是平局
                param.result = false
            }
        })
    XposedHelpers.findAndHookMethod("me.kyuubiran.xposedapp.Guess",
        lpparam.classLoader,
        "isWin",
        Int::class.java,
        object : XC_MethodReplacement() {   // 另一种回调，将方法直接替换成你要执行的内容（其实就是包装过的beforeHook，基本用不到）
            override fun replaceHookedMethod(param: MethodHookParam): Any {
                // 直接返回true
                return true
            }
        })
}


```

修改代码之后我们重新安装模块并且启动 app，这时我们发现，无论是我们本应该输或者平局，它都算我们胜利！  
虽然这么做你赢了游戏，但是别人一眼就看出来了！接下去我来教你如何让对手 “配合” 你一起演戏！

首先，我们观察点击猜拳按钮的代码，发现`isDraw`是先被执行的，而且`Guess`里面的`answer`是类被实例化之后就创建好了的，并且它也没有方法给我们 hook，此时就要出动我们的反射了！  
由于`isDraw`是先被执行的，我们需要在`isDraw`被执行之前就得使用反射进行 “偷梁换柱”，不多 bb 了上代码！  
`Java:`

```
private void hook(XC_LoadPackage.LoadPackageParam lpparam) {
    XposedHelpers.findAndHookMethod("me.kyuubiran.xposedapp.Guess",
            lpparam.classLoader,
            "isDraw",
            int.class,
            new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) {
                    try {                        
                        // 首先 我们要拿到他的字段
                        Field fAnswer = param.thisObject.getClass().getDeclaredField("answer");
                        // 并且让它可访问 否则会报非法访问的错误
                        fAnswer.setAccessible(true);
                        int win = 0;
                        // 根据猜拳应用的逻辑 0是石头 1是剪刀 2是布
                        switch ((int) param.args[0]) {  // 首先我们拿到方法的第一个参数 他是int类型
                            case 0:         // 石头->剪刀
                                win = 1;
                                break;
                            case 1:         // 剪刀->布
                                win = 2;
                                break;
                            case 2:         // 布->石头
                                win = 0;
                                break;
                        }
                        // 最后设置answer的值让对手根据我们的出拳来“演戏”
                        fAnswer.set(param.thisObject, win);
                    } catch (Exception ignored) {
                    }
                }
            });
}


```

`Kotlin:`

```
private fun hook(lpparam: XC_LoadPackage.LoadPackageParam) {
    XposedHelpers.findAndHookMethod(
        "me.kyuubiran.xposedapp.Guess",
        lpparam.classLoader,
        "isDraw",
        Int::class.java,
        object : XC_MethodHook() {
            override fun beforeHookedMethod(param: MethodHookParam) {
                // 首先 我们要拿到他的字段
                val fAnswer = param.thisObject.javaClass.getDeclaredField("answer")
                // 并且让它可访问 否则会报非法访问的错误
                fAnswer.isAccessible = true
                // 根据猜拳应用的逻辑 0是石头 1是剪刀 2是布
                val win = when (param.args[0] as Int) { // 首先我们拿到方法的第一个参数 他是Int类型 就是我们的出拳
                    0 -> 1         // 石头->剪刀
                    1 -> 2         // 剪刀->布
                    2 -> 0         // 布->石头
                    else -> 0
                }
                // 最后设置answer的值让对手根据我们的出拳来“演戏”
                fAnswer.set(param.thisObject, win)
            }
        })
}


```

重新安装运行，完美！现在对手会配合我们出拳来故意让我们取得胜利！从此，你就是赌圣！

### 另一种方式

Xposed 其实还提供了另一种方式进行 Hook，就是使用`XposedBridge.hookMethod`。  
其实基本上没有太大区别，只不过方法是你自己寻找罢了，举个例子：  
`Java:`

```
private void hook(XC_LoadPackage.LoadPackageParam lpparam) {
    Class<?> clz = XposedHelpers.findClass("me.kyuubiran.xposedapp.MainActivity", lpparam.classLoader);
    for (Method m : clz.getDeclaredMethods()) {
        if (m.getName().equals("onCreate")) {
            XposedBridge.hookMethod(m, new XC_MethodHook() {
                @Override
                protected void afterHookedMethod(MethodHookParam param) {
                    Toast.makeText((Activity) param.thisObject, "我又Hook了MainActivity的onCreate方法！", Toast.LENGTH_SHORT).show();
                }
            });
        }
    }
}


```

`Kotlin:`

```
private fun hook(lpparam: XC_LoadPackage.LoadPackageParam) {
    val clz = XposedHelpers.findClass("me.kyuubiran.xposedapp.MainActivity", lpparam.classLoader)
    for (m in clz.declaredMethods) {
        if (m.name == "onCreate") {
            XposedBridge.hookMethod(m,object : XC_MethodHook() {
                override fun afterHookedMethod(param: MethodHookParam) {
                    Toast.makeText(param.thisObject as Activity,"我又Hook了MainActivity的onCreate方法！",Toast.LENGTH_SHORT).show()
                }
            })
        }
    }
}


```

结束
--

以上便是 Xposed 入门教程了，应该是目前最友好的教程了，该讲的基本都讲了（虽然废话可能有点多）剩下就看造化了（

Q & A
-----

**Q1:** 什么时候用 beforeHook，什么时候用 afterHook，它们的区别是什么？  
**A1:** 你可以把 before 当成方法第一行执行之前，after 当成方法最后一行执行完毕准备返回。如果你要修改参数，那就用 before，如果你要修改返回值，那就用 after，如果你不需要方法执行直接结束方法，那就用 before，当然他俩是可以同时使用的，看具体情况。综合上述，基本用 before 就行了。