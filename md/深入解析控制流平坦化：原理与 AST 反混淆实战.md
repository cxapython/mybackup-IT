> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/_kofRwDiMjl4ig1wmlfIqQ?scene=1&click_id=29)

       国庆结束，继续回来码文章，之前我们研究了 Akamai 的反爬逻辑，接下来纠结了好久，还是准备分享一下之前搞某里的多层控制流平坦化的经验。今天先水一期原理，让大家先对控制流平坦化有一定的了解，方便之后文章的阅读。
=================================================================================================================

     在逆向工程与代码保护领域，控制流平坦化（Control Flow Flattening）是一种常见的代码混淆技术，它通过打乱程序正常的执行流程，增加逆向分析的难度。而随着混淆技术的升级，多层控制流平坦化更是让不少开发者和安全研究员望而却步。本文将从基础原理出发，带大家逐步理解控制流平坦化的 “套路”，并结合实例讲解如何利用 AST（抽象语法树）实现多层控制流平坦化的反混淆，让混乱的代码 “回归正轨”。

一、控制流平坦化：让代码 “迷路” 的混淆术
----------------------

在了解反混淆之前，我们首先要搞清楚：**控制流平坦化到底是什么？它是如何让代码变得难以分析的？**

### 1.1 正常代码的控制流：清晰的 “路线图”

正常情况下，代码的执行流程就像一张清晰的路线图。比如一个包含条件判断、循环的函数，执行路径会按照逻辑顺序展开 ——if 条件满足就走分支 A，不满足就走分支 B；for 循环会重复执行直到条件不成立。我们可以通过流程图轻松看出代码的逻辑走向，例如：

```
function calculate(a, b) {
  let result = 0;
  if (a > b) {
    result = a - b; // 分支1
  } else {
    result = b - a; // 分支2
  }
  for (let i = 0; i < result; i++) {
    result *= 2; // 循环逻辑
  }
  return result;
}
```

```
function calculate(a, b) {
  let result = 0;
  let state = 0; // 调度器的“状态变量”，控制执行顺序
  // 平坦化的循环调度器
  while (true) {
    switch (state) {
      case 0:
        // 原代码的条件判断（拆成基本块1）
        if (a > b) {
          state = 1; // 满足条件，下一步执行case1
        } else {
          state = 2; // 不满足条件，下一步执行case2
        }
        break;
      case 1:
        // 原代码的分支1（基本块2）
        result = a - b;
        state = 3; // 下一步执行case3（循环初始化）
        break;
      case 2:
        // 原代码的分支2（基本块3）
        result = b - a;
        state = 3; // 下一步执行case3
        break;
      case 3:
        // 原代码的循环初始化（基本块4）
        let i = 0;
        state = 4; // 下一步执行case4（循环条件判断）
        break;
      case 4:
        // 原代码的循环条件（基本块5）
        if (i < result) {
          state = 5; // 满足循环条件，执行case5（循环体）
        } else {
          state = 6; // 不满足，执行case6（返回结果）
        }
        break;
      case 5:
        // 原代码的循环体（基本块6）
        result *= 2;
        i++;
        state = 4; // 回到case4，再次判断循环条件
        break;
      case 6:
        // 原代码的返回（基本块7）
        return result;
      default:
        state = 6; // 异常情况，直接返回
        break;
    }
  }
}
```

这段代码的控制流非常直观：先做条件判断，再执行循环，最后返回结果，每个步骤的顺序和依赖关系清晰可见。

### 1.2 控制流平坦化的核心思想：“打乱路线，统一调度”

控制流平坦化的本质，是**将代码的多个逻辑块（分支、循环体等）拆分成独立的 “基本块”，然后用一个 “调度器” 来控制这些基本块的执行顺序**。原本清晰的 “路线图” 被拆成零散的 “节点”，再通过调度器强行串联，从而隐藏代码的真实逻辑。

我们以刚才的 calculate 函数为例，看看平坦化后的效果（伪代码）：

```
function compute(x, y) {
  let a = x * x;
  let b = y * y;
  let sum = 0;
  // 第一层调度器：state1控制外层流程
  let state1 = 0;
  while (true) {
    switch (state1) {
      case 0:
        // 第二层调度器：state2控制内层流程（嵌套平坦化）
        let state2 = 0;
        while (true) {
          switch (state2) {
            case 0:
              sum = a + b;
              state2 = 1;
              break;
            case 1:
              if (sum % 2 === 0) {
                state2 = 2;
              } else {
                state2 = 3;
              }
              break;
            case 2:
              console.log("偶数");
              state1 = 1; // 内层结束，外层下一步执行case1
              return sum;
            case 3:
              console.log("奇数");
              state1 = 1; // 内层结束，外层下一步执行case1
              return sum;
          }
        }
        break;
      case 1:
        console.log("计算完成");
        state1 = 2;
        break;
      case 2:
        return sum;
    }
  }
}
```

```
npm install @babel/parser @babel/traverse @babel/generator @babel/types
```

对比两段代码可以发现，平坦化后的代码有两个核心特征：

1.  **统一的调度器**
    
    ：用 while(true)+switch(state) 构成循环调度器，所有基本块都被塞进 switch 的 case 中；
    

2.  **状态变量驱动**
    
    ：通过 state 变量控制下一个执行的 case，原本的逻辑依赖被替换成 state 的赋值语句，代码的真实流程被隐藏。
    

此时再看代码的控制流，已经变成了 “调度器→caseN→更新 state→调度器→caseM” 的循环，原本清晰的分支和循环逻辑被彻底打乱，逆向分析时需要不断跟踪 state 的变化才能还原流程，难度大大增加。

### 1.3 多层控制流平坦化：混淆的 “升级版”

单层平坦化已经足够让代码变得混乱，而多层平坦化则是 “混淆叠混淆”——**在已经平坦化的代码基础上，对调度器或基本块再次进行平坦化**。例如：

第一层：用 switch(state1) 调度多个基本块，其中某个 case 里又包含一个 switch(state2) 的子调度器；

第二层：子调度器的 case 里可能还嵌套了第三个 switch(state3)，形成 “套娃” 结构。

这种多层结构会让控制流变得更加复杂，仅靠人工跟踪 state 变量几乎不可能还原逻辑，此时就需要借助 AST 工具进行自动化反混淆。

二、AST 反混淆：让代码 “回归正轨” 的核心工具
--------------------------

AST（抽象语法树）是代码的 “结构化表示”—— 它将代码的语法结构转化为树形节点，例如 if 语句对应 IfStatement 节点，switch 语句对应 SwitchStatement 节点。反混淆的本质，就是通过修改 AST 的节点结构，去除混淆代码（如调度器、多余变量），还原代码的原始逻辑。

对于控制流平坦化的反混淆，AST 的核心作用是：**识别调度器结构→提取基本块→根据 state 变量的流转关系，重新串联基本块→删除多余的调度器代码**。

### 2.1 AST 反混淆的核心步骤（通用）

无论单层还是多层平坦化，AST 反混淆的核心思路都可以概括为以下四步：

1.  **识别调度器特征**
    
    ：找到代码中的 “循环 + switch+state 变量” 结构（平坦化的标志性特征）；
    

2.  **收集基本块与 state 映射**
    
    ：提取 switch 中所有 case 对应的基本块，记录每个基本块执行后 state 的下一个值（即 “从 caseA 到 caseB” 的映射关系）；
    

3.  **还原控制流顺序**
    
    ：根据 state 的映射关系，将零散的基本块按原始执行顺序串联起来；
    

4.  **删除混淆代码**
    
    ：移除 while 循环、switch 结构和多余的 state 变量，生成还原后的代码。
    
      
    

三、实战：用 AST 反混淆多层控制流平坦化
----------------------

理论讲完，我们结合一个真实的多层平坦化 JavaScript 代码实例，用 Babel（JavaScript 的 AST 工具链）演示反混淆过程。

### 3.1 实例：多层平坦化的混淆代码

首先看一段经过两层控制流平坦化的混淆代码（简化版），核心逻辑是 “计算两个数的平方和并判断是否为偶数”：

```
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const generator = require("@babel/generator").default;
const t = require("@babel/types");
// 1. 混淆代码（输入）：待解析的多层平坦化代码
const obfuscatedCode = `function compute(x, y) {
  let a = x * x;
  let b = y * y;
  let sum = 0;
  let state1 = 0;
  while (true) {
    switch (state1) {
      case 0:
        let state2 = 0;
        while (true) {
          switch (state2) {
            case 0:
              sum = a + b;
              state2 = 1;
              break;
            case 1:
              if (sum % 2 === 0) {
                state2 = 2;
              } else {
                state2 = 3;
              }
              break;
            case 2:
              console.log("偶数");
              state1 = 1;
              return sum;
            case 3:
              console.log("奇数");
              state1 = 1;
              return sum;
          }
        }
        break;
      case 1:
        console.log("计算完成");
        state1 = 2;
        break;
      case 2:
        return sum;
    }
  }
}`;
// 2. 解析代码生成AST：核心配置与节点结构解析
const ast = parser.parse(obfuscatedCode, {
  sourceType: "script", // 关键配置1：指定代码类型为"script"（普通脚本）
  // 为什么不用"module"？因为示例代码是独立函数（无import/export），若设为"module"会报错
  plugins: ["jsx", "flow"] // 关键配置2：启用额外语法插件，避免解析非标准语法时失败
  // 此处"jsx"和"flow"插件是通用配置，即使代码中无JSX/Flow语法，也不会影响解析结果
});
// （新增）查看AST结构：通过打印关键节点，理解代码如何被转化为语法树
console.log("=== AST根节点结构 ===");
console.log("根节点类型：", ast.type); // 输出"File"，表示AST的根节点是文件级节点
console.log("文件内容节点类型：", ast.program.type); // 输出"Program"，存储代码的顶层语句
console.log("顶层语句数量：", ast.program.body.length); // 输出1，因为只有1个function声明
// （新增）查看函数声明节点的详细结构
const functionNode = ast.program.body[0];
console.log("\n=== 函数声明节点结构 ===");
console.log("节点类型：", functionNode.type); // 输出"FunctionDeclaration"，表示函数声明
console.log("函数名：", functionNode.id.name); // 输出"compute"，对应函数名
console.log("参数数量：", functionNode.params.length); // 输出2，对应参数x和y
console.log("参数1类型与名称：", functionNode.params[0].type, functionNode.params[0].name); // 输出"Identifier x"
console.log("参数2类型与名称：", functionNode.params[1].type, functionNode.params[1].name); // 输出"Identifier y"
console.log("函数体类型：", functionNode.body.type); // 输出"BlockStatement"，函数体由{}包裹
console.log("函数体语句数量：", functionNode.body.body.length); // 输出5，对应5句代码（let a=...、let b=...、let sum=0、let state1=0、while循环）
// （新增）查看变量声明节点（以let a = x * x为例）
const varDeclNode = functionNode.body.body[0];
console.log("\n=== 变量声明节点结构（let a = x * x） ===");
console.log("节点类型：", varDeclNode.type); // 输出"VariableDeclaration"，表示变量声明
console.log("声明关键字：", varDeclNode.kind); // 输出"let"，对应变量声明关键字
console.log("声明变量数量：", varDeclNode.declarations.length); // 输出1，当前仅声明a变量
const aDecl = varDeclNode.declarations[0];
console.log("变量名节点类型：", aDecl.id.type); // 输出"Identifier"，变量名是标识符
console.log("变量名：", aDecl.id.name); // 输出"a"
console.log("变量赋值表达式类型：", aDecl.init.type); // 输出"BinaryExpression"，二元运算表达式（x*x是乘法运算）
console.log("运算符号：", aDecl.init.operator); // 输出"*"，乘法运算符
console.log("左操作数类型与值：", aDecl.init.left.type, aDecl.init.left.name); // 输出"Identifier x"
console.log("右操作数类型与值：", aDecl.init.right.type, aDecl.init.right.name); // 输出"Identifier x"
// （新增）查看while循环节点（函数体中第5个语句）
const whileNode = functionNode.body.body[4];
console.log("\n=== While循环节点结构 ===");
console.log("节点类型：", whileNode.type); // 输出"WhileStatement"，表示while循环
console.log("循环条件类型：", whileNode.test.type); // 输出"BooleanLiteral"，表示布尔字面量
console.log("循环条件值：", whileNode.test.value); // 输出true，对应while(true)
console.log("循环体类型：", whileNode.body.type); // 输出"BlockStatement"，表示代码块（{}包裹）
console.log("循环体语句数量：", whileNode.body.body.length); // 输出1，循环体仅包含switch语句
const switchInWhile = whileNode.body.body[0];
console.log("循环体内部语句类型：", switchInWhile.type); // 输出"SwitchStatement"，对应循环内的switch
// （新增）查看switch语句节点（while循环内的switch(state1)）
console.log("\n=== Switch语句节点结构（switch(state1)） ===");
console.log("节点类型：", switchInWhile.type); // 输出"SwitchStatement"
console.log("判断变量节点类型：", switchInWhile.discriminant.type); // 输出"Identifier"，判断的是state1变量
console.log("判断变量名：", switchInWhile.discriminant.name); // 输出"state1"
console.log("case数量：", switchInWhile.cases.length); // 输出3，对应case0、case1、case2
const case0 = switchInWhile.cases[0];
console.log("第一个case判断值类型：", case0.test.type); // 输出"NumericLiteral"，数值字面量
console.log("第一个case判断值：", case0.test.value); // 输出0，对应case0
console.log("第一个case语句数量：", case0.consequent.length); // 输出4，包含let state2=0、while循环、break
console.log ("第一个 case 中 while 循环的内部结构：", case0.consequent [1].body.type); // 输出 "BlockStatement"，内层 while 循环体由 {} 包裹
const innerWhileSwitch = case0.consequent [1].body.body [0];
console.log ("内层 while 循环中的语句类型：", innerWhileSwitch.type); // 输出 "SwitchStatement"，对应内层 switch (state2)
console.log ("内层 switch 的判断变量：", innerWhileSwitch.discriminant.name); // 输出 "state2"，验证内层调度器的状态变量
// （新增）查看 return 语句节点（以 case2 中的 return sum 为例）
const innerCase2 = innerWhileSwitch.cases [2];
const returnStmt = innerCase2.consequent.find (item => item.type === "ReturnStatement");
console.log ("\n=== Return 语句节点结构（return sum） ===");
console.log ("节点类型：", returnStmt.type); // 输出 "ReturnStatement"
console.log ("返回值节点类型：", returnStmt.argument.type); // 输出 "Identifier"，返回的是 sum 变量
```

```
// 4. 递归处理多层控制流平坦化：遍历AST并修改节点（完整修正版）
function deobfuscateFlattening(path) {
  // 步骤A：识别调度器特征——while(true)循环
  if (path.isWhileStatement() && 
      path.node.test.type === "BooleanLiteral" && 
      path.node.test.value === true) {
    
    const whileBody = path.node.body;
    if (whileBody.type === "BlockStatement" && 
        whileBody.body.length === 1 && 
        whileBody.body[0].type === "SwitchStatement") {
      
      const switchStmt = whileBody.body[0];
      const stateVar = switchStmt.discriminant.name;
      console.log(`\n=== 识别到调度器：状态变量=${stateVar} ===`);
      // 步骤B：收集case映射关系
      const caseMap = new Map();
      let initialState = null;
      switchStmt.cases.forEach(caseItem => {
        const caseValue = caseItem.test.value;
        if (initialState === null) initialState = caseValue;
        // 提取case的有效代码块：过滤break语句
        const caseBody = caseItem.consequent.filter(item => !t.isBreakStatement(item));
        
        // 查找下一个state值
        let nextState = null;
        caseBody.forEach(stmt => {
          if (t.isExpressionStatement(stmt) && 
              t.isAssignmentExpression(stmt.expression) && 
              stmt.expression.left.name === stateVar) {
            nextState = stmt.expression.right.value;
          }
        });
        caseMap.set(caseValue, { body: caseBody, next: nextState });
        console.log(`case ${caseValue}：下一个state=${nextState || "无"}，代码块语句数=${caseBody.length}`);
      });
      // 步骤C：串联基本块
      const orderedBlocks = [];
      let currentState = initialState;
      while (caseMap.has(currentState)) {
        const { body, next } = caseMap.get(currentState);
        orderedBlocks.push(...body);
        if (next === null || !caseMap.has(next)) break;
        currentState = next;
      }
      console.log(`=== 串联完成：共${orderedBlocks.length}条语句 ===`);
      // 步骤D：递归处理嵌套调度器（修正版：复用当前path上下文）
      const tempBlock = t.blockStatement(orderedBlocks); // 构造临时代码块，包含所有待处理block
      path.parentPath.traverse({ // 复用当前path的父路径（上下文完整，不会undefined）
        WhileStatement: (nestedPath) => {
          // 仅处理临时代码块内的while节点，避免重复处理外层调度器
          if (nestedPath.findParent(p => p.node === tempBlock)) {
            deobfuscateFlattening(nestedPath); // 递归处理内层调度器（如state2对应的while）
          }
        }
      });
      // 步骤E：替换AST节点——用有序代码块替换原while+switch
      path.replaceWith(tempBlock); // 直接用tempBlock替换，避免重复构造
      console.log(`=== 替换调度器：${stateVar}对应的while+switch已移除 ===`);
      // 步骤F：删除多余state变量声明
      const parentScope = path.scope.parent;
      if (parentScope.bindings[stateVar]) {
        parentScope.bindings[stateVar].path.remove();
        console.log(`=== 清理残留：删除state变量声明=${stateVar} ===`);
      }
    }
  }
}
// 5. 启动AST遍历（无作用域问题：仅调用函数，不访问内部变量）
console.log("=== 开始遍历AST并反混淆 ===");
traverse(ast, {
  WhileStatement: deobfuscateFlattening  // 仅传递函数引用，不涉及 orderedBlocks
});
```

这段代码的混淆结构很明显：

第一层：state1 驱动的 while+switch 调度器；

第二层：在 state1=0 的 case 中，嵌套了 state2 驱动的 while+switch 调度器，核心计算逻辑（求和、判断奇偶）被藏在内层调度器中。

### 3.2 用 Babel 进行 AST 反混淆的具体实现

我们使用 @babel/parser（解析代码生成 AST）、@babel/traverse（遍历和修改 AST）、@babel/generator（将修改后的 AST 生成代码）这三个核心库，分步骤实现反混淆。

#### 步骤 1：环境准备与代码解析（新增 AST 生成详细介绍）

首先安装依赖：

```
// 5. 生成反混淆后的代码：AST → 可读代码
const { code: deobfuscatedCode } = generator(ast, {
  compact: false,  // 是否压缩代码：false表示格式化输出（便于阅读）
  comments: false, // 是否保留注释：false表示移除混淆中可能的无用注释
  indent: {        // 缩进配置：优化代码格式
    style: "  ",   // 缩进符号：2个空格
    base: 0,       // 基础缩进级别
    adjustMultilineComment: false
  }
});
// 输出结果：对比混淆代码与还原代码
console.log("\n=== 反混淆完成 ===");
console.log("混淆代码长度：", obfuscatedCode.length);
console.log("还原代码长度：", deobfuscatedCode.length);
console.log("\n还原后的代码：");
console.log("----------------------------------------");
console.log(deobfuscatedCode);
console.log("----------------------------------------");
```

然后编写基础代码，将混淆代码解析为 AST。这一步是 AST 反混淆的起点，我们需要深入理解代码如何被转化为结构化的语法树节点，以及解析配置的具体作用：

```
function compute(x, y) {
  let a = x * x;
  let b = y * y;
  let sum = 0;
  // 原内层调度器的逻辑（sum计算、奇偶判断）
  sum = a + b;
  if (sum % 2 === 0) {
    console.log("偶数");
    return sum;
  } else {
    console.log("奇数");
    return sum;
  }
  // 原外层调度器的后续逻辑（计算完成提示）
  console.log("计算完成");
  return sum;
}
```

```
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const generator = require("@babel/generator").default;
const t = require("@babel/types");
// 1. 混淆代码（输入）：待解析的多层平坦化代码
const obfuscatedCode = `function compute(x, y) {
  let a = x * x;
  let b = y * y;
  let sum = 0;
  let state1 = 0;
  while (true) {
    switch (state1) {
      case 0:
        let state2 = 0;
        while (true) {
          switch (state2) {
            case 0:
              sum = a + b;
              state2 = 1;
              break;
            case 1:
              if (sum % 2 === 0) {
                state2 = 2;
              } else {
                state2 = 3;
              }
              break;
            case 2:
              console.log("偶数");
              state1 = 1;
              return sum;
            case 3:
              console.log("奇数");
              state1 = 1;
              return sum;
          }
        }
        break;
      case 1:
        console.log("计算完成");
        state1 = 2;
        break;
      case 2:
        return sum;
    }
  }
}`;
// 2. 解析代码生成AST：核心配置与节点结构解析
const ast = parser.parse(obfuscatedCode, {
  sourceType: "script", // 关键配置1：指定代码类型为"script"（普通脚本）
  // 为什么不用"module"？因为示例代码是独立函数（无import/export），若设为"module"会报错
  plugins: ["jsx", "flow"] // 关键配置2：启用额外语法插件，避免解析非标准语法时失败
  // 此处"jsx"和"flow"插件是通用配置，即使代码中无JSX/Flow语法，也不会影响解析结果
});
// （新增）查看AST结构：通过打印关键节点，理解代码如何被转化为语法树
console.log("=== AST根节点结构 ===");
console.log("根节点类型：", ast.type); // 输出"File"，表示AST的根节点是文件级节点
console.log("文件内容节点类型：", ast.program.type); // 输出"Program"，存储代码的顶层语句
console.log("顶层语句数量：", ast.program.body.length); // 输出1，因为只有1个function声明
// （新增）查看函数声明节点的详细结构
const functionNode = ast.program.body[0];
console.log("\n=== 函数声明节点结构 ===");
console.log("节点类型：", functionNode.type); // 输出"FunctionDeclaration"，表示函数声明
console.log("函数名：", functionNode.id.name); // 输出"compute"，对应函数名
console.log("参数数量：", functionNode.params.length); // 输出2，对应参数x和y
console.log("参数1类型与名称：", functionNode.params[0].type, functionNode.params[0].name); // 输出"Identifier x"
console.log("参数2类型与名称：", functionNode.params[1].type, functionNode.params[1].name); // 输出"Identifier y"
console.log("函数体类型：", functionNode.body.type); // 输出"BlockStatement"，函数体由{}包裹
console.log("函数体语句数量：", functionNode.body.body.length); // 输出5，对应5句代码（let a=...、let b=...、let sum=0、let state1=0、while循环）
// （新增）查看变量声明节点（以let a = x * x为例）
const varDeclNode = functionNode.body.body[0];
console.log("\n=== 变量声明节点结构（let a = x * x） ===");
console.log("节点类型：", varDeclNode.type); // 输出"VariableDeclaration"，表示变量声明
console.log("声明关键字：", varDeclNode.kind); // 输出"let"，对应变量声明关键字
console.log("声明变量数量：", varDeclNode.declarations.length); // 输出1，当前仅声明a变量
const aDecl = varDeclNode.declarations[0];
console.log("变量名节点类型：", aDecl.id.type); // 输出"Identifier"，变量名是标识符
console.log("变量名：", aDecl.id.name); // 输出"a"
console.log("变量赋值表达式类型：", aDecl.init.type); // 输出"BinaryExpression"，二元运算表达式（x*x是乘法运算）
console.log("运算符号：", aDecl.init.operator); // 输出"*"，乘法运算符
console.log("左操作数类型与值：", aDecl.init.left.type, aDecl.init.left.name); // 输出"Identifier x"
console.log("右操作数类型与值：", aDecl.init.right.type, aDecl.init.right.name); // 输出"Identifier x"
// （新增）查看while循环节点（函数体中第5个语句）
const whileNode = functionNode.body.body[4];
console.log("\n=== While循环节点结构 ===");
console.log("节点类型：", whileNode.type); // 输出"WhileStatement"，表示while循环
console.log("循环条件类型：", whileNode.test.type); // 输出"BooleanLiteral"，表示布尔字面量
console.log("循环条件值：", whileNode.test.value); // 输出true，对应while(true)
console.log("循环体类型：", whileNode.body.type); // 输出"BlockStatement"，表示代码块（{}包裹）
console.log("循环体语句数量：", whileNode.body.body.length); // 输出1，循环体仅包含switch语句
const switchInWhile = whileNode.body.body[0];
console.log("循环体内部语句类型：", switchInWhile.type); // 输出"SwitchStatement"，对应循环内的switch
// （新增）查看switch语句节点（while循环内的switch(state1)）
console.log("\n=== Switch语句节点结构（switch(state1)） ===");
console.log("节点类型：", switchInWhile.type); // 输出"SwitchStatement"
console.log("判断变量节点类型：", switchInWhile.discriminant.type); // 输出"Identifier"，判断的是state1变量
console.log("判断变量名：", switchInWhile.discriminant.name); // 输出"state1"
console.log("case数量：", switchInWhile.cases.length); // 输出3，对应case0、case1、case2
const case0 = switchInWhile.cases[0];
console.log("第一个case判断值类型：", case0.test.type); // 输出"NumericLiteral"，数值字面量
console.log("第一个case判断值：", case0.test.value); // 输出0，对应case0
console.log("第一个case语句数量：", case0.consequent.length); // 输出4，包含let state2=0、while循环、break
console.log ("第一个 case 中 while 循环的内部结构：", case0.consequent [1].body.type); // 输出 "BlockStatement"，内层 while 循环体由 {} 包裹
const innerWhileSwitch = case0.consequent [1].body.body [0];
console.log ("内层 while 循环中的语句类型：", innerWhileSwitch.type); // 输出 "SwitchStatement"，对应内层 switch (state2)
console.log ("内层 switch 的判断变量：", innerWhileSwitch.discriminant.name); // 输出 "state2"，验证内层调度器的状态变量
// （新增）查看 return 语句节点（以 case2 中的 return sum 为例）
const innerCase2 = innerWhileSwitch.cases [2];
const returnStmt = innerCase2.consequent.find (item => item.type === "ReturnStatement");
console.log ("\n=== Return 语句节点结构（return sum） ===");
console.log ("节点类型：", returnStmt.type); // 输出 "ReturnStatement"
console.log ("返回值节点类型：", returnStmt.argument.type); // 输出 "Identifier"，返回的是 sum 变量
```

步骤 2：识别并处理多层调度器（核心逻辑：AST 遍历与修改）

通过解析生成 AST 后，下一步需要遍历 AST 节点，识别多层调度器结构并进行修改。这一步的核心是利用 @babel/traverse 的节点遍历能力，结合 @babel/types 的节点判断工具，精准定位混淆代码并还原逻辑：

```
// 4. 递归处理多层控制流平坦化：遍历AST并修改节点（完整修正版）
function deobfuscateFlattening(path) {
  // 步骤A：识别调度器特征——while(true)循环
  if (path.isWhileStatement() && 
      path.node.test.type === "BooleanLiteral" && 
      path.node.test.value === true) {
    const whileBody = path.node.body;
    if (whileBody.type === "BlockStatement" && 
        whileBody.body.length === 1 && 
        whileBody.body[0].type === "SwitchStatement") {
      const switchStmt = whileBody.body[0];
      const stateVar = switchStmt.discriminant.name;
      console.log(`\n=== 识别到调度器：状态变量=${stateVar} ===`);
      // 步骤B：收集case映射关系
      const caseMap = new Map();
      let initialState = null;
      switchStmt.cases.forEach(caseItem => {
        const caseValue = caseItem.test.value;
        if (initialState === null) initialState = caseValue;
        // 提取case的有效代码块：过滤break语句
        const caseBody = caseItem.consequent.filter(item => !t.isBreakStatement(item));
        // 查找下一个state值
        let nextState = null;
        caseBody.forEach(stmt => {
          if (t.isExpressionStatement(stmt) && 
              t.isAssignmentExpression(stmt.expression) && 
              stmt.expression.left.name === stateVar) {
            nextState = stmt.expression.right.value;
          }
        });
        caseMap.set(caseValue, { body: caseBody, next: nextState });
        console.log(`case ${caseValue}：下一个state=${nextState || "无"}，代码块语句数=${caseBody.length}`);
      });
      // 步骤C：串联基本块
      const orderedBlocks = [];
      let currentState = initialState;
      while (caseMap.has(currentState)) {
        const { body, next } = caseMap.get(currentState);
        orderedBlocks.push(...body);
        if (next === null || !caseMap.has(next)) break;
        currentState = next;
      }
      console.log(`=== 串联完成：共${orderedBlocks.length}条语句 ===`);
      // 步骤D：递归处理嵌套调度器（修正版：复用当前path上下文）
      const tempBlock = t.blockStatement(orderedBlocks); // 构造临时代码块，包含所有待处理block
      path.parentPath.traverse({ // 复用当前path的父路径（上下文完整，不会undefined）
        WhileStatement: (nestedPath) => {
          // 仅处理临时代码块内的while节点，避免重复处理外层调度器
          if (nestedPath.findParent(p => p.node === tempBlock)) {
            deobfuscateFlattening(nestedPath); // 递归处理内层调度器（如state2对应的while）
          }
        }
      });
      // 步骤E：替换AST节点——用有序代码块替换原while+switch
      path.replaceWith(tempBlock); // 直接用tempBlock替换，避免重复构造
      console.log(`=== 替换调度器：${stateVar}对应的while+switch已移除 ===`);
      // 步骤F：删除多余state变量声明
      const parentScope = path.scope.parent;
      if (parentScope.bindings[stateVar]) {
        parentScope.bindings[stateVar].path.remove();
        console.log(`=== 清理残留：删除state变量声明=${stateVar} ===`);
      }
    }
  }
}
// 5. 启动AST遍历（无作用域问题：仅调用函数，不访问内部变量）
console.log("=== 开始遍历AST并反混淆 ===");
traverse(ast, {
  WhileStatement: deobfuscateFlattening  // 仅传递函数引用，不涉及 orderedBlocks
});
```

AST 遍历与修改的关键细节：

1. 节点判断工具（@babel/types）：代码中 path.isWhileStatement()、t.isBreakStatement() 等方法，是判断节点类型的核心工具。例如 t.isAssignmentExpression(stmt.expression) 用于确认节点是否为赋值表达式，确保我们只处理状态变量的赋值语句，避免误改其他代码。

2. 作用域与变量清理：通过 path.scope.parent 获取状态变量的作用域（如函数级作用域），再通过 parentScope.bindings[stateVar].path.remove() 删除变量声明。这一步能彻底清理混淆残留的 state1、state2 变量，避免生成冗余代码。

3. 递归处理多层结构：在串联代码块后，通过 traverse(t.blockStatement([block]), { WhileStatement: deobfuscateFlattening }) 递归遍历子节点，确保内层嵌套的调度器（如 state2 对应的 while+switch）也能被识别和处理，解决多层平坦化的 “套娃” 问题。

步骤 3：生成还原后的代码（AST 转代码）

完成 AST 修改后，最后一步是通过 @babel/generator 将修改后的 AST 重新转化为可读的 JavaScript 代码。这一步会保留代码的语法正确性，并可通过配置优化代码格式：

```
// 5. 生成反混淆后的代码：AST → 可读代码
const { code: deobfuscatedCode } = generator(ast, {
  compact: false,  // 是否压缩代码：false表示格式化输出（便于阅读）
  comments: false, // 是否保留注释：false表示移除混淆中可能的无用注释
  indent: {        // 缩进配置：优化代码格式
    style: "  ",   // 缩进符号：2个空格
    base: 0,       // 基础缩进级别
    adjustMultilineComment: false
  }
});
// 输出结果：对比混淆代码与还原代码
console.log("\n=== 反混淆完成 ===");
console.log("混淆代码长度：", obfuscatedCode.length);
console.log("还原代码长度：", deobfuscatedCode.length);
console.log("\n还原后的代码：");
console.log("----------------------------------------");
console.log(deobfuscatedCode);
console.log("----------------------------------------");
```

最终还原代码结果：

运行上述代码后，生成的还原代码会彻底移除两层调度器，还原原始业务逻辑，代码如下：

```
function compute(x, y) {
  let a = x * x;
  let b = y * y;
  let sum = 0;
  // 原内层调度器的逻辑（sum计算、奇偶判断）
  sum = a + b;
  if (sum % 2 === 0) {
    console.log("偶数");
    return sum;
  } else {
    console.log("奇数");
    return sum;
  }
  // 原外层调度器的后续逻辑（计算完成提示）
  console.log("计算完成");
  return sum;
}
```

对比原混淆代码，还原后的代码：

1. 移除了 state1、state2 变量及对应的 while(true)+switch 调度器；

2. 按原始执行顺序串联了核心逻辑（平方计算→求和→奇偶判断→返回结果）；

3. 保留了所有业务逻辑代码，仅删除混淆相关的冗余结构。

四、AST 反混淆的关键原理与扩展

4.1 AST 反混淆的核心逻辑总结

通过上述实例，我们可以提炼出 AST 反混淆多层控制流平坦化的核心逻辑：

  1. 特征识别：基于代码结构特征（while(true)+switch+state 变量）定位  混淆节点；

  2. 结构分析：解析 AST 节点关系，收集 case 映射与 state 流转路径；

  3. 节点修改：用有序代码块替换混淆结构，递归处理多层嵌套；

  4. 冗余清理：删除混淆相关的变量与语句，生成可读代码。

4.2 实际场景的扩展：应对复杂混淆

上述实例是简化后的多层平坦化代码，实际混淆代码可能包含更复杂的干扰项，例如：

1. state 变量加密：state 值不是固定数值（如 state = (state + 1) % 3）；

2. 虚假 case：添加无实际逻辑的 case 节点（如 case 999）；

3. 动态跳转：通过函数调用获取下一个 state（如 state = getNextState()）。

应对这些复杂场景，需要在 AST 处理中增加额外逻辑：

1. 对于加密 state：通过 AST 计算表达式值（如 (state + 1) % 3）；

2. 对于虚假 case：通过代码覆盖率分析，过滤无执行路径的 case；

3. 对于动态跳转：通过函数内联或符号执行，还原 state 的真实值。

五、总结

控制流平坦化通过打乱代码执行流程增加逆向难度，而 AST 则为我们提供了 “透视” 混淆代码的工具 —— 通过将代码转化为结构化的语法树，我们可以精准定位混淆逻辑并还原原始流程。本文通过实例详细讲解了从 AST 生成、节点遍历、结构修改到代码生成的完整流程，希望能帮助大家理解 AST 反混淆的核心原理。

如果大家在实际操作中遇到复杂混淆场景（如带加密 state 的多层平坦化），或者想了解其他混淆技术（如字符串加密、控制流伪造）的 AST 反混淆方法，欢迎在评论区留言讨论！