> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_42857999/article/details/121586389#42_js_314)

### 极验滑块验证码破解与研究（一）：AST 还原混淆 JS

*   *   [声明](#_3)
    *   [一、环境安装](#_17)
    *   *   [1. node 安装](#1_node_19)
        *   *   [1.1. node 下载](#11_node_20)
            *   [1.2. 配置环境变量](#12__24)
            *   [1.3. node 安装检测](#13_node_25)
            *   [1.4. pycharm 配置 node 环境](#14_pycharmnode_29)
        *   [2. babel 库安装](#2_babel_32)
        *   *   [2.1. babel 库安装命令](#21_babel_33)
            *   [2.2. babel 库安装检测](#22_babel_37)
    *   [二、AST 还原混淆 JS](#ASTJS_45)
    *   *   [1. 模块导入](#1__53)
        *   [2. 复制还原需要用到的 js 源码](#2_js_63)
        *   [3. AST 还原函数详解](#3_AST_74)
        *   *   [3.1. replace_unicode](#31_replace_unicode_75)
            *   [3.2. replace_unicode, replace_name_array, replace_Dvn](#32_replace_unicode_replace_name_array_replace_Dvn_90)
            *   [3.3. replace_ForStatement](#33_replace_ForStatement_159)
            *   [3.4. delete_func](#34_delete_func_252)
        *   [4. AST 还原流程](#4_AST_272)
        *   *   [4.1. AST 还原整体流程](#41_AST_273)
            *   [4.2. 极验不同 js 或不同版本还原方式](#42_js_314)
    *   [三、结语](#_337)

声明
--

**原创文章，请勿转载！**

**本文内容仅限于安全研究，不公开具体源码。维护网络安全，人人有责。**

**本文关联文章超链接：**

1.  [极验滑块验证码破解与研究（一）：AST 还原混淆 JS](https://blog.csdn.net/qq_42857999/article/details/121586389)
2.  [极验滑块验证码破解与研究（二）：缺口图片还原](https://blog.csdn.net/qq_42857999/article/details/121628908)
3.  [极验滑块验证码破解与研究（三）：滑块缺口识别](https://blog.csdn.net/qq_42857999/article/details/121635961)
4.  [极验滑块验证码破解与研究（四）：滑块轨迹构造](https://blog.csdn.net/qq_42857999/article/details/121659205)
5.  [极验滑块验证码破解与研究（五）：请求分析及加密参数破解](https://blog.csdn.net/qq_42857999/article/details/121674134)

一、环境安装
------

### 1. [node](https://so.csdn.net/so/search?q=node&spm=1001.2101.3001.7020) 安装

#### 1.1. node 下载

点击此处 [node 官方下载链接](https://nodejs.org/zh-cn/download/) 下载安装包，并安装  
![](https://img-blog.csdnimg.cn/152a82095cee41289595052dee61663a.jpg?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)

#### 1.2. 配置[环境变量](https://so.csdn.net/so/search?q=%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F&spm=1001.2101.3001.7020)

#### 1.3. node 安装检测

```
node --version  # v14.16.0
```

#### 1.4. pycharm 配置 node 环境

![](https://img-blog.csdnimg.cn/e517a2f12a304090b793683c105ed725.jpg?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)

### 2. babel 库安装

#### 2.1. babel 库安装命令

```
npm install @babel/core --save-dev
```

#### 2.2. babel 库安装检测

```
node  # 进入node环境
require("@babel/parser")  # 引入babel库
```

![](https://img-blog.csdnimg.cn/88b8fa21c7cd439cbeeb050cd78c6df5.jpg?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)

二、AST 还原混淆 JS
-------------

**温馨提示：极验 js 都采用相同的混淆方式。后期文章用到的 fullpage.9.0.7.js ，click.3.0.2.js 等都用此方式还原**  
不懂 AST 语法树的小伙伴可以问问度娘哦  
本文以 slide.7.8.4.js 文件为例，[在线 AST 语法树转换工具](https://astexplorer.net/)

```
aHR0cHM6Ly9zdGF0aWMuZ2VldGVzdC5jb20vc3RhdGljL2pzL3NsaWRlLjcuOC40Lmpz
```

### 1. 模块导入

导入还原混淆 JS 需要用到的功能块

```
const parser = require("@babel/parser");
const traverse = require("@babel/traverse").default;
const t = require("@babel/types");
const generator = require("@babel/generator").default;
const fs = require("fs");
```

### 2. 复制还原需要用到的 js [源码](https://so.csdn.net/so/search?q=%E6%BA%90%E7%A0%81&spm=1001.2101.3001.7020)

博主太懒了，小伙伴自己复制吧。（源代码太长了，减少篇幅…）  
此部分代码为 slide.7.8.4.js 文件中的函数，一模一样的复制下来就行，还原时会用到。

```
AaWgt.Bq_ = function () {}();
AaWgt.CyZ = function () {}();
AaWgt.Dvn = function () {};
AaWgt.EeS = function () {};
function AaWgt() {}
```

### 3. AST 还原函数详解

#### 3.1. replace_unicode

js 代码中有很多字符为的二进制、[Unicode](https://so.csdn.net/so/search?q=Unicode&spm=1001.2101.3001.7020) 编码，此函数将代码中的其他字符编码转换为 utf-8 编码

```
// 删除节点中的extra属性（二进制、Unicode等编码 -> utf-8）
function replace_unicode(path) {
    let node = path.node;
    if (node.extra === undefined)
        return;
    delete node.extra;
}
```

还原前  
![](https://img-blog.csdnimg.cn/092817471676480c907c6ee9ccd9ed15.jpg?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)  
还原后  
![](https://img-blog.csdnimg.cn/5546186797bb4ccaa8941e73776344a9.jpg?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)

#### 3.2. replace_unicode, replace_name_array, replace_Dvn

代码中大量使用类似 var aVMl = AXzPo.DVn 的表达式，AXzPo.DVn 实际为一个大数组，下面函数将其替换为具体字符

```
// 定义一个全局变量，存放待替换变量名
let name_array = [];

function get_name_array(path) {
    let {kind, declarations} = path.node
    if (kind !== 'var'
        || declarations.length !== 3
        || declarations[0].init === null
        || declarations[0].init.property === undefined)
        return;
    if (declarations[0].init.property.name !== "Dvn")
        return;
    // 获取待替换节点变量名
    let name1 = declarations[0].id.name
    // 获取待输出变量名
    let name2 = declarations[2].id.name
    // 将变量名存入数组
    name_array.push(name1, name2)

    // 删除下一个节点
    path.getNextSibling().remove()
    // 删除下一个节点
    path.getNextSibling().remove()
    // 删除path节点
    path.remove()
}

function replace_name_array(path) {
    let {callee, arguments} = path.node
    if (callee === undefined || callee.name === undefined)
        return;
    // 不在name_array中的节点不做替换操作
    if (name_array.indexOf(callee.name) === -1)
        return;
    // 调用AaWgt.Dvn函数获取结果
    let value = AaWgt.Dvn(arguments[0].value);
    // 创建节点并替换结果
    let string_node = t.stringLiteral(value)
    path.replaceWith(string_node)
}

function replace_Dvn(path) {
    let {arguments, callee} = path.node
    // 解析arguments参数
    if (arguments.length !== 1) return;
    if (arguments[0].type !== 'NumericLiteral') return;

    // 解析callee
    if (callee.type !== 'MemberExpression') return;
    let {object, property} = callee;
    if (object.type !== 'Identifier' || property.type !== 'Identifier') return;

    if (property.name === 'Dvn') {
        // 计算值
        let value = AaWgt.Dvn(arguments[0].value);
        // 创建节点并替换
        let string_node = t.stringLiteral(value)
        path.replaceWith(string_node)
    }
}
```

还原前  
![](https://img-blog.csdnimg.cn/175c5a742dd1422e8c162ab90ab8c0db.jpg?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)  
还原后  
![](https://img-blog.csdnimg.cn/f1289bdc9100402e8d4e30aee2e72a41.jpg?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)

#### 3.3. replace_ForStatement

js 代码中大量 switch-case 结构，不利于代码逻辑解析，需要控制流平坦化

```
// 控制流平坦化
function replace_ForStatement(path) {
    var node = path.node;

    // 获取上一个节点，也就是VariableDeclaration
    var PrevSibling = path.getPrevSibling();

    // 判断上个节点的各个属性，防止报错
    if (PrevSibling.container === undefined
        || PrevSibling.container[0].declarations === undefined
        || PrevSibling.container[0].declarations[0].init === null
        || PrevSibling.container[0].declarations[0].init.object === undefined
        || PrevSibling.container[0].declarations[0].init.object.object === undefined)
        return;
    if (PrevSibling.container[0].declarations[0].init.object.object.callee.property.name !== 'EeS')
        return;

    // SwitchStatement节点
    var body = node.body.body;
    // 判断当前节点的body[0]属性和body[0].discriminant是否存在
    if (!t.isSwitchStatement(body[0]))
        return;
    if (!t.isIdentifier(body[0].discriminant))
        return;

    // 获取控制流的初始值
    var argNode = PrevSibling.container[0].declarations[0].init;
    var init_arg_f = argNode.object.property.value;
    var init_arg_s = argNode.property.value;
    var init_arg = AaWgt.EeS()[init_arg_f][init_arg_s];

    // 提取for节点中的if判断参数的value作为判断参数
    var break_arg_f = node.test.right.object.property.value;
    var break_arg_s = node.test.right.property.value;
    var break_arg = AaWgt.EeS()[break_arg_f][break_arg_s];

    // 提取switch下所有的case
    var case_list = body[0].cases;
    var resultBody = [];

    // 遍历全部的case
    for (var i = 0; i < case_list.length; i++) {
        for (; init_arg != break_arg;) {

            // 提取并计算case后的条件判断的值
            var case_arg_f = case_list[i].test.object.property.value;
            var case_arg_s = case_list[i].test.property.value;
            var case_init = AaWgt.EeS()[case_arg_f][case_arg_s];

            if (init_arg == case_init) {
                //当前case下的所有节点
                var targetBody = case_list[i].consequent;

                // 删除break节点，和break节点的上一个节点的一些无用代码
                if (t.isBreakStatement(targetBody[targetBody.length - 1])
                    && t.isExpressionStatement(targetBody[targetBody.length - 2])
                    && targetBody[targetBody.length - 2].expression.right.object.object.callee.object.name == "AaWgt") {

                    // 提取break节点的上一个节点AJgjJ.EMf()后面的两个索引值
                    var change_arg_f = targetBody[targetBody.length - 2].expression.right.object.property.value;
                    var change_arg_s = targetBody[targetBody.length - 2].expression.right.property.value;

                    // 修改控制流的初始值
                    init_arg = AaWgt.EeS()[change_arg_f][change_arg_s];

                    targetBody.pop(); // 删除break
                    targetBody.pop(); // 删除break节点的上一个节点
                }
                //删除break
                else if (t.isBreakStatement(targetBody[targetBody.length - 1])) {
                    targetBody.pop();
                }
                resultBody = resultBody.concat(targetBody);
                break;
            } else {
                break;
            }
        }
    }
    //替换for节点，多个节点替换一个节点用replaceWithMultiple
    path.replaceWithMultiple(resultBody);

    //删除上一个节点
    PrevSibling.remove();
}
```

还原前  
![](https://img-blog.csdnimg.cn/e5ae164342f54506be230cf336980be4.jpg?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)  
还原后  
![](https://img-blog.csdnimg.cn/719d614908e049b6adba1478924b8fa7.jpg?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_19,color_FFFFFF,t_70,g_se,x_16)

#### 3.4. delete_func

还原过程结束啦，是时候删除部分无关函数了。

```
// 删除无关函数
function delete_func(path) {
    let {expression} = path.node
    if (expression === undefined
        || expression.left === undefined
        || expression.left.property === undefined)
        return;
    if (expression.left.property.name === 'Bq_'
        || expression.left.property.name === 'Dvn'
        || expression.left.property.name === 'CyZ'
        || expression.left.property.name === 'EeS'
    ) {
        path.remove()
    }
}
```

### 4. AST 还原流程

#### 4.1. AST 还原整体流程

```
// 需要解码的文件位置
let encode_file = "slide.7.8.4.js"
// 解码后的文件位置
let decode_file = "ast_slide.7.8.4.js_init.js"

// 读取需要解码的js文件, 注意文件编码为utf-8格式
let jscode = fs.readFileSync(encode_file, {encoding: "utf-8"});

// 将js代码修转成AST语法树
let ast = parser.parse(jscode);
// AST结构修改逻辑
const visitor = {
    StringLiteral: {
        enter: [replace_unicode]
    },
    VariableDeclaration: {
        enter: [get_name_array]
    },
    CallExpression: {
        enter: [replace_name_array, replace_Dvn]
    },
    ForStatement: {
        enter: [replace_ForStatement]
    },
    ExpressionStatement: {
        enter: [delete_func]
    },
}

// 遍历语法树节点，调用修改函数
traverse(ast, visitor);

// 将ast转成js代码，{jsescOption: {"minimal": true}} unicode -> 中文
let {code} = generator(ast, opts = {jsescOption: {"minimal": true}});
// 将js代码保存到文件
fs.writeFile(decode_file, code, (err) => {
});
```

#### 4.2. 极验不同 js 或不同版本还原方式

本系列文章使用的 js 版本：

```
fullpage.9.0.7.js：https://static.geetest.com/static/js/fullpage.9.0.7.js
slide.7.8.4.js：https://static.geetest.com/static/js/slide.7.8.6.js
click.3.0.2.js（后期出点选系列文章会用到）：https://static.geetest.com/static/js/click.3.0.2.js
```

**小伙伴们可以发现，极验 fullpage.9.0.x.js、slide.7.8.x.js、click.3.0.x.js 均使用相同的混淆方式。所以，本文章的 AST 还原方式可以用于当前最新版本 js 的还原。下面介绍一个案例，将适用 slide.7.8.4.js 还原的 AST 代码替换成适用 fullpage.9.0.7.js 的**

第一步：将 fullpage.9.0.7.js 中部分源码替换到 AST 文件中。  
![](https://img-blog.csdnimg.cn/e7e669f4ea5c4069a355cada7734d010.jpg?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)

第二步：使用编译器的替换功能，将 AST 代码中的函数名称全部替换。  
![](https://img-blog.csdnimg.cn/3de10ab550444290a6a2518c7043ba3e.jpg?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)

第三步：最后，记得将需要解码的文件修改成 fullpage.9.0.7.js

```
// 需要解码的文件位置
let encode_file = "fullpage.9.0.7.js"
// 解码后的文件位置
let decode_file = "ast_fullpage.9.0.7.js_init.js"
```

三、结语
----

**友情链接：**[**极验滑块验证码破解与研究（二）：缺口图片还原**](https://blog.csdn.net/qq_42857999/article/details/121628908)

_本期文章结束啦，如果对您有帮助，记得收藏加关注哦，后期文章会持续更新 ~~~_

![](https://img-blog.csdnimg.cn/16c48d181d9a469283c514eb20b5adce.jpg?x-oss-process=image/watermark,type_ZHJvaWRzYW5zZmFsbGJhY2s,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)