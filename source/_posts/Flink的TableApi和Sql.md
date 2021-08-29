---
title: Flink的TableApi和Sql
tags: Flink
categories: Flink
abbrlink: 39601
date: 2020-12-05 18:10:32
summary_img:
encrypt:
enc_pwd:
---

## 一 Table Api和SQL

Flink本身是批流统一的处理框架，所以Table API和SQL，就是批流统一的上层处理API。

目前功能尚未完善，处于活跃的开发阶段。

Table API是一套内嵌在Java和Scala语言中的查询API，它允许我们以非常直观的方式，组合来自一些关系运算符的查询（比如select、filter和join）。而对于Flink SQL，就是直接可以在代码中写SQL，来实现一些查询（Query）操作。Flink的SQL支持，基于实现了SQL标准的Apache Calcite（Apache开源SQL解析工具）。

无论输入是批输入还是流式输入，在这两套API中，指定的查询都具有相同的语义，得到相同的结果。

依赖：Table API和SQL需要引入的依赖有两个：planner和bridge。

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-planner_2.11</artifactId>
    <version>1.10.0</version>
</dependency>
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table-api-scala-bridge_2.11</artifactId>
    <version>1.10.0</version>
</dependency>
```

## 两种planner（old & blink）的区别

1:批流统一：Blink将批处理作业，视为流式处理的特殊情况。所以，blink不支持表和DataSet之间的转换，批处理作业将不转换为DataSet应用程序，而是跟流处理一样，转换为DataStream程序来处理。

2:因为批流统一，Blink planner也不支持BatchTableSource，而使用有界的StreamTableSource代替。

3:Blink planner只支持全新的目录，不支持已弃用的ExternalCatalog。

4:旧planner和Blink planner的FilterableTableSource实现不兼容。旧的planner会把PlannerExpressions下推到filterableTableSource中，而blink planner则会把Expressions下推。

5:基于字符串的键值配置选项仅适用于Blink planner。

6:PlannerConfig在两个planner中的实现不同。

7:Blink planner会将多个sink优化在一个DAG中（仅在TableEnvironment上受支持，而在StreamTableEnvironment上不受支持）。而旧planner的优化总是将每一个sink放在一个新的DAG中，其中所有DAG彼此独立。

8:旧的planner不支持目录统计，而Blink planner支持。

## 二 API调用

Table API 和 SQL 的程序结构，与流式处理的程序结构类似；也可以近似地认为有这么几步：首先创建执行环境，然后定义source、transform和sink。

```java
StreamTableEnvironment  tableEnv = ...     // 创建表的执行环境
// 创建一张表，用于读取数据
tableEnv.connect(...).createTemporaryTable("inputTable");
// 注册一张表，用于把计算结果输出
tableEnv.connect(...).createTemporaryTable("outputTable");
// 通过 Table API 查询算子，得到一张结果表
Table result = tableEnv.from("inputTable").select(...);
// 通过 SQL查询语句，得到一张结果表
Table sqlResult  = tableEnv.sqlQuery("SELECT ... FROM inputTable ...");
// 将结果表写入输出表中
result.insertInto("outputTable");
```



## 三 流处理中的特殊概念



## 四 窗口



## 五 函数

