> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1913553-1-1.html)

> [md]## 声明 ` 本文章中所有内容仅供学习交流使用，不用于其他任何目的，不提供完整代码，抓包内容、敏感网址、数据接口等均已做脱敏处理，严禁用于商业用途和非法用途 ...

![](https://avatar.52pojie.cn/data/avatar/002/24/45/08_avatar_middle.jpg)kylin1020

### 声明

`本文章中所有内容仅供学习交流使用，不用于其他任何目的，不提供完整代码，抓包内容、敏感网址、数据接口等均已做脱敏处理，严禁用于商业用途和非法用途，否则由此产生的一切后果均与作者无关.本文章未经许可禁止转载，禁止任何修改后二次传播，擅自使用本文讲解的技术而导致的任何意外，作者均不负责，若有侵权，请联系作者立即删除.`

目标地址: `aHR0cHM6Ly9zZWFyY2guamQuY29tL1NlYXJjaD9rZXl3b3JkPWlwaG9uZSZzdWdnZXN0PTEuaGlzLjAuMCZ3cT1pcGhvbmU=`

### 前期参数分析

打开目标地址, 查看网络请求可以知道携带有`h5st`参数.  
![](https://attach.52pojie.cn/forum/202404/19/160057ozjzwjdztlldvbjo.png)

从调用栈中出现的 js 文件列表中搜索`h5st`关键词, 可找到两处相关代码, 且两处调用的函数一样, 都是`window.PSign.sign`.  
 ![](https://attach.52pojie.cn/forum/202404/19/160110euj7p7625qixh69q.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/160132adsgse0v9m4idad4.png)   
分别在两处代码打上断点调试, 然后重新刷新, 代码停在其中一处并且知道了传递进来的参数.  
 ![](https://attach.52pojie.cn/forum/202404/19/160155xkzpkjbap22z2ij3.png)   
尝试单步调试, js 文件跳转到 js_security_v3_0.1.6.js 的一处函数中.  
 ![](https://attach.52pojie.cn/forum/202404/19/160208gxz8r9m8falhmlnl.png) 

### 解混淆

`js_security_v3_0.1.6.js`文件进行的一定程度的混淆, 直接进行参数分析可能较困难, 所以尝试先利用 ast 解开部分混淆规则, 例如先解决字符串替换函数, 控制流等, 然后再进行分析会更容易理解代码逻辑.

#### 1.1 字符串函数还原

字符串函数意思是指传入指定参数返回特定解码字符串的函数, 该函数目的是将字符串隐藏起来, 防止逆向时通过关键词搜索到关键代码逻辑. 如`window.PSign.sign`函数入口处的代码所示, 其中`rt(o, e - 809))`即是字符串函数, 实际返回`apply`关键词.  
![](https://attach.52pojie.cn/forum/202404/19/160225r63ygpjyyhnzsp5o.png)

因此对于以下代码:

```
return nt[(e = t,
o = r,
rt(o, e - 809))](this, arguments)
```

实际上应该简化为:

```
return nt["apply"](this, arguments)
```

`js_security_v3_0.1.6.js`用到了大量的字符串函数, 并且调用的字符串函数还不相同, 有的甚至是嵌套传递一个新的偏移值后作为一个新的字符串函数.  
![](https://attach.52pojie.cn/forum/202404/19/160239zgnzn38dzbegendh.png)

因此首先要分析下这些字符串函数的共同特性或者最终会调用到哪个根字符串函数, 从根字符串函数 (假设函数名为`rootFunc`) 开始查找引用到的地方, 如果是直接传递数值则直接解码得到字符串并利用 ast 将函数调用节点替换成字符串的节点, 如果传递进来的是参数而不是具体的数值 (例如: `rootFunc(a-1, b-2)`, 其中 a, b 是参数, 不是数值), 则找到该调用所处的函数继续查找调用引用该衍生函数的地方继续进行判断, 如果是数值则可以加上偏移解码得到字符串, 否则继续重复上述操作, 直到所有引用的地方都是具体的数值传入为止.  
 ![](https://attach.52pojie.cn/forum/202404/19/160250zc6tuuha4vu4443a.png) 

##### 1.1.1 字符串函数分析

以`rt`字符串函数为例, 查看`rt`字符串函数的代码, 可知`rt`函数是返回`$v`函数加上偏移的结果; 继续跟踪`$v`函数, 得到`$v`函数来自`kA`函数加上特定偏移; 然后继续查看`kA`函数, 可以知道`kA`函数进行了具体的解码字符串操作, 是根字符串函数.  
![](https://attach.52pojie.cn/forum/202404/19/160304gx8fkqpxe11kvo1w.png)

 ![](https://attach.52pojie.cn/forum/202404/19/160314b4b0nr0aenfex2yf.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/160334idlzpwupk3wlflsk.png)   
通过对`kA`函数进行简单的理解分析, 可以发现决定其字符串返回值的只有第一个参数`r`, 参数`e`是无用的, 第一个参数与字符串的关系是一一对应的.  
首先需要知道`kA`能获取到哪些字符串, 通过对`kA`函数进行分析可以知道, `kA`函数先调用`EA`函数获取一个字符串数组, 取该字符串数组下标为`r-500`的字符串, 然后对该字符串解码得到真实字符串, 解码操作类似 base64, 不过还原成字符的时候每个`n`是取自字符串 "`abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789+/=`" 的下标而不是每个字符的 charCode.  
 ![](https://attach.52pojie.cn/forum/202404/19/160349owdw1z5b5j4kh5jy.png)   
所以我们知道`kA`函数的取值范围是`[500, 500+EA().length]`, 可以在`window.PSign.sign`函数入口处断点中计算好所有字符串和入参的关系.  
 ![](https://attach.52pojie.cn/forum/202404/19/160404m334323l2yuuulbl.png) 

##### 1.1.2 利用 ast 还原字符串函数

首先安装`@babel/parser`, `@babel/types`, `@babel/generator`, `@babel/traverse`, `fs`库

```
npm install @babel/parser @babel/types @babel/generator @babel/traverse fs
```

然后解析将 js 代码解析成 ast 对象并定位到`kA`函数的定义处, 可以先将 js 代码扔到`astexplorer.net`上辅助分析 (关于 babel 如果操作 ast 结构, 可以参考此文章: [https://juejin.cn/post/7045496002614132766](https://juejin.cn/post/7045496002614132766)).

```
const fs = require('fs');
const parser = require("@babel/parser");
const types = require("@babel/types");
const generator = require("@babel/generator").default;
const traverse = require("@babel/traverse").default;

const input = "js_security_v3_0.1.6.js";
const content = fs.readFileSync(input, {encoding: "utf-8"});
const ast = parser.parse(content);

function handleStringFunction(path) {
    const { node } = path;

  // 找到kA函数
  if (node.id.name !== "kA" || node.params.length !== 2) {
        return;
    }

}

traverse(ast, {
    FunctionDeclaration: {
        enter: [
            handleStringFunction
        ]
    }
});
```

然后把`kA`函数得到的字符串数组复制过来, 定义为`kAWords`数组.  
![](https://attach.52pojie.cn/forum/202404/19/160416cg319xbefuh9lax8.png)

接着使用 path.scope 查找所有调用 kA 的地方并将其放到一个 to_visit 的队列中, 便于遍历所有情况, 如果调用 kA 传递的参数不是具体数值还需要将当前调用者所处的函数的所有调用记录加到 to_visit 中, 如果是数值则可以直接得到结果并替换调用节点为字符串节点.  
![](https://attach.52pojie.cn/forum/202404/19/160426cruy2133ygprj329.png)

传入 kA 的参数需要分几种情况分析  
(1) 全是数字, 例如:

```
kA(100, 200)
```

在 astexplorer.net 中分析, 此时可以清楚地知道它的 arguments 中的第一个参数是`NumericLiteral`类型 (对于 kA 函数来说, 它的返回值只跟第一个参数有关, 所以只用看第一个参数就行).  
![](https://attach.52pojie.cn/forum/202404/19/160438xnsl6rvmqpunavub.png)

因此可以写如下函数来获取值:  
 ![](https://attach.52pojie.cn/forum/202404/19/160449hk2g88ddkg8hygga.png) 

(2) 传入参数是变量加偏移. 分析`kA`函数的调用列表可知道, 有很多这种情况.  
![](https://attach.52pojie.cn/forum/202404/19/160502frlafdf861ma1ar4.png)

抓住其中一个调用分析它的 ast 结构, 着重看第一个参数的 ast 结构.  
 ![](https://attach.52pojie.cn/forum/202404/19/160513oa7zyu90s39y924u.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/160520tvz0c3g72983g558.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/160529u999s9bnzhgt9st9.png) 

可以看出来, 第一个参数是一个`BinaryExpression`表达式, 这个表达式的 left 是一个`Identifier`, right 是`NumericLiteral`或`UnaryExpression`, operator 可以是 "-", "+" 之类., 现在我们只需要找到这个 left 和 right 对应的值是多少就计算出该表达式的具体数值. 使用 path.scope 可以获取到作用域内变量的定义和初始值 / 是否常量等信息, 可以找到初始值, 找一个`Identifier`的初始值的方法类似如下代码:

```
function getIdentifierNumber(identifier, scope) {
    // scope: 作用域

    // 得到binding
    const binding = scope.getBinding(identifier.name);
    if (!binding) {
        return;
    }

    // 当前变量是常量且初始值是数值类型则可以得到数值
    if (binding.constant && types.isNumericLiteral(binding.path.node.init)) {
        return binding.path.node.init.value;
    }

      // 初始值是null且只有一个constantViolations且是数值
    if (binding.path.node.init === null && binding.constantViolations.length === 1) {
        const node = binding.constantViolations[0].node;
        if (!types.isAssignmentExpression(node)) {
            return;
        }
        return tryGetArgNumber(node.right, binding.constantViolations[0].scope);
    }
}
```

`BinaryExpression`表达式可以分别计算 left 和 right 的值, 然后根据 operator 计算数值; `isUnaryExpression`可以计算 argument 的值并计算数值.

```
function tryGetArgNumber(arg, scope) {

    // 数值类型直接取value即可
    if (types.isNumericLiteral(arg)) {
        return arg.value;
    }

    // 变量类型查找所在变量的binding, 并获取初始值
    if (types.isIdentifier(arg)) {
        return getIdentifierNumber(arg, scope);
    }

    // UnaryExpression类型只需要获取UnaryExpression的argument并进行operator操作就可以得到数值
    if (types.isUnaryExpression(arg)) {
        const argumentValue = tryGetArgNumber(arg.argument, scope);
        if (typeof argumentValue !== "number") {
            return;
        }
        return eval(`${arg.operator} ${argumentValue}`)
    }

          // 计算left和right的值
    if (types.isBinaryExpression(arg)) {

        let leftValue = tryGetArgNumber(arg.left, scope);
        const rightValue = tryGetArgNumber(arg.right, scope);
        if (typeof leftValue !== "number" || typeof rightValue !== "number") {
            return ;
        }
        return eval(`${leftValue} ${arg.operator} ${rightValue}`);
    }

}
```

对于传入参数是 param 类型即传入的是当前函数的参数, 可以获取当前函数的所有调用记录并加上一个偏移值, 然后继续加入到 to_visit 数组中遍历.  
举个例子, 有如下函数定义:

```
function ob(e, t) {
  return kA(e + 100, t);
}

const result = ob(100, 0);
```

那么`result=ob(100, 0)=kA(200, 0)="逆向研究点滴"`

示例解析代码如下:

```
function getIdentifierNumber(identifier, scope) {
    // scope: 作用域

    // 得到binding
    const binding = scope.getBinding(identifier.name);
    if (!binding) {
        return;
    }

    // 当前变量是常量且初始值是数值类型则可以得到数值
    if (binding.constant && types.isNumericLiteral(binding.path.node.init)) {
        return binding.path.node.init.value;
    }

    // 初始值是null且只有一个constantViolations且是数值
    if (binding.path.node.init === null && binding.constantViolations.length === 1) {
        const node = binding.constantViolations[0].node;
        if (!types.isAssignmentExpression(node)) {
            return;
        }
        return tryGetArgNumber(node.right, binding.constantViolations[0].scope);
    }
}

function tryGetArgNumber(arg, scope) {

    // 数值类型直接取value即可
    if (types.isNumericLiteral(arg)) {
        return arg.value;
    }

    // 变量类型查找所在变量的binding, 并获取初始值
    if (types.isIdentifier(arg)) {
        return getIdentifierNumber(arg, scope);
    }

    // UnaryExpression类型只需要获取UnaryExpression的argument并进行operator操作就可以得到数值
    if (types.isUnaryExpression(arg)) {
        const argumentValue = tryGetArgNumber(arg.argument, scope);
        if (typeof argumentValue !== "number") {
            return;
        }
        return eval(`${arg.operator} ${argumentValue}`)
    }

    // 计算left和right的值
    if (types.isBinaryExpression(arg)) {

        let leftValue = tryGetArgNumber(arg.left, scope);
        const rightValue = tryGetArgNumber(arg.right, scope);
        if (typeof leftValue !== "number" || typeof rightValue !== "number") {
            return ;
        }
        return eval(`${leftValue} ${arg.operator} ${rightValue}`);
    }

    if (types.isMemberExpression(arg)) {
        let objectBinding = scope.getBinding(arg.object.name);
        let objectInit = objectBinding.path.node.init;
        // 一直查找到初始化值的地方
        while (types.isIdentifier(objectInit)) {
            objectBinding = objectBinding.scope.getBinding(objectInit.name);
            objectInit = objectBinding.path.node.init;
        }

        const targetName = arg.property.name;

        if (types.isObjectExpression(objectInit)) {
            const properties = objectInit.properties;
            for (const p of properties) {
                if (p.key.name === targetName) {
                    // 找到object类型的数值
                    return tryGetArgNumber(p.value, objectBinding.scope);
                }
            }

            // 可能object的属性在赋值语句中设置的, 遍历一下所有赋值语句
            for (const ref of objectBinding.referencePaths) {
                // 不是赋值语句跳过
                if (!types.isAssignmentExpression(ref.parentPath.parent)) {
                    continue;
                }
                const assign = ref.parentPath.parent;
                if (!types.isMemberExpression(assign.left)) {
                    continue;
                }
                if ((assign.left.property.name || assign.left.property.value) === targetName) {
                    return tryGetArgNumber(assign.right, ref.scope);
                }
            }

        }
    }

}

function handleStringFunction(path) {
    const { node } = path;

    // 找到kA函数
    if (node.id.name !== "kA" || node.params.length !== 2) {
        return;
    }

          // 这里仅是示例代码, 不提供完整kAWords数组, 按照前面的方法可以得到这个kAWords数组并填充进来
    const kAWords = [];

    // 得到当前kA函数定义处所在的父类作用域中查找kA函数的binding
    const binding = path.parentPath.scope.getBinding(node.id.name);
    // 找出所有引用stringFunction的地方
    const references = binding.referencePaths;
    const to_visit = [];

    for (const ref of references) {
        // 1. 若引用kA的parent是一个CallExpression, 查看其第一个参数, 根据第一个参数的类型来决定
        if (types.isCallExpression(ref.parent)) {
            to_visit.push([ref.parentPath, ref.parent, 0, ref.parentPath.scope, 0]);
        }
    }

    while (to_visit.length > 0) {
        const [callExpressionPath, callExpression, argIndex, scope, offset] = to_visit.shift();
        const targetArg = callExpression.arguments[argIndex];
        const value = tryGetArgNumber(targetArg, scope);
        if (typeof value === "number") {
            const word = kAWords[value+offset];
            if (word) {
                callExpressionPath.replaceWith(types.stringLiteral(word));
                console.log(`${generator(callExpression).code} 变更为字符串: ${word}`);
            } else {
                debugger;
            }
        } else if(types.isBinaryExpression(targetArg) && types.isIdentifier(targetArg.left)) {
            // targetArg是BinaryExpression且left是param类型变量, 则需要继续找到所在函数并添加所有该函数的调用到to_visit中继续查找.
            const leftBinding = scope.getBinding(targetArg.left.name);
            const rightValue = tryGetArgNumber(targetArg.right, scope);
            if (leftBinding.kind === "param" && typeof rightValue === "number") {
                const func = scope.getFunctionParent();
                const funcName = func.path.node.id.name;

                const funcBinding = func.path.parentPath.scope.getBinding(funcName);
                const funcReferences = funcBinding.referencePaths;

                // 计算left是func的第几个参数
                const leftIndex = func.path.node.params.indexOf(leftBinding.path.node);

                // 计算新的偏移值
                const newOffset = eval(`${targetArg.operator} ${rightValue}`) + offset;

                for (const ref of funcReferences) {
                    if (types.isCallExpression(ref.parent)) {
                        to_visit.push([ref.parentPath, ref.parent, leftIndex, ref.parentPath.scope, newOffset]);
                    }
                }
            }
        }

    }
}
```

替换完成后写入新的 js 文件

```
const { code } = generator(ast);

fs.writeFileSync("js_security_v3_0.1.6.example.js", code);
```

![](https://attach.52pojie.cn/forum/202404/19/160547q5xw5d851azddhgz.png)

 ![](https://attach.52pojie.cn/forum/202404/19/160555knwdzo2rgdwdeldg.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/160612a191pz9ccpdcpdnj.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/160625cbk5kuwg7e5aouec.png) 

可以查看前后文件的对比, 可以知道已经对很多字符串函数替换为字符串, 不过也有一部分没有替换成功, 分析可以发现并不是所有字符串函数都是基于`kA`函数得到, 不过解决思路是一致的, 仅需要把 kAWords 数组替换成对应的即可, 替换完成后效果如下:  
![](https://attach.52pojie.cn/forum/202404/19/160634vq7jha3jtk3knkn3.png)

然后稍微整理一下一些跟字符串相关的看上去很费解的结构, 例如多个字符串相加或者 MemberExpression 的 property 是一个 sequenceExpression 但是实际有用的只有最后的字符串.  
 ![](https://attach.52pojie.cn/forum/202404/19/160645rzqiie0264e420d2.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/160653tigszblll1x0zxq2.png) 

```
function handleBinaryExpressionString(path) {
// 去除多个字符串相加或常量相加的情况, 直接使用path.evaluate()即可. babel会计算好结果.
  const { confident, value } = path.evaluate();
    if (confident) {
        path.replaceInline(types.valueToNode(value));
    }
}

// 去除MemberExpression的property是一个sequenceExpression的情况
function removeSequenceExpressionForString(path) {
    const { node } = path;
    if (!types.isSequenceExpression(node.property)) {
        return;
    }
    const expressions = node.property.expressions;
    if (!types.isStringLiteral(expressions[expressions.length - 1])) {
        return;
    }
    path.replaceWith(types.memberExpression(node.object, expressions[expressions.length - 1], true));
}
```

![](https://attach.52pojie.cn/forum/202404/19/160704styyzv89nevytn9e.png)

以下是替换之后的效果:  
 ![](https://attach.52pojie.cn/forum/202404/19/160711tr5e5y5xefme3xrf.png) 

#### 1.2 还原控制流

继续分析字符串函数还原后的代码可以知道, 代码中有很多如下这种格式的控制流结构, for-switch 的形式, switch 的是一个字符数组, 根据匹配的字符顺序进行还原即可.  
![](https://attach.52pojie.cn/forum/202404/19/160721b7tgcrtcg53rzzr7.png)

首先找出用于 switch 的字符串数组:

```
function handleControlFlow(path) {
    const { node } = path;
    if (!types.isBlockStatement(node.body) || node.body.body.length < 1 || !types.isSwitchStatement(node.body.body[0])) {
        return;
    }
    const switchStatement = node.body.body[0];
    if (!types.isMemberExpression(switchStatement.discriminant)) {
        return;
    }

    // 找到discriminant变量的binding, 从而获得初始值.
    const stepVarBinding = path.scope.getBinding(switchStatement.discriminant.object.name);
    const stepVar = stepVarBinding.path.node.init.callee.object;

   // tryGetValue跟上面的tryGetArgNumber原理差不多, 不过这里拿的是stringLiteral类型.
    const value = tryGetValue(stepVarBinding.scope, stepVar);

    if (!value) {
        return;
    }

    if (typeof value !== "string") {
        return;
    }

          // 得到字符数组
    const steps = value.split("|");
}
```

拿到字符数组之后只需要按照字符数组的顺序重新编排代码块顺序即可, 然后替换整个 for 循环:

```
function handleControlFlow(path) {
    // ...上面的代码

    const cases = {};
    for (const c of switchStatement.cases) {
        cases[c.test.value] = c;
    }

    const blocks = [];

    for (const step of steps) {
        const switchCase = cases[step];
        let consequent = switchCase.consequent;
        const lastNode = consequent[consequent.length - 1];
        if (types.isContinueStatement(lastNode) || types.isBreakStatement(lastNode)) {
            consequent = consequent.slice(0, consequent.length - 1);
        }
        blocks.push(...consequent);
    }

    path.replaceWithMultiple(blocks);

}

// ...
traverse(ast, {
    ForStatement: {
        exit: [
            handleControlFlow
        ]
    }
});
```

前后代码对比:  
![](https://attach.52pojie.cn/forum/202404/19/160731vlwmsj9qsmsg0lg9.png)

#### 1.3 还原仅有一句表达式代码的函数

![](https://attach.52pojie.cn/forum/202404/19/160741kzfogf9a8z9bmrpg.png)

 ![](https://attach.52pojie.cn/forum/202404/19/160749vvq7jkvxgujrvfpi.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/160756bpplfco8oiicwl0z.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/160804elxpilfsfspfgsg2.png) 

全文 js 代码中有大量像上图`xtLZB`这样的函数, 这些函数作用仅仅是对传入的参数做某一种操作并且只有一行 return 代码, 它们的目的仅仅是用于增加代码的阅读难度, 所以可以将这些函数调用的节点替换为 return 中表达式的节点.  
例如:

```
var ut = {
        xtLZB: function (t, r, n, e) {
                return t(r, n, e);
              }
};

// ...
Xt = ut.xtLZB(Fg, Jt, null, 2)
```

替换为:

```
Xt = Fg(Jt, null, 2);
```

去除`xtLZB`函数调用后, 代码看上去会简洁很多.  
首先先筛选出符合上述特征的函数调用: 函数是只有一行代码且是 return 语句或者除了 return 语句外, 还有一行是无用的 var 语句.

```
function handleSimpleCallExpression(path) {

    const {node} = path;
    if (!types.isMemberExpression(node.callee) || !types.isIdentifier(node.callee.object)) {
        return;
    }
    const calleeBinding = path.scope.getBinding(node.callee.object.name);
    if (!calleeBinding) {
        return;
    }
    let func = null;
    let funcScope = null;

    function tryGetFunction(v, scope, targetName, binding) {
        if (types.isIdentifier(v)) {
            binding = scope.getBinding(v.name);
            if (binding.kind === "param") {
                return;
            }
            return tryGetFunction(binding.path.node.init, binding.scope, targetName, binding);
        }
        if (types.isObjectExpression(v)) {
            const properties = v.properties;
            for (const pro of properties) {
                if (pro.key.name === targetName) {
                    return [pro.value, scope];
                }
            }

            if (!binding) {
                return ;
            }

            // 从赋值语句中查找
            for (const ref of binding.referencePaths) {
                if (!types.isAssignmentExpression(ref.parentPath.parent)) {
                    continue;
                }
                const assign = ref.parentPath.parent;
                if (!types.isMemberExpression(assign.left)) {
                    continue;
                }
                if ((assign.left.property.name || assign.left.property.value) === targetName) {
                    return [assign.right, ref.parentPath.parentPath.scope];
                }
            }
        }
    }

    const targetName = node.callee.property.name || node.callee.property.value;
    const info = tryGetFunction(calleeBinding.path.node.init, calleeBinding.scope, targetName);
    if (Array.isArray(info)) {
        func = info[0];
        funcScope = info[1];
    }

    if (func === null || funcScope === null) {
        return;
    }
    if (!types.isFunctionExpression(func)) {
        return;
    }
}
```

之后开始替换这些函数调用节点为表达式节点:

```
function handleSimpleCallExpression(path) {
  // ... 上面的代码
  // 寻找function的path
    let funcPath = null;
    funcScope.path.traverse({
        FunctionExpression: function (mpath) {
            const {node: mnode} = mpath;
            if (mnode === func) {
                funcPath = mpath;
                mpath.skip();
            }
        }
    });

    if (funcPath === null) {
        return;
    }

    // 得到传入参数节点和函数参数节点的对应关系, 准备将return表达式中的所有设计这些函数参数的替换为实际的传入参数.
    const paramToArgument = {};
    for (let i = 0; i < func.params.length; i++) {
        const param = func.params[i];
        const argument = node.arguments[i];
        if (typeof argument === "undefined") {
            break;
        }
        paramToArgument[param.name] = argument;
    }

    const beforeCode = generator(node).code;

    // 替换先实现保留原来的function定义
    const originalFunc = types.cloneNode(func, true);

    // 遍历替换对应节点
    funcPath.traverse({
        Identifier: function (mpath) {
            const { node: mnode } = mpath;
            if (paramToArgument[mnode.name]) {
                mpath.replaceWith(paramToArgument[mnode.name]);
                mpath.skip();
            }
        }
    });

    const afterCode = generator(funcPath.node.body.body[0].argument).code;

    // 替换函数调用为return语句中的argument表达式
    path.replaceInline(returnStatement.argument);

    // 还原原来的function
    funcPath.replaceInline(originalFunc);

    console.log(`简化函数调用: ${beforeCode.slice(0, 10)}... -> ${afterCode.slice(0, 10)}...`);
}
```

前后对比:  
![](https://attach.52pojie.cn/forum/202404/19/160814mxyuc8x6nqcxeeua.png)

### 参数分析

去混淆之后 Enable Local Overrides, 将原 js_security_v3_0.1.6.js 替换为去混淆之后的 js 代码.  
![](https://attach.52pojie.cn/forum/202404/19/160822mwcqjz1vz08y0bsj.png)

全文搜索 "h5st", 得到两处相关代码, 通过分析知道 h5st 来自`__genSignParams`函数, 打上断点分析.  
 ![](https://attach.52pojie.cn/forum/202404/19/160831mkcjwlwjiffycj7w.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/160840i0k5t5dd7o5uh1tp.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/160847muotoue0fa5eteno.png)   
`__genSignParams`函数是对传入的 A, C, c, r 进行拼接, 其中关键代码:

```
var C = pw();
var c = Db(C, "yyyyMMddhhmmssSSS");
var v = c + "22";
var s = this["__genKey"](this["_token"], this["_fingerprint"], v, this["_appId"], this["algos"])["toString"]();
var A = this.__genSign(s, t);
```

另外 r 和 t 参数都是传入参数.  
![](https://attach.52pojie.cn/forum/202404/19/160859tz4r5qyuyoddwguy.png)

依次对这些变量进行研究

#### 2.1 pw 函数

如下, 是`Date.now`函数:  
![](https://attach.52pojie.cn/forum/202404/19/160910me3z3pl6saix9e6e.png)

#### 2.2 c 参数

```
var c = Db(C, "yyyyMMddhhmmssSSS");
```

![](https://attach.52pojie.cn/forum/202404/19/160920jtokec9pmu2miqkb.png)

可以明显看出是获取 C 变量 (也就是当前时间) 的日期字符串, 格式是`yyyyMMddhhmmssSSS`.

#### 2.3 s 参数

```
var s = this["__genKey"](this["_token"], this["_fingerprint"], v, this["_appId"], this["algos"])["toString"]();
```

![](https://attach.52pojie.cn/forum/202404/19/160930fgbvwqp4fibxyiyz.png)

简单跟一下`__genKey`函数代码, 发现是 VM 加载的 js 代码, 这段代码定义了使用哪种 hamc 签名算法, 在`js_security_v3_0.1.6.js`代码中搜一下关键词`__genKey`可以找到加载代码的位置.  
![](https://attach.52pojie.cn/forum/202404/19/160940fd1hdhdadodaq2pz.png)

在该位置打上断点, 分析下该代码的来源.  
 ![](https://attach.52pojie.cn/forum/202404/19/161026q0fe22787a72vz8q.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/161033pefqvfeu5czeeq5y.png)   
可以知道, 这段 js 代码来自 storage 的`WQ_dy_algo_s_f06cc_4.3`, 另外的`WQ_dy_tk_s_f06cc_4.3`是 token. 把当前 local storage 清空, 刷新请求, 可以发现有个请求返回了这些内容:  
 ![](https://attach.52pojie.cn/forum/202404/19/161043spzk1myhhjvz4x73.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/161057uf8ty78fi2i47lub.png)   
可以全文搜索解混淆之后的 js 代码, 关键词为请求体中的`expandParams`, 分析其中所有参数的生成逻辑, 就是一些环境检测的参数, 然后使用了 aes 加密得到 hex 字符串, 在此不详细展开了.  
 ![](https://attach.52pojie.cn/forum/202404/19/161105r8qt6lv58uvq8v20.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/161114stf8788q5sz887q5.png) 

#### 2.4 A 参数

```
var A = this.__genSign(s, t);
```

![](https://attach.52pojie.cn/forum/202404/19/161123wtp5rc76ggy7pg79.png)

 ![](https://attach.52pojie.cn/forum/202404/19/161131xpyppiijy0jy4bjr.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/161139sxhua076ypnnjyuj.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/161149sy92y602xa9ooye9.png) 

通过跟踪`__genSign`->`Zb`知道, A 是计算拼接字符串的 HmacSHA256 签名, 其中 salt 是`2.3 s参数`, 字符串由传入参数 t 拼接得到, 拼接示例数据如下:

```
[{"key":"appid","value":"search-pc-java"},{"key":"body","value":"4003786fdc49eae4d371309b4e42395f38e243887e470fe2cef2733dc03581c3"},{"key":"client","value":"pc"},{"key":"clientVersion","value":"1.0.0"},{"key":"functionId","value":"mixerOut"},{"key":"t","value":1713089999455}]
```

拼接成`appid:search-pc-java&body:xxxxxx&client:pc&clientVersion:1.0.0&functionId:mixerOut&t:1713089999455`

#### 2.5 t 参数

![](https://attach.52pojie.cn/forum/202404/19/161214jdpd7zvpdv7pij77.png)

 ![](https://attach.52pojie.cn/forum/202404/19/161226hv0bb7b9lte9f7vb.png)   
t 是传入的参数, 在调用栈上一层知道来自 V=t["sent"], 当前在 case 8 处, 它的上一步是 case 5, 在 case 5 这段代码处打上断点, 根据这个 Promise 函数的特性, 每一步的 sent 参数是由上一步的`t["abrupt"]("return", xxx);`得到的, 因此在这个`abrupt`函数上打上断点, 在 case 5 到 case 8 之间停留的最后一次`abrupt`函数就是 V 变量设置的代码位置 (注意是最后一次, 因为代码中可能有多处这种 Promise 函数调用).  
 ![](https://attach.52pojie.cn/forum/202404/19/161240c2982x9a8cbbuxl8.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/161249ug9uhoszbguc4oxk.png) 

经过 case5 到 case8 的最后一次`abrupt`的断点, 可以看到`V=t["sent"]`的生成代码逻辑:  
![](https://attach.52pojie.cn/forum/202404/19/161301pgx34m8113431hgh.png)

 ![](https://attach.52pojie.cn/forum/202404/19/161311w44e3r81zessvrj4.png)   
 ![](https://attach.52pojie.cn/forum/202404/19/161320zpdy777dgvdppb3v.png)   
经过分析可以知道, 这是对一个 K 变量字符串进行 aes 加密, aes key 是 "&d74&yWoV.EYbWbZ", iv 是`["01", "02", "03", "04", "05", "06", "07", "08"]["join"]("")`得到. 其中 K 是一些环境检测:  
 ![](https://attach.52pojie.cn/forum/202404/19/161330u4x6d65vdyg20z65.png)   
通过全文搜索 "sua" 关键词可以找到这些环境检测代码逻辑所在, 在此不详细展开了:  
 ![](https://attach.52pojie.cn/forum/202404/19/161340f77zgupopr4pxtk6.png)   
值得注意的是参与 h5st 时的环境检测变量只有: sua/pp/extend/random/v/fp 这几个.  
以下是检测的环境参数:

<table><thead><tr><th>参数名</th><th>类型</th><th>说明</th></tr></thead><tbody><tr><td>wc</td><td>number</td><td>是否 Chrome 浏览器 (是: 1, 否: 0)</td></tr><tr><td>wd</td><td>number</td><td>是否有 webdriver 属性 (是: 1, 否: 0)</td></tr><tr><td>l</td><td>string</td><td>当前语言: navigator["language"]</td></tr><tr><td>ls</td><td>string</td><td>支持的语言列表: navigator["languages"].join(",")</td></tr><tr><td>ml</td><td>number</td><td>navigator["mimeTypes"]["length"]</td></tr><tr><td>pl</td><td>number</td><td>插件数量: navigator["plugins"].length</td></tr><tr><td>av</td><td>number</td><td>navigator.appVersion</td></tr><tr><td>ua</td><td>string</td><td>window["navigator"]["userAgent"]</td></tr><tr><td>sua</td><td>string</td><td>使用正则 RegExp("Mozilla/5.0 <span class="MathJax" id="MathJax-Element-1-Frame" tabindex="0" style="position: relative;" data-mathml="<math xmlns=&quot;http://www.w3.org/1998/Math/MathML&quot;><mo stretchy=&quot;false&quot;>(</mo><mo>.</mo><mo>&amp;#x2217;</mo><mo>?</mo><mo stretchy=&quot;false&quot;>)</mo></math>" role="presentation"><nobr aria-hidden="true">(.∗?)</nobr><math xmlns="http://www.w3.org/1998/Math/MathML"><mo stretchy="false">(</mo><mo>.</mo><mo>∗</mo><mo>?</mo><mo stretchy="false">)</mo></math><script type="math/tex" id="MathJax-Element-1">(.*?)</script>") 取出 useragent 内容</span></td></tr><tr><td>pp</td><td>object</td><td>取 cookie 对应值, 没有就不设置, p1: pwdt_id, p2: pin, p3: pt_pin</td></tr><tr><td>extend</td><td>object</td><td>环境相关检测</td></tr></tbody></table>

extend 检测的关键逻辑代码为:

```
var  Sr = {};

Sr.wd = window["navigator"]["webdriver"] ? 1 : 0;

Sr.l = navigator.languages && 0 !== navigator["languages"]["length"] ? 0 : 1;

Sr.ls = navigator["plugins"]["length"];

Br = 0;
("cdc_adoQpoasnfa76pfcZLmcfl_Array" in window || "cdc_adoQpoasnfa76pfcZLmcfl_Promise" in window || "cdc_adoQpoasnfa76pfcZLmcfl_Symbol" in window) && (Br |= 1);
("$chrome_asyncScriptInfo" in window["document"] || "$cdc_asdjflasutopfhvcZLmcfl_" in window["document"]) && (Br |= 2);
Sr.wk = Br;

Sr["bu1"] = "0.1.6";

Or = 0;
Er = -1 !== gg(_r = window.location["host"]).call(_r, "sz.jd.com") || gg(jr = window.location["host"])["call"](jr, "ppzh.jd.com") !== -1;
Er && gg(Lr = document["body"]["innerHTML"]).call(Lr, "diantoushi.com") !== -1 && (Or |= 1);
Er && -1 !== gg(Mr = document.body["innerHTML"])["call"](Mr, "xiaowangshen.com") && (Or |= 2);
Sr["bu2"] = Or;

Sr["bu3"] = document["head"]["childElementCount"];

kr = 0;
Tr = typeof process !== "undefined" && process["release"] != null && process["release"].name === "node";
Pr = typeof process !== "undefined" && process["versions"] != null && process["versions"]["node"] != null;
Ir = typeof Deno !== "undefined" && typeof Deno.version !== "undefined" && void 0 !== Deno["version"]["deno"];
Wr = typeof Bun !== "undefined";
(Tr || Pr) && (kr |= 1);
Ir && (kr |= 2);
Wr && (kr |= 4);
Sr["bu4"] = kr;
```

### 感悟总结

h5st 整体难度不高, 不大熟悉如何使用 babel 操作 ast 的朋友可以用来练手熟悉. 本人最近才开始写逆向文章, 写作思路不是很熟练, 望海涵.  
ps: 吾爱破解论坛的 markdown 的图片链接貌似不会自动上传到站内图片链接, 凑活用.

![](https://avatar.52pojie.cn/data/avatar/000/00/00/01_avatar_middle.jpg)Hmily [@kylin1020](https://www.52pojie.cn/home.php?mod=space&uid=2244508) 很抱歉，论坛 Markdown 并不会自行去下载图片，这有很大的安全隐患，并且论坛已经强制走了 SSL 加密，由于你帖子使用的图床是 http 的，浏览器会直接禁止加载，最起码图床需要支持 https，最好是可以把图片上传论坛本地，贴图的方法参看：[https://www.52pojie.cn/misc.php? ... &id=29&messageid=36](https://www.52pojie.cn/misc.php?mod=faq&action=faq&id=29&messageid=36) ，这次我帮你编辑上传了，下次可以自行处理一下。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)cmsttasd ast 一直不得要领啊，mark 一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) cld61 ast 一直不得要领啊，mark 学习一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) grant73 ast 一直不得要领啊，mark 学习一下 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) tunnel213 大佬的感悟总结对小白是杀人诛心啊 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) pojie20230721 看不大懂, 收藏待学 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) BonnieRan 哇~ ast 解混淆的过程很详细，如楼主说的正好拿来练手熟悉 babel 操作 ast![](https://avatar.52pojie.cn/images/noavatar_middle.gif)LittleHedgehog 收藏学习 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) WuAi2024AiWu 厉害 &#128077;![](https://avatar.52pojie.cn/images/noavatar_middle.gif)yyysss153 感谢分享