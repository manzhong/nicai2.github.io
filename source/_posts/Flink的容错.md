---
title: Flink的容错
tags: Flink
categories: Flink
abbrlink: 50420
date: 2020-11-25 21:24:19
summary_img:
encrypt:
enc_pwd:
---

## 一 一致性检查点

![checkpoint](/images/flinkk/checkpoint.png)

```
  如上图sum_even （2+4），sum_odd（1 + 3 + 5），5这个数据之前的都处理完了，就出保存一个checkpoint；Source任务保存状态5，sum_event任务保存状态6，sum_odd保存状态是9；这三个保存到状态后端中就构成了CheckPoint；
```

```
1:Flink故障恢复机制的核心，就是应用状态的一致性检查点；
2:有状态流应用的一致性检查点（checkpoint），其实就是所有任务的状态，在某个时间点的一份拷贝（一份快照）；这个时间点，应该是所有任务都恰好处理完一个相同的输入数据的时候 。（这个同一时间点并不是物理上的在同一时刻）
```



## 二 从检查点恢复状态

![checkpoint](/images/flinkk/2.png)

```
 sum_even（2 + 4 + 6）；sum_odd（1 + 3 + 5）； 
```

 1:在执行应用程序期间，Flink会定期保存状态的一致性检查点；

 2:如果发生故障，Flink将会使用**最近的检查点**来一致恢复应用程序的状态，并重新启动处理流程；

![checkpoint](/images/flinkk/3.png)

 遇到故障之后，第一步就是重启应用；

![checkpoint](/images/flinkk/4.png)

 第二步是从checkpoint中读取状态，将状态重置；

​			从检查点重新启动应用程序后，其内部状态与检查点完成时的状态完全相同；

![checkpoint](/images/flinkk/5.png)

第三步 ：开始消费并处理检查点到发生故障之间的所有数据；

这种检查点的保存和恢复机制可以为应用程序状态提供“精确一次”（exactly-once）的一致性，因为所有算子都会保存检查点并恢复其所有状态，这样一来所有的输入流就都会被重置到检查点完成时的位置。

## 三 Flink的检查点算法

```
1:简单：暂停应用，保存状态到检查点，再重新恢复应用；
2:Flink的改进：
		2.1:基于Chandy-Lamport算法的分布式快照；
		2.2:将检查点的保存和数据处理分离开，不暂停整个应用；
```

### 3.1检查点分界线（CheckPoint Barrier）

```
1:Flink的检查点算法用到了一种称为分界线（barrier）的特殊数据形式，用来把一条流上数据按照不同的检查点分开；
2:分界线之前到来的数据导致的状态更改，都会被包含在当前分界线所属的检查点中；而基于分界线之后的数据导致的所有更改，就会被包含在之后的检查点中;

只有一个并行时: 会在source段要做checkpoint的数据后面插入一个barrier,并将这个barrier发向下游,则下游接受到的数据顺序是一致的 
若是多个并行子任务:就涉及barrier对齐,一下是介绍
```

![checkpoint](/images/flinkk/6.png)

现在是一个有两个输入流的应用程序，用并行的两个Source任务来读取：

两个并行输入源按奇偶数来做sum，类似keyBy重分区map为二元组再做奇偶keyBy，Sum odd（**1** + **1 + 3**），Sum even（**2**）

![checkpoint](/images/flinkk/7.png)

JobManager会向每个source任务发送一条带有**新检查点ID**的消息，通过这种方式来启动检查点；

![checkpoint](/images/flinkk/8.png)

数据源将它们的状态写入检查点，并发出一个检查点barrier；

状态后端在状态存入检查点之后，会返回通知给source任务，source任务就会向JobManager确认检查点完成。

source1和source2收到检查点ID = 2时，分别存入自己的偏移量蓝3和黄4，存完之后返回一个ID2通知JobManager快照已保存好；（在保存快照时它会暂停发送和处理数据，同事它也会向下游发送带有检查点ID的barrier，发送的方式直接广播；这个过程中Sum和sink任务也没闲着都在处理数据）

![checkpoint](/images/flinkk/9.png)

**分界线对齐（barrier对齐）**：barrier向下游传递，sum任务会等待所有输入分区的的barrier到达；

对于barrier已经到达的分区，继续到达的数据会被缓存；而barrier尚未到达的分区，数据会被正常处理；

```
（比如蓝2通知给了Sum even，它会等黄2的barrier到达，这时处理的数据4来了，会先被缓存因为它数据下一个checkpoint的数据； 黄2的checkpoint还没来这时它如果来数据还会正常处理更改状态，如上图的在黄2的barrier还没来之前，source2的数据来了条4，它会正常处理Sum event（**2** + **2 +** **4**））
```

![checkpoint](/images/flinkk/10.png)

当收到所有输入分区的barrier时，任务就将其状态保存到状态后端的检查点中，然后将barrier继续向下游转发。

**barrier对齐**之后（Sum even和Sum odd都接收到了两个source发来的barrier），将它们各自的8状态存入checkpoint中；接下来继续向下游Sink广播barrier；

![checkpoint](/images/flinkk/11.png)

向下游转发检查点的barrier后，任务继续正常的数据处理；

先处理缓存的数据，蓝4加载进来Sum event 12，黄6进来Sum event 18。

![checkpoint](/images/flinkk/12.png)

 1:Sink任务向JobManager确认状态保存到checkpoint完毕；（Sink接收到barrier后先保存状态到checkpoint，然后向JobManager汇报;

2:当所有任务都确认已成功将状态保存到检查点时，检查点就真正完成了。

## 四 checkpoint的配置

```scala
	 val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
    //Flink的checkpoint默认是关闭的使用需要手动开启 源码中默认的checkpoint的时间是500毫秒 也可以手动传,还可以有第二个参数mode 指定计算级别 EXACTLY_ONCE(默认),
    // 或AT_LEAST_ONCE(要求精确不高,且快的时候用到)
    env.enableCheckpointing(1000,CheckpointingMode.EXACTLY_ONCE)
    //mode
    env.getCheckpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE)
    //超时时间 1秒
    env.getCheckpointConfig.setCheckpointTimeout(1000)
    //异步checkpoint 需要在状态后端开启异步快照 FsStateBackend有个可选参数
    //异步快照其实是将当前的状态copy一份副本放在内存 ,其本身的状态就可以更改了,写checkpoint 时用副本,真实的状态后续继续处理
    //public FsStateBackend(String checkpointDataUri, boolean asynchronousSnapshots) {
    //  this(new Path(checkpointDataUri), asynchronousSnapshots);
    //}
    env.setStateBackend(new FsStateBackend(""))
    //最大会同时出现几个checkpoint在进行 不配默认为1个
    env.getCheckpointConfig.setMaxConcurrentCheckpoints(2)
    //两次checkpoint之间的最小间隔 前一个checkpoint的尾到下个的头的间隔时间
    // 而enableCheckpointing 的时间是两个checkpoint头到头的时间,不一定为配置时间,因为所有的状态保存成功才算一次checkpoing
    //这个配置会覆盖setMaxConcurrentCheckpoints
    env.getCheckpointConfig.setMinPauseBetweenCheckpoints(500)
    //是否用checkpoint做故障恢复 若为true则不会判断savepoint默认false 则savepoint和checkpoint谁近用谁
    env.getCheckpointConfig.setPreferCheckpointForRecovery(true)
    //容忍多少次checkpoint失败 若是设置为0 则checkpoint失败 job失败
    env.getCheckpointConfig.setTolerableCheckpointFailureNumber(4)
    // checkpoint挂了 会看做job失败 默认false 被setTolerableCheckpointFailureNumber替换
    env.getCheckpointConfig.setFailOnCheckpointingErrors(true)
    //开启外部持久化，即使job失败也不会清洗checkpoint
 env.getCheckpointConfig.enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.DELETE_ON_CANCELLATION) 
```

## 五 重启策略配置

```scala
import org.apache.flink.api.common.time.Time

//重启策略
    //固定时间间隔重启 传入重启的次数和间隔 两次重启的间隔
    //env.setRestartStrategy(RestartStrategies.fixedDelayRestart(2,10000))
    // 1:重启次数,2:失败的时间间隔,3:两次重启的间隔 (总体都是在 失败的时间间隔内(2))
env.setRestartStrategy(RestartStrategies.failureRateRestart(3,Time.of(5,TimeUnit.MINUTES),Time.of(10,TimeUnit.SECONDS)))
```

## 六 保存点(SavePoints)

```
1:Flink还提供了可以自定义的镜像保存功能，就是保存点（savepoints）；
2:原则上，创建保存点使用的算法与检查点完全相同，因此保存点可以认为就是具有一些额外元数据的检查点；
3:Flink不会自动创建保存点，因此用户（或者外部调度程序）必须明确地触发创建操作；
4:保存点是一个强大的功能，除了故障恢复外，保存点可以用于：有计划的手动备份，更新应用程序，版本迁移，暂停或重启应用，等等

savepoint的状态恢复要保证状态不变和算子不变,状态不变:名字和类型;算子不变:其实是当前算子的uid不变,可以设置算子的uid,不设置flink会自动分配,最好自己配
map(...).uid("map_id")

```



