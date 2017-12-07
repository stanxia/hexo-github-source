---
title: mac使用小技巧
date: 2017-10-28 01:13:44
tags: mac
categories: mac
---
{% note info %}记录mac使用的小技巧
持续更新ing {% endnote %}

<!--请开始装逼-->
## 开启充电提示音（类似于iphone充电提示音，默认关闭）

终端输入（开启）：

```shell
defaults write com.apple.PowerChime ChimeOnAllHardware -bool true; open /System/Library/CoreServices/PowerChime.app & 
```

关闭：

```shell
defaults write com.apple.PowerChime ChimeOnAllHardware -bool false;killall PowerChime
```

<!-- more -->

## 隐藏文件夹

更好的保护学习资料，有时候需要设置隐藏文件夹：

```shell
mv foldername .foldername 
```

## 查看隐藏文件夹

mac最新版本：

```shell
⌘⇧.(Command + Shift + .)  #隐藏 和显示
```

## Macbook Pro 用外接显示器时，如何关闭笔记本屏幕，同时开盖使用

```shell
sudo nvram boot-args="iog=0x0" #(10.10以前版本)
sudo nvram boot-args="niog=1" #(10.10及以后版本)这个命令的意思就是外接显示器时关闭自身屏幕，重启生效
```

开机流程：连上电源和外接显示器，按开机键，立即合盖，等外接显示器有信号时开盖即可如果报错 (已知 10.11/10.12 会报错)
nvram: Error setting variable - 'boot-args': (iokit/common) general error
{% note info %}1. 重启，按住command + r 进入恢复界面
2. 左上角菜单里面找到终端，输入nvram boot-args="niog=1"，回车问题解决。重启生效{% endnote %}

<!--对不起，到时间了，请停止装逼-->


