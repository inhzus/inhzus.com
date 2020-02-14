---
title: "Android 10 Launcher"
date: 2020-02-14T15:52:34+08:00
tags: [android]
---

过去将所有 Icon 不支持 i18n 的 App （大部分也都是毒瘤软件）放入冰箱，Drawer 中的 Icon 风格比较统一好看，最近开始使用”钱迹“，使用频率比较高，放入冰箱极不方便，遂打算找一支持 Rename Icon、且功能和使用与 Pixel Launcher 基本一致的 Launcher。以下为折腾记录。

## Launchers

### Action Launcher: Pixel Edition

标榜为 Customizable Pixel Launcher，免费版本不支持大多数的功能，Plus 版本 \$6.99，附加功能 Action Dash Plus \$4.99，烧钱，幸好及时退款止损。实测体验如下：

cons:

- Switch apps 必须 swipe pill upward，在 home screen 重复前述操作动画别扭：search box 向上跳动后消失时出现 overview 界面；
- Overview 无 app suggestions；
- 不支持 swipe pill horizontally to quickly switch app；
- 曾出现 bug：overview 界面为空。

pros:

- 默认主题非常舒适好看，略好于 Pixel Launcher，如 shortcuts 动画、 shutter 动画等；
- Open app animation 主题 Adaptive Zoom（Icon 颜色全屏铺满后打开）好看；
- Shutter 功能有些意思（并无卵用）；
- Home screen swipe right to show quick drawer，swipe left to show quick page。

### Hyperion

同样基于 Pixel Launcher，设置界面极简，风格比较极端（背景全透明，从上至下渐变色偏红），drawer app suggestions 需要 Plus \$1.99。

pros:

- 相比 Action Launcher，支持 quickly switch app，overview 界面动画较自然。

cons:

- 略有卡顿；
- Overview 无 app suggestions。

### Lawnchair 2

pros:

- 设置风格与 Pixel Launcher 无缝衔接；
- At a glance 支持自定义天气源与位置；
- hyperion 所支持的全部支持；
- 开源项目。

cons:

- 无 app suggestions in overview。

### Rootless Pixel Launcher

垃圾，Pixel Launcher 支持的它不支持，Pixel Launcher 不支持的它也不支持。

### Nova Launcher

最流行的 Launcher，无感，缺点同 Lawnchair 2。

### Evie

设置操作麻烦，功能较少且 UI 垃圾，累了，缺点同上。

倒是很有自知之明的在设置界面最下边提供了 "Switch to another launcher" 有点意思。

### Microsoft Launcher

很多组件自成体系，但本人欣赏不来巨硬的审美，关键问题同上。

### Smart Launcher 5

完全不支持 gesture navigation，UI 风格过于奇葩，app 分类自作聪明，慎入。

### ADW Launcher 2

2020 年了还有不支持 Live Wallpaper 的 Launcher，打开之后默认设置 home screen 一片绿，尝试设置 wallpaper 并不奏效，丑吐了。

### Apex Launcher

免费版全屏广告极多，懒得看有什么”特异功能“了，关键问题同 Nova Launcher。

### POCO Launcher

貌似是小米家的，不支持 system default light/dark mode，app categories 的 tabs 太多了，还不如直接翻 drawer，有些鸡肋（也有可能我姿势不到位吧），关键问题同上。

## 后续

根据 [GItHub issue: Dock in overview with Android 10 gesture navigation](https://github.com/LawnchairLauncher/Lawnchair/issues/1856)，可知，目前想要实现 app suggestions in overview 需要使用 root + Magisk + QuickSwitch module。

在另一 [issue](https://github.com/LawnchairLauncher/Lawnchair/issues/1885) 中提到了某种实现的可能性。

## Sesame

想起另一个有意思的 app：Sesame，之前在 Pixel Launcher 上使用体验极差，但集成在 third-party launcher（包括 Nova，Lawnchair，Hyperion）上，替换掉 Google Now，体验不错，至少这个位置不再是废物。

## 总结

目前为止没有支持 overview app suggestions 的 third-party launcher。功能上来说，最丰富的是 Action Launcher: Pixel Edition，花里胡哨的东西很多；个人最喜欢的是 Lawnchair，对开源软件有天然好感；Hyperion 的风格眼前一亮，不过经不起细细使用。

还是返回我 Pixel Launcher，期待 Google 开放更多的 API 方便开发者进一步优化。