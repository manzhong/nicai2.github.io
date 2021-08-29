---
title: Flink的TransFormation
tags: Flink
categories: Flink
encrypt: 
enc_pwd: 
abbrlink: 4907
date: 2020-10-20 19:22:30
summary_img:
---

## 一 简介

Flink 应用程序结构:

1、Source: 数据源，Flink 在流处理和批处理上的 source 大概有 4 类：基于本地集合的 source、基于文件的 source、基于网络套接字的 source、自定义的 source。自定义的 source 常见的有 Apache kafka、Amazon Kinesis Streams、RabbitMQ、Twitter Streaming API、Apache NiFi 等，当然你也可以定义自己的 source。

2、Transformation：数据转换的各种操作，有 Map / FlatMap / Filter / KeyBy / Reduce / Fold / Aggregations / Window / WindowAll / Union / Window join / Split / Select / Project 等，操作很多，可以将数据转换计算成你想要的数据。

3、Sink：接收器，Flink 将转换计算后的数据发送的地点 ，你可能需要存储下来，Flink 常见的 Sink 大概有如下几类：写入文件、打印出来、写入 socket 、自定义的 sink 。自定义的 sink 常见的有 Apache kafka、RabbitMQ、MySQL、ElasticSearch、Apache Cassandra、Hadoop FileSystem 等，同理你也可以定义自己的 Sink。

总体分为这三块,source 和sink之前已经说过,接下来介绍一下Transformation

## 二 Transformation(转换)

| Transformation                           | Description                              |
| ---------------------------------------- | ---------------------------------------- |
| **Map** DataStream → DataStream          | `dataStream.map { x => x * 2 }`          |
| **FlatMap** DataStream → DataStream      | `dataStream.flatMap { x => x.split(",") }` |
| **Filter** DataStream → DataStream       | `dataStream.filter { _ != 0 }`           |
| **KeyBy** DataStream → KeyedStream       | 将一个流分为不相交的区, 可以按照名称指定Key, 也可以按照角标来指定 `dataStream.keyBy(“key” |
| **Reduce** KeyedStream → DataStream      | **滚动**Reduce, 合并当前值和历史结果, 并发出新的结果值 `keyedStream.reduce { _ + _ }` |
| **Fold** KeyedStream → DataStream        | 按照初始值进行**滚动**折叠 `keyedStream.fold("start")((str, i) => { str + "-" + i })` |
| **Aggregations** KeyedStream → DataStream | **滚动**聚合, `sum`, `min`, `max`等 `keyedStream.sum(0)` |
| **Window** KeyedStream → DataStream      | 窗口函数, 根据一些特点对数据进行分组, 注意: 有可能是非并行的, 所有记录可能在一个任务中收集 `.window(TumblingEventTimeWindows.of(Time.seconds(5)))` |
| **WindowAll** DataStream → AllWindowedStream | 窗口函数, 根据一些特点对数据进行分组, 和window函数的主要区别在于可以不按照Key分组 `dataStream.windowAll (TumblingEventTimeWindows.of(Time.seconds(5)))` |
| **WindowApply** WindowedStream → DataStream | 将一个函数作用于整个窗口 `windowedStream.apply { WindowFunction }` |
| **WindowReduce** WindowedStream → DataStream | 在整个窗口上做一次reduce `windowedStream.reduce { _ + _ }` |
| **WindowFold** WindowedStream → DataStream | 在整个窗口上做一次fold `windowedStream.fold("start", (str, i) => { str + "-" + i })` |
| **Aggregations on windows** WindowedStream → DataStream | 在窗口上统计, `sub`, `max`, `min` `windowedStream.sum(10)` |
| **Union** DataStream* → DataStream       | 合并多个流 `dataStream.union(dataStream1, dataStream2, ...)` |
| **Window Join** DataStream → DataStream  | `dataStream.join(otherStream).where(...).equalTo(...) .window(TumblingEventTimeWindows.of(Time.seconds(3))).apply{..}` |
| **Window CoGroup** DataStream, DataStream → DataStream | `dataStream.coGroup(otherStream).where(0).equalTo(1).window(...).apply{...}` |
| **Connect** DataStream, DataStream → DataStream | 连接两个流, 并且保留各自的数据类型, 在这个连接中可以共享状态 `someStream.connect(otherStream)` |
| **Split** DataStream → SplitStream       | 将一个流切割为多个流 `someDataStream.split((x: Int) => x match ...)` |

### 1 map

和spark中的map并无区别,这是最简单的转换之一，其中输入是一个数据流，输出的也是一个数据流：

基础用法:

```scala
val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment 
val dataSet: DataSet[String] = env.fromElements("hello", "world", "java", "hello", "java") 
 dataSet.map(line => (line, 1)).print()
 /*(hello,1)
(world,1)
(java,1)
(hello,1)
(java,1)*/
```

重写map方法:

```scala
case class Student(id:Int,name:String,password:String,age:Int)

val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
val stu: DataStream[Student] = env.fromElements(new Student(1,"s","123",23))
stu.map(new MapFunction[Student,Student] {
  override def map(value: Student): Student = {
    //将每人年龄加5
    val student = new Student(value.id,value.name,value.password,value.age+5)
    return student
  }
}).print()
env.execute()
//result: Student(1,s,123,28)
```

### 2 flatmap

FlatMap 采用一条记录并输出零个，一个或多个记录

基础用法

```scala
    //wc.txt:  wq hello flink hello flink
    val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment
    val ds: DataSet[String] = env.readTextFile("path/wc.txt")
    val value: DataSet[String] = ds.flatMap(_.split(" "))
```

自定义用法:

```scala
case class Student(id:Int,name:String,password:String,age:Int)
val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    val stu: DataStream[Student] = env.fromElements(Student(1,"s","123",23),Student(2,"s2","1232",24)) stu.flatMap(new FlatMapFunction[Student, Student] {
      //将id为偶数的取出
      override def flatMap(value: Student, out: Collector[Student]): Unit = {
        if (value.id % 2 == 0) {
          out.collect(value)
        }
      }
 }).print()
 env.execute()
```

### 3 Filter

Filter 函数根据条件判断出结果。

```scala
    case class Student(id:Int,name:String,password:String,age:Int)
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    val stu: DataStream[Student] =        env.fromElements(Student(1,"s","123",23),Student(2,"s2","1232",24))
    stu.filter(new FilterFunction[Student] {
      //只取出id=2 的过滤出来，然后打印出来。
      override def filter(value: Student): Boolean = {
        if(value.id==2){
          return true
        }else {
          return false
        }
      }
    }).print()
    env.execute()
```

```scala
 //函数式
val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    val stu: DataStream[Student] = env.fromElements(Student(1,"s","123",23),Student(2,"s2","1232",24),Student(3,"s2","1232",24))
    val value1: DataStream[Student] = stu.filter(x => x.id==2)
```



### 3 keyby

 KeyBy 在逻辑上是基于 key 对流进行分区。在内部，它使用 hash 函数对流进行分区。它返回 KeyedDataStream 数据流。keyby后一般跟随聚合算子

```
min,sum,max('值取出最大值,其他的取第一条'),minBy('取出最小值对应的那一行数据'),maxBy('取出最大值对应的那一行数据')
```



```scala
 val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    val stu: DataStream[Student] =    env.fromElements(Student(1,"s","123",23),Student(2,"s2","1232",24),Student(3,"s2","1232",24))
    stu.keyBy(new KeySelector[Student,Int] {
     //对student 的 age 做 KeyBy 操作分区
      override def getKey(value: Student): Int = {
        return value.age
      }
    }).print()
    env.execute() 
//函数式
   // stu.keyBy(x => x.age)
```

### 4 reduce

  Reduce 返回单个的结果值，并且 reduce 操作每处理一个元素总是创建一个新值。常用的方法有 average, sum, min, max, count，使用 reduce 方法都可实现。

```scala
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    val stu: DataStream[Student] = env.fromElements(Student(1,"s","123",23),Student(2,"s2","1232",24),Student(3,"s2","1232",24))
stu.keyBy(new KeySelector[Student,Int] {
      override def getKey(value: Student): Int = {
        return value.age
      }
    }).reduce(new ReduceFunction[Student] {
      override def reduce(value1: Student, value2: Student): Student = {
        import mz.flinktransformtion.Student
        Student((value1.id + value2.id) / 2,value1.name + value2.name,
          value1.password + value2.password,(value1.age + value2.age) / 2
        )
      }
    }).print()
    env.execute()
/*
1> Student(2,s2,1232,24)
4> Student(1,s,123,23)
1> Student(2,s2s2,12321232,24)
*/
//函数时
  stu.keyBy(x => x.age).reduce( (curd,newD) =>
     Student(curd.id,newD.name,curd.password,curd.age)
    )
```

上面先将数据流进行 keyby 操作，因为执行 reduce 操作只能是 KeyedStream，然后将 student 对象的 age 做了一个求平均值的操作。

### 5 Fold

Fold 通过将最后一个文件夹流与当前记录组合来推出 KeyedStream。 它会发回数据流。可以定制初始值的算子

```java
KeyedStream.fold("1", new FoldFunction<Integer, String>() {
    @Override
    public String fold(String accumulator, Integer value) throws Exception {
        return accumulator + "=" + value;
    }
})
 /*输入 (1,2,3,4,5)  输出:"1=1","1=1=2","1=1=2=3"...
```

### 6 Aggregations

DataStream API 支持各种聚合，例如 min，max，sum 等。 这些函数可以应用于 KeyedStream 以获得 Aggregations 聚合。

```scala
KeyedStream.sum(0) 
KeyedStream.sum("key") 
KeyedStream.min(0) 
KeyedStream.min("key") 
KeyedStream.max(0) 
KeyedStream.max("key") 
KeyedStream.minBy(0) 
KeyedStream.minBy("key") 
KeyedStream.maxBy(0) 
KeyedStream.maxBy("key")
```

### 7 window

Window 函数允许按时间或其他条件对现有 KeyedStream 进行分组。 以下是以 10 秒的时间窗口聚合：

```scala
inputStream.keyBy(0).window(Time.seconds(10));
```

Flink 定义数据片段以便（可能）处理无限数据流。 这些切片称为窗口。 此切片有助于通过应用转换处理数据块。 要对流进行窗口化，我们需要分配一个可以进行分发的键和一个描述要对窗口化流执行哪些转换的函数

要将流切片到窗口，我们可以使用 Flink 自带的窗口分配器。 我们有选项，如 tumbling windows, sliding windows, global 和 session windows。 Flink 还允许您通过扩展 WindowAssginer 类来编写自定义窗口分配器,这一块请参看我其他文章

### 8 windowAll

windowAll 函数允许对常规数据流进行分组。 通常，这是非并行数据转换，因为它在非分区数据流上运行。

与常规数据流功能类似，我们也有窗口数据流功能。 唯一的区别是它们处理窗口数据流。 所以窗口缩小就像 Reduce 函数一样，Window fold 就像 Fold 函数一样，并且还有聚合。

```scala
inputStream.keyBy(0).windowAll(Time.seconds(10));
```

### 9 Window join

我们可以通过一些 key 将同一个 window 的两个数据流 join 起来。

```scala
inputStream.join(inputStream1)
           .where(0).equalTo(1)
           .window(Time.seconds(5))     
           .apply (new JoinFunction () {...});
//以上示例是在 5 秒的窗口中连接两个流，其中第一个流的第一个属性的连接条件等于另一个流的第二个属性。
```

### 10 Split 和select

此功能根据条件将流拆分为两个或多个流。 当您获得混合流并且您可能希望单独处理每个数据流时，可以使用此方法。

此方法并没有真正的将流分割,进行split后得到SplitStream, 只是在一个流里做了一个分类.若要得到两个流,要用到select

```scala
SplitStream<Integer> split = inputStream.split(new OutputSelector<Integer>() {
    @Override
    public Iterable<String> select(Integer value) {
        List<String> output = new ArrayList<String>(); 
        if (value % 2 == 0) {
            output.add("even");
        }
        else {
            output.add("odd");
        }
        return output;
    }
});
```

此功能允许您从拆分流中选择特定流。

```scala
SplitStream<Integer> split;
val even:DataStream[Int] = split.select("even"); 
val add:DataStream[Int] = split.select("odd"); 
val all:DataStream[Int] = split.select("even","odd");
```

```scala
val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    val stu: DataStream[Student] = env.fromElements(Student(1,"s","123",23),Student(2,"s2","1232",24),Student(3,"s2","1232",24))
val st:SplitStream[Student] = stu.split(st => {
  if (st.age > 30) Seq("中年以上") else Seq("青年")
})
val dd2: DataStream[Student] = st.select("中年以上") 
val dd3: DataStream[Student] = st.select("青年") 
val dd4: DataStream[Student] = st.select("青年","中年以上" ) 
```

split已经不推荐使用,推荐使用 side output instead 

### 11 合流操作

#### 1 connect和comap

连接(两个)类型的流,合并后,只是放在同一个流中,内部依然保持各自的形式和数据不变,两个流相互独立,

```scala
//comap和coflatmap
作用于connectedStream上,功能与map和flatmap一致,对connectedStream中的每一个流分别进行map和flatmap,后转为DataStream
```

```scala
    val st:SplitStream[Student] = stu.split(st => {
      if (st.age > 30) Seq("中年以上") else Seq("青年")
    })
    val da1: DataStream[Student] = st.select("青年")
    val da2: DataStream[Student] = st.select("中年以上")
    val conStream: ConnectedStreams[Student, Student] = da1.connect(da2)
    //用comap对数据进行分别处理
    val result: DataStream[String] = conStream.map(da1 =>
      da1.password
      , da2 => da2.name)
```

#### 2 union

Union 函数将两个或多个数据流结合在一起。 这样就可以并行地组合数据流。 如果我们将一个流与自身组合，那么它会输出每个记录两次。(**要求两个流必须是相同的数据类型**)

```scala
//inputStream.union(inputStream1, inputStream2, ...)
 val st:SplitStream[Student] = stu.split(st => {
   if (st.age > 30) Seq("中年以上") else Seq("青年")
 })
 val da1: DataStream[Student] = st.select("青年")
 val da2: DataStream[Student] = st.select("中年以上")
 val unionStream: DataStream[Student] = da1.union(da2)
```

### 12 Project

Project 函数允许您从事件流中选择属性子集，并仅将所选元素发送到下一个处理流。

```java
DataStream<Tuple4<Integer, Double, String, String>> in = // [...] 
DataStream<Tuple2<String, String>> out = in.project(3,2);
```

上述函数从给定记录中选择属性号 2 和 3。 以下是示例输入和输出记录：

```java
(1,10.0,A,B)=> (B,A)
(2,20.0,C,D)=> (D,C)
```

### 富函数

```scala
//富函数可以获取运行时上下文,还有生命周期
class MyRichMapper extends RichMapFunction[Student,String] {
  //重写map
  override def map(value: Student): String = ???
//关闭
  override def close(): Unit = super.close()
}
```

​     	

​	本文主要介绍了 Flink Data 的常用转换方式：Map、FlatMap、Filter、KeyBy、Reduce、Fold、Aggregations、Window、WindowAll、Union、Window Join、Split、Select、Project 等。并用了点简单的 demo 介绍了如何使用，具体在项目中该如何将数据流转换成我们想要的格式，还需要根据实际情况对待。



