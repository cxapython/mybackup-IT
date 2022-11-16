> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [gift1a.github.io](https://gift1a.github.io/2022/09/24/Frida-Practice/)

> Frida-Practice

Frida-Practice

[https://github.com/OWASP/owasp-mastg/tree/master/Crackmes](https://github.com/OWASP/owasp-mastg/tree/master/Crackmes)

[¶](#校验so和dex) 校验 so 和 dex
--------------------------

首先创建一个 [HashMap](https://zhuanlan.zhihu.com/p/127147909)<key,value>，<key,value > 是 Entry 的一个实体，获取资源值对应的字符串后转为 Long 类型，然后使用 put 方法存入 HashMap 中，获得到对应的文件后进行 CRC 校验

对 so 文件和 dex 文件进行 CRC 校验防止修改

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1664745653942.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1664745653942.png)

遍历 HashMap

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1665500859197.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1665500859197.png)

[¶](#调试检测) 调试检测
---------------

创建异步任务来检测调试

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1665501157986.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1665501157986.png)

doInBackground 方法是自定义的线程任务，onPostExecute 在线程任务执行后进行，参数接收线程任务执行结果，参数为对 UI 控件进行设置，execute() 用户手动调用异步任务

[¶](#Root-v2)Root
-----------------

Root 环境检测

*   test-keys 表示为非官方发布版本， release-keys 为官方发布版本
    
*   检查 su 命令 - 检测常用目录下是否存在 su、判断系统环境变量是否存在 su
    
*   检测指定路径下的文件是否存在
    

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1664745696303.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1664745696303.png)

```
var clazz=Java.use("sg.vantagepoint.util.RootDetection")
        clazz.checkRoot1.implementation=function(args){
            return false;
        }
        clazz.checkRoot2.implementation=function(args){
            return false;
        }
        clazz.checkRoot3.implementation=function(args){
            return false;
        }
        var clazz11=Java.use("sg.vantagepoint.util.IntegrityCheck");
        clazz11.isDebuggable.implementation=function(args){
            return false;
        }
```

[¶](#Frida检测)Frida 检测
---------------------

使用 **Interceptor.replace 替换循环检测 Frida 的函数**

> **第一个参数是要替换函数的地址，第二个参数是一个 NativePointer 类型的函数，一般由 new NativeCallback 创建，NativeCallback 包括（函数的实现，函数的返回类型，函数的参数类型列表）。返回类型最好保持和原函数一致。同时我们也可以使用 NativeFunction 来主动调用原函数或是 so 层函数**

```
var addr=Module.findExportByName("libc.so","strstr");
        console.log(addr);
        
        Interceptor.attach(addr,{
            onEnter:function(args){
                
            },
            onLeave:function(retval){
                retval.replace(0);
                return retval;
            }
        })

var pt_create_func=Module.findExportByName("libc.so","pthread_create");
       console.log("pt_create_func_addr:",pt_create_func);
       var detect_frida_addr=null;
        Interceptor.attach(pt_create_func,{
            onEnter:function(args){
                if(detect_frida_addr==null){
                    var base_addr=Module.findBaseAddress("libfoo.so");
                    detect_frida_addr=base_addr.add(0x30D0);
                    
                    console.log("Ptread_Addr:",base_addr);
                    Interceptor.replace(this.context.x2,new NativeCallback(function(a1){
                    console.log("Replace Success");
                    return;
                },'void',[]));
                }
            },
            onLeave:function(retval){
                
            }
        })
```

**如果想要在替换的函数中使用原函数的参数，需要注意参数类型**，比如传入的 env、jclass、data 是指针类型，那么**参数列表中就要声明为 "pointer"**

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/QQ%E5%9B%BE%E7%89%8720221003045438-1664744219994.jpg)](https://gift1a.github.io/2022/09/24/Frida-Practice/QQ%E5%9B%BE%E7%89%8720221003045438-1664744219994.jpg)

[¶](#Hook-函数用于获取参数)Hook 函数用于获取参数
--------------------------------

```
var MainActivity = Java.use("sg.vantagepoint.uncrackable3.MainActivity");
         MainActivity.$init.implementation = function() {
                this.$init();
                attachToSecretGenerator();
        };
        var addr2=Module.findExportByName("libfoo.so","Java_sg_vantagepoint_uncrackable3_CodeCheck_bar");
        Interceptor.attach(addr2,{
            onEnter:function(args){
                console.log((args[2].readByteArray(32)));
                var addr1=Module.findBaseAddress("libfoo.so");
                addr1=addr1.add(0x15038);
                console.log("=====xor_key=====");
                console.log(hexdump(addr1));
            },
            onLeave:function(retval){
                
            }
        })
    })
}

function attachToSecretGenerator() {
    
    Interceptor.attach(Module.findBaseAddress('libfoo.so').add(0x10E0), {
      onEnter: function(args) {
        
        this.answerLocation = args[0];
      },
      onLeave: function(retval) {
        
        console.log(this.answerLocation.readByteArray(32));
      }
    });
  }
```

[wp](https://zhuanlan.zhihu.com/p/397872252)

Challenge-Six

反编译后的代码

```
package com.dta.test.frida.activity;

import android.view.View;
import com.dta.test.frida.R;
import com.dta.test.frida.base.BaseActivity;
import com.dta.test.frida.base.Level;


public class SixthActivity extends BaseActivity {
    @Override 
    protected String getActivityTitle() {
        return "第六关";
    }

    @Override 
    protected String getDescription() {
        return getString(R.string.sixth);
    }

    @Override 
    public void onClick(View view) {
        super.onClick(view);
        try {
            Class<?> cls = Class.forName("com.dta.test.frida.activity.RegisterClass");
            if (((Boolean) cls.getDeclaredMethod("next", new Class[0]).invoke(cls.newInstance(), new Object[0])).booleanValue()) {
                gotoNext(Level.Seventh);
            } else {
                failTip();
            }
        } catch (Exception unused) {
            failTip();
        }
    }
}
```

代码中使用反射调用 com.dta.test.frida.activity.RegisterClass 中的 next 方法，然后接收其返回值

**Frida 为我们提供了 java.registerClass 来注册类到内存中**，其格式如下

```
var RegisterClass=Java.registerClass({
            name:"com.dta.test.frida.activity.RegisterClass",
            
            
            methods:{
                
                next: {
                    
                    returnType: "boolean",
                    
                    argumentTypes:[],
                    
                    implementation: function(){
                        return true
                    }
                }
				
                
                
                
                
            }
        })
```

这样子就注册好了 next 方法，但是我们仍无法通过这一关

```
console.log(RegisterClass.$new().next());
```

使用上述代码创建一个类的实例后调用 next 方法会发现我们的类成功注册并且 next 方法创建成功

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1665206578443.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1665206578443.png)

**这里的问题出在了 ClassLoader 上**

翻看 Class.forName() 函数参数列表可以知道在其参数列表中存在 ClassLoader

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1665206941779.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1665206941779.png)

**默认为当前类的 ClassLoader，在 Android 中默认的活动加载器为 PathClassLoader** ，Frida 加载我们自定义类 RegisterClass 使用的 **ClassLoader** 跟 SixthActivity 的 **PathClassLoader** 并不是同一个 ClassLoader，所以导致我们使用 PathClassLoader 来加载会出现 **ClassNotFoundException**

```
var targetClassLoader = RegisterClass.class.getClassLoader()
       console.log(targetClassLoader);
```

使用 **class.getClassLoader** 获取类加载器后打印可知其使用的是 DexClassLoader

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1665207462148.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1665207462148.png)

**由于使用双亲委派机制，所以我们可以修改其父加载器使其能正确加载**

安卓类加载器的继承关系图

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/R.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/R.png "查看源图像")

[查看源图像](https://gift1a.github.io/2022/09/24/Frida-Practice/R.png "查看源图像")

> **双亲委派机制是指当加载类时首先不会选择自身进行加载，而是先交由其父加载器进行，如果父加载器加载不了再由自己进行加载，所以相当于从 ClassLoader 往下开始进行加载**

我们可以在 PathClassLoader 与 BaseDexClassLoader 之间插入我们定义类的加载器

```
var targetClassLoader = RegisterClass.class.getClassLoader()
       console.log(targetClassLoader);
       Java.enumerateClassLoaders({
            onMatch: function(loader){
                try{
                    if(loader.findClass("com.dta.test.frida.activity.SixthActivity")){
                         
                         var PathClassLoader = loader
                         
                         var BootClassLoader = PathClassLoader.parent.value
                         PathClassLoader.parent.value = targetClassLoader
                         targetClassLoader.parent.value = BootClassLoader
                    }
 
                }catch(e){
                    
                }
            },
            onComplete: function(){
                console.log("Completed!")
            }
        })
```

也可以 Hook Class.forName 方法，指定 ClassLoader 为 RegisterClass 的加载器

[Frida-Detect](https://xxr0ss.github.io/post/frida_detection/)

** 由于 Frida 是代码注入工具，所以 Frida 的模块会在进程中留下痕迹，proc 是 Linux 中的伪文件系统，而在内存中存在 / proc/pid 目录存储进程的一些信息，应用程序可以访问获得内核的信息，/proc/pid/maps 显示进程的内存区域映射信息 **

1.  **通过 maps 检测**：扫描 / proc/pid/maps 文件中的内存分布，寻找是否打开了 / data/local/tmp 路径下的 so，（Frida 在运 行时会先确定 / data/local/tmp 路径下是否有 re.frida.server 文件夹，若没有则创建该文件夹并存放 frida-agent.so 等文件），我们使用 cat 命令即可获得
    
    [![](https://gift1a.github.io/2022/09/24/Frida-Practice/1664546229771.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1664546229771.png)
    
2.  **通过 task 检测**：扫描 task 目录下所有 / task/pid/status 中的 Name 字段是否存在 frida 注入的特征，具体线程名为 **gmain、gdbus、gum-js-loop**，一般这三个线程是在第 11-13 行，**同时也会存在 Name 字段为 pool-frida 的线程**
    
    [![](https://gift1a.github.io/2022/09/24/Frida-Practice/1664546751921.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1664546751921.png)
    
    [![](https://gift1a.github.io/2022/09/24/Frida-Practice/1664546732410.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1664546732410.png)
    
3.  **通过 fd 检测**：通过 readlink 查看 / proc/pid/fd 和 / proc/task/pid/fd 下所有的文件，检测是否有 frida 相关文件
    
    [![](https://gift1a.github.io/2022/09/24/Frida-Practice/1664547128892.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1664547128892.png)
    
4.  **端口检测**：Frida 默认的端口为 **27042**，检测该 TCP 端口是否开放
    
5.  **通过 D-Bus 检测**： Frida 是通过 D-Bus 协议进行通信的，所以可以遍历 / proc/net/tcp 文件，向每个开放的端口发送 D-Bus 的 认证消息 AUTH ，如果端口回复了 REJECT ，那么这个端口就是 frida-server
    

[¶](#PortCheck)PortCheck
------------------------

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1664729895041.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1664729895041.png)

[¶](#ThreadCheck)ThreadCheck
----------------------------

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1664730157823.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1664730157823.png)

[¶](#MapsCheck)MapsCheck
------------------------

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1664730381755.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1664730381755.png)

[¶](#TraceCheck)TraceCheck
--------------------------

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1664730547121.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1664730547121.png)

**maps 显示当前正在运行进程的地址映射或进程的内存地址空间的文件**

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1664730594599.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1664730594599.png)

[¶](#fdCheck)fdCheck
--------------------

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1664730742475.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1664730742475.png)

[¶](#总结) 总结
-----------

从上面的各种检测方式中，可以看出主要方式都是通过打开特定的文件和进程内存文件，然后使用字符串比较函数 strstr，或者使用 **stat** 判断文件是否存在，也可以进行端口检测，**Frida 默认端口是 27042**

对于端口检测，可以 Hook 掉 **connect 函数的返回值**

```
const module=Process.getModuleByName("libc.so");
        console.log(module.base);
        
        
        
        
        var connect=module.findExportByName("connect");
        console.log(connect);
        Interceptor.attach(connect,{
            onEnter:function(args){

            },
            onLeave:function(retval){
                console.log("Change Success!!");
                
                retval.replace(1);
            }
        })
```

> **也可以修改 frida-server 默认端口**，**/data/local/tmp # ./frida-server-15.1.17-android-arm64 -l 0.0.0.0:1234，然后 adb forward tcp:1234 tcp:1234，脚本运行： frida -H 127.0.0.1:1234 -f owasp.mstg.uncrackable3 -l .\Uncrackme3.js **

所以我们只需要 Hook 掉 **libc 中的 strstr 函数**即可

```
const module=Process.getModuleByName("libc.so");

var strstr=module.findExportByName("strstr");
        console.log(strstr);
        Interceptor.attach(strstr,{
            onEnter:function(args){
                
            },
            onLeave:function(retval){
                retval.replace(0);
                
            }
        })

        var stat=module.findExportByName("stat");
        console.log(stat);
        Interceptor.attach(stat,{
            onEnter:function(args){
                
            },
            onLeave:function(retval){
                retval.replace(1);
            }
        })
```

> **虽然这里的 System.LoadLibrary 是 static 属性的，但是不知道为啥没有提前加载 =。=，所以需要我们手动 Hook 主活动的构造器**

```
var MainActivity = Java.use("com.yimian.envcheck.MainActivity");
        MainActivity.$init.implementation = function() {
               this.$init();
       };
```

当然我们也可以 Hook 掉比较的字符串，首先先在内存中写入我们用于比较的字符串 (**allocUtf8String**)，然后将地址在 onEnter 回调函数中**进而修改 strstr 函数的参数**

```
var newStr="new String";
       var newstraddr=Memory.allocUtf8String(newStr);
       var strcpy=module.findExportByName("strstr");
       Interceptor.attach(strcpy,{
           
           onEnter:function(args){
               args[1]=newstraddr;
               console.log(args[1].readCString());
           },
           onLeave:function(retval){
               
           }
       })
```

在 Memory 的 check 中使用的是逐字符比较，我们可以 **Hook 掉汇编指令中的比较值**

```
const byte1=[0x5F,0x41,0x01,0x71];
      const byte2=[0x7F,0xE9,0x01,0x71];
      const byte3=[0x7F,0xE9,0x01,0x71];
      const byte4=[0x3F,0x42,0x01,0x71];
      const byte5=[0x5F,0x42,0x01,0x71];

      var module_base=Module.findBaseAddress("libtestfrida.so");
      
      var func=module_base.add(0x001D5C);
      console.log(Instruction.parse(func));
      
      console.log(func);
      var func2=module_base.add(0x01E54);
      var func5=module_base.add(0x1FF8);
      var func3=module_base.add(0x01F24);
      var func4=module_base.add(0x20BC);
      
      
     
      
      Memory.protect(func,8,'rwx');
      Memory.protect(func2,8,'rwx');
      Memory.protect(func3,8,'rwx');
      Memory.protect(func4,8,'rwx');
      Memory.protect(func5,8,'rwx');
      func.writeByteArray(byte1);
      func2.writeByteArray(byte2);
      func3.writeByteArray(byte3);
      func4.writeByteArray(byte4);
      func5.writeByteArray(byte5);
```

首先我们先找到想要 Hook 的 **CMP 汇编地址，然后根据机器码创建 byte 数组，最后使用 writeByteArray 写入即可**

以 0x1DA4 为例，[5F,C9,01,71] 是地址中原来的数据，C9 是用于比较的数，我们只需要修改其即可，最后写入内存中

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1664731679822.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1664731679822.png)

> **可以使用 Instruction.prase(addr) 打印汇编代码，使用 Memory.protect(addr,size,“rwx”); 来设置修改内存的属性和大小**

```
Interceptor.attach(func2,{
            onEnter:function(args){
                console.log(this.context.x11);
                this.context.x11=0x45;
                console.log(this.context.x11);
            },
            onLeave:function(retval){
                console.log(this.context.x11);
            }
        })
```

**也可以使用 Frida 进行 inlineHook，并且修改寄存器的值，这里的 W11 是 X11 的低 32 位**

最终效果

[![](https://gift1a.github.io/2022/09/24/Frida-Practice/1664734091908.png)](https://gift1a.github.io/2022/09/24/Frida-Practice/1664734091908.png)