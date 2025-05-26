---
title: 直方图均衡化的实现
date: 2019-03-14 01:59:03
tags: [Image, Matlab, Algorithm]
description: 使用 MatLab 实现直方图均衡化
extra:
  mathjax: true
---

### 要求

- 已知: 彩色/灰度图
- 目标: 直方图均衡化
- 限制: 不准使用 `histeq` 函数

### 理论知识准备

要实现直方图均衡化, 最关键的问题在于找到均衡化的单调变换函数.

这一函数的推导课上已经给出:
![](https://image.inhzus.io/2025/05/40a6097a58e9de2134337db27f7e0a56.jpeg)

### 彩色图的两种实现思路

#### RGB 通道

这种实现的思路非常直接, 在将图片提取出 RGB 三通道后, 统计得到每个颜色值的数量, 即**直方图**.

<!--more-->

 为了避免重复计算带来的性能问题, 通过计数循环与累加, 得到 $f(x)=\frac{L}{A_0}\sum_{n=0}^{x}H_A(n),x=1,2,\ldots,256​$ 其中, $L​$ 为颜色值范围内的数量, RGB 情况下为 256; $A_0​$ 为图片的总像素数. 该函数存于一个 $L​$ 长度的数组中, 其中某个值的 index 就是它的自变量.

之后, 逐行逐列遍历图片, 代入上述得到的函数即可实现**直方图均衡化**.

#### HSV 通道

HSV 这种色彩表示方式在课上也有提到.按照 [维基百科](<https://zh.wikipedia.org/wiki/HSL%E5%92%8CHSV%E8%89%B2%E5%BD%A9%E7%A9%BA%E9%97%B4>) 介绍: 色相(H), 饱和度(S), 明度(V).

经过尝试, 在 H 通道进行直方图均衡化会使图片色调"崩坏", 而 S 通道可以使饱和度得到增强, V 通道也可以得到期待的效果.

##### 第一次算法的失败

起初, 我打算仍然采用 RGB 通道的思路. 不过由于 HSV 的值在 0~1 间, 无法将颜色值映射到某个固定长度的数组上, 所以我打算将所有颜色值进行排序后, 然后根据每个像素值进行累加计算得到目标值. 但算法**复杂度过高**使得很难运行出结果, 遂放弃这种思路.

##### 使用估值代替准确值

迫于无奈, 我选择了将 0~1 的值放缩至 0~255, 使用 RGB 通道的函数进行直方图均衡化, 之后再缩小回 0~1 这个范围, 尽管损失了精度, 但是根据实验结果看, 并没有明显的影响.

### 显示效果

#### 灰度

下图可以看出, 原图的色域较小, 而经过均衡化, 黑白对比非常明显.

![](https://image.inhzus.io/2025/05/fa76040865c81918aceead8731eef403.jpeg)

#### RGB

通过以下图可以看出, 颜色分布较原图更加广泛平均.

![](https://image.inhzus.io/2025/05/c89424fa922d212c4b97e1183f108a61.jpeg)

#### HSV

下图与上图相比较, 能够看出, 颜色本就已经鲜艳的地方饱和度变得更高, 同时照片的亮度对比度(上下对比)也变大.

![](https://image.inhzus.io/2025/05/b7a7d6cf744dae5234da1683cd5e99d4.jpeg)

#### 观察: 直方图均衡化放大了噪声

以下图片是我使用手机拍摄得到, 整体色调偏暗, 黑色较多, 经过 RGB 直方图均衡化后可以观察到图片中的电脑上有大量噪点.

![](https://image.inhzus.io/2025/05/59863de3a3a835d5f3be829d485372f0.jpeg)

### 具体实现

由于在之前并没有接触过 Matlab, 我对其最佳实现不甚了解. 还望指正. 

*使用代码时, 对于 RGB 和 HSV 不同的实现方法, 需要修改文件中 hsv 这一 bool 值.*

#### RGB

关键在于如何实现 `hist_equal(input_channel)`, 分三步: 求直方图, 求转换函数, 赋值.

##### 遇到的问题

- Matlab 语言的 index 从 1 开始
- 通道的数据类型必须为 `uint8`

##### 求直方图

```matlab
function ret = get_hist(channel)
    ret = zeros(256, 1);
    [height, width] = size(channel);
    for m = 1:height
        for n = 1:width
            ret(channel(m, n)) = ret(channel(m, n)) + 1;
        end
    end
end
```

逐行逐列统计即可.

##### 求转换函数

```matlab
conversion = zeros(256, 1);
cur = 0;
for i = 1:256
    cur = cur + src(i);
    conversion(i) = 256 * cur / pixels;
end
```

##### 赋值

```matlab
ret = zeros(height, width, 'uint8');
for i = 1:height
    for j = 1:width
        ret(i, j) = conversion(input_channel(i, j) + 1);
    end
end
```

#### HSV

对于 HSV, 其实只是通过放缩至 RGB 通道的值就可以实现均衡化, 再放缩回取值范围即可. 代码如下

```matlab
ret = double(hist_equal(uint8(256 * a))) / 256;
```

完整文件链接: [Histogram_equalization.m](<https://gist.github.com/inhzus/c86da05e91c5abd9c5835b9c35d9109e>)

