> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-272751.htm)

> [原创] 拼夕夕 anti-token 分析

前言：拼夕夕 charles 抓包分析发现跟商品相关的请求头里都带了一个 anti-token 的字段且每次都不一样, 那么下面的操作就从分析 anti-token 开始了

### 1.jadx 反编译直接搜索

![](https://bbs.pediy.com/upload/attach/202205/884888_DT5PUXYW9PP9RB8.jpg)

选中跟 http 相关的类对这个方法进行打印堆栈

![](https://bbs.pediy.com/upload/attach/202205/884888_UKC5WA6HA6JHHCF.jpg)

![](https://bbs.pediy.com/upload/attach/202205/884888_ANCP6GWMUAFE6DW.jpg)

![](https://bbs.pediy.com/upload/attach/202205/884888_3UNHUFYT5HQBXTE.jpg)

结合堆栈方法调用的情况找到具体 anti-token 是由拦截器类 f.a 方法调用的, 在 http.a.c() 方法中生成并且 http.p.e() 方法中加入请求头

在 http.a.c() 方法中有个一个判断条件如果为 true 则走 d.a().e() 方法生成 anti-token

![](https://bbs.pediy.com/upload/attach/202205/884888_2922MVU795UA443.jpg)如果为 false 则走 j() 方法生成 anti-token

![](https://bbs.pediy.com/upload/attach/202205/884888_H5PQ89N83MTW3SK.jpg)

hook 这个 i() 方法返回值可知获取商品详情接口返回值为 false 所以走的是 j() 方法进行计算 anti-token。

![](https://bbs.pediy.com/upload/attach/202205/884888_M3KDHT6J3M43MFG.jpg)

SecureNative.deviceInfo3() 方法生成, 传入的 str 为 pdd 生成的固定 id 一个字符串.

![](https://bbs.pediy.com/upload/attach/202205/884888_KWESWRKN5TNA3S7.jpg)![](https://bbs.pediy.com/upload/attach/202205/884888_CCU9KTAAZW5S6VM.jpg)![](https://bbs.pediy.com/upload/attach/202205/884888_6CZUB9VBWYZ5QGC.jpg)

根据 hook_libart 得到 info3() 方法是在 libodd_secure.so 中, 那么 ida 打开看看这个 so 包

![](https://bbs.pediy.com/upload/attach/202205/884888_QJYC5G6V7JKPSG9.jpg)

### 2. 这部分我们采用 unidbg+jnitrace+frida 相结合的方式

unidbg 前期准备的代码这里就不发了直接调用这个 info3 方法

![](https://bbs.pediy.com/upload/attach/202205/884888_MGENH7FDXYQ684H.jpg)  

这里提示调用 gad() 方法返回一个字符串那么 frida hook 这个方法拿到这个值 如下图 一个固定的字符串

16 位长度看着像 AES 的密钥 ![](https://bbs.pediy.com/upload/attach/202205/884888_X3MA44PA8XWGBDP.jpg)

继续补环境 这里简单补环境就不发了

![](https://bbs.pediy.com/upload/attach/202205/884888_HHA86Q94NXJVY8X.jpg)

补完简单的环境代码后, 再次运行报这个错误 看错误应该是缺少文件 , 那么看看日志需要补那个文件

继续运行, 没有返回值报空指针。execve() 函数执行的时候程序 exit 了这里我们返回对象本身.![](https://bbs.pediy.com/upload/attach/202205/884888_ZEY95HDEZXR58SE.jpg)

execve() 函数执行的时候程序 exit 了![](https://bbs.pediy.com/upload/attach/202205/884888_A9W6FB76HJEBN4Z.jpg)  

execve filename=/system/bin/sh, args=[sh, -c, cat /proc/sys/kernel/random/boot_id]

这个函数相当于查看 boot_id 这个文件信息

![](https://bbs.pediy.com/upload/attach/202205/884888_4ZCKQRWR6UFZW56.jpg)

捋顺下逻辑应该就是先 fork 进程 然后在子进程中读取这个文件 然后把他写入 pip 中

那么自定义 syscallhandler 后 再次运行成功拿到结果

![](https://bbs.pediy.com/upload/attach/202205/884888_DQEVSSB6BG33NQM.jpg)

全部代码如下：

```
package pdd;
 
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Emulator;
import com.github.unidbg.Module;
import com.github.unidbg.file.FileResult;
import com.github.unidbg.file.IOResolver;
import com.github.unidbg.file.linux.AndroidFileIO;
import com.github.unidbg.linux.android.AndroidARMEmulator;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.*;
import com.github.unidbg.linux.file.ByteArrayFileIO;
import com.github.unidbg.memory.Memory;
import com.github.unidbg.memory.SvcMemory;
import com.github.unidbg.spi.SyscallHandler;
import com.github.unidbg.unix.UnixSyscallHandler;
 
import java.io.File;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;
 
public class Pddmain extends AbstractJni implements IOResolver {
 
    private AndroidEmulator androidEmulator;
    private static final String APK_PATH = "/Users/Downloads/com.xunmeng.pinduoduo_6.7.0_60700.apk";
    private static final String SO_PATH = "/Users/Downloads/com.xunmeng.pinduoduo_6.7.0_60700/lib/armeabi-v7a/libpdd_secure.so";
    private Module moduleModule;
    private VM dalvikVM;
 
    public static void main(String[] args) {
        Pddmain main = new Pddmain();
        main.create();
    }
 
    private void create() {
        AndroidEmulatorBuilder androidEmulatorBuilder = new AndroidEmulatorBuilder(false) {
            @Override
            public AndroidEmulator build() {
                return new AndroidARMEmulator("com.xunmeng.pinduoduo",rootDir,backendFactories) {
                    @Override
                    protected UnixSyscallHandler createSyscallHandler(SvcMemory svcMemory) {
                        return new PddArmSysCallHand(svcMemory);
                    }
                };
            }
        };
        androidEmulator = androidEmulatorBuilder.setProcessName("").build();
        androidEmulator.getSyscallHandler().addIOResolver(this);
        Memory androidEmulatorMemory = androidEmulator.getMemory();
        androidEmulatorMemory.setLibraryResolver(new AndroidResolver(23));
        dalvikVM = androidEmulator.createDalvikVM(new File(APK_PATH));
        DalvikModule module = dalvikVM.loadLibrary(new File(SO_PATH), true);
        moduleModule = module.getModule();
        dalvikVM.setJni(this);
        dalvikVM.setVerbose(true);
        dalvikVM.callJNI_OnLoad(androidEmulator, moduleModule);
        callInfo3();
    }
 
    @Override
    public void callStaticVoidMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        if ("com/tencent/mars/xlog/PLog->i(Ljava/lang/String;Ljava/lang/String;)V".equals(signature)) {
            return;
        }
        super.callStaticVoidMethodV(vm, dvmClass, signature, vaList);
    }
 
    private void callInfo3() {
        List argList = new ArrayList<>();
        argList.add(dalvikVM.getJNIEnv());
        argList.add(0);
        DvmObject context = dalvikVM.resolveClass("android/content/Context").newObject(null);
        argList.add(dalvikVM.addLocalObject(context));
        argList.add(dalvikVM.addLocalObject(new StringObject(dalvikVM, "api/oak/integration/render")));
        argList.add(dalvikVM.addLocalObject(new StringObject(dalvikVM, "dIrjGpkC")));
        Number number = moduleModule.callFunction(androidEmulator, 0xb6f9, argList.toArray())[0];
        String toString = dalvikVM.getObject(number.intValue()).getValue().toString();
        System.out.println(toString);
    }
 
    @Override
    public DvmObject callStaticObjectMethodV(BaseVM vm, DvmClass dvmClass, String signature, VaList vaList) {
        if ("com/xunmeng/pinduoduo/secure/EU->gad()Ljava/lang/String;".equals(signature)) {
            return new StringObject(vm, "cb14a9e76b72a627");
        } else if ("java/util/UUID->randomUUID()Ljava/util/UUID;".equals(signature)) {
            UUID uuid = UUID.randomUUID();
            DvmObject dvmObject = vm.resolveClass("java/util/UUID").newObject(uuid);
            return dvmObject;
        }
        return super.callStaticObjectMethodV(vm, dvmClass, signature, vaList);
    }
 
    @Override
    public DvmObject dvmObject, String signature, VaList vaList) {
        if ("java/util/UUID->toString()Ljava/lang/String;".equals(signature)) {
            UUID uuid = (UUID) dvmObject.getValue();
            return new StringObject(vm, uuid.toString());
        } else if ("java/lang/String->replaceAll(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;".equals(signature)) {
            String obj = dvmObject.getValue().toString();
            String arg0 = vaList.getObjectArg(0).toString();
            String arg1 = vaList.getObjectArg(1).toString();
            String replaceAll = obj.replaceAll(arg0, arg1);
            return new StringObject(vm, replaceAll);
 
        }
        return super.callObjectMethodV(vm, dvmObject, signature, vaList);
    }
 
    @Override
    public int callIntMethodV(BaseVM vm, DvmObject dvmObject, String signature, VaList vaList) {
        if ("java/lang/String->hashCode()I".equals(signature)) {
            return dvmObject.getValue().toString().hashCode();
        }
        return super.callIntMethodV(vm, dvmObject, signature, vaList);
    }
 
    @Override
    public FileResult resolve(Emulator emulator, String pathname, int oflags) {
        if ("/proc/stat".equals(pathname)) {
            String info = "cpu  15884810 499865 12934024 24971554 59427 3231204 945931 0 0 0\n" +
                    "cpu0 6702550 170428 5497985 19277857 45380 1821584 529454 0 0 0\n" +
                    "cpu1 4438333 121907 3285784 1799772 3702 504395 255852 0 0 0\n" +
                    "cpu2 2735453 133666 2450712 1812564 4626 538114 93763 0 0 0\n" +
                    "cpu3 2008473 73862 1699542 2081360 5716 367109 66860 0 0 0\n" +
                    "intr 1022419954 0 0 0 159719900 0 16265892 4846825 5 5 5 6 0 0 497 24817167 17 176595 1352 0 28375276 0 0 0 0 5239 698 0 0 0 0 0 0 3212852 0 12195284 0 0 0 0 0 43 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 12513 2743129 375 12477726 0 0 0 0 37 1351794 0 36 8 0 0 0 0 0 0 5846 0 0 0 0 0 0 0 0 0 141 32 0 55 0 0 0 0 0 0 0 0 18 0 18 0 0 0 0 0 0 66 0 0 0 0 0 0 0 77 0 166 0 0 0 0 0 394 0 0 0 0 0 1339137 0 0 0 0 0 0 313 0 0 0 55759 7 7 7 0 0 0 0 0 0 0 0 3066136 0 47 0 0 0 2 2 0 0 0 6 8 0 0 0 2 0 462 2952327 35420 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 495589 0 0 0 0 3 27 0 0 0 0 0 0 0 0 0 0 0 0 0 0 37662 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 4760 0 0 97 0 0 0 0 0 0 0 0 0 243 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 4649 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 22355451 0 0 0 14 0 24449357 96 49415 2 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 17067 780222 3211 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 2 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 1 1 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 649346 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0\n" +
                    "ctxt 1572087931\n" +
                    "btime 1649910663\n" +
                    "processes 230673\n" +
                    "procs_running 6\n" +
                    "procs_blocked 0\n" +
                    "softirq 374327567 12481657 139161248 204829 7276312 2275183 26796 12851725 80988196 1422751 117638870";
            return FileResult.success(new ByteArrayFileIO(oflags, pathname, info.getBytes(StandardCharsets.UTF_8)));
        }
        return null;
    }
} 
```

[[2022 夏季班]《安卓高级研修班 (网课)》月薪三万班招生中～](https://www.kanxue.com/book-section_list-84.htm)

最后于 2022-5-18 13:23 被那年没下雪编辑 ，原因：

[#逆向分析](forum-161-1-118.htm) [#协议分析](forum-161-1-120.htm)