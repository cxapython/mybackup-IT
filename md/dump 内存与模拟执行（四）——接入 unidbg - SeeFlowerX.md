> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.seeflower.dev](https://blog.seeflower.dev/archives/170/)

> 前言以绿洲 v4.5.6 为例（64 位），目标是 com.weibo.xvideo.NativeApi 的 s 函数通过 frida hook RegisterNative，可以得到其 native 函数位于 l...

以`绿洲 v4.5.6`为例（64 位），目标是`com.weibo.xvideo.NativeApi`的`s`函数

通过 frida hook RegisterNative，可以得到其 native 函数位于`liboasiscore.so + 0x116CC`

在上一篇文章中，已经实现了 APP 运行到`liboasiscore.so + 0x116CC`时的上下文 dump 了

现在可以开始尝试使用 unidbg 加载上下文并模拟执行，这里使用当前最新的 0.9.7 版本

先搭个框架如下，目前这里不需要载入任何 so，仅仅初始化虚拟机即可

```
public class NativeApi extends AbstractJni {

    private final AndroidEmulator emulator;
    private final VM vm;

    NativeApi() {
        emulator = AndroidEmulatorBuilder.for64Bit()
                .setProcessName("com.sina.oasis")
                .addBackendFactory(new Unicorn2Factory(true))
                .build(); // 创建模拟器实例，要模拟32位或者64位，在这里区分
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析

        vm = emulator.createDalvikVM(); // 创建Android虚拟机
        vm.setJni(this);
        vm.setVerbose(true); // 设置是否打印Jni调用细节
    }

    public static void main(String[] args) throws Exception {
        NativeApi mNativeApi = new NativeApi();
        mNativeApi.load_context("unidbg-android/src/test/resources/DumpContext_20220806_130827");
        mNativeApi.s();
    }

    private void load_context(String dump_dir) throws IOException, DataFormatException, IOException {

    }

    private void s() {

    }

}
```

*   Q: 先思考一个问题，已经有上下文了，那么加载上下文，使用 unidbg 接着上下文运行，需要做什么？
*   A: 首先当然是还原上下文状态，即设置寄存器，加载内存
*   A: 其次是从上下文的地方开始运行
*   Q 再想想为什么要使用 unidbg 而不是直接使用 unicorn 呢？
*   A unidbg 对 jni 和 syscall 的支持相对完善，接着上下文运行前，我们将 JNIEnv 和 JavaVM 替换为 unidbg 的，那么后续遇到 jni 调用或是 syscall 调用也能处理，而 unicorn 不具备这样的优势

在`load_context`通过下面的代码恢复上下文的寄存器，这里只处理了通用寄存器，浮点寄存器暂时没有写

这里一定要向`TPIDR_EL0`写入刚才 dump 得到的那个寄存器的值

*   Q: 为什么要向`CPACR_EL1`写入`0x300000`？
*   A: 没有仔细研究... 但是 unidbg 的`enableVFP`是这样的写的:(

```
backend.reg_write(Arm64Const.UC_ARM64_REG_CPACR_EL1, 0x300000L);
backend.reg_write(Arm64Const.UC_ARM64_REG_TPIDR_EL0, 0x0000006d4aa68000L);
```

*   Q: x29 和 x30 呢，为什么没有写？
*   A: FP 就是 x29，LR 就是 x30

简略代码如下：

```
Backend backend = emulator.getBackend();
Memory memory = emulator.getMemory();

String context_file = dump_dir + "\\" + "_index.json";
InputStream is = new FileInputStream(context_file);
String jsonTxt = IOUtils.toString(is, "UTF-8");
JSONObject context = JSONObject.parseObject(jsonTxt);
JSONObject regs = context.getJSONObject("regs");

backend.reg_write(Arm64Const.UC_ARM64_REG_X0, Long.decode(regs.getString("x0")));
// ...
backend.reg_write(Arm64Const.UC_ARM64_REG_X28, Long.decode(regs.getString("x28")));

backend.reg_write(Arm64Const.UC_ARM64_REG_FP, Long.decode(regs.getString("fp")));
backend.reg_write(Arm64Const.UC_ARM64_REG_LR, Long.decode(regs.getString("lr")));
backend.reg_write(Arm64Const.UC_ARM64_REG_SP, Long.decode(regs.getString("sp")));
backend.reg_write(Arm64Const.UC_ARM64_REG_PC, Long.decode(regs.getString("pc")));
backend.reg_write(ArmConst.UC_ARM_REG_CPSR, Long.decode(regs.getString("cpsr")));

backend.reg_write(Arm64Const.UC_ARM64_REG_CPACR_EL1, 0x300000L);
backend.reg_write(Arm64Const.UC_ARM64_REG_TPIDR_EL0, 0x0000006d4aa68000L);

// 好像不设置这个也不会有什么影响
memory.setStackPoint(Long.decode(regs.getString("sp")));
```

在`load_context`通过下面的代码恢复部分上下文内存

dump 下来的内存很多，如果全部加载会非常耗时，实际上我们需要模拟执行的代码往往只有一小部分，所以通过白名单机制加载

这里从配置文件中读取配置，只加载下面这些内存段

*   libc.so
*   liboasiscore.so
*   [anon:stack_and_tls:32529]
*   [anon:.bss]
*   [anon:scudo:primary]
*   Q: 这些内存段是如何确定下来的呢？
*   A: 最开始可以只加载`libc.so`和`liboasiscore.so`相关的，然后直接模拟执行看哪些地方出现报错，再去计算确定是哪些段没有加载

一般会涉及到`libc.so`和`上下文位置的so`

然后是`[anon:stack_and_tls:32529]`，这个需要执行测试确定

然后是`[anon:scudo:primary]`和`[anon:scudo:secondary]`(Android 11 之前是`[anon:libc_malloc]`)

再然后是`[anon:.bss]`

有很多分段都是相同名字，在完成算法成功模拟执行后，可以记录下访问了那些块，再改为通过`content_file`来加载内存段还能减少内存占用

逻辑很简单，就是根据配置文件 + 白名单机制先调用`mem_map`开辟内存空间，再调用`mem_write`写入内存数据

记得写内存数据之前先从文件中解压~

部分代码片段如下：

```
int UNICORN_PAGE_SIZE = 0x1000;

private long align_page_down(long x){
    return x & ~(UNICORN_PAGE_SIZE - 1);
}
private long align_page_up(long x){
    return (x + UNICORN_PAGE_SIZE - 1) & ~(UNICORN_PAGE_SIZE - 1);
}
private void map_segment(long address, long size, int perms){

    long mem_start = address;
    long mem_end = address + size;
    long mem_start_aligned = align_page_down(mem_start);
    long mem_end_aligned = align_page_up(mem_end);

    if (mem_start_aligned < mem_end_aligned){
        emulator.getBackend().mem_map(mem_start_aligned, mem_end_aligned - mem_start_aligned, perms);
    }
}
```

```
JSONArray segments = context.getJSONArray("segments");

for (int i = 0; i < segments.size(); i++) {
    JSONObject segment = segments.getJSONObject(i);
    String path = segment.getString("name");
    long start = segment.getLong("start");
    long end = segment.getLong("end");
    String content_file = segment.getString("content_file");
    JSONObject permissions = segment.getJSONObject("permissions");
    int perms = 0;
    if (permissions.getBoolean("r")){
        perms |= UnicornConst.UC_PROT_READ;
    }
    if (permissions.getBoolean("w")){
        perms |= UnicornConst.UC_PROT_WRITE;
    }
    if (permissions.getBoolean("x")){
        perms |= UnicornConst.UC_PROT_EXEC;
    }

    String[] paths = path.split("/");
    String module_name = paths[paths.length - 1];

    List<String> white_list = Arrays.asList(new String[]{"liboasiscore.so", "libc.so", "[anon:stack_and_tls:32529]", "[anon:.bss]", "[anon:scudo:primary]"});
    if (white_list.contains(module_name)){
        int size = (int)(end - start);

        map_segment(start, size, perms);
        String content_file_path = dump_dir + "\\" + content_file;

        File content_file_f = new File(content_file_path);
        if (content_file_f.exists()){
            InputStream content_file_is = new FileInputStream(content_file_path);
            byte[] content_file_buf = IOUtils.toByteArray(content_file_is);

            // zlib解压
            Inflater decompresser = new Inflater();
            decompresser.setInput(content_file_buf, 0, content_file_buf.length);
            byte[] result = new byte[size];
            int resultLength = decompresser.inflate(result);
            decompresser.end();

            backend.mem_write(start, result);
        }
        else {
            System.out.println("not exists path=" + path);
            byte[] fill_mem = new byte[size];
            Arrays.fill( fill_mem, (byte) 0 );
            backend.mem_write(start, fill_mem);
        }

    }
}
```

自此已经完成上下文的恢复了

现在可以开始模拟执行了，还记得前面的问题吗，我们要替换掉原来的 JNIEnv 和 JavaVM，还有其他传入参数

不过遇到的第一个问题是没有`module`，用过 unidbg 的同学都知道肯定会在开始的时候加载某个 so，不过这里我们并没有加载任何 so

经过分析，发现可以使用`Module.emulateFunction`这样来调用，这个问题顺利解决

于是修改参数，接着上下文的状态模拟执行的代码实现如下：

```
private void s() {
    List<Object> list = new ArrayList<>(4);

//    参数一 JNIEnv* env
    list.add(vm.getJNIEnv());

//    参数二 jobject thiz
    DvmClass NativeApiobj = vm.resolveClass("com/weibo/xvideo/NativeApi");
    list.add(NativeApiobj.hashCode());

//    参数三 java 层的参数一
    String data = "aid=01AxlUJKR0Ty44wiNo-ebcin69clFdov931m6rKA-DoQZ7Pkk.&cfrom=28C7295010&cuid=0&noncestr=g8g1N6V3t49z943Hx80395kb63f42A&platform=ANDROID×tamp=1659164634618&ua=Google-Pixel4__oasis__4.5.6__Android__Android11&version=4.5.6&vid=2007759688214&wm=2468_90123";
    ByteArray input_array = new ByteArray(vm, data.getBytes(StandardCharsets.UTF_8));
    vm.addLocalObject(input_array);
    list.add(input_array.hashCode());

//    参数四 java 层的参数二
    boolean flag = false;
    list.add((Boolean) flag ? VM.JNI_TRUE : VM.JNI_FALSE);

//    这里获取 dump 时的 pc 地址作为模拟执行起始地址
    long ctx_addr = emulator.getBackend().reg_read(Arm64Const.UC_ARM64_REG_PC).longValue();

//    开始模拟执行
    Number result = Module.emulateFunction(emulator, ctx_addr, list.toArray());

//    获取返回结果
    String sign_str = (String) vm.getObject(result.intValue()).getValue();
    System.out.println("sign_str=" + sign_str);
}
```

这里的关键点在于模拟执行的起始地址是 dump 上下文时的 pc 地址

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_23-55-06.png)

模拟执行结果和 hook 的结果一致

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_23-55-52.png)

在拿到上下文之后，使用 unidbg 的 JNIEnv、JavaVM 和其他由 unidbg 构造的参数对原上下文的参数进行替换

一定程度上完成了原上下文的接管

虽然没有主动完成上下文初始化（即 libc 和目标 so 的初始化），但这样操作可以使后续 jni 调用和 syscall 都落入 unidbg 的逻辑之中

相比自己去做目标 so 的初始化，通过 dump 上下文，可以一定程度上减少工作量，同时更贴近于真实情况

除了 unidbg，也可以使用相同的手法接入`ExAndroidNativeEmu`

*   Q: 还有什么需要处理？
*   A: 经过实践，对于 Android 10 之后的系统，libc.so 有些不一样，无法完全模拟执行，推测是 unicorn 本身对一些指令支持不够完善。
    
    *   关于这个问题，我想可以先用 unidbg 初始化自己的 libc，然后 dump 下来的模拟执行过程中如果遇到了 libc 的调用，将这些调用交给 unidbg 的 libc 处理
*   Q: 涉及复杂的函数效果如何？
*   A: 对于算法来说，一般不会特别特别复杂，通常只需要 hook 住 libc 的部分函数，再针对性 hook 一些函数即可
    
    *   经过实践，以快手的 sig 来说（10.6.50.26734），使用 dump 上下文方案，再分别对以下点位 hook 和补充后，就能跑出结果
        
        *   libc.so gettimeofday **手动实现**
        *   libc.so pthread_mutex_lock **跳过执行**
        *   libc.so pthread_mutex_unlock **跳过执行**
        *   libc.so malloc **手动实现**
        *   libc.so calloc **手动实现**
        *   libc.so free **手动实现**
        *   libc++_shared.so std::__libcpp_snprintf_l **特殊处理**
        *   替换原有`com/kuaishou/android/security/internal/common/ExceptionProxy`类 **特殊处理**
        *   两个需要补充的 jni 调用 **常规补环境**

![](https://blog.seeflower.dev/images/Snipaste_2022-08-07_18-24-52.png)

```
package com.weibo.xvideo;

import com.alibaba.fastjson.JSONArray;
import com.alibaba.fastjson.JSONObject;
import com.github.unidbg.AndroidEmulator;
import com.github.unidbg.Module;
import com.github.unidbg.arm.backend.Backend;
import com.github.unidbg.arm.backend.Unicorn2Factory;
import com.github.unidbg.linux.android.AndroidEmulatorBuilder;
import com.github.unidbg.linux.android.AndroidResolver;
import com.github.unidbg.linux.android.dvm.AbstractJni;
import com.github.unidbg.linux.android.dvm.DvmClass;
import com.github.unidbg.linux.android.dvm.VM;
import com.github.unidbg.linux.android.dvm.array.ByteArray;
import com.github.unidbg.memory.Memory;
import org.apache.commons.io.IOUtils;
import unicorn.Arm64Const;
import unicorn.ArmConst;
import unicorn.UnicornConst;

import java.io.*;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.zip.DataFormatException;
import java.util.zip.Inflater;

public class NativeApi extends AbstractJni {

    private final AndroidEmulator emulator;
    private final VM vm;

    NativeApi() {
        emulator = AndroidEmulatorBuilder.for64Bit()
                .setProcessName("com.sina.oasis")
                .addBackendFactory(new Unicorn2Factory(true))
                .build(); // 创建模拟器实例，要模拟32位或者64位，在这里区分
        final Memory memory = emulator.getMemory(); // 模拟器的内存操作接口
        memory.setLibraryResolver(new AndroidResolver(23)); // 设置系统类库解析

        vm = emulator.createDalvikVM(); // 创建Android虚拟机
        vm.setJni(this);
        vm.setVerbose(true); // 设置是否打印Jni调用细节
    }

    public static void main(String[] args) throws Exception {
        NativeApi mNativeApi = new NativeApi();
        mNativeApi.load_context("unidbg-android/src/test/resources/DumpContext_20220806_130827");
        mNativeApi.s();
    }

    int UNICORN_PAGE_SIZE = 0x1000;

    private long align_page_down(long x){
        return x & ~(UNICORN_PAGE_SIZE - 1);
    }
    private long align_page_up(long x){
        return (x + UNICORN_PAGE_SIZE - 1) & ~(UNICORN_PAGE_SIZE - 1);
    }

    private void map_segment(long address, long size, int perms){

        long mem_start = address;
        long mem_end = address + size;
        long mem_start_aligned = align_page_down(mem_start);
        long mem_end_aligned = align_page_up(mem_end);

        if (mem_start_aligned < mem_end_aligned){
            emulator.getBackend().mem_map(mem_start_aligned, mem_end_aligned - mem_start_aligned, perms);
        }
    }

    private void load_context(String dump_dir) throws IOException, DataFormatException, IOException {

        Backend backend = emulator.getBackend();
        Memory memory = emulator.getMemory();

        String context_file = dump_dir + "\\" + "_index.json";
        InputStream is = new FileInputStream(context_file);
        String jsonTxt = IOUtils.toString(is, "UTF-8");
        JSONObject context = JSONObject.parseObject(jsonTxt);
        JSONObject regs = context.getJSONObject("regs");

        backend.reg_write(Arm64Const.UC_ARM64_REG_X0, Long.decode(regs.getString("x0")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X1, Long.decode(regs.getString("x1")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X2, Long.decode(regs.getString("x2")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X3, Long.decode(regs.getString("x3")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X4, Long.decode(regs.getString("x4")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X5, Long.decode(regs.getString("x5")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X6, Long.decode(regs.getString("x6")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X7, Long.decode(regs.getString("x7")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X8, Long.decode(regs.getString("x8")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X9, Long.decode(regs.getString("x9")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X10, Long.decode(regs.getString("x10")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X11, Long.decode(regs.getString("x11")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X12, Long.decode(regs.getString("x12")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X13, Long.decode(regs.getString("x13")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X14, Long.decode(regs.getString("x14")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X15, Long.decode(regs.getString("x15")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X16, Long.decode(regs.getString("x16")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X17, Long.decode(regs.getString("x17")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X18, Long.decode(regs.getString("x18")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X19, Long.decode(regs.getString("x19")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X20, Long.decode(regs.getString("x20")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X21, Long.decode(regs.getString("x21")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X22, Long.decode(regs.getString("x22")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X23, Long.decode(regs.getString("x23")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X24, Long.decode(regs.getString("x24")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X25, Long.decode(regs.getString("x25")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X26, Long.decode(regs.getString("x26")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X27, Long.decode(regs.getString("x27")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_X28, Long.decode(regs.getString("x28")));

        backend.reg_write(Arm64Const.UC_ARM64_REG_FP, Long.decode(regs.getString("fp")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_LR, Long.decode(regs.getString("lr")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_SP, Long.decode(regs.getString("sp")));
        backend.reg_write(Arm64Const.UC_ARM64_REG_PC, Long.decode(regs.getString("pc")));
        backend.reg_write(ArmConst.UC_ARM_REG_CPSR, Long.decode(regs.getString("cpsr")));

        backend.reg_write(Arm64Const.UC_ARM64_REG_CPACR_EL1, 0x300000L);
        backend.reg_write(Arm64Const.UC_ARM64_REG_TPIDR_EL0, 0x0000006d4aa68000L);

//        好像不设置这个也不会有什么影响
        memory.setStackPoint(Long.decode(regs.getString("sp")));

        JSONArray segments = context.getJSONArray("segments");

        for (int i = 0; i < segments.size(); i++) {
            JSONObject segment = segments.getJSONObject(i);
            String path = segment.getString("name");
            long start = segment.getLong("start");
            long end = segment.getLong("end");
            String content_file = segment.getString("content_file");
            JSONObject permissions = segment.getJSONObject("permissions");
            int perms = 0;
            if (permissions.getBoolean("r")){
                perms |= UnicornConst.UC_PROT_READ;
            }
            if (permissions.getBoolean("w")){
                perms |= UnicornConst.UC_PROT_WRITE;
            }
            if (permissions.getBoolean("x")){
                perms |= UnicornConst.UC_PROT_EXEC;
            }

            String[] paths = path.split("/");
            String module_name = paths[paths.length - 1];

            List<String> white_list = Arrays.asList(new String[]{"liboasiscore.so", "libc.so", "[anon:stack_and_tls:32529]", "[anon:.bss]", "[anon:scudo:primary]"});
            if (white_list.contains(module_name)){
                int size = (int)(end - start);

                map_segment(start, size, perms);
                String content_file_path = dump_dir + "\\" + content_file;

                File content_file_f = new File(content_file_path);
                if (content_file_f.exists()){
                    InputStream content_file_is = new FileInputStream(content_file_path);
                    byte[] content_file_buf = IOUtils.toByteArray(content_file_is);

                    // 解压
                    Inflater decompresser = new Inflater();
                    decompresser.setInput(content_file_buf, 0, content_file_buf.length);
                    byte[] result = new byte[size];
                    int resultLength = decompresser.inflate(result);
                    decompresser.end();

                    backend.mem_write(start, result);
                }
                else {
                    System.out.println("not exists path=" + path);
                    byte[] fill_mem = new byte[size];
                    Arrays.fill( fill_mem, (byte) 0 );
                    backend.mem_write(start, fill_mem);
                }

            }
        }

    }

    private void s() {
        List<Object> list = new ArrayList<>(4);

//        参数一 JNIEnv* env
        list.add(vm.getJNIEnv());

//        参数二 jobject thiz
        DvmClass NativeApiobj = vm.resolveClass("com/weibo/xvideo/NativeApi");
        list.add(NativeApiobj.hashCode());

//        参数三 java 层的参数一
        String data = "aid=01AxlUJKR0Ty44wiNo-ebcin69clFdov931m6rKA-DoQZ7Pkk.&cfrom=28C7295010&cuid=0&noncestr=g8g1N6V3t49z943Hx80395kb63f42A&platform=ANDROID×tamp=1659164634618&ua=Google-Pixel4__oasis__4.5.6__Android__Android11&version=4.5.6&vid=2007759688214&wm=2468_90123";
        ByteArray input_array = new ByteArray(vm, data.getBytes(StandardCharsets.UTF_8));
        vm.addLocalObject(input_array);
        list.add(input_array.hashCode());

//        参数四 java 层的参数二
        boolean flag = false;
        list.add((Boolean) flag ? VM.JNI_TRUE : VM.JNI_FALSE);

//        这里获取 dump 时的 pc 地址作为模拟执行起始地址
        long ctx_addr = emulator.getBackend().reg_read(Arm64Const.UC_ARM64_REG_PC).longValue();

//        开始模拟执行
        Number result = Module.emulateFunction(emulator, ctx_addr, list.toArray());

//        获取返回结果
        String sign_str = (String) vm.getObject(result.intValue()).getValue();
        System.out.println("sign_str=" + sign_str);
    }

}
```

  
星球广告 & 交流群  
![](https://blog.seeflower.dev/images/xqyh.png)