---
title: SparkSql调优
tags:
  - Spark
categories: Spark
encrypt: 
enc_pwd: 
abbrlink: 13454
date: 2020-05-17 21:12:22
summary_img:
---

# 一 Spark的动态资源分配

## 1 问题背景:

用户提交Spark应用到Yarn上时，可以通过spark-submit的**num-executors**参数显示地指定executor个数，随后，ApplicationMaster会为这些executor申请资源，每个executor作为一个Container在Yarn上运行。Spark调度器会把Task按照合适的策略分配到executor上执行。所有任务执行完后，executor被杀死，应用结束。在job运行的过程中，无论executor是否领取到任务，都会一直占有着资源不释放。很显然，这在任务量小且显示指定大量executor的情况下会很容易造成资源浪费。

在探究Spark如何实现之前，首先思考下如果自己来解决这个问题，需要考虑哪些因素？大致的方案很容易想到：如果executor在一段时间内一直处于空闲状态，那么就可以kill该executor，释放其占用的资源。当然，一些细节及边界条件需要考虑到：

- executor动态调整的范围？无限减少？无限制增加？
- executor动态调整速率？线性增减？指数增减？
- 何时移除Executor？
- 何时新增Executor了？只要由新提交的Task就新增Executor吗？
- Spark中的executor不仅仅提供计算能力，还可能存储持久化数据，这些数据在宿主executor被kill后，该如何访问？

## 2 原理分析

### 2.1 Executor的生命周期





# 五: 参数

#### 提交:

```shell
./bin/spark-submit \
  --master spark://192.168.1.1:7077 \
  --num-executors 100 \
  --executor-memory 6G \
  --executor-cores 4 \
　--total-executor-cores 400 \ ##standalone default all cores 
  --driver-memory 1G \
  --conf spark.default.parallelism=1000 \
  --conf spark.storage.memoryFraction=0.5 \
  --conf spark.shuffle.memoryFraction=0.3 \
```

### yarn和log:

| 参数                                       | 描述   |
| ---------------------------------------- | ---- |
| spark.yarn.jars=hdfs://Ucluster/home/hadoop/lib/lib_20190327143532/spark2/*.jar |      |
| spark.yarn.historyServer.address=ip:port |      |
| spark.eventLog.dir=hdfs://Ucluster/var/log/spark |      |
| spark.eventLog.enabled=true              |      |
| spark.history.fs.logDirectory=hdfs://Ucluster/var/log/spark |      |



### 队列:

| 功能   |                           |      |
| ---- | ------------------------- | ---- |
| 队列   | spark.yarn.queue(--queue) |      |



### 动态资源分配:

| 功能     | 命令                                       | 解释                                       |
| ------ | ---------------------------------------- | ---------------------------------------- |
| 动态资源分配 | spark.dynamicAllocation.enabled=true                                        spark.shuffle.service.enabled=true | 动态资源分配开关(以上两个参数同时设置为true可开启动态资源分配，开启后可防止资源浪费情况。Spark-2.1.0默认关闭动态资源分配，Spark-2.3.3默认打开)(spark.shuffle.service.enabled  NodeManager中一个长期运行的辅助服务，用于提升Shuffle计算性能。默认为false，表示不启用该功能) |
|        | spark.dynamicAllocation.minExecutors=0   | Executor调整下限                             |
|        | spark.dynamicAllocation.maxExecutors=4   | Executor调整上限                             |
|        | spark.dynamicAllocation.initialExecutors=1 | Executor初始数量（默认值：minExecutors）三者的关系必须满足：minExecutors <= initialExecutors <= maxExecutors   如果显示指定了num-executors参数，那么initialExecutors就是num-executor指定的值。 |
|        | spark.dynamicAllocation.executorIdleTimeout=10 | 默认值60s  Executor超时：当Executor不执行任何任务时，会被标记为Idle状态。空闲一段时间后即被认为超时，会被kill |
|        | spark.dynamicAllocation.schedulerBacklogTimeout=1 | 默认1s  资源不足时，何时新增Executor：当有Task处于pending状态，意味着资源不足，此时需要增加Executor, |
|        | spark.dynamicAllocation.cachedExecutorIdleTimeout=12 | 如果Executor中缓存了数据，那么该Executor的Idle-timeout时间就不是由executorIdleTimeout决定，而是用spark.dynamicAllocation.cachedExecutorIdleTimeout控制，默认值：Integer.MAX_VALUE。如果手动设置了该值，当这些缓存数据的Executor被kill后，我们可以通过NodeManannger的External Shuffle Server来访问这些数据。这就要求NodeManager中spark.shuffle.service.enabled必须开启。 |
|        |                                          |                                          |
|        |                                          |                                          |

### Executor,task,cores,yarn等参数:

| 功能   |                                          |                                          |
| ---- | ---------------------------------------- | ---------------------------------------- |
|      | num-executors或spark.num.executors=4      | executor的数目是由每个节点运行的executor数目和集群的节点数共同决定	如40个节点 每个节点7个executor 则总的为280 (现实比这个小,driver也会占用core和内存)	Hive性能与用于运行查询的executor数量直接相关。 但是，不通查询还是不通。 通常，性能与executor的数量成比例。 例如，查询使用四个executor大约需要使用两个executor的一半时间。 但是，性能在一定数量的executor中达到峰值，高于此值时，增加数量不会改善性能并且可能产生不利影响。	在大多数情况下，使用一半的集群容量（executor数量的一半）可以提供良好的性能。 为了获得最佳性能，最好使用所有可用的executor。 例如，设置spark.executor.instances = 280。 对于基准测试和性能测量，强烈建议这样做。 |
|      | spark.executor.instances = 280           | `Executors`的个数。这个配置和`spark.dynamicAllocation.enabled`不兼容。当同时配置这两个配置时，动态分配关闭，`spark.executor.instances`被使用 |
|      | driver-memory(spark.driver.memory)=      | 当运行hive on spark的时候，每个spark driver能申请的最大jvm 堆内存,该参数结合 spark.driver.memoryOverhead共同决定着driver的内存大小。 |
|      | spark.yarn.driver.memoryOverhead(spark.driver.memoryOverhead)=executorMemory * 0.10`，并且不小于`384m | 每个driver能从yarn申请的堆外内存的大小。driver的内存大小并不直接影响性能，但是也不要job的运行受限于driver的内存,y一般 spark.driver.memory和 spark.driver.memoryOverhead内存的总和占总内存的10%-15%。假设 yarn.nodemanager.resource.memory-mb=100*1024MB,那么driver内存设置为12GB，此时 spark.driver.memory=10.5gb和spark.driver.memoryOverhead=1.5gb |
|      | executor-memory(spark.executor.memory)=3g | 能申请的最大jvm 堆内存                            |
|      | spark.executor.memoryOverhead(spark.yarn.executor.memoryOverhead)=executorMemory * 0.10`，并且不小于`384m | YARN 请求的每个 executor的额外堆内存大小,spark.executor.memory 和spark.executor.memoryOverhead 共同决定着 executor内存。建议 spark.executor.memoryOverhead站总内存的 15%-20%。		(或executorMemory * 0.07, with minimum of 384) 那么最终 spark.executor.memoryOverhead=2 G 和spark.executor.memory=12 G  executor总为14g |
|      | executor-cores(spark.executor.cores)=4   | 每个executor的核数(也为task的个数)在executor运行的task共享内存,其实，executor内部是用newCachedThreadPool运行task的。 |
|      | spark.task.cpus = 1                      | 每个task配置核数                               |
|      | spark.sql.windowExec.buffer.spill.threshold | 当用户的SQL中包含窗口函数时，并不会把一个窗口中的所有数据全部读进内存，而是维护一个缓存池，当池中的数据条数大于该参数表示的阈值时，spark将数据写到磁盘 |
|      | spark.driver.cores                       | driver的核数                                |
|      | spark.yarn.am.memory=512                 | 在客户端模式（`client mode`）下，`yarn`应用`master`使用的内存数。在集群模式（`cluster mode`）下，使用`spark.driver.memory`代替。 |
|      | spark.yarn.am.cores                      | 在客户端模式（`client mode`）下，`yarn`应用的`master`使用的核数。在集群模式下，使用`spark.driver.cores`代替。 |
|      | spark.cores.max =12                      | 限制使用的最大核数                                |
|      | total-executor-cores(spark.total.executor.cores) | executor 的总核数                            |

### 动态分区(hive)

| 功能   |                                          |      |
| ---- | ---------------------------------------- | ---- |
|      | hive.exec.dynamic.partition.mode=nonstrict; |      |
|      | hive.exec.dynamic.partition=true;        |      |
|      | hive.exec.max.dynamic.partitions=1000;   |      |
|      | hive.exec.max.dynamic.partitions.pernode=100; |      |

### 广播:

| 功能   |                                          |                                          |
| ---- | ---------------------------------------- | ---------------------------------------- |
|      | spark.sql.adaptive.join.enabled = true   | 在join操作时自动进行小表广播                         |
|      | spark.sql.autoBroadcastJoinThreshold = 33554432 | -1禁止广播，广播阈值                              |
|      | spark.sql.broadcastTimeout=600           | 广播超时时间(默认5min)当广播小表时，如果广播时间超过此参数设置值，会导致任务失败 |
|      |                                          | SELECT /*+ MAPJOIN(b) */    或SELECT /*+ BROADCASTJOIN(b) */    或  SELECT /*+ BROADCAST(b) */ |



### shuffle调优

| 参数                                       | 描述                                       |
| ---------------------------------------- | ---------------------------------------- |
| spark.storage.memoryFraction             | 参数说明：该参数用于设置RDD持久化数据在Executor内存中能占的比例，默认是0.6。也就是说，默认Executor 60%的内存，可以用来保存持久化的RDD数据。根据你选择的不同的持久化策略，如果内存不够时，可能数据就不会持久化，或者数据会写入磁盘。                                                                                                                                 参数调优建议：如果Spark作业中，有较多的RDD持久化操作，该参数的值可以适当提高一些，保证持久化的数据能够容纳在内存中。避免内存不够缓存所有的数据，导致数据只能写入磁盘中，降低了性能。但是如果Spark作业中的shuffle类操作比较多，而持久化操作比较少，那么这个参数的值适当降低一些比较合适。此外，如果发现作业由于频繁的gc导致运行缓慢（通过spark web ui可以观察到作业的gc耗时），意味着task执行用户代码的内存不够用，那么同样建议调低这个参数的值。 |
| spark.shuffle.memoryFraction             | 参数说明：该参数用于设置shuffle过程中一个task拉取到上个stage的task的输出后，进行聚合操作时能够使用的Executor内存的比例，默认是0.2。也就是说，Executor默认只有20%的内存用来进行该操作。shuffle操作在进行聚合时，如果发现使用的内存超出了这个20%的限制，那么多余的数据就会溢写到磁盘文件中去，此时就会极大地降低性能。                                                                                       参数调优建议：如果Spark作业中的RDD持久化操作较少，shuffle操作较多时，建议降低持久化操作的内存占比，提高shuffle操作的内存占比比例，避免shuffle过程中数据过多时内存不够用，必须溢写到磁盘上，降低了性能。此外，如果发现作业由于频繁的gc导致运行缓慢，意味着task执行用户代码的内存不够用，那么同样建议调低这个参数的值。 |
| spark.shuffle.service.enabled=true       | NodeManager中一个长期运行的辅助服务，用于提升Shuffle计算性能。默认为false，表示不启用该功能。spark.shuffle.service.port(Shuffle服务监听数据获取请求的端口。可选配置，默认值为“7337”。) |
| spark.reducer.maxSizeInFlight=48m        | **参数说明：**   该参数用于设置shuffle read task的buffer缓冲大小，而这个buffer缓冲决定了每次能够拉取多少数据。**调优建议：**   如果作业可用的内存资源较为充足的话，可以适当增加这个参数的大小（比如96m），从而减少拉取数据的次数，也就可以减少网络传输的次数，进而提升性能。在实践中发现，合理调节该参数，性能会有1%~5%的提升。 |
| spark.reducer.maxReqsInFlight=Int.MaxValue | 限制远程机器拉取本机器文件块的请求数，随着集群增大，需要对此做出限制。否则可能会使本机负载过大而挂掉 |
| spark.reducer.maxBlocksInFlightPerAddress=Int.MaxValue | 限制了每个主机每次reduce可以被多少台远程主机拉取文件块，调低这个参数可以有效减轻node manager的负载 |
| spark.maxRemoteBlockSizeFetchToMem=Int.MaxValue - 512 | 远程block大小大于该阈值多情况下，直接拉取存储                |
| spark.shuffle.compress=true              | map输出文件将采用 spark.io.compression.codec.格式压缩 |
| spark.shuffle.file.buffer=32k            | task的BufferedOutputStream的buffer缓冲大小。将数据写到磁盘文件之前，会先写入buffer缓冲中，待缓冲写满之后，才会溢写到磁盘。 |
| spark.shuffle.io.maxRetries=3            | 拉取失败重试次数                                 |
| spark.shuffle.io.numConnectionsPerPeer=1 |                                          |
| spark.shuffle.io.preferDirectBufs=true   | 堆外缓存区开关，可减少shuffle和block缓存传输中垃圾的回收，当堆外内存受到限制时关闭此功能 |
| spark.shuffle.io.retryWait=5s            | 重试拉取的时间间隔                                |
| spark.shuffle.service.enabled=false      | 动态移除空闲资源时打开，防止移除后文件丢失                    |
| spark.shuffle.service.index.cache.size=100m | 缓存index文件大小，fetch数据的时候会读取index文件，如果这过程比较慢，就需要调大 |
| spark.shuffle.sort.bypassMergeThreshold=200 | 当shuffle manager配置为sortShuffleManager ，而且reduce个数小于该值，map端不需要做combine，则采用BypassMergeSortShuffle |
| spark.shuffle.spill.compress=true        | shuffle 阶段spill文件是否需要压缩，压缩方法spark.io.compression.codec |



### sparksql的小文件合并

#### 1 sql有shuffle阶段

| 参数                                       | 说明                                       |
| ---------------------------------------- | ---------------------------------------- |
| spark.sql.adaptive.enabled=true          | 默认关闭，如果开启请确保资源动态分配生效,参数是用于开启spark的自适应执行(用来控制是否开启adaptive execution，默认为false。一直以来，Spark只能设置固定的并行度（参考spark.sql.shuffle.partitions），在大促期间，数据量激增，每个task处理的数量增加，很容易出现oom的情况。在开启此项参数后，Spark将会按照spark.sql.ataptive.shuffle.targetPostShuffleInputSize设置的每个task的目标处理数据量自动调整stage并行度，减少task出现oom的情况。) |
| spark.sql.adaptive.shuffle.targetPostShuffleInputSize=128000000 | reduce读最少数据量,用于控制之后的shuffle 阶段的平均输入数据大小，防止产生过多的task。Spark将会根据此参数值动态调整task个数， |
| spark.sql.adaptive.shuffle.targetPostShuffleRowCount= | 接收最少记录条数                                 |
| spark.sql.adaptive.minNumPostShufflePartitions= | reduce最小值                                |
| spark.sql.adaptive.maxNumPostShufflePartitions= | reduce最大值                                |
| spark.sql.ataptive.skewedJoin.enabled（spark-2.3.3） | 在开启adaptive execution时，用来控制是否开启自动处理join时的数据倾斜，默认为false。 |
| spark.sql.ataptive.skewedPartitionMaxSplits（spark-2.3.3） | 在开启adaptive execution时，控制处理一个倾斜 Partition 的 Task 个数上限，默认值为 5。 |
| spark.sql.ataptive.skewedPartitionRowCountThreshold（spark-2.3.3） | 在开启adaptive execution时，设置一个 Partition 被视为倾斜 Partition 的行数下限，也即行数低于该值的 Partition 不会被当作倾斜 Partition 处理。其默认值为 10L * 1000 * 1000 即一千万。 |
| spark.sql.ataptive.skewedPartitionSizeThreshold（spark-2.3.3） | 在开启adaptive execution时，设置一个 Partition 被视为倾斜 Partition 的大小下限，也即大小小于该值的 Partition 不会被视作倾斜 Partition。其默认值为 64 * 1024 * 1024 也即 64MB。 |
| spark.sql.ataptive.skewedPartitionFactor（spark-2.3.3） | 在开启adaptive execution时，设置倾斜因子。如果一个 Partition 的大小大于 `spark.sql.adaptive.skewedPartitionSizeThreshold` 的同时大于各 Partition 大小中位数与该因子的乘积，或者行数大于 `spark.sql.adaptive.skewedPartitionRowCountThreshold` 的同时大于各 Partition 行数中位数与该因子的乘积，则它会被视为倾斜的 Partition。默认为10。 |

当自适应关闭时,合并小文件:

| 参数                                    | 说明                                       |
| ------------------------------------- | ---------------------------------------- |
| spark.sql.shuffle.partitions=2 默认值200 | (对sql生效)shuffle默认设置，动态推测关闭有效(设置为1 则会只有一个文件)(只针对sql有效);(调整stage的并行度，也就是每个stage的task个数，默认值为40。此参数一般设置为任务申请的总core数的2-4倍，如：申请100个executor，每个executor申请2个core，那么总core数为200，此参数设置的合理范围是400-800。注意，此参数不能调整某些读外部数据stage的并行度，如：读hdfs的stage，绝大多数情况它的并行度取决于需要读取的文件数。) |
| spark.default.parallelism=5           | 对sql不生效 对df,ds生效(该参数用于设置每个stage的默认task数量。这个参数极为重要，如果不设置可能会直接影响你的Spark作业性能。因此Spark官网建议的设置原则是，设置该参数为num-executors * executor-cores的2~3倍较为合适，比如Executor的总CPU core数量为300个，那么设置1000个task是可以的，此时可以充分地利用Spark集群的资源。) |



#### 2 sql无shuffle阶段

思路:无shuffle

### 1文件为parquet,orc格式

#### 1.1源数据小文件多:

| 参数                                | 说明                                       |
| --------------------------------- | ---------------------------------------- |
| spark.sql.files.maxPartitionBytes | 控制一个分区最大多少，默认128M                        |
| spark.sql.files.openCostInBytes   | 控制当一个文件小于该阈值时会继续扫描新的文件将其放到到一个分区，默认4M     |
| spark.default.parallelism         | hdfs文件rdd分区最终多少个，也与spark.default.parallelism有关，启动spark集群的时候可以指定该参数 |
| spark.sql.files.maxRecordsPerFile | 写入一个文件的最大记录数。如果该值为零或负，则没有限制              |

#### 1.2 源数据大文件

| 参数                                      | 说明                                       |
| --------------------------------------- | ---------------------------------------- |
| spark.files.maxPartitionBytes=   默认128m |                                          |
| parquet.block.size   默认128m             |                                          |
|                                         | 第一个参数是针对session有效的，也就是因为这你在读的时候设置就会立刻生效。在没有第二个参数配合的情况下，就已经能够增加分区数了，缺点是，分区里的数据可能不会很均匀，因为均匀程度也会受到第二个参数的影响。 |

### 2 文件为其他格式

| 参数                                       | 说明   |
| ---------------------------------------- | ---- |
| spark.hadoop.mapreduce.input.fileinputformat.split.minsize |      |
| spark.hadoop.mapreduce.input.fileinputformat.split.maxsize |      |
| spark.hadoop.mapreduce.input.fileinputformat.split.minsize.per.node |      |
| spark.hadoop.mapreduce.input.fileinputformat.split.minsize.per.rack |      |
| spark.hadoop.hive.exec.max.created.files |      |

# 六 sparksql参数

```scala
//查看参数
sparkSession.sql("set -v").show(200,false)
```

| 配置项                                      | 默认值                                      | 概述                                       |
| ---------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| spark.sql.optimizer.maxIterations        | 100                                      | sql优化器最大迭代次数                             |
| spark.sql.optimizer.inSetConversionThreshold | 10                                       | 插入转换的集合大小阈值                              |
| spark.sql.inMemoryColumnarStorage.compressed | TRUE                                     | 当设置为true时，SCAPK SQL将根据数据的统计自动为每个列选择压缩编解码器 |
| spark.sql.inMemoryColumnarStorage.batchSize | 10000                                    | 控制用于列缓存的批处理的大小。较大的批处理大小可以提高内存利用率和压缩率，但缓存数据时会出现OOM风险 |
| spark.sql.inMemoryColumnarStorage.partitionPruning | TRUE                                     | 启用内存中的列表分区剪枝                             |
| spark.sql.join.preferSortMergeJoin       | TRUE                                     | When true, 使用sort merge join 代替 shuffle hash join |
| spark.sql.sort.enableRadixSort           | TRUE                                     | 使用基数排序，基数排序性能非常快，但是会额外使用over heap.当排序比较小的Row时，overheap 需要提高50% |
| spark.sql.autoBroadcastJoinThreshold     | 10L * 1024 * 1024                        | 当执行join时，被广播到worker节点上表最大字节。当被设置为-1，则禁用广播。当前仅仅支持 Hive Metastore  tables，表大小的统计直接基于hive表的源文件大小 |
| spark.sql.limit.scaleUpFactor            | 4                                        | 在执行查询时，两次尝试之间读取partation数目的增量。较高的值会导致读取过多分区，较少的值会导致执行时间过长，因为浙江运行更多的作业 |
| spark.sql.statistics.fallBackToHdfs      | FALSE                                    | 当不能从table metadata中获取表的统计信息，返回到hdfs。这否有用取决与表是否足够小到能够使用auto broadcast  joins |
| spark.sql.defaultSizeInBytes             | Long.MaxValue                            | 在查询计划中表默认大小，默认被设置成Long.MaxValue  大于spark.sql.autoBroadcastJoinThreshold的值，也就意味着默认情况下不会广播一个表，除非他足够小 |
| spark.sql.shuffle.partitions             | 200                                      | 当为join/aggregation shuffle数据时，默认partition的数量 |
| spark.sql.adaptive.shuffle.targetPostShuffleInputSize | 64 * 1024 * 1024byte                     | The target post-shuffle input size in bytes of a task. |
| spark.sql.adaptive.enabled               | FALSE                                    | 是否开启adaptive query execution（自适应查询执行）    |
| spark.sql.adaptive.minNumPostShufflePartitions | -1                                       | 测试用                                      |
| spark.sql.subexpressionElimination.enabled | TRUE                                     | When true, common subexpressions will be eliminated |
| 当为真时，将删除公共子表达式                           |                                          |                                          |
| spark.sql.caseSensitive                  | FALSE                                    | 查询分析器是否区分大小写，默认情况下不区分。强烈建议不区分大小写         |
| spark.sql.constraintPropagation.enabled  |                                          | 是否开启优化，在查询优化器期间推断和传播查询计划中的数据约束。对于某种类型的查询计划（例如有大量谓语和别名的查询），约束传播是昂贵的，会对整个运行时间产生负面影响。 |
| spark.sql.parser.escapedStringLiterals   | FALSE                                    | 2.0之前默认值为true，知否默认是否。正常文字能否包含在正则表达式中。    |
| spark.sql.parquet.mergeSchema            | FALSE                                    | 若为true,在读取parquet数据源时，schema从所有文件中合并出来。否则如果没有可用的摘要文件，则从概要文件或随机文件中选择模式 |
| spark.sql.parquet.respectSummaryFiles    | FALSE                                    | 若为ture,假设parquet的所有部分文件和概要文件一致，在合并模式时会忽略他们。否则将会合并所有的部分文件 |
| spark.sql.parquet.binaryAsString         | FALSE                                    | 是否向下兼容其他parquet生产系统（eg impala or older version spark sql  ），不区分字节数据和string数据写到parquet schema,这个配置促使spark sql将二进制数据作为string达到兼容 |
| spark.sql.parquet.int96AsTimestamp       | TRUE                                     | 是否使用Int96作为timestamp的存储格式，可以避免精度损失丢失纳秒部分，为其他parquet系统提供兼容（impala) |
| spark.sql.parquet.int64AsTimestampMillis | FALSE                                    | 当为true，timestamp值将以Int64作为mlibs的存储扩展类型，这种模式微秒将被丢弃 |
| spark.sql.parquet.cacheMetadata          | TRUE                                     | 是否缓存parquet的schema数据元，可以提升静态数据的查询性能      |
| spark.sql.parquet.compression.codec      | snappy                                   | 支持类型：uncompressed", "snappy", "gzip",  "lzo"。 指定parquet写文件的压缩编码方式 |
| spark.sql.parquet.filterPushdown         | TRUE                                     | 是否开启parquet过滤条件下推                        |
| spark.sql.parquet.writeLegacyFormat      | FALSE                                    | spark sql在拼接schema时是否遵循parquet的schema的规范 |
| spark.sql.parquet.output.committer.class | org.apache.parquet.hadoop.ParquetOutputCommitter | parquet输出提交器类，同城必须是org.apache.hadoop.mapreduce.OutputCommitter的子类，如果不是将不会创建数据源摘要，即使配置开启了parquet.enable.summary-metadata |
| spark.sql.parquet.enableVectorizedReader | TRUE                                     | 开启parquet向量解码                            |
| spark.sql.orc.filterPushdown             | FALSE                                    | 是否开启条件下推到orc文件写                          |
| spark.sql.hive.verifyPartitionPath       | FALSE                                    | 当为true时，在读取HDFS中存储的数据时，检查表根目录下的所有分区路径    |
| spark.sql.hive.metastorePartitionPruning | TRUE                                     | 当为true，spark sql的谓语将被下推到hive metastore中，更早的消除不匹配的分区，会影响到违背转换成文件源关系的hive表 |
| spark.sql.hive.manageFilesourcePartitions | TRUE                                     | 是否使用hive metastore管理spark sql的  dataSource表分区,若为true，dataSource表会在执行计划期间使用分区剪枝 |
| spark.sql.hive.filesourcePartitionFileCacheSize | 250 * 1024 * 1024                        | 当非0时，开启将分区文件数据元缓存到内存中，所有表共享一个缓存，当开启 hive filesource partition  management（spark.sql.hive.manageFilesourcePartitions）时才会生效 |
| spark.sql.hive.caseSensitiveInferenceMode | INFER_AND_SAVE                           | 设置无法从hive表属性读取分区大小写模式时所采取的操作，虽然Spice  SQL本身不区分大小写，但hive兼容的文件格式如parquet。Spark  sql必须使用一个保持情况的模式，当查询由包含区分大小写字段名或查询的文件支持的任何表可能无法返回准确的结果时。有效选项包括INFER_AND_SAVE(默认模式——从基础数据文件推断出区分大小写的模式，并将其写入表属性），INFER_ONLY（推断schema但不尝试将其写入表属性）和NEVER_INFER（回退到使用区分大小写间接转移模式代替推断) |
| spark.sql.optimizer.metadataOnly         | TRUE                                     | 当为true时，启用仅使用表的元数据的元数据查询优化来生成分区列，而不是表扫描。当扫描的所有列都是分区列，并且查询具有满足不同语义的聚合运算符时，它适用。 |
| spark.sql.columnNameOfCorruptRecord      | _corrupt_record                          | 当json/csv数据内部列解析失败时，失败列的名称               |
| spark.sql.broadcastTimeout"              | 5*60                                     | 在broadCast join时 ，广播等待的超时时间              |
| spark.sql.thriftserver.scheduler.pool    |                                          | 为JDBC客户端会话设置公平调度程序池                      |
| spark.sql.thriftServer.incrementalCollect | FALSE                                    | 当TRUE时，启用增量集合以在thrift server中执行          |
| spark.sql.thriftserver.ui.retainedStatements | 200                                      | JDBC/ODBC Web用户界面历史记录中SQL语句的数量           |
| spark.sql.thriftserver.ui.retainedSessions | 200                                      | JDBC/ODBC Web UI历史中保存的SQL客户端会话数          |
| spark.sql.sources.default                | parquet                                  | 输入输出默认数据元                                |
| spark.sql.hive.convertCTAS               | FALSE                                    | 如果时true，将使用spark.sql.sources.default.设置数据源，不指定任何存储属性到hive ctas语句 |
| spark.sql.hive.gatherFastStats           | TRUE                                     | 在修复表分区时，将快速收集STATS（文件数量和所有文件的总大小），以避免HIVE转移子中的顺序列表。 |
| spark.sql.sources.partitionColumnTypeInference.enabled | TRUE                                     | 是否自动推断分区列的数据类型                           |
| spark.sql.sources.bucketing.enabled      | TRUE                                     | 当false时，分桶表当作普通表处理                       |
| spark.sql.crossJoin.enabled              | FALSE                                    | 当false时，如果查询中语法笛卡儿积 却语法中没有显示join，将会抛出异常  |
| spark.sql.orderByOrdinal                 | TRUE                                     | 当为true时，排序字段放置到seleect List，否则被忽略        |
| spark.sql.groupByOrdinal                 | TRUE                                     | 当为true时，按组子句的序号被视为选择列表中的位置。当为false时，序数被忽略。 |
| spark.sql.groupByAliases                 | TRUE                                     | group by后的别名是否能够被用到 select list中，若为否将抛出分析异常 |
| spark.sql.sources.parallelPartitionDiscovery.threshold | 32                                       | 允许在driver端列出文件的最大路径数。如果在分区发现期间检测到的路径的数量超过该值，则尝试用另一个SCAPLE分布式作业来列出文件。这适用于parquet、ORC、CSV、JSON和LIbSVM数据源。 |
| spark.sql.sources.parallelPartitionDiscovery.parallelism | 10000                                    | 递归地列出路径集合的并行数，设置阻止文件列表生成太多任务的序号          |
| spark.sql.selfJoinAutoResolveAmbiguity   | TRUE                                     | 自动解决子链接中的连接条件歧义，修复bug SPARK-6231         |
| spark.sql.retainGroupColumns             | TRUE                                     | 是否保留分组列                                  |
| spark.sql.pivotMaxValues                 | 10000                                    |                                          |
| spark.sql.runSQLOnFiles                  | TRUE                                     | 当为true,在sql查询时，能够使用dataSource.path作为表(eg:"select a,b from  hdfs://xx/xx/*") |
| spark.sql.codegen.wholeStage             | TRUE                                     | 当为true,多个算子的整个stage将被便宜到一个java方法中        |
| spark.sql.codegen.maxFields              | 100                                      | 在激活整个stage codegen之前支持的最大字段（包括嵌套字段)      |
| spark.sql.codegen.fallback               | TRUE                                     | 当为true,在整个stage的codegen,对于编译generated code 失败的query 部分，将会暂时关闭 |
| spark.sql.codegen.maxCaseBranches        | 20                                       | 支持最大的codegen                             |
| spark.sql.files.maxPartitionBytes        | 128 * 1024 * 1024                        | 在读取文件时，一个分区最大被读取的数量，默认值=parquet.block.size |
| spark.sql.files.openCostInBytes          | 4 * 1024 * 1024                          | 为了测定打开一个文件的耗时，通过同时扫描配置的字节数来测定，最好是过度估计，那么小文件的分区将比具有较大文件的分区更快（首先调度 |
| spark.sql.files.ignoreCorruptFiles       | FALSE                                    | 是否自动跳过不正确的文件                             |
| spark.sql.files.maxRecordsPerFile        | 0                                        | 写入单个文件的最大条数，如果时0或者负数，则无限制                |
| spark.sql.exchange.reuse                 | TRUE                                     | planer是否尝试找出重复的 exchanges并复用             |
| spark.sql.streaming.stateStore.minDeltasForSnapshot | 10                                       | 在合并成快照之前需要生成的状态存储增量文件的最小数目               |
| spark.sql.streaming.checkpointLocation   |                                          | 检查点数据流的查询的默认存储位置                         |
| spark.sql.streaming.minBatchesToRetain   | 100                                      | 流式计算最小批次长度                               |
| spark.sql.streaming.unsupportedOperationCheck | TRUE                                     | streaming query的logical plan 检查不支持的操作    |
| spark.sql.variable.substitute            | TRUE                                     |                                          |
| spark.sql.codegen.aggregate.map.twolevel.enable |                                          | 启用两级聚合哈希映射。当启用时，记录将首先“插入/查找第一级、小、快的映射，然后在第一级满或无法找到键时回落到第二级、更大、较慢的映射。当禁用时，记录直接进入第二级。默认为真 |
| spark.sql.view.maxNestedViewDepth        | 100                                      | 嵌套视图中视图引用的最大深度。嵌套视图可以引用其他嵌套视图，依赖关系被组织在有向无环图（DAG）中。然而，DAG深度可能变得太大，导致意外的行为。此配置限制了这一点：当分析期间视图深度超过该值时，我们终止分辨率以避免潜在错误。 |
| spark.sql.objectHashAggregate.sortBased.fallbackThreshold | 128                                      | 在ObjectHashAggregateExec的情况下，当内存中哈希映射的大小增长过大时，我们将回落到基于排序的聚合。此选项为哈希映射的大小设置行计数阈值。 |
| spark.sql.execution.useObjectHashAggregateExec | TRUE                                     | 是否使用 ObjectHashAggregateExec             |
| spark.sql.streaming.fileSink.log.deletion | TRUE                                     | 是否删除文件流接收器中的过期日志文件                       |
| spark.sql.streaming.fileSink.log.compactInterval | 10                                       | 日志文件合并阈值，然后将所有以前的文件压缩到下一个日志文件中           |
| spark.sql.streaming.fileSink.log.cleanupDelay | 10min                                    | 保证一个日志文件被所有用户可见的时长                       |
| spark.sql.streaming.fileSource.log.deletion | TRUE                                     | 是否删除文件流源中过期的日志文件                         |
| spark.sql.streaming.fileSource.log.compactInterval | 10                                       | 日志文件合并阈值，然后将所有以前的文件压缩到下一个日志文件中           |
| spark.sql.streaming.fileSource.log.cleanupDelay | 10min                                    | 保证一个日志文件被所有用户可见的时长                       |
| spark.sql.streaming.schemaInference      | FALSE                                    | 基于文件的流,是否推断它的模式                          |
| spark.sql.streaming.pollingDelay         | 10L(MILLISECONDS)                        | 在没有数据可用时延迟查询新数据多长时间                      |
| spark.sql.streaming.noDataProgressEventInterval | 10000L(MILLISECONDS)                     | 在没有数据的情况下，在两个进度事件之间等待时间                  |
| spark.sql.streaming.metricsEnabled       | FALSE                                    | 是否为活动流查询报告DoopWalth/CODAHALE度量           |
| spark.sql.streaming.numRecentProgressUpdates | 100                                      | streaming query 保留的进度更新数量                |
| spark.sql.statistics.ndv.maxError        | 0.05                                     | 生成列级统计量时超对数G+++算法允许的最大估计误差               |
| spark.sql.cbo.enabled                    | FALSE                                    | 在设定true时启用CBO来估计计划统计信息                   |
| spark.sql.cbo.joinReorder.enabled        | FALSE                                    | Enables join reorder in CBO.             |
| spark.sql.cbo.joinReorder.dp.threshold   | 12                                       | The maximum number of joined nodes allowed in the dynamic programming  algorithm |
| spark.sql.cbo.joinReorder.card.weight    | 0.07                                     | The weight of cardinality (number of rows) for plan cost comparison in  join reorder: rows * weight + size * (1 - weight) |
| spark.sql.cbo.joinReorder.dp.star.filter | FALSE                                    | Applies star-join filter heuristics to cost based join enumeration |
| spark.sql.cbo.starSchemaDetection        | FALSE                                    | When true, it enables join reordering based on star schema detection |
| spark.sql.cbo.starJoinFTRatio            | 0.9                                      | Specifies the upper limit of the ratio between the largest fact  tables for a star join to be considered |
| spark.sql.session.timeZone               | TimeZone.getDefault.getID                | 时间时区                                     |
| spark.sql.windowExec.buffer.in.memory.threshold | 4096                                     | 窗口操作符保证存储在内存中的行数的阈值                      |
| spark.sql.windowExec.buffer.spill.threshold | spark.sql.windowExec.buffer.in.memory.threshold | 窗口操作符溢出的行数的阈值                            |
| spark.sql.sortMergeJoinExec.buffer.in.memory.threshold | Int.MaxValue                             | 由sortMergeJoin运算符保证存储在内存中的行数的阈值          |
| spark.sql.sortMergeJoinExec.buffer.spill.threshold | spark.sql.sortMergeJoinExec.buffer.in.memory.threshold | 由排序合并连接运算符溢出的行数的阈值                       |
| spark.sql.cartesianProductExec.buffer.in.memory.threshold | 4096                                     | 笛卡尔乘积算子保证存储在内存中的行数的阈值                    |
| spark.sql.cartesianProductExec.buffer.spill.threshold | spark.sql.cartesianProductExec.buffer.in.memory.threshold | 笛卡尔乘积算子溢出的行数阈值                           |
| spark.sql.redaction.options.regex        | (?i)url.r                                |                                          |

