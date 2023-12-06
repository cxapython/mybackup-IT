> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/6844904192780271630)

> 这篇文章主要讲解了 JNI 的基础语法和交叉编译的基本使用，通过这篇文章的学习就完全可以入门 Android 下 JNI 项目的开发了。

这篇文章主要讲解了 JNI 的基础语法和交叉编译的基本使用，通过这篇文章的学习就完全可以入门 Android 下 JNI 项目的开发了。

JNI 概念
------

从 JVM 角度，存在两种类型的代码：“Java” 和 “native”, native 一般指的是 c/c++，为了使 java 和 native 端能够进行交互，java 设计了 JNI（java native interface）。 JNI 允许 java 虚拟机（VM）内运行的 java 代码与 C++、C++ 和汇编等其他编程语言编写的应用程序和库进行互操作。

虽然大部分情况下我们的软件完全可以由 java 来实现，但是某些场景下使用 native 代码更加适合，比如：

*   代码效率：使用 native 代码的性能更高
*   跨平台特性：标准 Java 类库不支持应用程序所需的依赖于平台的特性，或者希望用较低级别的语言（如汇编语言）实现一小部分时间关键型代码。

native 层使用 JNI 主要可以做到：

*   创建、检查和更新 Java 对象（包括数组和字符串）。
*   调用 Java 方法。
*   加载类并获取类信息。

创建 android ndk 项目
-----------------

使用 as 创建一个 native c++ 项目

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bca7dc552f717~tplv-t2oaga2asx-watermark.awebp)

文件结构如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bca81bb1561b3~tplv-t2oaga2asx-watermark.awebp)

可以看到生成了一个 cpp 文件夹，里面有 CMakeLists.txt, native-lib.cpp,CMakeLists 后面再讲，这里先来看一下 native-lib.cpp 和 java 代码。

```
public class MainActivity extends AppCompatActivity {
    static {
        System.loadLibrary("native-lib");
    }
    ...
    public native String stringFromJNI();
}
复制代码

```

```
#include <jni.h>
#include <string>

extern "C" JNIEXPORT jstring JNICALL
Java_com_wangzhen_jnitutorial_MainActivity_stringFromJNI(JNIEnv* env, jobject thiz) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
复制代码

```

可以看到在 MainActivity 中先定义了一个 native 方法，然后编译器在 cpp 文件中创建一个一个对应的方法`Java_com_wangzhen_jnitutorial_MainActivity_stringFromJNI`。 它的命名规则就是 Java_packageName_methodName。

接下来我们详细的解读一下 cpp 中的代码。

native 代码解读
-----------

### extern "C"

在 c++ 中使用 c 代码

### JNIEXPORT

宏定义：`#define JNIEXPORT __attribute__ ((visibility ("default")))` 在 Linux/Unix/Mac os/Android 这种类 Unix 系统中，定义为`__attribute__ ((visibility ("default")))`

GCC 有个 visibility 属性, 该属性是说, 启用这个属性:

*   当 - fvisibility=hidden 时，动态库中的函数默认是被隐藏的即 hidden。
*   当 - fvisibility=default 时，动态库中的函数默认是可见的。

### JNICALL

宏定义，在 Linux/Unix/Mac os/Android 这种类 Unix 系统中，它是个空的宏定义： `#define JNICALL`，所以在 android 上删除它也可以。 快捷生成 .h 代码

### JNIEnv

*   JNIEnv 类型实际上代表了 Java 环境，通过这个 JNIEnv* 指针，就可以对 Java 端的代码进行操作：
    *   调用 Java 函数
    *   操作 Java 对象
*   JNIEnv 的本质是一个与线程相关的结构体，里面存放了大量的 JNI 函数指针：

```
struct _JNIEnv {
    /**
    * 定义了很多的函数指针
    **/
    const struct JNINativeInterface* functions;

#if defined(__cplusplus)
    /// 通过类的名称(类的全名，这时候包名不是用.号，而是用/来区分的)来获取jclass    
    jclass FindClass(const char* name)
    { return functions->FindClass(this, name); }
    ...
}    
复制代码

```

JNIEnv 的结构图如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bcaa3bb97ce17~tplv-t2oaga2asx-watermark.awebp)

### JavaVM

*   JavaVM : JavaVM 是 Java 虚拟机在 JNI 层的代表, JNI 全局只有一个
    
*   JNIEnv : JavaVM 在线程中的代表, 每个线程都有一个, JNI 中可能有很多个 JNIEnv，同时 JNIEnv 具有线程相关性，也就是 B 线程无法使用 A 线程的 JNIEnv
    

JVM 的结构图如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bcaaf419f24bc~tplv-t2oaga2asx-watermark.awebp)

### jobject thiz

这个 object 指向该 native 方法的 this 实例，比如我们在 MainActivity 调用的下面的 native 函数中打印一下 thiz 的 className：

```
#define  LOGE(...) __android_log_print(ANDROID_LOG_ERROR,"JNI",__VA_ARGS__);

extern "C" JNIEXPORT jstring JNICALL
Java_com_wangzhen_jnitutorial_MainActivity_stringFromJNI(JNIEnv *env, jobject thiz) {
    std::string hello = "Hello from C++";
    // 1. 获取 thiz 的 class，也就是 java 中的 Class 信息
    jclass thisclazz = env->GetObjectClass(thiz);
    // 2. 根据 Class 获取 getClass 方法的 methodID，第三个参数是签名(params)return
    jmethodID mid_getClass = env->GetMethodID(thisclazz, "getClass", "()Ljava/lang/Class;");
    // 3. 执行 getClass 方法，获得 Class 对象
    jobject clazz_instance = env->CallObjectMethod(thiz, mid_getClass);
    // 4. 获取 Class 实例
    jclass clazz = env->GetObjectClass(clazz_instance);
    // 5. 根据 class  的 methodID
    jmethodID mid_getName = env->GetMethodID(clazz, "getName", "()Ljava/lang/String;");
    // 6. 调用 getName 方法
    jstring name = static_cast<jstring>(env->CallObjectMethod(clazz_instance, mid_getName));
    LOGE("class name:%s", env->GetStringUTFChars(name, 0));

    return env->NewStringUTF(hello.c_str());
}
复制代码

```

打印结果如下：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bcabba9c3b0fd~tplv-t2oaga2asx-watermark.awebp)

JNI 基础
------

### 数据类型

#### 基础数据类型

<table><thead><tr><th>Java Type</th><th>Native Type</th><th>Description</th></tr></thead><tbody><tr><td>boolean</td><td>jboolean</td><td>unsigned 8 bits</td></tr><tr><td>byte</td><td>jbyte</td><td>signed 8 bits</td></tr><tr><td>char</td><td>jchar</td><td>unsigned 16 bits</td></tr><tr><td>short</td><td>jshort</td><td>signed 16 bits</td></tr><tr><td>int</td><td>jint</td><td>signed 32 bits</td></tr><tr><td>long</td><td>jlong</td><td>signed 64 bits</td></tr><tr><td>float</td><td>jfloat</td><td>32 bits</td></tr><tr><td>double</td><td>jdouble</td><td>64 bits</td></tr><tr><td>void</td><td>void</td><td>N/A</td></tr></tbody></table>

#### 引用类型

这里贴一张 oracle 文档中的图，虽然很丑但挺好：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bcac6f5314d1f~tplv-t2oaga2asx-watermark.awebp)

#### Field and Method IDs

JNIEvn 操作 java 对象时利用 java 中的反射，操作某个属性都需要 field 和 method 的 id，这些 id 都是指针类型：

```
struct _jfieldID;              /* opaque structure */ 
typedef struct _jfieldID *jfieldID;   /* field IDs */ 
 
struct _jmethodID;              /* opaque structure */ 
typedef struct _jmethodID *jmethodID; /* method IDs */ 
复制代码

```

### JNI 操作 java 对象

#### 操作 jarray

将一个 Java int[] 对象传入 C++ 中，如何操作这个数组呢？

```
JNIEXPORT void JNICALL
Java_com_wangzhen_jnitutorial_MainActivity_setArray(JNIEnv *env, jobject thiz, jintArray array) {

    // 1.获取数组长度
    jint len = env->GetArrayLength(array);
    LOGE("array.length:%d", len);

    jboolean isCopy;
    // 2.获取数组地址 
    // 第二个参数代表 javaArray -> c/c++ Array 转换的方式：
    // 0: 把指向Java数组的指针直接传回到本地代码中
    // 1: 新申请了内存，拷贝了数组
    // 返回值： 数组的地址（首元素地址）
    jint *firstElement = env->GetIntArrayElements(array, &isCopy);
    LOGE("is copy array:%d", isCopy);
    // 3.遍历数组（移动地址）
    for (int i = 0; i < len; ++i) {
        LOGE("array[%i] = %i", i, *(firstElement + i));
    }
    // 4.使用后释放数组
    // 第一个参数是 jarray，第二个参数是 GetIntArrayElements 返回值 
    // 第三个参数代表 mode
    env->ReleaseIntArrayElements(array,firstElement,0);

    // 5. 创建一个 java 数组
    jintArray newArray = env->NewIntArray(3);
}
复制代码

```

> *   mode = 0 刷新 java 数组 并 释放 c/c++ 数组
> *   mode = JNI_COMMIT (1) 只刷新 java 数组
> *   mode = JNI_ABORT (2) 只释放 c/c++ 数组

### 操作 jstring

```
extern "C"
JNIEXPORT void JNICALL
Java_com_wangzhen_jnitutorial_MainActivity_setString(JNIEnv *env, jobject thiz, jstring str) {
    // 1.jstring -> char*
    // java  中的字符创是 unicode 编码， c/C++ 是UTF编码，所以需要转换一下。第二个参数作用同上面
    const char *c_str = env -> GetStringUTFChars(str,NULL);

    // 2.异常处理
    if(c_str == NULL){
        return;
    }

    // 3.当做一个 char 数组打印
    jint len = env->GetStringLength(str);
    for (int i = 0; i < len; ++i) {
        LOGE("c_str: %c",*(c_str+i));
    }

    // 4.释放
    env->ReleaseStringUTFChars(str,c_str);
}
复制代码

```

调用完 GetStringUTFChars 之后不要忘记安全检查，因为 JVM 需要为新诞生的字符串分配内存空间，当内存空间不够分配的时候，会导致调用失败，失败后 GetStringUTFChars 会返回 NULL，并抛出一个 OutOfMemoryError 异常。JNI 的异常和 Java 中的异常处理流程是不一样的，Java 遇到异常如果没有捕获，程序会立即停止运行。而 JNI 遇到未决的异常不会改变程序的运行流程，也就是程序会继续往下走，这样后面针对这个字符串的所有操作都是非常危险的，因此，我们需要用 return 语句跳过后面的代码，并立即结束当前方法。

#### 操作 jobject

*   c/c++ 操作 java 中的对象使用的是 java 中反射，步骤分为：
*   获取 class 类
*   根据成员变量名获取 methodID / fieldID
*   调用 get/set 方法操作 field，或者 CallObjectMethod 调用 method

##### 操作 Field

*   非静态成员变量使用: GetXXXField, 比如 GetIntField，对于引用类型，比如 String，使用 GetObjectField
*   对于静态成员变量使用: GetStaticXXXField, 比如 GetStaticIntField

在 java 代码中，MainActivity 有两个成员变量：

```
public class MainActivity extends AppCompatActivity {

    String testField = "test1";

    static int staticField = 1;
}
复制代码

```

```
    // 1. 获取类 class
    jclass clazz = env->GetObjectClass(thiz);

    // 2. 获取成员变量 id
    jfieldID strFieldId = env->GetFieldID(clazz,"testField","Ljava/lang/String;");
    // 3. 根据 id 获取值
    jstring jstr = static_cast<jstring>(env->GetObjectField(thiz, strFieldId));
    const char* cStr = env->GetStringUTFChars(jstr,NULL);
    LOGE("获取 MainActivity 的 String field ：%s",cStr);

    // 4. 修改 String
    jstring newValue = env->NewStringUTF("新的字符创");
    env-> SetObjectField(thiz,strFieldId,newValue);

    // 5. 释放资源
    env->ReleaseStringUTFChars(jstr,cStr);
    env->DeleteLocalRef(newValue);
    env->DeleteLocalRef(clazz);
    
    // 获取静态变量
    jfieldID staticIntFieldId = env->GetStaticFieldID(clazz,"staticField","I");
    jint staticJavaInt = env->GetStaticIntField(clazz,staticIntFieldId);
复制代码

```

GetFieldID 和 GetStaticFieldID 需要三个参数：

*   jclass
*   filed name
*   类型签名： JNI 使用 jvm 的类型签名

###### 类型签名一览表

<table><thead><tr><th>Type</th><th>Signature Java Type</th></tr></thead><tbody><tr><td>Z</td><td>boolean</td></tr><tr><td>B</td><td>byte</td></tr><tr><td>C</td><td>char</td></tr><tr><td>S</td><td>short</td></tr><tr><td>I</td><td>int</td></tr><tr><td>J</td><td>long</td></tr><tr><td>F</td><td>float</td></tr><tr><td>D</td><td>double</td></tr><tr><td>V</td><td>void</td></tr><tr><td>L fully-qualified-class;</td><td>fully-qualified-class</td></tr><tr><td>[type</td><td>type[]</td></tr><tr><td>(arg-types) ret-type</td><td>method type</td></tr></tbody></table>

*   基本数据类型的比较好理解，不如要获取一个 int ，GetFieldID 需要传入签名就是 I；
    
*   如果是一个类，比如 String，签名就是 L + 全类名; ：Ljava.lang.String;
    
*   如果是一个 int array，就要写作 [I
    
*   如果要获取一个方法，那么方法的签名是:(参数签名) 返回值签名，参数如果是多个，中间不需要加间隔符，比如： | java 方法 | JNI 签名 | |--|--| |void f (int n); |(I)V| |void f (String s,int n); |(Ljava/lang/String;I)V| |long f (int n, String s, int[] arr); |(ILjava/lang/String;[I)J|
    

##### 操作 method

操作 method 和 filed 非常相似，先获取 MethodID，然后对应的 CallXXXMethod 方法

<table><thead><tr><th>Java 层返回值</th><th>方法族</th><th>本地返回类型 NativeType</th></tr></thead><tbody><tr><td>void</td><td>CallVoidMethod()</td><td>(无)</td></tr><tr><td>引用类型</td><td>CallObjectMethod( )</td><td>jobect</td></tr><tr><td>boolean</td><td>CallBooleanMethod ( )</td><td>jboolean</td></tr><tr><td>byte</td><td>CallByteMethod( )</td><td>jbyte</td></tr><tr><td>char</td><td>CallCharMethod( )</td><td>jchar</td></tr><tr><td>short</td><td>CallShortMethod( )</td><td>jshort</td></tr><tr><td>int</td><td>CallIntMethod( )</td><td>jint</td></tr><tr><td>long</td><td>CallLongMethod()</td><td>jlong</td></tr><tr><td>float</td><td>CallFloatMethod()</td><td>jfloat</td></tr><tr><td>double</td><td>CallDoubleMethod()</td><td>jdouble</td></tr></tbody></table>

在 java 中我们要想获取 MainActivity 的 className 会这样写：

```
 this.getClass().getName()
复制代码

```

可以看到需要先调用 getClass 方法获取 Class 对象，然后调用 Class 对象的 getName 方法，我们来看一下如何在 native 方法中调用：

```
extern "C" JNIEXPORT jstring JNICALL
Java_com_wangzhen_jnitutorial_MainActivity_stringFromJNI(JNIEnv *env, jobject thiz) {
    std::string hello = "Hello from C++";
    // 1. 获取 thiz 的 class，也就是 java 中的 Class 信息
    jclass thisclazz = env->GetObjectClass(thiz);
    // 2. 根据 Class 获取 getClass 方法的 methodID，第三个参数是签名(params)return
    jmethodID mid_getClass = env->GetMethodID(thisclazz, "getClass", "()Ljava/lang/Class;");
    // 3. 执行 getClass 方法，获得 Class 对象
    jobject clazz_instance = env->CallObjectMethod(thiz, mid_getClass);
    // 4. 获取 Class 实例
    jclass clazz = env->GetObjectClass(clazz_instance);
    // 5. 根据 class  的 methodID
    jmethodID mid_getName = env->GetMethodID(clazz, "getName", "()Ljava/lang/String;");
    // 6. 调用 getName 方法
    jstring name = static_cast<jstring>(env->CallObjectMethod(clazz_instance, mid_getName));
    LOGE("class name:%s", env->GetStringUTFChars(name, 0));
    
    // 7. 释放资源
    env->DeleteLocalRef(thisclazz);
    env->DeleteLocalRef(clazz);
    env->DeleteLocalRef(clazz_instance);
    env->DeleteLocalRef(name);
    
    return env->NewStringUTF(hello.c_str());
}
复制代码

```

#### 创建对象

首先定义一个 java 类：

```
public class Person {
    private int age;
    private String name;

    public Person(int age, String name){
        this.age = age;
        this.name = name;
    }

    public void print(){
        Log.e("Person",name + age + "岁了");
    }
}
复制代码

```

然后我们再 JNI 中创建一个 Person 并调用它的 print 方法：

```
// 1. 获取 Class
    jclass pClazz = env->FindClass("com/wangzhen/jnitutorial/Person");
    // 2. 获取构造方法，方法名固定为<init>
    jmethodID constructID = env->GetMethodID(pClazz,"<init>","(ILjava/lang/String;)V");
    if(constructID == NULL){
        return;
    }
    // 3. 创建一个 Person 对象
    jstring name = env->NewStringUTF("alex");
    jobject person = env->NewObject(pClazz,constructID,1,name);

    jmethodID printId = env->GetMethodID(pClazz,"print","()V");
    if(printId == NULL){
        return;
    }
    env->CallVoidMethod(person,printId);

    // 4. 释放资源
    env->DeleteLocalRef(name);
    env->DeleteLocalRef(pClazz);
    env->DeleteLocalRef(person);
复制代码

```

### JNI 引用

JNI 分为三种引用：

*   局部引用（Local Reference），类似 java 中的局部变量
*   全局引用（Global Reference），类似 java 中的全局变量
*   弱全局引用（Weak Global Reference），类似 java 中的弱引用

上面的代码片段中最后都会有释放资源的代码，这是 c/c++ 编程的良好习惯，对于不同 JNI 引用有不同的释放方式。

### 局部引用

#### 创建

JNI 函数返回的所有 Java 对象都是局部引用，比如上面调用的 NewObject/FindClass/NewStringUTF 等等都是局部引用。

#### 释放

*   自动释放 局部引用在方法调用期间有效，并在方法返回后被 JVM 自动释放。
*   手动释放

##### 手动释放的场景

有了自动释放之后为什么还需要手动释放呢？主要考虑一下场景：

*   本机方法访问大型 Java 对象，从而创建对 Java 对象的局部引用。然后，本机方法在返回到调用方之前执行附加计算。对大型 Java 对象的本地引用将防止对该对象进行垃圾收集，即使该对象不再用于计算的其余部分。
*   本机方法创建大量本地引用，但并非所有本地引用都同时使用。因为 JVM 需要一定的空间来跟踪本地引用，所以创建了太多的本地引用，这可能导致系统内存不足。例如，本机方法循环遍历一个大型对象数组，检索作为本地引用的元素，并在每次迭代时对一个元素进行操作。每次迭代之后，程序员不再需要对数组元素的本地引用。

所以我们应该养成手动释放本地引用的好习惯。

##### 手动释放的方式

*   GetXXX 就必须调用 ReleaseXXX。

> 在调用 GetStringUTFChars 函数从 JVM 内部获取一个字符串之后，JVM 内部会分配一块新的内存，用于存储源字符串的拷贝，以便本地代码访问和修改。即然有内存分配，用完之后马上释放是一个编程的好习惯。通过调用 ReleaseStringUTFChars 函数通知 JVM 这块内存已经不使用了。

*   对于手动创建的 jclass，jobject 等对象使用 DeleteLocalRef 方法进行释放

### 全局引用

#### 创建

JNI 允许程序员从局部引用创建全局引用：

```
 static jstring globalStr;
 if(globalStr == NULL){
   jstring str = env->NewStringUTF("C++");
   // 从局部变量 str 创建一个全局变量
   globalStr = static_cast<jstring>(env->NewGlobalRef(str));
   
   //局部可以释放，因为有了一个全局引用使用str，局部str也不会使用了
    env->DeleteLocalRef(str);
    }
复制代码

```

#### 释放

全局引用在显式释放之前保持有效，可以通过 DeleteGlobalRef 来手动删除全局引用调用。

### 弱全局引用

与全局引用类似，弱引用可以跨方法、线程使用。与全局引用不同的是，弱引用不会阻止 GC 回收它所指向的 VM 内部的对象

> 所以在使用弱引用时，必须先检查缓存过的弱引用是指向活动的对象，还是指向一个已经被 GC 的对象

#### 创建

```
    static jclass globalClazz = NULL;
    //对于弱引用 如果引用的对象被回收返回 true，否则为false
    //对于局部和全局引用则判断是否引用java的null对象
    jboolean isEqual = env->IsSameObject(globalClazz, NULL);
    if (globalClazz == NULL || isEqual) {
        jclass clazz = env->GetObjectClass(instance);
        globalClazz = static_cast<jclass>(env->NewWeakGlobalRef(clazz));
        env->DeleteLocalRef(clazz);
    }
复制代码

```

#### 释放

删除使用 DeleteWeakGlobalRef

### 线程相关

局部变量只能在当前线程使用，而全局引用可以跨方法、跨线程使用，直到它被手动释放才会失效。

加载动态库
-----

在 android 中有两种方式加载动态库：

*   System.load(String filename) // 绝对路径
*   system library path // 从 system lib 路径下加载

比如下面代码会报错，在 java.library.path 下找不到 hello

```
static{
    System.loadLibrary("Hello");
}
复制代码

```

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bcae4b92c0365~tplv-t2oaga2asx-watermark.awebp)

可以使用下面代码打印出 java.library.path , 并且吧 hello 拷贝到改路径下：

```
public static void main(String[] args){
    System.out.println(System.getProperty("java.library.path"));
}
复制代码

```

### JNI_OnLoad

调用 System.loadLibrary() 函数时， 内部就会去查找 so 中的 JNI_OnLoad 函数，如果存在此函数则调用。 JNI_OnLoad 必须返回 JNI 的版本，比如 JNI_VERSION_1_6、JNI_VERSION_1_8。

### 动态注册

JNI 匹配对应的 java 方法有两种方式：

*   静态注册： 之前我们使用的 Java_com_wangzhen_jnitutorial_MainActivity_stringFromJNI 来进行与 java 方法的匹配就是静态注册
*   动态注册：就是将 java 中的方法在代码中动态的与 JNI 方法对应起来

静态注册的名字需要包名，太长了，可以使用动态注册来缩短方法名。

比如我们再 Java 中有两个 native 方法：

```
public class MainActivity extends AppCompatActivity {
    public native void dynamicJavaFunc1();

    public native int dynamicJavaFunc2(int i);
}
复制代码

```

在 native 代码中，我们不使用静态注册，而使用动态注册

```
void dynamicNativeFunc1(){
    LOGE("调用了 dynamicJavaFunc1");
}
// 如果方法带有参数,前面要加上 JNIEnv *env, jobject thisz
jint dynamicNativeFunc2(JNIEnv *env, jobject thisz,jint i){
    LOGE("调用了 dynamicTest2，参数是:%d",i);
    return 66;
}

// 需要动态注册的方法数组
static const JNINativeMethod methods[] = {
        {"dynamicJavaFunc1","()V",(void*)dynamicNativeFunc1},
        {"dynamicJavaFunc2","(I)I",(int*)dynamicNativeFunc2},
};
// 需要动态注册native方法的类名
static const char *mClassName = "com/wangzhen/jnitutorial/MainActivity";


jint JNI_OnLoad(JavaVM* vm, void* reserved){
    JNIEnv* env = NULL;
    // 1. 获取 JNIEnv，这个地方要注意第一个参数是个二级指针
    int result = vm->GetEnv(reinterpret_cast<void **>(&env), JNI_VERSION_1_6);
    // 2. 是否获取成功
    if(result != JNI_OK){
        LOGE("获取 env 失败");
        return JNI_VERSION_1_6;
    }
    // 3. 注册方法
    jclass classMainActivity = env->FindClass(mClassName);
    // sizeof(methods)/ sizeof(JNINativeMethod)
    result = env->RegisterNatives(classMainActivity,methods, 2);

    if(result != JNI_OK){
        LOGE("注册方法失败")
        return JNI_VERSION_1_2;
    }

    return JNI_VERSION_1_6;
}
复制代码

```

这样我们再 MainActivity 中调用 dynamicJavaFunc1 方法就会调用 native 中的 dynamicNativeFunc1 方法。

### native 线程中调用 JNIEnv*

前面介绍过 JNIEnv* 是和线程相关的，那么如果在 c++ 中新建一个线程 A，在线程 A 中可以直接使用 JNIEnv* 吗？ 答案是否定的，如果想在 native 线程中使用 JNIEnv* 需要使用 JVM 的 AttachCurrentThread 方法进行绑定：

```
JavaVM *_vm;

jint JNI_OnLoad(JavaVM* vm, void* reserved){
    _vm = vm;
    return JNI_VERSION_1_6;
 }

void* threadTask(void* args){
    JNIEnv *env;
    jint result = _vm->AttachCurrentThread(&env,0);
    if (result != JNI_OK){
        return 0;
    }

    // ...

    // 线程 task 执行完后不要忘记分离
    _vm->DetachCurrentThread();
}

extern "C"
JNIEXPORT void JNICALL
Java_com_wangzhen_jnitutorial_MainActivity_nativeThreadTest(JNIEnv *env, jobject thiz) {
    pthread_t pid;
    pthread_create(&pid,0,threadTask,0);
}
复制代码

```

交叉编译
----

在一个平台上编译出另一个平台上可以执行的二级制文件的过程叫做交叉编译。比如在 MacOS 上编译出 android 上可用的库文件。 如果想要编译出可以在 android 平台上运行的库文件就需要使用 ndk。

### 两种库文件

linux 平台上的库文件分为两种：

*   静态库： 编译链接时，把库文件的代码全部加入到可执行文件中，因此生成的文件比较大，但在运行时也就不再需要库文件了，linux 中后缀名为”.a”。
*   动态库： 在编译链接时并没有把库文件的代码加入到可执行文件中，而是在程序执行时由运行时链接文件加载库。linux 中后缀名为”.so”，gcc 在编译时默认使用动态库。

Android 原生开发套件 (NDK)：这套工具使您能在 Android 应用中使用 C 和 C++ 代码。 CMake：一款外部编译工具，可与 Gradle 搭配使用来编译原生库。如果您只计划使用 ndk-build，则不需要此组件。 LLDB：Android Studio 用于调试原生代码的调试程序。

### NDK

原生开发套件 (NDK) 是一套工具，使您能够在 Android 应用中使用 C 和 C++ 代码，并提供众多平台库。 我们可以在 sdk/ndk-bundle 中查看 ndk 的目录结构，下面列举出三个重要的成员：

*   ndk-build: 该 Shell 脚本是 Android NDK 构建系统的起始点，一般在项目中仅仅执行这一个命令就可以编译出对应的动态链接库了。
*   platforms: 该目录包含支持不同 Android 目标版本的头文件和库文件， NDK 构建系统会根据具体的配置来引用指定平台下的头文件和库文件。
*   toolchains: 该目录包含目前 NDK 所支持的不同平台下的交叉编译器 - ARM 、X86、MIPS ，目前比较常用的是 ARM。 // todo ndk-depends.cmd

> ndk 为什么要提供多平台呢？ 不同的 Android 设备使用不同的 CPU，而不同的 CPU 支持不同的指令集。更具体的内容参考[官方文档](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Fguides%2Fabis "https://developer.android.com/ndk/guides/abis")

### 使用 ndk 手动编译动态库

在 ndk 目录下的 toolchains 下有多个平台的编译工具，比如在 /arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin 下可以找到 arm-linux-androideabi-gcc 执行文件，利用 ndk 的这个 gcc 可以编译出在 android（arm 架构） 上运行的动态库：

```
arm-linux-androideabi-gcc -fPIC -shared test.c -o libtest.so
复制代码

```

> 参数含义 -fPIC： 产生与位置无关代码 -shared：编译动态库，如果去掉代表静态库 test.c：需要编译的 c 文件 -o：输出 libtest.so：库文件名

> 独立工具链 版本比较新的 ndk 下已经找不到 gcc 了，如果想用的话需要参考[独立工具链](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Fguides%2Fstandalone_toolchain "https://developer.android.com/ndk/guides/standalone_toolchain")。 比如执行 `$NDK/build/tools/make_standalone_toolchain.py --arch arm --api 21 --install-dir/$yourDir` 可以产生 arm 的独立工具链

`$NDK 代表 ndk 的绝对路径， $yourDir 代表输出文件路径`

当源文件很多的时候，手动编译既麻烦又容易出错，此时出现了 makefile 编译。

### makefile

makefile 就是 “自动化编译”：一个工程中的源文件不计数，其按类型、功能、模块分别放在若干个目录中，makefile 定义了一系列的规则来指定，哪些文件需要先编译，哪些文件需要后编译，如何进行链接等等操作。 Android 使用 Android.mk 文件来配置 makefile，下面是一个最简单的 Android.mk：

```
# 源文件在的位置。宏函数 my-dir 返回当前目录（包含 Android.mk 文件本身的目录）的路径。
LOCAL_PATH := $(call my-dir)

# 引入其他makefile文件。CLEAR_VARS 变量指向特殊 GNU Makefile，可为您清除许多 LOCAL_XXX 变量
# 不会清理 LOCAL_PATH 变量
include $(CLEAR_VARS)

# 指定库名称，如果模块名称的开头已是 lib，则构建系统不会附加额外的前缀 lib；而是按原样采用模块名称，并添加 .so 扩展名。
LOCAL_MODULE := hello
# 包含要构建到模块中的 C 和/或 C++ 源文件列表 以空格分开
LOCAL_SRC_FILES := hello.c
# 构建动态库
include $(BUILD_SHARED_LIBRARY)
复制代码

```

我们配置好了 Android.mk 文件后如何告诉编译器这是我们的配置文件呢？ 这时候需要在 app/build.gradle 文件中进行相关的配置：

```
apply plugin: 'com.android.application'

android {
    compileSdkVersion 29

    defaultConfig {
      ...
        // 应该将源文件编译成几个 CPU so
        externalNativeBuild{
            ndkBuild{
                abiFilters 'x86','armeabi-v7a'
            }
        }
        // 需要打包进 apk 几种 so
        ndk {
            abiFilters 'x86','armeabi-v7a'
        }
    }
    // 配置 native 构建脚本位置
    externalNativeBuild{
        ndkBuild{
            path "src/main/jni/Android.mk"
        }
    }
    // 指定 ndk 版本
    ndkVersion "20.0.5594570"

    ...
}

dependencies {
    implementation fileTree(dir: "libs", include: ["*.jar"])
    ...
}

复制代码

```

Google 推荐开发者使用 cmake 来代替 makefile 进行交叉编译了，makefile 在引入第三方预编译好的 so 的时候会在 android 6.0 版本前后有些差异，比如在 6.0 之前需要手动 System.loadLibrary 第三方 so，在之后则不需要。 关于 makefile 还有很多配置参数，这里不在讲解，更多参考[官方文档](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Fguides%2Fandroid_mk "https://developer.android.com/ndk/guides/android_mk")。

> 在 6.0 以下，System.loadLibrary 不会自动加载 so 内部依赖的 so 在 6.0 以下，System.loadLibrary 会自动加载 so 内部依赖的 so 所以使用 mk 的话需要做版本兼容

### cmake

CMake 是一个跨平台的构建工具，可以用简单的语句来描述所有平台的安装 (编译过程)。能够输出各种各样的 makefile 或者 project 文件。Cmake 并不直接建构出最终的软件，而是产生其他工具的脚本（如 Makefile ），然后再依这个工具的构建方式使用。 Android Studio 利用 CMake 生成的是 ninja，ninja 是一个小型的关注速度的构建系统。我们不需要关心 ninja 的脚本，知道怎么配置 cmake 就可以了。

#### CMakeLists.txt

Make 的脚本名默认是 CMakeLists.txt，当我们用 android studio new project 勾选 include c/c++ 的时候，会默认生成以下文件：

|- app |-- src |--- main |---- cpp |----- CMakeLists.txt |----- native-lib.cpp

先来看一下 CMakeLists.txt：

```
# 设置 cmake 最小支持版本
cmake_minimum_required(VERSION 3.4.1)

# 创建一个库
add_library( # 库名称，比如现在会生成 native-lib.so
             native-lib

             # 设置是动态库（SHARED）还是静态库（STATIC）
             SHARED

             # 设置源文件的相对路径
             native-lib.cpp )
             
 # 搜索并指定预构建库并将路径存储为变量。
 # NDK中已经有一部分预构建库（比如 log），并且ndk库已经是被配置为cmake搜索路径的一部分
 # 可以不写 直接在 target_link_libraries 写上log
 find_library( # 设置路径变量的名称
              log-lib

              # 指定要CMake定位的NDK库的名称
              log )
              
 # 指定CMake应链接到目标库的库。你可以链接多个库，例如构建脚本、预构建的第三方库或系统库。
 target_link_libraries( # Specifies the target library.
                       native-lib
                       ${log-lib} )
 
           
复制代码

```

我们再来看下 gradle 中的配置：

```
android {
    compileSdkVersion 29
    buildToolsVersion "29.0.1"
    defaultConfig {
        ...
        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
        // 设置编译版本
        externalNativeBuild {
            cmake {
                abiFilters "armeabi-v7a","x86"
            }
        }
    }
    ...
    // 设置配置文件路径
    externalNativeBuild {
        cmake {
            path "src/main/cpp/CMakeLists.txt"
            version "3.10.2"
        }
    }
}
复制代码

```

这样在编译产物中就可以看到两个版本的 so：

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/6/16/172bcb4b3dd44795~tplv-t2oaga2asx-watermark.awebp)

#### 添加多个源文件

比如我们添加一个 extra.h：

```
#ifndef JNITUTORIAL_EXTRA_H
#define JNITUTORIAL_EXTRA_H
const char * getString(){
    return "string from extra";
}
#endif //JNITUTORIAL_EXTRA_H
复制代码

```

然后在 native-lib.cpp 中使用：

```
#include <jni.h>
#include <string>
#include <android/log.h>
#include "extra.h"
//  __VA_ARGS__ 代表... 可变参数
#define  LOGE(...) __android_log_print(ANDROID_LOG_ERROR,"JNI",__VA_ARGS__);

extern "C" JNIEXPORT jstring JNICALL
Java_com_wangzhen_jnitutorial_MainActivity_stringFromJNI(JNIEnv *env, jobject thiz) {
//    std::string hello = "Hello from new C++";
    std::string hello = getString();
    return env->NewStringUTF(hello.c_str());
}
复制代码

```

源文件已经写好了，这时候要修改一下 CMakeLists.txt：

```
add_library( 
             native-lib
             SHARED
             native-lib.cpp
             // 添加 extra.h 
             extra.h )
             
#==================================
# 当然如果源文件非常多，并且可能在不同的文件夹下，像上面明确的引入各个文件就会非常繁琐，此时可以批量引入

# 如果文件太多，可以批量加载，下面时将 cpp 文件夹下所有的源文件定义成了 SOURCE（后面的源文件使用相对路径）
file(GLOB SOURCE *.cpp *.h)

add_library(
        native-lib
        SHARED
        # 引入 SOURCE 下的所有源文件
        ${SOURCE}
        )
复制代码

```

#### 添加第三方动态库

那么如何添加第三方的动态库呢？

##### 第三方库的存放位置

动态库必须放到 src/main/jniLibs/xxabi 目录下才能被打包到 apk 中，这里用的是虚拟机，所以用的是 x86 平台，所以我们放置一个第三方库 libexternal.so 到 src/main/jniLibs/x86 下面。 libexternal.so 中只有一个 hello.c , 里面只有一个方法：

```
const char * getExternalString(){
    return "string from external";
}
复制代码

```

##### CMakeLists.txt 的位置

这里将 CMakeLists.txt 重新放到了 app 目录下，和 src 同级，这样方便找到 jniLibs 下面的库。 所以别忘了修改 gradle

```
externalNativeBuild {
        cmake {
            path "CMakeLists.txt"
            version "3.10.2"
        }
    }
复制代码

```

##### 配置 CMakeLists.txt

```
cmake_minimum_required(VERSION 3.4.1)

# 如果文件太多，可以批量加载，下面时将 cpp 文件夹下所有的源文件定义成了 SOURCE（后面的源文件使用相对路径）
file(GLOB SOURCE src/main/cpp/*.cpp src/main/cpp/*.h)

add_library(
        native-lib
        SHARED
        # 引入 SOURCE 下的所有源文件
        ${SOURCE}
        )
set_target_properties(native-lib PROPERTIES LINKER_LANGUAGE CXX)

#add_library( # Sets the name of the library.
#             native-lib
#
#             # Sets the library as a shared library.
#             SHARED
#
#             # Provides a relative path to your source file(s).
#             native-lib.cpp
#             extra.h )

find_library(
              log-lib
              log )

# ==================引入外部 so===================
message("ANDROID_ABI : ${ANDROID_ABI}")
message("CMAKE_SOURCE_DIR : ${CMAKE_SOURCE_DIR}")
message("PROJECT_SOURCE_DIR : ${PROJECT_SOURCE_DIR}")

# external 代表第三方 so - libexternal.so
# SHARED 代表动态库，静态库是 STATIC；
# IMPORTED: 表示是以导入的形式添加进来(预编译库)
add_library(external SHARED IMPORTED)

#设置 external 的 导入路径(IMPORTED_LOCATION) 属性,不可以使用相对路径
# CMAKE_SOURCE_DIR: 当前cmakelists.txt的路径 （cmake工具内置的）
# android cmake 内置的 ANDROID_ABI :  当前需要编译的cpu架构
set_target_properties(external PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/src/main/jniLibs/x86/libexternal.so)
#set_target_properties(external PROPERTIES LINKER_LANGUAGE CXX)

# ==================引入外部 so end===================

target_link_libraries( # Specifies the target library.
                       native-lib

                       # Links the target library to the log library
                       # included in the NDK.
                       ${log-lib}
                       # 链接第三方 so
                       external
        )

复制代码

```

##### 使用第三方库

```
#include <jni.h>
#include <string>
#include <android/log.h>
#include "extra.h"
//  __VA_ARGS__ 代表... 可变参数
#define  LOGE(...) __android_log_print(ANDROID_LOG_ERROR,"JNI",__VA_ARGS__);

extern "C"{
 const char * getExternalString();
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_wangzhen_jnitutorial_MainActivity_stringFromJNI(JNIEnv *env, jobject thiz) {
//    std::string hello = "Hello from new C++";
//    std::string hello = getString();
    // 这里调用了第三方库的方法
    std::string hello = getExternalString();
    return env->NewStringUTF(hello.c_str());
}
复制代码

```

##### 增加 CMake 查找路径

除了上面的方式还可以给 CMake 增加一个查找 so 的 path，当我们 target_link_libraries external 的时候就会在该路径下找到。

```
#=====================引入外部 so 的第二种方式===============================

# 直接给 cmake 在添加一个查找路径，在这个路径下可以找到 external

# CMAKE_C_FLAGS 代表使用 c 编译， CMAKE_CXX_FLAGS 代表 c++
# set 方法 定义一个变量 CMAKE_C_FLAGS = "${CMAKE_C_FLAGS} XXXX"
# -L: 库的查找路径 libexternal.so
#set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -L${CMAKE_SOURCE_DIR}/src/main/jniLibs/${ANDROID_ABI} ")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -L${CMAKE_SOURCE_DIR}/src/main/jniLibs/x86")
#=====================引入外部 so 的第二种方式 end===============================
复制代码

```

参考
--

[Android JNI 之 JNIEnv 解析](https://link.juejin.cn/?target=https%3A%2F%2Fblog.csdn.net%2Fhan1202012%2Farticle%2Fdetails%2F38012515 "https://blog.csdn.net/han1202012/article/details/38012515")

[操作 jarray](https://link.juejin.cn/?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F7f6dbdfbd7fd "https://www.jianshu.com/p/7f6dbdfbd7fd")

[NDK 官方资料](https://link.juejin.cn/?target=https%3A%2F%2Fdeveloper.android.com%2Fndk%2Fguides "https://developer.android.com/ndk/guides")

[NDK 中找不到 arm-linux-androideabi-gcc 的解决办法](https://link.juejin.cn/?target=https%3A%2F%2Fzhouxinan.github.io%2F2019%2F01%2F24%2FNDK%25E4%25B8%25AD%25E6%2589%25BE%25E4%25B8%258D%25E5%2588%25B0arm-linux-androideabi-gcc%25E7%259A%2584%25E8%25A7%25A3%25E5%2586%25B3%25E5%258A%259E%25E6%25B3%2595%2F "https://zhouxinan.github.io/2019/01/24/NDK%E4%B8%AD%E6%89%BE%E4%B8%8D%E5%88%B0arm-linux-androideabi-gcc%E7%9A%84%E8%A7%A3%E5%86%B3%E5%8A%9E%E6%B3%95/")

[cmake 实践](https://link.juejin.cn/?target=https%3A%2F%2Fwww.kancloud.cn%2Fitfanr%2Fcmake-practice%2F82983 "https://www.kancloud.cn/itfanr/cmake-practice/82983")