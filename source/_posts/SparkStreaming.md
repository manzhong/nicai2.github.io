---
title: SparkStreaming
tags:
  - Spark
categories: Spark
encrypt: 
enc_pwd: 
abbrlink: 9963
date: 2018-05-01 15:44:27
summary_img:
---

# 一概述

## 1 流计算与批量计算

批量计算  数据已经存在, 一次性读取所有的数据进行批量处理  hdfs > spark sql > hdfs	

流计算   数据源源不断的进来, 经过处理后落地   设备>  kafka  > SparkStreaming  >  hbase等

## 2 流和批 架构

流和批都是有意义的, 有自己的应用场景, 那么如何结合流和批呢? 如何在同一个系统中使用这两种不同的解决方案呢?

**混合架构**

混合架构的名字叫做 `Lambda 架构`, 混合架构最大的特点就是将流式计算和批处理结合起来,后在进行查询的时候分别查询流系统和批系统, 最后将结果合并在一起

一般情况下 Lambda 架构分三层

- 批处理层: 批量写入, 批量读取
- 服务层: 分为两个部分, 一部分对应批处理层, 一部分对应速度层
- 速度层: 随机读取, 随即写入, 增量计算

- 优点

  - 兼顾优点, 在批处理层可以全量查询和分析, 在速度层可以查询最新的数据
  - 速度很快, 在大数据系统中, 想要快速的获取结果是非常困难的, 因为高吞吐量和快速返回结果往往很难兼得, 例如 `Impala` 和 `Hive`, `Hive` 能进行非常大规模的数据量的处理, `Impala` 能够快速的查询返回结果, 但是很少有一个系统能够兼得两点, `Lambda` 使用多种融合的手段从而实现

- 缺点

  `Lambda` 是一个非常反人类的设计, 因为我们需要在系统中不仅维护多套数据层, 还需要维护批处理和流式处理两套框架, 这非常困难, 一套都很难搞定, 两套带来的运维问题是是指数级提升的

**流失架构**

流式架构常见的叫做 `Kappa 结构`, 是 `Lambda 架构` 的一个变种, 其实本质上就是删掉了批处理

优点

- 非常简单
- 效率很高, 在存储系统的发展下, 很多存储系统已经即能快速查询又能批量查询了, 所以 `Kappa 架构` 在新时代还是非常够用的

问题:

丧失了一些 `Lambda` 的优秀特点

## 3 SparkStreaming

| 特点                                       | 说明                                       |
| :--------------------------------------- | :--------------------------------------- |
| `Spark Streaming` 是 `Spark Core API` 的扩展 | `Spark Streaming` 具有类似 `RDD` 的 `API`, 易于使用, 并可和现有系统共用相似代码,,,,一个非常重要的特点是, `Spark Streaming` 可以在流上使用基于 `Spark` 的机器学习和流计算, 是一个一站式的平台 |
| `Spark Streaming` 具有很好的整合性               | `Spark Streaming` 可以从 `Kafka`, `Flume`, `TCP` 等流和队列中获取数据,,,,`Spark Streaming` 可以将处理过的数据写入文件系统, 常见数据库中 |
| `Spark Streaming` 是微批次处理模型               | 微批次处理的方式不会有长时间运行的 `Operator`, 所以更易于容错设计,,,,,,,微批次模型能够避免运行过慢的服务, 实行推测执行 |

**TCP三次握手回顾**

1

`Client` 向 `Server` 发送 `SYN(j)`, 进入 `SYN_SEND` 状态等待 `Server` 响应

2

`Server` 收到 `Client` 的 `SYN(j)` 并发送确认包 `ACK(j + 1)`, 同时自己也发送一个请求连接的 `SYN(k)` 给 `Client`, 进入 `SYN_RECV` 状态等待 `Client` 确认

3

`Client` 收到 `Server` 的 `ACK + SYN`, 向 `Server` 发送连接确认 `ACK(k + 1)`, 此时, `Client` 和 `Server` 都进入 `ESTABLISHED` 状态, 准备数据发送

**netcat**

- `Netcat` 简写 `nc`, 命令行中使用 `nc` 命令调用

- `Netcat` 是一个非常常见的 `Socket` 工具, 可以使用 `nc` 建立 `Socket server` 也可以建立 `Socket client`

  - `nc -l` 建立 `Socket server`, `l` 是 `listen` 监听的意思

  - `nc host port` 建立 `Socket client`, 并连接到某个 `Socket server`

    一台虚拟机 nc -lk 1567   另一台复制的虚拟机 nc localhost  1567  两台可以通信

## 4 小试牛刀

使用 `Spark Streaming` 程序和 `Socket server` 进行交互

**创建maven工程**

pom 依赖

```xml
 <properties>
        <scala.version>2.11.8</scala.version>
        <spark.version>2.2.0</spark.version>
        <slf4j.version>1.7.16</slf4j.version>
        <log4j.version>1.2.17</log4j.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>${scala.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.47</version>
        </dependency>

        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming_2.11</artifactId>
            <version>2.2.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql-kafka-0-10_2.11</artifactId>
            <version>2.2.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-hive_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>2.6.0</version>
        </dependency>

        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>jcl-over-slf4j</artifactId>
            <version>${slf4j.version}</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>${slf4j.version}</version>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <version>${slf4j.version}</version>
        </dependency>
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>${log4j.version}</version>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.10</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <sourceDirectory>src/main/scala</sourceDirectory>
        <testSourceDirectory>src/test/scala</testSourceDirectory>
        <plugins>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.0</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>

            <plugin>
                <groupId>net.alchim31.maven</groupId>
                <artifactId>scala-maven-plugin</artifactId>
                <version>3.2.0</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>testCompile</goal>
                        </goals>
                        <configuration>
                            <args>
                                <arg>-dependencyfile</arg>
                                <arg>${project.build.directory}/.scala_dependencies</arg>
                            </args>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <!--打包插件  因为默认的打包不包含maven的依赖-->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.1.1</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass></mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

```

```scala
//Spark Streaming  是微小的批处理 并不是实时流, 而是按照时间切分小批量, 一个一个的小批量处理
class StreamIngDemo extends Serializable {
  @Test
  def stream(): Unit = {

    // 1. 初始化环境
    // 在 SparkCore 中的内存, 创建 SparkContext 的时候使用
    // 在创建 Streaming Context 的时候也要用到 conf, 说明 Spark Streaming 是基于 Spark Core 的
    // 在执行 master 的时候, 不能指定一个线程  Spark Streaming 中至少要有两个线程
    // 因为在 Streaming 运行的时候, 需要开一个新的线程来去一直监听数据的获取
    val sparkConf = new SparkConf().setAppName("streaming").setMaster("local[3]")
    // StreamingContext 其实就是 Spark Streaming 的入口
    // 相当于 SparkContext 是 Spark Core 的入口一样
    // 它们也都叫做 XXContext  第二个参数是控制多长时间生成一个rdd去统计
    val context = new StreamingContext(sparkConf, Seconds(3))
    // socketTextStream 这个方法用于创建一个 DStream, 监听 Socket 输入, 当做文本来处理
    // sparkContext.textFile() 创建一个 rdd, 他们俩类似, 都是创建对应的数据集
    // RDD -> Spark Core     DStream -> Spark Streaming
    // DStream 可以理解为是一个流式的 RDD
    //调整 Log 级别, 避免过多的 Log 影响视线
    context.sparkContext.setLogLevel("WARN")
    val lines = context.socketTextStream(
      hostname = "node03",
      port = 9999,
        //选择 Receiver 获取到数据后的保存方式, 此处是内存和磁盘都有, 并且序列化后保存
      storageLevel = StorageLevel.MEMORY_AND_DISK_SER
    )
    //数据处理
    //把句子拆为单词
    val dstr1 = lines.flatMap(it => it.split(" "))
    //转换单词
    val dstr2 = dstr1.map((_, 1))
    //词频
    val res = dstr2.reduceByKey(_ + _)

    //展示和启动
    res.print()  //	类似 RDD 中的 Action, 执行最后的数据输出和收集
      //	启动流和 JobGenerator, 开始流式处理数据
    context.start()
    // main 方法执行完毕后整个程序就会退出, 所以需要阻塞主线程
    context.awaitTermination()
  }

```

可以在本地运行  也可以在集群运行

````
集群运行
spark-submit --class cn.itcast.streaming.StreamingWordCount  --master local[6] original-streaming-0.0.1.jar node02 9999
````

## 5 总结

**注意点:**

1`Spark Streaming` 并不是真正的来一条数据处理一条

`Spark Streaming` 的处理机制叫做小批量, 英文叫做 `mini-batch`, 是收集了一定时间的数据后生成 `RDD`, 后针对 `RDD` 进行各种转换操作, 这个原理提现在如下两个地方

2`Spark Streaming` 中至少要有两个线程

在使用 `spark-submit` 启动程序的时候, 不能指定一个线程

- 主线程被阻塞了, 等待程序运行
- 需要开启后台线程获取数据

**创建StreamingContext**

- `StreamingContext` 是 `Spark Streaming` 程序的入口
- 在创建 `StreamingContext` 的时候, 必须要指定两个参数, 一个是 `SparkConf`, 一个是流中生成 `RDD` 的时间间隔
- `StreamingContext` 提供了如下功能
  - 创建 `DStream`, 可以通过读取 `Kafka`, 读取 `Socket` 消息, 读取本地文件等创建一个流, 并且作为整个 `DAG` 中的 `InputDStream`
  - `RDD` 遇到 `Action` 才会执行, 但是 `DStream` 不是, `DStream` 只有在 `StreamingContext.start()` 后才会开始接收数据并处理数据
  - 使用 `StreamingContext.awaitTermination()` 等待处理被终止
  - 使用 `StreamingContext.stop()` 来手动的停止处理
- 在使用的时候有如下注意点
  - 同一个 `Streaming` 程序中, 只能有一个 `StreamingContext`
  - 一旦一个 `Context` 已经启动 (`start`), 则不能添加新的数据源 **

**算子**
这些算子类似 `RDD`, 也会生成新的 `DStream`

这些算子操作最终会落到每一个 `DStream` 生成的 `RDD` 中

reduceByKey   这个算子需要特别注意, 这个聚合并不是针对于整个流, 而是针对于某个批次的数据

# 二原理

## 1`Spark Streaming` **的特点**

- `Spark Streaming` 会源源不断的处理数据, 称之为流计算
- `Spark Streaming` 并不是实时流, 而是按照时间切分小批量, 一个一个的小批量处理
- `Spark Streaming` 是流计算, 所以可以理解为数据会源源不断的来, 需要长时间运行

## 2`Spark Streaming` **是按照时间切分小批量**

```
Spark Streaming` 中的编程模型叫做 `DStream`, 所有的 `API` 都从 `DStream` 开始, 其作用就类似于 `RDD` 之于 `Spark Core
```

可以理解为 `DStream` 是一个管道, 数据源源不断的从这个管道进去, 被处理, 再出去
  但是需要注意的是, `DStream` 并不是严格意义上的实时流, 事实上, `DStream` 并不处理数据, 而是处理 `RDD`

- `Spark Streaming` 是小批量处理数据, 并不是实时流
- `Spark Streaming` 对数据的处理是按照时间切分为一个又一个小的 `RDD`, 然后针对 `RDD` 进行处理

`RDD` 中针对数据的处理是使用算子, 在 `DStream` 中针对数据的操作也是算子

**难道 `DStream` 会把算子的操作交给 `RDD` 去处理? 如何交?**

## 3 `Spark Streaming` **是流计算, 流计算的数据是无限的**

无限的数据一般指的是数据不断的产生, 比如说运行中的系统, 无法判定什么时候公司会倒闭, 所以也无法断定数据什么时候会不再产生数据

- 那就会产生一个问题

  如何不简单的读取数据, 如何应对数据量时大时小?

​      如何数据是无限的, 意味着可能要一直运行下去

- 那就会又产生一个问题

  `Spark Streaming` 不会出错吗? 数据出错了怎么办?

**四个问题:**

- `DStream` 如何对应 `RDD`?
- 如何切分 `RDD`?
- 如何读取数据?
- 如何容错?

## 4 DAG的定义

`RDD` **和** `DStream` **的** `DAG`

**rdd的wordcount :**

```scala
val textRDD = sc.textFile(...)
val splitRDD = textRDD.flatMap(_.split(" "))
val tupleRDD = splitRDD.map((_, 1))
val reduceRDD = tupleRDD.reduceByKey(_ + _)
```

**DStream的wordcount**

```scala
val lines: DStream[String] = ssc.socketTextStream(...)
val words: DStream[String] = lines.flatMap(_.split(" "))
val wordCounts: DStream[(String, Int)] = words.map(x => (x, 1)).reduceByKey(_ + _)
```

### **1`RDD` 和 `DStream` 的区别**

![img](/images/sparkstreaming/ss1.png)

- `DStream` 的数据是不断进入的, `RDD` 是针对一个数据的操作
- 像 `RDD` 一样, `DStream` 也有不同的子类, 通过不同的算子生成
- 一个 `DStream` 代表一个数据集, 其中包含了针对于上一个数据的操作
- `DStream` 根据时间切片, 划分为多个 `RDD`, 针对 `DStream` 的计算函数, 会作用于每一个 `DStream` 中的 `RDD`

### 2`DStream` 如何形式 `DAG`

![img](/images/sparkstreaming/ss2.png)

- 每个 `DStream` 都有一个关联的 `DStreamGraph` 对象
- `DStreamGraph` 负责表示 `DStream` 之间的的依赖关系和运行步骤
- `DStreamGraph` 中会单独记录 `InputDStream` 和 `OutputDStream`

### **3切分流, 生成小批量**

**静态和动态**

- `DStream` 对应 `RDD`
- `DStreamGraph` 表示 `DStream` 之间的依赖关系和运行流程, 相当于 `RDD` 通过 `DAGScheduler` 所生成的 `RDD DAG`

`RDD` 的运行分为逻辑计划和物理计划

- 逻辑计划就是 `RDD` 之间依赖关系所构成的一张有向无环图
- 后根据这张 `DAG` 生成对应的 `TaskSet` 调度到集群中运行, 

但是在 `DStream` 中则不能这么简单的划分, 因为 `DStream` 中有一个非常重要的逻辑, 需要按照时间片划分小批量

- 在 `Streaming` 中, `DStream` 类似 `RDD`, 生成的是静态的数据处理过程, 例如一个 `DStream` 中的数据经过 `map` 转为其它模样
- 在 `Streaming` 中, `DStreamGraph` 类似 `DAG`, 保存了这种数据处理的过程

上述两点, 其实描述的是静态的一张 `DAG`, 数据处理过程, 但是 `Streaming` 是动态的, 数据是源源不断的来的

所以, 在 `DStream` 中, 静态和动态是两个概念, 有不同的流程

![img](/images/sparkstreaming/ss3.png)

- `DStreamGraph` 将 `DStream` 联合起来, 生成 `DStream` 之间的 `DAG`, 这些 `DStream` 之间的关系是相互依赖的关系, 例如一个 `DStream` 经过 `map` 转为另外一个 `DStream`
- 但是把视角移动到 `DStream` 中来看, `DStream` 代表了源源不断的 `RDD` 的生成和处理, 按照时间切片, 所以一个 `DStream DAG` 又对应了随着时间的推进所产生的无限个 `RDD DAG`

**动态生成 `RDD DAG` 的过程**

`RDD DAG` 的生成是按照时间来切片的, `Streaming` 会维护一个 `Timer`, 固定的时间到达后通过如下五个步骤生成一个 `RDD DAG` 后调度执行

1. 通知 `Receiver` 将收到的数据暂存, 并汇报存储的元信息, 例如存在哪, 存了什么
2. 通过 `DStreamGraph` 复制出一套新的 `RDD DAG`
3. 将数据暂存的元信息和 `RDD DAG` 一同交由 `JobScheduler` 去调度执行
4. 提交结束后, 对系统当前的状态 `Checkpoint`

### **4数据的产生和导入**

#### 1**Receiver**

在 `Spark Streaming` 中一个非常大的挑战是, 很多外部的队列和存储系统都是分块的, `RDD` 是分区的, 在读取外部数据源的时候, 会用不同的分区对照外部系统的分片, 例如

![img](/images/sparkstreaming/ss4.png)

不仅 `RDD`, `DStream` 中也面临这种挑战

`DStream` 中是 `RDD` 流, 只是 `RDD` 的分区对应了 `Kafka` 的分区就可以了吗?

答案是不行, 因为需要一套单独的机制来保证并行的读取外部数据源, 这套机制叫做 `Receiver`

**`Receiver` 的结构**

为了保证并行获取数据, 对应每一个外部数据源的分区, 所以 `Receiver` 也要是分布式的, 主要分为三个部分

- `Receiver` 是一个对象, 是可以有用户自定义的获取逻辑对象, 表示了如何获取数据
- `Receiver Tracker` 是 `Receiver` 的协调和调度者, 其运行在 `Driver` 上
- `Receiver Supervisor` 被 `Receiver Tracker` 调度到不同的几点上分布式运行, 其会拿到用户自定义的 `Receiver` 对象, 使用这个对象来获取外部数据

**`Receiver` 的执行过程**

1. 在 `Spark Streaming` 程序开启时候, `Receiver Tracker` 使用 `JobScheduler` 分发 `Job` 到不同的节点, 每个 `Job` 包含一个 `Task` , 这个 `Task` 就是 `Receiver Supervisor`, 这个部分的源码还挺精彩的, 其实是复用了通用的调度逻辑
2. `ReceiverSupervisor` 启动后运行 `Receiver` 实例
3. `Receiver` 启动后, 就将持续不断地接收外界数据, 并持续交给 `ReceiverSupervisor` 进行数据存储
4. `ReceiverSupervisor` 持续不断地接收到 `Receiver` 转来的数据, 并通过 `BlockManager` 来存储数据
5. 获取的数据存储完成后发送元数据给 `Driver` 端的 `ReceiverTracker`, 包含数据块的 `id`, 位置, 数量, 大小 等信息

#### 2 Direct

direct

使用Kafka低层次API实现的.

特点:

1. 会创建和Kafka分区数一样的RDD分区.提升性能
2. 使用checkpoint将分区/偏移量信息进行保存   `0:2342344`
3. 因为直接根据分区id+偏移量获取消息,也解决了偏移量不一致问题.

**总结:**

reciver

使用Kafka高层次API实现,

特点:

1. 启动一个Reciver进行消费,启动和Kafka分区数一样的线程数进行消费,所以性能提升不大
2. 为了确保数据安全,需要启用WAL预写日志功能,数据会在HDFS备份一份,Kafka里面默认也有一份,所以数据会被重复保存
3. 偏移量是在zookeeper上面存储的,因为信息不同步,有可能引起数据的重复消费问题.

### 5 容错

因为要非常长时间的运行, 对于任何一个流计算系统来说, 容错都是非常致命也非常重要的一环, 在 `Spark Streaming` 中, 大致提供了如下的容错手段

#### 1 **热备**

这行代码中的 `StorageLevel.MEMORY_AND_DISK_SER` 的作用是什么? 其实就是热备份

- 当 Receiver 获取到数据要存储的时候, 是交给 BlockManager 存储的
- 如果设置了 `StorageLevel.MEMORY_AND_DISK_SER`, 则意味着 `BlockManager` 不仅会在本机存储, 也会发往其它的主机进行存储, 本质就是冗余备份
- 如果某一个计算失败了, 通过冗余的备份, 再次进行计算即可

这是默认的容错手段

#### 2 冷备

冷备在 `Spark Streaming` 中的手段叫做 `WAL` (预写日志)

- 当 `Receiver` 获取到数据后, 会交给 `BlockManager` 存储
- 在存储之前先写到 `WAL` 中, `WAL` 中保存了 `Redo Log`, 其实就是记录了数据怎么产生的, 以便于恢复的时候通过 `Log`恢复
- 当出错的时候, 通过 `Redo Log` 去重放数据

#### 3 重放

- 有一些上游的外部系统是支持重放的, 比如说 `Kafka`
- `Kafka` 可以根据 `Offset` 来获取数据
- 当 `SparkStreaming` 处理过程中出错了, 只需要通过 `Kafka` 再次读取即可

# 三操作

## 1 中间状态

需求: 统计整个流中, 所有出现的单词数量, 而不是一个批中的数量

在小试牛刀中	只能统计某个时间段内的单词数量, 因为 `reduceByKey` 只能作用于某一个 `RDD`, 不能作用于整个流

如果想要求单词总数该怎么办?

**以使用状态来记录中间结果, 从而每次来一批数据, 计算后和中间状态求和, 于是就完成了总数的统计**

- 使用 `updateStateByKey` 可以做到这件事
- `updateStateByKey` 会将中间状态存入 `CheckPoint` 中

```scala
//使用中间状态(updateStateByKey)记录结果  可以统计整个流中出现的单词数量 等操作
  @Test
  def zhongjianStream(): Unit = {
    // 1. 创建 Context
    val conf = new SparkConf()
      .setAppName("updateStateBykey")
      .setMaster("local[3]")
    val ssc = new StreamingContext(conf, Seconds(2))
    ssc.sparkContext.setLogLevel("WARN")

    // 2. 读取数据生成 DStream
    val source = ssc.socketTextStream(
      hostname = "node03",
      port = 9999,
      storageLevel = StorageLevel.MEMORY_AND_DISK_SER_2
    )

    // 3. 词频统计
    val wordsTuple = source.flatMap(_.split(" "))
      .map((_, 1))

    // 4. 全局聚合
    ssc.checkpoint("checkpoint")

    def updateFunc(newValue: Seq[Int], runningValue: Option[Int]): Option[Int] = {
      // newValue : 对应当前批次中 Key 对应的所有 Value
      // runningValue : 当前的中间结果

      val currBatchValue = newValue.sum
      val state = runningValue.getOrElse(0) + currBatchValue

      Some(state)
    }

    val result = wordsTuple.updateStateByKey[Int](updateFunc _)

    // 5. 输出
    result.print()
    ssc.start()
    ssc.awaitTermination()

  }
```

## 2 window操作

需求: 计算过 `30s` 的单词总数, 每 `10s` 更新一次

```scala
//使用窗口Window操作   需求 计算过 30s 的单词总数, 每 10s 更新一次
  @Test
  def winStream(): Unit = {
    val sparkConf = new SparkConf().setAppName("NetworkWordCount").setMaster("local[6]")

    val ssc = new StreamingContext(sparkConf, Seconds(1))
    ssc.sparkContext.setLogLevel("WARN")
    val lines: DStream[String] = ssc.socketTextStream(
      hostname = "node03",
      port = 9999,
      storageLevel = StorageLevel.MEMORY_AND_DISK_SER)

    val words = lines.flatMap(_.split(" ")).map(x => (x, 1))

    // 通过 window 操作, 会将流分为多个窗口
    val wordsWindow = words.window(Seconds(30), Seconds(10))
    // 此时是针对于窗口求聚合
    val wordCounts = wordsWindow.reduceByKey((newValue, runningValue) => newValue + runningValue)

    wordCounts.print()

    ssc.start()
    ssc.awaitTermination()
  }

  //使用窗口函数的另一种方式'  既然 window 操作经常配合 reduce 这种聚合, 所以 Spark Streaming 提供了较为方便的方法
  @Test
  def win2(): Unit ={
    val sparkConf = new SparkConf().setAppName("NetworkWordCount").setMaster("local[6]")

    val ssc = new StreamingContext(sparkConf, Seconds(1))
    //调整 Log 级别, 避免过多的 Log 影响视线
     ssc.sparkContext.setLogLevel("warn")
    val lines: DStream[String] = ssc.socketTextStream(
      hostname = "node03",
      port = 9999,
      storageLevel = StorageLevel.MEMORY_AND_DISK_SER)

    val words = lines.flatMap(_.split(" ")).map(x => (x, 1))

    // 开启窗口并自动进行 reduceByKey 的聚合
    val wordCounts = words.reduceByKeyAndWindow(
      reduceFunc = (n, r) => n + r,
      windowDuration = Seconds(30),
      slideDuration = Seconds(10))

    wordCounts.print()

    ssc.start()
    ssc.awaitTermination()
  }
```

**窗口时间**

- 在 `window` 函数中, 接收两个参数

  - `windowDuration` 窗口长度, `window` 函数会将多个 `DStream` 中的 `RDD` 按照时间合并为一个, 那么窗口长度配置的就是将多长时间内的 `RDD` 合并为一个
  - `slideDuration` 滑动间隔, 比较好理解的情况是直接按照某个时间来均匀的划分为多个 `window`, 但是往往需求可能是统计最近 `xx分` 内的所有数据, 一秒刷新一次, 那么就需要设置滑动窗口的时间间隔了, 每隔多久生成一个 `window`

- 滑动时间的问题

  - 如果 `windowDuration > slideDuration`, 则在每一个不同的窗口中, 可能计算了重复的数据
  - 如果 `windowDuration < slideDuration`, 则在每一个不同的窗口之间, 有一些数据为能计算进去

  但是其实无论谁比谁大, 都不能算错, 例如, 我的需求有可能就是统计一小时内的数据, 一天刷新两次

