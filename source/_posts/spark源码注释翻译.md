---
title: spark源码注释翻译
date: 2017-11-06 16:57:05
tags: spark
categories: spark
---

{% note info %}
版本：spark2.1.1
目的：方便中文用户阅读源码，把时间花在理解而不是翻译上{% endnote %}

<!--请开始装逼-->

## 初衷

开始立项进行翻译，一方面方便日后阅读源码，另一方面先粗粒度的熟悉下spark框架和组件。优化完之后希望能帮助更多的中文用户，节省翻译时间。

<!-- more -->

## 进度

已完成：

正在作：spark core模块

|      模块名      |            模块介绍            | 完成度  |
| :-----------: | :------------------------: | :--: |
|      api      |                            |      |
|   broadcast   |                            |      |
|    deploy     |                            |      |
|   executor    | 执行器：用于启动线程池，是真正负责执行task的部件 | 已完成  |
|     input     |                            |      |
|   internal    |                            |      |
|      io       |                            |      |
|   launcher    |                            |      |
|    mapred     |                            |      |
|    memory     |                            |      |
|    metrics    |                            |      |
|    network    |                            |      |
|    partial    |                            |      |
|      rdd      |                            |      |
|      rpc      |                            |      |
|   scheduler   |    调度器：spark应用程序的任务调度器     | 正在作  |
|   security    |                            |      |
|  serializer   |                            |      |
|    shuffle    |                            |      |
| status.api.v1 |                            |      |
|    storage    |                            |      |
|     util      |                            |      |

<!--对不起，到时间了，请停止装逼-->


