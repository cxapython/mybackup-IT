> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/Moy0oSKQaKZ4sICKwCtZiA)

  

    之前有群友问 h5st，最近抽出时间来浅浅分析了一下。京东的 h5st 也是老朋友了，不管是不是搞电商业务的，肯定都多多少少有些了解，而且关于 h5st 的补环境文章很多，小破站上也有相关视频，这里我就不浪费时间了，咱们今天只分析算法。同时，这里卖个关子，发现个 h5st 的小 bug，可以直接绕开 h5st 校验，这个放到最后讲![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3MYwgXFeLCFTZ06xw2SNibbgI0iae52ZLjOaSoPvAvQL5XFCoEfEYhJpQ/640?from=appmsg#imgIndex=0)。开干！

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3WqZB4YOPOfZpL69rDTdl5A5ruDfkmWEvk8ia43qIfZuduATwdevZBDw/640?from=appmsg&watermark=1#imgIndex=1)

  

  

  

  

  

  

  

  

**申明**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/mcg0fttCKmSibVkhicchRWTTH90c1yuWKHamCI2NEVjb2PB09NwNUJvj1BKOb5qEB0mycZdbFfyHogiasYjuCcOvg/640?from=appmsg#imgIndex=2)

本文章中所有内容仅供学习交流使用，不用于其他任何目的，严禁用于商业用途和非法用途，否则由此产生的一切后果均与作者无关！若有侵权，请添加（wx：ShawYbo）联系删除

  

**目标网站**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/mcg0fttCKmSibVkhicchRWTTH90c1yuWKHamCI2NEVjb2PB09NwNUJvj1BKOb5qEB0mycZdbFfyHogiasYjuCcOvg/640?from=appmsg#imgIndex=3)

**网站**：

aHR0cHM6Ly9zZWFyY2guamQuY29tL1NlYXJjaD9rZXl3b3JkPeaJi+acug==

**目标**：h5st

  

**分析过程**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/mcg0fttCKmSibVkhicchRWTTH90c1yuWKHamCI2NEVjb2PB09NwNUJvj1BKOb5qEB0mycZdbFfyHogiasYjuCcOvg/640?from=appmsg#imgIndex=4)

    随便搜索一个商品，查看请求，跟栈，先找到 h5st 的生成入口位置。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3ib2S1MxzR4gb77uDkdsZw1XU98DG4c6HJ0sdXSXhOENHWzMxTXh6tvg/640?from=appmsg&watermark=1#imgIndex=5)

  

  

  

  

  

  

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic31cibA6fHqAJXatjQrjm4s2sKzsUbMNaAXg2oyvIEy3iciaRfsd9eD2n1A/640?from=appmsg&watermark=1#imgIndex=6)

  

  

  

  

  

  

  

    可以看到参数是通过 **js_security_v3_0.1.6.js** 生成。粗看一边这个 js，发现这个 vm 是通过 **for** 循环加 **switch-case** 控制，这种结构也很常见，比如阿里滑块，我们要做的依旧是找**函数调用**和**属性访问**。与之前遇到的此种类型结构不同点是，js 文件中有大量的函数是这种结构（for+switch），也就是说，每个函数都是一个小 vm，而且对于函数调用来说，不同参数量的调用都有与之对应的 case，这极大的增加了我们插桩的工作量，但是，我们依旧可以借助 ai 来帮我们插桩![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3Q4v6t1Ztn654Q8BcR7tnpuH3wq4GlsXf0Icne713DGvrP9GopP5VmQ/640?from=appmsg#imgIndex=7)。

PS: 直接告诉 cursor 对 xx 函数的所有 call 调用进行插桩即可。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3EiaWXjn9u0zf5wZiavsQdDcACxMuCOrWJMBNiaxl4x1X0yqOF9LcQ1GSw/640?from=appmsg&watermark=1#imgIndex=8)

  

  

  

  

  

  

  

    先来看一下 h5st 的构成。h5st 由多个部分组成，且每个部分参数固定。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3AAbIWzJDLe79VLnc2XWOTzuxiclUELFKpyjicb78ZlaVAYQ3Abje8DuA/640?from=appmsg&watermark=1#imgIndex=9)

  

  

  

  

  

  

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3Yia9nUxRN56icVtibQWkvrQzb79s1sOicbz40rm7iag1V3zuI8gIbqibYkQQ/640?from=appmsg&watermark=1#imgIndex=10)

  

  

  

  

  

  

  

  这里就不卖关子了，按照序号，我先依次说明一下各个参数作用，然后再分析是如何生成的。

*   **20260129175747017**：其实就是 Date.now()，转换为 yyyyMMddhhmmssSSS 格式就行。
    
*   **mi96aai6606wp3t8**：一个根据固定字符串而生成的随机字符串。
    
*   **f06cc**：appid
    
*   **tk05wd8592**...：defaultToken, 首次访问时，由 **mi96aai6606wp3t8** 等一系列参数生成。
    
*   **34bd29cbe5**...: 参数签名。
    
*   **5.2**：版本号。
    
*   **1769680664017**：这就是序号 1 参数的时间戳（20260129175747017）。
    
*   **eVxhk4BZpR**...：环境校验结果，这是重中之重。
    
*   **15fa059cc7**...：字符串'appid,body,client,.....'的签名。
    
*   **gRaW989Gy8**...：字符串'appid,body,client,.....'的自定义编码。
    

    先看一下 mi96aai6606wp3t8。mi96aai6606wp3t8 被存储在浏览器的 localStorage 中，key 为 **WQ_dy1_vk（WQ_dy1 这个前缀会变）**，我们可以拿来直接用，但我这里是分析他的生成过程，所以我们需要删掉他，并且 hook 一下这个值的 set，就可以找到具体的生成函数。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3ZaLgYV69aJqV7iaiaJHzkvH4icibWIzqT2ZWzs3dyVicpkibicjiaoWEhrtArw/640?from=appmsg&watermark=1#imgIndex=11)

  

  

  

  

  

  

  

    从日志中分析，可以得知一些逻辑。vk 生成的第一部是获取一个 4 位的字符串，该字符串来源于一个固定字符串：t6d0jhqw3p。这像是一种伪随机筛选，通过 Math.random() 和阈值比较（4）控制字符选择，小于 4 的被选中，最后按照特定的索引顺序（3,0,2,1）重新排列。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3bDtN9piaQDZN4OX4QA8ocibHSbhSksyyric3L0AuIWkZ6RMwmgsYHbiaicQ/640?from=appmsg&watermark=1#imgIndex=12)

  

  

  

  

  

  

  

    然后从原字符串中移除这些字符，从剩余字符集中随机生成 4 个和 7 个字符片段，将三者拼接后追加固定字符 "4"，最后将前 9 个字符反转并按 36 进制补码规则（35 - 原值）进行字符映射转换，与后 7 个字符拼接得到最终结果。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3gPnszSlr4tSiawpfxJ7uXjWdUvvAEQ1ufgkhF0X0ae9rdBx7cAZ6ia2A/640?from=appmsg&watermark=1#imgIndex=13)

  

  

  

  

  

  

  

```
def t6d0jhqw3p_to_result():
    original = "t6d0jhqw3p"
    selected_chars = []
    remain_to_select = 4
    for i, char in enumerate(original):
        if remain_to_select == 0:
            break
        r = random.random()  # 随机数决定是否选择
        threshold = r * (len(original) - i)
        if threshold < remain_to_select:
            selected_chars.append(char)
            remain_to_select -= 1
    for i in range(len(selected_chars) - 1, 0, -1):
        r_idx = random.random()
        j = math.floor(r_idx * (i + 1))
        selected_chars[i], selected_chars[j] = selected_chars[j], selected_chars[i]
    q3hj = ''.join(selected_chars)[::-1]
    temp_str = original
    for char in q3hj:
        temp_str = temp_str.replace(char, "", 1)
    param = {
        "size": 4,
        "num": temp_str
    }
    part_060t = generate_random_string(param)
    param = {
        "size": 7,
        "num": temp_str
    }
    part_3 = generate_random_string(param)
    part_dwdpw0p = part_060t + q3hj + part_3
    intermediate = part_dwdpw0p + "4"
    
    char_to_num = {
        '0': 0, '1': 1, '2': 2, '3': 3, '4': 4, '5': 5,
        '6': 6, '7': 7, '8': 8, '9': 9, 'a': 10, 'b': 11,
        'c': 12, 'd': 13, 'e': 14, 'f': 15, 'g': 16, 'h': 17,
        'i': 18, 'j': 19, 'k': 20, 'l': 21, 'm': 22, 'n': 23,
        'o': 24, 'p': 25, 'q': 26, 'r': 27, 's': 28, 't': 29,
        'u': 30, 'v': 31, 'w': 32, 'x': 33, 'y': 34, 'z': 35
    }
    # 前9个字符
    front_part = intermediate[:9]
    back_part = intermediate[9:]  # 后7个字符保持不变
    # 转换前9个字符
    converted_front = []
    num_to_char = {v: k for k, v in char_to_num.items()}
    reversed_front_part = front_part[::-1]
    for char in reversed_front_part:
        num = char_to_num.get(char, 0)
        converted_front.append(num_to_char[(35-num)])
    # 合并结果
    result = ''.join(converted_front) + back_part
    return result
```

    以上就是 vk 的生成逻辑，按照这个思路，我们继续分析 **eVxhk4BZpR** 环境校验结果，探究一下他是如何检验环境和加密的。

    查看日志或者直接搜索 clt。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3vwjthia9qwfaPBAu3s2IibP00Pt3KPh4q2jChIcCTxMch9LjpruMCLoA/640?from=appmsg&watermark=1#imgIndex=14)

  

  

  

  

  

  

  

```
{
  "sua":"Windows NT 6.1; Win64; x64; rv:136.0",
  "pp":{
    "p2":"jd_7489ced5d8940"
  },
  "extend":{
    "wd":0,
    "l":0,
    "ls":5,
    "wk":0,
    "bu1":"0.1.6",
    "bu3":52,
    "bu4":0,
    "bu5":0,
    "bu6":17,
    "bu7":0,
    "bu8":0,
    "random":"3012d4o9jMQH",
    "bu12":-8,
    "bu10":14,
    "bu11":2
  },
  "pf":"Win32",
  "random":"Egn20XhHC47",
  "v":"h5_file_v5.2.7",
  "bu4":"0",
  "canvas":"987969e34d38fed58836c965ea179a2a",
  "webglFp":"941d58d0cb7126b5f348a788f7aa0760",
  "ccn":32,
  "fp":"igtz6wzwwqdw3tp7"
}
```

    先来解释一下各个参数含义：

    sua 就不说了；pp 为账号名称；extend 这是环境校验结果，稍后插桩分析各个值的含义；pf、random、v、canvas、webglFp、ccn 这些都是见名知意，除了 random（11 为位随机字符）写死即可。fp 可不是 fingerprint，fp 就是咱们上一步的 vk，这里有点混淆视听![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3KiaHZaQ3gnwAykmoKAN5a4t296KSXd2ZvnicvXaH1fj8j60fRZC35X3w/640?from=appmsg#imgIndex=15)。

    关于 extend 的生成，因为京东的更新频率也很高，js 会经常变化，所以很难用关键词搜索，这一块需要根据 clt 的日志，先进入环境校验的父函数，然后 debug 再找到具体的 extend 生成函数。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3My5uduChJbcP4myHzwXuGIZkhuowxumuc8mZN2LibicpSwhWOyoGRX1w/640?from=appmsg&watermark=1#imgIndex=16)

  

  

  

  

  

  

  

    extend 的检测点还是很多的。

自动化工具检测类：

*   wd：检测是否存在 WebDriver 属性，用于识别自动化浏览器。
    
*   wk：检测 User-Agent 中是否包含 headlessChrome，或是否为 Selenium、PhantomJS、TestCafe 等自动化工具运行环境。
    
*   bu5：通过调用堆栈信息判断是否包含 playwright、node 等关键字，用于识别自动化框架。
    

浏览器环境特征检测：

*   l：Navigator.languages，浏览器语言设置数组，检测语言偏好配置。
    
*   ls：Navigator.pluginArray.length，检测已安装的插件数量。
    
*   bu3：document.head.childElementCount，head 元素的子元素数量，检测页面头部结构。
    
*   bu6：document.body.childElementCount，body 元素的子元素数量，检测页面主体结构.
    
*   bu8：HTMLAllCollection，验证 DOM 完整性。
    

环境完整性检测：

*   bu4：window.window === window，检测 window 对象的自引用完整性。主要针对 node 补环境。
    

函数保护检测：  

*   bu7：使用正则表达式判断函数的 toString() 方法，检测函数保护 function toString(){[native code]} 是否被篡改。
    

固定值：

*   bu1：固定值 0.1.6，可以理解为检测版本号。
    
*   bu10：14。
    
*   bu11：2。
    
*   bu12：时区，偏移值 - 8，表示 UTC+8（中国时区）。
    

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3AW85CibtfuXoQiaWcwl0DxBhILTnIqOXR6qZ0t4K9IDv9xSFic0uej1xQ/640?from=appmsg&watermark=1#imgIndex=17)

  

  

  

  

  

  

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic36QW39icYFKLI9xwcXYJlo3uL8eJ7o3hdFyv2CfyHtXicgcmlQ3Cic76uA/640?from=appmsg&watermark=1#imgIndex=18)

  

  

  

  

  

  

  

    ok，至此，关于环境校验的各个字段我们都已弄清他的含义，接下来就是要对其自定义编码。查找他的编码逻辑。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3ttWaAoDIibVYKicIqCYrkibuXL2mW5XtPjeOOvk4kuW40XeQiaVF8sKd3A/640?from=appmsg&watermark=1#imgIndex=19)

  

  

  

  

  

  

  

    这一块逻辑分析起来确实耗时，日志粒度细一点，虽然日志很长，但大多是循环结果，所以分析循环内的一到两组日志便可推理出结果。整体逻辑就是采用 base64 自定义编码集，然后又进行了一系列的反转重组操作：

*   第一步：字节数组填充：将输入字符串转换为 Uint8Array，按 3 的倍数填充（填充值为填充数量，若已为 3 的倍数则填充 3 个 3），确保长度是 3 的倍数。
    
*   第二步：字节重排序：将字节数组按每 3 个字节为一组，从后往前倒序处理，每组内按 [i-2, i-1, i] 的顺序重新排列，打乱原始字节顺序。
    
*   第三步：自定义 Base64 编码：使用自定义字符集 'hgfedcbaZYXWVUTSRQPONMLKJIHGFEDCBA-_9876543210zyxwvutsrqponmlkji'（共 64 个字符，顺序与标准 Base64 不同），将重排序后的每 3 个字节组合成 24 位整数，按 6 位分段得到 4 个索引，映射到自定义字符集；对于不足 3 字节的组，用'='填充。
    
*   第四步：结果重组，将编码结果按每 4 个字符为一组，从后往前切片，每组内部字符反转，然后整体反转并拼接，形成最终编码字符串。
    

```
// 将所有转换步骤合并为一个函数
function encodeToCustomBase64(str) {
    const chars = 'hgfedcbaZYXWVUTSRQPONMLKJIHGFEDCBA-_9876543210zyxwvutsrqponmlkji';
    // 1. 将字符串转换为Uint8Array并进行填充
    const bytes = new Uint8Array(str.length + (3 - str.length % 3) % 3);
    for (let i = 0; i < str.length; i++) bytes[i] = str.charCodeAt(i) & 0xFF;
    const fillCount = bytes.length - str.length || 3;
    bytes.fill(fillCount, str.length);
    // 2. 字节数组重新排序
    const reordered = [];
    for (let i = bytes.length - 1; i >= 0; i -= 3) {
        reordered.push(bytes[i - 2], bytes[i - 1], bytes[i]);
    }
    // 3. 自定义Base64编码
    let result = '';
    for (let i = 0; i < reordered.length; i += 3) {
        const triple = (reordered[i] << 16) | ((reordered[i + 1] || 0) << 8) | (reordered[i + 2] || 0);
        result += chars[(triple >> 18) & 0x3F] +
                  chars[(triple >> 12) & 0x3F] +
                  (i + 1 < reordered.length ? chars[(triple >> 6) & 0x3F] : '=') +
                  (i + 2 < reordered.length ? chars[triple & 0x3F] : '=');
    }
    // 4. 最终字符串重组
    return Array.from(
        { length: Math.ceil(result.length / 4) },
        (_, i) => result
            .slice(Math.max(0, result.length - (i + 1) * 4), result.length - i * 4)
            .split('')
            .reverse()
            .join('')
    )
    .reverse()
    .join('');
}
```

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3icnpO8iaGygO4rKvnLchFuT9KDmkXjib8crfJbt2bvQyFcdxfLSgdibsCg/640?from=appmsg&watermark=1#imgIndex=20)

  

  

  

  

  

  

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3EMBs9ibicR1rPNKyFo5F15bW4ua2Kwb9jg78gVopUT2qbfwe71wsxZMQ/640?from=appmsg&watermark=1#imgIndex=21)

  

  

  

  

  

  

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3FVYzknEOybhDGhvYR8l2OPfsCLwQSUMCPDYpoyO43wny94kkByb0TA/640?from=appmsg&watermark=1#imgIndex=22)

  

  

  

  

  

  

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3ILpUU11OEATO7VoRick7JwicrSs62uBXia0lUWgEo5dCbibFuPqGdHLOPg/640?from=appmsg&watermark=1#imgIndex=23)

  

  

  

  

  

  

  

    至此，关于环境校验这一块就算分析完了，剩下的其他参数，由于篇幅原因，且分析思路都大同小异，这里就不展开讨论了。

PS：

*   如果有补环境的，其实可以在本地运行，将所有日志保存至文件，然后交给 ai，只需要日志打印粒度细一点即可，ai 几乎能把整个生成逻辑的 95% 都挖掘出来，剩下的就是慢慢调试就好了。
    
*   京东的风控还是很严的，调试过程中，有概率会喜提七天封号套餐，望知悉。
    
*   重中之重，开头咱们卖了个关子，就是说，如果你把 h5st 设置为 undefined。就可以直接拿到结果，这可能是风控层面一个漏洞。
    

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic33KhE0iayKXLu8AYtvLvwjDgoiaNmrzZjjLqAj57EHylE3ScKnzmzibTUw/640?from=appmsg&watermark=1#imgIndex=24)

  

  

  

  

  

  

  

  

  

**总结**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/mcg0fttCKmSibVkhicchRWTTH90c1yuWKHamCI2NEVjb2PB09NwNUJvj1BKOb5qEB0mycZdbFfyHogiasYjuCcOvg/640?from=appmsg#imgIndex=25)

  

  

  

  

  

     h5st 还是有点点强度的，算是个中等级别的 vmp。这个强度主要来源于他把各个功能都独立拆解为小的 vm，而不是像其他 vm 一样，将所有操作码和操作数放入字节码中，这大大增加了我们插桩的工作量（这难道不就是人家反爬的意义？）。而且所有的算法，包括我们文章没有讲到的几个 sign 值的生成，用的全是自定义算法，并非标准的编码或者 hash 算法，所以这一块也比较考验日志分析的认真程度了。

    最后感谢各位大佬的阅读和支持，如果可以的话，点个小心心![](https://res.wx.qq.com/t/wx_fed/we-emoji/res/assets/Expression/Expression_67@2x.png#imgIndex=26)哈。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3UgFT2kejv0c3gspLJQAyreGkeicBA9ZJEtkyicYyS0eubbMIibicAvQB1w/640?from=appmsg&watermark=1#imgIndex=27)

  

  

  

  

  

  

  

**END**

![](https://mmbiz.qpic.cn/mmbiz_gif/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3JxaQ4KC0Jt2vWS8xI9YH67Nl2nGZcFdjXPkINP0mCsviaraeibXgyoAA/640?from=appmsg#imgIndex=28)

![](https://mmbiz.qpic.cn/mmbiz_jpg/97ib7mrowjiaxWLGByvsyGiaRxEPYMdJ3ic3CmK2cMcovxrlzxZ0tdey57B47ib6Yrt1mWibBAZwvjo8QBGxMpj0S4BQ/640?from=appmsg&watermark=1#imgIndex=29)

  

**窥破表象**

  

**方见本源**