> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.seeflower.dev](https://blog.seeflower.dev/archives/166/)

> 前言提到 dump，（做安卓逆向）大家可能会首先想到是 dump 内存中的 so，然后修复，接着再去逆向或者模拟执行 dump 后模拟执行，按道理说比自己做 so 解析并完整模拟会更有优势，毕竟省去了初始化 so...

提到 dump，（做安卓逆向）大家可能会首先想到是 dump 内存中的 so，然后修复，接着再去逆向或者模拟执行

dump 后模拟执行，按道理说比自己做 so 解析并完整模拟会更有优势，毕竟省去了初始化 so 的工作

但我也只见到过几篇文章有这样做，都是使用 frida 去 dump，然后修复 so 后再模拟执行

我想问题在于这样的 dump 操作没有拿到准确的**上下文**，构造参数仍然是一件很麻烦的事情

不过好在已经有人做过这件事了，只是大家可能没有注意到，亦或是脚本年久失修...

*   [afl-unicorn: Fuzzing The 'Unfuzzable'](https://www.bilibili.com/video/av83051615)

主要参考下面的脚本，这个脚本已经是两年前的了，存在诸多问题

*   [https://github.com/AFLplusplus/AFLplusplus/blob/stable/unicorn_mode/helper_scripts/unicorn_dumper_lldb.py](https://github.com/AFLplusplus/AFLplusplus/blob/stable/unicorn_mode/helper_scripts/unicorn_dumper_lldb.py)
*   Q: 为什么选择 lldb 而不是 gdb
*   A: 经过实践 lldb 更快，更方便，更现代化

起初我是想进行简单修补上面的脚本，后面发现还是得重写下

下面就详细讲解如何编写可以 **dump 上下文**的 lldb 脚本

编写过程中用到的 API 可以在下面的文档中查询

*   [https://lldb.llvm.org/use/python-reference.html](https://lldb.llvm.org/use/python-reference.html)

lldb 的 python 库源码在线版本，可以点击下面网页的`source code`查看

*   [https://lldb.llvm.org/python_reference/](https://lldb.llvm.org/python_reference/)

如果你已经在 termux 安装好 lldb 了，那么你也可以在下面的路径找到 lldb 库源代码

*   /data/data/com.termux/files/home/lib/python3.10/site-packages/lldb/__init__.py

建议把这个脚本拿出来，改名为`lldb.py`

然后新建一个`lldb_dumper.py`文件，把上面的`lldb.py`放在同一级目录

在`lldb_dumper.py`中写上如下代码，这样使用 vscode 或者其他 IDE 即可获得代码提示

```
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from lldb import *

import lldb
```

我希望命中断点后，进程都停下来，这个时候只需要执行一个命令就可以 dump 上下文

根据`Create a new lldb command using a Python function`小节

*   [https://lldb.llvm.org/use/python-reference.html#create-a-new-lldb-command-using-a-python-function](https://lldb.llvm.org/use/python-reference.html#create-a-new-lldb-command-using-a-python-function)

可以知道应该在代码中添加一个`__lldb_init_module`函数，然后通过`debugger.HandleCommand`去添加自定义命令

```
def __lldb_init_module(debugger: 'SBDebugger', internal_dict: dict):
    debugger.HandleCommand('command script add -f lldb_dumper.helloworld hello')
```

`command script add -f lldb_dumper.helloworld hello`这个命令的意思是，当执行 hello 命令的时候，lldb 会去调用`lldb_dumper`脚本中的`helloworld`函数

即`lldb_dumper.py`被视为一个模块

根据文档，可以知道被调用的函数接收下面这些参数，详细解释请参考文档

*   debugger
*   command
*   exe_ctx
*   result
*   internal_dict

对于 dump 上下文来说，最重要的就是`exe_ctx`了，其类型为`SBExecutionContext`，看到这个就知道它肯定和上下文紧密相关

写个`helloworld`如下

```
def helloworld(debugger: 'SBDebugger', command: str, exe_ctx: 'SBExecutionContext', result: 'SBCommandReturnObject', internal_dict: dict):
    print('helloWorld debugger:', debugger)
    print('helloWorld command:', command)
    print('helloWorld exe_ctx:', exe_ctx)
```

进入 lldb，通过下面的命令加载脚本

```
command script import lldb_dumper.py
```

这个时候执行`hello you`命令可以看到有输出信息，其中`command`是`you`，这个可以方便增强脚本特性

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_21-57-28.png)

现在你已经会 helloword 了，现在开始正式编写 dump 上下文脚本吧~

获取架构信息
------

通过下面的代码，得到`triple`是`aarch64-unknown-linux-android`这样的字符串

```
target = exe_ctx.GetTarget() # type: SBTarget
triple = target.GetTriple()
```

triple 的一般格式为`<arch><sub>-<vendor>-<sys>-<abi>`

*   arch => x86_64, i386, arm, thumb, mips
*   sub => v5, v6m, v7a, v7m
*   vendor => pc, apple, nvidia, ibm
*   sys => none, linux, win32, darwin, cuda
*   abi => eabi, gnu, android, macho, elf

可以根据第一个`-`之前的部分来判断是何种架构，记得区分大小端（不过一般都是小端）

代码片段如下：

```
def dump_arch_info(target: 'SBTarget'):
    triple = target.GetTriple()
    logger.debug(f'[dump_arch_info] triple => {triple}')
    # 'aarch64', 'unknown', 'linux', 'android'
    arch, vendor, sys, abi = triple.split('-')
    if arch == 'aarch64' or arch == 'arm64':
        return 'arm64le'
    elif arch == 'aarch64_be':
        return 'arm64be'
    elif arch == 'armeb':
        return 'armbe'
    elif arch == 'arm':
        return 'armle'
    else:
        return ''

target = exe_ctx.GetTarget() # type: SBTarget
arch_long = dump_arch_info(target)
```

获取寄存器信息
-------

查阅文档，要获取寄存器，需要按以下几步完成

*   通过`SBExecutionContext`拿到`SBFrame`
*   调用`SBFrame`的`GetRegisters`方法拿到全部寄存器
    
    *   是一个`SBValueList`对象，通过`GetName`可以知道是通用寄存器还是浮点寄存器
*   遍历全部寄存器，通过`GetName`获取名字，通过`GetValue`取值

**这里有一个重点：**

直接获取`SBValue`拿到的值通常得到的是贴近人类可读的字符串，比如浮点寄存器打印出来是科学计数法形式的，但是对于浮点寄存器来说我们应该拿到最为精确的值

通过查阅文档，可以将`SBValue`格式指定为`lldb.eFormatUnsigned`读取，这样就是最完整的数值了

但是`GetValue`拿到的类型依然是字符串 在使用时还需要一次转换

因此这里干脆指定为`lldb.eFormatHex`，即 16 进制的字符串，这样更加符合逆向人员的阅读习惯

*   [https://lldb.llvm.org/python_api_enums.html#format](https://lldb.llvm.org/python_api_enums.html#format)

代码片段如下：

```
def dump_regs(frame: 'SBFrame'):
    regs = {} # type: Dict[str, int]
    registers = None # type: List[SBValue]
    for registers in frame.GetRegisters():
        # - General Purpose Registers
        # - Floating Point Registers
        logger.debug(f'registers name => {registers.GetName()}')
        for register in registers:
            register_name = register.GetName()
            register.SetFormat(lldb.eFormatHex)
            register_value = register.GetValue()
            regs[register_name] = register_value
    logger.info(f'regs => {json.dumps(regs, ensure_ascii=False, indent=4)}')
    return regs

frame = exe_ctx.GetFrame() # type: SBFrame
regs = dump_regs(frame)
```

这部分拿到的信息一般用不到，似乎是排查 bug 用的，但是原脚本有这部分，所以一并重写了

简单来说就是遍历`SBTarget`的`module`，再遍历每个`module`的`section`

收集记录全部`section`信息

```
def dump_memory_info(target: 'SBTarget'):
    logger.debug('start dump_memory_info')
    sections = []
    # 先查找全部分段信息
    for module in target.module_iter():
        module: SBModule
        for section in module.section_iter():
            section: SBSection
            module_name = module.file.GetFilename()
            start, end, size, name = get_section_info(target, section)
            section_info = {
                'module': module_name,
                'start': start,
                'end': end,
                'size': size,
                'name': name,
            }
            # size 好像有负数的情况 不知道是什么情况
            logger.info(f'Appending: {name}')
            sections.append(section_info)
    return sections

target = exe_ctx.GetTarget() # type: SBTarget
sections = dump_memory_info(target)
```

![](https://blog.seeflower.dev/images/Snipaste_2022-08-06_22-22-49.png)

代码如下，参考原脚本的逻辑，就是先不断遍历当前进程的内存区域

通过`process.GetMemoryRegionInfo`获取信息，记录名字，权限，起始和结束地址

然后再通过`process.ReadMemory`去读取

读取之前加入了一些黑名单过滤规则，这样可以加快 dump 速度

另外保存内存数据的时候会进行压缩，因为有很多区域是连续的 0，压缩可以极大减少体积

并且保存时以 md5 作为文件名，存在多个内存分段都是 4096 字节内容全 0 的情况，这样也能省下不少空间

所以有个要点是，在使用的时候记得解压后再加载到 unicorn 的内存中去

代码片段如下：

```
def dump_memory(process: 'SBProcess', dump_path: Path, black_list: Dict[str, List[str]], max_seg_size: int):
    logger.debug('start dump memory')
    memory_list = []
    mem_info = lldb.SBMemoryRegionInfo()
    start_addr = -1
    next_region_addr = 0
    while next_region_addr > start_addr:
        # 从内存起始位置开始获取内存信息
        err = process.GetMemoryRegionInfo(next_region_addr, mem_info) # type: SBError
        if not err.success:
            logger.warning(f'GetMemoryRegionInfo failed, {err}, break')
            break
        # 获取当前位置的结尾地址
        next_region_addr = mem_info.GetRegionEnd()
        # 如果超出上限 结束遍历
        if next_region_addr >= sys.maxsize:
            logger.info(f'next_region_addr:0x{next_region_addr:x} >= sys.maxsize, break')
            break
        # 获取当前这块内存的起始地址和结尾地址
        start = mem_info.GetRegionBase()
        end = mem_info.GetRegionEnd()
        # 很多内存块没有名字 预设一个
        region_name = 'UNKNOWN'
        # 记录分配了的内存
        if mem_info.IsMapped():
            name = mem_info.GetName()
            if name is None:
                name = ''
            mem_info_obj = {
                'start': start,
                'end': end,
                'name': name,
                'permissions': {
                    'r': mem_info.IsReadable(),
                    'w': mem_info.IsWritable(),
                    'x': mem_info.IsExecutable(),
                },
                'content_file': '',
            }
            memory_list.append(mem_info_obj)
    # 开始正式dump
    for seg_info in memory_list:
        try:
            start_addr = seg_info['start'] # type: int
            end_addr = seg_info['end'] # type: int
            region_name = seg_info['name'] # type: str
            permissions = seg_info['permissions'] # type: Dict[str, bool]

            # 跳过不可读 之后考虑下是不是能修改权限再读
            if seg_info['permissions']['r'] is False:
                logger.warning(f'Skip dump {region_name} permissions => {permissions}')
                continue

            # 超过预设大小的 跳过dump
            predicted_size = end_addr - start_addr
            if predicted_size > max_seg_size:
                logger.warning(f'Skip dump {region_name} size:0x{predicted_size:x}')
                continue

            skip_dump = False

            for rule in black_list['startswith']:
                if region_name.startswith(rule):
                    skip_dump = True
                    logger.warning(f'Skip dump {region_name} hit startswith rule:{rule}')
            if skip_dump: continue

            for rule in black_list['endswith']:
                if region_name.endswith(rule):
                    skip_dump = True
                    logger.warning(f'Skip dump {region_name} hit endswith rule:{rule}')
            if skip_dump: continue

            for rule in black_list['includes']:
                if rule in region_name:
                    skip_dump = True
                    logger.warning(f'Skip dump {region_name} hit includes rule:{rule}')
            if skip_dump: continue

            # 开始读取内存
            ts = datetime.now()
            err = lldb.SBError()
            seg_content = process.ReadMemory(start_addr, predicted_size, err)
            tm = (datetime.now() - ts).total_seconds()
            # 读取成功的才写入本地文件 并计算md5
            # 内存里面可能很多地方是0 所以压缩写入文件 减少占用
            if seg_content is None:
                logger.debug(f'Segment empty: @0x{start_addr:016x} {region_name} => {err}')
            else:
                logger.info(f'Dumping @0x{start_addr:016x} {tm:.2f}s size:0x{len(seg_content):x}: {region_name} {permissions}')
                compressed_seg_content = zlib.compress(seg_content)
                md5_sum = hashlib.md5(compressed_seg_content).hexdigest() + '.bin'
                seg_info['content_file'] = md5_sum
                (dump_path / md5_sum).write_bytes(compressed_seg_content)
        except Exception as e:
            # 这里好像不会出现异常 因为前面有 SBError 处理了 不过还是保留
            logger.error(f'Exception reading segment {region_name}', exc_info=e)

    return memory_list

# 设置过滤黑名单 符合下面条件的跳过dump
black_list = {
    'startswith': ['/dev', '/system/fonts', '/dmabuf'],
    'endswith': ['(deleted)', '.apk', '.odex', '.vdex', '.dex', '.jar', '.art', '.oat', '.art]'],
    'includes': [],
}
# 设置单个内存分段dump大小上限
max_seg_size = 64 * 1024 * 1024

# dump内存
process = exe_ctx.GetProcess() # type: SBProcess
segments = dump_memory(process, dump_path, black_list, max_seg_size)
```

至此完成了上下文 dump 的脚本编写

**注意** 既然是上下文 dump，应当在命中断点的时候执行命令

脚本可以提前加载，因为加载的时候只是起到一个注册命令的作用

  
星球广告 & 交流群  
![](https://blog.seeflower.dev/images/xqyh.png)