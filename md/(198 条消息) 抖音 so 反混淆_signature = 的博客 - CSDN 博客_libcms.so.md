> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/acy007/article/details/121740760)

[arm](https://so.csdn.net/so/search?q=arm&spm=1001.2101.3001.7020) 架构下 ollvm 平坦化的反混淆

app 样本：douyin9.9.0  
so 样本：libcms.so  
逆向工具：ida,jadx

观察函数控制流程图 (CFG)
---------------

所要获取的签名算法位于 leviathan 方法中，该方法是个 native 方法，在 libcms.so 中没有静态关联函数，所以我们需要找到其动态注册的位置。  
![](https://img-blog.csdnimg.cn/aa157aa2e288416eb585ac72779ab3f2.png?)

首先，分析 Jni_OnLoad 函数，Jni 函数的动态注册一般都在这里进行：

![](https://img-blog.csdnimg.cn/33195305348b4eb0b2d15be1582cfa38.png?)

关于 ollvm 的基本原理，大家可以在 https://github.com/obfuscator-[llvm](https://so.csdn.net/so/search?q=llvm&spm=1001.2101.3001.7020)/obfuscator 上进行详细了解，在这里不做过多说明。

可以看到 JNI_OnLoad 已经被彻底地混淆，通过流程图结构，我们可以初步推断是 ollvm 的指令平坦，进一步观察发现大部分流程块都会跳转至 0x46FE，从而可以得出 0x46FE 是主分发器所在块，r0 寄存器作为保存索引值的寄存器 (在这里我将混淆里用于索引到相关块的常量值称为索引值)。

![](https://img-blog.csdnimg.cn/af26412adfab404fb52190ba0d91eb7f.png?)

去除一般混淆
------

仔细观察函数内容，发现其中还包含另外一种混淆结构，如下图所示：  
![](https://img-blog.csdnimg.cn/3ea9f0d1385a4aa7bf1c69aaf4d1bfab.png?)  
从 0x478c 的位置 PUSH {R0-R3} 开始是另一个混淆结构的开始，功能比较简单，分析之后其作用是直接跳转到另一个地址，跳转的位置是到 POP {R0-R3} 指令的下一个地址，即 0x47E4 的位置。  
去除这中混淆的方式也比较简单，直接把 0x478c 处修改成到 0x47E4 的跳转即可，中间的混淆代码可以用 nop 填充掉，鉴于整个函数有多处这样的混淆，可以写个 ida 脚本批量处理，通过搜索特征字节码来定位混淆代码的起始与结束位置，例如混淆开始处的 opcode 为 0F B4 78 46 79 46(为了避免错误识别可以多判断几个字节码)，结束处的 opcode 为 0F BC。

脚本内容如下：

```
from idc import *
from idautils import *
from idaapi import *
from keystone import *

ks=Ks(KS_ARCH_ARM,KS_MODE_THUMB)

def ks_disasm(dis_str):
    global ks
    encoding,count=ks.asm(dis_str)
    return encoding

func_start=0x44c8
func_end=0x498c
patch_start = None
patch_end = None
for i in range(func_start, func_end):
    #PUSH{R0-R3} or PUSH{R0-R3,R7}
    #MOV R0,PC
    #MOV R1,PC
    if get_bytes(i,6,0) == b'x0fxb4x78x46x79x46' or get_bytes(i,6,0) == b'x8fxb4x78x46x79x46':
        patch_start = i
    #POP{R0-R3}
    if get_bytes(i,2,0) == b'x0fxbc':
        if patch_start != None:
            patch_end = i + 2
    #POP{R7},POP{R0-R3}
    if get_bytes(i,4,0) == b'x07xbcx88xbc':
        if patch_start != None:
            patch_end = i + 4
    if nop_start != None and nop_end != None:
        for i in range(0, patch_end - patch_start, 2):
            patch_byte(nop_start+i,0x00)
            patch_byte(nop_start+i+1,0xbf)
        dis_str = 'b #{}-{}'.format(patch_end, patch_start)
        jmp_addr.append(patch_start)
        encoding = ks_disasm(dis_str)
        for item in encoding:
            print('{}'.format(hex(item)))
        for j in range(len(encoding)):
            patch_byte(patch_start+j,encoding[j])
        patch_start = None
        patch_end = None
```

寻找相关块
-----

准备工作已完成，正式进入我们的主题，还原 ollvm 的混淆代码——关于如何还原流程平坦化，有三个问题需要解决：

一、找出流程里所有的相关块  
二、找出各个相关块之间执行的先后顺序  
三、使用跳转指令将各个相关块连接起来

第一个问题，通过以下规律找到差不多所有的相关块：  
1、后继是分发器块的，一般是相关块 (注意是一般，因为有的次分发器也会跳转至主分发器  
2、相关块的前继块中，有且只有当前相关块一个后继块的，也是相关块

Github 上有许多二进制分析框架 (angr，barg，miasm 等等)，可以对函数生成 CFG(控制流程图)，由于 barg 和 miasm 对 arm-v7a 指令的支持不是很完全，有些代码无法反汇编，最终我选用了 angr。(github 地址：https://github.com/angr/angr）

这里我参考了 github 上的一个 x86 ollvm 反混淆脚本 (https://github.com/SnowGirls/deflat):

```
filename = 'your sopath/libcms.so'
#start of JNI_Onload
start_addr=0x44c8
end_addr=0x498c
project = angr.Project(filename, load_options={'auto_load_libs': False})
cfg = project.analyses.CFGFast(regions=[(startaddr,endaddr)],normalize='True')
#函数为Thumb-2指令，所以寻找functions时的函数地址应该加上1
target_function = cfg.functions.get(start_addr+1)
#将angr的cfg转化为转化为类似ida的cfg
supergraph = am_graph.to_supergraph(target_function.transition_graph)
#手动寻找主分发器，返回块，序言块
main_dispatcher_node=get_node(supergraph,0x46ff)
retn_node=get_node(supergraph,0x4967)
prologue_node=get_node(supergraph,0x44c9)
#手动填入保存索引值的寄存器
regStr='r1'
```

project.analyses.CFGFast 方法可以快速生成一个二进制代码的控制流程图，但这种流程图和 ida 的流程有一些不同 (比如会将 BL 函数调用前后的代码分割成两个基本块)，而 am_graph.to_supergraph 方法，则可将其进一步转化为类似 ida 流程图的结构。

生成 cfg 后我们通过之前提到的规律来寻找所有的相关块，先找出所有跳转至主分发器的基本块，再筛选其中的相关块。

![](https://img-blog.csdnimg.cn/c2e5be23982644ddab07de92a43d79d9.png?)  
还有一些如上图红框所示 1 其实是一个次分发器，但最后跳转到了主分发器，对于这种情况，我们需要通过匹配特征反汇编指令来过滤掉这些分发器。而红框所示 2 包含了栈指针操作，但可以发现其并未使栈平衡，观察整个函数也并未发现其他平衡栈的操作，所以可以初步判定此类型的块也可能是混淆块，也应该过滤掉。

在这里我把直接跳转到主分发器的相关块称作一级相关块，对于没有跳转到主分发器而是跳转到另一个相关块的相关块称为次级相关块，则可以通过递归的方块由一级相关快逐层向上寻找，直到找出所有相关块。

由于编译器的优化导致某些相关块在改变了后并未跳转至主分发器，而是跳到了另一个共用代码段。如下图所示，0x14f44 处是主分发器，因为编译优化的原因上面三处相关块都会引用同一个代码块，这样在后面的符号执行时，需要将 0x14ac8 处的相关块看做是上面三个相关块的一部分，符号执行时应该跳过这个共用的相关块。最后在 patch 指令时，将共用的相关块添加至每个引用它的相关块的末尾，然后再进行跳转。

![](https://img-blog.csdnimg.cn/d9b0fd4b028040e5ac7f06ec548251ab.png?)  
寻找所有相关块的代码：

```
def get_relevant_nodes(supergraph):
    global pre_dispatcher_node, prologue_node, retn_node,special_relevant_nodes,regstr
    relevants = {}

    #寻找那些没有直接跳转到主分发器的相关块
    def find_other_releventnodes(node,isSecondLevel):
        prenodes = list(supergraph.predecessors(node))
        for prenode in prenodes:
            if len(list(supergraph.successors(prenode)))==1:
                relevants[prenode.addr] = prenode
                if isSecondLevel and not(is_has_disasmes_in_node(node, [['mov', regstr]]) or is_has_disasmes_in_node(node, [['ldr', regstr]])):
                    #由于编译器的优化导致某些相关块在改变了索引值后并未跳转至主分发器，而是跳到了另一个共用代码段。
                    special_relevant_nodes[prenode.addr]=node.addr
                find_other_releventnodes(prenode,False)

    nodes = list(supergraph.predecessors(main_dispatcher_node))

    for node in nodes:
        #获取基本块的反汇编代码
        insns = project.factory.block(node.addr).capstone.insns
        if node in relevant_nodes:
            continue
        #过滤跳转到主分发器的子分发器
        elif len(insns)==4 and insns[0].insn.mnemonic.startswith('mov') and 
                insns[1].insn.mnemonic.startswith('mov') and 
                insns[2].insn.mnemonic.startswith('cmp') and 
                is_jmp_code(insns[3].insn.mnemonic):
            continue
        elif len(insns)==1 and is_jmp_code(insns[0].insn.mnemonic):
            continue
        elif len(insns)==2 and insns[0].insn.mnemonic.startswith('cmp') and 
            is_jmp_code(insns[1].insn.mnemonic):
            continue
        elif len(insns)==5 and (is_has_disasmes_in_node(node,[['mov',''],['mov',''],    ['cmp',''],['ldr',regstr]]) or 
            is_has_disasmes_in_node(node,[['mov',''],['mov',''],['cmp',''],                ['mov',regstr]]) )and 
            is_jmp_code(insns[4].insn.mnemonic):
            continue

        #过滤有add sp操作但没有sub sp操作的块
        elif is_has_disasmes_in_node(node,[['add','sp']]) and 
                is_has_disasmes_in_node(node,[['nop','']]):
            continue

        #将认定为相关块的基本块保存
        relevants[node.addr]=node
        #寻找其他没有跳转到主分发器的相关块
        find_other_releventnodes(node,True)
    return relevants

def is_startwith(str1,str2):
    if str2=='':
        return True
    return str1.startswith(str2)

#是否是跳转指令
def is_jmp_code(str):
    if not str.startswith('b'):
        return False
    if str.startswith('bl'):
        if str.startswith('ble') and not str.startswith('bleq'):
            return True
        else: return False
    return True

#是否是函数调用指令
def is_call_code(str):
    if not str.startswith('bl'):
        return False
    if str.startswith('ble') and not str.startswith('bleq'):
        return False
    return True

def is_has_disasmes_in_insns(insns,disinfolist):
    size = len(disinfolist)
    for i in range(len(insns) - (size-1)):
        is_has = True
        for j in range(size):
            insn=insns[i+j].insn
            disinfo=disinfolist[j]
            if not (is_startwith(insn.mnemonic,disinfo[0]) and is_startwith(insn.op_str,disinfo[1])):
                is_has=False
                break
        if is_has: return True
    return False

def is_has_disasmes_in_node(node,disinfolist):
    insns=project.factory.block(node.addr,node.size).capstone.insns
    return is_has_disasmes_in_insns(insns,disinfolist)
```

符号执行
----

找到所有相关块后，下一步要做的就是找出各个相关块的执行先后关系。使用 angr 的符号执行框架是一个不错的选择，它能够模拟 CPU 指令执行。当我们对一个相关块进行符号执行时，它能够正确找到下一个相关块。

符号执行先从函数序言处开始，我们将所有符号执行到达的相关块保存至一个队列里，将已经进行过符号执行的相关块从栈中弹出，然后在队列中取出新的相关块进行符号执行；同时我们在符号执行到一个新的相关块时需要保存当前 CPU 执行环境 (内存状态，寄存器状态)，以确保下次符号执行新的相关块时，所有的 CPU 执行环境都正确。

对于有分支结构的相关块，我们采用特征反汇编来识别：  
![](https://img-blog.csdnimg.cn/dc871343043144c2a7607f7a04eac242.png?)  
可以看到，在有分支的相关块中，是通过 IT(if then) 指令来实现不同条件分支的。IT 指令中 T 的个数代表了分支指令数量（比如 ITTT EQ 成立则会执行执行该指令后三条指令，否则会跳过这三条指令），在这里，寄存器 R1 作为状态值，在相关块中进行更新，然后返回主分发器，通过更新后的 R1 的值再找到下一个相关块。  
为了实现两个分支流程，我们需要自行改变执行流程，而 angr 的 hook 功能正好为我们实现了这一点，我们对 IT 指令的位置进行 hook，通过设置跳过的地址长度来实现分支流程。

符号执行代码如下：

```
#队列用于保存相关块的符号执行环境
    flow = defaultdict(list)
    queue = queue.Queue()
    addrinqueue=[]
    #从函数开始处符号执行
    queue.put((startaddr+1,None,0))

    while not queue.empty():
        env=queue.get()
        pc=env.addr
        #到达函数返回块或是相关块已经执行过，则跳过该次执行
        if pc in addrinqueue or pc==retaddr:
            continue
        state=env
           block=project.factory.block(pc,relevants[pc].size)
        has_branches=False
        bl_addr=[]
        it_addr=[]
        insns=block.capstone.insns

        for i in range(len(insns)):
            insn=insns[i].insn
            #判断相关块中是否有bl指令，有就将bl指令地址保存，后面符号执行时直接hook跳过
            if insn.mnemonic.startswith('bl'):
                bl_addr.append((insn.address,insn.size))
            if i==len(insns)-1:
                continue
            #判断相关块中是否存在分支结构，有就将IT指令地址保存，符号执行时通过hook到达不同分支
            if insn.mnemonic.startswith('it') and insns[i+1].insn.op_str.startswith(regstr):
                if pc in patch_info:
                    continue
                has_branches = True
                patch_addr_info=[]
                patch_addr_info.append(insn.address)
                j=insn.mnemonic.count('t')
                it_addr.append((insn.address,insns[i+j+1].insn.address-insn.address))
                it_addr.append((insn.address,insns[i+1].insn.address-insn.address))
                movinsns=None
                if insns[-1].insn.mnemonic.startswith('b'):
                    movinsns=insns[i+j+1:-1]
                else:
                    movinsns=insns[i+j+1:]
                movcodes=bytearray()
                #如果IT指令之后有改变ZF状态操作，则将改变ZF状态的功能去除，ZF状态改变会影响分支的执行
                for insnx in movinsns:
                    if insnx.insn.mnemonic.startswith('movs'):
                        encoding=ks_disasm('{} {}'.format('mov',insnx.insn.op_str))
                        movcodes.extend(encoding)
                    else: movcodes.extend(insnx.insn.bytes)
                patch_info[pc]=(patch_addr_info,insn.op_str,movcodes)

        if has_branches:
              #有分支结构，对两个分支都进行符号执行
            symbolic_execution(pc,state, bl_addr, it_addr[0])
            symbolic_execution(pc,state, bl_addr,it_addr[1])
        else:
            symbolic_execution(pc,state,bl_addr)
def symbolic_execution(start_addr, state, bl_addrs=None, branch_addr=None):
    global real_to_real_nodes
    global regs_init_info,queue,flow,addrinqueue
    def handle_bl(state):
        pass

    def handle_branch(state):
        pass

    def init_regs(state,regs_info):
        if len(regs_info)==0:
            return
        for regstr,regvalue in regs_info.items():
            if regstr=='r0': state.regs.r0=claripy.BVV(regvalue,32)
            elif regstr=='r1': state.regs.r1=claripy.BVV(regvalue,32)
            elif regstr=='r2': state.regs.r2=claripy.BVV(regvalue,32)
            elif regstr=='r3': state.regs.r3=claripy.BVV(regvalue,32)
            elif regstr=='r4': state.regs.r4=claripy.BVV(regvalue,32)
            elif regstr=='r5': state.regs.r5=claripy.BVV(regvalue,32)
            elif regstr=='r6': state.regs.r6=claripy.BVV(regvalue,32)
            elif regstr=='r7': state.regs.r7=claripy.BVV(regvalue,32)
            elif regstr=='r8': state.regs.r8=claripy.BVV(regvalue,32)
            elif regstr=='r9': state.regs.r9=claripy.BVV(regvalue,32)
            elif regstr=='r10': state.regs.r10=claripy.BVV(regvalue,32)
            elif regstr=='r11': state.regs.r11=claripy.BVV(regvalue,32)
            elif regstr=='r12': state.regs.r12=claripy.BVV(regvalue,32)
            elif regstr=='sp': state.regs.sp=claripy.BVV(regvalue,32)
            elif regstr=='lr': state.regs.lr=claripy.BVV(regvalue,32)
            elif regstr=='pc': state.regs.pc=claripy.BVV(regvalue,32)

    global project, relevant_block_addrs, modify_value
    if bl_addrs!=None:
        for addr in bl_addrs:
            #hook bl指令 跳过函数调用
            project.hook(addr[0], handle_bl, addr[1])
    if branch_addr!=None:
          #hook it指令 实现不同分支执行
        project.hook(branch_addr[0],handle_branch,branch_addr[1],replace=True)
    if state==None:
        #初始化,用于第一次符号执行时
        state = project.factory.blank_state(addr=start_addr, remove_options={angr.sim_options.LAZY_SOLVES},
                                        add_option={angr.options.SYMBOLIC_WRITE_ADDRESSES})
        #初始化寄存器
        init_regs(state,regs_init_info)
    sm = project.factory.simulation_manager(state)
    loopTime=0
    maxLoopTime=1
    skip_addr=None
    #如果是编译优化导致进行符号执行的相关块改变索引值后未跳转至主分发器，则跳过该相关块跳转到的下一个相关块
    if start_addr in special_relevant_nodes:
        skip_addr=special_relevant_nodes[start_addr]
        maxLoopTime+=1

    sm.step()
    while len(sm.active) > 0:
        for active_state in sm.active:
            #多次循环至主分发器而不经过任何相关块，可认定该路径为死循环
            if active_state.addr==main_dispatcher_node.addr:
                if loopTime<maxLoopTime:
                    loopTime+=1
                else:
                    return None
            if active_state.addr==start_addr:
                return None
            if active_state.addr==retaddr:
                return (active_state.addr, active_state, start_addr)
            #判断是否是相关块
            if active_state.addr in relevant_block_addrs and active_state.addr != skip_addr:
                #如果是相关块，将该块的地址符号执行状态保存放入队列里面
                queue.put((active_state))
                #将符号执行的相关块和其后续相关块对应保存在字典里
                flow[start_addr].append(active_state.addr)
                #保存已经符号执行过的相关块，以免重复执行浪费时间
                addrinqueue.append(start_addr)
        sm.step()
```

其中 project.hook 用于直接跳过一些指令：

project.hook(self, addr, hook=None, length=0, kwargs=None, replace=False)  
addr 参数为 hook 的地址，length 参数用于设置跳过的地址长度，hook 参数可以设置一个在执行到这条指令时的回调方法。

sm.step 会执行到下一个基本块的位置，这时判断如果该块是相关块的话，就停止符号执行，将该基本块的地址和当前的符号执行环境保存至之前所说的符号执行队列里，用于下一次对该块的符号执行。

这样等队列里所有块都符号执行完毕后，我们就理清了相关块之间的关系，下面一步，就是需要通过修改指令来建立相关块之间的跳转。

patch 指令建立相关块间的跳转
-----------------

通过 B 系列指令来构建跳转，因为大部分相关块最后一条指令都是跳转回主分发器，对于还原混淆来说是无用的，所以我们选择在这里进行 patch，将该指令替换成到下一个相关块的指令。如果是有分支结构的相关块，则需要 patch 两条跳转指令，这时哪里有空间存放另一条跳转指令呢？有两种方案可以解决：

1、相关块里最后用于改变索引值的指令都是无用的，所以我们可以将 IT 指令及其后面的分支指令去除，再将后面的指令移上去，这样就可以腾出空间放入另一条跳转指令。注意分支跳转指令的条件要和原先 IT 指令的条件保持一致。

patch 前：  
![](https://img-blog.csdnimg.cn/58e9fcbdc6564363bc82401a874ebfee.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAYWN5MDA3,size_20,color_FFFFFF,t_70,g_se,x_16)  
patch 后：  
![](https://img-blog.csdnimg.cn/124bcddda31843d482ce3d59999658e6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAYWN5MDA3,size_20,color_FFFFFF,t_70,g_se,x_16)  
2、如果去除 IT 相关指令后空间依然不够，第二种方法则是构建一个跳板，先跳转到一块无用代码区域 (足够放下我们的跳转指令)，再从这块区域跳转到后面的相关块。无用代码区域从哪里找呢？可以从我们之前寻找相关块时过滤掉的代码块中获取，在过滤的时候将这些无用块的地址和大小保存起来，当需要构建跳板时再从中找到符合条件的代码区域。  
![](https://img-blog.csdnimg.cn/85184a8ea1ae49a99e2fe99603131d77.png?)  
patch 过程中其他需要注意的地方因为该函数是 Thumb-2 代码，长度可以为 2 字节或 4 字节，如果原本到主分发器的跳转是 2 字节，而新的跳转范围如果过大则可能是 4 字节，所以在 patch 前都要先判断下预留空间是否足够，如果不够的话，再通过上述一二两种方法进行处理。

最终效果查看
------

我们对 JNI_OnLoad 函数进行完上述处理，可以看到平坦化结构已经被消除  
ida CFG：

![](https://img-blog.csdnimg.cn/4e41804666734641a26c2ad18afca26b.png?)

ida f5 的效果：  
![](https://img-blog.csdnimg.cn/0cfc72abae904c0aad4e2dc2dd3ef5da.png?)  
![](https://img-blog.csdnimg.cn/8d493513d5054820a514ef32bb776699.png?)

算法还原
----

刚对 Jni_Onload 的最外层进行了反混淆，f5 之后可以看到，主要调用了 sub_10710 和 sub_23B0 两个函数，跟进 sub_10710, 并没有发现对 vm 的引用，而在 sub_23B0 中引用了 vm，所以先分析 sub_23B0。

![](https://img-blog.csdnimg.cn/c190136503b14fe2a98722e19e414288.png?)

跟进后该函里面是一段简单的混淆数，最终其实是跳转到了 sub_23C6 处的函数。

sub_23C6 中存在这上一篇遇到的常规混淆，利用脚本将其清除之后，可以分析出该函数的作用是跳转到调用 sub_23B0 的第一个参数的地址，同时将 javaVM 指针的地址作为第一个参数传入到 sub_23C6, 通过 ida 动态调试我们得出 sub_23B0 的第一个参数为 0x2520，所以我们继续，跟进到 sub_0x2520。  
![](https://img-blog.csdnimg.cn/094091a0da0a4f82a38139c3c766e5e5.png?)

观察该函数 cfg 和内部特征，可以得出该函数又是经过 ollvm 平坦化处理，利用之前的去混淆脚本配置好之后进行处理：

![](https://img-blog.csdnimg.cn/12ce03a722ec4e1da7951c5e917f530c.png?)

部分 f5 之后的 c 代码：  
![](https://img-blog.csdnimg.cn/4cd48a3667244d41a4a95b0715f651fb.png?)

并没有直接找到后面引用到 javaVM 指针的代码，但是上图 2 处看结构很可能是调用 vm->getEnv，为了进一步优化 f5 代码和变量之间关系，我们需要进一步处理。可以注意到其中一些分支其实是不会执行的，如上图中 1 处分支的条件，从数学角度分析该条件不会成立，但 ida 并不能识别出来，所以我们在符号执行的时候需要特殊处理，去掉这些不会执行的相关块。

第二次优化处理后：  
![](https://img-blog.csdnimg.cn/de56c1047ad24edab9c75d4fe0231446.png?)

![](https://img-blog.csdnimg.cn/465c904b72304429bca3aa94860a2d96.png?)

这次可以看到 JavaVM 指针被直接引用到，函数执行的流程也基本上一览无遗，该函数里主要对一些全局变量进行解密，解密成用于注册 jni 方法的字符串或是其他用途的重要数据。

![](https://img-blog.csdnimg.cn/608cf6d07935497796462cd8a289d07e.png?)

我们找到 env->RegisterNatives 处，然后通过参数获取到 leviathan 方法的对应的 native 函数地址位于 0x576dc 处。

0x576dc 同样是和之前一样，存在常规跳转混淆和平坦化混淆，反混淆脚本伺候之：  
![](https://img-blog.csdnimg.cn/96c7008cd8e24028a21cc7de9dc2932c.png?)  
![](https://img-blog.csdnimg.cn/c8bfc2583bae4ae5a32e982a7d618cad.png?)  
最后又是调用了 sub_23B0 这个函数，在前面已经分析过该函数的作用，可以看出，并没有直接将 env 等一些 jni 参数直接传入该函数，而是构建了一个可能是数组的结构，将这些参数加上一定的偏移放入该结构，再将该数组地址作为第二个参数传入。  
第一个参数表达式其实是一个定值，其值是 sub_551c0 的地址，不会受到表达式中变量 a4 影响（大家有兴趣可以验证下）。

进入到 sub_551c0，又是 ollvm + 跳转混淆，老样子，脚本跑一跑。  
f5 之后：

![](https://img-blog.csdnimg.cn/e1b41d94d97b4b71a515a09bbdfb220a.png?)

![](https://img-blog.csdnimg.cn/ecbf3d7919614f81b898cc604ab723de.png?)

可以看到在上图处取出了 jni 方法的参数，紧接着，对 java 层传入的数组和参数 i2 进行了一些简单的字节变换操作，将变换后的的数组，数组大小以及 i1 的值作为一个新的结构传入到 sub_215BC。从后面调用的 jni 函数来看，sub_215BC 的返回值极可能是最终加密之后的 byte 数组，所以我们继续跟进去：

![](https://img-blog.csdnimg.cn/2d6f947a6d8f44a6a7bddf7c52816343.png?)

这里我对参数结构进行了一些转换，以便于更好的理解 c 代码，可以发现参数结构体作为一个数组的成员继续传入到下一层，继续跟进到 sub_229C8：

该函数就是加密算法的核心位置了，但乍一看混淆方式貌似和前面的方式不太一样，该函数是纯 ARM 指令，也就用不了之前的脚本，而且混淆方式也不太像是 ollvm，仔细观察这个函数可以发现里面有大量的 switch-case 跳转表结构，但 ida 没有识别出来，我们可以自己设置添加跳转表结构：

![](https://img-blog.csdnimg.cn/e4859fcd2abf4917a86c367a7ea6e1a0.png?)

如上图 r6 是索引值，r1 是跳转表基址，r3 是跳转表索引到的值，最后跳转的位置即 r1+r3。在 ida 依次选择 Edit->Other->Specify switch idiom，为此处配置一个跳转表结构：  
![](https://img-blog.csdnimg.cn/0ebfa87efb5c4c0d8d087e98ef8205cf.png?)

找到所有的跳转表后，便可以 f5 查看 c 代码：

![](https://img-blog.csdnimg.cn/4d1d53f197e34b018e786a3004313bfa.png?)

![](https://img-blog.csdnimg.cn/71d638e31007462aab78a81b941fbfff.png?)

![](https://img-blog.csdnimg.cn/22b2d1ff5915458db67ec32dcd1e8d8f.png?)

基本上无法解读，但大体上可以知道其结构，while 循环内的 switch-case 结构，但多数 case 内又嵌套着另一层 switch-case 结构，形成多层嵌套，所以代码看起来比较复杂。而最外层 switch 的 case 索引，是通过读取从 dword_85B60 起始的一串数据表，再通过转换运算得到。该数据表内的数据相当于一条命令，不同的命令可以执行不同的 case 组合，完成不同的功能。

用 C 代码实现流程
----------

还原该算法难度较大，主要是 while 内 switch case 循环次数很多，经验证，总共有一万多次循环，如果对每次循环都进行分析的话，工作量可想而知。所以这里可以另辟蹊径，可以试着新建一个 c 工程，将 f5 的伪代码复制出来，同时再构建一个与当前程序执行环境相同的内存环境，直接脱机运行：

```
FILE* fp = fopen("libcms-dump", "rb");
    if (!fp)return-1;
    fseek(fp, 0L, SEEK_END);
    int size = ftell(fp);
    //将dump下来的so放入到申请的内存里
    pBuf = new char[size] {0};
    fseek(fp, 0L, SEEK_SET);
    fread(pBuf, size, 1, fp);
    fclose(fp);
    //对so进行数据重定位修复，ori_addr是dump so时，so的加载基址
    relocation(pBuf, ori_addr);
    const char* data="b6a274acedea791afce92a344ccdd80d00000000000000000000000000000000063745505e61c692b79747ec710f8a3100000000000000000000000000000000";
    int ts = 1583457688;
    unsigned char* pBuf3 = (unsigned char*)malloc(64);
    HexStrTobytes((char*)data, pBuf3);
    int swapts = _byteswap_ulong(ts);
    char* pBuf2 = (char*)malloc(20);
    sub_112D8((uint8*)pBuf2, pBuf3, 4);
    sub_112D8((uint8*)pBuf2+4, pBuf3+16, 4);
    sub_112D8((uint8*)pBuf2+8, pBuf3+32, 4);
    sub_112D8((uint8*)pBuf2+12, pBuf3+48, 4);
    sub_112D8((uint8*)pBuf2+16, (unsigned char*)&swapts, 4);
    sub_112D8((uint8*)pBuf2 + 12, (unsigned char*)pBuf+0x8f09c, 4);
    MYINPUT input2 = { pBuf2,20,-1 };
    unsigned int a2[354] = { 0 };
    a2[0] = (unsigned int)&input2;
    a2[2] = (unsigned int)&pBuf[0x21ef5];
    a2[3] = (unsigned int)&a2[350];
    a2[352] = pBuf[0x90690];
    MYINPUT* pInput = &input;
    sub_229C8((unsigned int*)&pBuf[0x85b60], a2, (unsigned int*)&pBuf[0x8eb70], (unsigned int*)&pBuf[0x8eba0], &a2[2]);
```

因为该代码内用到了很多的全局变量，很多变量都是在 app 运行后才开始解密，我们也没有对那些代码进行分析，所以需要将 app 运行到 sub_229c8 时的整个 libcms.so 的内存 dump 出来，可以通过 xposed+cydia 或是 frida hook sub_229c8 来实现 dump，同时记录下其加载基址，因为 android 平台 pic（位置无关代码）编译的原因，所有全局变量的引用都是通过 got(全局偏移表) 完成的，加载器会根据加载基址来修正，并向 got 填入正确的全局变量的地址。当我们自己实现该函数功能，申请一段内存 pBuf 来存放 so 数据，把 got 内全局变量的地址修正到 pBuf 的位置，如某重定位数据 a=S，app 运行时的基址是 A，pBuf 的地址是 B，则重定位 a 的值为 S-A+B，这样便相当于从 pBuf 处加载 so。

通过 readelf -D 获取数据重定位信息：  
![](https://img-blog.csdnimg.cn/5e1a830002014aa7b2cb3b5c0442c261.png?)  
对数据进行重定位：

```
void relocation(char* bytes,uint32 ori_addr)
{
    uint32 new_addr = (uint32)bytes;
    unsigned int reldyn_start = 0xbac + new_addr;
    size_t reldyn_size = 5496;
    Elf32_Rel* pRel = (Elf32_Rel*)reldyn_start;

    //relocation for .rel.dyn
    for (int i = 0; i < reldyn_size / sizeof(Elf32_Rel); ++i) 
    {
        uint8 relType = (pRel->r_info)&0xff;
        if (relType == R_ARM_RELATIVE || relType == R_ARM_GLOB_DAT)
        {
            *(uint32*)(bytes + pRel->r_offset) = *(uint32*)(bytes + pRel->r_offset) -ori_addr + new_addr;
        }
        pRel += 1;
    }
}
```

在 sub_229c8 内有多处函数调用，同样需要把这样函数复制出来实现，需要注意的时这个地方：  
![](https://img-blog.csdnimg.cn/370006f9d4674a82843578757d0121f2.png?)

1147 行是一个函数调用，v102 的值分析后得出是 sub_21ef4 地址，其功能也很简单：

![](https://img-blog.csdnimg.cn/b7aa04d05e0e48fda1309f52a0553bfd.png)  
考虑到 a1 可能有多个不同值，所以通过 hook sub_21ef4，来获取所有 app 运行用到的 a1 的值，之后找到所有 a1 指向的函数并复制出来实现：

![](https://img-blog.csdnimg.cn/6335a7d0f27c4b048ac946416cdc4a98.png?)  
这样我们直接运行代码：![](https://img-blog.csdnimg.cn/ab464e6a292e4a32942b8fd9d1af43fe.png?)  
成功获取返回值，为了验证正确性，将我们程序得到的结果放入到请求参数中，可以正常返回数据！！！