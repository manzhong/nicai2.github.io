---
title: Flink的Stream_Window
tags: Flink
categories: Flink
encrypt: 
enc_pwd: 
abbrlink: 813
date: 2020-10-29 19:24:27
summary_img:
---

## 一 简介

​     目前有许多数据分析的场景从批处理到流处理的演变， 虽然可以将批处理作为流处理的特殊情况来处理，但是分析无穷集的流数据通常需要思维方式的转变并且具有其自己的术语（例如，“windowing（窗口化）”、“at-least-once（至少一次）”、“exactly-once（只有一次）” ）。

对于刚刚接触流处理的人来说，这种转变和新术语可能会非常混乱。 Apache Flink 是一个为生产环境而生的流处理器，具有易于使用的 API，可以用于定义高级流分析程序。

Flink 的 API 在数据流上具有非常灵活的窗口定义，使其在其他开源流处理框架中脱颖而出。

接下来，我们将讨论用于流处理的窗口的概念，介绍 Flink 的内置窗口，并解释它对自定义窗口语义的支持。

### 1 何为window

下面我们结合一个现实的例子来说明。

就拿交通传感器的示例：统计经过某红绿灯的汽车数量之和？

假设在一个红绿灯处，我们每隔 15 秒统计一次通过此红绿灯的汽车数量，如下图：

```java
->9,10,2,3,5,11,29, -> out
```

可以把汽车的经过看成一个流，无穷的流，不断有汽车经过此红绿灯，因此无法统计总共的汽车数量。但是，我们可以换一种思路，每隔 15 秒，我们都将与上一次的结果进行 sum 操作（滑动聚合），如下：

```java
->9,10,2,3,5,2,3,1 -> rolling sum -> 9,19,21,24,29,31,34,25 -> out  
```

这个结果似乎还是无法回答我们的问题，根本原因在于流是无界的，我们不能限制流，但可以在有一个有界的范围内处理无界的流数据。

因此，我们需要换一个问题的提法：每分钟经过某红绿灯的汽车数量之和？
这个问题，就相当于一个定义了一个 Window（窗口），window 的界限是1分钟，且每分钟内的数据互不干扰，因此也可以称为翻滚（不重合）窗口，如下图：

```
->9,10,2,3,5,2,3,1,3
window (9,10,2) (3,5,2) (3,1,3) -> sum 21 10 7  -> out  
```

第一分钟的数量为21，第二分钟是10，第三分钟是7。。。这样，1个小时内会有60个window。

再考虑一种情况，每30秒统计一次过去1分钟的汽车数量之和：

此时，window 出现了重合。这样，1个小时内会有120个 window。

扩展一下，我们可以在某个地区，收集每一个红绿灯处汽车经过的数量，然后每个红绿灯处都做一次基于1分钟的window统计，即并行处理

### 2 作用

   通常来讲，Window 就是用来对一个无限的流设置一个有限的集合，在有界的数据集上进行操作的一种机制。window 又可以分为基于时间（Time-based）的 window 以及基于数量（Count-based）的 window。还有基于session的window,或自定义window

## 二 window

   Flink DataStream API 提供了 Time 和 Count 的 window，同时增加了基于 Session 的 window。同时，由于某些特殊的需要，DataStream API 也提供了定制化的 window 操作，供用户自定义 window。

   下面，主要介绍 Time-Based window 以及 Count-Based window，以及自定义的 window 操作，Session-Based Window 操作将会在后续的文章中讲到。

### 1 基于time的windows

```scala
//原始写法
        val environment = StreamExecutionEnvironment.getExecutionEnvironment
    //environment.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
    //其他
    //environment.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime)
    //environment.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime)
	val ds: DataStream[String] = environment.fromElements("123","456")
    ds.keyBy(da => da)
      //滚动事件时间窗口,第一个参数窗口大小,第二个offset 偏移量,8:30 -9:30  一般用于时区的转化
      .window(TumblingEventTimeWindows.of(Time.hours(1),Time.minutes(30)))
    ds.keyBy(da => da)
      //滑动事件时间窗口,第一个参数窗口大小,第二个参数滑动窗口大小,第三个offset 偏移量,8:30 -9:30  一般用于时区的转化
      .window(SlidingEventTimeWindows.of(Time.hours(1),Time.minutes(30),Time.minutes(30)))
    //val myOutputTag = new OutputTag()
    ds.keyBy(da => da)
      //会话时间窗口,会话的大小 大于这个时间的为下一个会话
      .window(EventTimeSessionWindows.withGap(Time.seconds(10)))
//其中要是使用offset和会话时间窗口 必须使用以上写法  若是其他可以写为 timeWindow
```

正如命名那样，Time Windows 根据时间来聚合流数据。例如：一分钟的 tumbling time window 收集一分钟的元素，并在一分钟过后对窗口中的所有元素应用于一个函数。

在 Flink 中定义 tumbling time windows(翻滚时间窗口) 和 sliding time windows(滑动时间窗口) 非常简单：

#### 1.1**tumbling time windows(翻滚时间窗口,无重叠数据)**

输入一个时间参数

```scala
val environment = StreamExecutionEnvironment.getExecutionEnvironment
//时间语义
//environment.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
//其他
//environment.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime)
//environment.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime)
data.keyBy(1)
	.timeWindow(Time.minutes(1)) //tumbling time window 每分钟统计一次数量和  
	.sum(1);
```

#### 1.2**sliding time windows(滑动时间窗口,有重叠数据)**

输入两个时间参数

1. 如果窗口滑动时间 > 窗口时间，会出现数据丢失
2. 如果窗口滑动时间 < 窗口时间，会出现数据重复计算,比较适合实时排行榜
3. 如果窗口滑动时间 = 窗口时间，数据不会被重复计算

```scala
data.keyBy(1)
	.timeWindow(Time.minutes(1), Time.seconds(30)) //sliding time window 每隔 30s 统计过去一分钟的数  量和  
	.sum(1);
```

有一点我们还没有讨论，即“收集一分钟的元素”的确切含义，它可以归结为一个问题，“流处理器如何解释时间?”

#### 1.3 会话窗口

```
1:有一系列时间组合一个指定时间长度的timeout间隙组成,也就是一段时间没有接受到新的数据就会生成新的窗口
2: 时间无对齐
```

```scala
    ds.keyBy(da => da)
      //会话时间窗口,会话的大小 大于这个时间的为下一个会话
      .window(EventTimeSessionWindows.withGap(Time.seconds(10)))
```

#### 1.4 窗口的函数

```
//1 增量聚合函数
	每条数据到来就进行计算,保持一个简单的状态
	如:ReduceFunction,AggregateFunction
//2 全窗口函数
	先把窗口的所有数据收集起来,等到计算的时候会遍历所有的数据
	如:ProcessWindowFunction
```

```tex
//其他api
1: trigger() 触发器
	定义window什么时候关闭,触发计算并输出结果
2: evictor() 移除器
	定义移除某些数据的逻辑
3: allowedLateness() 允许处理迟到的数据  (传输时间)
4:sideOutputLateData() 将迟到的数据放入侧输出流
5:getSideOutput() 获取侧输出流
```

```scala
总览:
	stream
		.keyBy()
		.window()
		.trigger()
		.evictor()
		.allowedLateness()
		.sideOutputLateData()
		.reduce()/.flod()...
		.getSideOutput()
无keyby
  stream
  .windowAll()
  .trigger()
	.evictor()
	.allowedLateness()
	.sideOutputLateData()
	.[reduce()/.flod()...]
	.getSideOutput()
```

Apache Flink 具有三个不同的时间概念，即 processing time, event time 和 ingestion time。

#### 1.3 Time

##### 1.3.1 Processing Time(处理时间)

Processing Time 是指事件被处理时机器的系统时间。与机器相关.

当流程序在 Processing Time 上运行时，所有基于时间的操作(如时间窗口)将使用当时机器的系统时间。每小时 Processing Time 窗口将包括在系统时钟指示整个小时之间到达特定操作的所有事件。

例如，如果应用程序在上午 9:15 开始运行，则第一个每小时 Processing Time 窗口将包括在上午 9:15 到上午 10:00 之间处理的事件，下一个窗口将包括在上午 10:00 到 11:00 之间处理的事件。

Processing Time 是最简单的 “Time” 概念，不需要流和机器之间的协调，它提供了最好的性能和最低的延迟。但是，在分布式和异步的环境下，Processing Time 不能提供确定性，因为它容易受到事件到达系统的速度（例如从消息队列）、事件在系统内操作流动的速度以及中断的影响。

- 处理时间是指当前机器处理该条事件的时间。
- 它是当数据流入到具体某个算子时候相应的系统。
- 他提供了最小的延时和最佳的性能。
- 但是在分布式和异步环境中, 处理时间不能提供确定性。
- 因为其对时间到达 系统的速度和数据流在系统的各个operator 之间处理的速度很铭感

##### 1.3.2 Event Time(事件时间)

​	Event Time 是事件发生的时间，一般就是数据本身携带的时间。这个时间通常是在事件到达 Flink 之前就确定的，并且可以从每个事件中获取到事件时间戳。在 Event Time 中，时间取决于数据，而跟其他没什么关系。Event Time 程序必须指定如何生成 Event Time 水印，这是表示 Event Time 进度的机制。

完美的说，无论事件什么时候到达或者其怎么排序，最后处理 Event Time 将产生完全一致和确定的结果。但是，除非事件按照已知顺序（按照事件的时间）到达，否则处理 Event Time 时将会因为要等待一些无序事件而产生一些延迟。由于只能等待一段有限的时间，因此就难以保证处理 Event Time 将产生完全一致和确定的结果。

假设所有数据都已到达， Event Time 操作将按照预期运行，即使在处理无序事件、延迟事件、重新处理历史数据时也会产生正确且一致的结果。 例如，每小时事件时间窗口将包含带有落入该小时的事件时间戳的所有记录，无论它们到达的顺序如何。

请注意，有时当 Event Time 程序实时处理实时数据时，它们将使用一些 Processing Time 操作，以确保它们及时进行。

- 事件时间是每个事件在其生产设备上发生的时间。
- 此时间通常在进入Flink之前嵌入到记录中，并且可以从每个记录中提取该事件时间戳。
- 事件时间对于乱序、延时、或者数据重放等情况，都能给出正确的结果。
- 事件时间依赖于事件本身，而跟物理时钟没有关系。
- 基于事件时间的程序必须指定如何生成事件时间水印（watermark），这是指示事件时间进度的机制。
- 事件时间处理通常存在一定的延时，因此需要为延时和无序的事件等待一段时间。
- 因此，使用事件时间编程通常需要与处理时间相结合。

##### 1.3.3 Ingestion Time(摄入时间)

Ingestion Time 是事件进入 Flink 的时间。 在源操作处，每个事件将源的当前时间作为时间戳，并且基于时间的操作（如时间窗口）会利用这个时间戳。

Ingestion Time 在概念上位于 Event Time 和 Processing Time 之间。 与 Processing Time 相比，它稍微贵一些，但结果更可预测。因为 Ingestion Time 使用稳定的时间戳（在源处分配一次），所以对事件的不同窗口操作将引用相同的时间戳，而在 Processing Time 中，每个窗口操作符可以将事件分配给不同的窗口（基于机器系统时间和到达延迟）。

与 Event Time 相比，Ingestion Time 程序无法处理任何无序事件或延迟数据，但程序不必指定如何生成水印。

在 Flink 中，Ingestion Time 与 Event Time 非常相似，但 Ingestion Time 具有自动分配时间戳和自动生成水印功能。

- 摄入时间是数据进入Flink框架的时间，是在Source Operator中设置的

- 与ProcessingTime相比可以提供更可预测的结果，因为摄入时间的时间戳比较稳定(在源处只记录一次)

- 同一数据在流经不同窗口操作时将使用相同的时间戳

- 而对于ProcessingTime同一数据在流经不同窗口算子会有不同的处理时间戳

  **下面是对这三种的解释说明**

```
event producer --->   message queue  --> flink data source --> flink window operator
(enent time)                              (Ingestion Time)      (window Processing Time)
```

##### 1.3.4 设置时间特性

Flink DataStream 程序的第一部分通常是设置基本时间特性。 该设置定义了数据流源的行为方式（例如：它们是否将分配时间戳），以及像 `KeyedStream.timeWindow(Time.seconds(30))` 这样的窗口操作应该使用上面哪种时间概念。

以下示例显示了一个 Flink 程序，该程序在每小时时间窗口中聚合事件。

```scala
    val environment = StreamExecutionEnvironment.getExecutionEnvironment
    environment.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
    //其他
    //environment.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime)
    //environment.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime)
    import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer09
    val stream = environment.addSource(new FlinkKafkaConsumer09[Nothing](topic, schema, props))
    stream
      .keyBy( (event) -> event.getUser() )
      .timeWindow(Time.hours(1))
      .reduce( (a, b) -> a.add(b) )
      .addSink(...);
```

##### 1.3.5 事件时间与水印(Watermark)

​    事件时间可以处理延迟数据,但不能解决窗口已经计算完毕,但属于这个窗口的数据延迟到来时,再在这个窗口进行计算(已经计算完毕的窗口),水印就是解决这个问题

​	解释:

1.创建一个大小为10秒的SlidingWindow，每5秒滑动一次的窗口

2.假设源分别在时间13秒，第13秒和第16秒产生类型a的三个消息

这些消息会落在窗口中,在第13秒产生的前两个消息将落入窗口1 [5s-15s]和window2 [10s-20s],第16个时间生成的第三个消息将落入window2 [ 10s-20s]和window3 [15s-25s] ]。每个窗口发出的最终计数分别为（a，2），（a，3）和（a，1）。这个流程为期望的正常流程

```
		       5			10			15			20			25			30
window1		(           a,a  ) [a,2]
window2						(   a,a    a     ) [a,3]
window3                    ( a             ) [a,1]
```



若有消息延迟,假设其中一条消息（在第13秒生成）到达延迟6秒（第19秒），可能是由于某些网络拥塞。你能猜测这个消息会落入哪个窗口？

```
		       5			10			15			20			25			30
window1		(           a   ) [a,1]
window2						(   a    a    a延) [a,3]
window3                    ( a  a延        ) [a,2]
```



​    延迟的消息落入窗口2和3，因为19在10-20和15-25之间。在window2中计算没有任何问题（因为消息应该落入该窗口），但是它影响了window1和window3的结果。我们现在将尝试使用EventTime处理来解决这个问题。

**现在基于事件时间的系统(无水印)**

```
		       5			10			15			20			25			30
window1		(           a   ) [a,1]
window2						(   a    a    a延) [a,3]
window3                    ( a             ) [a,1]
```

​    结果看起来更好，窗口2和3现在发出正确的结果，但是window1仍然是错误的。Flink没有将延迟的消息分配给窗口3，因为它现在检查了消息的事件时间，并且理解它不在该窗口中。但是为什么没有将消息分配给窗口1？原因是在延迟的信息到达系统时（第19秒），窗口1的评估已经完成了（第15秒）。现在让我们尝试通过使用水印来解决这个问题

ps:请注意，在窗口2中，延迟的消息仍然位于第19秒，而不是第13秒（事件时间）。该图中的描述是故意表示窗口中的消息不会根据事件时间进行排序。（这可能会在将来改变）

**加水印的事件时间**

我们现在将水印设置为当前时间-5秒，这告诉Flink希望消息最多有5s的延迟，这是因为每个窗口仅在水印通过时被评估。由于我们的水印是当前时间-5秒，所以第一个窗口[5s-15s]将仅在第20秒被评估。类似地，窗口[10s-20s]将在第25秒进行评估，依此类推。

```
		       5			10			15			20			25			30
window1		(           a   )(    a延) [a,2]
window2						(   a    a    a延)(      ) [a,3]
window3                    ( a             )(       ) [a,1]
```

最后我们得到了正确的结果，所有这三个窗口现在都按照预期的方式发射计数，这是（a，2），（a，3）和（a，1）

###### 1.3.5.1 Watermark 产生的背景

- 流处理从事件产生，到数据流经source，再到operator，中间是有一个过程和时间的。
- 虽然大部分情况下，数据流到operator的数据都是按照事件产生的时间顺序来的，但是也不排除由于网络、背压等原因，导致乱序的产生（out-of-order或者说late element）。
- 但是对于late element（延迟数据），我们又不能无限期的等下去，必须要有个机制来保证一个特定的时间后，必须触发window去进行计算了。
- 水印告诉Flink一个消息延迟多少的方式
- 这个特别的机制，就是watermark。

###### 1.3.5.2 Watermark  介绍

- Watermark是Flink为了处理EventTime时间类型的窗口计算提出的一种机制, 本质上也是一种时间戳。

- Watermark是用于处理乱序事件的，而正确的处理乱序事件，通常用watermark机制结合window来实现。

- 当operator通过基于Event Time的时间窗口来处理数据时，它必须在确定所有属于该时间窗口的消息全部流入此操作符后，才能开始处理数据。

- 但是由于消息可能是乱序的，所以operator无法直接确认何时所有属于该时间窗口的消息全部流入此操作符。

- WaterMark包含一个时间戳，Flink使用WaterMark标记所有小于该时间戳的消息都已流入,window的执行也是由watermark触发的

- Flink的数据源在确认所有小于某个时间戳的消息都已输出到Flink流处理系统后，会生成一个包含该时间戳的WaterMark，插入到消息流中输出到Flink流处理系统中，Flink operator算子按照时间窗口缓存所有流入的消息。

- 当操作符处理到WaterMark时，它对所有小于该WaterMark时间戳的时间窗口的数据进行处理并发送到下一个操作符节点，然后也将WaterMark发送到下一个操作符节点。

- watermark用来让程序自己平衡延迟和结果的正确性

  **特点**

- 水印是一条特殊的数据记录

- 水印必须单调递增,确保任务的事件时间时钟在向前推进,而不是在后退

- 水印与数据的时间戳相关

```
水印详解: 1,2,3 单表事件时间
  8 -> 7 -> 5 ->6 -> 3 -> 2 -> 5 -> 4 -> 1
  划分两个窗口 [0,5):a [5,10):b,水印延后三秒(这个时间需要提前测试预估)
  8 -> 7 -> 5 ->6 -> 3 -> 2 -> 5 -> 4 -> 1
  第一个1到来进入窗口a,4到来进入窗口1,变为
   8 -> 7 -> 5 ->6 -> 3 -> 2 -> 5 ->(1) 4 -> 1
   (1)为水印(当前最大时间时间减去3 为1),代表1秒钟要关闭的窗口,但没有,不做处理 4进入a窗口
  5数据到来
      8 -> 7 -> 5 ->6 -> 3 -> 2 ->(2) 5 ->(1) 4 -> 1
    (2)(5-3),代表2秒的窗口要关闭,但没有,不做处理,5进入b窗口
  2到来
      8 -> 7 -> 5 ->6 -> 3 ->(2) 2 ->(2) 5 ->(1) 4 -> 1
    (2)(5-3),代表2秒的窗口要关闭,但没有,不做处理 2进入a窗口
  3到来
      8 -> 7 -> 5 ->6 ->(2) 3 ->(2) 2 ->(2) 5 ->(1) 4 -> 1
    (2)(5-3),代表2秒的窗口要关闭,但没有,不做处理 3进入a窗口  
  6到来
      8 -> 7 -> 5 ->(3) 6 ->(2) 3 ->(2) 2 ->(2) 5 ->(1) 4 -> 1
    (3)(6-3),代表3秒的窗口要关闭,但没有,不做处理 6进入b窗口  
  5到来
      8 -> 7 ->(3) 5 ->(3) 6 ->(2) 3 ->(2) 2 ->(2) 5 ->(1) 4 -> 1
    (3)(6-3),代表3秒的窗口要关闭,但没有,不做处理 5进入b窗口
  7到来
      8 ->(4) 7 ->(3) 5 ->(3) 6 ->(2) 3 ->(2) 2 ->(2) 5 ->(1) 4 -> 1
    (4)(6-3),代表4秒的窗口要关闭,但没有,不做处理 7进入b窗口
  8到来
     (5) 8 ->(4) 7 ->(3) 5 ->(3) 6 ->(2) 3 ->(2) 2 ->(2) 5 ->(1) 4 -> 1
    (5)(6-3),代表5秒的窗口要关闭,0-5窗口关闭,8进入b窗口
```

**水印的传递**(分布式传递)

```
上游向下游传递水印,直接广播出来(针对分区),下游每个节点设置分区水印,各个分区的水印可能不一致,节点以最慢的水印为主
```

```
[上游节点1 水印:4]--->
											(1,2,3)-->	[下游游节点1 水印:2 分区水印(4,3,2)]
[上游节点2 水印:3]--->
											(1,2,3)-->	[下游游节点2 水印:2 分区水印(4,3,2)]
[上游节点3 水印:2]--->
```

###### 1.3.5.3  加水印的时间窗口什么时候触发计算

```
1.watermark时间>=window_endtime
2.在[window_starttime,window_endtime]中有数据
```

###### 1.3.5.4 水印的分类

```
1.水印时间等于窗口结束时间(事件时间)
		此时和没有水印机制是一样的,窗口结束触发计算,所以matermark为0时和没有水印是一样的
2.水印时间为当前时间减去N秒,例如减去5秒
		watermark为当前时间减去5秒,例如对应现实的两块表a,b,a为慢5秒的表,b为正确的表,a就是watermark,b为现实时间轴,时间窗口的划分是根据b,触发计算的时间是参考的a表(慢5秒的表),触发计算的条件是改窗口的结束时间,例如在窗口1中,结束时间为10秒(b表的时间),而触发计算的时间是a表的10秒(对应b表的15秒),
```

```
		1秒		5		10		15		20		25		30秒
b表: ------------------------------------------------
  <--(	窗口1		)(	窗口2		)(	窗口3    ) <---
					 1秒	 5	   10    15    20    25秒
a表: ------------------------------------------------
窗口1在b表的10秒触发计算,但在a表的10秒进行评估(b表的15秒)
```

###### 1.3.5.5 水印的产生方式

- Punctuated
  - 数据流中每一个递增的EventTime都会产生一个Watermark。
  - 在实际的生产中Punctuated方式在TPS很高的场景下会产生大量的Watermark在一定程度上对下游算子造成压力，所以只有在实时性要求非常高的场景才会选择Punctuated的方式进行Watermark的生成。
- Periodic
  - 周期性的（一定时间间隔或者达到一定的记录条数）产生一个Watermark。
  - 在实际的生产中Periodic的方式必须结合时间和积累条数两个维度继续周期性产生Watermark，否则在极端情况下会有很大的延时。

```scala
def main(args: Array[String]): Unit = {
    //创建执行环境，并设置为使用EventTime
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)//注意控制并发数
    //置为使用EventTime
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
    val source = env.socketTextStream("node01", 9999)
    val dst1: DataStream[SalePrice] = source.map(value => {
      val columns = value.split(",")
      SalePrice(columns(0).toLong, columns(1), columns(2), columns(3).toDouble)
    })
    //todo 水印时间  assignTimestampsAndWatermarks
    val timestamps_data = dst1.assignTimestampsAndWatermarks(
      //周期生成水印
      new AssignerWithPeriodicWatermarks[SalePrice]{
      		var currentMaxTimestamp:Long = 0
      		val maxOutOfOrderness = 2000L //最大允许的乱序时间是2s
      		var wm : Watermark = null
      		val format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS")
      		override def getCurrentWatermark: Watermark = {
      		  wm = new Watermark(currentMaxTimestamp - maxOutOfOrderness)
      		  wm
      		}
     			 override def extractTimestamp(
       				element: SalePrice, previousElementTimestamp: Long): Long = {
        			val timestamp = element.time
        			currentMaxTimestamp = Math.max(timestamp, currentMaxTimestamp)
        			currentMaxTimestamp
      			}
    })
    val data: KeyedStream[SalePrice, String] = timestamps_data.keyBy(line => line.productName)
    val window_data: WindowedStream[SalePrice, String, TimeWindow] = data.timeWindow(Time.seconds(3))
    val apply: DataStream[SalePrice] = window_data.apply(new MyWindowFunc)
    apply.print()
    env.execute()

  }
}
case class SalePrice(time: Long, boosName: String, productName: String, price: Double)
class MyWindowFunc extends WindowFunction[SalePrice , SalePrice , String, TimeWindow]{
  override def apply(key: String, window: TimeWindow, input: Iterable[SalePrice], out: Collector[SalePrice]): Unit = {
    val seq = input.toArray
    val take: Array[SalePrice] = seq.sortBy(line => line.price).reverse.take(1)
    for(info <- take){
      out.collect(info)
    }
  }
}
```

```scala
import org.apache.flink.streaming.api.TimeCharacteristic
import org.apache.flink.streaming.api.scala.{DataStream, StreamExecutionEnvironment}
import org.apache.flink.streaming.api.windowing.time.Time
import org.apache.flink.api.scala._
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor
object WaterMarkFink {
  def main(args: Array[String]) {

    //1.创建执行环境，并设置为使用EventTime
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    //置为使用EventTime
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
    //周期生成水印 的方式 的周期长度 事件时间和Ingestion Time(摄入时间) 默认200毫秒 另外一种无 0 
    env.getConfig.setAutoWatermarkInterval(50)
    val hostname = args(0)
    val port = args(1).toInt
    //2.创建数据流，并进行数据转化val
     val source: DataStream[String] = env.socketTextStream(hostname, port)
    case class SalePrice(time: Long, boosName: String, productName: String, price: Double)
    val dst1: DataStream[SalePrice] = source.map(value => {
      val columns = value.split(",")
      SalePrice(columns(0).toLong, columns(1), columns(2), columns(3).toDouble)
    })

    //3.使用EventTime进行求最值操作
    val dst2: DataStream[SalePrice] = dst1
      //提取消息中的时间戳属性 升序
      //.assignAscendingTimestamps(_.time)
      //Time.seconds(3)最大乱序时间,extractTimestamp 定义事件时间字段 周期性的生成
      //public abstract class BoundedOutOfOrdernessTimestampExtractor<T> implements AssignerWithPeriodicWatermarks<T>
    //周期生成 Processing Time(处理时间) 无水印 其他两种默认200毫秒
        .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[SalePrice](Time.seconds(3)) {
          override def extractTimestamp(element: SalePrice): Long = element.time
        })
      .keyBy(_.productName)
      .timeWindow(Time.seconds(3)) //设置window方法一
      .max("price")
    //4.显示结果
    dst2.print()
    //5.触发流计算
    env.execute()
  }
}
```

flink 处理乱序数据的方法

```
1:窗口加水印,允许乱序处理的时间 时间短
2:窗口设置allowedLateness() 允许处理延迟数据  时间稍长 
3:最后兜底侧输出流 sideOutputLateData()
```

**窗口起始点的计算**

```java
//实际是这个方法
public static long getWindowStartWithOffset(long timestamp, long offset, long windowSize) {
		return timestamp - (timestamp - offset + windowSize) % windowSize;
	}
//timestamp(毫秒) 获取到的时间戳,offset默认为0,windowSize(毫秒)窗口大小

@Override
	public Collection<TimeWindow> assignWindows(Object element, long timestamp, WindowAssignerContext context) {
		final long now = context.getCurrentProcessingTime();
		long start = TimeWindow.getWindowStartWithOffset(now, offset, size)
      //窗口起始点
		return Collections.singletonList(new TimeWindow(start, start + size)); 
	}
```

### 2 Count Windows

​    Apache Flink 还提供计数窗口功能。如果计数窗口设置的为 100 ，那么将会在窗口中收集 100 个事件，并在添加第 100 个元素时计算窗口的值。

```scala
 def countWindowAll(size: Long, slide: Long): AllWindowedStream[T, GlobalWindow] = {
    new AllWindowedStream(stream.countWindowAll(size, slide))
  }

  /**
   * Windows this [[DataStream]] into tumbling count windows.
   *
   * Note: This operation can be inherently non-parallel since all elements have to pass through
   * the same operator instance. (Only for special cases, such as aligned time windows is
   * it possible to perform this operation in parallel).
   *
   * @param size The size of the windows in number of elements.
   */
  def countWindowAll(size: Long): AllWindowedStream[T, GlobalWindow] = {
    new AllWindowedStream(stream.countWindowAll(size))
  }
```

在 Flink 的 DataStream API 中，tumbling count window 和 sliding count window 的定义如下:

**tumbling count window**

输入一个控制参数

```scala
data.keyBy(1)
	.countWindow(100) //统计每 100 个元素的数量之和
	.sum(1);
```

**sliding count window**

输入两个控制参数

```scala
data.keyBy(1) 
	.countWindow(100, 10) //每 10 个元素统计过去 100 个元素的数量之和
	.sum(1);
```





## 三 flink的窗口机制

   Flink 的内置 time window 和 count window 已经覆盖了大多数应用场景，但是有时候也需要定制窗口逻辑，此时 Flink 的内置的 window 无法解决这些问题。为了还支持自定义 window 实现不同的逻辑，DataStream API 为其窗口机制提供了接口。到达窗口操作符的元素被传递给 WindowAssigner。WindowAssigner 将元素分配给一个或多个窗口，可能会创建新的窗口。窗口本身只是元素列表的标识符，它可能提供一些可选的元信息，例如 TimeWindow 中的开始和结束时间。注意，元素可以被添加到多个窗口，这也意味着一个元素可以同时在多个窗口存在。每个窗口都拥有一个 Trigger(触发器)，该 Trigger(触发器) 决定何时计算和清除窗口。当先前注册的计时器超时时，将为插入窗口的每个元素调用触发器。在每个事件上，触发器都可以决定触发(即、清除(删除窗口并丢弃其内容)，或者启动并清除窗口。一个窗口可以被求值多次，并且在被清除之前一直存在。注意，在清除窗口之前，窗口将一直消耗内存。当 Trigger(触发器) 触发时，可以将窗口元素列表提供给可选的 Evictor，Evictor 可以遍历窗口元素列表，并可以决定从列表的开头删除首先进入窗口的一些元素。然后其余的元素被赋给一个计算函数，如果没有定义 Evictor，触发器直接将所有窗口元素交给计算函数。计算函数接收 Evictor 过滤后的窗口元素，并计算窗口的一个或多个元素的结果。 DataStream API 接受不同类型的计算函数，包括预定义的聚合函数，如 sum（），min（），max（），以及 ReduceFunction，FoldFunction 或 WindowFunction。

这些是构成 Flink 窗口机制的组件。 接下来我们逐步演示如何使用 DataStream API 实现自定义窗口逻辑。 我们从 DataStream [IN] 类型的流开始，并使用 key 选择器函数对其分组，该函数将 key 相同类型的数据分组在一块。

## 四 自定义窗口

### 4.1Window Assigner

负责将元素分配到不同的 window。

Window API 提供了自定义的 WindowAssigner 接口，我们可以实现 WindowAssigner 的

### 4.2 Trigger

Trigger 即触发器，定义何时或什么情况下移除 window

我们可以指定触发器来覆盖 WindowAssigner 提供的默认触发器。 请注意，指定的触发器不会添加其他触发条件，但会替换当前触发器。

### 4.3 Evictor（可选）

驱逐者，即保留上一 window 留下的某些元素

### 4.4 通过 apply WindowFunction 来返回 DataStream 类型数据。

自定义窗口方法