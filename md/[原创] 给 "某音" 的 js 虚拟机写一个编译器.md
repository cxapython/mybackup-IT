> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-261414.htm)

> [原创] 给 "某音" 的 js 虚拟机写一个编译器

0x0 前言
------

其实这篇笔记很久之前就写了, 写的零零碎碎的, 最近翻硬盘的时候看到了, 就想着整理一下发出来, 和大家讨论一下, 其中有些逻辑可能自己都不太清楚了 (毕竟写代码不注释, 一周不看代码就不属于自己了, 手动狗头), 描述的有问题欢迎指出哈;

0x1 js 虚拟机逆向
------------

先看看原始文件是什么样的 (一部分 opcode 片段), 这时候看还是比较乱的...

```
.........
case 71:
    v[x++] = n;
    break;
case 72:
    v[x++] = +f();
    break;
case 73:
    u(parseInt(f(), 36));
    break;
case 75:
    if (v[--x]) {
        b++;
        break
    }
case 74:
    g = t.charCodeAt(b++) - 32 << 16 >> 16,
    b += g;
    break;
case 76:
    u(k[t.charCodeAt(b++) - 32]);
    break;
case 77:
    y = v[--x],
    u(v[--x][y]);
    break;
case 78:
    g = t.charCodeAt(b++) - 32,
    u(a(v, x -= g + 1, g));
    break;
case 79:
    g = t.charCodeAt(b++) - 32,
    u(k["$" + g]);
    break;
.........
```

如何逆向这部分我这里不细说了, 单步调试的话还是很容易看出每个操作的意义的, 经过一步步单步分析后, 逻辑就清晰了很多, 可以看到不少压栈出栈的操作, 是一个基于堆栈的栈式虚拟机, 利用数组模拟堆栈, 且有作用域管理 (完整的代码我会放在附件)

```
  .........
case 71:
    vm_stack[vm_esp++] = pthis;
    console.log('PUSH pthis {obj:%s type:%s}', my_tostring((vm_stack[vm_esp-1])), typeof((vm_stack[vm_esp-1])));
    break;
case 72:
    vm_stack[vm_esp++] = +vm_substring();
    console.log('PUSH STR {obj:%s type:%s}', my_tostring((vm_stack[vm_esp-1])), typeof((vm_stack[vm_esp-1])))
    break;
case 73:
    vm_push(parseInt(vm_substring(), 36));
    console.log('PUSH INT {obj:%s type:%s}', my_tostring((vm_stack[vm_esp-1])), typeof((vm_stack[vm_esp-1])))
    break;
case 75: //判断条件是否成立
    if (vm_stack[--vm_esp]) {
        index++;
        break
    }
case 74: //不成立则跳过, 或用于循环代码块
    g = codes.charCodeAt(index++) - 32
    g = g << 16 >> 16
    console.log('IDX+=%s', my_tostring(g))
    index += g;
    break;
case 76:
    var vars_idx = codes.charCodeAt(index) - 32;
    //vars_idx = vars_idx);
    vm_push(vm_vars[codes.charCodeAt(index++) - 32]);
    console.log('PUSH Vars[%s] {obj:%s type:%s}',my_tostring( vars_idx), my_tostring(vm_vars[vars_idx]), typeof((vm_vars[vars_idx])))
    break;
case 77:
    y = vm_stack[--vm_esp],
    vm_push(vm_stack[--vm_esp][y]);
    console.log('PUSH_OBJECT {obj:%s type:%s}', my_tostring((vm_stack[vm_esp-1])), typeof((vm_stack[vm_esp-1])))
    break;
case 78:
    g = codes.charCodeAt(index++) - 32
    vm_push(vmNewObject(vm_stack, vm_esp -= g + 1, g));
    console.log('NEW_OBJECT {obj:%s type:%s}', my_tostring((vm_stack[vm_esp-1])), typeof((vm_stack[vm_esp-1])))
    break;
case 79:
    g = codes.charCodeAt(index++) - 32;
    vm_push(vm_vars["$" + g]);
    console.log('PUSH Vars_2[%s]', my_tostring(g));
    break;
  .........
```

0x2 如何写一个编译器?
-------------

esprima 和 escodegen 的仓库

> [esprima](https://github.com/jquery/esprima)
> 
> [escodegen](https://github.com/estools/escodegen)

 

既然已经知道每个 opcode 的作用, 那么如何针对这些 opcode 去写一个编译器呢? emmm... 我这里选择了一个比较 low 的方式, 使用`esprima`把 js 代码解析成 ast, 再使用`escodegen`解析 ast 的过程中 (修改 escodegen.js), 根据不同的 type 来生成自己的 opcode 的代码, 最后再写入文件...

 

比如在 IfStatement 不同的位置插入接口`encrypt_code`  
![](https://bbs.pediy.com/upload/attach/202008/773600_BFAY2UYAJ62NEWH.jpg)

 

在`encrypt_code`根据不同的位置来生成 opcode  
![](https://bbs.pediy.com/upload/attach/202008/773600_U96G4KBERFVWXPP.jpg)

 

这里我举 ifelse 分支的例子, 看看如何把 js 代码编译成该虚拟机可以执行的 opcode 的, 以及 opcode 的格式

### if elseif else

测试代码:

```
var a = 11;
if (a < 10) {
    a = 10;
} else if (a == 10) {
    a = 11;
} else {
    a = 12;
}
```

使用编译器跑一遍, 在`encrypt_code`入口打印下的类型, 看一下经过编译器了哪些位置, 以及对应的 js 代码

```
Expression            //var a = 11;
IfStatement           //if(a < 10)
EnmaybeBlock
EnBlockStatement      //{     
ExpressionStatement   //a = 10
LvBlockStatement      //}  
LvmaybeBlock
IfStatement           //else if (a == 10)
EnmaybeBlock   
EnBlockStatement      //{     
ExpressionStatement   //a = 11
LvBlockStatement      //}
LvmaybeBlock
IfStatement_ELSEBEGIN //else
EnBlockStatement      //{
ExpressionStatement   //a = 12
LvBlockStatement      //}
IfStatement_ELSEEND
```

这里可以看到每种类型对应的 js 代码, 接下来就是根据不同的类型生成 opcode, 生成过程用文字描述会比较冗余, 这里我直接把生成的 opcode 结构贴上来, 就能很直观的理解了

 

以下为生成的 opcode (虚拟机解析 opcode 的时候会先减去 - 32, 这里的已经处理了):

```
14,11,83,15,76,15,14,10,66,60,75,6,14,10,83,15,74,20,76,15,14,10,68,2,29,29,75,6,14,11,83,15,74,4,14,12,83,15
```

直接看又是很蒙是不是, 没事, 我们拆开和 js 代码比对来看就很清晰了

```
var a = 11;  >>  14,11,83,15
```

<table><thead><tr><th>opcode</th><th>dest</th></tr></thead><tbody><tr><td>PUSH(14)</td><td>(Value)11 // 把 11 压栈</td></tr><tr><td>MOV_VARS(83)</td><td>(Index)15 // 把栈顶的值放到变量区 [15]</td></tr></tbody></table>

```
if(a < 10)  >> 76,15,14,10,66,60,75,6
```

<table><thead><tr><th>opcode</th><th>dest</th></tr></thead><tbody><tr><td>PUSH_VAR(76)</td><td>(Value)15 // 把变量区 [15] 的值压栈</td></tr><tr><td>PUSH(14)</td><td>(Value)10 // 把 10 压栈</td></tr><tr><td>EXPRESSION(66)</td><td>(Symbol '&lt;') 60 // 把栈顶的两个值做运算, 结果放在栈顶</td></tr><tr><td>IS_TURE(75)</td><td>(Value)6 // 判断栈顶的值是否为 True, 如果为 False 则把当前 opcode_index+Value</td></tr></tbody></table>

```
{ a = 10 } >>  14,10,83,15,74,20
```

<table><thead><tr><th>opcode</th><th>dest</th></tr></thead><tbody><tr><td>PUSH(14)</td><td>(Value)10 // 把 10 压栈</td></tr><tr><td>MOV_VARS(83)</td><td>(Index)15 // 把栈顶的值放到变量区 [15]</td></tr><tr><td>SKIP_BLOCK(74)</td><td>(Value)20 // 跳出 Block, 当前 opcode_index+Value</td></tr></tbody></table>

```
else if (a == 10)  >>  76,15,14,10,68,2,29,29,75,6
```

<table><thead><tr><th>opcode</th><th>dest1</th><th>dest2</th></tr></thead><tbody><tr><td>PUSH_VAR(76)</td><td>(Value)15 // 把变量区 [15] 的值压栈</td><td></td></tr><tr><td>PUSH(14)</td><td>(Value)10 // 把 10 压栈</td><td></td></tr><tr><td>EXPRESSION2(68)</td><td>(Value)2 // 符号数</td><td>(Symbol '==') 2929 // 把栈顶的两个值做运算, 结果放在栈顶</td></tr><tr><td>IS_TURE(75)</td><td>(Value)6 // 判断栈顶的值是否为 True, 如果为 False 则把当前 opcode_index+Value</td></tr></tbody></table>

```
{ a = 11 }  >>  14,11,83,15,74,4
```

<table><thead><tr><th>opcode</th><th>dest</th></tr></thead><tbody><tr><td>PUSH(14)</td><td>(Value)11 // 把 11 压栈</td></tr><tr><td>MOV_VARS(83)</td><td>(Index)15 // 把栈顶的值放到变量区 [15]</td></tr><tr><td>SKIP_BLOCK(74)</td><td>(Value)4 // 跳出 Block, 当前 opcode_index+Value</td></tr></tbody></table>

```
{ a = 12 }  >>  14,12,83,15
```

<table><thead><tr><th>opcode</th><th>dest</th></tr></thead><tbody><tr><td>PUSH(14)</td><td>(Value)12 // 把 12 压栈</td></tr><tr><td>MOV_VARS(83)</td><td>(Index)15 // 把栈顶的值放到变量区 [15]</td></tr></tbody></table>

 

到这里以上代码整个 opcode 的解析过程就已经结束了, 在贴一下虚拟机执行的过程吧, 可以清晰的看到分支最后走到了`a = 12`

```
vmEnter
vmRun
opcode = 14
PUSH Vlaue{ obj:11 type:number}
opcode = 83
MOV Vars[15], v {v:11 type:number}
opcode = 76
PUSH Vars[15] {obj:11 type:number}
opcode = 14
PUSH Vlaue{ obj:10 type:number}
opcode = 66
    ==Expression: 11 < 10
PUSH expr {obj:false type:boolean}
opcode = 75
IDX+=6
opcode = 76
PUSH Vars[15] {obj:11 type:number}
opcode = 14
PUSH Vlaue{ obj:10 type:number}
opcode = 68
    ==Expression: 11 == 10
PUSH expr_2 {obj:false type:boolean}
opcode = 75
IDX+=6
opcode = 14
PUSH Vlaue{ obj:12 type:number}
opcode = 83
MOV Vars[15], v {v:12 type:number}
```

0x3 具体例子
--------

上面描述了如何实现一个简单的逻辑运算, 分支派发的实现方式, 根据上述思路, 目前实现逻辑运算, IFELSE, 函数调用, 循环, 数组等等基本 js 语法, 我也找了 md5, crc 等一些算法测试, 改了下都可以正常跑

### MD5

源码执行  
![](https://bbs.pediy.com/upload/attach/202008/773600_FAJY9T3D57ZZUT6.jpg)

 

编译后 vm 执行  
![](https://bbs.pediy.com/upload/attach/202008/773600_UX55VFUFMHAQ7MW.jpg)

### CRC

源码执行  
![](https://bbs.pediy.com/upload/attach/202008/773600_DDMQ2GE4TV4M6T8.jpg)

 

编译后 vm 执行  
![](https://bbs.pediy.com/upload/attach/202008/773600_6Y4WS4NKGXS7TZX.jpg)

0x4 源码
------

[jsvm](https://github.com/StriveMario/jsvm)

 

用法其实也很简单, 安装完需要的模块后, 把修改后的`escodegen.js`替换`node_modules\escodegen`中的, 修改`encrypt.js`中需要编译的 js 文件路径再执行, 编译后会在`./build/vm.js`

0x5 最后
------

最后想说这个东西只是个玩具, 发出来是想和大家交流一下, 文章中因为篇幅的原因没有描述太多细节, 大家想了解更多的细节可以直接去看看源码 (写的很随性, 别吐槽...), 还有就是想吐槽下这种实现方式调试很痛苦, 中间很多一部分直接都花在找 bug 上, 大佬们有好想法可以不吝赐教哈...

[[培训] 优秀毕业生寄语：恭喜 id 咸鱼炒白菜拿到远超 3W 月薪的 offer，《安卓高级研修班》火热招生！！！](https://zhuanlan.kanxue.com/article-16096.htm)

最后于 2020-8-12 18:59 被 StriveMario 编辑 ，原因：