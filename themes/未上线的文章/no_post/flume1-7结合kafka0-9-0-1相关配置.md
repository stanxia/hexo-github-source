---
title: flume1.7结合kafka0.9.0.1相关配置
date: 2016-02-20 16:50:54
tags: flume
categories: 大数据
---

## 简介

利用flume1.7抓取数据，传入到kafka

## 配置文件设置

在flume/conf/新建一个 kafka.conf,修改该文件,相关配置如下：

```java
vi flume/conf/kafka.conf
#agent1表示代理名称
agent1.sources=source1
agent1.channels=channel1
agent1.sinks=sink1
#Spooling Directory是监控指定文件夹中新文件的变化，一旦新文件出现，就解析该文件内容，然后
写入到channle。写入完成后，标记该文件已完成或者删除该文件。
#配置source
#数据来源类型 spooldir表示 文件夹 ，command
agent1.sources.source1.type=spooldir
#指定监控的目录
agent1.sources.source1.spoolDir=/home/hadoop/logs
agent1.sources.source1.channels=channel1
agent1.sources.source1.fileHeader=false
agent1.sources.source1.interceptors=i1
agent1.sources.source1.interceptors.i1.type=timestamp
#配置channel1
agent1.channels.channel1.type=file
#channel数据存放的备份目录
agent1.channels.channel1.checkpointDir=/home/hadoop/channel_data.backup
#channel数据存放目录
agent1.channels.channel1.dataDir=/home/hadoop/channel_data
#配置sink1
agent1.sinks.sink1.type = org.apache.flume.sink.kafka.KafkaSink
#新版本开始使用如下配置：
agent1.sinks.sink1.kafka.bootstrap.servers=monsterxls:9092,slave1xls:9092,slave2xls:9092
#agent1.sinks.sink1.partition.key=0
#agent1.sinks.sink1.partitioner.class=org.apache.flume.plugins.SinglePartition
agent1.sinks.sink1.serializer.class=kafka.serializer.StringEncoder
agent1.sinks.sink1.max.message.size=1000000
agent1.sinks.sink1.producer.type=sync
agent1.sinks.sink1.custom.encoding=UTF-8
#新版本使用如下配置：
agent1.sinks.sink1.topic=stanxls
agent1.sinks.sink1.channel=channel1
```

## 小结

注意版本的问题。新版本改动了很多，在配置之前多看下帮助文档，了解下各种属性。