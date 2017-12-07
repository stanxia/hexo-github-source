---
title: spark关于parquet的优化
date: 2017-11-01 14:53:13
tags:
- spark
- parquet
categories: spark
---
{% note info %}parquet是一种列式存储。可以提供面向列的存储和查询。{% endnote %}
## Parquet的优势
在sparkSQL程序中使用parquet格式存储文件，在存储空间和查询性能方面都有很高的效率。

### 存储方面
因为是面向列的存储，同一列的类型相同，因而在存储的过程中可以使用更高效的压缩方案，可以节省大量的存储空间。

### 查询方面
在执行查询任务时，只会扫描需要的列，而不是全部，高度灵活性使查询变得非常高效。
<!-- more -->
### 实例测试
| 测试数据大小 | 存储类型     | 存储所占空间 | 查询性能 |
| ------ | -------- | ------ | ---- |
| 1T     | TEXTFILE | 897.9G | 698s |
| 1T     | Parquet  | 231.4G | 21s  |

## Parquet的使用
使用parquet的简单demo：

```scala
// Encoders for most common types are automatically provided by importing spark.implicits._
import spark.implicits._

val peopleDF = spark.read.json("examples/src/main/resources/people.json")

// DataFrames can be saved as Parquet files, maintaining the schema information
peopleDF.write.parquet("people.parquet")

// Read in the parquet file created above
// Parquet files are self-describing so the schema is preserved
// The result of loading a Parquet file is also a DataFrame
val parquetFileDF = spark.read.parquet("people.parquet")

// Parquet files can also be used to create a temporary view and then used in SQL statements
parquetFileDF.createOrReplaceTempView("parquetFile")
val namesDF = spark.sql("SELECT name FROM parquetFile WHERE age BETWEEN 13 AND 19")
namesDF.map(attributes => "Name: " + attributes(0)).show()
// +------------+
// |       value|
// +------------+
// |Name: Justin|
// +------------+
```
## Parquet 的问题
 spark 写入数据到 hive 中，使用 Parquet 存储格式，查询该表时报错如下：

``` scala
Error: java.io.IOException: org.apache.parquet.io.ParquetDecodingException: Can not read value at 0 in block -1 in file
```

当时设置的字段属性为：
![](http://oliji9s3j.bkt.clouddn.com/15118606648730.jpg)

经过比对，发现是 decimal 类型出了问题，查询 decimal 的字段时候就会报错，而查询其他的并不会报错。（这应该是 spark 引起的，因为在 hive 客户端执行 decimal 类型的操作时并不会出错。）

查阅网上，也有些朋友遇到了类似的事情，应该是官方的 bug ，暂时的解决办法是:

``` scala
1. 将 Parquet 的存储格式转换为 ORC 
2. 或将 decimal 换为 double 类型存储字段
```

<!--对不起，到时间了，请停止装逼-->


