---
title: spark取样函数分析
date: 2017-12-12 17:30:25
tags: spark
categories: spark
---
{% note info %}Spark取样操作
无法获取随机样本的解决方案{% endnote %}

<!--请开始装逼-->

## 背景

Dataset中sample函数源码如下：

```scala
/**
  * Returns a new [[Dataset]] by sampling a fraction of rows, using a user-supplied seed.
  *
  * 通过使用用户提供的种子，通过抽样的方式返回一个新的[[Dataset]]。
  *
  * @param withReplacement Sample with replacement or not.
  *                        如果withReplacement=true的话表示有放回的抽样，采用泊松抽样算法实现.
  *                        如果withReplacement=false的话表示无放回的抽样，采用伯努利抽样算法实现.
  * @param fraction        Fraction of rows to generate.
  *                        每一行数据被取样的概率.服从二项分布.当withReplacement=true的时候fraction>=0,当withReplacement=false的时候 0 < fraction < 1.
  * @param seed            Seed for sampling.
  *                        取样种子（与随机数生成有关）
  * @note This is NOT guaranteed to provide exactly the fraction of the count
  *       of the given [[Dataset]].
  *       不能保证准确的按照给定的分数取样。（一般结果会在概率值*总数左右）
  * @group typedrel
  * @since 1.6.0
  */
def sample(withReplacement: Boolean, fraction: Double, seed: Long): Dataset[T] = {
  require(fraction >= 0,
    s"Fraction must be nonnegative, but got ${fraction}")

  withTypedPlan {
    Sample(0.0, fraction, withReplacement, seed, logicalPlan)()
  }
}

/**
  * Returns a new [[Dataset]] by sampling a fraction of rows, using a random seed.
  *
  * 通过程序随机的种子，抽样返回新的DataSet
  *
  * @param withReplacement Sample with replacement or not.
  *                        取样结果是否放回
  * @param fraction        Fraction of rows to generate.
  *                        每行数据被取样的概率
  * @note This is NOT guaranteed to provide exactly the fraction of the total count
  *       of the given [[Dataset]].
  *       不能保证准确的按照给定的分数取样。（一般结果会在概率值*总数左右）
  * @group typedrel
  * @since 1.6.0
  */
def sample(withReplacement: Boolean, fraction: Double): Dataset[T] = {
  sample(withReplacement, fraction, Utils.random.nextLong)
}
```

<!-- more -->

## 问题

结果数据的行数一般在（fraction*总数）左右。没有一个固定的值，如果需要得到固定行数的随机数据的话不建议采用该方法。

## 办法

获取随机取样的替代方法：

```scala
df.createOrReplaceTempView("test_sample"); // 生成临时表
df.sqlContext() // 添加随机数列，并根据其进行排序
  .sql("select * ,rand() as random from test_sample order by random")
  .limit(2) // 根据参数的fraction计算需要获取的取样结果
  .drop("random") // 删除掉添加的随机列
  .show();
```

<!--对不起，到时间了，请停止装逼-->

