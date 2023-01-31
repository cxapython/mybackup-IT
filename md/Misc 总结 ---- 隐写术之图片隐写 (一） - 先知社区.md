> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [xz.aliyun.com](https://xz.aliyun.com/t/1833)

> 先知社区，先知安全技术社区

隐写术是关于信息隐藏，即不让计划的接收者之外的任何人知道信息的传递事件（而不只是信息的内容）的一门技巧与科学。英文写作 Steganography，而本套教程内容将带大家了解一下 CTF 赛场上常见的图片隐写方式，以及解决方法。有必要强调的是，隐写术与密码编码是完全不同的概念。

本次图片隐写实验包括四大部分

*   一、附加式的图片隐写
*   二、基于文件结构的图片隐写
*   三、基于 LSB 原理的图片隐写
*   四、基于 DCT 域的 JPG 图片隐写
*   五、数字水印的隐写
*   六、图片容差的隐写

**下面进行实验 Part 1 附加式图片隐写**

在附加式的图片隐写术中，我们通常是用某种程序或者某种方法在载体文件中直接附加上需要被隐写的目标，然后将载体文件直接传输给接受者或者发布到网站上，然后接受者者根据方法提取出被隐写的消息，这一个过程就是我们这里想提到的附加式图片隐写。

而在 CTF 赛事中，关于这种图片隐写的大概有两种经典方式，一是直接附加字符串，二是图种的形式出现。

*   操作机：Windows XP
    *   实验工具：
        *   Strings
        *   binwalk
        *   Winhex

附加字符串
-----

> *   实验：

```
- 在实验机中找到隐写术目录，打开图片隐写，打开图片隐写第一部分文件夹
- 在该文件夹找到 xscq.jpg，
- 双击打开图片，我们先确认一下图片内容并没有什么异常
- 正如前文所说，我们这个实验部分讲的是附加字符串的隐写方式，所以我们用Strings检查一下图片
- 在Strings工具的搜索下，我们看到了一串base64编码后的字符串
- 最终解码后，flag：flag{welcome_to_xianzhi}
```

**strings 使用方法**

strings 命令在对象文件或二进制文件中查找可打印的字符串。字符串是 4 个或更多可打印字符的任意序列，以换行符或空字符结束。 strings 命令对识别随机对象文件很有用。

选项：

*   -a --all：扫描整个文件而不是只扫描目标文件初始化和装载段
*   -f –print-file-name：在显示字符串前先显示文件名
*   -t --radix={o,d,x} ：输出字符的位置，基于八进制，十进制或者十六进制
*   -e --encoding={s,S,b,l,B,L} ：选择字符大小和排列顺序: s = 7-bit, S = 8-bit, {b,l} = 16-bit, {B,L} = 32-bit

Tips 我们使用 strings + 文件名字的命令即可  
**具体步骤如下：**

在 cmd 中打开 strings 工具，使用如下命令

￼  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20171222162932-3c17bfde-e6f2-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20171222162932-3c17bfde-e6f2-1.png)

得到如下字符串：ZmxhZ3t3ZWxjb21lX3RvX3hpYW56aGl9  
我们尝试用 base64 解码，代码过程如下：

```
Python 2.7.12 (v2.7.12:d33e0cf91556, Jun 27 2016, 15:24:40) [MSC v.1500 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
>>> import base64
>>> base64.b64decode('ZmxhZ3t3ZWxjb21lX3RvX3hpYW56aGl9')
'flag{welcome_to_xianzhi}'
>>>


```

有必要提到的是，为什么字符串要附加在文件的后面呢? 那是因为，如果图片附加在中间，有可能破坏了图片的信息，如果字符串附加在图片的头部位置，又破坏了文件头，可能导致图片无法识别。关于文件格式的具体内容，我们下一个部分的隐写还会提到。

思考
--

1.  我们是否可以使用 16 进制的编辑器找到这一串字符串？请用工具中的 winhex 尝试这种解法。
2.  隐写和密码学的区别是？

图种形式的隐写
-------

图种：  
一种采用特殊方式将图片文件（如 jpg 格式）与 rar 文件结合起来的文件。该文件一般保存为 jpg 格式，可以正常显示图片，当有人获取该图片后，可以修改文件的后缀名，将图片改为 rar 压缩文件，并得到其中的数据。  
图种这是一种以图片文件为载体，通常为 jpg 格式的图片，然后将 zip 等压缩包文件附加在图片文件后面。因为操作系统识别的过程中是，从文件头标志，到文件的结束标志位，当系统识别到图片的结束标志位后，默认是不再继续识别的，所以我们在通常情况下只能看到它是只是一张图片。

实验

```
- 在实验机中找到隐写术目录，打开图片隐写，打开图片隐写第一部分文件夹
- 在该文件夹找到cqzb.jpg，
- 双击打开图片，我们先确认一下图片内容并没有什么异常
- 对图片进行检测，确认是不是图种
- 使用winhex打开图片，并分离图片，得到一个压缩包
- 打开压缩包得到flag，flag：flag{This is easy}

```

简单的检测方式：  
打开工具中的 binwalk。使用如下命令：

￼  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20171222163131-832f03d2-e6f2-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20171222163131-832f03d2-e6f2-1.png)

我们可以发现，binwalk 自动识别出来了 zip 文件，而且偏移也告诉我们了, 当然我们这里如果使用

这样的命令，是很快就能把 ZIP 文件给提取出来的，但是这里我想讲的是如何用 winhex 等 16 进制编辑器，将压缩包提取出来。

**使用 winhex16 进制编辑器提取 ZIP 文件**

*   首先需要了解一下什么是文件头  
    文件头就是是位于文件开头的一段承担一定任务的数据。一般都在开头的部分。以 jpg 图片和 zip 压缩包文件为例。图 6 和图 7 分别是 jpg 图片的文件头以及 jpg 图片的结尾。  
    我们如何，找到 JPG 图片和 ZIP 图片呢？  
    **JPG 图片的文件头和结束标志**

[![](https://xzfile.aliyuncs.com/media/upload/picture/20171222163150-8e379b4a-e6f2-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20171222163150-8e379b4a-e6f2-1.png)

￼  
上图，FF D8 FF E1 就是 JPG 图片的文件头，一般当我们看到文件开头是如此的格式，我们就能认为这是一个 JPG 图片了。  
￼  
上图以 03 FF D9 为结束标志，这是 JPG 图片的结束标志位。

**ZIP 文件的文件头和结束标志**

[![](https://xzfile.aliyuncs.com/media/upload/picture/20171222163202-9575d890-e6f2-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20171222163202-9575d890-e6f2-1.png)

￼  
上图 50 4B 03 04 就是 ZIP 文件的文件头，一般以 PK 表示。

*   找到 cqzb.jpg 中隐藏的 ZIP 文件

上文我们讲述了，JPG 图片的结束标识是 03 FF D9,ZIP 文件的文件头是 50 4B 03 04，我们只需要在 winhex 中找到 ZIP 文件的文件头即可，滑动滚条到最底下。上文讲了一般附加的位置是在原本文件的后面，所以我们果断滑动滚动条到最后。  
￼  
[![](https://xzfile.aliyuncs.com/media/upload/picture/20171222163229-a5abc4b8-e6f2-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20171222163229-a5abc4b8-e6f2-1.png)

从图中我们可以明显看到 cqzb.jpg 明显不是以 FF D9 结尾，而且我们在上面不远的地方发现了 zip 的文件头 50 4B 03 04，所以我们可以断定这是个图种文件了

*   分离 ZIP 文件  
    下一步我们该如何用 winhex 截取我们所需要的文件呢？  
    我们选取以 50 开头以及到末尾的的数据，右键单击，选择编辑，复制选块到新文件，保存新文件为 zip 格式命名规则即可。  
    ￼  
    保存为 ZIP 文件，解压缩后就能得到 flag，所以最后的 flag 是 flag{This is easy}

[![](https://xzfile.aliyuncs.com/media/upload/picture/20171222163249-b18f804e-e6f2-1.png)](https://xzfile.aliyuncs.com/media/upload/picture/20171222163249-b18f804e-e6f2-1.png)

思考
--

1.  自己动手使用 binwalk 分离图片
2.  除了上述讲的方法我们是否可以使用其他手段分离文件？如 dd 这样的工具？