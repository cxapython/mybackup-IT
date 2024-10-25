> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.resourch.com](https://www.resourch.com/archives/104.html)

> 抖音 jsvmp 逆向教程

ast 预反混淆
--------

以抖音为例，首先获取 webmsdk.js，可以看到大约一共 9600 行，这为我们的逆向分析工作增添了不少难度，因此我们的首要工作是将代码反混淆后再进行分析

![](https://www.resourch.com/usr/typora/89967291467d41a38b32c5e74f4f6abc.png)

通过 ast 解混淆处理后，代码量削去了一半，并且大部分字符串也被还原，这一步为基础内容，不进行详细讲解

![](https://www.resourch.com/usr/typora/a4efcba3b76746a49925c0f55b905d61.png)

补上缺失的环境

![](https://www.resourch.com/usr/typora/fde3a6780d8d4faba56e301f840d66e2.png)

处理一下 canvas 指纹，注意，canvas 指纹是和硬件关联的，通过海量的收集真实设备从而达到判断哪些设备是伪造的。一般情况下随机生成一个也是能生产参数的，但是不一定能够通过服务器校验。而抖音这里在服务端没有做校验，我们可以放心使用

![](https://www.resourch.com/usr/typora/b85b4b3b190f428facda7d8ea89d6774.png)

进一步处理，在解决掉环境参数和 canvas 后，导出我们需要的目标函数，实际上已经能够获取到我们所需的加密参数了

![](https://www.resourch.com/usr/typora/dcb5a4ef98df4caf9a7c2bc33caee05b.png)

为什么我们一定要先 ast 进行还原，然后补环境拿到参数呢，直接分析不好吗？因为这样生成的代码可以保证我们能够在各种环境中正常地生成参数，只有这样我们才能更好地利用 ide 的调试工具，如果使用浏览器的调试工具，那么会大大降低逆向效率。当然，如果你不想学习 vmp 反编译的话，到这里也可以结束了。

借助 ide 对函数结构分析
--------------

仔细观察一下，按照我们上文的理解，vmp 在执行时一定会有大段的字节码传入，因此这是非常显著的一种特征。注意图中我标注的 vmpstr，整个参数的的生成，是通过`_0x5a8f25`这个函数进行的，而其中就有我们需要的 vmp 字节码

![](https://www.resourch.com/usr/typora/1130102cf1294e27ad16aa012ab5328a.png)

直接跟栈进入，找到`_$webrt_1668687510`这个函数，搜索一下函数名，能看到有大量传入字节码的操作，那么这个就是我们的 vmp 主入口了

![](https://www.resourch.com/usr/typora/2c1f99f105a04b399ea015e3526c12b5.png)

不幸的是，整个入口函数足足有 900 多行，即使在 ast 还原过后，其中也充斥着大量的混淆代码和看不懂的标识符。这时万不要在原本的打断代码中盲目硬分析，我们可以将这段代码复制出来，粘贴到新的文件中，再次进行分析

这时候 ide 的作用就显现了，利用 vscode 的大纲功能，我们可以看到这个函数是由 7 个闭包函数和大量的变量组成的，而如果利用浏览器，再未反混淆的情况下，仅仅这个函数就有 3000 行，我们将需要在浏览器中无任何工具的情况下进行分析

![](https://www.resourch.com/usr/typora/12cdddb7a06a47ef8eaafe721d7ffa6f.png)

![](https://www.resourch.com/usr/typora/e57a0148731b4ade98091d52dec9e807.png)

经过对这几个函数的观察，除`_0x1218ef`以外，其他函数都比较简短

![](https://www.resourch.com/usr/typora/bde91e9b91c24b19aaa5b5c549fcf74f.png)

在函数体最长的`_0x1218ef`中断点，能够看到传入了整段字节码

![](https://www.resourch.com/usr/typora/e353486358674d9e80404e4663579418.png)

查看函数体，发现整个函数体是由一个看似为 for 实则为 while 条件循环来控制的控制流，其中有大量的 if 分支跳转，那么这个就很有可能是虚拟机本身了，至于为什么靠这两点可以定位 vmp，我们先按下不表，这里的分析较为复杂，我们先分析较为简单的其他函数

![](https://www.resourch.com/usr/typora/aac3cf62240c424c8d4d6333adc7c66c.png)

虚拟机外层函数简化
---------

#### 反射对象检测

查看第一个闭包，检测当前 JavaScript 环境是否完全支持 `Reflect.construct` 和 `Proxy`，并且进行了字符串格式化检测，确保没有任何 “假冒” 实现。

因此可以得出简化方案：直接返回 true，或使用 ast 将调用`_0x2f9ebc`处全部替换为 true。

```
function _0x2f9ebc() {
    if ('undefined' == typeof Reflect || !Reflect['construct']) {
      return false;
    }
    if (Reflect['construct']['sham']) {
      return false;
    }
    if ('function' == typeof Proxy) {
      return true;
    }
    try {
      Date['prototype']['toString']['call'](Reflect['construct'](Date, [], function () { }));
      return true;
    } catch (_0x1a9721) {
      return false;
    }
  }
```

JavaScript点击复制

第二个函数 `_0x265a42` ，调用了`_0x2f9ebc`来检测`Reflect`，作用是根据当前环境是否支持 `Reflect.construct` 方法来决定如何创建一个新的对象实例。具体来说，它会根据 `_0x2f9ebc` 函数的返回值来选择不同的实现方式。

```
function _0x265a42(_0x3ce31a, _0x217675, _0x5569d0) {
    return (_0x265a42 = _0x2f9ebc() ? Reflect['construct'] : function (_0x2dc1b1, _0x54ae53, _0x5bec15) {
      var _0x2a793f = [null];
      _0x2a793f['push']['apply'](_0x2a793f, _0x54ae53);
      var _0x60cd25 = new (Function['bind']['apply'](_0x2dc1b1, _0x2a793f))();
      _0x5bec15 && _0x59d7fb(_0x60cd25, _0x5bec15['prototype']);
      return _0x60cd25;
    })['apply'](null, arguments);
}
```

JavaScript点击复制

提取一下核心部分

```
(_0x2f9ebc() ? Reflect['construct'] : function (_0x2dc1b1, _0x54ae53, _0x5bec15) {/.../})['apply'](null, arguments)
```

JavaScript点击复制

首先调用 `_0x2f9ebc` 函数来（也就是我们看到的第一个函数）检查当前环境是否支持 `Reflect.construct` 方法。

*   如果 `_0x2f9ebc` 返回 `true`，则使用 `Reflect.construct` 方法来创建新的对象实例。
*   如果 `_0x2f9ebc` 返回 `false`，则使用一个自定义的实现方式来创建新的对象实例。

自定义实现方式通过 `Function.bind` 和 `apply` 方法来模拟 `Reflect.construct` 的行为。

*   首先创建一个包含 `null` 的数组 `_0x2a793f`。
*   然后将 `_0x54ae53`（构造函数参数）推入 `_0x2a793f`。
*   使用 `Function.bind.apply` 方法将构造函数 `_0x2dc1b1` 绑定到 `_0x2a793f`，并创建一个新的实例 `_0x60cd25`。
*   如果提供了 `_0x5bec15`（原型对象），则将其原型属性赋给新创建的实例 `_0x60cd25`，否则调用`_0x59d7fb`，最后返回新创建的实例 `_0x60cd25` 完成代理行为
*   函数 `_0x59d7fb` （第三个函数）的作用是设置对象的原型（prototype）。它首先检查 `Object.setPrototypeOf` 方法是否存在，如果存在则使用它；如果不存在，则使用一个回退方法，通过直接设置对象的 `__proto__` 属性来实现相同的功能

```
function _0x59d7fb(_0x2e16aa, _0x2804b1) {
    return (_0x59d7fb =
      Object["setPrototypeOf"] ||
      function (_0x1d595d, _0x48cf70) {
        _0x1d595d["__proto__"] = _0x48cf70;
        return _0x1d595d;
      })(_0x2e16aa, _0x2804b1);
}
```

JavaScript点击复制

最后，`_0x265a42` 函数通过 `apply` 方法调用，传递 `null` 作为上下文，并将 `arguments` 作为参数传递给所选的实现方式（相当于直接调用方法）。

`Reflect.construct` 是 ES6 中引入的一个方法，用于调用构造函数并创建一个新的实例。那么我们可以得出结论，前三个闭包函数的作用仅为防止反射 hook，能够正确调用反射创建实例。

借助 ide 查看一下`_0x2f9ebc`（判断`Reflect.construct`是否被修改的函数）和`_0x59d7fb`（实现自定义原型对象的函数）的引用位置，发现仅在`_0x265a42`（实现`Reflect.construct`创建实例对象的函数）中发生过调用，因此我们可以简单删除掉这三个函数，并将第二个函数改为如下即可完成反混淆

```
const _0x265a42 = Reflect.construct
```

JavaScript点击复制

#### 可迭代对象转数组

检查一下下一个函数，这里由于我们疏忽，并没有将自执行函数转换为流程代码，不过可以很轻易的看出， `_0x30aa3b` 的作用是将一个可迭代对象（如数组或类数组对象）转换为一个数组。它通过一系列检查来确定输入对象的类型，并根据类型选择适当的方法进行转换。如果输入对象不是可迭代的，则抛出一个 `TypeError`。

```
function _0x30aa3b(_0x526356) {
    return (
      (function (_0x50954b) {
        if (Array["isArray"](_0x50954b)) {
          for (
            var _0x2e8a5e = 0, _0x5bc4e4 = new Array(_0x50954b["length"]);
            _0x2e8a5e < _0x50954b["length"];
            _0x2e8a5e++
          ) {
            _0x5bc4e4[_0x2e8a5e] = _0x50954b[_0x2e8a5e];
          }
          return _0x5bc4e4;
        }
      })(_0x526356) ||
      (function (_0x325d22) {
        if (
          Symbol["iterator"] in Object(_0x325d22) ||
          "[object Arguments]" ===
            Object["prototype"]["toString"]["call"](_0x325d22)
        ) {
          return Array["from"](_0x325d22);
        }
      })(_0x526356) ||
      (function () {
        throw new TypeError("Invalid attempt to spread non-iterable instance");
      })()
    );
  }
```

JavaScript点击复制

简要还原

```
const _0x30aa3b = Array.from;
```

JavaScript点击复制

剩下的函数几乎都是一些花指令，我们不需要在这个阶段继续深入分析了

![](https://www.resourch.com/usr/typora/4e52325e2f8940d29318e6eb855dfee4.png)

这样我们就将几块较大的函数分析完毕了

虚拟机解释器结构分析
----------

回到刚刚的`_0x1218ef`中，为什么我们能断定这个就是虚拟机的执行函数呢？

还记得我们上一篇文章中制作的一个简易 jsvmp 虚拟机吗？回忆一下当时的代码是如何写的

```
function VM() {
  this.stack = [];
  this.execute = function(instructions, context) {
    for (let i = 0; i < instructions.length; i++) {
      const instr = instructions[i];
      switch (instr.op) {
        case 'LOAD':
          this.stack.push(context[instr.arg]);
          break;
        case 'ADD':
          const b = this.stack.pop();
          const a = this.stack.pop();
          this.stack.push(a + b);
          break;
        case 'RETURN':
          return this.stack.pop();
      }
    }
  };
}
```

JavaScript点击复制

如何还不知道为什么，这就需要好好复习一下编译原理相关的知识了，虚拟机的内层循环的执行过程可以简化为如下状态

```
加载程序字节码；
while(程序未结束){
    取操作符; 
    根据操作符的值执行一个动作;
}
```

由于指令系统的简单性，使得虚拟机执行的过程十分简单，从而有利于提高执行的效率，因此，vmp 的虚拟机核心特点就是，一定会有一个大的循环结构来控制虚拟机的指令操作。并且在循环体内，有明显的根据不同指令执行不同任务的特征。可以看到`_0x1218ef`的核心特点便是如此，从外部加载程序字节码，并由一个大循环体来控制流程执行，在循环体内依据操作数`_0x2458f0`跳转不不同分支流程（if 跳转），此处的 if 跳转不便于对流程逻辑还原，我们后期可以利用 ast 更改为 switch 跳转

![](https://www.resourch.com/usr/typora/cee57fbcc1234fa2b6dfc64cffb1623d.png)

此外，我们还可以发现，这个函数中有从字节码中一次读取 2 个字节并转换为 16 进制操作码，并将字节码索引向后推 2 个的过程，这更加说明了这一段就是虚拟机执行流程

![](https://www.resourch.com/usr/typora/3a9b413b87d94969abada9c5105a614e.png)

这里不知道网上博客一堆文章你抄我我抄你，抄到哪里去了，都认定`_0x217611`（在这些相似的博客中，这个变量为`_0x1383c7`）这个循环变量的值为 0-255。事实上 0-255 是`_0x3f0f70`字节码范围，而并非指令的跳转顺序

```
var _0x3f0f70 = parseInt(
  "" + _0x2232d0[_0x217611] + _0x2232d0[_0x217611 + 1],
);
_0x217611 += 2;
var _0x2458f0 = 3 & (_0xf24f2b = (13 * _0x3f0f70) % 241);
```

JavaScript点击复制

我们截取一下这段初始化代码：

*   第一步：`_0x3f0f70` 是通过将字节码中两个连续的元素拼接成一个字符串，然后解析为十六进制数得到的。因此，`_0x3f0f70` 的范围是 0x00 到 0xFF，即 0 到 255。
*   第二步：`_0xf24f2b` 是通过将 `_0x3f0f70` 乘以 13 然后对 241 取模得到的。因此，`_0xf24f2b` 的范围是 0 到 240（因为取模 241 的结果范围是 0 到 240）。
*   第三步：`_0x2458f0` 是通过 `_0xf24f2b` 与 3 进行按位与操作得到的。按位与 3 的结果范围是 0 到 3（因为 3 的二进制表示是 11，即只保留最低两位）。

这也就得出了`_0x2458f0`初始值为 0-3 的结论。不过，`_0x2458f0` 代表的指令的值确实会随着流程的执行而变化，甚至大于 3，但将循环变量误认为为 0-255 仅仅是原创者的一项书写错误，各位其他的博主却原封不动的抄了下来

![](https://www.resourch.com/usr/typora/da4e1bf3e8f04db1995be655c362a817.png)

将整个执行流程折叠，可以看到有一个变量和否运算值被同时运用于两个 if，这还说明了虚拟机执行流程中可能存在一段虚假流程代码，如果直接将其削除，我们又可以减少一半的代码量

![](https://www.resourch.com/usr/typora/b6ed738f3d064a34ab658547ffa5b15c.png)

然而，在函数体开头和两个 if 前对条件变量进行插桩，发现所有函数体传入的条件值都是 0，但到了第二个 if 的时候极少数情况会变为 1，这下可就有点难受了，说明第二个分支流程可能并不是虚假流程分支，无法直接删除，我们需要在之后对流程分支进行判断

![](https://www.resourch.com/usr/typora/a86609d68a424e36b0b0e4946a83c5a9.png)

栈式虚拟机命名去混淆
----------

通过对虚拟机的初步分析，我们大致了解了这个虚拟机的基础结构

这段代码非常可能实现了一个栈式虚拟机，主要通过一个数组 `_0xcc6308` 来模拟栈，并使用变量 `_0x2e1055` 作为栈指针。为何如此断定呢？

观察一下源代码，我们可以从代码中确定这个 jsvmp 含有如下特征

### 指令生成

代码通过解析输入的字节码 `_0x2232d0` 来执行不同的操作。每次读取两个字节，并将其转换为一个整数 `_0x3f0f70`，然后根据 `_0x3f0f70` 的值来决定执行哪种操作

```
var _0x3f0f70 = parseInt("" + _0x2232d0[_0x217611] + _0x2232d0[_0x217611 + 1], 16);
_0x217611 += 2;
```

JavaScript点击复制

### 指令条件跳转

根据条件跳转到不同的代码位置

```
if (_0x2458f0 > 2) {
 // 执行某些操作*
} else {
 // 执行其他操作*
}
```

JavaScript点击复制

### 初始化存在的可疑的栈数组

初始化了两个可疑变量，在这两个变量上我们可以看到大量的栈操作

```
var _0xcc6308 = []; // 栈数组
var _0x2e1055 = 0;  // 栈指针，初始化为0
```

JavaScript点击复制

1.  **压栈操作**
    
    *   `_0xcc6308[++_0x2e1055] = value;`：将 `value` 压入栈中，并将栈指针 `_0x2e1055` 增加 1。
    *   例如：`_0xcc6308[++_0x2e1055] = null;` 将 `null` 压入栈中。
2.  **出栈操作**
    
    *   `var value = _0xcc6308[_0x2e1055--];`：从栈顶弹出一个值，并将栈指针 `_0x2e1055` 减少 1。
    *   例如：`_0x4db217 = _0xcc6308[_0x2e1055--];` 从栈顶弹出一个值赋给 `_0x4db217`。

### 栈操作

根据不同的指令值，代码会在 if 分支结构中执行不同的操作，而这些操作都是围绕栈展开的

#### 1. 值操作

*   **压栈**：将值压入栈中。
    
    ```
    _0xcc6308[++_0x2e1055] = value;
    ```
    
    JavaScript点击复制
*   **出栈**：从栈顶弹出一个值。
    
    ```
    var _0x4db217 = _0xcc6308[_0x2e1055--];
    ```
    
    JavaScript点击复制

#### 2. 算术操作

`_0x4db217`均为栈顶弹出的值

*   **加法**：将栈顶两个值相加，并将结果压入栈中。
    
    ```
    _0xcc6308[_0x2e1055] = _0xcc6308[_0x2e1055] + _0x4db217;
    ```
    
    JavaScript点击复制
*   **减法**：将栈顶两个值相减，并将结果压入栈中。
    
    ```
    _0xcc6308[_0x2e1055] = _0xcc6308[_0x2e1055] - _0x4db217;
    ```
    
    JavaScript点击复制
*   **乘法**：将栈顶两个值相乘，并将结果压入栈中。
    
    ```
    _0xcc6308[_0x2e1055] = _0xcc6308[_0x2e1055] * _0x4db217;
    ```
    
    JavaScript点击复制
*   **除法**：将栈顶两个值相除，并将结果压入栈中。
    
    ```
    _0xcc6308[_0x2e1055] = _0xcc6308[_0x2e1055] / _0x4db217;
    ```
    
    JavaScript点击复制

#### 3. 逻辑操作

*   **与操作**：将栈顶两个值进行与操作，并将结果压入栈中。
    
    ```
    _0xcc6308[_0x2e1055] = _0xcc6308[_0x2e1055] & _0x4db217;
    ```
    
    JavaScript点击复制
*   **或操作**：将栈顶两个值进行或操作，并将结果压入栈中。
    
    ```
    _0xcc6308[_0x2e1055] = _0xcc6308[_0x2e1055] | _0x4db217;
    ```
    
    JavaScript点击复制
*   **异或操作**：将栈顶两个值进行异或操作，并将结果压入栈中。
    
    ```
    _0xcc6308[_0x2e1055] = _0xcc6308[_0x2e1055] ^ _0x4db217;
    ```
    
    JavaScript点击复制

#### 4. 比较操作

*   **大于等于**：比较栈顶两个值，并将结果压入栈中。
    
    ```
    _0xcc6308[_0x2e1055] = _0xcc6308[_0x2e1055] >= _0x4db217;
    ```
    
    JavaScript点击复制
*   **小于**：比较栈顶两个值，并将结果压入栈中。
    
    ```
    _0xcc6308[_0x2e1055] = _0xcc6308[_0x2e1055] < _0x4db217;
    ```
    
    JavaScript点击复制
*   **等于**：比较栈顶两个值是否相等，并将结果压入栈中。
    
    ```
    _0xcc6308[_0x2e1055] = _0xcc6308[_0x2e1055] === _0x4db217;
    ```
    
    JavaScript点击复制
*   **不等于**：比较栈顶两个值是否不相等，并将结果压入栈中。
    
    ```
    _0xcc6308[_0x2e1055] = _0xcc6308[_0x2e1055] !== _0x4db217;
    ```
    
    JavaScript点击复制

#### 5. 类型操作

*   **获取类型**：获取栈顶值的类型，并将结果压入栈中。
    
    ```
    _0xcc6308[_0x2e1055] = typeof _0x4db217;
    ```
    
    JavaScript点击复制

#### 6. 函数调用

*   **调用函数**：从栈顶弹出一个函数并调用它。
    
    ```
    _0xcc6308[_0x2e1055] = _0x2458f0(_0x1f1790);
    ```
    
    JavaScript点击复制

#### 7. 异常处理

*   **抛出异常**：从栈顶弹出一个值并抛出异常。
    
    ```
    throw _0xcc6308[_0x2e1055--];
    ```
    
    JavaScript点击复制

### 上下文环境入栈

`stack[++stackPointer] = context;`

* 这里我使用 context 来代表压入的作用域，实际上传入的为`this`或`global`

根据上文给出的特征，我们可以改写变量名，以实现虚拟机简化

![](https://www.resourch.com/usr/typora/86d878a1276541fea58805f7f5983c3f.png)

1.  **`stack`**：
    
    *   用于模拟操作数栈，存储计算过程中需要的中间值。
2.  **`stackPointer`**：
    
    *   用于跟踪操作数栈的当前指针位置，指示栈顶的位置。
3.  **`PC`**（程序计数器）：
    
    *   用于跟踪当前执行的位置，初始化为 `startIndex`，并在每次读取操作码后递增。
4.  **`endIndex`**：
    
    *   计算并存储代码段的结束位置，确保程序计数器在执行过程中不会越界。
5.  **`operationCode`**：
    
    *   用于存储计算得到的操作码，通过对 `byteValue` 进行特定的数学运算和位操作得到。
6.  **`operandReg`**：
    
    用于存储操作数寄存器的值，通常用于临时存储从操作数栈中弹出的值，以便进行进一步的计算或操作。例如如下操作，从栈顶对两个操作数进行大小判断
    
    ```
    operandReg = stack[stackPointer--];
    stack[stackPointer] = stack[stackPointer] >= operandReg;
    ```
    
    JavaScript点击复制
7.  **`regA`** 和 **`regB`**：
    
    通用寄存器，用于储存从栈顶弹出的数据
    
    ```
    regA = stack[stackPointer--];
    regB = stack[stackPointer--];
    ```
    
    JavaScript点击复制

这些变量共同作用，模拟了一个简单的虚拟机执行环境，其中 `stack` 和 `stackPointer` 用于操作数栈的管理，`PC` 和 `endIndex` 用于控制程序的执行流程，`operandReg`、`regA` 和 `regB` 用于存储中间计算结果，`_0xf24f2b` 用于位操作和条件判断。到这里为止，虚拟机的结构就十分清晰了