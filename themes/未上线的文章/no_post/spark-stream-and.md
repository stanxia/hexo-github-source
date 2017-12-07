---
title: spark stream无法收取到kafka生产者的消息
date: 2016-03-02 15:26:57
tags: spark
categories: 问题集锦
---

## 问题

spark stream无法收取到kafka发过来的消息

## 解决方案

找到集群中 kafka的server.properties，修改如下：

`vi server.properties`

```properties
listeners=PLAINTEXT://slave2xls:9092    #改为本主机的ip：port
port=9092
```

![1](http://wx2.sinaimg.cn/mw1024/6aae3cf3gy1fd8jhc2u2dj21fi09ct9t.jpg)

