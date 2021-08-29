---
title: Flink的状态管理和后端
tags: Flink
categories: Flink
abbrlink: 6371
date: 2020-11-09 22:41:38
summary_img:
encrypt:
enc_pwd:
---

## Flink的状态

### 1 简介

**流式计算分为无状态和有状态两种情况**

- 无状态的计算观察每个独立事件，并根据最后一个事件输出结果。
- 有状态的计算则会基于多个事件输出结果。

```
无状态流处理分别接收每条数据记录，然后根据最新输入的数据生成输出数据
[] ---> [操作] --->[]
有状态流处理维护所有已处理记录的状态值，并根据每条新输入的记录更新状态，因此输出记录(灰条)反映的是综合考虑多个事件之后的结果
[] ---->[ 操作 ] ----> []
          ^ |
          | |
          | |
          | v
         [状态]        
```

1. 有一个任务维护,并且用来计算某个结果的所有数据,都属于这个任务的状态
2. 可以认为状态就是一个本地变量,可以被任务的业务逻辑访问
3. flink会进行状态管理,包括状态一致性,故障处理以及高效存储和访问,以便开发人员可以专注于应用程序的逻辑

### 2 : 分类

在flink中,状态始终与特定算子相关联,为了使运行的flink了解算子的状态,**算子需要预先注册其状态**

总的来说,有两种类型的状态:

```
算子状态(operator state)
	算子状态的作用范围为算子任务
键控状态(keyed state) 分组
	根据输入数据流中定义的键(key)来维护和访问,基于KeyBy--KeyedStream上有任务出现的状态，定义的不同的key来维护这个状态；不同的key也是独立访问的，一个key只能访问它自己的状态，不同key之间也不能互相访问
```

### 3 算子状态

```
1:算子状态的作用范围限定为算子任务,由同一并行任务所处理的所有数据都可以访问到相同的状态
2:状态对于同一子任务而言是共享的
3:算子的状态不能由相同或不同算子的另一个子任务访问
```

**算子状态的数据结构**

```
① 列表状态（List state），将状态表示为一组数据的列表；（会根据并行度的调整把之前的状态重新分组重新分配）
② 联合列表状态（Union list state），也将状态表示为数据的列表，它常规列表状态的区别在于，在发生故障时，或者从保存点（savepoint）启动应用程序时如何恢复（把之前的每一个状态广播到对应的每个算子中）。
③ 广播状态（Broadcast state），如果一个算子有多项任务，而它的每项任务状态又都相同，那么这种特殊情况最适合应用广播状态（把同一个状态广播给所有算子子任务）；
```

### 4 键控状态

```
1:键控状态是根据输入数据流中定义的键来维护和访问的
2:flink为每个key维护一个状态实例,并将具有相同键的所有数据,都分区到同一个算子任务中,这个任务会维护和处理这个key对应的状态
3:当任务处理一条数据时,他会自动将状态的访问范围限定为当前数据的key
```

**键控状态的数据结构**

```
① 值状态（ValueState<T>），将状态表示为单个值；（直接.value获取，Set操作是.update）
get操作: ValueState.value()
set操作: ValueState.update(T value)
② 列表状态（ListState<T>），将状态表示为一组数据的列表（存多个状态）；（.get，.update，.add）
ListState.add(T value)
ListState.addAll(List<T> values)
ListState.get()返回Iterable<T>
ListState.update(List<T> values)
③ 映射状态（MapState<K, V>），将状态表示为一组Key-Value对；（.get，.put ，类似HashMap）
MapState.get(UK key)
MapState.put(UK key, UV value)
MapState.contains(UK key)
MapState.remove(UK key)
④ 聚合状态（ReducingState<T>  & AggregatingState<I, O>），将状态表示为一个用于聚合操作的列表；（.add不像之前添加到列表，它是直接聚合到之前的结果中）
　　　　Reduce输入输出类型是不能变的，Aggregate可得到数据类型完全不一样的结果；
State.clear()是清空操作。
```

### 5 键控状态的使用

因为声明一个键控状态需要获取上下文对象,所以必须在富函数中声明

**声明一个键控状态**

```scala
class  MyReduce extends ReduceFunction[SalePrice]{
  override def reduce(value1: SalePrice, value2: SalePrice): SalePrice = ???
}
case class SalePrice(time: Long, boosName: String, productName: String, price: Double)
class MyRichMapper extends  RichMapFunction[SalePrice,String] {
  //声明值的状态
  var valueState:ValueState[Double]= _
  //声明list的状态
  lazy val listState:ListState[Int] = getRuntimeContext.getListState(new ListStateDescriptor[Int]("listState",classOf[Int]))
  //map
  lazy val mapState:MapState[String,Int] = getRuntimeContext.getMapState(new MapStateDescriptor[String,Int]("mapState",classOf[String],classOf[Int]))
  //reduce
  lazy val reduceState:ReducingState[SalePrice] = getRuntimeContext.getReducingState(new ReducingStateDescriptor[SalePrice]("reduceState",new MyReduce(),classOf[SalePrice]))

  override def open(parameters: Configuration): Unit = {
    //声明一个键控状态new ValueStateDescriptor[Double]("price",classOf[Double]) price 键控的名字,
  valueState = getRuntimeContext.getState(new ValueStateDescriptor[Double]("price",classOf[Double]))
  }
  override def map(value: SalePrice): String ={
    //状态的读写
    val myVal=valueState.value()
    //更新
    valueState.update(value.price)
    //list的状态
    //操作
    listState.add(1)
    listState.update(new util.ArrayList[Int](3))
    listState.get()
    value.boosName
  }
}
```

**读取状态**

```scala
class  MyReduce extends ReduceFunction[SalePrice]{
  override def reduce(value1: SalePrice, value2: SalePrice): SalePrice = ???
}
case class SalePrice(time: Long, boosName: String, productName: String, price: Double)
class MyRichMapper extends  RichMapFunction[SalePrice,String] {
  //声明值的状态
  var valueState:ValueState[Double]= _
  //声明list的状态
  lazy val listState:ListState[Int] = getRuntimeContext.getListState(new ListStateDescriptor[Int]("listState",classOf[Int]))
  //map
  lazy val mapState:MapState[String,Int] = getRuntimeContext.getMapState(new MapStateDescriptor[String,Int]("mapState",classOf[String],classOf[Int]))
  //reduce
  lazy val reduceState:ReducingState[SalePrice] = getRuntimeContext.getReducingState(new ReducingStateDescriptor[SalePrice]("reduceState",new MyReduce(),classOf[SalePrice]))

  override def open(parameters: Configuration): Unit = {
    //声明一个键控状态new ValueStateDescriptor[Double]("price",classOf[Double]) price 键控的名字,
  valueState = getRuntimeContext.getState(new ValueStateDescriptor[Double]("price",classOf[Double]))
  }
  override def map(value: SalePrice): String ={
    //状态的读写
    val myVal=valueState.value()
    //更新
    valueState.update(value.price)
    //list的状态
    //操作
    listState.add(1)
    listState.update(new util.ArrayList[Int](3))
    listState.get()
    value.boosName
  }
}
```

**对状态赋值**

```scala
class  MyReduce extends ReduceFunction[SalePrice]{
  override def reduce(value1: SalePrice, value2: SalePrice): SalePrice = ???
}
case class SalePrice(time: Long, boosName: String, productName: String, price: Double)
class MyRichMapper extends  RichMapFunction[SalePrice,String] {
  //声明值的状态
  var valueState:ValueState[Double]= _
  //声明list的状态
  lazy val listState:ListState[Int] = getRuntimeContext.getListState(new ListStateDescriptor[Int]("listState",classOf[Int]))
  //map
  lazy val mapState:MapState[String,Int] = getRuntimeContext.getMapState(new MapStateDescriptor[String,Int]("mapState",classOf[String],classOf[Int]))
  //reduce
  lazy val reduceState:ReducingState[SalePrice] = getRuntimeContext.getReducingState(new ReducingStateDescriptor[SalePrice]("reduceState",new MyReduce(),classOf[SalePrice]))

  override def open(parameters: Configuration): Unit = {
    //声明一个键控状态new ValueStateDescriptor[Double]("price",classOf[Double]) price 键控的名字,
  valueState = getRuntimeContext.getState(new ValueStateDescriptor[Double]("price",classOf[Double]))
  }
  override def map(value: SalePrice): String ={
    //状态的读写
    val myVal=valueState.value()
    //更新
    valueState.update(value.price)
    //list的状态
    //操作
    listState.add(1)
    listState.update(new util.ArrayList[Int](3))
    listState.get()
    value.boosName
  }
}
```

### 6 状态编程

利用Keyed State，实现这样一个需求：检测传感器的温度值，如果连续的两个温度差值超过10度，就输出报警。

```scala
//方法一： 
class TempChangeAlert(threshold: Double) extends KeyedProcessFunction[String, SensorReading, (String, Double, Double)] { } //底层API processFunction API
//方法二： 
class TempChangeAlert2(threshold: Double) extends RichFlatMapFunction[SensorReading, (String, Double, Double)]{ } //利用富函数 状态的初始值为0
//方法三： 
dataStream.keyBy(_.id)
        　　　　　　.flatMapWithState[(String, Double, Double), Double]{ } //FlatMap with keyed 
```

```scala
import akka.pattern.BackoffSupervisor.RestartCount
import com.xxx.fink.api.sourceapi.SensorReading
import org.apache.flink.api.common.functions.RichFlatMapFunction
import org.apache.flink.api.common.restartstrategy.RestartStrategies
import org.apache.flink.api.common.state.{ValueState, ValueStateDescriptor}
import org.apache.flink.streaming.api.functions.KeyedProcessFunction
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor
import org.apache.flink.streaming.api.scala.{DataStream, StreamExecutionEnvironment}
import org.apache.flink.streaming.api.windowing.time.Time
import org.apache.flink.util.Collector
import org.apache.flink.api.scala._
import org.apache.flink.configuration.Configuration
import org.apache.flink.contrib.streaming.state.RocksDBStateBackend
import org.apache.flink.runtime.executiongraph.restart.RestartStrategy
import org.apache.flink.streaming.api.environment.CheckpointConfig.ExternalizedCheckpointCleanup
import org.apache.flink.streaming.api.{CheckpointingMode, TimeCharacteristic}


/**
  *状态编程
  * 检测两次温度变化如果超过某个范围就报警，比如超过10°就报警；
  */
object StateTest {
  def main(args: Array[String]): Unit = {
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)


    //env.setStateBackend(new RocksDBStateBackend(""))
    val stream: DataStream[String] = env.socketTextStream("hadoop101", 7777)

    val dataStream: DataStream[SensorReading] = stream.map(data => {
      val dataArray: Array[String] = data.split(",")
      SensorReading(dataArray(0).trim, dataArray(1).trim.toLong, dataArray(2).trim.toDouble)
    })
      .assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[SensorReading](Time.seconds(1)) {
        override def extractTimestamp(element: SensorReading): Long = element.timestamp * 1000
      })

    //方法一:
    val processedStream1: DataStream[(String, Double, Double)] = dataStream.keyBy(_.id)
        .process(new TempChangeAlert(10.0))

    //方法二: 除了processFunction，其他也可以有状态
    val processedStream2: DataStream[(String, Double, Double)] = dataStream.keyBy(_.id)
      .flatMap(new TempChangeAlert2(10.0))

    //方法三: 带状态的flatMap
    val processedStream3: DataStream[(String, Double, Double)] = dataStream.keyBy(_.id)
        .flatMapWithState[(String, Double, Double), Double]{
      //如果没有状态的话，也就是没有数据过来，那么就将当前数据湿度值存入状态
      case (input: SensorReading, None) => (List.empty, Some(input.temperature))
      //如果有状态，就应该与上次的温度值比较差值，如果大于阈值就输出报警
      case(input: SensorReading, lastTemp: Some[Double]) =>
        val diff = (input.temperature - lastTemp.get).abs
        if (diff > 10.0){
          (List((input.id, lastTemp.get, input.temperature)), Some(input.temperature))
        }else{
          (List.empty, Some(input.temperature))
        }
    }

    dataStream.print("Input data:")
    processedStream3.print("process data:")

    env.execute("Window test")

  }

}

class TempChangeAlert(threshold: Double) extends KeyedProcessFunction[String, SensorReading, (String, Double, Double)] {
  //定义一个状态变量，保存上次的温度值
  lazy val lastTempState: ValueState[Double] = getRuntimeContext.getState(new ValueStateDescriptor[Double]("lastTemp", classOf[Double]))

  override def processElement(value: SensorReading, ctx: KeyedProcessFunction[String, SensorReading, (String, Double, Double)]#Context, out: Collector[(String, Double, Double)]): Unit = {
    //获取上次的温度值
    val lastTemp = lastTempState.value()
    //用当前的温度值和上次的做差，如果大于阈值，则输出报警
    val diff = (value.temperature - lastTemp).abs
    if (diff > threshold){
      out.collect(value.id, lastTemp, value.temperature)
    }
    lastTempState.update(value.temperature) //状态更新

  }
}

class TempChangeAlert2(threshold: Double) extends RichFlatMapFunction[SensorReading, (String, Double, Double)]{
  //flatMap本身是无状态的，富函数版本的函数类都可以去操作状态也有生命周期
  private var lastTempState: ValueState[Double] = _ //赋一个空值；
  //初始化的声明state变量
  override def open(parameters: Configuration): Unit = {
  //可以定义一个lazy；也可以在声明周期中拿；
    lastTempState = getRuntimeContext.getState(new ValueStateDescriptor[Double]("lastTemp", classOf[Double]))
  }

  override def flatMap(value: SensorReading, out: Collector[(String, Double, Double)]): Unit = {
    //获取上次的温度值
    val lastTemp = lastTempState.value()
    //用当前温度值和上次的求差值，如果大于阈值，输出报警信息
    val diff = (value.temperature - lastTemp).abs
    if (diff > threshold){
      out.collect(value.id, lastTemp, value.temperature)
    }
    lastTempState.update(value.temperature)
  }
}
```

### 7 状态后端

**状态管理（存储、访问、维护和检查点）**

1.每传入一条数据，有状态的算子任务都会读取和更新状态；

2.由于有效的状态访问对于处理数据的低延迟至关重要，因此每个并行任务都会在**本地内存**维护其状态，以确保快速的状态访问。

3.状态的存储、访问以及维护，由一个可插入的组件决定，这个组件就叫做状态后端（State Backend).

4.状态后端主要负责两件事：本地的状态管理，以及将检查点（checkpoint）状态写入远程存储。

**状态后端分类**

```
① MemoryStateBackend： 一般用于开发和测试
内存级的状态后端，会将键控状态作为内存中的对象进行管理，将它们存储在TaskManager的JVM堆上，而将checkpoint存储在JobManager的内存中；
特点快速、低延迟，但不稳定；
② FsStateBackend（文件系统状态后端）：生产环境
将checkpoint存到远程的持久化文件系统（FileSystem），HDFS上，而对于本地状态，跟MemoryStateBackend一样，也会存到TaskManager的JVM堆上。
同时拥有内存级的本地访问速度，和更好的容错保证；（如果是超大规模的需要保存还是无法解决，存到本地状态就可能OOM）
③ RocksDBStateBackend：
将所有状态序列化后，存入本地的RocksDB（本地数据库硬盘空间，序列化到磁盘）中存储，全部序列化存储到本地。
```

设置状态后端为FsStateBackend，并配置检查点和重启策略：

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-statebackend-rocksdb_2.12</artifactId>
    <version>1.10.1</version>
</dependency>
```

```scala
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(1);

// 1. 状态后端配置
//env.setStateBackend(new MemoryStateBackend()); 
env.setStateBackend(new FsStateBackend("", true))
//env.setStateBackend(new RocksDBStateBackend(""))


// 2. 检查点配置 开启checkpoint
env.enableCheckpointing(1000);

env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
env.getCheckpointConfig().setCheckpointTimeout(60000);
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500L);
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
env.getCheckpointConfig().setPreferCheckpointForRecovery(true);
env.getCheckpointConfig().setTolerableCheckpointFailureNumber(0);
env.getCheckpointConfig().setCheckpointInterval(10000L)

// 3. 重启策略配置
// 固定延迟重启（隔一段时间尝试重启一次）
env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
        3,  // 尝试重启次数
        100000L // 尝试重启的时间间隔，也可org.apache.flink.api.common.time.Time
));
env.setRestartStrategy(RestartStrategies.failureRateRestart(5, Time.minutes(5), Time.seconds(10)))
```



