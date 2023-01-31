> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [xz.aliyun.com](https://xz.aliyun.com/t/1836)

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
        *   010Editor
        *   CRC Calculator

背景知识
----

首先这里需要明确一下我这里所说的文件结构是什么意思。文件结构特指的是图片文件的文件结构。我们这里主要讲的是 PNG 图片的文件结构。

PNG，图像文件存储格式，其设计目的是试图替代 GIF 和 TIFF 文件格式，同时增加一些 GIF 文件格式所不具备的特性。是一种位图文件 (bitmap file) 存储格式，读作“ping”。PNG 用来存储灰度图像时，灰度图像的深度可多到 16 位，存储彩色图像时，彩色图像的深度可多到 48 位，并且还可存储多到 16 位的α通道数据。

对于一个正常的 PNG 图片来讲，其文件头总是由固定的字节来表示的，以 16 进制表示即位 89 50 4E 47 0D 0A 1A 0A，这一部分称作文件头。  
标准的 PNG 文件结构应包括：

*   PNG 文件标志
*   PNG 数据块  
    PNG 图片是有两种数据块的，一个是叫关键数据块，另一种是辅助数据块。正常的关键数据块，定义了 4 种标准数据块，个 PNG 文件都必须包含它们。  
    它们分别是长度，数据块类型码，数据块数据，循环冗余检测即 CRC。  
    我们这里重点先了解一下，png 图片文件头数据块以及 png 图片 IDAT 块，这次的隐写也是以这两个地方位基础的。  
    **png 图片文件头数据块**  
    即 IHDR，这是 PNG 图片的第一个数据块，一张 PNG 图片仅有一个 IHDR 数据块，它包含了哪些信息呢？IHDR 中，包括了图片的宽，高，图像深度，颜色类型，压缩方法等等。

[![](https://xzfile.aliyuncs.com/media/upload/picture/20171224094430-fba6109e-e84b-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20171224094430-fba6109e-e84b-1.png)  
如图中蓝色的部分即 IHDR 数据块。  
**IDAT 数据块**  
它存储实际的数据，在数据流中可包含多个连续顺序的图像数据块。这是一个可以存在多个数据块类型的数据块。它的作用就是存储着图像真正的数据。  
因为它是可以存在多个的，所以即使我们写入一个多余的 IDAT 也不会多大影响肉眼对图片的观察

**下面进行实验 Part 2 基于文件结构的隐写**

高度被修改引起的隐写
----------

背景知识中，我们了解到，图片的高度，宽度的值存放于 PNG 图片的文件头数据块，那么我们就是可以通过修改 PNG 图片的高度值，来对部分信息进行隐藏的。

> *   实验：

```
- 在实验机中找到隐写术目录，打开图片隐写，打开图片隐写第二部分文件夹
- 在该文件夹找到 hight.png，
- 双击打开图片，我们先确认一下图片内容并没有什么异常
- 正如前文所说，我们这个实验部分讲的是图片高度值被修改引起的的隐写方式，所以我们010Editor
- 在010Editor运行PNG模板，这样方便于我们修改PNG图片的高度值
- 找到PNG图片高度值对应的地方，然后修改为一个较大的值，并重新计算，修改CRC校验值，并保存文件
- 打开保存后的图片，发现底部看到了之前被隐写的信息
```

**用 010Editor 打开图片，运行 PNG 模板**  
10 editor 呢？因为这个 16 进制编辑器，有模版功能，当我们运行模版后，可以轻易的找到图片的各个数据块的位置以及内容。

[![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)](http://oayoilchh.bkt.clouddn.com/17-12-24/23726586.jpg)

10 editor 这个 16 进制编辑器，有模版功能，当我们运行模版后，可以轻易的找到图片的各个数据块的位置以及内容。  
**找到 PNG 图片高度值所对应的位置，并修改为一个较大的值**

我们找到 IHDR 数据块，并翻到 struct IHDR Ihdr 位置，修改 height 的值到一个较大的值，如从 700 修改到 800。  
**使用 CRC Calculator 重新计算 CRC 校验值**

[![](https://xzfile.aliyuncs.com/media/upload/picture/20171224094810-7f59ff18-e84c-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20171224094810-7f59ff18-e84c-1.png)  
输入参数，然后点击 Calculator 计算，得到 CRC 值  
为什么要重新计算 CRC 校验值呢？防止图片被我们修改后，自身的 CRC 校验报错，导致图片不能正常打开。

**修改相应的 CRC 校验值，为我们重新计算后数值**  
[![](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADIAAAAyCAYAAAAeP4ixAAACbklEQVRoQ+2aMU4dMRCGZw6RC1CSSyQdLZJtKQ2REgoiRIpQkCYClCYpkgIESQFIpIlkW+IIcIC0gUNwiEFGz+hlmbG9b1nesvGW++zxfP7H4/H6IYzkwZFwQAUZmpJVkSeniFJKA8ASIi7MyfkrRPxjrT1JjZ8MLaXUDiJuzwngn2GJaNd7vyP5IoIYY94Q0fEQIKIPRGS8947zSQTRWh8CwLuBgZx479+2BTkHgBdDAgGAC+fcywoyIFWqInWN9BSONbTmFVp/AeA5o+rjKRJ2XwBYRsRXM4ZXgAg2LAPzOCDTJYQx5pSIVlrC3EI45y611osMTHuQUPUiYpiVooerg7TWRwDAlhSM0TuI+BsD0x4kGCuFSRVzSqkfiLiWmY17EALMbCAlMCmI6IwxZo+INgQYEYKBuW5da00PKikjhNNiiPGm01rrbwDwofGehQjjNcv1SZgddALhlJEgwgJFxDNr7acmjFLqCyJuTd6LEGFttpmkYC91Hrk3s1GZFERMmUT01Xv/sQljjPlMRMsxO6WULwnb2D8FEs4j680wScjO5f3vzrlNJszESWq2LYXJgTzjZm56MCHf3zVBxH1r7ftU1splxxKYHEgoUUpTo+grEf303rPH5hxENJqDKQEJtko2q9zGeeycWy3JhpKhWT8+NM/sufIhBwKI+Mta+7pkfxKMtd8Qtdbcx4dUQZcFCQ2I6DcAnLUpf6YMPxhIDDOuxC4C6djoQUE6+tKpewWZ1wlRkq0qUhXptKTlzv93aI3jWmE0Fz2TeujpX73F9TaKy9CeMk8vZusfBnqZ1g5GqyIdJq+XrqNR5AahKr9CCcxGSwAAAABJRU5ErkJggg==)](http://oayoilchh.bkt.clouddn.com/17-12-24/55760464.jpg)

思考
--

这个实验，我们进行了 PNG 图片高度修改以及 CRC 校验值的重计算，那么请大家以下问题

1.  JPG 图片是否也有这样的隐写形式呢？
2.  了解 JPG 以及 GIF 等图片文件的格式。

隐写信息以 IDAT 块加入图片
----------------

在背景知识中，我们提到了一个重要的概念就是图片的 IDAT 块是可以存在多个的，这导致了我们可以将隐写西信息以 IDAT 块的形似加入图片。

> *   实验：

```
- 在实验机中找到隐写术目录，打开图片隐写，打开图片隐写第二部分文件夹
- 在该文件夹找到 hidden.png，
- 双击打开图片，我们先确认一下图片内容并没有什么异常
- 使用pngcheck先对图片检测
- 在pngcheck的检测下，我们会发现异常信息，我们对异常的块进行提取
- 编写脚本，提取异常信息
```

**前景知识**  
pngcheck 可以验证 PNG 图片的完整性（通过检查内部 CRC-32 校验和 & bra; 比特 & ket;) 和解压缩图像数据；它能够转储几乎所有任选的块级别信息在该图像中的可读数据。  
我们使用 pngcheck -v hidden.png 如此的命令对图片进行检测

**使用 pngcheck 对图片进行检测**

[![](https://xzfile.aliyuncs.com/media/upload/picture/20171224095019-cc16acac-e84c-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20171224095019-cc16acac-e84c-1.png)  
我们使用命令：

对图片的文件结构进行检测。

**发现异常，并判断异常的原因**  
我们会发现，图片的的数据块形式是如下的  
Type: IHDR  
Size: 13  
CRC ： 5412913F

Pos ： 33  
Type: IDAT  
Size: 10980  
CRC ： 98F96EEB

Pos ： 11025  
Type: IEND  
Size: 0  
CRC ： AE426082

我们会惊讶的发现 pos 为 11025 的 size 居然为 0，这是一块有问题的地方，我们可以怀疑，这一块是隐写的信息。  
**编写脚本并提取内容**

```
#!/usr/bin/python

from struct import unpack
from binascii import hexlify, unhexlify
import sys, zlib

# Returns [Position, Chunk Size, Chunk Type, Chunk Data, Chunk CRC]
def getChunk(buf, pos):
    a = []
    a.append(pos)
    size = unpack('!I', buf[pos:pos+4])[0]
    # Chunk Size
    a.append(buf[pos:pos+4])
    # Chunk Type
    a.append(buf[pos+4:pos+8])
    # Chunk Data
    a.append(buf[pos+8:pos+8+size])
    # Chunk CRC
    a.append(buf[pos+8+size:pos+12+size])
    return a

def printChunk(buf, pos):
    print 'Pos : '+str(pos)+''
    print 'Type: ' + str(buf[pos+4:pos+8])
    size = unpack('!I', buf[pos:pos+4])[0]
    print 'Size: ' + str(size)
    #print 'Cont: ' + str(hexlify(buf[pos+8:pos+8+size]))
    print 'CRC : ' + str(hexlify(buf[pos+size+8:pos+size+12]).upper())
    print

if len(sys.argv)!=2:
    print 'Usage: ./this Stegano_PNG'
    sys.exit(2)

buf = open(sys.argv[1]).read()
pos=0

print "PNG Signature: " + str(unpack('cccccccc', buf[pos:pos+8]))
pos+=8

chunks = []
for i in range(3):
    chunks.append(getChunk(buf, pos))
    printChunk(buf, pos)
    pos+=unpack('!I',chunks[i][1])[0]+12


decompressed = zlib.decompress(chunks[1][3])
# Decompressed data length = height x (width * 3 + 1)
print "Data length in PNG file : ", len(chunks[1][3])
print "Decompressed data length: ", len(decompressed)

height = unpack('!I',(chunks[0][3][4:8]))[0]
width = unpack('!I',(chunks[0][3][:4]))[0]
blocksize = width * 3 + 1
filterbits = ''
for i in range(0,len(decompressed),blocksize):
    bit = unpack('2401c', decompressed[i:i+blocksize])[0]
    if bit == '\x00': filterbits+='0'
    elif bit == '\x01': filterbits+='1'
    else:
        print 'Bit is not 0 or 1... Default is 0 - MAGIC!'
        sys.exit(3)

s = filterbits
endianess_filterbits = [filterbits[i:i+8][::-1] for i in xrange(0, len(filterbits), 8)]

flag = ''
for x in endianess_filterbits:
    if x=='00000000': break
    flag += unhexlify('%x' % int('0b'+str(x), 2))

print 'Flag: ' + flag

```

脚本如上，flag **DrgnS{WhenYouGazeIntoThePNGThePNGAlsoGazezIntoYou}.**

思考
--

1.  我们是否可以将一张二维码以 IDAT 块的形式写入图片呢？
2.  试着将信息以 IDAT 块的形式写入图片

保存后，重新打开图片，我们就能看到被隐藏的内容