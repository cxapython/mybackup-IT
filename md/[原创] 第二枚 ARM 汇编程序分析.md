> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [bbs.pediy.com](https://bbs.pediy.com/thread-176423.htm)

> [原创] 第二枚 ARM 汇编程序分析

前几天 发了一个 ARM 汇编版的 CrackMe，好像还挺 受欢迎的，但是由于. o 文件无法由安卓程序调用，所以只能在 dos 下用 adb shell 执行，看上去很不直观，引起一些朋友不解。

今天 我给大家 写了一个 so 文件 ，这样 android 程序 就可以调用它了，再加上安卓的界面，看起来就要舒服点，呵呵。

![](https://bbs.pediy.com/view/img/face/12.gif)

由于 本人根本没学过安卓开发，所以要写个复杂的程序还是比较困难的，而分析 arm 汇编的指令又不需要看安卓代码 ，因此我在 sample 中找了一个 HelloJni 程序，向里面

加了一段自己写的 c 代码，就这样，分析旅程就开始了。

我写这个程序旨在熟悉 ARM 汇编指令，所以我尽可能的让它产生各种指令，如 MUL,SUB,EOR,LDR,STR 等，再加入程序执行的各种流程，如顺序，if else,for 循环，while 循环，switch case，还有一个子程序调用。

```
libhello_jni.so:491DCD4C STMFD   SP!, {R4-R8,LR}                 ; 将r4,45,r6,r7,r8,lr寄存器入栈
libhello_jni.so:491DCD50 LDR     R3, =(aWww_pediy_com - 0xD64)   ; 获取字符串name的相对地址
libhello_jni.so:491DCD54 SUB     SP, SP, #0x80                   ; 分配栈空间
libhello_jni.so:491DCD58 ADD     R12, SP, #0x6C
libhello_jni.so:491DCD5C ADD     R3, PC, R3                      ; 相对地址加上当前地址，等于字符串Name的绝对地址
libhello_jni.so:491DCD60 MOV     R5, R0
libhello_jni.so:491DCD64 LDR     R6, =(unk_491DE3A8 - 0x491DCD78) ; 获取hash数组的首地址的相对地址
libhello_jni.so:491DCD68 LDMIA   R3, {R0-R3}                     ; 将R3所指向的值存入到R0,R1,R2,R3寄存器中
libhello_jni.so:491DCD6C STMIA   R12!, {R0-R2}                   ; 将R0,R1,R2中的数值放入R12所指向的内存，
;以便空出寄存器，继续读取数据，这里为什么没有读取R3中的数据呢？因为R3只有低8位有数据，不便于直接读入，在后面会读入
libhello_jni.so:491DCD70 ADD     R6, PC, R6                      ; 计算出hash数组的绝对地址
libhello_jni.so:491DCD74 STRH    R3, [R12]                       ; 这里读入R3中的数据
libhello_jni.so:491DCD78 MOV     R7, R6                          ; 将R6中保存的hash数组首地址放入R7中
libhello_jni.so:491DCD7C ADD     LR, SP, #4
libhello_jni.so:491DCD80 LDMIA   R7!, {R0-R3}                    ; 依次读入R7所以地址的四个数据到R0-R3寄存器，完成后R7的地址自动加0x10
libhello_jni.so:491DCD84 STMIA   LR!, {R0-R3}                    ; 将刚才读到的数据存入到lr所在的连续内存中，完成后lr的值自动加0x10
libhello_jni.so:491DCD88 LDMIA   R7!, {R0-R3}                    ; 继续取hash数组中的值
libhello_jni.so:491DCD8C LDR     R4, =(unk_491DFFB0 - 0x491DCDA0) ; 获取第三个数组data的首地址的相对地址
libhello_jni.so:491DCD90 STMIA   LR!, {R0-R3}                    ; 将上面读到的数据存入到lr所在的连续内存中
libhello_jni.so:491DCD94 LDMIA   R7!, {R0-R3}                    ; 继续取hash数组中的值
libhello_jni.so:491DCD98 LDR     R4, [PC,R4]                     ; 注意这句不是像上面ADD R6,PC,R6,这句行等价于ADD R4,[PC+R4]
libhello_jni.so:491DCD9C LDR     R8, [R4]
libhello_jni.so:491DCDA0 ADD     R6, R6, #0x34                   ; R6加上hash的长度，使其指向data数据的首地址
libhello_jni.so:491DCDA4 ADD     R12, SP, #0x38                  ; 调整栈空间，移到空位，准备接收新数据
libhello_jni.so:491DCDA8 STR     R8, [SP,#0x7C]
libhello_jni.so:491DCDAC STMIA   LR!, {R0-R3}                    ; 继续将上面未存完的hash数据存入到lr所在内存
libhello_jni.so:491DCDB0 LDMIA   R6!, {R0-R3}                    ; 现在开始将data数组 中的数据进行存入操作
libhello_jni.so:491DCDB4 LDR     R7, [R7]
libhello_jni.so:491DCDB8 STMIA   R12!, {R0-R3}                   ; 将从data数组中得到 的数据存入到R12所在内存，完成后r12自动加0x10
libhello_jni.so:491DCDBC LDMIA   R6!, {R0-R3}                    ; 继续获取data中的数据
libhello_jni.so:491DCDC0 STMIA   R12!, {R0-R3}                   ; 再存入
libhello_jni.so:491DCDC4 LDMIA   R6!, {R0-R3}                    ; 再读取
libhello_jni.so:491DCDC8 STMIA   R12!, {R0-R3}                   ; 再存入
libhello_jni.so:491DCDCC LDR     R3, [R6]
libhello_jni.so:491DCDD0 ADD     R0, SP, #0x6C
libhello_jni.so:491DCDD4 STR     R7, [LR]
libhello_jni.so:491DCDD8 STR     R3, [R12]
libhello_jni.so:491DCDDC BL      unk_491DCBFC                    ; 调用 strlen 计算name值长度
libhello_jni.so:491DCDE0 CMP     R0, #0                          ; 比较是否为空
libhello_jni.so:491DCDE4 BLE     loc_491DCE78                    ; 若长度小于等于0，则跳过后面的for循环
libhello_jni.so:491DCDE8 ADD     R6, SP, #0x6C
libhello_jni.so:491DCDEC MOV     R12, SP
libhello_jni.so:491DCDF0 ADD     R7, R6, R0
libhello_jni.so:491DCDF4 MOV     R3, R6
libhello_jni.so:491DCDF8
```

前面这一段，实际就是在做数据处理，把程序要用到的数据，放在各个地方，方便下面使用，这些代码就是把数据挪过去挪过来的，如果有不懂的请跳转到下面的链接，也就是我的上一篇文章中。

第一枚 ARM 汇编版 CrackMe 分析

[http://bbs.pediy.com/showthread.php?p=1204097#post1204097](http://bbs.pediy.com/showthread.php?p=1204097#post1204097)

```
for循环：
 
.text:00000DF8 loc_DF8                                 ; CODE XREF: .text:00000E24j
.text:00000DF8                 LDR     R1, [R12,#4]!   ; 取name[i]的值
.text:00000DFC                 LDRB    R2, [R3]        ; R3等于变量i，作为循环基数
.text:00000E00                 EOR     R2, R1, R2      ; 进行异或运算
.text:00000E04                 AND     R2, R2, #0xFF   ; 若得到的数越界，则取低8位
.text:00000E08                 SUB     R1, R2, #0x14   ; name[i]减20
.text:00000E0C                 TST     R1, #1          ; 判断数值是否为2的倍数
.text:00000E10                 STRB    R1, [R3],#1     ; 将R1的值存入R3所指向的地址，完成后R3加1
.text:00000E14                 SUBEQ   R2, R2, #0xA    ; 是2的倍数，则减10
.text:00000E18                 SUBNE   R2, R2, #0xF    ; 不是2的倍数，则减15
.text:00000E1C                 CMP     R3, R7          ; R7是字符串长度，循环的上限
.text:00000E20                 STRB    R2, [R3,#-1]    ; 将R2的值，存入到[R3-1]所指向的地址
.text:00000E24                 BNE     loc_DF8         ; for循环条件判断
```

注意这几句代码的逻辑：

.text:00000E08                 SUB     R1, R2, #0x14

.text:00000E0C                 TST     R1, #1

.text:00000E10                 STRB    R1, [R3],#1

.text:00000E14                 SUBEQ   R2, R2, #0xA

.text:00000E18                 SUBNE   R2, R2, #0xF

.text:00000E1C                 CMP     R3, R7

.text:00000E20                 STRB    R2, [R3,#-1]

一、先让 R1=R2-0x14，再检测 R1 的值是否为 2 的倍数，然后把 R1 的值存入到内存单元，假定内存单元为 m1，

二、如果该值是 2 的倍数，则 R2 的值 - 10，若不为 2 的倍数，则 - 15. 这里是不是感觉有点奇怪。原码中是如果是 2 的倍数，则加上 10，如果不是 2 的倍数，则 + 5，这里为什么是减呢？

我们继续往下看，发现 R2-10，或 - 15 后直接就存入了内存 m1 中，原为这里是结和上面 r1-20 一起算的。

调用子程序：

下面是子程序中的 while 循环中加一个 switch case 结构，本来不想写 switch case 结构的，因为这个太简单了，后来出于流程结构完整的考虑，还是把它写上吧。

```
.text:00000C74                 STMFD   SP!, {R4,R5}
.text:00000C78                 LDRB    R3, [R0]        ; 取name[i]的值，其实 就是while循环的首次条件判断
.text:00000C7C                 CMP     R3, #0
.text:00000C80                 BEQ     loc_CF8         ; 等于0则while循环结束
.text:00000C84                 LDR     R4, =0xCCCCCCCD ; 为下面的求模运算作准备
.text:00000C88                 ADD     R2, R0, #1   ;为下一次取name数组的值做准备
.text:00000C8C                 MOV     R12, #0xA
.text:00000C90
.text:00000C90 loc_C90                                 ; CODE XREF: function+80j
.text:00000C90                 MOV     R3, R3,LSR#2    ; R3=R3>>2，对应源码中的name[i]=name[i]>>2
.text:00000C94                 UMULL   R5, R1, R4, R3  ; R1存放结果中的高数数据，R5存入低位数据
.text:00000C98                 MOV     R1, R1,LSR#3    ; R1=R1>>3
.text:00000C9C                 MUL     R1, R12, R1     ; R1=R12*R1
.text:00000CA0                 RSB     R1, R1, R3      ; 逆减法指令，R1=R3-R1
.text:00000CA4                 AND     R1, R1, #0xFF   ; 如果越界，则取低8位的数
```

这一段指令看上去不知道是要做什么，各种乘法加减法的，实际上你只要分析一个数据 ，你就知道它是拿来实现求模运算的。

拿第一个数据来比如设 a=16，那么按照上面的算法有。

0xcccccccd*16=cccccccd0, 高位存入 R1, 于是 R1 等于 C;

下一句等价于 R1=R1>>3，=>R1=R1/8=1;

然后 mul r1,r12,r1=>r1=1*10=10;

rsb  r1,r1,r3=>r3-r1=16-10=6;

而 16%10=6，所以这就是求模运算的步骤了。

```
.text:00000CA8                 SUB     R1, R1, #1      ; R1=R1-1
.text:00000CAC                 STRB    R3, [R0]        ; 将R3的值存入R0所指向的寄存器中
.text:00000CB0                 CMP     R1, #8          ; switch 9 cases
.text:00000CB4                 ADDLS   PC, PC, R1,LSL#2 ; switch jump
.text:00000CB8                 B       loc_D38         ; jumptable 00000CB4 default case
.text:00000CBC ; ---------------------------------------------------------------------------
.text:00000CBC
.text:00000CBC loc_CBC                                 ; CODE XREF: function+40j
.text:00000CBC                 B       loc_D30         ; jumptable 00000CB4 case 0
.text:00000CC0 ; ---------------------------------------------------------------------------
.text:00000CC0
.text:00000CC0 loc_CC0                                 ; CODE XREF: function+40j
.text:00000CC0                 B       loc_D28         ; jumptable 00000CB4 case 1
.text:00000CC4 ; ---------------------------------------------------------------------------
.text:00000CC4
.text:00000CC4 loc_CC4                                 ; CODE XREF: function+40j
.text:00000CC4                 B       loc_D20         ; jumptable 00000CB4 case 2
.text:00000CC8 ; ---------------------------------------------------------------------------
.text:00000CC8
.text:00000CC8 loc_CC8                                 ; CODE XREF: function+40j
.text:00000CC8                 B       loc_D18         ; jumptable 00000CB4 case 3
.text:00000CCC ; ---------------------------------------------------------------------------
.text:00000CCC
.text:00000CCC loc_CCC                                 ; CODE XREF: function+40j
.text:00000CCC                 B       loc_D10         ; jumptable 00000CB4 case 4
.text:00000CD0 ; ---------------------------------------------------------------------------
.text:00000CD0
.text:00000CD0 loc_CD0                                 ; CODE XREF: function+40j
.text:00000CD0                 B       loc_D08         ; jumptable 00000CB4 case 5
.text:00000CD4 ; ---------------------------------------------------------------------------
.text:00000CD4
.text:00000CD4 loc_CD4                                 ; CODE XREF: function+40j
.text:00000CD4                 B       loc_D00         ; jumptable 00000CB4 case 6
.text:00000CD8 ; ---------------------------------------------------------------------------
.text:00000CD8
.text:00000CD8 loc_CD8                                 ; CODE XREF: function+40j
.text:00000CD8                 B       loc_CE0         ; jumptable 00000CB4 case 7
.text:00000CDC ; ---------------------------------------------------------------------------
.text:00000CDC
.text:00000CDC loc_CDC                                 ; CODE XREF: function+40j
.text:00000CDC                 B       loc_D40         ; jumptable 00000CB4 case 8
.text:00000CE0 ; ---------------------------------------------------------------------------
.text:00000CE0
.text:00000CE0 loc_CE0                                 ; CODE XREF: function+40j
.text:00000CE0                                         ; function:loc_CD8j
.text:00000CE0                 MOV     R3, #0x70       ; jumptable 00000CB4 case 7
.text:00000CE4
.text:00000CE4 loc_CE4                                 ; CODE XREF: function+90j
.text:00000CE4                                         ; function+98j ...
.text:00000CE4                 STRB    R3, [R0]        ; 将得case中得到 的值放入到name数组中
.text:00000CE8                 MOV     R0, R2
.text:00000CEC                 LDRB    R3, [R2],#1     ; 取下一个name[i]的值，继续进行一下轮循环
.text:00000CF0                 CMP     R3, #0
.text:00000CF4                 BNE     loc_C90         ; R3=R3>>2，对应源码中的name[i]=name[i]>>2
.text:00000CF8
.text:00000CF8 loc_CF8                                 ; CODE XREF: function+Cj
.text:00000CF8                 LDMFD   SP!, {R4,R5}
.text:00000CFC                 BX      LR              ; while循环结束
.text:00000D00 ; ---------------------------------------------------------------------------
.text:00000D00
.text:00000D00 loc_D00                                 ; CODE XREF: function+40j
.text:00000D00                                         ; function:loc_CD4j
.text:00000D00                 MOV     R3, #0x6E       ; jumptable 00000CB4 case 6
.text:00000D04                 B       loc_CE4         ; 将得case中得到 的值放入到name数组中
.text:00000D08 ; ---------------------------------------------------------------------------
.text:00000D08
.text:00000D08 loc_D08                                 ; CODE XREF: function+40j
.text:00000D08                                         ; function:loc_CD0j
.text:00000D08                 MOV     R3, #0x6C       ; jumptable 00000CB4 case 5
.text:00000D0C                 B       loc_CE4         ; 将得case中得到 的值放入到name数组中
.text:00000D10 ; ---------------------------------------------------------------------------
.text:00000D10
.text:00000D10 loc_D10                                 ; CODE XREF: function+40j
.text:00000D10                                         ; function:loc_CCCj
.text:00000D10                 MOV     R3, #0x6A       ; jumptable 00000CB4 case 4
.text:00000D14                 B       loc_CE4         ; 将得case中得到 的值放入到name数组中
.text:00000D18 ; ---------------------------------------------------------------------------
.text:00000D18
.text:00000D18 loc_D18                                 ; CODE XREF: function+40j
.text:00000D18                                         ; function:loc_CC8j
.text:00000D18                 MOV     R3, #0x68       ; jumptable 00000CB4 case 3
.text:00000D1C                 B       loc_CE4         ; 将得case中得到 的值放入到name数组中
.text:00000D20 ; ---------------------------------------------------------------------------
.text:00000D20
.text:00000D20 loc_D20                                 ; CODE XREF: function+40j
.text:00000D20                                         ; function:loc_CC4j
.text:00000D20                 MOV     R3, #0x66       ; jumptable 00000CB4 case 2
.text:00000D24                 B       loc_CE4         ; 将得case中得到 的值放入到name数组中
.text:00000D28 ; ---------------------------------------------------------------------------
.text:00000D28
.text:00000D28 loc_D28                                 ; CODE XREF: function+40j
.text:00000D28                                         ; function:loc_CC0j
.text:00000D28                 MOV     R3, #0x64       ; jumptable 00000CB4 case 1
.text:00000D2C                 B       loc_CE4         ; 将得case中得到 的值放入到name数组中
.text:00000D30 ; ---------------------------------------------------------------------------
.text:00000D30
.text:00000D30 loc_D30                                 ; CODE XREF: function+40j
.text:00000D30                                         ; function:loc_CBCj
.text:00000D30                 MOV     R3, #0x62       ; jumptable 00000CB4 case 0
.text:00000D34                 B       loc_CE4         ; 将得case中得到 的值放入到name数组中
.text:00000D38 ; ---------------------------------------------------------------------------
.text:00000D38
.text:00000D38 loc_D38                                 ; CODE XREF: function+40j
.text:00000D38                                         ; function+44j
.text:00000D38                 MOV     R3, #0x60       ; jumptable 00000CB4 default case
.text:00000D3C                 B       loc_CE4         ; 将得case中得到 的值放入到name数组中
.text:00000D40 ; ---------------------------------------------------------------------------
.text:00000D40
.text:00000D40 loc_D40                                 ; CODE XREF: function+40j
.text:00000D40                                         ; function:loc_CDCj
.text:00000D40                 MOV     R3, #0x72       ; jumptable 00000CB4 case 8
.text:00000D44                 B       loc_CE4         ; 将得case中得到 的值放入到name数组中
.text:00000D44 ; End of function function
.text:00000D44
```

switch case 简直太啰嗦了，但是为了全面 ，还是完整的贴上来了，呵呵。

末尾：

```
.text:00000E34 loc_E34                                 ; CODE XREF: .text:00000E48j
.text:00000E34                 LDRB    R2, [R6]        ; 取生成的数据
.text:00000E38                 LDR     R1, [R3,#4]!    ; 取一串数据，用于后面生成好看的字串
.text:00000E3C                 EOR     R2, R1, R2      ; 进行异或运算
.text:00000E40                 STRB    R2, [R6],#1     ; 把运算得到 的数据存入name数组中
.text:00000E44                 CMP     R6, R7          ; 判断循环是否结束
.text:00000E48                 BNE     loc_E34         ; for循环，用于遍历数组，完成异或运算
.text:00000E4C
.text:00000E4C loc_E4C                                 ; CODE XREF: .text:00000E80j
.text:00000E4C                 LDR     R3, [R5]
.text:00000E50                 MOV     R0, R5
.text:00000E54                 ADD     R1, SP, #0x6C
.text:00000E58                 LDR     R3, [R3,#JNINativeInterface.NewStringUTF]  ;jni函数
.text:00000E5C                 BLX     R3
.text:00000E60                 LDR     R2, [SP,#0x7C]
.text:00000E64                 LDR     R3, [R4]
.text:00000E68                 CMP     R2, R3
.text:00000E6C                 BNE     loc_E84
.text:00000E70                 ADD     SP, SP, #0x80
.text:00000E74                 LDMFD   SP!, {R4-R8,PC}
.text:00000E78 ; -------------------------------------------------------------------------
```

再把 c 程序帖上来吧，方便大学学习

```
#include #include void function(char name[])
{
    char key[]="0123456789";
    int CaseNum,i=0;
    while(name[i])
    {
        name[i]=name[i]>>2;
        CaseNum=name[i]%10;
        switch (CaseNum)
        {
        case 0: name[i]=key[0];
            break;
        case 1: name[i]=key[1];
            break;
        case 2: name[i]=key[2];
            break;
        case 3: name[i]=key[3];
            break;
        case 4: name[i]=key[4];
            break;
        case 5: name[i]=key[5];
            break;
        case 6: name[i]=key[6];
            break;
        case 7: name[i]=key[7];
            break;
        case 8: name[i]=key[8];
            break;
        case 9: name[i]=key[9];
            break;
        default:name[i]='x';
            break;
        }
        name[i]=name[i]<<1;
        i++;
    }
 
}
jstring
Java_com_example_hehe_MainActivity_stringFromJNI( JNIEnv* env,
                                                  jobject thiz )
{
    char name[]="www.pediy.com";
        int hash[]={0x10,0x12,0x14,0x46,0x18,0x1a,0x1c,0x1f,0x21,0x53,0x25,0x27,0x29};
        int data[13]={0x13,0x15,0x15,0x48,0x4,0x11,0x7,0xa,0x7,0x40,0x9,0x5,0x5};
        int i,length;
 
        length=strlen(name);
        for(i=0;iNewStringUTF(env, name);
} 
```

运行结果：

点击按钮前，也就是没有运行 so 中的程序前如图：

![](https://bbs.pediy.com/upload/attach/201308/512840_obqzew95y8t23fc.png)

点击按钮，经过运算后得到如下图所示的结果：

![](https://bbs.pediy.com/upload/attach/201308/512840_jem5w1vf2s1qmlq.png)

程序就分析到这里了，如果有不明白的地方，欢迎回帖交流。

另外简单说一下远程调试 so 库文件的设置

启动远程服务：

adb shell /data/local/tmp/android_server

进行端口转发：

adb forward tcp:23946 tcp:23946

在 IDA 中进行附加

![](https://bbs.pediy.com/upload/attach/201308/512840_cy11d2q4f47ace7.png)

进行如下设置

![](https://bbs.pediy.com/upload/attach/201308/512840_fjjiot8dv2hxna2.png)

远程要附加的进程

![](https://bbs.pediy.com/upload/attach/201308/512840_ath0qok3axs57ap.png)

附加过后，需要计算出原生程序的入口地址，便于进行跳转，用另一个 IDA 静态反编译文件，找到函数入口地址：

![](https://bbs.pediy.com/upload/attach/201308/512840_ntxrpy6xixoufqn.png)

回到动态调试 IDA 中，Ctrl+S，找到要调试的原生动态链接库。

![](https://bbs.pediy.com/upload/attach/201308/512840_e42impvmhslqbq5.png)

函数地址 = 动态链接库起始地址 + 偏移地址  即  491dc000+bc4=491dcbc4

正确下断后，运行程序，再到程序中触发这个函数调用 ，就会断在断点处，如图：

![](https://bbs.pediy.com/upload/attach/201308/512840_azluble5rqfbwm4.png)

希望大家继续把 arm 汇编的东西接下去。

[[2022 冬季班]《安卓高级研修班 (网课)》月薪两万班招生中～](https://www.kanxue.com/book-section_list-83.htm)

上传的附件：

*   [2.png](javascript:void(0)) （54.04kb，15 次下载）
*   [3.png](javascript:void(0)) （59.41kb，15 次下载）
*   [1.png](javascript:void(0)) （20.81kb，21 次下载）
*   [4.png](javascript:void(0)) （30.71kb，4 次下载）
*   [5.png](javascript:void(0)) （16.51kb，3 次下载）
*   [6.png](javascript:void(0)) （26.44kb，4 次下载）
*   [7.png](javascript:void(0)) （13.12kb，25 次下载）
*   [8.png](javascript:void(0)) （48.93kb，14 次下载）
*   [第二枚 ARM 汇编程序分析. rar](javascript:void(0)) （232.52kb，290 次下载）