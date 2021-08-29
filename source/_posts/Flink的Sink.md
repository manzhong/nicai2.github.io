---
title: Flink的Sink
tags: Flink
categories: Flink
abbrlink: 13166
date: 2020-10-14 22:38:31
summary_img:
encrypt:
enc_pwd:
---

## 一 flink的Sink

​	flink中没有类似于spark中的foreach方法,让用户进行迭代操作,虽然对外的操作都需要sink完成,flink一般通过一下方法

```scala
stream.addSink(new mySink('xxx'))
```

官方提供了一部分sink,其他的需要自己自定义实现sink

官方提供的api:

```
kafka,es,hdfs,rabbitMq,
```

第三方的包实现Apache Bahir:

```
flume,redis,akka,netty(source)
```

## 二 实现

### 1 写入文件和打印

```scala
//支持 
 val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    //val ds: DataStream[String] = env.socketTextStream("localhost",7777)
    val ds: DataStream[String] = env.fromElements("123","456")
    ds.print()
    ds.writeAsCsv("a.scv")
    //ds.writeToSocket("localhost",7777)
    ds.writeAsText("b.txt")
    //自定义文件输出
    ds.addSink(StreamingFileSink.forRowFormat(
      new Path("a")
      //序列化
      ,new SimpleStringEncoder[String]()
    ).build())
		//自定义打印
    ds.addSink(new PrintSinkFunction[String]())
    env.execute()
```

### 2 sinkTokafka

```scala
 val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    val ds: DataStream[String] = env.fromElements("123","456")
    ds.addSink(new FlinkKafkaProducer09[String]("localhost:9999","sinkTest",new SimpleStringSchema()))
```

### 3 sinkToRedis

引入依赖bahir

```xml
<dependency>
            <groupId>org.apache.bahir</groupId> 
            <artifactId>flink-connector-redis_2.11</artifactId>
            <version>1.0</version>
</dependency>
```

```scala
import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.connectors.redis.RedisSink
import org.apache.flink.streaming.connectors.redis.common.config.FlinkJedisPoolConfig
import org.apache.flink.streaming.connectors.redis.common.mapper.{RedisCommand, RedisCommandDescription, RedisMapper}
object SinkRedis {
  def main(args: Array[String]): Unit = {
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    val ds: DataStream[String] = env.fromElements("123","456")
    //第一个参数Redis的连接 第二个参数 定义写入的参数和写入的命令
    val conf = new FlinkJedisPoolConfig.Builder()
        .setHost("loaclhost")
        .setPort(6379)
        .build()

    ds.addSink( new RedisSink[String](conf,new MyRedisMapper))
    env.execute("redis")
  }

}
//第二个参数
class  MyRedisMapper extends RedisMapper[String] {
  //定义保存redis的命令 hset 表名 k和v
  override def getCommandDescription: RedisCommandDescription =new RedisCommandDescription(RedisCommand.HSET,"num")
  //指定k 值为k
  override def getKeyFromData(t: String): String = t.concat("_k")
  //指定v 值为v
  override def getValueFromData(t: String): String = t
}
```

### 4 sinkEs

引入依赖

```xml
<!--2.12 scala版本 -->
<dependency>
            <groupId>org.apache.flink</groupId> 
            <artifactId>flink-connector-elasticsearch6_2.12</artifactId>
            <version>1.10.1</version> <!--flink的版本-->
</dependency>
```

```scala
import java.util
import org.apache.flink.api.common.functions.RuntimeContext
import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.connectors.elasticsearch.{ElasticsearchSinkFunction, RequestIndexer}
import org.apache.flink.streaming.connectors.elasticsearch6.ElasticsearchSink
import org.apache.http.HttpHost
import org.elasticsearch.client.Requests

object SinkEs {
  def main(args: Array[String]): Unit = {
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    val ds: DataStream[String] = env.fromElements("123","456")
    //第一个参数:地址
    val httpHosts = new util.ArrayList[HttpHost]()
    httpHosts.add(new HttpHost("localhost",9200))
    //自定义写入es的Function 匿名类
    val myEsSinkFunction =new ElasticsearchSinkFunction[String] {
      override def process(t: String, runtimeContext: RuntimeContext, requestIndexer: RequestIndexer): Unit = {
        //包装一个map作为一个dataSource
        val dataSource = new util.HashMap[String,String]()
        dataSource.put("id",t.concat("_id"))
        dataSource.put("num",t)
        //创建index  request ,用于发送http请求
        val indexRe= Requests.indexRequest()
          .index("num")
          .`type`("reading")
          .source(dataSource)
        //用index发送请求
         requestIndexer.add(indexRe)
      }
    }
    ds.addSink(new ElasticsearchSink.Builder[String](httpHosts,myEsSinkFunction).build())
  }
}
```

### 5 sinkMysql --JDBC自定义Sink

引入依赖 MySQL:

```xml
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.38</version>
        </dependency>
```

添加MyJdbcSink

```scala
import java.sql.{Connection, Driver, DriverManager, PreparedStatement}
import org.apache.flink.configuration.Configuration
import org.apache.flink.streaming.api.functions.sink.{RichSinkFunction, SinkFunction}

//富函数
class MyJdbcSink extends  RichSinkFunction[String]{
  //定义连接 预编译语句
  var con:Connection =_
  var insertStmt:PreparedStatement = _
  var updateStmt:PreparedStatement = _
  override def open(parameters: Configuration): Unit = {
    val con = DriverManager.getConnection("jdbc:mysql://localhost:3306/test","username","password")
    insertStmt = con.prepareStatement("insert into flinktest (id,temp) values(?,?)")
    updateStmt = con.prepareStatement("update  flinktest set temp= ? where id=?")
  }

  override def invoke(value: String): Unit = {
    //更新
    updateStmt.setString(1,value.concat("_temp"))
    updateStmt.setString(2,value.concat("_id"))
    updateStmt.execute()
    //更新无数据,就插入
    if( updateStmt.getUpdateCount ==0){
       insertStmt.setString(1,value.concat("_id"))
      insertStmt.setString(2,value.concat("_temp"))
      insertStmt.execute()
    }

  }

  override def close(): Unit = {
    insertStmt.close()
    updateStmt.close()
    con.close()
  }
}
```

```scala
 val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    val ds: DataStream[String] = env.fromElements("123","456")
    ds.addSink( new MyJdbcSink())
env.execute("sinkmysql")
```



