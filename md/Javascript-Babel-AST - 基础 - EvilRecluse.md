> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [evilrecluse.top](https://evilrecluse.top/post/7389a59f/#%E4%BD%9C%E7%94%A8%E5%9F%9FScope-%E4%B8%8E-%E8%A2%AB%E7%BB%91%E5%AE%9A%E9%87%8FBinding)

> 自娱自乐

[](#抽象语法树-AST "抽象语法树 AST")抽象语法树 AST[](#抽象语法树-AST)
-------------------------------------------------

> 维基百科 - 抽象语法树：[https://zh.wikipedia.org/wiki/%E6%8A%……](https://zh.wikipedia.org/wiki/%E6%8A%BD%E8%B1%A1%E8%AA%9E%E6%B3%95%E6%A8%B9)

抽象语法树（Abstract Syntax Tree，AST）是源代码语法结构的一种抽象表示  
它以树状的形式表现编程语言的语法结构，树上的每个节点都表示源代码中的一种结构  
之所以说语法是 “抽象” 的，是因为这里的语法并不会表示出真实语法中出现的每个细节

> 例: 代码转抽象语法树

<table><tbody><tr><td></td><td><pre>while b ≠ 0
    if a &gt; b
        a := a − b
    else
        b := b − a
    return a

</pre></td></tr></tbody></table>

[![](https://evilrecluse.top/Javascript-Babel-AST-%E5%9F%BA%E7%A1%80/20201224050050034.png)](https://evilrecluse.top/Javascript-Babel-AST-%E5%9F%BA%E7%A1%80/20201224050050034.png)

需要注意的是，`Babel`并非操作的基础的`ESTree`，它 修改 / 扩充 了一些节点类型  
(自行记录：[`Babel parser`与`ESTree`的不同之处](https://evilrecluse.top/post/77d57432/#%E8%BE%93%E5%87%BA-Output)）

[](#链接信息 "链接信息")链接信息[](#链接信息)
-----------------------------

<table><thead><tr><th>信息</th><th>地址</th></tr></thead><tbody><tr><td>AST 在线解析</td><td><a target="_blank" rel="noopener" href="https://astexplorer.net/">https://astexplorer.net/</a></td></tr><tr><td>babel 中文文档</td><td><a target="_blank" rel="noopener" href="https://www.babeljs.cn/docs/">https://www.babeljs.cn/docs/</a></td></tr><tr><td>babel 英文文档</td><td><a target="_blank" rel="noopener" href="https://babeljs.io/docs/en/">https://babeljs.io/docs/en/</a></td></tr><tr><td>Github</td><td><a target="_blank" rel="noopener" href="https://github.com/babel/babel">https://github.com/babel/babel</a></td></tr><tr><td>插件手册</td><td><a target="_blank" rel="noopener" href="https://blog.csdn.net/weixin_33826609/article/details/93164633#toc-visitors">https://blog.csdn.net/weixin_33826609/article/details/93164633#toc-visitors</a></td></tr></tbody></table>

<table><thead><tr><th>信息</th><th>地址</th></tr></thead><tbody><tr><td>babel 各节点解释</td><td><a target="_blank" rel="noopener" href="https://github.com/babel/babylon/blob/master/ast/spec.md">https://github.com/babel/babylon/blob/master/ast/spec.md</a></td></tr><tr><td>babel 简单剖析</td><td><a target="_blank" rel="noopener" href="http://www.alloyteam.com/2017/04/analysis-of-babel-babel-overview/">http://www.alloyteam.com/2017/04/analysis-of-babel-babel-overview/</a></td></tr><tr><td>淘宝前端团队写的 babel 相关</td><td><a target="_blank" rel="noopener" href="https://fed.taobao.org/blog/taofed/do71ct/babel-plugins/">https://fed.taobao.org/blog/taofed/do71ct/babel-plugins/</a></td></tr><tr><td>babel 到底将代码转换成什么</td><td><a target="_blank" rel="noopener" href="http://www.alloyteam.com/2016/05/babel-code-into-a-bird-like/">http://www.alloyteam.com/2016/05/babel-code-into-a-bird-like/</a></td></tr></tbody></table>

[](#Node-js "Node.js")Node.js[](#Node-js)
-----------------------------------------

### [](#安装-Node-js-与Babel "安装 Node.js 与Babel")安装 Node.js 与 Babel[](#安装-Node-js-与Babel)

首先你要有`Node.js`环境

[Post not found: Node.js](#)

全局安装 `babel`的各个库

<table><tbody><tr><td></td><td><pre>npm install @babel/core -g
npm install @babel/parser -g
npm install @babel/traverse -g
npm install @babel/generator -g
npm install @babel/types -g

</pre></td></tr></tbody></table>

简单描述一下刚刚安装的库的作用：

*   `@babel/core`是 `Babel` 的核心代码
*   `@babel/parser`常用于解析代码语法树的模块
*   `@babel/traverse`常用于操作语法树的模块
*   `@babel/generator`常用于将语法树生成代码的模块
*   `@babel/types` 包含类型信息，在生成节点等与类型相关的操作会用到

### [](#安装测试 "安装测试")安装测试[](#安装测试)

启动 `Node.js` 环境

引入 babel 库，测试是否成功

<table><tbody><tr><td></td><td><pre>const parser = require("@babel/parser");

</pre></td></tr></tbody></table>

这一步可能会出现说没有库的问题，这是由于环境变量导致的  
一般而言，安装`Node.js`会默认配置上环境变量，但这个配置不一定有用  
此时你可以手动设置环境变量，或者用绝对路径来引入

<table><tbody><tr><td></td><td><pre>const parser = require("E:/Environment/Nodejs/node_global/node_modules/@babel/parser");

</pre></td></tr></tbody></table>

[](#在线解析网站 "在线解析网站")在线解析网站[](#在线解析网站)
-------------------------------------

链接： [https://astexplorer.net/](https://astexplorer.net/)  
这个是一个解析代码语法树的网站  
这里是需要用它的`Babel`解析语法树，所以要做对应设置  
[![](https://evilrecluse.top/Javascript-Babel-AST-%E5%9F%BA%E7%A1%80/20200830085956301.png)](https://evilrecluse.top/Javascript-Babel-AST-%E5%9F%BA%E7%A1%80/20200830085956301.png)  
打开以后将解析方式改为`@babel/parser`  
在左边的文本框里输入一些代码，就能在右边看到对应解析出来的语法树了

[](#思路 "思路")思路[](#思路)
---------------------

如果将代码中各类的语句都看成节点，那么整个程序的代码就像是由一个个节点组成的树状结构，代码的运行就像是不完全遍历这颗树一样

AST 解析源代码实际上就是这样干的  
在代码解析完成后，你会得到一颗语法树，这颗语法树包含了代码中的所有内容

你可以对它进行树（数据结构）可以进行的所有操作（从复杂的 `有序树转二叉树` 到简单的 `替换节点`，`计算树的深度`，`剪枝`，`拼接`都可以）

最后想象一下，程序运行 就是在这棵树里进行 `深度优先遍历`

> 实际上，它的结果更接近图（数据结构），因为可能存在调用循环 / 递归（树中的节点不允许连接自身或有多个父级）

[](#常见节点信息解析 "常见节点信息解析")常见节点信息解析[](#常见节点信息解析)
---------------------------------------------

<table><thead><tr><th>节点属性</th><th>记录的信息</th></tr></thead><tbody><tr><td>type</td><td>当前节点的类型</td></tr><tr><td>start</td><td>当前节点的起始位</td></tr><tr><td>end</td><td>当前节点的末尾</td></tr><tr><td>loc</td><td>当前节点所在的行列位置<br>起始于结束的行列信息</td></tr><tr><td>errors</td><td>File 节点所持有的特有属性，可以不用理会</td></tr><tr><td>program</td><td>包含整个源代码，不包含注释节点</td></tr><tr><td>comments</td><td>源代码中所有的注释会显示在这里</td></tr></tbody></table>

就这么直接列出来可能还不太容易理解  
可以利用 [在线解析网站](https://astexplorer.net/) 解析一小段`javascript`代码来帮助理解 / 记忆

输入仪式性的语句 `console.log("Hello World")` 的变种代码

<table><tbody><tr><td></td><td><pre>function test(){
  var i = ['H', 'e', 'l', 'l', 'o', ' ', 'W', 'o', 'r', 'l', 'd'];
  i = i.join('');
  return i
}
console.log(test())

</pre></td></tr></tbody></table>

### [](#最外层File节点 "最外层File节点")最外层`File节点`[](#最外层File节点)

[![](https://evilrecluse.top/Javascript-Babel-AST-%E5%9F%BA%E7%A1%80/20200901023343710.png)](https://evilrecluse.top/Javascript-Babel-AST-%E5%9F%BA%E7%A1%80/20200901023343710.png)  
先将节点收得差不多，观察最外面的`File节点`的信息

*   `type` `start` `end` `loc` 你能在绝大多数节点里看到
*   常用的类型判断方法`t.is****(node)`就是判断当前节点 type 是否为某个类型

### [](#层级结构分析 "层级结构分析")层级结构分析[](#层级结构分析)

实际上，可以通过观察 `type` 知道一些信息

> 层级分析例 1:  
> [![](https://evilrecluse.top/Javascript-Babel-AST-%E5%9F%BA%E7%A1%80/20200901023523804.png)](https://evilrecluse.top/Javascript-Babel-AST-%E5%9F%BA%E7%A1%80/20200901023523804.png)  
> 比如说这整个文件包含了了代码块主要有两部分： **一个函数声明** 和 **一句语句**

> 层级分析例 2:  
> [![](https://evilrecluse.top/Javascript-Babel-AST-%E5%9F%BA%E7%A1%80/20200901082003100.png)](https://evilrecluse.top/Javascript-Babel-AST-%E5%9F%BA%E7%A1%80/20200901082003100.png)  
> 又比如说，这个函数里包含了三个部分：**一个声明**，**一个语句**，**一个返回**

[](#解析代码-Code→AST "解析代码 Code→AST")解析代码 Code→AST[](#解析代码-Code→AST)
-----------------------------------------------------------------

`@babel/parser`能将`javascript`代码解析成语法树  
这个库在前文中已经安装好了，现在用它来解析语法树吧

> 例：用 babel 解析语法树

<table><tbody><tr><td></td><td><pre>const parser = require("@babel/parser");

var jscode = "var a = 123;";
let ast = parser.parse(jscode);
console.log(JSON.stringify(ast, null, '\t'));
</pre></td></tr></tbody></table>

此处会输出和网页解析出来的内容一样的结果  
做简单分析时，用网页解析比较快捷方便，很少写代码来解析

[](#AST寻路 "AST寻路")AST 寻路[](#AST寻路)
----------------------------------

在解析出语法树后，就可以进行各类的操作了  
但和想要对树（数据结构）进行任何操作一样，在对 AST 进行各类操作之前，你要先会 寻路 （到达 (遍历到) 指定的节点）才能进行操作

> 寻路这个词是我自己瞎编的

### [](#特征 "特征")特征[](#特征)

想要约朋友来家里玩，可以向别人描述 家的附近 或者 家本身 有什么特征。比如说:

> “我家门口有一颗歪脖子树” 或者 “我家房顶有一个大洞”

这样子能很方便得让别人找到位置

寻路的过程也一样。想要到达 某个 / 某些 特定节点上，可以根据这个节点的 `类型` 与 `路径` 给出判断依据来到达特定的 某个 / 某些 节点

### [](#遍历整个语法树 "遍历整个语法树")遍历整个语法树[](#遍历整个语法树)

绝大多数情况下，对 AST 进行寻路比上面描述的情况更糟糕一些

> **在 AST 中寻路的情况更像是：**  
> 你的朋友坐在一个城市观光车上，观光车会穿过城市里的每一个街道。你的朋友无法控制车前进的方向，你只能让车在你描述的情况下停下，用这种方式来到达你家

默认情况下，`@babel/traverse`能让你遍历整个 AST  
如果想要寻路到达某个节点，就需要给出你想要寻路的 那个 / 那些 节点的特征进行判断操作

### [](#使用-Path-进行寻路 "使用 Path 进行寻路")使用 Path 进行寻路[](#使用-Path-进行寻路)

在使用 `enter` 遍历所有节点的时候，参数 `path` 会传入当前的路径  
可以根据`path`进行各种判断，继而进行各类操作

#### [](#Path基本信息 "Path基本信息")Path 基本信息[](#Path基本信息)

*   获取 `Path`对应节点的 源代码 `path.toString()`
*   获取 `Path`对应节点的 类型 `path.type`

> 例: 寻找代码中包含 字符 a 的节点

```
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;

var jscode = `
    var b = 123;
    a = b + 1;
`;

const visitor = {
    enter(path)  
    {
        if(path.toString().indexOf('a') > -1){  
            console.log('当前路径类型', path.type); 
            console.log('当前路径源码：', path.toString()); 
            
            console.log('这是一个变量声明节点:\t', path.isVariableDeclaration())
            console.log('--------------------')
        }
    }
}
let ast = parser.parse(jscode);
traverse(ast, visitor);
```

会得到如下的结果

```
当前路径类型 Program
当前路径源码： var b = 123;
a = b + 1;
这是一个变量声明节点:	 false
--------------------
当前路径类型 VariableDeclaration
当前路径源码： var b = 123;
这是一个变量声明节点:	 true
--------------------
当前路径类型 ExpressionStatement
当前路径源码： a = b + 1;
这是一个变量声明节点:	 false
--------------------
当前路径类型 AssignmentExpression
当前路径源码： a = b + 1
这是一个变量声明节点:	 false
--------------------
当前路径类型 Identifier
当前路径源码： a
这是一个变量声明节点:	 false
--------------------
```

此处是对源代码进行判断，所以`var`这种关键词也是会被认为是包含`a`的

### [](#使用-节点-进行寻路 "使用 节点 进行寻路")使用 节点 进行寻路[](#使用-节点-进行寻路)

> 例子：根据特点寻找语法树中的节点
> 
> *   特点：这个节点是一个 变量声明节点

```
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;

var jscode = "var a = 123;";

const visitor = {
    
    VariableDeclaration(path)
    {   
        console.log('当前路径 源码:\n', path.toString());
        console.log('当前路径 节点:\n', path.node.toString())
        console.log('当前路径 父级节点:\n', path.parent.toString());
        console.log('当前路径 父级路径:\n', path.parentPath.toString())
        console.log('当前路径 类型:\n', path.type)
    }
}

let ast = parser.parse(jscode);  
traverse(ast, visitor);
```

由于直接输出`Node`会显示非常多的信息，此处用`toString()`意思意思  
得到的输出结果：

<table><tbody><tr><td></td><td><pre>当前路径 源码:
 var a = 123;
当前路径 节点:
 [object Object]
当前路径 父级节点:
 [object Object]
当前路径 父级路径:
 var a = 123;
当前路径 类型:
 VariableDeclaration

</pre></td></tr></tbody></table>

这里用了一个 `VariableDeclaration`（变量声明节点）来表述特点，更多得节点类型可以查看 [Github-Babel - 节点类型文档](https://github.com/babel/babylon/blob/master/ast/spec.md#node-objects)

**注意：所有符合你描述的特点的节点都会进行声明中的操作**（此处为输出信息）  
由于此处只有一句声明语句的代码，所以随意给出一个合适的特点就能轻松到达目标节点  
如果代码量大，又希望只改动部分，那么最好给出更多的语句以精确修改目标

[](#用AST生成代码-AST→Code "用AST生成代码 AST→Code")用 AST 生成代码 AST→Code[](#用AST生成代码-AST→Code)
-----------------------------------------------------------------------------------

尽管你现在可能还并不知道如何对语法树节点进行操作  
但根据 AST 生成代码的方法还是要学会的，要不然最终结果长什么样都不知道

可以使用`@babel/generator`来实现 根据 AST 生成代码的这个过程

> 例：根据 AST 生成代码以查看修改结果

```
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const generator = require("@babel/generator").default;

const jscode = 'function squire(){var n = 3; return n*n;}';
let ast = parser.parse(jscode);

const visitor = {
  BinaryExpression(path) {  
    if (path.node.operator == '*') {  
      path.node.operator = '+';  
    }
  }
}

traverse(ast, visitor);  
console.log(generator(ast)['code']);
```

运行后会输出结果

<table><tbody><tr><td></td><td><pre>function squire() {
  var n = 3;
  return n + n;
}

</pre></td></tr></tbody></table>

此处将所有的表达式乘法转为加法，并且将改变后的 AST 利用`@babel/generator`生成出新的代码

[](#常用基础操作 "常用基础操作")常用基础操作[](#常用基础操作)
-------------------------------------

### [](#创建节点 "创建节点")创建节点[](#创建节点)

`@babel/types`包含了各个节点的定义  
可以通过使用`@babel/types`的类型名，查阅 [`@babel/types`官方文档](https://babeljs.io/docs/en/babel-types)，获取对应类型的构造函数，创建对应类型的节点

> 例：利用 @babel/types 提供的类来直接创建节点，编写 ast 内容

<table><tbody><tr><td></td><td><pre>const t = require("@babel/types");
const generator = require("@babel/generator").default;

var callee = t.memberExpression(t.identifier('console'), t.identifier('log')),
    args = [t.NumericLiteral(666)],
    call_exp = t.callExpression(callee, args),
    exp_statement = t.ExpressionStatement(call_exp)

console.log(generator(exp_statement)['code'])
</pre></td></tr></tbody></table>

输出结果：

### [](#插入节点 "插入节点")插入节点[](#插入节点)

`NodePath.insertAfter()`方法用于在当前`path`前面插入节点  
`NodePath.insertBefore()`方法用于在当前`path`后面插入节点

> 例：向语法树中插入节点

```
const t = require("@babel/types");
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const generator = require("@babel/generator").default;

const jscode = `function square(n) {
  var a = 2;
}`;

const ast = parser.parse(jscode);
const visitor = {
  VariableDeclaration(path) {  
    var node = t.NumericLiteral(1)  
    path.insertAfter(node)  
    node = t.NumericLiteral(3)
    path.insertBefore(node)  
  }
}

traverse(ast, visitor);
console.log(generator(ast)['code'])
```

结果：

<table><tbody><tr><td></td><td><pre>function square(n) {
  3
  var a = 2;
  1
}

</pre></td></tr></tbody></table>

### [](#替换节点 "替换节点")替换节点[](#替换节点)

`NodePath.replaceInline` 方法用于替换对应 path 的节点

> 例：寻找计算节点，计算好了以后，生成新的数字节点，替换原本的节点

```
const t = require("@babel/types");
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const generator = require("@babel/generator").default;

const jscode = `function square(n) {
  return 1 + 1;
}`;

const ast = parser.parse(jscode);
const visitor = {
  BinaryExpression(path) {
    var result = eval(path.toString())  
    var node = t.NumericLiteral(result)  
    path.replaceInline(node);   
  }
}

traverse(ast, visitor);
console.log(generator(ast)['code'])
```

得到结果：

<table><tbody><tr><td></td><td><pre>function square(n) {
  return 2;
}

</pre></td></tr></tbody></table>

### [](#删除节点 "删除节点")删除节点[](#删除节点)

`NodePath.remove()`用于删除路径对应的节点  
由于是对`path`操作，所以务必注意不要误删

```
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const generator = require("@babel/generator").default;

const jscode = `function square(n) {
  var a = 1;
  return 1 + 1;
}`;

const ast = parser.parse(jscode);
const visitor = {
  VariableDeclaration(path) {  
      path.remove()
  }
}

traverse(ast, visitor);
console.log(generator(ast)['code'])
```

得到结果：

<table><tbody><tr><td></td><td><pre>function square(n) {
  return 1 + 1;
}

</pre></td></tr></tbody></table>

[](#词汇描述 "词汇描述")词汇描述[](#词汇描述)
-----------------------------

*   **作用域 scope** - 有效范围  
    作用域（`scope`，或译作有效范围）是名字（`name`）与实体（`entity`）的绑定（`binding`）保持有效的那部分计算机程序  
    不同的编程语言可能有不同的作用域和名字解析。而同一语言内也可能存在多种作用域，随实体的类型变化而不同  
    作用域类别影响变量的绑定方式，根据语言使用静态作用域还是动态作用域变量的取值可能会有不同的结果
    
    > 摘自：[维基百科 - 作用域](https://zh.wikipedia.org/wiki/%E4%BD%9C%E7%94%A8%E5%9F%9F)
    
*   **名字绑定 binding**  
    名字绑定 (简称绑定) 是把实体（数据 或 / 且 代码）关联到标识符  
    标识符绑定到实体被称为引用该对象  
    机器语言没有内建的标识符表示方法，但程序设计语言实现了名字与对象的绑定  
    绑定最初是与作用域相关，因为作用域确定了哪个名字绑定到哪个对象——在程序代码中的哪个位置与哪条执行路径
    
    > 摘自：[维基百科 - 名字与绑定](https://zh.wikipedia.org/wiki/%E5%90%8D%E5%AD%97%E7%BB%91%E5%AE%9A)
    

**最简易理解**

对于复杂程度不高的代码，可以可简单的理解为：

*   一个**函数**就是一个**作用域**
*   一个**变量**就是一个**绑定**，依附在**作用域**上

这对于复杂的代码可能并不会成立，但用于学习最基本的内容已经足够

[](#作用域-Scope "作用域 Scope")作用域 Scope[](#作用域-Scope)
-------------------------------------------------

`@Babel`解析出来的语法树节点对象会包含作用域信息，这个信息会作为节点`Node`对象的一个属性保存  
这个属性本身是一个`Scope`对象，其定义位于`node_modules/@babel/traverse/lib/scope/index.js`中

> 例: 查看基本的 作用域与绑定 信息

```
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;

const jscode = `
function squire(i){
    return i * i * i;
}
function i()
{
    var i = 123;
    i += 2;
    return 123;
}
`;
let ast = parser.parse(jscode);
const visitor = {
    "FunctionDeclaration"(path){
        console.log("\n\n这里是函数 ", path.node.id.name + '()')
        path.scope.dump();
    }
}

traverse(ast, visitor);
```

执行 `Scope.dump()`，会得到自底向上的 作用域与变量信息  
得到结果：

```
这里是函数  squire()
------------------------------------------------------------
# FunctionDeclaration
 - i { constant: true, references: 3, violations: 0, kind: 'param' }
# Program
 - squire { constant: true, references: 0, violations: 0, kind: 'hoisted' }
 - i { constant: true, references: 0, violations: 0, kind: 'hoisted' }
------------------------------------------------------------

这里是函数  i()
------------------------------------------------------------
# FunctionDeclaration
 - i { constant: false, references: 0, violations: 1, kind: 'var' }
# Program
 - squire { constant: true, references: 0, violations: 0, kind: 'hoisted' }
 - i { constant: true, references: 0, violations: 0, kind: 'hoisted' }
------------------------------------------------------------
```

**输出查看方法**

*   每一个作用域都以`#`标识输出
    
*   每一个绑定都以`-`标识输出
    
*   对于单次输出，都是自底向上的  
    先输出当前作用域，再输出父级作用域，再输出父级的父级作用域……
    
*   对于单个绑定`Binding`，会输出 4 种信息
    
    *   constant 声明后，是否会被修改
    *   references 被引用次数
    *   violations 被重新定义的次数
    *   kind 函数声明类型。param 参数, hoisted 提升，var 变量， local 内部
    
    后续会单独说明`Binding`对象，此处留个印象即可
    

**描述**  
此处从两个函数节点输出了其作用域的信息

*   这两个函数都是定义在同一级下的，所以都会输出相同的父级作用域`Program`的信息
*   你会发现，代码中有非常多个`i`，有的是函数定义，有的是参数，有的是变量。仔细观察它们的不同之处  
    解释器就是通过 不同层级的作用域 与 绑定定义信息 来区分不同的名称的量的

[](#绑定-Binding "绑定 Binding")绑定 Binding[](#绑定-Binding)
-----------------------------------------------------

`Binding` 对象用于存储 绑定 的信息  
这个对象会作为`Scope`对象的一个属性存在  
同一个作用域可以包含多个 `Binding`

你可以在 `@babel/traverse/lib/scope/binding.js` 中查看到它的定义

> 显示 Binding 的信息

```
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;

const jscode = `
function a(){
    var a = 1;
    a = a + 1;
    return a;
}
function b(){
    var b = 1;
    var c = 2;
    b = b - c;
    return b;
}

`;
let ast = parser.parse(jscode);
const visitor = {
    BlockStatement(path){
        console.log("\n此块节点源码：\n", path.toString())
        console.log('----------------------------------------')
        var bindings = path.scope.bindings
        console.log('作用域内 被绑定量 数量：', Object.keys(bindings).length)

        for(var binding_ in bindings){
            console.log('名字：', binding_)
            binding_ = bindings[binding_];
            console.log('类型：', binding_.kind)
            console.log('定义：', binding_.identifier)
            console.log('是否会被修改：', binding_.constant)
            console.log('被修改信息信息记录', binding_.constantViolations)
            console.log('是否会被引用：', binding_.referenced)
            console.log('被引用次数', binding_.references)
            console.log('被引用信息NodePath记录', binding_.referencePaths)
        }
    }
}

traverse(ast, visitor);
```

会输出一大堆信息。其对应的意义已经写在代码中，可以自行查看

[](#作用 "作用")作用[](#作用)
---------------------

在解混淆中，作用域与绑定 主要用来处理边界的问题  
即：某个量哪里引用了，在哪里定义

> 例：删除所有定义了, 却从未使用的变量

```
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const generator = require("@babel/generator").default;

const jscode = `
var a = 1;
var b = 2;
function squire(){
  var c = 3;
  var d = 4;
  return a * d;
  var e = 5;
}
var f = 6;
`;
let ast = parser.parse(jscode);
const visitor = {
    VariableDeclarator(path)
    {
        const func_name = path.node.id.name;
        const binding = path.scope.getBinding(func_name);
        
        
        if(binding && !binding.referenced){
            path.remove();
        }
    },
}

traverse(ast, visitor);
console.log(generator(ast)['code']);
```

得到输出

<table><tbody><tr><td></td><td><pre>var a = 1;

function squire() {
  var d = 4;
  return a * d;
}
</pre></td></tr></tbody></table>

这里使用了`Scope.getBinding()`方法来获取`Binding`对象, 判断其引用情况来对语法树进行修改

**相关链接**

*   蔡老板的教程：[https://wx.zsxq.com/dweb2/index/group/48415254524248](https://wx.zsxq.com/dweb2/index/group/48415254524248)