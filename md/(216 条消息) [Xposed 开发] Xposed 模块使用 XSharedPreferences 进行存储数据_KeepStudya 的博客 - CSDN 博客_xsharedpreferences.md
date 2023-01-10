> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/niubitianping/article/details/117674003)

一、简介
====

正常在编写模块的时候，我们想给模块增加一些开关，例如 模块是否开启 这个功能，这时候我们就得需要将开关状态给保存下来，然后在 hook 的时候拿到这个状态判断是否执行下去。 数据的保存方法有多种，这次是介绍`XSharedPreferences`（以下简称 xsp）。

正常使用 Android 开发的时候我们使用`SharedPreferences`(以下简称 sp)，sp 的工作流程是在对应 app 的内部数据目录下创建以 map 键值对为内容的 xml 文件，一般路径为`data/data/app包名/shared_prefs/包名.xml`，于是这里就有个问题，这个目录和文件只能 app 自己能访问，其他 app 不能直接访问，然后这里还会有个坑，详情看第三点。

但是得益于我们是 Xposed 模块的原因，在 hook 方法时候利用`SELinuxHelper.getAppDataFileService()`获取 AppData 的文件服务功能，然后利用`getFileInputStream`方法即可获取对应 app 的文件的文件流。  
源码如下, 在 xsp 里面的`loadFromDiskLocked`方法里面：  
![](https://img-blog.csdnimg.cn/20210608101744636.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L25pdWJpdGlhbnBpbmc=,size_16,color_FFFFFF,t_70#pic_center)

而且这整个流程，Xposed 开发者 robv 大佬已经帮我们封装好了 xsp，在 xposed 的 api 的 jar 包里面的包名路径为：`de.robv.android.xposed.XSharedPreferences`。

在模块里面读取 sp 数据的时候就可以用这个类。 读取的用法基本和 sp 一模一样，里面是一个 map 类型的键值对数据。

因为创建 sp 文件的时候是创建到模块 app 里面的数据目录，所以，创建写入的时候就在模块 app 里面进行正常的 sp 读取写入。

二、用法
====

又上面的介绍得知，用法分两种：

1.  模块的 hook 方法里面使用
2.  模块 app 里面使用

2.1 Hook 里使用
------------

#### 2.1.1 初始化

在`handleLoadPackage`里面判断了包名之后，进行 new 实例化一个`XSharedPreferences`，参数传入需要起的 xml 文件名的名字。创建出来的 xml 文件的路径为：`data/data/<packageName>/shared_prefs/<packageName>_preferences.xml`。 以下以微信为例：

```
public class TestHook implements IXposedHookLoadPackage {
    
    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) throws Throwable {
    
       if(lpparam.packageName.equals("com.tencent.mm")){
          //com.skyhand.testmm为当前模块的包名。可以利用BuildConfig.APPLICATION_ID获取当前包名(需要build之后才有)
           XSharedPreferences xsp = new XSharedPreferences("com.skyhand.testmm");
           
       }
       
    }
}
```

### 2.1.2 读取数据

读取数据和和 sp 一模一样，第一个参数为 key，第二个参数为默认值

**基本类型：**

```
XSharedPreferences xsp = new XSharedPreferences(packageName);
        //获取布尔类型值
        boolean boolV = xsp.getBoolean("key",false);
        //获取整数类型值
        int intV = xsp.getInt("key",0);
        //获取浮点数类型值
        float floatV = xsp.getFloat("key",0f);
        //获取长整形类型值
        long longV = xsp.getLong("key",0L);
        //获取字符串类值
        String stringV = xsp.getString("key","");
```

**特殊类型：**

```
//获取Set<String>类型值
        Set<String> defSet = new HashSet<String>();
        Set<String> setV = xsp.getStringSet("key", defSet);
        //获取当前xml的File
        File fileV = xsp.getFile();
        //获取当前xml的全部值，Map类型
        Map<String,?> mapV = xsp.getAll();
```

### 2.1.3 封装用法

上面演示 api 的使用，但是一般是封装成一个静态类来使用，如下的`XSPUtils`。  
更深一层的时候可以封装成枚举，这里就不演示了。

```
public class XSPUtils {
    
    public static XSharedPreferences xsp;

    /**
     * 初始化，在handleLoadPackage里面使用
     * @param packageName 包名 
     */
    public static void initXSP(String packageName) {
        XSPUtils.xsp = new XSharedPreferences(packageName);
    }

    public static boolean getBoolean(String key,boolean def){
        return xsp.getBoolean(key,def);
    }

    public static String getString(String key,String def){
        return xsp.getString(key,def);
    }

    public static int getInt(String key,int def){
        return xsp.getInt(key,def);
    }
    public static float getFloat(String key,float def){
        return xsp.getFloat(key,def);
    }
    public static long getLong(String key,long def){
        return xsp.getLong(key,def);
    }
}
```

初始化的时候就在`handleLoadPackage`hook 方法里面：

```
public class TestHook implements IXposedHookLoadPackage {

    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam lpparam) throws Throwable {

       if(lpparam.packageName.equals("com.tencent.mm")){

           //初始化xsp，com.skyhand.testmm为当前模块的包名
           XSPUtils.initXSP("com.skyhand.testmm");
            
           //获取一个数据
           boolean isOpen = XSPUtils.getBoolean("open",false);
       }

    }
}
```

2.2 模块 App 里面使用
---------------

在模块的 App 里面使用就是参考正常的 sp 用法，因为可以正常读取自己 data 目录的文件。

这里给个简单的例子，例如封装了一个`SPUtils.java`：

```
package com.store;

import android.content.Context;
import android.content.SharedPreferences;


public class SPUtils {


    public static SPUtils xsp;
    public SharedPreferences sp;

    private SPUtils() {

    }

    public static synchronized SPUtils getInstance() {
        if (xsp == null) {
            xsp = new SPUtils();
        }
        return xsp;
    }

    /**
     * 初始化
     * @param context
     */
    public void init(Context context) {
        sp = context.getSharedPreferences(context.getPackageName(), Context.MODE_PRIVATE);
    }

    /**
     * 下面的是读取数据
     * @param key
     * @param def
     * @return
     */
    public static String getString(String key, String def) {
        return SPUtils.getInstance().sp.getString(key, def);
    }

    public static int getInt(String key, int def) {
        return SPUtils.getInstance().sp.getInt(key, def);
    }

    public static float getFloat(String key, float def) {
        return SPUtils.getInstance().sp.getFloat(key, def);
    }

    public static long getLong(String key, long def) {
        return SPUtils.getInstance().sp.getLong(key, def);
    }
    public static boolean getBoolean(String key, boolean def) {
        return SPUtils.getInstance().sp.getBoolean(key, def);
    }

    /**
     * 下面是保存数据
     * @param key
     * @param v
     * @return
     */
    public static boolean setString(String key, String v) {
        return SPUtils.getInstance().sp.edit().putString(key, v).commit();
    }

    public static boolean setInt(String key, int v) {
        return SPUtils.getInstance().sp.edit().putInt(key, v).commit();
    }

    public static boolean setBoolean(String key, boolean v) {
        return SPUtils.getInstance().sp.edit().putBoolean(key, v).commit();
    }
    public static boolean setFloat(String key, float v) {
        return SPUtils.getInstance().sp.edit().putFloat(key, v).commit();
    }

    public static boolean setLong(String key, long v) {
        return SPUtils.getInstance().sp.edit().putLong(key, v).commit();
    }


}
```

然后在模块 app 里面使用的时候大概这样子：

```
public class TestMainActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        setContentView(R.layout.activity_main);

        //初始化Sp
        SPUtils.getInstance().init(this);

        Switch switchView = (Switch) findViewById(R.id.SwOpen);
        switchView.setOnCheckedChangeListener(new CompoundButton.OnCheckedChangeListener() {
            @Override
            public void onCheckedChanged(CompoundButton buttonView, boolean isChecked) {
                //保存到sp
                SPUtils.setBoolean("open", isChecked);
            }
        });
    }


}
```

三、坑
===

使用 Xsp 的情况，targetSdkVersion 必须设置 27 或者 27 以下，还需要在主 app 里面设置可读写, 例如 SettingsActivity 在 onPause 的时候可以可读写

```
class SettingsActivity : AppCompatActivity() {
    /**
     * 重点设置：xml文件可读写，不然模块里面的Xsp获取不到
     */
    override fun onPause() {
        super.onPause()
        LogH.log("设置本地xp文件可读写")
        Xsp.setPreferenceWorldWritable(this)
    }
}
```

Xsp 里面的方法：

```
/**
     * 设置sp文件可读写
     */
    @SuppressLint("SetWorldWritable", "SetWorldReadable")
    fun setPreferenceWorldWritable(context: Context, name: String = "") {
        var spName = name;
        if (spName == "") {
            spName = context.packageName + "_preferences"
        }
        var file: File? = getSharedPreferencesFile(context, spName)
        if (file != null && !file.exists()) return
        for (i in 0..2) {
            file?.setExecutable(true, false)
            file?.setWritable(true, false)
            file?.setReadable(true, false)
            file = file?.parentFile
            if (file == null) break
        }
    } 

    private fun getSharedPreferencesFile(context: Context, name: String): File {
        val dataDir = ContextCompat.getDataDir(context)
        val prefsDir = File(dataDir, "shared_prefs")
        return File(prefsDir, "$name.xml")
    }
```

四、总结
====

模块使用 xsp 进行数据存储，hook 的情况和模块自身使用的情况分开处理，hook 时候使用 xsp，模块自身正常使用 sp，就是以下的对应：

1.  xsp： hook 里面使用
2.  sp: 模块 app 里面使用

因为：xsp 和 sp 他们操作的都是同一个 sp 的 xml 文件。

五、拓展
====

在模块的 app 里面使用 xsp 和 sp 的时候，很多开发者喜欢使用`PreferenceFragment`或者`PreferenceActivity`，然后把 Key 写成枚举 enum，让 xsp 和 sp 同时使用。

这里就不给例子了，作为作业，自己查询其它资料。