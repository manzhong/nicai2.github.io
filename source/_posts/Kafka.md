---
title: Kafka
abbrlink: 13749
date: 2017-07-16 18:30:57
tags: Kafka
categories: Kafka
summary_img:
encrypt:
enc_pwd:
---

# Kafka消息队列

## 一消息队列概述

### 1 kafka企业级消息系统kafka企业级消息系统

**为何使用消息系统**

在没有使用消息系统以前，我们对于传统许多业务，以及跨服务器传递消息的时候，会采用串行方式或者并行方法；

与串行的差别是并行的方式可以缩短程序整体处理的时间。

**消息系统:**

消息系统负责将数据从一个应用程序传送到另一个应用程序，因此应用程序可以专注于数据，但是不必担心 如何共享它。分布式消息系统基于可靠的消息队列的概念。消息在客户端应用程序和消息传递系统之间的异步排队。

有两种类型的消息模式可用    点对点；发布-订阅消息系统

点对点消息系统中，消息被保留在队列中，一个或者多个消费者可以消费队列中的消息，但是特定的消 息只能有最多的一个消费者消费。一旦消费者读取队列中的消息，他就从该队列中消失。该系统的典型应用就是订单处理系统，其中每个订单将有一个订单处理器处理，但多个订单处理器可以同时工作。

大多数的消息系统是基于发布-订阅消息系统

**分类**

## 2.1、点对点

主要采用的队列的方式，如A->B 当B消费的队列中的数据，那么队列的数据就会被删除掉【如果B不消费那么就会存在队列中有很多的脏数据】

## 2.2、发布-订阅

发布与订阅主要三大组件

主题：一个消息的分类 

发布者：将消息通过主动推送的方式推送给消息系统；

订阅者：可以采用拉、推的方式从消息系统中获取数据

**应用场景**

## 3.1、应用解耦

将一个大型的任务系统分成若干个小模块，将所有的消息进行统一的管理和存储，因此为了解耦，就会涉及到kafka企业级消息平台

## 3.2、流量控制

秒杀活动当中，一般会因为流量过大，应用服务器挂掉，为了解决这个问题，一般需要在应用前端加上消息队列以控制访问流量。

1、 可以控制活动的人数 可以缓解短时间内流量大使得服务器崩掉

2、 可以通过队列进行数据缓存，后续再进行消费处理

## 3.3、日志处理

日志处理指将消息队列用在日志处理中，比如kafka的应用中，解决大量的日志传输问题；

日志采集工具采集 数据写入kafka中；kafka消息队列负责日志数据的接收，存储，转发功能；

日志处理应用程序：订阅并消费 kafka队列中的数据，进行数据分析。

## 3.4、消息通讯

消息队列一般都内置了高效的通信机制，因此也可以用在纯的消息通讯，比如点对点的消息队列，或者聊天室等。

# 二 kafka概述

kafka是最初由linkedin公司开发的，使用scala语言编写，kafka是一个分布式，分区的，多副本的，多订阅者的日志系统（分布式MQ系统），可以用于搜索日志，监控日志，访问日志等。

kafka目前支持多种客户端的语言：java、python、c++、php等

apache kafka是一个分布式发布-订阅消息系统和一个强大的队列，可以处理大量的数据，并使能够将消息从一个端点传递到另一个端点，kafka适合离线和在线消息消费。kafka消息保留在磁盘上，并在集群内复制以防止数据丢失。kafka构建在zookeeper同步服务之上。它与apache和spark非常好的集成，应用于实时流式数据分析。

其他消息队列:

RabbitMQ 

Redis 

ZeroMQ 

ActiveMQ

**kafka好处:**

可靠性：分布式的，分区，复制和容错的。

可扩展性：kafka消息传递系统轻松缩放，无需停机。

耐用性：kafka使用分布式提交日志，这意味着消息会尽可能快速的保存在磁盘上，因此它是持久的。 

性能：kafka对于发布和定于消息都具有高吞吐量。即使存储了许多TB的消息，他也爆出稳定的性能。 

kafka非常快：保证零停机和零数据丢失。

**应用场景**

## 5.1、指标分析

kafka   通常用于操作监控数据。这设计聚合来自分布式应用程序的统计信息，   以产生操作的数据集中反馈

## 5.2、日志聚合解决方法 

kafka可用于跨组织从多个服务器收集日志，并使他们以标准的合适提供给多个服务器。

## 5.3、流式处理

流式处理框架（spark，storm，ﬂink）重主题中读取数据，对齐进行处理，并将处理后的数据写入新的主题，供 用户和应用程序使用，kafka的强耐久性在流处理的上下文中也非常的有用。

# 三架构

![img](/images/kafka/jg.jpg)

四大核心:

##### 生产者API

允许应用程序发布记录流至一个或者多个kafka的主题（topics）。

##### 消费者API

允许应用程序订阅一个或者多个主题，并处理这些主题接收到的记录流。

##### StreamsAPI

允许应用程序充当流处理器（stream processor），从一个或者多个主题获取输入流，并生产一个输出流到一个或 者多个主题，能够有效的变化输入流为输出流。

##### ConnectorAPI

允许构建和运行可重用的生产者或者消费者，能够把kafka主题连接到现有的应用程序或数据系统。例如：一个连 接到关系数据库的连接器可能会获取每个表的变化。

**架构关系图**

![img](/images/kafka/jgg.jpg)

说明：kafka支持消息持久化，消费端为拉模型来拉取数据，消费状态和订阅关系有客户端负责维护，消息消费完 后，不会立即删除，会保留历史消息。因此支持多订阅时，消息只会存储一份就可以了

**整体架构**

![img](/images/kafka/ztjg.jpg)

一个典型的kafka集群中包含若干个Producer，若干个Broker，若干个Consumer，以及一个zookeeper集群； kafka通过zookeeper管理集群配置，选举leader，以及在Consumer Group发生变化时进行Rebalance（负载均 衡）；Producer使用push模式将消息发布到Broker；Consumer使用pull模式从Broker中订阅并消费消息。

**kafka术语介绍**

**Broker**：kafka集群中包含一个或者多个服务实例，这种服务实例被称为Broker

**Topic**：每条发布到kafka集群的消息都有一个类别，这个类别就叫做Topic 

**Partition**：Partition是一个物理上的概念，每个Topic包含一个或者多个Partition 

**Producer**：负责发布消息到kafka的Broker中。

**Consumer**：消息消费者,向kafka的broker中读取消息的客户端

**Consumer Group**：每一个Consumer属于一个特定的Consumer Group（可以为每个Consumer指定 groupName）

**kafka中topic说明**

 1,kafka将消息以topic为单位进行归类

  2,topic特指kafka处理的消息源（feeds of messages）的不同分类。

  3.topic是一种分类或者发布的一些列记录的名义上的名字。kafka主题始终是支持多用户订阅的；也就是说，一 个主题可以有零个，一个或者多个消费者订阅写入的数据。

  4.在kafka集群中，可以有无数的主题。

  5.生产者和消费者消费数据一般以主题为单位。更细粒度可以到分区级别。

**kafka中分区数**

Partitions：分区数：控制topic将分片成多少个log，可以显示指定，如果不指定则会使用 broker（server.properties）中的num.partitions配置的数量。

一个broker服务下，是否可以创建多个分区？

可以的，broker数与分区数没有关系； 在kafka中，每一个分区会有一个编号：编号从0开始

某一个分区的数据是有序的

如何保证一个主题是有序的:

```
一个主题下面只有一个分区即可
```

topic的Partition数量在创建topic时配置。

Partition数量决定了每个Consumer group中并发消费者的最大数量。

Consumer group A 有两个消费者来读取4个partition中数据；Consumer group B有四个消费者来读取4个 partition中的数据

![img](/images/kafka/fq.jpg)

**kafka中的副本数**

![img](/images/kafka/fb.jpg)

kafka分区副本数（kafka Partition Replicas)

副本数（replication-factor）：控制消息保存在几个broker（服务器）上，一般情况下等于broker的个数

一个broker服务下，是否可以创建多个副本因子？

​             不可以；创建主题时，副本因子应该小于等于可用的broker数。 

副本因子操作以分区为单位的。每个分区都有各自的主副本和从副本；主副本叫做leader，从副本叫做 follower（在有多个副本的情况下，kafka会为同一个分区下的分区，设定角色关系：一个leader和N个 follower），处于同步状态的副本叫做**in-sync-replicas**(ISR);follower通过拉的方式从leader同步数据。消费 者和生产者都是从leader读写数据，不与follower交互。

**副本因子的作用**：让kafka读取数据和写入数据时的可靠性.

副本因子是包含本身|同一个副本因子不能放在同一个Broker中。

如果某一个分区有三个副本因子，就算其中一个挂掉，那么只会剩下的两个钟，选择一个leader，但不会在其 他的broker中，另启动一个副本（因为在另一台启动的话，存在数据传递，只要在机器之间有数据传递，就 会长时间占用网络IO，kafka是一个高吞吐量的消息系统，这个情况不允许发生）所以不会在零个broker中启 动。

如果所有的副本都挂了，生产者如果生产数据到指定分区的话，将写入不成功。

lsr表示：当前可用的副本

**kafka的partition offset**

任何发布到此partition的消息都会被直接追加到log文件的尾部，每条消息在文件中的位置称为oﬀset（偏移量），

oﬀset是一个long类型数字，它唯一标识了一条消息，消费者通过（oﬀset，partition，topic）跟踪记录。

**kafka分区与消费组之间的关系**

消费组： 由一个或者多个消费者组成，同一个组中的消费者对于同一条消息只消费一次。

某一个主题下的分区数，对于消费组来说，应该小于等于该主题下的分区数。如下所示：

```
如：某一个主题有4个分区，那么消费组中的消费者应该小于4，而且最好与分区数成整数倍
1	2	4
同一个分区下的数据，在同一时刻，不能同一个消费组的不同消费者消费

```

总结：分区数越多，同一时间可以有越多的消费者来进行消费，消费数据的速度就会越快，提高消费的性能

# 四 集群搭建

## 1 jdk 与zookeeper必须安装

## 2 安装用户   

如默认用户安装则跳过这一步骤

安装hadoop，会创建一个hadoop用户 

安装kafka，创建一个kafka用户

或者 创建bigdata用户，用来安装所有的大数据软件

本例：使用root用户来进行安装

## 3验证环境

保证三台机器的zk服务都正常启动，且正常运行

  查看zk的运行装填，保证有一台zk的服务状态为leader，且两台为follower即可

## 4下载安装包

```
http://archive.apache.org/dist/kafka/0.10.0.0/kafka_2.11-0.10.0.0.tgz
```

## 5上传解压

## 6修改配置文件

```
cd /export/servers/kafka_2.11-0.10.0.0/config
vim server.properties


broker.id=0
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/export/servers/kafka_2.11-0.10.0.0/logs
num.partitions=2
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.flush.interval.messages=10000
log.flush.interval.ms=1000
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=node01:2181,node02:2181,node03:2181
zookeeper.connection.timeout.ms=6000
group.initial.rebalance.delay.ms=0
delete.topic.enable=true
host.name=node01            //每台主机不一样

```

创建数据文件存放目录

```
mkdir -p  /export/servers/kafka_2.11-0.10.0.0/logs
```

分发安装包:

```
cd /export/servers/
scp -r kafka_2.11-0.10.0.0/ node02:$PWD
scp -r kafka_2.11-0.10.0.0/ node03:$PWD

```

node02修改:

```
cd /export/servers/kafka_2.11-0.10.0.0/config
vim server.properties

broker.id=1
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/export/servers/kafka_2.11-0.10.0.0/logs
num.partitions=2
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.flush.interval.messages=10000
log.flush.interval.ms=1000
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=node01:2181,node02:2181,node03:2181
zookeeper.connection.timeout.ms=6000
group.initial.rebalance.delay.ms=0
delete.topic.enable=true
host.name=node02

```

node03修改

```
cd /export/servers/kafka_2.11-0.10.0.0/config
vim server.properties

broker.id=2
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/export/servers/kafka_2.11-0.10.0.0/logs
num.partitions=2
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.flush.interval.messages=10000
log.flush.interval.ms=1000
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=node01:2181,node02:2181,node03:2181
zookeeper.connection.timeout.ms=6000
group.initial.rebalance.delay.ms=0
delete.topic.enable=true
host.name=node03

```

启动集群:

**注意事项**：在kafka启动前，一定要让zookeeper启动起来。

前台启动:

node01服务器执行以下命令来启动kafka集群

```
cd /export/servers/kafka_2.11-0.10.0.0
bin/kafka-server-start.sh config/server.properties
```

后台启动:

**node01****执行以下命令将kafka进程启动在后台**

```
cd /export/servers/kafka_2.11-0.10.0.0
nohup bin/kafka-server-start.sh config/server.properties >/export/log/kafka.log 2>&1 &
```

停止命令:

node01执行以下命令便可以停止kakfa进程

```
cd /export/servers/kafka_2.11-0.10.0.0
bin/kafka-server-stop.sh
```

查看启动进程:

```
jps
```

# 五 集群操作

nohup bin/kafka-server-start.sh config/server.properties 2>&1 &

### 创建topic

三分区 两副本

```
bin/kafka-topics.sh --create  --partitions 3 --replication-factor 2 --topic test --zookeeper node01:2181,node02:2181,node03:2181
```

### 查看topic

```
 bin/kafka-topics.sh --list --zookeeper node01:2181,node02:2181,node03:2181
```

### 生产数据

```
bin/kafka-console-producer.sh --broker-list node01:9092,node02:9092,node03:9092 --topic test
```

### 消费数据

```
bin/kafka-console-consumer.sh --from-beginning  --topic test --zookeeper node01:2181,node02:2181,node03:2181
```

### 查看topic的一些信息

```
bin/kafka-topics.sh --describe  --topic test --zookeeper node01:2181
```

### 修改topic的配置属性

```
bin/kafka-topics.sh --zookeeper node01:2181 --alter --topic test --config flush.messages=1
```

### 删除topic

```
 bin/kafka-topics.sh --zookeeper node01:2181 --delete --topic test
```



## kafka集群当中JavaAPI操作

依赖

```
<dependencies>
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version> 0.10.0.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-streams</artifactId>
        <version>0.10.0.0</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <!-- java编译插件 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.2</version>
            <configuration>
                <source>1.8</source>
                <target>1.8</target>
                <encoding>UTF-8</encoding>
            </configuration>
        </plugin>
    </plugins>
</build>

```



### kafka集群当中ProducerAPI

生产者代码:

```java
public class MyProducer {
    /**
     * 实现生产数据到kafka test这个topic里面去
     * @param args
     */
    public static void main(String[] args) {
        Properties props = new Properties();
        //kafka的机器
        props.put("bootstrap.servers", "node01:9092");
        //消息确认机制
        props.put("acks", "all");
        //消息没有发送成功 重试几次
        props.put("retries", 0);
        //消息最大一批次 发送多少条
        props.put("batch.size", 16384);
        //消息每条都进行确认
        props.put("linger.ms", 1);
        //缓存内存大小
        props.put("buffer.memory", 33554432);
        //一下俩个  k和value 进行序列化 和反序列化
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        //获取kafakProducer这个类
        KafkaProducer<String, String> kafkaProducer = new KafkaProducer<>(props);
        //使用循环发送消息
        for(int i =0;i<100;i++){
           // kafkaProducer.send(new ProducerRecord<String, String>("test","mymessage" + i));
            kafkaProducer.send(new ProducerRecord<String, String>("mypartition","mymessage" + i));
        }
        //关闭生产者
        kafkaProducer.close();
    }

}

```

### kafka集群当中的consumerAPI

消费者:

自动提交offset：

```java
public class AutomaticConsumer {

    /**
     * 自动提交offset
     * @param args
     */
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "node01:9092");
        props.put("group.id", "test_group");  //消费组
        props.put("enable.auto.commit", "true");//允许自动提交offset
        props.put("auto.commit.interval.ms", "1000");//每隔多久自动提交offset
        props.put("session.timeout.ms", "30000");
        //指定key，value的反序列化类
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<>(props);

        //指定消费哪个topic里面的数据
        kafkaConsumer.subscribe(Arrays.asList("test"));
        //使用死循环来消费test这个topic里面的数据
        while (true){
            //这里面是我们所有拉取到的数据
            ConsumerRecords<String, String> consumerRecords = kafkaConsumer.poll(1000);
            for (ConsumerRecord<String, String> consumerRecord : consumerRecords) {
                long offset = consumerRecord.offset();
                String value = consumerRecord.value();
                System.out.println("消息的offset值为"+offset +"消息的value值为"+ value);
            }
        }
    }
}

```

手动提交offset：

```java
public class MannualConsumer {
    /**
     * 实现手动的提交offset
     * @param args
     */
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "node01:9092");
        props.put("group.id", "test_group");
        props.put("enable.auto.commit", "false"); //禁用自动提交offset，后期我们手动提交offset
        props.put("auto.commit.interval.ms", "1000");
        props.put("session.timeout.ms", "30000");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<>(props);
        kafkaConsumer.subscribe(Arrays.asList("test"));  //订阅test这个topic
        int minBatchSize = 200;  //达到200条进行批次的处理，处理完了之后，提交offset
        List<ConsumerRecord<String, String>> consumerRecordList = new ArrayList<>();//定义一个集合，用于存储我们的ConsumerRecorder
        while (true){

            ConsumerRecords<String, String> consumerRecords = kafkaConsumer.poll(1000);
            for (ConsumerRecord<String, String> consumerRecord : consumerRecords) {
                consumerRecordList.add(consumerRecord);
            }
            if(consumerRecordList.size() >=  minBatchSize){
                //如果集合当中的数据大于等于200条的时候，我们批量进行处理
                //将这一批次的数据保存到数据库里面去
                //insertToDb(consumerRecordList);
                System.out.println("手动提交offset的值");
                //提交offset，表示这一批次的数据全部都处理完了
               // kafkaConsumer.commitAsync();  //异步提交offset值
                kafkaConsumer.commitSync();//同步提交offset的值
                consumerRecordList.clear();//清空集合当中的数据
            }
        }
    }
}

```

offset：offset记录了每个分区里面的消息消费到了哪一条，下一次来的时候，我们继续从上一次的记录接着消费

### kafka的streamAPI 

需求：使用StreamAPI获取test这个topic当中的数据，然后将数据全部转为大写，写入到test2这个topic当中去

#### 第一步：创建一个topic 

node01服务器使用以下命令来常见一个topic 名称为test2

```shell
cd /export/servers/kafka_2.11-0.10.0.0/
bin/kafka-topics.sh --create  --partitions 3 --replication-factor 2 --topic test2 --zookeeper node01:2181,node02:2181,node03:2181

```

#### 第二步：开发StreamAPI 

注意：如果程序启动的时候抛出异常，找不到文件夹的路径，需要我们手动的去创建文件夹的路径

```java
public class Stream {

    /**
     * 通过streamAPI实现将数据从test里面读取出来，写入到test2里面去
     * @param args
     */
    public static void main(String[] args) {

        Properties properties = new Properties();
        properties.put(StreamsConfig.APPLICATION_ID_CONFIG,"bigger");
        properties.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG,"node01:9092");
        properties.put(StreamsConfig.KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());//key的序列化和反序列化的类
        properties.put(StreamsConfig.VALUE_SERDE_CLASS_CONFIG,Serdes.String().getClass());
        //获取核心类 KStreamBuilder
        KStreamBuilder kStreamBuilder = new KStreamBuilder();
        //通过KStreamBuilder调用stream方法表示从哪个topic当中获取数据
        //调用mapValues方法，表示将每一行value都给取出来
        //line表示我们取出来的一行行的数据
        //将转成大写的数据，写入到test2这个topic里面去
        kStreamBuilder.stream("test").mapValues(line -> line.toString().toUpperCase()).to("test2");
        //通过kStreamBuilder可以用于创建KafkaStream  通过kafkaStream来实现流失的编程启动
        KafkaStreams kafkaStreams = new KafkaStreams(kStreamBuilder, properties);
        kafkaStreams.start();  //调用start启动kafka的流 API
    }

}

```

#### 第三步：生产数据

```shell 
node01执行以下命令，向test这个topic当中生产数据
cd /export/servers/kafka_2.11-0.10.0.0
bin/kafka-console-producer.sh --broker-list node01:9092,node02:9092,node03:9092 --topic test

```

 **第四步：消费数据**

node02执行一下命令消费test2这个topic当中的数据

```shell 
cd /export/servers/kafka_2.11-0.10.0.0
bin/kafka-console-consumer.sh --from-beginning  --topic test2 --zookeeper node01:2181,node02:2181,node03:2181
```

## 六 kafka原理

### 一 生产者

生产者是一个向kafka Cluster发布记录的客户端；**生产者是线程安全的**，跨线程共享单个生产者实例通常比具有多个实例更快。

**必要条件**

生产者要进行生产数据到kafka  Cluster中，必要条件有以下三个：

```
#1、地址
bootstrap.servers=node01:9092
#2、序列化 key.serializer=org.apache.kafka.common.serialization.StringSerializer value.serializer=org.apache.kafka.common.serialization.StringSerializer
#3、主题（topic） 需要制定具体的某个topic（order）即可。
```

**生产者写数据**

流程:

1、总体流程

Producer连接任意活着的Broker，请求指定Topic，Partion的Leader元数据信息，然后直接与对应的Broker直接连接，发布数据

2、开放分区接口(生产者数据分发策略)

2.1、用户可以指定分区函数，使得消息可以根据key，发送到指定的Partition中。

2.2、kafka在数据生产的时候，有一个数据分发策略。默认的情况使用DefaultPartitioner.class类。 这个类中就定义数据分发的策略。

2.3、如果是用户指定了partition，生产就不会调用DefaultPartitioner.partition()方法

2.4、当用户指定key，使用hash算法。如果key一直不变，同一个key算出来的hash值是个固定值。如果是固定 值，这种hash取模就没有意义。

```
Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions
```

2.5、 当用既没有指定partition也没有key。

```
/**
The default partitioning strategy:
<ul>
<li>If a partition is specified in the record, use it
<li>If no partition is specified but a key is present choose a partition based on a hash of the key
<li>If no partition or key is present choose a partition in a round-robin fashion
*/
```

2.6、数据分发策略的时候，可以指定数据发往哪个partition。当ProducerRecord 的构造参数中有partition的时 候，就可以发送到对应partition上。

```java
public class PartitionProducer {

    /**
     * kafka生产数据 通过不同的方式，将数据写入到不同的分区里面去
     *
     * @param args
     */
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "node01:9092");
        props.put("acks", "all");
        props.put("retries", 0);
        props.put("batch.size", 16384);
        props.put("linger.ms", 1);
        props.put("buffer.memory", 33554432);
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        //配置我们自定义分区类
        props.put("partitioner.class","cn.itcast.kafka.partition.MyPartitioner");


        //获取kafakProducer这个类
        KafkaProducer<String, String> kafkaProducer = new KafkaProducer<>(props);
        //使用循环发送消息
        for(int i =0;i<100;i++){
            //分区策略第一种，如果既没有指定分区号，也没有指定数据key，那么就会使用轮询的方式将数据均匀的发送到不同的分区里面去
            //ProducerRecord<String, String> producerRecord1 = new ProducerRecord<>("mypartition", "mymessage" + i);
            //kafkaProducer.send(producerRecord1);
            //第二种分区策略 如果没有指定分区号，指定了数据key，通过key.hashCode  % numPartitions来计算数据究竟会保存在哪一个分区里面
            //注意：如果数据key，没有变化   key.hashCode % numPartitions  =  固定值  所有的数据都会写入到同一个分区里面去
            //ProducerRecord<String, String> producerRecord2 = new ProducerRecord<>("mypartition", "mykey", "mymessage" + i);
            //kafkaProducer.send(producerRecord2);
            //第三种分区策略：如果指定了分区号，那么就会将数据直接写入到对应的分区里面去
          //  ProducerRecord<String, String> producerRecord3 = new ProducerRecord<>("mypartition", 0, "mykey", "mymessage" + i);
           // kafkaProducer.send(producerRecord3);
            //第四种分区策略：自定义分区策略。如果不自定义分区规则，那么会将数据使用轮询的方式均匀的发送到各个分区里面去
            kafkaProducer.send(new ProducerRecord<String, String>("mypartition","mymessage"+i));
        }
        //关闭生产者
        kafkaProducer.close();
    }

}

```

自定义分区策略:

```java
//对应第四种分区
public class MyPartitioner implements Partitioner {
    /*
    这个方法就是确定数据到哪一个分区里面去
    直接return  2 表示将数据写入到2号分区里面去
     */
    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        return 2;
    }

    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> configs) {

    }
}

```

总体四种:

a、可根据主题和内容发送

```java
Producer<String, String> producer = new KafkaProducer<String, String>(props);

//可根据主题和内容发送

producer.send(new ProducerRecord<String, String>("my-topic","具体的数据"));
```

b、根据主题，key、内容发送

```
Producer<String, String> producer = new KafkaProducer<String, String>(props);

//可根据主题、key、内容发送

producer.send(new  ProducerRecord<String,  String>("my-topic","key","具体的数据"));
```

c、根据主题、分区、key、内容发送

```
Producer<String, String> producer = new KafkaProducer<String, String>(props);

//可根据主题、分区、key、内容发送

producer.send(new  ProducerRecord<String,  String>("my-topic",1,"key","具体的数据"));
```

d、根据主题、分区、时间戳、key，内容发送

```
Producer<String, String> producer = new KafkaProducer<String, String>(props);

//可根据主题、分区、时间戳、key、内容发送
producer.send(new  ProducerRecord<String,  String>("my-topic",1,12L,"key","具体的数据"));
```

总结：1如果指定了数据的分区号，那么数据直接生产到对应的分区里面去

2如果没有指定分区好，出现了数据key，通过key取hashCode来计算数据究竟该落在哪一个分区里面

3如果既没有指定分区号，也没有指定数据的key，使用round-robin轮询 的这种机制来是实现

### 二   消费者

消费者是一个从kafka Cluster中消费数据的一个客户端；该客户端可以处理kafka brokers中的故障问题，并且可以适应在集群内的迁移的topic分区；该客户端还允许消费者组使用消费者组来进行负载均衡。

消费者维持一个TCP的长连接来获取数据，使用后未能正常关闭这些消费者问题会出现，因此**消费者不是线程安全 的。**

**必要条件**

```
#1、地址
bootstrap.servers=node01:9092
#2、序列化 key.serializer=org.apache.kafka.common.serialization.StringSerializer value.serializer=org.apache.kafka.common.serialization.StringSerializer
#3、主题（topic） 需要制定具体的某个topic（order）即可。
#4、消费者组 group.id=test
```

一 自动提交offset的值(参考上面)

二 手动提交offset的值(参考上面)

三处理完每一个分区里面的数据，就马上提交这个分区里面的数据

```java
public class ConmsumerPartition {

    /**
     * 处理完每一个分区里面的数据，就马上提交这个分区里面的数据
     * @param args
     */
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "node01:9092");
        props.put("group.id", "test_group");
        props.put("enable.auto.commit", "false"); //禁用自动提交offset，后期我们手动提交offset
        props.put("auto.commit.interval.ms", "1000");
        props.put("session.timeout.ms", "30000");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<>(props);
        kafkaConsumer.subscribe(Arrays.asList("mypartition"));
        while (true){
            //通过while ture进行消费数据
            ConsumerRecords<String, String> records = kafkaConsumer.poll(1000);
            //获取mypartition这个topic里面所有的分区
            Set<TopicPartition> partitions = records.partitions();
            //循环遍历每一个分区里面的数据，然后将每一个分区里面的数据进行处理，处理完了之后再提交每一个分区里面的offset
            for (TopicPartition partition : partitions) {
                //获取每一个分区里面的数据
                List<ConsumerRecord<String, String>> records1 = records.records(partition);
                for (ConsumerRecord<String, String> record : records1) {
                    System.out.println(record.value()+"===="+ record.offset());
                }
                //获取我们分区里面最后一条数据的offset，表示我们已经消费到了这个offset了
                long offset = records1.get(records1.size() - 1).offset();
                //提交offset
                //提交我们的offset，并且给offset加1  表示我们下次从没有消费的那一条数据开始消费
                kafkaConsumer.commitSync(Collections.singletonMap(partition,new OffsetAndMetadata(offset + 1)));
            }
        }
    }
}

```

四 实现消费一个topic里面某些分区里面的数据

```java
public class ConsumerSomePartition {
    //实现消费一个topic里面某些分区里面的数据
    public static void main(String[] args) {

        Properties props = new Properties();
        props.put("bootstrap.servers", "node01:9092,node02:9092,node03:9092");
        props.put("group.id", "test_group");  //消费组
        props.put("enable.auto.commit", "true");//允许自动提交offset
        props.put("auto.commit.interval.ms", "1000");//每隔多久自动提交offset
        props.put("session.timeout.ms", "30000");
        //指定key，value的反序列化类
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        //获取kafkaConsumer
        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);

        //通过consumer订阅某一个topic，进行消费.会消费topic里面所有分区的数据
       // consumer.subscribe();

        //通过调用assign方法实现消费mypartition这个topic里面的0号和1号分区里面的数据

        TopicPartition topicPartition0 = new TopicPartition("mypartition", 0);
        TopicPartition topicPartition1 = new TopicPartition("mypartition", 1);
        //订阅我们某个topic里面指定分区的数据进行消费
        consumer.assign(Arrays.asList(topicPartition0,topicPartition1));

        int i =0;
        while(true){
            ConsumerRecords<String, String> records = consumer.poll(1000);
            for (ConsumerRecord<String, String> record : records) {
                i++;
                System.out.println("数据值为"+ record.value()+"数据的offset为"+ record.offset());
                System.out.println("消费第"+i+"条数据");
            }
        }
    }
}

```

五 消费者数据丢失-数据重复

说明：

1、已经消费的数据对于kafka来说，会将消费组里面的oﬀset值进行修改，那什么时候进行修改了？是在数据消费 完成之后，比如在控制台打印完后自动提交；

2、提交过程：是通过kafka将oﬀset进行移动到下个message所处的oﬀset的位置。

3、拿到数据后，存储到hbase中或者mysql中，如果hbase或者mysql在这个时候连接不上，就会抛出异常，如果在处理数据的时候已经进行了提交，那么kafka伤的oﬀset值已经进行了修改了，但是hbase或者mysql中没有数据，这个时候就会出现**数据丢失**。

4、什么时候提交oﬀset值？在Consumer将数据处理完成之后，再来进行oﬀset的修改提交。默认情况下oﬀset是 自动提交，需要修改为手动提交oﬀset值。

5、如果在处理代码中正常处理了，但是在提交oﬀset请求的时候，没有连接到kafka或者出现了故障，那么该次修 改oﬀset的请求是失败的，那么下次在进行读取同一个分区中的数据时，会从已经处理掉的oﬀset值再进行处理一 次，那么在hbase中或者mysql中就会产生两条一样的数据，也就是**数据重复**

**kafka当中数据消费模型:**

eactly  once：消费且仅仅消费一次，可以在事务里面执行kafka的操作

at  most  once：至多消费一次，数据丢失的问题

at  least  once ：至少消费一次，数据重复消费的问题

**kafka的消费模式：决定了offset值保存在哪里:**

kafka的highLevel API进行消费：将offset保存在zk当中，每次更新offset的时候，都需要连接zk

以及kafka的lowLevelAP进行消费：保存了消费的状态，其实就是保存了offset，将offset保存在kafka的一个默认的topic里面。kafka会自动的创建一个topic，保存所有其他topic里面的offset在哪里

kafka将数据全部都以文件的方式保存到了文件里面去了。

###三   kafka的log-存储机制

kafka里面一个topic有多个partition组成，每一个partition里面有多个segment组成，每个segment都由两部分组成，分别是.log文件和.index文件  。一旦.log文件达到1GB的时候，就会生成一个新的segment

.log文件：顺序的保存了我们的写入的数据

~~~
00000000000000000000.log
~~~



.index文件：索引文件，使用索引文件，加快kafka数据的查找速度   

~~~
00000000000000000000.index
~~~

**kafka日志的组成:**

1 segment  ﬁle组成：由两个部分组成，分别为index ﬁle和data ﬁle，此两个文件一一对应且成对出现； 后缀.index和.log分别表示为segment的索引文件、数据文件。

2 segment文件命名规则：partion全局的第一个segment从0开始，后续每个segment文件名为上一个全局 partion的最大oﬀset（偏移message数）。数值最大为64位long大小，19位数字字符长度，没有数字就用0 填充

3 通过索引信息可以快速定位到message。通过index元数据全部映射到memory，可以避免segment ﬁle的IO磁盘操作；

4 通过索引文件稀疏存储，可以大幅降低index文件元数据占用空间大小。 稀疏索引：为了数据创建索引，但范围并不是为每一条创建，而是为某一个区间创建；

**好处**：就是可以减少索引值的数量。

**不好的地方**：找到索引区间之后，要得进行第二次处理。

**kafka日志清理**

kafka中清理日志的方式有两种：delete和compact。

删除的阈值有两种：过期的时间和分区内总日志大小。

在kafka中，因为数据是存储在本地磁盘中，并没有像hdfs的那样的分布式存储，就会产生磁盘空间不足的情 况，可以采用删除或者合并的方式来进行处理

可以通过时间来删除、合并：默认7天 还可以通过字节大小、合并

**总结：查找数据的过程**

第一步：通过offset确定数据保存在哪一个segment里面了，

第二部：查找对应的segment里面的index文件  。index文件都是key/value对的。key表示数据在log文件里面的顺序是第几条。value记录了这一条数据在全局的标号。如果能够直接找到对应的offset直接去获取对应的数据即可

如果index文件里面没有存储offset，就会查找offset最近的那一个offset，例如查找offset为7的数据找不到，那么就会去查找offset为6对应的数据，找到之后，再取下一条数据就是offset为7的数据

### 四 kafka的消息不丢失机制

生产者：使用ack机制     有多少个分区，就启动多少个线程来进行同步数据

     ~~~
1发送数据的方式:

     可以采用同步或者异步的方式
      同步：发送一批数据给kafka后，等待kafka返回结果
         1、生产者等待10s，如果broker没有给出ack相应，就认为失败。
         2、生产者重试3次，如果还没有相应，就报错
      异步: 发送一批数据给kafka，只是提供一个回调函数
         1、先将数据保存在生产者端的buffer中。buffer大小是2万条 
         2、满足数据阈值或者数量阈值其中的一个条件就可以发送数据。
         3、发送一批数据的大小是500条
         说明：如果broker迟迟不给ack，而buﬀer又满了，开发者可以设置是否直接清空buﬀer中的数据。
     ~~~

~~~
ask机制   确认机制
服务端返回一个确认码，即ack响应码；ack的响应有三个状态值
   0：生产者只负责发送数据，不关心数据是否丢失，响应的状态码为0（丢失的数据，需要再次发送      ）
   1：partition的leader收到数据，响应的状态码为1
   -1：所有的从节点都收到数据，响应的状态码为-1
   说明：如果broker端一直不给ack状态，producer永远不知道是否成功；producer可以设置一个超时时间10s，超 过时间认为失败。
~~~

broker：使用partition的副本机制     

消费者：使用offset来进行记录     在消费者消费数据的时候，只要每个消费者记录好oﬀset值即可，就能保证数据不丢失。

### 五 CAP 理论  与kafka中的CAP	

分布式系统（distributed system）正变得越来越重要，大型网站几乎都是分布式的。

分布式系统的最大难点，就是各个节点的状态如何同步。

为了解决各个节点之间的状态同步问题，在1998年，由加州大学的计算机科学家 Eric Brewer 提出分布式系统的三个指标，分别是

~~~
 Consistency：一致性
Availability：可用性
Partition tolerance：分区容错性

这三个指标不可能同时做到。这个结论就叫做 CAP 定理
~~~

1  Partition tolerance

大多数分布式系统都分布在多个子网络。每个子网络就叫做一个区（partition）。分区容错的意思是，区间通信可能失败。比如，一台服务器放在中国，另一台服务器放在美国，这就是两个区，它们之间可能无法通信

  一般来说，分区容错无法避免，因此可以认为 CAP 的 P 总是存在的。即永远可能存在分区容错这个问题  

2  Consistency

Consistency 中文叫做"一致性"。意思是，写操作之后的读操作，必须返回该值。

   如  v0这个数据存在  s1和s2中  用户向s1中 把v0这个值改为v1  则s2中的值也应该改为 v1  

3  Availability

Availability 中文叫做"可用性"，意思是只要收到用户的请求，服务器就必须给出回应。

用户可以选择向服务器 G1 或 G2 发起读操作。不管是哪台服务器，只要收到请求，就必须告诉用户，到底是 v0 还是 v1，否则就不满足可用性。

 **kafka中的CAP 应用**

  kafka为一个分布式消息队列系统,一定满足CAP 定理

​	kafka满足的是CAP定律当中的CA，其中Partition  tolerance通过的是一定的机制尽量的保证分区容错性。

其中C表示的是数据一致性。A表示数据可用性。

kafka首先将数据写入到不同的分区里面去，每个分区又可能有好多个副本，数据首先写入到leader分区里面去，读写的操作都是与leader分区进行通信，保证了数据的一致性原则，也就是满足了Consistency原则。然后kafka通过分区副本机制，来保证了kafka当中数据的可用性。但是也存在另外一个问题，就是副本分区当中的数据与leader当中的数据存在差别的问题如何解决，这个就是Partition tolerance的问题。

**kafka为了解决Partition tolerance的问题，使用了ISR的同步策略，来尽最大可能减少Partition tolerance的问题**

~~~
每个leader会维护一个ISR（a set of in-sync replicas，基本同步）列表
ISR列表主要的作用就是决定哪些副本分区是可用的，也就是说可以将leader分区里面的数据同步到副本分区里面去，决定一个副本分区是否可用的条件有两个
•	replica.lag.time.max.ms=10000     副本分区与主分区心跳时间延迟
•	replica.lag.max.messages=4000    副本分区与主分区消息同步最大差
~~~

 总结  主分区与副本分区之间的数据同步：

两个指标，一个是副本分区与主分区之间的心跳间隔，超过10S就认为副本分区已经宕机，会将副本分区从ISR当中移除

主分区与副本分区之间的数据同步延迟，默认数据差值是4000条

例如主分区有10000条数据，副本分区同步了3000条，差值是7000 >  4000条，也会将这个副本分区从ISR列表里面移除掉

**kafka in zookeeper**

kafka集群中：包含了很多的broker，但是在这么的broker中也会有一个老大存在；是在kafka节点中的一个临时节 点，去创建相应的数据，这个老大就是 **Controller** **Broker**。

**Controller Broker职责**：管理所有的

## kafka 监控及运维

在开发工作中，消费在Kafka集群中消息，数据变化是我们关注的问题，当业务前提不复杂时，我们可以使用Kafka 命令提供带有Zookeeper客户端工具的工具，可以轻松完成我们的工作。随着业务的复杂性，增加Group和 Topic，那么我们使用Kafka提供命令工具，已经感到无能为力，那么Kafka监控系统目前尤为重要，我们需要观察 消费者应用的细节。

## 1、kafka-eagle概述

 为了简化开发者和服务工程师维护Kafka集群的工作有一个监控管理工具，叫做 Kafka-eagle。这个管理工具可以很容易地发现分布在集群中的哪些topic分布不均匀，或者是分区在整个集群分布不均匀的的情况。它支持管理多个集群、选择副本、副本重新分配以及创建Topic。同时，这个管理工具也是一个非常好的可以快速浏览这个集群的工具， 

## 2、环境和安装

### 1、环境要求

需要安装jdk，启动zk以及kafka的服务

### 2、安装步骤

#### 1、下载源码包

kafka-eagle官网：

http://download.kafka-eagle.org/ 

我们可以从官网上面直接下载最细的安装包即可kafka-eagle-bin-1.3.2.tar.gz这个版本即可

代码托管地址：

https://github.com/smartloli/kafka-eagle/releases

#### 2、解压

这里我们选择将kafak-eagle安装在第三台

直接将kafka-eagle安装包上传到node03服务器的/export/softwares路径下，然后进行解压

node03服务器执行一下命令进行解压

~~~
cd /export/softwares/
tar -zxf kafka-eagle-bin-1.3.2.tar.gz -C /export/servers/
cd /export/servers/kafka-eagle-bin-1.3.2
tar -zxf kafka-eagle-web-1.3.2-bin.tar.gz
~~~

#### 3、准备数据库

kafka-eagle需要使用一个数据库来保存一些元数据信息，我们这里直接使用msyql数据库来保存即可，在node03服务器执行以下命令创建一个mysql数据库即可

进入mysql客户端

~~~sql
create database eagle;

在这个库中创建表:
/*
 Navicat MySQL Data Transfer

 Source Server         : local
 Source Server Version : 50616
 Source Host           : localhost
 Source Database       : ke

 Target Server Version : 50616
 File Encoding         : utf-8

 Date: 06/23/2017 17:02:12 PM
*/
use eagle;
SET NAMES utf8;
SET FOREIGN_KEY_CHECKS = 0;

-- ----------------------------
--  Table structure for `ke_p_role`
-- ----------------------------
DROP TABLE IF EXISTS `ke_p_role`;
CREATE TABLE `ke_p_role` (
  `id` tinyint(4) NOT NULL AUTO_INCREMENT,
  `name` varchar(64) CHARACTER SET utf8 NOT NULL COMMENT 'role name',
  `seq` tinyint(4) NOT NULL COMMENT 'rank',
  `description` varchar(128) CHARACTER SET utf8 NOT NULL COMMENT 'role describe',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;

-- ----------------------------
--  Records of `ke_p_role`
-- ----------------------------
BEGIN;
INSERT INTO `ke_p_role` VALUES ('1', 'Administrator', '1', 'Have all permissions'), ('2', 'Devs', '2', 'Own add or delete'), ('3', 'Tourist', '3', 'Only viewer');
COMMIT;

-- ----------------------------
--  Table structure for `ke_resources`
-- ----------------------------
DROP TABLE IF EXISTS `ke_resources`;
CREATE TABLE `ke_resources` (
  `resource_id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) CHARACTER SET utf8 NOT NULL COMMENT 'resource name',
  `url` varchar(255) NOT NULL,
  `parent_id` int(11) NOT NULL,
  PRIMARY KEY (`resource_id`)
) ENGINE=InnoDB AUTO_INCREMENT=17 DEFAULT CHARSET=utf8;

-- ----------------------------
--  Records of `ke_resources`
-- ----------------------------
BEGIN;
INSERT INTO `ke_resources` VALUES ('1', 'System', '/system', '-1'), ('2', 'User', '/system/user', '1'), ('3', 'Role', '/system/role', '1'), ('4', 'Resource', '/system/resource', '1'), ('5', 'Notice', '/system/notice', '1'), ('6', 'Topic', '/topic', '-1'), ('7', 'Message', '/topic/message', '6'), ('8', 'Create', '/topic/create', '6'), ('9', 'Alarm', '/alarm', '-1'), ('10', 'Add', '/alarm/add', '9'), ('11', 'Modify', '/alarm/modify', '9'), ('12', 'Cluster', '/cluster', '-1'), ('13', 'ZkCli', '/cluster/zkcli', '12'), ('14', 'UserDelete', '/system/user/delete', '1'), ('15', 'UserModify', '/system/user/modify', '1'), ('16', 'Mock', '/topic/mock', '6');
COMMIT;

-- ----------------------------
--  Table structure for `ke_role_resource`
-- ----------------------------
DROP TABLE IF EXISTS `ke_role_resource`;
CREATE TABLE `ke_role_resource` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `role_id` int(11) NOT NULL,
  `resource_id` int(11) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=19 DEFAULT CHARSET=utf8;

-- ----------------------------
--  Records of `ke_role_resource`
-- ----------------------------
BEGIN;
INSERT INTO `ke_role_resource` VALUES ('1', '1', '1'), ('2', '1', '2'), ('3', '1', '3'), ('4', '1', '4'), ('5', '1', '5'), ('6', '1', '7'), ('7', '1', '8'), ('8', '1', '10'), ('9', '1', '11'), ('10', '1', '13'), ('11', '2', '7'), ('12', '2', '8'), ('13', '2', '13'), ('14', '2', '10'), ('15', '2', '11'), ('16', '1', '14'), ('17', '1', '15'), ('18', '1', '16');
COMMIT;

-- ----------------------------
--  Table structure for `ke_trend`
-- ----------------------------
DROP TABLE IF EXISTS `ke_trend`;
CREATE TABLE `ke_trend` (
  `cluster` varchar(64) NOT NULL,
  `key` varchar(64) NOT NULL,
  `value` varchar(64) NOT NULL,
  `hour` varchar(2) NOT NULL,
  `tm` varchar(16) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- ----------------------------
--  Table structure for `ke_user_role`
-- ----------------------------
DROP TABLE IF EXISTS `ke_user_role`;
CREATE TABLE `ke_user_role` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) NOT NULL,
  `role_id` tinyint(4) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;

-- ----------------------------
--  Records of `ke_user_role`
-- ----------------------------
BEGIN;
INSERT INTO `ke_user_role` VALUES ('1', '1', '1');
COMMIT;

-- ----------------------------
--  Table structure for `ke_users`
-- ----------------------------
DROP TABLE IF EXISTS `ke_users`;
CREATE TABLE `ke_users` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `rtxno` int(11) NOT NULL,
  `username` varchar(64) NOT NULL,
  `password` varchar(128) NOT NULL,
  `email` varchar(64) NOT NULL,
  `realname` varchar(128) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;

-- ----------------------------
--  Records of `ke_users`
-- ----------------------------
BEGIN;
INSERT INTO `ke_users` VALUES ('1', '1000', 'admin', '123456', 'admin@email.com', 'Administrator');
COMMIT;

SET FOREIGN_KEY_CHECKS = 1;

~~~

#### 4、修改kafak-eagle配置文件

node03执行以下命令修改kafak-eagle配置文件

~~~properties
cd /export/servers/kafka-eagle-bin-1.3.2/kafka-eagle-web-1.3.2/conf
vim system-config.properties

kafka.eagle.zk.cluster.alias=cluster1,cluster2
cluster1.zk.list=node01:2181,node02:2181,node03:2181
cluster2.zk.list=node01:2181,node02:2181,node03:2181

kafka.eagle.driver=com.mysql.jdbc.Driver
kafka.eagle.url=jdbc:mysql://node03:3306/eagle
kafka.eagle.username=root
kafka.eagle.password=123456

~~~

#### 5、配置环境变量

kafka-eagle必须配置环境变量，node03服务器执行以下命令来进行配置环境变量

~~~
vim /etc/profile

export KE_HOME=/export/servers/kafka-eagle-bin-1.3.2/kafka-eagle-web-1.3.2
export PATH=:$KE_HOME/bin:$PATH

source /etc/profile
~~~

#### 6、启动kafka-eagle

node03执行以下界面启动kafka-eagle

~~~shell 
cd /export/servers/kafka-eagle-bin-1.3.2/kafka-eagle-web-1.3.2/bin
chmod u+x ke.sh
./ke.sh start  stop  status
~~~

#### 7、主界面

访问kafka-eagle

~~~~
http://node03:8048/ke
用户名：admin
密码：123456
~~~~