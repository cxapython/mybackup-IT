> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [www.52pojie.cn](https://www.52pojie.cn/thread-1908744-1-1.html)

> [md]## 前言 > ** 本案例中所有内容仅供个人学习交流，抓包内容、敏感网址、数据接口均已做脱敏处理，严禁用于商业用途和非法用途，否则由此产生的一切后果均与作者无 ... 【JavaScript 逆向......

![](https://avatar.52pojie.cn/images/noavatar_middle.gif)Behind1 _ 本帖最后由 Behind1 于 2024-4-1 22:04 编辑_  

前言
--

> **本案例中所有内容仅供个人学习交流，抓包内容、敏感网址、数据接口均已做脱敏处理，严禁用于商业用途和非法用途，否则由此产生的一切后果均与作者无关！**

一、抓包分析
------

![](https://attach.52pojie.cn/forum/202404/01/215423ifrufl889rf8f8bf.jpg)

**1711979678912.jpg** _(144.36 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY4Njc5MHwwNWI1YjVhMHwxNzEyNDY3MDU0fDB8MTkwODc0NA%3D%3D&nothumb=yes)

1

2024-4-1 21:54 上传

`send_code` 触发验证码  将返回 detail 用于下个请求获取图片

`captcha/get`  通过 detail 获取图片， id  ， tip_y（这个后面轨迹需要用到，最开始一直没找到这个值）

`captcha/verify` 通过轨迹生成的`captchaBody` 进行校验

流程比较简单，最头疼的是 **jsvmp**，还有轨迹

二、mobile 加密
-----------

直接进 js 抠出来就完事了，这里直接用 Python 改写

```
def mobile_encode(t):
    """
    异或运算（XOR）
    """
    t = '+86 ' + t

    def encode_char(c):
        code = ord(c)
        if 0 <= code <= 127:
            return [code]
        elif 128 <= code <= 2047:
            return [192 | 31 & code >> 6, 128 | 63 & code]
        else:
            return [224 | 15 & code >> 12, 128 | 63 & code >> 6, 128 | 63 & code]

    encoded = [encode_char(char) for char in str(t)]

    result = []
    for byte_list in encoded:
        for byte in byte_list:
            result.append(hex(5 ^ byte)[2:])

    return "".join(result)


```

三、底图还原
------

打开开发者工具可以算出 返回乱序图片和还原后的图片的**缩放比**

![](https://attach.52pojie.cn/forum/202404/01/215536iri4jzziam3m4rzj.jpg)

**1711979746723.jpg** _(445.47 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY4Njc5MXwxY2FhNjVlYXwxNzEyNDY3MDU0fDB8MTkwODc0NA%3D%3D&nothumb=yes)

2024-4-1 21:55 上传

![](https://attach.52pojie.cn/forum/202404/01/215649h8prvbblw9zy5tkw.jpg)

**1711979818543.jpg** _(511.83 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY4Njc5MnxhMzI3NjkwOHwxNzEyNDY3MDU0fDB8MTkwODc0NA%3D%3D&nothumb=yes)

2024-4-1 21:56 上传

图片还原可以通过`canvas`断点找到 js 还原图片的方法

这里仅竖着切割，我们可以不用去扣 js   顺序固定的`[4, 0, 3, 5, 2, 1]`

直接改写 Python 实现还原

```
def parse_bg_captcha(img, save_path=None):
    """
    滑块乱序背景图还原
    还原前 h: 344, w: 552
    还原后 h: 212, w: 340
    :param img: 图片路径str/图片路径Path对象/图片二进制
        eg: 'assets/bg.webp'
            Path('assets/bg.webp')
    :param save_path: 保存路径, <type 'str'>/<type 'Path'>; default: None
    :return: 还原后背景图 RGB图片格式
    """
    if isinstance(img, (str, Path)):
        _img = Image.open(img)
    elif isinstance(img, bytes):
        _img = Image.open(io.BytesIO(img))
    else:
        raise ValueError(f'输入图片类型错误, 必须是<type str>/<type Path>/<type bytes>: {type(img)}')

    # 定义切割的参数
    cut_width = 92
    cut_height = 344
    k = [4, 0, 3, 5, 2, 1]

    # 创建新图像
    new_img = Image.new('RGB', (_img.width, _img.height))

    # 按照指定顺序进行切割和拼接
    for idx in range(len(k)):
        x = cut_width * k[idx]
        y = 0
        img_cut = _img.crop((x, y, x + cut_width, y + cut_height))  # 垂直切割
        new_x = idx * cut_width
        new_y = 0
        new_img.paste(img_cut, (new_x, new_y))

    # 调整图像大小
    # new_img = new_img.resize((340, 212))
    if save_path is not None:
        save_path = Path(save_path).resolve().__str__()
        new_img.save(save_path)

    img_byte_array = io.BytesIO()
    new_img.save(img_byte_array, format='PNG')
    img_byte_array = img_byte_array.getvalue()

    return img_byte_array

```

四、captchaBody
-------------

jsvmp 代码结构

![](https://attach.52pojie.cn/forum/202404/01/215809epaji2jge21gn212.jpg)

**1711979901638.jpg** _(47.07 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY4Njc5M3wwNTkzMmRlZHwxNzEyNDY3MDU0fDB8MTkwODc0NA%3D%3D&nothumb=yes)

2024-4-1 21:58 上传

jsvmp 一般采取补环境（是个 webpack ）和插桩本文主要讲解插桩日志分析他的算法

### 插桩分析

有很多个 vmp 需要每个 for 循环处进行插桩，还有关键的运算位置需要单独打印出来  

还有`fromCharCode`  `charAt`   `charCodeAt`都需要打印，有完整的日志才能更快更准的分析出。

```
console.log('待加密———>',m,'\n加密————>',b)

```

![](https://attach.52pojie.cn/forum/202404/01/215903vsbw1wfwsf91y4ww.jpg)

**1711979941331.jpg** _(161.24 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY4Njc5OHxiZDJkZTY4OHwxNzEyNDY3MDU0fDB8MTkwODc0NA%3D%3D&nothumb=yes)

2024-4-1 21:59 上传

日志中你会看到 aes 和 sha512 算法

![](https://attach.52pojie.cn/forum/202404/01/215947yorg99t1ooyjbbog.jpg)

**1711979984684.jpg** _(30.32 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY4Njc5OXxhMTA2MDcwOHwxNzEyNDY3MDU0fDB8MTkwODc0NA%3D%3D&nothumb=yes)

2024-4-1 21:59 上传

下面这段 js 你可能会需要用到

```
function hexStringToArrayBuffer(hexString) {
    // remove the leading 0x
    hexString = hexString.replace(/^0x/, '');

    // ensure even number of characters
    if (hexString.length % 2 != 0) {
        console.log('WARNING: expecting an even number of characters in the hexString');
    }

    // check for some non-hex characters
    var bad = hexString.match(/[G-Z\s]/i);
    if (bad) {
        console.log('WARNING: found non-hex characters', bad);   
    }

    // split the string into pairs of octets
    var pairs = hexString.match(/[\dA-F]{2}/gi);

    // convert the octets to integers
    var integers = pairs.map(function(s) {
        return parseInt(s, 16);
    });

    var array = new Uint8Array(integers);
    console.log(array);

    return array.buffer;
}

```

![](https://attach.52pojie.cn/forum/202404/01/220016p31ghn9neb1bwzny.jpg)

**1711980025649.jpg** _(633.04 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY4NjgwMHwxYmM2MmVkN3wxNzEyNDY3MDU0fDB8MTkwODc0NA%3D%3D&nothumb=yes)

2024-4-1 22:00 上传

### 轨迹

轨迹这里可以自己打上 log 去看，鼠标，滚动条

这里说一下这个参数

> "drag":  
> 
> "x": 144,  //  边框 20 + 滑动条 64.5 也就是 x∈[20-84,20 + 滑块滑动距离 - 84 + 滑动距离]  
> "y": 287,  // 整个滑块窗口到滑动条距离 y∈[285,325]

```
{
    "reply": "30h5Yr0hM",   
    "models": "pD6zZEe4",
    "in_modal": "xCiAyhLse",
    "in_slide": "vi5C7jc",
    "move": "8uRj",
    "touch": "ljA42",
    "drag": "YuLFijT",
    "crypt_salt": "1b4d7416e175dc54380b876c721e4af5b406e7a1700ff3118beb029635ba2fe26c794112bb938d17517af1fc36db87b7f43db04ebdc305d5e30f2ba47a184366"
}

```

动态加密, 目前还没有想到如何快速拿到映射。

```
code:500,msg:"VerifyErr" //滑块缺口距离错误
code:500,msg:"VerifyModelErr" //滑块位置正确轨迹有问题

```

五、结果展示
------

这里就不讲太多了，有问题可以私信我。  
![](https://attach.52pojie.cn/forum/202404/01/220133pn77277b8xnx77b2.jpg)

**1711980097639.jpg** _(84.09 KB, 下载次数: 0)_

[下载附件](forum.php?mod=attachment&aid=MjY4NjgwMXw1N2M3MTBiZnwxNzEyNDY3MDU0fDB8MTkwODc0NA%3D%3D&nothumb=yes)

2024-4-1 22:01 上传![](https://avatar.52pojie.cn/images/noavatar_middle.gif)Behind1

> [smartfind 发表于 2024-4-2 09:30](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=49982128&ptid=1908744)  
> 如何得出轨迹坐标的，还请解惑。

在浏览器打上日志点  多滑几下 对比坐标变化（一般都是有规律的），当然也可以用去细扣 js![](https://avatar.52pojie.cn/images/noavatar_middle.gif)policewang 看了个寂寞 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) Lty20000423

> [policewang 发表于 2024-4-2 04:23](https://www.52pojie.cn/forum.php?mod=redirect&goto=findpost&pid=49979402&ptid=1908744)  
> 看了个寂寞

研究算法的可以看看![](https://avatar.52pojie.cn/images/noavatar_middle.gif)xjcyxyx 优秀，收藏学习 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) jinzhu160 好高深，这种底层原理讲的多一点挺好。![](https://avatar.52pojie.cn/images/noavatar_middle.gif)BigMon  
优秀，收藏学习 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) kexing 厉害  看了一头雾水  还得努力学习 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) doubleKill 看了一头雾水 ![](https://avatar.52pojie.cn/images/noavatar_middle.gif) doubleKill 图像还原明白，但请问是如何算出轨迹坐标的？![](https://avatar.52pojie.cn/images/noavatar_middle.gif)smartfind 如何得出轨迹坐标的，还请解惑。