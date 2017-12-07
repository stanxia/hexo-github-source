---
title: MAC应用无法打开或文件损坏的处理方法
date: 2015-02-15 14:17:24
tags: mac
categories: mac
---

## 问题

下载了一些程序之后，却发现无法在MAC中安装，安装时会弹出下图所示警告框：“打不开 xxx，因为它来自身份不明的开发者”

![img1](http://img.xclient.info/attachment/cdn/large/006ehIt6jw1execfbx4xnj30nc0b6dgh.jpg)

<!-- more -->

## 原因

在MAC下安装一些软件时提示"来自身份不明开发者"，其实这是MAC新系统启用了新的安全机制。
默认只信任 **Mac App Store** 下载的软件和拥有开发者 ID 签名的应用程序。
换句话说就是 MAC 系统默认只能安装靠谱渠道（有苹果审核的 **Mac App Store**）下载的软件或被认可的人开发的软件。

这当然是为了用户不会稀里糊涂安装流氓软件中招，但没有开发者签名的 “老实软件” 也受影响了，安装就会弹出下图所示警告框：“打不开 xxx，因为它来自身份不明的开发者”。

## 解决方案

1. 最简单的方式：按住Control后，再次点击软件图标，即可。

2. 修改系统配置：系统偏好设置... -> 安全性与隐私... ->通用... ->选择任何来源。![img2](http://ww2.sinaimg.cn/large/006ehIt6jw1exed22xlgpj30os0m6ae7.jpg)

   ## ![imag3](http://ww2.sinaimg.cn/large/006ehIt6jw1exed2kg4wbj30oe0jqtbd.jpg)

   ## ![img4](http://ww2.sinaimg.cn/large/006ehIt6jw1exed0cuqtyj30oe0js77b.jpg)

3. ***macOs Sierra 10.2***以上版本，打开<u>终端</u>，执行:`sudo spctl --master-disable` 就可以啦。