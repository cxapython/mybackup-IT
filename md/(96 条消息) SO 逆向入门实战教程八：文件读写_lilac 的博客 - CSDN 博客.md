> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38851536/article/details/118024298?spm=1001.2014.3001.5502)

### 文章目录

*   *   *   [一、前言](#_2)
        *   [二、demo1 设计](#demo1_10)
        *   [三、Unidbg 模拟执行 demo1](#Unidbgdemo1_223)
        *   [四、demo2 设计](#demo2_507)
        *   [五、Unidbg 模拟执行 demo2](#Unidbgdemo2_892)
        *   [六、尾声](#_1094)

### 一、前言

本篇分析的是自写 demo，帮助大家熟悉 Unidbg 中对文件读写的处理，例子中主要涉及

*   Sharedpreference 读写
*   Assets 读写
*   文件 读写

### 二、demo1 设计

> Sharedpreferences 是 Android 平台上常用的存储方式，用来保存应用程序的各种配置信息，其本质是一个以 “键 - 值” 对的方式保存数据的 xml 文件，其文件保存在 / data/data/selfPackage/shared_prefs 目录下。

在 APP 刚启动时，我们新建两个 Sharedpreference 文件，分别填入两个键值对，name-lilac 和 location-china，代码实现

（如果对开发不感兴趣，也可以直接去百度网盘中获取 demo 文件的 apk 进行分析）

```
package com.roysue.readsp;

import androidx.appcompat.app.AppCompatActivity;

import android.content.Context;
import android.content.SharedPreferences;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {

    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        //步骤1：创建SharedPreferences对象
        SharedPreferences sharedPreferences1= getSharedPreferences("one", Context.MODE_PRIVATE);
        SharedPreferences sharedPreferences2= getSharedPreferences("two", Context.MODE_PRIVATE);
        //步骤2： 实例化SharedPreferences.Editor对象
        SharedPreferences.Editor editor1 = sharedPreferences1.edit();
        SharedPreferences.Editor editor2 = sharedPreferences2.edit();
        //步骤3：将获取过来的值放入文件
        editor1.putString("name", "lilac");
        editor2.putString("location", "china");
        //步骤4：提交
        editor1.apply();
        editor2.apply();


        TextView tv = findViewById(R.id.sample_text);
        Button btn = findViewById(R.id.button);
        btn.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                tv.setText(stringFromJNI(getApplicationContext()));
            }
        });
        
    }

    /**
     * A native method that is implemented by the 'native-lib' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI(Context context);
}
```

在 shared_prefs 下反映为两个如下的 xml

one.xml

```
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string >lilac</string>
</map>
```

two.xml

```
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string >china</string>
</map>
```

在程序逻辑中，我们打算在 stringFromJni 中做一件很简单的事：在 1.xml 中读取 gt_m 的值，在 2.xml 中读取 gt_fp 的值，然后拼接后打印。

但在代码实现上，我们打算绕个弯

在读取 1.xml 的 name 上，我们使用 JNI 去调用 JAVA 层对 SharedPreference 操纵的 API。

```
//反射Context类
    jclass cls_Context = env->FindClass("android/content/Context");
    //反射Context类getSharedPreferences方法
    jmethodID mid_getSharedPreferences = env->GetMethodID(cls_Context,
                                                          "getSharedPreferences",
                                                          "(Ljava/lang/String;I)Landroid/content/SharedPreferences;");
    //获取Context类MODE_PRIVATE属性值
    //执行反射方法
    jobject obj_sharedPreferences = env->CallObjectMethod(mycontext,
                                                          mid_getSharedPreferences, env->NewStringUTF("one"),
                                                          0);

    jclass cls_SharedPreferences = env->FindClass("android/content/SharedPreferences");
    //反射SharedPreferences类的getString方法
    jmethodID mid_getString = env->GetMethodID(cls_SharedPreferences,
                                               "getString",
                                               "(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;");
    //参数类型转换
    jstring key_name = env->NewStringUTF("name");
    //参数类型转换
    jstring default_value = env->NewStringUTF(" ");
    //执行反射方法
    jstring key_value1 = (jstring) env->CallObjectMethod(obj_sharedPreferences,
                                                                  mid_getString, key_name, default_value);

    const char *c_key_value1 =  env->GetStringUTFChars(key_value1, 0);
```

在读取 2.xml 的 location 上，我们换个方式，既然 Sharedpreference 本质上是个 xml 文件，那么用 native 中原生的 open 函数去读可能会更隐蔽，可是 open 本质上也是通过底层系统调用 open 的方式实现，那我们直截了当通过系统调用 open 打开这个 xml 也是一样。（处理时，我懒得切割字符串，所以直接拼接 two.xml 整个 xml 吧。）

完整代码

```
#include <jni.h>
#include <string>
#include <sys/syscall.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/stat.h>

__attribute__((naked))
long raw_syscall(long _number,...){
    __asm__ __volatile__ ("MOV R12,SP\r\n"
                          "STMFD SP!,{R4-R7}\r\n"
                          "MOV R7,R0\r\n"
                          "MOV R0,R1\r\n"
                          "MOV R1,R2\r\n"
                          "MOV R2,R3\r\n"
                          "LDMIA R12,{R3-R6}\r\n"
                          "SVC 0\r\n"
                          "LDMFD SP!,{R4-R7}\r\n"
                          "mov pc,lr");
};

char* test_syscall(const char* file_path){
    char *result = "";
    long fd = raw_syscall(5,file_path, O_RDONLY | O_CREAT, 400);
    if(fd != -1){
        char buffer[0x100] = {0};
        raw_syscall(3, fd, buffer, 0x100);
        result = buffer;
        raw_syscall(6, fd);
    }
    return result;
}


extern "C" JNIEXPORT jstring JNICALL
Java_com_roysue_readsp_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */, jobject mycontext) {

    //反射Context类
    jclass cls_Context = env->FindClass("android/content/Context");
    //反射Context类getSharedPreferences方法
    jmethodID mid_getSharedPreferences = env->GetMethodID(cls_Context,
                                                          "getSharedPreferences",
                                                          "(Ljava/lang/String;I)Landroid/content/SharedPreferences;");
    //获取Context类MODE_PRIVATE属性值
    //执行反射方法
    jobject obj_sharedPreferences = env->CallObjectMethod(mycontext,
                                                          mid_getSharedPreferences, env->NewStringUTF("one"),
                                                          0);

    jclass cls_SharedPreferences = env->FindClass("android/content/SharedPreferences");
    //反射SharedPreferences类的getString方法
    jmethodID mid_getString = env->GetMethodID(cls_SharedPreferences,
                                               "getString",
                                               "(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;");
    //参数类型转换
    jstring key_name = env->NewStringUTF("name");
    //参数类型转换
    jstring default_value = env->NewStringUTF(" ");
    //执行反射方法
    jstring key_value1 = (jstring) env->CallObjectMethod(obj_sharedPreferences,
                                                                  mid_getString, key_name, default_value);
    const char *c_key_value1 =  env->GetStringUTFChars(key_value1, 0);
    const char *path = "/data/data/com.roysue.readsp/shared_prefs/two.xml";
    const char *result = test_syscall(path);

    char dest[1000];

    strcpy(dest,c_key_value1);
    strcat(dest,result);
    return env->NewStringUTF(dest);
}
```

结果即

```
lilac<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <string >china</string>
</map>
```

需要注意的是，因为只是一个 demo，所以有些地方采用了硬编码，自己跑代码的时候，如果无法顺利跑出结果，请检查是否是路径等硬编码与本机环境不符，或者在微信联系我，很荣幸和同侪交流互动。

### 三、Unidbg 模拟执行 demo1

首先想一下我们可能遇到的问题，应该就是两个

*   补 JAVA 环境（one.xml 的读取是通过 JNI 调用 JAVA 层 API 实现的）
*   补文件访问 / 重定向 (two.xml 的读取，最终通过 open 系统调用实现，需要将文件重定向到本机某个位置

快开干

```
package com.lession8;

import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.memory.Memory;

import java.io.File;
import java.io.FileNotFoundException;
import java.util.ArrayList;
import java.util.List;

public class demo1 extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    demo1(){
        // 防止进程名检测
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.readSp").build(); // 创建模拟器实例，要模拟32位或者64位，在这里区分
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析
        vm = emulator.createDalvikVM(null);


        DalvikModule dm = vm.loadLibrary(new File("C:\\Users\\pr0214\\Desktop\\DTA\\unidbg\\versions\\unidbg-2021-5-17\\unidbg-master\\unidbg-android\\src\\test\\java\\com\\lession8\\libnative-lib.so"), true);
        module = dm.getModule();
        
        vm.setJni(this);
        vm.setVerbose(true);
    }


    public String call(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
        Object custom = null;
        DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(custom);// context
        list.add(vm.addLocalObject(context));
        Number number = module.callFunction(emulator, 0xAAC + 1, list.toArray())[0];
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }

    public static void main(String[] args) throws FileNotFoundException {
        demo1 test = new demo1();
        System.out.println(test.call());
    }
}
```

运行

![](https://img-blog.csdnimg.cn/20210618132454781.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

JAVA 环境缺失，在获取 getSharedPreferences，对应于开发中的这一步

![](https://img-blog.csdnimg.cn/20210618132510134.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
方法参数 1 是 Sharedpreference 的名字，参数 2 默认为 0 即可，返回 SharedPreferences 对象。

我们补一个空的 SharedPreferences 返回

```
@Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "android/content/Context->getSharedPreferences(Ljava/lang/String;I)Landroid/content/SharedPreferences;":
                return vm.resolveClass("android/content/SharedPreferences").newObject(null);
        }
        
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }
```

但我们需要思考一下，这样真的好吗？

假设 APP 在 JNI 中从五个 SharedPreferences 里读了十五个键值对，并且不同 xml 的键名有重复，如果每次取 SharedPreferences 时我们都返回空对象，那后面怎么区分 a.xml 和 b.xml 里键名都是 name 的数据呢？

先前我们说，参数 1 是想要获取的 SharedPreferences 的名字，应该把它放对象里返回，这样就有了” 标识性 “

```
@Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "android/content/Context->getSharedPreferences(Ljava/lang/String;I)Landroid/content/SharedPreferences;":
                return vm.resolveClass("android/content/SharedPreferences").newObject(vaList.getObject(0));
        }

        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }
```

继续运行

![](https://img-blog.csdnimg.cn/20210618132525627.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

显然，这儿就是在获取 one.xml 的 name 所对应的值，我们应该返回 lilac

```
@Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "android/content/Context->getSharedPreferences(Ljava/lang/String;I)Landroid/content/SharedPreferences;":
                return vm.resolveClass("android/content/SharedPreferences").newObject(vaList.getObject(0));
            case "android/content/SharedPreferences->getString(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;":
                return new StringObject(vm, "lilac");
        }

        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }
```

这样做固然可以，但能不能更严谨规范一些呢？当然可以

```
@Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "android/content/Context->getSharedPreferences(Ljava/lang/String;I)Landroid/content/SharedPreferences;":
                return vm.resolveClass("android/content/SharedPreferences").newObject(vaList.getObject(0));
            case "android/content/SharedPreferences->getString(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;": {
                // 如果是one.xml
                if(((StringObject) dvmObject.getValue()).getValue().equals("one")){
                    // 如果键是name
                    if (vaList.getObject(0).getValue().equals("name")) {
                        return new StringObject(vm, "lilac");
                    }
                }
            }
        }

        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }
```

接着运行

![](https://img-blog.csdnimg.cn/20210618132535497.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

显然，对 one.xml 的读取已经顺利完成了，可是千万别忘了，two.xml 呢？

为什么没有做拼接？或者说，为什么 Unidbg 没有提醒我们对文件进行重定向？two.xml 我们还没操作呢！

这是因为我们对 two.xml 的操作采用系统调用的方式完成，但我们没有打开 Unidbg 中系统调用的日志显示。

如下打开对 arm32 中系统调用的日志显示

![](https://img-blog.csdnimg.cn/20210618132544212.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
再次运行  
![](https://img-blog.csdnimg.cn/20210618132557819.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
事实上，我建议运行真实样本时，开启所有的日志以防错过重要的环境缺失。

```
Logger.getLogger("com.github.unidbg.linux.ARM32SyscallHandler").setLevel(Level.DEBUG);
Logger.getLogger("com.github.unidbg.unix.UnixSyscallHandler").setLevel(Level.DEBUG);
Logger.getLogger("com.github.unidbg.AbstractEmulator").setLevel(Level.DEBUG);
Logger.getLogger("com.github.unidbg.linux.android.dvm.DalvikVM").setLevel(Level.DEBUG);
Logger.getLogger("com.github.unidbg.linux.android.dvm.BaseVM").setLevel(Level.DEBUG);
Logger.getLogger("com.github.unidbg.linux.android.dvm").setLevel(Level.DEBUG);
```

上篇中，我们讲了两种方式补文件访问

1 是 Unidbg 提供的 rootfs 虚拟文件系统，2 是代码方式文件重定向 大家自行选择，有人可能会问，如果我不想传入文件，能不能只传入” 字符串 “，当然可以，从 SimpleFileIO 换成 ByteArrayFileIO 即可。

```
@Override
    public FileResult resolve(Emulator emulator, String pathname, int oflags) {
        if ("/data/data/com.roysue.readsp/shared_prefs/two.xml".equals(pathname)) {
            return FileResult.success(new ByteArrayFileIO(oflags, pathname, "mytest".getBytes()));
        }
        return null;
    }
```

看一下完整代码

```
package com.lession8;

import com.github.unidbg.Emulator;
import com.github.unidbg.file.FileResult;
import com.github.unidbg.file.IOResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.file.ByteArrayFileIO;
import com.github.unidbg.memory.Memory;
import org.apache.log4j.Level;
import org.apache.log4j.Logger;

import java.io.File;
import java.io.FileNotFoundException;

import java.util.*;

public class demo1 extends AbstractJni implements IOResolver {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    demo1(){
        // 防止进程名检测
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.readSp").build(); // 创建模拟器实例，要模拟32位或者64位，在这里区分
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析
        vm = emulator.createDalvikVM(null);

        emulator.getSyscallHandler().addIOResolver(this);

        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\lession8\\libnative-lib.so"), true);
        module = dm.getModule();

        vm.setJni(this);
        vm.setVerbose(true);
    }


    public String call(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
        Object custom = null;
        DvmObject<?> context = vm.resolveClass("android/content/Context").newObject(custom);// context
        list.add(vm.addLocalObject(context));
        Number number = module.callFunction(emulator, 0xAAC + 1, list.toArray())[0];
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }

    @Override
    public DvmObject<?> callObjectMethodV(BaseVM vm, DvmObject<?> dvmObject, String signature, VaList vaList) {
        switch (signature) {
            case "android/content/Context->getSharedPreferences(Ljava/lang/String;I)Landroid/content/SharedPreferences;":
                return vm.resolveClass("android/content/SharedPreferences").newObject(vaList.getObject(0));
            case "android/content/SharedPreferences->getString(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;": {
                // 如果是one.xml
                if(((StringObject) dvmObject.getValue()).getValue().equals("one")){
                    // 如果键是name
                    if (vaList.getObject(0).getValue().equals("name")) {
                        return new StringObject(vm, "lilac");
                    }
                }
            }
        }

        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }

    public static void main(String[] args) throws FileNotFoundException {
        Logger.getLogger("com.github.unidbg.linux.ARM32SyscallHandler").setLevel(Level.DEBUG);
        demo1 test = new demo1();
        System.out.println(test.call());
    }

    @Override
    public FileResult resolve(Emulator emulator, String pathname, int oflags) {
        if ("/data/data/com.roysue.readsp/shared_prefs/two.xml".equals(pathname)) {
            return FileResult.success(new ByteArrayFileIO(oflags, pathname, "mytest".getBytes()));
        }
        return null;
    }
}
```

![](https://img-blog.csdnimg.cn/20210618132625361.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

### 四、demo2 设计

各类加密算法大部分都有密钥的存在，非对称加密算法还有公钥私钥之分，所以加密算法运算时需要传入密钥，但是直接参数方式传递很容易被分析者意识到这是密钥，有没有办法更隐蔽一些呢？

比如我们把密钥放在 xml 中，在 native 中读取它，就类似于 demo1

有没有更隐蔽一些些的呢，我们可以把密钥藏在资源文件的图片里，即在 so 里读取资源文件里的某张图片，以它的某部分或者整体的 md5 结果作为密钥？这是一个更好的方案。

demo2 就是这个方案的简单实现——native 中读取资源文件的 1.jpg，并求其 md5 值，返回 JAVA 层。

看一下代码

JAVA 层

```
package com.roysue.readasset;

import androidx.appcompat.app.AppCompatActivity;

import android.content.res.AssetManager;
import android.os.Bundle;
import android.widget.TextView;

public class MainActivity extends AppCompatActivity {

    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Example of a call to a native method
        TextView tv = findViewById(R.id.sample_text);
        tv.setText(setNativeAssetManager(getAssets()));
    }

    public native String setNativeAssetManager(AssetManager assetManager);

}
```

Native 层

```
#include <jni.h>
#include <string>
#include "android/asset_manager.h"
#include "android/asset_manager_jni.h"

#define MD5_LONG unsigned long
// 分组大小
#define MD5_CBLOCK	64
// 分块个数
#define MD5_LBLOCK	(MD5_CBLOCK/4)
// 摘要长度（字节）
#define MD5_DIGEST_LENGTH 16

#define MD32_REG_T long
// 小端序
#define DATA_ORDER_IS_LITTLE_ENDIAN

// 四个初始化常量
#define INIT_DATA_A (unsigned long)0x67452301L
#define INIT_DATA_B (unsigned long)0xefcdab89L
#define INIT_DATA_C (unsigned long)0x98badcfeL
#define INIT_DATA_D (unsigned long)0x10325476L

// 循环左移以及取低64位
#define ROTATE(a,n)     (((a)<<(n))|(((a)&0xffffffff)>>(32-(n))))

// 大小端序互转
#define HOST_c2l(c,l)	(l =(((unsigned long)(*((c)++)))    ),		\
			 l|=(((unsigned long)(*((c)++)))<< 8),		\
			 l|=(((unsigned long)(*((c)++)))<<16),		\
			 l|=(((unsigned long)(*((c)++)))<<24)		)

#define HOST_l2c(l,c)	(*((c)++)=(unsigned char)(((l)    )&0xff),	\
			 *((c)++)=(unsigned char)(((l)>> 8)&0xff),	\
			 *((c)++)=(unsigned char)(((l)>>16)&0xff),	\
			 *((c)++)=(unsigned char)(((l)>>24)&0xff),	\
			 l)

// 更新链接变量值
#define	HASH_MAKE_STRING(c,s)	do {	\
	unsigned long ll;		\
	ll=(c)->A; (void)HOST_l2c(ll,(s));	\
	ll=(c)->B; (void)HOST_l2c(ll,(s));	\
	ll=(c)->C; (void)HOST_l2c(ll,(s));	\
	ll=(c)->D; (void)HOST_l2c(ll,(s));	\
	} while (0)

// 四个初始化非线性函数，或者叫逻辑函数
#define	F(b,c,d)	((((c) ^ (d)) & (b)) ^ (d))
#define	G(b,c,d)	((((b) ^ (c)) & (d)) ^ (c))
#define	H(b,c,d)	((b) ^ (c) ^ (d))
#define	I(b,c,d)	(((~(d)) | (b)) ^ (c))

// F函数，每隔16步/轮 换下一个
#define R0(a,b,c,d,k,s,t) { \
	a+=((k)+(t)+F((b),(c),(d))); \
	a=ROTATE(a,s); \
	a+=b; };

#define R1(a,b,c,d,k,s,t) { \
	a+=((k)+(t)+G((b),(c),(d))); \
	a=ROTATE(a,s); \
	a+=b;};

#define R2(a,b,c,d,k,s,t) { \
	a+=((k)+(t)+H((b),(c),(d))); \
	a=ROTATE(a,s); \
	a+=b; };

#define R3(a,b,c,d,k,s,t) { \
	a+=((k)+(t)+I((b),(c),(d))); \
	a=ROTATE(a,s); \
	a+=b; };

typedef struct MD5state_st1{
    MD5_LONG A,B,C,D; // ABCD
    MD5_LONG Nl,Nh; // 数据的bit数计数器(对2^64取余)，Nh存储高32位，Nl存储低32位。这种设计是服务于32位处理器，MD5的设计就是为了服务于32位处理器的。
    MD5_LONG data[MD5_LBLOCK];//数据缓冲区
    unsigned int num;
}MD5_CTX; // 存放MD5算法相关信息的结构体定义

unsigned char cleanse_ctr = 0;


// 初始化链接变量/幻数
int MD5_Init(MD5_CTX *c){
    memset (c,0,sizeof(*c));
    c->A=INIT_DATA_A;
    c->B=INIT_DATA_B;
    c->C=INIT_DATA_C;
    c->D=INIT_DATA_D;
    return 1;
}

// md5 一个分组中的全部运算
void md5_block_data_order(MD5_CTX *c, const void *data_, unsigned int num){
    const unsigned char *data= static_cast<const unsigned char *>(data_);
    register unsigned MD32_REG_T A,B,C,D,l;
#ifndef MD32_XARRAY
    unsigned MD32_REG_T	XX0, XX1, XX2, XX3, XX4, XX5, XX6, XX7,
            XX8, XX9,XX10,XX11,XX12,XX13,XX14,XX15;
# define X(i)	XX##i
#else
    MD5_LONG XX[MD5_LBLOCK];
# define X(i)	XX[i]
#endif
    A=c->A;
    B=c->B;
    C=c->C;
    D=c->D;
    // 64轮
    // 前16轮需要改变分组中每个分块的
    for (;num--;){
        HOST_c2l(data,l); X( 0)=l;		HOST_c2l(data,l); X( 1)=l;
        /* Round 0 */
        R0(A,B,C,D,X( 0), 7,0xd76aa478L);	HOST_c2l(data,l); X( 2)=l;
        R0(D,A,B,C,X( 1),12,0xe8c7b756L);	HOST_c2l(data,l); X( 3)=l;
        R0(C,D,A,B,X( 2),17,0x242070dbL);	HOST_c2l(data,l); X( 4)=l;
        R0(B,C,D,A,X( 3),22,0xc1bdceeeL);	HOST_c2l(data,l); X( 5)=l;
        R0(A,B,C,D,X( 4), 7,0xf57c0fafL);	HOST_c2l(data,l); X( 6)=l;
        R0(D,A,B,C,X( 5),12,0x4787c62aL);	HOST_c2l(data,l); X( 7)=l;
        R0(C,D,A,B,X( 6),17,0xa8304613L);	HOST_c2l(data,l); X( 8)=l;
        R0(B,C,D,A,X( 7),22,0xfd469501L);	HOST_c2l(data,l); X( 9)=l;
        R0(A,B,C,D,X( 8), 7,0x698098d8L);	HOST_c2l(data,l); X(10)=l;
        R0(D,A,B,C,X( 9),12,0x8b44f7afL);	HOST_c2l(data,l); X(11)=l;
        R0(C,D,A,B,X(10),17,0xffff5bb1L);	HOST_c2l(data,l); X(12)=l;
        R0(B,C,D,A,X(11),22,0x895cd7beL);	HOST_c2l(data,l); X(13)=l;
        R0(A,B,C,D,X(12), 7,0x6b901122L);	HOST_c2l(data,l); X(14)=l;
        R0(D,A,B,C,X(13),12,0xfd987193L);	HOST_c2l(data,l); X(15)=l;
        R0(C,D,A,B,X(14),17,0xa679438eL);
        R0(B,C,D,A,X(15),22,0x49b40821L);
        /* Round 1 */
        R1(A,B,C,D,X( 1), 5,0xf61e2562L);
        R1(D,A,B,C,X( 6), 9,0xc040b340L);
        R1(C,D,A,B,X(11),14,0x265e5a51L);
        R1(B,C,D,A,X( 0),20,0xe9b6c7aaL);
        R1(A,B,C,D,X( 5), 5,0xd62f105dL);
        R1(D,A,B,C,X(10), 9,0x02441453L);
        R1(C,D,A,B,X(15),14,0xd8a1e681L);
        R1(B,C,D,A,X( 4),20,0xe7d3fbc8L);
        R1(A,B,C,D,X( 9), 5,0x21e1cde6L);
        R1(D,A,B,C,X(14), 9,0xc33707d6L);
        R1(C,D,A,B,X( 3),14,0xf4d50d87L);
        R1(B,C,D,A,X( 8),20,0x455a14edL);
        R1(A,B,C,D,X(13), 5,0xa9e3e905L);
        R1(D,A,B,C,X( 2), 9,0xfcefa3f8L);
        R1(C,D,A,B,X( 7),14,0x676f02d9L);
        R1(B,C,D,A,X(12),20,0x8d2a4c8aL);
        /* Round 2 */
        R2(A,B,C,D,X( 5), 4,0xfffa3942L);
        R2(D,A,B,C,X( 8),11,0x8771f681L);
        R2(C,D,A,B,X(11),16,0x6d9d6122L);
        R2(B,C,D,A,X(14),23,0xfde5380cL);
        R2(A,B,C,D,X( 1), 4,0xa4beea44L);
        R2(D,A,B,C,X( 4),11,0x4bdecfa9L);
        R2(C,D,A,B,X( 7),16,0xf6bb4b60L);
        R2(B,C,D,A,X(10),23,0xbebfbc70L);
        R2(A,B,C,D,X(13), 4,0x289b7ec6L);
        R2(D,A,B,C,X( 0),11,0xeaa127faL);
        R2(C,D,A,B,X( 3),16,0xd4ef3085L);
        R2(B,C,D,A,X( 6),23,0x04881d05L);
        R2(A,B,C,D,X( 9), 4,0xd9d4d039L);
        R2(D,A,B,C,X(12),11,0xe6db99e5L);
        R2(C,D,A,B,X(15),16,0x1fa27cf8L);
        R2(B,C,D,A,X( 2),23,0xc4ac5665L);
        /* Round 3 */
        R3(A,B,C,D,X( 0), 6,0xf4292244L);
        R3(D,A,B,C,X( 7),10,0x432aff97L);
        R3(C,D,A,B,X(14),15,0xab9423a7L);
        R3(B,C,D,A,X( 5),21,0xfc93a039L);
        R3(A,B,C,D,X(12), 6,0x655b59c3L);
        R3(D,A,B,C,X( 3),10,0x8f0ccc92L);
        R3(C,D,A,B,X(10),15,0xffeff47dL);
        R3(B,C,D,A,X( 1),21,0x85845dd1L);
        R3(A,B,C,D,X( 8), 6,0x6fa87e4fL);
        R3(D,A,B,C,X(15),10,0xfe2ce6e0L);
        R3(C,D,A,B,X( 6),15,0xa3014314L);
        R3(B,C,D,A,X(13),21,0x4e0811a1L);
        R3(A,B,C,D,X( 4), 6,0xf7537e82L);
        R3(D,A,B,C,X(11),10,0xbd3af235L);
        R3(C,D,A,B,X( 2),15,0x2ad7d2bbL);
        R3(B,C,D,A,X( 9),21,0xeb86d391L);

        A = c->A += A;
        B = c->B += B;
        C = c->C += C;
        D = c->D += D;
    }
}

// 传入需要哈希的明文,支持多次调用
int MD5_Update(MD5_CTX *c, const void *data_, size_t len){
    const unsigned char *data= static_cast<const unsigned char *>(data_);
    unsigned char *p;
    MD5_LONG l;
    size_t n;

    if (len==0) return 1;
    // 低位
    l=(c->Nl+(((MD5_LONG)len)<<3))&0xffffffffUL;
    if (l < c->Nl)
        c->Nh++;
    // 高位
    c->Nh+=(MD5_LONG)(len>>29);
    c->Nl=l;

    n = c->num;
    if (n != 0){
        p=(unsigned char *)c->data;

        if (len >= MD5_CBLOCK || len+n >= MD5_CBLOCK){
            memcpy (p+n,data,MD5_CBLOCK-n);
            md5_block_data_order(c,p,1);
            n      = MD5_CBLOCK-n;
            data  += n;
            len   -= n;
            c->num = 0;
            memset (p,0,MD5_CBLOCK);
        }else{
            memcpy (p+n,data,len);
            c->num += (unsigned int)len;
            return 1;
        }
    }

    n = len/MD5_CBLOCK;
    if (n > 0){
        md5_block_data_order(c,data,n);
        n    *= MD5_CBLOCK;
        data += n;
        len  -= n;
    }

    if (len != 0){
        p = (unsigned char *)c->data;
        c->num = (unsigned int)len;
        memcpy (p,data,len);
    }
    return 1;
}

// 得出最终结果
int MD5_Final(unsigned char *md, MD5_CTX *c){
    unsigned char *p = (unsigned char *)c->data;
    size_t n = c->num;

    p[n] = 0x80; /* there is always room for one */
    n++;

    if (n > (MD5_CBLOCK-8)){
        memset (p+n,0,MD5_CBLOCK-n);
        n=0;
        md5_block_data_order(c,p,1);
    }
    memset (p+n,0,MD5_CBLOCK-8-n);

    p += MD5_CBLOCK-8;
#if   defined(DATA_ORDER_IS_BIG_ENDIAN)
    (void)HOST_l2c(c->Nh,p);
	(void)HOST_l2c(c->Nl,p);
#elif defined(DATA_ORDER_IS_LITTLE_ENDIAN)
    (void)HOST_l2c(c->Nl,p);
    (void)HOST_l2c(c->Nh,p);
#endif
    p -= MD5_CBLOCK;
    md5_block_data_order(c,p,1);
    c->num=0;
    memset (p,0,MD5_CBLOCK);

#ifndef HASH_MAKE_STRING
#error "HASH_MAKE_STRING must be defined!"
#else
    HASH_MAKE_STRING(c,md);
#endif

    return 1;
}

//清除加载的各种算法，包括对称算法、摘要算法以及 PBE 算法，并清除这些算法相关的哈希表的内容。
void OPENSSL_cleanse(void *ptr, size_t len){
    unsigned char *p = static_cast<unsigned char *>(ptr);
    size_t loop = len, ctr = cleanse_ctr;
    while(loop--){
        *(p++) = (unsigned char)ctr;
        ctr += (17 + ((size_t)p & 0xF));
    }
    p= static_cast<unsigned char *>(memchr(ptr, (unsigned char) ctr, len));
    if(p)
        ctr += (63 + (size_t)p);
    cleanse_ctr = (unsigned char)ctr;
}


// 计算图片的md5值并返回
extern "C"
JNIEXPORT jstring JNICALL
Java_com_roysue_readasset_MainActivity_setNativeAssetManager(JNIEnv *env, jobject thiz,jobject asset_manager) {
    AAssetManager *nativeasset = AAssetManager_fromJava(env, asset_manager);
    
    AAsset *assetFile = AAssetManager_open(nativeasset, "1.png", AASSET_MODE_BUFFER);
    size_t fileLength = AAsset_getLength(assetFile);
    char *dataBuffer = (char *) malloc(fileLength);
    //read file data
    AAsset_read(assetFile, dataBuffer, fileLength);
    //the data has been copied to dataBuffer2, so , close it
    AAsset_close(assetFile);

    // 初始化MD5的上下文结构体
    MD5_CTX context = {0};
    MD5_Init(&context);

    // 传入待处理的内容以及内容的长度
    MD5_Update(&context, dataBuffer, fileLength);

    // 收尾和输出
    // 输出的缓冲区
    unsigned char dest[16] = {0};
    MD5_Final(dest, &context);


    // 结果转成十六进制字符串
    int i = 0;
    char szMd5[33] = {0};
    for(i=0; i<16; i++){
        sprintf(szMd5, "%s%02x", szMd5, dest[i]);
    }

    //free malloc
    free(dataBuffer);
    // 传回Java世界
    return env->NewStringUTF(szMd5);
}
```

运行结果

![](https://img-blog.csdnimg.cn/20210618132838300.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)

### 五、Unidbg 模拟执行 demo2

模拟执行 demo2 会出现什么问题呢？  
先看一下 IDA F5 效果  
![](https://img-blog.csdnimg.cn/20210618132924964.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
再写一下 Unidbg 代码

```
package com.lession8;

import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.memory.Memory;
import org.apache.log4j.Level;
import org.apache.log4j.Logger;

import java.io.File;
import java.io.FileNotFoundException;
import java.util.ArrayList;
import java.util.List;

public class demo2 extends AbstractJni {
    private final AndroidEmulator emulator;
    private final VM vm;
    private final Module module;

    demo2(){
        emulator = AndroidEmulatorBuilder.for32Bit().setProcessName("com.readAssets").build(); // 创建模拟器实例，要模拟32位或者64位，在这里区分
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析
        vm = emulator.createDalvikVM(new File("unidbg-android\\src\\test\\java\\com\\lession8\\demo2.apk"));

        DalvikModule dm = vm.loadLibrary(new File("unidbg-android\\src\\test\\java\\com\\lession8\\libnative-lib.so"), true);
        module = dm.getModule();

        vm.setJni(this);
        vm.setVerbose(true);
    }


    public String call(){
        List<Object> list = new ArrayList<>(10);
        list.add(vm.getJNIEnv()); // 第一个参数是env
        list.add(0); // 第二个参数，实例方法是jobject，静态方法是jclazz，直接填0，一般用不到。
        Object custom = null;
        DvmObject<?> assetManager = vm.resolveClass("android/content/res/AssetManager").newObject(custom);// context
        list.add(vm.addLocalObject(assetManager));

        Number number = module.callFunction(emulator, 0x207C + 1, list.toArray())[0];
        String result = vm.getObject(number.intValue()).getValue().toString();
        return result;
    }

    public static void main(String[] args) throws FileNotFoundException {
        Logger.getLogger("com.github.unidbg.linux.ARM32SyscallHandler").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.unix.UnixSyscallHandler").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.AbstractEmulator").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.linux.android.dvm.DalvikVM").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.linux.android.dvm.BaseVM").setLevel(Level.DEBUG);
        Logger.getLogger("com.github.unidbg.linux.android.dvm").setLevel(Level.DEBUG);
        demo2 test = new demo2();
        System.out.println("call demo2");
        System.out.println(test.call());
    }

}
```

运行

![](https://img-blog.csdnimg.cn/20210618132952419.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
直截了当的报错，根据 traceCode 定位  
![](https://img-blog.csdnimg.cn/20210618133004387.png)  
AAssetManager_fromJava 函数哪来的？为什么报错？这需要我们思考两个问题

1.Android 开发中，Native 层如何读取 Assets 文件  
2.Unidbg 如何处理这情况

首先，apk 资源文件的读取与 demo1 不同，并非简单的 open 了事。  
![](https://img-blog.csdnimg.cn/20210618133030487.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
![](https://img-blog.csdnimg.cn/20210618133039120.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
Assets 加载相关的 API，比如 AAssetManager_fromJava，就是由 libandroid.so 这个系统 SO 实现，但是呢，Unidbg 并没有内置加载这个系统 SO，我们首先看一下 Unidbg 完整支持的系统 SO

![](https://img-blog.csdnimg.cn/20210618133057694.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
可以发现，其实并不是很多。

考虑两个个问题

1.  为什么 Unidbg 不内置支持所有系统 SO 的加载
2.  如果一个 SO 的依赖 SO 里包含 Unidbg 尚未支持的系统 SO，那该怎么办？

先讨论第一个问题

一部分原因是大部分 SO 中主要的依赖项，就是 Unidbg 已经支持的这些，即已经够用了。把 Android 系统中全部 SO 都成功加载进 Unidbg 虚拟内存中，既是很大的工作量，又会占用过多内存。

另一个更主要的原因是，比如 libandroid.so，其依赖 SO 实在太多了，想顺利加载整个 SO 确确实实是个苦差事！

![](https://img-blog.csdnimg.cn/20210618133116258.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
再看问题 2

如果 SO 的依赖项中有 Unidbg 不支持的系统 SO，怎么办？

首先，Unidbg 会给予提示

![](https://img-blog.csdnimg.cn/2021061813313088.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
其次，尽管 SO 加载了 Unidbg 不支持的 SO，但有可能我们的目标函数并没有使用到这个系统 SO，这种情况下就不用理会，当作不存在就行。

但如果目标函数使用到了这个系统 SO，那就麻烦了，我们就得直面这个问题，一般有两种处理办法

*   Patch/Hook 这个不支持的 SO 所使用的函数
*   使用 Unidbg VirtualModule

方法一没什么技术含量而且并不总是能用，我们主要看一下方法二。

VirtualModule 是 Unidbg 为此种情况所提供的官方解决方案，并在代码中提供了两个示例

![](https://img-blog.csdnimg.cn/20210618133148902.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)分别是对 libandroid.so 以及 libJniGraphics 的处理  
我们使用一下  
![](https://img-blog.csdnimg.cn/20210618133211197.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4ODUxNTM2,size_16,color_FFFFFF,t_70)  
只用这一句即可，需要注意，一定要在样本 SO 加载前加载它，道理也很简单，系统 SO 肯定比用户 SO 加载早鸭。

Unidbg 如何实现一个 VirtualModule？此类问题我们在更后面的文章去讨论它。需要注意的是，VirtualModule 并不是一种真正意义上的加载 SO，它本质上也是 Hook，只不过实现了 SO 中少数几个函数罢了。

比如 AndroidModule 中，只实现了 libandroid 中这几个常用的导出函数

```
@Override
    protected void onInitialize(Emulator<?> emulator, final VM vm, Map<String, UnidbgPointer> symbols) {
        boolean is64Bit = emulator.is64Bit();
        SvcMemory svcMemory = emulator.getSvcMemory();
        symbols.put("AAssetManager_fromJava", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return fromJava(emulator, vm);
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return fromJava(emulator, vm);
            }
        }));
        symbols.put("AAssetManager_open", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return open(emulator, vm);
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return open(emulator, vm);
            }
        }));
        symbols.put("AAsset_close", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return close(emulator, vm);
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return close(emulator, vm);
            }
        }));
        symbols.put("AAsset_getBuffer", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return getBuffer(emulator, vm);
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return getBuffer(emulator, vm);
            }
        }));
        symbols.put("AAsset_getLength", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return getLength(emulator, vm);
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return getLength(emulator, vm);
            }
        }));
        symbols.put("AAsset_read", svcMemory.registerSvc(is64Bit ? new Arm64Svc() {
            @Override
            public long handle(Emulator<?> emulator) {
                throw new BackendException();
            }
        } : new ArmSvc() {
            @Override
            public long handle(Emulator<?> emulator) {
                return read(emulator, vm);
            }
        }));
    }
```

### 六、尾声

链接：https://pan.baidu.com/s/14MNPM3Rayb2RYgDS_T5pnw  
提取码：4974