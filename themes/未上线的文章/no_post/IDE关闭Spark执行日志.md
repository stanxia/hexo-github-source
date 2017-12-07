---
title: IDE关闭Spark执行日志
date: 2016-02-28 11:25:24
tags: spark
categories: 大数据
---

## 引言

用IDE执行spark任务会出现很多的日志信息，当我们不需要看这些日志信息的时候该怎么整，如何优雅的搞掉这些日志，看看下面的，或许对你有帮助。

## 内容

添加以下代码：

```scala

import org.apache.log4j.{Level, Logger}

//设置spark的日志级别为 warn，才会打印日志
 Logger.getLogger("org.apache.spark").setLevel(Level.WARN)
//直接关闭 jetty日志  
 Logger.getLogger("org.eclipse.jetty.server").setLevel(Level.OFF)
//直接关闭 spark的运行日志
 SparkContext().setLogLevel("OFF")

```

## groupBy&&cube&&rollup

### 比较

### 测试

```scala
// 创建sparkSession
val ss = SparkSession.builder().appName("test").master("local").enableHiveSupport().getOrCreate()
// 制造数据
val data = Seq(
  (1000.00, "a", "b", "c"),
  (1000.00, "a", "b", "c"),
  (1000.00, "a", "b", "c")
)
// 关闭spark的运行日志
ss.sparkContext.setLogLevel("off")
// 倒入隐式转换
import ss.implicits._
// 转换dataFrame
val df = ss.sparkContext.parallelize(data,2).toDF("sales","a", "b", "c")
// 测试 cube
df.cube($"a",$"b",$"c").agg(Map("sales" -> "sum")).show()
// 测试 groupBy
df.groupBy($"a",$"b",$"c").agg(Map("sales" -> "sum")).show()
// 测试rollup
df.rollup($"a",$"b",$"c").agg(Map("sales" -> "sum")).show()

```

结果：

