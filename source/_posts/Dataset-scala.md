---
title: Dataset.scala
date: 2017-11-23 09:34:44
tags: [spark,源码]
categories: spark
---
## 前言
Dataset 是一种强类型的领域特定对象集合，可以在使用功能或关系操作的同时进行转换。每个 Dataset 也有一个名为 “DataFrame” 的无类型视图，它是 [[Row]] 的 Dataset。
Dataset 上可用的操作分为转换和动作:
> 转换：产生新的 Dataset ；包括 map, filter, select, and aggregate (`groupBy`).
> 动作：触发计算并返回结果 ；包括 count, show, or 写数据到文件系统。

Dataset是懒加载的，例如：只有提交动作的时候才会触发计算。在内部，Datasets表示一个逻辑计划，它描述生成数据所需的计算。当提交动作时，Spark的查询优化器会优化逻辑计划，并以并行和分布式的方式生成有效执行的物理计划。请使用`explain` 功能，探索逻辑计划和优化的物理计划。

为了有效地支持特定于领域的对象，需要[[Encoder]]。编码器将特定类型的“T”映射到Spark的内部类型系统。例如：给一个 `Person` 类，并带有两个属性：`name` (string) and `age` (int),编码器告诉Spark在运行时生成代码，序列化 `Person` 对象为二进制结构。

通常有两种创建Dataset的方法:
> 使用 `SparkSession` 上可用的 `read` 方法读取 Spark 指向的存储系统上的文件。
> 用现存的 Datasets 转换而来。

Dataset操作也可以是无类型的，通过多种领域专用语言（DSL）方法定义：这些操作非常类似于 R或Python语言中的 数据框架抽象中可用的操作。
<!-- more -->

<!--请开始装逼-->
## basic-基础方法
### toDF
```scala
  /**
    * Converts this strongly typed collection of data to generic Dataframe. In contrast to the
    * strongly typed objects that Dataset operations work on, a Dataframe returns generic [[Row]]
    * objects that allow fields to be accessed by ordinal or name.
    * 将这种强类型的数据集合转换为一般的Dataframe。
    * 与Dataset操作所使用的强类型对象相反，
    * Dataframe返回泛型[[Row]]对象，这些对象允许通过序号或名称访问字段
    *
    * @group basic
    * @since 1.6.0
    */
  // This is declared with parentheses to prevent the Scala compiler from treating
  // `ds.toDF("1")` as invoking this toDF and then apply on the returned DataFrame.
  // 这是用括号声明的，以防止Scala编译器处理ds.toDF(“1”)调用这个toDF，然后在返回的DataFrame上应用。
  def toDF(): DataFrame = new Dataset[Row](sparkSession, queryExecution, RowEncoder(schema))
  
    /**
    * Converts this strongly typed collection of data to generic `DataFrame` with columns renamed.
    * This can be quite convenient in conversion from an RDD of tuples into a `DataFrame` with
    * meaningful names. For example:
    *
    * 将这种强类型的数据集合转换为通用的“DataFrame”，并将列重命名。
    * 在将tuple的RDD转换为富有含义的名称的“DataFrame”时，这是非常方便的，如：
    *
    * {{{
    *   val rdd: RDD[(Int, String)] = ...
    *   rdd.toDF()  // 隐式转换创建了 DataFrame ，列名为： `_1` and `_2`
    *   rdd.toDF("id", "name")  // 创建了 DataFrame ，列名为： "id" and "name"
    * }}}
    *
    * @group basic
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def toDF(colNames: String*): DataFrame = {
    require(schema.size == colNames.size,
      "The number of columns doesn't match.\n" +
        s"Old column names (${schema.size}): " + schema.fields.map(_.name).mkString(", ") + "\n" +
        s"New column names (${colNames.size}): " + colNames.mkString(", "))

    val newCols = logicalPlan.output.zip(colNames).map { case (oldAttribute, newName) =>
      Column(oldAttribute).as(newName)
    }
    select(newCols: _*)
  }
```
### as
```scala
/**
    * :: Experimental ::
    * Returns a new Dataset where each record has been mapped on to the specified type. The
    * method used to map columns depend on the type of `U`:
    * 
    * 返回一个新的Dataset，其中每个记录都被映射到指定的类型。用于映射列的方法取决于“U”的类型:
    * 
    *  - When `U` is a class, fields for the class will be mapped to columns of the same name
    * (case sensitivity is determined by `spark.sql.caseSensitive`).
    * 
    * 当“U”是类时：类的属性将映射到相同名称的列
    * 
    *  - When `U` is a tuple, the columns will be be mapped by ordinal (i.e. the first column will
    * be assigned to `_1`).
    * 
    * 当“U”是元组时：列将由序数映射 （例如，第一列将为 "_1"）
    * 
    *  - When `U` is a primitive type (i.e. String, Int, etc), then the first column of the
    * `DataFrame` will be used.
    * 
    * 当“U”是 基本类型（如 String，Int等）：然后将使用“DataFrame”的第一列。
    *
    * If the schema of the Dataset does not match the desired `U` type, you can use `select`
    * along with `alias` or `as` to rearrange or rename as required.
    * 
    * 如果数据集的模式与所需的“U”类型不匹配，您可以使用“select”和“alias”或“as”来重新排列或重命名。
    *
    * @group basic
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def as[U: Encoder]: Dataset[U] = Dataset[U](sparkSession, logicalPlan)
```
### schema
```scala
  /**
    * Returns the schema of this Dataset.
    * 返回该Dataset的模版
    *
    * @group basic
    * @since 1.6.0
    */
  def schema: StructType = queryExecution.analyzed.schema
```
### printSchema
```scala
  /**
    * Prints the schema to the console in a nice tree format.
    * 
    * 以一种漂亮的树格式将模式打印到控制台。
    *
    * @group basic
    * @since 1.6.0
    */
  // scalastyle:off println
  def printSchema(): Unit = println(schema.treeString)
```
### explain
```scala
  /**
    * Prints the plans (logical and physical) to the console for debugging purposes.
    * 
    * 将计划(逻辑和物理)打印到控制台以进行调试。
    * 参数：extended = false 为物理计划
    *
    * @group basic
    * @since 1.6.0
    */
  def explain(extended: Boolean): Unit = {
    val explain = ExplainCommand(queryExecution.logical, extended = extended)
    sparkSession.sessionState.executePlan(explain).executedPlan.executeCollect().foreach {
      // scalastyle:off println
      r => println(r.getString(0))
      // scalastyle:on println
    }
  }
  
    /**
    * Prints the physical plan to the console for debugging purposes.
    * 将物理计划打印到控制台以进行调试。
    *
    * @group basic
    * @since 1.6.0
    */
  def explain(): Unit = explain(extended = false)
```
### dtypes
```scala
  /**
    * Returns all column names and their data types as an array.
    * 以数组的形式返回所有列名称和它们的数据类型
    *
    * @group basic
    * @since 1.6.0
    */
  def dtypes: Array[(String, String)] = schema.fields.map { field =>
    (field.name, field.dataType.toString)
  }
```
### columns
```scala
  /**
    * Returns all column names as an array.
    * 以数组的形式返回 所有列名
    *
    * @group basic
    * @since 1.6.0
    */
  def columns: Array[String] = schema.fields.map(_.name)
```
### isLocal
```scala
  /**
    * Returns true if the `collect` and `take` methods can be run locally
    * (without any Spark executors).
    * 如果`collect` and `take` 方法能在本地运行，则返回true
    *
    * @group basic
    * @since 1.6.0
    */
  def isLocal: Boolean = logicalPlan.isInstanceOf[LocalRelation]
```
### checkpoint
```scala
  /**
    * Eagerly checkpoint a Dataset and return the new Dataset. Checkpointing can be used to truncate
    * the logical plan of this Dataset, which is especially useful in iterative algorithms where the
    * plan may grow exponentially. It will be saved to files inside the checkpoint
    * directory set with `SparkContext#setCheckpointDir`.
    * 
    * 急切地检查一个数据集并返回新的数据集。
    * 检查点能用来清除Dataset的逻辑计划，尤其是在可能生成指数级别的迭代算法中尤其有用。
    * 将会在检查点目录中保存检查文件。可以在`SparkContext#setCheckpointDir`中设置。
    *
    * @group basic
    * @since 2.1.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def checkpoint(): Dataset[T] = checkpoint(eager = true)
  
    /**
    * Returns a checkpointed version of this Dataset. Checkpointing can be used to truncate the
    * logical plan of this Dataset, which is especially useful in iterative algorithms where the
    * plan may grow exponentially. It will be saved to files inside the checkpoint
    * directory set with `SparkContext#setCheckpointDir`.
    * 返回Dataset 之前检查过的版本。
    * 检查点能用来清除Dataset的逻辑计划，尤其是在可能生成指数级别的迭代算法中尤其有用。
    * 将会在检查点目录中保存检查文件。可以在`SparkContext#setCheckpointDir`中设置。
    *
    * @group basic
    * @since 2.1.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def checkpoint(eager: Boolean): Dataset[T] = {
    val internalRdd = queryExecution.toRdd.map(_.copy())
    internalRdd.checkpoint()

    if (eager) {
      internalRdd.count()
    }

    val physicalPlan = queryExecution.executedPlan

    // Takes the first leaf partitioning whenever we see a `PartitioningCollection`. Otherwise the
    // size of `PartitioningCollection` may grow exponentially for queries involving deep inner
    // joins.
    // 每当我们看到“PartitioningCollection”时，就采用第一个叶子分区
    // 否则，用于涉及深度内连接的查询，“PartitioningCollection”的大小可能会以指数形式增长。
    def firstLeafPartitioning(partitioning: Partitioning): Partitioning = {
      partitioning match {
        case p: PartitioningCollection => firstLeafPartitioning(p.partitionings.head)
        case p => p
      }
    }

    val outputPartitioning = firstLeafPartitioning(physicalPlan.outputPartitioning)

    Dataset.ofRows(
      sparkSession,
      LogicalRDD(
        logicalPlan.output,
        internalRdd,
        outputPartitioning,
        physicalPlan.outputOrdering
      )(sparkSession)).as[T]
  }
```
### persist
```scala
  /**
    * Persist this Dataset with the default storage level (`MEMORY_AND_DISK`).
    *
    * 持久化。
    * 根据默认的 存储级别 (`MEMORY_AND_DISK`)  持久化Dataset。
    *
    * @group basic
    * @since 1.6.0
    */
  def persist(): this.type = {
    sparkSession.sharedState.cacheManager.cacheQuery(this)
    this
  }
  
    /**
    * Persist this Dataset with the given storage level.
    *
    * 根据指定的 存储级别 持久化 Dataset。
    *
    * @param newLevel One of:
    *                 `MEMORY_ONLY`,
    *                 `MEMORY_AND_DISK`,
    *                 `MEMORY_ONLY_SER`,
    *                 `MEMORY_AND_DISK_SER`,
    *                 `DISK_ONLY`,
    *                 `MEMORY_ONLY_2`, 与MEMORY_ONLY的区别是会备份数据到其他节点上
    *                 `MEMORY_AND_DISK_2`, 与MEMORY_AND_DISK的区别是会备份数据到其他节点上
    *                 etc.
    * @group basic
    * @since 1.6.0
    */
  def persist(newLevel: StorageLevel): this.type = {
    sparkSession.sharedState.cacheManager.cacheQuery(this, None, newLevel)
    this
  }
```
### cache
```scala
  /**
    * Persist this Dataset with the default storage level (`MEMORY_AND_DISK`).
    *
    * 持久化。
    * 根据默认的 存储级别 (`MEMORY_AND_DISK`)  持久化Dataset。
    * 和 persist 一致。
    *
    * @group basic
    * @since 1.6.0
    */
  def cache(): this.type = persist()
```
### storageLevel
```scala
  /**
    * Get the Dataset's current storage level, or StorageLevel.NONE if not persisted.
    *
    * 获取当前Dataset的当前存储级别。如果没有缓存则 StorageLevel.NONE。
    *
    * @group basic
    * @since 2.1.0
    */
  def storageLevel: StorageLevel = {
    sparkSession.sharedState.cacheManager.lookupCachedData(this).map { cachedData =>
      cachedData.cachedRepresentation.storageLevel
    }.getOrElse(StorageLevel.NONE)
  }
```
### unpersist
```scala
  /**
    * Mark the Dataset as non-persistent, and remove all blocks for it from memory and disk.
    *
    * 解除持久化。
    * 将Dataset标记为非持久化，并从内存和磁盘中移除所有的块。
    *
    * @param blocking Whether to block until all blocks are deleted.
    *                 是否阻塞，直到删除所有的块。
    * @group basic
    * @since 1.6.0
    */
  def unpersist(blocking: Boolean): this.type = {
    sparkSession.sharedState.cacheManager.uncacheQuery(this, blocking)
    this
  }
  
    /**
    * Mark the Dataset as non-persistent, and remove all blocks for it from memory and disk.
    *
    * 解除持久化。
    * 将Dataset标记为非持久化，并从内存和磁盘中移除所有的块。
    *
    * @group basic
    * @since 1.6.0
    */
  def unpersist(): this.type = unpersist(blocking = false)
```
### rdd
```scala
  /**
    * Represents the content of the Dataset as an `RDD` of [[T]].
    *
    * 转换为[[T]]的“RDD”，表示Dataset的内容
    *
    * @group basic
    * @since 1.6.0
    */
  lazy val rdd: RDD[T] = {
    val objectType = exprEnc.deserializer.dataType
    val deserialized = CatalystSerde.deserialize[T](logicalPlan)
    sparkSession.sessionState.executePlan(deserialized).toRdd.mapPartitions { rows =>
      rows.map(_.get(0, objectType).asInstanceOf[T])
    }
  }
```
### toJavaRDD
```scala
  /**
    * Returns the content of the Dataset as a `JavaRDD` of [[T]]s.
    * 
    * 转换为JavaRDD
    *
    * @group basic
    * @since 1.6.0
    */
  def toJavaRDD: JavaRDD[T] = rdd.toJavaRDD()
  
    /**
    * Returns the content of the Dataset as a `JavaRDD` of [[T]]s.
    *
    * 转换为JavaRDD
    *
    * @group basic
    * @since 1.6.0
    */
  def javaRDD: JavaRDD[T] = toJavaRDD
```
### registerTempTable
```scala
  /**
    * Registers this Dataset as a temporary table using the given name. The lifetime of this
    * temporary table is tied to the [[SparkSession]] that was used to create this Dataset.
    *
    * 根据指定的表名，注册临时表。
    * 生命周期为[[SparkSession]]的生命周期。
    *
    * @group basic
    * @since 1.6.0
    */
  @deprecated("Use createOrReplaceTempView(viewName) instead.", "2.0.0")
  def registerTempTable(tableName: String): Unit = {
    createOrReplaceTempView(tableName)
  }
```
### createTempView
```scala
  /**
    * Creates a local temporary view using the given name. The lifetime of this
    * temporary view is tied to the [[SparkSession]] that was used to create this Dataset.
    *
    * 用指定的名字创建本地临时表。
    * 与[[SparkSession]] 同生命周期。
    *
    * Local temporary view is session-scoped. Its lifetime is the lifetime of the session that
    * created it, i.e. it will be automatically dropped when the session terminates. It's not
    * tied to any databases, i.e. we can't use `db1.view1` to reference a local temporary view.
    *
    * 本地临时表是 session范围内的。当创建它的session停止的时候，该表也随之停止。
    *
    * @throws AnalysisException if the view name already exists
    * @group basic
    * @since 2.0.0
    */
  @throws[AnalysisException]
  def createTempView(viewName: String): Unit = withPlan {
    createTempViewCommand(viewName, replace = false, global = false)
  }
```
### createOrReplaceTempView
```scala
  /**
    * Creates a local temporary view using the given name. The lifetime of this
    * temporary view is tied to the [[SparkSession]] that was used to create this Dataset.
    *
    * 用指定的名字创建本地临时表。如果已经有了则替换。
    *
    * @group basic
    * @since 2.0.0
    */
  def createOrReplaceTempView(viewName: String): Unit = withPlan {
    createTempViewCommand(viewName, replace = true, global = false)
  }
```
### createGlobalTempView
```scala
  /**
    * Creates a global temporary view using the given name. The lifetime of this
    * temporary view is tied to this Spark application.
    *
    * 创建全局临时表。
    * 生命周期为整个Spark application.
    *
    * Global temporary view is cross-session. Its lifetime is the lifetime of the Spark application,
    * i.e. it will be automatically dropped when the application terminates. It's tied to a system
    * preserved database `_global_temp`, and we must use the qualified name to refer a global temp
    * view, e.g. `SELECT * FROM _global_temp.view1`.
    *
    * 全局临时表是跨session的。属于 _global_temp 数据库。e.g. `SELECT * FROM _global_temp.view1`.
    *
    * @throws AnalysisException if the view name already exists
    *                           如果表已经存在，则报错。
    * @group basic
    * @since 2.1.0
    */
  @throws[AnalysisException]
  def createGlobalTempView(viewName: String): Unit = withPlan {
    createTempViewCommand(viewName, replace = false, global = true)
  }
```
### write
```scala
  /**
    * Interface for saving the content of the non-streaming Dataset out into external storage.
    * 
    * 将非流Dataset的内容保存到外部存储中的接口。
    *
    * @group basic
    * @since 1.6.0
    */
  def write: DataFrameWriter[T] = {
    if (isStreaming) {
      logicalPlan.failAnalysis(
        "'write' can not be called on streaming Dataset/DataFrame")
    }
    new DataFrameWriter[T](this)
  }
```
### writeStream
```scala
  /**
    * :: Experimental ::
    * Interface for saving the content of the streaming Dataset out into external storage.
    *
    * 将流Dataset保存在外部存储。
    *
    * @group basic
    * @since 2.0.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def writeStream: DataStreamWriter[T] = {
    if (!isStreaming) {
      logicalPlan.failAnalysis(
        "'writeStream' can be called only on streaming Dataset/DataFrame")
    }
    new DataStreamWriter[T](this)
  }
```
### toJSON
```scala
  /**
    * Returns the content of the Dataset as a Dataset of JSON strings.
    *
    * 将Dataset转换为JSON。
    *
    * @since 2.0.0
    */
  def toJSON: Dataset[String] = {
    val rowSchema = this.schema
    val rdd: RDD[String] = queryExecution.toRdd.mapPartitions { iter =>
      val writer = new CharArrayWriter()
      // create the Generator without separator inserted between 2 records
      val gen = new JacksonGenerator(rowSchema, writer)

      new Iterator[String] {
        override def hasNext: Boolean = iter.hasNext

        override def next(): String = {
          gen.write(iter.next())
          gen.flush()

          val json = writer.toString
          if (hasNext) {
            writer.reset()
          } else {
            gen.close()
          }

          json
        }
      }
    }
    import sparkSession.implicits.newStringEncoder
    sparkSession.createDataset(rdd)
  }
```
### inputFiles
```scala
  /**
    * Returns a best-effort snapshot of the files that compose this Dataset. This method simply
    * asks each constituent BaseRelation for its respective files and takes the union of all results.
    * Depending on the source relations, this may not find all input files. Duplicates are removed.
    *
    * 返回组成这个Dataset的所有文件的最佳快照。
    * 该方法简单地要求每个组件BaseRelation对其各自的文件进行处理，并联合所有结果。
    * 基于源关系，应该可以找到所有的输入文件。
    * 重复的也会被移除。
    *
    * @group basic
    * @since 2.0.0
    */
  def inputFiles: Array[String] = {
    val files: Seq[String] = queryExecution.optimizedPlan.collect {
      case LogicalRelation(fsBasedRelation: FileRelation, _, _) =>
        fsBasedRelation.inputFiles
      case fr: FileRelation =>
        fr.inputFiles
    }.flatten
    files.toSet.toArray
  }
```
## streaming
### isStreaming
```scala
  /**
    * Returns true if this Dataset contains one or more sources that continuously
    * return data as it arrives. A Dataset that reads data from a streaming source
    * must be executed as a `StreamingQuery` using the `start()` method in
    * `DataStreamWriter`. Methods that return a single answer, e.g. `count()` or
    * `collect()`, will throw an [[AnalysisException]] when there is a streaming
    * source present.
    * 
    * 如果Dataset包含一个或多个持续返回数据的源，则返回true；
    * 如果Dataset从streaming源读取数据，则必须像 `StreamingQuery` 一样执行：使用 `DataStreamWriter` 中的 `start()`方法。
    * 返回单个值的方法，例如： `count()` or `collect()`，当存在streaming源时，将会抛出[[AnalysisException]]。
    *
    * @group streaming
    * @since 2.0.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def isStreaming: Boolean = logicalPlan.isStreaming
```
### withWatermark
```scala
  /**
    * :: Experimental :: 实验性的
    * Defines an event time watermark for this [[Dataset]]. A watermark tracks a point in time
    * before which we assume no more late data is going to arrive.
    * 
    * 为这个[[Dataset]]定义事件时间水印。
    * 我们假设没有更多的晚期数据将到达之前，一个水印跟踪一个时间点。
    *
    * Spark will use this watermark for several purposes:
    * Spark用水印有几个目的：
    *  - To know when a given time window aggregation can be finalized and thus can be emitted when
    * using output modes that do not allow updates.
    * 
    * 可以知道何时完成给定的时间窗口聚合能够完成，因此当使用不允许更新的输出模式时能够被放出。
    *  - To minimize the amount of state that we need to keep for on-going aggregations.
    * 为了最小化我们需要持续不断的聚合的状态数量。
    *
    *
    * The current watermark is computed by looking at the `MAX(eventTime)` seen across
    * all of the partitions in the query minus a user specified `delayThreshold`.  Due to the cost
    * of coordinating this value across partitions, the actual watermark used is only guaranteed
    * to be at least `delayThreshold` behind the actual event time.  In some cases we may still
    * process records that arrive more than `delayThreshold` late.
    * 
    * 当前的水印 = 查看查询中所有分区上看到的`MAX(eventTime)` - 用户指定的`delayThreshold`
    * 由于在分区之间协调这个值的花销，实际使用的水印只保证在实际事件时间后至少是“delayThreshold”。
    * 在某些情况下，我们可能还会处理比“delayThreshold”晚些时候到达的记录。
    *
    * @param eventTime      the name of the column that contains the event time of the row.
    *                       包含行的事件时间的列名
    * @param delayThreshold the minimum delay to wait to data to arrive late, relative to the latest
    *                       record that has been processed in the form of an interval
    *                       (e.g. "1 minute" or "5 hours").
    *                       等待晚到数据的最少延迟，相对于以间隔形式处理的最新记录
    * @group streaming
    * @since 2.1.0
    */
  @Experimental
  @InterfaceStability.Evolving
  // We only accept an existing column name, not a derived column here as a watermark that is
  // defined on a derived column cannot referenced elsewhere in the plan.
  // 我们只接受一个现有的列名，而不是作为一个在派生列上定义的水印的派生列，而不能在该计划的其他地方引用。
  def withWatermark(eventTime: String, delayThreshold: String): Dataset[T] = withTypedPlan {
    val parsedDelay =
      Option(CalendarInterval.fromString("interval " + delayThreshold))
        .getOrElse(throw new AnalysisException(s"Unable to parse time delay '$delayThreshold'"))
    EventTimeWatermark(UnresolvedAttribute(eventTime), parsedDelay, logicalPlan)
  }
```
## action
### show
```scala
  /**
    * Displays the Dataset in a tabular form. Strings more than 20 characters will be truncated,
    * and all cells will be aligned right. For example:
    * 
    * 以表格形式显示数据集。
    * 字符串超过20个字符将被截断，
    * 所有单元格将被对齐。
    * {{{
    *   year  month AVG('Adj Close) MAX('Adj Close)
    *   1980  12    0.503218        0.595103
    *   1981  01    0.523289        0.570307
    *   1982  02    0.436504        0.475256
    *   1983  03    0.410516        0.442194
    *   1984  04    0.450090        0.483521
    * }}}
    *
    * @param numRows Number of rows to show 要显示的行数
    * @group action
    * @since 1.6.0
    */
  def show(numRows: Int): Unit = show(numRows, truncate = true)
    /**
    * Displays the top 20 rows of Dataset in a tabular form. Strings more than 20 characters
    * will be truncated, and all cells will be aligned right.
    * 显示头20行
    *
    * @group action
    * @since 1.6.0
    */
  def show(): Unit = show(20)

  /**
    * Displays the top 20 rows of Dataset in a tabular form.
    * 显示头20行
    *
    * @param truncate Whether truncate long strings. If true, strings more than 20 characters will
    *                 be truncated and all cells will be aligned right
    *                 是否截断长字符串。如果 true：超过20个字符就会被截断
    * @group action
    * @since 1.6.0
    */
  def show(truncate: Boolean): Unit = show(20, truncate)

  /**
    * Displays the Dataset in a tabular form. For example:
    * {{{
    *   year  month AVG('Adj Close) MAX('Adj Close)
    *   1980  12    0.503218        0.595103
    *   1981  01    0.523289        0.570307
    *   1982  02    0.436504        0.475256
    *   1983  03    0.410516        0.442194
    *   1984  04    0.450090        0.483521
    * }}}
    *
    * @param numRows  Number of rows to show 显示的行数
    * @param truncate Whether truncate long strings. If true, strings more than 20 characters will
    *                 be truncated and all cells will be aligned right
    *                 是否截断长字符串
    * @group action
    * @since 1.6.0
    */
  // scalastyle:off println
  def show(numRows: Int, truncate: Boolean): Unit = if (truncate) {
    println(showString(numRows, truncate = 20))
  } else {
    println(showString(numRows, truncate = 0))
  }

  // scalastyle:on println

  /**
    * Displays the Dataset in a tabular form. For example:
    * {{{
    *   year  month AVG('Adj Close) MAX('Adj Close)
    *   1980  12    0.503218        0.595103
    *   1981  01    0.523289        0.570307
    *   1982  02    0.436504        0.475256
    *   1983  03    0.410516        0.442194
    *   1984  04    0.450090        0.483521
    * }}}
    *
    * @param numRows  Number of rows to show
    * @param truncate If set to more than 0, truncates strings to `truncate` characters and
    *                 all cells will be aligned right.
    *                 设置 触发截断字符串的阈值
    * @group action
    * @since 1.6.0
    */
  // scalastyle:off println
  def show(numRows: Int, truncate: Int): Unit = println(showString(numRows, truncate))
```
### reduce
```scala
  /**
    * :: Experimental ::
    * (Scala-specific)
    * Reduces the elements of this Dataset using the specified binary function. The given `func`
    * must be commutative and associative or the result may be non-deterministic.
    *
    * 使用指定的二进制函数减少这个数据集的元素。给定的“func”必须是可交换的和关联的，否则结果可能是不确定性的。
    *
    * @group action
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def reduce(func: (T, T) => T): T = rdd.reduce(func)

  /**
    * :: Experimental ::
    * (Java-specific)
    * Reduces the elements of this Dataset using the specified binary function. The given `func`
    * must be commutative and associative or the result may be non-deterministic.
    * 
    * 使用指定的二进制函数减少这个数据集的元素。给定的“func”必须是可交换的和关联的，否则结果可能是不确定性的。
    *
    * @group action
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def reduce(func: ReduceFunction[T]): T = reduce(func.call(_, _))
```
### describe
```scala
 /**
    * Computes statistics for numeric and string columns, including count, mean, stddev, min, and
    * max. If no columns are given, this function computes statistics for all numerical or string
    * columns.
    *
    * 计算数字和字符串列的统计数据，包括count、mean、stddev、min和max。
    * 如果没有给出任何列，该函数计算所有数值或字符串列的统计信息。
    *
    * This function is meant for exploratory data analysis, as we make no guarantee about the
    * backward compatibility of the schema of the resulting Dataset. If you want to
    * programmatically compute summary statistics, use the `agg` function instead.
    *
    * 这个函数用于探索性的数据分析，因为我们不能保证生成数据集的模式的向后兼容性。
    * 如果您想通过编程计算汇总统计信息，可以使用“agg”函数。
    *
    * {{{
    *   ds.describe("age", "height").show()
    *
    *   // output:
    *   // summary age   height
    *   // count   10.0  10.0
    *   // mean    53.3  178.05
    *   // stddev  11.6  15.7
    *   // min     18.0  163.0
    *   // max     92.0  192.0
    * }}}
    *
    * @group action
    * @since 1.6.0
    */
  @scala.annotation.varargs
  def describe(cols: String*): DataFrame = withPlan {

    // The list of summary statistics to compute, in the form of expressions.
    val statistics = List[(String, Expression => Expression)](
      "count" -> ((child: Expression) => Count(child).toAggregateExpression()),
      "mean" -> ((child: Expression) => Average(child).toAggregateExpression()),
      "stddev" -> ((child: Expression) => StddevSamp(child).toAggregateExpression()),
      "min" -> ((child: Expression) => Min(child).toAggregateExpression()),
      "max" -> ((child: Expression) => Max(child).toAggregateExpression()))

    val outputCols =
      (if (cols.isEmpty) aggregatableColumns.map(usePrettyExpression(_).sql) else cols).toList

    val ret: Seq[Row] = if (outputCols.nonEmpty) {
      val aggExprs = statistics.flatMap { case (_, colToAgg) =>
        outputCols.map(c => Column(Cast(colToAgg(Column(c).expr), StringType)).as(c))
      }

      val row = groupBy().agg(aggExprs.head, aggExprs.tail: _*).head().toSeq

      // Pivot the data so each summary is one row
      row.grouped(outputCols.size).toSeq.zip(statistics).map { case (aggregation, (statistic, _)) =>
        Row(statistic :: aggregation.toList: _*)
      }
    } else {
      // If there are no output columns, just output a single column that contains the stats.
      statistics.map { case (name, _) => Row(name) }
    }

    // All columns are string type
    val schema = StructType(
      StructField("summary", StringType) :: outputCols.map(StructField(_, StringType))).toAttributes
    // `toArray` forces materialization to make the seq serializable
    LocalRelation.fromExternalRows(schema, ret.toArray.toSeq)
  }
```
### head
```scala
  /**
    * Returns the first `n` rows.
    *
    * 返回前n行
    *
    * @note this method should only be used if the resulting array is expected to be small, as
    *       all the data is loaded into the driver's memory.
    *       仅适用于结果很少的时候使用，因为会将结果加载进内存中
    * @group action
    * @since 1.6.0
    */
  def head(n: Int): Array[T] = withTypedCallback("head", limit(n)) { df =>
    df.collect(needCallback = false)
  }

  /**
    * Returns the first row.
    *
    * 返回第一行（默认1）
    *
    * @group action
    * @since 1.6.0
    */
  def head(): T = head(1).head
```
### first
```scala
  /**
    * Returns the first row. Alias for head().
    *
    * 返回第一行 ，与head()一样
    *
    * @group action
    * @since 1.6.0
    */
  def first(): T = head()
```
### foreach
```scala
  /**
    * Applies a function `f` to all rows.
    *
    * 对所有行应用函数f。
    *
    * @group action
    * @since 1.6.0
    */
  def foreach(f: T => Unit): Unit = withNewExecutionId {
    rdd.foreach(f)
  }

  /**
    * (Java-specific)
    * Runs `func` on each element of this Dataset.
    *
    * 在这个数据集的每个元素上运行“func”。
    *
    * @group action
    * @since 1.6.0
    */
  def foreach(func: ForeachFunction[T]): Unit = foreach(func.call(_))
```
### foreachPartition
```scala
  /**
    * Applies a function `f` to each partition of this Dataset.
    *
    * 对这个数据集的每个分区应用一个函数f。
    *
    * @group action
    * @since 1.6.0
    */
  def foreachPartition(f: Iterator[T] => Unit): Unit = withNewExecutionId {
    rdd.foreachPartition(f)
  }

  /**
    * (Java-specific)
    * Runs `func` on each partition of this Dataset.
    *
    * 对这个数据集的每个分区应用一个函数f。
    *
    * @group action
    * @since 1.6.0
    */
  def foreachPartition(func: ForeachPartitionFunction[T]): Unit =
    foreachPartition(it => func.call(it.asJava))
```
### take
```scala
  /**
    * Returns the first `n` rows in the Dataset.
    *
    * 返回数据集中的前“n”行。
    * 同head(n)
    *
    * Running take requires moving data into the application's driver process, and doing so with
    * a very large `n` can crash the driver process with OutOfMemoryError.
    *
    * take在driver端执行，n太大会造成oom
    *
    * @group action
    * @since 1.6.0
    */
  def take(n: Int): Array[T] = head(n)
```
### takeAsList
```scala
  /**
    * Returns the first `n` rows in the Dataset as a list.
    *
    * 以List形式返回 前n行
    *
    * Running take requires moving data into the application's driver process, and doing so with
    * a very large `n` can crash the driver process with OutOfMemoryError.
    *
    * take在driver端执行，n太大会造成oom
    *
    * @group action
    * @since 1.6.0
    */
  def takeAsList(n: Int): java.util.List[T] = java.util.Arrays.asList(take(n): _*)
```
### collect
```scala
  /**
    * Returns an array that contains all of [[Row]]s in this Dataset.
    *
    * 返回包含所有Row的 一个数组
    *
    * Running collect requires moving all the data into the application's driver process, and
    * doing so on a very large dataset can crash the driver process with OutOfMemoryError.
    *
    * 会将所有数据移动到driver，所以可能会造成oom
    *
    * For Java API, use [[collectAsList]].
    *
    * @group action
    * @since 1.6.0
    */
  def collect(): Array[T] = collect(needCallback = true)
```
### collectAsList
```scala
  /**
    * Returns a Java list that contains all of [[Row]]s in this Dataset.
    *
    * 返回包含所有Row的一个Java List
    *
    * Running collect requires moving all the data into the application's driver process, and
    * doing so on a very large dataset can crash the driver process with OutOfMemoryError.
    *
    * 会将所有数据移动到driver，所以可能会造成oom
    *
    * @group action
    * @since 1.6.0
    */
  def collectAsList(): java.util.List[T] = withCallback("collectAsList", toDF()) { _ =>
    withNewExecutionId {
      val values = queryExecution.executedPlan.executeCollect().map(boundEnc.fromRow)
      java.util.Arrays.asList(values: _*)
    }
  }
```
### toLocalIterator
```scala
  /**
    * Return an iterator that contains all of [[Row]]s in this Dataset.
    *
    * 返回包含所有Row的一个迭代器
    *
    * The iterator will consume as much memory as the largest partition in this Dataset.
    *
    * 迭代器将消耗与此数据集中最大的分区一样多的内存。
    *
    * @note this results in multiple Spark jobs, and if the input Dataset is the result
    *       of a wide transformation (e.g. join with different partitioners), to avoid
    *       recomputing the input Dataset should be cached first.
    *       这将导致多个Spark作业，如果输入数据集是宽依赖转换的结果(例如，与不同的分区连接)，
    *       那么为了避免重新计算输入数据，应该首先缓存输入数据集。
    * @group action
    * @since 2.0.0
    */
  def toLocalIterator(): java.util.Iterator[T] = withCallback("toLocalIterator", toDF()) { _ =>
    withNewExecutionId {
      queryExecution.executedPlan.executeToIterator().map(boundEnc.fromRow).asJava
    }
  }
```
### count
```scala
  /**
    * Returns the number of rows in the Dataset.
    *
    * 返回总行数
    *
    * @group action
    * @since 1.6.0
    */
  def count(): Long = withCallback("count", groupBy().count()) { df =>
    df.collect(needCallback = false).head.getLong(0)
  }
```
## untypedrel-无类型转换
### na
```scala
  /**
    * Returns a [[DataFrameNaFunctions]] for working with missing data.
    * 返回一个用于处理丢失数据的[[DataFrameNaFunctions]]。
    * {{{
    *   // Dropping rows containing any null values. 删除包含任何null 值的行
    *   ds.na.drop()
    * }}}
    *
    * @group untypedrel
    * @since 1.6.0
    */
  def na: DataFrameNaFunctions = new DataFrameNaFunctions(toDF())
```
### stat
```scala
  /**
    * Returns a [[DataFrameStatFunctions]] for working statistic functions support.
    * 返回用于支持统计功能的[[DataFrameStatFunctions]]。
    * {{{
    *   // Finding frequent items in column with name 'a'. 查询列名为"a"中的频繁数据。
    *   ds.stat.freqItems(Seq("a"))
    * }}}
    *
    * @group untypedrel
    * @since 1.6.0
    */
  def stat: DataFrameStatFunctions = new DataFrameStatFunctions(toDF())
```
### join
```scala
/**
    * Join with another `DataFrame`.
    * 和 另一个 `DataFrame`  jion
    *
    * Behaves as an INNER JOIN and requires a subsequent join predicate.
    * 作为一个内部连接，并需要一个后续的连接谓词。
    *
    * @param right Right side of the join operation. join操作的右侧
    * @group untypedrel
    * @since 2.0.0
    */
  def join(right: Dataset[_]): DataFrame = withPlan {
    Join(logicalPlan, right.logicalPlan, joinType = Inner, None)
  }

  /**
    * Inner equi-join with another `DataFrame` using the given column.
    * 给定列名的内部等值连接
    *
    * Different from other join functions, the join column will only appear once in the output,
    * i.e. similar to SQL's `JOIN USING` syntax.
    *
    * {{{
    *   // Joining df1 and df2 using the column "user_id" 用"user_id"  连接 df1 和df2
    *   df1.join(df2, "user_id")
    * }}}
    *
    * @param right       Right side of the join operation. join连接右侧
    * @param usingColumn Name of the column to join on. This column must exist on both sides.
    *                    列名。必须在两边都存在
    * @note If you perform a self-join using this function without aliasing the input
    *       `DataFrame`s, you will NOT be able to reference any columns after the join, since
    *       there is no way to disambiguate which side of the join you would like to reference.
    *       自连接的时候，请指定 表别名。不然干不了事
    * @group untypedrel
    * @since 2.0.0
    */
  def join(right: Dataset[_], usingColumn: String): DataFrame = {
    join(right, Seq(usingColumn))
  }

  /**
    * Inner equi-join with another `DataFrame` using the given columns.
    * 根据指定多个列进行join
    *
    * Different from other join functions, the join columns will only appear once in the output,
    * i.e. similar to SQL's `JOIN USING` syntax.
    *
    * {{{
    *   // Joining df1 and df2 using the columns "user_id" and "user_name"
    *   df1.join(df2, Seq("user_id", "user_name"))
    * }}}
    *
    * @param right        Right side of the join operation.
    * @param usingColumns Names of the columns to join on. This columns must exist on both sides.
    * @note If you perform a self-join using this function without aliasing the input
    *       `DataFrame`s, you will NOT be able to reference any columns after the join, since
    *       there is no way to disambiguate which side of the join you would like to reference.
    * @group untypedrel
    * @since 2.0.0
    */
  def join(right: Dataset[_], usingColumns: Seq[String]): DataFrame = {
    join(right, usingColumns, "inner")
  }

  /**
    * Equi-join with another `DataFrame` using the given columns.
    *
    * Different from other join functions, the join columns will only appear once in the output,
    * i.e. similar to SQL's `JOIN USING` syntax.
    *
    * @param right        Right side of the join operation.
    * @param usingColumns Names of the columns to join on. This columns must exist on both sides.
    * @param joinType     One of: `inner`, `outer`, `left_outer`, `right_outer`, `leftsemi`.
    *                     连接类型：内连接，外连接，左外连接，右外连接，左内连接
    * @note If you perform a self-join using this function without aliasing the input
    *       `DataFrame`s, you will NOT be able to reference any columns after the join, since
    *       there is no way to disambiguate which side of the join you would like to reference.
    * @group untypedrel
    * @since 2.0.0
    */
  def join(right: Dataset[_], usingColumns: Seq[String], joinType: String): DataFrame = {
    // Analyze the self join. The assumption is that the analyzer will disambiguate left vs right
    // by creating a new instance for one of the branch.
    // 自连接的时候，为其中一个分支创建一个新实例来消除左vs右的歧义。
    val joined = sparkSession.sessionState.executePlan(
      Join(logicalPlan, right.logicalPlan, joinType = JoinType(joinType), None))
      .analyzed.asInstanceOf[Join]

    withPlan {
      Join(
        joined.left,
        joined.right,
        UsingJoin(JoinType(joinType), usingColumns),
        None)
    }
  }

  /**
    * Inner join with another `DataFrame`, using the given join expression.
    * 用给定的表达式进行join
    * {{{
    *   // The following two are equivalent:
    *   df1.join(df2, $"df1Key" === $"df2Key")
    *   df1.join(df2).where($"df1Key" === $"df2Key")
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  def join(right: Dataset[_], joinExprs: Column): DataFrame = join(right, joinExprs, "inner")

  /**
    * Join with another `DataFrame`, using the given join expression. The following performs
    * a full outer join between `df1` and `df2`.
    *
    * {{{
    *   // Scala:
    *   import org.apache.spark.sql.functions._
    *   df1.join(df2, $"df1Key" === $"df2Key", "outer")
    *
    *   // Java:
    *   import static org.apache.spark.sql.functions.*;
    *   df1.join(df2, col("df1Key").equalTo(col("df2Key")), "outer");
    * }}}
    *
    * @param right     Right side of the join.
    * @param joinExprs Join expression.
    * @param joinType  One of: `inner`, `outer`, `left_outer`, `right_outer`, `leftsemi`.
    * @group untypedrel
    * @since 2.0.0
    */
  def join(right: Dataset[_], joinExprs: Column, joinType: String): DataFrame = {
    // Note that in this function, we introduce a hack in the case of self-join to automatically
    // resolve ambiguous join conditions into ones that might make sense [SPARK-6231].
    // Consider this case: df.join(df, df("key") === df("key"))
    // Since df("key") === df("key") is a trivially true condition, this actually becomes a
    // cartesian join. However, most likely users expect to perform a self join using "key".
    // With that assumption, this hack turns the trivially true condition into equality on join
    // keys that are resolved to both sides.

    // Trigger analysis so in the case of self-join, the analyzer will clone the plan.
    // After the cloning, left and right side will have distinct expression ids.
    // 针对自连接的优化：正常情况下，自连接如果使用  df.join(df, df("key") === df("key"))
    // 会造成 笛卡尔积
    // 这种情况下，分析器会 克隆计划，克隆完成后，左右两边则有不同的 id

    val plan = withPlan(
      Join(logicalPlan, right.logicalPlan, JoinType(joinType), Some(joinExprs.expr)))
      .queryExecution.analyzed.asInstanceOf[Join]

    // If auto self join alias is disabled, return the plan.
    if (!sparkSession.sessionState.conf.dataFrameSelfJoinAutoResolveAmbiguity) {
      return withPlan(plan)
    }

    // If left/right have no output set intersection, return the plan.
    val lanalyzed = withPlan(this.logicalPlan).queryExecution.analyzed
    val ranalyzed = withPlan(right.logicalPlan).queryExecution.analyzed
    if (lanalyzed.outputSet.intersect(ranalyzed.outputSet).isEmpty) {
      return withPlan(plan)
    }

    // Otherwise, find the trivially true predicates and automatically resolves them to both sides.
    // By the time we get here, since we have already run analysis, all attributes should've been
    // resolved and become AttributeReference.
    val cond = plan.condition.map {
      _.transform {
        case catalyst.expressions.EqualTo(a: AttributeReference, b: AttributeReference)
          if a.sameRef(b) =>
          catalyst.expressions.EqualTo(
            withPlan(plan.left).resolve(a.name),
            withPlan(plan.right).resolve(b.name))
      }
    }

    withPlan {
      plan.copy(condition = cond)
    }
  }

```
### crossJoin
```scala
  /**
    * Explicit cartesian join with another `DataFrame`.
    * 显式笛卡尔积join
    *
    * @param right Right side of the join operation.
    * @note Cartesian joins are very expensive without an extra filter that can be pushed down.
    *       如果没有额外的过滤器，笛卡尔连接非常昂贵。
    * @group untypedrel
    * @since 2.1.0
    */
  def crossJoin(right: Dataset[_]): DataFrame = withPlan {
    Join(logicalPlan, right.logicalPlan, joinType = Cross, None)
  }
```
### apply
```scala
  /**
    * Selects column based on the column name and return it as a [[Column]].
    *
    * 选择基于列名的列，并将其作为[[Column]]返回。
    *
    * @note The column name can also reference to a nested column like `a.b`.
    *
    *       列名也可以引用像“a.b”这样的嵌套列。
    * @group untypedrel
    * @since 2.0.0
    */
  def apply(colName: String): Column = col(colName)
```
### col
```scala
  /**
    * Selects column based on the column name and return it as a [[Column]].
    *
    * 选择基于列名的列，并将其作为[[Column]]返回。
    *
    * @note The column name can also reference to a nested column like `a.b`.
    *
    *       列名也可以引用像“a.b”这样的嵌套列。
    * @group untypedrel
    * @since 2.0.0
    */
  def col(colName: String): Column = colName match {
    case "*" =>
      Column(ResolvedStar(queryExecution.analyzed.output))
    case _ =>
      val expr = resolve(colName)
      Column(expr)
  }
```
### select
```scala
  /**
    * Selects a set of column based expressions.
    * {{{
    *   ds.select($"colA", $"colB" + 1)
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def select(cols: Column*): DataFrame = withPlan {
    Project(cols.map(_.named), logicalPlan)
  }

  /**
    * Selects a set of columns. This is a variant of `select` that can only select
    * existing columns using column names (i.e. cannot construct expressions).
    *
    * 只能是已经存在的列名
    *
    * {{{
    *   // The following two are equivalent:
    *   ds.select("colA", "colB")
    *   ds.select($"colA", $"colB")
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def select(col: String, cols: String*): DataFrame = select((col +: cols).map(Column(_)): _*)
```
### selectExpr
```scala
  /**
    * Selects a set of SQL expressions. This is a variant of `select` that accepts
    * SQL expressions.
    *
    * 接受SQL表达式
    *
    * {{{
    *   // The following are equivalent:
    *   以下是等价的:
    *   ds.selectExpr("colA", "colB as newName", "abs(colC)")
    *   ds.select(expr("colA"), expr("colB as newName"), expr("abs(colC)"))
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def selectExpr(exprs: String*): DataFrame = {
    select(exprs.map { expr =>
      Column(sparkSession.sessionState.sqlParser.parseExpression(expr))
    }: _*)
  }
```
### groupBy
```scala
  /**
    * Groups the Dataset using the specified columns, so we can run aggregation on them. See
    * [[RelationalGroupedDataset]] for all the available aggregate functions.
    *
    * 使用指定的列对数据集进行分组，这样我们就可以对它们进行聚合。
    * 查看[[RelationalGroupedDataset]]为所有可用的聚合函数。
    *
    *
    * {{{
    *   // Compute the average for all numeric columns grouped by department.
    *
    *   计算按部门分组的所有数字列的平均值。
    *
    *   ds.groupBy($"department").avg()
    *
    *   // Compute the max age and average salary, grouped by department and gender.
    *   ds.groupBy($"department", $"gender").agg(Map(
    *     "salary" -> "avg",
    *     "age" -> "max"
    *   ))
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def groupBy(cols: Column*): RelationalGroupedDataset = {
    RelationalGroupedDataset(toDF(), cols.map(_.expr), RelationalGroupedDataset.GroupByType)
  }
  
    /**
    * Groups the Dataset using the specified columns, so that we can run aggregation on them.
    * See [[RelationalGroupedDataset]] for all the available aggregate functions.
    *
    * This is a variant of groupBy that can only group by existing columns using column names
    * (i.e. cannot construct expressions).
    *
    * {{{
    *   // Compute the average for all numeric columns grouped by department.
    *   ds.groupBy("department").avg()
    *
    *   // Compute the max age and average salary, grouped by department and gender.
    *   ds.groupBy($"department", $"gender").agg(Map(
    *     "salary" -> "avg",
    *     "age" -> "max"
    *   ))
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def groupBy(col1: String, cols: String*): RelationalGroupedDataset = {
    val colNames: Seq[String] = col1 +: cols
    RelationalGroupedDataset(
      toDF(), colNames.map(colName => resolve(colName)), RelationalGroupedDataset.GroupByType)
  }
```
### rollup
```scala
/**
    * Create a multi-dimensional rollup for the current Dataset using the specified columns,
    * so we can run aggregation on them.
    * See [[RelationalGroupedDataset]] for all the available aggregate functions.
    *
    * 使用指定的列为当前数据集创建多维的汇总，因此我们可以在它们上运行聚合。
    *
    *
    * {{{
    *   // Compute the average for all numeric columns rolluped by department and group.
    *
    *   汇总后 求平均值
    *
    *   ds.rollup($"department", $"group").avg()
    *
    *   // Compute the max age and average salary, rolluped by department and gender.
    *   ds.rollup($"department", $"gender").agg(Map(
    *     "salary" -> "avg",
    *     "age" -> "max"
    *   ))
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def rollup(cols: Column*): RelationalGroupedDataset = {
    RelationalGroupedDataset(toDF(), cols.map(_.expr), RelationalGroupedDataset.RollupType)
  }
  
    /**
    * Create a multi-dimensional rollup for the current Dataset using the specified columns,
    * so we can run aggregation on them.
    * See [[RelationalGroupedDataset]] for all the available aggregate functions.
    *
    * 使用指定的列为当前数据集创建多维的rollup，因此我们可以在它们上运行聚合。
    * rollup可以实现 从右到左一次递减的多级统计，显示统计某一层次结构的聚合
    * 例如 rollup(a,b,c,d) =结果=> (a,b,c,d),(a,b,c),(a,b),a
    *
    * This is a variant of rollup that can only group by existing columns using column names
    * (i.e. cannot construct expressions).
    *
    * {{{
    *   // Compute the average for all numeric columns rolluped by department and group.
    *   ds.rollup("department", "group").avg()
    *
    *   // Compute the max age and average salary, rolluped by department and gender.
    *   ds.rollup($"department", $"gender").agg(Map(
    *     "salary" -> "avg",
    *     "age" -> "max"
    *   ))
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def rollup(col1: String, cols: String*): RelationalGroupedDataset = {
    val colNames: Seq[String] = col1 +: cols
    RelationalGroupedDataset(
      toDF(), colNames.map(colName => resolve(colName)), RelationalGroupedDataset.RollupType)
  }
```
### cube
```scala
/**
    * Create a multi-dimensional cube for the current Dataset using the specified columns,
    * so we can run aggregation on them.
    * See [[RelationalGroupedDataset]] for all the available aggregate functions.
    *
    * 使用指定的列为当前数据集创建多维数据集，因此我们可以在它们上运行聚合。
    *
    *
    * {{{
    *   // Compute the average for all numeric columns cubed by department and group.
    *   ds.cube($"department", $"group").avg()
    *
    *   // Compute the max age and average salary, cubed by department and gender.
    *   ds.cube($"department", $"gender").agg(Map(
    *     "salary" -> "avg",
    *     "age" -> "max"
    *   ))
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def cube(cols: Column*): RelationalGroupedDataset = {
    RelationalGroupedDataset(toDF(), cols.map(_.expr), RelationalGroupedDataset.CubeType)
  }
  
   /**
    * Create a multi-dimensional cube for the current Dataset using the specified columns,
    * so we can run aggregation on them.
    * See [[RelationalGroupedDataset]] for all the available aggregate functions.
    *
    * 魔方 例如：cube(a,b,c) =结果=> (a,b),(a,c),a,(b,c),b,c 结果为所有的维度
    * 使用指定的列为当前数据集创建多维多维数据集，因此我们可以在它们上运行聚合。
    *
    * This is a variant of cube that can only group by existing columns using column names
    * (i.e. cannot construct expressions).
    *
    * 这是一个多维数据集的变体，它只能通过使用列名的现有列来分组
    *
    * {{{
    *   // Compute the average for all numeric columns cubed by department and group.
    *   ds.cube("department", "group").avg()
    *
    *   // Compute the max age and average salary, cubed by department and gender.
    *   ds.cube($"department", $"gender").agg(Map(
    *     "salary" -> "avg",
    *     "age" -> "max"
    *   ))
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def cube(col1: String, cols: String*): RelationalGroupedDataset = {
    val colNames: Seq[String] = col1 +: cols
    RelationalGroupedDataset(
      toDF(), colNames.map(colName => resolve(colName)), RelationalGroupedDataset.CubeType)
  }
```
### agg
```scala
 /**
    * (Scala-specific) Aggregates on the entire Dataset without groups.
    * 对整个数据集进行聚合，无需分组。
    * {{{
    *   // ds.agg(...) is a shorthand for ds.groupBy().agg(...)
    *   ds.agg("age" -> "max", "salary" -> "avg")
    *   ds.groupBy().agg("age" -> "max", "salary" -> "avg")
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  def agg(aggExpr: (String, String), aggExprs: (String, String)*): DataFrame = {
    groupBy().agg(aggExpr, aggExprs: _*)
  }

  /**
    * (Scala-specific) Aggregates on the entire Dataset without groups.
    * 对整个数据集进行聚合，无需分组。
    *
    * {{{
    *   // ds.agg(...) is a shorthand for ds.groupBy().agg(...)
    *   ds.agg(Map("age" -> "max", "salary" -> "avg"))
    *   ds.groupBy().agg(Map("age" -> "max", "salary" -> "avg"))
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  def agg(exprs: Map[String, String]): DataFrame = groupBy().agg(exprs)

  /**
    * (Java-specific) Aggregates on the entire Dataset without groups.
    *
    * 对整个数据集进行聚合，无需分组。
    *
    * {{{
    *   // ds.agg(...) is a shorthand for ds.groupBy().agg(...)
    *   ds.agg(Map("age" -> "max", "salary" -> "avg"))
    *   ds.groupBy().agg(Map("age" -> "max", "salary" -> "avg"))
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  def agg(exprs: java.util.Map[String, String]): DataFrame = groupBy().agg(exprs)

  /**
    * Aggregates on the entire Dataset without groups.
    *
    * 对整个数据集进行聚合，无需分组。
    *
    * {{{
    *   // ds.agg(...) is a shorthand for ds.groupBy().agg(...)
    *   ds.agg(max($"age"), avg($"salary"))
    *   ds.groupBy().agg(max($"age"), avg($"salary"))
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def agg(expr: Column, exprs: Column*): DataFrame = groupBy().agg(expr, exprs: _*)

```
### explode
```scala
  /**
    * (Scala-specific) Returns a new Dataset where each row has been expanded to zero or more
    * rows by the provided function. This is similar to a `LATERAL VIEW` in HiveQL. The columns of
    * the input row are implicitly joined with each row that is output by the function.
    *
    * 根据提供的方法，该数据集的每一行都被扩展为零个或更多的行，返回一个新的数据集。
    * 这类似于HiveQL的“LATERAL VIEW”。
    * 输入行的列 隐式地加入了由函数输出的每一行。
    *
    * Given that this is deprecated, as an alternative, you can explode columns either using
    * `functions.explode()` or `flatMap()`. The following example uses these alternatives to count
    * the number of books that contain a given word:
    *
    * 考虑到这已经被弃用，作为替代，您可以使用“functions.explode()”或“flatMap()”来引爆列。
    * 下面的示例使用这些替代方法来计算包含给定单词的图书的数量:
    *
    * {{{
    *   case class Book(title: String, words: String)
    *   val ds: Dataset[Book]
    *
    *   val allWords = ds.select('title, explode(split('words, " ")).as("word"))
    *
    *   val bookCountPerWord = allWords.groupBy("word").agg(countDistinct("title"))
    * }}}
    *
    * Using `flatMap()` this can similarly be exploded as:
    *
    * {{{
    *   ds.flatMap(_.words.split(" "))
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0 已经过时，用 flatMap() 或 functions.explode() 代替
    */
  @deprecated("use flatMap() or select() with functions.explode() instead", "2.0.0")
  def explode[A <: Product : TypeTag](input: Column*)(f: Row => TraversableOnce[A]): DataFrame = {
    val elementSchema = ScalaReflection.schemaFor[A].dataType.asInstanceOf[StructType]

    val convert = CatalystTypeConverters.createToCatalystConverter(elementSchema)

    val rowFunction =
      f.andThen(_.map(convert(_).asInstanceOf[InternalRow]))
    val generator = UserDefinedGenerator(elementSchema, rowFunction, input.map(_.expr))

    withPlan {
      Generate(generator, join = true, outer = false,
        qualifier = None, generatorOutput = Nil, logicalPlan)
    }
  }

  /**
    * (Scala-specific) Returns a new Dataset where a single column has been expanded to zero
    * or more rows by the provided function. This is similar to a `LATERAL VIEW` in HiveQL. All
    * columns of the input row are implicitly joined with each value that is output by the function.
    *
    * Given that this is deprecated, as an alternative, you can explode columns either using
    * `functions.explode()`:
    *
    * {{{
    *   ds.select(explode(split('words, " ")).as("word"))
    * }}}
    *
    * or `flatMap()`:
    *
    * {{{
    *   ds.flatMap(_.words.split(" "))
    * }}}
    *
    * @group untypedrel
    * @since 2.0.0
    */
  @deprecated("use flatMap() or select() with functions.explode() instead", "2.0.0")
  def explode[A, B: TypeTag](inputColumn: String, outputColumn: String)(f: A => TraversableOnce[B])
  : DataFrame = {
    val dataType = ScalaReflection.schemaFor[B].dataType
    val attributes = AttributeReference(outputColumn, dataType)() :: Nil
    // TODO handle the metadata?
    val elementSchema = attributes.toStructType

    def rowFunction(row: Row): TraversableOnce[InternalRow] = {
      val convert = CatalystTypeConverters.createToCatalystConverter(dataType)
      f(row(0).asInstanceOf[A]).map(o => InternalRow(convert(o)))
    }

    val generator = UserDefinedGenerator(elementSchema, rowFunction, apply(inputColumn).expr :: Nil)

    withPlan {
      Generate(generator, join = true, outer = false,
        qualifier = None, generatorOutput = Nil, logicalPlan)
    }
  }
```
### withColumn
```scala
  /**
    * Returns a new Dataset by adding a column or replacing the existing column that has
    * the same name.
    * 通过添加一个列或替换具有相同名称的现有列返回新的数据集。
    *
    * @group untypedrel
    * @since 2.0.0
    */
  def withColumn(colName: String, col: Column): DataFrame = {
    val resolver = sparkSession.sessionState.analyzer.resolver
    val output = queryExecution.analyzed.output
    val shouldReplace = output.exists(f => resolver(f.name, colName))
    if (shouldReplace) {
      val columns = output.map { field =>
        if (resolver(field.name, colName)) {
          col.as(colName)
        } else {
          Column(field)
        }
      }
      select(columns: _*)
    } else {
      select(Column("*"), col.as(colName))
    }
  }

  /**
    * Returns a new Dataset by adding a column with metadata.
    * 通过添加带有元数据的列返回一个新的数据集。
    */
  private[spark] def withColumn(colName: String, col: Column, metadata: Metadata): DataFrame = {
    val resolver = sparkSession.sessionState.analyzer.resolver
    val output = queryExecution.analyzed.output
    val shouldReplace = output.exists(f => resolver(f.name, colName))
    if (shouldReplace) {
      val columns = output.map { field =>
        if (resolver(field.name, colName)) {
          col.as(colName, metadata)
        } else {
          Column(field)
        }
      }
      select(columns: _*)
    } else {
      select(Column("*"), col.as(colName, metadata))
    }
  }
```
### withColumnRenamed
```scala
  /**
    * Returns a new Dataset with a column renamed.
    * This is a no-op if schema doesn't contain existingName.
    * 返回一个重命名的列的新数据集。
    * 如果模式不包含存在名称，那么这是不操作的。
    *
    * @group untypedrel
    * @since 2.0.0
    */
  def withColumnRenamed(existingName: String, newName: String): DataFrame = {
    val resolver = sparkSession.sessionState.analyzer.resolver
    val output = queryExecution.analyzed.output
    val shouldRename = output.exists(f => resolver(f.name, existingName))
    if (shouldRename) {
      val columns = output.map { col =>
        if (resolver(col.name, existingName)) {
          Column(col).as(newName)
        } else {
          Column(col)
        }
      }
      select(columns: _*)
    } else {
      toDF()
    }
  }
```
### drop
```scala
  /**
    * Returns a new Dataset with a column dropped. This is a no-op if schema doesn't contain
    * column name.
    *
    * 返回删除指定列之后的新Dataset
    *
    * This method can only be used to drop top level columns. the colName string is treated
    * literally without further interpretation.
    *
    * 仅用于删除顶层的列
    *
    * @group untypedrel
    * @since 2.0.0
    */
  def drop(colName: String): DataFrame = {
    drop(Seq(colName): _*)
  }

  /**
    * Returns a new Dataset with columns dropped.
    * This is a no-op if schema doesn't contain column name(s).
    *
    * 删除指定的多个列，并返回新的dataset
    *
    * This method can only be used to drop top level columns. the colName string is treated literally
    * without further interpretation.
    *
    * @group untypedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def drop(colNames: String*): DataFrame = {
    val resolver = sparkSession.sessionState.analyzer.resolver
    val allColumns = queryExecution.analyzed.output
    val remainingCols = allColumns.filter { attribute =>
      colNames.forall(n => !resolver(attribute.name, n))
    }.map(attribute => Column(attribute))
    if (remainingCols.size == allColumns.size) {
      toDF()
    } else {
      this.select(remainingCols: _*)
    }
  }

  /**
    * Returns a new Dataset with a column dropped.
    * This version of drop accepts a [[Column]] rather than a name.
    * This is a no-op if the Dataset doesn't have a column
    * with an equivalent expression.
    *
    * 删除指定的 列（根据Column）
    *
    * @group untypedrel
    * @since 2.0.0
    */
  def drop(col: Column): DataFrame = {
    val expression = col match {
      case Column(u: UnresolvedAttribute) =>
        queryExecution.analyzed.resolveQuoted(
          u.name, sparkSession.sessionState.analyzer.resolver).getOrElse(u)
      case Column(expr: Expression) => expr
    }
    val attrs = this.logicalPlan.output
    val colsAfterDrop = attrs.filter { attr =>
      attr != expression
    }.map(attr => Column(attr))
    select(colsAfterDrop: _*)
  }
```
## typedrel-有类型的转换

### joinWith
```scala
  /**
    * :: Experimental ::  实验的
    * Joins this Dataset returning a `Tuple2` for each pair where `condition` evaluates to
    * true.
    * 连接这个数据集返回一个“Tuple2”对每一对的“条件”计算为true。
    *
    * This is similar to the relation `join` function with one important difference in the
    * result schema. Since `joinWith` preserves objects present on either side of the join, the
    * result schema is similarly nested into a tuple under the column names `_1` and `_2`.
    * 这类似于关系“join”函数，在结果模式中有一个重要的区别。
    * 由于“joinWith”保存了连接的任何一边的对象，因此结果模式类似地嵌套在列名称“_1”和“_2”下面的tuple中。
    *
    * This type of join can be useful both for preserving type-safety with the original object
    * types as well as working with relational data where either side of the join has column
    * names in common.
    * 这种类型的联接既可以用于保存与原始对象类型的类型安全性，
    * 也可以用于处理连接的任何一端都有列名的关系数据。
    *
    * @param other     Right side of the join.
    * @param condition Join expression.
    * @param joinType  One of: `inner`, `outer`, `left_outer`, `right_outer`, `leftsemi`.
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def joinWith[U](other: Dataset[U], condition: Column, joinType: String): Dataset[(T, U)] = {
    // Creates a Join node and resolve it first, to get join condition resolved, self-join resolved,
    // 创建一个联接节点并首先解析它，使Join条件得到解析，self - Join解析，
    // etc.
    val joined = sparkSession.sessionState.executePlan(
      Join(
        this.logicalPlan,
        other.logicalPlan,
        JoinType(joinType),
        Some(condition.expr))).analyzed.asInstanceOf[Join]

    // For both join side, combine all outputs into a single column and alias it with "_1" or "_2",
    // to match the schema for the encoder of the join result.
    // 对于这两个连接，将所有输出合并为一个列，并将其别名为“_1”或“_2”，以匹配连接结果的编码器的模式。

    // Note that we do this before joining them, to enable the join operator to return null for one
    // side, in cases like outer-join.
    // 请注意，在join它们之前，我们这样做，使join操作符在像outer - join这样的情况下返回null。
    val left = {
      val combined = if (this.exprEnc.flat) {
        assert(joined.left.output.length == 1)
        Alias(joined.left.output.head, "_1")()
      } else {
        Alias(CreateStruct(joined.left.output), "_1")()
      }
      Project(combined :: Nil, joined.left)
    }

    val right = {
      val combined = if (other.exprEnc.flat) {
        assert(joined.right.output.length == 1)
        Alias(joined.right.output.head, "_2")()
      } else {
        Alias(CreateStruct(joined.right.output), "_2")()
      }
      Project(combined :: Nil, joined.right)
    }

    // Rewrites the join condition to make the attribute point to correct column/field, after we
    // combine the outputs of each join side.
    // 在将每个连接的输出组合在一起之后,重写联接条件，使属性指向正确的列/字段。

    val conditionExpr = joined.condition.get transformUp {
      case a: Attribute if joined.left.outputSet.contains(a) =>
        if (this.exprEnc.flat) {
          left.output.head
        } else {
          val index = joined.left.output.indexWhere(_.exprId == a.exprId)
          GetStructField(left.output.head, index)
        }
      case a: Attribute if joined.right.outputSet.contains(a) =>
        if (other.exprEnc.flat) {
          right.output.head
        } else {
          val index = joined.right.output.indexWhere(_.exprId == a.exprId)
          GetStructField(right.output.head, index)
        }
    }

    implicit val tuple2Encoder: Encoder[(T, U)] =
      ExpressionEncoder.tuple(this.exprEnc, other.exprEnc)

    withTypedPlan(Join(left, right, joined.joinType, Some(conditionExpr)))
  }

  /**
    * :: Experimental ::
    * Using inner equi-join to join this Dataset returning a `Tuple2` for each pair
    * where `condition` evaluates to true.
    *
    * 使用内部的等连接加入这个数据集，为每一对返回一个“Tuple2”，其中“条件”的计算结果为true。
    *
    * @param other     Right side of the join.
    * @param condition Join expression.
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def joinWith[U](other: Dataset[U], condition: Column): Dataset[(T, U)] = {
    joinWith(other, condition, "inner")
  }
```
### sortWithinPartitions
```scala
  /**
    * Returns a new Dataset with each partition sorted by the given expressions.
    *
    * 返回一个新的数据集，每个分区按照给定的表达式排序。
    *
    * This is the same operation as "SORT BY" in SQL (Hive QL).
    *
    * 这与SQL(Hive QL)中“SORT BY”的操作相同。
    *
    * @group typedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def sortWithinPartitions(sortCol: String, sortCols: String*): Dataset[T] = {
    sortWithinPartitions((sortCol +: sortCols).map(Column(_)): _*)
  }

  /**
    * Returns a new Dataset with each partition sorted by the given expressions.
    *
    * 返回一个新的数据集，每个分区按照给定的表达式排序。
    *
    * This is the same operation as "SORT BY" in SQL (Hive QL).
    *
    * 这与SQL(Hive QL)中“SORT BY”的操作相同。
    *
    * @group typedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def sortWithinPartitions(sortExprs: Column*): Dataset[T] = {
    sortInternal(global = false, sortExprs)
  }
```
### sort
```scala
/**
    * Returns a new Dataset sorted by the specified column, all in ascending order.
    * 排序 升序
    * {{{
    *   // The following 3 are equivalent
    *   下面3个是等价的
    *   ds.sort("sortcol")
    *   ds.sort($"sortcol")
    *   ds.sort($"sortcol".asc)
    * }}}
    *
    * @group typedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def sort(sortCol: String, sortCols: String*): Dataset[T] = {
    sort((sortCol +: sortCols).map(apply): _*)
  }

  /**
    * Returns a new Dataset sorted by the given expressions. For example:
    *
    * 返回一个由给定表达式排序的新数据集。例如:
    *
    * {{{
    *   ds.sort($"col1", $"col2".desc)
    * }}}
    *
    * @group typedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def sort(sortExprs: Column*): Dataset[T] = {
    sortInternal(global = true, sortExprs)
  }
```
### orderBy
```scala
 /**
    * Returns a new Dataset sorted by the given expressions.
    * This is an alias of the `sort` function.
    * 这是“sort”函数的别名。
    *
    * @group typedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def orderBy(sortCol: String, sortCols: String*): Dataset[T] = sort(sortCol, sortCols: _*)

  /**
    * Returns a new Dataset sorted by the given expressions.
    * This is an alias of the `sort` function.
    * 这是“sort”函数的别名。
    *
    * @group typedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def orderBy(sortExprs: Column*): Dataset[T] = sort(sortExprs: _*)
```
### as
```scala
/**
    * Returns a new Dataset with an alias set.
    *
    * 返回一个具有别名集的新数据集。
    *
    * @group typedrel
    * @since 1.6.0
    */
  def as(alias: String): Dataset[T] = withTypedPlan {
    SubqueryAlias(alias, logicalPlan, None)
  }

  /**
    * (Scala-specific) Returns a new Dataset with an alias set.
    *
    * @group typedrel
    * @since 2.0.0
    */
  def as(alias: Symbol): Dataset[T] = as(alias.name)
```
### alias
```scala
/**
    * Returns a new Dataset with an alias set. Same as `as`.
    * 返回一个具有别名集的新数据集。与“as”相同。
    *
    * @group typedrel
    * @since 2.0.0
    */
  def alias(alias: String): Dataset[T] = as(alias)

  /**
    * (Scala-specific) Returns a new Dataset with an alias set. Same as `as`.
    *
    * @group typedrel
    * @since 2.0.0
    */
  def alias(alias: Symbol): Dataset[T] = as(alias)
```
### select
```scala
/**
    * :: Experimental ::
    * Returns a new Dataset by computing the given [[Column]] expression for each element.
    *
    * 通过计算每个元素的给定[[列]]表达式返回一个新的数据集。
    *
    * {{{
    *   val ds = Seq(1, 2, 3).toDS()
    *   val newDS = ds.select(expr("value + 1").as[Int])
    * }}}
    *
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def select[U1](c1: TypedColumn[T, U1]): Dataset[U1] = {
    implicit val encoder = c1.encoder
    val project = Project(c1.withInputType(exprEnc, logicalPlan.output).named :: Nil,
      logicalPlan)

    if (encoder.flat) {
      new Dataset[U1](sparkSession, project, encoder)
    } else {
      // Flattens inner fields of U1
      // 使U1的内部区域变平
      new Dataset[Tuple1[U1]](sparkSession, project, ExpressionEncoder.tuple(encoder)).map(_._1)
    }
  }
  
  /**
    * :: Experimental ::
    * Returns a new Dataset by computing the given [[Column]] expressions for each element.
    *
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def select[U1, U2](c1: TypedColumn[T, U1], c2: TypedColumn[T, U2]): Dataset[(U1, U2)] =
    selectUntyped(c1, c2).asInstanceOf[Dataset[(U1, U2)]]

  /**
    * :: Experimental ::
    * Returns a new Dataset by computing the given [[Column]] expressions for each element.
    *
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def select[U1, U2, U3](
                          c1: TypedColumn[T, U1],
                          c2: TypedColumn[T, U2],
                          c3: TypedColumn[T, U3]): Dataset[(U1, U2, U3)] =
    selectUntyped(c1, c2, c3).asInstanceOf[Dataset[(U1, U2, U3)]]

  /**
    * :: Experimental ::
    * Returns a new Dataset by computing the given [[Column]] expressions for each element.
    *
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def select[U1, U2, U3, U4](
                              c1: TypedColumn[T, U1],
                              c2: TypedColumn[T, U2],
                              c3: TypedColumn[T, U3],
                              c4: TypedColumn[T, U4]): Dataset[(U1, U2, U3, U4)] =
    selectUntyped(c1, c2, c3, c4).asInstanceOf[Dataset[(U1, U2, U3, U4)]]

  /**
    * :: Experimental ::
    * Returns a new Dataset by computing the given [[Column]] expressions for each element.
    *
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def select[U1, U2, U3, U4, U5](
                                  c1: TypedColumn[T, U1],
                                  c2: TypedColumn[T, U2],
                                  c3: TypedColumn[T, U3],
                                  c4: TypedColumn[T, U4],
                                  c5: TypedColumn[T, U5]): Dataset[(U1, U2, U3, U4, U5)] =
    selectUntyped(c1, c2, c3, c4, c5).asInstanceOf[Dataset[(U1, U2, U3, U4, U5)]]
```
### filter
```scala
/**
    * Filters rows using the given condition.
    *
    * 用给定的条件过滤rows
    *
    * {{{
    *   // The following are equivalent:
    *   以下是等价的：
    *   peopleDs.filter($"age" > 15)
    *   peopleDs.where($"age" > 15)
    * }}}
    *
    * @group typedrel
    * @since 1.6.0
    */
  def filter(condition: Column): Dataset[T] = withTypedPlan {
    Filter(condition.expr, logicalPlan)
  }

  /**
    * Filters rows using the given SQL expression.
    *
    * 用给定的 SQL 表达式 过滤rows
    *
    * {{{
    *   peopleDs.filter("age > 15")
    * }}}
    *
    * @group typedrel
    * @since 1.6.0
    */
  def filter(conditionExpr: String): Dataset[T] = {
    filter(Column(sparkSession.sessionState.sqlParser.parseExpression(conditionExpr)))
  }
```
### where
```scala
 /**
    * Filters rows using the given condition. This is an alias for `filter`.
    *
    * 使用给定条件过滤行。
    * 这是“filter”的别名。
    *
    * {{{
    *   // The following are equivalent:
    *   peopleDs.filter($"age" > 15)
    *   peopleDs.where($"age" > 15)
    * }}}
    *
    * @group typedrel
    * @since 1.6.0
    */
  def where(condition: Column): Dataset[T] = filter(condition)

  /**
    * Filters rows using the given SQL expression.
    *
    * 使用给定的 SQL 表达式  过滤 rows
    *
    * {{{
    *   peopleDs.where("age > 15")
    * }}}
    *
    * @group typedrel
    * @since 1.6.0
    */
  def where(conditionExpr: String): Dataset[T] = {
    filter(Column(sparkSession.sessionState.sqlParser.parseExpression(conditionExpr)))
  }
```
### groupByKey
```scala
  /**
    * :: Experimental ::
    * (Scala-specific)
    * Returns a [[KeyValueGroupedDataset]] where the data is grouped by the given key `func`.
    * 返回一个[[KeyValueGroupedDataset]]，数据由给定键' func '分组。
    *
    * @group typedrel
    * @since 2.0.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def groupByKey[K: Encoder](func: T => K): KeyValueGroupedDataset[K, T] = {
    val inputPlan = logicalPlan
    val withGroupingKey = AppendColumns(func, inputPlan)
    val executed = sparkSession.sessionState.executePlan(withGroupingKey)

    new KeyValueGroupedDataset(
      encoderFor[K],
      encoderFor[T],
      executed,
      inputPlan.output,
      withGroupingKey.newColumns)
  }

  /**
    * :: Experimental ::
    * (Java-specific)
    * Returns a [[KeyValueGroupedDataset]] where the data is grouped by the given key `func`.
    * 返回一个[[KeyValueGroupedDataset]]，数据由给定键' func '分组。
    *
    * @group typedrel
    * @since 2.0.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def groupByKey[K](func: MapFunction[T, K], encoder: Encoder[K]): KeyValueGroupedDataset[K, T] =
    groupByKey(func.call(_))(encoder)
```
### limit
```scala
  /**
    * Returns a new Dataset by taking the first `n` rows. The difference between this function
    * and `head` is that `head` is an action and returns an array (by triggering query execution)
    * while `limit` returns a new Dataset.
    *
    * 通过使用第一个“n”行返回一个新的数据集。
    * 这个函数和“head”的区别在于“head”是一个动作，
    * 并返回一个数组(通过触发查询执行)，而“limit”则返回一个新的数据集。
    *
    * @group typedrel
    * @since 2.0.0
    */
  def limit(n: Int): Dataset[T] = withTypedPlan {
    Limit(Literal(n), logicalPlan)
  }
```
### unionAll-已过时
```scala
  /**
    * Returns a new Dataset containing union of rows in this Dataset and another Dataset.
    * This is equivalent to `UNION ALL` in SQL.
    *
    * 返回一个新的数据集，该数据集包含该数据集中的行和另一个数据集。
    * 这相当于SQL中的“UNION ALL”。
    *
    * To do a SQL-style set union (that does deduplication of elements), use this function followed
    * by a [[distinct]].
    *
    * 如果需要去重的话，在该方法后继续直接  [[distinct]]
    *
    * @group typedrel
    * @since 2.0.0 已经过时
    */
  @deprecated("use union()", "2.0.0")
  def unionAll(other: Dataset[T]): Dataset[T] = union(other)
```
### union
```scala
  /**
    * Returns a new Dataset containing union of rows in this Dataset and another Dataset.
    * This is equivalent to `UNION ALL` in SQL.
    *
    * 返回一个新的数据集，该数据集包含该数据集中的行和另一个数据集。
    * 这相当于SQL中的“UNION ALL”。
    *
    * To do a SQL-style set union (that does deduplication of elements), use this function followed
    * by a [[distinct]].
    *
    * 如果需要去重的话，在该方法后继续直接  [[distinct]]
    *
    * @group typedrel
    * @since 2.0.0
    */
  def union(other: Dataset[T]): Dataset[T] = withSetOperator {
    // This breaks caching, but it's usually ok because it addresses a very specific use case:
    // using union to union many files or partitions.
    // 这打破了缓存，但通常是可以的，因为它解决了一个非常具体的用例:使用union来联合许多文件或分区。
    CombineUnions(Union(logicalPlan, other.logicalPlan))
  }
```
### intersect-交集
```scala
  /**
    * Returns a new Dataset containing rows only in both this Dataset and another Dataset.
    * This is equivalent to `INTERSECT` in SQL.
    *
    * 返回一个新的数据集，只包含该数据集和另一个数据集相同的行.
    * 这相当于在SQL中“INTERSECT”。
    * 会去重.
    *
    * @note Equality checking is performed directly on the encoded representation of the data
    *       and thus is not affected by a custom `equals` function defined on `T`.
    *
    *       等式检查直接执行数据的编码表示，因此不受定义为“T”的自定义“equals”函数的影响。
    * @group typedrel
    * @since 1.6.0
    */
  def intersect(other: Dataset[T]): Dataset[T] = withSetOperator {
    Intersect(logicalPlan, other.logicalPlan)
  }
```
### except-只显示另个Dataset中没有的值
```scala
  /**
    * Returns a new Dataset containing rows in this Dataset but not in another Dataset.
    * This is equivalent to `EXCEPT` in SQL.
    *
    * 返回一个新的数据集，该数据集包含该数据集中的行，而不是在另一个数据集。
    * 这等价于SQL中的“EXCEPT”。
    * 会去重.
    *
    * @note Equality checking is performed directly on the encoded representation of the data
    *       and thus is not affected by a custom `equals` function defined on `T`.
    * @group typedrel
    * @since 2.0.0
    */
  def except(other: Dataset[T]): Dataset[T] = withSetOperator {
    Except(logicalPlan, other.logicalPlan)
  }
```
### sample-随机抽样
```scala
  /**
    * Returns a new [[Dataset]] by sampling a fraction of rows, using a user-supplied seed.
    *
    * 通过使用用户提供的种子，通过抽样的方式返回一个新的[[Dataset]]。
    *
    * @param withReplacement Sample with replacement or not.
    *                        样本已经取过的值是否放回
    * @param fraction        Fraction of rows to generate.
    *                        每一行数据被取样的概率
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
### randomSplit-按照权重分割
```scala
/**
    * Randomly splits this Dataset with the provided weights.
    *
    * 随机将此数据集按照所提供的权重进行分割。
    *
    * @param weights weights for splits, will be normalized if they don't sum to 1.
    *                切分的权重。如果和不为1就会被标准化。
    * @param seed    Seed for sampling.
    *                取样的种子（影响随机数生成器）
    *
    *                For Java API, use [[randomSplitAsList]].
    *                Java API 使用 [[randomSplitAsList]].
    * @group typedrel
    * @since 2.0.0
    */
  def randomSplit(weights: Array[Double], seed: Long): Array[Dataset[T]] = {
    require(weights.forall(_ >= 0),
      s"Weights must be nonnegative, but got ${weights.mkString("[", ",", "]")}")
    require(weights.sum > 0,
      s"Sum of weights must be positive, but got ${weights.mkString("[", ",", "]")}")

    // It is possible that the underlying dataframe doesn't guarantee the ordering of rows in its
    // constituent partitions each time a split is materialized which could result in
    // overlapping splits. To prevent this, we explicitly sort each input partition to make the
    // ordering deterministic.
    // MapType cannot be sorted.
    val sorted = Sort(logicalPlan.output.filterNot(_.dataType.isInstanceOf[MapType])
      .map(SortOrder(_, Ascending)), global = false, logicalPlan)
    val sum = weights.sum
    // scanLeft 从右到右依次累计算 scanLeft(0.0d)(_+_): (0.0,(0.0+0.2),(0.0+0.2+0.8))
    val normalizedCumWeights = weights.map(_ / sum).scanLeft(0.0d)(_ + _)
    // sliding(n) 每次取n个值，以步长为1向右滑动，如：(0.0,0.2,0.8).sliding(2)=(0.0,0.2),(0.2,0.8)
    normalizedCumWeights.sliding(2).map { x =>
      new Dataset[T](
        sparkSession, Sample(x(0), x(1), withReplacement = false, seed, sorted)(), encoder)
    }.toArray
  }
  
    /**
    * Randomly splits this Dataset with the provided weights.
    *
    * 程序自动生成随机数种子，随机将此数据集按照所提供的权重进行分割。
    *
    * @param weights weights for splits, will be normalized if they don't sum to 1.
    *                切分的权重。如果和不为1就会被标准化。
    * @group typedrel
    * @since 2.0.0
    */
  def randomSplit(weights: Array[Double]): Array[Dataset[T]] = {
    randomSplit(weights, Utils.random.nextLong)
  }

  /**
    * Randomly splits this Dataset with the provided weights. Provided for the Python Api.
    * Python 使用该方法
    *
    * @param weights weights for splits, will be normalized if they don't sum to 1.
    * @param seed    Seed for sampling.
    */
  private[spark] def randomSplit(weights: List[Double], seed: Long): Array[Dataset[T]] = {
    randomSplit(weights.toArray, seed)
  }

```
### randomSplitAsList
```scala
  /**
    * Returns a Java list that contains randomly split Dataset with the provided weights.
    *
    * 根据提供的权重分割DataFrames，返回Java list
    *
    * @param weights weights for splits, will be normalized if they don't sum to 1.
    *                切分的权重。如果和不为1就会被标准化。
    * @param seed    Seed for sampling.
    *                取样的种子（影响随机数生成器）
    * @group typedrel
    * @since 2.0.0
    */
  def randomSplitAsList(weights: Array[Double], seed: Long): java.util.List[Dataset[T]] = {
    val values = randomSplit(weights, seed)
    java.util.Arrays.asList(values: _*)
  }
```
### dropDuplicates-去重
```scala
  /**
    * Returns a new Dataset that contains only the unique rows from this Dataset.
    * This is an alias for `distinct`.
    *
    * 删除重复的row数据，是distinct的别名
    *
    * @group typedrel
    * @since 2.0.0
    */
  def dropDuplicates(): Dataset[T] = dropDuplicates(this.columns)

  /**
    * (Scala-specific) Returns a new Dataset with duplicate rows removed, considering only
    * the subset of columns.
    *
    * 只删除指定列的重复数据
    *
    * @group typedrel
    * @since 2.0.0
    */
  def dropDuplicates(colNames: Seq[String]): Dataset[T] = withTypedPlan {
    val resolver = sparkSession.sessionState.analyzer.resolver
    val allColumns = queryExecution.analyzed.output
    val groupCols = colNames.flatMap { colName =>
      // It is possibly there are more than one columns with the same name,
      // so we call filter instead of find.
      val cols = allColumns.filter(col => resolver(col.name, colName))
      if (cols.isEmpty) {
        throw new AnalysisException(
          s"""Cannot resolve column name "$colName" among (${schema.fieldNames.mkString(", ")})""")
      }
      cols
    }
    val groupColExprIds = groupCols.map(_.exprId)
    val aggCols = logicalPlan.output.map { attr =>
      if (groupColExprIds.contains(attr.exprId)) {
        attr
      } else {
        // Removing duplicate rows should not change output attributes. We should keep
        // the original exprId of the attribute. Otherwise, to select a column in original
        // dataset will cause analysis exception due to unresolved attribute.
        // 删除重复行不应该更改输出属性。
        // 我们应该保留这个属性的原始属性。
        // 否则，在原始数据集中选择一个列将导致分析异常，原因是未解析的属性。
        Alias(new First(attr).toAggregateExpression(), attr.name)(exprId = attr.exprId)
      }
    }
    Aggregate(groupCols, aggCols, logicalPlan)
  }

  /**
    * Returns a new Dataset with duplicate rows removed, considering only
    * the subset of columns.
    *
    * 只针对特定列做去重
    *
    * @group typedrel
    * @since 2.0.0
    */
  def dropDuplicates(colNames: Array[String]): Dataset[T] = dropDuplicates(colNames.toSeq)

  /**
    * Returns a new [[Dataset]] with duplicate rows removed, considering only
    * the subset of columns.
    *
    * 只针对特定多列做去重
    *
    * @group typedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def dropDuplicates(col1: String, cols: String*): Dataset[T] = {
    val colNames: Seq[String] = col1 +: cols
    dropDuplicates(colNames)
  }
```
### transform-自定义转换
```scala
  /**
    * Concise syntax for chaining custom transformations.
    *
    * 用于链接自定义转换的简明语法。
    *
    * {{{
    *   def featurize(ds: Dataset[T]): Dataset[U] = ...
    *
    *   ds
    *     .transform(featurize)
    *     .transform(...)
    * }}}
    *
    * @group typedrel
    * @since 1.6.0
    */
  def transform[U](t: Dataset[T] => Dataset[U]): Dataset[U] = t(this)
```
### filter-过滤
```scala
/**
    * :: Experimental ::
    * (Scala-specific)
    * Returns a new Dataset that only contains elements where `func` returns `true`.
    *
    * 该数据集只包含“func”返回“true”的元素。
    *
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def filter(func: T => Boolean): Dataset[T] = {
    withTypedPlan(TypedFilter(func, logicalPlan))
  }

  /**
    * :: Experimental ::
    * (Java-specific)
    * Returns a new Dataset that only contains elements where `func` returns `true`.
    *
    * 返回一个新数据集，该数据集只包含“func”返回“true”的元素。
    *
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def filter(func: FilterFunction[T]): Dataset[T] = {
    withTypedPlan(TypedFilter(func, logicalPlan))
  }
```
### map
```scala
  /**
    * :: Experimental ::
    * (Scala-specific)
    * Returns a new Dataset that contains the result of applying `func` to each element.
    *
    * 返回一个新的数据集，该数据集包含对每个元素应用“func”的结果。
    *
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def map[U: Encoder](func: T => U): Dataset[U] = withTypedPlan {
    MapElements[T, U](func, logicalPlan)
  }

  /**
    * :: Experimental ::
    * (Java-specific)
    * Returns a new Dataset that contains the result of applying `func` to each element.
    *
    * 返回一个新的数据集，该数据集包含对每个元素应用“func”的结果。
    *
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def map[U](func: MapFunction[T, U], encoder: Encoder[U]): Dataset[U] = {
    implicit val uEnc = encoder
    withTypedPlan(MapElements[T, U](func, logicalPlan))
  }
```
### mapPartitions
```scala
  /**
    * :: Experimental ::
    * (Scala-specific)
    * Returns a new Dataset that contains the result of applying `func` to each partition.
    *
    * 返回一个新的数据集，该数据集包含对每个分区应用“func”的结果。
    *
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def mapPartitions[U: Encoder](func: Iterator[T] => Iterator[U]): Dataset[U] = {
    new Dataset[U](
      sparkSession,
      MapPartitions[T, U](func, logicalPlan),
      implicitly[Encoder[U]])
  }

  /**
    * :: Experimental ::
    * (Java-specific)
    * Returns a new Dataset that contains the result of applying `f` to each partition.
    *
    * 返回一个新的数据集，该数据集包含对每个分区应用“f”的结果。
    *
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def mapPartitions[U](f: MapPartitionsFunction[T, U], encoder: Encoder[U]): Dataset[U] = {
    val func: (Iterator[T]) => Iterator[U] = x => f.call(x.asJava).asScala
    mapPartitions(func)(encoder)
  }
```
### flatMap-将map结果flat扁平化
```scala
  /**
    * :: Experimental ::
    * (Scala-specific)
    * Returns a new Dataset by first applying a function to all elements of this Dataset,
    * and then flattening the results.
    *
    * 返回一个新的数据集，首先对该数据集的所有元素应用一个函数，然后将结果扁平化。
    *
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def flatMap[U: Encoder](func: T => TraversableOnce[U]): Dataset[U] =
    mapPartitions(_.flatMap(func))

  /**
    * :: Experimental ::
    * (Java-specific)
    * Returns a new Dataset by first applying a function to all elements of this Dataset,
    * and then flattening the results.
    *
    * 返回一个新的数据集，首先对该数据集的所有元素应用一个函数，然后将结果扁平化。
    *
    * @group typedrel
    * @since 1.6.0
    */
  @Experimental
  @InterfaceStability.Evolving
  def flatMap[U](f: FlatMapFunction[T, U], encoder: Encoder[U]): Dataset[U] = {
    val func: (T) => Iterator[U] = x => f.call(x).asScala
    flatMap(func)(encoder)
  }
```
### repartition-重分区
```scala
/**
    * Returns a new Dataset that has exactly `numPartitions` partitions.
    *
    * 返回一个 给定分区数量的新DataSet
    *
    * @group typedrel
    * @since 1.6.0
    */
  def repartition(numPartitions: Int): Dataset[T] = withTypedPlan {
    Repartition(numPartitions, shuffle = true, logicalPlan)
  }

  /**
    * Returns a new Dataset partitioned by the given partitioning expressions into
    * `numPartitions`. The resulting Dataset is hash partitioned.
    *
    * 返回一个由给定的分区表达式划分为“num分区”的新数据集。
    * 生成的Dataset是哈希分区的。
    *
    * This is the same operation as "DISTRIBUTE BY" in SQL (Hive QL).
    *
    * 和 SQL (Hive QL) 中的 "DISTRIBUTE BY" 作用相同
    *
    * @group typedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def repartition(numPartitions: Int, partitionExprs: Column*): Dataset[T] = withTypedPlan {
    RepartitionByExpression(partitionExprs.map(_.expr), logicalPlan, Some(numPartitions))
  }

  /**
    * Returns a new Dataset partitioned by the given partitioning expressions, using
    * `spark.sql.shuffle.partitions` as number of partitions.
    * The resulting Dataset is hash partitioned.
    *
    * 根据指定的分区表达式进行重分区。
    * 分区数量由`spark.sql.shuffle.partitions` 获得。
    * 结果Dataset 是哈希分区的。
    *
    * This is the same operation as "DISTRIBUTE BY" in SQL (Hive QL).
    *
    * 和 SQL (Hive QL) 中的 "DISTRIBUTE BY" 作用相同
    *
    * @group typedrel
    * @since 2.0.0
    */
  @scala.annotation.varargs
  def repartition(partitionExprs: Column*): Dataset[T] = withTypedPlan {
    RepartitionByExpression(partitionExprs.map(_.expr), logicalPlan, numPartitions = None)
  }
```
### coalesce-合并分区
```scala
/**
    * Returns a new Dataset that has exactly `numPartitions` partitions.
    * Similar to coalesce defined on an `RDD`, this operation results in a narrow dependency, e.g.
    * if you go from 1000 partitions to 100 partitions, there will not be a shuffle, instead each of
    * the 100 new partitions will claim 10 of the current partitions.
    *
    * 合并。
    * 返回确定分区数量的Dataset。
    * 和RDD中的合并方法类似，这个操作导致了一个窄依赖。
    * 例如：将1000个分区合并为100个分区，这个过程没有shuffle，而是100个新分区中的每个分区将声明当前的10个分区。
    *
    * @group typedrel
    * @since 1.6.0
    */
  def coalesce(numPartitions: Int): Dataset[T] = withTypedPlan {
    Repartition(numPartitions, shuffle = false, logicalPlan)
  }
```
### distinct-去重
```scala
/**
    * Returns a new Dataset that contains only the unique rows from this Dataset.
    * This is an alias for `dropDuplicates`.
    *
    * 去重。
    * 返回去重后的Dataset。
    * 和 `dropDuplicates` 方法一致。
    *
    * @note Equality checking is performed directly on the encoded representation of the data
    *       and thus is not affected by a custom `equals` function defined on `T`.
    * @group typedrel
    * @since 2.0.0
    */
  def distinct(): Dataset[T] = dropDuplicates()
```

<!--对不起，到时间了，请停止装逼-->


