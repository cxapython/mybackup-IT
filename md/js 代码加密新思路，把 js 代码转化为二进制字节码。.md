> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.fairysoftware.com](https://www.fairysoftware.com/js_er_jing_zhi.html)

> js 加密，js 代码编译成机器码，js 代码加密转化为汇编指令，js 虚拟机加密。js 解释器。汇编指令，js 代码转为二进制字节码。js 模拟器

**js 代码加密新思路，把 js 代码转化为二进制字节码。**

本文次讨论把 js 代码加密保护方法，本文中将把 js 代码转化为汇编指令，再最终转化为二制作字节码。文中会讨论技术可行性，实现方染，并给出例程，最后总结方案的可用行，优劣等等。

很多时候，我们都会觉得混淆脚本程序是件困难的事，效果远不及传统程序的混淆力度。毕竟，脚本的初衷就是简单易用。诸多先天不足的特征，使得混淆难以深入实施。

然而从理论上这似乎也说不通，只要是图灵完备的语言，解决问题的能力都是相同的。举个最简单的例子，网上有使用 JavaScript 实现的 x86 模拟器，我们抛开性能不说，单论功能，它和本地系统是一样的。因此使用传统工具混淆的程序，同样也是能在浏览器中运行的！

当然，这个代价不免有些太大。为了保护一段逻辑，还得加载一个庞大的模拟器和操作系统，显然是难以接受的。但是这个思路还是很有意义的 —— 将需要保护的代码逻辑，放入模拟器中执行。

事实上类似的方案也早已存在，例如大名鼎鼎的 VMProtect。在浏览器端同样也有应用的案例，例如 Google 曾经开发的 reCaptcha 验证系统，也用到了模拟器来保护重要逻辑。

如何将前端脚本程序，变成可被模拟器运行的指令？我们从最简单的案例开始讲解。  
字节码

和传统的编译型程序不同，脚本程序始终是带语法的文本代码。如何将一段充满各种可读单词的代码，尽可能多得使用数字来描述？例如这段代码：

var el = document.createElement('script');  
el.text = 'alert(123)';  
document.body.appendChild(el);

其中就有变量名 el、字符串'script'、全局变量 document、属性 body 等可读单词。

对于变量名来说，普通的压缩工具就能很好处理，变成诸如 a、b、c 这样的短名字；但是字符串和属性，又该如何处理？

熟悉 JS 的都知道 obj.key 和 obj['key'] 是相等的。而且全局变量都是 window 下的属性。因此，我们可把全局变量和属性都变成字符串的形式：

var el = window['document']['createElement']('script');  
el['text'] = 'alert(123)';  
window['document']['body']['appendChild'](el);

这时，整个代码中除了 window 之外，都是字符串了。

既然我们的目标是将代码数字化，那就将数字以外的常量都提取出来，放到一个单独的数组里：

var MEM = [  
window, 'document', 'createElement', 'script',  
'text', 'alert(123)', 'body', 'appendChild'  
];

这样，就可以用 MEM[数字] 代替一切了：

var el = MEM[0][ MEM[1] ][ MEM[2] ]( MEM[3] );  
el[MEM[4] ] = MEM[5];  
MEM[0][ MEM[1] ][ MEM[6] ][ MEM[7] ](el);

看起来有些眼花缭乱了吧。不过这只是对常量进行替换，语法仍然存在，因此还是能推测出大致的逻辑。不少基于语法树的混淆工具，大多就到这一步。

下面我们进一步，将语法展开：

var A, X, Y, Z

A = MEM[0] // window  
X = MEM[1] // 'document'  
X = A[X] // X = window['document']

A = MEM[2] // 'createElement'  
Y = MEM[3] // 'script'  
A = X[A](Y) // A = document['createElement']('script')

Y = MEM[4] // 'text'  
Z = MEM[5] // 'alert(123)'  
A[Y] = Z // A['text'] = 'alert(123)'

Y = MEM[6] // 'body'  
X = X[Y] // X = document['body']

Y = MEM[7] // 'appendChild'  
X[Y](A) // body['appendChild'](A)

由于失去了语法，因此需要一些临时变量来保存中间值，这里使用 A、X、Y、Z 四个变量来暂存。

这时的每一步，都是一个基本操作。我们到了脚本层面最低级的形式。（可以试着粘到控制台，仍能正常运行~ 或者点击 jsfiddle.net/qLtojr5z/ 演示）

观察上述代码，其中有大量相似操作，我们尝试用代号来进行替换。例如读取 MEM[i] 操作，使用 LDR（Load Reg）来描述：

r = MEM[i] => LDR r, i

同样的，属性读写操作，也进行类似替换：

r1 = r2[r3] => GET r1, r2, r3  
r1[r2] = r3 => SET r1, r2, r3

对于方法调用操作，暂且用 CAL 来表示参数正好为 1 个的情况，并且返回值统一存放在 A 中：

A = r1[r2](r3) => CAL r1, r2, r3

现在，我们用这个几个虚拟代号，重新描述上述逻辑：

LDR A, 0  
LDR X, 1  
GET X, A, X

LDR A, 2  
LDR Y, 3  
CAL X, A, Y

LDR Y, 4  
LDR Z, 5  
SET A, Y, Z

LDR Y, 6  
GET X, X, Y

LDR Y, 7  
CAL X, Y, A

这是不是有一种汇编指令的感觉！之后的处理过程自然就很明确了，我们将这些可读的文本汇编码，转换成二进制字节码。

例如用 1 代表 LDR 指令，2 代表 GET 指令。。。同样的，暂存器也可以用数字表示，例如用 0 代表 A ，1 代表 X。。。  
汇编码 字节码  
LDR A, 5 01 00 00 05  
GET X, Y, Z 02 01 02 03  
SET Z, Y, X 03 03 02 01  
... ...

于是之前那段程序逻辑，最终就能用纯数字表示了：

01 00 00 00 01 01 00 01 02 01 00 01 01 00 00 02  
01 02 00 03 04 01 00 02 01 02 00 04 01 03 00 05  
03 00 02 03 01 02 00 06 02 01 01 02 01 02 00 07  
04 01 02 00

注意，这部分只是程序逻辑的指令数据，那些字符串等常量数据并不在此，需要另外存储。  
模拟器

我们的字节码在浏览器看来，只是一堆数据而已，并无实际意义。因此需要一个模拟器，来解释执行这些数据。

模拟器听起来高大上，其实原理是非常简单的 —— 根据指令数据，做相应操作而已。例如遇到 1，执行读取存储操作；遇到 2，执行访问属性操作。。。

REG = []; // 暂存器

do {  
opcode = MEM[pc++];

switch (opcode) {  
case 1: // LDR  
...  
case 2: // GET  
...  
case 3: // SET  
r1 = MEM[pc++];  
r2 = MEM[pc++];  
r3 = MEM[pc++];

obj = REG[r1];  
key = REG[r2];  
val = REG[r3];

obj[key] = val;  
...  
}  
} while (...)

我们将字节码当做二进制数据加载到存储中，然后使用一个计数器，指向当前指令所在的存储位置，暂且称之 pc（program counter）。每执行一条指令，pc 进行相应增加，指向下一条指令。周而复始。

这样，一个模拟器的雏形就出现了。

我们可以添加更多的指令，例如算数、位运算等等，使模拟器变得更完善。同一个指令，也可以有多种模式。例如 LDR 指令，地址可以是立即数、暂存器，或是 暂存器 + 立即数、暂存器 + 暂存器 等多种模式，方便各种寻址操作。

指令越丰富，相应的逻辑实现就越简单。相反，指令越少，同样的操作就需要多个指令组合才能完成。一个极端的例子就是 Brainfuck 程序，它只提供极少的指令，因此即便非常简单的功能，也需要大量冗长的组合才能完成。

当然，指令越丰富模拟器也会越庞大，因此得根据实际需求折中考虑。  
跳转指令

程序不可能永远都是顺着执行的，否则一下就执行完了。因此还需跳转操作，可反复执行先前指令。最简单的跳转，就是无条件跳转，我们暂且用 JMP（Jump）来表示：

Label:  
...  
JMP Label

和传统语言 BASIC 或 C 的 goto 一样，在汇编文本层面，可以使用 label 作为跳转的目标。当然 label 只是个标记而已，并不存在于最终的字节码中。最终存储的，只是目标指令所在的位置。

因此当模拟器解释 JMP 指令时，仅仅是修改 pc 而已：

...  
switch (opcode) {  
...  
case OP_JMP:  
...  
pc = r;  
...  
}

有跳转指令，我们就可以灵活操控流程，完全不必按照 JS 那死板的流程控制了。

事实上，这个指令集和 JS 源码已经毫无关系。我们完全可以使用其他语言，编译出相应的虚拟指令。最终的字节码，显然也是无法还原出 语义化 的 JS 代码的。  
分支指令

除了无条件跳转，还有带条件的。例如这段代码：

var str = prompt('password');

if (str == 'hello') {  
alert('OK');  
} else {  
alert('Fail');  
}

按照先前的方式，我们将其转换成最低级的 JS 代码：

var MEM = [window, 'prompt', 'password', 'hello', 'alert', 'OK', 'Fail']  
var A, X, Y, Z

X = MEM[0] // X = window  
A = MEM[1]  
Y = MEM[2]  
A = X[A](Y) // A = window['prompt']('password')

Y = MEM[3] // Y = 'hello'

if (A == Y)  
A = MEM[5] // A = 'OK'  
else  
A = MEM[6] // A = 'Fail'

Y = MEM[4]  
X[Y](A) // window['alert'](A)

相比之前，现在多了判断操作。因此，我们再添加一个带条件的跳转指令。例如当 r1 != r2 时执行跳转：

JNE r1, r2, label

这样，我们就能和 JMP 指令组合，来表达上述逻辑了：

... ; 注释  
LDR Y, 3 ; Y = 'hello'

JNE A, Y, L_ELSE ; if (A != Y) goto L_ELSE

LDR A, 5 ; A = 'OK'  
JMP L_END  
L_ELSE:  
LDR A, 6 ; A = 'Fail'  
L_END:  
...  
CAL X, Y, A ; alert(A)

有了 != 判断，自然也可实现 == 判断。不过为了方便使用，我们可提供更丰富的分支操作。例如 JS 中的各种判断：  
跳转指令 条件 备注  
JE r1 == r2 Jump if Equal  
JNE r1 != r2 Jump if Not Equal  
JES r1 === r2 Jump if Equal Strict  
JNES r1 !== r2 Jump if Not Equal Strict  
JG r1 > r2 Jump if Greater  
JGE r1 >= r2 Jump if Greater or Equal  
JL r1 < r2 Jump if Less  
JLE r1 <= r2 Jump if Less or Equal  
JIN r1 in r2 Jump if IN  
JINSOF r1 instanceof r2 Jump if INStanceOF

甚至对于一些常见情况，还可再进一步封装：  
跳转指令 条件  
JTRUE r1 === true  
JFALSE r1 === false  
JZERO r1 === 0  
JNULL r1 === null  
JUNDEF r1 === undefined  
... ...

不过，有时我们只想判断，未必要跳转。例如：

isOK = (stat == 200);

对于这种情况，使用跳转指令也能满足，只是显得略为累赘。如果想更精简，则可添加纯粹的判断指令，例如：

A = (r1 != r2) => TEST_NE r1, r2  
A = (r1 in r2) => TEST_IN r1, r2  
...

当然，其本质都是一样的。  
JS 操作

既然我们的模拟器是用于浏览器环境，显然应该提供完善的 JS/DOM 操作。因此我们再添加几个脚本相关的指令，例如：  
指令 功能 备注  
CONCAT r1, r2, r3 r1 = r2 + r3 字符拼接  
OBJECT r1 r1 = {} 创建对象  
TYPEOF r1, r2 r1 = typeof r2 typeof  
DELETE r1, r2 delete r1[r2] delete  
NEWCAL r1, ... A = new r1(...) new

这里提一下 JS 的 + 操作符：它既可以用于数字加法，也可用于字符串拼接。为了不和 ADD 指令混在一起，我们可单独提供一个字符串拼接的指令。

现在来思考一个问题：如何提供回调函数？

从理论上说，我们可实现一个完全兼容 JS 的字节码模拟器，但事实上这是相当复杂的。JS 有众多灵活的特征，例如闭包、with、eval 等等，要实现这些，相当于得重新造一个 JS 引擎，显然是不现实的。

因此，我们只需提供一些常用的操作就可以了。闭包之类的特性，就可以不考虑了。不过回调函数还是需要支持的，例如这段代码：

button.onclick = function() { ...};

我们可设计一个指令，将相应的 label 封装成一个函数对象：

FUN r, label ; r = makeCallback(...)

label:  
...

这样，就能提供给 DOM 使用了：

L_CLICK:  
...

L_MAIN:  
... ; A = button, X = 'onclick'

FUNC Y, L_CLICK ; Y = makeCallback(...)  
SET A, X, Y ; A['onclick'] = Y

至于封装的细节，大致就这样：

function makeCallback(pc) {  
return function() {  
return vm.run(pc);  
};  
}

在回调函数里，让模拟器从 pc 的位置开始解释，这样就让某些指令异步执行了。

这里简单的演示一下。例如这个回调函数：

var i = 0;

function render() {  
txt.value = i++;  
if (i <= 255) {  
requestAnimationFrame(render);  
}  
}  
render();

将其转换成字节码：

0000 05 03 00 00 MOV Z, 0  
0004 01 00 00 00 L_TIMER: LDR A, 0  
0008 01 01 00 01 LDR X, 1  
000C 02 02 00 01 GET Y, A, X  
0010 01 01 00 02 LDR X, 2  
0014 03 02 01 03 SET Y, X, Z  
0018 06 03 00 00 INC Z  
001C 05 01 00 ff MOV X, 255  
0020 07 03 01 10 JG Z, X, L_END  
0024 01 01 00 03 LDR X, 3  
0028 08 02 00 04 FUN Y, L_TIMER  
002C 04 00 01 02 CAL A, X, Y  
0030 00 00 00 00 L_END: BRK

在脚本层面上还有个特殊流程，那就是错误捕获。例如这样的 JS 逻辑：

try {  
// safe  
} catch (...) {  
// handler  
}

这使用指令并不难描述。我们可定义两个指令，分别用于捕获的开启和关闭：

CATCH L_ERR  
... ; safe  
...  
UNCATCH  
...  
L_ERR:  
... ; handler

当模拟器遇到 CATCH 指令时，使用 try 解释后续指令，若有错误发生，则进入 label 的位置；当遇到 UNCATCH 指令时，则退出当前递归，返回上一层的捕获：

function run(...) {  
...  
case OP_CATCH:  
try {  
run(...); // 安全模式 递归  
} catch (e) {  
pc = ... // 错误处理流程  
}  
...  
case OP_UNCATCH:  
return;

这样，就能放心地执行一些可能报错的操作了。

类似的逻辑实现还有很多，这里就不详细介绍了。关于模拟器的基本原理简介，就到此为止。不过我们的目标并非只是为了实现一个模拟器，而是利用模拟器来保护代码逻辑。  
逻辑保护

相比过去那些基于 AST（抽象语法树）的混淆方案，使用模拟器可以实施得更深入。大致可以在这几点上对抗：

编译过程

指令编码

指令混淆

编译过程

从源程序到字节码，需要一个编译的过程。这个过程本身就有一定的混淆效果，例如一些优化工作会对逻辑进行调整。和传统的编译型语言一样，这个过程是不可逆的。反编译的代码，是很难回到原始语义的。（不知大家是否见过那些自称能把 exe 程序还原成 c 代码的工具，结果当然是惨不忍睹）

由于模拟器难以完全兼容 JS 所有的特性，因此不能直接用于现有的脚本。需混淆的代码必须遵循一定的规范编写，例如不能使用 with、eval 等高级特性。所以，不推荐对整个程序都进行混淆，而是只针对一些核心逻辑。

如果核心部分只是算法，甚至完全可以不用 JS 编写，而是选择 C 这种更适合计算的语言。我们可以使用 clang 编译出 LLVM 中间码，然后开发一个 LLVM Backend 插件，将中间码编译成我们模拟器的目标指令。

LLVM 是个非常有意义的系统。它不仅可用于程序的优化，同样也可实现程序的「劣化」，让逻辑变得更乱更难分析。例如在计算过程中，插入大量的中间步骤，干扰逻辑的分析。  
指令编码

因为模拟器的指令是我们自创的，所以对方在逆向分析之前，必须了解指令的编码格式，才能成功反编译。因此，在编码上又可以进行一些对抗。

传统的指令编码大多都有规律，因为那是从解码复杂度以及性能上考虑。例如：

switch (opcode) {  
...  
case OP_SET:  
r1 = MEM[pc++];  
r2 = MEM[pc++];  
r3 = MEM[pc++];  
...

这么简单明了的解码过程，显然是很容易分析的。而我们最终目标是混淆，性能并非是第一位。因此可使出各种千奇百怪的编码格式，来增加解码的复杂度。

例如，使用各种逻辑位运算，并且不同的指令格式也各不相同，没有任何规律。在性能损失可接受的范围内，将解码过程变得极其复杂，使分析变得更困难。

a = MEM[pc++]  
b = MEM[pc++]  
if (a & 128)  
if (a & 64) // OP_SET  
r1 = (a>> 4) & 16  
r2 = (b & 16) ^ ~r1  
r3 = (b>> 4 & 16) ^ r1  
...

当然再复杂的格式也有破解的时候。因此我们不能永远使用一种格式，而必须不定期的进行升级。不过，每次升级都得重新设计一遍，会不会很麻烦？

如果编码格式由人工制定，那显然是很麻烦的。因此必须借助工具，自动化生成「编码器」和「解释器」。我们只需设计一些策略就可以了，让工具将这些套路随机组合，生成千奇百怪的格式。最终格式是什么样的，我们自己都不需要了解:)

总之，用最简单的正向设计达到最困难的逆向分析，这就符合对抗的意义了。  
指令混淆

指令本身也是内存中的数据。因此和普通数据一样，指令数据也能被修改，例如当前指令可以修改即将执行的下一条指令，这样就可以在运行时动态调整程序行为了。

利用这个特征，我们可对程序的大部分指令事先进行加密，然后在运行时再逐步解密。假如程序有 a、b、c、d 几个部分，我们事先将 b、c、d 部分进行简单加密，只保留明文的 a 部分。

当程序执行 a 部分时，将 b 部分的二进制数据进行解密，还原出明文指令；执行到 b 部分时，还原 c 部分，同时再将 a 部分加密回去。。。这样变执行边释放，就能避免一出来就能看到所有指令，从而增加分析成本。

另外，在字节码的层面上，跳转是以字节为单位的，因此可跳到某个指令的中间：

位置 字节码 汇编码  
0000 02 01 02 03 GET X, Y, Z  
0004 05 00 01 JMP 0001

这样就能执行 01 02 03 05 这串字节码，即 LDR Y, 0x0305 了。利用这个方法，就可以将一些指令伪装起来，实现花指令的效果。

类似的对抗思路还有很多，这里就不详细讨论了。事实上，这些大多是传统程序的混淆方案，之所以能用到 JS 上，得益于模拟器消除了平台间的差距，从而使得前端脚本也能享受到前人积累的对抗技术，完全不必自创一些看似炫酷实则毫无意义的混淆方案。

总结：  
这种方式最大的缺陷是，要实现一个完整的模拟器很困难，需要将 js 语法全部二进制化方案，编码工作量很大。而且稳定性也是个值的顾虑的地方，还好考虑到跟着 js 标准的发展而发展，总的来说是个很复杂、长期、艰苦的工作。  
用于 js 加密保护，目前的现实意义，在于：对于部分核心代码可以进行部分二进制化，但无法大量进行。如果要进行完整代码的 js 混淆加密，靠谱稳妥的方案还是得使用类似 [jshaman](http://www.jshaman.com/) 这类成熟的 js 加密平台。