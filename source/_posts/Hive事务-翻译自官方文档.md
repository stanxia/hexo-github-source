---
title: Hive事务-翻译自官方文档
date: 2017-12-13 16:34:17
tags: hive
categories: hive
---
{% note info %}
## ACID简介
 {% endnote %}
ACID代表数据库事务的四个特性:

* 原子性（Atomicity）:一个事务是一个不可再分割的工作单位，事务中的所有操作要么都发生，要么都不发生。
* 一致性（Consistency）:事务开始之前和事务结束以后，数据库的完整性约束没有被破坏。这是说数据库事务不能破坏关系数据的完整性以及业务逻辑上的一致性。
* 隔离性（Isolation）:多个事务并发访问，事务之间是隔离的，一个事务不影响其它事务运行效果。这指的是在并发环境中，当不同的事务同时操作相同的数据时，每个事务都有各自完整的数据空间。事务查看数据更新时，数据所处的状态要么是另一事务修改它之前的状态，要么是另一事务修改后的状态，事务不会查看到中间状态的数据。事务之间的相应影响，分别为：脏读、不可重复读、幻读、丢失更新。
* 持久性（Durability）:意味着在事务完成以后，该事务锁对数据库所作的更改便持久的保存在数据库之中，并不会被回滚。

在Hive0.13中添加事务之后，在分区级别上提供了原子性、一致性和持久性。可以通过打开可用的锁定机制(ZooKeeper或内存)来提供隔离。现在可以在行级别上提供完整的ACID语义，这样一个应用程序可以添加行，而另一个应用程序可以在相同的分区上读取，而不会相互干扰。
<!-- more -->
使用ACID语义的事务被添加到Hive中，以解决以下用例:

* 流数据的接入。许多用户都使用 Apache Flume, Apache Storm, or Apache Kafka 将流式数据导入Hadoop集群。 这些工具都是每秒百万行级的数据写入，而Hive只能每十五分钟到一个小时添加一次分区。快速的增加分区会对表中的分区数量形成压力。当然可以事先创建好分区再将数据导入，但这样会引起脏读，而且目录下生成的小文件会对namenode造成很大的压力。而新特性可以很好的解决上述问题
* 维度表的缓慢变化：在典型的星型架构数据仓库中，维度表随时间慢慢变化。例如，零售商将会打开新的商店，这些商店需要被添加到商店的表中，或者现有的商店可能会改变它的平方英尺或其他一些被跟踪的特性。这些更改会导致插入单个记录或更新记录(取决于所选择的策略)。Hive从0.14开始支持。
* 数据更新。从Hive 0.14开始，可以 INSERT, UPDATE, 和 DELETE。
* 使用SQL MERGE语句进行批量更新。

{%note info%}
## 事务限制
{%endnote%}

* BEGIN, COMMIT, and ROLLBACK还不支持。所有操作都是自动提交的。该计划将在未来的版本中提供支持。
* 现目前只支持ORC文件格式。已经构建了这样的功能，可以通过任何存储格式来使用事务，以确定更新或删除如何应用于基本记录(基本上，具有显式或隐式行id)，但到目前为止，集成工作还只是用于ORC。
* 事务默认为关闭的。
* 表必须要分桶才能支持事务。因为外部表不受分桶的控制，因而外部表并不支持事务。
* 从一个 non_ACID 表读写 ACID 表 是不允许的。事务管理器必须设置为：org.apache.hadoop.hive.ql.lockmgr.DbTxnManager 。
* 目前只支持快照级别的隔离。
* 现有的ZooKeeper和内存中的锁管理器与事务不兼容。
* ACID表不支持使用ALTER TABLE的模式更改。
* 使用 ORACLE 作为元数据库，"datanucleus.connectionPoolingType=BONECP" 可能会出现："No such lock.." 和 "No such transaction..."等错误。请设置为："datanucleus.connectionPoolingType=DBCP" 。
* 事务表不支持 ：LOAD DATA...  语法。

{%note info%}
## 流 APIs
{%endnote%}

Hive 提供数据 流式插入和流式更新的 APIs:

* Hive HCatalog Streaming API
* HCatalog Streaming Mutation API (Hive 2.0.0 及以上可用)

{%note info%}
## 语法修改
{%endnote%}

从Hive 0.14. 开始，INSERT...VALUES, UPDATE, and DELETE 已经添加进 SQL 语法。几个新命令已经添加进 Hive 的 DDL 支持 ACID 和 事务, 并修改一些已经存在的 DDL 。  

新命令：

* SHOW TRANSACTIONS 
* SHOW COMPACTIONS 
* ABORT TRANSACTIONS 

修改的命令：

* SHOW LOCKS  :提供与事务相关的新锁的信息。如果正在使用 ZooKeeper 或者内存锁管理，命令的输出将有不同。
* ALTER TABLE :添加新功能，请求表或分区的压缩。 一般来说，用户不需要请求压缩，因为系统将检测到对它们的需求并启动压缩。 然而，如果对一个表关闭了压缩，或者用户希望在系统不选择的时候压缩表, ALTER TABLE 可用于启动压缩。为了观察压缩的进度，用户可以使用 SHOW COMPACTIONS 。

{%note info%}
## 基础设计
{%endnote%}

HDFS 不支持对文件进行就地修改。也不提供读的一致性，并且当有数据追加到文件，HDFS 不对读数据的用户提供一致性的。为了在 HDFS 上提供这些特性，我们遵循了其他数据仓库工具中使用的标准方法。 表或分区的数据存储在一组 base 文件中。 新的记录、更新和删除存储在 delta 文件中。每个改变表或分区的事务产生一组新的 delta 文件 。读的时候，合并 base 和 delta 文件, 应用所有的更新和删除。
### Base 和 Delta 目录
一个分区或一个非分区表的所有文件都在同一个目录下。有更新的时候，任何 ACID 的分区（或非分区表）都有一个 base 文件目录，和一个 delta 文件目录。 

```shell
hive> dfs -ls -R /user/hive/warehouse/t;
drwxr-xr-x   - ekoifman staff          0 2016-06-09 17:03 /user/hive/warehouse/t/base_0000022
-rw-r--r--   1 ekoifman staff        602 2016-06-09 17:03 /user/hive/warehouse/t/base_0000022/bucket_00000
drwxr-xr-x   - ekoifman staff          0 2016-06-09 17:06 /user/hive/warehouse/t/delta_0000023_0000023_0000
-rw-r--r--   1 ekoifman staff        611 2016-06-09 17:06 /user/hive/warehouse/t/delta_0000023_0000023_0000/bucket_00000
drwxr-xr-x   - ekoifman staff          0 2016-06-09 17:07 /user/hive/warehouse/t/delta_0000024_0000024_0000
-rw-r--r--   1 ekoifman staff        610 2016-06-09 17:07 /user/hive/warehouse/t/delta_0000024_0000024_0000/bucket_00000
```

### 合并
合并是一组运行在 Metastore中支持 ACID 系统的后台程序。包含 Initiator, Worker, Cleaner, AcidHouseKeeperServiceis 和一些其他的。

#### Delta 文件合并
随着修改的操作，越来越多的delta文件被创建，需要合并以保持足够的性能。有两种类型的合并，minor和major。

* Minor 合并 ：在每一个 bucket ，将一组已经存在的 delta 文件合并重写到一个单独的 delta 文件中。
* Major 合并 ：在每一个 bucket ，将一个或多个 delta 文件和 bucket 的 base 文件合并重写到一个新的 base 文件中.  Major 合并效率高但花销大。

所有的合并都在后台完成，并且不妨碍对数据当前的读和写。 合并完所有的旧文件之后，就删除掉这些旧文件。

#### Initiator 初始化器
这个模块负责发现哪些表或分区应该被合并。  通过设置：hive.compactor.initiator.on 开启。每个合并任务处理一个分区 (或一个非分区表).  如果连续合并失败次数超过 hive.compactor.initiator.failed.compacts.threshold 设定的值，该分区的自动合并调度将会被停止。

#### Worker 工作执行器
每个 worker 处理一个单独的合并任务.  合并是 MapReduce job，名字是: <hostname>-compactor-<db>.<table>.<partition>. 每一个worker 提交 job 到集群上 (通过定义的 hive.compactor.job.queue) 并等着任务完成。hive.compactor.worker.threads 决定了每个 Metastore 中 Workers 的数量。Hive 仓库中的 Workers 的总数决定了当前合并任务的最大值。

#### Cleaner 清除器
该程序的职责：删除合并后不再需要的 delta 文件.

#### AcidHouseKeeperService ACID管家服务
寻找在 hive.txn.timeout 时间内没有心跳的事务，并将其终止。该系统假定发起事务的客户机停止了心跳，而锁定的资源应该被释放。

#### SHOW COMPACTIONS 查看合并
这个命令显示目前正在运行的合并信息，和最近的合并历史 (配置的时间范围内)。 从 HIV-12353 开始支持历史显示。

### 事务／锁 管理器
添加了一个名为“事务管理器”的新逻辑实体，它包含了以前的“数据库/表/分区锁管理器”概念。(hive.lock.manager with default of org.apache.hadoop.hive.ql.lockmgr.zookeeper.ZooKeeperHiveLockManager). 事务管理器现在还负责管理事务锁。默认的 DummyTxnManager 模拟了老版本 Hive 的行为: 没有事务和使用 hive.lock.manager 属性用来为表，分区，数据库创建锁管理器。在 Hive metastore 中新添加的 DbTxnManager 管理所有的锁/事务和 DbLockManager (在服务器出现故障时，事务和锁是持久的). 这意味着在启用事务时，以前锁定ZooKeeper的行为不再存在。 从 lock 持有者和事务 initiators 到 metastore 的心跳连接，如果心跳超时，则这个锁或事务将被舍弃。

从hive 1.30起，DbLockManger 持续获取锁的周期可以通过 hive.lock.numretires 和 hive.lock.sleep.between.retries. 两个属性设置。如果DbLockManger 获取锁失败，过一段时间之后会进行重试。为了支持短查询同时不对metastore造成负担，DbLockManger 在每次重试后加倍等待时长。

注意：DbTxnManager 可以获取所有的表的锁，即便那些没有设置transactional=true属性的表。默认对一个非事务表的插入操会获取一个排他锁从而阻止其他的插入和读取。技术上正确的，但是这背离了hive之前的工作方式。为了向后兼容，可以设置hive.txn.strict.locking.mode 属性来使锁管理器在对非事务表的插入操作时，获取共享锁。保留之前的语义，还有一个好处就是能防止表在读取是被删除。而对于事务表，插入总是获取的共享锁。是因为这些表实现了MVCC的架构，在存储的底层实现了很强的读一致性（快照隔离的方式），甚至能应对并发的修改操作。

* 共享锁【S锁】:又称读锁，若事务T对数据对象A加上S锁，则事务T可以读A但不能修改A，其他事务只能再对A加S锁，而不能加X锁，直到T释放A上的S锁。这保证了其他事务可以读A，但在T释放A上的S锁之前不能对A做任何修改。
* 排他锁【X锁】:又称写锁。若事务T对数据对象A加上X锁，事务T可以读A也可以修改A，其他事务不能再对A加任何锁，直到T释放A上的锁。这保证了其他事务在T释放A上的锁之前不能再读取和修改A。）

{%note info%}
## 配置信息
{%endnote%}

为了开启hive的事务支持，以下是需要开启的最少的hive配置：

客户端：

```xml
hive.support.concurrency – true
hive.enforce.bucketing – true (Not required as of Hive 2.0)
hive.exec.dynamic.partition.mode – nonstrict
hive.txn.manager – org.apache.hadoop.hive.ql.lockmgr.DbTxnManager

```
服务端： (Metastore)

```xml
hive.compactor.initiator.on – true (See table below for more details)
hive.compactor.worker.threads – a positive number on at least one instance of the Thrift metastore service
```

详细配置信息请看：[hive事务配置信息](https://cwiki.apache.org/confluence/display/Hive/Hive+Transactions)

{%note info%}
## 表属性
{%endnote%}

如果表要使用 ACID ，则必须指定： ` "transactional=true" ` 。如果该表被设置为 ACID 表，则不能转换为非 ACID 表，设置 ` "transactional"="false" ` 也是不行的。在 hive-site.xml 中必须设置：
` hive.txn.manager = org.apache.hadoop.hive.ql.lockmgr.DbTxnManager `

如果需希望系统自动进行合并，则可以设置： ` "NO_AUTO_COMPACTION" ` ,也可以通过` Alter Table/Partition Compact ` 语法来设置。

```[sql] [Example: Set compaction options in TBLPROPERTIES at table level]
CREATE TABLE table_name (
  id                int,
  name              string
)
CLUSTERED BY (id) INTO 2 BUCKETS STORED AS ORC
TBLPROPERTIES ("transactional"="true",
  "compactor.mapreduce.map.memory.mb"="2048",     -- specify compaction map job properties
  "compactorthreshold.hive.compactor.delta.num.threshold"="4",  -- trigger minor compaction if there are more than 4 delta directories
  "compactorthreshold.hive.compactor.delta.pct.threshold"="0.5" -- trigger major compaction if the ratio of size of delta files to
                                                                   -- size of base files is greater than 50%
);
```

```[sql] [Example: Set compaction options in TBLPROPERTIES at request level]
ALTER TABLE table_name COMPACT 'minor' 
   WITH OVERWRITE TBLPROPERTIES ("compactor.mapreduce.map.memory.mb"="3072");  -- specify compaction map job properties
ALTER TABLE table_name COMPACT 'major'
   WITH OVERWRITE TBLPROPERTIES ("tblprops.orc.compress.size"="8192");         -- change any other Hive table properties
```




