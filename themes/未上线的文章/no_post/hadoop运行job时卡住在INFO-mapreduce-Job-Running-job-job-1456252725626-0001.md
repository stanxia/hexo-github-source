---
title: 'hadoop运行job时卡住在INFO mapreduce.Job: Running job:job _1456252725626_0001'
date: 2016-02-25 14:30:49
tags: hadoop问题
categories: 问题集锦
---

## 问题详情：

在运行hadoop MapReduce任务时，一直卡在INFO mapreduce.Job: Running job:job _ 1456252725626 _ 0001的位置。

<!-- more -->

## 问题原因：

##### 遇到了问题，首先从哪里找？

当发现问题的时候，首先去查看相应的日志，查找问题产生的原因，对症下药。

在hadoop2.6以及以上版本中，mapreduce任务都是交给yarn资源管理器 管理的，所以首先去 查看 **yarn-hadoop-nodemanager-slave01.log** 日志。

终端输入：

`cat hadoop所在目录/logs/yarn-hadoop-nodemanager-slave01.log `

如果显示如下信息：

```log
INFO org.apache.hadoop.ipc.Client: Retrying connect to server: 0.0.0.0/0.0.0.0:8031. Already tried 4 time(s); retry policy is RetryUpToMaximumCountWithFixedSleep(maxRetries=10, sleepTime=1000 MILLISECONDS)
```

字面上的意思每次尝试连接0.0.0.0/0.0.0.0:8031失败，google 百度一下 找到如下办法（<u>不能保证百分百有效果，但也是值得一试的</u>）

## 解决方案：

从问题可以推断出，可能是 配置文件有问题，打开文件 hadoop文件目录/etc/hadoop/yarn-site.xml查看详细配置；

终端中输入：

```shell
vi hadoop目录/etc/hadoop/yarn-site.xml
```

 添加如下 配置到 文件中：（只需要将下面的 monsterxls修改为你集群中的namenode所在的主机名即可 ）

```xml
<property>  
    <name>yarn.resourcemanager.address</name>  
    <value>monsterxls:8032</value>  
  </property>  
  <property>  
    <name>yarn.resourcemanager.scheduler.address</name>  
    <value>monsterxls:8030</value>  
  </property>  
  <property>  
    <name>yarn.resourcemanager.resource-tracker.address</name>  
    <value>monsterxls:8031</value>  
  </property> 
```

**注意：**集群中的每台服务器都修改一下。

## 重启hadoop服务再试试mapreduce

重启服务，执行任务试试效果。

Good Luck!

