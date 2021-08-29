---
title: SparkSql随笔
tags:
  - Spark
categories: Spark
encrypt: 
enc_pwd: 
abbrlink: 46696
date: 2020-04-12 16:40:36
summary_img:
---

# 一 spark的Rdd,DF,DS的转换及用法

## 1、三者的区别与联系

三者发展历程:

RDD(spark1.0)  ===>  DataFrame(spark1.3)  ===> DataSet(spark1.6)

大概可以这么说:

rdd + 表结构 = df

rdd + 表结构 + 数据类型 = ds

df + 数据类型 = ds

共性:

1）都是spark中得弹性分布式数据集，轻量级

2）都是惰性机制，延迟计算

3）根据内存情况，自动缓存，加快计算速度

4）都有partition分区概念

5）众多相同得算子：map flatmap 等等

区别：

1）RDD不支持SQL

2）DF每一行都是Row类型，不能直接访问字段，必须解析才行

3）DS每一行是什么类型是不一定的，在自定义了case class之后可以很自由的获 得每一行的信息

4）DataFrame与Dataset均支持spark sql的操作，比如select，group by之类，还 能注册临时表/视窗，进行sql语句操作

5）可以看出，Dataset在需要访问列中的某个字段时是非常方便的，然而，如果要 写一些适配性很强的函数时，如果使用Dataset，行的类型又不确定，可能是 各种case class，无法实现适配，这时候用DataFrame即Dataset[Row]就能比较 好的解决问题。

6) rdd的优缺点:

**- 优点:** 编译时类型安全 编译时就能检查出类型错误 面向对象的编程风格 直接通过类名点的方式来操作数据
**- 缺点:** 序列化和反序列化的性能开销 无论是集群间的通信, 还是IO操作都需要对对象的结构和数据进行序列化和反序列化 

**GC的性能开销**

频繁的创建和销毁对象, 势必会增加GC

7) DF和DS的优缺点

DataFrame引入了schema和off-heap
- schema : RDD每一行的数据, 结构都是一样的.
  这个结构就存储在schema中. Spark通过schame就能够读懂数据, 因此在通信和IO时就只需要序列化和反序列化数据,
  而结构的部分就可以省略了.
- off-heap : 意味着JVM堆以外的内存,
  这些内存直接受操作系统管理（而不是JVM）。Spark能够以二进制的形式序列化数据(不包括结构)到off-heap中, 当要操作数据时,
  就直接操作off-heap内存. 由于Spark理解schema, 所以知道该如何操作 其API不是面向对象的
  这里我们就可以看出spark为了解决RDD的问题进行的取舍

RDD是分布式的Java对象的集合。DataFrame是分布式的Row对象的集合
DataFrame除了提供了比RDD更丰富的算子以外，更重要的特点是提升执行效率、减少数据读取以及执行计划的优化，比如filter下推、裁剪等
Dataset和DataFrame拥有完全相同的成员函数，区别只是每一行的数据类型不同
DataFrame也可以叫Dataset[Row],每一行的类型是Row，不解析，每一行究竟有哪些字段，各个字段又是什么类型都无从得知，只能用getAS方法或者共性中的第七条提到的模式匹配拿出特定字段
而Dataset中，每一行是什么类型是不一定的，在自定义了case class之后可以很自由的获得每一行的信息

## 2三者的转化

1）DF/DS转RDD

1. Val Rdd = DF/DS.rdd

2) DS/RDD转DF

1. import spark.implicits._
2. 调用 toDF（就是把一行数据封装成row类型）
3. sparkSession.createDataFrame(rdd,schema)  toDF报错  可以吧rdd转df

3）RDD转DS

将RDD的每一行封装成样例类，再调用toDS方法

4）DF转DS

根据row字段定义样例类，再调用asDS方法[样例类]

或者df.as[case class]

特别注意：

在使用一些特殊的操作时，一定要加上 import spark.implicits._ 不然toDF、toDS无法使用

## 3 三者的性能比较

计数时 DS>RDD>DF  (其他计算不一定  具体看使用的)

````scala
package com.huawei.spark.areaRoadFlow

import java.util.UUID

import org.apache.spark.rdd.RDD
import org.apache.spark.sql.{Dataset, SparkSession}

object Test_DF_DS_RDD_Speed {
  def main(args: Array[String]): Unit = {
    val spark: SparkSession = SparkSession.builder().appName("无测试").master("local").getOrCreate()
    spark.sparkContext.setLogLevel("ERROR")

    val firstRdd: RDD[(String, Int)] = spark.sparkContext.parallelize(0 to 400000).map(num => {
      (UUID.randomUUID().toString, num)
    })
    firstRdd
    firstRdd.cache()

    val beginTimeRdd: Long = System.currentTimeMillis()
    firstRdd.map(tp=>{tp._1+"-"+tp._2}).collect()
    val endTimeRdd: Long = System.currentTimeMillis()

    import spark.implicits._
    val beginTimeDF: Long = System.currentTimeMillis()
    firstRdd.toDF().map(row=>{row.get(0)+"-"+row.get(1)}).collect()
    val endTimeDF: Long = System.currentTimeMillis()

    val beginTimeDS: Long = System.currentTimeMillis()
    firstRdd.toDS().map(tp=>{tp._1+"-"+tp._2}).collect()
    val endTimeDS: Long = System.currentTimeMillis()

    println(s"RDD算子耗时${endTimeRdd-beginTimeRdd}")
    println(s"DF算子耗时${endTimeDF-beginTimeDF}")
    println(s"DS算子耗时${endTimeDS-beginTimeDS}")
  }
}
````

结果:

```
RDD算子耗时1782
DF算子耗时3071
DS算子耗时460
```

## 4 sparkSession读取不同的数据文件的返回值

```scala
1  val ds:DataSet[String] =sparkSession.read.textFile(s"path")//可以是压缩文件
  jdbc: DataFrame
  text: DataFrame
  load: DataFrame
  csv: DataFrame
  json,parquet,orc,table:DataFrame
```

# 二 SparkSql 解析josn

## 1 常用方法

1.1 读取json文件

```
sparkSession.read.json(path).show()   直接就可以打印表{json为单层}
json为多层的解析
```













#  SparkSql 读取配置文件 解析json并把数据按照类型插入hive

也可解析来的json每一条的字段个数不一致,如:有的有android有的无,但配置文件 中必须每个都有

## 1 配置文件

```
插入数据库的name,json中的name,类型
username,u,String
```

## 2 代码

```scala
import java.text.SimpleDateFormat
import java.util.Locale
import com.google.gson.{JsonObject, JsonParser}
import org.apache.spark.sql.{DataFrame, Dataset, SparkSession}
import scala.collection.JavaConversions._
import scala.collection.mutable
import scala.collection.mutable.{ArrayBuffer, ListBuffer, Map}


object SparkParseFlowJsonTest2 {
  val spark = new SparkSession.Builder()
    .appName("sql")
    .master("local[3]")
    .getOrCreate()
  import spark.implicits._

  val mapType: mutable.Map[String, String] = Map()

  def main(args: Array[String]): Unit = {
    val CONFIG_MAP = readConfig()
    val dataset = spark.read.textFile(s"json文件地址")
    val CONFIG_LIST:List[String] = CONFIG_MAP.keySet.toList
    //print(CONFIG_MAP)

    val dataset2 = dataset.map(re => {
      val mapJson: mutable.Map[String, String] = handleMessage2CaseClass(re)
      var arrayData:ArrayBuffer[String] = ArrayBuffer()
      for ( key <- CONFIG_LIST ){
        var valueOption = mapJson.get(key)
        var value = ""
        if (valueOption != None){
          value=valueOption.get
          if (key.equalsIgnoreCase("_timestamp")){
            val formatter = new SimpleDateFormat("dd/MMM/yyyy:hh:mm:ss Z", Locale.ENGLISH)
            val formatStr = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
            value = formatStr.format(formatter.parse(value))
          }
        }
        arrayData += value
      }
      arrayData
    })
    spark.udf.register("strToMap", (field: String) => strToMapUDF(field))
    var lb: ListBuffer[String] = ListBuffer()
    for( i <- 0 to CONFIG_LIST.length-1){
      var keyname=CONFIG_MAP.get(CONFIG_LIST.get(i)).get
      val str = mapType.get(keyname).toString.replace("Some(","").replace(")","")
      if (str.equalsIgnoreCase("Map")){
        lb = lb :+ f"strToMap(value[$i]) as  $keyname"
      }else{
        lb = lb :+ f"cast(value[$i] as ${str}) as  $keyname"
      }

    }
    val frame: DataFrame = dataset2.selectExpr(lb: _*)
    frame.show()
    frame.printSchema()


  }
  //自定义udf string to map
  def strToMapUDF(field: String): Map[String,String] = {
    val mapJson2: mutable.Map[String, String] = Map()
    val strings = field.split(",")
    for (i <- 0 until strings.length){
      mapJson2.put(strings(i).split(":")(0),strings(i).split(":")(1))
    }
    mapJson2
  }

  //josnToMap
  def handleMessage2CaseClass(jsonStr: String): Map[String,String] = {
    val mapJson: mutable.Map[String, String] = Map()
    val parser = new JsonParser
    val element = parser.parse(jsonStr)
    if(element.isJsonObject) {
      val jsonObject: JsonObject = element.getAsJsonObject()
      val set = jsonObject.entrySet()
      val ite= set.iterator()
      while (ite.hasNext){
        val el = ite.next()
        val key: String = el.getKey
        val value = el.getValue.toString.replace("\"","")
        mapJson.put(key,value)
      }
    }
    mapJson
  }

  //readConfig
  def readConfig(): Map[String,String] ={
    val mapAll: mutable.Map[String, String] = Map()
    val data: Dataset[String] = spark.read.textFile("配置文件地址")
    val stringses: Array[Array[String]] = data.map(x =>
      x.split(",")
    ).collect()

    val iterator: Iterator[Array[String]] = stringses.iterator
    while(iterator.hasNext){
      val arr: Array[String] = iterator.next()
      mapAll.put(arr(1),arr(0))
      mapType.put(arr(0),arr(2))
    }
    mapAll
  }
}
```

