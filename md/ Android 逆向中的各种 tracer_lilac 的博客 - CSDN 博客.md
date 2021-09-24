> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_38851536/article/details/115082548?spm=1001.2014.3001.5501)

### JAVA 层

*   正向开发中的 trace 工具：DDMS  
    永远的神，从很久以前到现在都有人用，这个工具说不上好坏，就挺牛的。
*   XPOSED TRACER  
    使用 Xposed 编写插件，HOOK 所有 JAVA 调用，以前见过，比较容易崩溃，现在应该几乎没人用了。
*   [Zentrace](https://github.com/hluwa/ZenTracer)  
    基于 Frida 构建的 trace 脚本，非常优雅，大部分时候很好用，现在也依然是选择之一。
*   [Dwarf](https://github.com/iGio90/Dwarf)  
    稳定性好像不太行，我也没摸过几次
*   [Objection](https://github.com/sensepost/objection) 和 [r0trace](https://github.com/r0ysue/r0tracer)  
    基于 Frida 构建，应该是现在 Java 层分析和 trace 的主力工具。
*   Frida-trace  
    Frida 官方 trace 工具，支持 Java 方法的模糊匹配，挺好的。

Native 层

*   [trace-natives](https://github.com/Pr0214/trace_natives)  
    基于 Frida trace 写的小脚本，挺有用。
*   IDA trace  
    IDA，yyds。
*   Frida Stalker  
    指令级 trace，yyds，但在 Android 上只支持 arm64。
*   [Unicorn](https://github.com/unicorn-engine/unicorn)/[Unidbg](https://github.com/zhkl0228/unidbg) trace  
    万能的神。
