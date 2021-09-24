> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [kevinspider.github.io](https://kevinspider.github.io/antijs/)

> 算法还原白盒还原 直接扣算法, 或者是标准算法 理解对方 js 的意思, 能够翻译成其他语言, 可能会占用较长的分析时间 黑盒还原 直接整体调用 (加密复杂, 更新频繁, 整体关联度高) ......

[](#算法还原 "算法还原")算法还原
--------------------

### [](#白盒还原 "白盒还原")白盒还原

*   直接扣算法, 或者是标准算法
*   理解对方 js 的意思, 能够翻译成其他语言, 可能会占用较长的分析时间

### [](#黑盒还原 "黑盒还原")黑盒还原

*   直接整体调用 (加密复杂, 更新频繁, 整体关联度高)
*   不需要关注算法逻辑, 需要模拟浏览器环境, 需要对抗环境检测

### [](#rpc-调用 "rpc 调用")rpc 调用

*   算法复杂度高, 浏览器环境难以模拟
*   找到算法位置, 暴露出来, 直接 rpc 调用, 需要保证浏览器状态 (内存泄漏, 保活)

### [](#浏览器自动化 "浏览器自动化")浏览器自动化

*   无法逆向
*   接近真人, 但是有大量的自动化痕迹;

[](#基本数据类型 "基本数据类型")基本数据类型
--------------------------

*   数值 (Number): 整数和小数
*   字符串 (String): 文本
*   布尔值 (Boolean): 布尔值, true 表示真, false 表示假
*   undefined: 未定义, 或者不存在
*   null: 表示空值
*   对象 (Object): 各种值组成的集合

### [](#原始类型 "原始类型")原始类型

1.  数值
2.  字符串
3.  布尔值

原始类型就是最基本的数据类型, 不能再进行细分;

undefined 和 null 一般看成是两个特殊值;

### [](#合成类型 "合成类型")合成类型

*   对象
    *   狭义的对象 Object
    *   数组 Array
    *   函数 Function

一个对象往往是由多个类型的值组成, 可以看成是一个存放各种值的容器

### [](#查看类型 "查看类型")查看类型

*   typeof: 返回一个值的数据类型
*   instanceof: 表示对象是否是某个构造函数的实例
*   Object.prototype.toString

`typeof` 可以用来检查一个未声明的变量, 而不报错;

```
var tmp1 = "字符串";
var tmp2 = 1;
var tmp3 = 1.1;
var tmp4 = true;


var tmp5=  undefined;
var tmp6 = null; 


var tmp7 = {}; 
var tmp8 = []; 
var tmp9 = function(){}; 

console.log("typeof(tmp1)", typeof (tmp1));
console.log("typeof(tmp2)", typeof (tmp2));
console.log("typeof(tmp3)", typeof (tmp3));
console.log("typeof(tmp4)", typeof (tmp4));
console.log("typeof(tmp5)", typeof (tmp5));
console.log("typeof(tmp6)", typeof (tmp6));
console.log("typeof(tmp7)", typeof (tmp7));
console.log("typeof(tmp8)", typeof (tmp8));
console.log("typeof(tmp9)", typeof (tmp9));


console.log("tmp9.name", tmp9.name);
```

### [](#null-undefined-和布尔值 "null undefined 和布尔值")null undefined 和布尔值

> null 和 undefined 的区别

*   null 表示一个空对象, undefined 表示未定义;
*   null 转为数值的时候为 0, undefined 转为数值的实收为 NaN;

> boolean

布尔值表示真和假, true 表示真, false 表示假;

下列运算符会返回布尔值:

1.  ! (not)
2.  相等运算符: ===, ==, !==, !===
3.  比较运算符: >=, <=, <,>

> 表示 false 的值

在自动数据转换中, 下列值会表示 false:

1.  undefined
2.  null
3.  false
4.  0
5.  NaN
6.  “” 或者 ‘’ 空字符串

其他的值都会被当成 true;

空数组 [] 和空对象 {} 对应的布尔值都是 true;

### [](#数值 "数值")数值

在 js 中, 所有的数值都是 64 位浮点数的形式进行存储的, 也就是说在 js 底层, 没有整数只有浮点数;

因为浮点数精度的问题, js 在进行浮点数运算的时候经常会出现问题:

```
console.log(0.1 + 0.2);
```

#### [](#进制 "进制")进制

*   十进制: 没有前导, 直接用数值表示
*   八进制: 有前缀 0o 或者 0O
*   十六进制: 有前缀 0x 或者 0X
*   二进制: 有前缀 0b 或者 0B

默认情况下, js 内部会将八进制, 十六进制, 二进制转为十进制;

#### [](#NaN "NaN")NaN

NaN 是 js 中的特殊值, 表示非数字 (Not a Number), 主要出现在字符串解析成数字出错的时候;

1.  NaN 不是独立的数据类型, 它是一个特殊值, 它的数据类型依然是 Number
2.  NaN 不等于任何值, 包括它本身 (不等于本身可以用来检测某个值是否是 NaN)
3.  NaN 和任何数运算, 得到的结果都是 NaN;

#### [](#Infinity "Infinity")Infinity

Infinity 用来表示无穷, 一般出现在两种场景下:

1.  正数的数值太大, 或者负数的数值太小;
2.  非 0 的数除以 0, 得到 Infinity

*   js 中数值正向溢出或者负向溢出或者被 0 除都不会报错, 所以单纯的数学运算几乎没有可能抛出异常
*   Infinity 大于一切数值 (除了 NaN); -Infinity 小于一切数值 (除了 NaN)
*   Infinity 和 NaN 比较, 总是返回 false;

#### [](#全局-api "全局 api")全局 api

> parseInt(string[,radix])

`parseInt(string[,radix])`将字符串解析成数值; 如果入参非字符串, 则会调用入参的`toString`方法转换为字符串再进行转换; 如果设置了第二个参数 radix, 则会将字符串按指定的 radix 进制转换成十进制; 返回数值或者 NaN

```
var a = '0xf';
console.log(a, 16);
```

> parseFloat(string)

`parseFloat(string)`将字符串入参解析成浮点数; 返回浮点数或者 NaN ;

```
var a = "4.567";
console.log(parseFloat(a)); 


var b = "4.567abcd";
console.log(parseFloat(b)); 


var c = "1.2.3";
console.log(parseFloat(c)); 


var d = "aaa1.2"; 
console.log(parseFloat(d));
```

> isNaN()

判断某个入参是否是 NaN; 可以利用 NaN 的不等性来进行判断

```
var a = NaN;
if (a != a ){
    console.log("it is NaN");
}
```

> isFinite()

判断某个入参是否是 Infinity;

### [](#字符串 "字符串")字符串

*   字符串和数组相似, 都支持使用 [] 运算符来通过指定索引来获取值;
*   length: 可以获取字符串的长度
*   字符串不能通过 [] 运算符和索引来修改字符串的值

#### [](#字符集 "字符集")字符集

js 中使用的字符集为 Unicode 字符集, 所有的字符串都使用 Unicode 表示;

```
var f\u006F\u006F = 'abc';
console.log(f\u006F\u006F);
```

#### [](#base64-转码 "base64 转码")base64 转码

*   浏览器:
    *   btoa(): 任意值转为 base64 编码
    *   atob(): base64 值解码
    *   非 ASCII 码 (如中文) 要转码之后再 base64;
        *   `encodeURIComponent`
        *   `decodeURIComponent`
*   Nodejs
    *   `var base64encode = Buffer.from("js").toString("base64");`
    *   `var base64decode = Buffer.from(base64encode,'base64').toString();`

### [](#对象 "对象")对象

*   对象就是一组键值对 key-value 的集合, 是一种无序的复合数据集合
*   对象的每个键名又称为属性 property; 它的值可以是任意数据类型;
*   如果一个对象的某个属性的值是函数, 则通常将这个属性称为该对象的方法, 可以像函数一样调用这个方法;
*   属性可以动态创建, 不必在对象声明的时候就全部定义;

#### [](#对象引用 "对象引用")对象引用

*   如果不同的变量名指向同一个对象, 那么他们都是对这个对象的引用; 也就是说这些变量都指向了同一个内存地址, 修改其中任意一个的值都会影响到其他的变量;
*   如果取消某一个变量对于原对象的引用, 不会影响到其他引用该对象的变量
*   这种引用只局限于对象, 如果两个变量指向同一个原始类型的值, 这些变量都是对原始类型的值的拷贝, 改变任意变量都不会影响其他变量

#### [](#属性查看 "属性查看")属性查看

`Obejct.keys()`可以查看该对象自身的属性, 继承来的属性无法查看

```
var a = {
    hello: function(){
        consoel.log("hi");
    },
    table: [1, 2, 3],
    name: "kevin",
    age: 21,
    married: false
}

console.log(Object.keys(a));
```

#### [](#属性删除 "属性删除")属性删除

*   delete 用于删除对象的属性, 删除成功后返回 true
*   删除一个不存在的属性, delete 不会报错, 而且返回 true
*   只有一种情况, delete 命令会返回 false; 那就是删除一条存在的属性, 但是这条属性被定义成不能删除; (定义该属性不能删除 defineProperty)
*   只能删除对象自身的属性, 继承来的属性不能删除

```
var a = {
    hello: function(){
        consoel.log("hi");
    },
    table: [1, 2, 3],
    name: "kevin",
    age: 21,
    married: false
}

delete a.age, delete a.hello;

console.log(Object.keys(a));
```

#### [](#属性存在判断 "属性存在判断")属性存在判断

*   in 运算符可以用于检查对象是否包含某个属性; 检查的是键名, 存在这个属性返回 true, 不存在则返回 false;
*   `hasOwnProperty()`: 判断某个属性是否是该对象自身的属性;

```
var a = {
    hello: function(){
        consoel.log("hi");
    },
    table: [1, 2, 3],
    name: "kevin",
    age: 21,
    married: false
}

console.log(a.hasOwnProperty('table'));

if ("table" in a) {
    console.log("table is property of a");
}
```

#### [](#属性遍历 "属性遍历")属性遍历

*   for in 循环可以用于遍历对象的全部属性; 不仅可以遍历对象自身的属性, 还可以遍历对象继承来的属性;
*   如果只想遍历对象自身的属性, 可以配合 `hasOwnProperty()`进行筛选

```
var a = {
    hello: function(){
        consoel.log("hi");
    },
    table: [1, 2, 3],
    name: "kevin",
    age: 21,
    married: false
}

for (const aKey in a) {
    
    if (a.hasOwnProperty(aKey)) {
        console.log(aKey);
    }
}
```

### [](#函数 "函数")函数

#### [](#函数声明 "函数声明")函数声明

js 中函数声明有三种方式

1.  使用 function 申明 `function a(){}`
2.  函数表达式 `var a = function(){}`
3.  Function 构造函数 `var a = Function("a","b", "return a+b")`或者`var a = new Function("a","b", "return a+b")` 这两种方式效果一样

#### [](#函数是一等公民 "函数是一等公民")函数是一等公民

js 将函数看成是一个值, 与其他数据类型一样, 凡是可以使用其他数据类型的地方都可以使用函数, 例如:

1.  可以将函数赋值给变量或者对象的属性
2.  可以将函数作为参数传递给其他函数
3.  可以将函数作为其他函数的返回值

#### [](#函数变量名提升 "函数变量名提升")函数变量名提升

js 中全局变量名存在变量提升, 函数内部的局部变量也存在变量提升;

```
function outer(){
    console.log(a); 
    console.log(b); 
    var b = 2;
}
outer();
var a = 1;
```

js 中函数的声明也存在变量提升, 可以先调用该方法, 再定义该方法

```
b();

function b(){
    console.log("b called");
}
```

#### [](#函数的属性和方法 "函数的属性和方法")函数的属性和方法

*   name (属性): 返回函数的名字
*   length(属性): 返回函数预期传入的形参数量
*   toString() 方法: 返回函数的字符串源码

#### [](#函数作用域 "函数作用域")函数作用域

*   作用域 scope 是指变量存在的范围
    *   es5 中 js 只有两个作用域
        *   全局作用域: 变量在整个程序中一直存在, 所有地方都可以读取到该变量
        *   函数作用域: 变量只在函数内部存在
    *   es6 中新增了块级作用域
*   函数外部声明的变量就是全局变量, 它可以在函数内部读取到;
*   在函数内部声明的变量就是局部变量, 函数外部无法读取;
*   函数本身的作用域就是其声明时所在的作用域, 与其运行时的作用域无关;
*   函数内部声明的函数, 作用域绑定函数内部 (闭包)

#### [](#函数参数省略 "函数参数省略")函数参数省略

js 中函数的参数不是必须的, 允许省略函数的参数;

函数的 length 属性只和函数声明时形参的个数有关, 和实际调用时传入的参数个数无关;

#### [](#参数传递方式 "参数传递方式")参数传递方式

*   函数的参数如果是原始数据类型 (数值, 字符串, 布尔), 参数传递使用按值传递的方式, 在函数内部修改参数的值不会影响函数外部
*   如果函数的参数是复合数据类型 (数组, 对象, 其他函数), 参数的传递方式是按址传递, 传入的是引用的地址, 因此在函数内部修改参数, 会影响到原始值;

#### [](#arguments-对象 "arguments 对象")arguments 对象

*   因为 js 允许函数有不定数目的参数, 所以需要在函数体的内部可以读取到所有参数, 这就是 arguments 对象的由来
*   arguments 对象包含了函数运行时的所有参数, 这是一个类数组对象, `arguments[0]`就是第一个参数;
*   arguments 对象只能在函数内部使用
*   arguments.length 可以获取函数调用时入参的真正个数
*   arguments.callee 属性可以获取对应的原函数
*   arguments 对象是一个类数组对象, 如果要让他使用真正的数组方法, 需要将 arguments 转换成数组:
    *   `Array.prototype.slice.call(arguments)`
    *   新建数组, 遍历 arguments 将元素 push 到新数组中;

#### [](#闭包 "闭包")闭包

*   要理解闭包首先要理解 js 的作用域; 前面提到的 js 在 es5 中只有两种作用域:
    *   全局作用域
    *   函数作用域
*   在函数的内部可以全局作用域的变量
*   js 中特有的链式作用域结构, 子级会向上一级一级寻找所有父级的变量, 父级的所有变量对于子级来说都是可见的, 反之不成立;

```
var a = 1;
var b = 2;

function f(){
    var b = 3;
    console.log(a,b);
    
    function f1(){
        var b = 4;
        console.log(a,b);
    }
    f1();
}

f();
```

> 链式作用域查找

子级会优先使用自己的作用域, 如果变量存在则使用, 不存在则会依次向上寻找, 直至全局作用域;

> 闭包定义

*   闭包可以简单理解成定义在一个函数内部的函数
*   闭包最大的特点就是它可以记住自己诞生的环境, 本质上, 闭包就是将函数内部和函数外部连接起来的桥梁;
*   闭包最大的用处有两个
    1.  可以直接读取到外层函数内部的变量
    2.  可以让这些变量始终保存在内存中, 闭包让自己诞生的环境一直存在

通过闭包实现简单的计数器

```
function count(){
    var count = 0;
    function f(){
        count++;
        console.log("count", count);
    }
    return f;
}

f = count();

f();
f();
f();
```

#### [](#立即调用函数表达式 "立即调用函数表达式")立即调用函数表达式

js 中有三种立即调用函数的方式

1.  `var f = function(){}();`
2.  `(function(){}())`
3.  `(function(){})()`

通常情况下, 只对匿名函数使用这种立即执行的表达式, 这样有两个目的:

1.  不必为函数命名, 避免污染全局环境
2.  立即调用函数的内部会形成单独的作用域, 可以封装一些外部无法读取的私有变量

#### [](#eval-命令 "eval 命令")eval 命令

*   eval 可以接受一个字符串, 并将字符串当做代码执行
*   eval 没有自己的作用域, 都是使用当前运行的作用域, 所以 eval 会修改当前作用域下的变量的值
*   eval 的本质是在当前作用域中, 注入代码, 经常用于混淆和反爬

> eval 别名调用

eval 的别名调用在 nodejs 下无法跑通, 需要在浏览器下运行;

需要注意, 在 eval 通过别名调用的时候, 作用域永远是全局作用域;

```
var a = 1;
var e = eval;

(function(){
    var a = 2;
    e("console.log(a);"); 
}())
```

### [](#数组 "数组")数组

*   数组 Array 是按次序排列的一组值
*   每个值的位置都有对应的索引
*   数组使用 [ ] 来表示
*   任何类型的数据都可以放入数组中
*   本质上, 数组是特殊的对象, typeof 查看数组的类型返回的是 object
*   Object.keys() 可以返回数组的键名 (索引)

#### [](#数组的属性 "数组的属性")数组的属性

*   length: 表示数组的元素个数, 这个属性是可写的, 可以直接修改数组的 length 属性, 来实现清空数组或删除数组中元素的效果
*   数组本质上是特殊的对象, 支持使用点操作符对数组添加属性;

```
var a = [1, 1.1, true, {}, [], "hello", null, undefined];
console.log("a.length", a.length);

a.name = "add a.name property";
for (const aKey in a) {
    console.log(aKey, a[aKey]);
}

console.log("Object.keys(a)", Object.keys(a));

a.length = 0;
console.log("a", a);
console.log("a['name']", a['name']);
console.log("a.name", a.name);
console.log("a[0]", a[0]);
```

#### [](#数组循环 "数组循环")数组循环

*   for in 循环
*   for 循环
*   while 循环
*   forEach : 只有数组才有该方法; 该方法接受一个回调函数, 回调函数入参为 value 和 key;

```
var a = [1, 1.1, true, {}, [], "hello", null, undefined];


for (const aKey in a) {
    console.log("aKey:", aKey, "value:", a[aKey]);
}

console.log("-------------------------------")


for (var i = 0; i <= a.length; i++) {
    console.log("index:", i, "value:", a[i]);
}

console.log("-------------------------------")


var index = 0;
while (index <= a.length) {
    console.log("index:", index, "value:", a[index]);
    index++;
}

console.log("-------------------------------")


a.forEach(function (value,key) {
    console.log("key:", key, "value:", value);
})
```

#### [](#数组空值 "数组空值")数组空值

js 中的数组支持空值, 出现空值时会占用该索引位, 但是遍历的时候不会遍历该索引的值

```
var a = [1, 2, 3, , 5];
a.forEach(function (value,key) {
    console.log("key", key, "value", value);
})
```

#### [](#类数组对象 "类数组对象")类数组对象

*   如果一个对象的所有键名都是正整数或者 0, 且有 length 属性, 那么这个对象就是类数组对象
*   典型的类数组对象有 arguments 对象, 字符串, 以及大部分的 dom 元素集
*   数组的 slice 方法可以将类似数组的对象变成真正的数组 `var arr = Array.prototype.slice.call(arrayLike);`
*   除了将类数组对象转成真正的数组, 还可以使用 call() 将数组的方法直接放到类数组对象上使用 `Array.prototype.forEach.call(arrayLike, function(){});`

[](#数据类型转换 "数据类型转换")数据类型转换
--------------------------

### [](#自动数据类型转换 "自动数据类型转换")自动数据类型转换

#### [](#其他类型转字符串 "其他类型转字符串")其他类型转字符串

当`+`加号作为操作符, 且操作数中含有字符串时, 会自动将另一个操作数转为字符串;

规则如下:

1.  字符串 + 基础数据类型: 会直接将基础数据类型转为和字面量相同的字符串
2.  字符串 + 复合数据类型: 复合数据类型会先调用 `valueOf()`方法, 如果该方法返回基础数据类型则将其转为字符串, 如果返回的是复合数据类型, 则调用`toString()`方法, 如果返回的是基础数据类型则将其转为字符串, 如果不是则报错;

```
var tmp1 = "" + 3;
console.log("tmp1", tmp1); 

var tmp2 = "" + true;
console.log("tmp2", tmp2); 

var tmp3 = "" + undefined;
console.log("tmp3", tmp3); 

var tmp4 = "" + null;
console.log("tmp4", tmp4); 


var tmp5 = [1, 2, 3] + "";
console.log("tmp5", tmp5); 

var tmp6 = {} + "";
console.log("tmp6", tmp6); 


var o = {
    toString: function () {
        return 1;
    }
}

var tmp7 = o + "";
console.log("tmp7", tmp7) 


o.valueOf = function () {
    return 2;
}
var tmp8 = "" + o;
console.log("tmp8", tmp8); 



var a = {
    valueOf: function () {
        return {}
    },
    toString: function () {
        return "toString"
    }
};


console.log("" + a);
```

#### [](#其他类型转布尔值 "其他类型转布尔值")其他类型转布尔值

> 数值转布尔值

数值在逻辑判断条件下会自动转成布尔值; +-0 和 NaN 为 false, 其他数值都是 true;

```
if (0) {
    console.log("0 is true");
}else{
    console.log("0 is false");
}

if (NaN) {
    console.log("NaN is true");
}else{
    console.log("NaN is false");
}

if (-0) {
    console.log("-0 is true");
} else {
    console.log("-0 is false")
}
```

> 字符串转布尔值

空字符串”” 或者’’为 false, 其他都是 true

> undefined 和 null 转布尔值

undefined 和 null 转为布尔值都是 false

> 对象转布尔值

只有对象为 null 或者 undefined 时, 转为布尔值才是 false; 其他情况下 (包括空对象{} 和空数组 []) 转为布尔值都是 true;

```
var o = {};
if(o){
    console.log("{} is true");
}else{
    console.log("[] is true");
}
```

#### [](#其他类型转为数值 "其他类型转为数值")其他类型转为数值

一元操作符`+`和`-`都会触发其他类型转为数值;

数学运算符操作的两个操作数都不是字符串时, 也会触发其他类型转为数值的操作;

转化规则如下:

1.  字符串转数值: 空字符串转为 0, 非空字符串转为对应的数字; 不合法则返回 NaN;
2.  布尔值转数值: true 为 1, false 为 0;
3.  null 转为数值为 0
4.  undefined 转数值为 NaN
5.  对象转数值会先调用 `valueOf()`方法, 如果其返回值是基础数据类型, 则将其转为数值返回, 如果是复合数据类型, 则会调用`toString()`方法, 再将其转为数值, 如果不合法则返回 NaN;

### [](#强制类型转换 "强制类型转换")强制类型转换

*   Number(): 将其他类型转为数值; 将对象转为数值时, 会先调用对象的`valueOf()`方法, 不满足条件再调用`toString()`方法
*   String(): 将其他类型转为字符串; 将对象转为字符串时, 会先调用对象的`toString()`方法, 不满足条件时再调用对象的`valueOf()`方法;
*   Boolean(): 将其他类型转为布尔值

[](#异常处理 "异常处理")异常处理
--------------------

### [](#Error-对象 "Error 对象")Error 对象

Error 对象通常包含常用的三个属性, 且必须包含 message 属性;

1.  name: 异常名
2.  message: 异常提示信息
3.  stack: 异常调用栈信息

```
var e = new Error("自定义异常触发");
e.name = "自定义异常名称";
console.log(e.name);
console.log(e.message);
console.log(e.stack);
```

### [](#try-catch-finally "try catch finally")try catch finally

手动抛出异常并捕获

```
try {
    var e = new Error("message 自定义异常");
    e.name = "自定义异常名称";
    throw e;
} catch (e) {
    console.log("e.name", e.name, "e.message", e.message);
    console.log("e.stack", e.stack);
}
```

> throw

`throw`语句用来抛出用户定义的异常, 当前函数的执行将被停止, `throw`之后的代码不会被执行; 并且代码将会进入调用栈中第一个`catch`块中; 如果没有`catch`块, 程序将会终止;

```
try{
    console.log("before throw error");
    throw new Error("throw error");
    console.log("after throw error");
}catch(e){
    console.log("catch error", e.message);
}
```

> try/catch/finally

try 的三种声明形式:

1.  try…catch
2.  try…finally
3.  try…catch…finally

try/catch 主要用于捕获异常, try/catch 语句包含一个 try 块, 和至少一个 catch 块或者一个 finally 块;

*   try: try 块中放入可能会产生异常的语句或者函数
    
*   catch: catch 块中包含要执行的语句, 当 try 块中抛出异常时, catch 块会捕获这个异常, 并执行 catch 块中的代码; 如果 try 块中没有异常抛出, 则 catch 块会被跳过;
    
*   finally: finally 块在 try 块和 catch 块之后执行, 无论是否有异常产生, finally 块都会执行; 当 finally 块中有异常产生时, 会覆盖掉 try 块中的异常信息;
    
    ```
    try {
    
        try {
            throw new Error("error1");
        } finally {
            throw new Error("error2");
        }
    } catch (e) {
        console.log("e.message", e.message);
    }
    ```
    
*   如果 finally 块中返回一个值, 那么这个值将会作为整个 try/catch/finally 的返回值, 无论在 try 和 catch 有没有 return, 都是返回 finally 中的值;
    
    ```
    function test() {
        try {
            throw new Error("error1");
            return 1
        }catch (e) {
            throw new Error("error2");
            return 2;
        }finally {
            return 3;
        }
    }
    
    console.log("test()", test());
    ```
    

[](#对象详解和-hook "对象详解和 hook")对象详解和 hook
--------------------------------------

### [](#Object-静态方法和实例方法 "Object 静态方法和实例方法")Object 静态方法和实例方法

*   js 中所有其他对象都是继承自 Object ; 即所有对象都是 Object 的实例;
*   Object 的原生方法分为两类: Object 本身的方法和 Object 的实例方法
    1.  Object 本身的方法: 直接定义在 Object 上的方法, 相当于 java 的静态方法
    2.  Object 的实例方法: 定义在 Object 的原型对象 Object.prototype 上的方法; 相当于 java 中的动态方法, 可以被 Object 对象直接使用

#### [](#Object-的静态方法 "Object 的静态方法")Object 的静态方法

所谓静态方法, 就是设置在 Object 类上的方法;

*   Object.keys(): 遍历对象的属性, 返回可枚举的属性名; 这个方法只能查看对象自身的属性, 继承来的属性无法查看
*   Object.getOwnpeopertyNames(): 遍历对象的属性, 可以返回不可枚举的属性名;
*   Object.getOwnPropertyDescriptors(): 获取对象所有属性的描述对象;
*   Object.defineProperty(): 通过描述对象, 定义某个属性, 可以重写 get 和 set 方法来进行 hook
*   Object.defineProperties(): 通过描述对象, 定义多个属性, 可以重写 set 和 get 方法来进行 hook
*   Object.getPrototypeOf(): 返回参数对象的原型, 可以用于环境监测;
*   Object.setPrototypeOf(): 为参数对象设置原型, 返回该参数对象; 它接收两个参数, 第一个是现有对象, 第二个是原型对象;

```
function Test(a, b, c) {
    this.a = a;
    this.b = b;
    this.c = c;
};

var test = new Test(1, 2, 3);
Object.defineProperty(test,"d",{
    configurable: false, 
    enumerable: false,  
    value: "4",
    writable: true,
    
    
    
    
    
    
    
    
})



console.log(Object.keys(test)); 

console.log(Object.getOwnPropertyNames(test)); 

console.log(Object.getOwnPropertyDescriptors(test));














Object.setPrototypeOf(test, Test);

console.log(Object.getPrototypeOf(test));
```

#### [](#Object-的实例方法 "Object 的实例方法")Object 的实例方法

*   Object.prototype.valueOf(): 返回当前对象对应的值
*   Object.prototype.toString(): 返回当前对象对应的字符串形式
    *   因为实例对象可能自定义了 toString 方法, 覆盖了 Object.prototype.toString 方法; 所以为了得到类型字符串, 最好使用 Object.prototype.toString 方法, 通过 call 的方式可以在任意对象上调用这个方法, 帮助判断;
*   Object.prototype.hasOwnProperty(): 判断某个属性是否是当前对象自身的属性, 可以区分某个属性是对象自身的还是继承来的;
*   Object.prototype.isPrototypeOf(): 判断当前对象是否是另一个对象的原型;
*   Object.prototype.propertyIsEnumerable(): 判断某个属性是否是可枚举的

### [](#构造函数 "构造函数")构造函数

*   典型的面向对象的变成语言都有类的概念, 所谓类就是对象的模板, 对象就是类的实例;
*   js 不是基于类的, 而是基于构造函数和原型链;
*   构造函数就是一个简单的函数, 当函数和 new 关键字配合使用, 就可以返回一个实例对象;

```
var init = function (a, b, c) {
    this.a = a;
    this.b = b;
    this.c = c;

};

var initObj = new init(1, 2, 3);
console.log("initObj", initObj);
```

### [](#创建对象的方式 "创建对象的方式")创建对象的方式

1.  直接赋值, `var obj = {};`或者`var obj=Object({});`(强制转为对象)
2.  new 关键字创建: `var obj = new Object();`
3.  Object.create(arg, property): 以传入的对象的构造函数作为模板, 可以生成实例对象; 一般用于在拿不到构造函数时, 直接通过现有的实例作为参数, 来构造新的实例;

这三种创建对象方式的区别:

1.  字面量和 new 关键字创建的对象是 Object 的实例, 原型指向 Object.prototype; 继承内置对象 Object
2.  Object.create(arg, property) 创建的对象的原型取决入参 arg, arg 为 null 时, 创建的是空对象; arg 为指定对象时, 则新对象的原型指向指定对象, 继承指定对象;

```
var init = function (a, b, c) {
    this.a = a;
    this.b = b;
    this.c = c;
};


var obj = new init(1, 2, 3);

var obj2 = Object.create(obj,{
    "p": {
        "value": "p1"
    }
})

console.log("obj2", obj2); 
console.log("obj2.p", obj2.p);
```

### [](#原型对象-Object-prototype "原型对象 Object.prototype")原型对象 Object.prototype

*   js 的继承机制的设计思想是: 原型对象上的所有属性和方法, 都能被实例对象共享; 也就是说, 如果属性和方法定义在原型对象上, 那么所有的实例都可以共享这些属性和方法;
*   js 规定, 每个对象都有一个 `__proto__`指向原型对象
*   js 规定, 每个函数除了有`__proto__`之外, 还有一个`prototype`属性; `__proto__`指向函数原型 Function.prototype, `prototype`指向一个原型对象;
*   原型对象上定义的属性和方法不是实例对象自身的, 所以修改原型对象上的属性和方法, 这种变动会影响到所有实例属性;
*   当实例本身没有某个属性或者方法的时候, 它会到原型对象上寻找该属性和方法, 类似 java 的双亲委派机制;

> `prototype` 和 `__proto__` 区别

![](https://kevinspider-1258012111.cos.ap-shanghai.myqcloud.com/2021-08-20-072915.jpg)

```
function test(){
    console.log("test");
}

console.log(test.prototype.__proto__.__proto__); 
console.log(test.__proto__.__proto__.__proto__);
```

```
function DTA(boss, employee) {
    this.boss = boss;
    this.employee = employee;
}

DTA.prototype.room = {
    "roomNum": 1
}
DTA.prototype.name = "kevin";

DTA.prototype.dowork = function () {
    return this.boss +" " + this.employee +" "+ "do work now";
}

var dta = new DTA("kevin", "rub");
var dta2 = new DTA("kevin2", "rub2");


DTA.prototype.name = "change";
console.log(dta2.__proto__ === DTA.prototype); 
console.log(dta2.name); 

dta2.room.roomNum = 2; 
dta2.__proto__.name = "change2";
console.log(dta.name); 
console.log("dta.boss", dta.boss);
console.log("dta.employee", dta.employee);
console.log("dta.dowork()", dta.dowork());
console.log("dta.room.roomNum", dta.room.roomNum);
```

### [](#原型链 "原型链")原型链

*   js 规定, 所有对象都有自己的原型对象 prototype;
*   任何一个对象都可以充当其他对象的原型;
*   由于原型也是一个对象, 所以它也有自己的原型对象, 最终形成一个原型链, 对象 –> 原型 –> 另一个原型…

### [](#constructor-NaN "constructor")constructor

*   原型对象 property 有一个 constructor 属性, 这个属性指向 prototype 对象所在的构造函数
*   constructor 属性的作用是: 可以得知某一个实例对象, 到底是由哪一个构造函数产生的
*   有了 constructor 属性, 就可以找到构造函数, 可以通过一个实例对象来新建另一个实例
*   constructor 还可以用来设置一些环境监测:
    1.  `document.constructor === HTMLDocument`
    2.  `navigator.constructor === Navigator`
    3.  经常和`toString()`配合进行检测, 这样在 node 中运行不会报错, 更难发现; `navigator.constructor.toString() === "function Navigator() { [native code] }"`

```
function test() {};
console.log(test.prototype.constructor === test);


function HTMLDocument() {};
var document = new HTMLDocument();
console.log(document.constructor === HTMLDocument);

var document2 = new document.__proto__.constructor();
console.log(document2.constructor === HTMLDocument);


function Navigator() {};
var navigator = new Navigator();
console.log(navigator.constructor === Navigator);
var navigator2 = new navigator.constructor();
console.log(navigator2.constructor === Navigator); 



function Navigator() {}
Navigator.toString = function () { 
    return "function Navigator() { [native code] }"
}

var navigator = new Navigator();
console.log(navigator.constructor.toString());
console.log(navigator.constructor.toString() === "function Navigator() { [native code] }");
```

### [](#hook-cookie "hook cookie")hook cookie

简易版 hook cookie

```
var cookie = document.cookie;
Object.defineProperty(document, 'cookie', {
    get: function(){
        console.log("getter:" + cookie);
        return cookie;
    },
    set: function(value){
        console.log("setter:" + cookie);
        if (value.indexOf("targetCookie") > -1){
            debugger;
        }
        cookie = value;
    }
})
```

油猴 hook cookie

```
(function () {
    'use strict';
    console.log("hook cookies start ...");
    var cookie_cache = document.cookie;
    Object.defineProperty(document, 'cookie', {
        get: function () {
            console.log("get cookie:" + cookie_cache);
            return cookie_cache;
        },
        set: function (val) {
            console.log('Setting cookie', val);
            
            if (val.indexOf('w_token') != -1) {
                debugger;
            }
            var cookie = val.split(";")[0];
            var ncookie = cookie.split("=");
            var flag = false;
            var cache = cookie_cache.split("; ");
            cache = cache.map(function (a) {
                if (a.split("=")[0] === ncookie[0]) {
                    flag = true;
                    return cookie;
                }
                return a;
            })
            cookie_cache = cache.join("; ");
            if (!flag) {
                cookie_cache += cookie + "; ";
            }
            return cookie_cache;
        }
    });
})();
```

### [](#object-copy "object copy")object copy

对象拷贝需要做到两件事:

1.  确保拷贝后的对象, 与原对象具有相同的原型
2.  确保拷贝后的对象, 与原对象具有同样的实例属性

```
function copyObj(obj){
    return Object.create(
    	Object.getPrototypeOf(obj),
        Object.getOwnPropertyDescriptors(obj)
    )
}

function kevin(a){
    this.a = a;
}

var kevinObj = new kevin(1);
var kevinObj2 = copyObj(kevinObj);
```

### [](#Array-对象 "Array 对象")Array 对象

*   isArray(): 返回一个布尔值, 表示参数是否是一个数组, 可以弥补 typeof 运算符的不足
*   push(), pop()
    *   push: 数组的末端添加一个或者多个元素, 并返回添加新元素后的数组长度
    *   pop: 删除数组的最后一个元素, 并返回该元素
*   shift(), unshift()
    *   shift: 用于删除数组的第一个元素, 并返回该元素
    *   unshift: 用于在数组的第一个位置添加元素, 并返回添加新元素后数组长度
*   join(): 以指定的参数作为分隔符, 将数组中所有成员连接成一个字符串并返回, 如果不指定分隔符, 则默认使用逗号分隔
*   concat(): 用于多个数组合并, 将新数组的成员添加到原数组成员的后面, 然后返回一个新数组, 原数组保持不变
*   reverse(): 用于颠倒排列数组元素, 返回改变后的数组, 该方法将会改变原数组
*   slice(): 提取目标数组的一部分, 返回一个新数组, 原数组保持不变
*   splice(): 删除原数组的一部分, 并可以在删除的位置添加新的数组成员, 返回值是被删除的元素, 该方法会改变原数组;
*   sort(): 对数组成员进行排序, 默认是按照字典顺序排序, 排序后原数组将改变
*   map(): 将数组的所有成员依次传入回调函数中, 然后把回调函数每一次执行结果组成一个新数组返回
*   forEach(): 与 map() 类似, 无返回值, 只是操作原数组
*   filter(): 用于过滤数组成员, 满足条件的成员组成一个新数组返回;
*   indexOf(): 返回给定元素在数组中第一次出现的位置, 如果没有出现则返回 - 1;

### [](#包装对象 "包装对象")包装对象

三种原始类型的值在一定条件下也会自动转为对象, Boolean(), String() 和 Number();

当使用 new 关键字的时候就会创建一个新的包装对象, 当不使用 new 关键字时就是强制类型转换;

### [](#Math "Math")Math

*   Math.abs(): 返回入参的绝对值
*   Math.ceil(): 返回入参向上取整之后的值
*   Math.floor(): 返回入参向下取整之后的值
*   Math.max(): 返回数组中的最大值
*   Math.min(): 返回数组中的最小值
*   Math.random(): 返回 0-1 的伪随机值
*   Math.round(): 返回入参四舍五入后的整数

### [](#Date "Date")Date

创建一个新的 Date 对象的唯一方式是通过 new 操作符;

```
new Date();
new Date(value);
new Date(dateString);
new Date(year, monthIndex [, day [, hours [, minutes [, seconds [, milliseconds]]]]]);
```

*   Date.now(): 返回当前时间戳
*   Date.parse(): 解析时间字符串, 返回时间戳

### [](#控制台-API "控制台 API")控制台 API

*   inspect(obj): 打开相关面板并选中相应的元素, 展示它的细节
*   getEventListeners(): 返回一个对象, 该对象的键为事件, 值为数组; 数组的成员为该事件的回调函数;
*   keys(), values(): 返回一个数组, 包含 object 的所有键名和值
*   monitorEvents(object,[,events]): 监听特定对象上发生的特定事件; 事件发生时, 会返回一个 Event 对象; 包含该事件的相关信息; unmonitorEvents 方法用来停止监听;
    *   monitorEvents: 允许监听同一大类的事件, 所有事件可以分为四大类:
        1.  mouse: mousedown, mouseup, click, dblclick, mousemove, mouseover, mouseout, mousewheel
        2.  key: keydown, keyup, keypress, textInput
        3.  touch: touchstart, touchmove, touchend, touchcancel
        4.  control: resize, scroll, zoom, focus, blur, select, change, submit, reset
*   copy(obj): 复制特定对象到剪贴板
*   debugger: 断点

### [](#this-关键字 "this 关键字")this 关键字

*   this 就是属性或者方法当前所在的对象
*   this 主要使用场景:
    1.  全局环境使用 this, 它指的就是顶层对象 window
    2.  构造函数中使用 this, 它指的就是实例对象
    3.  如果对象的方法包含 this, this 的指向就是方法运行时所在的对象; 该方法赋值给另一个对象, 就会改变 this 的指向;

#### [](#绑定-this-的方法 "绑定 this 的方法")绑定 this 的方法

1.  Function.prototype.call(thisValue, arg1, arg2 …) : 函数实例的 call 方法, 可以指定函数内部的 this 的指向 (即函数执行时所在的作用域); 然后在指定作用域中调用该函数
2.  Function.prototype.apply(thisValue, [arg1, arg2. ..]): apply 和 call 方法类似, 都是可以改变 this 的指向; 唯一区别就是它接受一个数组作为函数执行时的参数;
3.  Function.prototype.bind(thisValue, arg1, arg2. ..): 将函数体内的 this 绑定到某个对象, 然后返回一个新函数;

```
var d = new Date();
console.log(d.getTime());

var getTime = d.getTime;
getTime(); 
var d_getTime = getTime.bind(d, []);
console.log(d_getTime());


var d = new Date();
var getTime = d.getTime;
var result = getTime.apply(d, []);
console.log(result);

var d = new Date();
var getTime = d.getTime;
var result = getTime.call(d);
console.log(result);
```

[](#es6-部分新特性 "es6 部分新特性")es6 部分新特性
-----------------------------------

es6 是 js 语言的下一代标准, 在 2015 年 6 月正式发布; 它的目标是让 js 语言可以用来编写复杂的大型应用程序; es 和 js 的关系是: es 是 js 的规格, js 是 es 的实现;

### [](#块级作用域 "块级作用域")块级作用域

es6 中提出了两个新的变量声明命令: let 和 const; let 可以完全取代 var; var 命令存在变量提升的效果, let 命令则不存在这个问题; 所以尽量使用 let, 减少 var 的使用;

在全局环境中使用 var 声明变量, 会直接将变量定义到全局环境中, 在全局环境中使用 let 和 const 是不会直接将变量定义到全局环境上的;

```
var a = 10;
let b = 20;
const c = 30;

console.log(window.a); 
console.log(window.b); 
console.log(window.c);
```

在 es5 中, 只有函数作用域和全局作用域; 在 es6 中新增了块级作用域; 比如典型的 for 循环; 使用 var 会导致变量泄漏到全局环境中, 使用 let 就不会存在该问题;

```
for (var i = 0; i < 10; i++){
    console.log(i);
}
console.log(i);


for (let a = 0; a < 10; a++){
    console.log(a);
}
console.log(a);
```

### [](#属性简洁表达 "属性简洁表达")属性简洁表达

```
const company = "dta"; 
const team = {company};  
console.log(team);


let boss = "Dta boss";

let Dta = {
    [boss]: "roysue"
}

console.log("Dta['Dta boss']", Dta['Dta boss']);
```

### [](#对象方法简洁表达 "对象方法简洁表达")对象方法简洁表达

```
var team = {
    dowork: function(){
        return "work!";
    }
}


var team = {
    dowork(){
        return "work";
    }
}
```

### [](#字符串模板 "字符串模板")字符串模板

在 es6 中新增了动态的字符串模板, 使用反引号表示, 使用 ${} 进行填充;

```
var name = "kevin";
console.log(`hi, ${name}`);
```

### [](#解构赋值 "解构赋值")解构赋值

> 数组解构

```
const arr = [1,2,3,4];
const [first, second] = arr; 
console.log(first, second);
```

> 对象解构

```
var obj = {
    firstName: "chen",
    lastName: "guilin"
}

function getFullName(obj){
    const {firstName, lastName} = obj;
    console.log(firstName, lastName); 
}

getFullName(obj);
```

> 快速复制对象

```
let Dta = {
    name: "kevin",
    age: 29,
    dowork(){
        return "work";
    }
}

let newDta = {...Dta}; 
console.log("newDta", newDta);


let roysue = {};
let r1ysue = Object.assign({}, roysue);

let r2ysue = {...roysue};



var obj = {};
const clone1 = {
    __proto__: Object.getPrototypeOf(obj),
    ...obj
};

const clone2 = Object.assign(
    Object.create(Object.getPrototypeOf(obj),
    obj
);

const clone3 = Object.create(
    Object.getPrototypeOf(obj),
    Object.getOwnPropertyDescriptors(obj)
)
```

> 数组创建对象

```
function processInput(){
    let [a,b,c,d] = [1,2,3,4];
    return {a,b,c,d}
}

console.log(processInput());
```

### [](#箭头函数 "箭头函数")箭头函数

立即执行函数可以写成箭头函数的形式:

```
(() => {
    console.log("welcome to the DTA");
})();
```

在那些使用匿名函数当做参数的场景下, 尽量使用箭头函数作为替代, 因为使用箭头函数更加简洁, 而且箭头函数在定义函数的时候就绑定了 this; 不需要在执行的时候考虑 this 的问题;

```
let a = [1, 2, 3].map((x) => {
    return x * x;
});

console.log("a", a); 

function Test(){
    this.s1 = 0;
    this.s2 = 0;
    
    setInterval(()=>{this.s1++;}, 1000);
    setInterval(function () {
        this.s2++;
     }, 1000);
}

var test = new Test();
setInterval(() => {
    console.log(test.s1);
}, 3000); 

setInterval(() => {
    console.log(test.s2);
}, 3000);
```

### [](#模块的引入 "模块的引入")模块的引入

```
let {stat, exists, readfile} = require("fs");


let _fs = require("fs");
let stat = _fs.stat;
let exists = _fs.exists;
let readfile = _fs.readfile;


import {stat, exists, readfile} from 'fs';
```

在 js 中引入其他文件中的函数或者对象

```
var dta = {
    name: "dta",
    version : '1.0.0',
    boss: "roysue",
    dowork: function (name) {
        console.log(`${this.name} is working`);
    }
}
module.exports = dta; 


let dta = require("./testImport");
console.log(dta);
dta.dowork();


function add(x,y){
    return x+y;
}
exports.add = add;


let {add} = require("./testImport");
var result = add(1,2);
console.log("result",result);
```

### [](#链式判断运算符 "链式判断运算符")链式判断运算符

如果读取对象内部的某个属性, 往往需要判断一下该对象是否存在这属性; 比如 `message.body.user.firstName`, 安全的写法为:

```
const firstName = (message && message.body && message.body.user && message.body.user.firstName) || 'default';
```

这样写相对比较麻烦, 在 es6 中可以使用链式判断运算符进行判断;

```
let Dta = {
    name: "kevin",
    age: 29,
    dowork(){
        return "work";
    },
    room: {
        r1: {
            l: "l1"
        }
    }
}

console.log(Dta.room.r1.l); 
console.log(Dta?.room?.r1?.l?.f || "default");
```

### [](#null-判断运算符 "null 判断运算符")null 判断运算符

在读取对象属性的时候, 如果该属性是 null 或者 undefined, 可以通过 || 设置默认值`const roysue = Dta.Boss.roysue || 'roysue'` 但是这样写的话在, 当`Dta.Boss.roysue`的值为空字符串或者 false 或者 0 的时候也会触发默认值;

为了避免这个错误, es6 中新增了 null 判断运算符 ??; 只有当运算符的左侧值为 null 或者 undefined 时, 才会返回右侧的默认值

```
const roysue = Dta.Boss.roysue ?? 'roysue';
```

### [](#对象的新增方法 "对象的新增方法")对象的新增方法

*   Object.is(a,b) 用来比较两个值是否是严格相等; 与严格比较符 === 基本一致;
*   Object.assign(target, source1, source2): 用于对象的合并, 将原对象 source 的所有可枚举属性, 全部复制到 target 对象中;
    *   注意点:
        *   浅拷贝: 如果原对象某个属性是对象, 那么目标对象拷贝得到的属性时这个对象属性的引用;
        *   同名属性替换: 一旦遇到同名的属性, Object.assign() 处理的方法是替换, 而不是添加
    *   常见用途:
        *   为对象添加属性
        *   为对象添加方法
        *   克隆对象: 只能克隆对象自身的值, 不能克隆它继承的值
        *   合并多个对象
*   Object.getOwnPropertyDescriptors(): 返回指定对象的所有自身属性的描述对象 (非继承来的属性)
*   `__proto__`: 该属性可以用来读取和设置当前对象的原型对象 (prototype);

### [](#类的表示 "类的表示")类的表示

在 es5 中, 类的表示通常是:

```
function Dta(boss, employee) {
    this.boss = boss;
    this.employee = employee;
}
Dta.prototype.toString = function () {
    return '(' + this.boss + ',' + this.employee + ')';
}

var dta = new Dta('kevin', 'rub');
console.log(dta.toString());
```

在 es6 中, 为了让类的概念更接近 java 和 c++ 的类, 新增了 class 关键字;

```
class Dta{
    constructor(boss, employee) {
        this.boss = boss;
        this.employee = employee;
    }
    toString(){
        return '(' + this.boss + ',' + this.employee + ')';
    }
}

var dta = new Dta("kevin", "rub");
console.log(dta.toString());
```

[](#Proxy-和-Reflect "Proxy 和 Reflect")Proxy 和 Reflect
-----------------------------------------------------

*   Proxy 可以理解成, 在目标对象之前架设一层拦截, 外界对该对象的访问, 都必须先通过这一层拦截; 因此提供了一种机制, 可以对外界的访问进行过滤和改写; Proxy 这个词的原意是代理, 在这里可以理解成 Proxy 为一个代理器;
    *   Proxy 支持的 13 中拦截操作
        *   get(target, propertyKey, receiver): 拦截对象属性的读取, 比如: proxy.foo 或者 proxy[‘foo’];
        *   set(target, propertyKey, propertyValue, receiver): 拦截对象属性的设置, 比如 proxy.foo=1 或者 proxy[‘foo’]=1; 返回布尔值
        *   has(target, propertyKey): 拦截 propertyKey in proxy 的操作, 返回布尔值;
        *   deleteProperty(target, propertyKey): 拦截 delete proxy[propertyKey] 的操作, 返回布尔值;
        *   ownKeys(target): 拦截 Object.getOwnPropertyNames(), Obejct.getOwnPropertySymbols(),Obejct.keys(),for..in 循环; 返回数组, 该方法返回目标对象所有自身的属性的属性名; 而 Object.keys() 仅返回目标对象自身可遍历的属性名;
        *   getOwnPropertyDescriptor(target, propertyKey): 拦截 Object.getOwnPropertyDescriptor(); 返回属性的描述对象
        *   defineProperty(target, propertyKey, propertyDesc): 拦截 Object.defineProperty() 和 Object.defineProperties(), 返回布尔值;
        *   preventExtensions(target): 拦截 Object.preventExtensions() 返回布尔值
        *   getPrototypeOf(target): 拦截 Object.getPrototypeOf(), 返回一个对象;
        *   isExtensible(target): 拦截 Object.isExtensible(), 返回布尔值;
        *   setPrototypeOf(target, proto): 拦截 Object.setPrototypeOf(), 返回布尔值;
        *   apply(target, object, args): 拦截 proxy 实例作为函数调用的操作; 比如 proxy(..args), proxy.call(obj, args), proxy.apply(…) ;
        *   construct: 拦截 proxy 实例作为构造函数调用的操作, 比如 new proxy(…args);
*   Reflect 对象和 Proxy 对象一样, 是 es6 为了操作对象而提供的新 API; 大部分与 Object 对象的同名方法的作用都是相同的; 而且 Reflect 与 Proxy 对象的方法是一一对应的;
    *   Reflect 设计的目的:
        1.  将 Object 对象的一些明显属于语言内部的方法 (Object.defineProperty) 放在 Reflect 对象上; 现阶段某些方法同时在 Object 和 Reflect 对象上部署, 未来新的方法只能部署在 Reflect 对象上; 也就是说 Reflect 可以拿到语言内部的方法
        2.  修改某些 Object 方法的返回结果, 让其变得更合理
        3.  让 Object 操作都变成函数行为
        4.  Reflect 对象的方法和 Proxy 对象的方法一一对应;
    *   Reflect 静态方法
        *   Reflect.get(target, propertyName, receiver): 查找并返回 target 对象的 name 属性, receiver 绑定 this;
        *   Reflect.set(target, propertyName, propertyValue, receiver): 设置 target 对象的 propertyName 属性等于 propertyValue
        *   Reflect.has(obj, propertyName): 判断 propertyName 是否存在在对象中, 相当于 in
        *   Reflect.deleteProperty(obj, propertyName): 方法等同于 delete obj[propertyName]; 用于删除对象的属性
        *   Reflect.construct(target, args): 等同于 new target(args); 调用构造函数的方法
        *   Reflect.getPrototypeOf(obj): 读取对象的`__proto__`属性; 对应 Object.getPrototypeOf()
        *   Reflect.setPrototypeOf(obj, newProto): 设置目标对象的原型, 对应 Object.setPrototypeOf();
        *   Reflect.apply(func, thisArg, args): 等同于 Function.prototype.apply.call(func, thisArg, args)
        *   Reflect.defineProperty(target, propertyKey, attributes): 等同于 Obejct.defineProperty();
        *   Reflect.getOwnPropertyDescriptor(target, propertyKey): 等同于 Object.getOwnPropertyDescriptor();
        *   Reflect.isExtensible(target): 对应 Object.isExtensible() 表示当前对象是否可扩展;
        *   Reflect.preventExtensions(target): 对用 Object.preventExtensions() 让一个对象变为不可扩展
        *   Reflect.ownKeys(target): 返回对象的所有属性, 可以返回 Symbol 对象;

> 代理器案例

```
let obj = new Proxy({}, {
    get(target, p, receiver) {
        console.log(`get ${p}`);
        return Reflect.get(target, p, receiver);
    },
    set(target, p, value, receiver) {
        console.log(`set key: ${p}, value: ${value}`);
        return Reflect.set(target, p, value, receiver);
    }
});

obj.name = "kevin";
obj.name;
```

> Proxy 作为其他对象的原型

Proxy 实例也可以作为其他对象的原型对象, 可以用来拦截原型链的访问:

```
var proxy = new Proxy({}, {
    get: function(target, propertyKey, receiver){
        return 35;
    }
})

var obj = Object.create(proxy);
console.log(obj.time);
```

> Proxy 拦截函数调用

```
function test(a, b) {
    return a + b;
}

test = new Proxy(test, {
    apply(target, thisArg, argArray) {
        console.log("thisArg", thisArg);
        console.log("target", target);
        console.log("argArray", argArray);
        let result = Reflect.apply(target, thisArg, argArray);
        console.log("result", result);
        return result;
    }
})

test(1, 2);
```

> Proxy 监控构造函数的调用

```
function Test(a,b) {
    this.a = a;
    this.b = b;
    return a + b;
}

Test = new Proxy(Test, {
    construct(target, argArray, newTarget) {
        console.log("target", target);
        console.log("argArray", argArray);
        console.log("newTarget", newTarget);
        var result =  Reflect.construct(target, argArray, newTarget);
        console.log("result", JSON.stringify(result));
        return result

    }
})


let test = new Test(1, 2);
console.log("test", test);
```

[](#window-和-navigator-Proxy-实战 "window 和 navigator Proxy 实战")window 和 navigator Proxy 实战
-----------------------------------------------------------------------------------------

采用 Proxy 帮助补充环境

```
let mywindow = {};
let mynavigator = {};

let rawstringify = JSON.stringify;
JSON.stringify = function (Object){
    if((Object?.value ?? Object) === global){
        return "global"
    }else{
        return rawstringify(Object)
    }
}
function checkproxy(){
    
    window.a = {
        "b":{
            "c":{
                "d":123
            }
        }
    }
    window.a.b.c.d = 456
    window.a.b
    window.btoa("123")
    window.atob.name
    "c" in window.a
    delete window.a.b
    Object.defineProperty(window, "b",{
        value:"bbb"
    })
    Object.getOwnPropertyDescriptor(window,"b")
    Object.getPrototypeOf(window)
    Object.setPrototypeOf(window,{"dta":"dta"})
    
    
    
    Object.preventExtensions(window)
    Object.isExtensible(window)
}
function getMethodHandler(WatchName){
    let methodhandler = {
      apply(target, thisArg, argArray) {
        let result = Reflect.apply(target, thisArg, argArray)
        console.log(`[${WatchName}] apply function name is [${target.name}], argArray is [${argArray}], result is [${result}].`)
        return result
      },
      construct(target, argArray, newTarget) {
        var result = Reflect.construct(target, argArray, newTarget)
        console.log(`[${WatchName}] construct function name is [${target.name}], argArray is [${argArray}], result is [${JSON.stringify(result)}].`)
        return result;
      }
    }
    return methodhandler
}
function getObjhandler(WatchName){
    let handler = {
        get(target, propKey, receiver) {
            let result = Reflect.get(target, propKey, receiver)
            if (result instanceof Object){
                if (typeof result === "function"){
                    console.log(`[${WatchName}] getting propKey is [${propKey}] , it is function`)
                    
                }else{
                    console.log(`[${WatchName}] getting propKey is [${propKey}], result is [${JSON.stringify(result)}]`);
                }
                return new Proxy(result,getObjhandler(`${WatchName}.${propKey}`))
            }
            console.log(`[${WatchName}] getting propKey is [${propKey}], result is [${result}]`);
            return result;
        },
        set(target, propKey, value, receiver) {
            if(value instanceof Object){
                console.log(`[${WatchName}] setting propKey is [${propKey}], value is [${JSON.stringify(value)}]`);
            }else{
                console.log(`[${WatchName}] setting propKey is [${propKey}], value is [${value}]`);
            }
            return Reflect.set(target, propKey, value, receiver);
        },
        has(target, propKey){
            var result = Reflect.has(target, propKey);
            console.log(`[${WatchName}] has propKey [${propKey}], result is [${result}]`)
            return result;
        },
        deleteProperty(target, propKey){
            var result = Reflect.deleteProperty(target, propKey);
            console.log(`[${WatchName}] delete propKey [${propKey}], result is [${result}]`)
            return result;
        },
        getOwnPropertyDescriptor(target, propKey){
            var result = Reflect.getOwnPropertyDescriptor(target, propKey);
            console.log(`[${WatchName}] getOwnPropertyDescriptor  propKey [${propKey}] result is [${JSON.stringify(result)}]`)
            return result;
        },
        defineProperty(target, propKey, attributes){
            var result = Reflect.defineProperty(target, propKey, attributes);
            console.log(`[${WatchName}] defineProperty propKey [${propKey}] attributes is [${JSON.stringify(attributes)}], result is [${result}]`)
            return result
        },
        getPrototypeOf(target){
            var result = Reflect.getPrototypeOf(target)
            console.log(`[${WatchName}] getPrototypeOf result is [${JSON.stringify(result)}]`)
            return result;
        },
        setPrototypeOf(target, proto){
            console.log(`[${WatchName}] setPrototypeOf proto is [${JSON.stringify(proto)}]`)
            return Reflect.setPrototypeOf(target, proto);
        },
        preventExtensions(target){
            console.log(`[${WatchName}] preventExtensions`)
            return Reflect.preventExtensions(target);
        },
        isExtensible(target){
            var result = Reflect.isExtensible(target)
            console.log(`[${WatchName}] isExtensible, result is [${result}]`)
            return result;
        },
        ownKeys(target){
            var result = Reflect.ownKeys(target)
            console.log(`[${WatchName}] invoke ownkeys, result is [${JSON.stringify(result)}]`)
            return result
        },
        apply(target, thisArg, argArray) {
            let result = Reflect.apply(target, thisArg, argArray)
            console.log(`[${WatchName}] apply function name is [${target.name}], argArray is [${argArray}], result is [${result}].`)
            return result
          },
        construct(target, argArray, newTarget) {
            var result = Reflect.construct(target, argArray, newTarget)
            console.log(`[${WatchName}] construct function name is [${target.name}], argArray is [${argArray}], result is [${JSON.stringify(result)}].`)
            return result;
        }
    }
    return handler;
}
const window = new Proxy(Object.assign(global,mywindow), getObjhandler("window"));
const navigator = new Proxy(Object.create(mynavigator), getObjhandler("navigator"));

checkproxy()

module.exports = {
    window,
    navigator
}
```

[](#异步操作与-Ajax "异步操作与 Ajax")异步操作与 Ajax
--------------------------------------

### [](#单线程模型 "单线程模型")单线程模型

*   单线程模型指的是: js 只在一个线程上运行, 也就是说, js 同时只能执行一个任务, 其他任务都必须在后面排队等待
*   js 只在一个线程上运行, 不代表 js 引擎只有一个线程, 实际上 js 引擎有多个线程, 单个脚本只能在一个线程上运行 (主线程); 其他线程都在后台配合主线程;

### [](#同步任务和异步任务 "同步任务和异步任务")同步任务和异步任务

*   同步任务是那些没有被引擎挂起, 在主线程上排队执行的任务; 只有前一个任务执行完毕, 才能执行后一个任务;
*   异步任务是那些被引擎放在一边, 不进入主线程, 而是会进入任务队列的任务; 只有引擎认为某些异步任务可以执行了 (比如 Ajax 操作从服务器得到了响应), 该任务 (采用回调函数的形式) 才会进入到主线程进行执行; 排在异步任务后面的代码不用等待异步任务结束就可以马上开始运行, 所以说异步任务不会阻塞其他任务代码;

Ajax 操作可以被当做同步任务处理, 也可以当做异步任务处理, 由参数 async 控制; 如果是同步任务, 主线程就会一直等待 Ajax 操作返回结果, 才会继续执行后面的代码; 如果是异步任务, 那么主线程发出 Ajax 请求后会直接向下执行其他代码, 等到 Ajax 操作有了结果, 引擎会其对应的回调函数放入主线程进行处理;

### [](#任务队列和事件循环 "任务队列和事件循环")任务队列和事件循环

*   js 运行时, 除了正在运行的主线程, 引擎还提供了一个任务队列 (task queue); 任务队列中是各种需要当前程序处理的异步任务;
*   首先, 主线程会执行所有的同步任务, 等到同步任务全部执行完成, 主线程才会去任务队列中寻找异步任务;
*   如果异步任务满足执行条件, 则异步任务重新进入到主线程中执行, 此时异步任务就变成了同步任务; 当这个异步任务执行完成, 下一个异步任务再进入主线程进行执行, 一旦任务队列被清空, 程序就结束运行;
*   事件循环指的是只要同步任务执行完成, 引擎就会不断去检查那些挂起的异步任务是不是满足进入主线程的条件; 这种循环机制就叫做事件循环, 是一种程序结构, 用于等待和发送消息和事件;

### [](#异步操作模式 "异步操作模式")异步操作模式

1.  回调函数
    *   回调函数是异步操作最简单的方法
    *   回调函数的优点是简单, 容易理解和实现; 缺点是不利于代码的阅读和维护; 多个部分之间高度耦合; 在多个回调函数相互嵌套的情况下, 会导致程序结构混乱, 难以阅读和追踪;
2.  事件监听
    *   采用事件驱动模式, 异步任务的执行不取决于代码的顺序, 而是取决于某个事件是否发生;
    *   优点是比较容易理解, 可以绑定多个事件, 每个事件可以指定多个回调函数; 可以解耦, 实现模块化; 缺点是整个程序都要变成事件驱动型, 运行流程不清晰, 阅读难以看出主流程;
3.  发布 / 订阅
    *   这种方式的性质和事件监听类似; 但是可以通过查看消息, 了解存在多少信号, 每个信号有多少个订阅者, 从而监控程序的运行;

### [](#定时器 "定时器")定时器

定时器是 js 中提供定时执行代码的功能; 主要由`setTimeout()`和`setInterval()`这两个函数来完成; 它们可以向任务队列中添加定时任务;

> 运行机制

*   setTimeout 和 setInterval 的运行机制, 是将指定的代码移出本轮事件循环, 等到下一轮事件循环, 再检测是否满足指定时间, 如果满足就执行对应的代码, 如果不满足则继续等待;
*   setTimeout 和 setInterval 指定的回调函数, 必须等到本轮事件循环的所有同步任务都执行完, 才会开始执行; 由于前面的任务到底需要多少时间执行是不确定的, 所以没有办法保证 setTimeout 和 setInterval 指定的任务一定会按照预定时间执行;
*   setTimeout(f,0); 也不会立即执行, f 方法会被移出本轮事件循环;

> setTimeout

*   `function setTimeout(callback, ms, ...args)`
*   setTimeout 函数用于指定某个函数或者某段代码在多少毫秒之后执行; 它返回一个整数, 该整数表示定时器的编号, 之后可以通过这个编号取消该定时器;
*   需要注意: node 中如果回调函数是对象的方法, 那么 setTimeout 会使方法内部的 this 关键字指向全局环境, 而不是定义这个方法时的那个对象;
    *   为了防止该问题, 可以将对象的方法放在一个函数内
    *   使用 bind 方法, 将对象的方法 bind 到对象上

```
var x = "from window";

var obj = {
    x: "from obj",
    y: function () {
        console.log(this.x);
    }
}

setTimeout(obj.y, 1000); 

setTimeout(function () {
    obj.y(); 
}, 1000);

setTimeout(obj.y.bind(obj), 1000);
```

setTimeout 也可以实现指定间隔时间执行, 相当于下面的 setInterval 方法; 区别在于 setTimeout 会考虑自身的运行时间, 可以保证严格的间隔时间;

```
var i = 1;

function f() {
    console.log(i);
    i++;
    setTimeout(f, 1000);
}

setTimeout(f, 1000);
```

> setInterval

*   setInterval 函数的用法和 setTimeout 完全一致, 区别仅仅是在以 setInterval 指定某个任务每隔一段时间就执行一次, 也就是无限次的定时执行;
*   setInterval 指定的是开始执行的间隔时间, 并不考虑每次任务执行本身所消耗的时间, 所以两次执行之间的间隔会小于指定的时间;

```
var i = "from global";
var obj = {
    a : 1,
    i: "from obj",
    f: function () {
        console.log("this.i", this.i);
    }
}

setInterval(obj.f, 1000); 
setInterval(function () {
    obj.f();
}, 1000) 
setInterval(obj.f.bind(obj), 1000);
```

> 清除定时器

*   可以使用 clearTimeout() 和 clearInterval() 方法来清除定时器
*   setTimeout() 和 setInterval() 都返回一个整数值, 该整数值表示定时器编号; 将这个编号传入 clearTimeout() 和 clearInterval() 就可以取消对应的定时器;
*   需要注意, setTimeout 和 setInterval 返回的定时器编号的数值都是连续的且单调递增的, 可以利用这一点取消所有的定时器 (过一部分检测)
*   在 node 环境下, setTimeout 和 setInterval 返回的不是定时器编号, 而是一个定时器对象;

```
(function(){
    var gid = setInterval(clearAllTimeouts,0);
    console.log(gid);
    function clearAllTimeouts(){
        var id = setTimeout(function(){},0);
        while (id > 0){
            if (id !== gid){
                clearTimeout(id);
            }
            id--;
        }
    }
})()
```

### [](#Promise-对象 "Promise 对象")Promise 对象

*   Promise 对象是 js 的异步操作解决方案, 为异步操作提供了统一的接口; 它起到了代理作用, 充当异步操作与回调函数之间的中介; 让异步操作具备同步操作的接口; Promise 可以让异步操作写起来就像在写同步操作的流程, 而不必一层层嵌套回调函数;
*   Promise 是一个对象, 也是一个构造函数;

```
var objPromise = function (input) {
    return new Promise(function (resolve, reject) {
        var obj = {};
        obj.name = input;
        setTimeout(resolve, 1000, obj.name);
        console.log("继续执行");
    });
}

function resolve(name) {
    console.log("name", name);
    return name;
}

function resolve2(name) {
    console.log("name2", name);
    return name;
}

objPromise("kevin").then(resolve).then(resolve2).catch((error)=>{
    console.log("error", error);});
```

使用 Promise 的时候, 业务逻辑代码都在 newPromise 和 then 的回调方法中;

### [](#Ajax-请求 "Ajax 请求")Ajax 请求

*   Ajax 是一种在无需重新加载整个页面的情况下, 能够更新部分网页的技术;
*   Ajax = 异步 js 和 xml
*   传统的页面如果需要更新内容, 必须要重新加载整个页面; Ajax 可以使网页实现异步更新, 在不加载整个页面的情况下, 对页面的部分进行更新;

```
var xmlhttp = new XMLHttpRequest();

xmlhttp.open("method", "url", async);






xmlhttp.send(string);
```

在异步 async 为 true 时, 需要设置 XMLHttpRequest 对象的 onreadystatechange 一个回调函数; 或者设置 XMLHttpReuqest 对象的 onload 一个回调函数;

```
xmlhttp.onreadystatechange = function(){
    if (xmlhttp.readyState === 4) && (xmlhttp.status === 200){
    console.log(xmlhttp.responseText);
    }
}



xmlhttp.onload = function(){
    if (xmlhttp.readyState === 4) && (xmlhttp.status === 200){
    console.log(xmlhttp.responseText);
    }
}
```

> 简单示例

python 服务端代码

```
__author__ = "Kevin"
__time__ = "2021/8/26"
__blog__ = "https://kevinspider.github.io/"

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

origins = ["*"]

app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/")
async def root():
    return {"message": "Hello World"}
```

html 请求页面

```
<!DOCTYPE html>
<html lang="en" xmlns:v-on="http://www.w3.org/1999/xhtml">
<head>
    <meta charset="UTF-8">
    <title>Demo</title>
    <script src="../snip/vue.js"></script>
</head>
<body>
<div>
    <button v-on:click="promiseOnloadRequest">发送请求</button>
</div>
</body>
<script>
    var vue = new Vue({
        el: "#root",
        methods: {
            request: function () {
                
                var xmlhttp = new XMLHttpRequest();
                
                xmlhttp.onreadystatechange = function () {
                    if (xmlhttp.readyState == 4 && xmlhttp.status == 200) {
                        console.log("success: ", xmlhttp.status);
                        console.log("xmlhttp.responseText: ", xmlhttp.responseText);
                    }
                }
                xmlhttp.open("GET", "http://127.0.0.1:8000", true);
                xmlhttp.send()
            },
            promiseRequest: function () {
                var promise = new Promise(function (resolve, reject) {
                    var xmlhttp = new XMLHttpRequest();
                    xmlhttp.onreadystatechange = function () {
                        if ((xmlhttp.readyState === 4) && (xmlhttp.status == 200)) {
                            resolve(xmlhttp.responseText);
                        }
                    }
                    xmlhttp.open("GET", "http://127.0.0.1:8000", true);
                    xmlhttp.send()
                })

                function f(res) {
                    console.log("use promiseRequest");
                    console.log("res", res);
                }

                promise.then(f).catch((error) => {
                    console.log(error);
                })
            },
            promiseOnloadRequest: function () {
                var promise = new Promise(function (resolve, reject) {
                    var xml = new XMLHttpRequest();
                    xml.open("GET", "http://127.0.0.1:8000", true);
                    xml.onload = resolve;
                    xml.send()
                });

                function parseResponse(res) {
                    console.log("promiseOnloadRequest", res);
                }

                function parseError(error) {
                    console.log("error", error);
                }

                promise.then(parseResponse).catch(parseError);
            }
        }
    });

</script>
</html>
```

[](#Dom-和浏览器模型 "Dom 和浏览器模型")Dom 和浏览器模型
--------------------------------------

### [](#Dom-简介 "Dom 简介")Dom 简介

*   Dom 是 js 操作网页的接口, 全称是文档对象模型; 它的作用是将网页转为一个 js 对象, 从而可以通过用脚本进行各种操作 (增删改查)
*   浏览器会根据 Dom 模型, 将结构化的文档 (html 或 XML) 解析成一系列的节点, 再由这些节点组成一个树状结构; 所有的节点和最终的树状结构都有规范的对外接口;
*   Dom 是一个接口规范, 可以用任何语言实现; 严格来说 Dom 不是 js 语法的一部分, 但是 Dom 操作是 js 最常见的操作, 离开了 Dom, js 就无法控制网页; 另一方面, js 也是最常用于操作 Dom 的语言;

### [](#节点 "节点")节点

*   Dom 的最小组成单位是节点 (node); 文档的树形结构就是由各种不同类型的节点组成;
*   节点一共有七种:
    1.  Document: 整个文档树的顶层节点
    2.  DocumentType: doctype 标签 `<! DOCTYPE html>`
    3.  Element: 网页的各种 HTML 标签, 比如 `<body> <a>`等
    4.  Attr: 网页元素的属性 比如`class="right"`
    5.  Text: 标签之间或标签包含的文本
    6.  Comment: 注释
    7.  DocumentFragment: 文档的片段
    8.  浏览器提供一个原生的节点对象 Node, 上面这七种节点都继承了 Node, 因此具有一些相同的属性和方法

### [](#事件 "事件")事件

事件名称参照: [https://developer.mozilla.org/zh-CN/docs/Web/Events](https://developer.mozilla.org/zh-CN/docs/Web/Events)

*   事件的本质是程序各个组成部分之间的一种通信方式, 也是异步编程的一种实现; Dom 支持大量的事件
*   Dom 的事件操作 (监听和触发) 都定义在 EventTarget 接口; 所有节点对象都部署在这个接口, 其他一些需要事件通信的浏览器内置对象(比如 XMLHttpRequest, AudioNode, AudioContext 等) 也都部署了这个接口;
*   EventTarget 接口主要提供了三个方法
    *   addEventListener(type, listener[,useCapture]): 绑定事件的监听函数
    *   removeEventListener(type, listener[,useCapture]): 移除事件的监听函数
    *   dispatchEvent(event): 触发事件

> 事件传播

*   一个事件发生后, 会在子元素和父元素之间传播, 这种传播分成三个阶段:
    1.  第一个阶段: 从 window 对象传导到目标节点 (上层传到底层), 称为捕获阶段;
    2.  第二个阶段: 在目标节点上触发, 称为目标阶段
    3.  第三个阶段: 从目标节点传导回 window 对象 (从底层传回上层); 称为冒泡阶段;
*   三种阶段的传播模型, 让同一个事件会在多个节点上触发;

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>节点</title>
</head>
<body>
    <div>
        <p>点击触发</p>
    </div>
</body>

<script>
    var phases = {
        "1": "capture", 
        "2": "target", 
        "3": "bubble" 
    }

    var div = document.getElementById("div");
    var p = document.getElementById("p");

    function callback(event) {
        let tag = event.currentTarget.tagName;
        let phase = phases[event.eventPhase];
        console.log("tag", tag);
        console.log("phase", phase);
    }
    
    
    
    
    
    
    div.addEventListener("click", callback, false);
    p.addEventListener("click", callback, false);


</script>
</html>
```

> Event 对象

*   事件发生后, 会产生一个事件对象, 作为参数传递给监听函数; 浏览器原生提供了一个 Event 对象, 所有的事件都是这个对象的实例, 或者说所有的事件都继承了 Event.prototype 对象;
*   Event 对象本身就是一个构造函数, 可以用来生成一个新的事件实例; `var event = new Event(type, options);`
*   Event 构造函数接受两个参数, 第一个参数 type 是字符串, 表示事件的名称; 第二个参数 options 是一个对象, 表示事件对象的配置; 该对象主要有两个属性:
    *   bubbles: 布尔值, 可选, 默认是 false; 表示事件对象是否冒泡;
    *   cancelable: 布尔值, 可选, 默认是 false; 表示事件是否可以被取消, 即能否用`Event.proventDefault()`取消这个事件; 一旦事件被取消, 就好像没有发生过一样, 不会触发浏览器对该事件的默认行为;

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>节点</title>
</head>
<body>
    <div>
        <p>点击触发</p>
    </div>
</body>

<script>
    var phases = {
        "1": "capture", 
        "2": "target", 
        "3": "bubble" 
    }

    var div = document.getElementById("div");
    var p = document.getElementById("p");

    function callback(event) {
        let tag = event.currentTarget.tagName;
        let phase = phases[event.eventPhase];
        console.log("tag", tag);
        console.log("phase", phase);
    }

    var event = new Event("look", {
        bubbles: true,
        cancelable: true
    });

    p.addEventListener("click", callback, false);
    p.dispatchEvent(event);


</script>
</html>
```

> 监听函数

*   浏览器的事件模型, 就是通过监听函数 (listener) 对事件作出反应; 事件发生后, 浏览器监听到了这个事件, 就会执行对应的监听函数;
    
*   三种方式为事件绑定监听函数
    
    *   HTML 的 on - 属性
        
        ```
        <div onclick="btnClick()">点击触发</div>
        <script>
            function btnClick(event) {
                console.log("event", event);
                console.log("button clicked");
            }
        </script>
        ```
        
    *   元素节点的事件属性
        
        ```
        <div>点击触发</div>
        <script>
            var button = document.getElementById("button");
            button.onclick = function (event) {
               console.log("event", event);
               console.log("按钮点击");
            }
        </script>
        ```
        
    *   EventTarget.addEventListener()
        
        ```
        <div>点击触发</div>
        <script>
            var button = document.getElementById("button");
            function btnClick(event) {
                console.log("event", event);
                console.log("addEventListener", "clicked");
            }
            button.addEventListener("click", btnClick);
        </script>
        ```
        

### [](#window "window")window

浏览器中, window 对象指的是当前的浏览器窗口; 它也是当前页面的顶层对象, 它也是当前页面的顶层对象, 即最高一层的对象, 所有其他对象都是它的下属, 一个变量如果未声明, 那么默认就是顶层对象的一个属性;

#### [](#Document "Document")Document

*   document 节点对象代表整个文档, 每个网页都有自己的 document 对象; window.document 属性就指向了这个对象; 只要浏览器开始载入 HTML 文档, 该对象就存在了, 可以直接使用;
*   document 对象有不同的办法可以获取
    *   正常的网页, 直接使用 document 或者 window.document 获取
    *   iframe 框架里面的网页, 使用 iframe 节点的 contentDocuemnt 属性
    *   Ajax 操作返回的文档, 使用 XMLHttpRequest 对象的 responseXML 属性;
    *   内部节点的 ownerDocument 属性
    *   document 对象继承了 EventTarget 接口和 Node 接口; 还 mixin 了 ParentNode 接口;

> Document 常用方法

*   document.open(): 清除当前文档的所有内容, 让文档处于可写状态, 供 document.write() 方法写入内容
*   document.write(): 写入内容
*   document.close(): 关闭文档
*   document.querySelector(): 接受一个 css 选择器作为参数, 返回匹配到该选择器的元素节点; 如果有多个节点满足匹配条件, 则返回第一个匹配的节点; 如果没有发现匹配的节点, 返回 null;
*   document.querySelectorAll(): 接受一个 css 选择器作为参数, 返回匹配到该选择器的所有元素节点数组; 如果没有发现匹配的节点, 返回 null;
*   document.getElementsByClassName(): 返回一个类似数组的对象, 包括了所有 class 名字符合指定条件的元素;
*   document.getElementsByName(): 返回一个类似数组的对象, 包含了所有 name 属性满足条件的元素;
*   document.getElementById(): 返回匹配指定 id 的元素

#### [](#navigator "navigator")navigator

navigator 对象的属性:

*   navigator.userAgent: 浏览器请求头
*   navigator.plugins: 返回一个类数组的对象, 成员是 plugin;
*   navigator.platform: 返回用户的操作系统信息
*   navigator.onLine: 表示当前用户在线还是离线
*   navigator.languages: 表示当前浏览器语言
*   navigator.language: 表示当前浏览器的首选语言
*   navigator.geolocation: 返回一个 Geolocation 地理位置对象
*   navigator.cookieEnabled: 表示浏览器的 Cookie 功能是否开启

navigator 对象的方法

*   navigator.javaEnabled(): 表示浏览器是否能运行 Java Applet 小程序
*   navigator.sendBeacon(): 用于向服务器异步发送数据
*   navigator.deviceMemory(): 返回当前计算机的内存数量 (单位为 G)
*   navigator.connection(): 包含当前网络连接的相关信息;

#### [](#screen "screen")screen

*   screen.height: 浏览器窗口所在的屏幕的高度 (单位像素)
*   screen.width: 浏览器窗口所在的屏幕的宽度 (单位像素)
*   screen.availHeight: 浏览器窗口可用的屏幕高度 (单位像素); 因为部分空间可能不可以用, 比如系统的任务栏或者 mac 系统屏幕底部的 Dock 区; 这个属性等于 height 减去那些被系统占用的组件的高度;
*   screen.availWidth: 浏览器窗口可用的屏幕宽度 (单位像素);
*   screen.pixelDepth: 整数, 表示屏幕的色彩位数, 比如 24 表示屏幕提供 24 位的色彩
*   screen.colorDepth: screen.pixelDepth 的别名, 严格来说, colorDepth 表示应用程序的颜色深度, pixelDepth 表示屏幕的颜色深度, 绝大多数情况下, 它们都是相同的一件事;
*   screen.orientation: 返回一个对象, 表示屏幕的方向;

#### [](#cookie "cookie")cookie

*   cookie 是服务器保存在浏览器的一小段文本信息, 一般大小不能超过 4KB; 浏览器每次向服务器发出请求, 会自动附上这段信息
*   cookie 主要保存状态信息, 一般用于:
    1.  对话 session 管理; 保存登录, 购物车等需要记录的信息
    2.  个性化信息: 保存用户偏好, 比如网页的字体大小, 背景色等;
    3.  追踪用户: 记录和分析用户的行为和偏好

> cookie 的生成

*   服务器如果希望在浏览器保存 cookie, 就需要在 http 的 response 中设置 Set-Cookie 字段;
*   document.cookie 用于读写当前页面的 cookie; 读取的时候, 会返回当前网页中所有的不含 HttpOnly 属性的 cookie 信息;
*   document.cookie 属性时可写的; 可以通过对 document.cookie 设置值来添加 cookie;
*   `document.cookie = "dta=dta; expires=Mon, Aug 30 2021 15:44:17 GMT; path=/; domain=www.dtasecurity.cn"`
*   删除现有的 cookie 的唯一方法, 就是设置它的 expires 属性为一个过去的时间;

#### [](#storage "storage")storage

*   window.sessionStorage 和 window.localStorage 接口都实现了 Storage 接口;
*   window.sessionStorage 保存的数据用于浏览器的一次会话; 当会话结束 (通常是窗口关闭), 数据被清空;
*   localStorage 保存的数据长期存在, 下一次访问该网站的时候, 网页可以直接读取以前的数据
*   除了保存数据的时间长短不同, 这个两个对象其他方面都基本一致

> Storage 属性

*   Storage.length: 返回保存的数据个数

> Storage 方法

*   Storage.setItem(): 接受两个参数, 一个是键名, 一个是要保存的数据; 写入不一定要用这个方法, 也可以直接赋值
*   Storage.getItem(): 用于读取数据, 它只有一个参数, 就是键名;
*   Storage.removeItem(): 用于清除某个键名对应的键值
*   Storage.clear(): 用于清除所有保存的数据
*   Storage.key(): 接受一个整数作为参数, 返回该位置对应的键名; 结合 Storage.length 属性和 Storage.key() 方法, 可以遍历所有的键;

#### [](#history "history")history

*   window.history 属性指向 History 对象, 它表示当前窗口的浏览历史;
*   History.length: 当前窗口访问过的网址数量 (包括当前页面)
*   History.state: History 堆栈最上层的状态值, 通常是 undefined;
*   History.back(): 移动到上一个网址, 等同于点击浏览器的后退键
*   History.forward(): 移动到下一个网址, 等同于点击浏览器的前进键;
*   History.go(): 接受一个整数作为参数, 以当前网址作为基准, 移动到参数指定的网址
*   History.pushState(): 在历史记录中添加一条记录
*   History.replaceState(): 修改 History 对象的当前记录;

#### [](#location "location")location

location 对象是浏览器提供的原生对象, 提供 URL 相关的信息和操作方法; 通过 window.location 或者 document.location 都可以获得这个对象;

> location 属性

*   location.href: 整个 URL
*   location.protocol: 当前 URL 的协议, 包括冒号 (:)
*   location.host: 主机, 如果端口不是协议默认的 80 和 443, 则还会返回冒号和端口号
*   location.hostname: 主机名, 不包括端口号;
*   location.port: 端口号
*   location.pathname: URL 的路径部分, 从根路径 / 开始
*   location.search: 查询字符串部分, 从? 开始;
*   location.hash: 片段字符串部分, 从 #开始
*   location.username: 域名签名的用户名
*   location.password: 域名前面的密码;
*   location.origin: URL 的协议, 主机名和端口

> location 方法

*   location.assign(): 接受一个 URL 字符串参数, 让浏览器立即跳转到新的 URL;
    
*   location.replace(): 同上, 但是会在浏览器的历史 history 里面删除当前网址;
    
*   location.reload(): 使得浏览器重新加载当前网址, 相当于刷新
    
*   location.toString(): 返回整个 URL 字符串, 相当于读取了 location.href 属性;
    
*   location.encodeURI(): 会将除了元字符和语义字符之外的字符都进行转义, 一般用于转义整个 url;
    
    ```
    var url = "https://www.baidu.com?q=你好";
    encodeURI(url);
    ```
    
*   location.encodeURIComponent(): 会将除了语义字符之外的所有字符都转义, 一般用于转义部分 url;
    
    ```
    var url = "https://www.baidu.com?q=你好";
    encodeURIComponent(url);
    ```
    
*   location.decodeURI(): encodeURI() 方法的逆运算
    
*   location.decodeURIComponent(): encodeURIComponent() 方法的逆运算
    

[](#滑块 "滑块")滑块
--------------

### [](#hook-定位 "hook 定位")hook 定位

比较常见的通过 Canvas 绘制滑块验证码的 hook 点:

```
(function() {
    'use strict';
    
    console.log("hook CanvasRenderingContext2D");
    var _drawImage = CanvasRenderingContext2D.prototype.drawImage;
    var _putImageData = CanvasRenderingContext2D.prototype.putImageData;
    CanvasRenderingContext2D.prototype.drawImage = function(){
        console.log("CanvasRenderingContext2D.prototype.drawImage", arguments);
        debugger;
        return _drawImage.apply(this, arguments);
    };
    CanvasRenderingContext2D.prototype.putImageData = function(){
        console.log("CanvasRenderingContext2D.prototype.putImageData", arguments);
        debugger;
        return _putImageData.apply(this, arguments);
    }
})();
```

通过 hook 追踪堆栈, 找到 js 中是如何对正序数组进行还原的;

### [](#hook-获取轨迹 "hook 获取轨迹")hook 获取轨迹

```
(function () {
    let _addEventListener = EventTarget.prototype.addEventListener;

    window.flag = false;
    window.zedtail = [];
    window.start_x;
    window.start_y;
    window.start_t;

    EventTarget.prototype.addEventListener = function () {
        let eventname = arguments[0];
        let eventfunc = arguments[1];
        let neweventfunc = function (events) {
            if (events.type === "mousedown") {
                window.zedtail = [];
                window.start_x = events.clientX;
                window.start_y = events.clientY;
                window.start_t = +new Date;
                window.flag = true;
            } else if (events.type === "mousemove") {
                if (window.flag) {
                    let movex = parseInt(events.clientX - window.start_x);
                    let movey = parseInt(events.clientY - window.start_y);
                    let movet = (new Date).getTime() - window.start_t;
                    console.log([movex, movey, movet]);
                    window.zedtail.push([movex, movey, movet]);
                }

            } else if (events.type === "mouseup") {
                console.log(window.zedtail);
                window.flag = false;
            }
            eventfunc(events);
        };
        console.log(eventname, eventfunc.toString());
        return _addEventListener.apply(this, [eventname, neweventfunc]);
    }
})();
```

### [](#完整代码 "完整代码")完整代码

通过正序数组将乱序图片还原成正序图片

```
__author__ = "Kevin"
__time__ = "2021/9/1"
__blog__ = "https://kevinspider.github.io/"

"""
e.canvasCtx.drawImage(
    e.img, 
    30 * a,             sx可选 需要绘制到目标上下文中的，image的矩形（裁剪）选择框的左上角 X 轴坐标。 
    0,                  sy可选 需要绘制到目标上下文中的，image的矩形（裁剪）选择框的左上角 Y 轴坐标。
    30,                 sWidth可选 需要绘制到目标上下文中的，image的矩形（裁剪）选择框的宽度。如果不说明，整个矩形（裁剪）从坐标的sx和sy开始，到image的右下角结束。
    400,                sHeight可选  需要绘制到目标上下文中的，image的矩形（裁剪）选择框的高度。
    30 * t[a] / 1.5,    dx image的左上角在目标canvas上 X 轴坐标。
    0,                  dy  image的左上角在目标canvas上 Y 轴坐标。
    20,                 image 在目标canvas上绘制的宽度。 允许对绘制的image进行缩放。 如果不说明， 在绘制时image宽度不会缩放。
    200)                dHeight可选 image在目标canvas上绘制的高度。 允许对绘制的image进行缩放。 如果不说明， 在绘制时image高度不会缩放。
"""

import json
import math
import random
import requests
import base64
import cv2 as cv
import numpy as np
import matplotlib.pyplot as plt
from PIL import Image




def getImg():
    headers = {
        'Connection': 'keep-alive',
        'Pragma': 'no-cache',
        'Cache-Control': 'no-cache',
        'Accept': 'application/json, text/plain, */*',
        'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36',
        'Origin': 'http://www.dtasecurity.cn:30080',
        'Referer': 'http://www.dtasecurity.cn:30080/',
        'Accept-Language': 'zh,zh-CN;q=0.9',
    }

    params = (
        ('w', '400'),
        ('h', '200'),
    )

    response = requests.get('http://www.dtasecurity.cn:35555/picture', headers=headers, params=params, verify=False)
    data = json.loads(response.text)

    
    sid = data['sid']
    
    c = data['c']
    p1_path = "./p1.jpeg"
    p2_path = "./p2.jpeg"
    
    p1 = data['p1'].split(',')[-1]
    
    p2 = data['p2'].split(',')[-1]
    with open(p1_path, 'wb') as f:
        f.write(base64.b64decode(p1))
    with open(p2_path, 'wb') as f:
        f.write(base64.b64decode(p2))
    
    y = data['y']
    return sid, c, p1_path, p2_path, y



def decodeKey(input):
    order_list = []
    for each in input:
        order_list.append(ord(each) ^ 66)
    print(order_list)
    return order_list



def getRightImg(path, order_list, offset):
    right_path = "./right.jpg"
    wrong_img = Image.open(path)
    width, height = wrong_img.size

    
    right_order = []
    for i in range(len(order_list)):
        right_order.append(0)

    
    for index, i in enumerate(order_list):
        box = (index * offset, 0, (index + 1) * offset, height)
        tmp_img = wrong_img.crop(box)
        right_order[i] = tmp_img

    
    x = y = 0
    right_img = Image.new("RGB", (width, height), 255)
    for img in right_order:
        right_img.paste(img, (x, y))
        x += offset

    
    right_img.save(right_path)
    return right_path



def getDistance(right_path, p2_path):
    
    def find_template(canny_img, canny_slide_img):
        result = cv.matchTemplate(canny_img, canny_slide_img, cv.TM_CCOEFF_NORMED)
        min_val, max_val, min_loc, max_loc = cv.minMaxLoc(result)
        return min_val, max_val, min_loc, max_loc

    
    gray_img = cv.imread(right_path, cv.IMREAD_GRAYSCALE)
    gray_slide_img = cv.imread(p2_path, cv.IMREAD_GRAYSCALE)
    
    gs_img = cv.GaussianBlur(gray_img, (5, 5), 0)
    gs_slide_img = cv.GaussianBlur(gray_slide_img, (5, 5), 0)
    
    canny_img = cv.Canny(gs_img, 25, 45)
    canny_slide_img = cv.Canny(gs_slide_img, 25, 45)

    result = find_template(canny_img, canny_slide_img)
    print(result)
    return result

    
    
    
    
    
    

    
    
    

    
    
    
    
    







def show_plt():
    for i in range(1, 6):
        trail_list = eval(f"trail_{i}")
        print(f"trail_{i}")
        x_trail = []
        y_trail = []
        t_trail = []
        for trail in trail_list:
            x = trail[0]
            y = trail[1]
            t = trail[2]
            x_trail.append(x)
            y_trail.append(y)
            t_trail.append(t)
        print(np.diff(x_trail))
        
        
        plt.plot(t_trail, x_trail)
    plt.show()









def show_easeOutQuint(distance):
    def func(x):
        return (1 - pow(1 - x, 5)) + 6.59375  

    size = 400
    
    x = np.linspace(-0.5, 1, size)
    
    print(func(-0.5), func(1))
    
    
    y = [func(i) for i in x]
    plt.plot(x, y)
    plt.show()

    
    t = np.linspace(200, 3000, size)

    
    delta_pt = abs(np.random.normal(scale=1.1, size=size))
    for index in range(len(delta_pt)):
        change_y = int(x[index] + delta_pt[index])
        if (index + 1 < size and y[index + 1] > change_y):
            y[index] += change_y
            

    print("np.diff(y)", np.diff(y))
    plt.plot(t, y)
    plt.show()



def get_trail(move_distence, show=False):
    def easeOutQuint(x):
        return (1 - math.pow(1 - x, 5))

    def __set_pt_time(_dist):
        if _dist < 100:
            __need_time = int(random.uniform(500, 1500))
        else:
            __need_time = int(random.uniform(1000, 2000))
        __end_pt_time = []
        __move_pt_time = []
        __pos_z = []

        total_move_time = __need_time * random.uniform(0.8, 0.9)
        start_point_time = random.uniform(110, 200)
        __start_pt_time = [int(start_point_time)]

        sum_move_time = 0

        _tmp_total_move_time = total_move_time
        while True:
            delta_time = random.uniform(15, 20)
            if _tmp_total_move_time < delta_time:
                break

            sum_move_time += delta_time
            _tmp_total_move_time -= delta_time
            __move_pt_time.append(int(start_point_time + sum_move_time))

        last_pt_time = __move_pt_time[-1]
        __move_pt_time.append(int(last_pt_time + _tmp_total_move_time))

        sum_end_time = start_point_time + total_move_time
        other_point_time = __need_time - sum_end_time
        end_first_ptime = other_point_time / 2

        while True:
            delta_time = random.uniform(110, 200)
            if end_first_ptime - delta_time <= 0:
                break

            end_first_ptime -= delta_time
            sum_end_time += delta_time
            __end_pt_time.append(int(sum_end_time))

        __end_pt_time.append(int(sum_end_time + (other_point_time / 2 + end_first_ptime)))
        __pos_z.extend(__start_pt_time)
        __pos_z.extend(__move_pt_time)
        __pos_z.extend(__end_pt_time)
        return __pos_z

    def __get_pos_y(point_count):
        _pos_y = []
        start_y = random.randint(-1, 1)
        end_y = random.randint(-13, -5)
        x = np.linspace(start_y, end_y, point_count)
        for _, val in enumerate(x):
            _pos_y.append(int(val))

        return _pos_y

    time_list = __set_pt_time(move_distence)
    trail_length = len(time_list)
    t = np.linspace(-0.5, 1, trail_length)  

    
    print(easeOutQuint(-0.5), easeOutQuint(1))

    mult = move_distence / 7.59375

    x = [int(mult * (easeOutQuint(i) + 6.59375)) for i in t]
    y = __get_pos_y(trail_length)
    
    
    delta_pt = abs(np.random.normal(scale=3, size=trail_length))
    for index in range(len(delta_pt)):
        change_x = int(x[index] + delta_pt[index])
        if index + 1 < trail_length and x[index + 1] > change_x:
            x[index] = change_x

    if show:
        delta_t = [i for i in range(trail_length)]
        plt.plot(delta_t, delta_pt, color='green')
        plt.plot(time_list, x, color='red')
        plt.show()

    result = []
    print(x[-1] - x[0])
    for idx in range(trail_length):
        result.append([x[idx], y[idx], time_list[idx]])
    return result


def encodeData(trail):
    postStr = ""
    for each in trail:
        x = each[0]
        y = each[1]
        t = each[2]
        tmp = f"{x},{y},{t}"

        postStr += base64.b64encode(tmp.encode()).decode() + "*"
    postStr = postStr.rstrip("*")
    return postStr


def postData(postStr, sid):

    headers = {
        'Connection': 'keep-alive',
        'Pragma': 'no-cache',
        'Cache-Control': 'no-cache',
        'Accept': 'application/json, text/plain, */*',
        'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/92.0.4515.159 Safari/537.36',
        'Content-Type': 'application/json;charset=UTF-8',
        'Origin': 'http://www.dtasecurity.cn:30080',
        'Referer': 'http://www.dtasecurity.cn:30080/',
        'Accept-Language': 'zh,zh-CN;q=0.9',
    }

    data = {
        "sid": sid,
        "trail": postStr
    }

    response = requests.post('http://www.dtasecurity.cn:35555/slide',
                             headers=headers, json=data, verify=False)
    print(response.text)


if __name__ == '__main__':
    sid, c, p1_path, p2_path, y = getImg()
    print(sid, c, p1_path, p2_path, y)

    order_list = decodeKey(c)
    right_path = getRightImg(p1_path, order_list, 30)

    min_val, max_val, min_loc, max_loc = getDistance(right_path, p2_path)

    distance = max_loc[0]
    trail = get_trail(distance / (30 / 20), True)  
    postStr = encodeData(trail)
    postData(postStr,sid)
```

[](#v8-引擎代替-node "v8 引擎代替 node")v8 引擎代替 node
--------------------------------------------

v8 引擎内置的对象参考链接: [https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects)

除了上述文档中的内容, 其他的内容都是浏览器实现的;

### [](#补环境 "补环境")补环境

难点:

1.  如何找到缺少哪些环境
2.  如何实现虚拟环境

> 如何找到缺少哪些环境