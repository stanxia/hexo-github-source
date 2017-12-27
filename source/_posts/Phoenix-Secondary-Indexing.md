---
title: Phoenix Secondary Indexing
date: 2017-12-27 16:48:35
tags: phoenix
categories: phoenix
---

二级索引是从主访问路径访问数据的正交方式。在 HBase 中，您有一个索引按字典顺序排序在主键行上。以不同于主要行的方式访问记录需要扫描表中的所有行，以针对您的过滤器对其进行测试。通过二级索引，您索引的列或表达式形成一个备用行键，以允许沿着这个新轴进行点查找和范围扫描。

{% note info %}
## Covered Indexes 覆盖索引
{% endnote %}

Phoenix 是特别强大的，因为我们提供覆盖索引 - 一旦找到索引条目，我们不需要返回主表。相反，我们将我们关心的数据绑定到索引行，节省了读取时间的开销。

例如，以下内容将在v1和v2列上创建索引，并在索引中包含v3列以防止从数据表中获取该列：

```sql
CREATE INDEX my_index ON my_table (v1,v2) INCLUDE(v3)
```
<!--more-->
{% note info %}
## Functional Indexes 函数索引
{% endnote %}

函数索引（在4.3和更高版本中可用）允许您不仅在列上而且在任意表达式上创建索引。然后，当一个查询使用该表达式时，索引可以用来检索结果而不是数据表。例如，您可以在 UPPER（FIRST_NAME ||''|| LAST_NAME）上创建一个索引，以允许您对组合的名字和姓氏进行不区分大小写的搜索。

例如，下面将创建这个功能索引：

```sql
CREATE INDEX UPPER_NAME_IDX ON EMP (UPPER(FIRST_NAME||' '||LAST_NAME))
```

有了这个索引，发出下面的查询时，将使用索引而不是数据表来检索结果：
```sql
SELECT EMP_ID FROM EMP WHERE UPPER(FIRST_NAME||' '||LAST_NAME)='JOHN DOE'
```

Phoenix 支持两种索引技术：全局索引和本地索引。每个在不同的情况下都很有用，并且有自己的故障概况和性能特点。

{% note info %}
## Global Indexes 全局索引
{% endnote %}

全局索引目标是重度读操作用例。使用全局索引，索引的所有性能损失都是在写入时发生的。我们拦截数据表更新写（DELETE，UPSERT VALUES和UPSERT SELECT），建立索引更新，然后发送任何必要的更新到所有感兴趣的索引表。在读的时候，Phoenix 会选择使用的索引表，这将产生最快的查询时间，并直接扫描它，就像任何其他的 HBase 表一样。默认情况下，除非暗示，否则索引不会用于引用不属于索引的列的查询。

{% note info %}
## Local Indexes 本地索引
{% endnote %}

本地索引目标针对重度写操作，空间受限的用例。就像全局索引一样，Phoenix 会在查询时自动选择是否使用本地索引。使用本地索引，索引数据和表数据共同驻留在同一台服务器上，防止写入期间的任何网络开销。即使查询没有完全覆盖，也可以使用本地索引（即，Phoenix 自动检索不在索引中的列，通过与数据表相对应的索引）。与全局索引不同，表中的所有本地索引都存储在4.8.0版之前的单独的共享表中。从4.8.0开始，我们将所有本地索引数据存储在相同数据表中的单独的影子列族中。在读取本地索引时，由于不能确定索引数据的确切区域位置，所以必须检查每个区域的数据。因此在读取时会发生一些开销。

{% note info %}
## Index Population
{% endnote %}

默认情况下，创建索引时，会在 CREATE INDEX 调用期间同步填充该索引。根据数据表的当前大小，这可能是不可行的。从4.5开始，可以通过在索引创建 DDL 语句中包含 ASYNC 关键字来异步完成索引的填充：
```sql
CREATE INDEX async_index ON my_schema.my_table (v) ASYNC
```

必须通过 HBase 命令行单独启动填充索引表的 map-reduce 作业，如下所示：
```shell
${HBASE_HOME}/bin/hbase org.apache.phoenix.mapreduce.index.IndexTool
  --schema MY_SCHEMA --data-table MY_TABLE --index-table ASYNC_IDX
  --output-path ASYNC_IDX_HFILES
```

只有 map-reduce 作业完成后，索引才会被激活并开始在查询中使用。这项工作对于退出的客户端是有弹性的。输出路径选项用于指定用于写入 HFile 的 HDFS 目录。

{% note info %}
## Index Usage 索引使用
{% endnote %}

Phoenix 自动使用索引来服务一个查询，当它确定更有效的时候。但是，除非查询中引用的所有列都包含在索引中，否则不会使用全局索引。例如，以下查询不会使用索引，因为在查询中引用了v2，但未包含在索引中：
```sql
SELECT v2 FROM my_table WHERE v1 = 'foo'
```

在这种情况下，有三种获取索引的方法：

1.通过在索引中包含v2来创建一个覆盖索引：
```sql
CREATE INDEX my_index ON my_table (v1) INCLUDE (v2)
```
这将导致v2列值被复制到索引中，并随着更改而保持同步。这显然会增加索引的大小。

2.提示查询强制它使用索引：
```sql
SELECT /*+ INDEX(my_table my_index) */ v2 FROM my_table WHERE v1 = 'foo'
```
这将导致在遍历索引时找到每个数据行以找到缺少的v2列值。这个提示只有在你知道索引有很好的选择性的时候才可以使用（例如，在这个例子中有少量的表格行的值是'foo'），否则你可以通过默认的行为来获得更好的性能全表扫描。

3.创建一个本地索引：
```sql
CREATE LOCAL INDEX my_index ON my_table (v1)
```
与全局索引不同，即使查询中引用的所有列都不包含在索引中，本地索引也将使用索引。这是默认为本地索引完成的，因为我们知道在同一个区域服务器上的表和索引数据coreside确保查找是本地的。

{% note info %}
## Index Removal 索引移除
{% endnote %}

要删除索引，使用以下语句：
```sql
DROP INDEX my_index ON my_table
```
如果索引列在数据表中被删除，索引将自动被删除。另外，如果在数据表中删除一个被覆盖的列，它也会自动从索引中删除。

{% note info %}
## Index Properties 索引属性
{% endnote %}

就像使用CREATE TABLE语句一样，CREATE INDEX语句可以通过属性应用到底层的HBase表，包括对其进行限制的能力:
```sql
CREATE INDEX my_index ON my_table (v2 DESC, v1) INCLUDE (v3)
    SALT_BUCKETS=10, DATA_BLOCK_ENCODING='NONE'
```
请注意，如果主表是加盐的，则对于全局索引，该索引将以相同的方式自动被加盐。另外，相对于主索引表与索引表的大小，索引的MAX_FILESIZE向下调整。欲了解更多信息，请参阅[这里](http://phoenix.apache.org/salted.html)。另一方面，使用本地索引时，不允许指定SALT_BUCKETS。

{% note info %}
## Consistency Guarantees 一致性保证
{% endnote %}

在提交后成功返回到客户端，所有数据保证写入所有相关的索引和主表。换句话说，索引更新与HBase提供的相同强一致性保证是同步的。

但是，由于索引存储在与数据表不同的表中，因此根据表的属性和索引的类型，表和索引之间的一致性会因服务器端崩溃而失败。这是您的需求和使用案例驱动的重要设计考虑因素。

下面概述了各种一致性保证的不同选项。

### Transactional Tables 事务表
通过将您的表声明为事务性的，您可以实现表和索引之间最高级别的一致性保证。在这种情况下，您的表突变和相关索引更新的提交是具有强 ACID 保证的原子。如果提交失败，那么您的数据（表或索引）都不会更新，从而确保您的表和索引始终保持同步。

为什么不总是把你的表声明为事务性的？这可能很好，特别是如果你的表被声明为不可变的，因为在这种情况下事务开销非常小。但是，如果您的数据是可变的，请确保与事务性表发生冲突检测相关的开销和运行事务管理器的运行开销是可以接受的。此外，具有二级索引的事务表可能会降低写入数据表的可用性，因为数据表及其辅助索引表必须可用，否则写入将失败。

### Immutable Tables 不可变表
对于其中数据只写入一次而从不更新的表，可以进行某些优化以减少增量维护的写入时间开销。这是常见的时间序列数据，如日志或事件数据，一旦写入行，它将永远不会被更新。要利用这些优化，通过将 IMMUTABLE_ROWS = true 属性添加到您的 DDL 语句中，将您的表声明为不可变：
```sql
CREATE TABLE my_table (k VARCHAR PRIMARY KEY, v VARCHAR) IMMUTABLE_ROWS=true
```
使用 IMMUTABLE_ROWS = true 声明的表上的所有索引都被认为是不可变的（请注意，默认情况下表被认为是可变的）。对于全局不可变索引，索引完全在客户端维护，索引表是在数据表发生更改时生成的。另一方面，本地不可变索引在服务器端保持不变。请注意，没有任何保护措施可以强制执行，声明为不可变的表实际上不会改变数据（因为这会否定所达到的性能增益）。如果发生这种情况，索引将不再与表同步。

如果您有一个现有的表，您想从不可变索引切换到可变索引，请使用ALTER TABLE命令，如下所示：
```sql
ALTER TABLE my_table SET IMMUTABLE_ROWS=false
```
非事务性，不可变表的索引没有自动处理提交失败的机制。保持表和索引之间的一致性留给客户端处理。因为更新是幂等的，所以最简单的解决方案是客户端继续重试一批突变，直到它们成功。

### Mutable Tables 可变表
对于非事务性可变表，我们通过将索引更新添加到主表行的预写日志（WAL）条目来维护索引更新持久性。只有在WAL条目成功同步到磁盘后，我们才会尝试更新索引/主表。我们默认并行编写索引更新，从而导致非常高的吞吐量。如果服务器在我们写索引更新的时候崩溃了，我们会重播所有索引更新到WAL恢复过程中的索引表，并依赖更新的幂等性来确保正确性。因此，非事务性可变表上的索引只是主表背后的一批编辑。

重要的是要注意几点：

* 对于非事务性表，您可以看到索引表与主表不同步。
* 如上所述，这是可以的，因为我们在很短的时间内只有很小的一部分，并且不同步
* 每个数据行及其索引行都保证被写入或丢失 - 我们从来没有看到部分更新，因为这是HBase原子性保证的一部分。
* 首先将数据写入表中，然后写入索引表（如果禁用WAL，则反之亦然）。

#### Singular Write Path
有一个保证失败属性的写入路径。所有写入HRegion的内容都被我们的协处理器拦截。然后，我们根据挂起更新（或更新，如果是批处理）构建索引更新。然后这些更新被附加到原始更新的WAL条目。

如果在此之前我们遇到任何问题，我们会将失败返回给客户端，并且没有任何数据被持久化或者不可见。

一旦WAL被写入，我们确保即使在失败的情况下，索引和主表数据也将变得可见。

* 如果服务器发生崩溃，我们会使用通常的WAL重播机制重播索引更新。
* 如果服务器没有崩溃，我们只是将索引更新插入到它们各自的表中。
    * 如果索引更新失败，下面概述了保持一致性的各种方法。
    * 如果Phoenix系统目录表发生故障时无法到达，我们强制服务器立即中止并失败，请在JVM上调用System.exit，强制服务器死机。通过杀死服务器，我们确保WAL将在恢复时重播，将索引更新重播到相应的表中。这确保了二级索引在知道无效状态时不会继续使用。

#### 禁止表写入，直到可变的索引是一致的
在非事务性表和索引之间保持一致性的最高级别是声明在更新索引失败的情况下应暂时禁止写入数据表。在此一致性模式下，表和索引将保留在发生故障之前的时间戳，写入数据表将被禁止，直到索引重新联机并与数据表同步。该索引将保持活动状态，并像往常一样继续使用查询。

以下服务器端配置控制此行为：

* phoenix.index.failure.block.write必须为true，以便在发生提交失败时写入数据表以失败，直到可以使用数据表追上索引。
* phoenix.index.failure.handling.rebuild必须为true（默认值），以便在发生提交失败的情况下在后台重建可变索引。

#### 写入失败时禁用可变索引，直到一致性恢复
如果在提交时写入失败，具有可变索引的默认行为是将索引标记为禁用,在后台部分重建它们，然后在恢复一致性时再次将其标记为活动状态。在这种一致性模式下，在重建二级索引时，写入数据表不会被阻塞。但是，在重建过程中，二级索引不会被查询使用。

以下服务器端配置控制此行为：

* phoenix.index.failure.handling.rebuild必须为true（缺省值），以便在发生提交失败的情况下在后台重建可变索引。
* phoenix.index.failure.handling.rebuild.interval控制服务器检查是否需要部分重建可变索引以赶上数据表更新的毫秒频率。默认值是10000或10秒。
* phoenix.index.failure.handling.rebuild.overlap.time控制执行部分重建时从发生故障的时间戳开始返回的毫秒数。默认值是1。

#### 写入失败时禁用可变索引，需要手动重建
这是可变二级索引的最低一致性水平。在这种情况下，当写入二级索引失败时，索引将被标记为禁用，并且手动重建所需的索引以使其再次被查询使用。

以下服务器端配置控制此行为：

* 如果提交失败，phoenix.index.failure.handling.rebuild必须设置为false，以禁止在后台重建可变索引。

{% note info %}
## Setup 设置
{% endnote %}

非事务性，可变索引需要在区域服务器和主服务器上运行特殊的配置选项 - Phoenix确保在表上启用可变索引时，它们已正确设置;如果未设置正确的属性，则将无法使用辅助索引。将这些设置添加到您的hbase-site.xml后，您需要重启集群。

您将需要将以下参数添加到每个区域服务器上的hbase-site.xml：
```xml
<property>
  <name>hbase.regionserver.wal.codec</name>
  <value>org.apache.hadoop.hbase.regionserver.wal.IndexedWALEditCodec</value>
</property>
```

上面的属性使定制的WAL编辑能够被写入，确保索引更新的正确写入/重播。这个编解码器支持通常的主机WALEdit选项，最显着的是WALEdit压缩。
```xml
<property>
  <name>hbase.region.server.rpc.scheduler.factory.class</name>
  <value>org.apache.hadoop.hbase.ipc.PhoenixRpcSchedulerFactory</value>
  <description>Factory to create the Phoenix RPC Scheduler that uses separate queues for index and metadata updates</description>
</property>
<property>
  <name>hbase.rpc.controllerfactory.class</name>
  <value>org.apache.hadoop.hbase.ipc.controller.ServerRpcControllerFactory</value>
  <description>Factory to create the Phoenix RPC Scheduler that uses separate queues for index and metadata updates</description>
</property>
```

通过确保索引更新的优先级高于数据更新，上述属性可防止在全局索引（HBase 0.98.4+和Phoenix 4.3.1+）的索引维护过程中发生死锁。它还通过确保元数据rpc调用比数据rpc调用具有更高的优先级来防止死锁。

从Phoenix 4.8.0开始，不需要更改配置就可以使用本地索引。在Phoenix 4.7及更低版本中，主服务器节点和区域服务器节点上的服务器端hbase-site.xml需要进行以下配置更改：
```xml
<property>
  <name>hbase.master.loadbalancer.class</name>
  <value>org.apache.phoenix.hbase.index.balancer.IndexLoadBalancer</value>
</property>
<property>
  <name>hbase.coprocessor.master.classes</name>
  <value>org.apache.phoenix.hbase.index.master.IndexMasterObserver</value>
</property>
<property>
  <name>hbase.coprocessor.regionserver.classes</name>
  <value>org.apache.hadoop.hbase.regionserver.LocalIndexMerger</value>
</property>
```

### 升级4.8.0之前创建的本地索引
在服务器上将Phoenix升级到4.8.0以上版本时，如果存在，请从hbase-site.xml中除去以上三个与本地索引相关的配置。从客户端，我们支持在线（在初始化来自4.8.0+版本的phoenix客户端的连接时）和离线（使用psql工具）在4.8.0之前创建的本地索引的升级。作为升级的一部分，我们以ASYNC模式重新创建本地索引。升级后用户需要使用IndexTool建立索引。

在升级之后使用客户端配置。

1. phoenix.client.localIndexUpgrade
    * 值为 true 则表示在线升级, false 表示离线升级。
    * 默认为 true 。

命令使用psql工具$ psql [zookeeper] -l运行离线升级。

{% note info %}
## Tuning
{% endnote %}

索引是相当快的。但是，要优化您的特定环境和工作负载，可以调整一些属性。

以下所有参数必须在hbase-site.xml中设置 - 对于整个集群和所有索引表，以及在同一台服务器上的所有区域上都是如此（例如，一台服务器也不会一次写入许多不同的索引表）。

1. index.builder.threads.max
    * 用于从主表更新构建索引更新的线程数
    * 增加此值克服了从底层HRegion读取当前行状态的瓶颈。调整这个值太高，只会增加HRegion瓶颈，因为它将无法处理太多的并发扫描请求，以及一般的线程交换问题。
    * Default: 10
2. index.builder.threads.keepalivetime
    * 在构建器线程池中使线程过期后的时间（以秒为单位）。
    * 未使用的线程会在这段时间后立即释放，而不会保留核心线程
    * Default: 60
3. index.writer.threads.max
    * Number of threads to use when writing to the target index tables.
    * The first level of parallelization, on a per-table basis - it should roughly correspond to the number of index tables
    * Default: 10
4. index.writer.threads.keepalivetime
    * Amount of time in seconds after we expire threads in the writer thread pool.
    * Unused threads are immediately released after this amount of time and not core threads are retained (though this last is a small concern as tables are expected to sustain a fairly constant write load), but simultaneously allows us to drop threads if we are not seeing the expected load.
    * Default: 60
5. hbase.htable.threads.max
    * Number of threads each index HTable can use for writes.
    * Increasing this allows more concurrent index updates (for instance across batches), leading to high overall throughput.
    * Default: 2,147,483,647
6. hbase.htable.threads.keepalivetime
    * Amount of time in seconds after we expire threads in the HTable’s thread pool.
    * Using the “direct handoff” approach, new threads will only be created if it is necessary and will grow unbounded. This could be bad but HTables only create as many Runnables as there are region servers; therefore, it also scales when new region servers are added.
    * Default: 60
7. index.tablefactory.cache.size
    * Number of index HTables we should keep in cache.
    * Increasing this number ensures that we do not need to recreate an HTable for each attempt to write to an index table. Conversely, you could see memory pressure if this value is set too high.
    * Default: 10
8. org.apache.phoenix.regionserver.index.priority.min
    * Value to specify to bottom (inclusive) of the range in which index priority may lie.
    * Default: 1000
9. org.apache.phoenix.regionserver.index.priority.max
    * Value to specify to top (exclusive) of the range in which index priority may lie.
    * Higher priorites within the index min/max range do not means updates are processed sooner.
    * Default: 1050
10. org.apache.phoenix.regionserver.index.handler.count
    * Number of threads to use when serving index write requests for global index maintenance.
    * Though the actual number of threads is dictated by the Max(number of call queues, handler count), where the number of call queues is determined by standard HBase configuration. To further tune the queues, you can adjust the standard rpc queue length parameters (currently, there are no special knobs for the index queues), specifically ipc.server.max.callqueue.length and ipc.server.callqueue.handler.factor. See the HBase Reference Guide for more details.
    * Default: 30

{% note info %}
## Performance
{% endnote %}   

We track secondary index performance via our performance framework. This is a generic test of performance based on defaults - your results will vary based on hardware specs as well as you individual configuration.

That said, we have seen secondary indexing (both immutable and mutable) go as quickly as < 2x the regular write path on a small, (3 node) desktop-based cluster. This is actually pretty reasonable as we have to write to multiple tables as well as build the index update.

{% note info %}
## Index Scrutiny Tool
{% endnote %}

With Phoenix 4.12, there is now a tool to run a MapReduce job to verify that an index table is valid against its data table. The only way to find orphaned rows in either table is to scan over all rows in the table and do a lookup in the other table for the corresponding row. For that reason, the tool can run with either the data or index table as the “source” table, and the other as the “target” table. The tool writes all invalid rows it finds either to file or to an output table PHOENIX_INDEX_SCRUTINY. An invalid row is a source row that either has no corresponding row in the target table, or has an incorrect value in the target table (i.e. covered column value).

The tool has job counters that track its status. VALID_ROW_COUNT, INVALID_ROW_COUNT, BAD_COVERED_COL_VAL_COUNT. Note that invalid rows - bad col val rows = number of orphaned rows. These counters are written to the table PHOENIX_INDEX_SCRUTINY_METADATA, along with other job metadata.

The Index Scrutiny Tool can be launched via the hbase command (in hbase/bin) as follows:

```
hbase org.apache.phoenix.mapreduce.index.IndexScrutinyTool -dt my_table -it my_index -o
```

It can also be run from Hadoop using either the phoenix-core or phoenix-server jar as follows:

```
HADOOP_CLASSPATH=$(hbase mapredcp) hadoop jar phoenix-<version>-server.jar org.apache.phoenix.mapreduce.index.IndexScrutinyTool -dt my_table -it my_index -o
```

By default two mapreduce jobs are launched, one with the data table as the source table and one with the index table as the source table.

The following parameters can be used with the Index Scrutiny Tool:

翻译自:[secondary_indexing](http://phoenix.apache.org/secondary_indexing.html)


  




