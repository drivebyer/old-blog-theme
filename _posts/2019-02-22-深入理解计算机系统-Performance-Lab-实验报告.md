---
layout: post
title: "深入理解计算机系统 Performance Lab 实验报告"
comments: true
description: "深入理解计算机系统 Performance Lab 实验报告"
keywords: "Performance Lab, 深入理解计算机系统, CSAPP"
---

# 介绍

这个实验的目的是优化内存紧凑的代码。在这个实验中，我们来看两个图像处理过程，分别对应本实验的两个部分：

1. Rotate：将图片逆时针旋转 90 度
2. Smooth：将图片变得平滑

我们将实验中的图像当做一个行列相等的矩阵 M（0 ~ N-1）。二维矩阵中每个元素相当于图像的像素，像素是 RGB 的一个三元值。

本实验唯一需要修改的就是 `kernels.c`。在 `kernels.c` 中完成响应的代码后。执行下面的命令来查看所得分数：

```bash
unix> make driver
unix> ./driver
```

# 实验指导

## 数据结构

```c
typedef struct {
    unsigned short red; /* R value */
    unsigned short green; /* G value */
    unsigned short blue; /* B value */
} pixel;
```



# Part A：Rotate

下面是文档中提供对 Rotate 的解题思路：

![思路](http://ww1.sinaimg.cn/large/c9caade4ly1g0fhbf9ecvj20le040t94.jpg)

![Picture1](http://ww1.sinaimg.cn/large/c9caade4ly1g0fhcw0syrj20gr0camxj.jpg)



