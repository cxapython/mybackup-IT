> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [xxr0ss.github.io](https://xxr0ss.github.io/post/frida_detection/)

> NOTICE: 本文提到的手段的代码实现在我的代码库 xxr0ss/AntiFrida 中， 不详细之处，请参照代码实现。

**NOTICE:** 本文提到的手段的代码实现在我的代码库 [xxr0ss/AntiFrida](https://github.com/xxr0ss/AntiFrida) 中， 不详细之处，请参照代码实现。

来看看 Frida 工作的过程
---------------

[OSDC 2015](https://web.archive.org/web/20160803154827/http://act.osdc.no/osdc2015no/ "OSDC 2015"): [The engineering behind the reverse engineering](https://web.archive.org/web/20160413183418/http://act.osdc.no/osdc2015no/talk/6195 "The engineering behind the reverse engineering") ([PDF](https://frida.re/slides/osdc-2015-the-engineering-behind-the-reverse-engineering.pdf "PDF") · [Recording](https://youtu.be/uc1mbN9EJKQ "Recording"))

概括来说就是，frida-agent 是 frida 实现注入时，注入进程的模块，和 frida-server 通信完成 frida 的功能。

在 frida 注入模式不可用时，我们还可以通过 frida-gadget 来完成插桩工作，有如下几种实现方式：

*   修改程序源码使其加载 frida-agent
    
*   Patch 程序或程序加载的库
    
*   使用一些动态链接的特性，比如`LD_PRELOAD`和`DYLD_INSERT_LIBRARIES`
    

创建 socket 去连接指定端口（27042），如果 frida-server 在默认端口下运行，则可以连接成功。

缺陷是 frida-server 可以直接指定端口

```
frida-server -l 127.0.0.1:2333
```

Native 代码
---------

```
extern "C"
JNIEXPORT jboolean JNICALL
Java_com_xxr0ss_antifrida_utils_AntiFridaUtil_checkFridaByPort(JNIEnv *env, jobject thiz,
                                                               jint port) {
    struct sockaddr_in sa{};
    sa.sin_family = AF_INET;
    sa.sin_port = htons(port);
    inet_aton("127.0.0.1", &sa.sin_addr);
    int sock = socket(AF_INET, SOCK_STREAM, 0);

    if (connect(sock, (struct sockaddr *) &sa, sizeof(sa)) == 0) {
        // we can connect to frida-server when it's running
        close(sock);
        return JNI_TRUE;
    }

    return JNI_FALSE;
}
```

注意
--

需要在`AndroidManifest.xml`中添加网络权限

```
<uses-permission android: />
```

也就是要检测 frida-agent 和 frida-gadget 的存在。在哪检测呢？答案是`/proc/self/maps` Linux 下存在一个伪文件系统`/proc`，可用通过`/proc/<PID>`来访问指定进程的各种信息。把 pid 换成`self`，进程就可以读取自身信息。

`maps`中包含进程内存地址空间映射信息

Java 层
------

以 Kotlin 为例，直接读取`/proc/self/maps`，看看当前进程地址空间是不是包含 frida-agent

```
fun readProcMaps(): String {
    try{
        val mapsFile = File("/proc/self/maps")
        return mapsFile.readText()
    }catch (e: Exception) {
        Log.e(TAG, e.stackTraceToString())
    }
    return ""
}
```

Native 层
--------

说一下项目文件结构，不然后面容易迷惑

```
main
 ├── AndroidManifest.xml
 ├── cpp
 │   ├── antifrida.cpp
 │   ├── bionic_asm.h
 │   ├── CMakeLists.txt
 │   └── syscall.S
 ├── java
 └── res
```

和前面一样，也是读取`/proc/self/maps`，不过这层的便利之处在于：

*   可以自行实现系统调用，一定程度上规避 hook
    
*   可以直接扫描内存中的模块是否存在特定字符串，避免 frida 模块通过改个名就绕过
    

最简单的写法，其实就是循环读取文件，然后检查关键字

```
char line[512];
FILE* fp;
fp = fopen("/proc/self/maps", "r");
if (fp) {
    while (fgets(line, 512, fp)) {
        if (strstr(line, "frida")) {
            /* Evil library is loaded. Do something… */
        }
    }
    fclose(fp);
    } else {
       /* Error opening /proc/self/maps. If this happens, something is off. */
    }
}
```

但这里我想要实现将`maps`内容返回 Java 层，所以实现稍微复杂了一点点。

```
JNIEXPORT jstring JNICALL
Java_com_xxr0ss_antifrida_utils_AntiFridaUtil_nativeReadProcMaps(JNIEnv *env, jobject thiz,
                                                                 jboolean useCustomizedSyscall) {
    char *data = nullptr;
    size_t data_size = 0;

    int res = read_pseudo_file_at(MAPS_FILE, &data, &data_size, useCustomizedSyscall);
    if (res == -1) {
        __android_log_print(ANDROID_LOG_ERROR, TAG,
                            "read_pseudo_file %s failed, errno %s: %d",
                            MAPS_FILE, strerror(errno), errno);
        if (data) {
            free(data);
        }
        return nullptr;
    } else if (res == 0) {
        __android_log_print(ANDROID_LOG_INFO, TAG, "read_pseudo_file had read 0 bytes");
        if (data) {
            free(data);
        }
        return nullptr;
    }
    jstring str = env->NewStringUTF(data);
    free(data);
    return str;
}
```

`read_pseudo_file_at()`的实现：

*   `openat()`打开文件
    
*   `read()`读取文件
    
*   动态增长内存以读入文件
    

```
/*  Read pseudo files in paths like /proc /sys
 *  *buf_ptr can be existing dynamic memory or nullptr (if so, this function
 *  will alloc memory automatically).
 *  remember to free the *buf_ptr because in no cases will *buf_ptr be
 *  freed inside this function
 *  return -1 on error, or non-negative value on success
 * */
int read_pseudo_file_at(const char *path, char **buf_ptr, size_t *buf_size_ptr,
                        bool use_customized_syscalls) {
    if (!path || !*path || !buf_ptr || !buf_size_ptr) {
        errno = EINVAL;
        return -1;
    }

    char *buf;
    size_t buf_size, total_read_size = 0;

    /* Existing dynamic buffer, or a new buffer? */
    buf_size = *buf_size_ptr;
    if (!buf_size)
        *buf_ptr = nullptr;
    buf = *buf_ptr;

    /* Open pseudo file */
    int fd = use_customized_syscalls ?
             my_openat(AT_FDCWD, MAPS_FILE, O_RDONLY | O_CLOEXEC, 0)
                                     : openat(AT_FDCWD, MAPS_FILE, O_RDONLY | O_CLOEXEC, 0);

    if (fd == -1) {
        __android_log_print(ANDROID_LOG_INFO, TAG, "openat error %s : %d", strerror(errno), errno);
        return -1;
    }

    while (true) {
        if (total_read_size >= buf_size) {
            /* linear size growth
             * buf_size grow ~4k bytes each time, 32 bytes for zero padding
             * */
            buf_size = (total_read_size | 4095) + 4097 - 32;
            buf = (char *) realloc(buf, buf_size);
            if (!buf) {
                close(fd);
                errno = ENOMEM;
                return -1;
            }
            *buf_ptr = buf;
            *buf_size_ptr = buf_size;
        }

        size_t n = use_customized_syscalls ?
                   my_read(fd, buf + total_read_size, buf_size - total_read_size)
                                           : read(fd, buf + total_read_size,
                                                  buf_size - total_read_size);
        if (n > 0) {
            total_read_size += n;
        } else if (n == 0) {
            break;
        } else if (n == -1) {
            const int saved_errno = errno;
            close(fd);
            errno = saved_errno;
            return -1;
        }
    }

    if (close(fd) == -1) {
        /* errno set by close(). */
        return -1;
    }

    if (total_read_size + 32 > buf_size)
        memset(buf + total_read_size, 0, 32);
    else
        memset(buf + total_read_size, 0, buf_size - total_read_size);

    errno = 0;
    return (int)total_read_size;
}
```

`use_customized_syscalls`指定是否使用自己实现的系统调用。

### 系统调用的自定义实现

Syscall 在 Android 上的实现可以参照 [Gityuan 博客](http://gityuan.com/2016/05/21/syscall/ "Gityuan博客")

> 在用户空间和内核空间之间，有一个叫做 Syscall(系统调用, system call) 的中间层，是连接用户态和内核态的桥梁。这样即提高了内核的安全型，也便于移植，只需实现同一套接口即可。Linux 系统，用户空间通过向内核空间发出 Syscall，产生软中断，从而让程序陷入内核态，执行相应的操作。

但是这篇文章稍微有点老，提到的源码路径有变化，在现在的 Android 12 上涉及的源码如下

```
common/include/uapi/asm-generic/unistd.h    # 包含调用号
bionic/libc/bionic/__set_errno.cpp          # 包含设置errno的代码
bionic/libc/tools/gensyscalls.py            # 用于生成不同架构下的syscall汇编代码
```

[cs.android.com](https://xxr0ss.github.io/post/frida_detection/cs.android.com "cs.android.com") 上可查阅

#### 自行实现系统调用的过程

`CMakeLists.txt`中启用汇编

```
set(can_use_assembler TRUE)
enable_language(ASM)
```

复制和修改 Android 源码`bionic/libc/private/bionic_asm.h`到我们项目中

```
/* https://github.com/android/ndk/issues/1422 */
- #include <features.h>
+ // 去掉这个导入，否则我们的汇编代码会无法编译，原因就是源码中给出的这个issue，是个NDK的bug

// ...

- #if defined(__aarch64__)
- #include <private/bionic_asm_arm64.h>
- #elif defined(__arm__)
- #include <private/bionic_asm_arm.h>
- #elif defined(__i386__)
- #include <private/bionic_asm_x86.h>
- #elif defined(__x86_64__)
- #include <private/bionic_asm_x86_64.h>
- #endif
+ // 前面这些其实目的就是为了下面这个
+ #define __bionic_asm_align 16
```

参照源码中`bionic/libc/tools/gensyscalls.py`脚本的代码：

```
syscall_stub_header = \
"""
ENTRY(%(func)s)
"""

arm64_call = syscall_stub_header + """\
    mov     x8, %(__NR_name)s
    svc     #0

    cmn     x0, #(MAX_ERRNO + 1)
    cneg    x0, x0, hi
    b.hi    __set_errno_internal

    ret
END(%(func)s)
"""

x86_64_call = """\
    movl    $%(__NR_name)s, %%eax
    syscall
    cmpq    $-MAX_ERRNO, %%rax
    jb      1f
    negl    %%eax
    movl    %%eax, %%edi
    call    __set_errno_internal
1:
    ret
END(%(func)s)
"""
```

编写汇编代码

```
#include "bionic_asm.h" // 我们修改过的版本

#if defined(__aarch64__)

ENTRY(my_read)
    mov     x8, __NR_read
    svc     #0
    cmn     x0, #(MAX_ERRNO + 1)
    cneg    x0, x0, hi
    b.hi    __set_errno_internal
    ret
END(my_read)

// ...
#endif
```

汇编代码里的`__set_errno_internal`我们自行进行实现

`antifrida.cpp`:

```
// Our customized __set_errno_internal for syscall.S to use.
// we do not use the one from libc due to issue https://github.com/android/ndk/issues/1422
extern "C" long __set_errno_internal(int n) {
    errno = n;
    return -1;
}
```

别忘了在 C++ 代码中声明外部函数，以调用我们刚写好的汇编代码

copy 标准系统调用函数原型，基本上改个名就可以了

```
// customized syscalls
extern "C" int my_read(int, void *, size_t);
extern "C" int my_openat(int dirfd, const char *const __pass_object_size pathname, int flags, mode_t modes);
extern "C" long my_ptrace(int __request, ...);
```

Android Lollipop 及以后（Build.VERSION.SDK_INT >= 21）就无法再使用

`ActivityManager`的`getRunningAppProcesses()`来获取运行中的进程了。

所以这里我们获取 root 权限，然后调用 ps 来遍历进程列表

获取 root 权限核心代码很简单

```
process = Runtime.getRuntime().exec("su")
```

然后手机上装的 root 管理工具，比如 magisk 就会弹框提示授权。

封装一下

```
fun execRootCmd(cmd: String): String {
    if (!rooted) return ""
    var out = ""
    try {
        val process = Runtime.getRuntime().exec("su")
        val stdin = DataOutputStream(process.outputStream)
        val stdout = process.inputStream
        val stderr = process.errorStream

        Log.i(TAG, "execRootCmd: $cmd")
        stdin.writeBytes(cmd + "\n")
        stdin.flush()
        stdin.writeBytes("exit\n")
        stdin.flush()
        stdin.close()
        var br = BufferedReader(InputStreamReader(stdout))
        var line: String?

        while ((br.readLine().also { line = it }) != null) {
            out += line
        }
        br.close()
        br = BufferedReader(InputStreamReader(stderr))
        while ((br.readLine().also { line = it }) != null) {
            out += line
        }
        br.close()
    }catch (e: Exception) {
        Log.e(TAG, e.stackTraceToString())
    }
    return out
}
```

进行检测：

```
binding.btnCheckProcesses.setOnClickListener {
    if (!SuperUser.rooted) {
        SuperUser.tryRoot(packageCodePath)
        if (!SuperUser.rooted)
            return@setOnClickListener
    }
    val result = SuperUser.execRootCmd("ps -ef")
    Log.i(TAG, "Root cmd result (size ${result.length}): $result ")
    binding.textStatus.text.clear()
    binding.textStatus.text.append(result)

    Toast.makeText(
        this, if (result.contains("frida-server"))
            "frida-server process detected" else "no frida-server process found",
        Toast.LENGTH_SHORT
    ).show()
}
```

前面提到的检查内存模块列表的方式，可以通过改名绕过，所以这里介绍检查内存特征的方式。

比如检测模块特征字符串`"frida:rpc"`

扫描核心代码：

```
while ((read_line(fd, buf, buf_size, use_customized_syscalls)) > 0) {
    if (sscanf(buf, "%lx-%lx %4s %lx %*s %*s %s", &base, &end, perm, &offset, path) != 5) {
        continue;
    }

    if (perm[0] != 'r') continue;
    if (perm[3] != 'p') continue; //do not touch the shared memory
    if (0 != offset) continue;
    if (strlen(path) == 0) continue;
    if ('[' == path[0]) continue;
    if (end - base <= 1000000) continue;
    if (wrap_endsWith(path, ".oat")) continue;
    if (elf_check_header(base) != 1) continue;

    // 扫描指定内存范围是否存在特定字符串
    if (find_mem_string(base, end, (unsigned char *) sig, sig_len) == 1) {
        __android_log_print(ANDROID_LOG_INFO, TAG,
                            "frida signature \"%s\" found in %lx - %lx", sig, base, end);
        result = JNI_TRUE;
        break;
    }
}
```

针对性的进行了扫描，提高了运行效率。

bypass 手段参考 [CrackerCat/strongR-frida-android](https://github.com/CrackerCat/strongR-frida-android "CrackerCat/strongR-frida-android")

Linux 调试是通过系统调用 ptrace 实现的，我们可以通过如下代码检查是否被调试器附加：

```
extern "C"
JNIEXPORT jboolean JNICALL
Java_com_xxr0ss_antifrida_utils_AntiFridaUtil_checkBeingDebugged(JNIEnv *env, jobject thiz, jboolean use_customized_syscall) {

    long res = use_customized_syscall ? my_ptrace(PTRACE_TRACEME, 0) : ptrace(PTRACE_TRACEME, 0);
    return res < 0 ? JNI_TRUE: JNI_FALSE;
}
```

（这里也可以使用我们自己实现的系统调用）