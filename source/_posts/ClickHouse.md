---
title: ClickHouse
abbrlink: 40347
date: 2021-08-29 19:18:34
tags: ClickHouse
categories: ClickHouse
summary_img:
encrypt:
enc_pwd:
---

## 一 简介

​	ClickHouse 是由俄罗斯的Yandex用c++编写的列式存储数据库,主要用于在线分析处理(OLAP),可提供海量数据的存储和分析，同时利用其数据压缩和[向量化](https://so.csdn.net/so/search?q=向量化&spm=1001.2101.3001.7020)引擎的特性，能提供快速的数据搜索。注意到ClickHouse是一个数据库管理系统，而不是单个数据库。不依赖hadoop.但集群版需要依赖于zk

## 二 安装

```shell
20.6.3 之前不支持explain
20.8 加了个引擎能实时同步mysql
本文 21.7.3.14
```

```text
1: 验证系统是否能安装
	grep -q sse4_2 /proc/cpuinfo && echo “SSE 4.2 supported” || echo “SSE 4.2 not supported.
	打印:“SSE 4.2 supported” 就是支持,否则自求多福
2 centos 安装
sudo yum install yum-utils
sudo rpm --import https://repo.clickhouse.com/CLICKHOUSE-KEY.GPG
sudo yum-config-manager --add-repo https://repo.clickhouse.com/rpm/clickhouse.repo
sudo yum install clickhouse-server clickhouse-client
	(yum报错  File "/usr/libexec/urlgrabber-ext-down", line 28
    except OSError, e:
	原因与解决: python版本更改,导致yum不可用 修改usr/libexec/urlgrabber-ext-down头文件为python2
	) 或者升级yum
3 启动
sudo /etc/init.d/clickhouse-server start
clickhouse-client

mac(x86):
wget 'https://builds.clickhouse.com/master/macos/clickhouse'
chmod a+x ./clickhouse
./clickhouse

mac(arm):
wget 'https://builds.clickhouse.com/master/macos-aarch64/clickhouse'
chmod a+x ./clickhouse
./clickhouse
```

## 三 特点

### 1 列式存储

以下面的表为例:

| **Id** | **Name** | **Age** |
| ------ | -------- | ------- |
| 1      | 张三       | 18      |
| 2      | 李四       | 22      |
| 3      | 王五       | 34      |

**1**)采用行式存储时，数据在磁盘上的组织结构为:

```
1 张三 18 2 李四 22 3 王五 34
```

好处是想查某个人所有的属性时，可以通过一次磁盘查找加顺序读取就可以。但是当想 查所有人的年龄时，需要不停的查找，或者全表扫描才行，遍历的很多数据都是不需要的

**2**)采用列式存储时，数据在磁盘上的组织结构为:

```
1 2 3 张三 李四 王五 18 22 34
```

 这时想查所有人的年龄只需把年龄那一列拿出来就可以了

**3**)列式储存的好处:

```
1.对于列的聚合，计数，求和等统计操作原因优于行式存储。
2.由于某一列的数据类型都是相同的，针对于数据存储更容易进行数据压缩，每一列 选择更优的数据压缩算法，大大提高了数据的压缩比重。
3.由于数据压缩比更好，一方面节省了磁盘空间，另一方面对于cache也有了更大的 发挥空间。
```

### 2 DBMS的功能

几乎覆盖了标准 SQL 的大部分语法，包括 DDL 和 DML，以及配套的各种函数，用户管理及权限管理，数据的备份与恢复。

### 3 多样化引擎

​	ClickHouse 和 MySQL 类似，把表级的存储引擎插件化，根据表的不同需求可以设定不同的存储引擎。目前包括合并树、日志、接口和其他四大类 20 多种引擎.

### 4 高吞吐写入能力

​	ClickHouse 采用类 LSM Tree 的结构，数据写入后定期在后台 Compaction。通过类 LSM tree 的结构，ClickHouse 在数据导入时全部是顺序 append 写，写入后数据段不可更改，在后台 compaction 时也是多个段 merge sort 后顺序写回磁盘。顺序写的特性，充分利用了磁盘的吞 吐能力，即便在 HDD 上也有着优异的写入性能。官方公开 benchmark 测试显示能够达到 50MB-200MB/s 的写入吞吐能力，按照每行 100Byte 估算，大约相当于 50W-200W 条/s 的写入速度。

### 5 数据分区域线程级并行

​	ClickHouse 将数据划分为多个 partition，每个 partition 再进一步划分为多个 index granularity(索引粒度)，然后通过多个 CPU 核心分别处理其中的一部分来实现并行数据处理。 在这种设计下，单条 Query 就能利用整机所有 CPU。极致的并行处理能力，极大的降低了查 询延时。所以，ClickHouse 即使对于大量数据的查询也能够化整为零平行处理。但是有一个弊端 就是对于单条查询使用多 cpu，就不利于同时并发多条查询。所以对于高 qps 的查询业务， ClickHouse 并不是强项。MySQL单条SQL是单线程的，只能跑满一个core

### 6 分布式计算

ClickHouse会自动将查询拆解为多个task下发到集群中，然后进行多机并行处理，最后把结果汇聚到一起

### 7 数据有序存储

​	ClickHouse支持在建表时，指定将数据按照某些列进行sort by。排序后，保证了相同sort key的数据在磁盘上连续存储，且有序摆放。在进行等值、范围查询时，where条件命中的数据都紧密存储在一个或若干个连续的Block中，而不是分散的存储在任意多个Block， 大幅减少需要IO的block数量。另外，连续IO也能够充分利用操作系统page cache的预取能力，减少page fault

### 8 灵活多变

​	不适合预先建模 分析场景下，随着业务变化要及时调整分析维度、挖掘方法，以尽快发现数据价值、更新业务指标。而数据仓库中通常存储着海量的历史数据，调整代价十分高昂。预先建模技术虽然可以在特定场景中加速计算，但是无法满足业务灵活多变的发展需求，维护成本过高

### 9 无需事务

数据一致性要求低

### 10 实时数据更新

​	ClickHouse支持具有主键的表。为了在主键范围内快速执行查询，使用合并树对数据进行增量排序。因此，可以将数据连续添加到表中。摄取新数据时不采取任何锁定。

### 11 性能

​	ClickHouse 像很多 OLAP 数据库一样，单表查询速度由于关联查询，而且 ClickHouse 的两者差距更为明显,避免join操作,适用于大宽表

## 四 数据类型

```
1 整形
	整型范围(-2n-1~2n-1-1):
	Int8 - [-128 : 127]
	Int16 - [-32768 : 32767]
	Int32 - [-2147483648 : 2147483647]
	Int64 - [-9223372036854775808 : 9223372036854775807] 无符号整型范围(0~2n-1):
	UInt8 - [0 : 255]
	UInt16 - [0 : 65535]
	UInt32 - [0 : 4294967295]
	UInt64 - [0 : 18446744073709551615]
	使用场景: 个数、数量、也可以存储型 id
2 浮点型
	Float32 - float
	Float64 – double
	建议尽可能以整数形式存储数据。例如，将固定精度的数字转换为整数值，如时间用毫秒为单位表示，因为浮点型进行计算时可能引起四舍五入的误差。
3 布尔型
	没有单独的类型来存储布尔值。可以使用 UInt8 类型，取值限制为 0 或 1。
4 Decimal 型
	有符号的浮点数，可在加、减和乘法运算过程中保持精度。对于除法，最低有效数字会 被丢弃(不舍入)。
	有三种声明:
			➢ Decimal32(s)，相当于Decimal(9-s,s)，有效位数为1~9
			➢ Decimal64(s)，相当于Decimal(18-s,s)，有效位数为1~18
		➢ Decimal128(s)，相当于Decimal(38-s,s)，有效位数为1~38
	s 标识小数位
使用场景: 一般金额字段、汇率、利率等字段为了保证小数点精度，都使用 Decimal进行存储。
5 字符串
1)String 字符串可以任意长度的。它可以包含任意的字节集，包含空字节。
2)FixedString(N)
	固定长度 N 的字符串，N 必须是严格的正自然数。当服务端读取长度小于 N 的字符
	串时候，通过在字符串末尾添加空字节来达到 N 字节长度。 当服务端读取长度大于 N 的 字符串时候，将返回错误消息。
	与 String 相比，极少会使用 FixedString，因为使用起来不是很方便。

6 枚举
	包括 Enum8 和 Enum16 类型。Enum 保存 'string'= integer 的对应关系。 Enum8 用 'String'= Int8 对描述。
Enum16 用 'String'= Int16 对描述。	
1)用法演示
	创建一个带有一个枚举 Enum8('hello' = 1, 'world' = 2) 类型的列
	CREATE TABLE t_enum(x Enum8('hello' = 1, 'world' = 2))ENGINE = TinyLog;
ENGINE = TinyLog;
2)这个 x 列只能存储类型定义中列出的值:'hello'或'world'
	 INSERT INTO t_enum VALUES ('hello'), ('world'), ('hello');
3)如果尝试保存任何其他值，ClickHouse 抛出异常 insert into t_enum values('a')
4)如果需要看到对应行的数值，则必须将 Enum 值转换为整数类型  SELECT CAST(x, 'Int8') FROM t_enum;
使用场景:
	对一些状态、类型的字段算是一种空间优化，也算是一种数据约束。但是实 际使用中往往因为一些数据内容的变化增加一定的维护成本，甚至是数据丢失问题。所以谨 慎使用。
	
7 时间戳	
	目前 ClickHouse 有三种时间类型
		➢ Date接受年-月-日的字符串比如‘2019-12-16’
		➢ Datetime 接受年-月-日 时:分:秒的字符串比如 ‘2019-12-16 20:50:10’
		➢ Datetime64 接受年-月-日 时:分:秒.亚秒的字符串比如‘2019-12-16 20:50:10.66’ 日期类型，用两个字节存储，表示从 1970-	01-01 (无符号) 到当前的日期值。 
	还有很多数据结构，可以参考官方文档:https://clickhouse.yandex/docs/zh/data_types/
	
8 数组
	Array(T):由 T 类型元素组成的数组。
	T 可以是任意类型，包含数组类型。 但不推荐使用多维数组，ClickHouse 对多维数组 的支持有限。例如，不能在 MergeTree 表中存储多维数组。
	1)创建数组方式 1，使用 array 函数
	SELECT array(1, 2) AS x, toTypeName(x) ;
	2)创建数组方式 2:使用方括号
	SELECT [1, 2] AS x, toTypeName(x);
```

补充:一般

## 五 表引擎

表引擎是 ClickHouse 的一大特色。可以说， 表引擎决定了如何存储表的数据。包括: 

```
1数据的存储方式和位置，写到哪里以及从哪里读取数据。
2支持哪些查询以及如何支持。
3并发数据访问。
4索引的使用(如果存在)。
5是否可以执行多线程请求。
6数据复制参数。
```

表引擎的使用方式就是必须显式在创建表时定义该表使用的引擎，以及引擎使用的相关参数。

```
引擎的名称大小写敏感
```

### 5.1 **TinyLog**

以列文件的形式保存在磁盘上，不支持索引，没有并发控制。一般保存少量数据的小表， 生产环境上作用有限。可以用于平时练习测试用。

```sql
create table t_tinylog ( id String, name String) engine=TinyLog;
```

### 5.2 **Memory**

​	内存引擎，数据以未压缩的原始形式直接保存在内存当中，服务器重启数据就会消失。 读写操作不会相互阻塞，不支持索引。简单查询下有非常非常高的性能表现(超过 10G/s)。 一般用到它的地方不多，除了用来测试，就是在需要非常高的性能，同时数据量又不太大(上限大概 1 亿行)的场景。

### 5.3 **MergeTree**

​	ClickHouse 中最强大的表引擎当属 MergeTree(合并树)引擎及该系列(*MergeTree) 中的其他引擎，支持索引和分区，地位可以相当于 innodb 之于 Mysql。而且基于 MergeTree， 还衍生除了很多小弟，也是非常有特色的引擎。

**1**)建表语句

```sql
create table t_order_mt(
   id UInt32,
   sku_id String,
   total_amount Decimal(16,2),
   create_time Datetime
) engine =MergeTree
partition by toYYYYMMDD(create_time) 
primary key (id)
order by (id,sku_id);
```

**2**)插入数据

```sql
insert into t_order_mt values (101,'sku_001',1000.00,'2020-06-01 12:00:00') , 
(102,'sku_002',2000.00,'2020-06-01 11:00:00'), (102,'sku_004',2500.00,'2020-06-01 12:00:00'), (102,'sku_002',2000.00,'2020-06-01 13:00:00'), (102,'sku_002',12000.00,'2020-06-01 13:00:00'), (102,'sku_002',600.00,'2020-06-02 12:00:00');
```

MergeTree 其实还有很多参数(绝大多数用默认值即可)，但是三个参数是更加重要的， 也涉及了关于 MergeTree 的很多概念。

1 partition by 分区(可选)

```
	1.1作用: 分区的目的主要是降低扫描的范围，优化查询速度
  1.2若是不填只会使用一个分区。
  1.3分区目录: MergeTree 是以列文件+索引文件+表定义文件组成的，但是如果设定了分区那么这些文件就会保存到不同的分区目录中。
  1.4并行: 分区后，面对涉及跨分区的查询统计，ClickHouse 会以分区为单位并行处理。
	1.5数据写入与分区合并: 任何一个批次的数据写入都会产生一个临时分区，不会纳入任何一个已有的分区。写入后的某个时刻(大概 10-15 分钟	后)，ClickHouse 会自动执行合并操作(等不及也可以手动 通过 optimize 执行)，把临时分区的数据，合并到已有分区中。
	optimize table xxxx final;
```

2 **primary key** 主键**(**可选**)**

```
ClickHouse 中的主键，和其他数据库不太一样，它只提供了数据的一级索引，但是却不 是唯一约束。这就意味着是可以存在相同 primary key 的数据的。
主键的设定主要依据是查询语句中的 where 条件。
根据条件通过对主键进行某种形式的二分查找，能够定位到对应的index granularity,避 免了全表扫描。
index granularity: 直接翻译的话就是索引粒度，指在稀疏索引中两个相邻索引对应数 据的间隔。ClickHouse 中的 MergeTree 默认是 8192。官方不建议修改这个值，除非该列存在 大量重复值，比如在一个分区中几万行才有一个不同数据。

稀疏索引:
稀疏索引的好处就是可以用很少的索引数据，定位更多的数据，代价就是只能定位到索 引粒度的第一行，然后再进行进行一点扫描。
```

3 **order by**(必选)

```
order by 设定了分区内的数据按照哪些字段顺序进行有序保存。
order by 是 MergeTree 中唯一一个必填项，甚至比 primary key 还重要，因为当用户不 设置主键的情况，很多处理会依照 order by 的字段进行处理(比如后面会讲的去重和汇总)。
要求:主键必须是 order by 字段的前缀字段。
比如 order by 字段是 (id,sku_id) 那么主键必须是 id 或者(id,sku_id)
```

4 **二级索引**

目前在 ClickHouse 的官网上二级索引的功能在 v20.1.2.4 之前是被标注为实验性的，在 这个版本之后默认是开启的。

**1**)老版本使用二级索引前需要增加设置

```sql
--是否允许使用实验性的二级索引(v20.1.2.4 开始，这个参数已被删除，默认开启)
set allow_experimental_data_skipping_indices=1;
```

```sql
--创建测试表
create table t_order_mt2(
   	id UInt32,
		sku_id String,
		total_amount Decimal(16,2), create_time Datetime,
    INDEX a total_amount TYPE minmax GRANULARITY 5
 ) engine =MergeTree
  partition by toYYYYMMDD(create_time)
   primary key (id)
order by (id, sku_id);
其中 GRANULARITY N 是设定二级索引对于一级索引粒度的粒度。
--插入数据
insert into t_order_mt2 values (101,'sku_001',1000.00,'2020-06-01 12:00:00') , (102,'sku_002',2000.00,'2020-06-01 11:00:00'), (102,'sku_004',2500.00,'2020-06-01 12:00:00'), (102,'sku_002',2000.00,'2020-06-01 13:00:00'), (102,'sku_002',12000.00,'2020-06-01 13:00:00'), (102,'sku_002',600.00,'2020-06-02 12:00:00');

clickhouse-client --send_logs_level=trace<<<'select * from t_order_mt2 where total_amount > toDecimal32(900., 2)';
 
```

5 **数据TTL**

TTL 即 Time To Live，MergeTree 提供了可以管理数据表或者列的生命周期的功能

**1**)列级别 **TTL**

```sql
--创建测试表
create table t_order_mt3(
   id UInt32,
sku_id String,
total_amount Decimal(16,2) TTL create_time+interval 10 SECOND, create_time Datetime
 ) engine =MergeTree
 partition by toYYYYMMDD(create_time)
  primary key (id)
  order by (id, sku_id);
--插入数据 注意:根据实际时间改变 
 insert into t_order_mt3 values (106,'sku_001',1000.00,'2022-04-12 22:52:30'), (107,'sku_002',2000.00,'2022-04-12 22:52:30'), (110,'sku_003',600.00,'2022-04-12 12:00:00');
 --手动合并，查看效果 到期后，指定的字段数据归0
 optimize table t_order_mt3 final;
```

**2**)表级 **TTL**

下面的这条语句是数据会在 create_time 之后 10 秒丢失

```sql
alter table t_order_mt3 MODIFY TTL create_time + INTERVAL 10 SECOND;
```

涉及判断的字段必须是 Date 或者 Datetime 类型，推荐使用分区的日期字段。

能够使用的时间周期:

```
- SECOND
- MINUTE
- HOUR
- DAY
- WEEK
- MONTH
- QUARTER 
- YEAR
```

### 5.4 **ReplacingMergeTree**

​	ReplacingMergeTree 是 MergeTree 的一个变种，它存储特性完全继承 MergeTree，只是 多了一个去重的功能。 尽管 MergeTree 可以设置主键，但是 primary key 其实没有唯一约束 的功能。如果你想处理掉重复的数据，可以借助这个 ReplacingMergeTree。

**1**)去重时机

数据的去重只会在合并的过程中出现。合并会在未知的时间在后台进行，所以你无法预 先作出计划。有一些数据可能仍未被处理。

**2**)去重范围

如果表经过了分区，去重只会在分区内部进行去重，不能执行跨分区的去重。
 所以 ReplacingMergeTree 能力有限， ReplacingMergeTree 适用于在后台清除重复的数据以节省空间，但是它不保证没有重复的数据出现。

```sql
--建表
create table t_order_rmt(
   id UInt32,
sku_id String,
total_amount Decimal(16,2) , create_time Datetime
) engine =ReplacingMergeTree(create_time)
  partition by toYYYYMMDD(create_time)
  primary key (id)
  order by (id, sku_id);
--ReplacingMergeTree() 填入的参数为版本字段，重复数据保留版本字段值最大的。 如果不填版本字段，默认按照插入顺序保留最后一条。
--插入数据
insert into t_order_rmt values (101,'sku_001',1000.00,'2020-06-01 12:00:00') , (102,'sku_002',2000.00,'2020-06-01 11:00:00'), (102,'sku_004',2500.00,'2020-06-01 12:00:00'), (102,'sku_002',2000.00,'2020-06-01 13:00:00'), (102,'sku_002',12000.00,'2020-06-01 13:00:00'), (102,'sku_002',600.00,'2020-06-02 12:00:00');
--查询
select * from t_order_rmt;
--手动合并
OPTIMIZE TABLE t_order_rmt FINAL;
--查询
select * from t_order_rmt;
```

3)结论

```
➢ 实际上是使用 order by 字段作为唯一键
➢ 去重不能跨分区
➢ 只有同一批插入(新版本)或合并分区时才会进行去重 
➢ 认定重复的数据保留，版本字段值最大的
➢ 如果版本字段相同则按插入顺序保留最后一笔
```

### 5.5 **SummingMergeTree**

​	对于不查询明细，只关心以维度进行汇总聚合结果的场景。如果只使用普通的 MergeTree 的话，无论是存储空间的开销，还是查询时临时聚合的开销都比较大。ClickHouse 为了这种场景，提供了一种能够“预聚合”的引擎 SummingMergeTree

```sql
--建表
create table t_order_smt(
   id UInt32,
   sku_id String,
   total_amount Decimal(16,2) ,
   create_time Datetime
) engine =SummingMergeTree(total_amount)
  partition by toYYYYMMDD(create_time)
  primary key (id)
  order by (id,sku_id );
--插入数据
insert into t_order_smt values (101,'sku_001',1000.00,'2020-06-01 12:00:00'), (102,'sku_002',2000.00,'2020-06-01 11:00:00'), (102,'sku_004',2500.00,'2020-06-01 12:00:00'), (102,'sku_002',2000.00,'2020-06-01 13:00:00'), (102,'sku_002',12000.00,'2020-06-01 13:00:00'), (102,'sku_002',600.00,'2020-06-02 12:00:00');
--查询
select * from t_order_smt;
--手动合并
OPTIMIZE TABLE t_order_smt FINAL;
--查询
select * from t_order_smt;
```

结论:

- ➢  以SummingMergeTree()中指定的列作为汇总数据列

- ➢  可以填写多列必须数字列，如果不填，以所有非维度列且为数字列的字段为汇总数

  据列

- ➢  以 order by 的列为准，作为维度列

- ➢  其他的列按插入顺序保留第一行

- ➢  不在一个分区的数据不会被聚合

- ➢  只有在同一批次插入(新版本)或分片合并时才会进行聚合

```sql
--设计聚合表的话，唯一键值、流水号可以去掉，所有字段全部是维度、度量或者时间戳。
能不能直接执行以下 SQL 得到汇总值
select total_amount from XXX where province_name=’’ and create_date=’xxx’
不行,可能会包含一些还没来得及聚合的临时明细
如果要是获取汇总值，还是需要使用 sum 进行聚合，这样效率会有一定的提高，但本 身 ClickHouse 是列式存储的，效率提升有限，不会特别明显。
select sum(total_amount) from province_name=’’ and create_date=‘xxx’
```

## 六 sql操作

​	基本上来说传统关系型数据库(以 MySQL 为例)的 SQL 语句，ClickHouse 基本都支持， 这里不会从头讲解 SQL 语法只介绍 ClickHouse 与标准 SQL(MySQL)不一致的地方。	

### 6.1 插入insert

基本与标准 SQL(MySQL)基本一致,支持直接插入,和从表查询插入

### 6.2 **Update** 和 **Delete**

lickHouse 提供了 Delete 和 Update 的能力，这类操作被称为 Mutation 查询，它可以看 做 Alter 的一种。

虽然可以实现修改和删除，但是和一般的 OLTP 数据库不一样，**Mutation** 语句是一种很 “重”的操作，而且不支持事务。

“重”的原因主要是每次修改或者删除都会导致放弃目标数据的原有分区，重建新分区。 所以尽量做批量的变更，不要进行频繁小数据的操作。

(1)删除操作

```sql
 alter table t_order_smt delete where sku_id ='sku_001';
```

(2)修改操作

```sql
alter table t_order_smt update total_amount=toDecimal32(2000.00,2) where id
=102;
```

由于操作比较“重”，所以 Mutation 语句分两步执行，同步执行的部分其实只是进行 新增数据新增分区和并把旧分区打上逻辑上的失效标记。直到触发分区合并的时候，才会删 除旧数据释放磁盘空间，一般不会开放这样的功能给用户，由管理员完成。

### 6.3 查询

ClickHouse 基本上与标准 SQL 差别不大

- ➢  支持子查询

- ➢  支持 CTE(Common Table Expression 公用表表达式 with 子句)

- ➢  支持各种JOIN，但是JOIN操作无法使用缓存，所以即使是两次相同的JOIN语句，

  ClickHouse 也会视为两条新 SQL

- ➢  窗口函数(官方正在测试中...)

- ➢  不支持自定义函数

- ➢  GROUP BY 操作增加了 with rollup\with cube\with total 用来计算小计和总计。

```
with rollup:从右至左去掉维度进行小计
with cube : 从右至左去掉维度进行小计，再从左至右去掉维度进行小计
with totals: 只计算合计
```

### 6.4 alter

同 MySQL 的修改字段基本一致

```
1)新增字段
alter table tableName add column newcolname String after col1;
2)修改字段类型
alter table tableName modify column newcolname String;
3)删除字段
alter table tableName drop column newcolname;
```

### 6.5 导出数据

```sql
clickhouse-client --query "select * from t_order_mt where
create_time='2020-06-01 12:00:00'" --format CSVWithNames>
/opt/module/data/rs1.csv
```

更多格式参考:https://clickhouse.tech/docs/en/interfaces/formats/

## 七 副本

副本的目的主要是保障数据的高可用性，即使一台 ClickHouse 节点宕机，那么也可以从 其他服务器获得相同的数据。

https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/replication/

### 7.1 副本写入流程

![img](/images/ck/cki.png)

### 7.2 配置步骤

(1)启动 zookeeper 集群

(2)在 节点 的/etc/clickhouse-server/config.d 目录下创建一个名为 metrika.xml 的配置文件,内容如下:

注:也可以不创建外部文件，直接在 config.xml 中指定<zookeeper>

```xmL
<?xml version="1.0"?>
<yandex>
    <zookeeper-servers>
       <node index="1">
<host>hadoop102</host>
           <port>2181</port>
       </node>
       <node index="2">
           <host>hadoop103</host>
           <port>2181</port>
       </node>
       <node index="3">
           <host>hadoop104</host>
           <port>2181</port>
       </node>
    </zookeeper-servers>
</yandex>
```

(3)节点同步

```shell
sudo /home/atguigu/bin/xsync /etc/clickhouse-server/config.d/metrika.xml
```

(4)在 节点 的/etc/clickhouse-server/config.xml 中增加

```xml
<zookeeper incl="zookeeper-servers" optional="true" />
<include_from>/etc/clickhouse-server/config.d/metrika.xml</include_from>
```

(5)节点同步

```shell
sudo /home/atguigu/bin/xsync /etc/clickhouse-server/config.xml
```

分别在 另外的节点启动clickhouse服务

注意:因为修改了配置文件，如果以前启动了服务需要重启

```
sudo clickhouse restart
```

(6)在两个节点上分别建表

副本只能同步数据，不能同步表结构，所以我们需要在每台机器上自己手动建表

```sql
节点1:
create table t_order_rep2 (
   id UInt32,
sku_id String,
total_amount Decimal(16,2), create_time Datetime
 ) engine =ReplicatedMergeTree('/clickhouse/table/01/t_order_rep','rep_102')
  partition by toYYYYMMDD(create_time)
  primary key (id)
  order by (id,sku_id);
节点2:
create table t_order_rep2 (
   id UInt32,
sku_id String,
total_amount Decimal(16,2), create_time Datetime
 ) engine =ReplicatedMergeTree('/clickhouse/table/01/t_order_rep','rep_103')
  partition by toYYYYMMDD(create_time)
  primary key (id)
  order by (id,sku_id);
```

(7) 参数解释

```
ReplicatedMergeTree 中， 第一个参数是分片的zk_path一般按照:/clickhouse/table/{shard}/{table_name} 的格式写，如果只有一个分片就写 01 即可。
第二个参数是副本名称，相同的分片副本名称不能相同。
```

```sql
--节点1上插入数据
insert into t_order_rep2 values
(101,'sku_001',1000.00,'2020-06-01 12:00:00'),
(102,'sku_002',2000.00,'2020-06-01 12:00:00'),
(103,'sku_004',2500.00,'2020-06-01 12:00:00'),
(104,'sku_002',2000.00,'2020-06-01 12:00:00'),
(105,'sku_003',600.00,'2020-06-02 12:00:00');
--节点2执行查询 ,可以查询出结果，说明副本配置正确
select * from t_order_rep2 a 
```

## 八 分片集群

​	副本虽然能够提高数据的可用性，降低丢失风险，但是每台服务器实际上必须容纳全量 数据，对数据的横向扩容没有解决。

​	要解决数据水平切分的问题，需要引入分片的概念。通过分片把一份完整的数据进行切 分，不同的分片分布到不同的节点上，再通过 **Distributed 表引擎把数据拼接起来一同使用。 Distributed 表引擎本身不存储数据**，有点类似于 MyCat 之于 MySql，成为一种中间件，

​	通过分布式逻辑表来写入、分发、路由来操作多台节点不同分片的分布式数据。

注意:ClickHouse 的集群是表级别的，实际企业中，大部分做了高可用，但是没有用分 片，避免降低查询性能以及操作集群的复杂性。

### 8.1 集群写入流程(以3分片2副本共6节点)

![img](/images/ck/ckw.png)

### 8.2 集群读取流程(以3分片2副本共6节点)

![img](/images/ck/ckr.png)

### 8.3 集群配置

```xml
<!--3 分片 2 副本共 6 个节点集群配置 供参考-->
<!--配置的位置还是在之前的/etc/clickhouse-server/config.d/metrika.xml-->
<yandex>
    <remote_servers>
<gmall_cluster> <!-- 集群名称--> <shard> <!--集群的第一个分片-->
                <internal_replication>true</internal_replication>
<!--该分片的第一个副本--> <replica>
                  <host>hadoop101</host>
<port>9000</port> </replica> <!--该分片的第二个副本--> <replica>
       <host>hadoop102</host>
       <port>9000</port>
    </replica>
</shard>
<shard> <!--集群的第二个分片--> <internal_replication>true</internal_replication> <replica> <!--该分片的第一个副本-->
       <host>hadoop103</host>
        <port>9000</port>
</replica>
<replica> <!--该分片的第二个副本-->
       <host>hadoop104</host>
       <port>9000</port>
    </replica>
</shard>
<shard> <!--集群的第三个分片--> <internal_replication>true</internal_replication> <replica> <!--该分片的第一个副本-->
       <host>hadoop105</host>
<port>9000</port>
</replica>
<replica> <!--该分片的第二个副本-->
       <host>hadoop106</host>
       <port>9000</port>
    </replica>
</shard>
        </gmall_cluster>
    </remote_servers>
</yandex>
```

### 8.4建表

先有本地表,后有分布式表,分布式表不存数据,可以控制本地表

```sql
--本地表
create table st_order_mt on cluster gmall_cluster (
   id UInt32,
sku_id String,
total_amount Decimal(16,2), create_time Datetime
 ) engine
=ReplicatedMergeTree('/clickhouse/tables/{shard}/st_order_mt','{replica}')
  partition by toYYYYMMDD(create_time)
  primary key (id)
  order by (id,sku_id);
--分布式表
 create table st_order_mt_all2 on cluster gmall_cluster
(
		id UInt32,
   	sku_id String,
		total_amount Decimal(16,2), create_time Datetime
)engine = Distributed(gmall_cluster,default, st_order_mt,hiveHash(sku_id));
```

**参数含义:**

Distributed(集群名称，库名，本地表名，分片键) 分片键必须是整型数字，所以用 hiveHash 函数转换，也可以 rand()

```sql
查询分布表
SELECT * FROM st_order_mt_all; 会查出所有节点的这个st_order_mt表的数据
本地表
select * from st_order_mt; 只会查出当前节点这个表的数据
```



