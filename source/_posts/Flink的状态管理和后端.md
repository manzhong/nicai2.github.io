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

## 一 简介

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

## 二  分类

在flink中,状态始终与特定算子相关联,为了使运行的flink了解算子的状态,**算子需要预先注册其状态**

1. 托管状态（Managed State）和原始状态（Raw State）Flink 的状态有两种：托管状态（Managed State）和原始状态（Raw State）。托管状态就是由 Flink 统一管理的，状态的存储访问、故障恢复和重组等一系列问题都由 Flink 实现，我们只要调接口就可以；而原始状态则是自定义的，相当于就是开辟了一块内存，需要我们自己管理，实现状态的序列化和故障恢复。具体来讲，托管状态是由 Flink 的运行时（Runtime）来托管的；在配置容错机制后，状态会自动持久化保存，并在发生故障时自动恢复。当应用发生横向扩展时，状态也会自动地重组分配到所有的子任务实例上。对于具体的状态内容，Flink 也提供了值状态（ValueState）、列表状态（ListState）、映射状态（MapState）、聚合状态（AggregateState）等多种结构，内部支持各种数据类型。聚合、窗口等算子中内置的状态，就都是托管状态；我们也可以在富函数类（RichFunction）中通过上下文来自定义状态，这些也都是托管状态。而对比之下，原始状态就全部需要自定义了。Flink 不会对状态进行任何自动操作，也不知道状态的具体数据类型，只会把它当作最原始的字节（Byte）数组来存储。我们需要花费大量的精力来处理状态的管理和维护。所以只有在遇到托管状态无法实现的特殊需求时，我们才会考虑使用原始状态；一般情况下不推荐使用。绝大多数应用场景，我们都可以用 Flink 提供的算子或者自定义托管状态来实现需求。

2. 算子状态（Operator State）和按键分区状态（Keyed State）

   我们知道在 Flink 中，一个算子任务会按照并行度分为多个并行子任务执行，而不同的子241任务会占据不同的任务槽（task slot）。由于不同的 slot 在计算资源上是物理隔离的，所以 Flink能管理的状态在并行任务间是无法共享的，每个状态只能针对当前子任务的实例有效。而很多有状态的操作（比如聚合、窗口）都是要先做 keyBy 进行按键分区的。按键分区之后，任务所进行的所有计算都应该只针对当前 key 有效，所以状态也应该按照 key 彼此隔离。在这种情况下，状态的访问方式又会有所不同。基于这样的想法，我们又可以将托管状态分为两类：算子状态和按键分区状态

```
算子状态
状态作用范围限定为当前的算子任务实例，也就是只对当前并行子任务实例有效。这就意
味着对于一个并行子任务，占据了一个“分区”，它所处理的所有数据都会访问到相同的状态，
状态对于同一任务而言是共享的,算子状态可以用在所有算子上，使用的时候其实就跟一个本地变量没什么区别——因为本地变量的作用域也是当前任务实例。在使用时，我们还需进一步实现 CheckpointedFunction 接
口
按键分区状态
状态是根据输入流中定义的键（key）来维护和访问的，所以只能定义在按键分区流
（KeyedStream）中，也就 keyBy 之后才可以使用，按键分区状态应用非常广泛。之前讲到的聚合算子必须在 keyBy 之后才能使用，就是因为聚合的结果是以 Keyed State 的形式保存的。另外，也可以通过富函数类（Rich Function）来自定义 Keyed State，所以只要提供了富函数类接口的算子，也都可以使用 Keyed State。所以即使是 map、filter 这样无状态的基本转换算子，我们也可以通过富函数类给它们“追
加”Keyed State，或者实现 CheckpointedFunction 接口来定义 Operator State；从这个角度讲，
Flink 中所有的算子都可以是有状态的，不愧是“有状态的流处理”。

无论是 Keyed State 还是 Operator State，它们都是在本地实例上维护的，也就是说每个并
行子任务维护着对应的状态，算子的子任务之间状态不共享
```

总的来说,有两种类型的状态:

```
算子状态(operator state)
	算子状态的作用范围为算子任务
键控状态(keyed state) 分组
	根据输入数据流中定义的键(key)来维护和访问,基于KeyBy--KeyedStream上有任务出现的状态，定义的不同的key来维护这个状态；不同的key也是独立访问的，一个key只能访问它自己的状态，不同key之间也不能互相访问
```

## 三 算子状态

算子状态（Operator State）就是一个算子并行实例上定义的状态，作用范围被限定为当前算子任务。算子状态跟数据的 key 无关，所以不同 key 的数据只要被分发到同一个并行子任务，就会访问到同一个 Operator State。

算子状态的实际应用场景不如 Keyed State 多，一般用在 Source 或 Sink 等与外部系统连接的算子上，或者完全没有 key 定义的场景。比如 Flink 的 Kafka 连接器中，就用到了算子状态。在我们给 Source 算子设置并行度后，Kafka 消费者的每一个并行实例，都会为对应的主题261（topic）分区维护一个偏移量， 作为算子状态保存起来。这在保证 Flink 应用“精确一次”（exactly-once）状态一致性时非常有用

```
1:算子状态的作用范围限定为算子任务,由同一并行任务所处理的所有数据都可以访问到相同的状态
2:状态对于同一子任务而言是共享的
3:算子的状态不能由相同或不同算子的另一个子任务访问
```

**算子状态的数据结构**

```
① 列表状态（List state），将状态表示为一组数据的列表；（会根据并行度的调整把之前的状态重新分组重新分配）当算子并行度进行缩放调整时，算子的列表状态中的所有元素项会被统一收集起来，相当
于把多个分区的列表合并成了一个“大列表”，然后再均匀地分配给所有并行任务。这种“均
匀分配”的具体方法就是“轮询”（round-robin）
② 联合列表状态（Union list state），也将状态表示为数据的列表，它常规列表状态的区别在于，在发生故障时，或者从保存点（savepoint）启动应用程序时如何恢复（把之前的每一个状态广播到对应的每个算子中）。
③ 广播状态（Broadcast state），如果一个算子有多项任务，而它的每项任务状态又都相同，那么这种特殊情况最适合应用广播状态（把同一个状态广播给所有算子子任务）；
```

#### 1 代码实现

我们已经知道，状态从本质上来说就是算子并行子任务实例上的一个特殊本地变量。它的特殊之处就在于 Flink 会提供完整的管理机制，来保证它的持久化保存，以便发生故障时进行状态恢复；另外还可以针对不同的 key 保存独立的状态实例。按键分区状态（Keyed State）对这两个功能都要考虑；而算子状态（Operator State）并不考虑 key 的影响，所以主要任务就是要让 Flink 了解状态的信息、将状态数据持久化后保存到外部存储空间。看起来算子状态的使用应该更加简单才对。不过仔细思考又会发现一个问题：我们对状态进行持久化保存的目的是为了故障恢复；在发生故障、重启应用后，数据还会被发往之前分配的分区吗？显然不是，因为并行度可能发生了调整，不论是按键（key）的哈希值分区，还是直接轮询（round-robin）分区，数据分配到的分区都会发生变化。这很好理解，当打牌的人数从 3 个增加到 4 个时，即使牌的次序不变，轮流发到每个人手里的牌也会不同。数据分区发生变化，带来的问题就是，怎么保证原先的状态跟故障恢复后数据的对应关系呢？对于 Keyed State 这个问题很好解决：状态都是跟 key 相关的，而相同 key 的数据不管发往哪个分区，总是会全部进入一个分区的；于是只要将状态也按照 key 的哈希值计算出对应的分区，进行重组分配就可以了。恢复状态后继续处理数据，就总能按照 key 找到对应之前的状态，就保证了结果的一致性。所以 Flink 对 Keyed State 进行了非常完善的包装，我们不需实现任何接口就可以直接使用。而对于 Operator State 来说就会有所不同。因为不存在 key，所有数据发往哪个分区是不可预测的；也就是说，当发生故障重启之后，我们不能保证某个数据跟之前一样，进入到同一个并行子任务、访问同一个状态。所以 Flink 无法直接判断该怎样保存和恢复状态，而是提供了接口，让我们根据业务需求自行设计状态的快照保存（snapshot）和恢复（restore）逻辑。

自定义的 SinkFunction 会在CheckpointedFunction 中进行数据缓存，然后统一发送到下游

```java
public class BufferingSinkExample {
 public static void main(String[] args) throws Exception{
 StreamExecutionEnvironment env =
StreamExecutionEnvironment.getExecutionEnvironment();
 env.setParallelism(1);
 SingleOutputStreamOperator<Event> stream = env.addSource(new
ClickSource())
 .assignTimestampsAndWatermarks(WatermarkStrategy.<Event>forMonot
onousTimestamps()
 .withTimestampAssigner(new
SerializableTimestampAssigner<Event>() {
 @Override
 public long extractTimestamp(Event element, long
recordTimestamp) {
 return element.timestamp;
 }
 })
 );
 stream.print("input");
 // 批量缓存输出
 stream.addSink(new BufferingSink(10));
 env.execute();
 }
 public static class BufferingSink implements SinkFunction<Event>,
CheckpointedFunction {
 private final int threshold;
 private transient ListState<Event> checkpointedState;
 private List<Event> bufferedElements;
 public BufferingSink(int threshold) {
 this.threshold = threshold;
 this.bufferedElements = new ArrayList<>();
 }
 @Override
 public void invoke(Event value, Context context) throws Exception {
 bufferedElements.add(value);
 if (bufferedElements.size() == threshold) {
 for (Event element: bufferedElements) {
 // 输出到外部系统，这里用控制台打印模拟
 System.out.println(element);
 }
 System.out.println("==========输出完毕=========");
 bufferedElements.clear();
 }
 }
 @Override
 public void snapshotState(FunctionSnapshotContext context) throws
Exception {
 checkpointedState.clear();
 // 把当前局部变量中的所有元素写入到检查点中
 for (Event element : bufferedElements) {
 checkpointedState.add(element);
 }
 }
 @Override
 public void initializeState(FunctionInitializationContext context) throws
Exception {
 ListStateDescriptor<Event> descriptor = new ListStateDescriptor<>(
 "buffered-elements",
 Types.POJO(Event.class));
 checkpointedState =
context.getOperatorStateStore().getListState(descriptor);
 // 如果是从故障中恢复，就将 ListState 中的所有元素添加到局部变量中
 if (context.isRestored()) {
 for (Event element : checkpointedState.get()) {
 bufferedElements.add(element);
 }
 }
 }
 }
}
```

当初始化好状态对象后，我们可以通过调用. isRestored()方法判断是否是从故障中恢复。在代码中 BufferingSink 初始化时，恢复出的 ListState 的所有元素会添加到一个局部变量bufferedElements 中，以后进行检查点快照时就可以直接使用了。在调用.snapshotState()时，直接清空 ListState，然后把当前局部变量中的所有元素写入到检查点中。对于不同类型的算子状态，需要调用不同的获取状态对象的接口，对应地也就会使用不同的状态分配重组算法。比如获取列表状态时，调用.getListState() 会使用最简单的 平均分割重组（even-split redistribution）算法；而获取联合列表状态时，调用的是.getUnionListState() ，对应就会使用联合重组（union redistribution） 算法

#### 2 广播状态

算子状态中有一类很特殊，就是广播状态（Broadcast State）。从概念和原理上讲，广播状态非常容易理解：状态广播出去，所有并行子任务的状态都是相同的；并行度调整时只要直接复制就可以了。然而在应用上，广播状态却与其他算子状态大不相同。广播状态与其他算子状态的列表（list）结构不同，底层是以键值对（key-value）形式描述的，所以其实就是一个映射状态（MapState）

```
场景举例：
1动态更新计算规则: 如事件流需要根据最新的规则进行计算，则可将规则作为广播状态广播到下游Task中。
2实时增加额外字段: 如事件流需要实时增加用户的基础信息，则可将用户的基础信息作为广播状态广播到下游Task中。
```

在代码上，可以直接调用 DataStream 的.broadcast()方法，传入一个“映射状态描述器”（MapStateDescriptor）说明状态的名称和类型，就可以得到一个“广播流”（BroadcastStream）；进而将要处理的数据流与这条广播流进行连接（connect），就会得到“广播连接流”（BroadcastConnectedStream）。**注意广播状态只能用在广播连接流中**

```
API介绍：

        首先创建一个Keyed 或Non-Keyed 的DataStream，然后再创建一个BroadcastedStream，最后通过DataStream来连接(调用connect 方法)到Broadcasted Stream 上，这样实现将BroadcastState广播到Data Stream 下游的每个Task中。
```

1: 如果DataStream是Keyed Stream ，则连接到Broadcasted Stream 后， 添加处理ProcessFunction 时需要使用KeyedBroadcastProcessFunction 来实现， 下面是KeyedBroadcastProcessFunction 的API

```java
public abstract class KeyedBroadcastProcessFunction<KS, IN1, IN2, OUT> extends BaseBroadcastProcessFunction {
    public abstract void processElement(final IN1 value, final ReadOnlyContext ctx, final Collector<OUT> out) throws Exception;
    public abstract void processBroadcastElement(final IN2 value, final Context ctx, final Collector<OUT> out) throws Exception;
}
```

上面泛型中的各个参数的含义，说明如下：

KS：表示Flink 程序从最上游的Source Operator 开始构建Stream，当调用keyBy 时所依赖的Key 的类型；
IN1：表示非Broadcast 的Data Stream 中的数据记录的类型；
IN2：表示Broadcast Stream 中的数据记录的类型；
OUT：表示经过KeyedBroadcastProcessFunction 的processElement()和processBroadcastElement()方法处理后输出结果数据记录的类型。
2 如果Data Stream 是Non-Keyed Stream，则连接到Broadcasted Stream 后，添加处理ProcessFunction 时需要使用BroadcastProcessFunction 来实现， 下面是BroadcastProcessFunction 的API，代码如下所示：

```java
public abstract class BroadcastProcessFunction<IN1, IN2, OUT> extends BaseBroadcastProcessFunction {
		public abstract void processElement(final IN1 value, final ReadOnlyContext ctx, final Collector<OUT> out) throws Exception;
		public abstract void processBroadcastElement(final IN2 value, final Context ctx, final Collector<OUT> out) throws Exception;
}
```

注意事项：

Broadcast State 是Map 类型，即K-V 类型。
Broadcast State 只有在广播的一侧, 即在BroadcastProcessFunction 或KeyedBroadcastProcessFunction 的processBroadcastElement 方法中可以修改。在非广播的一侧， 即在BroadcastProcessFunction 或KeyedBroadcastProcessFunction 的processElement 方法中只读。
Broadcast State 中元素的顺序，在各Task 中可能不同。基于顺序的处理，需要注意。
Broadcast State 在Checkpoint 时，每个Task 都会Checkpoint 广播状态。
Broadcast State 在运行时保存在内存中，目前还不能保存在RocksDB State Backend 中。

##### 1 实现配置动态更新

实时过滤出配置中的用户，并在事件流中补全这批用户的基础信息。

1事件流：表示用户在某个时刻浏览或点击了某个商品，格式如下

```
{"userID": "user_3", "eventTime": "2019-08-17 12:19:47", "eventType": "browse", "productID": 1}

{"userID": "user_2", "eventTime": "2019-08-17 12:19:48", "eventType": "click", "productID": 1}
```

2 配置数据: 表示用户的详细信息，在Mysql中，如下

```sql
DROP TABLE IF EXISTS user_info;

CREATE TABLE user_info  (

  userID varchar(20) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL,

  userName varchar(10) CHARACTER SET utf8 COLLATE utf8_general_ci NULL DEFAULT NULL,

  userAge int(11) NULL DEFAULT NULL,

  PRIMARY KEY (userID) USING BTREE

) ENGINE = MyISAM CHARACTER SET = utf8 COLLATE = utf8_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------

-- Records of user_info

-- ----------------------------

INSERT INTO user_info VALUES ('user_1', '张三', 10);

INSERT INTO user_info VALUES ('user_2', '李四', 20);

INSERT INTO user_info VALUES ('user_3', '王五', 30);

INSERT INTO user_info VALUES ('user_4', '赵六', 40);

SET FOREIGN_KEY_CHECKS = 1;

```

2 输出结果

```
(user_3,2019-08-17 12:19:47,browse,1,王五,33)
(user_2,2019-08-17 12:19:48,click,1,李四,20)
```

步骤：

```
1.	env
 
2.	source
    2.1. 构建实时数据事件流-自定义随机
        <userID, eventTime, eventType, productID>
    2.2.构建配置流-从MySQL
        <用户id,<姓名,年龄>>
 
3.	transformation
    3.1. 定义状态描述器
        MapStateDescriptor<Void, Map<String, Tuple2<String, Integer>>> descriptor =
        new MapStateDescriptor<>("config",Types.VOID, Types.MAP(Types.STRING, Types.TUPLE(Types.STRING, Types.INT)));
    3.2. 广播配置流
        BroadcastStream<Map<String, Tuple2<String, Integer>>> broadcastDS = configDS.broadcast(descriptor);
    3.3. 将事件流和广播流进行连接
        BroadcastConnectedStream<Tuple4<String, String, String, Integer>, Map<String, Tuple2<String, Integer>>> connectDS =eventDS.connect(broadcastDS);
    3.4. 处理连接后的流-根据配置流补全事件流中的用户的信息
 
 
4.	sink
5.	execute
```

````java
import org.apache.flink.api.common.state.BroadcastState;
import org.apache.flink.api.common.state.MapStateDescriptor;
import org.apache.flink.api.common.state.ReadOnlyBroadcastState;
import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.api.java.tuple.Tuple4;
import org.apache.flink.api.java.tuple.Tuple6;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.datastream.BroadcastConnectedStream;
import org.apache.flink.streaming.api.datastream.BroadcastStream;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.co.BroadcastProcessFunction;
import org.apache.flink.streaming.api.functions.source.RichSourceFunction;
import org.apache.flink.streaming.api.functions.source.SourceFunction;
import org.apache.flink.util.Collector;
 
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.Random;
 
/**
 * Desc
 * 需求:
 * 使用Flink的BroadcastState来完成
 * 事件流和配置流(需要广播为State)的关联,并实现配置的动态更新!
 */
public class BroadcastStateConfigUpdate {
    public static void main(String[] args) throws Exception{
        //1.env
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        //2.source
        //-1.构建实时的自定义随机数据事件流-数据源源不断产生,量会很大
        //<userID, eventTime, eventType, productID>
        DataStreamSource<Tuple4<String, String, String, Integer>> eventDS = env.addSource(new MySource());
 
        //-2.构建配置流-从MySQL定期查询最新的,数据量较小
        //<用户id,<姓名,年龄>>
        DataStreamSource<Map<String, Tuple2<String, Integer>>> configDS = env.addSource(new MySQLSource());
 
        //3.transformation
        //-1.定义状态描述器-准备将配置流作为状态广播
        MapStateDescriptor<Void, Map<String, Tuple2<String, Integer>>> descriptor =
                new MapStateDescriptor<>("config", Types.VOID, Types.MAP(Types.STRING, Types.TUPLE(Types.STRING, Types.INT)));
        //-2.将配置流根据状态描述器广播出去,变成广播状态流
        BroadcastStream<Map<String, Tuple2<String, Integer>>> broadcastDS = configDS.broadcast(descriptor);
 
        //-3.将事件流和广播流进行连接
        BroadcastConnectedStream<Tuple4<String, String, String, Integer>, Map<String, Tuple2<String, Integer>>> connectDS =eventDS.connect(broadcastDS);
        //-4.处理连接后的流-根据配置流补全事件流中的用户的信息
        SingleOutputStreamOperator<Tuple6<String, String, String, Integer, String, Integer>> result = connectDS
                //BroadcastProcessFunction<IN1, IN2, OUT>
                .process(new BroadcastProcessFunction<
                //<userID, eventTime, eventType, productID> //事件流
                Tuple4<String, String, String, Integer>,
                //<用户id,<姓名,年龄>> //广播流
                Map<String, Tuple2<String, Integer>>,
                //<用户id，eventTime，eventType，productID，姓名，年龄> //需要收集的数据
                Tuple6<String, String, String, Integer, String, Integer>>() {
 
            //处理事件流中的元素
            @Override
            public void processElement(Tuple4<String, String, String, Integer> value, ReadOnlyContext ctx, Collector<Tuple6<String, String, String, Integer, String, Integer>> out) throws Exception {
                //取出事件流中的userId
                String userId = value.f0;
                //根据状态描述器获取广播状态
                ReadOnlyBroadcastState<Void, Map<String, Tuple2<String, Integer>>> broadcastState = ctx.getBroadcastState(descriptor);
                if (broadcastState != null) {
                    //取出广播状态中的map<用户id,<姓名,年龄>>
                    Map<String, Tuple2<String, Integer>> map = broadcastState.get(null);
                    if (map != null) {
                        //通过userId取map中的<姓名,年龄>
                        Tuple2<String, Integer> tuple2 = map.get(userId);
                        //取出tuple2中的姓名和年龄
                        String userName = tuple2.f0;
                        Integer userAge = tuple2.f1;
                        out.collect(Tuple6.of(userId, value.f1, value.f2, value.f3, userName, userAge));
                    }
                }
            }
 
            //处理广播流中的元素
            @Override
            public void processBroadcastElement(Map<String, Tuple2<String, Integer>> value, Context ctx, Collector<Tuple6<String, String, String, Integer, String, Integer>> out) throws Exception {
                //value就是MySQLSource中每隔一段时间获取到的最新的map数据
                //先根据状态描述器获取历史的广播状态
                BroadcastState<Void, Map<String, Tuple2<String, Integer>>> broadcastState = ctx.getBroadcastState(descriptor);
                //再清空历史状态数据
                broadcastState.clear();
                //最后将最新的广播流数据放到state中（更新状态数据）
                broadcastState.put(null, value);
            }
        });
        //4.sink
        result.print();
        //5.execute
        env.execute();
    }
 
    /**
     * <userID, eventTime, eventType, productID>
     */
    public static class MySource implements SourceFunction<Tuple4<String, String, String, Integer>>{
        private boolean isRunning = true;
        @Override
        public void run(SourceContext<Tuple4<String, String, String, Integer>> ctx) throws Exception {
            Random random = new Random();
            SimpleDateFormat df = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            while (isRunning){
                int id = random.nextInt(4) + 1;
                String user_id = "user_" + id;
                String eventTime = df.format(new Date());
                String eventType = "type_" + random.nextInt(3);
                int productId = random.nextInt(4);
                ctx.collect(Tuple4.of(user_id,eventTime,eventType,productId));
                Thread.sleep(500);
            }
        }
 
        @Override
        public void cancel() {
            isRunning = false;
        }
    }
    /**
     * <用户id,<姓名,年龄>>
     */
    public static class MySQLSource extends RichSourceFunction<Map<String, Tuple2<String, Integer>>> {
        private boolean flag = true;
        private Connection conn = null;
        private PreparedStatement ps = null;
        private ResultSet rs = null;
 
        @Override
        public void open(Configuration parameters) throws Exception {
            conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/bigdata", "root", "root");
            String sql = "select `userID`, `userName`, `userAge` from `user_info`";
            ps = conn.prepareStatement(sql);
        }
        @Override
        public void run(SourceContext<Map<String, Tuple2<String, Integer>>> ctx) throws Exception {
            while (flag){
                Map<String, Tuple2<String, Integer>> map = new HashMap<>();
                ResultSet rs = ps.executeQuery();
                while (rs.next()){
                    String userID = rs.getString("userID");
                    String userName = rs.getString("userName");
                    int userAge = rs.getInt("userAge");
                    //Map<String, Tuple2<String, Integer>>
                    map.put(userID,Tuple2.of(userName,userAge));
                }
                ctx.collect(map);
                Thread.sleep(5000);//每隔5s更新一下用户的配置信息!
            }
        }
        @Override
        public void cancel() {
            flag = false;
        }
        @Override
        public void close() throws Exception {
            if (conn != null) conn.close();
            if (ps != null) ps.close();
            if (rs != null) rs.close();
        }
    }
}
````















## 四 键控状态

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

### 4.1 键控状态的使用

因为声明一个键控状态需要获取上下文对象,所以必须在富函数中声明,在富函数中，我们可以调用.getRuntimeContext()获取当前的运行时上下文（RuntimeContext），进而获取到访问状态的句柄；这种富函数中自定义的状态也是 Keyed State

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

#### 1 值状态

我们这里会使用用户 id 来进行分流，然后分别统计每个用户的 pv 数据，由于我们并不想每次 pv 加一，就将统计结果发送到下游去，所以这里我们注册了一个定时器，用来隔一段时间发送 pv 的统计结果，这样对下游算子的压力不至于太大。具体实现方式是定义一个用来保存定时器时间戳的值状态变量。当定时器触发并向下游发送数据以后，便清空储存定时器时间戳的状态变量，这样当新的数据到来时，发现并没有定时器存在，就可以注册新的定时器了，注册完定时器之后将定时器的时间戳继续保存在状态变量中。

```java
// 统计每个用户的 pv，隔一段时间（10s）输出一次结果 
stream.keyBy(data -> data.user)
 .process(new PeriodicPvResult())
 .print();
 env.execute();
 }
 // 注册定时器，周期性输出 pv
 public static class PeriodicPvResult extends
KeyedProcessFunction<String ,Event, String>{
 // 定义两个状态，保存当前 pv 值，以及定时器时间戳
 ValueState<Long> countState;
 ValueState<Long> timerTsState;
 @Override
 public void open(Configuration parameters) throws Exception {
 countState = getRuntimeContext().getState(new
ValueStateDescriptor<Long>("count", Long.class));
 timerTsState = getRuntimeContext().getState(new
ValueStateDescriptor<Long>("timerTs", Long.class));
 }
 @Override
 public void processElement(Event value, Context ctx, Collector<String> out)
throws Exception {
 // 更新 count 值
 Long count = countState.value();
 if (count == null){
 countState.update(1L);
 } else {
 countState.update(count + 1);
 }
 // 注册定时器
 if (timerTsState.value() == null){
 ctx.timerService().registerEventTimeTimer(value.timestamp + 10 *
250
1000L);
 timerTsState.update(value.timestamp + 10 * 1000L);
 }
 }
 @Override
 public void onTimer(long timestamp, OnTimerContext ctx, Collector<String>
out) throws Exception {
 out.collect(ctx.getCurrentKey() + " pv: " + countState.value());
 // 清空状态
 timerTsState.clear();
 }
 }}

```

#### 2 映射状态

映射状态的用法和 Java 中的 HashMap 很相似。在这里我们可以通过 MapState 的使用来探索一下窗口的底层实现，也就是我们要用映射状态来完整模拟窗口的功能。这里我们模拟一个滚动窗口。我们要计算的是每一个 url 在每一个窗口中的 pv 数据。我们之前使用增量聚合和全窗口聚合结合的方式实现过这个需求。这里我们用 MapState 再来实现一下

```java
// 统计每 10s 窗口内，每个 url 的 pv
 stream.keyBy(data -> data.url)
 .process(new FakeWindowResult(10000L))
 .print();
 env.execute();
 }
 public static class FakeWindowResult extends KeyedProcessFunction<String,
Event, String>{
 // 定义属性，窗口长度
 private Long windowSize;
 public FakeWindowResult(Long windowSize) {
 this.windowSize = windowSize;
 }
 // 声明状态，用 map 保存 pv 值（窗口 start，count）
 MapState<Long, Long> windowPvMapState;
 @Override
 public void open(Configuration parameters) throws Exception {
 windowPvMapState = getRuntimeContext().getMapState(new
MapStateDescriptor<Long, Long>("window-pv", Long.class, Long.class));
 }
 @Override
 public void processElement(Event value, Context ctx, Collector<String> out)
throws Exception {
 // 每来一条数据，就根据时间戳判断属于哪个窗口
 Long windowStart = value.timestamp / windowSize * windowSize;
 Long windowEnd = windowStart + windowSize;
 // 注册 end -1 的定时器，窗口触发计算
 ctx.timerService().registerEventTimeTimer(windowEnd - 1);
 // 更新状态中的 pv 值
 if (windowPvMapState.contains(windowStart)){
 Long pv = windowPvMapState.get(windowStart);
 windowPvMapState.put(windowStart, pv + 1);
 } else {
 windowPvMapState.put(windowStart, 1L);
 }
 }
 // 定时器触发，直接输出统计的 pv 结果
 @Override
 public void onTimer(long timestamp, OnTimerContext ctx, Collector<String>
out) throws Exception {
 Long windowEnd = timestamp + 1;
 Long windowStart = windowEnd - windowSize;
 Long pv = windowPvMapState.get(windowStart);
 out.collect( "url: " + ctx.getCurrentKey()
 + " 访问量: " + pv
 + " 窗 口 ： " + new Timestamp(windowStart) + " ~ " + new
Timestamp(windowEnd));
 // 模拟窗口的销毁，清除 map 中的 key
 windowPvMapState.remove(windowStart);
 }
 } }
```

#### 3 聚合状态

我们举一个简单的例子，对用户点击事件流每 5 个数据统计一次平均时间戳。这是一个类似计数窗口（CountWindow）求平均值的计算，这里我们可以使用一个有聚合状态的RichFlatMapFunction 来实现

```java
// 统计每个用户的点击频次，到达 5 次就输出统计结果
 stream.keyBy(data -> data.user)
 .flatMap(new AvgTsResult())
 .print();
 env.execute();
 }
 public static class AvgTsResult extends RichFlatMapFunction<Event, String>{
 // 定义聚合状态，用来计算平均时间戳
 AggregatingState<Event, Long> avgTsAggState;
 // 定义一个值状态，用来保存当前用户访问频次
 ValueState<Long> countState;
 @Override
 public void open(Configuration parameters) throws Exception {
 avgTsAggState = getRuntimeContext().getAggregatingState(new
AggregatingStateDescriptor<Event, Tuple2<Long, Long>, Long>(
 "avg-ts",
 new AggregateFunction<Event, Tuple2<Long, Long>, Long>() {
 @Override
 public Tuple2<Long, Long> createAccumulator() {
 return Tuple2.of(0L, 0L);
 }
 @Override
 public Tuple2<Long, Long> add(Event value, Tuple2<Long, Long>
accumulator) {
 return Tuple2.of(accumulator.f0 + value.timestamp,
accumulator.f1 + 1);
 }
 @Override
 public Long getResult(Tuple2<Long, Long> accumulator) {
 return accumulator.f0 / accumulator.f1;
 }
 @Override
 public Tuple2<Long, Long> merge(Tuple2<Long, Long> a,
Tuple2<Long, Long> b) {
 return null;
 }
 },
 Types.TUPLE(Types.LONG, Types.LONG)
 ));
 countState = getRuntimeContext().getState(new
ValueStateDescriptor<Long>("count", Long.class));
 }
 @Override
 public void flatMap(Event value, Collector<String> out) throws Exception
{
 Long count = countState.value();
 if (count == null){
 count = 1L;
 } else {
 count ++;
 }
 countState.update(count);
 avgTsAggState.add(value);
 // 达到 5 次就输出结果，并清空状态
 if (count == 5){
 out.collect(value.user + " 平均时间戳： " + new
Timestamp(avgTsAggState.get()));
 countState.clear();
 }
 }
 }
}

```



### 4.2 状态生存时间（TTL）

在实际应用中，很多状态会随着时间的推移逐渐增长，如果不加以限制，最终就会导致存储空间的耗尽。一个优化的思路是直接在代码中调用.clear()方法去清除状态，但是有时候我们的逻辑要求不能直接清除。这时就需要配置一个状态的“生存时间”（time-to-live，TTL），当状态在内存中存在的时间超出这个值时，就将它清除。具体实现上，如果用一个进程不停地扫描所有状态看是否过期，显然会占用大量资源做无用功。状态的失效其实不需要立即删除，所以我们可以给状态附加一个属性，也就是状态的“失效时间”。状态创建的时候，设置 失效时间 = 当前时间 + TTL；之后如果有对状态的访问和修改，我们可以再对失效时间进行更新；当设置的清除条件被触发时（比如，状态被访问的时候，或者每隔一段时间扫描一次失效状态），就可以判断状态是否失效、从而进行清除了。配置状态的 TTL 时，需要创建一个 StateTtlConfig 配置对象，然后调用状态描述器的.enableTimeToLive()方法启动 TTL 功能。

```java
StateTtlConfig ttlConfig = StateTtlConfig .newBuilder(Time.seconds(10)) .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite) .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired) .build();ValueStateDescriptor stateDescriptor = new ValueStateDescriptor<>("mystate", String.class);260stateDescriptor.enableTimeToLive(ttlConfig)
```

;这里用到了几个配置项：

⚫ .newBuilder()状态 TTL 配置的构造器方法，必须调用，返回一个 Builder 之后再调用.build()方法就可以得到 StateTtlConfig 了。方法需要传入一个 Time 作为参数，这就是设定的状态生存时间。

⚫ .setUpdateType()设置更新类型。更新类型指定了什么时候更新状态失效时间，这里的 OnCreateAndWrite表示只有创建状态和更改状态（写操作）时更新失效时间。另一种类型 OnReadAndWrite 则表示无论读写操作都会更新失效时间，也就是只要对状态进行了访问，就表明它是活跃的，从而延长生存时间。这个配置默认为 OnCreateAndWrite。

⚫ .setStateVisibility()设置状态的可见性。所谓的“状态可见性”，是指因为清除操作并不是实时的，所以当状态过期之后还有可能基于存在，这时如果对它进行访问，能否正常读取到就是一个问题了。这里设置的 NeverReturnExpired 是默认行为，表示从不返回过期值，也就是只要过期就认为它已经被清除了，应用不能继续读取；这在处理会话或者隐私数据时比较重要。对应的另一种配置是 ReturnExpireDefNotCleanedUp，就是如果过期状态还存在，就返回它的值。除此之外，TTL 配置还可以设置在保存检查点（checkpoint）时触发清除操作，或者配置增量的清理（incremental cleanup），还可以针对 RocksDB 状态后端使用压缩过滤器（compactionfilter）进行后台清理。关于检查点和状态后端的内容，我们会在后续章节继续讲解。这里需要注意，目前的 TTL 设置只支持处理时间。另外，所有集合类型的状态（例如ListState、MapState）在设置 TTL 时，都是针对每一项（per-entry）元素的。也就是说，一个列表状态中的每一个元素，都会以自己的失效时间来进行清理，而不是整个列表一起清理。

### 4.3 状态编程例子

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

## 五 状态后端

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



