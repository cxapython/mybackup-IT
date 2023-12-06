> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.anquanke.com](https://www.anquanke.com/post/id/86422)

> 安全客 - 安全资讯平台

**![](https://p2.ssl.qhimg.com/t014c808ebf4ff98ac2.png)**

译者：[arnow117](http://bobao.360.cn/member/contribute?uid=941579989)

预估稿费：160RMB

投稿方式：发送邮件至 linwei#360.cn，或登陆网页版在线投稿

传送门

[【系列分享】ARM 汇编基础速成 1：ARM 汇编以及汇编语言基础介绍](http://bobao.360.cn/learning/detail/4070.html)

[【系列分享】ARM 汇编基础速成 2：ARM 汇编中的数据类型](http://bobao.360.cn/learning/detail/4075.html)

[【系列分享】ARM 汇编基础速成 3：ARM 模式与 THUMB 模式](http://bobao.360.cn/learning/detail/4082.html)

**[【系列分享】ARM 汇编基础速成 4：ARM 汇编内存访问相关指令](http://bobao.360.cn/learning/detail/4087.html)**

[**【系列分享】ARM 汇编基础速成 5：连续存取**](http://bobao.360.cn/learning/detail/4097.html)

**条件执行**

在之前讨论 CPSR 寄存器那部分时，我们大概提了一下条件执行这个词。条件执行用来控制程序执行跳转，或者满足条件下的特定指令的执行。相关条件在 CPSR 寄存器中描述。寄存器中的比特位的变化决定着不同的条件。比如说当我们比较两个数是否相同时，我们使用的 Zero 比特位 (Z=1)，因为这种情况下发生的运算是 a-b=0。在这种情况下我们就满足了 EQual 的条件。如果第一个数更大些，我们就满足了更大的条件 Grater Than 或者相反的较小 Lower Than。条件缩写都是英文首字母缩写，比如小于等于 Lower Than(LE)，大于等于 Greater Equal(GE) 等。

下面列表是各个条件的含义以及其检测的状态位 (条件指令都是其英文含义的缩写，为了便于记忆不翻译了)：

![](https://p1.ssl.qhimg.com/t01614edecda79f64b7.png)

我们使用如下代码来实践条件执行相加指令：

```
.global main
main:
        mov     r0, #2     /* 初始化值 */
        cmp     r0, #3     /* 将R0和3相比做差，负数产生则N位置1 */
        addlt   r0, r0, #1 /* 如果小于等于3，则R0加一 */
        cmp     r0, #3     /* 将R0和3相比做差，零结果产生则Z位置一，N位置恢复为0 */
        addlt   r0, r0, #1 /* 如果小于等于3，则R0加一R0 IF it was determined that it is smaller (lower than) number 3 */
        bx      lr
```

上面代码段中的第一条 CMP 指令将 N 位置一同时也就指明了 R0 比 3 小。之后 ADDLT 指令在 LT 条件下执行，对应到 CPSR 寄存器的情况时 V 与 N 比特位不能相同。在执行第二条 CMP 前，R0=3。所以第二条置了 Z 位而消除了 N 位。所以 ADDLT 不会执行 R0 也不会被修改，最终程序结果是 3。

**Thumb 模式中的条件执行**

在指令集那篇文章中我们谈到了不同的指令集，对于 Thumb 中，其实也有条件执的 (Thumb-2 中有)。有些 ARM 处理器版本支持 IT 指令，允许在 Thumb 模式下条件执行最多四条指令。

相关引用：[http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0552a/BABIJDIC.html](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0552a/BABIJDIC.html) 

指令格式：Syntax: IT{x{y{z}}} cond

cond 代表在 IT 指令后第一条条件执行执行指令的需要满足的条件。

x 代表着第二条条件执行指令要满足的条件逻辑相同还是相反。

y 代表着第三条条件执行指令要满足的条件逻辑相同还是相反。

z 代表着第四条条件执行指令要满足的条件逻辑相同还是相反。

IT 指令的含义是 “IF-Then-(Else)”，跟这个形式类似的还有：

IT，If-Then，接下来的一条指令条件执行。

ITT，If-Then-Then，接下来的两条指令条件执行。

ITE，If-Then-Else，接下来的两条指令条件执行。

ITTE，If-Then-Then-Else，接下来的三条指令条件执行。

ITTEE，If-Then-Then-Else-Else，接下来的四条指令条件执行。

在 IT 块中的每一条条件执行指令必须是相同逻辑条件或者相反逻辑条件。比如说 ITE 指令，第一条和第二条指令必须使用相同的条件，而第三条必须是与前两条逻辑上相反的条件。这有一些 ARM reference 上的例子：

```
ITTE   NE           ; 后三条指令条件执行
ANDNE  R0, R0, R1   ; ANDNE不更新条件执行相关flags
ADDSNE R2, R2, #1   ; ADDSNE更新条件执行相关flags
MOVEQ  R2, R3       ; 条件执行的move
ITE    GT           ; 后两条指令条件执行
ADDGT  R1, R0, #55  ; GT条件满足时执行加
ADDLE  R1, R0, #48  ; GT条件不满足时执行加
ITTEE  EQ           ; 后两条指令条件执行
MOVEQ  R0, R1       ; 条件执行MOV
ADDEQ  R2, R2, #10  ; 条件执行ADD
ANDNE  R3, R3, #1   ; 条件执行AND
BNE.W  dloop        ; 分支指令只能在IT块的最后一条指令中使用
```

错误的格式：

```
IT     NE           ; 下一条指令条件执行
ADD    R0, R0, R1   ; 格式错误：没有条件指令
```

下图是条件指令后缀含义以及他们的逻辑相反指令：

![](https://p0.ssl.qhimg.com/t0159a38754c0e1e375.png)

让我们试试下面这段代码：

```
.syntax unified    @ 这很重要！
.text
.global _start
_start:
    .code 32
    add r3, pc, #1   @ R3=pc+1
    bx r3            @ 分支跳转到R3并且切换到Thumb模式下由于最低比特位为1
    .code 16         @ Thumb模式
    cmp r0, #10      
    ite eq           @ if R0 == 10
    addeq r1, #2     @ then R1 = R1 + 2
    addne r1, #3     @ else R1 = R1 + 3
    bkpt
```

.code16 是在 Thumb 模式下执行的代码。这段代码中的条件执行前提是 R0 等于 10。ADDEQ 指令代表了如果条件满足，那么就执行 R1=R1+2，ADDNE 代表了不满足时候的情况。

**分支指令**

分支指令 (也叫分支跳转) 允许我们在代码中跳转到别的段。当我们需要跳到一些函数上执行或者跳过一些代码块时很有用。这部分的最佳例子就是条件跳转 IF 以及循环。先来看看 IF 分支。

```
.global main
main:
mov r1, #2 / 初始化 a /
mov r2, #3 / 初始化 b /
cmp r1, r2 / 比较谁更大些 /
blt r1_lower / 如果R2更大跳转到r1_lower /
mov r0, r1 / 如果分支跳转没有发生，将R1的值放到到R0 /
b end / 跳转到结束 /
r1_lower:
mov r0, r2 / 将R2的值放到R0 /
b end / 跳转到结束 /
end:
bx lr / THE END /
```

上面的汇编代码的含义就是找到较大的数，类似的 C 伪代码是这样的:

```
int main() {
int max = 0;
int a = 2;
int b = 3;
if(a < b) {
max = b;
}
else {
max = a;
}
return max;
}
```

再来看看循环中的条件分支:

```
.global main
main:
mov r0, #0 / 初始化 a /
loop:
cmp r0, #4 / 检查 a==4 /
beq end / 如果是则结束 /
add r0, r0, #1 / 如果不是则加1 /
b loop / 重复循环 /
end:
bx lr / THE END /
```

对应的 C 伪代码长这样子:

```
int main() {
int a = 0;
while(a < 4) {
a= a+1;
}
return a;
}
```

**B/BX/BLX**

有三种类型的分支指令:

Branch(B)

简单的跳转到一个函数

Branch link(BL)

将下一条指令的入口 (PC+4) 保存到 LR，跳转到函数

Branch exchange(BX) 以及 Branch link exchange(BLX)

与 B/BL 相同，外加执行模式切换 (ARM 与 Thumb)

需要寄存器类型作为第一操作数: BX/BLX reg

BX/BLX 指令被用来从 ARM 模式切换到 Thumb 模式。

```
.text
.global _start
_start:
.code 32 @ ARM模式
add r2, pc, #1 @ PC+1放到R2
bx r2 @ 分支切换到R2
.code 16          @ Thumb模式
 mov r0, #1
```

上面的代码将当前的 PC 值加 1 存放到了 R2 中 (此时 PC 指向其后两条指令的偏移处)，通过 BX 跳转到了寄存器指向的位置，由于最低有效位为 1，所以切换到 Thumb 模式执行。下面 GDB 调试的动图说明了这一切。

![](https://p3.ssl.qhimg.com/t01b67d1715923f790b.gif)

**条件分支指令**

条件分支指令是指在满足某种特定条件下的跳转指令。指令模式是跳转指令后加上条件后缀。我们用 BEQ 来举例吧。下面这段汇编代码对一些值做了操作，然后依据比较结果进行条件分支跳转。

![](https://p0.ssl.qhimg.com/t01fc20124173675b7d.png)

对应汇编代码如下:

```
.text
.global _start
_start:
mov r0, #2
mov r1, #2
add r0, r0, r1
cmp r0, #4
beq func1
add r1, #5
b func2
func1:
mov r1, r0
bx lr
func2:
mov r0, r1
bx lr
```

传送门

[【系列分享】ARM 汇编基础速成 1：ARM 汇编以及汇编语言基础介绍](http://bobao.360.cn/learning/detail/4070.html)

[【系列分享】ARM 汇编基础速成 2：ARM 汇编中的数据类型](http://bobao.360.cn/learning/detail/4075.html)

[【系列分享】ARM 汇编基础速成 3：ARM 模式与 THUMB 模式](http://bobao.360.cn/learning/detail/4082.html)

**[【系列分享】ARM 汇编基础速成 4：ARM 汇编内存访问相关指令](http://bobao.360.cn/learning/detail/4087.html)**

[**【系列分享】ARM 汇编基础速成 5：连续存取**](http://bobao.360.cn/learning/detail/4097.html)