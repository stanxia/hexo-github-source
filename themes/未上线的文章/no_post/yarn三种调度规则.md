---
title: yarn三种调度规则
date: 2016-02-16 14:23:32
tags: yarn
categories: 大数据
---

## yarn三种调度机制

1. FIFO Scheduler先进先出调度机制

2. Fair Scheduler公平调度机制

3. Capacity Scheduler容量机制

   <!-- more -->


## FIFO Scheduler

按照先进先出的调度机制，所有的application将按照提交的顺序来执行，这些application都放在一个队列里面，顺序执行，执行完一个之后，才会执行下一个。

##### 缺点：

如果任务耗时长，后面提交的任务会一直处于等待状态，影响效率。所以只适合单人跑任务。

面对以上缺点，yarn提出了另两种策略，更加适合共享集群。

![3](http://wx2.sinaimg.cn/mw1024/6aae3cf3gy1fcsouzgzg1j20q80eqabe.jpg)



## Capacity Scheduler

定位：多人共享调度器。

机制：为每人分配一个队列，每个队列占用集群固定的资源，每个队列占用的资源可以不同，每个队列内部还是按照FIFO的策略。

特性：queue elasticity （弹性队列）根据实际情况分配资源

Capacity Scheduler 的队列时支持层级关系的：

![capacity1](http://wx1.sinaimg.cn/mw1024/6aae3cf3gy1fcsoa7b1tkj216m068mxf.jpg)

相关配置如下：

![capacity2](http://wx2.sinaimg.cn/mw1024/6aae3cf3gy1fcsoa7u5psj20vy13m44h.jpg)

![1](http://wx3.sinaimg.cn/mw1024/6aae3cf3gy1fcsouw6wsej20p60g8401.jpg)



##### 队列设置

如果是mapreduce任务，通过 `mapreduce.job.queuename`来设置执行队列。

## Fair Scheduler

机制：为每一个任务均匀分配资源，一个任务就可以用整个集群资源，两个任务就平分集群资源，依次类推。

![2](http://wx3.sinaimg.cn/mw1024/6aae3cf3gy1fcsouwra8hj20og0ekjsr.jpg)



##### 开启Fair Scheduler 

在**yarn-site.xml**中设置 `yarn.resourcemanager.scheduler.class`为`org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler` 。NOTE:CDH默认的就是Faire Scheduler ，CDH并不支持 Capacity Scheduler.

##### 队列设置

设置fair-scheduler.xml文件，可参考下图：

![fair](http://wx2.sinaimg.cn/mw1024/6aae3cf3gy1fcsortg5yjj215c0litcz.jpg)



