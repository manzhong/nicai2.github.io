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

#### 1.4 Time

Apache Flink 具有三个不同的时间概念，即 processing time, event time 和 ingestion time。

##### 1.4.1 Processing Time(处理时间)

Processing Time 是指事件被处理时机器的系统时间。与机器相关.

当流程序在 Processing Time 上运行时，所有基于时间的操作(如时间窗口)将使用当时机器的系统时间。每小时 Processing Time 窗口将包括在系统时钟指示整个小时之间到达特定操作的所有事件。

例如，如果应用程序在上午 9:15 开始运行，则第一个每小时 Processing Time 窗口将包括在上午 9:15 到上午 10:00 之间处理的事件，下一个窗口将包括在上午 10:00 到 11:00 之间处理的事件。

Processing Time 是最简单的 “Time” 概念，不需要流和机器之间的协调，它提供了最好的性能和最低的延迟。但是，在分布式和异步的环境下，Processing Time 不能提供确定性，因为它容易受到事件到达系统的速度（例如从消息队列）、事件在系统内操作流动的速度以及中断的影响。

- 处理时间是指当前机器处理该条事件的时间。
- 它是当数据流入到具体某个算子时候相应的系统。
- 他提供了最小的延时和最佳的性能。
- 但是在分布式和异步环境中, 处理时间不能提供确定性。
- 因为其对时间到达 系统的速度和数据流在系统的各个operator 之间处理的速度很铭感

##### 1.4.2 Event Time(事件时间)

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

##### 1.4.3 Ingestion Time(摄入时间)

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

##### 1.4.4 设置时间特性

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

##### 1.4.5 事件时间与水印(Watermark)

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

###### 1.4.5.1 Watermark 产生的背景

- 流处理从事件产生，到数据流经source，再到operator，中间是有一个过程和时间的。
- 虽然大部分情况下，数据流到operator的数据都是按照事件产生的时间顺序来的，但是也不排除由于网络、背压等原因，导致乱序的产生（out-of-order或者说late element）。
- 但是对于late element（延迟数据），我们又不能无限期的等下去，必须要有个机制来保证一个特定的时间后，必须触发window去进行计算了。
- 水印告诉Flink一个消息延迟多少的方式
- 这个特别的机制，就是watermark。

###### 1.4.5.2 Watermark  介绍

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

###### 1.4.5.3  加水印的时间窗口什么时候触发计算

```
1.watermark时间>=window_endtime
2.在[window_starttime,window_endtime]中有数据
```

###### 1.4.5.4 水印的分类

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

###### 1.4.5.5 水印的产生方式

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

### 2 窗口的api

#### 1 按键分区（Keyed）和非按键分区（Non-Keyed）

​	在定义窗口操作之前，首先需要确定，到底是基于按键分区（Keyed）的数据流 KeyedStream来开窗，还是直接在没有按键分区的 DataStream 上开窗。也就是说，在调用窗口算子之前，是否有 keyBy 操作。

```
（1）按键分区窗口（Keyed Windows）
经过按键分区 keyBy 操作后，数据流会按照 key 被分为多条逻辑流（logical streams），这就是 KeyedStream。基于 KeyedStream 进行窗口操作时, 窗口计算会在多个并行子任务上同时执行。相同 key 的数据会被发送到同一个并行子任务，而窗口操作会基于每个 key 进行单独的处理。所以可以认为，每个 key 上都定义了一组窗口，各自独立地进行统计计算。
stream.keyBy(...)
.window(...)
（2）非按键分区（Non-Keyed Windows）
如果没有进行 keyBy，那么原始的 DataStream 就不会分成多条逻辑流。这时窗口逻辑只能在一个任务（task）上执行，就相当于并行度变成了 1。所以在实际应用中一般不推荐使用这种方式。
stream.windowAll(...)
这里需要注意的是，对于非按键分区的窗口操作，手动调大窗口算子的并行度也是无效的，windowAll 本身就是一个非并行的操作。******************
```

#### 2 窗口分配器（Window Assigners）

​	定义窗口分配器（Window Assigners）是构建窗口算子的第一步，它的作用就是定义数据应该被“分配”到哪个窗口,窗口分配数据的规则，其实就对
应着不同的窗口类型。所以可以说，窗口分配器其实就是在指定窗口的类型。

```scala
.window(TumblingEventTimeWindows.of(Time.hours(1),Time.minutes(30)))
```

#### 3 窗口的函数

​	定义了窗口分配器，我们只是知道了数据属于哪个窗口，可以将数据收集起来了；至于收集起来到底要做什么，其实还完全没有头绪。所以在窗口分配器之后，必须再接上一个定义窗口如何进行计算的操作，这就是所谓的“窗口函数”（window functions）。

​		经窗口分配器处理之后，数据可以分配到对应的窗口中，而数据流经过转换得到的数据类型是 WindowedStream。这个类型并不是 DataStream，所以并不能直接进行其他转换，而必须进一步调用窗口函数，对收集到的数据进行处理计算之后，才能最终再次得到 DataStream，

​		窗口函数定义了要对窗口中收集的数据做的计算操作，根据处理的方式可以分为两类：增量聚合函数和全窗口函数。下面我们来进行分别讲解。

###### **1 增量聚合函数**

​	窗口将数据收集起来，最基本的处理操作当然就是进行聚合。窗口对无限流的切分，可以看作得到了一个有界数据集。如果我们等到所有数据都收集齐，在窗口到了结束时间要输出结果的一瞬间再去进行聚合，显然就不够高效了——这相当于真的在用批处理的思路来做实时流处理。为了提高实时性，我们可以再次将流处理的思路发扬光大：就像 DataStream 的简单聚合一样，每来一条数据就立即进行计算，中间只要保持一个简单的聚合状态就可以了；区别只是在于不立即输出结果，而是要等到窗口结束时间。等到窗口到了结束时间需要输出计算结果的时候，我们只需要拿出之前聚合的状态直接输出，这无疑就大大提高了程序运行的效率和实时性。典型的增量聚合函数有两个：**ReduceFunction 和 AggregateFunction。**

**（1）归约函数（ReduceFunction）**

```
	最基本的聚合方式就是归约（reduce）。我们在基本转换的聚合算子中介绍过 reduce 的用法，窗口的归约聚合也非常类似，就是将窗口中收集到的数据两两进行归约。当我们进行流处理时，就是要保存一个状态；每来一个新的数据，就和之前的聚合状态做归约，这样就实现了增量式的聚合。
	窗口函数中也提供了 ReduceFunction：只要基于 WindowedStream 调用.reduce()方法，然后传入 ReduceFunction 作为参数，就可以指定以归约两个元素的方式去对窗口中数据进行聚合了。这里的ReduceFunction 其实与简单聚合时用到的 ReduceFunction 是同一个函数类接口，所以使用方式也是完全一样的。
```

**ReduceFunction 中需要重写一个 reduce 方法，它的两个参数代表输入的两个元素，而归约最终输出结果的数据类型，与输入的数据类型必须保持一致。也就是说，中间聚合的状态和输出的结果，都和输入的数据类型是一样的。**

```java
import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.functions.MapFunction;
import org.apache.flink.api.common.functions.ReduceFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import 
org.apache.flink.streaming.api.windowing.assigners.TumblingProcessingTimeWindo
ws;
import org.apache.flink.streaming.api.windowing.time.Time;
import java.time.Duration;
public class WindowReduceExample {
  public static void main(String[] args) throws Exception {
    StreamExecutionEnvironment env = 
    StreamExecutionEnvironment.getExecutionEnvironment();
    env.setParallelism(1);
    // 从自定义数据源读取数据，并提取时间戳、生成水位线
    SingleOutputStreamOperator<Event> stream = env.addSource(new ClickSource())
    	.assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ZERO)
    	.withTimestampAssigner(new SerializableTimestampAssigner<Event>() 
          {
            @Override
            public long extractTimestamp(Event element, long recordTimestamp) {
                return element.timestamp;
            }
   		 }));
    stream.map(new MapFunction<Event, Tuple2<String, Long>>() {
        @Override
        public Tuple2<String, Long> map(Event value) throws Exception { 
        // 将数据转换成二元组，方便计算
        return Tuple2.of(value.user, 1L);
        }
    })
    .keyBy(r -> r.f0)
    // 设置滚动事件时间窗口
    .window(TumblingEventTimeWindows.of(Time.seconds(5)))
    .reduce(new ReduceFunction<Tuple2<String, Long>>() {
        @Override
        public Tuple2<String, Long> reduce(Tuple2<String, Long> value1, 
            Tuple2<String, Long> value2) throws Exception {
            // 定义累加规则，窗口闭合时，向下游发送累加结果
            return Tuple2.of(value1.f0, value1.f1 + value2.f1);
        }
      })
      .print();
    env.execute();
  }
}
    
```

**（2）聚合函数（AggregateFunction）**

​		ReduceFunction 可以解决大多数归约聚合的问题，但是这个接口有一个限制，就是聚合状态的类型、输出结果的类型都必须和输入数据类型一样。这就迫使我们必须在聚合前，先将数据转换（map）成预期结果类型；而在有些情况下，还需要对状态进行进一步处理才能得到输出结果，这时它们的类型可能不同，使用 ReduceFunction 就会非常麻烦。

​		例如，如果我们希望计算一组数据的平均值，应该怎样做聚合呢？很明显，这时我们需要计算两个状态量：数据的总和（sum），以及数据的个数（count），而最终输出结果是两者的商（sum/count）。如果用 ReduceFunction，那么我们应该先把数据转换成二元组(sum, count)的形式，然后进行归约聚合，最后再将元组的两个元素相除转换得到最后的平均值。本来应该只是一个任务，可我们却需要 map-reduce-map 三步操作，这显然不够高效。

​		于是自然可以想到，如果取消类型一致的限制，让输入数据、中间状态、输出结果三者类型都可以不同，不就可以一步直接搞定了吗？
​		Flink 的 Window API 中的 aggregate 就提供了这样的操作。直接基于 WindowedStream 调用.aggregate()方法，就可以定义更加灵活的窗口聚合操作。这个方法需要传入一个AggregateFunction 的实现类作为参数。源码定义:

```java
public interface AggregateFunction<IN, ACC, OUT> extends Function, Serializable{
  ACC createAccumulator();
  ACC add(IN value, ACC accumulator);
  OUT getResult(ACC accumulator);
  ACC merge(ACC a, ACC b);
}
```

​		AggregateFunction 可以看作是 ReduceFunction 的通用版本，这里有三种类型：输入类型（IN）、累加器类型（ACC）和输出类型（OUT）。输入类型 IN 就是输入流中元素的数据类型；累加器类型 ACC 则是我们进行聚合的中间状态类型；而输出类型当然就是最终计算结果的类型了。

接口中有四个方法：

```
⚫ createAccumulator()：创建一个累加器，这就是为聚合创建了一个初始状态，每个聚合任务只会调用一次。
⚫ add()：将输入的元素添加到累加器中。这就是基于聚合状态，对新来的数据进行进一步聚合的过程。方法传入两个参数：当前新到的数据 value，和当前的累加器
accumulator；返回一个新的累加器值，也就是对聚合状态进行更新。每条数据到来之后都会调用这个方法。
⚫ getResult()：从累加器中提取聚合的输出结果。也就是说，我们可以定义多个状态，然后再基于这些聚合的状态计算出一个结果进行输出。比如之前我们提到的计算平均
值，就可以把 sum 和 count 作为状态放入累加器，而在调用这个方法时相除得到最终结果。这个方法只在窗口要输出结果时调用。
⚫ merge()：合并两个累加器，并将合并后的状态作为一个累加器返回。这个方法只在需要合并窗口的场景下才会被调用；最常见的合并窗口（Merging Window）的场景
就是会话窗口（Session Windows）。
```

​		所以可以看到，AggregateFunction 的工作原理是：首先调用 createAccumulator()为任务初始化一个状态(累加器)；而后每来一个数据就调用一次 add()方法，对数据进行聚合，得到的结果保存在状态中；等到了窗口需要输出时，再调用 getResult()方法得到计算结果。很明显，与 ReduceFunction 相同，AggregateFunction 也是增量式的聚合；而由于输入、中间状态、输出的类型可以不同，使得应用更加灵活方便。

​		下面来看一个具体例子。我们知道，在电商网站中，PV（页面浏览量）和 UV（独立访客数）是非常重要的两个流量指标。一般来说，PV 统计的是所有的点击量；而对用户 id 进行去重之后，得到的就是 UV。所以有时我们会用 PV/UV 这个比值，来表示“人均重复访问量”，也就是平均每个用户会访问多少次页面，这在一定程度上代表了用户的粘度。
​		代码实现如下：

```java
import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.functions.AggregateFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
import java.util.HashSet;
public class WindowAggregateFunctionExample {
		public static void main(String[] args) throws Exception {
					StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
					env.setParallelism(1);
      		SingleOutputStreamOperator<Event> stream = env.addSource(new ClickSource())
          	.assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forMonotonousTimestamps()
            .withTimestampAssigner(new SerializableTimestampAssigner<Event>(){
                @Override
                public long extractTimestamp(Event element, long recordTimestamp) {
                		return element.timestamp;
                }
						}));
           // 所有数据设置相同的 key，发送到同一个分区统计 PV 和 UV，再相除
      		stream.keyBy(data -> true)
								.window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(2)))
								.aggregate(new AvgPv())
								.print();
					env.execute();
   }
   public static class AvgPv implements AggregateFunction<Event,Tuple2<HashSet<String>, Long>, Double> {
     		@Override
				public Tuple2<HashSet<String>, Long> createAccumulator() {
          			// 创建累加器
								return Tuple2.of(new HashSet<String>(), 0L);
				}
        @Override
        public Tuple2<HashSet<String>, Long> add(Event value,Tuple2<HashSet<String>, Long> accumulator) {
              // 属于本窗口的数据来一条累加一次，并返回累加器
              accumulator.f0.add(value.user);
              return Tuple2.of(accumulator.f0, accumulator.f1 + 1L);
        }
        @Override
        public Double getResult(Tuple2<HashSet<String>, Long> accumulator) {
              // 窗口闭合时，增量聚合结束，将计算结果发送到下游
              return (double) accumulator.f1 / accumulator.f0.size();
        }
        @Override
        public Tuple2<HashSet<String>, Long> merge(Tuple2<HashSet<String>, Long> a, Tuple2<HashSet<String>, Long> b) {
        				return null;
        }
		}
}
```

​		代码中我们创建了事件时间滑动窗口，统计 10 秒钟的“人均 PV”，每 2 秒统计一次。由于聚合的状态还需要做处理计算，因此窗口聚合时使用了更加灵活的 AggregateFunction。为了统计 UV，我们用一个 HashSet 保存所有出现过的用户 id，实现自动去重；而 PV 的统计则类似一个计数器，每来一个数据加一就可以了。所以这里的状态，定义为包含一个 HashSet 和一个 count 值的二元组（Tuple2<HashSet<String>, Long>），每来一条数据，就将 user 存入 HashSet，
同时 count 加 1。这里的 count 就是 PV，而 HashSet 中元素的个数（size）就是 UV；所以最终窗口的输出结果，就是它们的比值。

​		这里没有涉及会话窗口，所以 merge()方法可以不做任何操作。

​		另外，Flink 也为窗口的聚合提供了一系列预定义的简单聚合方法，可以直接基于WindowedStream 调用。主要包括.sum()/max()/maxBy()/min()/minBy()，与 KeyedStream 的简单聚合非常相似。它们的底层，其实都是通过 AggregateFunction 来实现的。

​		通过 ReduceFunction 和 AggregateFunction 我们可以发现，**增量聚合函数其实就是在用流处理的思路来处理有界数据集，核心是保持一个聚合状态，当数据到来时不停地更新状态。这就是 Flink 所谓的“有状态的流处理”**，通过这种方式可以极大地提高程序运行的效率，所以在实际应用中最为常见。

###### 2 全窗口函数

​		窗口操作中的另一大类就是全窗口函数。与增量聚合函数不同，全窗口函数需要先收集窗口中的数据，并在内部缓存起来，等到窗口要输出结果的时候再取出数据进行计算。很明显，这就是典型的批处理思路了——先攒数据，等一批都到齐了再正式启动处理流程。这样做毫无疑问是低效的：因为窗口全部的计算任务都积压在了要输出结果的那一瞬间，而在之前收集数据的漫长过程中却无所事事

​		那为什么还需要有全窗口函数呢？这是因为有些场景下，我们要做的计算必须基于全部的数据才有效，这时做增量聚合就没什么意义了；另外，输出的结果有可能要包含上下文中的一些信息（比如窗口的起始时间），这是增量聚合函数做不到的。所以，我们还需要有更丰富的窗口计算方式，这就可以用全窗口函数来实现。

​		在 Flink 中，全窗口函数也有两种：**WindowFunction 和 ProcessWindowFunction。**

**（1）窗口函数（WindowFunction）**

​		WindowFunction 字面上就是“窗口函数”，它其实是老版本的通用窗口函数接口。我们可以基于 WindowedStream 调用.apply()方法，传入一个 WindowFunction 的实现类。

```java
stream
	.keyBy(<key selector>)
	.window(<window assigner>)
	.apply(new MyWindowFunction());
```

这个类中可以获取到包含窗口所有数据的可迭代集合（Iterable），还可以拿到窗口（Window）本身的信息。WindowFunction 接口在源码中实现如下：

```java
public interface WindowFunction<IN, OUT, KEY, W extends Window> extends Function,Serializable {
			void apply(KEY key, W window, Iterable<IN> input, Collector<OUT> out) throws Exception;
}
```

​		当窗口到达结束时间需要触发计算时，就会调用这里的 apply 方法。我们可以从 input 集合中取出窗口收集的数据，结合 key 和 window 信息，通过收集器（Collector）输出结果。这里 Collector 的用法，与 FlatMapFunction 中相同。WindowFunction 能提供的上下文信息较少，也没有更高级的功能。事实上，它的作用可以被 ProcessWindowFunction 全覆盖，所以之后可能会逐渐弃用。**我们公司实际应用，直接使用 ProcessWindowFunction。**

```scala
import org.apache.flink.api.java.tuple.Tuple
import org.apache.flink.streaming.api.TimeCharacteristic
import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.api.scala.function.WindowFunction
import org.apache.flink.streaming.api.windowing.time.Time
import org.apache.flink.streaming.api.windowing.windows.TimeWindow
import org.apache.flink.util.Collector

import scala.collection.immutable.HashSet


case class UserBehavior(userId: Long, itemId: Long, categoryId: Int, behavior: String, timestamp: Long)
// 使用全窗口函数 WindowFunction 计算窗口内的独立访客数
object WindowFunctionTest {
  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
    env.setParallelism(1)
    //UserBehavior  incre_test
    val stream = env.readTextFile("/Users/manzhong/Desktop/project/me_project/java/FlinkUserBehaviorAnalysis/HotItemsAnalysis/src/main/resources/incre_test.csv")
      .map(line => {
        val lineSplit = line.split(",")
        UserBehavior(lineSplit(0).toLong, lineSplit(1).toLong, lineSplit(2).toInt, lineSplit(3), lineSplit(4).toLong)
      })
      .assignAscendingTimestamps(_.timestamp*1000)
    stream.filter(_.behavior=="pv")
      .keyBy("itemId")
      .timeWindow(Time.minutes(60),Time.minutes((10)))
      //.sum()
      .apply(new countUv())
      .print()
    env.execute()
  }

}

//@tparam IN The type of the input value. UserBehavior
//@tparam OUT The type of the output value. (String,Long,Long)
//@tparam KEY The type of the key.  Tuple 分组的key
//TimeWindow 窗口

 class countUv() extends WindowFunction[UserBehavior,(String,Long,Long),Tuple,TimeWindow]{
   override def apply(key: Tuple, window: TimeWindow, input: Iterable[UserBehavior], out: Collector[(String,Long, Long)]): Unit = {
     var hs: HashSet[Long] = new HashSet[Long]
     val itemid: Long = key.getField(0)
     hs=hs+input.iterator.next().userId
     out.collect(window.getEnd.toString,itemid,hs.size)
     Thread.sleep(10000)
   }

 }
```



**（2）处理窗口函数（ProcessWindowFunction）**

​		ProcessWindowFunction 是 Window API 中最底层的通用窗口函数接口。之所以说它“最底层”，是因为除了可以拿到窗口中的所有数据之外，ProcessWindowFunction 还可以获取到一个“上下文对象”（Context）。这个上下文对象非常强大，不仅能够获取窗口信息，还可以访问当前的时间和状态信息。这里的时间就包括了处理时间（processing time）和事件时间水位线（event time watermark）。这就使得 ProcessWindowFunction 更加灵活、功能更加丰富。事实上，ProcessWindowFunction 是 Flink 底层 API——处理函数（process function）中的一员.

​		当 然 ， 这 些 好 处 是 以 牺 牲 性 能 和 资 源 为 代 价 的 。 作 为 一 个 全 窗 口 函 数 ，ProcessWindowFunction 同样需要将所有数据缓存下来、等到窗口触发计算时才使用。它其实就是一个增强版的 WindowFunction。具体使用跟 WindowFunction 非常类似，我们可以基于 WindowedStream 调用.process()方
法，传入一个 ProcessWindowFunction 的实现类.

```java
import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.windowing.ProcessWindowFunction;
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.util.Collector;
import java.sql.Timestamp;
import java.util.HashSet;
public class UvCountByWindowExample {
  public static void main(String[] args) throws Exception {
      StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
      env.setParallelism(1);
      SingleOutputStreamOperator<Event> stream = env.addSource(new ClickSource())
      .assignTimestampsAndWatermarks(WatermarkStrategy<Event>forBoundedOutOfOrderness(Duration.ZERO)
      .withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
        @Override
          public long extractTimestamp(Event element, long 
          recordTimestamp) {
          return element.timestamp;
          }
			}));
    	// 将数据全部发往同一分区，按窗口统计 UV
    	stream.keyBy(data -> true)
    		.window(TumblingEventTimeWindows.of(Time.seconds(10)))
    		.process(new UvCountByWindow())
    		.print();
    	env.execute();
   }
// 自定义窗口处理函数	
  public static class UvCountByWindow extends ProcessWindowFunction<Event,String, Boolean, TimeWindow>{
    @Override
    public void process(Boolean aBoolean, Context context, Iterable<Event> elements, Collector<String> out) throws Exception {
    			HashSet<String> userSet = new HashSet<>();
          // 遍历所有数据，放到 Set 里去重
          for (Event event: elements){
          userSet.add(event.user);
    			}
    			// 结合窗口信息，包装输出内容
    			Long start = context.window().getStart();
   				Long end = context.window().getEnd();
   			  out.collect(" 窗 口 : " + new Timestamp(start) + " ~ " + new Timestamp(end) + " 的独立访客数量是：" + userSet.size());
    }
  }
}
```

​			这里我使用的是事件时间语义。定义 10 秒钟的滚动事件窗口后，直接使用ProcessWindowFunction 来定义处理的逻辑。我们可以创建一个 HashSet，将窗口所有数据的userId 写入实现去重，最终得到 HashSet 的元素个数就是 UV 值。当 然 ， 这 里 我 们 并 没 有 用 到 上 下 文 中 其 他 信 息 ， 所 以 其 实 没 有 必 要 使 用ProcessWindowFunction。全窗口函数因为运行效率较低，很少直接单独使用，往往会和增量聚合函数结合在一起，共同实现窗口的处理计算。

###### 3 增量聚合和全窗口函数的结合使用

​		增量聚合函数处理高效,输出更加实时,而全窗口函数提供了更多的信息,是更加通用的窗口函数,负责收集数据、提供上下文相关信息，把所有的原材料都准备好，至于拿来做什么我们完全可以任意发挥。这就使得窗口计算更加灵活，功能更加强大。

​		所以在实际应用中，我们往往希望兼具这两者的优点，把它们结合在一起使用.

​		Flink 的Window API 就给我们实现了这样的用法。我们之前在调用 WindowedStream 的.reduce()和.aggregate()方法时，只是简单地直接传入了一ReduceFunction 或 AggregateFunction 进行增量聚合。除此之外，其实还可以传入第二个参数：一个全窗口函数，可以是 WindowFunction 或者 ProcessWindowFunction。

```java
// ReduceFunction 与 WindowFunction 结合
public <R> SingleOutputStreamOperator<R> reduce(ReduceFunction<T> reduceFunction, WindowFunction<T, R, K, W> function) 
// ReduceFunction 与 ProcessWindowFunction 结合
public <R> SingleOutputStreamOperator<R> reduce(ReduceFunction<T> reduceFunction, ProcessWindowFunction<T, R, K, W> function)
// AggregateFunction 与 WindowFunction 结合
public <ACC, V, R> SingleOutputStreamOperator<R> aggregate(AggregateFunction<T, ACC, V> aggFunction, WindowFunction<V, R, K, W>
windowFunction)
// AggregateFunction 与 ProcessWindowFunction 结合
public <ACC, V, R> SingleOutputStreamOperator<R> aggregate(AggregateFunction<T, ACC, V> aggFunction,ProcessWindowFunction<V, R, K, W> windowFunction)
```

​		这样调用的处理机制是：基于第一个参数（增量聚合函数）来处理窗口数据，每来一个数据就做一次聚合；等到窗口需要触发计算时，则调用第二个参数（全窗口函数）的处理逻辑输出结果。需要注意的是，这里的全窗口函数就不再缓存所有数据了，而是直接将增量聚合函数的结果拿来当作了 Iterable 类型的输入。一般情况下，这时的可迭代集合中就只有一个元素了。

​		url的热门度,统计 10 秒钟的 url 浏览量，每 5 秒钟更新一次

```java
import org.apache.flink.api.common.eventtime.SerializableTimestampAssigner;
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.functions.AggregateFunction;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.windowing.ProcessWindowFunction;
import org.apache.flink.streaming.api.windowing.assigners.SlidingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.util.Collector;

public class UrlViewCountExample {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        SingleOutputStreamOperator<Event> stream = env.addSource(new ClickSource())
        .assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forMonotonousTimestamps()
        .withTimestampAssigner(new SerializableTimestampAssigner<Event>() {
            @Override
            public long extractTimestamp(Event element, long recordTimestamp) {
            		return element.timestamp;
						}
				}));    
      // 需要按照 url 分组，开滑动窗口统计
      stream.keyBy(data -> data.url)
      .window(SlidingEventTimeWindows.of(Time.seconds(10), Time.seconds(5)))
      // 同时传入增量聚合函数和全窗口函数
      .aggregate(new UrlViewCountAgg(), new UrlViewCountResult())
      .print();
      env.execute();
   }
// 自定义增量聚合函数，来一条数据就加一
	public static class UrlViewCountAgg implements AggregateFunction<Event, Long, Long> {
			@Override
			public Long createAccumulator() {
				return 0L;
			}
			@Override
			public Long add(Event value, Long accumulator) {
				return accumulator + 1;
			}
			@Override
			public Long getResult(Long accumulator) {
				return accumulator;
			}
			@Override
			public Long merge(Long a, Long b) {
				return null;
			}
	}
// 自定义窗口处理函数，只需要包装窗口信息
	public static class UrlViewCountResult extends ProcessWindowFunction<Long,UrlViewCount, String, TimeWindow> {
		@Override
		public void process(String url, Context context, Iterable<Long> elements, Collector<UrlViewCount> out) throws Exception {
			// 结合窗口信息，包装输出内容
			Long start = context.window().getStart();
			Long end = context.window().getEnd();
			// 迭代器中只有一个元素，就是增量聚合函数的计算结果
			out.collect(new UrlViewCount(url, elements.iterator().next(), start, end));
		}
	}
}
```

这里我们为了方便处理，单独定义了一个 POJO 类 UrlViewCount 来表示聚合输出结果的数据类型，包含了 url、浏览量以及窗口的起始结束时间。

```java
import java.sql.Timestamp;
public class UrlViewCount {
	public String url;
	public Long count;
	public Long windowStart;
	public Long windowEnd;
	public UrlViewCount() {}
	public UrlViewCount(String url, Long count, Long windowStart, Long windowEnd) {
		this.url = url;
		this.count = count;
		this.windowStart = windowStart;
		this.windowEnd = windowEnd;
	}
	@Override
	public String toString() {
		return "UrlViewCount{" +
		"url='" + url + '\'' +
		", count=" + count +
		", windowStart=" + new Timestamp(windowStart) +
		", windowEnd=" + new Timestamp(windowEnd) +
		'}';
	}
}
```

​		代码中用一个 AggregateFunction 来实现增量聚合，每来一个数据就计数加一；得到的结果交给 ProcessWindowFunction，结合窗口信息包装成我们想要的 UrlViewCount，最终输出统计结果。
​		窗口处理的主体还是增量聚合，而引入全窗口函数又可以获取到更多的信息包装输出，这样的结合兼具了两种窗口函数的优势，在保证处理性能和实时性的同时支持了更加丰富的应用场景。

###### 4 允许延迟数据

​	在事件时间语义下，窗口中可能会出现数据迟到的情况。这是因为在乱序流中，水位线（watermark）并不一定能保证时间戳更早的所有数据不会再来。当水位线已经到达窗口结束时间时，窗口会触发计算并输出结果，这时一般也就要销毁窗口了；如果窗口关闭之后，又有本属于窗口内的数据姗姗来迟，默认情况下就会被丢弃。这也很好理解：窗口触发计算就像发车，如果要赶的车已经开走了，又不能坐其他的车（保证分配窗口的正确性），那就只好放弃坐班车了。

​		Flink 提供了一个特殊的接口，可以为窗口算子设置一个“允许的最大延迟”（Allowed Lateness）。也就是说，我们可以设定允许延迟一段时间，在这段
时间内，窗口不会销毁，继续到来的数据依然可以进入窗口中并触发计算。直到水位线推进到了 窗口结束时间 + 延迟时间，才真正将窗口的内容清空，正式关闭窗口。

```java
stream.keyBy(...)
	.window(TumblingEventTimeWindows.of(Time.hours(1)))
	.allowedLateness(Time.minutes(1))
```

**allowedLateness与水印的区别**重要

```
1:只添加水印时allowedLateness默认为0,即end_of_window的时间=水印的时间是,触发计算,并销毁窗口(watermaker大于等于end-of-window(窗口结束时间))
2:添加水印2s并allowedLateness为5s 则end_of_window的时间=水印的时间时,触发计算,但窗口不销毁,等到watermark>=allowedLateness+end_of_window的时间,再次触发计算,并销毁窗口

默认情况下，当watermark大于等于某窗口的end-of-window（结束时间）时便会触发window计算，激活window计算结束之后，再有之前的数据到达时，这些数据会被删除。为了避免在设置了WaterMaker后，仍有些迟到的数据被删除，因此产生了allowedLateness，通过使用allowedLateness来延迟销毁窗口，允许有一段时间（也是以event time来衡量）来等待之前的数据到达，以便再次处理这些数据!!!!如果设置了allowedLateness，当延迟数据来到时会再次触发之前窗口（延迟数据事件时间归属的窗口）的计算，而之前触发的数据，会buffer起来，直到watermark超过end-of-window + allowedLateness的时间，窗口的数据及元数据信息才会被删除。

区别:
	1:watermark 通过时间戳来控制窗口触发时机，主要是为了解决数据乱序到达的问题
  2:allowedLateness 用来控制窗口的销毁时间，解决窗口计算销毁后，延迟数据到来，被丢弃的问题
  3:我们需要将二者结合使用，才可进一步保证延迟数据的不丢失，以及进一步保证窗口计算的准确性（来的数据更全了）


```

###### 5 将迟到的数据放入侧输出流

我们可以将未收入窗口的迟到数据，放入“侧输出流”（side output）进行另外的处理

```java
stream.keyBy(...)
.window(TumblingEventTimeWindows.of(Time.hours(1)))
.sideOutputLateData(outputTag)
```

将迟到数据放入侧输出流之后，还应该可以将它提取出来。基于窗口处理完成之后的DataStream，调用.getSideOutput()方法，传入对应的输出标签，就可以获取到迟到数据所在的流了。

```java
SingleOutputStreamOperator<AggResult> winAggStream = stream.keyBy(...)
.window(TumblingEventTimeWindows.of(Time.hours(1)))
.sideOutputLateData(outputTag)
.aggregate(new MyAggregateFunction())
DataStream<Event> lateStream = winAggStream.getSideOutput(outputTag);
```

这里注意，getSideOutput()是 SingleOutputStreamOperator 的方法，获取到的侧输出流数据类型应该和 OutputTag 指定的类型一致，与窗口聚合之后流中的数据类型可以不同。

###### 6 其他api

```tex
//其他api
1: evictor() 移除器
	定义移除某些数据的逻辑
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

​	Window API 首先按照时候按键分区分成两类。keyBy 之后的 KeyedStream，可以调用.window()方法声明按键分区窗口（Keyed Windows）；而如果不做 keyBy，DataStream 也可以直接调用.windowAll()声明非按键分区窗口。之后的方法调用就完全一样了。接下来首先是通过.window()/.windowAll()方法定义窗口分配器，得到 WindowedStream；然 后 通 过 各 种 转 换 方 法 （ reduce/aggregate/apply/process ） 给 出 窗 口 函 数(ReduceFunction/AggregateFunction/ProcessWindowFunction)，定义窗口的具体计算处理逻辑，转换之后重新得到 DataStream。这两者必不可少，是窗口算子（WindowOperator）最重要的组成部分。

### 4 迟到数据的处理

```
1:窗口加水印,允许乱序处理的时间 时间短
	既然水位线这么重要，那一般情况就不应该把它的延迟设置得太大，否则流处理的实时性就会大大降低。因为水位线的延迟主要是用来对付分布式网络传输导致的数据乱序，而网络传输的乱序程度一般并不会很大，大多集中在几毫秒至几百毫秒。所以实际应用中，我们往往会给水位线设置一个“能够处理大多数乱序数据的小延迟”，视需求一般设在毫秒~秒级,当我们设置了水位线延迟时间后，所有定时器就都会按照延迟后的水位线来触发。如果一个数据所包含的时间戳，小于当前的水位线，那么它就是所谓的“迟到数据”。
2:窗口设置allowedLateness() 允许处理延迟数据  时间稍长 
		水位线延迟设置的比较小，那之后如果仍有数据迟到该怎么办？对于窗口计算而言，如果水位线已经到了窗口结束时间，默认窗口就会关闭，那么之后再来的数据就要被丢弃了。自然想到，Flink 的窗口也是可以设置延迟时间，允许继续处理迟到数据的。这种情况下，由于大部分乱序数据已经被水位线的延迟等到了，所以往往迟到的数据不会
太多。这样，我们会在水位线到达窗口结束时间时，先快速地输出一个近似正确的计算结果；然后保持窗口继续等到延迟数据，每来一条数据，窗口就会再次计算，并将更新后的结果输出。这样就可以逐步修正计算结果，最终得到准确的统计值了。最后的效果就是先快速实时地输出一个近似的结果，而后再不断调整，最终得到正确的计算结果。
3:最后兜底侧输出流 sideOutputLateData()
		即使我们有了前面的双重保证，可窗口不能一直等下去，最后总要真正关闭。窗口一旦关闭，后续的数据就都要被丢弃了。那如果真的还有漏网之鱼又该怎么办呢？
那就要用到最后一招了：用窗口的侧输出流来收集关窗以后的迟到数据。这种方式是最后“兜底”的方法，只能保证数据不丢失；因为窗口已经真正关闭，所以是无法基于之前窗口的结果直接做更新的。我们只能将之前的窗口计算结果保存下来，然后获取侧输出流中的迟到数据，判断数据所属的窗口，手动对结果进行合并更新。尽管有些烦琐，实时性也不够强，但能够保证最终结果一定是正确的。
```

### 5 窗口的生命周期

1. 窗口的创建
   窗口的类型和基本信息由窗口分配器（window assigners）指定，但窗口不会预先创建好，而是由数据驱动创建。当第一个应该属于这个窗口的数据元素到达时，就会创建对应的窗口。
2. 窗口计算的触发
   除了窗口分配器，每个窗口还会有自己的窗口函数（window functions）和触发器（trigger）。窗口函数可以分为增量聚合函数和全窗口函数，主要定义了窗口中计算的逻辑；而触发器则是指定调用窗口函数的条件。对于不同的窗口类型，触发计算的条件也会不同。例如，一个滚动事件时间窗口，应该在水位线到达窗口结束时间的时候触发计算，属于“定点发车”；而一个计数窗口，会在窗口中元素数量达到定义大小时触发计算，属于“人满就发车”。所以 Flink 预定义的窗口类型都有对应内置的触发器。对于事件时间窗口而言，除去到达结束时间的“定点发车”，还有另一种情形。当我们设置了允许延迟，那么如果水位线超过了窗口结束时间、但还没有到达设定的最大延迟时间，这期间内到达的迟到数据也会触发窗口计算。这类似于没有准时赶上班车的人又追上了车，这时
   车要再次停靠、开门，将新的数据整合统计进来。
3. 窗口的销毁
   一般情况下，当时间达到了结束点，就会直接触发计算输出结果、进而清除状态销毁窗口。这时窗口的销毁可以认为和触发计算是同一时刻。这里需要注意，Flink 中只对时间窗口TimeWindow）有销毁机制；由于计数窗口（CountWindow）是基于全局窗口（GlobalWindw）实现的，而全局窗口不会清除状态，所以就不会被销毁。在特殊的场景下，窗口的销毁和触发计算会有所不同。事件时间语义下，如果设置了允许延迟，那么在水位线到达窗口结束时间时，仍然不会销毁窗口；窗口真正被完全删除的时间点，是窗口的结束时间加上用户指定的允许延迟时间。

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

