> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/InIxM4fWvLxCf-cCx2bQBQ)

  

动态字面量混淆遇到好多次了，今天记录一下此类问题的公式化解法。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxd87vhibtZPkmcm9QZ3w0RKPRkhdibHYDicCaBTHtDGZbnatcwAC8ichBjGzicj3WaFb4xlZgjiabyGj6g/640?from=appmsg&watermark=1#imgIndex=0)

  

  

  

  

  

  

  

  

**申明**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/aq2oXv2j2QShDQrZy66OdT4MoOfql7j8WcjebvKlf4ZmkicTZia12HMlvnnNibmTXrpiaickhfVzohUBnXPaIicoHUYw/640?from=appmsg#imgIndex=1)

本文章中所有内容仅供学习交流使用，不用于其他任何目的，严禁用于商业用途和非法用途，否则由此产生的一切后果均与作者无关！若有侵权，请添加（wx：ShawYbo）联系删除

  

**目标网站**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/aq2oXv2j2QShDQrZy66OdT4MoOfql7j8WcjebvKlf4ZmkicTZia12HMlvnnNibmTXrpiaickhfVzohUBnXPaIicoHUYw/640?from=appmsg#imgIndex=2)

**网站**：https://vidfast.pro/ （国外网站，就不脱敏了）

**目标**：算法分析

  

**分析过程**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/aq2oXv2j2QShDQrZy66OdT4MoOfql7j8WcjebvKlf4ZmkicTZia12HMlvnnNibmTXrpiaickhfVzohUBnXPaIicoHUYw/640?from=appmsg#imgIndex=3)

我用这个国外视频播放网站来示例。先看一下这个播放网站请求链：

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxd87vhibtZPkmcm9QZ3w0RKdHUiciaL95LRA3u4oiaJB4siajG7q9BqtT6RbuxQv5EicXIVfsbEDq52q3w/640?from=appmsg&watermark=1#imgIndex=4)

  

  

  

  

  

  

  

第一个请求的响应会返回第二个请求的参数，第二请求响应是视频播放地址（m3u8），这两个关键请求的参数都是由一个 webpack 打包混淆后的 js 生成，从启动器跟栈，找到具体参数的生成位置：

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxd87vhibtZPkmcm9QZ3w0RKBRkf84vzzmEibQia9rxdyWr6icCAbTQiceXXOxGN8jTH1q7Tt4JP7ToXuQ/640?from=appmsg&watermark=1#imgIndex=5)

  

  

  

  

  

  

  

算法逻辑很清晰也很简单，就是 aes-cbc-256, 上两行定义的 m 和 k 就是 key 和 iv，我们把鼠标放上去，就直接显示出 key 和 iv，理论上拷贝出来直接用即可，但是。。。这个 js 每个会更新一次，变量名称和密钥统统会变化。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxd87vhibtZPkmcm9QZ3w0RKENnecEo3v2OFPD4fsUHOibnIBwY4TYvTH9OZyPcfJiaM8JoAVjOy8qsQ/640?from=appmsg&watermark=1#imgIndex=6)

  

  

  

  

  

  

  

笨办法就是，每天醒来到公司的第一件事就是打开该网站，把断点打好，刷新页面，把鼠标放上去显示出 key、iv，然后更新代码。。。

当然，有 ast 和 ai 的加持下，我们绝对不会向笨办法妥协，不就是字面量混淆吗，直接公式化打法，上来先给他一套组合拳。

  

**第一步：找解密函数**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/aq2oXv2j2QShDQrZy66OdT4MoOfql7j8WcjebvKlf4ZmkicTZia12HMlvnnNibmTXrpiaickhfVzohUBnXPaIicoHUYw/640?from=appmsg#imgIndex=7)

随便找一个混淆的字面量，比如示例中 k 的定义：GD[r(21136, 14516, "sSF8", 13737, 10778)]，跟栈 r 的调用，就能找到真正的解密函数：

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxd87vhibtZPkmcm9QZ3w0RKiaEVYeibqawXYegZc5V6qowmZwzSKeI7NJtQdL9slPiaOqlpoDLeZAzkQ/640?from=appmsg&watermark=1#imgIndex=8)

  

  

  

  

  

  

  

此类解密函数的结构都大差不差，一般的结构就是一个超长数组配合自定义编码集解密，其中 let r = kx()，kx 即为超长数组，先看一下这个数组：

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxd87vhibtZPkmcm9QZ3w0RKMeia1ac8RSj5aA3B8gAcO5bjGBibMCHYNiaibbDAiaBtgMhZGohNTiabCZ8w/640?from=appmsg&watermark=1#imgIndex=9)

  

  

  

  

  

  

  

而且，大多样本中，这个数组并不是拿来直接用的，它存在一个数组打乱逻辑，也就是说先得执行一段代码，对**数组重新排序**（直接把断点打到数组里就能找到这个逻辑代码）：

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxd87vhibtZPkmcm9QZ3w0RKn2MhNYyUfAA6LxyJgbC4xGLRPWw2AN5ldJnejB6Gy7gdho2OvLtDpw/640?from=appmsg&watermark=1#imgIndex=10)

  

  

  

  

  

  

  

可以看到上图中是一段自执行代码对数组排序，有的样本是在解密函数中对数组重新排序（先判断数组是否初始化过，没有初始化就会对数组重排序）。有了**数组、解密函数、重排序函数**这三部分，就算是完成了最核心的内容了。将这三部分使用正则或者 AST 识别都可以，将其提取出来。

```
function extractDecryptFunctions(ast) {
    console.log("步骤1: 提取解密函数...");
    
    let shuffleCode = null;
    let arrayFunctionName = null;
    let baseDecryptFuncs = [];
    let extractedCode = [];
    
    traverse(ast, {
        ExpressionStatement(path) {
            if (t.isUnaryExpression(path.node.expression, { operator: "!" }) &&
                t.isCallExpression(path.node.expression.argument)) {
                
                const callExpr = path.node.expression.argument;
                if (t.isFunctionExpression(callExpr.callee)) {
                    const code = generate(path.node).code;
                    
                    if (code.includes(".push(") && code.includes(".shift()")) {
                        shuffleCode = code;
                        
                        if (callExpr.arguments.length > 0 && t.isIdentifier(callExpr.arguments[0])) {
                            arrayFunctionName = callExpr.arguments[0].name;
                        }
                    }
                }
            }
        }
    });
    
    traverse(ast, {
        FunctionDeclaration(path) {
            const funcName = path.node.id.name;
            const funcCode = generate(path.node).code;
            
            if (funcName === arrayFunctionName) {
                extractedCode.push(funcCode);
                console.log(`  ✓ 数组函数: ${funcName}`);
            } else if (funcCode.includes(BASE64_CHARS)) {
                extractedCode.push(funcCode);
                baseDecryptFuncs.push(funcName);
                console.log(`  ✓ 解密函数: ${funcName}`);
            }
        }
    });
    
    if (shuffleCode) {
        extractedCode.push(shuffleCode);
    }
    
    const fileContent = extractedCode.join("\n\n");
    fs.writeFileSync("字面量解密函数.js", fileContent, "utf-8");
    
    return { baseDecryptFuncs, decryptCode: fileContent };
}
```

我是通过使用 AST 识别其中的特征来提取的，比如重排序函数中包含了 **push** 和 **shift** 等关键字。

**PS: 特征识别指的是不管这个 JS 每天怎么变化，每次变化时必须包含的一些结构或者关键词，这样才能使用 AST 来识别。**

  

**解析函数调用链**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/aq2oXv2j2QShDQrZy66OdT4MoOfql7j8WcjebvKlf4ZmkicTZia12HMlvnnNibmTXrpiaickhfVzohUBnXPaIicoHUYw/640?from=appmsg#imgIndex=11)

我们可以看到所有的字面量混淆并不是直接调用的解密函数，而是创建了多层包装函数，所以需要追踪调用链。

比如你在代码中看到 o(11211, -73, -6169, 4345, -210)，实际上这个 o 可能只是一个中间包装，它内部会调用 qb，而 qb 又调用 ko，最终才到达真正的解密函数。更麻烦的是，每一层调用都可能对参数进行变换，比如 o(t, e, r, n, o) 内部返回的是  qb(t-330, n- -318, r-422,n-115,r)。我们的目标就是穿透这些层层包装，找到最终的解密函数和它需要的真实参数。

```
function ko(t, e) {
//    最终解密函数
}
function qb(t, e, r, n, o) {
    return ko(e - 1074 - -181, o)
}
function o(t, e, r, n, o) {
    return qb(t - 330, n - -318, r - 422, n - 115, r)
}
let a = GD[o(11211, -73, -6169, 4345, -210)]
```

**递****归解****析****逻辑**：拿到函数名和参数后，先判断是否已是基础解密函数，不是则查找定义并继续追踪。用 visited 集合防止循环引用：

```
function resolveFunctionCall(funcName, args, scope, baseDecryptFuncs, visited = new Set()) {
    if (visited.has(funcName)) return null;  // 防循环
    visited.add(funcName);
    
    if (baseDecryptFuncs.includes(funcName)) {
        return { finalFunc: funcName, finalArgs: args };  // 找到终点
    }
```

**作****用域查****找**：不能全局搜索函数名，要用 Babel 的 scope.getBinding() 找到正确的定义：

```
const binding = scope.getBinding(funcName);
    if (!binding || !binding.path.isFunctionDeclaration()) return null;
    
    const funcPath = binding.path;
    const params = funcPath.node.params.map(p => p.name);
```

**查****找 return 语句**：只关心当前函数直接的 return，排除嵌套函数：

```
let returnCall = null;
    funcPath.traverse({
        ReturnStatement(returnPath) {
            if (returnPath.getFunctionParent() !== funcPath) return;  // 跳过嵌套
            
            const arg = returnPath.node.argument;
            if (t.isCallExpression(arg) && t.isIdentifier(arg.callee)) {
                returnCall = { targetFunc: arg.callee.name, callNode: arg };
            }
        }
    });
```

**参****数变****换与****表达式求值**：这是核心难点。当函数返回 _0x1c8a(a - 0x10, b) 时，需要计算出具体值。我们创建形参到实参的映射表，然后用它来求值表达式中的每个节点：

```
function resolveFunctionCall(funcName, args, scope, baseDecryptFuncs, visited = new Set()) {
    // 1. 防止无限递归
    if (visited.has(funcName)) return null;
    visited.add(funcName);
    
    // 2. 如果已经是基础解密函数，直接返回
    if (baseDecryptFuncs.includes(funcName)) {
        return { finalFunc: funcName, finalArgs: args };
    }
    
    // 3. 查找函数定义
    const binding = scope.getBinding(funcName);
    if (!binding) return null;
    
    // 4. 分析函数体的return语句
    const returnCall = findReturnCall(binding.path);
    if (!returnCall) return null;
    
    // 5. 计算参数变换（关键！）
    const paramMap = new Map();
    for (let i = 0; i < params.length; i++) {
        paramMap.set(params[i], args[i]);
    }
    
    const transformedArgs = returnCall.arguments.map(arg => 
        evalExprWithParams(arg, scope, paramMap)
    );
    
    // 6. 递归解析下一层
    return resolveFunctionCall(returnCall.callee.name, transformedArgs, scope, baseDecryptFuncs, visited);
}
```

这一步的核心要点：**作用****域查找**确保找到正确定义，**参数****映射表**为表达式求值提供变量上下文，**递****归求****值**处理嵌套运算，三者配合穿透多层包装。

  

**批量解密和节点替换**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/aq2oXv2j2QShDQrZy66OdT4MoOfql7j8WcjebvKlf4ZmkicTZia12HMlvnnNibmTXrpiaickhfVzohUBnXPaIicoHUYw/640?from=appmsg#imgIndex=12)

完成调用链解析后，我们得到了一堆待解密任务，每个任务包含最终的解密函数名和参数。最直观的做法是循环遍历，对每个任务都 eval 一次解密函数，但这样效率极低。一个混淆文件可能有上千个加密字符串，每次 eval 都要重新解析代码、初始化环境，这会导致整个解混淆过程耗时几分钟甚至更久。更高效的方案是把所有解密任务打包成一段代码，只执行一次 eval，这能将性能提升 n 倍以上。

批量解密的核心思路是构造一段 JavaScript 代码，这段代码包含原始的解密函数定义和所有解密调用。我们已经在步骤一提取并保存了解密函数（包括**数组函数、重排序函数、解密函数**），现在要做的是为每个任务生成一行调用代码，把结果存入数组：

```
function batchDecrypt(decryptCode, decryptTasks) {
    if (decryptTasks.length === 0) return new Map();
    
    console.log(`  批量解密 ${decryptTasks.length} 个任务...`);
    
    const results = new Map();
    
    // 为每个任务生成调用代码
    const batchCode = decryptTasks.map((task, idx) => {
        const argsStr = task.args.map(arg => {
            if (typeof arg === 'string') {
                return `"${arg.replace(/\\/g, '\\\\').replace(/"/g, '\\"')}"`;
            }
            return String(arg);
        }).join(", ");
        
        return `  try { results[${idx}] = ${task.funcName}(${argsStr}); } catch(e) {}`;
    }).join("\n");
```

这里需要注意参数格式化：字符串参数要加引号并转义特殊字符（反斜杠和引号），数字参数直接转字符串。**每个调用都包在 try-catch 中**，因为某些参数组合可能不合法，不能让一个失败影响整批任务。假设有三个任务，生成的代码类似：

```
try { results[0] = _0x1c8a(10, 'test'); } catch(e) {}
try { results[1] = _0x1c8a(25, 'abc'); } catch(e) {}
try { results[2] = _0x1c8a(8, 'xyz'); } catch(e) {}
```

接下来将解密函数代码和批量调用代码拼接，构成完整的可执行代码：

```
const fullCode = `
${decryptCode}
const results = [];
${batchCode}
results;
`;
```

这段代码的结构是：先定义所有解密相关的函数（包括数组初始化、重排序），然后创建结果数组，执行所有解密调用，最后返回结果数组。一次性 eval 执行后，我们就能拿到所有解密结果：

```
try {
        const batchResults = eval(fullCode);
        
        for (let i = 0; i < decryptTasks.length; i++) {
            if (batchResults[i] !== undefined && batchResults[i] !== null) {
                results.set(decryptTasks[i].key, batchResults[i]);
            }
        }
    } catch (e) {
        console.error(`  批量解密失败: ${e.message}`);
    }
    
    return results;
}
```

返回的 results 是一个 Map，键是任务的标识（如 "o(11211, -73, -6169, 4345, -210)"），值是解密后的字符串。这个 Map 会被传递给替换步骤使用。

替换操作需要精确定位到原始的调用表达式节点。在收集任务时，我们用一个 callMap 记录了每个 AST 节点对应的任务索引，现在利用这个映射来找到需要替换的位置：

```
function collectAndDecryptCalls(ast, baseDecryptFuncs, decryptCode) {
    console.log("\n步骤2: 收集解密调用并解析...");
    
    const decryptTasks = [];
    const callMap = new Map();
    
    traverse(ast, {
        CallExpression(path) {
            // ... 解析调用链 ...
            
            if (resolved) {
                const key = `${resolved.finalFunc}(${resolved.finalArgs.join(',')})`;
                
                const taskIndex = decryptTasks.length;
                decryptTasks.push({
                    key: key,
                    funcName: resolved.finalFunc,
                    args: resolved.finalArgs
                });
                
                callMap.set(path, taskIndex);  // 记录节点与任务的对应关系
            }
        }
    });
    
    console.log(`  收集到 ${decryptTasks.length} 个调用`);
    
    // 批量解密
    console.log("\n步骤3: 批量解密...");
    const decryptResults = batchDecrypt(decryptCode, decryptTasks);
    console.log(`  成功解密 ${decryptResults.size} 个值`);
```

解密完成后立即进行替换。再次遍历 AST，如果遇到的节点在 callMap 中存在，说明它是我们要处理的调用，通过任务索引找到解密结果，将整个调用表达式替换为字符串字面量：

```
// 替换
    console.log("\n步骤4: 替换解密后的值...");
    let replaceCount = 0;
    
    traverse(ast, {
        CallExpression(path) {
            if (callMap.has(path)) {
                const taskIndex = callMap.get(path);
                const task = decryptTasks[taskIndex];
                const result = decryptResults.get(task.key);
                
                if (result !== undefined) {
                    path.replaceWith(t.stringLiteral(result));
                    replaceCount++;
                }
            }
        }
    });
    
    console.log(`  替换了 ${replaceCount} 个值`);
}
```

使用 path.replaceWith()，传入 t.stringLiteral(result) 创建的字符串字面量节点，就完成了从 o(11211,-73,-6169,4345,-210) 到 "https://api.example.com" 的转换。这里有个细节：只替换解密成功的调用（result !== undefined），如果某个调用解密失败，保留原样，避免破坏代码结构。

**流****程**：**遍历 AST 收集调用并建立映射** → **构造批量代码一次性解密** → **再次遍历根据映射替换节点**。关键点是用 callMap 保证替换的准确性，用批量 eval 优化性能（O(n) → O(1)）。

字面量解混淆的**核心思想**是：

*   **识别模式**：找出解密函数的特征
    
*   **静态分析**：通过 AST 解析调用关系
    
*   **动态执行**：利用原始解密逻辑批量解密
    
*   **结果替换**：将结果回填到 AST
    

整体流程如下：

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxd87vhibtZPkmcm9QZ3w0RK5AFQUTTBThOrTFfmV6cGiciaVcO5SCMnA6LWf1g9dOX0uf4bdGPc5mow/640?from=appmsg&watermark=1#imgIndex=13)

  

  

  

  

  

  

  

最后附一张解混淆后的图片：

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxd87vhibtZPkmcm9QZ3w0RKSb1NFZxzK3qsrycuYNRywLOx8A0pTGBPIJSe1hb4UH7es090t3WhBg/640?from=appmsg&watermark=1#imgIndex=14)

  

  

  

  

  

  

  

  

**总结**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/aq2oXv2j2QShDQrZy66OdT4MoOfql7j8WcjebvKlf4ZmkicTZia12HMlvnnNibmTXrpiaickhfVzohUBnXPaIicoHUYw/640?from=appmsg#imgIndex=15)

这套方法具有很强的**通用性**，只需针对不同混淆器调整特征识别部分，整体框架可以复用。代码放在后台了，回复：**字面量解混淆** 即可。求佬儿点个**在看**或者**点个赞**！

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_jpg/97ib7mrowjiaxd87vhibtZPkmcm9QZ3w0RKjmxsDoJ2YFFSxYvQTPLGNHvqJsu6vUnAMy6DqZPXUOLHoyUTVlmzxw/640?from=appmsg&watermark=1#imgIndex=16)

  

  

  

  

  

  

  

**END**

![](https://mmbiz.qpic.cn/mmbiz_gif/97ib7mrowjiaxd87vhibtZPkmcm9QZ3w0RKSKhNibO5wmiamKUpxHbuiacoc7omqrUvSwsW97f9FExzZq1Nf4ZaWBq3Q/640?from=appmsg#imgIndex=17)

![](https://mmbiz.qpic.cn/mmbiz_jpg/97ib7mrowjiaxd87vhibtZPkmcm9QZ3w0RKue2MEgdQCV2m9iaPAn5y1lgGHsOdWvtdXInOLlQuibibpgUDQDiccGkkcg/640?from=appmsg&watermark=1#imgIndex=18)

  

**窥破表象**

  

**方见本源**