---
title: XPS-9360 安装 Linux 的体验
date: 2019-06-17 00:12:31
tags: [Linux, Configuration]
description: 在 XPS-9360 上安装各个 Linux 发行版的体验
---

# 起因

XPS-9360 对 Linux 的兼容性是众所周知的，考虑到 Windows 的以下问题：

- 注册表对于强迫症比较难受。

- Linux 上可以方便地使用各种 Windows 不支持的 C++ 第三方库，且 Windows 不能提供 32 位子系统用于一些操作系统方面的实验。

- 环境配置过于恶心。

决定装双系统体验 Linux。

配置：Intel Core i7-8550U @ 8x 4GHz, 16 GB, Mesa DRI Intel(R) UHD Graphics 620 (Kabylake GT2)

# Ubuntu 18.04

## 安装

第一次安装双系统比较困扰，后来发现事实上，只需要关闭 quick boot 和 AHCI，为系统预留一定的空间，直接从刻录系统的 U 盘启动系统就能进入安装流程。

美化过程参考 [知乎美化教程](https://zhuanlan.zhihu.com/p/37314255)。必须使用的软件主要有：

|软件|备注|
|---|---|
|zsh|替代 bash，使用 oh-my-zsh 进行美化，传闻启动较慢，对我来说 0.5s 完全可以接受|
|clash|翻墙软件，替代 SS/SSR|
|proxychains-ng, graftcp|翻墙普遍做法是设置 http_proxy 等，实际操作认为不是每个操作都需要翻墙，故选择使用 proxychains-ng，graftcp 用于 go 程序的代理|
|fusuma|由于 linux 不支持复杂的触摸板手势，故使用这一程序设置手势如关闭标签，切换标签等|

其他同 Windows.

<!--more-->

## 评价

### 优点

- 我很不理解论坛里的一种论调即 Ubuntu 很丑，我个人觉得 Ubuntu 稍作美化就能比很多其它发行版好看很多（当然这一点也和自己的审美有关，个人偏向带 dock、顶部任务栏的风格，不过 Ubuntu 想要变好看确实是需要改一改的。

- 国内的讨论非常多。

### 缺点

- 尽管笔记本配置已经相当不错，然而某些动画仍然有让人难受的卡顿感觉。

- Ubuntu 的 ppa 过于麻烦，apt package 版本陈旧。

- 续航尿崩。原本使用 Windows 续航可达 10h, 使用 Ubuntu 可能只有 6h 左右。

# Manjaro kde

由于无法忍受 Ubuntu 差劲的续航，考虑使用内存和 GPU 占用更少的桌面环境。在这期间，发现很多关于 Manjaro 的推荐，打算使用 Manjaro，考虑到：xfce 和 mate 过于简陋，gnome 无法解决续航问题，考虑使用人数比较多的 kde。

## 优点

解决了 Ubuntu 缺点中的前两点问题，kde 相比 gnome 确实不怎么卡顿，pacman & aur 的包管理很舒服。

## 缺点

- 勉强能看的界面，不符合个人审美。

- 续航依然尿崩，在把各种乱七八糟的进程都清理一遍后，可能比 gnome 好那么百分之 5%？

# Manjaro i3

在两三个月之前曾经尝试安装 i3wm，但是发现需要一定的学习成本，而那时候的课业负担有些重，体验了一下就直接格盘了。在多次尝试各种 de 未果之后，打算再次尝试 i3。

## 优点

- 由于只有 bottom bar，没有各种 dock 等等，特别是在隐藏 bottom bar 后，整个电脑好不好看就已经和 i3 没有任何关系了。

- 续航太好了，果然这种动画比较少，内存占用低的 wm 非常省电，续航居然可能有 11 个小时。

- 配置不复杂，还能达到装逼的效果。

## 缺点

- 需要一定的学习成本。

- 很多设置不能使用图形界面，比如启动项需要手动在 i3config 中写入。不过这些折腾都比较简单，还能接受。

## 展示

最后附两张截图，几乎没有进行过显示上的配置。

![](https://image.inhzus.io/2025/05/949f161c31f77cc0262d8bc347c7c157.png)

![](https://image.inhzus.io/2025/05/640d32dd27c97d548fb207edaeb568f4.png)
