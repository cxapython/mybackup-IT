> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_42857999/article/details/122364712)

### 极验第四代滑块验证码破解（三）：滑块轨迹构造

*   *   [声明](#_3)
    *   [一、极验滑动轨迹分析](#_16)
    *   *   [1. 生成滑动轨迹的 js 入口](#1_js_19)
        *   [2. 滑动轨迹的 python 实现](#2_python_34)
    *   [二、结语](#_95)
    *   *   [* 本期文章结束啦，如果对您有帮助，记得收藏加关注哦，后期文章会持续更新 ~~~*](#__97)

声明
--

**原创文章，请勿转载！**

**本文内容仅限于安全研究，不公开具体源码。维护网络安全，人人有责。**

**本文关联文章超链接：**

1.  [极验第四代滑块验证码破解（一）：AST 还原混淆 JS](https://blog.csdn.net/qq_42857999/article/details/122364575)
2.  [极验第四代滑块验证码破解（二）：滑块缺口识别](https://blog.csdn.net/qq_42857999/article/details/122364690)
3.  [极验第四代滑块验证码破解（三）：滑块轨迹构造](https://blog.csdn.net/qq_42857999/article/details/122364712)
4.  [极验第四代滑块验证码破解（四）：请求分析及加密参数破解](https://blog.csdn.net/qq_42857999/article/details/122364731)

一、极验滑动轨迹分析
----------

想构造一个成功率高的滑动轨迹，首先得知道 js 代码中是如何记录滑动过程，不同厂家的记录方式不同。但是，万变不离其宗，记录的过程无非就是通过触发鼠标的点击、移动等事件，记录时间、位置等信息。

### 1. 生成滑动轨迹的 js 入口

破解极验最终的目的就是构造 w 参数，而 w 参数是由滑动轨迹通过加密算法得到的。所以，可以先找到 w 参数的位置，通过 w 参数跳到滑动轨迹加密之前的位置。

*   第一步：将混淆的 js 代码还原，还原过程请看 [极验第四代滑块验证码破解（一）：AST 还原混淆 JS](https://blog.csdn.net/qq_42857999/article/details/122364575)
    
*   第二步：使用 reres 插件将 gcaptcha4.js 替换成还原后的 js 文件
    
*   第三步：在 gcaptcha4.js 中搜索 "w"，适当位置打上 debugger
    
*   第四步：打开 [极验 4 代官方测试网站](https://www.geetest.com/adaptive-captcha-demo) ，滑动滑块触发 debugger  
    ![](https://img-blog.csdnimg.cn/3830d7f842884c8faf82feabb6aa816a.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)
    
*   第五步：通过调用栈找到 B F F C 函 数 中 调 用 _BFFC 函数中调用 B​FFC 函数中调用_BBDx 函数的位置，并打上 debugger  
    ![](https://img-blog.csdnimg.cn/e71b5228458644979c39fb83bd0b1b2d.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)
    
*   第六步：多滑动几次，观察滑动轨迹规律，便于自己构造
    
*   最后：这里我要吐槽下了，极验的研发太不上心了，看看下面的图片就知道了  
    ![](https://img-blog.csdnimg.cn/8af8fba271074b9c8e1a3530358211c7.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)
    

### 2. 滑动轨迹的 python 实现

此方式构造的滑动轨迹，成功率在 95% 左右，其中各项参数还可以优化。  
优化方案：滑动的距离在 0-222 范围内，可能性不大。每次可保存校验成功的轨迹。下次遇到相同距离时，可直接调用。

```
# -*- coding: utf-8 -*-
import random


def __ease_out_expo(x):
    if x == 1:
        return 1
    else:
        return 1 - pow(2, -10 * x)


def __ease_out_quart(x):
    return 1 - pow(1 - x, 4)


def get_slide_track(distance):
    """
    根据滑动距离生成滑动轨迹
    :param distance: 需要滑动的距离
    :return: 滑动轨迹<type 'list'>: [[x,y,t], ...]
        x: 已滑动的横向距离
        y: 已滑动的纵向距离, 除起点外, 均为0
        t: 滑动过程消耗的时间, 单位: 毫秒
    """

    if not isinstance(distance, int) or distance < 0:
        raise ValueError(f"distance类型必须是大于等于0的整数: distance: {distance}, type: {type(distance)}")
    # 初始化轨迹列表
    slide_track = [
        [random.randint(20, 60), random.randint(10, 40), 0]
    ]
    # 共记录count次滑块位置信息
    count = 30 + int(distance / 2)
    # 初始化滑动时间
    t = random.randint(50, 100)
    # 记录上一次滑动的距离
    _x = 0
    _y = 0
    for i in range(count):
        # 已滑动的横向距离
        x = round(__ease_out_expo(i / count) * distance)
        # 滑动过程消耗的时间
        t = random.randint(10, 20)
        if x == _x:
            continue
        slide_track.append([x - _x, _y, t])
        _x = x
    slide_track.append([0, 0, random.randint(200, 300)])
    return slide_track


if __name__ == '__main__':
    for _ in get_slide_track(100):
        print(_)
```

二、结语
----

**友情链接：**[**极验第四代滑块验证码破解（四）：请求分析及加密参数破解**](https://blog.csdn.net/qq_42857999/article/details/122364731)

### _本期文章结束啦，如果对您有帮助，记得收藏加关注哦，后期文章会持续更新 ~~~_

![](https://img-blog.csdnimg.cn/9165b8e64ddc428cad7363250da8c8f6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)