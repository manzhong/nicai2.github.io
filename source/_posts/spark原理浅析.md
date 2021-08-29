---
title: Spark原理浅析
abbrlink: 33510
date: 2020-08-20 20:45:25
tags: Spark
categories: Spark
summary_img:
encrypt:
enc_pwd:
---

# 一spark特性

## 1 rdd的分区和shuffle

分区的作用

RDD 使用分区来分布式并行处理数据, 并且要做到尽量少的在不同的 Executor 之间使用网络交换数据, 所以当使用 RDD 读取数据的时候, 会尽量的在物理上靠近数据源, 比如说在读取 Cassandra 或者 HDFS 中数据的时候, 会尽量的保持 RDD 的分区和数据源的分区数, 分区模式等一一对应

分区和 Shuffle 的关系

分区的主要作用是用来实现并行计算, 本质上和 Shuffle 没什么关系, 但是往往在进行数据处理的时候, 例如 `reduceByKey`, `groupByKey` 等聚合操作, 需要把 Key 相同的 Value 拉取到一起进行计算, 这个时候因为这些 Key 相同的 Value 可能会坐落于不同的分区, 于是理解分区才能理解 Shuffle 的根本原理

Spark 中的 Shuffle 操作的特点

- 只有 `Key-Value` 型的 RDD 才会有 Shuffle 操作, 例如 `RDD[(K, V)]`, 但是有一个特例, 就是 `repartition` 算子可以对任何数据类型 Shuffle
- 早期版本 Spark 的 Shuffle 算法是 `Hash base shuffle`, 后来改为 `Sort base shuffle`, 更适合大吞吐量的场景

## 2分区操作

很多算子(支持shuffle的 )都可以指定分区数

- 这些算子,可以在最后一个参数的位置传入新的分区数
- 如没有重新指定分区数,默认从父rdd中继承分区数



### 1 查看分区数

~~~scala
scala> sc.parallelize(1 to 100).count
res0: Long = 100
~~~

`spark-shell --master local[8]`, 这样会生成 1 个 Executors, 这个 Executors 有 8 个 Cores, 所以默认会有 8 个 Tasks, 每个 Cores 对应一个分区, 每个分区对应一个 Tasks, 可以通过 `rdd.partitions.size` 来查看分区数量

默认的分区数量是和 Cores 的数量有关的, 也可以通过如下三种方式修改或者重新指定分区数量

- ### **创建 RDD 时指定分区数**

~~~scala
scala> val rdd1 = sc.parallelize(1 to 100, 6)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[1] at parallelize at <console>:24

scala> rdd1.partitions.size
res1: Int = 6

scala> val rdd2 = sc.textFile("hdfs:///dataset/wordcount.txt", 6)
rdd2: org.apache.spark.rdd.RDD[String] = hdfs:///dataset/wordcount.txt MapPartitionsRDD[3] at textFile at <console>:24

scala> rdd2.partitions.size
res2: Int = 7
~~~

rdd1 是通过本地集合创建的, 创建的时候通过第二个参数指定了分区数量. rdd2 是通过读取 HDFS 中文件创建的, 同样通过第二个参数指定了分区数, 因为是从 HDFS 中读取文件, 所以最终的分区数是由 Hadoop 的 InputFormat 来指定的, 所以比指定的分区数大了一个.

- **通过** `coalesce` **算子指定**

~~~scala
coalesce(numPartitions: Int, shuffle: Boolean = false)(implicit ord: Ordering[T] = null): RDD[T]
~~~

numPartitions        新生成的 RDD 的分区数

shuffle            是否 Shuffle

~~~scala
scala> val source = sc.parallelize(1 to 100, 6)
source: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[0] at parallelize at <console>:24

scala> source.partitions.size
res0: Int = 6

scala> val noShuffleRdd = source.coalesce(numPartitions=8, shuffle=false)
noShuffleRdd: org.apache.spark.rdd.RDD[Int] = CoalescedRDD[1] at coalesce at <console>:26

scala> noShuffleRdd.toDebugString 
res1: String =
(6) CoalescedRDD[1] at coalesce at <console>:26 []
 |  ParallelCollectionRDD[0] at parallelize at <console>:24 []

 scala> val noShuffleRdd = source.coalesce(numPartitions=8, shuffle=false)
 noShuffleRdd: org.apache.spark.rdd.RDD[Int] = CoalescedRDD[1] at coalesce at <console>:26

scala> shuffleRdd.toDebugString 
res3: String =
(8) MapPartitionsRDD[5] at coalesce at <console>:26 []
 |  CoalescedRDD[4] at coalesce at <console>:26 []
 |  ShuffledRDD[3] at coalesce at <console>:26 []
 +-(6) MapPartitionsRDD[2] at coalesce at <console>:26 []
    |  ParallelCollectionRDD[0] at parallelize at <console>:24 []

scala> noShuffleRdd.partitions.size     
res4: Int = 6

scala> shuffleRdd.partitions.size
res5: Int = 8

如果 shuffle 参数指定为 false, 运行计划中确实没有 ShuffledRDD, 没有 shuffled 这个过程
如果 shuffle 参数指定为 true, 运行计划中有一个 ShuffledRDD, 有一个明确的显式的 shuffled 过程
如果 shuffle 参数指定为 false 却增加了分区数, 分区数并不会发生改变, 这是因为增加分区是一个宽依赖, 没有 shuffled 过程无法做到, 后续会详细解释宽依赖的概念
~~~

- **通过** `repartition` **算子指定**

~~~scala
repartition(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T]
~~~

`repartition` 算子本质上就是 `coalesce(numPartitions, shuffle = true)`

~~~scala
scala> val source = sc.parallelize(1 to 100, 6)
source: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[7] at parallelize at <console>:24

scala> source.partitions.size
res7: Int = 6

scala> source.repartition(100).partitions.size 
res8: Int = 100

scala> source.repartition(1).partitions.size 
res9: Int = 1
增加分区有效
减少分区有效

~~~

`repartition` 算子无论是增加还是减少分区都是有效的, 因为本质上 `repartition` 会通过 `shuffle` 操作把数据分发给新的 RDD 的不同的分区, 只有 `shuffle` 操作才可能做到增大分区数, 默认情况下, 分区函数是 `RoundRobin`, 如果希望改变分区函数, 也就是数据分布的方式, 可以通过自定义分区函数来实现

![img](/images/spark/fqd.png)

## 2 RDD 的 Shuffle 是什么

**partitioner用于计算一个数据应该发往哪个机器上 (就是哪个分区)**

- HashPartitioner 函数  对key求hashcode和对应的reduce的数量取模 摸出来是几就发往哪个分区
- key .hashCode % reduce count 的值   取值范围在0 到 reduce count之间

**hash base 和sort base 用于描述中间过程如何存放文件***

rdd2 =rdd1.reduceByKey(_+_)    这个reduceByKey() 是属于rdd2的

~~~scala 
val sourceRdd = sc.textFile("hdfs://node01:9020/dataset/wordcount.txt")
val flattenCountRdd = sourceRdd.flatMap(_.split(" ")).map((_, 1))
val aggCountRdd = flattenCountRdd.reduceByKey(_ + _)  //这个reduceByKey() 是属于aggCountRdd
val result = aggCountRdd.collect
~~~

![img](/images/spark/rbk.png)

`reduceByKey` 这个算子本质上就是先按照 Key 分组, 后对每一组数据进行 `reduce`, 所面临的挑战就是 Key 相同的所有数据可能分布在不同的 Partition 分区中, 甚至可能在不同的节点中, 但是它们必须被共同计算.

为了让来自相同 Key 的所有数据都在 `reduceByKey` 的同一个 `reduce` 中处理, 需要执行一个 `all-to-all` 的操作, 需要在不同的节点(不同的分区)之间拷贝数据, 必须跨分区聚集相同 Key 的所有数据, 这个过程叫做 `Shuffle`.

## 3 RDD 的 Shuffle 原理

Spark 的 Shuffle 发展大致有两个阶段: `Hash base shuffle` 和 `Sort base shuffle`

### 3.1 **Hash base shuffle**

![img](/images/spark/hbs.png)

大致的原理是分桶, 假设 Reducer 的个数为 R, 那么每个 Mapper 有 R 个桶, 按照 Key 的 Hash 将数据映射到不同的桶中, Reduce 找到每一个 Mapper 中对应自己的桶拉取数据.

假设 Mapper 的个数为 M, 整个集群的文件数量是 `M * R`, 如果有 1,000 个 Mapper 和 Reducer, 则会生成 1,000,000 个文件, 这个量非常大了.

过多的文件会导致文件系统打开过多的文件描述符, 占用系统资源. 所以这种方式并不适合大规模数据的处理, 只适合中等规模和小规模的数据处理, 在 Spark 1.2 版本中废弃了这种方式.

### 3.2 **Sort base shuffle**

![img](/images/spark/sbs.png)

对于 Sort base shuffle 来说, 每个 Map 侧的分区只有一个输出文件, Reduce 侧的 Task 来拉取, 大致流程如下

1. Map 侧将数据全部放入一个叫做 AppendOnlyMap 的组件中, 同时可以在这个特殊的数据结构中做聚合操作
2. 然后通过一个类似于 MergeSort 的排序算法 TimSort 对 AppendOnlyMap 底层的 Array 排序
   - 先按照 Partition ID 排序, 后按照 Key 的 HashCode 排序
3. 最终每个 Map Task 生成一个 输出文件, Reduce Task 来拉取自己对应的数据

从上面可以得到结论, Sort base shuffle 确实可以大幅度减少所产生的中间文件, 从而能够更好的应对大吞吐量的场景, 在 Spark 1.2 以后, 已经默认采用这种方式.

但是需要大家知道的是, Spark 的 Shuffle 算法并不只是这一种, 即使是在最新版本, 也有三种 Shuffle 算法, 这三种算法对每个 Map 都只产生一个临时文件, 但是产生文件的方式不同, 一种是类似 Hash 的方式, 一种是刚才所说的 Sort, 一种是对 Sort 的一种优化(使用 Unsafe API 直接申请堆外内存)

# 二 缓存

缓存是将数据缓存到blockMananger 中   导入隐式转换的意义 是可以获得ds和df的底层 row 和internelRow 

## 1 缓存的意义

**使用缓存的原因 - 多次使用 RDD**  减少shuffle,减少其他算子执行,缓存算子生成结果

需求:

在日志文件中找到访问次数最少的 IP 和访问次数最多的 IP

```scala 
package com.nicai.demo

import org.apache.commons.lang3.StringUtils
import org.apache.spark.{SparkConf, SparkContext}
import org.junit.Test

class CacheDemo {
  /**
    * 1. 创建sc
    * 2. 读取文件
    * 3. 取出IP, 赋予初始频率
    * 4. 清洗
    * 5. 统计IP出现的次数
    * 6. 统计出现次数最少的IP
    * 7. 统计出现次数最多的IP
    */
  @Test
  def prepare(): Unit = {
    // 1. 创建 SC
    val conf = new SparkConf().setAppName("cache_prepare").setMaster("local[6]")
    val sc = new SparkContext(conf)

    // 2. 读取文件
    val source = sc.textFile("G:\\develop\\data\\access_log_sample.txt")

    // 3. 取出IP, 赋予初始频率
    val countRDD = source.map( item => (item.split(" ")(0), 1) )

    // 4. 数据清洗
    val cleanRDD = countRDD.filter( item => StringUtils.isNotEmpty(item._1) )

    // 5. 统计IP出现的次数(聚合)
    val aggRDD = cleanRDD.reduceByKey( (curr, agg) => curr + agg )

    // 6. 统计出现次数最少的IP(得出结论)  
    val lessIp = aggRDD.sortBy(item => item._2, ascending = true).first()

    // 7. 统计出现次数最多的IP(得出结论) 降序
    val moreIp = aggRDD.sortBy(item => item._2, ascending = false).first()

    println((lessIp, moreIp))
  }
}
5是一个 Shuffle 操作, Shuffle 操作会在集群内进行数据拷贝
在上述代码中, 多次使用到了 aggRDD并执行了action操作, 导致文件读取两次,6之前的代码(执行)计算两次, 性能有损耗
```



**使用缓存的原因 - 容错**

![img](/images/spark/rc2.png)

当在计算 RDD3 的时候如果出错了, 会怎么进行容错?

会再次计算 RDD1 和 RDD2 的整个链条, 假设 RDD1 和 RDD2 是通过比较昂贵的操作得来的, 有没有什么办法减少这种开销?

上述两个问题的解决方案其实都是 `缓存`, 除此之外, 使用缓存的理由还有很多, 但是总结一句, 就是缓存能够帮助开发者在进行一些昂贵操作后, 将其结果保存下来, 以便下次使用无需再次执行, 缓存能够显著的提升性能.

所以, 缓存适合在一个 RDD 需要重复多次利用, 并且还不是特别大的情况下使用, 例如迭代计算等场景.

## 2 缓存相关的 API

### 2.1 **可以使用** `cache` **方法进行缓存**

~~~~scala
  //使用cache缓存
  @Test
  def cacheDemo(): Unit = {
    val conf = new SparkConf().setMaster("local[3]").setAppName("cache")
    val context =new  SparkContext(conf)
    //读取文件
    val file = context.textFile("G:\\develop\\bigdatas\\BigData\\day26sparkActions\\data\\access_log_sample.txt")
    val rdd1 = file.map(it => ((it.split(" ")) (0), 1))
      .reduceByKey((curr, agg) => curr + agg)
      .cache()                //缓存结果
    val rddMax = rdd1
      .sortBy(item => item._2, true)
      .first()
    val rddMin = rdd1.sortBy(item2 => item2._2,false).first()
    println(rddMax,rddMin)
  }
  //其实 cache 方法底层就是persist()方法  这个persist()  底层有调用了persist(StorageLevel.MEMORY_ONLY)方法   StorageLevel.MEMORY_ONLY为其默认
~~~~



### 2.2 **也可以使用 persist 方法进行缓存**

````scala
 //使用persist缓存
  @Test
  def persistDemo(): Unit = {
    val conf = new SparkConf().setMaster("local[3]").setAppName("cache")
    val context =new  SparkContext(conf)
    //读取文件
    val file = context.textFile("G:\\develop\\bigdatas\\BigData\\day26sparkActions\\data\\access_log_sample.txt")
    val rdd1 = file.map(it => ((it.split(" ")) (0), 1))
      .reduceByKey((curr, agg) => curr + agg)
      .persist(StorageLevel.MEMORY_ONLY)              //缓存结果  与cache, persist()一样
    //.persist(参数)    //缓存级别
    val rddMax = rdd1
      .sortBy(item => item._2, true)
      .first()
    val rddMin = rdd1.sortBy(item2 => item2._2,false).first()
    println(rddMax,rddMin)
  }
  //其实 cache 方法底层就是persist()方法  这个persist()  底层有调用了persist(StorageLevel.MEMORY_ONLY)方法   StorageLevel.MEMORY_ONLY为其默认
````



### 2.3 清理缓存

~~~~scala
 @Test
  def cacheDemo(): Unit = {
    val conf = new SparkConf().setMaster("local[3]").setAppName("cache")
    val context =new  SparkContext(conf)
    //读取文件
    val file = context.textFile("G:\\develop\\bigdatas\\BigData\\day26sparkActions\\data\\access_log_sample.txt")
    val rdd1 = file.map(it => ((it.split(" ")) (0), 1))
      .reduceByKey((curr, agg) => curr + agg)
      .cache()                //缓存结果
      
      //清除缓存
      rdd.unpersist()  
    val rddMax = rdd1
      .sortBy(item => item._2, true)
      .first()
    val rddMin = rdd1.sortBy(item2 => item2._2,false).first()
    println(rddMax,rddMin)
    context.close()
  }
  根据缓存级别的不同, 缓存存储的位置也不同, 但是使用 unpersist 可以指定删除 RDD 对应的缓存信息, 并指定缓存级别为 NONE
~~~~



## 3 缓存级别

缓存是一个技术活,  要考虑很多

- 是否使用磁盘缓存?
- 是否使用内存缓存?
- 是否使用堆外内存?
- 缓存前是否先序列化?
- 是否需要有副本?

### 3.1 查看缓存级别对象

~~~~
 @Test
  def persistDemo(): Unit = {
    val conf = new SparkConf().setMaster("local[3]").setAppName("cache")
    val context =new  SparkContext(conf)
    //读取文件
    val file = context.textFile("G:\\develop\\bigdatas\\BigData\\day26sparkActions\\data\\access_log_sample.txt")
    val rdd1 = file.map(it => ((it.split(" ")) (0), 1))
      .reduceByKey((curr, agg) => curr + agg)
      .persist(StorageLevel.MEMORY_ONLY)              //缓存结果  与cache, persist()一样
    //.persist(参数)    //缓存级别
    //查看缓存级别
    println(rdd1.getStorageLevel)
    val rddMax = rdd1
      .sortBy(item => item._2, true)
      .first()
    val rddMin = rdd1.sortBy(item2 => item2._2,false).first()
    println(rddMax,rddMin)
    context.close()
  }
~~~~

deserialized是否以反序列化形式存储   若是true 则不序列化 ,若是false 则存储序列化后的值

其值为true 存储的是对象   若是false 则存储的是二进制文件

带2的为副本数

| 缓存级别                                 | `userDisk` 是否使用磁盘 | `useMemory` 是否使用内存 | `useOffHeap` 是否使用堆外内存 | `deserialized`是否以反序列化形式存储 | `replication` 副本数 |
| :----------------------------------- | :---------------- | :----------------- | :-------------------- | :------------------------ | :---------------- |
| `NONE` 就是不存                          | false             | false              | false                 | false                     | 1                 |
| `DISK_ONLY`  存到磁盘里                   | true              | false              | false                 | false                     | 1                 |
| `DISK_ONLY_2`                        | true              | false              | false                 | false                     | 2                 |
| `MEMORY_ONLY`  存到内存                  | false             | true               | false                 | true                      | 1                 |
| `MEMORY_ONLY_2`                      | false             | true               | false                 | true                      | 2                 |
| `MEMORY_ONLY_SER`存到内存以二进制文件的形式       | false             | true               | false                 | false                     | 1                 |
| `MEMORY_ONLY_SER_2`                  | false             | true               | false                 | false                     | 2                 |
| `MEMORY_AND_DISK`内存和磁盘都有             | true              | true               | false                 | true                      | 1                 |
| `MEMORY_AND_DISK`                    | true              | true               | false                 | true                      | 2                 |
| `MEMORY_AND_DISK_SER`内存和磁盘都有以二进制形式存储 | true              | true               | false                 | false                     | 1                 |
| `MEMORY_AND_DISK_SER_2`              | true              | true               | false                 | false                     | 2                 |
| `OFF_HEAP`                           | true              | true               | true                  | false                     | 1                 |

### 3.2如何选择分区级别

Spark 的存储级别的选择，核心问题是在 memory 内存使用率和 CPU 效率之间进行权衡。建议按下面的过程进行存储级别的选择:

如果您的 RDD 适合于默认存储级别（MEMORY_ONLY），leave them that way。这是 CPU 效率最高的选项，允许 RDD 上的操作尽可能快地运行.

如果不是，试着使用 MEMORY_ONLY_SER 和 selecting a fast serialization library 以使对象更加节省空间，但仍然能够快速访问。(Java和Scala)

不要溢出到磁盘，除非计算您的数据集的函数是昂贵的，或者它们过滤大量的数据。否则，重新计算分区可能与从磁盘读取分区一样快.

如果需要快速故障恢复，请使用复制的存储级别（例如，如果使用 Spark 来服务 来自网络应用程序的请求）。All 存储级别通过重新计算丢失的数据来提供完整的容错能力，但复制的数据可让您继续在 RDD 上运行任务，而无需等待重新计算一个丢失的分区.

# 三 Checkpoint

## 1 Checkpoint 的作用

​	Checkpoint 的主要作用是斩断 RDD 的依赖链, 并且将数据存储在可靠的存储引擎中, 例如支持分布式存储和副本机制的 HDFS.

**Checkpoint 的方式**

- **可靠的** 将数据存储在可靠的存储引擎中, 例如 HDFS
- **本地的** 将数据存储在本地

**什么是斩断依赖链**

斩断依赖链是一个非常重要的操作, 接下来以 HDFS 的 NameNode 的原理来举例说明

HDFS 的 NameNode 中主要职责就是维护两个文件, 一个叫做 `edits`, 另外一个叫做 `fsimage`. `edits` 中主要存放 `EditLog`, `FsImage` 保存了当前系统中所有目录和文件的信息. 这个 `FsImage` 其实就是一个 `Checkpoint`.

HDFS 的 NameNode 维护这两个文件的主要过程是, 首先, 会由 `fsimage` 文件记录当前系统某个时间点的完整数据, 自此之后的数据并不是时刻写入 `fsimage`, 而是将操作记录存储在 `edits` 文件中. 其次, 在一定的触发条件下, `edits` 会将自身合并进入 `fsimage`. 最后生成新的 `fsimage` 文件, `edits` 重置, 从新记录这次 `fsimage` 以后的操作日志.

如果不合并 `edits` 进入 `fsimage` 会怎样? 会导致 `edits` 中记录的日志过长, 容易出错.

所以当 Spark 的一个 Job 执行流程过长的时候, 也需要这样的一个斩断依赖链的过程, 使得接下来的计算轻装上阵.

**Checkpoint 和 Cache 的区别**

Cache 可以把 RDD 计算出来然后放在内存中, 但是 RDD 的依赖链(相当于 NameNode 中的 Edits 日志)是不能丢掉的, 因为这种缓存是不可靠的, 如果出现了一些错误(例如 Executor 宕机), 这个 RDD 的容错就只能通过回溯依赖链, 重放计算出来.

但是 Checkpoint 把结果保存在 HDFS 这类存储中, 就是可靠的了, 所以可以斩断依赖, 如果出错了, 则通过复制 HDFS 中的文件来实现容错.

所以他们的区别主要在以下两点

- Checkpoint 可以保存数据到 HDFS 这类可靠的存储上, Persist 和 Cache 只能保存在本地的磁盘和内存中
- Checkpoint 可以斩断 RDD 的依赖链, 而 Persist 和 Cache 不行
- 因为 CheckpointRDD 没有向上的依赖链, 所以程序结束后依然存在, 不会被删除. 而 Cache 和 Persist 会在程序结束后立刻被清除.

## 2  使用 Checkpoint

~~~SCALA 
//checkpoint
  @Test
  def checkPointDemo(): Unit = {
    val conf = new SparkConf().setMaster("local[3]").setAppName("cache")
    val context =new  SparkContext(conf)
    //设置保存checkpoint的目录 也可以在hdfs上
    context.setCheckpointDir("checkpoint")
    //读取文件
    val file = context.textFile("G:\\develop\\bigdatas\\BigData\\day26sparkActions\\data\\access_log_sample.txt")
    val rdd1 = file.map(it => ((it.split(" ")) (0), 1))
      .reduceByKey((curr, agg) => curr + agg)
      /*
      checkpoint的返回值为unit空
       1 不准确的说checkpoint是一个action操作,
       2 如果调用 checkpoint则会重新计算rdd,然后把结果存在hdfs或本地目录
       所以在checkpoint之前应该先缓存一下 
       */
      .persist(StorageLevel.MEMORY_ONLY)
      rdd1.checkpoint()
    val rddMax = rdd1
      .sortBy(item => item._2, true)
      .first()
    val rddMin = rdd1.sortBy(item2 =>item2._2,false).first()
    println(rddMax,rddMin)
  }
~~~

