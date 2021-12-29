> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.jianshu.com](https://www.jianshu.com/p/92796f7cf1a2)

```
Java.perform(function () {
    var class1 = Java.use("com.ss.a.b.a");
    var class2 = Java.use("com.ss.sys.ces.a"); // 底层
    var JavaString = Java.use("java.lang.String");
    var ByteString = Java.use("com.android.okhttp.okio.ByteString");

    class2.leviathan.implementation = function (s1, s2) {
        console.log("com.ss.sys.ces.a => leviathan");
        var result = this.leviathan(s1, s2);
        console.log("com.ss.sys.ces.a => result:" + result);
        console.log("com.ss.sys.ces.a => result string:" + ByteString.of(result).hex());
        return result;
    };

    class1.a.overload('java.lang.String').implementation = function (s1) {
        console.log("com.ss.a.b.a = >a");
        console.log("参数1:" + s1);
        var result = this.a(s1);
        console.log("com.ss.a.b.a => result:" + result);
        // console.log("com.ss.a.b.a => result:" + JavaString.$new(result));
        return result;
    }
});


```

```
# 以下代码来自龙哥分享
console.log("加载脚本成功！");
Java.perform(function x() {
    //定位StringBuilder,StringBuffer类
    const stringbuilder = Java.use("com.dianping.util.NativeHelper");

    //定位方法
    const func = "ndug";

    // 使用log类和Exception类产生堆栈
    var jAndroidLog = Java.use("android.util.Log");
    var jException = Java.use("java.lang.Exception");

    var ByteString = Java.use("com.android.okhttp.okio.ByteString");

    stringbuilder[func].implementation = function(x, y, z){
        //打印输入参数

        //方法一
        var argsArray = [];
        for(var i = 0; i < x.length; i++) {
            argsArray.push(x[i]);
        }
        console.log("测试一："+"["+argsArray.join(",")+"]");

        //方法二
        var arr = Java.use("java.util.Arrays");
        console.log("测试二：" + arr.toString(x));

        console.log("参数二："+arr.toString(y));
        console.log("参数三："+arr.toString(z));

        //方法三
        console.log("测试三"+JSON.stringify(x));

        // 打印十六进制字符串
        var arg1 = ByteString.of(x).hex();
        console.log("Arg1:"+arg1);
        console.log("Arg2:"+ByteString.of(y).hex());
        console.log("Arg3:"+ByteString.of(z).hex());

        //执行原逻辑
        const result = this[func](x, y, z);
        // 打印返回的字符串内容

        var ret = ByteString.of(result).hex();
        console.log("Result:"+ret);

        console.log("Result Length:"+ret.length);
        // 只有长度大于30时，才打印堆栈
        if (result.length > 15) {
            // 抛出异常。打印堆栈
            console.log(jAndroidLog.getStackTraceString(jException.$new()));
        }

        //return出去
        return result;
    };
});


```