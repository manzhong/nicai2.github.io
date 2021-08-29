---
title: Flink的ProcessFunction
tags: Flink
categories: Flink
abbrlink: 11750
date: 2020-11-15 22:42:10
summary_img:
encrypt:
enc_pwd:
---

### 一 概念

​	之前的转换算子 是无法访问事件的时间戳信息和 水位线 信息的。而这在一些应用场景下极为重要。例如 MapFunction 这样的 map 转换算子就无法访问时间戳或者当前事件的事件时间。基于此，DataStream API 提供了一系列的 Low Level 转换算子。可以访问**时间戳、 watermark 以及注册定时事件** 。还可以**输出特定的一些事件** ，例如超时事件等。Process Function 用来构建事件驱动的应用以及实现自定义的业务逻辑 使用之前的window 函数和转换算子无法实现 。

例如， Flink SQL 就是使用 Process Function 实现的。

**Flink提供了 8 个 Process Function**

```scala
 ProcessFunction
 KeyedProcessFunction
 CoProcessFunction
 ProcessJoinFunction
 BroadcastProcessFunction
 KeyedBroadcastProcessFunction
 ProcessWindowFunction
 ProcessAllWindowFunction
```

### 二 使用

**KeyedProcessFunction**
  用来操作KeyedStream ,KeyedProcessFunction会处理流的每一个元素（每条数据来了之后都可以处理、过程处理函数），输出为0个、1个或者多个元素。
所有的 Process Function 都继承自RichFunction接口（富函数，它可以有各种生命周期、状态的一些操作，获取watermark、定义闹钟定义定时器等），所以都有open()、close()和getRuntimeContext() 等方法。
而KeyedProcessFunction[KEY, IN, OUT] 还额外提供了两个方法:

```
①.processElement(I value, Context ctx, Collector<O> OUt), 流中的每一个元素都会调用这个方法，调用结果将会放在Collector数据类型中输出。 Context可以访问元素的时间戳，元素的key，以及TimerService时间服务。Context还可以将结果输出到别的流(side outputs）
②.onTimer( long timestamp, OnTimerContext ctx, Collector<O> OUT )是一个回调函数。当之前注册的定时器触发时调用（定时器触发时候的操作）。参数timestamp为定时器所设定的触发的时间戳。Collector为输出结果的集合。OnTimerContext和processElement的Context 参数一样，提供了上下文的一些信息，例如定时器触发的时间信息: 事件时间或者处理时间 。
```

```
TimerService 和 定时器 Timers
Context和OnTimerContext所持有的TimerService对象拥有以下方法:
　　long currentProcessingTime() 返回当前处理时间
　　long currentWatermark() 返回当前watermark的时间戳
　　void registerProcessingTimeTimer(long timestamp) 会注册当前key的processing time的定时器。当processing time到达定时时间时，触发timer。
　　void registerEventTimeTimer(long timestamp) 会注册当前key的event time 定时器。当水位线大于等于定时器注册的时间时，触发定时器执行回调函数。
　　void deleteProcessingTimeTimer(long timestamp) 删除之前注册处理时间定时器。如果没有这个时间戳的定时器，则不执行。
　　void deleteEventTimeTimer(long timestamp) 删除之前注册的事件时间定时器，如果没有此时间戳的定时器，则不执行。
当定时器timer触发时，会执行回调函数onTimer()。注意定时器timer只能在keyed streams上面使用。
```

### 三 实例

#### 1**要求：监控温度传感器的温度值，如果温度值在10秒钟之内(processing time)连续上升，则报警。**

```
实际可以用窗口去做,但有问题:(0:降,1:升)
 1:滚动窗口(10s)
 	[0,1,1,1,1,1,1,1,1,1][1,1,1,1,1,1,1,1,1,0]  这两个窗口中间的没法判断
 2:滑动窗口
 	除非滑动间隔特别小,否则也会有问题,两个窗口的间隔
```

**用定时器**

```
用温度上升做一个定时器,然后判断
```

```scala
import org.apache.flink.api.common.state.{ValueState, ValueStateDescriptor}
import org.apache.flink.streaming.api.functions.KeyedProcessFunction
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor
import org.apache.flink.streaming.api.scala.{DataStream, StreamExecutionEnvironment}
import org.apache.flink.streaming.api.windowing.time.Time
import org.apache.flink.util.Collector
import org.apache.flink.api.scala._
import org.apache.flink.streaming.api.TimeCharacteristic
case class SensorReading(id:String,timestamp:Long,temperature:Double)
object FlinkProcessFunctionTest {
  def main(args: Array[String]): Unit = {
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
    val stream: DataStream[String] = env.socketTextStream("localhost", 7777)

    val dataStream: DataStream[SensorReading] = stream.map(data => {
      val dataArray: Array[String] = data.split(",")
      SensorReading(dataArray(0).trim, dataArray(1).trim.toLong, dataArray(2).trim.toDouble)
    })
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[SensorReading](Time.seconds(1)) {
        override def extractTimestamp(element: SensorReading): Long = element.timestamp * 1000
      })

    val processedStream: DataStream[String] = dataStream.keyBy(_.id)
      .process(new TempIncreAlert())

    dataStream.print("Input data:")
    processedStream.print("process data:")
    env.execute("Window test")

  }
}
class TempIncreAlert() extends KeyedProcessFunction[String/* key */, SensorReading/* input */, String/* output */] {
  //温度连续上升，跟上一条数据做对比; 把上一条数据保存成当前状态
  //定义一个状态，保存上一个数据的温度值
  lazy val lastTemp: ValueState[Double] = getRuntimeContext.getState(new ValueStateDescriptor[Double]("lastTemp", classOf[Double]))
  //定义一个状态，用来保存定时器的时间戳
  lazy val currentTimer: ValueState[Long] = getRuntimeContext.getState(new ValueStateDescriptor[Long]("currentTimer", classOf[Long]))

  override def processElement(value: SensorReading, ctx: KeyedProcessFunction[String, SensorReading, String]#Context, out: Collector[String]): Unit = {
    //先取出上一个温度值
    val preTemp = lastTemp.value()
    //更新温度值
    lastTemp.update(value.temperature)

    val curTimerTs = currentTimer.value() //从当前定时器中取出来，肯定是有值的，默认值为0；

    //A. 如果温度上升且没有设过定时器，则注册定时器
    if (value.temperature > preTemp && curTimerTs == 0) {
      val timerTs = ctx.timerService().currentProcessingTime() + 10000L //当前时间 + 10s
      ctx.timerService().registerProcessingTimeTimer(timerTs) //注册定时器
      currentTimer.update(timerTs)

      //B. 如果温度下降，或是第一条数据（定时器默认为0），删除定时器并清空状态
    } else if (preTemp > value.temperature || preTemp == 0.0) { //否则就删除定时器
      ctx.timerService().deleteProcessingTimeTimer(curTimerTs)
      currentTimer.clear()  //把对应的转态清空； 不然它一直涨会撑爆内存

    }
  }
  //在回调函数中定义： 定时器要做的事情
  override def onTimer(timestamp: Long /* 定时器触发时的时间戳 */, ctx: KeyedProcessFunction[String, SensorReading, String]#OnTimerContext, out: Collector[String]): Unit = {
    //输出报警信息
    out.collect(ctx.getCurrentKey + "温度连续上升")
    currentTimer.clear() //把状态清空
  }
}
```

#### 2**侧输出流**

​	大部分的DataStream API 的大多数算子的输出是单一输出（从一条流出来还是一条流）。除了**split算子，可以将一条流分成多条流**，这些流的数据类型也都相同。 process function的**side outputs**功能可以产生多条流，**并且这些流的数据类型可以不一样。**一个 side output 可以定义为 Out putTag[X] 对象，X是输出流的数据类型。 process function 可以通过Context对象发射一个事件到一个或者多个 side outputs。

```scala
import org.apache.flink.api.common.state.{ValueState, ValueStateDescriptor}
import org.apache.flink.streaming.api.functions.{KeyedProcessFunction, ProcessFunction}
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor
import org.apache.flink.streaming.api.scala.{DataStream, OutputTag, StreamExecutionEnvironment}
import org.apache.flink.streaming.api.windowing.time.Time
import org.apache.flink.util.Collector
import org.apache.flink.api.scala._
import org.apache.flink.streaming.api.TimeCharacteristic

case class SensorReading(id:String,timestamp:Long,temperature:Double)
class FlinkSideoutputStream {
  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)
    val stream = env.socketTextStream("hadoop101", 7777)
    val dataStream = stream.map(data => {
      val dataArray = data.split(",")
      SensorReading(dataArray(0).trim, dataArray(1).trim.toLong, dataArray(2).trim.toDouble)
    }).assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[SensorReading](Time.seconds(1)) {
      override def extractTimestamp(element: SensorReading): Long = element.timestamp * 1000
    })
    val processedStream: DataStream[SensorReading] = dataStream.process(new FreezingAlert())

    processedStream.print("processed data: ") //打印的是主流

    processedStream.getSideOutput(new OutputTag[String]("freezing alert")).print("alert data") //打印侧输出流

    env.execute("Window test")
  }
}
//冰点报警，如果小于32F，输出报警信息到侧输出流
class FreezingAlert() extends ProcessFunction[SensorReading, SensorReading] { //继承ProcessFunction，没有keyBy了；后一个为主输出流

  lazy val alertOutPut: OutputTag[String] = new OutputTag[String]("freezing alert")

  override def processElement(value: SensorReading, ctx: ProcessFunction[SensorReading, SensorReading]#Context, out: Collector[SensorReading]): Unit = {
    //如果小于32F，输出报警信息到侧输出流
    if (value.temperature < 32.0) {
      ctx.output(alertOutPut, "freezing alert for" + value.id) //用什么来标记侧输出流，OutputTag作为戳
      //否则就输出到正常流中
    } else {
      out.collect(value)
    }

  }
}
```

#### 3. CoProcessFunction

对于两条输入流，DataStream API提供了CoProcessFunction这样的low-level操作。CoProcessFunction提供了操作每一个输入流的方法: processElement1()和processElement2()。

类似于ProcessFunction，这两种方法都通过Context对象来调用。这个Context对象可以访问事件数据，定时器时间戳，TimerService，以及side outputs。CoProcessFunction也提供了onTimer()回调函数。 

