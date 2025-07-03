> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/forum.php?mod=viewthread&tid=2019762) ![](https://avatar.52pojie.cn/images/noavatar_middle.gif)Kyctz  _ 本帖最后由 Kyctz 于 2025-4-2 20:58 编辑_  

#### 前言

本文主要是对 Angr 官方文档内容的简要解读，同时附带了一些我在使用 angr 过程中习得的使用技巧  
在阅读本文前推荐学习一遍 angr_ctf, 以了解 angr 的基本用法  
[https://github.com/jakespringer/angr_ctf](https://github.com/jakespringer/angr_ctf)

angr 是一个支持多处理架构的用于二进制文件分析的工具包，它提供了动态符号执行的能力以及多种静态分析的能力，

### 安装

angr 适用于 Python 3.10+ 版本

具体安装方法可以参考官方文档  
[https://angr-docs-zh-cn.readthedocs.io/zh-cn/latest/getting-started/installing.html](https://angr-docs-zh-cn.readthedocs.io/zh-cn/latest/getting-started/installing.html)

顶层接口 Project
------------

Project 类是 angr 的主类，表示程序的初始镜像，通过传入二进制文件路径初始化该类的对象，可以加载想要分析的二进制文件:

```
import angr
proj = angr.Project('/bin/true')
```

Project 对象的常用属性

```
>>> proj.arch     # 二进制文件的架构
<Arch AMD64 (LE)>
>>> hex(proj.entry)    # 二进制文件的入口点
'0x401060'
>>> proj.filename # 二进制文件名称
'./test'

# 此外还有：
arch.bits  # CPU字长
arch.name：# 架构名，例如 X86
arch.memory_endness：# 端序，大端为 Endness.BE ，小端为 Endness.LE。
...
```

### 加载选项

#### main_opts 和 lib_opts

angr 在载入时，可以附加 main_opts 和 lib_opts 参数指定特定二进制文件的加载选项  
main_opts 指示加载的主要可执行文件设置，而 lib_opts 指示外部库的载入设置

main_opts 通过传入从选项名称到选项值的映射字典进行设置  
而 lib_opts 是从库名称到字典的映射，字典的内容则是从选项名称到选项值的映射

使用示例：

> ```
> base_addr = 0x100000
> p = angr.Project('examples/fauxware/fauxware', main_opts={'backend': 'blob', 'arch': 'i386'}, lib_opts={'libc.so.6': {'backend': 'elf'}},auto_load_libs=False)
> ```

常用的加载选项如下

*   base_addr - 指定基址
*   entry_point - 指定入口点
*   arch - 指定架构
*   backend 指定使用的后端

<table><thead><tr><th>后端名</th><th>描述</th></tr></thead><tbody><tr><td>elf</td><td>基于 PyELFTools 的 ELF 文件加载器</td></tr><tr><td>pe</td><td>基于 PEFile 的 PE 文件加载器</td></tr><tr><td>mach-o</td><td>Mach-O 文件的静态加载器。不支持动态链接或重设基址</td></tr><tr><td>cgc</td><td>Cyber Grand Challenge 二进制文件的静态加载器</td></tr><tr><td>backedcgc</td><td>CGC 二进制文件的静态加载器，允许指定内存和寄存器支持者</td></tr><tr><td>elfcore</td><td>ELF 代码转储加载器</td></tr><tr><td>bolb</td><td>将文件作为平坦镜像加载到内存中</td></tr></tbody></table>

##### 符号摘要

angr 将一大堆函数实现为 SimProcedures 中的符号摘要 (本质上是模仿库函数对状态的影响的 Python 函数)。SIM_PROCEDURES 字典是两级的，首先键入包名（libc、posix、win32、stubs），然后键入库函数的名称。

默认情况下，对于一些存在预定义的 SimProcedures 模型的外部函数，用预定义的符号化模型（SimProcedures）替代实际的外部函数调用，通过抽象行为减少约束复杂度，使分析变得更加易于处理，但同时也可能会**出现一些不准确的情况**。

可以通过 angr.procedures 或 angr.SimProcedures 查看列表  
手动使用 SimProcedures 替代实际库函数示例

```
stub_func = angr.SIM_PROCEDURES['stubs']['ReturnUnconstrained']
proj.hook(0x10000, stub_func()) # 将该函数调用改为直接返回一个不受约束的符号值
```

外部库的载入相关设置如下

<table><thead><tr><th>名称</th><th>描述</th><th>传入参数</th></tr></thead><tbody><tr><td>auto_load_libs　</td><td>是否自动加载程序的依赖 (默认为是)，如果为否，则会在每次调用未载入的依赖时返回一个不受约束的符号值</td><td>布尔值</td></tr><tr><td>skip_libs</td><td>希望避免加载的库</td><td>库名</td></tr><tr><td>except_missing_libs</td><td>无法解析库时是否抛出异常</td><td>布尔值</td></tr><tr><td>force_load_libs</td><td>强制加载的库</td><td>库名</td></tr><tr><td>ld_path</td><td>共享库的优先搜索路径</td><td>路径名</td></tr><tr><td>use_sim_procedures</td><td>如果为否，则只有 extern 对象提供的符号将被 SimProcedures 替换，并返回一个不受约束的符号值</td><td>布尔值</td></tr><tr><td>exclude_sim_procedures_list 和 &lt;br&gt;exclude_sim_procedures_func</td><td>在 use_sim_procedures=True 时，选择性排除某些函数不被 SimProcedures 替代，</td><td>List[str] 和 回调函数 Callable[[str], bool]</td></tr></tbody></table>

> 在进行一些程序分析时，如果 auto_load_libs 为 True, angr 会同时分析动态链接库，导致耗时非常久，一般将其设置为 False，使用 hook 来用 angr 自身的实现代替库函数
> 
> 对于非常复杂的外部调用函数，直接符号化会导致路径爆炸或无法收敛。  
> 或对于未定义的外部函数，无法直接进行模拟运行求解  
> 启用 use_sim_procedures 选项时，angr 会用预定义的 SimProcedures 模型替代这些外部函数调用，仅模拟函数的关键行为，返回符号值或标记为未解析，以降低模拟的复杂度

### 加载器 loader

angr 采用 CLE 模块来实现从二进制文件到其在虚拟地址空间中的实现。CLE 的结果称为加载器，可以在 .loader 属性中访问。

```
>>>proj.loader # 可用虚拟地址空间范围
<Loaded true, maps [0x400000:0x5004000]>

>>>proj.loader.shared_objects # 已载入的共享库
{'ld-linux-x86-64.so.2': <ELF Object ld-2.24.so, maps [0x2000000:0x2227167]>,
 'libc.so.6': <ELF Object libc-2.24.so, maps [0x1000000:0x13c699f]>}

>>>proj.loader.min_addr
0x400000
>>> proj.loader.max_addr
0x5004000

>>> proj.loader.main_object  # 将已载入的二进制文件
<ELF Object true, maps [0x400000:0x60721f]>

>>> proj.loader.main_object.execstack  # 示例查询：此二进制文件是否具有可执行堆栈？
False
>>> proj.loader.main_object.pic  # 示例查询：此二进制文件是否是位置无关的？
True

# 此外：
>>> proj.loader.all_objects # 已加载的所有对象
>>> proj.loader.find_object_containing(0x400000) # 目标地址所属的对象
>>> proj.loader.kernel_object # 模拟系统调用对象
>>> proj.loader.extern_object # 未解析的外部符号对象
```

#### 已加载对象

可以从加载器中提取出特定的对象，从中可以得到更多数据

```
>>> obj = proj.loader.main_object

# 对象的入口点
>>> obj.entry
0x400580

>>> obj.min_addr, obj.max_addr
(0x400000, 0x60105f)
```

获取 ELF 的段信息

```
>>> obj.segments
<Regions: [<ELFSegment flags=0x5, relro=0x0, vaddr=0x400000, memsize=0x5d04, filesize=0x5d04, offset=0x0>, <ELFSegment flags=0x6, relro=0x0, vaddr=0x605d08, memsize=0x258, filesize=0x250, offset=0x5d08>]>
```

获取 ELF 的内存分段和文件分段

```
>>> obj.sections
<Regions: [<Unnamed | offset 0x0, vaddr 0x400000, size 0x0>, <.interp | offset 0x238, vaddr 0x400238, size 0x1c>, <.note.ABI-tag | offset 0x254, vaddr 0x400254, size 0x20>, <.note.gnu.build-id | offset 0x274, vaddr 0x400274, size 0x24>, <.gnu.hash | offset 0x298, vaddr 0x400298, size 0x64>,...
```

获取 PLT 表信息

```
>>> obj.plt
{'__uflow': 0x401400,
 'getenv': 0x401410,
 'free': 0x401420,
 'abort': 0x401430,
 '__errno_location': 0x401440,
 'strncmp': 0x401450,
 '_exit': 0x401460,
```

其它属性

```
>>> obj.find_segment_containing(obj.entry) # 查找地址所在的段

>>> obj.plt # 对象中的符号(由符号名和地址组成的键值对)

>>> obj.reverse_plt[addr] # 相当于obj.plt的键值反转

>>> obj.linked_base # 对象链接时的逻辑地址

>>> obj.mapped_base # 对象映射到的虚拟地址
0x400000
```

#### 符号

符号对象用于将名称映射到某个地址  
可以使用 loader 的 find_symbol 方法通过符号名称查找对应符号对象

```
>>>strcmp = proj.loader.find_symbol('strcmp')
```

如果要获得特定对象的 symbol，则需要使用 get_symbol 方法：

```
malloc = proj.loader.main_object.get_symbol('malloc')
```

得到一个 symbol 对象后，可以获取获取符号名 / 所属者 / 链接地址 / 相对地址等信息。

```
>>> strcmp.name # 符号名
'strcmp'

>>> strcmp.owner # 符号的所属对象
<ELF Object libc-2.23.so, maps [0x1000000:0x13c999f]>

# symbol 对象有三种获取其地址的方式：
>>> strcmp.rebased_addr # 符号在全局地址空间的地址。
0x1089cd0
>>> strcmp.linked_addr # 符号相对于二进制的预链接基址的地址
0x89cd0
>>> strcmp.relative_addr # 符号相对于对象基址的地址
0x89cd0
```

更多属性如下：

```
strcmp.is_common           strcmp.is_local          
strcmp.owner_obj           strcmp.resolvedby
strcmp.is_export           strcmp.is_static      
strcmp.rebased_addr        strcmp.size
strcmp.is_extern           strcmp.is_weak             strcmp.relative_addr       strcmp.subtype
strcmp.is_forward          strcmp.linked_addr       
strcmp.is_function         strcmp.name                
strcmp.is_import           strcmp.owner       
strcmp.type                strcmp.resolved
```

实用类工厂 factory
-------------

#### 基本块 block

angr 以基本块为单位分析代码  
可以通过工厂的 block 方法通过给定地址，获取对应的基本块，为 Block 对象

```
>>> block = proj.factory.block(proj.entry)
<Block for 0x4017b0, 42 bytes>
```

打印基本块中的汇编语句

```
>>>block.pp()

_start:
400550  xor     ebp, ebp
400552  mov     r9, rdx
400555  pop     rsi
400556  mov     rdx, rsp
400559  and     rsp, 0xfffffffffffffff0
40055d  push    rax
40055e  push    rsp
40055f  mov     r8, __libc_csu_fini
400566  mov     rcx, __libc_csu_init
40056d  mov     rdi, main
400574  call    __libc_start_main
```

查询指令数量  
`block.instructions`

查询指令地址  
`block.instruction_addrs`

使用 capstone 反汇编  
`block.capstone`

使用 VEX 中间表示  
`block.vex`  
IRSB 是 VEX 的一个基本块

PyVEX 是 angr 的核心组件之一，用于将机器码转换为中间表示语言 VEX IR  
PyVEX 的相关介绍  
[https://zhuanlan.zhihu.com/p/347706328](https://zhuanlan.zhihu.com/p/347706328)  
[https://api.angr.io/projects/pyvex/en/latest/quickstart.html](https://api.angr.io/projects/pyvex/en/latest/quickstart.html)

### 状态 state

使用 Project 加载进目标二进制文件后，要想执行它，需要使用它创建一个状态，即 SimState 对象。

SimState 对象代表程序的一个实例镜像，模拟执行到某个时刻的状态。

#### 创建状态

要创建状态，需要使用 Project 对象中的 factory，如下：

```
init_state = p.factory.entry_state()
```

有下列方式进入状态

<table><thead><tr><th>预设状态方式</th><th>描述</th></tr></thead><tbody><tr><td>entry_state()</td><td>初始化状态为程序运行到程序入口点处的状态</td></tr><tr><td>blank_state(addr=)</td><td>大多数数据都没有初始化，状态中下一条指令为 addr 处的指令的空白状态</td></tr><tr><td>full_init_state()</td><td>共享库和预定义内容已经加载完毕，例如刚加载完共享库</td></tr><tr><td>call_state(addr, arg1, arg2, ...)</td><td>准备调用函数的状态, addr 为函数的地址，其余的为参数，可以是 Python 整数、字符串、数字、位向量</td></tr></tbody></table>

状态包含了程序运行时的一切信息，寄存器、内存的值、文件系统以及符号变量等

##### 命令行参数

通过使用 args 选项设置模拟执行时的命令行参数

```
st = p.factory.full_init_state(
        args=['./engine',flag],
        add_options=angr.options.unicorn,
       )
```

其中 flag 既可以是字符串，也可以是一个 BVV 或 BVS

> 当运行需要命令行参数的程序时，应该使用`entry_state()`或`full_init_state()`预设状态，以确保命令行参数被正确载入

##### 模拟运行时输入

通过使用 stdin 选项设置模拟执行时的输入

```
    initial_state = project.factory.entry_state(
            add_options=angr.options.unicorn,
            stdin=flag
    )
```

flag 既可以是字符串，也可以是一个 BVV 或 BVS  
这种方法支持`getc()`函数和`cin`函数的输入

##### 附加选项

在创建 state 时，有多个选项用以配置 state 的属性  
`state = proj.factory.entry_state(add_options={angr.options.LAZY_SOLVES})`  
此外，对于已经创建好的 state，也可以使用`state.options.add(angr.options.LAZY_SOLVES)`  
添加新的配置  
或  
`start_state.options.remove(angr.options.LAZY_SOLVES)`  
删去原有的配置

###### unicorn

使用 unicorn 模拟执行，这有助于提高运行速度

```
state = p.factory.full_init_state(
        add_options=angr.options.unicorn,
       )
```

angr.options.unicorn 的类型为 set，是多个有关选项设置的集合，在与其它选项搭配使用时需要注意 (例如使用`angr.options.unicorn| {angr.options.LAZY_SOLVES}`)

###### LAZY_SOLVES

正常情况下某个 state 被发现是不可满足条件的，那么 state 会进行回溯，来确定到底是哪个 state 不被满足，之后所有的 state 会被放入到 stash 中。但是回溯操作将会花费大量的时间，尤其是运算量比较大的时候，

此时可以添加 LAZY_SOLVES 选项，该选项启用时，state 会先将所有可能的路径运行完，而不会对这些路径的可达性进行检查

```
# 添加LAZY_SOLVES
state = p.factory.blank_state(addr=START_ADDR, add_options={angr.options.LAZY_SOLVES})
# 去除LAZY_SOLVES
state = p.factory.blank_state(addr=START_ADDR, remove_options={angr.options.LAZY_SOLVES})
```

适用于**库函数调用比较少，执行路径较为简单，但是运算量比较庞大**的程序

> 由于不会对这些路径的可达性进行检查，也就不会在执行时将不可达的路径提前去除，更容易导致**路径爆炸问题**  
> 此外，在到达目标位置时，还需要使用`state.satisfiable()` **判断该状态是否存在至少一个解满足约束**，以确保该路径的可解性

###### 其它

更多附加选项可以查阅官方文档  
[https://angr-docs-zh-cn.readthedocs.io/zh-cn/latest/appendix/options.html](https://angr-docs-zh-cn.readthedocs.io/zh-cn/latest/appendix/options.html)

#### 接口

一个 state 类似于程序执行过程中的 CPU 和内存  
具有如下对象：

```
state.regs #寄存器状态组，其中每个寄存器都为一个位向量（BitVector），可以通过寄存器名称来访问对应的寄存器（例如 state.regs.esp ）

state.memory # 内存访问接口

state.posix # POSIX 相关的环境接口，例如 state.posix.dumps(fileno) 获取对应文件描述符上的流。

state.solver #基于状态的求解接口
#在需要对位向量符号进行具体值的求解时，可以先将位向量符号存放到状态的内存 / 寄存器中，之后用 simgr 探索到对应的状态后，再使用 state.solver.eval() 成员函数来获取对应位向量在当前状态下的值
```

#### 常用操作

以下操作涉及到的值的类型通常为 BVV 或 BVS

##### 寄存器操作

通过`state.regs.<寄存器名>`来访问和修改寄存器值

##### 栈操作

栈访问涉及两个寄存器：ebp 和 esp  
push 和 pop 指令可以通过以下方法调用

```
state.stack_push(value)
state.stack_pop()
```

可将想要的数据放入栈中  
若想改变栈中的变量，可与修改寄存器值的命令配合使用

##### 内存访问

```
#读内存
state.memory.load(addr, size, endness)
#写内存
state.memory.store(addr,data, endness)
```

参数 endness 用于设置端序  
其中 data 可以是字符串、BVV 或 BVS

```
LE – 小端序(little endian, least significant byte is stored at lowest address)
BE – 大端序(big endian, most significant byte is stored at lowest address)

ME – 中间序(Middle-endian. Yep.)(每个字内部采用Little Endian字节顺序，而两个字之间采用Big Endian字节顺序组织，即MiddleEndian字节顺序。)
```

`endness=archinfo.Endness.LE`

##### 文件操作 Simfile

angr 提供了 SimFile 类来模拟文件

```
filename = 'test.txt'
# 创建一个SimFile对象
simfile = angr.storage.SimFile(name=filename, content=data, size=0x40)
# 将SimFile对象插入state对象中
state.fs.insert(filename, simfile)
```

##### 块执行操作

state 状态本身可以进行简单的按块执行操作  
可以免去创建 Simgr 的步骤

```
>>>s2.step()[0].step()[0]
<SimState @ 0x700020>

>>>s2.step()[0].step()[0].step()[0]
<SimState @ 0x4009f3>

>>>s2.step()[0].step()[0].step()[0].step()[0]
<SimState @ 0x4009ff>
```

#### state 插件

##### 全局 global

state.global 实现了标准 python dict 接口，使用它能够在模拟执行的 state 中存储任意数据，并且很方便的就能访问到。可以使用如下的语句访问全局变量：

`state.globals['x'] = a_symbol`  
全局变量名 x 不需要提前声明，但当需要使用这个全局变量之前，需要对它赋值。

在`symbol hook`的`angr.SimProcedure`子类对象函数中  
要在函数作用域外访问函数作用域内定义的符号变量，通过`self.state.globals['<key>'] = <BVS name>`来实现

```
def run(self, format_string, scanf0_address, scanf1_address):
        # 在run中完成scanf的功能，即向目标地址写入符号
        # 定义两个符号变量，用于写入buffer
        scanf0 = claripy.BVS('scanf0', 32)
        scanf1 = claripy.BVS('scanf1', 32)

        # 写入内存，最后一个参数指定字节序
        self.state.memory.store(scanf0_address,
                                scanf0,
                    endness=project.arch.memory_endness)
        self.state.memory.store(scanf1_address,
                                scanf1,
                    endness=project.arch.memory_endness)

        # 保存变量到全局，方便后续约束求解
        self.state.globals['solution0'] = scanf0
        self.state.globals['solution1'] = scanf1
```

##### 执行路径信息 history

history 记录了一个状态的执行路径，它是一个链表，每个节点代表一次执行的状态。可以不断使用. parent 来回溯这个链表，history 的使用方式如下：

`state.history.bbl_addrs`：返回当前 state 执行的基本块的地址列表  
`state.history.parent.bbl_addrs`：返回上一个 state 执行的基本块的地址列表  
`state.history.descriptions`：描述 state 上每轮执行的状态的字符串列表  
`state.histroy.guards`：到达当前 state 所走路径需要满足的条件的列表（也就是每次分支判断时为 True 还是 False）  
`state.history.events`：执行过程中发生的事件列表，比如符号跳转条件、弹出消息框等

<table><thead><tr><th>名称</th><th>描述</th></tr></thead><tbody><tr><td>history.descriptions</td><td>对状态执行的每轮执行的字符串描述的列表</td></tr><tr><td>history.bbl_addrs</td><td>状态执行的基本块地址的列表。</td></tr><tr><td>history.jumpkinds</td><td>以 VEX 枚举字符串形式列出状态历史中每个控制流转换的配置。</td></tr><tr><td>history.events</td><td>执行期间发生的 “事件” 的语义列表，例如存在符号跳转条件、程序弹出消息框或执行以退出代码终止。&lt;br&gt;</td></tr><tr><td>history.actions</td><td>通常为空，但如果将 angr.options.refs 选项添加到状态，它将填充程序执行的所有内存、寄存器和临时值访问的日志。</td></tr></tbody></table>

按顺序输出经过的基本块的地址

```
for addr in state.history.bbl_addrs: 
    print (hex(addr))
#或
state.history.bbl_addrs.hardcopy
```

##### 栈信息 callstack

callstack 调用栈插件记录执行时栈帧的信息，也是链表格式。可以直接对 state.callstack 进行迭代获得每次执行的栈帧信息。直接访问 state.callstack 可以获得当前状态的调用栈。

*   callstack.func_addr ： 当前执行的函数地址
*   callstack.call_site_addr：调用当前函数的指令的地址
*   callstack.stack_ptr : 当前函数开头的堆栈指针的值
*   callstack.ret_addr : 如果当前函数返回，则返回到的位置

##### Other

`state.copy()`  获取当前状态的副本，之后在该副本上添加约束以避免影响到原状态  
`state.satisfiable()` 判断是否存在至少一个解满足约束（是否有解）  
`state.ip.args[0]` 获取当前状态的指令指针

### 模拟管理器 Simulation Manager

SimulationManager 是 angr 中最重要的控制接口，它允许你同时控制多个状态的符号执行，应用搜索策略来探索程序的状态空间。

传入一个 state 或者 state 的列表创建一个 SM：

```
simgr  = p.factory.simgr(state)
```

SM 中可以保存多个状态。状态被组织成不同种类的 “存储区”Stash，可以根据需要前进、过滤、合并和移动这些存储区。  
大多数操作的默认存储区是 active 存储区

#### 执行

state 有两种主要的执行方式:

```
simgr.step() #以基本块为单位的单步执行。返回simsucessors对象
simgr.explore() #进行路径探索找到满足相应条件的状态。
```

##### 块执行

###### step 执行

`simger.step()`  
让所有处于 active 的 state 执行一个基本块，直到执行块结尾，遇到函数时**步进**并停止在函数的入口点  
可以通过 IDA 的 CFG 图查看基本块结构  
基本块按照**入口**和**条件跳转与无条件跳转的目标语句**进行划分，因此基本块通常会以分支语句结束  
执行的结果一般会返回两个 active stash 中的 state，是该基本块下的两个**可达**分支  
如果其中一个分支不可达，active stash 中的 state 只会有一个

###### run 执行

此外，也可以使用`run`方法执行，`run`方法的底层是`step`方法的循环

通过传入参数设置`run`方法的终止条件

*   使用`simgr.run(n = 10)`可以指定运行的`step`步数
*   使用`simgr.run(until = Callable[[SimulationManager], bool])`指定运行的终止条件
*   `sm.run(completion_mode=any/all)`，指定任一状态完成时终止或所有状态完成时终止
*   如果不传入任何参数，当 active stach 为空时终止

###### 源码分析

相关代码截取自 angr.sim_manager.SimulationManager

```
def step()
    ...
    for state in self._fetch_states(stash=stash):
        ...
        successors = self.step_state(state, successor_func=successor_func, error_list=error_list, **run_args)
    ...
```

`step()`方法的底层是通过`self.step_state`方法对目标`stash`中的每个`state`进行一次模拟执行，从返回的`successors`中提取出新得到的`state`，再分类放回模拟管理器自己的`stash`中

##### 探索

SM 最常用的技术：探索技术

探索技术用于找出程序满足条件的执行状态，条件可以使用执行到的地址 (直接使用整数作为参数)、输入输出(使用以 state 作为参数的函数) 等

`simgr.explore(find = <>,avoid = <>)`

`find`：传入目标指令的地址或地址列表，或者一个用于判断的函数，函数以 state 为形参，返回布尔值  
`avoid`：传入要避免的指令的地址或地址列表，或者一个用于判断的函数，用于减少路径  
(探索时默认使用 DFS（深度优先搜索）策略)

```
# 可以使用地址或地址数组
sm.explore(find=0x080485E5, avoid=[0x080485A8,0x080485F2])

# 可以使用以state为参数的函数
def is_good(state):
    return b'Good Job.' in state.posix.dumps(1)

def is_bad(state):
    return b'Try again.' in state.posix.dumps(1)
sm.explore(find=is_good, avoid=is_bad)

# 可以传入参数n限制explore执行的步数
sm.explore(n=100, find=is_good, avoid=is_bad)
```

explorer 找到的符合 find 的状态会被保存在`simgr.found`这个列表当中，可以遍历其中元素获取状态。

执行 explore 后的结果是一个 SimState 对象类

```
found_state = sm.found[0]
```

此外，还可以通过指定 num_find 参数来指定需要寻找的符合条件的状态的数量

> `explore`方法是通过`exploration_techniques.Explorer(ExplorationTechnique)`类实现的：
> 
> 对于 find 传入的静态地址，加入到 static_find 并作为断点加入 simgr，然后使用 step 执行到断点，对于 avoid 传入的静态地址同理
> 
> 对于 find 传入的函数，则是在每次产生新状态时通过 filter 方法调用该函数，判断是否满足搜索条件

###### 探索机制

探索机制 可以让你自定义模拟管理器的行为。 其中典型的例子是修改程序状态空间的探索模式，如修改为深度优先搜索等，相关文档：  
[https://angr-docs-zh-cn.readthedocs.io/zh-cn/latest/core-concepts/pathgroups.html#id6](https://angr-docs-zh-cn.readthedocs.io/zh-cn/latest/core-concepts/pathgroups.html#id6)

##### 执行引擎

如果前一个 step 导致程序到达一个无法继续的 state，会启用 failure engine。  
如果前一个 step 是以系统调用结束的，会启用 syscall engine。  
如果当前地址被 hook，会启用 hook engine。  
当 unicorn 选项被启用并且 state 中没有符号数据时，会启用 unicorn engine。  
如果上述条件都不满足，则启用 VEX engine。

#### 单步执行

此部分由阅读源码分析得出，可能作为基于 angr 调试器的编写参考

angr 文档并未直接提供单步执行相关内容介绍，且对于不同的引擎，实现单步执行所使用的 API 也不同  
以下是一些解决方案

##### extra_stop_points

```
#%%
import angr
import claripy

p = angr.Project("a.exe",auto_load_libs=False)

state = p.factory.blank_state(addr = 0x140001462)

#%%

sm = p.factory.simulation_manager(state)

sm.step(extra_stop_points = set([0x140001469]))
```

使用引擎底层的 extra_stop_points 参数，作用是执行到目的地址时停止执行 (与 explore 方法的底层调用相同)

##### num_inst

`num_inst` 需要使用的模拟执行引擎支持该参数

`successors`：由引擎执行某个状态后产生的普通的、可满足的状态。它的指令指针可以是符号化的（例如，以用户输入为判断条件的跳转指令），因此这个列表中存储的状态可能实际产生多个后继状态

```
states = p.factory.successors(state,num_inst = 1).successors[0]
```

使用`project.factory.successors(<state>).successors`接口，该接口返回一个`state`数组，为当前`<state>`执行下一步时的状态，`num_inst = 1`指示执行的指令数目为 1，若不指定则按照基本块执行 (相当于`step()`)

一般该数组中只有一个元素，若该 state 所在的指令是一条分支指令 (两个分支都能够到达)，则会出现两个元素，分别为两条分支对应 state

#### Stash

SM 中有许多列表，这些列表被称为 stash，它保存了处于某种状态的 state，stash 有如下几种

<table><thead><tr><th>stash</th><th>描述</th></tr></thead><tbody><tr><td>active</td><td>保存接下来可以执行并且将要执行的状态</td></tr><tr><td>deadended</td><td>由于某些原因不能继续执行的状态，例如没有合法指令，或者有非法指针</td></tr><tr><td>pruned</td><td>与 solve 的策略有关，当发现一个<strong>不可解</strong>的节点后，其后面所有的节点都优化掉放在 pruned 里</td></tr><tr><td>unconstrained</td><td>如果创建 SM 时启用了 save_unconstrained，则被认定为<strong>不受约束</strong>的 state 会放在这，不受约束的 state 是指由用户数据或符号控制的指令指针（例如 eip）</td></tr><tr><td>unsat</td><td>如果创建 SM 时启用了 save_unsat，则被认为<strong>不可满足</strong> (约束冲突) 的 state 会放在这里</td></tr></tbody></table>

默认情况下，state 会被存放在 active 中。  
stash 中的 state 可以通过 move() 方法来转移, 将 fulter_func 筛选出来的 state 从 from_stash 转移到 to_stash。

将 deadended 中输出的字符串包含'100'的 state 转移到 more_then_50 这个 stash 中：  
`simgr.move(from_stash='deadended', to_stash='more_then_50', filter_func=lambda s: '100' in s.posix.dumps(1))`

##### 溢出漏洞利用

在某些情况下，程序存在**栈溢出劫持控制流**漏洞，通过不断加长输入数据的长度直到覆盖跳转地址，此时程序的跳转地址是**被输入的向量控制**的，被放入 unconstrained 列表中

```
def run(self, fmtstr, input_addr):
        input_bvs = claripy.BVS('input_addr', 200 * 8)
        for chr in input_bvs.chop(bits = 8):
            self.state.add_constraints(chr >= '0', chr <= 'z')
        # 向self中投入输入不同长度BVS的状态进行查找
        self.state.memory.store(input_addr, input_bvs)
        self.state.globals['input_val'] = input_bvs

    while not simgr.found:
        # 没有可以继续执行的状态
        if (not simgr.active) and (not simgr.unconstrained):
            break

        # 检查处于 unconstrained stash 的状态，此时该状态的跳转地址被输入的向量所控制，可能存在栈溢出漏洞
        simgr.move(from_stash = 'unconstrained', 
                  to_stash = 'found',
                  filter_func = filter_func)

        # 执行到下一基本块
        simgr.step()
```

执行完成后，使用`posix.dumps`方法获取输入输出

```
state.posix.dumps(0) # 表示到达当前状态所对应的程序输入
state.posix.dumps(1) # 表示到达当前状态所对应的程序输出
```

#### SimSuccessors

```
states = p.factory.successors(state).successors[0]
```

SimSuccessor 的目的是对产生的后继状态进行一个简单的分类，并将这些状态分别存储在不同的属性

可以通过`SimSuccessor[0]`的方式访问

<table><thead><tr><th>属性</th><th>条件</th><th>指令指针</th><th>描述</th></tr></thead><tbody><tr><td>successors</td><td>True（状态约束条件可以满足）</td><td>可以是符号值（解的数目必须小于等于 256）</td><td>由引擎处理某状态后得到的普通的、可满足的后续状态 &lt;br&gt;&lt;br &gt; 指令指针可以是符号值（例如基于用户输入进行跳转）&lt;br&gt;&lt;br &gt; 因此这个列表中存储的状态可能产生多个后继状态</td></tr><tr><td>unsat_successors</td><td>False（状态约束条件不可满足）</td><td>可以是符号值</td><td>不可满足的后继状态 &lt;br&gt;&lt;br &gt; 这些状态的约束条件不可能被满足（如跳转不可能发生，或必须跳转到默认分支）</td></tr><tr><td>flat_successors</td><td>True（状态约束条件可以满足）</td><td>具体值</td><td>如上所述，successors 列表中状态的指令指针可以是符号值，这会带来问题：在执行一次 step 时，一个 state 只能表示某个基本代码块的执行结果 &lt;br&gt;&lt;br &gt; 为了解决这种情况，在处理 successors 列表中符号指令指针的状态时，会先计算出所有可能的具体解决方案（上限为 256），并为每一种解决方案创建一个状态的副本，称为平坦化</td></tr><tr><td>unconstrained_successors</td><td>True（状态约束条件可以满足）</td><td>符号值</td><td>在上述平坦化的过程中，如果符号指令指针的解决方案数目超过 256，就假设这个符号指令指针是一个不受约束的数据（例如用户数据导致栈溢出）&lt;br&gt;&lt;br &gt; 这个假设通常是不合理的，将这些状态放入 unconstrained_successors 列表中</td></tr><tr><td>all_successors</td><td>任何</td><td>可以是符号值</td><td>successors+unsat_successors+ unconstrained_successors&lt;br&gt;</td></tr></tbody></table>

##### 编写自定义函数

可以通过创建一个**继承自 angr.SimProcedure 的类**并**重写 run() 方法**的方式来表示一个自定义函数

```
class MyProcedure(angr.SimProcedure):
    def run(self, arg1, arg2):
        # do something, this's an example
        return self.state.memory.load(arg1, arg2)
```

使用 hook 方式将自定义函数加入到模拟程序执行过程中

```
TARGET_ADDR = 0x401000
HOOK_LENGTH = 5
proj.hook(TARGET_ADDR,  MyProcedure(), HOOK_LENGTH)
proj.hook_symbol('func_to_hook', MyProcedure())
```

#### 断点

angr 支持断点

```
>>> import angr
>>> b = angr.Project('examples/fauxware/fauxware')

# 获取state
>>> s = b.factory.entry_state()

# 添加内存写入断点
>>> s.inspect.b('mem_write')

>>> def debug_func(state):
    print("State %s is about to do a memory write!")

# 添加断点触发事件，该事件将在内存写入完成后触发
>>> s.inspect.b('mem_write', when=angr.BP_AFTER, action=debug_func)

# 或者可以让它回到IPython中
>>> s.inspect.b('mem_write', when=angr.BP_AFTER, action=angr.BP_IPYTHON)
```

### 分析器 analyses

<table><thead><tr><th>名称</th><th>描述</th></tr></thead><tbody><tr><td>CFGFast</td><td>构建程序的快速 控制流图</td></tr><tr><td>CFGEmulated</td><td>构建程序的准确 控制流图</td></tr><tr><td>VFG</td><td>对程序的每个函数执行 VSA，创建 值流图 并检测堆栈变量</td></tr><tr><td>DDG</td><td>计算 数据依赖图 ，允许确定给定值依赖于哪些语句</td></tr><tr><td>BackwardSlice</td><td>根据特定目标计算程序的 反向切片</td></tr><tr><td>Identifier</td><td>在 CGC 二进制文件中识别常见的库函数</td></tr></tbody></table>

求解引擎 Claripy
------------

Claripy 是 angr 所使用的求解引擎，也是 state 有关位向量的底层实现

#### 位向量 BVV

BVV: 位向量  
位向量就是一串比特的序列，这于 python 中的 int 不同，例如 python 中的 int 提供了整数溢出上的包装。而位向量可以理解为 CPU 中使用的一串比特流，需要注意的是，angr 封装的位向量有两个属性：**值**以及它的**长度**

注意 BVV 在创建时具有了初始值放入 state 中后不可更改

相同长度的位向量可以进行运算，对于不同长度的位向量则可以通过 `.zero_extend(extended_bits)` 完成位扩展（0 填充）后进行运算

**位向量 BVV 在 angr 中被用来固定某些数据的初始值**

#### 符号变量 BVS

BVS: 符号变量

BVS 的参数分别是**符号变量名**和**长度**，BVS 中符号变量名参数会影响位向量的名称，但这与 angr 脚本中使用这个符号变量的变量无关。  
此时对符号变量进行运算，做比较判断，都不会得到一个具体的值，而是将这些操作统统保存到符号变量中：

每个符号变量本质上可以看做是一颗抽象语法树 (AST)  
单独生成的符号变量 <BV64 x_42_64> 可以看作是只有一层的 AST，对它进行操作实际上是在扩展 AST，这样的 AST 的构造规则如下：

*   如果 AST 只有根节点的话，那么它必定是符号变量或位向量
*   如果 AST 有多层，那么叶子节点为符号变量和位向量，其他节点为运算符

**BVS 在 angr 中通常被放在需要被求解的数据的位置，并在模拟执行过程中被添加约束**

可以通过 `.op` 与 `.args` 获得位向量的运算类型与参数

#### 符号约束与求解

符号变量的特点是其可以被添加约束并求解

##### 约束

针对符号变量 BVS  
可以通过使用 SM 计算分支得到约束 (约束是在 factory 模拟执行的过程中自动添加的)

```
#创建BVS
p = init_state.solver.BVS('pass',64)
#将BVS放入state中
init_state.memory.store(p_address, p)

# 通过模拟执行得到约束
simgr  = p.factory.simgr(state)
simgr.explore(find=is_good, avoid=is_bad)
found_state = simgr.found[0]

# 使用模拟执行得到的新state求解约束计算得到其中一个结果
found_state.solver.eval(p,cast_to= bytes).decode("utf-8")
```

也可以自行手动添加约束

```
x = state.solver.BVS('x',64)
...
...
state.solver.add(x>0)
state.solver.add(bvs2 > bvs * bvv + bvv2)
```

更多约束操作可以查阅以下文档  
[https://angr-docs-zh-cn.readthedocs.io/zh-cn/latest/appendix/ops.html](https://angr-docs-zh-cn.readthedocs.io/zh-cn/latest/appendix/ops.html)

##### 求解

添加完约束后，可以使用 state.solver.eval(x) 来求解当前状态（即 state）中的符号约束下，x 的值

```
state.solver.eval(x)
```

```
solver.eval(x)  # 给出表达式的一个可能解
sovler.eval_one(x)  # 给出表达式的解，如果有多个解，将抛出错误
solver.eval_upto(x,n)  # 给出表达式的至多n个解
sovler.eval_taleast(x,n)  # 给出表达式的n个解，如果解的数量少于n，则抛出错误
solver.eval_exact(x,n)  # 给出表达式的n个解，如果解的个数不为n，则抛出错误
sovler.min(x)  # 给出表达式的最小解
sovler.max(x)  # 给出表达式的最大解
```

cast_to：将传递结果转换成指定数据类型，目前只能是 int 和 bytes，例如：

```
state.solver.eval(state.solver.BVV(0x41424344，32),cast_to=bytes).decode("utf-8")
```

#### 符号变量之间的运算

符号变量并不能直接同其它类型进行比较（无论是符号变量、位向量、常数、变量）  
如果需要进行符号变量之间的运算，一种方法是使用 Claripy 中的库

示例：与符号变量有关的比较：是返回 x，否返回 y

```
claripy.If(x > y, x, y)
```

该方法能够兼容不同类型参数之间的比较，并返回期望的结果

更多运算方法请查阅文档  
[https://angr-docs-zh-cn.readthedocs.io/zh-cn/latest/advanced-topics/claripy.html#claripy-asts](https://angr-docs-zh-cn.readthedocs.io/zh-cn/latest/advanced-topics/claripy.html#claripy-asts)

其它
--

### Hook

当程序运行到满足某个条件时，调用 hook 函数  
angr 提供了简便的 Hook 接口，能够用于 Hook 被 angr 模拟执行的程序，它通常有两种方式：

*   装饰器`@project.hook`
*   函数`project.hook_symbol`

#### 装饰器`@project.hook`:

该方式适用于简单地修改几行代码的情况

```
# 此处的project指的是将要被hook的angr.Project对象
@project.hook(hook_addr, length=ins_length)
def hook_ins(state):
    #your code
```

这里的`project.hook`来自于使用`angr.Project`方法创建的顶层模块（即`project=angr.Project('filename')`）

需要传入待 hook 的**地址**，以及要跳过的指令的长度  
若要跳过的指令的长度设为 0，则不跳过指令，而是在每次执行该指令之前先执行一次 hook 函数

#### `project.hook/hook_symbol`方法

该方式适用于 hook 某个函数的情况，它能够自动处理函数参数与返回值

```
class Replace_my_add(angr.SimProcedure):
    def run(self, a, b):
        return a - b

hooked_func_name = 'my_add'
project.hook_symbol(hooked_func_name, Replace_my_add())
```

首先创建一个继承自 angr.SimProcedure 的类  
实例化 run 函数，第一个参数为 self，后面的参数与待 hook 的函数参数相同，返回值为 bool  
最后在主函数中，调用 project.hook_symbol，并传入待 hook 的**函数名**以及新建的继承自 SimProcedure 的类的对象即可。

#### 使用库函数 hook 特定地址

```
printf_address = 0x140002560

base_addr = 0x140000000
p = angr.Project("tk.exe", load_options={'auto_load_libs': False},main_opts={'base_addr': base_addr})
p.hook(printf_address, angr.SIM_PROCEDURES['libc']['printf']())
```

注意其中`printf_address`的值是**被 hook 函数的首行指令的地址**，即所有参数已经被传入的状态

如果 hook 的是调用该参数的地址，则需要手动进行输入参数等操作  
找要 hook 的函数，在`\Lib\site-packages\angr\procedures`路径中可以查找有哪些已实现的库函数

#### 返回值

###### 对于装饰器

该方法相当于代码片段的替换，因此如果选择跳过了某条函数调用语句（例如`call my_add`这条汇编指令），那么就需要去查看该函数的返回值，或者说代码片段的最终计算结果，被保存在了什么地方。  
找到这个地方，然后使用内存或者寄存器操作，将返回值填入即可。

###### 对于 hook_symbol

该方式相当于替换了函数的内容，因此直接在重写的 run 方法中使用 return 返回即可

返回值的时候，如果需要根据某个符号变量来决定返回的内容（例如判断一个符号变量等于某个字符串时返回 1，否则返回 0），此时不能用 python 的 if 语句来判断，然后分别返回 1 和 0，这有两个问题：

符号变量与其他内容进行比较（无论是符号变量、位向量、常数、变量），它的真值永远是 false  
如果返回值后续参与了其他的路径选择，此时若只返回了值而不是带约束的符号变量，可能会造成约束丢失  
这种情况下，需要使用`claripy.If`来返回值，该方法用于处理符号变量有关的比较：

```
state.regs.eax = claripy.If(buffer == target, claripy.BVV(1, 32),claripy.BVV(0, 32))
```

上面代码的意思是，当`buffer == target`时，将位向量`<BVV32 1>`传递给 eax，否则将位向量`<BVV32 0>`传递给 eax

错误的做法：

```
if buffer==target:
    state.regs.eax = claripy.BVV(1,32)
else:
    state.regs.eax = claripy.BVV(0,32)
```

若采用这种写法，如果 buffer 或 target 中至少存在一个符号变量，下面的方式将永远判断为 false。

两种方式传递给 eax 的内容也不同，下面一种方式传递的是一个位向量，而上面的方式传递的是**带约束的符号变量**

### SimProcedure

前文已提到使用`SimProcedure`编写 hook 函数，`SimProcedure`同时也是`angr`所有预定义库函数符号化实现的基类，你可以在`angr`源码中的`procedures`文件夹中找到它们的具体实现，也可以编写`SimProcedure`的子类，用于创建自己的实现

当使用 SimProcedure 的子类对象进行 hook 时，该子类对象将会保存在`project._sim_procedures[address]`字典中，

每当执行流程到达某个被 hooked 的地址时，此方法将：

1.  根据调用约定提取参数；
2.  复制当前 SimProcedure 实例以确保线程安全；
3.  运行该实例的 run() 方法。

相关的具体实现可以查看 angr 源码中`sim_procedure.SimProcedure`类的`execute`方法。

#### run 方法额外参数

定义一个继承自 angr.SimProcedure 的类的`run()`方法时，尽管 self 后面的参数需要与待 hook 的函数参数相同，你仍可以使用`**kwargs`来让`run()`方法能够接受额外的关键字参数，实现复用 SimProcedure 子类时的微调

```
# 定义 SimProcedure 时接受 kwargs
class MyProc(SimProcedure):
    def run(self, arg1, **kwargs):
        custom_param = kwargs.get('custom_param', 0)
        ...

# 挂钩时传递自定义参数
project.hook(address, MyProc(custom_param=42))
```

#### 控制流

`SimProcedure`中模拟函数的控制流并不是通过`run`方法直接控制的

对于`run`方法的返回，分析相关源码：

```
class SimProcedure:
    def execute():
        ...
        if inst.returns and inst.is_function and not inst.inhibit_autoret:
            inst.ret(r) # inst 是 SimProcedure 自身的拷贝
        return inst
```

其实质是调用了`self.ret(value)`方法，

`SimProcedure`还提供了其它操作控制流的方法

*   `ret(expr)`:  从一个函数返回
*   `jump(addr)`: 跳转到二进制文件的某个地址
*   `exit(code)`: 终止程序
*   `call(addr, args, continue_at)`: 调用二进制中的函数
*   `inline_call(procedure, *args)`:  通过内联方式调用另一个 SimProcedure 对象，并得到返回

注意：当 SimProcedure 调用了`jump`,`exit`,`call`方法时，`self.inhibit_autoret`属性被设置为真，该对象将不会再自动调用`ret`方法

#### 辅助静态分析

SimProcedure 的一些标志位，这些变量仅用于标记函数属性，帮助静态分析工具理解其行为

NO_RET: 标记该函数 是否不会返回控制流  
ADDS_EXITS:  标记该函数是否会 新增控制流出口，如主动跳转或分支  
IS_SYSCALL： 是否为系统调用

### 获取特定函数执行的 state

该方法用于单独模拟执行二进制文件中的某个函数，为保证分析结果准确，该函数的执行结果必须仅与传入的参数有关，不需要额外的初始化步骤

#### 手动设定起始位置及参数

以一个简单的解密验证函数为例：该函数接受一个 char 指针`*s1`和一个指示字符串长度的整数`a2`  
![](https://attach.52pojie.cn/forum/202503/29/152040xfy1zf5z06vg6giq.png)

**Pasted image 20240528011517.png** _(105.89 KB, 下载次数: 1)_

[下载附件](forum.php?mod=attachment&aid=Mjc2Njk0MnxhMjRiODE0ZHwxNzUxNTEyNDAzfDB8MjAxOTc2Mg%3D%3D&nothumb=yes)

2025-3-29 15:20 上传

参数在栈上的相对位置，可以通过反汇编代码得到，其**偏移值为正数** (s1,arg_4)

```
# 设置起始地址为函数地址
init_state = p.factory.blank_state(addr=<function_addr>)
# esp是有数值的，而ebp未初始化             
# 初始化栈底指针
init_state.regs.ebp = init_state.regs.esp

# 添加参数arg_4/s2
init_state.stack_push(8)

# 添加参数 s1, password_addr 可以为任一空地址，模拟执行过程中会自动为该地址生成符号变量
init_state.stack_push(password_addr)

# 此处本应存储调用该函数的地址，由于直接执行函数跳过了该步骤，因此要执行push命令一次以还原该步骤
init_state.stack_push(0)
```

初始化 init_state 完成后，仅需执行函数到结尾，提取出`password_addr`中的符号变量，即可求解出正确输入

#### angr 自带调用函数方法

`init_state = p.factory.call_state(func_addr, param1, param2)`

还需要注意的一点是，动态链接库在加载时需要重定位，可以在建立项目时用 load_options 设定重定位的**基址**，就像这样：

```
p = angr.Project(path_to_binary,
                 auto_load_libs=False,
                 load_options={'main_opts': {
                     'custom_base_addr': base_addr
                 }})
```

如果不设立基址，通常 angr 会默认加载到 0x400000 处，在 IDA 中看到的各个指令的地址都只是相对地址，需要加上基址才能找到

### 断点

[https://docs.angr.io/en/latest/core-concepts/simulation.html#breakpoints](https://docs.angr.io/en/latest/core-concepts/simulation.html#breakpoints)

angr 的断点功能通过`state`类的`inspect`接口实现

```
def track_reads(state):
    print('Read', state.inspect.mem_read_expr, 'from', state.inspect.mem_read_address)

s.inspect.b('mem_read', when=angr.BP_AFTER, action=track_reads)
```

效果：在发生内存读取操作之后，执行 track_reads，打印指令地址和读取地址  
`when`指示断点在指令前面 (`angr.BP_BEFORE` ) 还是后面 (`angr.BP_AFTER`)

```
# 下面的例子将会在程序可能(考虑到符号化的地址)往0x1000地址处写之前断下
>>> s.inspect.b('mem_write', mem_write_address=0x1000)

# 下面的例子将会在程序只能往内存0x1000处写数据之前断下：
>>> s.inspect.b('mem_write', mem_write_address=0x1000, mem_write_address_unique=True)

# 下面的例子将会在指令地址0x8000执行后生效，但是只有0x1000是从内存中读出的表达式的可行解：
>>> s.inspect.b('instruction', when=angr.BP_AFTER, instruction=0x8000, mem_read_expr=0x1000)
```

#### 断点事件

[https://blog.csdn.net/water_likly/article/details/88326858](https://blog.csdn.net/water_likly/article/details/88326858)  
[https://docs.angr.io/en/latest/core-concepts/simulation.html#breakpoints](https://docs.angr.io/en/latest/core-concepts/simulation.html#breakpoints)

IR:[https://decaf-lang.github.io/minidecaf-tutorial-deploy/docs/lab1/ir.html](https://decaf-lang.github.io/minidecaf-tutorial-deploy/docs/lab1/ir.html)

<table><thead><tr><th>断点事件</th><th>含义</th><th>补充</th></tr></thead><tbody><tr><td>mem_read</td><td>内存读取</td><td>若读取的是符号向量，则每读取一个向量触发一次</td></tr><tr><td>mem_write</td><td>内存写入</td><td></td></tr><tr><td>address_concretization</td><td>要访问内存的地址是符号化向量</td><td></td></tr><tr><td>reg_read</td><td>读取寄存器</td><td></td></tr><tr><td>reg_write</td><td>写入寄存器</td><td></td></tr><tr><td>tmp_read</td><td>读取临时变量</td><td>临时变量即 symbolic 所创建的向量</td></tr><tr><td>tmp_write</td><td>写入临时变量</td><td></td></tr><tr><td>expr</td><td>一个表达式被创建</td><td>IR 中的算术运算结果或常数</td></tr><tr><td>statement</td><td>正在翻译一条 IR 语句 (中间码)</td><td></td></tr><tr><td>instruction</td><td>正在翻译一条汇编语句</td><td></td></tr><tr><td>irsb</td><td>正在翻译一个新的块</td><td></td></tr><tr><td>constraints</td><td>新的约束被加入到状态中</td><td></td></tr><tr><td>exit</td><td>该状态执行时产生了一个后继</td><td>若有多个分支，则触发多次</td></tr><tr><td>fork</td><td>一个符号块执行状态出现分支</td><td>多个后继</td></tr><tr><td>symbolic_variable</td><td>创建了一个新的符号变量</td><td>第一次<strong>使用未初始化的寄存器</strong>或<strong>读取未知的内存</strong>时，都会为该未知数据创建一个符号向量以保存运行时的约束</td></tr><tr><td>call</td><td>执行了一个 Call 指令</td><td>函数调用</td></tr><tr><td>return</td><td>执行了一个 Return 指令</td><td>函数返回</td></tr><tr><td>simprocedure</td><td>执行了外部程序 (Dll) 或系统调用</td><td>如<code>strlen fopen malloc</code>等库函数</td></tr><tr><td>dirty</td><td>执行 dirty IR 回调</td><td></td></tr><tr><td>syscall</td><td>执行了系统调用</td><td></td></tr><tr><td>engine_process</td><td>调用了模拟执行引擎</td><td></td></tr></tbody></table>

需要注意的是，当使用`state`的各种方法进行内存、寄存器等操作时，都会触发断点，从而执行断点函数

#### 事件属性

事件属性是触发断点事件下与该断点相关的属性  
既可以用来获取某断点触发时的信息，也可以用来限制触发断点的条件

事件属性`mem_write_address`作为`mem_write`事件的限制，限制了触发断点的内存地址

```
new_state.inspect.b('mem_write', mem_write_address=0x6021f1, when=angr.BP_AFTER, action=_func)
```

属性在 action 函数使用时，需加上`state.inspect.`

##### 通用属性

在执行任意代码状态下时都有值的属性

<table><thead><tr><th>属性名</th><th>含义</th><th>补充</th></tr></thead><tbody><tr><td>instruction</td><td>触发断点处的指令地址</td><td>当调用程序外的库函数、触发 engine_process 时，该属性值为 None</td></tr><tr><td>address</td><td>触发断点处的块地址</td><td>包括外部函数的块地址</td></tr><tr><td>expr</td><td>IR 表达式</td><td>输出的是表达式的运算结果或目标寄存器编号</td></tr><tr><td>expr_result</td><td>IR 表达式的值</td><td>输出运算结果或目标寄存器的值</td></tr><tr><td>statement</td><td>IR 表达式的索引</td><td></td></tr></tbody></table>

要获取断点处的指令地址，更好的方法是使用`state.scratch.ins_addr`

##### 特殊属性

<table><thead><tr><th>事件类型</th><th>属性名</th><th>含义</th><th>限制</th></tr></thead><tbody><tr><td>mem_read</td><td>mem_read_address</td><td>读取的内存的地址</td><td>无</td></tr><tr><td>mem_read</td><td>mem_read_expr</td><td>执行读内存操作的表达式地址若读取的是符号向量，则会打印向量名，需使用 instruction 获取指令地址</td><td>触发后</td></tr><tr><td>mem_read</td><td>mem_read_length</td><td>读取的内存长度 (字节单位)</td><td></td></tr><tr><td>mem_read</td><td>mem_read_condition</td><td>读取的内存状态</td><td></td></tr><tr><td>mem_write</td><td>mem_write_address</td><td>写入的内存的地址</td><td></td></tr><tr><td>mem_write</td><td>mem_write_length</td><td>写入的内存的长度 (字节单位)</td><td></td></tr><tr><td>mem_write</td><td>mem_write_expr</td><td>执行写内存操作的表达式地址若读取的是符号向量，则会打印向量名</td><td></td></tr><tr><td>mem_write</td><td>mem_write_condition</td><td>写入的内存状态</td><td></td></tr><tr><td>reg_read</td><td>reg_read_offset</td><td>访问寄存器偏移量 (/8 = 编号)</td><td></td></tr><tr><td>reg_read</td><td>reg_read_length</td><td>读取寄存器长度 (字节单位)</td><td></td></tr><tr><td>reg_read</td><td>reg_read_expr</td><td>访问得到的寄存器值</td><td>触发后</td></tr><tr><td>reg_read</td><td>reg_read_condition</td><td>读取寄存器状态</td><td></td></tr><tr><td>reg_write</td><td>reg_write_offset</td><td>写入寄存器偏移量 (/8 = 编号)</td><td></td></tr><tr><td>reg_write</td><td>reg_write_length</td><td>写入寄存器长度 (字节单位)</td><td></td></tr><tr><td>reg_write</td><td>reg_write_expr</td><td>写入寄存器值</td><td></td></tr><tr><td>reg_write</td><td>reg_write_condition</td><td>写入寄存器状态</td><td></td></tr><tr><td>tmp_read</td><td>tmp_read_num</td><td>读取的临时值的数量</td><td></td></tr><tr><td>tmp_read</td><td>tmp_read_expr</td><td>读取的临时值的真值</td><td>触发后</td></tr><tr><td>tmp_write</td><td>tmp_write_num</td><td>写入的临时值的数量</td><td></td></tr><tr><td>tmp_write</td><td>tmp_write_expr</td><td>写入的临时值的真值</td><td>触发后</td></tr><tr><td>constraints</td><td>added_constraints</td><td>已添加的约束表达式列表</td><td></td></tr><tr><td>call</td><td>function_address</td><td>调用的函数的地址</td><td></td></tr><tr><td>exit</td><td>exit_target</td><td>返回的目标地址</td><td></td></tr><tr><td>exit</td><td>exit_guard</td><td>返回保护的表达式</td><td></td></tr><tr><td>exit</td><td>exit_jumpkind</td><td>返回的跳转方式</td><td></td></tr><tr><td>symbolic_variable</td><td>symbolic_name</td><td>创建符号变量的变量名</td><td>触发后</td></tr><tr><td>symbolic_variable</td><td>symbolic_size</td><td>创建符号变量的大小</td><td>触发后</td></tr><tr><td>symbolic_variable</td><td>symbolic_expr</td><td>创建符号变量的真值</td><td>触发后</td></tr><tr><td>address_concretization</td><td>n_strategy</td><td>用于解析地址的 SimConcreteization 策略。断点处理程序可以修改这一点，以更改将应用的策略。如果断点处理程序将其设置为 None，则将跳过此策略</td><td></td></tr><tr><td>address_concretization</td><td>n_action</td><td>对符号化向量的内存操作</td><td></td></tr><tr><td>address_concretization</td><td>n_memory</td><td>对符号化向量的内存对象</td><td></td></tr><tr><td>address_concretization</td><td>n_expr</td><td>操作的符号向量的真值</td><td></td></tr><tr><td>address_concretization</td><td>n_add_constraints</td><td>是否为此符号向量添加约束</td><td></td></tr><tr><td>address_concretization</td><td>n_result</td><td>已解析内存地址（整数）的列表。断点处理程序可以覆盖这些，以实现不同的解析结果</td><td>触发后</td></tr><tr><td>syscall</td><td>syscall_name</td><td>系统调用函数名</td><td></td></tr><tr><td>simprocedure</td><td>simprocedure_name</td><td>外部调用函数名</td><td></td></tr><tr><td>simprocedure</td><td>simprocedure_addr</td><td>外部调用函数地址</td><td></td></tr><tr><td>simprocedure</td><td>simprocedure_result</td><td>外部调用返回值</td><td>触发后</td></tr><tr><td>simprocedure</td><td>simprocedure</td><td>外部实际调用对象 (区分实际调用和模拟调用)</td><td></td></tr><tr><td>engine_process</td><td>sim_engine</td><td>使用的模拟执行引擎</td><td></td></tr><tr><td>engine_process</td><td>successors</td><td>执行引擎结果的 successors 对象</td><td></td></tr></tbody></table>

### 库函数

在默认情况下 angr 会使用 SimProcedures 里面的符号函数摘要集替换库函数，本质上是在库函数上设置了 Hooking，这些 hook 函数高效地模仿库函数对状态的影响，而 hook 函数只模仿了库函数对状态的影响，实际内部的操作并没有实现，因此也就不会产生额外分支

其结果是 angr 对于库函数只会分出一条路径，而不关心库函数内部是怎样实现的，从而减少了产生的路径

因此在模拟执行前，要尽可能地将静态函数替换为存在的相同功能库函数

### 优化执行速度

[https://docs.angr.io/en/latest/advanced-topics/speed.html](https://docs.angr.io/en/latest/advanced-topics/speed.html)

*   尝试使用 pypy 执行。pypy 是 CPython 的一种快速且功能强大的替代方案  
    PyPy 适合纯 Python 应用程序，不适用于 C 扩展，有时它的运行速度都要比在 CPython 中慢得多。
    
*   尽量避免加入共享库，即导入`Project`时设置`auto_load_libs=False`，angr 会主动寻找兼容的共享库以保证模拟效率
    
*   多使用`Hook`和`SimProcedures`，将复杂函数 Hook 为直接的代码或使用 angr 自带的模拟函数符号
    
*   使用 SimInspect
    
*   写出 concretization strategy
    
*   使用替换求解器。可以使用`angr.options.REPLACEMENT_SOLVER`状态选项启用它。替换求解器允许指定在求解时应用的 AST 替换。如果添加替换，以便在执行求解时将所有符号数据替换为具体数据，则运行时间将大大缩短。  
    用来添加替换的 api 是`state.se._solver.add_replacement(old, new).`
    
*   使用`LAZY_SOLVES`选项  
    `st1 = p.factory.full_init_state(add_options={angr.options.LAZY_SOLVES})`  
    这是一个尽可能少地检查状态可满足性的选项  
    在**运行步骤多且分支少**的程序中很有效
    

##### 大批代码的模拟执行

*   使用`unicorn`引擎，通过`angr.options.unicorn`选项启用
    
    ```
    state = p.factory.full_init_state(
        add_options=angr.options.unicorn,
       )
    ```
    
    需要注意的是，`angr.options.unicorn`选项是多个选项的集合
    
    ```
    {
    'UNICORN',
    'UNICORN_HANDLE_CGC_TRANSMIT_SYSCALL',
    'UNICORN_SYM_REGS_SUPPORT',
    'UNICORN_TRACK_BBL_ADDRS',
    'UNICORN_TRACK_STACK_POINTERS',
    'ZERO_FILL_UNCONSTRAINED_REGISTERS'}
    ```
    

与其它选项混合使用：`|`

> ```
> p.factory.full_init_state(add_options=angr.options.unicorn | {angr.options.LAZY_SOLVES})
> ```

*   启用快速内存和快速寄存器。相关的状态选项是`angr.options.FAST_MEMORY`和`angr.options.FAST_REGISTERS`  
    这些选项将内存 / 寄存器切换到一个不太密集的内存模型，以牺牲精度提高速度。  
    注意：与具体化策略不兼容。
    
*   提前精确要输入的数据  
    当输入的数据是可以确定时，直接填入具体  
    当发现输入的数据的具体特征时，向这些数据添加约束
    
    如已知数据长度，应创建对应的符号向量  
    已知输入的数据是通过键盘输入的，应限制输入范围在 ascii 码可打印字符内
    
    ```
    for k in flag_chars:
        st.solver.add(k < 0x7f)
        st.solver.add(k > 0x20)
    ```
    
*   使用`afterburner`：在使用 unicorn 引擎时添加选项`UNICORN_THRESHOLD_CONCRETIZATION`，设置一个阈值，若 angr 模拟执行时达到了阈值，它将使符号值凝结，以便模拟执行可以花费更多时间在 Unicorn 上
    

`state.unicorn.concretization_threshold_memory`  
在具体化开始后容忍内存被 unicorn 拒绝的次数  
`state.unicorn.concretization_threshold_registers`  
在具体化开始后容忍寄存器被 unicorn 拒绝的次数

一旦 afterburner 具体化了一些东西，就会失去对这个变量的跟踪。

##### 内存优化

*   要释放状态执行的历史数据，使用`state.history.trim()`
    *   要直接禁用对 ip 数据和 sp 数据的跟踪，使用`UNICORN_TRACK_BBL_ADDRS`和`UNICORN_TRACK_STACK_POINTERS.`

### 执行结果分析

如果某次执行得到了`errored`的`state`，使用`simgr.errored`打印具体的错误信息  
使用`simgr.errored[n].debug()`获得错误状态的 PDB shell\  
使用`import pprint; pprint.pprint(state.history.descriptions.hardcopy)`查看状态经过的路径  
使用`state.history.bbl_addrs.hardcopy`查看模拟执行过程中经过的块  
使用`print(state.solver.constraints)`查看该状态下的所有约束  
使用`state.history.events[-1]`查看状态最后经过的一个分支

### 获取调试信息

```
import logging
logging.getLogger('angr').setLevel('DEBUG')
```

正常情况下，angr 在警告时发出调试记录

对于`angr.analyses.cfg`，使用`logging.getLogger('angr.analyses').setLevel('INFO')`获取使用 CFG 的详细调试记录

### 状态插件编写

在前文已经介绍了 angr.state 自带的一些插件：`state_plugins.callstack`、`state_plugins.history`  
这些插件的主要功能是获取模拟执行过程中产生的各种信息

你也可以编写自己的状态插件实现对于这些信息的分析

```
import angr

# 首先，需要定义一个继承于angr.SimStatePlugin的类
class MyFirstPlugin(angr.SimStatePlugin):
    def __init__(self, foo):
        super(MyFirstPlugin, self).__init__()
        self.foo = foo

    # 实现copy方法，以保证在对安装了该插件的state使用copy()方法时，插件本身也能被正确拷贝
    @angr.SimStatePlugin.memo
    def copy(self, memo):
        return MyFirstPlugin(self.foo)

# 创建state
state = angr.SimState(arch='AMD64')

# 使用register_plugin方法为state安装插件
state.register_plugin('my_plugin', MyFirstPlugin('bar'))

# 为state安装插件后，该插件实例会自动变为状态的可用属性
state.my_plugin.foo

# 此外，，也可以使用getplugin方法进行访问
state.getplugin("plugin_name")
print(state.get_plugin("my_plugin").foo)
```

##### 状态插件初始化

状态插件`SimStatePlugin`需要访问其对应的状态`state`，但它们之间的绑定关系并不是在插件初始化时就确定的，这是因为一个状态可能拥有多个插件，而一个插件又有可能与其它插件有交互，这存在大量的初始化顺序和依赖性问题

为了避免这一问题，状态并未作为状态插件的初始化参数传入，而是通过单独的 `set_state` 方法设置到插件中。

然而，由于无法保证插件之间 `set_state` 方法的调用顺序，插件无法安全地与其它插件进行交互，因此额外设置了`init_state` 方法用于需要与其他插件交互的初始化操作

以下是从 angr 源码中截取的片段

```
class SimState(Generic[IPTypeConc, IPTypeSym], PluginHub[SimStatePlugin]):
    ...
    def _set_plugin_state(self, plugin: SimStatePlugin, inhibit_init: bool = False):
        plugin.set_state(self)
        if plugin.STRONGREF_STATE:
            plugin.set_strongref_state(self)
        if not inhibit_init:
            plugin.init_state()
```

分析代码可以得知：插件刚开始初始化时，只会调用`plugin.set_state`方法，只有当`inhibit_init`为`false`时，才会进一步初始化插件，调用`plugin.init_state()`方法，此时其它大部分插件也已初始化完成

代码示例：

```
class MyFirstPlugin(angr.SimStatePlugin):

    # 插件第一步初始化，该阶段插件只能够访问state本身，进行基础初始化以保证插件自身可用性
    def set_state(self, state):
        super(SimStatePlugin, self).set_state(state)
        self.symbolic_word = claripy.BVS('my_variable', self.state.arch.bits)

    # 跨插件依赖的初始化，此时插件可以与其它插件进行交互
    def init_state(self):
        if self.region is None:
           self.region = self.state.memory.map_region(SOMEWHERE, 0x1000, 7)
```

##### 访问状态

状态插件通过执行`set_state`方法后得到`state`属性，该属性是实际状态的一个引用，状态插件通过使用`self.state`来获得状态的更多消息

##### 序列化

如果需要对使用了自定义插件的状态进行序列化，则需要重写`__getstate__/__setstate__`魔术方法。  
不需要保留`state`属性，因为在反序列化时，`set_state`方法将会被重新调用

##### 设置为默认插件

要注册插件，除了对特定状态使用`state.register_plugin('my_plugin', MyFirstPlugin('bar'))`，还有一种方法是：直接将该插件注册为**默认插件**，当通过`state.my_plugin`访问插件时，若状态中尚未存在该插件，系统会调用`MyPlugin`的构造函数生成新实例，并通过`set_state`将其绑定到当前状态。

```
# 使用register_default类方法时，该方法会将插件对象的定义加入到PluginPreset._default_plugins[name]中，state调用未注册对象时，会首先从此时搜索，搜索成功时直接注册
MyFirstPlugin.register_default("my_plugin")

state = angr.SimState(arch='AMD64')
print(state.my_plugin.foo)
```

### 分析器编写

分析器用于分析顶层接口 Project 对象

```
class MockAnalysis(angr.Analysis):
    def __init__(self, option):
        self.option = option

# 注册该类到ANGR的全局分析列表中
angr.AnalysesHub.register_default('MockAnalysis', MockAnalysis) 

proj = angr.Project("/bin/true")

# 注册完成后，该类将成为analyses的一个属性并在被访问时实例化
mock = proj.analyses.MockAnalysis('this is my option')
assert mock.option == 'this is my option'
```

#### 访问 Project 对象

要访问分析器对应的 Project 对象，可以直接使用自身的 project 属性进行访问

```
class ProjectSummary(angr.Analysis):
    def __init__(self):
        self.result = 'This project is a %s binary with an entry point at %#x.' % (self.project.arch.name, self.project.entry)
```

#### 分析异常处理

在批量分析（如遍历程序所有函数）时，即使部分分析失败，仍可以保留成功部分的结果。

angr 分析器自带的异常处理方法：`self._resilience()`

使用示例：

```
class ComplexFunctionAnalysis(angr.Analysis):
    def __init__(self):
        self._cfg = self.project.analyses.CFG()
        self.results = { }
        for addr, func in self._cfg.function_manager.functions.items():
            with self._resilience():
                if addr % 2 == 0:
                    raise ValueError("can't handle functions at even addresses")
                else:
                    self.results[addr] = "GOOD"
```

默认情况下，所有异常被记录到`self.errors`属性中

你也可以通过传递`name`参数，为这部分异常分类

```
with self._resilience():
```

此时异常将被记录到`self.named_errors`（字典形式）中

指定`exception`参数，指定捕获的异常类型

```
with self._resilience(exception=(TypeError, ValueError)):
```

### 相关开源项目

基于 angr 分析引擎的可视化二进制分析工具  
[https://github.com/angr/angr-management](https://github.com/angr/angr-management)

angr 二进制分析框架的资源 / 工具和分析的集合  
[https://github.com/degrigis/awesome-angr](https://github.com/degrigis/awesome-angr)

从调试器状态生成 angr state 对象的抽象库  
[https://github.com/andreafioraldi/angrdbg](https://github.com/andreafioraldi/angrdbg)

### 底层源码解析

[https://www.anquanke.com/post/id/231460](https://www.anquanke.com/post/id/231460)  
[https://github.com/another1024/angr-analysis/tree/master](https://github.com/another1024/angr-analysis/tree/master)

### REFERENCE

官方网站  
[https://angr.io/](https://angr.io/)  
[https://angr-docs-zh-cn.readthedocs.io/zh-cn/latest/quickstart.html](https://angr-docs-zh-cn.readthedocs.io/zh-cn/latest/quickstart.html)

[https://www.cnblogs.com/fancystar/p/7851736.html](https://www.cnblogs.com/fancystar/p/7851736.html)  
[https://www.cnblogs.com/level5uiharu/p/16925991.html](https://www.cnblogs.com/level5uiharu/p/16925991.html)  
[https://arttnba3.cn/2022/11/24/ANGR-0X00-ANGR_CTFhttps://xz.aliyun.com/t/7117](https://arttnba3.cn/2022/11/24/ANGR-0X00-ANGR_CTFhttps://xz.aliyun.com/t/7117)