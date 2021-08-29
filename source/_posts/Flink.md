---
title: Flink
tags:
  - Flink
categories:
  - Flink
encrypt: 
enc_pwd: 
abbrlink: 3113
date: 2020-01-29 09:20:09
summary_img:
---

# Flink

目标:

1. 批数据处理编程ExecutionEnviroment
2. 流数据处理编程StreamExecutionEnviroment
3. Flink原理
4. checkpoint、watermark

## Flink是什么

- Flink是什么

  - Flink是一个分布式计算引擎   MapReduce Tez Spark Storm

    - 同时支持流计算和批处理,Spark也能做批和流
    - 和Spark不同, Flink是使用流的思想做批, Spark是采用做批的思想做流

  - Flink的优势

    - 和Hadoop相比, Flink使用内存进行计算, 速度明显更优
    - 和同样使用内存的Spark相比, Flink对于流的计算是实时的, **延迟更低**
    - 和同样使用实时流的Storm相比, Flink明显具有更优秀的API, 以及更多的支持, 并且支持批量计算

  - 速度

    测试环境：
    1.CPU：7000个； 

    2.内存：单机128GB； 

    3.版本：Hadoop 2.3.0，Spark 1.4，Flink 0.9 

    4.数据：800MB，8GB，8TB； 

    5.算法：K-means：以空间中K个点为中心进行聚类，对最靠近它们的对象归类。通过迭代的方法，逐次
    更新各聚类中心的值，直至得到最好的聚类结果。

     6.迭代：K=10，3组数据

    纵坐标是秒，横坐标是次数

    ![img](/images/flink/1562028424379.png)

    结论:

    Spark和Flink全部都运行在Hadoop YARN上，性能为Flink > Spark > Hadoop(MR)，迭代次数越多越明显,**性能上，Flink优于Spark和Hadoop最主要的原因是Flink支持增量迭代，具有对迭代自动优化的功能**

    - 在单机上, Storm大概能达到30万条/秒的吞吐量, Flink的吞吐量大概是Storm得3-5倍.在阿里中,Flink集群能达到每秒能处理17亿数据量,一天可处理上万亿条数据
    - 在单机上, Flink消息处理的延迟大概在50毫秒左右, 这个数据大概是Spark的3-5倍

- Flink的发展现状

- 08年Flink在德国柏林大学

- 14年Apache立为顶级项目.阿里15年开始使用

  - Flink在很多公司的生产环境中得到了使用, 例如: ebay, 腾讯, 阿里, 亚马逊, 华为等
  - Blink

- Flink的母公司被阿里全资收购, 阿里一直致力于Flink在国内的推广使用

- Flink的适用场景

  - 零售业和市场营销(运营)
  - 物联网,5G 300M/s  延迟低  50ms  100ms 无人驾驶

  华人运通:hiphi1 10万辆 560个  没200ms采集一次数据 2800条

  - 电信业
  - 银行和金融业

- 对比Flink、Spark、Storm

  Flink、Spark Streaming、Storm都可以进行实时计算，但各有特点

  | 计算框架            | 处理模型                  | 保证次数                    | 容错机制                | 延时   | 吞吐量  |
  | --------------- | --------------------- | ----------------------- | ------------------- | ---- | ---- |
  | Storm           | native（数据进入立即处理）      | At-least-once<br />至少一次 | ACK机制               | 低    | 低    |
  | Spark Streaming | micro-batching        | Exactly-once            | 基于RDD和 checkpoint   | 中    | 高    |
  | Flink           | native、micro-batching | Exactly-once            | checkpoint（Flink快照） | 低    | 高    |



## Flink的体系架构

![1552262880777](/images/flink/1552262880777.png)

## 有界流和无界流

无界流：意思很明显，只有开始没有结束。必须连续的处理无界流数据，也即是在事件注入之后立即要对其
进行处理。不能等待数据到达了再去全部处理，因为数据是无界的并且永远不会结束数据注入。处理无界流数
据往往要求事件注入的时候有一定的顺序性，例如可以以事件产生的顺序注入，这样会使得处理结果完整。

有界流：也即是有明确的开始和结束的定义。有界流可以等待数据全部注入完成了再开始处理。注入的顺序
不是必须的了，因为对于一个静态的数据集，我们是可以对其进行排序的。有界流的处理也可以称为批处
理。

Data Streams ，Flink认为有界数据集是无界数据流的一种特例，所以说有界数据集也是一种数据流，事件流
也是一种数据流。Everything is streams ，即Flink可以用来处理任何的数据，可以支持批处理、流处理、
AI、MachineLearning等等。
Stateful Computations，即有状态计算。有状态计算是最近几年来越来越被用户需求的一个功能。比如说一个
网站一天内访问UV数，那么这个UV数便为状态。Flink提供了内置的对状态的一致性的处理，即如果任务发生
了Failover，其状态不会丢失、不会被多算少算，同时提供了非常高的性能。

其它特点:
​	性能优秀(尤其在流计算领域)
​	高可扩展性
​	支持容错
​	纯内存式的计算引擎，做了内存管理方面的大量优化
​	支持eventime 的处理
​	支持超大状态的Job(在阿里巴巴中作业的state大小超过TB的是非常常见的)
​	**支持exactly-once 的处理。**



## Flink安装及任务提交

三种:

1 local（本地）——单机模式，一般不使用
2 standalone——独立模式，Flink自带集群，开发测试环境使用
3 yarn——计算资源统一由Hadoop YARN管理，生产测试环境使用

- Standalone单机模式
- Standalone集群模式 
- Standalone的高可用HA模式

### Standalone方式安装

1. 将Flink解压到指定目录，

2. ![1562030371983](/images/flink/1562030371983.png)

3. 进入到Flink目录，使用以下命令启动Flink

   ```shell
   ./bin/start-cluster.sh
   ```

4. ![1562030558179](/images/flink/1562030558179.png)

5. 打开浏览器，使用`http://服务器地址:8081`，进入到Flink的Web UI中

   ![1551371410207](/images/flink/1551371410207.png)

### standalone集群方式安装

1. 下载Flink，并解压到指定目录

2. 配置`conf/flink-conf.yaml`

3. ![1562031534374](/images/flink/1562031534374.png)

   ```yaml
   # 配置Master的机器名（IP地址）
   jobmanager.rpc.address: node01
   # 配置Master的端口号
   jobmanager.rpc.port: 6123
   # 配置Master的堆大小（默认MB）
   jobmanager.heap.size: 1024m
   # 配置每个TaskManager的堆大小（默认MB）
   taskmanager.heap.size: 1024m
   # 配置每个TaskManager可以运行的槽
   taskmanager.numberOfTaskSlots: 4
   # 配置每个taskmanager生成的临时文件夹
   taskmanager.tmp.dirs: /export/data/flink
   # 配置webui启动的机器名（IP地址）
   web.address: node01
   # 配置webui启动的端口号
   rest.port: 8081
   # 是否支持通过web ui提交Flink作业
   web.submit.enable: true
   ```

4. 配置`masters`

```
node01:8081
```

1. 配置`slaves`文件

   ```properties
   node01
   node02
   node03
   ```

2. 分发Flink到集群中的其他节点

   ```shell
   scp -r flink-1.7.2 node02:$PWD
   scp -r flink-1.7.2 node03:$PWD
   ```

3. 启动集群

   ```shell
   ./bin/start-cluster.sh
   ```

4. ![1562032031225](/images/flink/1562032031225.png)

5. 浏览Flink UI界面

   ```shell
   http://node01:8081
   ```

   Flink主界面：通过主界面可以查看到当前的TaskManager和多少个Slots

   ![1549597292752](/images/flink/1549597292752.png)

   TaskManager界面：可以查看到当前Flink集群中有多少个TaskManager，每个TaskManager的slots、内存、CPU Core是多少。

   ​

### HA集群搭建

Flink的JobManager存在单点故障，在生产环境中，需要对JobManager进行高可用部署。JobManager高可用基于`ZooKeeper`实现，同时HA的信息需要存储在`HDFS`中，故也需要HDFS集群。

1. 前提：启动ZooKeeper--->zkServer.sh start
2. 前提：启动HDFS --->start-dfs.sh
3. 修改node02的`conf/flink-conf.yaml`配置文件

> web.address: node02
>
> rest.port: 8081

```yaml
#node01/02/03的每个flink-conf.yaml配置文件开启HA
state.backend: filesystem
state.backend.fs.checkpointdir: hdfs://node01:8020/flink-checkpoints
high-availability: zookeeper
high-availability.storageDir: hdfs://node01:8020/flink/ha/
high-availability.zookeeper.quorum: node01:2181,node02:2181,node03:2181
high-availability.zookeeper.client.acl: open
```

1. 修改3台机器的`conf/masters`配置文件

   ```yaml
   node01:8081
   node02:8081
   ```

2. 启动Zookeeper集群

3. 启动HDFS集群

4. 启动Flink集群

### Flink程序提交方式

​	在企业生产中,为了最大化利用资源,一般都会在一个集群中同时运行多种类型的任务,我们Flink也是支持在Yarn/Mesos等平台运行.Flink的任务提交有两种方式,分别是Session和Job

![1552262977056](/images/flink/1552262977056.png)

- 首先需要配置相关Hadoop的环境

1. 修改yarn-site.xml

```
vim $HADOOP_HOME/etc/hadoop/yarn-site.xml
```

添加如下配置,**并拷贝到node02/node03**:

```xml
<property>
         <name>yarn.nodemanager.vmem-check-enabled</name>
         <value>false</value>
</property>
```

> 是否启动一个线程检查每个任务正使用的虚拟内存量，如果任务超出分配值，则直接将其杀掉，默认是true。在这里面我们需要关闭，因为对于flink使用yarn模式下，很容易内存超标，这个时候yarn会自动杀掉job

1. 添加HADOOP_CONF_DIR环境变量

```
vim /etc/profile
# 此处的路径为服务器上的hadoop路径
export HADOOP_CONF_DIR=/export/servers/hadoop路径/etc/hadoop
export HADOOP_CONF_DIR=/export/servers/hadoop-2.7.5/etc/hadoop
export HADOOP_CONF_DIR=/export/servers/hadoop-2.7.5/etc/hadoop
修改后记得:
将node02和node03的环境变量都做修改.
source /etc/profle
```

#### 方式1:session

- 在`yarn`上启动一个Flink Job，执行以下命令

  ```shell
  #启动Yarn集群
  start-yarn.sh
  #通过-h参数可以查看yarn-session的参数功能
  bin/yarn-session.sh -h
  #使用Flink自带yarn-session.sh脚本开启Yarn会话
  bin/yarn-session.sh -n 2 -tm 800 -s 2
  #可以事先关闭  否则会出现程序跑不完
  ./bin/stop-cluster.sh
  ```

  - `-n` 表示分配多少个container，这里指的就是多少个taskmanager
  - `-tm` 表示每个TaskManager的内存大小
  - `-s` 表示每个TaskManager的slots数量

  > 上面的命令的意思是，同时向Yarn申请3个container（**即便只申请了两个，因为ApplicationMaster和Job Manager有一个额外的容器。一旦将Flink部署到YARN群集中，它就会显示Job Manager的连接详细信息。**），其中 2 个 Container 启动 TaskManager（-n 2），每个 TaskManager 拥有两个 Task Slot（-s 2），并且向每个 TaskManager 的 Container 申请 800M 的内存，以及一个ApplicationMaster（Job Manager）。
  >
  > 如果不想让Flink YARN客户端始终运行，那么也可以启动分离的 YARN会话。该参数被称为-d或--detached。在这种情况下，Flink YARN客户端只会将Flink提交给群集，然后关闭它自己

  ![1552235254266](/images/flink/1552235254266.png)

- 然后使用flink提交任务：

  ```
  bin/flink run examples/batch/WordCount.jar
  ```

  ![1552235561890](/images/flink/1552235561890.png)

  通过上方的ApplicationMaster可以进入Flink的管理界面:

  ![1552235588450](/images/flink/1552235588450.png)

- 停止当前任务:

  yarn application -kill application_1562034096080_0001

#### 方式2:job

上面的YARN session是在Hadoop YARN环境下启动一个Flink cluster集群，里面的资源是可以共享给其他的Flink作业。我们还可以在YARN上启动一个Flink作业，这里我们还是使用./bin/flink，但是不需要事先启动YARN session：

```tex
bin/flink run -m yarn-cluster -yn 2 ./examples/batch/WordCount.jar
```

以上命令在参数前加上y前缀，-yn表示TaskManager个数

- 停止yarn-cluster

  ```
  yarn application -kill application的ID

  ```

这种方式一般适用于长时间工作的任务,如果任务比较小,或者工作时间短,建议适用session方式,减少资源创建的时间.实际生产环境中,job方式适用较多.



区别:

Session方式适合提交小任务,因为资源的开辟需要的时间比较长,session方式资源是共享的,

Job适合提交长时间运行的任务,大作业,资源是独有的.

## 入门案例

#### 创建工程

#### 导入pom文件

```xml
<properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <encoding>UTF-8</encoding>
        <scala.version>2.11.2</scala.version>
        <scala.compat.version>2.11</scala.compat.version>
        <hadoop.version>2.6.0</hadoop.version>
        <flink.version>1.6.0</flink.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>${scala.version}</version>
        </dependency>

        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-scala_2.11</artifactId>
            <version>${flink.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-scala_2.11</artifactId>
            <version>${flink.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-clients_2.11</artifactId>
            <version>${flink.version}</version>
        </dependency>

        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table_2.11</artifactId>
            <version>${flink.version}</version>
        </dependency>

        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>${hadoop.version}</version>
            <exclusions>
                <exclusion>
                    <artifactId>xml-apis</artifactId>
                    <groupId>xml-apis</groupId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>5.1.38</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.22</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-kafka-0.9_2.11</artifactId>
            <version>${flink.version}</version>
        </dependency>
    </dependencies>


    <build>
        <sourceDirectory>src/main/scala</sourceDirectory>
        <testSourceDirectory>src/test/scala</testSourceDirectory>
        <plugins>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.5.1</version>
                <configuration>
                    <source>${maven.compiler.source}</source>
                    <target>${maven.compiler.target}</target>
                    <!--<encoding>${project.build.sourceEncoding}</encoding>-->
                </configuration>
            </plugin>

            <plugin>
                <groupId>net.alchim31.maven</groupId>
                <artifactId>scala-maven-plugin</artifactId>
                <version>3.2.0</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>testCompile</goal>
                        </goals>
                        <configuration>
                            <args>
                                <!--<arg>-make:transitive</arg>-->
                                <arg>-dependencyfile</arg>
                                <arg>${project.build.directory}/.scala_dependencies</arg>
                            </args>

                        </configuration>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>2.18.1</version>
                <configuration>
                    <useFile>false</useFile>
                    <disableXmlReport>true</disableXmlReport>
                    <includes>
                        <include>**/*Test.*</include>
                        <include>**/*Suite.*</include>
                    </includes>
                </configuration>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.3</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <!--
                                        zip -d learn_spark.jar META-INF/*.RSA META-INF/*.DSA META-INF/*.SF
                                        -->
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass>com.itheima.batch.WordCount</mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>

                </executions>
            </plugin>
        </plugins>
    </build>
```

#### 编写代码

- WordCount

  ```scala
    def main(args: Array[String]): Unit = {
      val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment
      val dataSet: DataSet[String] = env.fromElements("hello", "world", "java", "hello", "java")
      val mapData: DataSet[(String, Int)] = dataSet.map(line => (line, 1))
      val groupData: GroupedDataSet[(String, Int)] = mapData.groupBy(0)
      val sumData: AggregateDataSet[(String, Int)] = groupData.sum(1)
      sumData.print()
    }
  ```

![1556163566631](J:/bigdata/flink/%E5%8D%95%E8%AE%B2flink/Flink_Day01/%E7%AC%94%E8%AE%B0/assets/1556163566631.png)



#### 在Yarn上运行WordCount

```scala
def main(args: Array[String]): Unit = {
    val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment
    val dataSet: DataSet[String] = env.fromElements("hello", "world", "java", "hello", "java")
    val result: AggregateDataSet[(String, Int)] = dataSet.map(line => (line, 1)).groupBy(0).sum(1)
    result.setParallelism(1)
    result.writeAsText("hdfs://node01:8020/wordcount")
    env.execute()
  }
```

- 修改pom.xml中的主类名

```xml
<transformers>
	<transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
		<mainClass>cn.itcast.flink.WorldCount_02</mainClass>
	</transformer>
</transformers>
```

- 打包

![image-20190310032347894](/images/flink/image-20190310032347894.png)

- 提交执行

  - 将打出的jar放入服务器

  ![1552237399762](/images/flink/1552237399762.png)

使用session方式提交

```
1.先启动yarnsession
 bin/yarn-session.sh -n 2 -tm 800 -s 2
2.去web界面提交jar包.或者  bin/flink run /home/elasticsearch/flinkjar/itcast_learn_flink-1.0-SNAPSHOT.jar com.itcast.DEMO.WordCount
```

使用job方式提交

```shell
bin/flink run -m yarn-cluster -yn 2 /home/elasticsearch/flinkjar/itcast_learn_flink-1.0-SNAPSHOT.jar com.itcast.DEMO.WordCount
```



## 任务调度与执行

Flink的所有操作都称之为Operator,客户端在提交任务的时候会对Operator进行优化操作,能进行合并的Operator会被合并为一个Operator,合并后的Operator称为Operator chain

![1556113247963](/images/flink/1556113247963.png)



![1556113313917](J:/bigdata/flink/%E5%8D%95%E8%AE%B2flink/Flink_Day01/%E7%AC%94%E8%AE%B0/assets/1556113313917.png)

- 客户端
  - 主要职责是提交任务, 提交后可以结束进程, 也可以等待结果返回
- `JobManager`
  - 主要职责是调度工作并协调任务做检查点
  - `JobManager`从客户端接收到任务以后, 首先生成优化过的执行计划, 再调度到`TaskManager`中执行
- `TaskManager`
  - 主要职责是从`JobManager`处接收任务, 并部署和启动任务, 接收上游的数据并处理
  - `TaskManager`在创建之初就设置好了`Slot`, 每个`Slot`可以执行一个任务

![1556105954132](/images/flink/1556105954132.png)

## Flink的API

### DataSet的转换操作

| Transformation          | Description                              |
| ----------------------- | ---------------------------------------- |
| **Map**                 | 在算子中得到一个元素并生成一个新元素<br>`data.map { x => x.toInt }` |
| **FlatMap**             | 在算子中获取一个元素, 并生成任意个数的元素<br>`data.flatMap { str => str.split(" ") }` |
| **MapPartition**        | 类似Map, 但是一次Map一整个并行分区<br>`data.mapPartition { in => in map { (_, 1) } }` |
| **Filter**              | 如果算子返回`true`则包含进数据集, 如果不是则被过滤掉<br>`data.filter { _ > 100 }` |
| **Reduce**              | 通过将两个元素合并为一个元素, 从而将一组元素合并为一个元素<br>`data.reduce { _ + _ }` |
| **ReduceGroup**         | 将一组元素合并为一个或者多个元素<br>`data.reduceGroup { elements => elements.sum }` |
| **Aggregate**           | 讲一组值聚合为一个值, 聚合函数可以看作是内置的`Reduce`函数<br>`data.aggregate(SUM, 0).aggregate(MIN, 2)`<br>`data.sum(0).min(2)` |
| **Distinct**            | 去重                                       |
| **Join**                | 按照相同的Key合并两个数据集<br>`input1.join(input2).where(0).equalTo(1)`<br>同时也可以选择进行合并的时候的策略, 是分区还是广播, 是基于排序的算法还是基于哈希的算法<br>`input1.join(input2, JoinHint.BROADCAST_HASH_FIRST).where(0).equalTo(1)` |
| **OuterJoin**           | 外连接, 包括左外, 右外, 完全外连接等<br>`left.leftOuterJoin(right).where(0).equalTo(1) { (left, right) => ... }` |
| **CoGroup**             | 二维变量的Reduce运算, 对每个输入数据集中的字段进行分组, 然后join这些组<br>`input1.coGroup(input2).where(0).equalTo(1)` |
| **Cross**               | 笛卡尔积<br>`input1.cross(input2)`           |
| **Union**               | 并集<br>`input1.union(input2)`             |
| **Rebalance**           | 分区重新平衡, 以消除数据倾斜<br>`input.rebalance()`   |
| **Hash-Partition**      | 按照Hash分区<br>`input.partitionByHash(0)`   |
| **Range-Partition**     | 按照Range分区<br>`input.partitionByRange(0)` |
| **CustomParititioning** | 自定义分区<br>`input.partitionCustom(partitioner: Partitioner[K], key)` |
| **First-n**             | 返回数据集中的前n个元素<br>`input.first(3)`         |

#### `flatmap`

map => 1 => 1

flatmap => 1 => 多个数据

```scala
  def main(args: Array[String]): Unit = {
    val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment
    val dataSet: DataSet[(String, Int)] = env.fromElements(("A" , 1) , ("B" , 1) , ("C" , 1))
    dataSet.map(line => line._1 + "=" + line._2).print()
    println("------------------")
    dataSet.flatMap(line => line._1 + "=" + line._2).print()
  }
```



#### `filter`

```scala
//TODO fileter=>
val filter:DataSet[String] = elements.filter(line => line.contains("java"))//过滤出带java的数据
filter.print()
```



#### `reduce`

```scala
//默认并行度为8,全局并行度设为1
//    env.setParallelism(1)

    //加载数据
    val sourceData: DataSet[String] = env.readTextFile("access.log")

    val mapData: DataSet[Array[String]] = sourceData.map(line => line.split(" "))

    val flatMapData: DataSet[String] = mapData.flatMap(line => line)

    val mData: DataSet[(String, Int)] = flatMapData.map(line => (line, 1))

    //对数据进行分组操作,groupBy()可以指定要按照哪个来进行分组
    val groupData: GroupedDataSet[(String, Int)] = mData.groupBy(0)

    //reduce((之前的数据,最新的数据) => )
    val reduceData: DataSet[(String, Int)] = groupData.reduce((x, y) => (x._1, x._2 + y._2))

    reduceData
      //将结果输出到本地文件
      .writeAsText("result")
      //设置输出的并行度为1
      .setParallelism(1)

    env.execute()
```



#### `reduceGroup`

> reduceGroup是reduce的一种优化方案；
>
> 它会先分组reduce，然后在做整体的reduce；这样做的好处就是可以减少网络IO；

![1552237668392](/images/flink/1552237668392.png)

![1552237681751](/images/flink/1552237681751.png)

#### `join`

- 求每个班级最高分

```scala
  def main(args: Array[String]): Unit = {
    //TODO join
    val data1 = new mutable.MutableList[(Int, String, Double)]
    //学生学号---学科---分数
    data1.+=((1, "yuwen", 90.0))
    data1.+=((2, "shuxue", 20.0))
    data1.+=((3, "yingyu", 30.0))
    data1.+=((4, "yuwen", 40.0))
    data1.+=((5, "shuxue", 50.0))
    data1.+=((6, "yingyu", 60.0))
    data1.+=((7, "yuwen", 70.0))
    data1.+=((8, "yuwen", 20.0))
    val data2 = new mutable.MutableList[(Int, String)]
    //学号 ---班级
    data2.+=((1,"class_1"))
    data2.+=((2,"class_1"))
    data2.+=((3,"class_2"))
    data2.+=((4,"class_2"))
    data2.+=((5,"class_3"))
    data2.+=((6,"class_3"))
    data2.+=((7,"class_4"))
    data2.+=((8,"class_1"))

    val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment
    val input1: DataSet[(Int, String, Double)] = env.fromCollection(Random.shuffle(data1))
    val input2: DataSet[(Int, String)] = env.fromCollection(Random.shuffle(data2))
    val data = input2.join(input1).where(0).equalTo(0){
      (input2, input1) => (input2._1,input2._2, input1._2, input1._3)
    }
    data.groupBy(1).aggregate(Aggregations.MAX, 3).print()
  }
```



#### `distinct`去重

```scala
  val data = new mutable.MutableList[(Int, String, Double)]
  data.+=((1, "yuwen", 90.0))
  data.+=((2, "shuxue", 20.0))
  data.+=((3, "yingyu", 30.0))
  data.+=((4, "wuli", 40.0))
  data.+=((5, "yuwen", 50.0))
  data.+=((6, "wuli", 60.0))
  data.+=((7, "yuwen", 70.0))
  //    //fromCollection将数据转化成DataSet
  val input: DataSet[(Int, String, Double)] = env.fromCollection(Random.shuffle(data))
  val distinct = input.distinct(1)
  distinct.print()
```



### DataStream的转换操作

Flink中的DataStream程序是实现数据流转换（例如，过滤，更新状态，定义窗口，聚合）的常规程序。数据流最初由各种来源（例如，消息队列，套接字流，文件）创建。结果通过接收器返回，例如可以将数据写入文件，或者写入标准输出（例如命令行终端）。Flink程序可以在各种情况下运行，可以独立运行，也可以嵌入其他程序中。执行可以发生在本地JVM或许多机器的集群中。

| Transformation                           | Description                              |
| ---------------------------------------- | ---------------------------------------- |
| **Map**<br>DataStream → DataStream       | `dataStream.map { x => x * 2 }`          |
| **FlatMap**<br/>DataStream → DataStream  | `dataStream.flatMap { x => x.split(",") }` |
| **Filter**<br/>DataStream → DataStream   | `dataStream.filter { _ != 0 }`           |
| **KeyBy**<br/>DataStream → KeyedStream   | 将一个流分为不相交的区, 可以按照名称指定Key, 也可以按照角标来指定<br>`dataStream.keyBy("key" |
| **Reduce**<br/>KeyedStream → DataStream  | **滚动**Reduce, 合并当前值和历史结果, 并发出新的结果值<br>`keyedStream.reduce { _ + _ }` |
| **Fold**<br/>KeyedStream → DataStream    | 按照初始值进行**滚动**折叠<br>`keyedStream.fold("start")((str, i) => { str + "-" + i })` |
| **Aggregations**<br/>KeyedStream → DataStream | **滚动**聚合, `sum`, `min`, `max`等<br>`keyedStream.sum(0)` |
| **Window**<br/>KeyedStream → DataStream  | 窗口函数, 根据一些特点对数据进行分组, 注意: 有可能是非并行的, 所有记录可能在一个任务中收集<br>`.window(TumblingEventTimeWindows.of(Time.seconds(5)))` |
| **WindowAll**<br/>DataStream → AllWindowedStream | 窗口函数, 根据一些特点对数据进行分组, 和window函数的主要区别在于可以不按照Key分组<br>`dataStream.windowAll (TumblingEventTimeWindows.of(Time.seconds(5)))` |
| **WindowApply**<br/>WindowedStream → DataStream | 将一个函数作用于整个窗口<br>`windowedStream.apply { WindowFunction }` |
| **WindowReduce**<br/>WindowedStream → DataStream | 在整个窗口上做一次reduce<br>`windowedStream.reduce { _ + _ }` |
| **WindowFold**<br/>WindowedStream → DataStream | 在整个窗口上做一次fold<br>`windowedStream.fold("start", (str, i) => { str + "-" + i })` |
| **Aggregations on windows**<br/>WindowedStream → DataStream | 在窗口上统计, `sub`, `max`, `min`<br>`windowedStream.sum(10)` |
| **Union**<br/>DataStream* → DataStream   | 合并多个流<br>`dataStream.union(dataStream1, dataStream2, ...)` |
| **Window Join**<br/>DataStream → DataStream | `dataStream.join(otherStream).where(...).equalTo(...) .window(TumblingEventTimeWindows.of(Time.seconds(3))).apply{..}` |
| **Window CoGroup**<br/>DataStream, DataStream → DataStream | `dataStream.coGroup(otherStream).where(0).equalTo(1).window(...).apply{...}` |
| **Connect**<br/>DataStream, DataStream → DataStream | 连接两个流, 并且保留各自的数据类型, 在这个连接中可以共享状态<br>`someStream.connect(otherStream)` |
| **Split**<br/>DataStream → SplitStream   | 将一个流切割为多个流<br>`someDataStream.split((x: Int) => x match ...)` |

#### 入门案例

编写代码:

```scala
  def main(args: Array[String]): Unit = {
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    val data: DataStream[String] = env.socketTextStream("node01", 9999)//netcat
    val mapData: DataStream[(String, Int)] = data.map(line => (line, 1))
    mapData.keyBy(0).sum(1).print()
    env.execute()
  }
```

在Linux窗口中发送消息:

```
 nc -lk 9999
```

#### `keyby`

```scala
 def main(args: Array[String]): Unit = {
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(3)
    val textStream: DataStream[String] = env.socketTextStream("localhost" , 12345)
    val flatMap_data: DataStream[String] = textStream.flatMap(line => line.split("\t"))
    val map_data: DataStream[(String, Int)] = flatMap_data.map(line => (line , 1))
    //TODO 逻辑上将一个流分成不相交的分区，每个分区包含相同键的元素。在内部，这是通过散列分区来实现的
    val keyByData: KeyedStream[(String, Int), String] = map_data.keyBy(line => line._1)
    keyByData.writeAsText("keyByData")
    env.execute()
  }
```

## Flink SQL



Flink SQL可以让我们通过基于Table API和SQL来进行数据处理。Flink的批处理和流处理都支持Table API。Flink SQL完全遵循ANSI SQL标准。



### 批数据SQL

------



**用法**

1. 构建Table运行环境
2. 将DataSet注册为一张表
3. 使用Table运行环境的`sqlQuery`方法来执行SQL语句



**示例**

使用Flink SQL统计用户消费订单的总金额、最大金额、最小金额、订单总数。

| 订单id | 用户名      | 订单日期             | 消费基恩  |
| ---- | -------- | ---------------- | ----- |
| 1    | zhangsan | 2018-10-20 15:30 | 358.5 |

测试数据（订单ID、用户名、订单日期、订单金额）

```html
(1,"zhangsan","2018-10-20 15:30",358.5),
(2,"zhangsan","2018-10-20 16:30",131.5),
(3,"lisi","2018-10-20 16:30",127.5),
(4,"lisi","2018-10-20 16:30",328.5),
(5,"lisi","2018-10-20 16:30",432.5),
(6,"zhaoliu","2018-10-20 22:30",451.0),
(7,"zhaoliu","2018-10-20 22:30",362.0),
(8,"zhaoliu","2018-10-20 22:30",364.0),
(9,"zhaoliu","2018-10-20 22:30",341.0)
```



**步骤**

1. 获取一个批处理运行环境
2. 获取一个Table运行环境
3. 创建一个样例类`Order`用来映射数据（订单名、用户名、订单日期、订单金额）
4. 基于本地`Order`集合创建一个DataSet source
5. 使用Table运行环境将DataSet注册为一张表
6. 使用SQL语句来操作数据（统计用户消费订单的总金额、最大金额、最小金额、订单总数）
7. 使用TableEnv.toDataSet将Table转换为DataSet
8. 打印测试



**参考代码**

```scala
val env = ExecutionEnvironment.getExecutionEnvironment
val tableEnv = TableEnvironment.getTableEnvironment(env)

val orderDataSet = env.fromElements(
    Order(1, "zhangsan", "2018-10-20 15:30", 358.5),
    Order(2, "zhangsan", "2018-10-20 16:30", 131.5),
    Order(3, "lisi", "2018-10-20 16:30", 127.5),
    Order(4, "lisi", "2018-10-20 16:30", 328.5),
    Order(5, "lisi", "2018-10-20 16:30", 432.5),
    Order(6, "zhaoliu", "2018-10-20 22:30", 451.0),
    Order(7, "zhaoliu", "2018-10-20 22:30", 362.0),
    Order(8, "zhaoliu", "2018-10-20 22:30", 364.0),
    Order(9, "zhaoliu", "2018-10-20 22:30", 341.0)
)

tableEnv.registerDataSet("t_order", orderDataSet)
val allOrderTable: Table = tableEnv.sqlQuery{
    """
        |select 
        | userName,
        | count(1) as totalCount, -- 订单总数
        | max(money) as maxMoney, -- 最大订单金额
        | min(money) as minMoney  -- 最小订单金额
        |from
        | t_order
        |group by
        | userName
      """.stripMargin
}

allOrderTable.printSchema()
tableEnv.toDataSet[Row](allOrderTable).print()
```





### 流数据SQL

------



流处理中也可以支持SQL。但是需要注意以下几点：



1. 要使用流处理的SQL，必须要添加水印时间
2. 使用`registerDataStream`注册表的时候，使用`'`来指定字段
3. 注册表的时候，必须要指定一个rowtime，否则无法在SQL中使用窗口
4. 必须要导入`import org.apache.flink.table.api.scala._`隐式参数
5. SQL中使用`tumble(时间列名, interval '时间' sencond)`来进行定义窗口
   1. TUMBLE(time_attr, interval)固定时间窗口
   2. HOP(time_attr, interval, interval)滑动窗口,
   3. SESSION(time_attr, interval)会话窗口



**示例**

使用Flink SQL来统计5秒内`用户的`订单总数、订单的最大金额、订单的最小金额。



**步骤**

1. 获取流处理运行环境
2. 获取Table运行环境
3. 设置处理时间为`EventTime`
4. 创建一个订单样例类`Order`，包含四个字段（订单ID、用户ID、订单金额、时间戳）
5. 创建一个自定义数据源
   - 使用for循环生成1000个订单
   - 随机生成订单ID（UUID）
   - 随机生成用户ID（0-2）
   - 随机生成订单金额（0-100）
   - 时间戳为当前系统时间
   - 每隔1秒生成一个订单
6. 添加水印，允许延迟2秒
7. 导入`import org.apache.flink.table.api.scala._`隐式参数
8. 使用`registerDataStream`注册表，并分别指定字段，还要指定rowtime字段
9. 编写SQL语句统计用户订单总数、最大金额、最小金额
   - 分组时要使用`tumble(时间列, interval '窗口时间' second)`来创建窗口
10. 使用`tableEnv.sqlQuery`执行sql语句
11. 将SQL的执行结果转换成DataStream再打印出来
12. 启动流处理程序



**参考代码**

```scala
  // 3. 创建一个订单样例类`Order`，包含四个字段（订单ID、用户ID、订单金额、时间戳）
  case class Order(orderId:String, userId:Int, money:Long, timestamp:Long)

  def main(args: Array[String]): Unit = {
    // 1. 创建流处理运行环境
    val env = StreamExecutionEnvironment.getExecutionEnvironment

    //   2. 设置处理时间为`EventTime`
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

    val tableEnv = TableEnvironment.getTableEnvironment(env)


    // 4. 创建一个自定义数据源
    val orderDataStream = env.addSource(new RichSourceFunction[Order] {
      override def run(ctx: SourceFunction.SourceContext[Order]): Unit = {
        //   - 随机生成订单ID（UUID）
        // - 随机生成用户ID（0-2）
        // - 随机生成订单金额（0-100）
        // - 时间戳为当前系统时间
        // - 每隔1秒生成一个订单
        for (i <- 0 until 1000) {
          val order = Order(UUID.randomUUID().toString, Random.nextInt(3), Random.nextInt(101), System.currentTimeMillis())
          TimeUnit.SECONDS.sleep(1)
          ctx.collect(order)
        }
      }

      override def cancel(): Unit = {}
    })


    // 5. 添加水印，允许延迟2秒
    val watermarkDataStream: DataStream[Order] = orderDataStream.assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarks[Order] {
      var currentTimestamp = 0L
      val delayTime = 2000

      override def getCurrentWatermark: Watermark = {
        new Watermark(currentTimestamp - delayTime)
      }

      override def extractTimestamp(element: Order, previousElementTimestamp: Long): Long = {
        val timestamp = element.timestamp
        currentTimestamp = Math.max(currentTimestamp, timestamp)
        currentTimestamp
      }
    })


    // 6. 导入`import org.apache.flink.table.api.scala._`隐式参数
    //   7. 使用`registerDataStream`注册表，并分别指定字段，还要指定rowtime字段
    tableEnv.registerDataStream("t_order", watermarkDataStream, 'orderId, 'userId, 'money, 'orderDate.rowtime)

    // 8. 编写SQL语句统计用户订单总数、最大金额、最小金额
    // - 分组时要使用`tumble(时间列, interval '窗口时间' second)`来创建窗口
    val sql =
      """
        |select
        | userId,
        | count(1) as totalCount,
        | max(money) as maxMoney,
        | min(money) as minMoney
        |from
        | t_order
        |group by
        | tumble(orderDate, interval '5' second),
        | userId
      """.stripMargin

    // 9. 使用`tableEnv.sqlQuery`执行sql语句
    val table: Table = tableEnv.sqlQuery(sql)

    //   10. 将SQL的执行结果转换成DataStream再打印出来
    table.toRetractStream[Row].print()

    env.execute("StreamSQLApp")
  }
```





> 在SQL语句中，不要将名字取成SQL中的关键字，例如：timestamp。



#### Table转换为DataStream或DataSet

------



**转换为DataSet**

- 直接使用`tableEnv.toDataSet`方法就可以将Table转换为DataSet
- 转换的时候，需要指定泛型，可以是一个样例类，也可以是指定为`Row`类型



**转换为DataStream**

- 使用`tableEnv.toAppendStream`，将表直接附加在流上
- 使用`tableEnv.toRetractStream`，返回一个元组（Boolean, DataStream），Boolean表示数据是否被成功获取
- 转换的时候，需要指定泛型，可以是一个样例类，也可以是指定为`Row`类型



## 窗口/水印

- 源源不断地数据是无法进行统计工作的，因为数据流`没有边界`，无法统计到底有多少数据经过了这个流
- window操作就是在数据流上，截取固定大小的一部分，这个部分是可以统计的
- 截取方式有两种
  - 按照`时间`截取，例如：10秒钟、10分钟统计一次
  - 按照`消息数量`截取，例如：每5个数据、或者50个数据统计一次

![1556117026180](/images/flink/1556117026180.png)



### 窗口

Flink的窗口划分方式分为2种:time/count,即按时间划分和数量划分

##### tumbling-time-window (无重叠数据)

> 1.红绿灯路口会有汽车通过，一共会有多少汽车通过，无法计算。因为车流源源不断，计算没有边界。
>
> 2.统计每15秒钟通过红路灯的汽车数量，第一个15秒为2辆，第二个15秒为3辆，第三个15秒为1辆。。。



![1552238687243](/images/flink/1552238687243.png)

```
发送内容
9,3
9,2
9,7
4,9
2,6
1,5
2,3
5,7
5,4
```

编码:

```scala
  def main(args: Array[String]): Unit = {
    //TODO time-window
    //1.创建运行环境
    val env = StreamExecutionEnvironment.getExecutionEnvironment

    //2.定义数据流来源
    val text = env.socketTextStream("node01", 9000)

    //3.转换数据格式，text->CarWc
    case class CarWc(sensorId: Int, carCnt: Int)
    val ds1: DataStream[CarWc] = text.map {
      line => {
        val tokens = line.split(",")
        CarWc(tokens(0).trim.toInt, tokens(1).trim.toInt)
      }
    }

    //4.执行统计操作，每个sensorId一个tumbling窗口，窗口的大小为5秒
    //也就是说，每5秒钟统计一次，在这过去的5秒钟内，各个路口通过红绿灯汽车的数量。
    val ds2: DataStream[CarWc] = ds1
      .keyBy("sensorId")
      .timeWindow(Time.seconds(5))
      .sum("carCnt")

    //5.显示统计结果
    ds2.print()

    //6.触发流计算
    env.execute(this.getClass.getName)

  }

```

##### sliding-time-window (有重叠数据)

![1552238935817](/images/flink/1552238935817.png)

编码:

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment

//2.定义数据流来源
val text = env.socketTextStream("localhost", 9999)

//3.转换数据格式，text->CarWc
case class CarWc(sensorId: Int, carCnt: Int)
val ds1: DataStream[CarWc] = text.map {
  line => {
    val tokens = line.split(",")
    CarWc(tokens(0).trim.toInt, tokens(1).trim.toInt)
  }
}
//4.执行统计操作，每个sensorId一个sliding窗口，窗口时间10秒,滑动时间5秒
//也就是说，每5秒钟统计一次，在这过去的10秒钟内，各个路口通过红绿灯汽车的数量。
val ds2: DataStream[CarWc] = ds1
  .keyBy("sensorId")
  .timeWindow(Time.seconds(10), Time.seconds(5))
  .sum("carCnt")

//5.显示统计结果
ds2.print()

//6.触发流计算
env.execute(this.getClass.getName)

```

##### tumbling-count-window (无重叠数据)

> 按照个数进行统计，比如：
>
> 每个路口分别统计，收到关于它的5条消息时统计在最近5条消息中，各自路口通过的汽车数量

代码:

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment

//2.定义数据流来源
val text = env.socketTextStream("localhost", 9999)

//3.转换数据格式，text->CarWc
case class CarWc(sensorId: Int, carCnt: Int)
val ds1: DataStream[CarWc] = text.map {
  (f) => {
    val tokens = f.split(",")
    CarWc(tokens(0).trim.toInt, tokens(1).trim.toInt)
  }
}
//4.执行统计操作，每个sensorId一个tumbling窗口，窗口的大小为5
//按照key进行收集，对应的key出现的次数达到5次作为一个结果
val ds2: DataStream[CarWc] = ds1
  .keyBy("sensorId")
  .countWindow(5)
  .sum("carCnt")

//5.显示统计结果
ds2.print()

//6.触发流计算
env.execute(this.getClass.getName)

```

##### sliding-count-window (有重叠数据)

> 同样也是窗口长度和滑动窗口的操作：窗口长度是5，滑动长度是3

编码:

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment

//2.定义数据流来源
val text = env.socketTextStream("localhost", 9999)

//3.转换数据格式，text->CarWc
case class CarWc(sensorId: Int, carCnt: Int)
val ds1: DataStream[CarWc] = text.map {
  (f) => {
    val tokens = f.split(",")
    CarWc(tokens(0).trim.toInt, tokens(1).trim.toInt)
  }
}
//4.执行统计操作，每个sensorId一个sliding窗口，窗口大小3条数据,窗口滑动为3条数据
//也就是说，每个路口分别统计，收到关于它的3条消息时统计在最近5条消息中，各自路口通过的汽车数量
val ds2: DataStream[CarWc] = ds1
  .keyBy("sensorId")
  .countWindow(5, 3)
  .sum("carCnt")

//5.显示统计结果
ds2.print()

//6.触发流计算
env.execute(this.getClass.getName)

```



> 问题:
>
> Flink中的窗口分为两类，一类是按时间来分，一类是按照事件的种类来分。对吗？



> 1. 如果窗口滑动时间 > 窗口时间，会出现数据丢失
> 2. 如果窗口滑动时间 < 窗口时间，会出现数据重复计算,比较适合实时排行榜
> 3. 如果窗口滑动时间 = 窗口时间，数据不会被重复计算

### 水印

#### Flink的时间划分方式

- 事件时间：事件时间是每条事件在它产生的时候记录的时间，该时间记录在事件中，在处理的时候可以被提取出来。小时的时间窗处理将会包含事件时间在该小时内的所有事件，而忽略事件到达的时间和到达的顺序
- 摄入时间：摄入时间是事件进入flink的时间，在source operator中，每个事件拿到当前时间作为时间戳，后续的时间窗口基于该时间。
- 处理时间：当前机器处理该条事件的时间

![1552239345226](/images/flink/1552239345226.png)

> 问题:
>
> ProcessingTime是指的进入到Flink数据流处理系统的时间，对吗？

#### 如何处理水印

> 需求：
>
> 以EventTime划分窗口，计算3秒钟内出价最高的产品

```
1527911155000,boos1,pc1,100.0
1527911156000,boos2,pc1,200.0
1527911157000,boos1,pc1,300.0
1527911158000,boos2,pc1,500.0
1527911159000,boos1,pc1,600.0
1527911160000,boos1,pc1,700.0
1527911161000,boos2,pc2,700.0
1527911162000,boos2,pc2,900.0
1527911163000,boos2,pc2,1000.0
1527911164000,boos2,pc2,1100.0
1527911165000,boos1,pc2,1100.0
1527911166000,boos2,pc2,1300.0
1527911167000,boos2,pc2,1400.0
1527911168000,boos2,pc2,1600.0
1527911170000,boos1,pc2,1300.0
1527911171000,boos2,pc2,1700.0
1527911172000,boos2,pc2,1800.0
1527911173000,boos1,pc2,1500.0

```

代码:

```scala
def main(args: Array[String]) {

    //1.创建执行环境，并设置为使用EventTime
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    //置为使用EventTime
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

    //2.创建数据流，并进行数据转化
    val source = env.socketTextStream("localhost", 9999)
    case class SalePrice(time: Long, boosName: String, productName: String, price: Double)
    val dst1: DataStream[SalePrice] = source.map(value => {
      val columns = value.split(",")
      SalePrice(columns(0).toLong, columns(1), columns(2), columns(3).toDouble)
    })

    //3.使用EventTime进行求最值操作
    val dst2: DataStream[SalePrice] = dst1
      //提取消息中的时间戳属性
      .assignAscendingTimestamps(_.time)
      .keyBy(_.productName)
      .timeWindow(Time.seconds(3))//设置window方法一
      .max("price")

    //4.显示结果
    dst2.print()

    //5.触发流计算
    env.execute()
  }
}

```

> 当前代码理论上看没有任何问题，在实际使用的时候就会出现很多问题,甚至接收不到数据或者接收到的数据是不准确的；这是因为对于flink最初设计的时候，就考虑到了网络延迟，网络乱序等问题，所以提出了一个抽象概念水印（WaterMark）

![1552239660524](/images/flink/1552239660524.png)

水印分成两种形式：

![1552239700782](/images/flink/1552239700782.png)

![1552239709601](/images/flink/1552239709601.png)

代码中就需要添加水印操作:

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
    val timestamps_data = dst1.assignTimestampsAndWatermarks(new AssignerWithPeriodicWatermarks[SalePrice]{

      var currentMaxTimestamp:Long = 0
      val maxOutOfOrderness = 2000L //最大允许的乱序时间是2s
      var wm : Watermark = null
      val format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS")
      override def getCurrentWatermark: Watermark = {
        wm = new Watermark(currentMaxTimestamp - maxOutOfOrderness)
        wm
      }

      override def extractTimestamp(element: SalePrice, previousElementTimestamp: Long): Long = {
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



## 容错

批处理系统比较容易实现容错机制，由于文件可以重复访问，当某个任务失败后，重启该任务即可。但是在流处理系统中，由于数据源是无限的数据流，一个流处理任务甚至可能会执行几个月，将所有数据缓存或是持久化，留待以后重复访问基本上是不可行的。

Checkpoint是Flink实现容错机制最核心的功能，它能够根据配置周期性地基于Stream中各个Operator的状态来生成Snapshot，从而将这些状态数据定期持久化存储下来，当Flink程序一旦意外崩溃时，重新运行程序时可以有选择地从这些Snapshot进行恢复，从而修正因为故障带来的程序数据状态中断。

### Checkpoint

![1552240900014](/images/flink/1552240900014.png)

Checkpoint流程

1. CheckpointCoordinator周期性的向该流应用的所有source算子发送barrier。 
2. 当某个source算子收到一个barrier时，会向自身所有下游算子广播该barrier，同时将自己的当前状态制作成快照(**异步**)，并保存到指定的持久化存储中，最后向CheckpointCoordinator报告自己快照制作情况
3. 下游算子收到barrier之后，会向自身所有下游算子广播该barrier，同时将自身的相关状态制作成快照(**异步**)，并保存到指定的持久化存储中，最后向CheckpointCoordinator报告自身快照情况
4. 每个算子按照步骤3不断制作快照并向下游广播，直到最后barrier传递到sink算子，快照制作完成。 
5. 当CheckpointCoordinator收到所有算子的报告之后，认为该周期的快照制作成功; 否则，如果在规定的时间内没有收到所有算子的报告，则认为本周期快照制作失败 



#### 单流的barrier

1. 屏障作为数据流的一部分随着记录被注入到数据流中。屏障永远不会赶超通常的流记录，它会严格遵循顺序。
2. 屏障将数据流中的记录隔离成一系列的记录集合，并将一些集合中的数据加入到当前的快照中，而另一些数据加入到下一个快照中。
3. 每一个屏障携带着快照的ID，快照记录着ID并且将其放在快照数据的前面。
4. 屏障不会中断流处理，因此非常轻量级。

![1552241036506](/images/flink/1552241036506.png)

#### 并行barrier

1. 不止一个输入流的时的operator，需要在快照屏障上对齐(align)输入流，才会发射出去。 
2. 可以看到1,2,3会一直放在Input buffer，直到另一个输入流的快照到达Operator。

![1552241154668](/images/flink/1552241154668.png)

> 问题:
>
> Flink中Barrier的对齐指的是Flink处理数据流的时候，会加入barrier，某一个operator接收到一个barriern，会等到接收到所有数据流的barrier，才继续往下处理。这样可以实现数据Exatly Once语义。对吗？
>
> 一个Operator处理完数据流后，会将数据流中的barrier删除，这样可以减少处理的数据量，提高运行效率。对吗？



### 持久化存储

#### MemoryStateBackend

state数据保存在java堆内存中，执行checkpoint的时候，会把state的快照数据保存到jobmanager的内存中 基于内存的state backend在生产环境下不建议使用。

#### FsStateBackend

state数据保存在taskmanager的内存中，执行checkpoint的时候，会把state的快照数据保存到配置的文件系统中，可以使用hdfs等分布式文件系统。

#### RocksDBStateBackend

基于RocksDB + FS

RocksDB跟上面的都略有不同，它会在本地文件系统中维护状态，state会直接写入本地rocksdb中。同时RocksDB需要配置一个远端的filesystem。

代码:

```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
// start a checkpoint every 1000 ms
//开启Flink的Checkpoint
env.enableCheckpointing(5000)
// advanced options:
// 设置checkpoint的执行模式，最多执行一次或者至少执行一次
env.getCheckpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE)
// 设置checkpoint的超时时间
env.getCheckpointConfig.setCheckpointTimeout(60000)
// 如果在只做快照过程中出现错误，是否让整体任务失败：true是  false不是
env.getCheckpointConfig.setFailTasksOnCheckpointingErrors(false)
//设置同一时间有多少 个checkpoint可以同时执行 
env.getCheckpointConfig.setMaxConcurrentCheckpoints(1)
//设置checkpoint路径
    env.setStateBackend(new FsStateBackend("hdfs://node01:8020/flink_checkpoint0000000"))

```





