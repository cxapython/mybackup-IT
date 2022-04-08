> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/qq_42857999/article/details/122364690)

### 极验第四代滑块验证码破解（二）：滑块缺口识别

*   *   [声明](#_3)
    *   [一、环境安装](#_16)
    *   *   [1. 第三方库安装](#1__18)
    *   [二、滑块缺口识别](#_25)
    *   *   [1. 与极验三代滑块对比](#1__27)
        *   [2. 缺口识别完整代码](#2__34)
    *   [三、结语](#_163)
    *   *   [* 本期文章结束啦，如果对您有帮助，记得收藏加关注哦，后期文章会持续更新 ~~~*](#__165)

声明
--

**原创文章，请勿转载！**

**本文内容仅限于安全研究，不公开具体源码。维护网络安全，人人有责。**

**本文关联文章超链接：**

1.  [极验第四代滑块验证码破解（一）：AST 还原混淆 JS](https://blog.csdn.net/qq_42857999/article/details/122364575)
2.  [极验第四代滑块验证码破解（二）：滑块缺口识别](https://blog.csdn.net/qq_42857999/article/details/122364690)
3.  [极验第四代滑块验证码破解（三）：滑块轨迹构造](https://blog.csdn.net/qq_42857999/article/details/122364712)
4.  [极验第四代滑块验证码破解（四）：请求分析及加密参数破解](https://blog.csdn.net/qq_42857999/article/details/122364731)

一、环境安装
------

### 1. 第三方库安装

```
pip install Pillow
pip install numpy
pip install opencv-python
```

二、滑块缺口识别
--------

### 1. 与极验三代滑块对比

*   极验四代有更多的缺口形状
*   极验四验证码图片不需要还原，难度变小

![](https://img-blog.csdnimg.cn/114c0c10f0f34c9387b3524101fdef52.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)

### 2. 缺口识别完整代码

经过测试，极验 3 的缺口识别代码适用极验 4 缺口识别，所有我就不做过都的讲解了。  
极验 3 缺口识别链接：[极验滑块验证码破解与研究（三）：滑块缺口识别](https://blog.csdn.net/qq_42857999/article/details/121635961)  
**下面的代码是基于极验 3 的缺口识别代码，做了一点优化，增加了 cv2.pyrMeanShiftFiltering(金字塔均值漂移) 做预处理**

```
# -*- coding: utf-8 -*-
from pathlib import Path

import PIL
import cv2
import numpy as np


def imshow(img, winname='test', delay=0):
    """cv2展示图片"""
    cv2.imshow(winname, img)
    cv2.waitKey(delay)
    cv2.destroyAllWindows()


def pil_to_cv2(img):
    """
    pil转cv2图片
    :param img: pil图像, <type 'PIL.JpegImagePlugin.JpegImageFile'>
    :return: cv2图像, <type 'numpy.ndarray'>
    """
    img = cv2.cvtColor(np.asarray(img), cv2.COLOR_RGB2BGR)
    return img


def bytes_to_cv2(img):
    """
    二进制图片转cv2
    :param img: 二进制图片数据, <type 'bytes'>
    :return: cv2图像, <type 'numpy.ndarray'>
    """
    # 将图片字节码bytes, 转换成一维的numpy数组到缓存中
    img_buffer_np = np.frombuffer(img, dtype=np.uint8)
    # 从指定的内存缓存中读取一维numpy数据, 并把数据转换(解码)成图像矩阵格式
    img_np = cv2.imdecode(img_buffer_np, 1)
    return img_np


def cv2_open(img, flag=None):
    """
    统一输出图片格式为cv2图像, <type 'numpy.ndarray'>
    :param img: <type 'bytes'/'numpy.ndarray'/'str'/'Path'/'PIL.JpegImagePlugin.JpegImageFile'>
    :param flag: 颜色空间转换类型, default: None
        eg: cv2.COLOR_BGR2GRAY（灰度图）
    :return: cv2图像, <numpy.ndarray>
    """
    if isinstance(img, bytes):
        img = bytes_to_cv2(img)
    elif isinstance(img, (str, Path)):
        img = cv2.imread(str(img))
    elif isinstance(img, np.ndarray):
        img = img
    elif isinstance(img, PIL.Image):
        img = pil_to_cv2(img)
    else:
        raise ValueError(f'输入的图片类型无法解析: {type(img)}')
    if flag is not None:
        img = cv2.cvtColor(img, flag)
    return img


def get_distance(bg, tp, im_show=False, save_path=None):
    """
    :param bg: 背景图路径或Path对象或图片二进制
        eg: 'assets/bg.jpg'
            Path('assets/bg.jpg')
    :param tp: 缺口图路径或Path对象或图片二进制
        eg: 'assets/tp.jpg'
            Path('assets/tp.jpg')
    :param im_show: 是否显示结果, <type 'bool'>; default: False
    :param save_path: 保存路径, <type 'str'/'Path'>; default: None
    :return: 缺口位置
    """
    # 读取图片
    bg_img = cv2_open(bg)
    tp_gray = cv2_open(tp, flag=cv2.COLOR_BGR2GRAY)

    # 金字塔均值漂移
    bg_shift = cv2.pyrMeanShiftFiltering(bg_img, 5, 50)

    # 边缘检测
    tp_gray = cv2.Canny(tp_gray, 255, 255)
    bg_gray = cv2.Canny(bg_shift, 255, 255)

    # 目标匹配
    result = cv2.matchTemplate(bg_gray, tp_gray, cv2.TM_CCOEFF_NORMED)
    # 解析匹配结果
    min_val, max_val, min_loc, max_loc = cv2.minMaxLoc(result)

    distance = max_loc[0]
    if save_path or im_show:
        # 需要绘制的方框高度和宽度
        tp_height, tp_width = tp_gray.shape[:2]
        # 矩形左上角点位置
        x, y = max_loc
        # 矩形右下角点位置
        _x, _y = x + tp_width, y + tp_height
        # 绘制矩形
        bg_img = cv2_open(bg)
        cv2.rectangle(bg_img, (x, y), (_x, _y), (0, 0, 255), 2)
        # 保存缺口识别结果到背景图
        if save_path:
            save_path = Path(save_path).resolve()
            save_path = save_path.parent / f"{save_path.stem}.{distance}{save_path.suffix}"
            save_path = save_path.__str__()
            cv2.imwrite(save_path, bg_img)
        # 显示缺口识别结果
        if im_show:
            imshow(bg_img)
    return distance


if __name__ == '__main__':
    d = get_distance(
        bg='assets/bg.png',
        tp='assets/slice.png',
        im_show=True,
        save_path='assets/bg.png'
    )
    print(d)
```

三、结语
----

**友情链接：**[**极验第四代滑块验证码破解（三）：滑块轨迹构造**](https://blog.csdn.net/qq_42857999/article/details/122364712)

### _本期文章结束啦，如果对您有帮助，记得收藏加关注哦，后期文章会持续更新 ~~~_

![](https://img-blog.csdnimg.cn/49889af6548044a48995275978ab2191.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5biv5rOq55qE6bG8,size_20,color_FFFFFF,t_70,g_se,x_16)