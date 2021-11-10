> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-252319.htm)

> [原创]frida 学习记录

上次记录了下 xposed hook java 层的学习，这次记录下 frida

例子是在上次写的 apk 上稍加改动，添加几个简单的 native 方法；

*   MainActivity  
    

```
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button bt=(Button)findViewById(R.id.bt1);
 
        bt.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                HookGoal h=new HookGoal(0);
                h.editnum(1);
                Log.i("jni str",h.strtest("abcdefg"));
                DiyClass[] array={new DiyClass(1),new DiyClass(2)};
                Log.i("jni array","jni调用前 参数数组 成员0: "+array[0].getData()+" 成员1: "+array[1].getData());
                DiyClass[] myarray=h.arraytest(array);
                Log.i("jni array","jni调用后 返回新数组 成员0: "+myarray[0].getData()+" 成员1: "+myarray[1].getData());
                h.show();
                TextView t=(TextView)findViewById(R.id.sample_text);
                //t.setText("ooo");
                t.setText(h.sayhello());
                //Log.i("jni ","");
                Toast.makeText(MainActivity.this,"HOOKGOAL",Toast.LENGTH_LONG).show();
            }
        });
    }
}
```

  

*   类文件

```
abstract class person {
    public int age = 0;
 
    public void eat(String food) {
    }
}
 
public class HookGoal {
    static {
        System.loadLibrary("native-lib");
    }
    private static String TAG = "HookGoal:";
    private  int hookGoalNumber;
 
    public HookGoal(int number) {
        hookGoalNumber = number;
        Log.i(TAG, "HookGoal hookGoalNumber:" + hookGoalNumber);
    }
 
    public void func0() {
        Log.i(TAG, "welcome");
    }
 
    private void func1() {
        new person() {
            @Override
            public void eat(String food) {
                Log.i(TAG, "eat " + food);
            }
        }.eat("apple");
    }
 
    private static void func2(String s) {
        Log.i(TAG, "func2 " + s);
    }
 
    private void func3(DiyClass[] arry) {
        for (int i = 0; i < arry.length; i++)
            Log.i(TAG, "DiyClass[" + i + "].getData:" + arry[i].getData());
    }
 
    private class InnerClass {
        private int innerNumber;
 
        public InnerClass(String s) {
            innerNumber = 0;
            Log.i(TAG, "InnerClass 构造函数 " + s);
            Log.i(TAG, "InnerClass innerNumber:" + innerNumber);
        }
 
        private void innerFunc(String s) {
            Log.i(TAG, "InnerClass innerFunc " + s);
        }
    }
    public native String sayhello();
    public native String strtest(String s);
    public native DiyClass[] arraytest(DiyClass[] array);
    public native void editnum(int n);
    public void show() {
        Log.i(TAG,"hookGoalNumber:"+hookGoalNumber);
        func1();
        func2("私有静态方法");
        DiyClass[] arry = {new DiyClass(0), new DiyClass(0), new DiyClass(0)};
        func3(arry);
        InnerClass inner = new InnerClass("私有内部类");
        inner.innerFunc("内部类方法调用");
    }
 
}
 
 
public class DiyClass{
    private int data;
    public DiyClass(int data){
        this.data=data;
    }
    public int getData() {
        return data;
    }
 
    public void setData(int data) {
        this.data = data;
    }
}
```

  

*   native 方法，动态注册

```
int add(int a,int b){
    return a+b;
}
 
static jstring say(JNIEnv *env, jobject) {
    LOGI("native 函数say()返回字符串my jni hhh");
    string hello = "my jni hhh";
    return env->NewStringUTF(hello.c_str());
}
static jstring mystr(JNIEnv *env, jobject,jstring s) {
    char buf[256];
    env->GetStringUTFRegion(s,0,env->GetStringLength(s),buf);
    LOGI("GetStringUTFRegion 截取jstring保存到本地buf中：%s",buf);
 
    const char* ptr_c=env->GetStringUTFChars(s,NULL);
    LOGI("GetStringUTFChars 返回const char* 类型：%s",ptr_c);
    env->ReleaseStringUTFChars(s,ptr_c);
 
    const jchar *wc=env->GetStringChars(s, NULL);
    LOGI("GetStringChars 返回const jchar* 类型");
    env->ReleaseStringChars(s,wc);
 
    jstring js=env->NewStringUTF(buf);
/*  jstring NewStringUTF(const char* bytes)   返回jstring类型
    访问java.lang.String对应的JNI类型jstring时，不能像访问基本数据类型一样直接使用，因为它在Java是一个引用类型，所以在本地代码中只能通过GetStringUTFChars这样的JNI函数来访问字符串的内容*/
    LOGI("NewStringUTF(buf) 返回jstring类型：%s",env->GetStringUTFChars(js,NULL));
    return env->NewString(wc,env->GetStringLength(s));
}
static void edit(JNIEnv *env, jobject obj,jint n) {
    jint x=add(n,n);
    LOGI("edit()内调用了add()，原参数：%d",n);
    jclass clazz=env->GetObjectClass(obj);
    jfieldID numberid=env->GetFieldID(clazz,"hookGoalNumber","I");
    env->SetIntField(obj,numberid,x);
    LOGI("edit()内设置hookGoalNumber字段为：%d",x);
 
}
static jobjectArray myarray(JNIEnv *env, jobject obj,jobjectArray array){
    jclass diyclass=env->FindClass("com/example/goal/DiyClass");
    jmethodID initid=env->GetMethodID(diyclass,"","(I)V");
    jmethodID getdataid=env->GetMethodID(diyclass,"getData","()I");
    jobject a=env->GetObjectArrayElement(array,0);
    jobject a1=env->GetObjectArrayElement(array,1);
    jint d=env->CallIntMethod(a,getdataid);
    jint d1=env->CallIntMethod(a1,getdataid);
    jobject diy=env->NewObject(diyclass,initid,d);
    jobject diy2=env->NewObject(diyclass,initid,d1);
    jobjectArray myarray=env->NewObjectArray(2,diyclass,0);
    env->SetObjectArrayElement(myarray,0,diy2);
    env->SetObjectArrayElement(myarray,1,diy);
    LOGI("myarray()使用array参数，返回交换元素位置的新数组");
    return myarray;
//    return env->NewObjectArray(2,diyclass,diy2);
 
}
 
static const char *className = "com/example/goal/HookGoal";
 
static JNINativeMethod gJni_Methods_table[] = {
        {"editnum", "(I)V", (void*)edit},
        {"sayhello", "()Ljava/lang/String;",(void*)say},
        {"strtest", "(Ljava/lang/String;)Ljava/lang/String;",(void*)mystr},
        {"arraytest","([Lcom/example/goal/DiyClass;)[Lcom/example/goal/DiyClass;",(void*)myarray}
 
};
 
static int jniRegisterNativeMethods(JNIEnv* env, const char* className,
                                    const JNINativeMethod* gMethods, int numMethods)
{
    jclass clazz;
 
    clazz = (env)->FindClass( className);
    if (clazz == NULL) {
        return -1;
    }
 
    int result = 0;
    if ((env)->RegisterNatives(clazz, gJni_Methods_table, numMethods) < 0) {
        result = -1;
    }
 
    (env)->DeleteLocalRef(clazz);
    return result;
}
 
jint JNI_OnLoad(JavaVM* vm, void* reserved){
    JNIEnv* env = NULL;
    jint result = -1;
 
    if (vm->GetEnv((void**) &env, JNI_VERSION_1_4) != JNI_OK) {
        return result;
    }
    jniRegisterNativeMethods(env, className, gJni_Methods_table, sizeof(gJni_Methods_table) / sizeof(JNINativeMethod));
    return JNI_VERSION_1_4;
} 
```

  
开始 frida hook

*   首先是 java 层 hook  
    

```
setImmediate(function () {
    Java.perform(function () {
        console.log("start");
        //java层hook
        var hookgoal = Java.use("com.example.goal.HookGoal");
        var clazz = Java.use("java.lang.Class");
        var obj = Java.use("java.lang.Object");
        var Exception = Java.use("java.lang.Exception");
        var str = Java.use("java.lang.String");
        //hook 构造方法
        hookgoal.$init.overload("int").implementation = function (number) {
            send("HookGoal构造函数的参数number:" + number);
            send("HookGoal构造函数的参数修改为666");
            return this.$init(666);
        };
 
        //hook 静态变量TAG
        var reflectField = Java.cast(hookgoal.class, clazz).getDeclaredField("TAG");
        reflectField.setAccessible(true);
        reflectField.set("java.lang.String", "frida hooking");
        send("修改HookGoal的静态变量TAG为：frida hooking");
 
 
        //实例化对象way1
        var newhg = hookgoal.$new(0);
        send("new HookGoal instance newhg: " + newhg);
        // 实例化对象way2
        var newhg1 = hookgoal.$new.overload("int").call(hookgoal, 0);
        send("new HookGoal instance newhg1: " + newhg1);
 
        //hook匿名内部类，修改参数
        var nminner = Java.use("com.example.goal.HookGoal$1");
        nminner.eat.overload("java.lang.String").implementation = function (s) {
            var arg = arguments[0];
            send("eat参数获取 way1:" + arg);
            send("eat参数获取 way2:" + s);
            //修改参数
            return this.eat("is hooking");
        };
        var diy = Java.use("com.example.goal.DiyClass");
        hookgoal.func2.implementation = function (s) {
            //func2为静态方法
            var arg = arguments[0];
            send("func2()参数获取:" + s);
            //调用成员方法func0()在静态方法内只能通过创建的实例访问
            newhg.func0();
            send("func2()内调用func0()  通过创建实例newhg调用");
            newhg1.func0();
            send("func2()内调用func0()  通过创建实例newhg1调用");
 
            //修改实例的hookGoalNumber值，前面hook构造函数时已经将值改为666
            //修改字段值 通过反射得到字段，
            //var num1 = Java.cast(newhg1.getClass(), clazz).getDeclaredField("hookGoalNumber");
            var num1 = Java.cast(hookgoal.class, clazz).getDeclaredField("hookGoalNumber");
            num1.setAccessible(true);
            send("实例newhg1的hookGoalNumber:" + num1.get(newhg1));
            num1.setInt(newhg1, 777);
            send("修改实例newhg1的hookGoalNumber:" + num1.get(newhg1));
 
            send("实例newhg的hookGoalNumber:" + num1.get(newhg));
 
            // 反射调用方法
            var func = hookgoal.class.getDeclaredMethod("func0", null);
            send("func0:" + func);
            //var funcs = hookgoal.class.getDeclaredMethods();
            //for(var i=0;i
```

  

*   native 层 hook  
    

```
 //so层hook
        //导出函数
        //var exports = Module.enumerateExportsSync("libnative-lib.so");
        //for(var i=0;i", "(I)V");
                send("initid:" + initid);
                var setid = env.getMethodId(cla, "setData", "(I)V");
                send("setid:" + setid);
                var getid = env.getMethodId(cla, "getData", "()I");
                send("getid:" + getid);
                //frida 中env 方法参考frida-java/lib/env.js  本人能力有限，有些方法确实搞不懂
                //调用env中的allocObject()方法创建对象，未初始化，
                var obj1 = env.allocObject(cla);
                send("obj1:" + obj1);
 
                var obj2 = env.allocObject(cla);
                send("obj2:" + obj2);
 
                var rtarray = env.newObjectArray(2, cla, ptr(0));
                send("env.newObjectArray:" + rtarray);
 
                //获取DiyClass类中public void setData(int data)方法
                var nvmethod = env.nonvirtualVaMethod("void", ["int"]);
                //NativeType CallNonvirtualMethod(JNIEnv *env, jobject obj,jclass clazz, jmethodID methodID, ...);
                //设置obj1中data值
                nvmethod(env, obj1, cla, setid, 11);
                //设置obj2中data值
                nvmethod(env, obj2, cla, setid, 22);
                send("env.nonvirtualVaMethod(JNIEnv,jobject,jclass,jmethodid,args):" + nvmethod);
                //设置数组中的元素
                env.setObjectArrayElement(rtarray, 0, obj1);
                env.setObjectArrayElement(rtarray, 1, obj2);
                send("env.newObjectArray:" + rtarray);
 
                send("原retval:" + retval);
                retval.replace(ptr(rtarray));
                send("修改后retval:" + retval);
 
                // //堆中分配空间
                // var memo=Memory.alloc(4);
                // //写入数据
                // Memory.writeInt(memo,0x40302010);
                // // 读取数据
                // console.log(hexdump(memo, {
                //         offset: 0,
                //         length: 64,
                //         header: true,
                //         ansi: true
                // }));
            }
        });
 
    });
}); 
```

我喜欢写好 js 后放到 python 中用

```
import frida, sys
 
 
def on_message(message, data):
    if message['type'] == 'send':
        print("[*] {0}".format(message['payload']))
    else:
        print(message)
 
 
jscode = """
 """
# 启动时hook   //在测试时出现了问题
# devices=frida.get_usb_device()
# pid=devices.spawn(['com.example.goal'])   #以挂起方式创建进程 真机报错frida.PermissionDeniedError: unable to access  process
#找到原因了，我的是Android8.0 使用了Magisk，默认开启了Magisk Hide选项与zygote64冲突。
#解决方法：Magisk   -->设置-->Magisk下 去掉勾选Magisk Hide 
   
#模拟器报错Failed to spawn: the connection is closed
# session=devices.attach(pid)
# devices.resume(pid)    #创建完脚本, 恢复进程运行
# script=session.create_script(jscode)
 
# 命令行frida -U -f com.example.goal --no-pause -l  # 运行中hook
process = frida.get_usb_device().attach('com.example.goal')
script = process.create_script(jscode)
script.on('message', on_message)
print('[*] Running test')
script.load()
sys.stdin.read() 
```

  
也可以使用 frida 命令行 打开. js  （打印使用 console.log()）frida -U <进程名或 pid> -l <js 文件 >代码中涉及到的一些点大部分给出了注释，当然你只能这里看到一丢丢的 frida 方法的使用，更多的方法还要请参考官方文档、源代码 ···例如 native 层会用到大量 frida JavaScript API 的 Memory 操作，frida 中的 env 方法与 jni 开发中的方法并不完全相同，请参考源码 frida-java/lib/env.js，代码中涉及到的地址偏移请根据实际情况修改

*   hook 前 log 输出

```
4513-4513/com.example.goal I/frida hooking: HookGoal hookGoalNumber:0
4513-4513/com.example.goal I/jni test: edit()内调用了add()，原参数：1
4513-4513/com.example.goal I/jni test: edit()内设置hookGoalNumber字段为：2
4513-4513/com.example.goal I/jni test: GetStringUTFRegion 截取jstring保存到本地buf中：abcdefg
4513-4513/com.example.goal I/jni test: GetStringUTFChars 返回const char* 类型：abcdefg
4513-4513/com.example.goal I/jni test: GetStringChars 返回const jchar* 类型
4513-4513/com.example.goal I/jni str: abcdefg
4513-4513/com.example.goal I/jni array: jni调用前 参数数组 成员0: 1 成员1: 2
4513-4513/com.example.goal I/jni test: myarray()使用array参数，返回交换元素位置的新数组
4513-4513/com.example.goal I/jni array: jni调用后 返回新数组 成员0: 2 成员1: 1
4513-4513/com.example.goal I/frida hooking: hookGoalNumber:2
4513-4513/com.example.goal I/frida hooking: eat apple
4513-4513/com.example.goal I/frida hooking: func2 私有静态方法
4513-4513/com.example.goal I/frida hooking: DiyClass[0].getData:0
4513-4513/com.example.goal I/frida hooking: DiyClass[1].getData:0
4513-4513/com.example.goal I/frida hooking: DiyClass[2].getData:0
4513-4513/com.example.goal I/frida hooking: InnerClass 构造函数 私有内部类
4513-4513/com.example.goal I/frida hooking: InnerClass innerNumber:0
4513-4513/com.example.goal I/frida hooking: InnerClass innerFunc 内部类方法调用
4513-4513/com.example.goal I/jni test: native 函数say()返回字符串my jni hhh
```

  

*   hook 后 log 输出

```
4513-4513/com.example.goal I/frida hooking: HookGoal hookGoalNumber:666
4513-4513/com.example.goal I/jni test: edit()内调用了add()，原参数：4
4513-4513/com.example.goal I/jni test: edit()内设置hookGoalNumber字段为：16
4513-4513/com.example.goal I/jni test: GetStringUTFRegion 截取jstring保存到本地buf中：abcdefg
4513-4513/com.example.goal I/jni test: GetStringUTFChars 返回const char* 类型：abcdefg
4513-4513/com.example.goal I/jni test: GetStringChars 返回const jchar* 类型
4513-4513/com.example.goal I/jni str: frida hook native
4513-4513/com.example.goal I/jni array: jni调用前 参数数组 成员0: 1 成员1: 2
4513-4513/com.example.goal I/jni test: myarray()使用array参数，返回交换元素位置的新数组
4513-4513/com.example.goal I/jni array: jni调用后 返回新数组 成员0: 11 成员1: 22
4513-4513/com.example.goal I/frida hooking: hookGoalNumber:16
4513-4513/com.example.goal I/frida hooking: eat is hooking
4513-4513/com.example.goal I/frida hooking: welcome
4513-4513/com.example.goal I/frida hooking: welcome
4513-4513/com.example.goal I/frida hooking: welcome
4513-4513/com.example.goal I/frida hooking: func2 is hooking
4513-4513/com.example.goal I/frida hooking: welcome
4513-4513/com.example.goal I/frida hooking: DiyClass[0].getData:111
4513-4513/com.example.goal I/frida hooking: DiyClass[1].getData:222
4513-4513/com.example.goal I/frida hooking: DiyClass[2].getData:333
4513-4513/com.example.goal I/frida hooking: InnerClass 构造函数 frida is hooking
4513-4513/com.example.goal I/frida hooking: InnerClass innerNumber:0
4513-4513/com.example.goal I/frida hooking: InnerClass innerFunc frida is hooking
4513-4513/com.example.goal I/jni test: native 函数say()返回字符串my jni hhh
```

  

*   hook 脚本的输出

```
[*] Running test
start
[*] 修改HookGoal的静态变量TAG为：frida hooking
[*] HookGoal构造函数的参数number:0
[*] HookGoal构造函数的参数修改为666
[*] new HookGoal instance newhg: com.example.goal.HookGoal@4a854cb4
[*] HookGoal构造函数的参数number:0
[*] HookGoal构造函数的参数修改为666
[*] new HookGoal instance newhg1: com.example.goal.HookGoal@4a854f70
[*] enumerateModules find
[*] libnative-lib.so|0x7d0b0000|221184|/data/app-lib/com.example.goal-1/libnative-lib.so
[*] {'name': 'libnative-lib.so', 'base': '0x7d0b0000', 'size': 221184, 'path': '/data/app-lib/com.example.goal-1/libnative-lib.so'}
[*] enumerateModules stop
[*] soAddr:0x7d0b0000
[*] 函数add() nativeptr:0x7d0b8ad0
[*] 调用native 方法fun():11
[*] findExportByName add():0x7d0b8ad0
[*] fmystrptr:0x7d0b9200
[*] fmyarrayptr:0x7d0b9420
[*] HookGoal构造函数的参数number:0
[*] HookGoal构造函数的参数修改为666
[*] onEnter edit()
[*] edit() env：0xb83a6e50  jobject：0xf7c00025 jint:1
[*] hook edit() 修改后的参数jint：0x4
[*] onEnter add()
[*] hook add()修改参数为原来的两倍 args[0]:8  args[1]:8
[*] onLeave  add()
[*] onLeave edit()
[*] onEnter mystr()
[*] mystr() env：0xb83a6e50  jobject：0xf7d00025 jstring:0x3d300029
[*] mystr() jstring参数：abcdefg
[*] onLeave mystr()
[*] 修改返回值jstring:0xc5b00035
[*] onEnter myarray()
[*] mystr() env：0xb83a6e50  jobject：0xf8000025 jobjectArray:0x3d600029
[*] jobjectArray参数：0x3d600029
[*] onLeave myarray()
[*] argptr:0x3d600029
[*] clazz:0x26800045
[*] initid:0x881d1ea8
[*] setid:0x881d1f20
[*] getid:0x881d1ee8
[*] obj1:0x1e000049
[*] obj2:0x1e00004d
[*] env.newObjectArray:0x1dc00051
[*] env.nonvirtualVaMethod(JNIEnv,jobject,jclass,jmethodid,args):0xb4d88bb0
[*] env.newObjectArray:0x1dc00051
[*] 原retval:0x2f900041
[*] 修改后retval:0x1dc00051
[*] eat参数获取 way1:apple
[*] eat参数获取 way2:apple
[*] func2()参数获取:私有静态方法
[*] func2()内调用func0()  通过创建实例newhg调用
[*] func2()内调用func0()  通过创建实例newhg1调用
[*] 实例newhg1的hookGoalNumber:666
[*] 修改实例newhg1的hookGoalNumber:777
[*] 实例newhg的hookGoalNumber:666
[*] func0:public void com.example.goal.HookGoal.func0()
[*] func2()内调用func0()  way2 通过反射调用
[*] func2()内调用DiyClass下的getData() 通过创建实例d调用 返回：666
[*] func3()内调用func0()  way2 成员方法中直接调用其他成员方法
[*] func3参数：com.example.goal.DiyClass@4a8596b4,com.example.goal.DiyClass@4a8596bc,com.example.goal.DiyClass@4a8596c4
[*] func3参数修改：com.example.goal.DiyClass@4a859a08,com.example.goal.DiyClass@4a859a78,com.example.goal.DiyClass@4a859ae8
[*] innerClass构造函数的参数:私有内部类
[*] frida hook 前innerFunc()的参数：内部类方法调用
[*] 通过this.innerNumber.value获取值:0
[*] 通过this.innerNumber.value设置值后:1
[*] 反射方式 innerNumber修改前:1
[*] 反射方式 innerNumber修改后:2
[*] onEnter say()
[*] onLeave say()
[*] say() 原返回值：my jni hhh
[*] 修改say()返回值:frida hook native
```

frida 功能十分强大，在这里我作为初学者简单的记录了下学习经历，当然这仅仅是一些常规操作，是基础，日后的学习应该是在实战中，实践出真知。

> 合抱之木，生于毫末；九层之台，起于累土；千里之行，始于足下

我在学习的过程之中也遇到了各种各样的坑，困惑过，苦恼过。解决问题绝对是一种很棒的学习方式， 希望各位能够提出自己的问题，大家一起交流，共同成长！  
同时文中难免会有错误，欢迎各位坛友指出，不胜感激！  
  
链接：https://pan.baidu.com/s/1KX1fY16NgaYB1FnCrpu0lQ 提取码：t38k （打好基础，改日实战一番！）

[[注意] 欢迎加入看雪团队！base 上海，招聘安全工程师、逆向工程师多个坑位等你投递！](https://bbs.pediy.com/thread-267474.htm)

最后于 2019-11-22 17:11 被堂前燕编辑 ，原因： 修改注释