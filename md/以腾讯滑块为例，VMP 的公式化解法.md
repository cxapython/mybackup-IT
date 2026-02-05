> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [mp.weixin.qq.com](https://mp.weixin.qq.com/s/0gQ-6JU4hnNjcWoUmUGUtQ)

  

腾讯滑块也是 VMP，该样本难度不高，适合练手，而且关于腾讯滑块的文章很多，我这里也冷饭热炒一下，依旧是**公式化解法**：**找关键点（函数调用、属性访问）、插桩、补环境、分析算法。**

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiawgkIROwfezfu1csYH3SkAohDT5rwjnLDG370DQquQBgmAibX2W7wMSOic9KFG0G5UZLiadDPR3B7nXQ/640?from=appmsg&watermark=1#imgIndex=0)

  

  

  

  

  

  

  

  

**申明**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/DeUVC3rzZGmpc0hgWFVpoFBx1emBfQo7IUk2Ess9Biad4c1wJyDib0uLJwEFpWcqpsrpl5PicaScUYN7Wd5VCwUibA/640?from=appmsg#imgIndex=1)

本文章中所有内容仅供学习交流使用，不用于其他任何目的，严禁用于商业用途和非法用途，否则由此产生的一切后果均与作者无关！若有侵权，请添加（wx：ShawYbo）联系删除

  

**目标网站**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/DeUVC3rzZGmpc0hgWFVpoFBx1emBfQo7IUk2Ess9Biad4c1wJyDib0uLJwEFpWcqpsrpl5PicaScUYN7Wd5VCwUibA/640?from=appmsg#imgIndex=2)

**网站**：

aHR0cHM6Ly9jbG91ZC50ZW5jZW50LmNvbS9wcm9kdWN0L2NhcHRjaGE=

**目标**：验证请求参数 collect

  

**分析过程**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/DeUVC3rzZGmpc0hgWFVpoFBx1emBfQo7IUk2Ess9Biad4c1wJyDib0uLJwEFpWcqpsrpl5PicaScUYN7Wd5VCwUibA/640?from=appmsg#imgIndex=3)

    整体请求链路干净整洁，就两个请求，第一个请求 **cap_union_prehandle**，获取初始化信息（js 地址、滑块图片地址）；第二个请求 **cap_union_new_verify**，提交结果。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiawgkIROwfezfu1csYH3SkAoZPPicGhrbIrwJ7hicER5jLB30L2nM5FPibzOlnkSHtIZcJ7XPWBAmaKrA/640?from=appmsg&watermark=1#imgIndex=4)

  

  

  

  

  

  

  

    先看第一个请求 /**cap_union_prehandle**，参数大多为固定值，没什么好分析的，直接看响应。响应中有几个关键值需要留一下吗，sess 为验证时携带的 ticket，pow_cfg 为 pow 验算时的 nonce 和 md5，bg_elem_cfg 和 fg_elem_cfg 为底图和滑块的下载地址和图片属性，滑块 fg_elem_cfg 中 size_2d 是裁切尺寸，sprite_pos 为裁切位置，如下两张图。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiawgkIROwfezfu1csYH3SkAolCXULrBxpDj45KC4R5cYqOKYLvNuDGcxWNxCSAibpE1SmWhGZ6vZtMA/640?from=appmsg&watermark=1#imgIndex=5)

  

  

  

  

  

  

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiawgkIROwfezfu1csYH3SkAoX7Xe9QjtKvJGqabFRbjZb9GibZof1MDQNooPIFMH6Fy5iao2uD9GbicjA/640?from=appmsg&watermark=1#imgIndex=6)

  

  

  

  

  

  

  

    然后看第二个请求 /**cap_union_new_verify**，参数 **collect** 就今天的主角，稍后我们详细分析；tlg 为 collect 的长度；eks 是主角 tdc.js 中一个动态字符串，可直接获取；sess 就上一步响应的；ans 为滑块对齐的位置；pow_answer 是 pow 计算的结果；pow_calc_time 为 pow 计算消耗时长。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiawgkIROwfezfu1csYH3SkAoNTm9zibBMXicWYhUshYU5umSTJBjn1mSSnicQpDOgJvician53NeFth57gw/640?from=appmsg&watermark=1#imgIndex=7)

  

  

  

  

  

  

  

    pow_answer 是 pow 计算结果，关于 **pow(工作量证明)**，之前的文章有提到过，简言之，就是对服务端提供的 md5 和 nonce 进行暴力搜索的过程：客户端接收服务端下发的随机数 nonce 和目标哈希 target，通过暴力搜索一个数字后缀 suffix，对拼接后的字符串 **workload_nonce + suffix** 进行 **MD5** 哈希计算，将得到的哈希值与 workload_target 进行比较，直到找到完全匹配的后缀值为止，从而完成一次可验证的、需要消耗一定计算时间的工作量证明。一般通过**异步**或 **worker 边缘计算**来完成。全局搜索 pow_answer，就可以找到具体代码位置，不过一般 pow 都为标准 md5，所以我们知道逻辑后，直接计算即可，无需分析源码。

```
import hashlib
import itertools
import string
import time
def calculate_pow(workload_nonce, workload_target, max_nonce_length=6):
    """暴力计算POW答案"""
    start_time = time.time()
    attempts = 0
    for length in range(1, max_nonce_length + 1):
        for combo in itertools.product(string.digits, repeat=length):
            attempts += 1
            nonce = ''.join(combo)
            test_str = workload_nonce + nonce
            if hashlib.md5(test_str.encode()).hexdigest() == workload_target:
                calc_time = time.time() - start_time
                return nonce, calc_time
    return None, time.time() - start_time
if __name__ == "__main__":
    workload_nonce = "d162aeb605f109df#"
    workload_target = "cdfab89aa7a42730b8ddb5f8f8d4f367"
    pow_answer, _ = calculate_pow(workload_nonce, workload_target)
    print(pow_answer)
```

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiawgkIROwfezfu1csYH3SkAoOkvREua1gMrhm0BMZVZq5kxp5RnS0bkfO8ko8hGJybjtRICfwjzzAA/640?from=appmsg&watermark=1#imgIndex=8)

  

  

  

  

  

  

  

    ok，接下来着重分析一下 collect 的生成过程。  

  

**补环境**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/DeUVC3rzZGmpc0hgWFVpoFBx1emBfQo7IUk2Ess9Biad4c1wJyDib0uLJwEFpWcqpsrpl5PicaScUYN7Wd5VCwUibA/640?from=appmsg#imgIndex=9)

    依旧先从补环境起手，按照以往文章公式化思路，第一步先找关键插桩点：属性访问和函数调用 (apply、call)。  

    该样本代码结构清晰且无混淆，很容易就能找到对应位置（复杂样本可丢给 AI 协助分析）。属性访问和函数调用都有两个位置，一个返回上下文的和不返回的，统统插上桩，打印日志。直接拿日志来补环境。  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiawgkIROwfezfu1csYH3SkAou2ia87LuicoQNLpVq9sZAP4micPgkQ9QMwQDX4knfoxofo1e90MgYrcoA/640?from=appmsg&watermark=1#imgIndex=10)

  

  

  

  

  

  

  

    关于该样本的补环境，其实没什么难点，就是费时间，对照浏览器日志，依旧是缺啥补啥的思路，将出现的每个 dom 或 bom 函数补齐就行。具体补环境代码我放后台了，有需要的可以后台回复**企鹅滑块**获取，我这里就简单列举一些需要补齐的函数：

```
window.TCaptchaReferrer = 'https://cloud.tencent.com/product/captcha';
global.navigator = {
    userAgent: 'Mozilla/5.0 (Windows NT 10.0; WOW64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/134.0.0.0 Safari/537.36',
    platform: 'Win32',
    language: 'zh-CN',
    languages: ['zh-CN', 'zh'],
    hardwareConcurrency: 32,
    deviceMemory: 8,
    maxTouchPoints: 10,
    vendor: 'Google Inc.',
    webdriver: false,
    cookieEnabled: true,
    product: 'Gecko',
    productSub: '20030107',
    requestMIDIAccess: {}, 
    serviceWorker: {}
};
window.innerWidth = 360;
window.innerHeight = 360;
canvas相关：fillText、fillRect、fillStyle、font
webgl相关：getExtension、getSupportedExtensions、getParameter
localStorage、sessionStorage
global.screen = {
    width: 2560,
    height: 1440,
    availWidth: 2560,
    availHeight: 1392,
    colorDepth: 24,
    pixelDepth: 24,
    availLeft: 0,
    availTop: 0,
    orientation: {
        type: 'landscape-primary',
        angle: 0
    }
};
```

    这里也有小坑，就是会判断某些属性是否存在于对象中，比如判断 navigator 中是否有 requestMIDIAccess。如果你是采用 proxy 方式补环境，可能很难发现这个地方。具体代码如下：

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiawgkIROwfezfu1csYH3SkAojnhEC99bvszM69ZA2yunBMtypiaoGNQtcFgpJaHQiaeibx7jq4HR3QZzQ/640?from=appmsg&watermark=1#imgIndex=11)

  

  

  

  

  

  

  

    在一块可以插个桩，打印一下日志，方便我们补齐缺失的内容。

```
function () {
                let child = I[I.length - 2]
                let obj = I[I.length - 1]
                kfc_log({
                    idx: __VM_EXEC_INDEX,
                    op: 66,
                    type: '从属',
                    child: child,
                    Obj: obj,
                    result: child in obj,
                });
                I[I.length - 2] = I[I.length - 2] in I.pop()
            }
```

    虽然这是滑块，但是并不需要传入轨迹，可以直接省略鼠标事件的绑定和轨迹模拟，具体原因稍后说明。

  

**算法分析**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/DeUVC3rzZGmpc0hgWFVpoFBx1emBfQo7IUk2Ess9Biad4c1wJyDib0uLJwEFpWcqpsrpl5PicaScUYN7Wd5VCwUibA/640?from=appmsg#imgIndex=12)

    在上面的分析中我们知道 tdc.js 的地址是在初始化请求中的，而不是一个固定地址，所以，这是一个动态 vmp，虽然不同 js 的代码一样，但是字节码数组是不一样的。也就是说无法做到真正意义上的**纯算**，但这并不妨碍我们来分析一下他的参数生成逻辑。

    先从日志入手，可以看到将生成的环境信息字符串切割，具体切割逻辑为每次切割两段，每段长度为 4，然后将这两小段字符串转为一个 int，接下来做的就是先找到字符串转 int 的具体逻辑。  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiawgkIROwfezfu1csYH3SkAozPUz581ufX5QJWrfdNbV3TO3LSGDPUyQuNH0X3PW9AwhsFNrd6ayDQ/640?from=appmsg&watermark=1#imgIndex=13)

  

  

  

  

  

  

  

    如上图第一个红框内，“7360” 是如何变成 808858423？这一块比较繁琐，需要单步调试，然后打印出做逻辑运算步骤的日志。这里给几个逻辑运算函数插桩，如：左移、右移、按位与等等，由于当前算法只用到了**左移**和**按位或**，所以下面示例插桩只展示这两个：  

```
function() {
            let operand2 = I[I.length - 1];
            let operand1 = I[I.length - 2];
            kfc_log({
                idx: __VM_EXEC_INDEX,
                type: 'bitwise_or',
                operand1: operand1,
                operand2: operand2,
                result: operand1 | operand2
            });
            I[I.length - 2] = I[I.length - 2] | I.pop()
        }
function() {
            let operand2 = I[I.length - 1];
            let operand1 = I[I.length - 2];
            kfc_log({
                idx: __VM_EXEC_INDEX,
                type: 'left_shift',
                operand1: operand1,
                operand2: operand2,
                result: operand1 << operand2
            });
            I[I.length - 2] = I[I.length - 2] << I.pop()
        }
```

然后查看日志：

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiawgkIROwfezfu1csYH3SkAoWR4ib9cqUI18qkWDJ0q64H2zqU5NlqvG1NNQpk9gAicRFhAk369lvebg/640?from=appmsg&watermark=1#imgIndex=14)

  

  

  

  

  

  

  

此过程实际上就是小端序的字符打包。字符串 "7360" 的每个字符被取出其 ASCII 值：

*   '7'-> 55 (0x37)
    
*   '3'-> 51 (0x33)
    
*   '6'-> 54 (0x36)
    
*   '0'-> 48 (0x30)
    

接着，每个值依次左移，移动的位数是其字节位置的 8 倍，然后通过按位或合并：

*   第 1 个字节左移 0 位：55 << 0= 55
    
*   第 2 个字节左移 8 位：51 << 8= 13056
    
*   第 3 个字节左移 16 位：54 << 16= 3538944
    
*   第 4 个字节左移 24 位：48 << 24= 805306368
    

最终合并为：55 | 13056 | 3538944 | 805306368 = 808858423。代码如下：

```
function stringToInt(s) {
    let result = 0;
    for (let i = 0; i < Math.min(s.length, 4); i++) {
        const asciiVal = s.charCodeAt(i);
        result |= asciiVal << (8 * i);
    }
    return result >>> 0;  // 转换为无符号32位整数
}
// 测试
console.log(stringToInt("7360"));  // 输出: 808858423
```

    继续往下看，从日志上可以看出，上一步的获得的结果为下一步的参数，且每两个为一组参数，比如字符串："{\"cd\":[1,1736092800," 截取其 [0,4) 位后转 int 的值为 80808080，再次截取 [4,8) 位后转 int 位 80909090，然后 80808080 和 80909090 为一组参数，进行下一步计算。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiawgkIROwfezfu1csYH3SkAopBlxyhoDgYZ4naicPAUsu3OOwGN3iauxYLMbiaPicenicL5f502zeVqibOCw/640?from=appmsg&watermark=1#imgIndex=15)

  

  

  

  

  

  

  

分析过程和 str 转 int 一样，我这里直接说结论了：  

**1. 初始化**：输入 v0=808858423，v1=808989241，密钥 k=[1330013038,1213685070,1146703714,1248151874]，常数 delta=0x9E3779B9（2654435769）。

**2. 解密循环**：执行多轮解密操作（日志中显示大量移位和与密钥的交互），每轮操作形式符合 XTEA 解密结构：

*   `v1 -= (((v0 << 4) ^ (v0 >> 5)) + v0) ^ (sum + k[(sum>>11) & 3])`
    
*   `sum -= delta`
    
*   `v0 -= (((v1 << 4) ^ (v1 >> 5)) + v1) ^ (sum + k[sum & 3])`
    

**3. 解密结果**：得到新的 v0 和 v1，在日志中显示为 6763017792 和 1162296179（实际应视为 32 位有符号整数：-1826916800 和 1162296179）。

**4. 字节提取**：从解密后的整数中提取 4 个字节：

*   `v0 & 0xFF = 64`→ `'@'`
    
*   `(v0 >> 8) & 0xFF = 118`→ `'v'`
    
*   `v1 & 0xFF = 27`→ `'\u001b'`
    
*   `(v0 >> 24) & 0xFF = 147`→ `''`
    

**5. 字符串生成**：调用 String.fromCharCode(64,118,27,147) 得到最终字符串 "@v\u001b"。

    以上就是从插桩日志中推导出来的过程，这其实就是一个 TEA 算法，算法通过多轮操作（包括移位、加法和异或）对输入块进行解密，得到两个新的 32 位整数（6763017792 和 1162296179），再从中提取字节序列 [64,118,27,147]，最后通过 String.fromCharCode 生成目标字符串。其中密钥 k 为动态的，每个 JS 都不一样。

```
function teaEncrypt(num_lis) {
    var num1 = num_lis[0];
    var num2 = num_lis[1];
    var sum = 0;
    var key = [1330013038,1213685070,1146703714,1248151874];//动态的，无法写死
    var delta = 2654435769;
    for (var i = 0; i < 32; i++) {
        num1 += (((num2 << 4) ^ (num2 >>> 5)) + num2) ^ (sum + key[sum & 3]);
        sum += delta;
        num2 += (((num1 << 4) ^ (num1 >>> 5)) + num1) ^ (sum + key[(sum >> 11) & 3]);
    }
    return [num1, num2];
}
```

    至此，最关键的两个算法推导出来了，接下来就是将所有的处理结果拼接，然后再将拼接结果求 base64，这就是 collect。

    "{\"cd\":[1,1736092800,1000000000,17360....“这只是其中一个需要处理的字符串，一共有三个字符串（如果除去轨迹的话），将这三个处理后的结果再次拼接。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiawgkIROwfezfu1csYH3SkAovyc5lPEr84nM0yFVwJFPu0fMqJ0fGZYgWrSFEj0RlGQtwoibal9MgsA/640?from=appmsg&watermark=1#imgIndex=16)

  

  

  

  

  

  

  

    到这里就基本结束了，但是在上面我们还遗留了一个小问题，就是**为什么不需要传入轨迹也能校验成功？**

    仔细看这三个环境消息的字符串，这些字符串中并没有携带滑块位置坐标，或者标签的 DOMRect 等信息，也就是说服务端压根不知道当前滑块在浏览器中的位置坐标，所以校验轨迹没有任何意义。当然，人家就这样设计的，这里提出来只是一个小技巧，帮我们省去了模拟轨迹的时间。

    觉得有用的话，佬儿们帮忙点个赞。

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_jpg/97ib7mrowjiawgkIROwfezfu1csYH3SkAoX8TL6p6DuosOBoUbOeC7icxpblonpUdC354kGLncUGLsCv7G7f1ZHPg/640?from=appmsg&watermark=1#imgIndex=17)

  

  

  

  

  

  

  

  

**结果验证**

  

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_gif/DeUVC3rzZGmpc0hgWFVpoFBx1emBfQo7IUk2Ess9Biad4c1wJyDib0uLJwEFpWcqpsrpl5PicaScUYN7Wd5VCwUibA/640?from=appmsg#imgIndex=18)

  

  

  

  

  

  

  

![](https://mmbiz.qpic.cn/mmbiz_png/97ib7mrowjiawgkIROwfezfu1csYH3SkAoHfjvhFAjpNSicLVUNpHdibHWfccYtOnycUPAoCChC7Lwibry1tCFb1ianA/640?from=appmsg&watermark=1#imgIndex=19)

  

  

  

  

  

  

  

**END**

![](https://mmbiz.qpic.cn/mmbiz_gif/WFvB6j2FhgcmCxtEY3EhtyDnuk7odevcOk3KLQSDk4JnFEZ6TBoJpGhtNEpF3KjLE0J4pavuosDia9q9cQmyRwQ/640?from=appmsg#imgIndex=20)

![](https://mmbiz.qpic.cn/mmbiz_jpg/97ib7mrowjiawgkIROwfezfu1csYH3SkAodMISiacYTP9o5XwWQLCHibGSYG11HWSqjaWQMdticxlut6x6Uuj5JIomA/640?from=appmsg&watermark=1#imgIndex=21)

  

**窥破表象**

  

**方见本源**