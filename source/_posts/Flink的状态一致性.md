---
title: Flink的状态一致性
tags: Flink
categories: Flink
abbrlink: 33641
date: 2020-11-29 16:19:14
summary_img:
encrypt:
enc_pwd:
---

## 一 状态一致性

### 1.1 状态一致性简介与级别

#### 1.1.1简介

当在分布式系统中引入状态时，自然也引入了一致性问题。一致性实际上是"正确性级别"的另一种说法，也就是说在成功处理故障并恢复之后得到的结果，与没有发生任何故障时得到的结果相比，前者到底有多

正确？举例来说，假设要对最近一小时登录的用户计数。在系统经历故障之后，计数结果是多少？如果有偏差，是有漏掉的计数还是重复计数？

有状态的流处理，内部每个算子任务都可以有自己的状态；

对于流处理器内部（没有接入sink）来说，所谓的状态一致性，其实就是我们所说的计算结果要保证准确；一条数据不应该丢失，也不应该重复计算；

在遇到故障时可以恢复状态，恢复以后的重新计算，结果应该也是完全正常的；

#### 1.1.2状态一致性级别

```
1:AT_MOST_ONCE（最多一次），当任务故障时最简单做法是什么都不干，既不恢复丢失状态，也不重播丢失数据。At-most-once语义的含义是最多处理一次事件。
2:AT_LEAST_ONCE（至少一次），在大多数真实应用场景，我们希望不丢失数据。这种类型的保障称为at-least-once，意思是所有的事件都得到了处理，而一些事件还可能被处理多次。
3:EXACTLY_ONCE（精确一次），恰好处理一次是最严格的的保证，也是最难实现的。恰好处理一次语义不仅仅意味着没有事件丢失，还意味着针对每一个数据，内部状态仅仅更新一次。	
```

曾经，at-least-once非常流行。第一代流处理器(如Storm和Samza)刚问世时只保证at-least-once，原因有二。

- 保证exactly-once的系统实现起来更复杂。这在基础架构层(决定什么代表正确，以及exactly-once的范围是什么)和实现层都很有挑战性。
- 流处理系统的早期用户愿意接受框架的局限性，并在应用层想办法弥补(例如使应用程序具有幂等性，或者用批量计算层再做一遍计算)。

最先保证exactly-once的系统(Storm Trident和Spark Streaming)在性能和表现力这两个方面付出了很大的代价。为了保证exactly-once，这些系统无法单独地对每条记录运用应用逻辑，而是同时处理多条(一批)记录，保证对每一批的处理要么全部成功，要么全部失败。这就导致在得到结果前，必须等待一批记录处理结束。因此，用户经常不得不使用两个流处理框架(一个用来保证exactly-once，另一个用来对每个元素做低延迟处理)，结果使基础设施更加复杂。曾经，用户不得不在保证exactly-once与获得低延迟和效率之间权衡利弊。Flink避免了这种权衡。Flink的一个重大价值在于，**它既保证了****exactly-once****，也具有低延迟和高吞吐的处理能力**。

从根本上说，Flink通过使自身满足所有需求来避免权衡，它是业界的一次意义重大的技术飞跃。

## 二 端到端的状态一致性

目前我们看到的一致性保证都是由流处理器实现的，也就是说都是在Flink流处理内部保证的；而在真实应用中，流处理应用除了流处理器以外还包含了数据源（例如kafka）和输出到持久化系统； 端到端的一致性保证，意味着结果的正确性贯穿了整个流处理应用的始终，每个组件都保证了它自己的一致性；

整个端到端的一致性级别取决于所有组件中一致性最弱的组件；具体划分如下：

- 内部保证 --- checkpoint
- source端 --- 可重设数据的读取位置；**可重新读取偏移量**
- sink端 -- 从故障恢复时，数据不会重复写入外部系统，有两种实现方式：幂等（Idempotent）写入和事务性（Transactional）写入；

**1 幂等写入（Idempotent Writes）**

   幂等操作即一个操作可以重复执行很多次，但只导致一次结果更改，也就是说，后面再重复执行就不起作用了；

![img](/images/flinkstate/md.png)

它的原理不是不重复写入而是重复写完之后结果还是一样；它的瑕疵是不能做到完全意义上exactly-once（在故障恢复时，突然把外部系统写入操作跳到之前的某个状态然后继续往里边写，故障之前发生的那一段的状态变化又重演了直到最后发生故障那一刻追上就正常了；假如中间这段又被读取了就可能会有些问题）；

```
如1，2，3，5（checkpoint）,8,6,7（故障），9
则5，8，6，7，8，6，7，9
```



**2 事务写入（Transactional Writes）**

- 应用程序中一系列严密的操作，所有操作必须成功完成，否则在每个操作中所作的所有更改都会被撤销；
- 具有原子性，一个事务中的一系列的操作要么全部成功，要么一个都不做。

实现思想：需要构建事务来写入外部系统，构建的事务对应着checkpoint，等到checkpoint真正完成的时候，才把所有对应的结果写入sink系统中。

实现方式：预写日志（WAL）和两阶段提交（2PC）；

DataStream API 提供了GenericWriteAheadSink模板类和TwoPhaseCommitSinkFunction 接口，可以方便地实现这两种方式的事务性写入。

**预写日志（Write-Ahead-Log，WAL）**

把结果数据先当成状态保存，然后在收到checkpoint完成的通知时，一次性写入sink系统；简单易于实现，由于数据提前在状态后端中做了缓存，所以无论什么sink系统，都能用这种方式一批搞定；DataStream API提供了一个模板类：GenericWriteAheadSink，来实现这种事务性sink；

瑕疵：A. sink系统没说它支持事务，有可能出现一部分写进去了一部分没写进去（如果算失败，再写一次就写了两次了）；

           B. checkpoint做完了sink才去真正写入（但其实得等sink都写完checkpoint才能生效，所以WAL这个机制jobmanager确定它写完还不算真正写完，还得有一个外部系统已经确认完成的checkpoint）

**两阶段提交（Two--Phase--Commit，2PC）-- 真正能够实现exactly-once**

对于每个checkpoint，sink任务会启动一个事务，并将接下来所有接收的数据添加到事务里；然后将这些数据写入外部sink系统，但不提交他们 -- 这时只是预提交；当它收到checkpoint完成时的通知，它才正式提交事务，实现结果的真正写入；这种方式真正实现了exactly-once，它需要一个提供事务支持的外部sink系统，Flink提供了TwoPhaseCommitSinkFunction接口。

**2PC对外部sink的要求**

1:外部sink系统必须事务支持，或者sink任务必须能够模拟外部系统上的事务；

2:在checkpoint的间隔期间里，必须能够开启一个事务，并接受数据写入；

3:在收到checkpoint完成的通知之前，事务必须是“等待提交”的状态，在故障恢复的情况下，这可能需要一些时间。如果这个时候sink系统关闭事务（例如超时了），那么未提交的数据就会丢失；

4:sink任务必须能够在进程失败后恢复事务；

5:提交事务必须是幂等操作；

**不同Source和sink的一致性保证**

| sink\source | 不可重置         | 可重置                         |
| ----------- | ------------ | --------------------------- |
| 任意（any）     | At-most-once | At-most-once                |
| 幂等          | At-most-once | exactly-once(故障恢复时会出现短暂不一致) |
| 预写日志（wal）   | At-most-once | At-most-once                |
| 两阶段提交（2pc）  | At-most-once | exactly-once                |

## 三 一致性检查点（checkpoint）

### 3.1 checkpoint的机制

Flink使用了一种轻量级快照机制 --- 检查点（Checkpoint）来保证exactly-one语义，在出现故障时将系统重置回正确状态；有状态流应用的一致检查点，其实就是：所有任务的状态，在某个时间点的一份拷贝（一份快照）。而这个时间点，应该是所有任务都恰好处理完一个相同的输入数据的时候。

应用状态的一致性检查点，是Flink故障恢复机制的核心。

![img](/images/flinkstate/ck.png)

### 3.2 Flink+kafka端到端的exactly-once语义

**Flink和kafka天生就是一对，用kafka做为source，用kafka做完sink  <===>  实现端到端的一致性**

- 内部 -- 利用checkpoint机制，把状态存盘，发生故障的时候可以恢复，保证内部的状态一致性；
- source -- kafka consumer作为source，可以将偏移量保存下来，如果后续任务出现了故障，恢复的时候可以由连接器重置偏移量，重新消费数据，保证一致性；
- sink -- kafka producer作为sink，采用两阶段提交sink，需要实现一个TwoPhaseCommitSinkFunction

内部的checkpoint机制见上；source和 sink见下：

```
public class FlinkKafkaProducer011<IN>
	extends TwoPhaseCommitFunction<....>
```



Flink由JobManager协调各个TaskManager进行checkpoint存储，checkpoint保存在 StateBackend中，默认StateBackend是内存级的，也可以改为文件级的进行持久化保存。

![img](/images/flinkstate/pc1.png)

当 checkpoint 启动时，JobManager 会将检查点分界线（barrier）注入数据流；barrier会在算子间传递下去。

![img](/images/flinkstate/pc2.png)

每个算子会对当前的状态做个快照，保存到状态后端；

对于source任务而言，就会把当前的offset作为状态保存起来。下次从checkpoint恢复时，source任务可以重新提交偏移量，从上次保存的位置开始重新消费数据。

![img](/images/flinkstate/pc3.png)

 每个内部的 transform 任务遇到 barrier 时，都会把状态存到 checkpoint 里。

sink 任务首先把数据写入外部 kafka，这些数据都属于预提交的事务（还不能被消费）；当遇到 barrier 时，把状态保存到状态后端，并开启新的预提交事务。（以barrier为界之前的数

据属于上一个事务，之后的数据属于下一个新的事务）；

![img](/images/flinkstate/pc4.png)

 当所有算子任务的快照完成，也就是这次的 checkpoint 完成时，JobManager 会向所有任务发通知，确认这次 checkpoint 完成。

当sink 任务收到确认通知，就会正式提交之前的事务，kafka 中未确认的数据就改为“已确认”，数据就真正可以被消费了。

![img](/images/flinkstate/pc5.png)

所以我们看到，执行过程实际上是一个两段式提交，每个算子执行完成，会进行“预提交”，直到执行完sink操作，会发起“确认提交”，如果执行失败，预提交会放弃掉。

**具体的两阶段提交步骤总结如下：**

- 第一条数据来了之后，开启一个 kafka 的事务（transaction），正常写入 kafka 分区日志但标记为未提交，这就是“预提交”
- jobmanager 触发 checkpoint 操作，barrier 从 source 开始向下传递，遇到 barrier 的算子将状态存入状态后端，并通知 jobmanager
- sink 连接器收到 barrier，保存当前状态，存入 checkpoint，通知 jobmanager，并开启下一阶段的事务，用于提交下个检查点的数据
- jobmanager 收到所有任务的通知，发出确认信息，表示 checkpoint 完成
- sink 任务收到 jobmanager 的确认信息，正式提交这段时间的数据
- 外部kafka关闭事务，提交的数据可以正常消费了。

所以我们也可以看到，如果宕机需要通过StateBackend进行恢复，只能恢复所有确认提交的操作。

在代码中真正实现flink和kafak的端到端exactly-once语义：

```
FlinkKafkaProducer011 内部有个枚举 默认为AT_LEAST_ONCE；
```

1.这里需要配置下，因为它默认的是AT_LEAST_ONCE；

2.对于外部kafka读取的消费者的隔离级别，默认是read_on_commited，如果默认是可以读未提交的数据，就相当于整个一致性还没得到保证（未提交的数据没有最终确认那边就可以读了，相当于那边已经消费数据了，事务就是假的了）  所以需要修改kafka的隔离级别；

3.timeout超时问题，flink和kafka 默认sink是超时1h，而kafak集群中配置的tranctraction事务的默认超时时间是15min，flink-kafak这边的连接器的时间长，这边还在等着做操作 ，kafak那边等checkpoint等的时间太长直接关闭了。所以两边的超时时间最起码前边要比后边的小。

