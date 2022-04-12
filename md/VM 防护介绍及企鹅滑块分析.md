> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/C8gB-D6EUliPXoMgjk0Bag)

![](https://mmbiz.qpic.cn/mmbiz_jpg/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojtr9UY0OVtopxgKIgVk6odktT3w89viczE3Xiapdjfwpo0tGjERusDTPw/640?wx_fmt=jpeg)
=============================================================================================================================================

1 本文概览  

=========

    前一阵反编译了企鹅的 jsvm，就跟着清晰了很多的逻辑逆向了一下。最近终于是全部搞定了，借着反编译之后的代码和大家分享一下逆向的过程吧。

    本文主要分为两大部分，第一部分介绍下什么是 vm 和企鹅 jsvm 的设计，第二部分说一下企鹅滑块在 vm 之下具体包含了哪些检测逻辑。

2 vm 防护介绍
=========

    这个问题很大，本节就从实用的角度出发，少理论，多举例，尽量让大家对 vm 有一个直观的认识吧，理解了正向设计，才会想到更多逆向的奇淫巧技。

2.1 vm 的概述
----------

    虚拟机实现的基础，是任何复杂的源码，都可以将其复杂指令拆解，最终用有限种类的简单指令代替，这个思想与所有语言的复杂代码都可以转换为某一架构的有限种类的汇编指令是一样。

    所以从理解的角度，虚拟机指令可以看做汇编指令，而虚拟机可以看做是特定架构的 cpu。

    既然虚拟机的作用只是对虚拟机指令进行解释执行，那肯定另外还有一个程序，将源码转换成虚拟机指令，这个程序就是编译器。

    上面所述的流程，用流程表示如下：

```
start->源码->编译器->虚拟机指令->虚拟机->end
```

2.2 vm 的实现原理
------------

    我们举个小例子，看下编译器和虚拟机是如何工作的。

### 2.2.1 编译器的实现

    定义几个虚拟机指令如下：

<table><thead><tr><th>虚拟机指令</th><th>释义</th></tr></thead><tbody><tr><td>10 number</td><td>将 number 压入操作数栈</td></tr><tr><td>11</td><td>将两栈顶元素弹出，相加，再将结果压回操作数栈</td></tr><tr><td>12</td><td>将栈顶元素弹出，返回</td></tr></tbody></table>

    则下列代码，

```
var a = 2;var b = 3;var c = a + b;return c;
```

    可以转换为虚拟机指令如下，这是编译器的任务（具体实现代码比较复杂，故此处略过，直接给出结果。）

`10 2 10 3 11 12`

    对上述虚拟机指令进行解释：

<table><thead><tr><th>虚拟机指令</th><th>操作数</th><th>释义</th></tr></thead><tbody><tr><td>10</td><td>2</td><td>将 2 压入操作数栈</td></tr><tr><td>10</td><td>3</td><td>将 3 压入操作数栈</td></tr><tr><td>11</td><td><br></td><td>将栈顶的两个元素弹出，相加，再将结果压回操作数栈</td></tr><tr><td>12</td><td><br></td><td>将栈顶元素弹出，return 该元素</td></tr></tbody></table>

### 2.2.2 虚拟机的实现

    而为了对上述虚拟机指令进行执行，我们将虚拟机设计如下：

```
var opcodes = [10, 2, 10, 3, 11, 12];     // 编译器编译出的虚拟机指令var operand_stack = [];                   // 操作数栈var index = 0;                            // 根据index读取opcodes数组中的内容while(index != opcodes.length){        var opcode = opcodes[index++];        switch opcode:                case 10:             var number = opcodes[index++];             opreater_arr.push(number);             break;                            case 11:             var number1 = operand_stack.pop();             var number2 = operand_stack.pop();             var number3 = number1 + number2;             opreater_arr.push(number3);             break;                            case 12:             var number = operand_stack.pop();             return number;}
```

    虚拟机执行过程中，操作数栈的变化如下：

<table><thead><tr><th>虚拟机指令</th><th>操作数</th><th>指令运行后操作数栈内容（左侧栈底）</th></tr></thead><tbody><tr><td>10</td><td>2</td><td>[2]</td></tr><tr><td>10</td><td>3</td><td>[2, 3]</td></tr><tr><td>11</td><td><br></td><td>[5]</td></tr><tr><td>12</td><td><br></td><td>[]</td></tr></tbody></table>

    如果有人逆向过某个 vm 的话，应该会感觉上述的虚拟机实现代码似曾相识。

    当然也有虚拟机把 switch 转为嵌套的 if...else... 语句，或者加些无效代码之类的，让逆向工程师分析起来更困难。面对这种有混淆的情况，可以先把代码改造成易于理解的结构，再进行分析。

2.3 另一种看似 vmp 的实现
-----------------

    这个实现就是抽象语法树解释器，它对抽象语法树进行解释执行。

    一般源码对应语言的抽象语法树有多少种类型，解释器就要有多少种 opcode。因为这个实现会导致 opcode 种类很多，所以如果真的见到，会感觉有些唬人。但是由于抽象语法树本就是源码的映射，所以在真正调试的时候，或许会感觉比较流畅。

2.4 小结
------

    通过一个上面的描述，大家应该对编译器和虚拟机的工作原理有了一个直观的认识，有了这个初步认识，我们再向下介绍企鹅的 jsvm。

    另外，如果各位对编译器和虚拟机的实现感兴趣，可以找编译原理相关的资料进行深入的研究，真实的实现情况远比此例复杂，但同时也更加有趣。

3 企鹅 jsvm 分析
============

3.1 概述
------

    有了上面小节的基础之后，我们再来实际看一下，企鹅 jsvm 这样的成熟商用版本的设计是什么样的。

    这就相当于，既然有了 1+1=2 的基础，我想大家应该可以解哥德巴赫猜想了吧？![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmoj3NiaUNqjVib2xqyfianoxP3XlJiaoGBU5WfOMFaZRPfNvlianKzxyibn11OQ/640?wx_fmt=png)

    好的，开个玩笑，我继续向下介绍。

    为了理解企鹅 jsvm 的设计，我们需要先提升脑海中对虚拟机的认知，然后再分析企鹅 jsvm。

3.2 vm 认知升级
-----------

    第二节中我只是介绍了虚拟机对加法和 return 语句的支持，这显然是不够的。现在，我们再来考虑更多的问题，问题如下：

1.  虚拟机如何支持如`if`这样的分支语句
    
2.  如何支持如`for`和`while`这样的循环语句
    
3.  如何支持函数
    
4.  如何支持一元运算，二元运算
    
5.  如何完成函数调用
    
6.  如何完成异常捕获
    

    好像难度开始升级了是吗，不过不要慌，各位先在脑海中思考一下这些问题的答案。

    然后，下文中我借着企鹅 jsvm 的真实实现，来一一说明。

3.3 企鹅 jsvm 设计
--------------

### 3.3.1 跳转指令

    跳转语句分为条件跳转与无条件跳转

#### 3.3.1.1 条件跳转指令

    像`if`、`for`、`while`这样，根据不同条件进入不同的代码块执行的语句，都可以通过条件跳转指令实现。虚拟机通过修改`2.2.2`节中那个 index 索引，就可以实现分支和循环的效果，在企鹅 jsvm 的实现及注释如下：

```
function () {    var A = opcodes[index++];                                // 该指令有一个操作数A    operand_stack[operand_stack.length - 1] && (index = A);  // 如果操作数栈中最后一个元素是true，就将index索引设置为A}
```

    如果`g=A`后，跳过了一部分 opcode 不执行，就是 if 这样的分支语句的实现。

    如果`g=A`后，又回到了之前已经执行过的 opcode，对这部分 opcode 进行重复执行，就是 for 等循环语句的实现。

#### 3.3.1.2 无跳转跳转指令

    这种指令可以忽视，就是各语句顺序运行的效果，在企鹅 jsvm 中的实现如下：

```
function () {    var A = = opcodes[index++];   // 该指令有一个操作数    index = A;                    // 将index直接赋值为操作数}
```

### 3.3.2 函数

    函数就涉及到了作用域管理，例子如下：

```
var a=1;           // scope1var b=2;           // scope1function fun(){    var a = 3;             // scope2    console.log(a);        // scope2    console.log(b);        // scope2}  fun();             // scope1console.log(a);    // scope1
```

    该代码的运行结果为

    函数外和函数内是两个作用域，我们设为 scope1 和 scope2。

    在上面例子中，我们可以看到两点：

    1，尽管变量 a 名称相同，但函数内对 a 的赋值不会影响函数外 a 的值，这就是作用域的体现。在 JavaScript 语言中操作数赋值的操作是与当前作用域绑定的 (这是语义分析决定的，所以其他语言未必如此)。

    2，scope2 作用域中没有 b 变量，但可以读取函数外 scope1 作用域中 b 变量的值，这说明了作用域间有嵌套的关系。

    我们通过这个例子对作用域有个简单的认知，函数本质就是可以复用的代码块，在虚拟机的设计中，对函数的作用域支持无疑是更复杂的。

    然后我们看下企鹅 jsvm 的设计，其对函数的定义如下：

```
function () {    var K = m[g++];    var p = [];    var A = m[g++];    var C = m[g++];    var Q = [];    var B = 0;    for (; B < A; B++) {        p[m[g++]] = n[m[g++]];    }    B = 0    for (; B < C; B++) {        Q[B] = m[g++];    }    n.push(function w() {        var A = p.slice(0);    // A为函数使用的参数        A[0] = [this];        A[1] = [arguments];        A[2] = [w];        var C = 0;        for (; C < Q.length && C < arguments.length; C++) {            0 < Q[C] && (A[Q[C]] = [arguments[C]]);        }        return __TENCENT_CHAOS_VM(K, m, U, A, E, F, Y, c);    });}
```

    可以看到，这里将一个函数 w 压栈，而函数 w 中返回了`__TENCENT_CHAOS_VM`这个自身的函数。

    也就是说，它对函数作用域的支持，其实是直接使用了 JavaScript 本身的函数作用域。

    将不同的虚拟机指令片段，各自在`__TENCENT_CHAOS_VM`函数中运行，那些虚拟机指令片段自然就使用了单独的函数作用域。这里有点绕，可能需要跟着代码调试一下才好理解。

### 3.3.3 函数调用

    说完函数，我们再来看一下函数调用。上文中看到函数已经被压入操作数栈了，那调用的话，我们想来，其实 pop() 出这个函数，然后直接执行就可以了。

    我们看下企鹅 jsvm 的实现

```
function anonymous() {    var A = m[g++];                  // 在虚拟机指令集中读取一个操作数    var C = A ? n.slice(-A) : [];    // 在操作数栈中读取出一部分，作为函数参数，这里本质和pop()是一致的    n.length -= A;                   // 操作数栈舍弃上一条指令中读出的部分    n.push(n.pop().apply(U, C));     // 操作数栈pop()出一个函数，然后通过apply设置参数及调用函数。}
```

    嗯，和我们预想中的一样。

### 3.3.4 一元运算指令与二元运算指令

    这些指令因为逻辑简单，所以很好理解，基本都是 出栈 -> 计算 -> 入栈 的流程。

    比如下面这个取反的一元运算

```
function () {    n.push(!n.pop());   # 从操作数栈中弹出一个值，取反，再push回操作数栈。}
```

    再比如这个相加的二元运算

```
function () {    n[n.length - 2] = n[n.length - 2] + n.pop(); # 相当于pop()两次，相加，再push()回结果。}
```

        有了前文的基础，各位应该看这些指令很容易，这里就不用细说了。

### 3.3.5 异常捕获

```
function () {    C.push([m[g++], n.length, m[g++]]); // 这里push了一个此刻的运行状态到异常栈中。}
```

```
try {    // 依次读取opcodes，进行执行的逻辑。} catch (D) {            0;    var c = C.pop();    if (c === undefined) throw D;                              // 如果运行过程中出现异常，将上面的状态在异常栈中弹出。    h = D, w = c[0], U.length = c[1], c[2] && (U[c[2]][0] = h) // 做一些保存异常信息的操作，当前__TENCENT_CHAOS_VM函数执行结束，返回调用处。}
```

### 3.3.6 小结

    本部分针对几个有代表性的问题，分析了下企鹅 jsvm。有了现在的基础，再加上实际上手调试，相信各位已经对该 vm 十分了解了。而以后遇到其他厂商的 vm 产品，也可以通过我上述说的那些问题去分析。从这些关键的问题入手，层层剖析，vm 也就变的不再难以理解。

    上面对 vm 进行介绍的内容确实有些不好讲，讲的深入了，就容易写起来没完没了，讲的浅了又感觉大部分人都会，多此一举。所以目前的篇幅是我暂时感觉比较恰当的，既能对 vm 的关键知识进行介绍，逆向分析已经够用，又不至于陷入实现的细节中去，无法自拔。以后如果有更好的对 vm 的介绍思路，可能我会继续发文进行补充。

    无论如何，此刻，我们关于 vm 的知识基础已经打好了。下面我们就要基于这些知识，去深入分析企鹅滑块具体做了哪些检测了。

4 加密参数分析
========

    终于终于，又到了熟悉的逆向环节，

    加密参数我们就分析 collect 这个生成逻辑极其复杂，还被 vm 保护的参数就好了。

    至于 vData，如果补好的环境能生成正确的 collect 参数，那基本就能复用该环境直接生成 vData 参数，就不多提了。

    其他的参数都比较好找，也不多说。

    对哪个参数有疑问的话可以去群里讨论。

4.1 目标网址
--------

    目标网址：`aHR0cHM6Ly93d3cud2VnYW1lLmNvbS5jbi8=`，登录页面，输入账号密码点击登录时就会弹出验证码。

4.2 collect 参数分析
----------------

### 4.2.1 逆向思路确定

    逆向思路：补环境。

    效果：放入官网获取的 tdc.js，可直接获取与浏览器完全一致的 collect 参数

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojgSzSoYfuxkspQFmEMTknHsjQMSvZcCuF2YMrHnFRnO33kfcuIKicSIw/640?wx_fmt=png)

优点：

1.  不用考虑任何魔改算法，只关注补环境即可，哪怕逻辑里用了 rsa+aes+tea + 等等等的各种算法，全都不用管，只要环境补全，就能直接生成想要的参数，就是这么省心。
    
2.  另外我们算一下对抗成本：
    
    2.1 加解密算法是开发人员可以复制过来魔改下就用的，算法的对抗意味着一个人对抗世界所有已知的算法。
    
    2.2 环境检测只能是开发人员自己探索，毕竟开发人员要根据自己设置的检测点做校验。这么想来，设置几十个环境检测点就很难为开发人员了，就算开发人员肝到头秃，探索了几百个校验点，那又如何呢？逆向补环境可比正向探索环境校验点容易多了。而且不同的反爬产品，用的环境还大概率相同。环境的对抗相当于一个人对抗有限的浏览器环境，还能根据积累越补越容易。
    
3.  补环境天然适合反爬代码时常变形的场景，无论变量名如何变，函数排序如何变，只要检测的环境不变就可以直接无视。如果环境也变，那多补一些 js 文件样本，总能补出一个该反爬产品的环境全集来。
    

疑点：有人会认为补环境效率慢，但那是用 jsdom 之类的库造成的，时间浪费在了第三方库复杂的初始化上。如果所有的环境都是自己手动补的，那代码运行效率和扣算法几乎没有区别。

    经过如上分析，想必补环境和还原算法该选择哪个，你心里已经有了答案。

### 4.2.2 生成位置定位

随便拉一下滑块，可以看到验证请求里有 collect 参数。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojJHTyoctv8oh4Y4qlScEVfyu8iczjE2FltqbBQanqfOvdyGrtKtOJghA/640?wx_fmt=png)

追一下这个请求的堆栈

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojRe3KqQhr58UEFBM214UWibvPJ9Ojao2XPkmkmlQB1WKK2WTEg8I7OMA/640?wx_fmt=png)  

    可以看到 collect 参数是这里的 R 函数生成的，打个断点，再进入 R 函数里看下。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojqajSIQfR3v0G2icNyShA2bAEFpcs0WMdm7IcZFpzMjiat6NxdAHUGcxA/640?wx_fmt=png)  

    发现跟踪到了 a.getData(!0) 处，我们继续进入 a.getData 函数。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojz0rTdn4nEqZaPetn7uiblRpxVIVpPxhNGpvVpHkFXTe7tM2Qts6AezQ/640?wx_fmt=png)  

    发现就进入了 tdc.js 文件中，该文件中就是我们上文分析的那个 jsvm 的虚拟机。

    定位到此结束，我们开始研究下 tdc.js 有哪些逻辑

### 4.2.3 tdc.js 的初始化

    提前先明确一个重要的地方：我们要有函数的概念来看待虚拟机指令。

    下面开始看逻辑:

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojxQKZrXhBSllGTloicY3DZn2d19eicZsrLiaEPRDvPwGzibWUbNvZ0SQCuA/640?wx_fmt=png)  

    如上图所示，从 index=0 的位置开始，其实就是进入了一个函数，然后初始化的步骤就是新建了一个空数组，一直向数组中添加函数。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojMjxnmiaHqXd5vbHXIvkSO5X2j3qU5JicP2hGM4cwuo3yXyF6hSTMXR4A/640?wx_fmt=png)  

    向数组里添加了几十个函数之后，开始调用`var2`所调用的函数，继续进行初始化操作。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojapZYicDX0GGwKkIGpG1pGR0VPa08oXDWrxQBm1ymJmIjvhoFOEjvKwg/640?wx_fmt=png)  

    可以看到这次是新建了一个对象，向上绑定属性，然后继续进入 var45 定义的函数。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojiaSnSolicyybg0VQo5lTThTR3Q4W4hMiaJMD9qJ5wzEraVicQn932ZaStg/640?wx_fmt=png)

    如上图所示，var45 函数的作用就是检测当前函数是不是已经在`fun_20_5[0]`中存在了，如果已存在，直接返回该函数，如果未存在，就开始进行初始化，初始化过程中，很多函数的调用都会先进入这个函数，这个逻辑和 webpack 的分发器有些像，或许，这部分的代码就是 webpack 转换过来的。

    我反编译后的代码只是作用和源码一致，具体的结构肯定是有差别的。

    ok，上图中进入到了函数初始化的分支中，我们继续跟着代码深入。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojFWPEF8KibT7PribkLnIcoWzrCXJ1WXTmOKkicT82lvAsssu6eGbsFznPA/640?wx_fmt=png)

    该函数向 window 上绑定了 TDC 对象，这里就已经有我们要找的那个 getData 函数了。而 getData 函数是从`var30.apply(window, [3]);`这个函数调用返回的对象上获取的，继续向里跟。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojjPMUwOAkia2H4L7kckUuiadz34dvVUUZ50e0ahMSJ5vRW4KggSzCu8nw/640?wx_fmt=png)  

    上图中有两个小监测点，然后就是通过几个函数调用返回了一些函数。

    这里主要关注一下标号三的位置，它初始化了与指纹相关的 38 个对象，其中一些对象在本处初始化时就获取完指纹了，然后通过闭包传递给后文调用。还有一些对象在后文调用的时候再获取指纹，其中详情后文再说。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojXZMfQaR1rcfzlRyBof8VANxHx1YzC3ZNcPEum9wfRMHrz8MoV2RbCA/640?wx_fmt=png)  

    该函数最后构造了一个有 mSet,mInit 等属性的对象，然后函数结束。

回退到上一级继续看逻辑。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojUwIpmkZGCHTDSo9bG7jxQnCn7xpBWDZHkZ3YhBW0RQVTQ7TkybzUJw/640?wx_fmt=png)  

    本处，开始进入 tdc.js 文件初始化的最后一个函数：mInit，继续跟进去

    这个函数有两部分操作:

    第一部分，如上图所示，遍历一个有 38 个对象的数组，将有 on 属性的，有 reset 属性的和有 get 属性的对象各自放在三个数组中。如果一个对象同时有这三个属性，那该对象同时放入三个数组。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojT2l7ic4yx9BaXkTjhS9iczxmcwByOpB9ib0e1aib9lX7sfZZZQlJqLr2Cg/640?wx_fmt=png)  

    第二部分：遍历存 on 属性元素的那个数组，依次调用其中的 on 函数。这三个 on 函数的作用是绑定了些事件，就不再进去看了。

    到此，tdc.js 文件初始化阶段结束。

    整个初始化阶段，最主要的就是那 38 个对象，对应了 38 个获取浏览器指纹的函数。

### 4.2.4 window.TDC 对象介绍

    初始化之后，window 上被绑定了 TDC 对象。TDC 对象里有四个函数。

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojh6WymULcQ9wX21Cz9CI5egic8ibFxD0rQxZibKsdwjSdvcHUXic7TnvGSQ/640?wx_fmt=png)  

    我们对该对象及这四个函数做监控，就会发现 TDC 对象被操作的步骤如下：

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmoj2fn5p3oVHJGqB36ul2m2UlasrLEnFEAicDNUtPO4aOzic7I4S50icsDyQ/640?wx_fmt=png)  

    调用完这四次 setData 之后，就可以调用 getData 函数获取 collect 参数了。

    下面我们进入 getData 函数，看一下内部逻辑及逆向思路

### 4.2.5 window.TDC.getData() 逻辑分析

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojZg4gO1icjsh3FyJ2WdKtSaXSRy8IvZFvkiaquLjiaTZvnVaRLnZ01IbAQ/640?wx_fmt=png)

    可以看到最开始有两个小检测，然后执行最终的 collect 参数由两部分拼接而成。

    第一部分：执行`var165.apply(window, []);`返回的对象里的 cd 参数，所有的环境检测都在该部分。

    第二部分：之前我们调用那四次 setData 函数传进来的内容组成了对象，被加密后得到，无环境检测，可无视。

    所以我们只看下第一部分的内容，下面进入 var165 函数：

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojMkMRMicWPZ5rKibtNZ6bLKVOHsiaPyjhM2Yuibm1ibgQQHvop6bNgpIJAew/640?wx_fmt=png)

    这个函数很清晰，循环那 38 个对象，依次调用每个元素的 get 方法，获取指纹明文，那我们看一下这 38 个指纹明文各自是什么：

```
0 ： [ 2 ] 1 ： [ 1 ]2 ： [ 1080 ]3 ： [ '""', null, 1 ]4 ： [ 4 ]5 ： [ '98k' ]6 ： [ 1920 ]7 ： [  'ANGLE (NVIDIA, NVIDIA GeForce GTX ***********************************)' ]8 ： [ '["zh-CN","zh-HK","en"]', null, 1 ]9 ： [ '+08' ]10 ： [ 'https://xui.ptlogin2.qq.com/?rand=***********************************' ]11 ： [ 0 ]12 ： [ 'Google Inc. (NVIDIA)' ]13 ： [ '' ]14 ： [ '[300,230]', null, 1 ]15 ： [ 0 ]16 ： [ 16 ]17 ： [ 16 ]18 ： [ '1920-1080-1040-24-*-*-|-*' ]19 ： [ 'Win32' ]20 ： [ 'https://t.captcha.qq.com/cap_union_new_show?rand=***********************************' ]21 ： [  'G4DiPeJDADK46nKuigdUgAAASwAAA3Xn7gFX08P4nxW8gJAAAoZAACWCAYAAAs0MThdAANSUhEgABJRU5ErkJggg==']22 ： [  'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/******** Safari/********']23 ： [ '703312397,1637248154', null, 1 ]24 ： [ '210.193.226.203' ]25 ： [ 511 ]26 ： [ 'ca55wIRRQQnXuVgcsc21oTk+kfam3efV84x9G0LvOMA+oIm/je008N4CFL/7kTxzOtCM3unzmFNMPWKCmsJuUe3QvHZPlm4jV3jECmJhfKb4OdzD+Ye+EWYEhf5UYgah2mGc2TvOnyTofpm/je008N4BKxbv8uExD5CM3unzmFNMOdHA8RUoFx+qvaLqSNJRKeg0ePoHt4eFSom2FHbgSK3cHMUGZZR28nEwhpJX8a2/slIWsGc2TvOnyTofpDg0UfBDCraOKGoh9WntJyOUO8iMMBhBZ0cK9pbv09Ar69v8dDQRjGyL1WIhe3vub/qgxBJuW5+g0ePoHt4eFUaeivzsHyK64BoTkkn6DDf1bCp3XsIMeQRDHaRa35zF+kfam1udrLeiDXdCfA1EGDCgVS6ZZR28nEwhpJX8a2/slIWsGc2TvOnyTofpm/je008N4Cy7hXcc04xmCM3unzmFNMO4BQu+TK/scb6OqCODNdXbI2CEPmouQria5g/TpZnCSF/lRiBqHaZWougNRUiq68LdjMU1bBcAIUv/uRPHM60qwc4eTEISYXw/JERW/pC1mrZUkxXLSP/S7wAysWvOeB+2R2NxvII8VCCGhkHLwFf+2cprfJlKSP7Zymt8mUpI/tnKa3yZSkiQK2cZY+WZtria5g/TpZnCuJrmD9OlmcK4muYP06WZwria5g/TpZnCezl06GzlWT87cZoDds7oGyXcLTUncwYT',  null,  2]27 ： [ '1', null, 1 ]28 ： [ '5335919002380139475' ]29 ： [ 24 ]30 ： [ '[]', null, 1 ]31 ： [ 'iframe' ]32 ： [ 1 ]33 ： [ 1647888831 ]34 ： [ 'UTF-8' ]35 ： [ 1647888831 ]36 ： [ 744384758 ]37 ： [ 0.9723439840980835 ]
```

    这里分享一个断点小技巧:

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojHooOz2wPm0NEWderUic2QDerACR5GFcWm1AbeibqWIGPfxHgteo4jtDA/640?wx_fmt=png)  

    该处代码改造一下，就能依次调试那 38 个函数补环境了。

    逻辑就分享到这把，整体流程已经很明确了，只剩下 38 个关卡需要逐个击破，就留待各位解决了~~

### 4.2.6 一个取巧的解决办法

    两个前提：

    1，每次获取 tdc.js 文件执行，可以发现这 38 个环境指纹明文只是位置在变，值除了其中一两个，大部分是不变的。

    2，jsdom 补环境是很容易的，不过 jsdom 的环境生成的环境指纹明文不能通过校验

    那有了这两个前提，你是不是可以构造一个错误明文与正确明文的映射呢？然后在 jsdom 的环境下，在我分享的那个断点处将错误明文替换成正确明文，这样生成的 collect 就可以通过校验了，而且有了 jsdom 的帮助，补环境也十分简单~

    这个方法在量特别大的情况下会不会被风控我不清楚，但是少量的打码肯定是够用了，具体支持的量级各位可以测一测。

    jsdom 补环境如下即可（本截图来源见结尾）：

![](https://mmbiz.qpic.cn/mmbiz_png/huXBGBmwjMU1sCHTl8S2pmXoTw8HTmojB6arcSAVtUDU8Agr2vic3ibTFk7Nd2W0yOj50EGD9yMpOGgficubUaXiaQ/640?wx_fmt=png)img

5 结尾
====

1.  因为每篇都有很多的东西要写，所以限于篇幅，每篇的内容都尽量不重复，比如`vData`参数需要在`XMLHttpRequest`的`send`函数中获取，再如补环境的细节，这些是在《[某音新版本逻辑分析](https://mp.weixin.qq.com/s?__biz=MzkwMTE3NzI5NA==&mid=2247483839&idx=1&sn=17a8b2b3a2e3a588587c8d19ffba8076&scene=21#wechat_redirect)》文章里，又比如补环境时的鼠标轨迹处理思路在《[DX 验证码全过程详解](https://mp.weixin.qq.com/s?__biz=MzkwMTE3NzI5NA==&mid=2247483922&idx=1&sn=31f5ae80e32a2a176340d26f729c2586&scene=21#wechat_redirect)》文章中。所以如果有不理解的地方，可以翻下我以前的文章，或者在群里问一下。
    
2.  可能不知什么时候该滑块的检测逻辑就改版了，希望本文对 jsvm 分析的思路能帮助各位读者更轻松的面对那些未知的反爬产品。
    
3.  补环境的参考链接我贴一下：
    

1.  《[某音新版本逻辑分析](https://mp.weixin.qq.com/s?__biz=MzkwMTE3NzI5NA==&mid=2247483839&idx=1&sn=17a8b2b3a2e3a588587c8d19ffba8076&scene=21#wechat_redirect)》：我之前的一篇介绍补环境文章.
    
2.  《[JS 逆向之字节系列某量算数 jsvmp 算法分析及深度还原](https://mp.weixin.qq.com/s?__biz=Mzg2NDY4Njc4MQ==&mid=2247484192&idx=1&sn=1db676ecaff6aca791782fefacecf829&scene=21#wechat_redirect)》：该文章补环境部分和我补环境的思路一致，对同一份文件在浏览器和 node 环境各自输出的日志对比，来把控细节，判断补的环境是否正确.
    
3.  《某鹅滑块验证码破解笔记 - https://www.52pojie.cn/thread-1521480-1-1.html》: 这是上面 jsdom 环境的出处，这篇文章对 vData 的介绍可以看一下。
    
      
    
    ![](https://mmbiz.qpic.cn/mmbiz_jpg/huXBGBmwjMWq6U8cJFU8fZvCKoG7h4xNuJqlp0ia8hUK1RkriaQ8OE8jncEe7yhenEhyHV3nvZLRUk2sOgCfFIjQ/640?wx_fmt=jpeg)  
    
      
    
      
    

付费区放一下参考资料吧，总要给看客们一个打赏的机会，哈哈~ 

各位需要自取。