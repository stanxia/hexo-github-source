---
title: zookeeper启动时数组越界异常
date: 2016-02-14 13:39:29
tags: zookeeper
categories: 问题集锦
---


# 问题
启动zookeeper时，出现以下异常信息：

![1](http://wx1.sinaimg.cn/mw1024/6aae3cf3gy1fcsp95necuj21ji0jidla.jpg)

# 解决方案

修改 ／zookeeper/conf/zoo.cfg文件
修改服务器id和ip映射时注意格式为：
```xml
vi /zookeeper/conf/zoo.cfg
server.1=host:port:port或者host:port或者host:port:port:type
```
