---
title: Flink的基础
tags: Flink
categories: Flink
abbrlink: 7486
date: 2020-09-12 21:36:33
summary_img:
encrypt:
enc_pwd:
---

## 一 Flink的基础上手

### 1 maven依赖

```xml
	<properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <encoding>UTF-8</encoding>
        <scala.version>2.11.8</scala.version>
        <scala.compat.version>2.11</scala.compat.version>
        <hadoop.version>2.6.0</hadoop.version>
        <flink.version>1.10.0</flink.version>
        <spark.version>2.2.0</spark.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>${scala.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.10</version>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.hbase</groupId>
            <artifactId>hbase-client</artifactId>
            <version>2.0.0</version>
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
            <artifactId>flink-table-planner_2.11</artifactId>
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
<!--        <sourceDirectory>src/main/scala</sourceDirectory>-->
<!--        <testSourceDirectory>src/test/scala</testSourceDirectory>-->
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
                                    <mainClass>com.batch.WordCount</mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>

                </executions>
            </plugin>
        </plugins>
    </build>

```

### 2 批处理的wc

```scala
def main(args: Array[String]): Unit = {
  //获取执行环境
  val env: ExecutionEnvironment = ExecutionEnvironment.
   //读取文件
  val ds: DataSet[String] = env.readTextFile("path/wc.txt")
  val result: AggregateDataSet[(String, Int)] = ds.flatMap(_.split(" "))
    .map((_, 1))
    .groupBy(0)//0代表下标
    .sum(1)//1代表下标
  result.print()
}
```

2 流处理的wc

```scala
def main(args: Array[String]): Unit = {
  
  val env = StreamExecutionEnvironment.getExecutionEnvironment
   //从外部命令中获取参数 flink封装的有  格式 --host localhost --port 7777
  val paramTool:ParameterTool = ParameterTool.fromArgs(args)
  val host:String = paramTool.get("host")
  val port:String = paramTool.getInt("port")
  //env.setParallelism(8) 设置全局并行度 ,且每个算子都可单独设置并行度
  val stream: DataStream[String] = env.socketTextStream(host,port)
  val result: DataStream[(String, Int)] = stream.flatMap(_.split(" ")).setParallelism(1)
    .map((_, 1)).setParallelism(1)
    //流式的没有groupBy 担忧keyby
    .keyBy(0)//.setParallelism(1)
    .sum(1).setParallelism(1)
  result.print()//.setParallelism(1)//可以将数据写到一个文件中
  env.execute("Runing")//运行任务的名称
}
```

```
运行结果:
3> (flink,1)
2> (hadoop,2)
```

若是不设置并行度,默认的并行度是当前机器的核数

运行结果的2>,3>代表并行度(当前结果在哪个子任务中执行),默认是按照key做hash计算出来,这样相同的key会在同一个子任务中运行,可以避免各个子任务间拉取数据

```
当只在print的时候设置.setParallelism(1) 并行度为1,之前的不进行设置,则输入的数据顺序,和输出的结果数据的顺序可能不一致,因为前面几个算子的默认并行度为核数(我的4)
flink--->		map1  sum1
hapoop--->		map2  sum1
								print(并行度1)
flume--->		map3  sum1
kettle--->		map4  sum1
这样处理数据因为网络延迟等原因,有些sum算的快,有些慢,导致进入print的数据是乱序的
```

## 二 flink的部署

### 2.1 Standalone模式

