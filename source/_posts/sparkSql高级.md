---
title: sparkSql高级
tags:
  - Spark
categories:
  - Spark
abbrlink: 21375
date: 2018-04-12 09:38:17
summary_img:
---

# 一聚合操作

1. `groupBy`
2. `rollup`
3. `cube`
4. `pivot`
5. `RelationalGroupedDataset` 上的聚合操作

## 1自定义udf与udaf

````scala
package com.nicai.www


import org.apache.spark.sql.expressions.{MutableAggregationBuffer, UserDefinedAggregateFunction}
import org.apache.spark.sql.types._
import org.apache.spark.sql.{Dataset, Row, SparkSession}
import org.junit.Test

/**
  * Created by 春雨里洗过的太阳
  */
class Udf extends Serializable {

  private val spark: SparkSession = new SparkSession.Builder()
    .master("local[3]")
    .appName("act")
    .getOrCreate()

  import spark.implicits._

  //自定义udf  需求 增加一列  值为第二列中 (1995)
  @Test
  def udfDemo(): Unit = {
    val ds: Dataset[String] = spark.read
      .text("G:\\develop\\data\\movies.dat")
      .as[String]
    val unit: Dataset[(Long, String, String)] = ds.map(it => {
      val arr: Array[String] = it.split("::")
      (arr(0).toLong, arr(1), arr(2))
    })
    val frame = unit.toDF("id", "title", "gen")
    import org.apache.spark.sql.functions._
    //自定以udf
    def zudf(title: String): String = {
      val pattern = "(?<=\\s\\()\\d{4}(?=\\))".r
      pattern.findFirstIn("Sabrina (1995)").getOrElse(" ")
    }

    //注册udf
    val udf2 = udf(zudf _) //这个_ 是方法和函数的转换
    frame.withColumn("year", udf2('title))
      .show()
  }


  //自定义udaf  需求
  /*给定数据集 Dataset(userId: String, tags: Map(Tag, Weight))

  求每个用户的标签聚合字符串 "tag1:weight1,tag2:weight2"*/
  @Test
  def udafMain(): Unit = {
    val source = Seq(
      ("user1", Map("tag1" -> 1.0, "tag2" -> 2.0)),
      ("user1", Map("tag1" -> 1.5, "tag3" -> 2.0, "tag4" -> 2.1)),
      ("user2", Map("tag1" -> 1.5, "tag4" -> 2.1))
    ).toDF("id", "tags")
    source.groupBy("id")
      .agg(udafDemo2('id, 'tags) as "tags")
      .show()

  }

}
//自定义udaf
object udafDemo2 extends UserDefinedAggregateFunction {
  //聚合函数的输入数据结构
  override def inputSchema: StructType = {
    new StructType()
      .add("id", StringType) //就是数据的ID
      .add("tags", MapType(StringType, DoubleType)) //就是数据的Map
  }

  //  缓存区数据结构
  override def bufferSchema: StructType = {
    new StructType()
      .add("tags", MapType(StringType, DoubleType)) //要处理的 数据列
  }

  //聚合函数返回值数据结构
  override def dataType: DataType = StringType

  //聚合函数是否是幂等的  即相同输入是否总能的到相同的输出
  override def deterministic: Boolean = true

  //初始化缓冲区
  override def initialize(buffer: MutableAggregationBuffer): Unit = {
    buffer(0) = Map[String, Double]()
  }

  //给聚合函数传入一条新数据进行处理
  override def update(buffer: MutableAggregationBuffer, input: Row): Unit = {
    buffer(0) = mergeTagMap(buffer.getAs[Map[String, Double]](0), input.getAs[Map[String, Double]](1))
  }

  def mergeTagMap(m1: Map[String, Double], m2: Map[String, Double]): Map[String, Double] = {
    val merge: Map[String, Double] = m1.map {
      case (key, value) => if (m2.contains(key)) (key, value + m2(key)) else (key, value)
    }
    m2 ++ merge
  }

  //合并聚合函数缓冲区
  override def merge(buffer1: MutableAggregationBuffer, buffer2: Row): Unit = {
    buffer1(0) = mergeTagMap(buffer1.getAs[Map[String, Double]](0), buffer2.getAs[Map[String, Double]](0))
  }

  //计算最终结果
  override def evaluate(buffer: Row): Any = {
    val result = buffer.getAs[Map[String, Double]](0)
    result.map {
      case (key, value) => key + ":" + value
    }
      .mkString(",")
  }
}


````

## 2 聚合

`groupBy` 算子会按照列将 `Dataset` 分组, 并返回一个 `RelationalGroupedDataset` 对象, 通过 `RelationalGroupedDataset` 可以对分组进行聚合

```scala
package com.nicai.www

import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.types.{DoubleType, IntegerType, StructField, StructType}
import org.junit.Test

/**
  * Created by 春雨里洗过的太阳
  */
class Juhe {
  private val spark: SparkSession = new SparkSession.Builder()
    .master("local[3]")
    .appName("act")
    .getOrCreate()

  import spark.implicits._
  import org.apache.spark.sql.functions._

  //使用 functions 函数进行聚合
  @Test
  def groupBy(): Unit = {
    val schema = StructType(
      List(
        StructField("id", IntegerType),
        StructField("year", IntegerType),
        StructField("month", IntegerType),
        StructField("day", IntegerType),
        StructField("hour", IntegerType),
        StructField("season", IntegerType),
        StructField("pm", DoubleType)
      )
    )
    val frame = spark.read
      .schema(schema)
      .option("header", true)
      .csv("G:\\develop\\bigdatas\\BigData\\day29sparksql2\\data\\pm_final.csv")
    //分组
    val dataset = frame.groupBy('year)
    //聚合
    dataset.agg(avg('pm) as "avg_pm")
      .orderBy('avg_pm)
      .show()


  }

  // 除了使用 functions 进行聚合, 还可以直接使用 RelationalGroupedDataset 的 API 进行聚合
  @Test
  def groupBy2(): Unit = {
    val schema = StructType(
      List(
        StructField("id", IntegerType),
        StructField("year", IntegerType),
        StructField("month", IntegerType),
        StructField("day", IntegerType),
        StructField("hour", IntegerType),
        StructField("season", IntegerType),
        StructField("pm", DoubleType)
      )
    )
    val frame = spark.read
      .schema(schema)
      .option("header", true)
      .csv("G:\\develop\\data\\pm_final.csv")
    //分组
    val dataset = frame.groupBy('year)
    //聚合
    dataset.avg("pm")
      .orderBy("avg(pm)")
      .show()

    dataset.max("pm")
      .show()
  }
}

```

## 3 多维聚合

我们可能经常需要针对数据进行多维的聚合, 也就是一次性统计小计, 总计等, 一般的思路如下

```scala
class DuoWeiJuHe {
  private val spark: SparkSession = new SparkSession.Builder()
    .master("local[3]")
    .appName("act")
    .getOrCreate()

  import spark.implicits._
  import org.apache.spark.sql.functions._

  //使用groupBy 多维聚合
  @Test
  def groupByDuo(): Unit = {
    val schema = StructType(
      List(
        StructField("source", StringType),
        StructField("year", IntegerType),
        StructField("month", IntegerType),
        StructField("day", IntegerType),
        StructField("hour", IntegerType),
        StructField("season", IntegerType),
        StructField("pm", DoubleType)
      )
    )
    val frame = spark.read
      .schema(schema)
      .option("header", true)
      .csv("G:\\develop\\bigdatas\\BigData\\day29sparksql2\\data\\pm_final.csv")

    //需求一 按地区 年份 求平均值
    val sourceandYear = frame.groupBy('source, 'year)
    val sY = sourceandYear.agg(avg('pm) as "avg_pm")

    //需求二  按 地区 求平均值
    val sour = frame.groupBy('source)
    val res = sour.agg(avg('pm) as "avg_pm")
      .select('source, lit(null) as "year", 'avg_pm)

    //聚合
    val result = sY.union(res)
      .sort('source, 'year asc_nulls_last, 'avg_pm)
    result.show()
  }

  //使用rollup 多维聚合
  @Test
  def groupByDuo2(): Unit = {
    val schema = StructType(
      List(
        StructField("source", StringType),
        StructField("year", IntegerType),
        StructField("month", IntegerType),
        StructField("day", IntegerType),
        StructField("hour", IntegerType),
        StructField("season", IntegerType),
        StructField("pm", DoubleType)
      )
    )
    val frame = spark.read
      .schema(schema)
      .option("header", true)
      .csv("G:\\develop\\bigdatas\\BigData\\day29sparksql2\\data\\pm_final.csv")

    frame.rollup('source, 'year)
      .agg(avg("pm"))
      .sort('source, 'year.desc_nulls_last)
      .show()
  }

  //多维聚合 cube
  @Test
  def groupByDuo3(): Unit = {
    val schema = StructType(
      List(
        StructField("source", StringType),
        StructField("year", IntegerType),
        StructField("month", IntegerType),
        StructField("day", IntegerType),
        StructField("hour", IntegerType),
        StructField("season", IntegerType),
        StructField("pm", DoubleType)
      )
    )
    val frame = spark.read
      .schema(schema)
      .option("header", true)
      .csv("G:\\develop\\bigdatas\\BigData\\day29sparksql2\\data\\pm_final.csv")

    frame.cube('source, 'year)
      .agg(avg("pm"))
      .sort('source, 'year.desc_nulls_last)
      .show()
  }
}
```

## 4 连接 与广播连接

连接分为两种 

1. 无类型连接 `join`
2. 连接类型 `Join Types`

```properties
1  cross 交叉连接 笛卡尔积  全连接* 交叉连接是一个非常重的操作, 在生产中, 尽量不要将两个大数据集交叉连接, 如果一定要交叉连接, 也需要在交叉连接后进行过滤, 优化器会进行优化
2  inner  内连接 只存在有关联的数据
3  outer full fullouter  全外连接
4  leftouter  left  左外连接  包含两侧连接上的数据和左侧没有连接上的数据
5  leftanti   是一种特殊的连接形式和左外连接类似,但是其结果集中没有右侧的数据,只包含左边集合中没连接上的数据
6  leftsemi   和 LeftAnti 恰好相反, LeftSemi 的结果集也没有右侧集合的数据, 但是只包含左侧集合中连接上的数据
7  rightouter  right  右外连接  与左连接相反 包含两侧连接上的数据 和右侧没有连接上的数据
8  广播连接  解决join的性能问题   应为join是一个shuffle操作  是一个小文件的广播操作
```

```scala
@Test
  def leftanti(): Unit = {
    val person = Seq((0, "Lucy", 0), (1, "Lily", 0), (2, "Tim", 2), (3, "Danial", 3))
      .toDF("id", "name", "cityId")
    person.createOrReplaceTempView("person") //创建临时表

    val cities = Seq((0, "Beijing"), (1, "Shanghai"), (2, "Guangzhou"))
      .toDF("id", "name")
    cities.createOrReplaceTempView("cities") //创建临时表

    person.join(cities, person.col("cityId") === cities.col("id"), "leftanti")
      .show()
  }
```

广播连接

`Join` 会在集群中分发两个数据集, 两个数据集都要复制到 `Reducer` 端, 是一个非常复杂和标准的 `ShuffleDependency`

优化:

`Map` **端** `Join`   之所以说它效率很低, 原因是需要在集群中进行数据拷贝, 如果能减少数据拷贝, 就能减少开销

可以将小数据集收集起来, 分发给每一个 `Executor`, 然后在需要 `Join` 的时候, 让较大的数据集在 `Map` 端直接获取小数据集, 从而进行 `Join`, 这种方式是不需要进行 `Shuffle` 的, 所以称之为 `Map` 端 `Join`

正常join图:

![img](/images/sparksql/join.png)

开启mapjoin图

![img](/images/sparksql/join2.png)

```scala
//广播连接
  @Test
  def leftanti2(): Unit = {
    val personRDD = spark.sparkContext.parallelize(Seq((0, "Lucy", 0),
      (1, "Lily", 0), (2, "Tim", 2), (3, "Danial", 3)))

    val citiesRDD = spark.sparkContext.parallelize(Seq((0, "Beijing"),
      (1, "Shanghai"), (2, "Guangzhou")))
    //设置广播
    val guangbo = spark.sparkContext.broadcast(citiesRDD.collectAsMap())
    //进行操作
    val tuples = personRDD.mapPartitions(
      iter => {
        val value = guangbo.value
        val result = for (person <- iter if (value.contains(person._3)))
          yield (person._1, person._2, value(person._3))
        result
      }
    ).collect()
    tuples.foreach(println(_))

  }

```

### 1 **使用** `Dataset` **实现** `Join` **的时候会自动进行** `Map` **端** `Join`

```scala
自动进行 Map 端 Join 需要依赖一个系统参数 spark.sql.autoBroadcastJoinThreshold, 当数据集小于这个参数的大小时, 会自动进行 Map 端 Join

如下, 开启自动 Join

println(spark.conf.get("spark.sql.autoBroadcastJoinThreshold").toInt / 1024 / 1024)
println(person.crossJoin(cities).queryExecution.sparkPlan.numberedTreeString)

当关闭这个参数的时候, 则不会自动 Map 端 Join 了
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", -1)
println(person.crossJoin(cities).queryExecution.sparkPlan.numberedTreeString)
```

### 2 **也可以使用函数强制开启 Map 端 Join**

```scala
在使用 Dataset 的 join 时, 可以使用 broadcast 函数来实现 Map 端 Join

import org.apache.spark.sql.functions._
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", -1)
println(person.crossJoin(broadcast(cities)).queryExecution.sparkPlan.numberedTreeString)
即使是使用 SQL 也可以使用特殊的语法开启

spark.conf.set("spark.sql.autoBroadcastJoinThreshold", -1)
val resultDF = spark.sql(
  """
    |select /*+ MAPJOIN (rt) */ * from person cross join cities rt
  """.stripMargin)
println(resultDF.queryExecution.sparkPlan.numberedTreeString)
```

## 5 窗口函数

```
窗口函数  解决了  在一个大的数据集里 进行分组 求前几个等需求
** 窗口操作分为两个部分**   
1窗口定义, 定义时可以指定 Partition, Order, Frame**   
2函数操作, 可以使用三大类函数, 排名函数, 分析函数, 聚合函数
```

```scala
package com.nicai.www

import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.expressions.Window
import org.junit.Test

/**
  * Created by 春雨里洗过的太阳 on 2019/8/20
  *
  * 窗口函数  解决了  在一个大的数据集里 进行分组 求前几个等需求
  *
  * 窗口操作分为两个部分
  *
  *   窗口定义, 定义时可以指定 Partition, Order, Frame
  *
  *   函数操作, 可以使用三大类函数, 排名函数, 分析函数, 聚合函数
  */
class WindowsDemo {
  val spark = SparkSession.builder()
    .appName("window")
    .master("local[6]")
    .getOrCreate()

  import spark.implicits._
  import org.apache.spark.sql.functions._
//需求 求每个类别(第二列)的其中的值与这个类别中最大值的差值
  @Test
  def chazhi(): Unit = {
    val source = Seq(
      ("Thin", "Cell phone", 6000),
      ("Normal", "Tablet", 1500),
      ("Mini", "Tablet", 5500),
      ("Ultra thin", "Cell phone", 5500),
      ("Very thin", "Cell phone", 6000),
      ("Big", "Tablet", 2500),
      ("Bendable", "Cell phone", 3000),
      ("Foldable", "Cell phone", 3000),
      ("Pro", "Tablet", 4500),
      ("Pro2", "Tablet", 6500)
    ).toDF("product", "category", "rev")

    val win = Window.partitionBy('category)
      .orderBy('rev.desc)
    source.select(
      'product,'category,'rev,(max('rev) over win )-'rev as "rev_diff"
    ).show()
    /*可以作用于 窗口函数的 函数 (max的位置)
    * 排名函数: rank排名函数, 计算当前数据在其 Frame 中的位置

                    如果有重复, 则重复项后面的行号会有空挡
      * dense_rank  和 rank 一样, 但是结果中没有空挡
      *
      * row_number   和 rank 一样, 也是排名, 但是不同点是即使有重复想, 排名依然增长
    *  分析函数:
    *
    *       first_value  获得这个租的第一条数据
    *       last_value  获得这个租的最后 一条数据
    *       lag     lag(field, n) 获取当前数据的 field 列向前 n 条数据
    *       lead     lead(field, n) 获取当前数据的 field 列向后 n 条数据
    *聚合函数   所有
    * */

  }


  @Test
  def partTopN(): Unit ={
    val source = Seq(
      ("Thin", "Cell phone", 6000),
      ("Normal", "Tablet", 1500),
      ("Mini", "Tablet", 5500),
      ("Ultra thin", "Cell phone", 5500),
      ("Very thin", "Cell phone", 6000),
      ("Big", "Tablet", 2500),
      ("Bendable", "Cell phone", 3000),
      ("Foldable", "Cell phone", 3000),
      ("Pro", "Tablet", 4500),
      ("Pro2", "Tablet", 6500)
    ).toDF("product", "category", "rev")

    val win = Window.partitionBy('category)
      .orderBy('rev.desc)
    source.select(
      'product,'category,'rev,rank() over win  as "rank"
    )
      .where('rank <= 2)
      .show()
  }
}

```

- 问题1: `Spark` 和 `Hive` 这样的系统中, 有自增主键吗? 没有
- 问题2: 为什么分布式系统中很少见自增主键? 因为分布式环境下数据在不同的节点中, 很难保证顺序

- 解决方案: 按照某一列去排序, 取前两条数据
- 遗留问题: 不容易在分组中取每一组的前两个

**窗口函数在sql中的语法**

~~~sql
SELECT
  product,
  category,
  revenue
FROM (
  SELECT
    product,
    category,
    revenue,
    dense_rank() OVER (PARTITION BY category ORDER BY revenue DESC) as rank
  FROM productRevenue) tmp
WHERE
  rank <= 2
~~~

窗口函数在 `SQL` 中的完整语法如下

```sql
function OVER (PARITION BY ... ORDER BY ... FRAME_TYPE BETWEEN ... AND ...)
```

在 `Spark` 中, 使用 `SQL` 或者 `DataFrame` 都可以操作窗口

窗口函数和 `GroupBy` 最大的区别, 就是 `GroupBy` 的聚合对每一个组只有一个结果, 而窗口函数可以对每一条数据都有一个结果

说白了, 窗口函数其实就是根据当前数据, 计算其在所在的组中的统计数据

- 窗口函数会针对 **每一个组中的每一条数据** 进行统计聚合或者 `rank`, 一个组又称为一个 `Frame`
- 分组由两个字段控制, `Partition` 在整体上进行分组和分区
- 而通过 `Frame` 可以通过 **当前行** 来更细粒度的分组控制