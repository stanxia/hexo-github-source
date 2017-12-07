---
title: JavaRDDLike.scala
date: 2017-11-21 14:21:23
tags: 
- spark
- 源码
categories: spark
---
{% note info %}使用Java开发Spark程序，JavaRDD的功能算子中英文注释
JavaRDDLike的实现应该扩展这个虚拟抽象类，而不是直接继承这个特性。{% endnote %}

<!--请开始装逼-->
## JavaRDD
``` scala
package org.apache.spark.api.java

private[spark] abstract class AbstractJavaRDDLike[T, This <: JavaRDDLike[T, This]]
  extends JavaRDDLike[T, This]

/**
  * Defines operations common to several Java RDD implementations.
  *
  * 定义几个Java RDD实现的常见操作。
  *
  * @note This trait is not intended to be implemented by user code.
  *
  *       该特性不打算由用户代码实现。
  */
trait JavaRDDLike[T, This <: JavaRDDLike[T, This]] extends Serializable {
  def wrapRDD(rdd: RDD[T]): This

  implicit val classTag: ClassTag[T]

  def rdd: RDD[T]
```

<!-- more -->

### partitions

```scala
  /** Set of partitions in this RDD.
    * 在这个RDD中设置的分区。
    * */
  def partitions: JList[Partition] = rdd.partitions.toSeq.asJava

```
### getNumPartitions
```scala
  /** Return the number of partitions in this RDD.
    * 返回该RDD中的分区数。
    * */
  @Since("1.6.0")
  def getNumPartitions: Int = rdd.getNumPartitions
```
### partitioner
```scala
  /** The partitioner of this RDD.
    * 这个RDD的分区。
    * */
  def partitioner: Optional[Partitioner] = JavaUtils.optionToOptional(rdd.partitioner)
```
### context
```scala
  /** The [[org.apache.spark.SparkContext]] that this RDD was created on.
    *
    * 这个RDD是在[[org.apache.spark.SparkContext]]上面创建的。
    * */
  def context: SparkContext = rdd.context
```
### id
```scala
  /** A unique ID for this RDD (within its SparkContext).
    * 这个RDD的惟一ID(在它的SparkContext内)。
    * */
  def id: Int = rdd.id
```
### name
```scala
  def name(): String = rdd.name
```
### getStorageLevel
```scala
  /** Get the RDD's current storage level, or StorageLevel.NONE if none is set.
    * 获取RDD的当前存储级别，或StorageLevel。如果没有设置就没有。
    * */
  def getStorageLevel: StorageLevel = rdd.getStorageLevel
```
### iterator
```scala
    /**
      * Internal method to this RDD; will read from cache if applicable, or otherwise compute it.
      * This should ''not'' be called by users directly, but is available for implementors of custom
      * subclasses of RDD.
      * 内部方法的RDD;将从缓存读取，如果适用的话，或者计算它。
      * 这应该“不是”直接由用户调用，而是用于RDD的自定义子类的实现者
      *
      */
    def iterator(split: Partition, taskContext: TaskContext): JIterator[T] =
      rdd.iterator(split, taskContext).asJavs
```
## Transformations (return a new RDD)
### map
```scala
    /**
      * Return a new RDD by applying a function to all elements of this RDD.
      * 将一个函数应用于这个RDD的所有元素，返回一个新的RDD。
      *
      */
    def map[R](f: JFunction[T, R]): JavaRDD[R] =
      new JavaRDD(rdd.map(f)(fakeClassTag))(fakeClassTag)
```
### mapPartitionsWithIndex
```scala
    /**
      * Return a new RDD by applying a function to each partition of this RDD, while tracking the index
      * of the original partition.
      * 通过在RDD的每个分区上应用一个函数来返回一个新的RDD，同时跟踪原始分区的索引。
      *
      */
    def mapPartitionsWithIndex[R](
                                   f: JFunction2[jl.Integer, JIterator[T], JIterator[R]],
                                   preservesPartitioning: Boolean = false): JavaRDD[R] =
      new JavaRDD(rdd.mapPartitionsWithIndex((a, b) => f.call(a, b.asJava).asScala,
        preservesPartitioning)(fakeClassTag))(fakeClassTag)
```
### mapToDouble
```scala
    /**
      * Return a new RDD by applying a function to all elements of this RDD.
      * 将一个函数应用于这个RDD的所有元素，返回一个新的RDD。
      */
    def mapToDouble[R](f: DoubleFunction[T]): JavaDoubleRDD = {
      new JavaDoubleRDD(rdd.map(f.call(_).doubleValue()))
    }
```
### mapToPair
```scala
/**
  * Return a new RDD by applying a function to all elements of this RDD.
  * 将一个函数应用于这个RDD的所有元素，返回一个新的RDD。
  *
  */
def mapToPair[K2, V2](f: PairFunction[T, K2, V2]): JavaPairRDD[K2, V2] = {
  def cm: ClassTag[(K2, V2)] = implicitly[ClassTag[(K2, V2)]]
  new JavaPairRDD(rdd.map[(K2, V2)](f)(cm))(fakeClassTag[K2], fakeClassTag[V2])
}
```
### flatMap
```scala
/**
  *  Return a new RDD by first applying a function to all elements of this
  *  RDD, and then flattening the results.
  *  返回一个新的RDD，首先将一个函数应用于这个RDD的所有元素，然后将结果扁平化。
  *
  */
def flatMap[U](f: FlatMapFunction[T, U]): JavaRDD[U] = {
  def fn: (T) => Iterator[U] = (x: T) => f.call(x).asScala
  JavaRDD.fromRDD(rdd.flatMap(fn)(fakeClassTag[U]))(fakeClassTag[U])
}
```
### flatMapToDouble
```scala
/**
  *  Return a new RDD by first applying a function to all elements of this
  *  RDD, and then flattening the results.
  *  返回一个新的RDD，首先将一个函数应用于这个RDD的所有元素，然后将结果扁平化。
  *
  */
def flatMapToDouble(f: DoubleFlatMapFunction[T]): JavaDoubleRDD = {
  def fn: (T) => Iterator[jl.Double] = (x: T) => f.call(x).asScala
  new JavaDoubleRDD(rdd.flatMap(fn).map(_.doubleValue()))
}
```
### flatMapToPair
```scala
/**
  *  Return a new RDD by first applying a function to all elements of this
  *  RDD, and then flattening the results.
  *  返回一个新的RDD，首先将一个函数应用于这个RDD的所有元素，然后将结果扁平化。
  *
  */
def flatMapToPair[K2, V2](f: PairFlatMapFunction[T, K2, V2]): JavaPairRDD[K2, V2] = {
  def fn: (T) => Iterator[(K2, V2)] = (x: T) => f.call(x).asScala
  def cm: ClassTag[(K2, V2)] = implicitly[ClassTag[(K2, V2)]]
  JavaPairRDD.fromRDD(rdd.flatMap(fn)(cm))(fakeClassTag[K2], fakeClassTag[V2])
}
```
### mapPartitions
```scala
/**
  * Return a new RDD by applying a function to each partition of this RDD.
  * 通过将一个函数应用于这个RDD的每个分区，返回一个新的RDD。
  *
  */
def mapPartitions[U](f: FlatMapFunction[JIterator[T], U]): JavaRDD[U] = {
  def fn: (Iterator[T]) => Iterator[U] = {
    (x: Iterator[T]) => f.call(x.asJava).asScala
  }
  JavaRDD.fromRDD(rdd.mapPartitions(fn)(fakeClassTag[U]))(fakeClassTag[U])
}

/**
  * Return a new RDD by applying a function to each partition of this RDD.
  * 通过将一个函数应用于这个RDD的每个分区，返回一个新的RDD。
  *
  */
def mapPartitions[U](f: FlatMapFunction[JIterator[T], U],
                     preservesPartitioning: Boolean): JavaRDD[U] = {
  def fn: (Iterator[T]) => Iterator[U] = {
    (x: Iterator[T]) => f.call(x.asJava).asScala
  }
  JavaRDD.fromRDD(
    rdd.mapPartitions(fn, preservesPartitioning)(fakeClassTag[U]))(fakeClassTag[U])
}
```
### mapPartitionsToDouble
```scala
/**
  * Return a new RDD by applying a function to each partition of this RDD.
  * 通过将一个函数应用于这个RDD的每个分区，返回一个新的RDD。
  *
  */
def mapPartitionsToDouble(f: DoubleFlatMapFunction[JIterator[T]]): JavaDoubleRDD = {
  def fn: (Iterator[T]) => Iterator[jl.Double] = {
    (x: Iterator[T]) => f.call(x.asJava).asScala
  }
  new JavaDoubleRDD(rdd.mapPartitions(fn).map(_.doubleValue()))
}

/**
  * Return a new RDD by applying a function to each partition of this RDD.
  * 通过将一个函数应用于这个RDD的每个分区，返回一个新的RDD。
  *
  */
def mapPartitionsToDouble(f: DoubleFlatMapFunction[JIterator[T]],
                          preservesPartitioning: Boolean): JavaDoubleRDD = {
  def fn: (Iterator[T]) => Iterator[jl.Double] = {
    (x: Iterator[T]) => f.call(x.asJava).asScala
  }
  new JavaDoubleRDD(rdd.mapPartitions(fn, preservesPartitioning)
    .map(_.doubleValue()))
}
```
### mapPartitionsToPair
```scala
/**
  * Return a new RDD by applying a function to each partition of this RDD.
  * 通过将一个函数应用于这个RDD的每个分区，返回一个新的RDD。
  *
  */
def mapPartitionsToPair[K2, V2](f: PairFlatMapFunction[JIterator[T], K2, V2]):
JavaPairRDD[K2, V2] = {
  def fn: (Iterator[T]) => Iterator[(K2, V2)] = {
    (x: Iterator[T]) => f.call(x.asJava).asScala
  }
  JavaPairRDD.fromRDD(rdd.mapPartitions(fn))(fakeClassTag[K2], fakeClassTag[V2])
}

/**
  * Return a new RDD by applying a function to each partition of this RDD.
  * 通过将一个函数应用于这个RDD的每个分区，返回一个新的RDD。
  *
  */
def mapPartitionsToPair[K2, V2](f: PairFlatMapFunction[JIterator[T], K2, V2],
                                preservesPartitioning: Boolean): JavaPairRDD[K2, V2] = {
  def fn: (Iterator[T]) => Iterator[(K2, V2)] = {
    (x: Iterator[T]) => f.call(x.asJava).asScala
  }
  JavaPairRDD.fromRDD(
    rdd.mapPartitions(fn, preservesPartitioning))(fakeClassTag[K2], fakeClassTag[V2])
}
```
### foreachPartition
```scala
/**
  * Applies a function f to each partition of this RDD.
  * 将函数f应用于该RDD的每个分区。
  *
  */
def foreachPartition(f: VoidFunction[JIterator[T]]): Unit = {
  rdd.foreachPartition(x => f.call(x.asJava))
}
```
### glom
```scala
/**
  * Return an RDD created by coalescing all elements within each partition into an array.
  * 返回一个RDD，它将每个分区中的所有元素合并到一个数组中。
  *
  */
def glom(): JavaRDD[JList[T]] =
  new JavaRDD(rdd.glom().map(_.toSeq.asJava))
```
### cartesian
```scala
/**
  * Return the Cartesian product of this RDD and another one, that is, the RDD of all pairs of
  * elements (a, b) where a is in `this` and b is in `other`.
  * 返回这个RDD和另一个的笛卡尔乘积，即所有元素对的RDD(a,b) ：a在该RDD中，b在另一个RDD中
  *
  */
def cartesian[U](other: JavaRDDLike[U, _]): JavaPairRDD[T, U] =
  JavaPairRDD.fromRDD(rdd.cartesian(other.rdd)(other.classTag))(classTag, other.classTag)
```
### groupBy
```scala
/**
  * Return an RDD of grouped elements. Each group consists of a key and a sequence of elements
  * mapping to that key.
  * 返回分组元素的RDD。
  * 每个组由一个键和一个映射到该键的元素序列组成。
  *
  */
def groupBy[U](f: JFunction[T, U]): JavaPairRDD[U, JIterable[T]] = {
  // The type parameter is U instead of K in order to work around a compiler bug; see SPARK-4459
  // 类型参数是U而不是K，是为了绕过编译器错误
  implicit val ctagK: ClassTag[U] = fakeClassTag
  implicit val ctagV: ClassTag[JList[T]] = fakeClassTag
  JavaPairRDD.fromRDD(groupByResultToJava(rdd.groupBy(f)(fakeClassTag)))
}

/**
  * Return an RDD of grouped elements. Each group consists of a key and a sequence of elements
  * mapping to that key.
  * 返回分组元素的RDD。
  * 每个组由一个键和一个映射到该键的元素序列组成。
  */
def groupBy[U](f: JFunction[T, U], numPartitions: Int): JavaPairRDD[U, JIterable[T]] = {
  // The type parameter is U instead of K in order to work around a compiler bug; see SPARK-4459
  implicit val ctagK: ClassTag[U] = fakeClassTag
  implicit val ctagV: ClassTag[JList[T]] = fakeClassTag
  JavaPairRDD.fromRDD(groupByResultToJava(rdd.groupBy(f, numPartitions)(fakeClassTag[U])))
}
```
### pipe
```scala
/**
  * Return an RDD created by piping elements to a forked external process.
  * 返回由管道元素调用外部程序返回新的RDD
  *
  */
def pipe(command: String): JavaRDD[String] = {
  rdd.pipe(command)
}

/**
  * Return an RDD created by piping elements to a forked external process.
  * 返回由管道元素调用外部程序返回新的RDD
  *
  */
def pipe(command: JList[String]): JavaRDD[String] = {
  rdd.pipe(command.asScala)
}

/**
  * Return an RDD created by piping elements to a forked external process.
  * 返回由管道元素调用外部程序返回新的RDD
  *
  */
def pipe(command: JList[String], env: JMap[String, String]): JavaRDD[String] = {
  rdd.pipe(command.asScala, env.asScala)
}

/**
  * Return an RDD created by piping elements to a forked external process.
  * 返回由管道元素调用外部程序返回新的RDD
  *
  */
def pipe(command: JList[String],
         env: JMap[String, String],
         separateWorkingDir: Boolean,
         bufferSize: Int): JavaRDD[String] = {
  rdd.pipe(command.asScala, env.asScala, null, null, separateWorkingDir, bufferSize)
}

/**
  * Return an RDD created by piping elements to a forked external process.
  * 返回由管道元素调用外部程序返回新的RDD
  *
  */
def pipe(command: JList[String],
         env: JMap[String, String],
         separateWorkingDir: Boolean,
         bufferSize: Int,
         encoding: String): JavaRDD[String] = {
  rdd.pipe(command.asScala, env.asScala, null, null, separateWorkingDir, bufferSize, encoding)
}
```
### zip
```scala
/**
  * Zips this RDD with another one, returning key-value pairs with the first element in each RDD,
  * second element in each RDD, etc. Assumes that the two RDDs have the *same number of
  * partitions* and the *same number of elements in each partition* (e.g. one was made through
  * a map on the other).
  * 将此RDD与另一个RDD进行Zips，返回键值对，每个RDD中的第一个元素，每个RDD中的第二个元素，等等。
  * 假设两个RDDs拥有相同数量的分区和每个分区中相同数量的元素
  * (例如，一个是通过另一个map的)。
  */
def zip[U](other: JavaRDDLike[U, _]): JavaPairRDD[T, U] = {
  JavaPairRDD.fromRDD(rdd.zip(other.rdd)(other.classTag))(classTag, other.classTag)
}
```
### zipPartitions
```scala
/**
  * Zip this RDD's partitions with one (or more) RDD(s) and return a new RDD by
  * applying a function to the zipped partitions. Assumes that all the RDDs have the
  * *same number of partitions*, but does *not* require them to have the same number
  * of elements in each partition.
  * 用一个(或多个)RDD(或多个)来压缩这个RDD的分区，并返回一个新的RDD将函数应用于压缩分区。
  * 假设所有RDDs拥有相同数量的分区，但不要求它们在每个分区中拥有相同数量的元素。
  */
def zipPartitions[U, V](
                         other: JavaRDDLike[U, _],
                         f: FlatMapFunction2[JIterator[T], JIterator[U], V]): JavaRDD[V] = {
  def fn: (Iterator[T], Iterator[U]) => Iterator[V] = {
    (x: Iterator[T], y: Iterator[U]) => f.call(x.asJava, y.asJava).asScala
  }
  JavaRDD.fromRDD(
    rdd.zipPartitions(other.rdd)(fn)(other.classTag, fakeClassTag[V]))(fakeClassTag[V])
}
```
### zipWithUniqueId
```scala
/**
  * Zips this RDD with generated unique Long ids. Items in the kth partition will get ids k, n+k,
  * 2*n+k, ..., where n is the number of partitions. So there may exist gaps, but this method
  * won't trigger a spark job, which is different from [[org.apache.spark.rdd.RDD#zipWithIndex]].
  * 用生成的唯一长的id来压缩这个RDD。
  * 第k个分区的项将得到id k,n + k,2 *n+ k，…，其中n是分区数。
  * 因此，可能存在差距，但这种方法不会触发spark作业，它与[org .apache.spark. spark.rdd. rdd. rdd # zipWithIndex]不同。
  */
def zipWithUniqueId(): JavaPairRDD[T, jl.Long] = {
  JavaPairRDD.fromRDD(rdd.zipWithUniqueId()).asInstanceOf[JavaPairRDD[T, jl.Long]]
}
```
### zipWithIndex
```scala
  /**
   * Zips this RDD with its element indices. The ordering is first based on the partition index
   * and then the ordering of items within each partition. So the first item in the first
   * partition gets index 0, and the last item in the last partition receives the largest index.
   * This is similar to Scala's zipWithIndex but it uses Long instead of Int as the index type.
   * This method needs to trigger a spark job when this RDD contains more than one partitions.
    *
    * 用它的元素索引来压缩这个RDD。
    * 排序首先基于分区索引，然后是每个分区中的条目的排序。
    * 因此，第一个分区中的第一个项的索引值为0，最后一个分区中的最后一个项得到最大的索引。
    * 这类似于Scala的zipWithIndex，但它使用的是Long而不是Int作为索引类型。
    * 当这个RDD包含多个分区时，这个方法需要触发一个spark作业。
   */
  def zipWithIndex(): JavaPairRDD[T, jl.Long] = {
    JavaPairRDD.fromRDD(rdd.zipWithIndex()).asInstanceOf[JavaPairRDD[T, jl.Long]]
  }
```
## Actions (launch a job to return a value to the user program)
### foreach
```scala
/**
  * Applies a function f to all elements of this RDD.
  * 将函数f应用于该RDD的所有元素。
  */
def foreach(f: VoidFunction[T]) {
  rdd.foreach(x => f.call(x))
}
```
### collect
```scala
/**
  * Return an array that contains all of the elements in this RDD.
  * 返回包含该RDD中所有元素的数组。
  *
  * @note this method should only be used if the resulting array is expected to be small, as
  * all the data is loaded into the driver's memory.
  * 该方法只在预期的数组很小的情况下使用，因为所有的数据都被加载到驱动程序的内存中。
  */
def collect(): JList[T] =
  rdd.collect().toSeq.asJava
```
### toLocalIterator
```scala
/**
  * Return an iterator that contains all of the elements in this RDD.
  * 返回包含该RDD中所有元素的迭代器。
  *
  * The iterator will consume as much memory as the largest partition in this RDD.
  * 迭代器将消耗与此RDD中最大的分区一样多的内存。
  */
def toLocalIterator(): JIterator[T] =
  asJavaIteratorConverter(rdd.toLocalIterator).asJava
```
### collectPartitions
```scala
 /**
  * Return an array that contains all of the elements in a specific partition of this RDD.
  * 返回包含该RDD的特定分区中的所有元素的数组。
  */
def collectPartitions(partitionIds: Array[Int]): Array[JList[T]] = {
  // This is useful for implementing `take` from other language frontends
  // like Python where the data is serialized.
  // 这有助于从其他语言的前沿实现“take”，如Python，数据被序列化。
  val res = context.runJob(rdd, (it: Iterator[T]) => it.toArray, partitionIds)
  res.map(_.toSeq.asJava)
}
```
### reduce
```scala
/**
  * Reduces the elements of this RDD using the specified commutative and associative binary
  * operator.
  * 使用指定的交换和关联二元运算符来减少该RDD的元素。
  */
def reduce(f: JFunction2[T, T, T]): T = rdd.reduce(f)
```
### treeReduce
```scala
/**
  * Reduces the elements of this RDD in a multi-level tree pattern.
  * 将此RDD的元素简化为多层树模式。
  *
  * @param depth suggested depth of the tree 建议树的深度
  * @see [[org.apache.spark.api.java.JavaRDDLike#reduce]]
  */
def treeReduce(f: JFunction2[T, T, T], depth: Int): T = rdd.treeReduce(f, depth)

/**
  * [[org.apache.spark.api.java.JavaRDDLike#treeReduce]] 建议深度 2 .
  */
def treeReduce(f: JFunction2[T, T, T]): T = treeReduce(f, 2)
```
### fold
```scala
 /**
  * Aggregate the elements of each partition, and then the results for all the partitions, using a
  * given associative function and a neutral "zero value". The function
  * op(t1, t2) is allowed to modify t1 and return it as its result value to avoid object
  * allocation; however, it should not modify t2.
  * 对每个分区的元素进行聚合，然后使用给定的关联函数和中立的“零值”，对所有分区进行结果。
  * 函数op(t1,t2)被允许修改t1，并将其作为其结果值返回，以避免对象分配;但是，它不应该修改t2。
  *
  * This behaves somewhat differently from fold operations implemented for non-distributed
  * collections in functional languages like Scala. This fold operation may be applied to
  * partitions individually, and then fold those results into the final result, rather than
  * apply the fold to each element sequentially in some defined ordering. For functions
  * that are not commutative, the result may differ from that of a fold applied to a
  * non-distributed collection.
  * 这与在Scala等函数式语言中实现非分布式集合的折叠操作有一定的不同。
  * 这个折叠操作可以单独应用于分区，然后将这些结果折叠到最终结果中，而不是在某些定义的排序中顺序地对每个元素进行折叠。
  * 对于非交换的函数，结果可能与应用于非分布式集合的函数不同。
  *
  */
def fold(zeroValue: T)(f: JFunction2[T, T, T]): T =
  rdd.fold(zeroValue)(f)
```
### aggregate
```scala
 /**
  * Aggregate the elements of each partition, and then the results for all the partitions, using
  * given combine functions and a neutral "zero value". This function can return a different result
  * type, U, than the type of this RDD, T. Thus, we need one operation for merging a T into an U
  * and one operation for merging two U's, as in scala.TraversableOnce. Both of these functions are
  * allowed to modify and return their first argument instead of creating a new U to avoid memory
  * allocation.
  * 对每个分区的元素进行聚合，然后使用给定的组合函数和一个中立的“零值”，对所有分区进行结果。
  * 这个函数可以返回一个不同的结果类型U，而不是这个RDD的类型。
  * 因此，我们需要一个操作来将一个T合并到一个U和一个合并两个U的操作，就像在scala . traversableonce中那样。
  * 这两个函数都可以修改和返回第一个参数，而不是创建一个新的U，以避免内存分配。
  *
  */
def aggregate[U](zeroValue: U)(seqOp: JFunction2[U, T, U],
                               combOp: JFunction2[U, U, U]): U =
  rdd.aggregate(zeroValue)(seqOp, combOp)(fakeClassTag[U])
```
### treeAggregate
```scala
  /**
  * Aggregates the elements of this RDD in a multi-level tree pattern.
  * 将此RDD的元素聚集在多层树模式中。
  *
  * @param depth suggested depth of the tree  建议的树的深度
  * @see [[org.apache.spark.api.java.JavaRDDLike#aggregate]]
  */
def treeAggregate[U](
                      zeroValue: U,
                      seqOp: JFunction2[U, T, U],
                      combOp: JFunction2[U, U, U],
                      depth: Int): U = {
  rdd.treeAggregate(zeroValue)(seqOp, combOp, depth)(fakeClassTag[U])
}

/**
  * [[org.apache.spark.api.java.JavaRDDLike#treeAggregate]] with suggested depth 2.
  * 建议的树的深度为 2
  */
def treeAggregate[U](
                      zeroValue: U,
                      seqOp: JFunction2[U, T, U],
                      combOp: JFunction2[U, U, U]): U = {
  treeAggregate(zeroValue, seqOp, combOp, 2)
}
```
### count
```scala
/**
  * Return the number of elements in the RDD.
  * 返回RDD中元素的数量。
  *
  */
def count(): Long = rdd.count()
```
### countApprox
```scala
  /**
  * Approximate version of count() that returns a potentially incomplete result
  * within a timeout, even if not all tasks have finished.
  * 近似版本的count()，即使不是所有的任务都完成了，也会在一个超时中返回一个潜在的不完整的结果。
  *
  *
  * The confidence is the probability that the error bounds of the result will
  * contain the true value. That is, if countApprox were called repeatedly
  * with confidence 0.9, we would expect 90% of the results to contain the
  * true count. The confidence must be in the range [0,1] or an exception will
  * be thrown.
  * 置信值是结果的误差边界包含真实值的概率。
  * 也就是说，如果countApprox被反复调用，confidence 0.9，我们将期望90%的结果包含真实的计数。
  * confidence必须在范围[0,1]中，否则将抛出异常。
  *
  *
  * @param timeout maximum time to wait for the job, in milliseconds
  *                等待工作的最大时间，以毫秒为单位
  * @param confidence the desired statistical confidence in the result
  *                   对结果的期望的统计信心
  * @return a potentially incomplete result, with error bounds
  *         一个可能不完整的结果，有错误界限
  */
def countApprox(timeout: Long, confidence: Double): PartialResult[BoundedDouble] =
  rdd.countApprox(timeout, confidence)

/**
  * Approximate version of count() that returns a potentially incomplete result
  * within a timeout, even if not all tasks have finished.
  * 近似版本的count()，即使不是所有的任务都完成了，也会在一个超时中返回一个潜在的不完整的结果。
  *
  *
  * @param timeout maximum time to wait for the job, in milliseconds
  *                等待工作的最大时间，以毫秒为单位
  */
def countApprox(timeout: Long): PartialResult[BoundedDouble] =
  rdd.countApprox(timeout)
```
### countByValue
```scala
/**
  * Return the count of each unique value in this RDD as a map of (value, count) pairs. The final
  * combine step happens locally on the master, equivalent to running a single reduce task.
  * 将此RDD中的每个惟一值的计数作为(值、计数)对的映射。
  * 最后的联合步骤在master的本地发生，相当于运行一个reduce任务。
  *
  */
def countByValue(): JMap[T, jl.Long] =
  mapAsSerializableJavaMap(rdd.countByValue()).asInstanceOf[JMap[T, jl.Long]]
```
### countByValueApprox
```scala
  /**
  * Approximate version of countByValue().
  * countByValue()近似的版本。
  *
  * The confidence is the probability that the error bounds of the result will
  * contain the true value. That is, if countApprox were called repeatedly
  * with confidence 0.9, we would expect 90% of the results to contain the
  * true count. The confidence must be in the range [0,1] or an exception will
  * be thrown.
  * 置信值是结果的误差边界包含真实值的概率。
  * 也就是说，如果countApprox被反复调用，confidence 0.9，我们将期望90%的结果包含真实的计数。
  * confidence必须在范围[0,1]中，否则将抛出异常。
  *
  *
  * @param timeout maximum time to wait for the job, in milliseconds
  *                等待工作的最大时间，毫秒为单位。
  * @param confidence the desired statistical confidence in the result
  *                   对结果的期望的统计信心
  * @return a potentially incomplete result, with error bounds
  *         一个可能不完整的结果，有错误界限
  */
def countByValueApprox(
                        timeout: Long,
                        confidence: Double
                      ): PartialResult[JMap[T, BoundedDouble]] =
  rdd.countByValueApprox(timeout, confidence).map(mapAsSerializableJavaMap)

/**
  * Approximate version of countByValue().
  * countByValue().的近似版本.
  *
  * @param timeout maximum time to wait for the job, in milliseconds
  *                等待工作的最大时间，毫秒为单位。
  * @return a potentially incomplete result, with error bounds
  *         一个可能不完整的结果，有错误界限
  */
def countByValueApprox(timeout: Long): PartialResult[JMap[T, BoundedDouble]] =
  rdd.countByValueApprox(timeout).map(mapAsSerializableJavaMap)
```
### take
```scala
  /**
  * Take the first num elements of the RDD. This currently scans the partitions *one by one*, so
  * it will be slow if a lot of partitions are required. In that case, use collect() to get the
  * whole RDD instead.
  * 获取RDD的第一个num元素。
  * 这将会一次一个地扫描分区，所以如果需要很多分区，它将会很慢。
  * 在这种情况下，使用collect()来获得整个RDD。
  *
  *
  * @note this method should only be used if the resulting array is expected to be small, as
  * all the data is loaded into the driver's memory.
  * 该方法只在预期的数组很小的情况下使用，因为所有的数据都被加载到驱动程序的内存中。
  *
  */
def take(num: Int): JList[T] =
  rdd.take(num).toSeq.asJava
```
### takeSample
```scala
  def takeSample(withReplacement: Boolean, num: Int): JList[T] =
    takeSample(withReplacement, num, Utils.random.nextLong)

  def takeSample(withReplacement: Boolean, num: Int, seed: Long): JList[T] =
    rdd.takeSample(withReplacement, num, seed).toSeq.asJava
```
### first
```scala
  /**
  * Return the first element in this RDD.
  * 返回这个RDD中的第一个元素。
  */
def first(): T = rdd.first()
```
### isEmpty
```scala
/**
  * @return true if and only if the RDD contains no elements at all. Note that an RDD
  *         may be empty even when it has at least 1 partition.
  *         当且仅当RDD不包含任何元素，则为真。
  *         请注意，即使在至少有一个分区的情况下，RDD也可能是空的。
  */
def isEmpty(): Boolean = rdd.isEmpty()
```
### saveAsTextFile
```scala
/**
  * Save this RDD as a text file, using string representations of elements.
  * 将此RDD保存为文本文件，使用元素的字符串表示形式。
  */
def saveAsTextFile(path: String): Unit = {
  rdd.saveAsTextFile(path)
}

/**
  * Save this RDD as a compressed text file, using string representations of elements.
  * 将此RDD保存为一个压缩文本文件，使用元素的字符串表示形式。
  */
def saveAsTextFile(path: String, codec: Class[_ <: CompressionCodec]): Unit = {
  rdd.saveAsTextFile(path, codec)
}
```
### saveAsObjectFile
```scala
/**
  * Save this RDD as a SequenceFile of serialized objects.
  * 将此RDD保存为序列化对象的序列文件。
  */
def saveAsObjectFile(path: String): Unit = {
  rdd.saveAsObjectFile(path)
}
```
### keyBy
```scala
/**
  * Creates tuples of the elements in this RDD by applying `f`.
  * 通过应用“f”创建这个RDD中元素的元组。
  */
def keyBy[U](f: JFunction[T, U]): JavaPairRDD[U, T] = {
  // The type parameter is U instead of K in order to work around a compiler bug; see SPARK-4459
  // 类型参数用U替代K，为了绕过编译器错误;
  implicit val ctag: ClassTag[U] = fakeClassTag
  JavaPairRDD.fromRDD(rdd.keyBy(f))
}
```
### checkpoint
```scala
/**
  * Mark this RDD for checkpointing. It will be saved to a file inside the checkpoint
  * directory set with SparkContext.setCheckpointDir() and all references to its parent
  * RDDs will be removed. This function must be called before any job has been
  * executed on this RDD. It is strongly recommended that this RDD is persisted in
  * memory, otherwise saving it on a file will require recomputation.
  * 将此RDD标记为检查点。
  * 它将被保存到由SparkContext.setCheckpointDir()设置的检查点目录下的文件中。
  * 所有对其父RDDs的引用将被删除。
  * 在此RDD上执行任何作业之前，必须调用此函数。
  * 强烈建议将此RDD保存在内存中，否则将其保存在文件中需要重新计算。
  *
  */
def checkpoint(): Unit = {
  rdd.checkpoint()
}
```
### isCheckpointed
```scala
/**
  * Return whether this RDD has been checkpointed or not
  * 返回  RDD是否已被检查过
  */
def isCheckpointed: Boolean = rdd.isCheckpointed
```
### getCheckpointFile
```scala
/**
  * Gets the name of the file to which this RDD was checkpointed
  * 获取该RDD所指向的checkpointed文件的名称
  */
def getCheckpointFile(): Optional[String] = {
  JavaUtils.optionToOptional(rdd.getCheckpointFile)
}
```
### toDebugString
```scala
/** A description of this RDD and its recursive dependencies for debugging.
  * 对该RDD及其对调试的递归依赖的描述。
  * */
def toDebugString(): String = {
  rdd.toDebugString
}
```
### top
```scala
/**
  * Returns the top k (largest) elements from this RDD as defined by
  * the specified Comparator[T] and maintains the order.
  * 根据指定的比较器[T]，从这个RDD中返回最大的k(最大)元素，并维护顺序。
  *
  *
  * @note this method should only be used if the resulting array is expected to be small, as
  * all the data is loaded into the driver's memory.
  * 该方法只在预期的数组很小的情况下使用，因为所有的数据都被加载到驱动程序的内存中。
  *
  * @param num k, the number of top elements to return  返回的元素数量
  * @param comp the comparator that defines the order  定义排序的比较器
  * @return an array of top elements  返回最大元素的数组
  */
def top(num: Int, comp: Comparator[T]): JList[T] = {
  rdd.top(num)(Ordering.comparatorToOrdering(comp)).toSeq.asJava
}

/**
  * Returns the top k (largest) elements from this RDD using the
  * natural ordering for T and maintains the order.
  * 使用T的自然顺序，从这个RDD中返回最大的k(最大)元素，并维护顺序。
  *
  * @note this method should only be used if the resulting array is expected to be small, as
  * all the data is loaded into the driver's memory.
  * 该方法只在预期的数组很小的情况下使用，因为所有的数据都被加载到驱动程序的内存中。
  *
  * @param num k, the number of top elements to return  返回的元素数量
  * @return an array of top elements 最大元素的数组
  */
def top(num: Int): JList[T] = {
  val comp = com.google.common.collect.Ordering.natural().asInstanceOf[Comparator[T]]
  top(num, comp)
}
```
### takeOrdered
```scala
/**
  * Returns the first k (smallest) elements from this RDD as defined by
  * the specified Comparator[T] and maintains the order.
  * 从这个RDD中返回第一个k(最小)元素，由指定的Comparator[T]定义，并维护该顺序。
  *
  *
  * @note this method should only be used if the resulting array is expected to be small, as
  * all the data is loaded into the driver's memory.
  * 该方法只在预期的数组很小的情况下使用，因为所有的数据都被加载到驱动程序的内存中。
  *
  * @param num k, the number of elements to return  返回的元素数量
  * @param comp the comparator that defines the order  排序比较器
  * @return an array of top elements  元素数组
  */
def takeOrdered(num: Int, comp: Comparator[T]): JList[T] = {
  rdd.takeOrdered(num)(Ordering.comparatorToOrdering(comp)).toSeq.asJava
}

/**
  * Returns the first k (smallest) elements from this RDD using the
  * natural ordering for T while maintain the order.
  * 使用原生的 T排序比较器，返回 k个 最小值，并维护这个顺序
  *
  * @note this method should only be used if the resulting array is expected to be small, as
  * all the data is loaded into the driver's memory.
  * 尽量应用于小的数组，因为会加载到driver内存中。
  * @param num k, the number of top elements to return
  * @return an array of top elements
  */
def takeOrdered(num: Int): JList[T] = {
  val comp = com.google.common.collect.Ordering.natural().asInstanceOf[Comparator[T]]
  takeOrdered(num, comp)
}
```
### max
```scala
/**
  * Returns the maximum element from this RDD as defined by the specified
  * Comparator[T].
  * 按照指定比较器[T]定义的RDD，
  * 返回最大元素。
  *
  * @param comp the comparator that defines ordering  指定的比较器
  * @return the maximum of the RDD  最大值
  */
def max(comp: Comparator[T]): T = {
  rdd.max()(Ordering.comparatorToOrdering(comp))
}
```
### min
```scala
/**
  * Returns the minimum element from this RDD as defined by the specified
  * Comparator[T].
  * 按照指定比较器[T]定义的RDD，
  * 返回最小元素。
  *
  * @param comp the comparator that defines ordering   指定的比较器
  * @return the minimum of the RDD  最小值
  */
def min(comp: Comparator[T]): T = {
  rdd.min()(Ordering.comparatorToOrdering(comp))
}
```
### countApproxDistinct
```scala
/**
* Return approximate number of distinct elements in the RDD.
* 返回RDD中不重复元素的数量近似数。
*
* The algorithm used is based on streamlib's implementation of "HyperLogLog in Practice:
* Algorithmic Engineering of a State of The Art Cardinality Estimation Algorithm", available
  * <a href="http://dx.doi.org/10.1145/2452376.2452456">here</a>.
  * 所使用的算法是基于streamlib在实践中的“HyperLogLog”的实现:
  * “一种艺术基数估计算法状态的算法工程”，
  *
  *
  * @param relativeSD Relative accuracy. Smaller values create counters that require more space.
  *                   It must be greater than 0.000017.
  *                   相对精度。
  *                   较小的值创建需要更多空间的计数器。
  *                   它必须大于0.000017。
  */
  def countApproxDistinct(relativeSD: Double): Long = rdd.countApproxDistinct(relativeSD)
```
### countAsync
```scala
  /**
    * The asynchronous version of `count`, which returns a
    * future for counting the number of elements in this RDD.
    * “count”的异步版本，它为计算这个RDD中元素的数量返回一个未来。
    */
  def countAsync(): JavaFutureAction[jl.Long] = {
  new JavaFutureActionWrapper[Long, jl.Long](rdd.countAsync(), jl.Long.valueOf)
  }
```
### collectAsync
```scala
  /**
    * The asynchronous version of `collect`, which returns a future for
    * retrieving an array containing all of the elements in this RDD.
    * “collect”的异步版本，
    * 它返回一个用于检索包含该RDD中所有元素的数组的未来。
    *
    * @note this method should only be used if the resulting array is expected to be small, as
    * all the data is loaded into the driver's memory.
    * 尽量应用于小数量数组。
    */
  def collectAsync(): JavaFutureAction[JList[T]] = {
  new JavaFutureActionWrapper(rdd.collectAsync(), (x: Seq[T]) => x.asJava)
  }
```
### takeAsync
```scala
  /**
    * The asynchronous version of the `take` action, which returns a
    * future for retrieving the first `num` elements of this RDD.
    * “take”操作的异步版本，
    * 它将返回用于检索此RDD的第一个“num”元素的未来。
    *
    * @note this method should only be used if the resulting array is expected to be small, as
    * all the data is loaded into the driver's memory.
    */
  def takeAsync(num: Int): JavaFutureAction[JList[T]] = {
  new JavaFutureActionWrapper(rdd.takeAsync(num), (x: Seq[T]) => x.asJava)
  }
```
### foreachAsync
```scala
  /**
    * The asynchronous version of the `foreach` action, which
    * applies a function f to all the elements of this RDD.
    * “foreach”操作的异步版本，
    * 它将函数f应用于这个RDD的所有元素。
    *
    */
  def foreachAsync(f: VoidFunction[T]): JavaFutureAction[Void] = {
  new JavaFutureActionWrapper[Unit, Void](rdd.foreachAsync(x => f.call(x)),
  { x => null.asInstanceOf[Void] })
  }
```
### foreachPartitionAsync
```scala
  /**
    * The asynchronous version of the `foreachPartition` action, which
    * applies a function f to each partition of this RDD.
    * “foreachPartition”操作的异步版本，
    * 它将函数f应用于该RDD的每个分区。
    */
  def foreachPartitionAsync(f: VoidFunction[JIterator[T]]): JavaFutureAction[Void] = {
  new JavaFutureActionWrapper[Unit, Void](rdd.foreachPartitionAsync(x => f.call(x.asJava)),
  { x => null.asInstanceOf[Void] })
  }
  }
```
<!--对不起，到时间了，请停止装逼-->


