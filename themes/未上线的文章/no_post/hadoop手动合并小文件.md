---
title: hadoop手动合并小文件
date: 2016-03-06 23:11:30
tags: hadoop
categories: 大数据
---

## 前提

下载主要jar包：[filecrush-2.0-SNAPSHOT.jar]( https://pan.baidu.com/s/1eSqWp9o)密码: x9mh

## 执行

```sh
Hadoop jar filecrush-2.0-SNAPSHOT.jar crush.Crush \

-Ddfs.block.size=134217728 \

--input-format=text  \

--output-format=text \

--compress=none \

/要合并的目录 /合并到哪里去 时间戳(20170221175612)

```





Good Luck!