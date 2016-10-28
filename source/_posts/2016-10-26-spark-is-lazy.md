---
layout: post
title:  "Spark 的惰性运算"
date:   2016-10-26 16:21:39
comments: true
ads: true
categories: [大数据,Spark]
tags: [Spark,Scala,函数式编程,惰性计算]
---

今天在检视项目代码的时候，无意中发现了下面一段代码：

```scala
class RddTransformer{

  def doTransform(data: RDD[Data]): RDD[NewData]={
    val newDataRdd = data.flatmap(DataTransformer.doTransform)

    if(DataTransformer.exceptionCount > 0) {
      logger.error(s"There are some illegal data, count: ${DataTransformer.exceptionCount}")
    }

    newDataRdd
  }
}

object DataTransformer{
  var exceptionCount:Int = 0

  def doTransform(data: Data): Option[NewData]={
    if(data.isIllegal){
      exceptionCount += 1
      None
    }else{
      // do something transform data to new data
      .....

      Some(newData)
    }
  }
}
```

<!-- more -->

作者的意图很简单，就是将RDD中的数据转换为新的数据格式，并统计非法数据的个数。咋一看代码，似乎没有什么问题，可是，这段代码真的能得到正确的结果么？答案是否定的，事实上，不管RDD中包含多少非法数据，`if(DataTransformer.exceptionCount > 0)`这个条件永远都不会为真。为什么？你现在肯定充满了疑惑，让我们先来看看 Spark 的文档上对 RDD 操作的解释：

>All transformations in Spark are lazy, in that they do not compute their results right away. Instead, they just remember the transformations applied to some base dataset (e.g. a file). The transformations are only computed when an action requires a result to be returned to the driver program. ([RDD Operations](http://spark.apache.org/docs/latest/programming-guide.html#rdd-operations))

> 在 Spark 中，所有的 transformation() 类型操作都是延迟计算的，Spark 只是记录了将要对数据集进行的操作。只有需要数据集将数据返回到 Driver 程序时（即触发 Action 类型操作），所有已记录的 transformation() 才会执行。

回到上面的代码，由于针对`RDD[Data]`的`flatmap`操作属于 transformation() 类型操作，所以`val newDataRdd = data.flatmap(DataTransformer.doTransform)`这段代码只是记录了一下对 RDD 的操作，并没有真正的去执行`DataTransformer.doTransform`方法中的代码。我们可以尝试在 Spark Shell 中实验一下：

```shell
scala> var counter = 0
counter: Int = 0

scala> var rdd = sc.parallelize(Seq(1,2,3,4,5,6)).map(x => counter += x)
rdd: spark.RDD[Int] = spark.MappedRDD@2ee9b6e3

scala> counter
counter: Int = 0

```

显然累加操作并没有被执行，根据 Shell 终端的输出，Spark 似乎只是记录了一下我们的操作，并返回了一个新的 RDD。当对 RDD 进行 transformation() 操作的时候，在 Spark 内部究竟发生了什么？我们先看一个图，典型的 Spark Job 逻辑执行图如下所示：

![GeneralLogicalPlan](/assets/images/2016-10-26-spark-is-lazy/GeneralLogicalPlan.png)

根据上图，Spark Job 经过下面四个步骤可以得到最终执行结果：

+ 从数据源（可以是本地 file，内存数据结构， HDFS，HBase 等）读取数据创建最初的 RDD。上一段代码中的 parallelize() 相当于 createRDD()。
+ 对 RDD 进行一系列的 transformation() 操作，每一个 transformation() 会产生一个或多个包含不同类型 T 的 RDD[T]。T 可以是 Scala 里面的基本类型或数据结构，不限于 (K, V)。但如果是 (K, V)，K 不能是 Array 等复杂类型（因为难以在复杂类型上定义 partition 函数）。
+ 对最后的 final RDD 进行 action() 操作，每个 partition 计算后产生结果 result。
+ 将 result 回送到 driver 端，进行最后的 f(list[result]) 计算。例子中的 count() 实际包含了action() 和 sum() 两步计算。

由此可知，RDD 中数据的计算是惰性的，一系列 transformation() 操作只有在遇到 action() 操作时候，才会真的去计算 RDD 分区内的数据。你一定很好奇这整个过程是如何实现的，那我们就来看看 RDD 的计算实现。

## 数据计算过程

下面的代码段，展现了`RDD.flatmap()`和`MapPartitionsRDD`的实现，在代码中，我们看到，当调用`RDD`的`map`并传入一个函数`f`的时候，Spark 并没有做什么运算，而是用`f`作为一个入参创建了一个叫`MapPartitionsRDD`的对象并返回给调用者。而在`MapPartitionsRDD.scala`中，我们也看到只有当`compute`方法被调用的时候，我们之前传入的函数`f`才会真正的被执行

```scala

  // RDD.scala
  ...
  /**
   * Return a new RDD by applying a function to all elements of this RDD.
   */
  def flatmap[U: ClassTag](f: T => U): RDD[U] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.map(cleanF))
  }

  // MapPartitionsRDD.scala
  private[spark] class MapPartitionsRDD[U: ClassTag, T: ClassTag](
    var prev: RDD[T],
    f: (TaskContext, Int, Iterator[T]) => Iterator[U],  // (TaskContext, partition index, iterator)
    preservesPartitioning: Boolean = false)
  extends RDD[U](prev) {

  override val partitioner = if (preservesPartitioning) firstParent[T].partitioner else None

  override def getPartitions: Array[Partition] = firstParent[T].partitions

  override def compute(split: Partition, context: TaskContext): Iterator[U] =
    f(context, split.index, firstParent[T].iterator(split, context))

  override def clearDependencies() {
    super.clearDependencies()
    prev = null
  }
}
```

实际计算过程大概是这样的：

1. 根据动作操作来将一个应用程序划分成多个作业。
2. 一个作业经历 DAG 调度和任务调度之后，被划分成一个一个的任务，对应 Task 类。
3. 任务被分配到不同核心去执行，执行 Task.run。
4. Task.run 会调用阶段末 RDD 的 iterator 方法，获取该 RDD 某个分区内的数据记录，而 iterator 方法有可能会调用 RDD 类的 compute 方法来负责父 RDD 与子 RDD 之间的计算逻辑。

整个过程会比较复杂，在此不进行展开，我们只需要知道 Apache Spark 最终会调用 RDD 的 iterator 和 compute 方法来计算分区数据即可。

### compute 方法
在 RDD 中，`compute()`被定义为抽象方法，要求其所有子类都必须实现，该方法接受的参数之一是一个`Partition`对象，目的是计算该分区中的数据。以之前`flatmap`操作生成得到的`MapPartitionsRDD`类为例。

```scala
override def compute(split: Partition, context: TaskContext): Iterator[U] =
  f(context, split.index, firstParent[T].iterator(split, context))
```

其中，`firstParent`在 RDD 中定义。

```scala
  /** Returns the first parent RDD */
  protected[spark] def firstParent[U: ClassTag] = {
    dependencies.head.rdd.asInstanceOf[RDD[U]]
  }
```

`MapPartitionsRDD`类的`compute`方法调用当前 RDD 内的第一个父 RDD 的`iterator`方法，该方的目的是拉取父 RDD 对应分区内的数据，它返回一个迭代器对象，迭代器内部存储的每个元素即父 RDD 对应分区内已经计算完毕的数据记录。得到的迭代器作为`f`方法的一个参数。`compute`方法会将迭代器中的记录一一输入`f`方法，得到的新迭代器即为所求分区中的数据。

### iterator方法

`iterator`方法的实现在 RDD 类中。

```scala
  /**
   * Internal method to this RDD; will read from cache if applicable, or otherwise compute it.
   * This should ''not'' be called by users directly, but is available for implementors of custom
   * subclasses of RDD.
   */
  final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
    if (storageLevel != StorageLevel.NONE) {
      SparkEnv.get.cacheManager.getOrCompute(this, split, context, storageLevel)
    } else {
      computeOrReadCheckpoint(split, context)
    }
  }
```

`iterator`方法首先检查当前 RDD 的存储级别，如果存储级别不为`None`，说明分区的数据要么已经存储在文件系统当中，要么当前 RDD 曾经执行过`cache`、`persise`等持久化操作，因此需要想办法把数据从存储介质中提取出来。`iterator`方法继续调用`CacheManager`的`getOrCompute`方法。

```scala
 /** Gets or computes an RDD partition. Used by RDD.iterator() when an RDD is cached. */
  def getOrCompute[T](
      rdd: RDD[T],
      partition: Partition,
      context: TaskContext,
      storageLevel: StorageLevel): Iterator[T] = {
    val key = RDDBlockId(rdd.id, partition.index)
    blockManager.get(key) match {
       case Some(blockResult) =>
         // Partition is already materialized, so just return its values
         context.taskMetrics.inputMetrics = Some(blockResult.inputMetrics)
         new InterruptibleIterator(context, blockResult.data.asInstanceOf[Iterator[T]])    
       case None =>
         // 省略部分源码
         val computedValues = rdd.computeOrReadCheckpoint(partition, context)
         val cachedValues = putInBlockManager(key, computedValues, storageLevel, updatedBlocks)
         new InterruptibleIterator(context, cachedValues)
    }
    // 省略部分源码
 }
 ```

`getOrCompute`方法会根据 RDD 编号与分区编号计算得到当前分区在存储层对应的块编号，通过存储层提供的数据读取接口提取出块的数据。这时候会有两种可能情况发生：

+ 数据之前已经存储在存储介质当中，可能是数据本身就在存储介质（如读取 HDFS 中的文件创建得到的 RDD）当中，也可能是 RDD 经过持久化操作并经历了一次计算过程。这时候就能成功提取得到数据并将其返回。
+ 数据不在存储介质当中，可能是数据已经丢失，或者 RDD 经过持久化操作，但是是当前分区数据是第一次被计算，因此会出现拉取得到数据为 None 的情况。这就意味着我们需要计算分区数据，继续调用 RDD 类 computeOrReadCheckpoint 方法来计算数据，并将计算得到的数据缓存到存储介质中，下次就无需再重复计算。
+ 如果当前RDD的存储级别为 None，说明为未经持久化的 RDD，需要重新计算 RDD 内的数据，这时候调用 RDD 类的 computeOrReadCheckpoint 方法，该方法也在持久化 RDD 的分区获取数据失败时被调用。

```scala
  /**
   * Compute an RDD partition or read it from a checkpoint if the RDD is checkpointing.
   */
  private[spark] def computeOrReadCheckpoint(split: Partition, context: TaskContext): Iterator[T] = {
    if (isCheckpointed) firstParent[T].iterator(split, context) else compute(split, context)
  }
```

`computeOrReadCheckpoint`方法会检查当前 RDD 是否已经被标记成检查点，如果未被标记成检查点，则执行自身的`compute`方法来计算分区数据，否则就直接拉取父 RDD 分区内的数据。

## 如何正确的获取计算结果

说了那么多理论，我们回到问题本身，怎么才是获取运算结果的正确方法？你也许会说，既然 transformation() 操作是惰性的，那么在之后马上触发一个 action() 操作就 OK 了。但这也是不正确的，这就涉及到了 Spark 的另外一个重要概念：分布式，在这里就不展开讲了，有兴趣可以参考官方文档：[Understanding closures ](http://spark.apache.org/docs/latest/programming-guide.html#understanding-closures-a-nameclosureslinka)。

下面是一个正确的实现：

```scala

class RddTransformer{

  def doTransform(data: RDD[Data]): RDD[NewData]={
    val newDataRdd = data.flatmap(DataTransformer.doTransform).cache()

    val exceptionCount = newDataRdd.filter(_.isEmpty).count()
    if(exceptionCount > 0) {
      logger.error(s"There are some illegal data, count: ${DataTransformer.exceptionCount}")
    }

    newDataRdd
  }
}

object DataTransformer{

  def doTransform(data: Data): Option[NewData]={
    if(data.isIllegal){
      None
    }else{
      // do something transform data to new data
      .....

      Some(newData)
    }
  }
}

```
