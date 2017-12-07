---
title: mongodb操作指南
date: 2016-03-15 23:15:33
tags: MongoDB
categories: 数据库
---

## 什么是MongoDB 

MongoDB 是由C++语言编写的，是一个基于分布式文件存储的开源数据库系统。
在高负载的情况下，添加更多的节点，可以保证服务器性能。
MongoDB 旨在为WEB应用提供可扩展的高性能数据存储解决方案。
MongoDB 将数据存储为一个文档，数据结构由键值(key=>value)对组成。MongoDB 文档类似于 JSON 对象。字段值可以包含其他文档，数组及文档数组。

<!-- more -->

## 主要特点

MongoDB的提供了一个面向文档存储，操作起来比较简单和容易。
你可以在MongoDB记录中设置任何属性的索引 (如：FirstName="Sameer",Address="8 Gandhi Road")来实现更快的排序。
你可以通过本地或者网络创建数据镜像，这使得MongoDB有更强的扩展性。
如果负载的增加（需要更多的存储空间和更强的处理能力） ，它可以分布在计算机网络中的其他节点上这就是所谓的分片。
Mongo支持丰富的查询表达式。查询指令使用JSON形式的标记，可轻易查询文档中内嵌的对象及数组。
MongoDb 使用update()命令可以实现替换完成的文档（数据）或者一些指定的数据字段 。
Mongodb中的Map/reduce主要是用来对数据进行批量处理和聚合操作。
Map和Reduce。Map函数调用emit(key,value)遍历集合中所有的记录，将key与value传给Reduce函数进行处理。
Map函数和Reduce函数是使用Javascript编写的，并可以通过db.runCommand或mapreduce命令来执行MapReduce操作。
GridFS是MongoDB中的一个内置功能，可以用于存放大量小文件。
MongoDB允许在服务端执行脚本，可以用Javascript编写某个函数，直接在服务端执行，也可以把函数的定义存储在服务端，下次直接调用即可。
MongoDB支持各种编程语言:RUBY，PYTHON，JAVA，C++，PHP，C#等多种语言。
MongoDB安装简单。

## 概念介绍

### 数据库

- 一个mongodb中可以建立多个数据库。
- MongoDB的默认数据库为"db"，该数据库存储在data目录中。
- MongoDB的单个实例可以容纳多个独立的数据库，每一个都有自己的集合和权限，不同的数据库也放置在不同的文件中。
- "show dbs" 命令可以显示所有数据库的列表。

### 文档

文档是一个键值(key-value)对(即BSON)。MongoDB 的文档不需要设置相同的字段，并且相同的字段不需要相同的数据类型，这与关系型数据库有很大的区别，也是 MongoDB 非常突出的特点。

#### 注意：

- 文档中的键/值对是有序的。


- 文档中的值不仅可以是在双引号里面的字符串，还可以是其他几种数据类型（甚至可以是整个嵌入的文档)。


- MongoDB区分类型和大小写。


- MongoDB的文档不能有重复的键。


- 文档的键是字符串。除了少数例外情况，键可以使用任意UTF-8字符。

#### 文档键命名规范：

- 键不能含有\0 (空字符)。这个字符用来表示键的结尾。


- .和$有特别的意义，只有在特定环境下才能使用。


- 以下划线"_"开头的键是保留的(不是严格要求的)。



### 集合

集合就是 MongoDB 文档组，类似于 RDBMS （关系数据库管理系统：Relational Database Management System)中的表格。
集合存在于数据库中，集合没有固定的结构，这意味着你在对集合可以插入不同格式和类型的数据，但通常情况下我们插入集合的数据都会有一定的关联性。

#### 合法的集合名

- 集合名不能是空字符串""。
- 集合名不能含有\0字符（空字符)，这个字符表示集合名的结尾。
- 集合名不能以"system."开头，这是为系统集合保留的前缀。
- 用户创建的集合名字不能含有保留字符。有些驱动程序的确支持在集合名里面包含，这是因为某些系统生成的集合中包含该字符。除非你要访问这种系统创建的集合，否则千万不要在名字里出现$。

#### capped collections

Capped collections 就是固定大小的collection。
它有很高的性能以及队列过期的特性(过期按照插入的顺序). 有点和 "RRD" 概念类似。
Capped collections是高性能自动的维护对象的插入顺序。它非常适合类似记录日志的功能 和标准的collection不同，你必须要显式的创建一个capped collection， 指定一个collection的大小，单位是字节。collection的数据存储空间值提前分配的。
要注意的是指定的存储大小包含了数据库的头信息。

在capped collection中，你能添加新的对象。
能进行更新，然而，对象不会增加存储空间。如果增加，更新就会失败 。
数据库不允许进行删除。使用drop()方法删除collection所有的行。
注意: 删除之后，你必须显式的重新创建这个collection。
在32bit机器中，capped collection最大存储为1e9( 1X10^9)个字节。

## 操作

### 插入文档：

文档的数据结构和JSON基本一样。
所有存储在集合中的数据都是BSON格式。
BSON是一种类json的一种二进制形式的存储格式,简称Binary JSON。

MongoDB 使用 insert() 或 save() 方法向集合中插入文档。

```markdown
插入文档:            
db.COLLECTION_NAME.insert(document)

开启mongodb服务： 
$MONGO_HOME/bin/mongod

查询当前所在数据库：
db

查询所有的数据库(不会显示没有数据的数据库)：
show dbs

使用数据库（如果没有数据库会自动创建）：
use database_name 

插入数据：
db.database_name.insert({"":""},{},{}....)

删除数据库：
use database_name
db.dropDatabase()

删除集合：
db.collection.drop()

显示集合：
show tables

插入文档(如果该集合不在该数据库中， MongoDB 会自动创建该集合并插入文档)：
db.COLLECTION_NAME.insert(document)

查看集合中的数据：
db.col.find()

使用变量：
document=({"":""}.....)
db.table1.insert(document)

更新文档：
update() 和 save() 方法
db.col.update({原来的数据},{$set:{更新后的数据}})
以上语句只会修改第一条发现的文档，如果你要修改多条相同的文档，则需要设置 multi 参数为 true。
db.col.update({原来的数据},{$set:{更新后的数据}},{multi:true})
```

