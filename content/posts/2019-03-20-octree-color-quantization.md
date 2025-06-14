---
title: 八叉树颜色量化
date: 2019-03-20 19:58:40
tags: [Image, Algorithm, C++]
description: 使用 C++ 实现八方图颜色量化的各种算法
extra:
  mathjax: true
---

### 实验目的

实现 BMP 文件从真彩色到 256 色的颜色量化算法

### 实验原理

在计算机图形学中, 颜色量化是应用于颜色空间的一种量化方法. 它能够减少一张图片中不同颜色的数量, 使得到的新图像在视觉上和原图像非常相似, 且使占用.

<style>
table {
    width: auto;
    height: auto;
}
</style>


|       | 0    | 1    | 2    | 3    | 4    | 5    | 6    | 7    |
| ----- | ---- | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
| blue  | 1    | 0    | 0    | 0    | 1    | 0    | 1    | 0    |
| green | 0    | 1    | 1    | 0    | 0    | 0    | 1    | 0    |
| red   | 0    | 0    | 1    | 1    | 0    | 1    | 0    | 0    |

对于真彩色图像的某个像素, 从高位至低位, 每位的三个颜色通道可以使用 0-7 的数字表示, 因此可看做共有八层的八叉树. 在颜色量化过程中, 不断地对八叉树进行剪枝, 直到"叶子"数量小于等于 256, 则得到一个有 256 个颜色的调色板. 

#### 剪枝策略

剪枝的策略实现我考虑了如下两种.

<!--more-->

- 构建八叉树时, 将所有"叶子"节点构建为最小堆, 从最小堆中逐个出栈, 此时, 父节点变为叶子节点, 压栈. 同时,  修改父节点的颜色值.
  $$
  parent.color=\frac{parent.color\cdot{parent.mix}+child.color\cdot{child.cnt}}{parent.mix+child.cnt}
  $$
  `node.mix`: 父节点已归并的子节点的像素数量

  `node.cnt`: 该节点的像素数量

  当 `node.mix == node.cnt`, 该节点的所有子节点都已经被归并剪枝.

- 构建八叉树时, 将不同深度的节点插入至不同数组中. 从倒数第二层开始, 排序后(按像素数量)逐个删除其子节点, 颜色数减少 (非空子节点数 - 1). 修改父节点的颜色值.
  $$
  parent.color=\frac{\sum_{i=0}^{7}{child.color\cdot{child.cnt}}}{parent.cnt}
  $$

以上两个策略的区别在于

- 策略一, 在极端情况下, 占比大的颜色最后一层都没有进行归并, 而占比少的颜色, 已经归并至无法分辨.

  故, 该策略会把占比比较少的颜色直接无视.

- 策略二, 由于是逐层归并, 占比大的颜色的细节可能失真, 但占比小的颜色细节可以更好地保持.

**实际使用中发现, 在主题颜色占比大的情况下, 策略一极为鸡肋, 占用过多调色板位置. 而在颜色饱和度较高的情况下, 策略一过多地渲染了占比偏大的颜色, 而使得占比偏小细节丰富的颜色归并过多导致失真. 因此舍弃.**

#### 抖动算法

在渲染天空时, 由于天空的渐变色颜色过多, 考虑使用抖动算法 [Floyd-Steinberg dithering](<https://en.wikipedia.org/wiki/Floyd%E2%80%93Steinberg_dithering>). 其将当前像素的调色板匹配颜色和原颜色的差值 作用于 即将匹配调色板的像素上. 抖动矩阵为:
$$
\begin{bmatrix}
& & * & \frac{\displaystyle 7}{\displaystyle 16} & \ldots \\
\ldots & \frac{\displaystyle 3}{\displaystyle 16} & \frac{\displaystyle 5}{\displaystyle 16} & \frac{\displaystyle 1}{\displaystyle 16} & \ldots \\
\end{bmatrix}
$$
经过实战我认为, 对于**照片**, 抖动算法能够在较小程度上使得生成的图片的过渡较为平缓, 但对于**插画**, 由于其构图一般情况下颜色边界较为明显, 抖动算法会使得两种颜色的边界变得模糊. 为了避免作用于插画时带来的负面效果, 代码中未启用抖动算法部分. 

[GitHub Repository](https://github.com/inhzus/Octree-Color-Quantization)

### 实验结果

以下实验结果均在策略二**, **关闭**抖动转换**条件下生成.

#### Pic A

![](https://image.inhzus.io/2025/05/bf8e12bdfd08e377057271697be0a9a9.jpeg)

- 天空和水面的颜色过渡不够自然(策略一时天空颜色会更为丰富), 相比策略一颜色相对丰富. 
- 水面倒印的四处光源的颜色基本与原图一致. 说明逐层剪枝的方法有效避免了剪去关键特征颜色, 即便剪枝, 父节点保持了子节点的加权平均值, 使得颜色不会过于失真.

#### Pic B

![](https://image.inhzus.io/2025/05/fe1f31edac840770a5ce6a0dd791304f.jpeg)

Windows 画图生成的图片过于垃圾, 为了节省版面, 省略.

过渡依然不是很流畅, 但作为颜色量化图效果还可以.

#### Pic 3

![](https://image.inhzus.io/2025/05/b032287ad14740dfa2d5e4852eee6680.jpeg)

对于这种颜色不是过于丰富的图来说, 生成的 256 色图片相比原图很难看出有什么不同. 但对于渐变色较多的图片来说, 生成的图片就不是很能令人满意.

### 实验小结

本次实验过程中涉及到了

- BMP 文件的读写
- 八叉树颜色量化的剪枝问题

八叉树颜色量化问题最大的困难我认为在于如何进行渐变色的处理, 尽管有抖动算法, 但是网上的相关资料不是很丰富, 再加上可能是我的算法具体问题, 在使用时效果并不是很好.