> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [ainrm.cn](https://ainrm.cn/2022/js_ast.html)

> js 文件在日常测试中是一个容易被忽略的点，其代码繁杂冗长具有较差的可读性，但往往承载着重要业务逻辑，如前后端分离站点前端操作逻辑写在 js 中，开发者未对代码做安全处理直接发布便可能存在安全风险，而鉴于 js......

### [](#一、前言 "一、前言")一、前言

js 文件在日常测试中是一个容易被忽略的点，其代码繁杂冗长具有较差的可读性，但往往承载着重要业务逻辑，如前后端分离站点前端操作逻辑写在 js 中，开发者未对代码做安全处理直接发布便可能存在安全风险，而鉴于 js 代码的透明性，提高阅读难度似乎是最直接高效的办法

混淆前：

js

```
console.log("Hello World!");
```

混淆后：

js

```
// obfuscator default模式混淆
function _0x5036(_0x92e953, _0x27bdba) {
    var _0xd97dbd = _0xd97d();
    return _0x5036 = function (_0x5036a0, _0x39efd5) {
        _0x5036a0 = _0x5036a0 - 0x144;
        var _0x3beebf = _0xd97dbd[_0x5036a0];
        return _0x3beebf;
    }, _0x5036(_0x92e953, _0x27bdba);
}
var _0x1721d6 = _0x5036;
(function (_0x46d758, _0x1fbdfa) {
    var _0x284cfd = _0x5036,
        _0x14b3cc = _0x46d758();
    while (!![]) {
        try {
            var _0x2e2de6 = -parseInt(_0x284cfd(0x14b)) / 0x1 * (-parseInt(_0x284cfd(0x148)) / 0x2) + parseInt(
                    _0x284cfd(0x14c)) / 0x3 * (parseInt(_0x284cfd(0x144)) / 0x4) + parseInt(_0x284cfd(0x14d)) / 0x5 +
                -parseInt(_0x284cfd(0x14f)) / 0x6 + -parseInt(_0x284cfd(0x147)) / 0x7 * (parseInt(_0x284cfd(0x14e)) /
                    0x8) + -parseInt(_0x284cfd(0x14a)) / 0x9 + -parseInt(_0x284cfd(0x150)) / 0xa * (parseInt(
                    _0x284cfd(0x149)) / 0xb);
            if (_0x2e2de6 === _0x1fbdfa) break;
            else _0x14b3cc['push'](_0x14b3cc['shift']());
        } catch (_0x107939) {
            _0x14b3cc['push'](_0x14b3cc['shift']());
        }
    }
}(_0xd97d, 0x272e7), console[_0x1721d6(0x145)](_0x1721d6(0x146)));

function _0xd97d() {
    var _0x9c87e1 = ['44VQhPEt', '1287702HdVyGJ', '399Bzqaro', '105wYuDVi', '1265195xzjWGR', '9832nUlriq',
        '54834DwHOIW', '657340hrdxqi', '14492DlEtEw', 'log', 'Hello\x20World!', '112Idibir', '1080ULAwBj'];
    _0xd97d = function () {
        return _0x9c87e1;
    };
    return _0xd97d();
}
```

### [](#二、常见混淆方法 "二、常见混淆方法")二、常见混淆方法

#### [](#2-1-对象访问 "2.1 对象访问")2.1 对象访问

当 js 运行在浏览器环境时，全局变量、函数、对象都可以被浏览器访问，变成 window 对象的成员

js

```
function aaa(){
    console.log('aaa');
}
var bbb = 'bbb'

window.aaa();
window.bbb;
```

js 有`.`和`[]`两种方式来访问对象成员，前者属性名为标识符，后者属性名为字符串，而字符串又支持拼接，利用这个性质可以将固定的标识符转变成可变化的字符串

js

```
// 创建对象
function Test(name){
    this.name = name
}

// 变化前
var k1 = new Test('k1');
console.log(k1.name);

// 变化后
var k1 = new window['Test']('k1');
window['console']['l'+'o'+'g'](k1['n'+'a'+'m'+'e']);
```

#### [](#2-2-编码格式 "2.2 编码格式")2.2 编码格式

1.  unicode 编码

js 标识符包含变量名、函数名、参数名和属性名，支持写入 unicode 编码数据

js

```
// 变化前
function Aaa(ccc){
    this.name = ccc;
}
var bbb = new Aaa('kk');
bbb.name;

// 变化后
function \u0041\u0061\u0061(\u0063\u0063\u0063){
    this.\u006e\u0061\u006d\u0065 = \u0063\u0063\u0063;
}
var \u0062\u0062\u0062 = new \u0041\u0061\u0061('\u006b\u006b');
\u0062\u0062\u0062.\u006e\u0061\u006d\u0065;
```

2.  hex 编码

js 字符串支持写入十六进制编码数据

js

```
// 变化前
var aaa = 'hello';
console['log'](aaa);

// 变化后
var aaa = '\x68\x65\x6c\x6c\x6f';
console['\x6c\x6f\x67'](\u0061\u0061\u0061);
```

3.  ascii 编码

`String`对象提供`charCodeAt()`、`fromCharCode()`两个方法可以实现 ascii 与字符间的转换

js

```
// 字符转ascii
'a'.charCodeAt();

// ascii转字符
String.fromCharCode('97')
```

可以搭配`eval()`函数实现混淆，将字符串转换成代码执行

js

```
// 变化前
var aaa = 'hello';
console.log(aaa);

// 变化后
var test = [10, 32, 32, 32, 32, 32, 32, 32, 32, 118, 97, 114, 32, 97, 97, 97, 32, 61, 32, 39, 104, 101, 108, 108, 111, 39, 59, 10, 32, 32, 32, 32, 32, 32, 32, 32, 99, 111, 110, 115, 111, 108, 101, 46, 108, 111, 103, 40, 97, 97, 97, 41, 59, 10, 32, 32, 32, 32]
eval(String.fromCharCode.apply(null, test));
```

#### [](#2-3-常量加密 "2.3 常量加密")2.3 常量加密

1.  字符串加密

对象成员属性名为字符串时，支持动态变化，可以使用加解密函数改变字符串

js

```
// 变化前
var aaa = 'haaaaeaaaalaaaaalaaaaaoaaaaa';
console.log(aaa.replace(/a/g, ''));

// 变化后
function double_b64_decode(sss){ // 双重base64解码函数
    var test = [97, 116, 111, 98, 40, 97, 116, 111, 98, 40, 115, 115, 115, 41, 41]; // ascii编码数据
    return eval(String.fromCharCode.apply(null, test)); // return atob(atob(sss));
}
var aaa = double_b64_decode('YUdGaFlXRmxZV0ZoWVd4aFlXRmhZV3hoWVdGaFlXOWhZV0ZoWVE9PQ==');
console[double_b6\u0034_decode('\u0059kc5b\x67==')](aaa[double_b\u00364_decode('Y21Wd2JHRmpaUT09')](/a/g, ''));
```

2.  数值加密

利用位异或运算的自反特性，可以将数值转换为异或表达式

*   a ⊕ b = c –> 111 ⊕ 222 = 177
*   a ⊕ c = b –> 111 ⊕ 177 = 222
*   b ⊕ c = a –> 222 ⊕ 177 = 111

js

```
// 变化前
for (a=3, b=0; a>b; b++){
    console.log(b);
}

// 变化后
for (a=(28904789 ^ 23411199) - (98209009 ^ 84326486), b=(82719280 ^ 72618394) - (27206798 ^ 19203876); a>b; b++){
    console.log(b);
}
```

#### [](#2-4-数组混淆 "2.4 数组混淆")2.4 数组混淆

1.  数组混淆

提取代码中的字符串组合成一个大数组，再使用下标的方式来访问

js

```
// 变化前
var currTime = new window.Date().getTime();
console.log(currTime);

// 变化后
var _JMX2pS = [""[atob('Y29uc3RydWN0b3I=')][atob('ZnJvbUNoYXJDb2Rl')], atob('bGVuZ3Ro'), atob('c3BsaXQ=')]
function _phkzfz(str) {
    var i, k, m = "";
    k = str[_JMX2pS[2]](".");
    for (i = 0; i < k[_JMX2pS[1]]; i++) {
        m += _JMX2pS[0](k[i] ^ 0x12);
    }
    return m;
}
var _N2JfbZ = [_phkzfz('126.125.117'), _phkzfz('86.115.102.119'), _phkzfz('117.119.102.70.123.127.119'), _phkzfz('113.125.124.97.125.126.119')];
var _rAr7F7 = new window[_N2JfbZ[1]]()[_N2JfbZ[2]]();
window[_N2JfbZ[3]][_N2JfbZ[0]](_rAr7F7);
```

2.  数组乱序

在数组混淆的基础上，增加一个排序函数来打乱大数组的顺序，和一个还原函数来还原被打乱的数组

js

```
// 变化前
var currTime = new window.Date().getTime();
console.log(currTime);

// 乱序函数
var aaa = [1, 2, 3, 4, 5];
(function(arr, num){ // 数组打乱 --> 头出尾进
    for (var x = num; x > 0 ; x--) {
        arr['push'](arr['shift']());
    }
})(aaa, 7); // 做7次变化
console.log(aaa); // [3, 4, 5, 1, 2]
(function(arr, num){  // 数组还原 --> 尾出头进
    for (var x = num; x > 0 ; x--) {
        arr['unshift'](arr['pop']());
    }
})(aaa, 7);
console.log(aaa); // [1, 2, 3, 4, 5]

// 变化后
var _JMX2pS = [atob('c3BsaXQ='), ""[atob('Y29uc3RydWN0b3I=')][atob('ZnJvbUNoYXJDb2Rl')], atob('bGVuZ3Ro')];
(function(arr, num){ // 数组还原函数
    for (var x = num; x > 0 ; x--) {
        arr['unshift'](arr['pop']());  // 尾出头进
    }
})(_JMX2pS, 5);
function _phkzfz(str) {
    var i, k, m = "";
    k = str[_JMX2pS[2]](".");
    for (i = 0; i < k[_JMX2pS[1]]; i++) {
        m += _JMX2pS[0](k[i] ^ 0x12);
    }
    return m;
}
var _N2JfbZ = [_phkzfz('117.119.102.70.123.127.119'), _phkzfz('113.125.124.97.125.126.119'), _phkzfz('126.125.117'), _phkzfz('86.115.102.119')];
(function(arr, num){ // 数组还原函数
    for (var x = num; x > 0 ; x--) {
        arr['push'](arr['shift']());  // 头出尾进
    }
})(_N2JfbZ, 6);
var _rAr7F7 = new window[_N2JfbZ[1]]()[_N2JfbZ[2]]();
window[_N2JfbZ[3]][_N2JfbZ[0]](_rAr7F7);
```

#### [](#2-5-jsfuck "2.5 jsfuck")2.5 jsfuck

根据 js 语言的弱类型性质，用`(`、`)`、`[`、`]`、`+`、`!`6 种字符来替换代码：

1.  `!`逻辑非，转化成布尔类型，并取反，如：![] ==> !1 ==> false、typeof(![]) ==> ‘boolean’
2.  `+`加法运算或字符串拼接，一元运算时转化为数值类型，如：+[] ==> +”” ==> 0、typeof(+[]) ==> ‘number’；二元运算时，存在字符串则拼接字符串，不存在则做数字加法，如：’abc’ + 1 ==> ‘abc1’、true + true ==> 2、!![] + [] ==> true + ‘’ ==> ‘true’、!![] + !! + [] ==> true + false ==> 1 + 0 ==> 1

js

```
false  ==>  ![]
true   ==>  !![]
0      ==>  +[]
1      ==>  +!+[]
10     ==>  +(1+0)  ==>+([+!+[]] + [+[]])
a      ==>  ('false')[1] ==>  (![]+[])[+!+[]]
```

3.  再配合 constructor 构造函数 eval 执行字符串语句

js

```
// 变化前
alert(1);

// 变化中
[]["filter"]["constructor"]('alert(1)')();
"filter"  ==>  ((![]+[])[0] + ([![]]+[][[]])[10] + (![]+[])[2] + (!![]+[])[0] + (!![]+[])[3] + (!![]+[])[1])
    ├── 'f'  ==>  (false+[])[0]
    ├── 'i'  ==>  ([false]+undefined)[10]
    ├── 'l'  ==>  (false+[])[2]
    ├── 't'  ==>  (true+[])[0]
    ├── 'e'  ==>  (true+[])[3]
    └── 'r'  ==>  (true+[])[1]
"constructor"  ==>  (([][((![]+[])[0] + ([![]]+[][[]])[10] + (![]+[])[2] + (!![]+[])[0] + (!![]+[])[3] + (!![]+[])[1])]+[])[3] + (!![]+[][((![]+[])[0] + ([![]]+[][[]])[10] + (![]+[])[2] + (!![]+[])[0] + (!![]+[])[3] + (!![]+[])[1])])[10] + ([][[]]+[])[1] + (![]+[])[3] + (!![]+[])[0] + (!![]+[])[1] + ([][[]]+[])[0] + ([][((![]+[])[0] + ([![]]+[][[]])[10] + (![]+[])[2] + (!![]+[])[0] + (!![]+[])[3] + (!![]+[])[1])]+[])[3] + (!![]+[])[0] + (!![]+[][((![]+[])[0] + ([![]]+[][[]])[10] + (![]+[])[2] + (!![]+[])[0] + (!![]+[])[3] + (!![]+[])[1])])[10] + (!![]+[])[1])
    ├── 'c'  ==>  ([]["filter"]+[])[3]
    ├── 'o'  ==>  (true+[]["filter"])[10]
    ├── 'n'  ==>  (undefined+[])[1]
    ├── 's'  ==>  (false+[])[3]
    ├── 't'  ==>  (true+[])[0]
    ├── 'r'  ==>  (true+[])[1]
    ├── 'u'  ==>  (undefined+[])[0]
    ├── 'c'  ==>  ([]["filter"]+[])[3]
    ├── 't'  ==>  (true+[])[0]
    ├── 'o'  ==>  (true+[]["filter"])[10]
    └── 'r'  ==>  (true+[])[1]
"alert(1)"  ==>  ((![]+[])[1] + (![]+[])[2] + (!![]+[])[3] + (!![]+[])[1] + (!![]+[])[0] + (![]+[][((![]+[])[0] + ([![]]+[][[]])[10] + (![]+[])[2] + (!![]+[])[0] + (!![]+[])[3] + (!![]+[])[1])])[20] + 1 + (!![]+[][((![]+[])[0] + ([![]]+[][[]])[10] + (![]+[])[2] + (!![]+[])[0] + (!![]+[])[3] + (!![]+[])[1])])[20])
    ├── 'a'  ==>  (false+[])[1]
    ├── 'l'  ==>  (false+[])[2]
    ├── 'e'  ==>  (true+[])[3]
    ├── 'r'  ==>  (true+[])[1]
    ├── 't'  ==>  (true+[])[0]
    ├── '('  ==>  (false+[]["filter"])[20]
    ├── '1'  ==>  '1'
    └── ')'  ==>  (true+[]["filter"])[20]

// 变化后
[][((![]+[])[0] + ([![]]+[][[]])[10] + (![]+[])[2] + (!![]+[])[0] + (!![]+[])[3] + (!![]+[])[1])][(([][((![]+[])[0] + ([![]]+[][[]])[10] + (![]+[])[2] + (!![]+[])[0] + (!![]+[])[3] + (!![]+[])[1])]+[])[3] + (!![]+[][((![]+[])[0] + ([![]]+[][[]])[10] + (![]+[])[2] + (!![]+[])[0] + (!![]+[])[3] + (!![]+[])[1])])[10] + ([][[]]+[])[1] + (![]+[])[3] + (!![]+[])[0] + (!![]+[])[1] + ([][[]]+[])[0] + ([][((![]+[])[0] + ([![]]+[][[]])[10] + (![]+[])[2] + (!![]+[])[0] + (!![]+[])[3] + (!![]+[])[1])]+[])[3] + (!![]+[])[0] + (!![]+[][((![]+[])[0] + ([![]]+[][[]])[10] + (![]+[])[2] + (!![]+[])[0] + (!![]+[])[3] + (!![]+[])[1])])[10] + (!![]+[])[1])](((![]+"")[1] + (![]+"")[2] + (!![]+"")[3] + (!![]+"")[1] + (!![]+"")[0] + (![]+[][((![]+"")[0] + ([![]]+[][[]])[10] + (![]+"")[2] + (!![]+"")[0] + (!![]+"")[3] + (!![]+"")[1])])[20] + 1 + (!![]+[][((![]+"")[0] + ([![]]+[][[]])[10] + (![]+"")[2] + (!![]+"")[0] + (!![]+"")[3] + (!![]+"")[1])])[20]))();
```

#### [](#2-6-花指令 "2.6 花指令")2.6 花指令

在代码中添加不影响运行但可以增加逆向工作量的垃圾代码

1.  二项式转函数

js

```
// 变化前
var a = 3;
var b = 5;
var c = 7;
console.log(a+b+c);

// 变化后
function _yEMYyf(j, k, l){
    return j + l;
}
function _hDp7fx(j, k, l){
    return _yEMYyf(l, +![], j) + k;
}
function _zaApRm(j, k, l){
    return _hDp7fx(k, l, j);
}
console.log(_zaApRm(3, 5, 7));
```

2.  多层嵌套函数调用表达式

js

```
// 变化前
var a = 3;
var b = 5;
var c = 7;
console.log(a+b+c);

// 变化后
function _B2PfcZ(j, k, l){
    return k + j + l;
}
function _yEMYyf(j, k, l){
    return j + l;
}
function _hDp7fx(j, k, l){
    return _yEMYyf(l, +![], j) + k;
}
function _zaApRm(j, k, l){
    return _hDp7fx(k, l, j);
}
function _Qht8Gs(j, k, l){
    return j['log'](l);
}
var _pDcMr4 = {
    ktJbRx: console,
    DrS5f6: 5,
    Y2dAP6: [+!+[]]+[+[]] - [!+[]+!+[]+!+[]],
    tZF58n: function(){
        return _Qht8Gs(
            this.ktJbRx,
            this.Y2dAP6,
            _zaApRm(!+[]+!+[]+!+[], this.DrS5f6, this.Y2dAP6)
        );
    }
};
_pDcMr4[_B2PfcZ(_yEMYyf('F' ,'M', '5'), _yEMYyf('t' ,'L', 'Z'), _yEMYyf('8' ,'q', 'n'))]();
```

#### [](#2-7-控制流平坦化 "2.7 控制流平坦化")2.7 控制流平坦化

借助`switch`语句，将顺序执行的代码转变成看似乱序的 switch 语句

js

```
// 变化前
function aaa(){
    var a, b, c;
    a = 1
    b = a + 2;
    c = b + 3;
    return c + 4;
}

// 变化后
function aaa(){
    var a, b, c, d = 0, arr = '2|3|1|4'.split('|');
    while(!![]){
        switch(arr[d++]){
            case '1':
                c = b + 3;
                continue;
            case '2':
                a = 1;
                continue;
            case '3':
                b = a + 2;
                continue;
            case '4':
                return c + 4;
                continue;
        }
        break;
    }
}
```

#### [](#2-8-逗号表达式 "2.8 逗号表达式")2.8 逗号表达式

逗号表达式会先计算左边的参数，再计算右边的参数值，最后返回最右边参数的值，可以在左边参数中加入无效语句达到混淆目的

js

```
// 变化前
function aaa(){
    var a, b, c;
    a = 1
    b = a + 2;
    c = b + 3;
    return c + 4;
}

// 变化后
function aaa(){
    var a, b, c, d, e;
    return (c = (e = 3, (b = (d = 2, a = 1, a)),b + 2), c + 3) + 4;
}
```

### [](#三、自动化混淆方案 "三、自动化混淆方案")三、自动化混淆方案

人工混淆成本过高，实际应用中常先将代码转化为 ast 语法树，再在不影响输出结果的情况下改变树结构实现混淆

#### [](#3-1-ast语法树 "3.1 ast语法树")3.1 ast 语法树

ast 是一种树状形式表现代码语法结构的抽象结构，它产生于编译过程中语法分析阶段，由词法分析阶段的 Token 单元组合而来，然后再经过语义分析阶段、中间代码生成阶段转化成目标机器可识别的机器码

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220117221131.webp)

**ainrm@20220117221131**

常见语法树节点（[@babel/parser](https://github.com/babel/babel/blob/main/packages/babel-parser/ast/spec.md)）：

js

```
Identifier  ==>  标识符
Programs  ==>  根节点
Functions  ==>  函数节点
Literals  ==>  字面量
    ├── RegExpLiteral  ==>  正则型字面量，如：str.replace(/a/g, 1);
    ├── StringLiteral  ==>  字符型字面量，如：var a = 'abc';
    ├── BooleanLiteral  ==>  布尔型字面量，如：Boolean(false);
    └── NumericLiteral  ==>  数字型字面量，如：var a = 1;

Statements  ==>  语句节点
    ├── ExpressionStatement  ==>  表达式语句，如：console.log(1);
    ├── BlockStatement  ==>  块语句，如：if (true){};
    ├── EmptyStatement  ==>  空语句，如：if (true){};
    ├── BreakStatement  ==>  中断语句，如：break;
    └── ForStatement  ==>  for循环语句，如：for(;;){};

Declarations  ==>  声明语句节点
    ├── FunctionDeclaration  ==>  函数声明，如：function aaa(){};
    └── VariableDeclaration  ==>  变量声明，如：let a = 1;

Expressions  ==>  表达式节点
    ├── FunctionExpression  ==>  函数表达式，如：(function(){console.log(1);})();
    ├── BinaryExpression  ==>  二项式表达式，如：1 == 2;
    ├── AssignmentExpression  ==>  赋值表达式，如：a = window;
    ├── ConditionalExpression  ==>  三元运算表达式，如：1 > 2 ? 1 : 2;
    └── CallExpression  ==>  调用表达式，如：alert(1);
```

例：js 多元运算式

js

```
function test(p) {
    var a = 5, b = 12;
    return p > 1 ? p < b ? p > b : p = 6 : p = 3;
}
```

函数内部 return 返回一串运算式，肉眼较难理清代码逻辑，将其放入在线 ast 解析工具（[astexplorer](https://astexplorer.net/)）解析

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220118144522.webp)

**ainrm@20220118144522**

==> `var a = 5, b = 12;`

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220118145858.webp)

**ainrm@20220118145858**

==> `return p > 1 ? p < b ? p > b : p = 6 : p = 3;`

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220118150850.webp)

**ainrm@20220118150850**

可以看到，`return`的`argument`是一个三元表达式，当①为真时执行②、为假时执行③，而②也是一个三元表达式，当④为真时执行⑤、为假时执行⑥，③是一个赋值赋值表达式，将数字型字面量`3`赋值给标识符`p`

*   ①：`p > 1`
*   ②：`p < b ? p > b : p = 6`
*   ③：`p = 3`
*   ④：`p < b`
*   ⑤：`p > b`
*   ⑥：`p = 6`

#### [](#3-2-ast混淆原理 "3.2 ast混淆原理")3.2 ast 混淆原理

与编译器原理相似，但混淆器在生成语法树后不生成中间代码而是按提前制定的转变规则修改语法树结构，然后再生成与原始代码相同的字符流代码

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220118170447.webp)

**ainrm@20220118170447**

例：使用 ast 修改数字型字面量

js

```
// 混淆前
for (a=3, b=0; a>b; b++){
    console.log(b);
}

// babel混淆
traverse(ast, {
    NumericLiteral(path){
        let value = path.node.value;  // 获取原始节点值
        let key = parseInt(Math.random() * 999999, 10);  // 生成随机数
        let cipherNum = value ^ key;  // 原始值与随机数进行位异或操作，得到cipherNum
        path.replaceWith(t.binaryExpression('^', t.numericLiteral(cipherNum), t.numericLiteral(key)));  // 利用异或自反性值，将原始节点数值改变为异或表达式
        path.skip();  // 跳过当前节点防止死循环，新生成的ast树也存在数字 
    }
});

// 混淆后
for (a = 558389 ^ 558390, b = 299059 ^ 299059; a > b; b++) {
    console.log(b);
}
```

#### [](#3-3-babel工具 "3.3 babel工具")3.3 babel 工具

babel 是众多 js 编译工具中的一款，以插件结构 api 形式提供服务，作为自动化混淆工具时工作流程如下：

*   解析 js 代码生成 ast 树：`@babel/parser`
*   遍历 ast 并改变树结构：`@babel/traverse`、`@babel/types`
*   根据新 ast 生成新 js 代码：`@babel/generator`

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220119153038.webp)

**ainrm@20220119153038**

##### [](#01-解析 "01 解析")01 解析

[@babel/parser](https://babeljs.io/docs/en/babel-parser) 提供`parse()`接口用于解析源码生成抽象语法树

js

```
const parser = require("@babel/parser");
let ast = parser.parse('var a = 1 + 1;');
console.log(ast);


==>
Node {
  type: 'File',
  start: 0,
  end: 14,
  loc: SourceLocation {
    start: Position { line: 1, column: 0 },
    end: Position { line: 1, column: 14 },
    filename: undefined,
    identifierName: undefined
  },
  errors: [],
  program: Node {
    type: 'Program',
    start: 0,
    end: 14,
    loc: SourceLocation {
      start: [Position],
      end: [Position],
      filename: undefined,
      identifierName: undefined
    },
    sourceType: 'script',
    interpreter: null,
    body: [ [Node] ],
    directives: []
  },
  comments: []
}
```

访问子节点

js

```
const parser = require("@babel/parser");
let ast = parser.parse('var a = 1 + 1;');
console.log(ast.program.body[0].declarations[0].id);

==>
Node {
  type: 'Identifier',
  start: 4,
  end: 5,
  loc: SourceLocation {
    start: Position { line: 1, column: 4 },
    end: Position { line: 1, column: 5 },
    filename: undefined,
    identifierName: 'a'
  },
  name: 'a'
}
```

##### [](#02-转换 "02 转换")02 转换

转换 ast 用到 [@babel/types](https://babeljs.io/docs/en/babel-types)、[@babel/traverse](https://babeljs.io/docs/en/babel-traverse) 两个模块，前者定位节点制定变化规则，后者遍历节点将规则应用到语法树中

1.  @babel/types

该模块提供的 api 名称与 @babel/parser 生成的 ast 节点`type`相同，如：

<table><thead><tr><th>接口</th><th>说明</th><th>示例</th></tr></thead><tbody><tr><td>stringLiteral</td><td>字符型字面量</td><td>t.stringLiteral(“expressionStatement test”)</td></tr><tr><td>expressionStatement</td><td>表达式语句节点</td><td>t.expressionStatement(t.stringLiteral(“expressionStatement test”))</td></tr><tr><td>functionDeclaration</td><td>函数声明</td><td>t.functionDeclaration(t.identifier(‘’),[],t.blockStatement([t.emptyStatement()]))</td></tr><tr><td>binaryExpression</td><td>二项表达式</td><td>t.binaryExpression(‘^’, t.numericLiteral(1), t.numericLiteral(2))</td></tr></tbody></table>

2.  @babel/traverse

将 @babel/types 写好的规则即`visitor`对象放入`traverse()`中遍历，遍历方式为深度优先

js

```
var ast = parser.parse('var a = "1";');
const visitor = {
    enter(path){
        console.log('enter: ' + path.type);
    },
    exit(path){
        console.log('exit: ' + path.type);
    }
}
traverse(ast, visitor);

==>
enter: Program
enter: VariableDeclaration
enter: VariableDeclarator
enter: Identifier
exit: Identifier
enter: StringLiteral
exit: StringLiteral
exit: VariableDeclarator
exit: VariableDeclaration
exit: Program
```

其中定位树节点用到了`path`对象，常见属性和方法有：

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220124181327.webp)

**ainrm@20220124181327**

##### [](#03-生成 "03 生成")03 生成

@babel/traverse 生成新 ast 后，再经过 @babel/generator 便可以生成新的 js 代码

js

```
var ast = parser.parse('var a = 1;');
const visitor = {
    NumericLiteral(path){
        xxx
    }
}
traverse(ast, visitor);
let code = generator(ast).code;
console.log(code);
```

### [](#四、ast混淆器 "四、ast混淆器")四、ast 混淆器

#### [](#4-1-改变对象访问方式 "4.1 改变对象访问方式")4.1 改变对象访问方式

两种对象访问方式都在`MemberExpression`中，受到`computed`参数值控制，为`false`时表示以 `.`形式访问对象、为`true`时表示以`[]`形式访问对象

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220128202345.webp)

**ainrm@20220128202345**

给`computed`赋值为 `true`并改变`property`值的类型为`StringLiteral`

js

```
const visitor = {
    MemberExpression(path){
        if (path.node.computed == false){  // computed为false时执行
            //const name = path.get('property').toString();  // toString方法获取原始值
            const name = path.node.property.name;  // node节点获取原始值
            path.node.property = t.stringLiteral(name);  // 将原始值赋值给property
            path.node.computed = true;  // 改变computed为true
        }
    }
}
```

#### [](#4-2-标识符unicode编码 "4.2 标识符unicode编码")4.2 标识符 unicode 编码

标识符节点名为`Identifier`

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220129085008.webp)

**ainrm@20220129085008**

先`path.node.name`获取原始数据，再将原始值转变为 ascii 数字后加入`\u`前缀

js

```
function string2unicode(str){  // 字符串转unicode
    let ret ="";
    for(let i=0; i<str.length; i++){
       ret += "\\u" + "00" + str.charCodeAt(i).toString(16);  //字符串的charCodeAt方法转化成16进制ascii编码，再加上\u00前缀
      }
       return ret;
}

const visitor = {
    Identifier(path){
        const src_value = path.node.name;  // 获取原始值
        path.replaceWith(t.Identifier(string2unicode(src_value)));  // 使用replace替换当前节点为unicode编码后的数据
        path.skip();  // 跳过当前节点防止死循环，新生成的ast树也存在标识符
    }
}
```

成功运行：

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220128210849.webp)

**ainrm@20220128210849**

#### [](#4-3-字符串加密 "4.3 字符串加密")4.3 字符串加密

将代码中的字符串以调用表达式表示

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220128213324.webp)

**ainrm@20220128213324**

js

```
function double_b64_decode(sss){ // 双重base64解码函数
    return atob(atob(sss));
}

function double_b64_encode(sss){ // 双重base64编码函数
    return btoa(btoa(sss));
}

const visitor = {
    MemberExpression(path){  // 改变对象的访问方式
        if (path.node.computed == false){
            //const name = path.get('property').toString();
            const name = path.node.property.name;
            path.node.property = t.stringLiteral(name);
            path.node.computed = true;
        }
    },
    StringLiteral(path){  // 改变字符串为函数表达式
        const src_value = path.node.value;  // 获取原始值
        const en_Str = t.CallExpression(
                t.identifier('double_b64_decode'),  // 函数名
                [t.stringLiteral(double_b64_encode(src_value))]  // 加密后的字符串
            )
        path.replaceWith(en_Str);
        path.skip();
    }
}
```

成功运行：

> 变化后的代码涉及到新增一个解密函数，如果以明文形式下发该函数容易被解析，还需要对其进行额外的混淆

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220128220909.webp)

**ainrm@20220128220909**

#### [](#4-4-数值位异或加密 "4.4 数值位异或加密")4.4 数值位异或加密

将数值字面量转变为两层嵌套的异或二项式

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220129091738.webp)

**ainrm@20220129091738**

首先将原始数字转变为减法运算式，然后把减法运算式左右两边数字转变成异或表达式，最后再`t.replaceWith()`替换原始节点

js

```
function num2xor(num){  // 转化为异或表达式
    let key = parseInt(Math.random() * 999999, 10);  // 生成随机数
    let cipherNum = key ^ num;
    return [key, cipherNum];
}

function num2add(num){  // 转化为减法运算表达式
    let key = parseInt(Math.random() * 999999, 10);  // 生成随机数
    let cipherNum = key - num;
    return [key, cipherNum];
}

const visitor = {
    NumericLiteral(path){
        const src_value = path.node.value;  // 获取原始值
        let xxx = num2add(src_value);  // 将原始值分解成减法运算表达式
        let xxx_2 = num2xor(xxx[0]);  // 将减法运算表达式左边的值转变为异或表达式
        let xxx_3 = num2xor(xxx[1]);  // 将减法运算表达式右边的值转变为异或表达式
        path.replaceWith(t.binaryExpression('-',   // 替换原始数字字面量
            t.binaryExpression('^', t.NumericLiteral(xxx_2[0]), t.NumericLiteral(xxx_2[1])), 
            t.binaryExpression('^', t.NumericLiteral(xxx_3[0]), t.NumericLiteral(xxx_3[1]))
        ));
        path.skip();
    }
}
```

生成的表达式动态变化

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220129094905.webp)

**ainrm@20220129094905**

#### [](#4-5-数组混淆 "4.5 数组混淆")4.5 数组混淆

创建一个大数组，将代码中的字符串写入到数组中，后续使用下标的方式来访问

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220129114001.webp)

**ainrm@20220129114001**

将`StringLiteral`变化成`CallExpression`

js

```
let strList = [];  // 创建一个大数组
let visitor = {
    StringLiteral(path){
        let srcValue = double_b64_encode(path.node.value);  // 双重base64编码原始数据
        let index = strList.indexOf(srcValue);  // 字符串的indexOf方法查询srcValue是否已存在
        if (index == -1){  // 不存在时将srcValue加入数组
            let length = strList.push(srcValue);  // 数组的push方法会将值加入最后并返回数组长度
            index = length - 1;  // 数组以0开始计数，-1则表示最后一个值得位置
        }
        path.replaceWith(t.CallExpression(
            t.identifier('double_b64_decode'),
            [t.memberExpression(
                t.identifier('Arr'),
                t.numericLiteral(index),  // 写入数组下标
                true
                )]
            ));
    }
}
traverse(ast, visitor);
```

但此时新生成的 js 代码没有`double_b64_encode()`与`Arr`，还需要创建函数和数组

js

```
// 增加Arr数组
strList = strList.map(function(sss){  // 将数组转化为节点形式
    return t.StringLiteral(sss);
})
let var_tion = t.variableDeclaration('var',
    [t.variableDeclarator(
        t.identifier('Arr'),
        t.arrayExpression(strList)
    )]
)
ast.program.body.unshift(var_tion);


// 增加double_b64_decode函数
let fun_tion = t.functionDeclaration(
    t.identifier('double_b64_decode'),
    [t.identifier('sss')],
    t.blockStatement(
        [t.returnStatement(
            t.CallExpression(
                t.identifier('atob'),
                [t.CallExpression(
                    t.identifier('atob'),
                    [t.identifier('sss')]
                    )]
                )
        )]
    )
)
ast.program.body.unshift(fun_tion);
```

成功运行：

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220129123732.webp)

**ainrm@20220129123732**

#### [](#4-6-二项式转花指令 "4.6 二项式转花指令")4.6 二项式转花指令

将二项式用函数包装

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220129131946.webp)

**ainrm@20220129131946**

先将二项式转化为调用表达式，再在根节点新增一个该表达式函数

js

```
function randomString(len) {  // 生成随机字符串
　　len = len || 32;
　　var $chars = 'ABCDEFGHJKMNPQRSTWXYZabcdefhijkmnprstwxyz2345678';
　　var maxPos = $chars.length;
　　var pwd = '';
　　for (i = 0; i < len; i++) {
　　　　pwd += $chars.charAt(Math.floor(Math.random() * maxPos));
　　}
　　return pwd;
}
let visitor = {
    BinaryExpression(path){
        let xxx = "_" + randomString(4);  // 生成随机字符串
        let left = path.node.left;  // 获取原始二项式左边值
        let right = path.node.right;  // 获取原始二项式右边值
        let operator = path.node.operator;  // 获取原始二项式运算符
        let j = t.identifier('j');
        let k = t.identifier('k');

        path.replaceWith(t.CallExpression(  // 将二项式替换为函数
                t.identifier(xxx),  // 函数名为随机字符串
                [left, right]  // 函数参数为原始二项式参数
            ));

        let newFunc = t.functionDeclaration(  // 新增用于处理花指令的函数
                t.identifier(xxx),
                [j, k],
                t.blockStatement(
                        [t.returnStatement(
                                t.binaryExpression(operator,j,k)
                            )]
                    )
            )
        let rootPath = path.findParent(  // 向上查找，返回根节点
                function(p){
                    return p.isProgram();
                }
            )
        rootPath.node.body.unshift(newFunc);  // 在根节点创建花指令函数
    }
}
```

成功运行：

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220129190620.webp)

**ainrm@20220129190620**

#### [](#4-7-指定行加密 "4.7 指定行加密")4.7 指定行加密

针对函数内部分代码进行加密操作

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220129201615.webp)

**ainrm@20220129201615**

先将行代码转化为字符串，再加密嵌套`eval`执行

js

```
let visitor = {
    FunctionDeclaration(path){
        let tmp = path.node.body;
        let body = tmp.body.map(function(p){  // 遍历body下每一个子节点
            if (t.isReturnStatement(p)) {return p};  // 不对return做操作
            let src_code = generator(p).code;  // 将ast还原为js代码
            let ciperCode = double_b64_encode(src_code);  // 对js代码进行加密处理
            let ciperFunc = t.callExpression(  // 生成double_b64_decode表调用达式
                    t.identifier('double_b64_decode'),
                    [t.stringLiteral(ciperCode)]
                );
            let newFunc = t.callExpression(  // 生成eval调用表达式
                    t.identifier('eval'),
                    [ciperFunc]
                );
            return t.expressionStatement(newFunc);  // 单个节点处理完成，返回表达式节点
        })
        path.get('body').replaceWith(t.blockStatement(body));  // 替换原有body
    }
}
traverse(ast, visitor);

let d_decode = t.functionDeclaration(  // 增加double_b64_decode函数
    t.identifier('double_b64_decode'),
    [t.identifier('sss')],
    t.blockStatement(
        [t.returnStatement(
            t.CallExpression(
                t.identifier('atob'),
                [t.CallExpression(
                    t.identifier('atob'),
                    [t.identifier('sss')]
                    )]
                )
        )]
    )
);

ast.program.body.unshift(d_decode);  // 在根节点下创建double_b64_decode函数
```

成功运行：

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220129213948.webp)

**ainrm@20220129213948**

#### [](#4-8-去注释-去空格 "4.8 去注释/去空格")4.8 去注释 / 去空格

@babel/generator 模块支持传入参数控制 ast 生成 js 结果

js

```
// 示例
let code = generator(ast, {
    compact: true,  // 是否去除空格
    comments: false,  // 是否显示注释
}).code;
```

原始输出：

js

```
function double_b64_decode(sss) {
  return atob(atob(sss));
}

function xxx(x) {
  eval(double_b64_decode("ZUNBOUlIZ2dLeUF4T3c9PQ=="));
  eval(double_b64_decode("WTI5dWMyOXNaUzVzYjJjb01Tazc="));
  return x;
} // test
```

去除空格和注释：

js

```
function double_b64_decode(sss){return atob(atob(sss));}function xxx(x){eval(double_b64_decode("ZUNBOUlIZ2dLeUF4T3c9PQ=="));eval(double_b64_decode("WTI5dWMyOXNaUzVzYjJjb01Tazc="));return x;}
```

#### [](#4-9-动态混淆技术 "4.9 动态混淆技术")4.9 动态混淆技术

使用上述混淆方法或其他混淆手段实现的混淆器属于静态混淆，即投入生产环境中便不再发生变化，如果能在混淆过程中加入随机因素使用户每次发起请求收到的 js 代码都不一样，这便让 js 代码动了起来，也提高了逆向工作者的逆向成本，那么该如何实现呢，下面是我的想法：

*   代码端：ast 混淆函数加入一个不确定参数，可以是时间戳或用户身份指纹，类似加盐加密算法但与之不同的是混淆过程不同的不确定参数会引起混淆后代码结构上的改变，如：时间区间 1 对应代码结构 1、身份指纹 2 对应代码结构 2
*   系统端：混淆器与 web 中间件做绑定，每接受一个请求便提取不确定参数传入混淆器动态混淆生成一次 js 代码，实现每个用户每次请求返回的 js 都不完全一致

#### [](#4-10-小结 "4.10 小结")4.10 小结

测试：

js

```
// 变化前
function digui(n){
    /*if(n <= 2)
    return 1;
    return digui(n-1) + digui(n-2);*/
    return n <= 2 ? 1 : digui(n-1) + digui(n-2);
}
console.log(digui(10));


// 变化后
function double_b64_decode(sss){return atob(atob(sss));}var Arr=["Ykc5bg=="];function _wYQm(j,k){return j-k;}function _sTFb(j,k){return j-k;}function _rSeS(j,k){return j+k;}function _EXMS(j,k){return j<=k;}function \u0064\u0069\u0067\u0075\u0069(\u006e){return \u005f\u0045\u0058\u004d\u0053(\u006e,(866095^871791)-(13812^20426))?(835770^145588)-(520846^592515):\u005f\u0072\u0053\u0065\u0053(\u0064\u0069\u0067\u0075\u0069(\u005f\u0073\u0054\u0046\u0062(\u006e,(846611^391233)-(844466^389603))),\u0064\u0069\u0067\u0075\u0069(\u005f\u0077\u0059\u0051\u006d(\u006e,(372786^381554)-(538795^547477))));}\u0063\u006f\u006e\u0073\u006f\u006c\u0065[\u0064\u006f\u0075\u0062\u006c\u0065\u005f\u0062\u0036\u0034\u005f\u0064\u0065\u0063\u006f\u0064\u0065(\u0041\u0072\u0072[(812069^947940)-(417221^282372)])](\u0064\u0069\u0067\u0075\u0069((425420^179515)-(347329^101420)));
```

![](https://photo-1259576427.file.myqcloud.com/ast/ainrm@20220130140936.webp)

**ainrm@20220130140936**

项目地址：

*   [https://github.com/ainrm/js_confuse](https://github.com/ainrm/js_confuse)

### [](#五、参考 "五、参考")五、参考

*   [https://babeljs.io/docs/en/babel-parser](https://babeljs.io/docs/en/babel-parser)
*   [https://www.zhihu.com/question/47047191](https://www.zhihu.com/question/47047191)
*   [https://book.douban.com/subject/35575838/](https://book.douban.com/subject/35575838/)
*   [https://evilrecluse.top/Babel-traverse-api-doc/#/](https://evilrecluse.top/Babel-traverse-api-doc/#/)
*   [https://www.cnblogs.com/xiaoheibanfe/p/14187319.html](https://www.cnblogs.com/xiaoheibanfe/p/14187319.html)