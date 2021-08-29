---
title: Flink与Spark对比
abbrlink: 3864
date: 2021-08-18 23:31:15
tags: Flink
categories: Flink
summary_img:
encrypt:
enc_pwd:
---

## 一 应用场景

```
spark:离线批处理,对延迟要求不高的实时处理(微批),ds和df也支持流批一体
Flink:实时处理,Flink1.12开始支持流批一体
```

## 二 API

```
spark:rdd(不推荐)/dstream(不推荐)/df/ds
flink:ds(1.12软弃用)/df(主推)/table&sql(发展中)
```

## 三 核心组件/流程原理

### spark

![img](/images/flinkk/s2.png)

![img](/images/flinkk/s1.png)

### flink

![img](/images/flinkk/f1.png)

![img](/images/flinkk/f2.png)

![img](/images/flinkk/f3.png)

## 四 事件机制

```
spark:sparkstreaming只支持处理时间,structuredStreaming开始支持事件时间 

flink:事件时间/处理时间/摄入时间
```

## 五 容错机制

```
spark:缓存/持久化+checkpoint(应用级别) structuredStreaming开始借鉴flink使用Chandy-Lamport algorithm分布快照式算法

flink: state+checkpoint(operator级别)+自动重启策略+savepoint
```

## 六 窗口

```
flink:基于时间/数量的滚动,滑动窗口和会话窗口

spark:基于时间/数量的滚动,滑动窗口 要求窗口时间和滑动时间必须是batchDurtion(批处理时间)的倍数

```

## 七 整合kafka

```
spark:支持offset手动/自动维护  支持动态分区监测,无需配置
flink:支持offset手动/自动维护(一般自动有checkpoint维护即可)
	支持动态分区监测,需要配置:
	Flink Kafka Consumer支持动态发现Kafka分区，且能保证exactly-once。 
默认禁止动态发现分区，把flink.partition-discovery.interval-millis设置大于0即可启用：
properties.setProperty(“flink.partition-discovery.interval-millis”, “3000”) //会开启一个后台线程每隔3秒监测kafka的分区情况
```

## 八 流式实现原理

```
sparkstreaming:微批
flink:基于事件的流式处理
	flink可以数据来一条处理一条然后发给下游,真正流处理,但为了平衡低延迟和吞吐量支持如下机制:
		数据来一条处理一条不变,可以攒够一个缓存快在发送给下游处理或达到一定时间阈值发送给下游 流失处理折中方案 
		但和spark的微批有本质区别(spark 是数据来了攒够一批在处理,并且每一批都要生成dag等流程)
		env.setBufferTimeout(3) //默认100毫秒
		taskmanager.memory.segment-size=32  //默认32kb
```

## 九 背压/反压 backPressure

```
spark与kafka有反压: 
		通过PIDRateEsimator 通过pid算法实现一个速率评估器(统计调度时间,任务处理时间,数据条数,得出一个消息处理最大速率,进而调整根据offset从kafka消费数据的速率 (是spark取拉kafka的数据))

flink:
		1.5之后是自己实现了一个credit-based流控机制(1.5之前的背压解决方案有问题),在应用层模拟tcp的流控机制,就是ResultSubPartition向InputChanel发送消息的时候会发送一个backlog size告诉下游准备发送多少消息,下游会计算需要多少buffer去接受消息,算完之后若有充足的buffer就会返回一个credit,告知他可以发送消息,若是没有返回凭证上游等待下次通信返回,jobmanager针对每一个task没50ms发送100次 Thrad.getStackTrace()调用,求出每个task阻塞的占比
		阻塞占比:
		ok:0<= ratio <= 0.1 良好
		low:0.1<ratio <= 0.5 有待观察
		high: 0.5<ratio <=1 要处理 (增加并行度/subtask/是否数据倾斜/增加内存....)
		这些信息在flink的监控平台可以看
```



