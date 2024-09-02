> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.cnblogs.com](https://www.cnblogs.com/WXjzc/p/18351364)

> 这个我实际上弄了很久了，一开始更新的时候，发现数据库操作都是在 so 里，那时候是在 libkernel.so 里直接 hook sqlcipher 的密钥函数拿到的密钥，32 位字符串，很容易让人联想到 md5，但是......

这个我实际上弄了很久了，一开始更新的时候，发现数据库操作都是在 so 里，那时候是在`libkernel.so`里直接 hook sqlcipher 的密钥函数拿到的密钥，32 位字符串，很容易让人联想到 md5，但是没有找到在哪里计算的

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733028-623519868.png)

最近又想着做一下，这时打开数据库的 so 就变了，这是`easyFrida`的`sofileopen`插件 hook 出来的结果，目前使用的是`libbasic_share.so`这个 so

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733290-460681254.png)

不过这个 so 在 apk 包里面并没有，应该是运行的时候释放出来的

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733527-641651587.png)

在应用目录里是可以拿到的

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733516-1956706750.png)

结果这个 so 的导出表里有 md5 相关的函数，而以前的`libkernel`并没有，感谢 qq！！！！！！！！！！！！！

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190732996-1732971800.png)

而且很多以前没有函数名的函数，这次也能看到函数名了，就比如密钥初始化函数

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809191054285-1292250991.png)

 hook 一下拿到密钥（直接 attach，然后在登录、查看个人信息等时机可以 hook 到）

```
setImmediate(function () {
    Java.perform(function () {
        function buf2str(buffer) {
            let result = '';
            const byteArray = new Uint8Array(buffer);
            for (let i = 0; i < byteArray.length; i++) {
                result += String.fromCharCode(byteArray[i]);
            }
            return result;
        }
        var baseAddr = Module.findBaseAddress('libbasic_share.so');
        console.log(`baseAddr:${ baseAddr }`);
        var keyFuncAddr = baseAddr.add(5083692);
        //nt_sqlite3_key_v2的地址
        Interceptor.attach(keyFuncAddr, {
            onEnter: function (args) {
                var nk = args[3].toInt32();
                var pk = args[2].readByteArray(nk);
                console.log('pKey--------->' + buf2str(pk));
            },
            onLeave: function (retval) {
            }
        });
    });
});
```

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733337-1568720065.png)

拿到的这个 key，非常符合 md5 的结果，前面又有 md5 导出函数，干脆就看一下?

先 hook 了`MD5DigestToBase16`，可以看到`aa030bae477df5ced6093d4702523a3a`和`d28951ee5178259c2916f46f37a14c30`，前者与数据库路径有关，后者就是数据库密钥

```
setImmediate(function () {
    function readStdString(str) {
        const isTiny = (str.readU8() & 1) === 0;
        // 检查是否为小字符串
        if (isTiny) {
            return str.add(1).readUtf8String();	// 读取小字符串
        }
        return str.add(2 * Process.pointerSize).readPointer().readUtf8String();	// 读取大字符串
    }
    Java.perform(function () {
        Interceptor.attach(Module.findExportByName('libbasic_share.so', '_ZN4xpng17MD5DigestToBase16ERKNS_9MD5DigestE'), {
            onEnter: function (args) {
            },
            onLeave: function (retval) {
                console.log('md5 ret----------->' + readStdString(retval));
            }
        });
    });
});
```

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733362-620440245.png)

然后再看一下 update 函数的参数，能够看到到底是计算了哪些，能看到密钥对应的参数是`c8d05c49e391e2e1d0bda579cc5250085sWvEGXu`，恰好是上一条`u_bv0QrXXA4YqAB3PaYr3PpQ`的 md5 结果，而前面路径中的值则和这个结果也有关联，是`md5(md5(u_bv0QrXXA4YqAB3PaYr3PpQ)+'nt_kernel')`，从而得到`aa030bae477df5ced6093d4702523a3a`，那么密钥参数`c8d05c49e391e2e1d0bda579cc5250085sWvEGXu`的后面几个字符串是哪来的？

```
setImmediate(function () {
    Java.perform(function () {
            Interceptor.attach(Module.findExportByName("libbasic_share.so",'_ZN4xpng9MD5UpdateEPA88_cPKhm'), {
                onEnter: function (args) {
                    console.log('update data ------->', args[1].readUtf8String(args[2].toInt32()));
                },
                onLeave: function (retval) {
                }
            });
    });
})
```

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733550-482426090.png)

居然都是在数据库里？

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733331-1389012134.png)

原来是在文件头写好的

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733566-698617533.png)

那么密钥的算法就是`md5(md5(u_bv0QrXXA4YqAB3PaYr3PpQ)+数据库文件头中的随机字符串)`，就是`0x1208`和`0x1a07`之间的字符串

那么现在`u_bv0QrXXA4YqAB3PaYr3PpQ`这一串又是什么呢？其实这个就是 QQ 的 uid，之前分析手机大师日志的时候，恰好看到了，从 mmkv（读取这玩意儿还不是个容易事儿，得通过腾讯开源的 mmkv/Python 去编译。。索性搓了一个解析工具，省的换机器要重新编译）中读取用户的 uid，实际上是通过 QQ 号（uin）去获取的

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733303-827812281.png)

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733358-167928804.png)

因此最终，数据库路径中的哈希 =`md5(md5(uid)+nt_kernel)`，密钥 =`md5(md5(uid)+文件头字符串)`

而要打开数据库的话，需要先把数据库的前 1024 字节删掉，然后进行解密，解密参数中，hmac 算法应该是文件头中的 sha1，kdf_iter 是 4000，其他都是 sqlcipher4 的配置

打开数据库后`40800`字段是聊天记录，内容是 protobuf 编码，`45101`是聊天内容

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733345-2006412541.png)

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733556-229171106.png)

ForensicsTool 集成了 ntqq 数据库解密

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733341-623595044.png)

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733557-266779650.png)

[mmkvReader](https://github.com/WXjzcccc/WXjzc-tool/tree/main/mmkvReader) 会同步上传至 GitHub

![](https://img2024.cnblogs.com/blog/2817142/202408/2817142-20240809190733366-1748622261.png)

想上传到之前刷到过的一个 github，hook 密钥就是从那学的，结果发现人家已经把算法搞出来了。。差距啊