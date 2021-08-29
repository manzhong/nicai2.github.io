---
title: Spark原理浅析2
abbrlink: 33510
date: 2017-08-19 20:45:25
tags: Spark
categories: Spark
summary_img:
encrypt:
enc_pwd:
---

# 一 原理

## 1 spark部署情况

在 `Spark` 部分的底层执行逻辑开始之前, 还是要先认识一下 `Spark` 的部署情况, 根据部署情况, 从而理解如何调度.

![img](/images/sparkyl/dc1.png)

针对于上图, 首先可以看到整体上在集群中运行的角色有如下几个:

- `Master Daemon`

  负责管理 `Master` 节点, 协调资源的获取, 以及连接 `Worker` 节点来运行 `Executor`, 是 Spark 集群中的协调节点

- `Worker Daemon`

  `Workers` 也称之为叫 `Slaves`, 是 Spark 集群中的计算节点, 用于和 Master 交互并管理 `Executor`.

  当一个 `Spark Job` 提交后, 会创建 `SparkContext`, 后 `Worker` 会启动对应的 `Executor`.

- `Executor Backend`

  上面有提到 `Worker` 用于控制 `Executor` 的启停, 其实 `Worker` 是通过 `Executor Backend` 来进行控制的, `Executor Backend` 是一个进程(是一个 `JVM` 实例), 持有一个 `Executor` 对象

另外在启动程序的时候, 有三种程序需要运行在集群上:

- `Driver`  **action 操作的最终获得结果,是把结果存放在Driver中**

  `Driver` 是一个 `JVM` 实例, 是一个进程, 是 `Spark Application` 运行时候的领导者, 其中运行了 `SparkContext`.

  `Driver` 控制 `Job` 和 `Task`, 并且提供 `WebUI`.

- `Executor`

  `Executor` 对象中通过线程池来运行 `Task`, 一个 `Executor` 中只会运行一个 `Spark Application` 的 `Task`, 不同的 `Spark Application` 的 `Task` 会由不同的 `Executor` 来运行

## 2 逻辑执行图  简述

**描述数据如何流动,如何计算  实际并不存在,只是保存了rdd之间的关系**

![img](/images/sparkyl/dc2.png)

其实 RDD 并没有什么严格的逻辑执行图和物理执行图的概念, 这里也只是借用这个概念, 从而让整个 RDD 的原理可以解释, 好理解.

对于 RDD 的逻辑执行图, 起始于第一个入口 RDD 的创建, 结束于 Action 算子执行之前, 主要的过程就是生成一组互相有依赖关系的 RDD, 其并不会真的执行, 只是表示 RDD 之间的关系, 数据的流转过程.

~~~
rdd.toDebugString  预运行  结果就是一个逻辑执行图
~~~

## 3 物理执行图 简述

当触发 Action 执行的时候, 这一组互相依赖的 RDD 要被处理, 所以要转化为可运行的物理执行图, 调度到集群中执行.

因为大部分 RDD 是不真正存放数据的, 只是数据从中流转, 所以, 不能直接在集群中运行 RDD, 要有一种 Pipeline 的思想, 需要将这组 RDD 转为 Stage 和 Task, 从而运行 Task, 优化整体执行速度.

以上的逻辑执行图会生成如下的物理执行图, 这一切发生在 Action 操作被执行时.

![img](/images/sparkyl/dc3.png)

从上图可以总结如下几个点

- 1 ---> 2-->3在第一个 `Stage` 中, 每一个这样的执行流程是一个 `Task`, 也就是在同一个 Stage 中的所有 RDD 的对应分区, 在同一个 Task 中执行
- Stage 的划分是由 Shuffle 操作来确定的, 有 Shuffle 的地方, Stage 断开

## 4 逻辑执行图 详述

### 1 明确逻辑计划的边界

在 `Action` 调用之前, 会生成一系列的 `RDD`, 这些 `RDD` 之间的关系, 其实就是整个逻辑计划

 如果生成逻辑计划的, 会生成如下一些 `RDD`, 这些 `RDD` 是相互关联的, 这些 `RDD` 之间, 其实本质上生成的就是一个 **计算链**

![img](/images/sparkyl/dc4.png)

接下来, 采用迭代渐进式的方式, 一步一步的查看一下整体上的生成过程

开始: 第一个rdd创建开始

结束:  逻辑图到action算子执行之前及结束 

逻辑图就是一组rdd和其之间的关系

rdd五大属性: 1 分区列表 2 依赖关系 3 计算函数  4 最佳位置5 分区函数

### 2`textFile` **算子的背后**

研究 `RDD` 的功能或者表现的时候, 其实本质上研究的就是 `RDD` 中的五大属性, 因为 `RDD` 透过五大属性来提供功能和表现, 所以如果要研究 `textFile` 这个算子, 应该从五大属性着手, 那么第一步就要看看生成的 `RDD` 是什么类型的 `RDD`

#### 1`textFile` 生成的是 `HadoopRDD`

HadoopRDD 重写了分区列表和计算函数

源码:

~~~scala
def textFile(
      path: String,
      minPartitions: Int = defaultMinPartitions): RDD[String] = withScope {
    assertNotStopped()
    hadoopFile(path, classOf[TextInputFormat], classOf[LongWritable], classOf[Text],
      minPartitions).map(pair => pair._2.toString).setName(path)
  }

def hadoopFile[K, V](
      path: String,
      inputFormatClass: Class[_ <: InputFormat[K, V]],
      keyClass: Class[K],
      valueClass: Class[V],
      minPartitions: Int = defaultMinPartitions): RDD[(K, V)] = withScope {
    assertNotStopped()

    // This is a hack to enforce loading hdfs-site.xml.
    // See SPARK-11227 for details.
    FileSystem.getLocal(hadoopConfiguration)

    // A Hadoop configuration can be about 10 KB, which is pretty big, so broadcast it.
    val confBroadcast = broadcast(new SerializableConfiguration(hadoopConfiguration))
    val setInputPathsFunc = (jobConf: JobConf) => FileInputFormat.setInputPaths(jobConf, path)
    new HadoopRDD(
      this,
      confBroadcast,
      Some(setInputPathsFunc),
      inputFormatClass,
      keyClass,
      valueClass,
      minPartitions).setName(path)
  }
~~~

#### 2`HadoopRDD` 的 `Partitions` 对应了 `HDFS` 的 `Blocks`

![img](/images/sparkyl/dc5.png)

其实本质上每个 `HadoopRDD` 的 `Partition` 都是对应了一个 `Hadoop` 的 `Block`, 通过 `InputFormat` 来确定 `Hadoop` 中的 `Block` 的位置和边界, 从而可以供一些算子使用

#### 3`HadoopRDD` 的 `compute` 函数就是在读取 `HDFS` 中的 `Block`

本质上, `compute` 还是依然使用 `InputFormat` 来读取 `HDFS` 中对应分区的 `Block`

#### 4`textFile` 这个算子生成的其实是一个 `MapPartitionsRDD`

`textFile` 这个算子的作用是读取 `HDFS` 上的文件, 但是 `HadoopRDD` 中存放是一个元组, 其 `Key` 是行号, 其 `Value` 是 `Hadoop` 中定义的 `Text` 对象, 这一点和 `MapReduce` 程序中的行为是一致的

但是并不适合 `Spark` 的场景, 所以最终会通过一个 `map` 算子, 将 `(LineNum, Text)` 转为 `String` 形式的一行一行的数据, 所以最终 `textFile` 这个算子生成的 `RDD` 并不是 `HadoopRDD`, 而是一个 `MapPartitionsRDD`

### 2 `map` **算子的底层**

~~~~scala
def map[U: ClassTag](f: T => U): RDD[U] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.map(cleanF))
  }
//iter为每个分区的数组
// MapPartitionsRDD的compute函数,处理整个rdd中的每个分区的每一条数据,通过穿入map的函数来进行处理

~~~~

- `map` 算子生成了 `MapPartitionsRDD`

  由源码可知, 当 `val rdd2 = rdd1.map()` 的时候, 其实生成的新 `RDD` 是 `rdd2`, `rdd2` 的类型是 `MapPartitionsRDD`, 每个 `RDD` 中的五大属性都会有一些不同, 由 `map` 算子生成的 `RDD` 中的计算函数, 本质上就是遍历对应分区的数据, 将每一个数据转成另外的形式

- `MapPartitionsRDD` 的计算函数是 `collection.map( function )`

  真正运行的集群中的处理单元是 `Task`, 每个 `Task` 对应一个 `RDD` 的分区, 所以 `collection` 对应一个 `RDD` 分区的所有数据, 而这个计算的含义就是将一个 `RDD` 的分区上所有数据当作一个集合, 通过这个 `Scala` 集合的 `map` 算子, 来执行一个转换操作, 其转换操作的函数就是传入 `map` 算子的 `function`

- 传入 `map` 算子的函数会被清理

~~~
 val cleanF = sc.clean(f)
~~~

这个清理主要是处理闭包中的依赖, 使得这个闭包可以被序列化发往不同的集群节点运行

### 3 `flatMap` 算子的底层

~~~scala
def flatMap[U: ClassTag](f: T => TraversableOnce[U]): RDD[U] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.flatMap(cleanF))
  }
map与flatmap的区别从本质上来说就是作用于分区数据集合的算子不同  一个是iter.flatMap(cleanF))和iter.map(cleanF))
// MapPartitionsRDD的compute函数,处理整个rdd中的每个分区的每一条数据,通过传入flatMap的函数来进行处理
~~~

```
flatMap` 和 `map` 算子其实本质上是一样的, 其步骤和生成的 `RDD` 都是一样, 只是对于传入函数的处理不同, `map` 是 `collect.map( function )` 而 `flatMap` 是 `collect.flatMap( function )
```

从侧面印证了, 其实 `Spark` 中的 `flatMap` 和 `Scala` 基础中的 `flatMap` 其实是一样的

**总结**

**如何生成 `RDD` ?**

生成 `RDD` 的常见方式有三种

- 从本地集合创建
- 从外部数据集创建
- 从其它 `RDD` 衍生

通过外部数据集创建 `RDD`, 是通过 `Hadoop` 或者其它外部数据源的 `SDK` 来进行数据读取, 同时如果外部数据源是有分片的话, `RDD` 会将分区与其分片进行对照

通过其它 `RDD` 衍生的话, 其实本质上就是通过不同的算子生成不同的 `RDD` 的子类对象, 从而控制 `compute` 函数的行为来实现算子功能

**生成哪些 `RDD` ?**

不同的算子生成不同的 `RDD`, 生成 `RDD` 的类型取决于算子, 例如 `map` 和 `flatMap` 都会生成 `RDD` 的子类 `MapPartitions` 的对象

**如何计算 `RDD` 中的数据 ?**

虽然前面我们提到过 `RDD` 是偏向计算的, 但是其实 `RDD` 还只是表示数据, 纵观 `RDD` 的五大属性中有三个是必须的, 分别如下

- `Partitions List` 分区列表
- `Compute function` 计算函数
- `Dependencies` 依赖

虽然计算函数是和计算有关的, 但是只有调用了这个函数才会进行计算, `RDD` 显然不会自己调用自己的 `Compute` 函数, 一定是由外部调用的, 所以 `RDD` 更多的意义是用于表示数据集以及其来源, 和针对于数据的计算

所以如何计算 `RDD` 中的数据呢? 一定是通过其它的组件来计算的, 而计算的规则, 由 `RDD` 中的 `Compute` 函数来指定, 不同类型的 `RDD` 子类有不同的 `Compute` 函数

### 4 rdd之间的依赖关系

这个关系不是指rdd之间的关系,而是分区之间的关系   map 和flatMap这两个算子的关系是1 对1 

有shuffle的不是一对一  而是多对一

#### 4.1 什么是依赖

![img](/images/sparkyl/dcy1.png)

- 什么是关系(依赖关系) ?

  从算子视角上来看, `splitRDD` 通过 `map` 算子得到了 `tupleRDD`, 所以 `splitRDD` 和 `tupleRDD` 之间的关系是 `map`

  但是仅仅这样说, 会不够全面, 从细节上来看, `RDD` 只是数据和关于数据的计算, 而具体执行这种计算得出结果的是一个神秘的其它组件, 所以, 这两个 `RDD` 的关系可以表示为 `splitRDD` 的数据通过 `map` 操作, 被传入 `tupleRDD`, 这是它们之间更细化的关系

  但是 `RDD` 这个概念本身并不是数据容器, 数据真正应该存放的地方是 `RDD` 的分区, 所以如果把视角放在数据这一层面上的话, 直接讲这两个 RDD 之间有关系是不科学的, 应该从这两个 RDD 的分区之间的关系来讨论它们之间的关系

- 那这些分区之间是什么关系?

  如果仅仅说 `splitRDD` 和 `tupleRDD` 之间的话, 那它们的分区之间就是一对一的关系

  但是 `tupleRDD` 到 `reduceRDD` 呢? `tupleRDD` 通过算子 `reduceByKey` 生成 `reduceRDD`, 而这个算子是一个 `Shuffle`操作, `Shuffle` 操作的两个 `RDD` 的分区之间并不是一对一, `reduceByKey` 的一个分区对应 `tupleRDD` 的多个分区

**`reduceByKey` 算子会生成 `ShuffledRDD`**

`reduceByKey` 是由算子 `combineByKey` 来实现的, `combineByKey` 内部会创建 `ShuffledRDD` 返回, 而整个 `reduceByKey` 操作大致如下过程

![img](/images/sparkyl/dcy2.png)

去掉两个 `reducer` 端的分区, 只留下一个的话, 如下

![img](/images/sparkyl/dcy3.png)

所以, 对于 `reduceByKey` 这个 `Shuffle` 操作来说, `reducer` 端的一个分区, 会从多个 `mapper` 端的分区拿取数据, 是一个多对一的关系

至此为止, 出现了两种分区见的关系了, 一种是一对一, 一种是多对一

整体流程:

![img](/images/sparkyl/dcy4.png)

#### 4.2 rdd之间依赖详解

窄依赖  无shuffle的为窄依赖  

宽依赖  有shuffle的为宽依赖    

判断一个是什么依赖  只需看返回值是NarrowDependency 还是ShuffleDependency

**如果有shuffle是没办法吧两个rdd的分区放在同一个流水线工作的**

4.2.1 窄依赖 NarrowDependency

假如 `rddB = rddA.transform(…)`, 如果 `rddB` 中一个分区依赖 `rddA` 也就是其父 `RDD` 的少量分区, 这种 `RDD` 之间的依赖关系称之为窄依赖

换句话说, 子 RDD 的每个分区依赖父 RDD 的少量个数的分区, 这种依赖关系称之为窄依赖

例如:

~~~scala
lass ZDemo {
  @Test
  def zD(): Unit ={
    val conf = new SparkConf().setMaster("local[3]").setAppName("cache")
    val context =new  SparkContext(conf)
    val rdd1 = context.parallelize(Seq(1,2,3))
    val rdd2 = context.parallelize(Seq("a","b"))
    //求笛卡尔积  是一个窄依赖
    rdd1.cartesian(rdd2).collect().foreach(println(_))
  }
}
~~~

- 上述代码的 `cartesian` 是求得两个集合的笛卡尔积
- 上述代码的运行结果是 `rddA` 中每个元素和 `rddB` 中的所有元素结合, 最终的结果数量是两个 `RDD` 数量之和
- `rddC` 有两个父 `RDD`, 分别为 `rddA` 和 `rddB`

对于 `cartesian` 来说, 依赖关系如下

![img](/images/sparkyl/dcy5.png)
  上述图形中清晰展示如下现象

1. `rddC` 中的分区数量是两个父 `RDD` 的分区数量之乘积
2. `rddA` 中每个分区对应 `rddC` 中的两个分区 (因为 `rddB` 中有两个分区), `rddB` 中的每个分区对应 `rddC` 中的三个分区 (因为 `rddA` 有三个分区)

它们之间是窄依赖, 事实上在 `cartesian` 中也是 `NarrowDependency` 这个所有窄依赖的父类的唯一一次直接使用, 为什么呢?

**因为所有的分区之间是拷贝关系, 并不是 Shuffle 关系**

- `rddC` 中的每个分区并不是依赖多个父 `RDD` 中的多个分区
- `rddC` 中每个分区的数量来自一个父 `RDD` 分区中的所有数据, 是一个 `FullDependence`, 所以数据可以直接从父 `RDD` 流动到子 `RDD`
- 不存在一个父 `RDD` 中一部分数据分发过去, 另一部分分发给其它的 `RDD`

##### 4.2.2 宽依赖

并没有所谓的宽依赖, 宽依赖应该称作为 `ShuffleDependency`

在 `ShuffleDependency` 的类声明上如下写到

```text
Represents a dependency on the output of a shuffle stage.
```

上面非常清楚的说道, 宽依赖就是 `Shuffle` 中的依赖关系, 换句话说, 只有 `Shuffle` 产生的地方才是宽依赖

那么宽窄依赖的判断依据就非常简单明确了, **是否有 Shuffle ?**

举个 `reduceByKey` 的例子, `rddB = rddA.reduceByKey( (curr, agg) ⇒ curr + agg )` 会产生如下的依赖关系

![img](/images/sparkyl/dcy6.png)

分区函数:  key.hashCode% rddb的分区数 = 1 或者2

Hadoop.分区 =1  spark.分区=2

- `rddB` 的每个分区都几乎依赖 `rddA` 的所有分区
- 对于 `rddA` 中的一个分区来说, 其将一部分分发给 `rddB` 的 `p1`, 另外一部分分发给 `rddB` 的 `p2`, 这不是数据流动, 而是分发

##### 4.2.3 宽窄依赖分辨

~~~
首先  看是不是一对一  如是  就是窄依赖
若是多对一  则看是否有数据分发  就是是否有shuffle  
若是看不出来  直接看源码  的getDependencies 的返回值是什么是NarrowDependency为窄依赖  shuffleDependency为宽依赖  若是没有getDependencies方法 就看他的父类
~~~

##### 4.2.4 常见窄依赖 类型

NarrowDependency和shuffleDependency的共同父类为Dependency

**一对一窄依赖**

其实 `RDD` 中默认的是 `OneToOneDependency`, 后被不同的 `RDD` 子类指定为其它的依赖类型, 常见的一对一依赖是 `map` 算子所产生的依赖, 例如 `rddB = rddA.map(…)`
  每个分区之间一一对应, 所以叫做一对一窄依赖

**Range 窄依赖**

`Range` 窄依赖其实也是一对一窄依赖, 但是保留了中间的分隔信息, 可以通过某个分区获取其父分区, 目前只有一个算子生成这种窄依赖, 就是 `union` 算子, 例如 `rddC = rddA.union(rddB)`

![img](/images/sparkyl/dcy7.png)

- `rddC` 其实就是 `rddA` 拼接 `rddB` 生成的, 所以 `rddC` 的 `p5` 和 `p6` 就是 `rddB` 的 `p1` 和 `p2`
- 所以需要有方式获取到 `rddC` 的 `p5` 其父分区是谁, 于是就需要记录一下边界, 其它部分和一对一窄依赖一样

**多对一窄依赖**

多对一窄依赖其图形和 `Shuffle` 依赖非常相似, 所以在遇到的时候, 要注意其 `RDD` 之间是否有 `Shuffle` 过程, 比较容易让人困惑, 常见的多对一依赖就是重分区算子 `coalesce`, 例如 `rddB = rddA.coalesce(2, shuffle = false)`, 但同时也要注意, 如果 `shuffle = true` 那就是完全不同的情况了

![img](/images/sparkyl/dcy8.png)

因为没有 `Shuffle`, 所以这是一个窄依赖

**再谈宽窄依赖的区别**

宽窄依赖的区别非常重要, 因为涉及了一件非常重要的事情: **如何计算 RDD ?**

宽窄以来的核心区别是: **窄依赖的 RDD 可以放在一个 Task 中运行**

## 5 物理图执行详解

### 1物理图作用

**问题一: 物理图的意义是什么?**

物理图解决的其实就是 `RDD` 流程生成以后, 如何计算和运行的问题, 也就是如何把 RDD 放在集群中执行的问题

![img](/images/sparkyl/dcw1.png)

**问题二: 如果要确定如何运行的问题, 则需要先确定集群中有什么组件**

- 首先集群中物理元件就是一台一台的机器
- 其次这些机器上跑的守护进程有两种: `Master`, `Worker`
  - 每个守护进程其实就代表了一台机器, 代表这台机器的角色, 代表这台机器和外界通信
  - 例如我们常说一台机器是 `Master`, 其含义是这台机器中运行了一个 `Master` 守护进程, 如果一台机器运行了 `Master`的同时又运行了 `Worker`, 则说这台机器是 `Master` 也可以, 说它是 `Worker` 也行
- 真正能运行 `RDD` 的组件是: `Executor`, 也就是说其实 `RDD` 最终是运行在 `Executor` 中的, 也就是说, 无论是 `Master` 还是 `Worker` 其实都是用于管理 `Executor` 和调度程序的

结论是 `RDD` 一定在 `Executor` 中计算, 而 `Master` 和 `Worker` 负责调度和管理 `Executor`

**问题三: 物理图的生成需要考虑什么问题?**

- 要计算 `RDD`, 不仅要计算, 还要很快的计算 → 优化性能
- 要考虑容错, 容错的常见手段是缓存 → `RDD` 要可以缓存

结论是在生成物理图的时候, 不仅要考虑效率问题, 还要考虑一种更合适的方式, 让 `RDD` 运行的更好

### **2谁来计算 RDD ?**

- 问题一: RDD 是什么, 用来做什么 ?

  回顾一下 `RDD` 的五个属性`A list of partitions``A function for computing each split``A list of dependencies on other RDDs``Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)``Optionally, a list of preferred locations to compute each split on (e.g. block locations for an HDFS file)`简单的说就是: 分区列表, 计算函数, 依赖关系, 分区函数, 最佳位置分区列表, 分区函数, 最佳位置, 这三个属性其实说的就是数据集在哪, 在哪更合适, 如何分区计算函数和依赖关系, 这两个属性其实说的是数据集从哪来所以结论是 `RDD` 是一个数据集的表示, 不仅表示了数据集, 还表示了这个数据集从哪来, 如何计算但是问题是, 谁来计算 ? 如果为一台汽车设计了一个设计图, 那么设计图自己生产汽车吗 ?

- 问题二: 谁来计算 ?

  前面我们明确了两件事, `RDD` 在哪被计算? 在 `Executor` 中. `RDD` 是什么? 是一个数据集以及其如何计算的图纸.直接使用 `Executor` 也是不合适的, 因为一个计算的执行总是需要一个容器, 例如 `JVM` 是一个进程, 只有进程中才能有线程, 所以这个计算 `RDD` 的线程应该运行在一个进程中, 这个进程就是 `Exeutor`, `Executor` 有如下两个职责

  和 `Driver` 保持交互从而认领属于自己的任务![img](/images/sparkyl/dcw2.png)

  接受任务后, 运行任务

  ![img](/images/sparkyl/dcw4.png)

所以, 应该由一个线程来执行 `RDD` 的计算任务, 而 `Executor` 作为执行这个任务的容器, 也就是一个进程, 用于创建和执行线程, 这个执行具体计算任务的线程叫做 `Task`

### 3 Task 该如何设计

第一个想法是每个 `RDD` 都由一个 `Task` 来计算 第二个想法是一整个逻辑执行图中所有的 `RDD` 都由一组 `Task` 来执行 第三个想法是分阶段执行

第一个想法: 为每个 RDD 的分区设置一组 Task

![img](/images/sparkyl/dcw2.png)

大概就是每个 `RDD` 都有三个 `Task`, 每个 `Task` 对应一个 `RDD` 的分区, 执行一个分区的数据的计算

但是这么做有一个非常难以解决的问题, 就是数据存储的问题, 例如 `Task 1, 4, 7, 10, 13, 16` 在同一个流程上, 但是这些 `Task` 之间需要交换数据, 因为这些 `Task` 可能被调度到不同的机器上上, 所以 `Task1` 执行完了数据以后需要暂存, 后交给 `Task4` 来获取

这只是一个简单的逻辑图, 如果是一个复杂的逻辑图, 会有什么表现? 要存储多少数据? 无论是放在磁盘还是放在内存中, 是不是都是一种极大的负担?

第二个想法: 让数据流动

很自然的, 第一个想法的问题是数据需要存储和交换, 那不存储不就好了吗? 对, 可以让数据流动起来

第一个要解决的问题就是, 要为数据创建管道(`Pipeline`), 有了管道, 就可以流动

![img](/images/sparkyl/dcw5.png)

简单来说, 就是为所有的 `RDD` 有关联的分区使用同一个 `Task`, 但是就没问题了吗? 请关注红框部分

![img](/images/sparkyl/dcw6.png)

这两个 `RDD` 之间是 `Shuffle` 关系, 也就是说, 右边的 `RDD` 的一个分区可能依赖左边 `RDD` 的所有分区, 这样的话, 数据在这个地方流不动了, 怎么办?

第三个想法: 划分阶段

既然在 `Shuffle` 处数据流不动了, 那就可以在这个地方中断一下, 后面 `Stage` 部分详解

### 4**如何划分阶段 ?**

为了减少执行任务, 减少数据暂存和交换的机会, 所以需要创建管道, 让数据沿着管道流动, 其实也就是原先每个 `RDD` 都有一组 `Task`, 现在改为所有的 `RDD` 共用一组 `Task`, 但是也有问题, 问题如下

![img](/images/sparkyl/dcw7.png)

就是说, 在 `Shuffle` 处, 必须断开管道, 进行数据交换, 交换过后, 继续流动, 所以整个流程可以变为如下样子

![img](/images/sparkyl/dcw8.png)

把 `Task` 断开成两个部分, `Task4` 可以从 `Task 1, 2, 3` 中获取数据, 后 `Task4` 又作为管道, 继续让数据在其中流动

但是还有一个问题, 说断开就直接断开吗? 不用打个招呼的呀? 这个断开即没有道理, 也没有规则, 所以可以为这个断开增加一个概念叫做阶段, 按照阶段断开, 阶段的英文叫做 `Stage`, 如下

![img](/images/sparkyl/dcw9.png)

所以划分阶段的本身就是设置断开点的规则, 那么该如何划分阶段呢?

1. 第一步, 从最后一个 `RDD`, 也就是逻辑图中最右边的 `RDD` 开始, 向前滑动 `Stage` 的范围, 为 `Stage0`
2. 第二步, 遇到 `ShuffleDependency` 断开 `Stage`, 从下一个 `RDD` 开始创建新的 `Stage`, 为 `Stage1`
3. 第三步, 新的 `Stage` 按照同样的规则继续滑动, 直到包裹所有的 `RDD`

总结来看, 就是针对于宽窄依赖来判断, 一个 `Stage` 中只有窄依赖, 因为只有窄依赖才能形成数据的 `Pipeline`.

如果要进行 `Shuffle` 的话, 数据是流不过去的, 必须要拷贝和拉取. 所以遇到 `RDD` 宽依赖的两个 `RDD` 时, 要切断这两个 `RDD` 的 `Stage`.

这样一个 RDD 依赖的链条, 我们称之为 RDD 的血统, 其中有宽依赖也有窄依赖

### 5 数据流动

~~~scala
val sc = ...

val textRDD = sc.parallelize(Seq("Hadoop Spark", "Hadoop Flume", "Spark Sqoop"))
val splitRDD = textRDD.flatMap(_.split(" "))
val tupleRDD = splitRDD.map((_, 1))
val reduceRDD = tupleRDD.reduceByKey(_ + _)
val strRDD = reduceRDD.map(item => s"${item._1}, ${item._2}")

strRDD.collect.foreach(item => println(item))
~~~

上述代码是这个章节我们一直使用的代码流程, 如下是其完整的逻辑执行图

![img](/images/sparkyl/dcw10.png)

如果放在集群中运行, 通过 `WebUI` 可以查看到如下 `DAG` 结构

![img](/images/sparkyl/dcw11.png)
  Step 1: 从 `ResultStage` 开始执行

最接近 `Result` 部分的 `Stage id` 为 0, 这个 `Stage` 被称之为 `ResultStage`

由代码可以知道, 最终调用 `Action` 促使整个流程执行的是最后一个 `RDD`, `strRDD.collect`, 所以当执行 `RDD` 的计算时候, 先计算的也是这个 `RDD`

Step 2: `RDD` 之间是有关联的

前面已经知道, 最后一个 `RDD` 先得到执行机会, 先从这个 `RDD` 开始执行, 但是这个 `RDD` 中有数据吗 ? 如果没有数据, 它的计算是什么? 它的计算是从父 `RDD` 中获取数据, 并执行传入的算子的函数

简单来说, 从产生 `Result` 的地方开始计算, 但是其 `RDD` 中是没数据的, 所以会找到父 `RDD` 来要数据, 父 `RDD` 也没有数据, 继续向上要, 所以, 计算从 `Result` 处调用, 但是从整个逻辑图中的最左边 `RDD` 开始, 类似一个递归的过程

![img](/images/sparkyl/dcw12.png)

### 6 调度过程

#### 6.1 逻辑图调度

是什么 怎么生成 具体怎么生成

```scala
val textRDD = sc.parallelize(Seq("Hadoop Spark", "Hadoop Flume", "Spark Sqoop"))
val splitRDD = textRDD.flatMap(_.split(" "))
val tupleRDD = splitRDD.map((_, 1))
val reduceRDD = tupleRDD.reduceByKey(_ + _)
val strRDD = reduceRDD.map(item => s"${item._1}, ${item._2}")
```

- 逻辑图如何生成

  上述代码在 `Spark Application` 的 `main` 方法中执行, 而 `Spark Application` 在 `Driver` 中执行, 所以上述代码在 `Driver` 中被执行, 那么这段代码执行的结果是什么呢?一段 `Scala` 代码的执行结果就是最后一行的执行结果, 所以上述的代码, 从逻辑上执行结果就是最后一个 `RDD`, 最后一个 `RDD`也可以认为就是逻辑执行图, 为什么呢?例如 `rdd2 = rdd1.map(…)` 中, 其实本质上 `rdd2` 是一个类型为 `MapPartitionsRDD` 的对象, 而创建这个对象的时候, 会通过构造函数传入当前 `RDD` 对象, 也就是父 `RDD`, 也就是调用 `map` 算子的 `rdd1`, `rdd1` 是 `rdd2` 的父 `RDD`

  ![img](/images/sparkyl/dcw13.png)

  一个 `RDD` 依赖另外一个 `RDD`, 这个 `RDD` 又依赖另外的 `RDD`, 一个 `RDD` 可以通过 `getDependency` 获得其父 `RDD`, 这种环环相扣的关系, 最终从最后一个 `RDD` 就可以推演出前面所有的 `RDD`

- 逻辑图是什么, 干啥用

  逻辑图其实本质上描述的就是数据的计算过程, 数据从哪来, 经过什么样的计算, 得到什么样的结果, 再执行什么计算, 得到什么结果

可是数据的计算是描述好了, 这种计算该如何执行呢?

#### 6.2 物理图

**DAGScheduler  1 Stage中有多个task  2 每个task计算一个分区  job就是一个求值过程**

数据的计算表示好了, 该正式执行了, 但是如何执行? 如何执行更快更好更酷? 就需要为其执行做一个规划, 所以需要生成物理执行图

```scala
strRDD.collect.foreach(item => println(item))
```

上述代码其实就是最后的一个 `RDD` 调用了 `Action` 方法, 调用 `Action` 方法的时候, 会请求一个叫做 `DAGScheduler` 的组件, `DAGScheduler` 会创建用于执行 `RDD` 的 `Stage` 和 `Task`

`DAGScheduler` 是一个由 `SparkContext` 创建, 运行在 `Driver` 上的组件, 其作用就是将由 `RDD` 构建出来的逻辑计划, 构建成为由真正在集群中运行的 `Task` 组成的物理执行计划, `DAGScheduler` 主要做如下三件事

1. 帮助每个 `Job` 计算 `DAG` 并发给 `TaskSheduler` 调度
2. 确定每个 `Task` 的最佳位置
3. 跟踪 `RDD` 的缓存状态, 避免重新计算

从字面意思上来看, `DAGScheduler` 是调度 `DAG` 去运行的, `DAG` 被称作为有向无环图, 其实可以将 `DAG` 理解为就是 `RDD` 的逻辑图, 其呈现两个特点: `RDD` 的计算是有方向的, `RDD` 的计算是无环的, 所以 `DAGScheduler` 也可以称之为 `RDD Scheduler`, 但是真正运行在集群中的并不是 `RDD`, 而是 `Task` 和 `Stage`, `DAGScheduler` 负责这种转换

#### 6.3 job简介

**`job` 什么时候生成 ?**

当一个 `RDD` 调用了 `Action` 算子的时候, 在 `Action` 算子内部, 会使用 `sc.runJob()` 调用 `SparkContext` 中的 `runJob`方法, 这个方法又会调用 `DAGScheduler` 中的 `runJob`, 后在 `DAGScheduler` 中使用消息驱动的形式创建 `Job`

简而言之, `Job` 在 `RDD` 调用 `Action` 算子的时候生成, 而且调用一次 `Action` 算子, 就会生成一个 `Job`, 如果一个 `SparkApplication` 中调用了多次 `Action` 算子, 会生成多个 `Job` 串行执行, 每个 `Job` 独立运作, 被独立调度, 所以 `RDD` 的计算也会被执行多次

**`Job` 是什么 ?**

如果要将 `Spark` 的程序调度到集群中运行, `Job` 是粒度最大的单位, 调度以 `Job` 为最大单位, 将 `Job` 拆分为 `Stage` 和 `Task` 去调度分发和运行, 一个 `Job` 就是一个 `Spark` 程序从 `读取 → 计算 → 运行` 的过程

一个 `Spark Application` 可以包含多个 `Job`, 这些 `Job` 之间是串行的, 也就是第二个 `Job` 需要等待第一个 `Job` 的执行结束后才会开始执行

#### 6.4 job 与Stage的关系

**一个job有多个Stage   stage之间是串行的   左侧的接近起始Stage的最先执行,从左向右执行**

`Job` 是一个最大的调度单位, 也就是说 `DAGScheduler` 会首先创建一个 `Job` 的相关信息, 后去调度 `Job`, 但是没办法直接调度 `Job`, 比如说现在要做一盘手撕包菜, 不可能直接去炒一整颗包菜, 要切好撕碎, 再去炒

**为什么** `Job` **需要切分 ?**

![img](/images/sparkyl/dcw14.png)

- 因为 `Job` 的含义是对整个 `RDD` 血统求值, 但是 `RDD` 之间可能会有一些宽依赖

- 如果遇到宽依赖的话, 两个 `RDD` 之间需要进行数据拉取和复制

  如果要进行拉取和复制的话, 那么一个 `RDD` 就必须等待它所依赖的 `RDD` 所有分区先计算完成, 然后再进行拉取

- 由上得知, 一个 `Job` 是无法计算完整个 `RDD` 血统的

**如何切分 ?**

创建一个 `Stage`, 从后向前回溯 `RDD`, 遇到 `Shuffle` 依赖就结束 `Stage`, 后创建新的 `Stage` 继续回溯. 这个过程上面已经详细的讲解过, 但是问题是切分以后如何执行呢, 从后向前还是从前向后, 是串行执行多个 `Stage`, 还是并行执行多个 `Stage`

**问题一: 执行顺序**

在图中, `Stage 0` 的计算需要依赖 `Stage 1` 的数据, 因为 `reduceRDD` 中一个分区可能需要多个 `tupleRDD` 分区的数据, 所以 `tupleRDD` 必须先计算完, 所以, 应该在逻辑图中自左向右执行 `Stage`

**问题二: 串行还是并行**

还是同样的原因, `Stage 0` 如果想计算, `Stage 1` 必须先计算完, 因为 `Stage 0` 中每个分区都依赖 `Stage 1` 中的所有分区, 所以 `Stage 1` 不仅需要先执行, 而且 `Stage 1` 执行完之前 `Stage 0` 无法执行, 它们只能串行执行

**总结**

- 一个 `Stage` 就是物理执行计划中的一个步骤, 一个 `Spark Job` 就是划分到不同 `Stage` 的计算过程
- `Stage` 之间的边界由 `Shuffle` 操作来确定
  - `Stage` 内的 `RDD` 之间都是窄依赖, 可以放在一个管道中执行
  - 而 `Shuffle` 后的 `Stage` 需要等待前面 `Stage` 的执行

**`Stage` 有两种**

- `ShuffMapStage`, 其中存放窄依赖的 `RDD`
- `ResultStage`, 每个 `Job` 只有一个, 负责计算结果, 一个 `ResultStage` 执行完成标志着整个 `Job` 执行完毕

#### 6.5 `Stage` **和** `Task` **的关系**

**rdd本身不存储数据,数据存储在分区中**

**一个task对应一个分区,一个Stage 中有几个task(分区) 由Stage中的最后一个rdd决定 一个stage就是一个TaskSet  一个TaskSet就是一组task**

![img](/images/sparkyl/dcw15.png)

前面我们说到 `Job` 无法直接执行, 需要先划分为多个 `Stage`, 去执行 `Stage`, 那么 `Stage` 可以直接执行吗?

- 第一点: `Stage` 中的 `RDD` 之间是窄依赖

  因为 `Stage` 中的所有 `RDD` 之间都是窄依赖, 窄依赖 `RDD` 理论上是可以放在同一个 `Pipeline(管道, 流水线)` 中执行的, 似乎可以直接调度 `Stage` 了? 其实不行, 看第二点

- 第二点: 别忘了 `RDD` 还有分区

  一个 `RDD` 只是一个概念, 而真正存放和处理数据时, 都是以分区作为单位的

  `Stage` 对应的是多个整体上的 `RDD`, 而真正的运行是需要针对 `RDD` 的分区来进行的

- 第三点: 一个 `Task` 对应一个 `RDD` 的分区

  一个比 `Stage` 粒度更细的单元叫做 `Task`, `Stage` 是由 `Task` 组成的, 之所以有 `Task` 这个概念, 是因为 `Stage` 针对整个 `RDD`, 而计算的时候, 要针对 `RDD` 的分区

  假设一个 `Stage` 中有 10 个 `RDD`, 这些 `RDD` 中的分区各不相同, 但是分区最多的 `RDD` 有 30 个分区, 而且很显然, 它们之间是窄依赖关系

  那么, 这个 `Stage` 中应该有多少 `Task` 呢? 应该有 30 个 `Task`, 因为一个 `Task` 计算一个 `RDD` 的分区. 这个 `Stage` 至多有 30 个分区需要计算

- 总结

  - 一个 `Stage` 就是一组并行的 `Task` 集合
  - Task 是 Spark 中最小的独立执行单元, 其作用是处理一个 RDD 分区
  - 一个 Task 只可能存在于一个 Stage 中, 并且只能计算一个 RDD 的分区

#### 6.6 **TaskSet**

梳理一下这几个概念, `Job > Stage > Task`, `Job 中包含 Stage 中包含 Task`

而 `Stage` 中经常会有一组 `Task` 需要同时执行, 所以针对于每一个 `Task` 来进行调度太过繁琐, 而且没有意义, 所以每个 `Stage`中的 `Task` 们会被收集起来, 放入一个 `TaskSet` 集合中

- 一个 `Stage` 有一个 `TaskSet`
- `TaskSet` 中 `Task` 的个数由 `Stage` 中的最大分区数决定

#### 6.7 整体流程

![img](/images/sparkyl/dcw16.png)

#### 6.8 shuffle过程

##### 6.8.1`Shuffle` **过程的组件结构**

从整体视角上来看, `Shuffle` 发生在两个 `Stage` 之间, 一个 `Stage` 把数据计算好, 整理好, 等待另外一个 `Stage` 来拉取

![img](/images/sparkyl/dcw17.png)

放大视角, 会发现, 其实 `Shuffle` 发生在 `Task` 之间, 一个 `Task` 把数据整理好, 等待 `Reducer` 端的 `Task` 来拉取

![img](/images/sparkyl/dcw18.png)

如果更细化一下, `Task` 之间如何进行数据拷贝的呢? 其实就是一方 `Task` 把文件生成好, 然后另一方 `Task` 来拉取

![img](/images/sparkyl/dcw19.png)

现在是一个 `Reducer` 的情况, 如果有多个 `Reducer` 呢? 如果有多个 `Reducer` 的话, 就可以在每个 `Mapper` 为所有的 `Reducer` 生成各一个文件, 这种叫做 `Hash base shuffle`, 这种 `Shuffle` 的方式问题大家也知道, 就是生成中间文件过多, 而且生成文件的话需要缓冲区, 占用内存过大

![img](/images/sparkyl/dcw20.png)

那么可以把这些文件合并起来, 生成一个文件返回, 这种 `Shuffle` 方式叫做 `Sort base shuffle`, 每个 `Reducer` 去文件的不同位置拿取数据

![img](/images/sparkyl/dcw21.png)
  如果再细化一下, 把参与这件事的组件也放置进去, 就会是如下这样

![img](/images/sparkyl/dcw22.png)

**有哪些 `ShuffleWriter` ?**

大致上有三个 `ShufflWriter`, `Spark` 会按照一定的规则去使用这三种不同的 `Writer`

- `BypassMergeSortShuffleWriter`

  这种 `Shuffle Writer` 也依然有 `Hash base shuffle` 的问题, 它会在每一个 `Mapper` 端对所有的 `Reducer` 生成一个文件, 然后再合并这个文件生成一个统一的输出文件, 这个过程中依然是有很多文件产生的, 所以只适合在小量数据的场景下使用

  `Spark` 有考虑去掉这种 `Writer`, 但是因为结构中有一些依赖, 所以一直没去掉

  当 `Reducer` 个数小于 `spark.shuffle.sort.bypassMergeThreshold`, 并且没有 `Mapper` 端聚合的时候启用这种方式

- `SortShuffleWriter`

  这种 `ShuffleWriter` 写文件的方式非常像 `MapReduce` 了, 后面详说

  当其它两种 `Shuffle` 不符合开启条件时, 这种 `Shuffle` 方式是默认的

- `UnsafeShuffleWriter`

  这种 `ShuffWriter` 会将数据序列化, 然后放入缓冲区进行排序, 排序结束后 `Spill` 到磁盘, 最终合并 `Spill` 文件为一个大文件, 同时在进行内存存储的时候使用了 `Java` 得 `Unsafe API`, 也就是使用堆外内存, 是钨丝计划的一部分

  也不是很常用, 只有在满足如下三个条件时候才会启用

  - 序列化器序列化后的数据, 必须支持排序
  - 没有 `Mapper` 端的聚合
  - `Reducer` 的个数不能超过支持的上限 (2 ^ 24)

**`SortShuffleWriter` 的执行过程**

![img](/images/sparkyl/dcw23.png)

整个 `SortShuffleWriter` 如上述所说, 大致有如下几步

1. 首先 `SortShuffleWriter` 在 `write` 方法中回去写文件, 这个方法中创建了 `ExternalSorter`
2. `write` 中将数据 `insertAll` 到 `ExternalSorter` 中
3. 在 `ExternalSorter` 中排序
   1. 如果要聚合, 放入 `AppendOnlyMap` 中, 如果不聚合, 放入 `PartitionedPairBuffer` 中
   2. 在数据结构中进行排序, 排序过程中如果内存数据大于阈值则溢写到磁盘
4. 使用 `ExternalSorter` 的 `writePartitionedFile` 写入输入文件
   1. 将所有的溢写文件通过类似 `MergeSort` 的算法合并
   2. 将数据写入最终的目标文件中

### 7 闭包    RDD 的分布式共享变量

**什么是闭包**

闭包是一个必须要理解, 但是又不太好理解的知识点, 先看一个小例子

```java
@Test
def test(): Unit = {
  val areaFunction = closure()
  val area = areaFunction(2)
  println(area)
}

def closure(): Int => Double = {
  val factor = 3.14
  val areaFunction = (r: Int) => math.pow(r, 2) * factor
  areaFunction
}
```

上述例子中, `closure`方法返回的一个函数的引用, 其实就是一个闭包, 闭包本质上就是一个封闭的作用域, 要理解闭包, 是一定要和作用域联系起来的.

- 能否在 `test` 方法中访问 `closure` 定义的变量?

  ```scala
  @Test
  def test(): Unit = {
    println(factor)
  }

  def closure(): Int => Double = {
    val factor = 3.14
  }
  ```


- 有没有什么间接的方式?

  ~~~
  @Test
  def test(): Unit = {
    val areaFunction = closure()
    areaFunction()
  }

  def closure(): () => Unit = {
    val factor = 3.14
    val areaFunction = () => println(factor)
    areaFunction
  }
  ~~~


- 什么是闭包?

  ~~~~scala
  val areaFunction = closure()
  areaFunction()
  ~~~~

  通过 `closure` 返回的函数 `areaFunction` 就是一个闭包, 其函数内部的作用域并不是 `test` 函数的作用域, 这种连带作用域一起打包的方式, 我们称之为闭包, 在 Scala 中**Scala 中的闭包本质上就是一个对象, 是 FunctionX 的实例**

**分发闭包**

​	

~~~scala
class MyClass {
  val field = "Hello"

  def doStuff(rdd: RDD[String]): RDD[String] = {
    rdd.map(x => field + x)
  }
}       
// rdd.map(x => field + x)引用了MyClass 对象中的一个成员变量 说明其可以访问MyClass这个类的作用域
//也是一个闭包  封闭的是MyClass这个作用域
~~~

![img](/images/sparkyl/bb.jpg)

**总结**

1. 闭包就是一个封闭的作用域, 也是一个对象
2. Spark 算子所接受的函数, 本质上是一个闭包, 因为其需要封闭作用域, 并且序列化自身和依赖, 分发到不同的节点中运行

#### 7.1 累加器

~~~scala
class ZDemo {
  var num= 0
  @Test
  def addErr(): Unit ={
    val conf = new SparkConf().setMaster("local[3]").setAppName("cache")
    val context =new  SparkContext(conf)
    context.parallelize(Seq(1,2,3,4,5,6),3)
      //闭包会报错
      .foreach(num += _)
    println(num)
  }
}

~~~

上面这段代码是一个非常错误的使用, 请不要仿照, 这段代码只是为了证明一些事情

先明确两件事, `var count = 0` 是在 Driver 中定义的, `foreach(count += _)` 这个算子以及传递进去的闭包运行在 Executor 中

这段代码整体想做的事情是累加一个变量, 但是这段代码的写法却做不到这件事, 原因也很简单, 因为具体的算子是闭包, 被分发给不同的节点运行, 所以这个闭包中累加的并不是 Driver 中的这个变量

**全局累加器**

Accumulators(累加器) 是一个只支持 `added`(添加) 的分布式变量, 可以在分布式环境下保持一致性, 并且能够做到高效的并发.

原生 Spark 支持数值型的累加器, 可以用于实现计数或者求和, 开发者也可以使用自定义累加器以实现更高级的需求

~~~
val config = new SparkConf().setAppName("ip_ana").setMaster("local[6]")
val sc = new SparkContext(config)

val counter = sc.longAccumulator("counter")

sc.parallelize(Seq(1, 2, 3, 4, 5))
  .foreach(counter.add(_))

// 运行结果: 15
println(counter.value)
~~~

注意点:

- Accumulator 是支持并发并行的, 在任何地方都可以通过 `add` 来修改数值, 无论是 Driver 还是 Executor
- 只能在 Driver 中才能调用 `value` 来获取数值

在 WebUI 中关于 Job 部分也可以看到 Accumulator 的信息, 以及其运行的情况

![img](/images/sparkyl/ljq.png)

**累计器件还有两个小特性**

 第一, 累加器能保证在 Spark 任务出现问题被重启的时候不会出现重复计算. 

第二, 累加器只有在 Action 执行的时候才会被触发.

~~~~scala
val config = new SparkConf().setAppName("ip_ana").setMaster("local[6]")
val sc = new SparkContext(config)

val counter = sc.longAccumulator("counter")

sc.parallelize(Seq(1, 2, 3, 4, 5))
  .map(counter.add(_)) // 这个地方不是 Action, 而是一个 Transformation

// 运行结果是 0
println(counter.value)
~~~~

**自定义累加器**

开发者可以通过自定义累加器来实现更多类型的累加器, 累加器的作用远远不只是累加, 比如可以实现一个累加器, 用于向里面添加一些运行信息

~~~~scala
//自定义累加器
  //参数 一个累加的值是什么类型String  一个返回值类型Set[String]
  class NumAccu extends AccumulatorV2[String, Set[String]] {

    private val nums: mutable.Set[String] = mutable.Set()

    //告诉spark框架 这个spark累加器是否是空的
    override def isZero: Boolean = {
      nums.isEmpty
    }

    //提供给spark框架一个拷贝的累加器
    override def copy(): AccumulatorV2[String, Set[String]] = {
      val accu = new NumAccu()
      nums.synchronized() {
        accu.nums ++= this.nums
      }
      accu
    }

    //帮助spark清理,累加器里的内容
    override def reset(): Unit = {
      nums.clear()
    }
    //外部传入要累加的内容 ,在这个方法中累加
    override def add(v: String): Unit = {
      nums += v
    }
    //累加器在累加的时候,可能每个分布式节点都有一个实例,在最后Driver进行合并,把所有的实例内容合并
    //起来,会调用这个方法
    override def merge(other: AccumulatorV2[String, Set[String]]): Unit = {
      nums ++= other.value
    }
    //就是上面的value
    //提供给外部累加结果
    //为何不可变 ,因为外部有可能再次进行修改,如是可变集合,外部修改会影响内部得值
    override def value: Set[String] = {
      nums.toSet    //不可变得set
    }
  }
  //测试自定义累加器
  @Test
  def accTest(): Unit ={
     val conf = new SparkConf().setMaster("local[3]").setAppName("cache")
    val context = new SparkContext(conf)
    val accu = new NumAccu()
    //注册给Spark
    context.register(accu,"acc")

    context.parallelize(Seq("1","2"))
      .foreach(it => accu.add(it))
    println(accu.value)
    context.stop()
  }

//外部这个类要继承Serializable 才可以否则可能报错
~~~~

注意点:

- 可以通过继承 `AccumulatorV2` 来创建新的累加器
- 有几个方法需要重写
  - reset 方法用于把累加器重置为 0
  - add 方法用于把其它值添加到累加器中
  - merge 方法用于指定如何合并其他的累加器
- `value` 需要返回一个不可变的集合, 因为不能因为外部的修改而影响自身的值

#### 7.2 广播变量

**广播变量的作用**

广播变量允许开发者将一个 `Read-Only` 的变量缓存到集群中每个节点中, 而不是传递给每一个 Task 一个副本.

- 集群中每个节点, 指的是一个机器
- 每一个 Task, 一个 Task 是一个 Stage 中的最小处理单元, 一个 Executor 中可以有多个 Stage, 每个 Stage 有多个 Task

所以在需要跨多个 Stage 的多个 Task 中使用相同数据的情况下, 广播特别的有用

![img](/images/sparkyl/sgb.png)

**广播变量的API**

| 方法名         | 描述                    |
| :---------- | :-------------------- |
| `id`        | 唯一标识                  |
| `value`     | 广播变量的值                |
| `unpersist` | 在 Executor 中异步的删除缓存副本 |
| `destroy`   | 销毁所有此广播变量所关联的数据和元数据   |
| `toString`  | 字符串表示                 |

使用广播变量的一般套路

可以通过如下方式创建广播变量

```scala
val b = sc.broadcast(1)
```

如果 Log 级别为 DEBUG 的时候, 会打印如下信息

```text
DEBUG BlockManager: Put block broadcast_0 locally took  430 ms
DEBUG BlockManager: Putting block broadcast_0 without replication took  431 ms
DEBUG BlockManager: Told master about block broadcast_0_piece0
DEBUG BlockManager: Put block broadcast_0_piece0 locally took  4 ms
DEBUG BlockManager: Putting block broadcast_0_piece0 without replication took  4 ms
```

创建后可以使用 `value` 获取数据

```scala
b.value
```

获取数据的时候会打印如下信息

```text
DEBUG BlockManager: Getting local block broadcast_0
DEBUG BlockManager: Level for block broadcast_0 is StorageLevel(disk, memory, deserialized, 1 replicas)
```

广播变量使用完了以后, 可以使用 `unpersist` 删除数据

```scala
b.unpersist
```

删除数据以后, 可以使用 `destroy` 销毁变量, 释放内存空间

```scala
b.destroy
```

销毁以后, 会打印如下信息

```text
DEBUG BlockManager: Removing broadcast 0
DEBUG BlockManager: Removing block broadcast_0_piece0
DEBUG BlockManager: Told master about block broadcast_0_piece0
DEBUG BlockManager: Removing block broadcast_0
```

使用 `value` 方法的注意点

方法签名 `value: T`

在 `value` 方法内部会确保使用获取数据的时候, 变量必须是可用状态, 所以必须在变量被 `destroy` 之前使用 `value` 方法, 如果使用 `value` 时变量已经失效, 则会爆出以下错误

```text
org.apache.spark.SparkException: Attempted to use Broadcast(0) after it was destroyed (destroy at <console>:27)
  at org.apache.spark.broadcast.Broadcast.assertValid(Broadcast.scala:144)
  at org.apache.spark.broadcast.Broadcast.value(Broadcast.scala:69)
  ... 48 elided
```

- 使用 `destroy` 方法的注意点

  方法签名 `destroy(): Unit``destroy` 方法会移除广播变量, 彻底销毁掉, 但是如果你试图多次 `destroy` 广播变量, 则会爆出以下错误

  ~~~
  org.apache.spark.SparkException: Attempted to use Broadcast(0) after it was destroyed (destroy at <console>:27)
    at org.apache.spark.broadcast.Broadcast.assertValid(Broadcast.scala:144)
    at org.apache.spark.broadcast.Broadcast.destroy(Broadcast.scala:107)
    at org.apache.spark.broadcast.Broadcast.destroy(Broadcast.scala:98)
    ... 48 elided
  ~~~

  广播变量的使用场景

  假设我们在某个算子中需要使用一个保存了项目和项目的网址关系的 `Map[String, String]` 静态集合, 如下

  ```scala
  val pws = Map("Apache Spark" -> "http://spark.apache.org/", "Scala" -> "http://www.scala-lang.org/")

  val websites = sc.parallelize(Seq("Apache Spark", "Scala")).map(pws).collect
  ```

  上面这段代码是没有问题的, 可以正常运行的, 但是非常的低效, 因为虽然可能 `pws` 已经存在于某个 `Executor` 中了, 但是在需要的时候还是会继续发往这个 `Executor`, 如果想要优化这段代码, 则需要尽可能的降低网络开销

  可以使用广播变量进行优化, 因为广播变量会缓存在集群中的机器中, 比 `Executor` 在逻辑上更 "大"

  ```scala
  val pwsB = sc.broadcast(pws)
  val websites = sc.parallelize(Seq("Apache Spark", "Scala")).map(pwsB.value).collect
  ```

  上面两段代码所做的事情其实是一样的, 但是当需要运行多个 `Executor` (以及多个 `Task`) 的时候, 后者的效率更高

**扩展**

正常情况下使用 Task 拉取数据的时候, 会将数据拷贝到 Executor 中多次, 但是使用广播变量的时候只会复制一份数据到 Executor 中, 所以在两种情况下特别适合使用广播变量

- 一个 Executor 中有多个 Task 的时候
- 一个变量比较大的时候

而且在 Spark 中还有一个约定俗称的做法, 当一个 RDD 很大并且还需要和另外一个 RDD 执行 `join` 的时候, 可以将较小的 RDD 广播出去, 然后使用大的 RDD 在算子 `map` 中直接 `join`, 从而实现在 Map 端 `join`

```scala
val acMap = sc.broadcast(myRDD.map { case (a,b,c,b) => (a, c) }.collectAsMap)
val otherMap = sc.broadcast(myOtherRDD.collectAsMap)

myBigRDD.map { case (a, b, c, d) =>
  (acMap.value.get(a).get, otherMap.value.get(c).get)
}.collect
```

一般情况下在这种场景下, 会广播 Map 类型的数据, 而不是数组, 因为这样容易使用 Key 找到对应的 Value 简化使用

**总结**

1. 广播变量用于将变量缓存在集群中的机器中, 避免机器内的 Executors 多次使用网络拉取数据
2. 广播变量的使用步骤: (1) 创建 (2) 在 Task 中获取值 (3) 销毁