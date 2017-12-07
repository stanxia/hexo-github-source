---
title: spark报错集
date: 2017-10-30 13:58:58
tags: spark
categories: spark
---
## 有话要说
针对一个老毛病：有些错误屡犯屡改，屡改屡犯，没有引起根本上的注意，或者没有从源头理解错误发生的底层原理，导致做很多无用功。

总结历史，并从中吸取教训，减少无用功造成的时间浪费。特此将从目前遇到的spark问题全部记录在这里，搞清楚问题，自信向前。
<!--more-->
## 问题汇总

### 关键词：spark-hive

#### 概述：
```
Exception in thread "main" java.lang.IllegalArgumentException: Unable to instantiate SparkSession with Hive support because Hive classes are not found.
```

#### 场景：
```
在本地调试spark程序，连接虚拟机上的集群，尝试执行sparkSQL时，启动任务就报错。
```

#### 原理：
```
缺少sparkSQL连接hive的必要和依赖jar包
```

#### 办法：
```
在项目／模块的pom.xml中添加相关的spark-hive依赖jar包。
<!-- https://mvnrepository.com/artifact/org.apache.spark/spark-hive_2.11 -->
<dependency>
    <groupId>org.apache.spark</groupId>
    <artifactId>spark-hive_2.11</artifactId>
    <version>2.1.1</version>
    <scope>provided</scope>
</dependency>
重新编译项目／模块即可。
```




<!--对不起，到时间了，请停止装逼-->

