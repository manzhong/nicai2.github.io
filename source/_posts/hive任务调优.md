---
title: hive任务调优
tags: Hive
categories: Hive
abbrlink: 40074
date: 2021-04-10 23:23:57
summary_img:
encrypt:
enc_pwd:
---

## 一 常用优化参数

| 参数组                                      | 参数                                       | 参数说明                                     |
| ---------------------------------------- | ---------------------------------------- | ---------------------------------------- |
| 输出合并                                     | set hive.merge.mapfiles=true;            | 在Map-only的任务结束时合并小文件                     |
|                                          | set hive.merge.mapredfiles=true;         | 在Map-Reduce的任务结束时合并小文件                   |
|                                          | set hive.merge.size.per.task=100000000;  | 合并文件的大小                                  |
|                                          | set hive.merge.smallfiles.avgsize=64000000; | 当输出文件的平均大小小于该值时，启动一个独立的map-reduce任务进行文件merge |
| 动态分区                                     | set hive.exec.dynamic.partition=true;    | 使用动态分区时候，该参数必须设置成true;                   |
|                                          | set hive.exec.dynamic.partition.mode=nonstrict; | 默认值：strict 动态分区的模式，默认strict，表示必须指定至少一个分区为静态分区，nonstrict模式表示允许所有的分区字段都可以使用动态分区。 |
|                                          | set hive.exec.max.dynamic.partitions.pernode=100; | 默认值：100 在每个执行MR的节点上，最大可以创建多少个动态分区。       |
|                                          | set hive.exec.max.dynamic.partitions=1000; | 在所有执行MR的节点上，最大一共可以创建多少个动态分区。             |
|                                          | hive.exec.max.created.files              | 默认值：100000 整个MR Job中，最大可以创建多少个HDFS文件。    |
|                                          | hive.error.on.empty.partition            | 默认值：false 当有空分区生成时，是否抛出异常。               |
| 设置队列                                     | set mapreduce.job.queuename=队列名;         | 设置队列                                     |
| 设置任务名称yarn                               | set mapred.job.name=库.表名                 |                                          |
| 并行执行：hive会将一个查询转化为一个或多个阶段，包括：MapReduce阶段、抽样阶段、合并阶段、limit阶段等。默认情况下，一次只执行一个阶段。 不过，如果某些阶段不是互相依赖，是可以并行执行的。 | set hive.exec.parallel=true              | ,可以开启并发执行。                               |
|                                          | set hive.exec.parallel.thread.number=16; | //同一个sql允许最大并行度，默认为8。                    |
| 本地模式：有时hive的输入数据量是非常小的。在这种情况下，为查询出发执行任务的时间消耗可能会比实际job的执行时间要多的多。对于大多数这种情况，hive可以通过本地模式在单台机器上处理所有的任务。对于小数据集，执行时间会明显被缩短 | set hive.exec.mode.local.auto=true       |                                          |
|                                          | 当一个job满足如下条件才能真正使用本地模式：1.job的输入数据大小必须小于参数：hive.exec.mode.local.auto.inputbytes.max(默认128MB)2.job的map数必须小于参数：hive.exec.mode.local.auto.tasks.max(默认4)3.job的reduce数必须为0或者1。可用参数hive.mapred.local.mem(默认0)控制child jvm使用的最大内存数。 |                                          |
| 输入合并：                                    | set mapred.max.split.size=256000000;     | //每个Map最大输入大小(这个值决定了合并后文件的数量)            |
|                                          | set mapred.min.split.size.per.node=100000000; | //一个节点上split的至少的大小(这个值决定了多个DataNode上的文件是否需要合并) |
|                                          | set mapred.min.split.size.per.rack=100000000; | //一个交换机下split的至少的大小(这个值决定了多个交换机上的文件是否需要合并) |
|                                          | set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat; | //执行Map前进行小文件合并                          |
| map段聚合                                   | set hive.map.aggr=true;                  |                                          |
| 有数据倾斜的时候进行负载均衡                           | set hive.groupby.skewindata=true;        |                                          |
| Fetch task获取数据                           | set hive.fetch.task.conversion=more;     | 对于简单的不需要聚合的                              |
| jvm重用                                    | set mapred.job.reuse.jvm.num.tasks=10; --10为重用个数 | 用于避免小文件的场景或者task特别多的场景，这类场景大多数执行时间都很短，因为hive调起mapreduce任务，JVM的启动过程会造成很大的开销，尤其是job有成千上万个task任务时，JVM重用可以使得JVM实例在同一个job中重新使用N次 |
| 压缩设置                                     | set mapreduce.map.output.compress=true   | 设置是否启动map输出的压缩机制，默认为false。在需要减少网络传输的时候，可以设置为true。t |
| 推测执行：是通过加快获取单个task的结果以及进行侦测将执行慢的TaskTracker加入到黑名单的方式来提高整体的任务执行效率 | mapreduce.map.speculative                | 控制Map任务的推测执行（默认true）                     |
|                                          | mapreduce.reduce.speculative             | 控制Reduce任务的推测执行（默认true）                  |
|                                          | mapreduce.job.speculative.speculativecap | 推测执行功能的任务能够占总任务数量的比例（默认0.1，范围0~1）        |
|                                          | mapreduce.job.speculative.slownodethreshold | 判断某个TaskTracker是否适合启动某个task的speculative task（默认1） |
|                                          | mapreduce.job.speculative.slowtaskthreshold | 判断某个task是否可以启动speculative task（默认1）      |
|                                          | mapreduce.input.fileinputformat.split.minsize                          FileInputFormat | 做切片时最小切片大小，默认 1。                         |
| 数据倾斜                                     | set hive.map.aggr=true;                  |                                          |
|                                          | hive.groupby.mapaggr.checkinterval = 100000 (默认) |                                          |
|                                          | hive.map.aggr.hash.min.reduction=0.5(默认)解释：预先取100000条数据聚合,如果聚合后的条数小于100000*0.5，则不再聚合。 |                                          |
|                                          | set hive.groupby.skewindata=true；        | 决定  group by 操作是否支持倾斜数据。配置真正发生作用，只会在以下三种情况下，能够将1个job转化为2个job：select count distinct ... from ...。select a,count(*) from ... group by a。select sum(*),count(distinct ...) from。设置后,但select count(distinct id),count(distinct x) from test;会报错，可以用select count(distinct id, x) from test; |
|                                          |                                          |                                          |
|                                          |                                          |                                          |
|                                          |                                          |                                          |
|                                          |                                          |                                          |

## 二 场景解析

### 1 hive 的map和reduce

#### 1.1 Map阶段优化

map执行时间：map任务启动和初始化的时间+逻辑处理的时间。

1.通常情况下，作业会通过input的目录产生一个或者多个map任务。

主要的决定因素有： input的文件总个数，input的文件大小，集群设置的文件块大小(目前为128M, 可在hive中通过set dfs.block.size;命令查看到，该参数不能自定义修改)；

2.举例：

a)假设input目录下有1个文件a,大小为780M,那么hadoop会将该文件a分隔成7个块（6个128m的块和1个12m的块），从而产生7个map数

b)假设input目录下有3个文件a,b,c,大小分别为10m，20m，130m，那么hadoop会分隔成4个块（10m,20m,128m,2m）,从而产生4个map数

即，如果文件大于块大小(128m),那么会拆分，如果小于块大小，则把该文件当成一个块。

3.是不是map数越多越好？

答案是否定的。如果一个任务有很多小文件（远远小于块大小128m）,则每个小文件也会被当做一个块，用一个map任务来完成，而一个map任务启动和初始化的时间远远大于逻辑处理的时间，就会造成很大的资源浪费。而且，同时可执行的map数是受限的。

4.是不是保证每个map处理接近128m的文件块，就高枕无忧了？

答案也是不一定。比如有一个127m的文件，正常会用一个map去完成，但这个文件只有一个或者两个小字段，却有几千万的记录，如果map处理的逻辑比较复杂，用一个map任务去做，肯定也比较耗时。

针对上面的问题3和4，我们需要采取两种方式来解决：即减少map数和增加map数；

如何合并小文件，减少map数？

 假设一个SQL任务：Select count(1) from popt_tbaccountcopy_mes where pt = ‘2012-07-04’;

 该任务的inputdir  /group/p_sdo_data/p_sdo_data_etl/pt/popt_tbaccountcopy_mes/pt=2012-07-04

 共有194个文件，其中很多是远远小于128m的小文件，总大小9G，正常执行会用194个map任务。

 Map总共消耗的计算资源： SLOTS_MILLIS_MAPS= 623,020

 **通过以下方法来在map执行前合并小文件，减少map数**

```sql
	set mapred.max.split.size=256000000;    每个Map最大输入大小 ,限制大小  (一般只设置这个参数即可,会把小文件进行合并到256m,若需增加map数可以适当减小这个参数)
    set mapred.min.split.size.per.node=100000000;  100m    一个节点上split的至少的大小 ,决定了多个data node上的文件是否需要合并
    set mapred.min.split.size.per.rack=100000000;   一个交换机下split的至少的大小,决定了多个交换机上的文件是否需要合并
    set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat;  表示执行前进行小文件合并,，一个data node节点上多个小文件会进行合并
		前面三个参数确定合并文件块的大小，大于文件块大小128m的，按照128m来分隔，小于128m,大于100m的，按照100m来分隔，把那些小于100m的（包括小文件和分隔大文件剩下的），
    进行合并,最终生成了74个块。
```

**如何适当的增加map数**

当input的文件都很大，任务逻辑复杂，map执行非常慢的时候，可以考虑增加Map数，

来使得每个map处理的数据量减少，从而提高任务的执行效率。

 假设有这样一个任务：

```sql
 Select data_desc,
  count(1),
  count(distinct id),
  sum(case when …),
  sum(case when ...),
  sum(…)
from a group by data_desc
```

如果表a只有一个文件，大小为120M，但包含几千万的记录，

如果用1个map去完成这个任务，肯定是比较耗时的，

这种情况下，我们要考虑将这一个文件合理的拆分成多个，

这样就可以用多个map任务去完成。

```sql
  set mapred.map.tasks=10；
   create table a_1 as 
   select * from a 
   distribute by rand(123); 
set mapred.map.tasks；  这个参数设置的map数量仅仅是一个提示，只有当InputFormat 决定了map任务的个数比mapred.map.tasks值小时才起作用。
JobConf 的conf.setNumMapTasks(int num)方法来手动地设置  这个方法能够用来增加map任务的个数，但是不能设定任务的个数小于Hadoop系统通过分割输入数据得到的
```

这样会将a表的记录，随机的分散到包含10个文件的a_1表中，再用a_1代替上面sql中的a表，则会用10个map任务去完成。

每个map任务处理大于12M（几百万记录）的数据，效率肯定会好很多。

看上去，貌似这两种有些矛盾，一个是要合并小文件，一个是要把大文件拆成小文件，这点正是重点需要关注的地方。

正常的map数量的并行规模大致是每一个Node是 10~100个，对于CPU消耗较小的作业可以设置Map数量为300个左右，
因此比较合理 的情况是每个map执行的时间至少超过1分钟。
具体的数据分片是这样的，InputFormat在默认情况下会根据hadoop集群的DFS块大小进行分 片，每一个分片会由一个map任务来进行处理，
当然用户还是可以通过参数mapred.min.split.size参数在作业提交客户端进行自定义设 置。
还有一个重要参数就是mapred.map.tasks，这个参数设置的map数量仅仅是一个提示，只有当InputFormat 决定了map任务的个数比mapred.map.tasks值小时才起作用。
同样，Map任务的个数也能通过使用JobConf 的conf.setNumMapTasks(int num)方法来手动地设置。
这个方法能够用来增加map任务的个数，但是不能设定任务的个数小于Hadoop系统通过分割输入数据得到的值。
当然为了提高 集群的并发效率，可以设置一个默认的map数量，当用户的map数量较小或者比本身自动分割的值还小时可以使用一个相对交大的默认值，从而提高整体 hadoop集群的效率。

控制map数量需要遵循两个原则：使大数据量利用合适的map数；使单个map任务处理合适的数据量；

#### 1.2 控制hive任务的reduce数：

1.Hive自己如何确定reduce数：

reduce个数的设定极大影响任务执行效率，不指定reduce个数的情况下，Hive会猜测确定一个reduce个数，基于以下两个设定：

```sql
hive.exec.reducers.bytes.per.reducer（每个reduce任务处理的数据量，默认为1000^3=1G） 
hive.exec.reducers.max（每个任务最大的reduce数，默认为999）
计算reducer数的公式很简单N=min(参数2，总输入数据量/参数1)
reduce个数 = InputFileSize / bytes per reducer
```

即，如果reduce的输入（map的输出）总大小不超过1G,那么只会有一个reduce任务；

如：

```sql
select pt,count(1) from popt_tbaccountcopy_mes where pt = '2012-07-04' group by pt; 

-- /group/p_sdo_data/p_sdo_data_etl/pt/popt_tbaccountcopy_mes/pt=2012-07-04 总大小为9G多，

-- 因此这句有10个reduce
```

2.调整reduce个数方法一：

```sql
调整hive.exec.reducers.bytes.per.reducer参数的值；
set hive.exec.reducers.bytes.per.reducer=500000000; （500M）
select pt,count(1) from popt_tbaccountcopy_mes where pt = '2012-07-04' group by pt; 这次有20个reduce
```

3.调整reduce个数方法二；

````sql
 --若为-1 要系统自动评估,一般应用于一段sql,有临时表,上一段设置了reduce的值(不为-1),下一个要系统自动评估时设置
set mapred.reduce.tasks = 15; 
select pt,count(1) from popt_tbaccountcopy_mes where pt = '2012-07-04' group by pt;
--这次有15个reduce
````

4.reduce个数并不是越多越好；

同map一样，启动和初始化reduce也会消耗时间和资源；

另外，有多少个reduce,就会有多少个输出文件，如果生成了很多个小文件，

那么如果这些小文件作为下一个任务的输入，则也会出现小文件过多的问题；

5.什么情况下只有一个reduce；

```
很多时候你会发现任务中不管数据量多大，不管你有没有设置调整reduce个数的参数，任务中一直都只有一个reduce任务；
其实只有一个reduce任务的情况，除了数据量小于hive.exec.reducers.bytes.per.reducer参数值的情况外，还有以下原因：
a)没有group by的汇总，比如把select pt,count(1) from popt_tbaccountcopy_mes where pt = '2012-07-04' group by pt; 
写成 select count(1) from popt_tbaccountcopy_mes where pt = '2012-07-04';
这点非常常见，希望大家尽量改写。
b)用了Order by
c)有笛卡尔积
```

使大数据量利用合适的reduce数；使单个reduce任务处理合适的数据量。

### 2 GC overhead limit exceeded

设置jvm参数set mapreduce.map.java.opts 或set mapreduce.reduce.java.opts
以map任务为例，Container其实就是在执行一个脚本文件，而脚本文件中，会执行一个 Java 的子进程，这个子进程就是真正的 Map Task，
mapreduce.map.java.opts 其实就是启动 JVM 虚拟机时，传递给虚拟机的启动参数，而默认值 -Xmx200m 表示这个 Java 程序可以使用的最大堆内存数，一旦超过这个大小，
JVM 就会抛出 Out of Memory 异常，并终止进程。而 mapreduce.map.memory.mb 设置的是 Container 的内存上限，
这个参数由 NodeManager 读取并进行控制，当 Container 的内存大小超过了这个参数值，NodeManager 会负责 kill 掉 Container。
在后面分析 yarn.nodemanager.vmem-pmem-ratio 这个参数的时候，会讲解 NodeManager 监控 Container 内存（包括虚拟内存和物理内存）及 kill 掉 Container 的过程。
也就是说，mapreduce.map.java.opts一定要小于mapreduce.map.memory.mb
mapreduce.reduce.java.opts同mapreduce.map.java.opts一样的道理。


mapreduce.map.java.opts一定要小于mapreduce.map.memory.mb
mapreduce.reduce.java.opts同mapreduce.map.java.opts一样的道理。

set mapreduce.map.memory.mb=2048; 一个 Map Task 可使用的内存上限（单位:MB），默认为 1024。如果 Map Task 实际使用的资源量超过该值，则会被强制杀死。
set mapreduce.map.java.opts=-Xmx1024m;map 堆大小
set mapreduce.reduce.memory.mb=3072; 一个 Reduce Task 可使用的资源上限（单位:MB），默认为 1024。如果 Reduce Task 实际使用的资源量超过该值，则会被强制杀死。
set mapreduce.reduce.java.opts=-Xmx2048m;  reduce 堆的大小
--set mapred.child.java.opts=-Xmx2048M;新版本中已经标准为过期，
--取而代之的是区分Map Task 和Reduce Task 的jvm opts , mapred.map.child.java.opts和mapred.reduce.child.java.opts(默认值为-Xmx200m)
mapred.child.java.opts设置成多大比较合适：
这个参数是配置每个map或reduce使用的内存数量，默认是200m，一般情况下，该值设置为 总内存/并发数量(=核数)

mapred.map.child.java.opts和mapreduce.map.memeory.mb的区别：
mapreduce.map.memory.mb是向RM申请的内存资源大小，这些资源可用用于各种程序语言编写的程序, mapred.map.child.java.opts 一般只用于配置JVM参数

mapreduce.task.io.sort.mb 100 shuffle 的环形缓冲区大小，默认 100m
mapreduce.map.sort.spill.percent 0.8 环形缓冲区溢出的阈值，默认 80%

yarn.nodemanager.resource.memory-mb 表示该节点上YARN可使用的物理内存总量，默认是 8192（MB），注意，如果你的节点内存资源不够 8GB，则需要调减小这个值，而 YARN不会智能的探测节点的物理内存总量。
shuffle 性能优化的关键参数，应在 yarn 启动之前就配置好

### 3 join

Hive在进行join时，按照join的key进行分发，而在join左边的表的数据会首先读入内存，如果左边表的key相对分散，读入内存的数据会比较小，
join任务执行会比较快；而如果左边的表key比较集中，而这张表的数据量很大，那么数据倾斜就会比较严重，而如果这张表是小表，则还是应该把这张表放在join左边。
解决方式：使用map join让小的维度表先进内存。在map端完成reduce。
在0.7.0版本之前：需要在sql中使用 /*+ MAPJOIN(smallTable) */ ；
SELECT /*+ MAPJOIN(b) */ a.key, a.value
FROM a
JOIN b ON a.key = b.key;
0.7.0之后可以配置参数:set hive.auto.convert.join=true; 
注意：使用默认启动该优化的方式如果出现默名奇妙的BUG(比如MAPJOIN并不起作用),就将以下两个属性置为fase手动使用MAPJOIN标记来启动该优化
set hive.auto.convert.join=false; (关闭自动MAPJOIN转换操作)
set hive.ignore.mapjoin.hint=false; (不忽略MAPJOIN标记)

--小表的最大文件大小，默认为25000000，即25M
set hive.mapjoin.smalltable.filesize = 25000000;
以下两个:是预先知道表大小是能够被加载进内存的
--是否将多个mapjoin合并为一个
set hive.auto.convert.join.noconditionaltask = true;
--多个mapjoin转换为1个时，所有小表的文件大小总和的最大值。
set hive.auto.convert.join.noconditionaltask.size = 10000000;

map join并不会涉及reduce操作。map端join的优势就是在于没有shuffle

### 4 union all

在使用union all的时候，系统资源足够的情况下，为了加快hive处理速度，可以设置如下参数实现并发执行
set mapred.job.priority=VERY_HIGH;
set hive.exec.parallel=true;

### 5 动态分区的坑

1,动态分区的值是在reduce运行阶段确定的.也就是会把所有的记录distribute by。 可想而知表记录非常大的话，只有一个reduce 去处理,
2,静态分区在编译阶段已经确定，不需要reduce处理

动态分区有多个字段时,且最后一个job只有一个map阶段,无reduce阶段,数据量大有4000个map,这种情况下map任务在往hive分区中写的时候，
每个map几乎都要产生28个文件，这样就会产生4000*28个文件，带来大量的小文件
如:
insert overwrite table test1 partition(week,type)
select
    *
from test_table
实际上不仅仅是最后一个job只有map的任务有影响，reduce同样如此，但是一般情况下reduce的数目不会太大，并且reduce数目比较好控制
解决:
最后一个阶段只有map，若是有reduce的话，把相同分区的数据发送到一个reduce处理
1 :

```sql
insert overwrite table test1 partition(week,type)
select
*
from test_table
DISTRIBUTE BY week,type;
```

这样的话产生的文件数就等于分区数目了（在不限制reduce的情况下），文件数目大大减小，但是文件数目也太少了吧，
并且由于数据分布不均匀，分区下的文件大小差异特别大。并且由于不同reduce处理的数据量差异，造成部分reduce执行速度过慢，影响了整体的速度，
2:

```sql
set hive.exec.reducers.max=500;
insert overwrite table test1 partition(week,type)
select* from test_table
DISTRIBUTE BY rand();
```

若是想把数据均匀的分配的reduce上，DISTRIBUTE BY的字段就不能使用分区下的字段，
可以使用DISTRIBUTE BY rand(),这样rand取哈希然后对reduce数目取余，
保证了每条数据分配到所有reduce的可能性是相等的，这样reduce处理的数据量就是均匀的，在数据量比较大的情况下每个reduce产生的文件数为动态分区的个数，产生的文件总个数m*分区个数。

动静结合使用时需要注意静态分区值必须在动态分区值的前面. 

