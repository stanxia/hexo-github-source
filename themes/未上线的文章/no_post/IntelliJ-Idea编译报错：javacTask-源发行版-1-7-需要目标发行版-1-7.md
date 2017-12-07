---
title: 'IntelliJ Idea编译报错：javacTask: 源发行版 1.8 需要目标发行版 1.8'
date: 2016-02-21 13:04:48
tags: IDEA
categories: 问题集锦
---

## 问题一：java compiler error

运行java程序时，编译报错：java compiler error，如下图所示：

![1](http://wx2.sinaimg.cn/mw1024/6aae3cf3gy1fcy0jlezsnj214m08ytaq.jpg)

<!-- more -->

## 解决办法

打开idea设置=>>Build,Execution,Deployment=>>Compiler=>>Java Compiler=>>左边框Pre-module bytecode version =>>找到程序所在的模块=>>Target Bytecode version 选择提示中的需要目标发行版本=>>Apply=>>ok,如下图所示：

![2](http://wx4.sinaimg.cn/mw1024/6aae3cf3gy1fcy0jpf2uzj21kw0x6q9o.jpg)

## 问题二：Usage of API documented as  @since 1.6/1.7/...

当使用了一些api之后，idea会提示Usage of API documented as  @since 1.6/1.7/…如下图所示：

![3](http://wx4.sinaimg.cn/mw1024/6aae3cf3gy1fcy0jq6pbnj20t809wwga.jpg)

## 解决方案：

右键项目=>>open module setting=>>Laguage Level =>>选择（大于或等于）提示中@since的版本，如下图所示：

![4](http://wx2.sinaimg.cn/mw1024/6aae3cf3gy1fcy0jpvbxfj21kw0xzahh.jpg)

