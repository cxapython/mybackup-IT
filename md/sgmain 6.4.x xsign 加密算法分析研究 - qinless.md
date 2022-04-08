> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.qinless.com](https://www.qinless.com/1708)

> 仅供学习研究 。

> `sgmain 6.4.x` 版本的 `x-sign` 参数加密算法研究分析  
> 样本 `tianmao-8.11.0`

本文主要使用 `ida + unidbg` 动静态分析

样本 `unidbg` 参考文章

*   [某宝系 tb tm sgmain x-sign 分析 - unidbg](https://www.qinless.com/179)

该算法有白盒 `aes`，会使用 `dfa` 攻击获取密钥

参考文章

*   [第一讲——从黑盒攻击模型到白盒攻击模型](https://www.qinless.com/1642)
    
*   [密码学学习记录｜aes dfa 练习样本一](https://www.qinless.com/1698)
    

**_Tips: 博主也是 dfa 的初学者，文中有描述错误的情况，还望各位大佬轻喷_**

**_因为 so 对抗的原因，ida 看不到 c 代码，所以会使用 unidbg 大量 trace。trace 的过程很心酸，这一块不会写的很详细_**

之前有位大佬发了篇 `x-sign` 的加密流程（已经被删），我也是基于此来的灵感，大概的流程是

**_hmac_sha1 -> white_aes -> base64 -> hmac_sha1_**

基于此流程开始分析

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f91b128853.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f91b128853.png)

这里使用龙哥的 `findhash` 跑一下，定位到了 `sha1` 特征函数

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f91b672985.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f91b672985.png)

查看函数的交叉引用，往上跟踪，来到这个函数。这里就是 `hmac` 函数了，稍后在这里 `debugger` 一下，看看输入跟 `key`

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f91ba2d7fa.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f91ba2d7fa.png)

这里先把输入随便改一下

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f91bfcea9b.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f91bfcea9b.png)

然后在 `hmac` 函数处下个断点

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f91c461bfb.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f91c461bfb.png)

成功断下来，参数四是输入，参数三看起来是 `hmac key` 稍后试一下

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f91c8a8860.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f91c8a8860.png)

函数执行完查看输出，这里就是 `hmac sha1` 的加密结果了

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f91cce3617.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f91cce3617.png)

使用逆向之友加密，结果相同，说明是标准的 `hmac sha1` 函数了

**_`hmac sha1` 算法分析出来之后，就要开始使用 `unidbg trace` 大法，追踪了_**

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f91d1bdc74.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f91d1bdc74.png)

直接 `traceRead` 这个地址的 20 个字节数据，代码跑起来发现没读取

`unidbg 0xbfff` 地址开头是默认是栈，所以数据可能在堆里，去搜索一下

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f91d7cffa9.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f91d7cffa9.png)

使用 `shr` 搜索可读堆（注意后面会经常用，就不再解释了），发现数据在 `0x4020d600` 地址里也有的

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f91e3537e5.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f91e3537e5.png)

这里改改地址，跑起来，看到有 `trace` 结果了

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f92119d212.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f92119d212.png)

`ida` 跳转过来，也看不懂是个啥，下面写个代码 `hook` 一下看看

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f92166864a.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f92166864a.png)

通过胡乱一通调试，修改后最终的 `inlinehook` 代码如上

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f921de72fe.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f921de72fe.png)

`trace` 之后结果又写入到 `r2` 的地址了

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f922178d1a.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f922178d1a.png)

看看 `r2` 的地址，好吧这就是 `hmac sha1` 的结果转成了 `hex`，继续 `trace`

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f922b89be8.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f922b89be8.png)

好消息 `trace` 又没结果，堆找下，结果还挺多

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f923073e4e.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f923073e4e.png)

这里发现了有趣的地方，这不就是 `hmac_sha1 + & + hmac_sha1_key` 吗

重要的是后面的 `07` 这是啥 ？，这不妥妥的就是 `pkcs7` 填充的特征吗，很明显是 `aes` 的输入

所以果断放弃前面跟踪的结果，拥抱新欢，开始跟踪这个

先别高兴的太早，输入有 `73 byte`，填充 `7 byte`（太长了 = =），使用 `dfa` 攻击太麻烦了，所以需要先把输入改成一组也就是 `16 byte` 以下（`dfa` 不了解的去看看前面推荐文章，到这里默认了解）

### unidbg 修改 aes 输入长度

继续需要 `trace` 这些数据哪来的

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f9238ad062.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f9238ad062.png)

`trace` 发现在 `libc.so` 里写入的，猜测是 `memcpy` 函数，有兴趣的小伙伴可以去看看这个 `so` 的函数，这里是在 `libc.so` 操作的那我们咋整呢，直接 `hook` 也不太现实，那没办只好去 `hook` 后面的 `lr` 地址了

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f923c60ffe.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f923c60ffe.png)

这里 `hook` 成功，查看调用栈

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f9241bf300.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f9241bf300.png)

这么多，一个一个看，其中一个地址在这里，抱着怀疑心情 `hook` 一下这里，发现该函数的参数并没有我们想要的数据，那可能就不是在这里，那后面咋整呢

博主也是卡了挺久，这里就直接说答案了，大家可以自行测试分析一下，入口就在上图的 `sub_b46dc` 函数，`hook` 一下

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f92488b129.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f92488b129.png)

`r0` 正确，`r1` 刚好是 `r0` 的长度，直接在这里修改即可

具体咋修改参考龙哥的 `unidbg hook 大全`

修改完成咋看有没有修改成功呢，第一可以通过内存地址，第二可以通过 `hmac sha1` 参数

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f924c67d93.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f924c67d93.png)

在第二次执行到 `hmac sha1` 查看第三个参数，就是 `aes -> base64` 之后的结果，正常应该是这么长的

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f9252072a8.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f9252072a8.png)

修改之后的结果就只有这么长了，所以以上的逻辑是可行的

修改了输入就需要确定 `aes` 有没有 `iv` 干扰了，还要确定循环的轮数在哪

### unidbg 分析 aes iv

这个如何分析呢，正常 `aes cbc` 模式，输入会先跟 `iv` 异或，所以直接 `trace` 输入

修改输入之后发现之前的 `trace` 没有了，那我继续搜索堆

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f925720b2f.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f925720b2f.png)

这里的 `090806` 是我修改的，发现在 `0x4020d618` 地址里，继续 `traceRead`

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f925cac0ff.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f925cac0ff.png)

`trace` 成功，`ida` 跳过去看看

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f9262688c6.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f9262688c6.png)

这里看下 `arm` 就可以了，取值异或存储，下面使用 `unidbg inlinehook` 看看

```
emulator.getBackend().hook_add_new(new CodeHook() {
    @Override
    public void hook(Backend backend, long address, int size, Object user) {
        if (address == (base + 0x6AE84)) {
            Arm32RegisterContext ctx = emulator.getContext();

            int r1 = ctx.getR1Int();
            int r2 = ctx.getR2Int();
            int res = r1 ^ r2;

            System.out.println(
                    "eors r1, r2 --- " +
                            "r1:" + Long.toHexString(r1) + ", " +
                            "r2:" + Long.toHexString(r2) + ", " +
                            "res:" + Long.toHexString(res)
            );
        }
    }

    @Override
    public void onAttach(UnHook unHook) {
    }

    @Override
    public void detach() {
    }
}, base + 0x6AE84, base + 0x6AE84, null);
```

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f926a81bbe.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f926a81bbe.png)

`r1` 是我们的输入，`r2` 就是 `iv` 了

找到了 `iv` 那就需要，就需要去分析轮数在哪里，分析轮数之前最好找到 `state` 这样可以更快速的定位到轮数，正常 `state` 的第一轮，`data ^ iv` 结果会跟 `key` 进行异或，但这是白盒 `aes` 是直接查表操作，没啥好的办法 = = 还是只能硬着头皮进行 `trace` 了

### unidbg 分析 aes state

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f926f983f0.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f926f983f0.png)

根据 `data ^ iv` 的结果，搜索堆，有结果，开始 `trace`

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f92753368f.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f92753368f.png)

`ida` 跳过去看看

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f927a2becb.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f927a2becb.png)

仔细看看就会发现，数据取出来 给到 `r5` 啥都没干，在存起来，有点离谱，`debugger` 看一下

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f928011ee4.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f928011ee4.png)

写到这个栈里了，目前不知道干啥在 `trace` 一波

在控制台输入 `traceWrite [0xbffff028 0xbffff038]` 这个会把结果写到文件里

打开文件分析一波

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f92866130e.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f92866130e.png)

搜索 `0xbffff028` 可以看到有写入 `data ^ iv` 的结果

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f928ac7373.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f928ac7373.png)

[![](https://www.qinless.com/wp-content/uploads/2022/04/624f929134b5f.png)](https://www.qinless.com/wp-content/uploads/2022/04/624f929134b5f.png)

再往下跟踪看到了写入 `aes` 的加密结果，所以猜测这个地址就是 `state`

好了这一篇就到了这里，后面就是 `dfa` 注入攻击了

偷懒了（主要还是因为太多了，有点写不下去了）