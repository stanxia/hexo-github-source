---
title: Hive与Impala
date: 2016-02-23 23:31:13
tags: hive
categories: 大数据
---

## HiveQl执行过程：

1. 驱动模块

2. 编译器进行编译  Antlr
3. 优化器进行优化
4. 执行器执行（执行map reduce任务）   全表扫描* 不会执行 map reduce 任务

## HiveQL查询的 MapReduce 作业转化流程：

1. 用户输入sql

2. 抽象语法树 AST Tree
3. 查询块 QueryBlock
4. 逻辑查询计划  OperatorTree
5. 重写逻辑查询计划
6. 物理计划
7. 选择最优的优化查询策略
8. 输出

![1](http://wx4.sinaimg.cn/mw1024/6aae3cf3gy1fd0twkpnl9j20t2104wjn.jpg)



## Impala与Hive的区别：

1. Hive适合长时间的批处理查询分析；Impala适合实时的sql查询

2. Hive依赖于MapReduce，执行计划组合成管道形的MapReduce任务模式；Impala执行计划表现为一颗完整的执行计划树
3. Hive在查询过程中，内存不够用时会使用外存；Impala在内存不够用时，不会使用外存，所有Impala在查询的时候会存在一定的限制

![2](http://wx2.sinaimg.cn/mw1024/6aae3cf3gy1fd0twll2zoj20qc0myn04.jpg)



## Impala与Hive的相同点：

1. 使用相同的存储数据池，都支持 HDFS,HBase
2. 使用相同的元数据
3. 对SQL的解释处理比较相似，都是通过 语法分析 生成执行计划