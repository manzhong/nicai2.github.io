---
title: Flink的DataSource
tags: Flink
categories: Flink
abbrlink: 53370
date: 2020-10-13 20:52:51
summary_img:
encrypt:
enc_pwd:
---

# Flink的Data Source

## 一 概述

​		Flink 做为一款流式计算框架，它可用来做批处理，即处理静态的数据集、历史的数据集；也可以用来做流处理，即实时的处理些实时数据流，实时的产生数据流结果，只要数据源源不断的过来，Flink 就能够一直计算下去，这个 Data Sources 就是数据的来源地。

​		Flink 中你可以使用 `StreamExecutionEnvironment.addSource(sourceFunction)` 来为你的程序添加数据来源。

​		Flink 已经提供了若干实现好了的 source functions，当然你也可以通过实现 SourceFunction 来自定义非并行的 source 或者实现 ParallelSourceFunction 接口或者扩展 RichParallelSourceFunction 来自定义并行的 source，

## 二 数据来源分类

​	Flink 已经提供了若干实现好了的 source functions，当然你也可以通过实现 SourceFunction 来自定义非并行的 source 或者实现 ParallelSourceFunction 接口或者扩展 RichParallelSourceFunction 来自定义并行的 source

StreamExecutionEnvironment 中可以使用以下几个已实现的 stream sources:

```scala
def fromElements[T: TypeInformation](data: T*): DataStream[T] = {....}
def fromCollection[T: TypeInformation](data: Seq[T]): DataStream[T] = {....}
def fromCollection[T: TypeInformation] (data: Iterator[T]): DataStream[T] = {... }
def fromParallelCollection[T: TypeInformation] (data: SplittableIterator[T]):  DataStream[T] = {.....}
def readTextFile(filePath: String): DataStream[String] = asScalaStream(javaEnv.readTextFile(filePath))
def readTextFile(filePath: String, charsetName: String): DataStream[String] = asScalaStream(javaEnv.readTextFile(filePath, charsetName))
def readFile[T: TypeInformation](inputFormat: FileInputFormat[T], filePath: String): DataStream[T] = asScalaStream(javaEnv.readFile(inputFormat, filePath))
def readFileStream(StreamPath: String, intervalMillis: Long = 100,
                     watchType: FileMonitoringFunction.WatchType =
                     FileMonitoringFunction.WatchType.ONLY_NEW_FILES): DataStream[String] =
    asScalaStream(javaEnv.readFileStream(StreamPath, intervalMillis, watchType))
  @PublicEvolving
  @Deprecated
def readFile[T: TypeInformation](
                                    inputFormat: FileInputFormat[T],
                                    filePath: String,
                                    watchType: FileProcessingMode,
                                    interval: Long,
                                    filter: FilePathFilter): DataStream[T] = {
    asScalaStream(javaEnv.readFile(inputFormat, filePath, watchType, interval, filter))
}
def readFile[T: TypeInformation](
    inputFormat: FileInputFormat[T],
    filePath: String,
    watchType: FileProcessingMode,
    interval: Long): DataStream[T] = {
  val typeInfo = implicitly[TypeInformation[T]]
  asScalaStream(javaEnv.readFile(inputFormat, filePath, watchType, interval, typeInfo))
}
def socketTextStream(hostname: String, port: Int, delimiter: Char = '\n', maxRetry: Long = 0):
      DataStream[String] =
  asScalaStream(javaEnv.socketTextStream(hostname, port))
def createInput[T: TypeInformation](inputFormat: InputFormat[T, _]): DataStream[T] =
  if (inputFormat.isInstanceOf[ResultTypeQueryable[_]]) {
    asScalaStream(javaEnv.createInput(inputFormat))
  } else {
    asScalaStream(javaEnv.createInput(inputFormat, implicitly[TypeInformation[T]]))
  }
def addSource[T: TypeInformation](function: SourceFunction[T]): DataStream[T] = {...}
def addSource[T: TypeInformation](function: SourceContext[T] => Unit): DataStream[T] = {...}
```

总体来说分为以下几类:

### 1 数据源为集合

1、fromCollection(Collection) - 从 Java 的 Java.util.Collection 创建数据流。集合中的所有元素类型必须相同。

2、fromCollection(Iterator, Class) - 从一个迭代器中创建数据流。Class 指定了该迭代器返回元素的类型。

3、fromElements(T …) - 从给定的对象序列中创建数据流。所有对象类型必须相同。

4、fromParallelCollection(SplittableIterator, Class) - 从一个迭代器中创建并行数据流。Class 指定了该迭代器返回元素的类型。

5、generateSequence(from, to) - 创建一个生成指定区间范围内的数字序列的并行数据流。

```scala
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<Event> input = env.fromElements(
	new Event(1, "barfoo", 1.0),
	new Event(2, "start", 2.0),
	new Event(3, "foobar", 3.0),
	...
);
```

### 2 基于文件

1、readTextFile(path) - 读取文本文件，即符合 TextInputFormat 规范的文件，并将其作为字符串返回。

2、readFile(fileInputFormat, path) - 根据指定的文件输入格式读取文件（一次）。

3、readFile(fileInputFormat, path, watchType, interval, pathFilter, typeInfo) - 这是上面两个方法内部调用的方法。它根据给定的 fileInputFormat 和读取路径读取文件。根据提供的 watchType，这个 source 可以定期（每隔 interval 毫秒）监测给定路径的新数据（FileProcessingMode.PROCESS_CONTINUOUSLY），或者处理一次路径对应文件的数据并退出（FileProcessingMode.PROCESS_ONCE）。你可以通过 pathFilter 进一步排除掉需要处理的文件。

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<MyEvent> stream = env.readFile(
        myFormat, myFilePath, FileProcessingMode.PROCESS_CONTINUOUSLY, 100,
        FilePathFilter.createDefaultFilter(), typeInfo);
```

 实现:

在具体实现上，Flink 把文件读取过程分为两个子任务，即目录监控和数据读取。每个子任务都由单独的实体实现。目录监控由单个非并行（并行度为1）的任务执行，而数据读取由并行运行的多个任务执行。后者的并行性等于作业的并行性。单个目录监控任务的作用是扫描目录（根据 watchType 定期扫描或仅扫描一次），查找要处理的文件并把文件分割成切分片（splits），然后将这些切分片分配给下游 reader。reader 负责读取数据。每个切分片只能由一个 reader 读取，但一个 reader 可以逐个读取多个切分片。

重要注意：

如果 watchType 设置为 FileProcessingMode.PROCESS_CONTINUOUSLY，则当文件被修改时，其内容将被重新处理。这会打破“exactly-once”语义，因为在文件末尾附加数据将导致其所有内容被重新处理。

如果 watchType 设置为 FileProcessingMode.PROCESS_ONCE，则 source 仅扫描路径一次然后退出，而不等待 reader 完成文件内容的读取。当然 reader 会继续阅读，直到读取所有的文件内容。关闭 source 后就不会再有检查点。这可能导致节点故障后的恢复速度较慢，因为该作业将从最后一个检查点恢复读取。

### 3 基于Socket

socketTextStream(String hostname, int port) - 从 socket 读取。元素可以用分隔符切分。

### 4 自定义Source

addSource - 添加一个新的 source function。例如，你可以 addSource(new FlinkKafkaConsumer09<>(…)) 以从 Apache Kafka 读取数据

```scala
  def addSource[T: TypeInformation](function: SourceFunction[T]): DataStream[T] = {
    require(function != null, "Function must not be null.")
    
    val cleanFun = scalaClean(function)
    val typeInfo = implicitly[TypeInformation[T]]
    asScalaStream(javaEnv.addSource(cleanFun, typeInfo))
  }

  def addSource[T: TypeInformation](function: SourceContext[T] => Unit): DataStream[T] = {
    require(function != null, "Function must not be null.")
    val sourceFunction = new SourceFunction[T] {
      val cleanFun = scalaClean(function)
      override def run(ctx: SourceContext[T]) {
        cleanFun(ctx)
      }
      override def cancel() = {}
    }
    addSource(sourceFunction)
  }
```

​     比如去消费 Kafka 某个 topic 上的数据，这时候就需要用到这个 addSource，可能因为用的比较多的原因吧，Flink 直接提供了 FlinkKafkaConsumer09 等类可供你直接使用。你可以去看看 FlinkKafkaConsumerBase 这个基础类，它是 Flink Kafka 消费的最根本的类

#### 4.1 Flink的kafka的source

**首先启动zk,kafka,Flink**

topic的实体类:

```scala
import scala.collection.mutable.Map
case class Metric(var name: String,var timestamp: Long,var fields:Map[String,String],var tags: Map[String,String]){
  def this(){
    this("aa",2,Map(),Map())
  }
}
```

kafka发消息的工具类

```scala
package mz.kafkasouirce

import java.util.Properties
import scala.collection.mutable.Map
import com.alibaba.fastjson.JSON
import org.apache.kafka.clients.producer.KafkaProducer
import org.apache.kafka.clients.producer.ProducerRecord
//kafka发数据的工具
object KafkaUtils {
  val broker_list = "localhost:9092"
  val topic = "metric" // kafka topic，Flink 程序中需要和这个统一

  @throws[InterruptedException]
  def writeToKafka(): Unit = {
    val props = new Properties
    props.put("bootstrap.servers", broker_list)
    props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer") //key 序列化

    props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer") //value 序列化

    val producer = new KafkaProducer[String, String](props)
    val metric = new Metric()
    metric.timestamp=System.currentTimeMillis
    metric.name=("mz")
    var tags:Map[String,String] = Map()
    var fields:Map[String,String] = Map()
    tags += ("cluster" -> "mz")
    tags += ("host_ip" -> "127.0.0.1")
    fields += ("used_percent" -> "a")
    fields += ("max" -> "b")
    fields += ("used" -> "c")
    fields += ("init" -> "d")
    metric.tags=tags
    metric.fields=fields
    val record: ProducerRecord[String, String] = new ProducerRecord[String, String](topic, null, null, metric.toString)
    producer.send(record)
    System.out.println("发送数据: " + metric.toString)
    producer.flush()
  }

  @throws[InterruptedException]
  def main(args: Array[String]): Unit = {
    while ( {
      true
    }) {
      Thread.sleep(300)
      writeToKafka()
    }
  }

}
```

flink程序:

```scala
import java.util.Properties

import org.apache.calcite.avatica.Handler.ResultSink
import org.apache.flink.api.common.typeinfo.{TypeHint, TypeInformation}
import org.apache.flink.api.java.tuple.Tuple2
import org.apache.flink.streaming.api.functions.source.SourceFunction
import org.apache.flink.streaming.api.{CheckpointingMode, TimeCharacteristic}
import org.apache.flink.streaming.api.scala.StreamExecutionEnvironment
import org.apache.flink.streaming.connectors.kafka.{FlinkKafkaConsumer09, KafkaDeserializationSchema}
import org.apache.flink.streaming.util.serialization.{SimpleStringSchema, TypeInformationKeyValueSerializationSchema}
import org.apache.flink.streaming.api.scala._
import org.apache.kafka.clients.consumer.ConsumerRecord
object FlinkKafkaSource {
    //自定义KafkaDeserializationSchema  反序列化类
  class RecordKafkaSchema extends KafkaDeserializationSchema[ConsumerRecord[String, String]] {
    /*是否流结束，比如读到一个key为end的字符串结束，这里不再判断，直接返回false 不结束*/
    override def isEndOfStream(nextElement: ConsumerRecord[String, String]): Boolean = false

    override def deserialize(record: ConsumerRecord[Array[Byte], Array[Byte]]): ConsumerRecord[String, String] = {
      var key: String = null
      var value: String = null
      if (record.key != null) {
        key = new String(record.key(),"UTF-8")
      }
      if (record.value != null) {
        value = new String(record.value(),"UTF-8")
      }
      new ConsumerRecord[String, String](
        record.topic(),
        record.partition(),
        record.offset(),
        //record.timestamp(),
        //record.timestampType(),
        //record.checksum,
        //record.serializedKeySize,
        //record.serializedValueSize(),
        key,
        value
        )
    }

    override def getProducedType: TypeInformation[ConsumerRecord[String, String]] = TypeInformation.of(new TypeHint[ConsumerRecord[String, String]] {})
  }
  
  def main(args: Array[String]): Unit = {
    val streamEnv: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
   // streamEnv.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
    //streamEnv.enableCheckpointing(1000)
   // streamEnv.getCheckpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE)
    val kafkaProps = new Properties()
    kafkaProps.put("bootstrap.servers", "localhost:9092")
    kafkaProps.put("zookeeper.connect", "localhost:2181")
    kafkaProps.put("group.id", "metric-group")
    kafkaProps.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer") //key 反序列化
    kafkaProps.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer")//value 反序列化
    kafkaProps.put("auto.offset.reset", "latest") 

       // 1 返回的结果只有Kafka的key,value，而没有其它信息tring
    //val schema1: TypeInformationKeyValueSerializationSchema[String, String] = new TypeInformationKeyValueSerializationSchema(classOf[String], classOf[String], streamEnv.getConfig)
    //val ds1: DataStream[Tuple2[String, String]] = streamEnv.addSource(new FlinkKafkaConsumer09[Tuple2[String, String]]("metric",schema1, kafkaProps))

    //2 返回的结果只有Kafka的value，而没有其它信息：
    //val schema2 =new SimpleStringSchema()
   //val ds: DataStream[String] = streamEnv.addSource(new FlinkKafkaConsumer09[String]("metric",schema2, kafkaProps))


    //3  很多时候我们需要获得Kafka的topic或者其它信息，就需要通过实现KafkaDeserializationSchema接口来自定义返回数据的结构：
    var schema3 = new RecordKafkaSchema
    val kafkaSource: FlinkKafkaConsumer09[ConsumerRecord[String, String]] = new FlinkKafkaConsumer09[ConsumerRecord[String, String]]("metric",schema3, kafkaProps)
    /*指定消费位点*/
    val specificStartOffsets = new java.util.HashMap[KafkaTopicPartition, java.lang.Long]()
    /*这里从metric 的0分区的第一条开始消费*/
    specificStartOffsets.put(new KafkaTopicPartition("metric", 0), 0L)
    kafkaSource.setStartFromSpecificOffsets(specificStartOffsets)

    val ds: DataStream[ConsumerRecord[String, String]] = streamEnv.addSource(kafkaSource)
    //遍历
    val keyValue=ds.map(new MapFunction[ConsumerRecord[String, String],String] {
      override def map(message: ConsumerRecord[String, String]): String = {
        "key" + message.key + "  value:" + message.value
      }
    })
    /*打印接收的数据*/
    keyValue.print()
    /*启动执行*/
    streamEnv.execute()
       
  }
}
```

若是想自定义Source,就要去看SourceFunction 接口了，它是所有 stream source 的根接口，它继承自一个标记接口（空接口）Function。

SourceFunction 定义了两个接口方法：

```txt
1、run ： 启动一个 source，即对接一个外部数据源然后 emit 元素形成 stream（大部分情况下会通过在该方法里运行一	 个 while 循环的形式来产生 stream）。
2、cancel ： 取消一个 source，也即将 run 中的循环 emit 元素的行为终止。
正常情况下，一个 SourceFunction 实现这两个接口方法就可以了。其实这两个接口方法也固定了一种实现模板。
```

#### 4.2 Mysql的自定义Source

添加mysql的依赖

```xml
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
	<version>5.1.34</version>
</dependency>
```

建表:

```sql
DROP TABLE IF EXISTS `user`;
CREATE TABLE `user` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(25) COLLATE utf8_bin DEFAULT NULL,
  `age` int(10) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=5 DEFAULT CHARSET=utf8 COLLATE=utf8_bin;
```

插入数据:

```sql
INSERT INTO `user` VALUES ('1', 'a', '18'), ('2', 'b', '17'), ('3', 'c', '18'), ('4', 'd', '16');
```

建实体类user:

```scala
case class User(var id:Int,var name:String,var age:Int)
```

新建 Source SourceFromMySQL.java，该类继承 RichSourceFunction ，实现里面的 open、close、run、cancel 方法：

```scala
package mz.mysqlsource

import java.lang.Exception
import java.sql

import org.apache.flink.streaming.api.functions.source.{RichSourceFunction, SourceFunction}
import java.sql.{DriverManager, PreparedStatement, ResultSet}

import org.apache.flink.configuration.Configuration
import org.apache.flink.streaming.api.functions.source.SourceFunction.SourceContext
import org.apache.flink.streaming.api.scala._


case class FlinkMysqlSource() extends  RichSourceFunction[User](){

  var connection:sql.Connection=_
  var ps:PreparedStatement=_
  /**
   * open() 方法中建立连接，这样不用每次 invoke 的时候都要建立连接和释放连接。
   *
   * @param parameters
   * @throws Exception
   */
  @throws[Exception]
  override def open(parameters: Configuration): Unit = {
    super.open(parameters)
    connection = getConnection
    val sql = "select * from user;"
    ps = connection.prepareStatement(sql)

  }

  /**
   * 程序执行完毕就可以进行，关闭连接和释放资源的动作了
   *
   * @throws Exception
   */
  @throws[Exception]
  override def close(): Unit = {
    super.close
    if (connection != null) { //关闭连接和释放资源
      connection.close
    }
    if (ps != null) ps.close
  }

  /**
   * DataStream 调用一次 run() 方法用来获取数据
   *
   * @param ctx
   * @throws Exception
   */
  @throws[Exception]
  override def run(ctx:SourceContext[User]): Unit = {
    val resultSet = ps.executeQuery
    while ( {resultSet.next}) {
      val user: User = new User(resultSet.getInt("id"), resultSet.getString("name").trim, resultSet.getInt("age"))
      ctx.collect(user)
    }
  }

  override def cancel(): Unit = {
  }

  private def getConnection:sql.Connection = {
    try {
      Class.forName("com.mysql.jdbc.Driver")
      connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/lcflink?useUnicode=true&characterEncoding=UTF-8", "root", "123456")
    }catch {
      case e:Exception => {
        print(e.getMessage)
      }
    }
    connection

  }
}
```

Flink的代码:

```scala
package mz.mysqlsource

import org.apache.flink.streaming.api.scala.StreamExecutionEnvironment
import org.apache.flink.streaming.api.scala._
object FlinkMysqlTest {
  def main(args: Array[String]): Unit = {
    val environment = StreamExecutionEnvironment.getExecutionEnvironment
    environment.addSource(new FlinkMysqlSource()).print()
    environment.execute("mysql_query")
  }
}
```

RichSourceFunction的上层继承体系:

```java
1 public abstract class RichSourceFunction<OUT> extends AbstractRichFunction implements SourceFunction<OUT> {}
 2.1 public abstract class AbstractRichFunction implements RichFunction, Serializable {}
	 2.1.1public interface RichFunction extends Function {}
 2.2 public interface SourceFunction<T> extends Function, Serializable {}
```

一个抽象类，继承自 AbstractRichFunction。为实现一个 Rich SourceFunction 提供基础能力。该类的子类有三个，两个是抽象类，在此基础上提供了更具体的实现，另一个是 ContinuousFileMonitoringFunction。

RichSourceFunction的实现:

```java
1 public class ContinuousFileMonitoringFunction<OUT>
	extends RichSourceFunction<TimestampedFileInputSplit> implements CheckpointedFunction {}
	
2 public abstract class MessageAcknowledgingSourceBase<Type, UId>
	extends RichSourceFunction<Type>
	implements CheckpointedFunction, CheckpointListener {}
 2.1 public abstract class MultipleIdsMessageAcknowledgingSourceBase<Type, UId, SessionId>
		extends MessageAcknowledgingSourceBase<Type, UId> {}
```

- MessageAcknowledgingSourceBase ：它针对的是数据源是消息队列的场景并且提供了基于 ID 的应答机制。
- MultipleIdsMessageAcknowledgingSourceBase ： 在 MessageAcknowledgingSourceBase 的基础上针对 ID 应答机制进行了更为细分的处理，支持两种 ID 应答模型：session id 和 unique message id。
- ContinuousFileMonitoringFunction：这是单个（非并行）监视任务，它接受 FileInputFormat，并且根据 FileProcessingMode 和 FilePathFilter，它负责监视用户提供的路径；决定应该进一步读取和处理哪些文件；创建与这些文件对应的 FileInputSplit 拆分，将它们分配给下游任务以进行进一步处理。

### 5 说下上面几种的特点

1、基于集合：有界数据集，更偏向于本地测试用

2、基于文件：适合监听文件修改并读取其内容

3、基于 Socket：监听主机的 host port，从 Socket 中获取数据

4、自定义 addSource：大多数的场景数据都是无界的，会源源不断的过来。或者其他复杂源数据





