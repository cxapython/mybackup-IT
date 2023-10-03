> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/dongry/p/10491031.html)

**1 栈**：栈是一种具有后进先出的数据组织方式，也就是说后存放的先取出，先存放的后取出。栈底是第一个进栈的数据所处位置，栈顶是最后一个数据进栈所处的位置。

![](https://img2018.cnblogs.com/blog/1526920/201903/1526920-20190307084501361-1495151457.png)

  数据组织：有链表、图、树等等（就数据结构那些东东）

**2 满 / 空栈**

  根据 SP 指针指向的位置，栈可以分为满栈和空栈。

  满栈：当堆栈指针总是指向最后压入堆栈的数据

  空栈：当堆栈指针总是指向下一个将要放入数据的空位置

![](https://img2018.cnblogs.com/blog/1526920/201903/1526920-20190307085933944-1357739342.png)

　　ARM 采用满栈

**3 升 / 降栈**

  根据 SP 指针移动的方向，栈可以分为升栈和降栈

  升栈：随着数据的入栈，SP 指针从低地址 -> 高地址移动

  降栈：随着数据的入栈，SP 指针从高地址 -> 低地址移动

![](https://img2018.cnblogs.com/blog/1526920/201903/1526920-20190307090937702-663211520.png)

    ARM 采用降栈

**注：ARM 是满降栈**

**4 栈帧**

**![](https://img2018.cnblogs.com/blog/1526920/201903/1526920-20190307151515958-509645464.png)**

  上图描述的是 ARM 的栈帧布局方式，main stack frame 为调用函数的栈帧，func1 stack frame 为当前函数 (被调用者) 的栈帧，栈底在高地址，栈向下增长。图中 FP 就是栈基址，它指向函数的栈帧起始地址；SP 则是函数的栈指针，它指向栈顶的位置。ARM 压栈的顺序依次为当前函数指针 PC、返回指针 LR、栈指针 SP、栈基址 FP、传入参数个数及指针、本地变量和临时变量。如果函数准备调用另一个函数，跳转之前临时变量区先要保存另一个函数的参数。  
  ARM 也可以用栈基址和栈指针明确标示栈帧的位置，栈指针 SP 一直移动。

  栈帧 (stack frame)：就是一个函数所使用的那部分栈，所有函数的栈帧串起来就组成了一个完整的栈。栈帧的两个边界分别由 fp(r11) 和 sp(r13)来限定。

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
#include <stdio.h>

int main()
{
    ...
    func1();
    ...
}

int func1()
{
    ...
}

```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

  例子中有两个函数，程序运行起来会有一个栈。

![](https://img2018.cnblogs.com/blog/1526920/201903/1526920-20190307092732744-1058957054.png)

  fp(r11) 栈帧指针，栈帧上边界由 fp 指针界定，下边界有 sp 指针界定。从 main 函数进入到 func1 函数，main 函数的上边界和下边界保存在被它调用的栈帧里面。

**5 栈的作用**

**5.1 保存局部变量**

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
#include <stdio.h>

int main()
{
    int a;
    a++;
    return a;
}

```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
/******************************************************
反汇编找到main函数
dongry@d-linux:~/test/hardwork/stack$ arm-linux-gcc -g stack1.c -o stack1
dongry@d-linux:~/test/hardwork/stack$ arm-linux-objdump -D -S stack1 >dump
dongry@d-linux:~/test/hardwork/stack$ vim dump

/main
*******************************************************/

```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
/*反汇编代码*/
 000083a0 <main>:
 #include <stdio.h>
 
 int main()
 {
     83a0:       e1a0c00d        mov     ip, sp                    
     83a4:       e92dd800        stmdb   sp!, {fp, ip, lr, pc}
     83a8:       e24cb004        sub     fp, ip, #4      ; 0x4
     83ac:       e24dd004        sub     sp, sp, #4      ; 0x4
         int a;
         a++;
     83b0:       e51b3010        ldr     r3, [fp, #-16]
     83b4:       e2833001        add     r3, r3, #1      ; 0x1
     83b8:       e50b3010        str     r3, [fp, #-16]
         return  a;
     83bc:       e51b3010        ldr     r3, [fp, #-16]
 }

```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
/*分析*/
    mov ip，sp        //保存sp到ip
    stmdb sp!,{fp,ip,lr,pc}   /*先对sp-4，再对fp,ip,lr,pc压栈*/
                             //sp=sp-4;push {pc};sp=pc;  /*先压pc*/
                             //sp=sp-4;push {lr};sp=lr; /*压lr*/
                             //sp=sp-4;push {ip};sp=ip;  /*压ip*/
                             //sp=sp-4;push {fp};sp=fp; /*压fp*/
    sub fp,ip,#4        //fp指向ip-4
    sub sp,sp,#4       //开辟一块空间
  
    ldr r3,[fp,#-16]   //临时存放在[fp-16]
    add r3,r3,#1
    str r3,[fp,#-16]

```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

**5.2 参数传递**

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
#include <stdio.h>

void func(int a,int b,int c,int d,int e,int f)
{
    int k;
    int l;
    k=e+f;
    l=a+b;
}

int main()
{
    func(1,2,3,4,5,6);
    return 0;
}

```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

 参数大于 4 个的时候，多出来的参数用栈传递；

**5.3 保存寄存器的值**

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

```
#include <stdio.h>

void func2(int a,int b)
{
    int k;
    k=a+b;
}

void func1(int a,int b)
{
    int c;
    func2(3,4);
}

int main()
{
    func1(1,2);
    return 0;
}

```

[![](http://common.cnblogs.com/images/copycode.gif)](javascript:void(0); "复制代码")

 如果不用栈，会将原来 r0、r1 寄存器中的值覆盖掉