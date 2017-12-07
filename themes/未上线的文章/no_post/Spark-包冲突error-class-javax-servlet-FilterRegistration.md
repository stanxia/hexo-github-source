---
title: Spark 包冲突error class javax.servlet.FilterRegistration
date: 2016-02-27 20:04:37
tags: spark
categories: 问题集锦
---

## 问题描述

在idea上运行spark程序时，出现以下信息：

Spark error class "javax.servlet.FilterRegistration"'s signer information does not match signer information of other classes in the same package

如图：

![1](http://wx3.sinaimg.cn/mw1024/6aae3cf3gy1fd5a3imie6j21kw09oq96.jpg)

看了一圈网上的答案，应该是包冲突，试过了各种方法，终于找到了一种答案，非常有效。

<!-- more -->

## 解决方案

右键模块项目==>Open Module Settings ==> 选择Dependencies==>找到javax.servlet:servlet-api:xx==>移动到列表的最末端，如下图：

![2](http://wx4.sinaimg.cn/mw1024/6aae3cf3gy1fd5amqp9jcj21ak0hu425.jpg)

Apply==>Ok==>运行试试！



Good Luck!