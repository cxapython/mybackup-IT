> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [xz.aliyun.com](https://xz.aliyun.com/t/1844)

> 先知社区，先知安全技术社区

隐写术是关于信息隐藏，即不让计划的接收者之外的任何人知道信息的传递事件（而不只是信息的内容）的一门技巧与科学。英文写作 Steganography，而本套教程内容将带大家了解一下 CTF 赛场上常见的图片隐写方式，以及解决方法。有必要强调的是，隐写术与密码编码是完全不同的概念。

本次图片隐写实验包括四大部分

*   一、附加式的图片隐写
*   二、基于文件结构的图片隐写
*   三、基于 LSB 原理的图片隐写
*   四、基于 DCT 域的 JPG 图片隐写
*   五、数字水印的隐写
*   六、图片容差的隐写

*   操作机：Windows XP
    *   实验工具：
        *   Stegsolve
        *   Python
        *   Java 环境  
            ## 背景知识  
            **什么是 LSB 隐写？**

LSB，最低有效位，英文是 Least Significant Bit 。我们知道图像像素一般是由 RGB 三原色（即红绿蓝）组成的，每一种颜色占用 8 位，0x00~0xFF，即一共有 256 种颜色，一共包含了 256 的 3 次方的颜色，颜色太多，而人的肉眼能区分的只有其中一小部分，这导致了当我们修改 RGB 颜色分量种最低的二进制位的时候，我们的肉眼是区分不出来的。  
**Stegosolve 介绍**  
CTF 中，最常用来检测 LSB 隐写痕迹的工具是 Stegsolve，这是一款可以对图片进行多种操作的工具，包括对图片进行 xor,sub 等操作，对图片不同通道进行查看等功能。

简单的 LSB 隐写
----------

实验

```
- 在实验机中找到隐写术目录，打开图片隐写，打开图片隐写第三部分文件夹
- 在该文件夹找到chal.png
- 双击打开图片，我们先确认一下图片内容并没有什么异常
- 使用Stegsolve打开图片，在不同的通道查看图片
- 在通道切换的过程中，我们看到了flag
- 最后的flag是flag:key{forensics_is_fun}
```

用 Stegsolve 打开图片，并在不同的通道中切换，

[![](https://xzfile.aliyuncs.com/media/upload/picture/20171226091830-aea13f72-e9da-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20171226091830-aea13f72-e9da-1.png)

flag 是 flag:key{forensics_is_fun}

思考
--

1.  我们如何实现这种 LSB 隐写的？是否可以通过 photoshop 这样的工具实现？
2.  查阅更多关于 LSB 隐写的资料。

有一点难度的 LSB 隐写
-------------

我们从第一个部分可以知道，最简单的隐写我们只需要通过工具 Stegsolve 切换到不通通道，我们就可以直接看到隐写内容了，那么更复杂一点就不是这么直接了，而是只能这样工具来查看 LSB 的隐写痕迹，再通过工具或者脚本的方式提取隐写信息。

```
- 在实验机中找到隐写术目录，打开图片隐写，打开图片隐写第三部分文件夹
- 在该文件夹找到LSB.bmp
- 双击打开图片，我们先确认一下图片内容并没有什么异常
- 使用Stegsolve打开图片，在不同的通道查看图片
- 在通道切换的过程中，来判断隐写痕迹
- 编写脚本提取隐写信息。
- 最后打开提取后的文件，得到flag
```

**首先：从 Stegsolve 中打开图片**  
首先点击上方的`FIle`菜单，选择 open，在题目文件夹中找到这次所需要用的图片 whereswaldo.bmp

[![](https://xzfile.aliyuncs.com/media/upload/picture/20171226091118-ad24b030-e9d9-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20171226091118-ad24b030-e9d9-1.png)

**其次：切换到不同通道，通过痕迹来判断是否是 LSB 隐写**  
分析是否有可能是 LSB 隐写，我们开始点击下面的按钮，切换到不同通道，我们逐渐对比不同通道我们所看到的图片是怎么样子的。  
我们发现在 Red plane0 和 Greee plane 0 以及 B 略 plane 0 出现了相同的异常情况，我们这里基本可以断定就是 LSB 隐写了  
[![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)](http://oayoilchh.bkt.clouddn.com/17-12-26/77648426.jpg)

**编写代码提取信息**  
因为是 LSB 隐写，我们只按位提取 RGB 的最低位即可，代码如下：

```
from PIL import Image

im = Image.open("extracted.bmp")
pix = im.load()
width, height = im.size

extracted_bits = []
for y in range(height):
    for x in range(width):
        r, g, b = pix[(x,y)]
        extracted_bits.append(r & 1)
        extracted_bits.append(g & 1)
        extracted_bits.append(b & 1)

extracted_byte_bits = [extracted_bits[i:i+8] for i in range(0, len(extracted_bits), 8)]
with open("extracted2.bmp", "wb") as out:
    for byte_bits in extracted_byte_bits:
                byte_str = ''.join(str(x) for x in byte_bits)
        byte = chr(int(byte_str, 2))
        out.write(byte)


```

打开我们需要提取信息的的图片，y，x 代表的是图片的高以及宽度，进行一个循环提取。  
运行代码，extracted.py，打开图片即可。

思考
--

1.  我们这里用的 LSB 隐均对 R,G,B，三种颜色都加以修改是否可以只修改一个颜色？
2.  参考 2016 HCTF 的官方 Writeup 学习如何实现将一个文件以 LSB 的形式加以隐写。