---
title: Kudu
abbrlink: 6637
date: 2017-07-14 18:31:05
tags: Kudu
categories: Kudu
summary_img:
encrypt:
enc_pwd:
---

# KuDu

## 一概述

### 背景介绍

在KUDU之前，大数据主要以两种方式存储；   

（1）静态数据：

   以 HDFS 引擎作为存储引擎，适用于高吞吐量的离线大数据分析场景。

这类存储的局限性是数据无法进行随机的读写。 

（2）动态数据：

   以 HBase、Cassandra 作为存储引擎，适用于大数据随机读写场景。

局限性是批量读取吞吐量远不如 HDFS，不适用于批量数据分析的场景。

从上面分析可知，这两种数据在存储方式上完全不同，进而导致使用场景完全不同，但在真实的场景中，边界可能没有那么清晰，面对既需要随机读写，又需要批量分析的大数据场景，该如何选择呢？

这个场景中，单种存储引擎无法满足业务需求，我们需要通过多种大数据工具组合来满足这一需求

![img](/images/kudu/t.png)

如上图所示，数据实时写入 HBase，实时的数据更新也在 HBase 完成，为了应对 OLAP 需求，我们定时将 HBase 数据写成静态的文件（如：Parquet）导入到 OLAP 引擎（如：Impala、hive）。这一架构能满足既需要随机读写，又可以支持 OLAP 分析的场景，但他有如下缺点：

(1)**架构复杂**。从架构上看，数据在HBase、消息队列、HDFS 间流转，涉及环节太多，运维成本很高。并且每个环节需要保证高可用，都需要维护多个副本，存储空间也有一定的浪费。最后数据在多个系统上，对数据安全策略、监控等都提出了挑战。

(2)**时效性低**。数据从HBase导出成静态文件是周期性的，一般这个周期是一天（或一小时），在时效性上不是很高。

(3)**难以应对后续的更新**。真实场景中，总会有数据是延迟到达的。如果这些数据之前已经从HBase导出到HDFS，新到的变更数据就难以处理了，一个方案是把原有数据应用上新的变更后重写一遍，但这代价又很高。

为了解决上述架构的这些问题，KUDU应运而生。**KUDU**的定位是Fast Analytics on Fast Data，**是一个既支持随机读写、又支持 OLAP 分析的大数据存储引擎**。

KUDU 是一个折中的产品，在 HDFS 和 HBase 这两个偏科生中平衡了随机读写和批量分析的性能。从 KUDU 的诞生可以说明一个观点：底层的技术发展很多时候都是上层的业务推动的，脱离业务的技术很可能是空中楼阁。

kudu是什么

- 是一个大数据存储引擎  用于大数据的存储，结合其他软件开展数据分析。
- 汲取了hdfs中高吞吐数据的能力和hbase中高随机读写数据的能力  
- 既满足有传统OLAP分析 又满足于随机读写访问数据
- kudu来自于cloudera 后来贡献给了apache

kudu应用场景

适用于那些既有随机访问，也有批量数据扫描的复合场景

高计算量的场景

使用了高性能的存储设备，包括使用更多的内存

支持数据更新，避免数据反复迁移

支持跨地域的实时数据备份和查询

## 二架构

- kudu集群是主从架构
  - 主角色 master ：管理集群  管理元数据
  - 从角色 tablet server：负责最终数据的存储 对外提供数据读写能力 里面存储的都是一个个tablet
- kudu tablet
  - 是kudu表中的数据水平分区  一个表可以划分成为多个tablet(类似于hbase  region)
  - tablet中主键是不重复连续的  所有tablet加起来就是一个table的所有数据
  - tablet在存储的时候 会进行冗余存放 设置多个副本 
  - 在一个tablet所有冗余中 任意时刻 一个是leader 其他的冗余都是follower

与HDFS和HBase相似，Kudu使用单个的Master节点，用来管理集群的元数据，并且使用任意数量的Tablet Server（类似HBase中的RegionServer角色）节点用来存储实际数据。可以部署多个Master节点来提高容错性。

![img](/images/kudu/t2.png)

## 1． Table

表（Table）是数据库中用来存储数据的对象，是有结构的数据集合。kudu中的表具有schema（纲要）和全局有序的primary key（主键）。kudu中一个table会被水平分成多个被称之为tablet的片段。

## 2． Tablet

一个 tablet 是一张 table连续的片段，tablet是kudu表的水平分区，类似于HBase的region。每个tablet存储着一定连续range的数据（key），且tablet两两间的range不会重叠。一张表的所有tablet包含了这张表的所有key空间。

tablet 会冗余存储。放置到多个 tablet server上，并且在任何给定的时间点，其中一个副本被认为是leader tablet,其余的被认之为follower tablet。每个tablet都可以进行数据的读请求，但只有Leader tablet负责写数据请求。

## 3． Tablet Server

tablet server集群中的小弟，负责数据存储，并提供数据读写服务

一个 tablet server 存储了table表的tablet，向kudu client 提供读取数据服务。对于给定的 tablet，一个tablet server 充当 leader，其他 tablet server 充当该 tablet 的 follower 副本。

只有 leader服务写请求，然而 leader 或 followers 为每个服务提供读请求 。一个 tablet server 可以服务多个 tablets ，并且一个 tablet 可以被多个 tablet servers 服务着。

## 4． Master Server

集群中的老大，负责集群管理、元数据管理等功能。

## 三 kudu安装

1节点规划

| **节点** | **kudu-master** | **kudu-tserver** |
| ------ | --------------- | ---------------- |
| node01 | 是               | 是                |
| node02 | 是               | 是                |
| node03 | 是               | 是                |

本次配置node01 和node02 不配置 kudu-master

2本地yum源配置

配过了在 node03上

3 安装KUDU

| **服务器** | **安装命令**                                 |
| ------- | ---------------------------------------- |
| node01  | yum   install -y kudu kudu-tserver kudu-client0 kudu-client-devel |
| node02  | yum   install -y kudu kudu-tserver kudu-client0 kudu-client-devel |
| node03  | yum   install -y kudu kudu-master kudu-tserver kudu-client0 kudu-client-devel |

```
yum install kudu # Kudu的基本包
yum install kudu-master # KuduMaster 
yum install kudu-tserver # KuduTserver 
yum install kudu-client0 #Kudu C ++客户端共享库
yum install kudu-client-devel # Kudu C ++客户端共享库 SDK
```

4 kudu节点配置

安装完成之后。 需要在所有节点的/etc/kudu/conf目录下有两个文件：master.gflagfile和tserver.gflagfile。

### 1.1． 修改master.gflagfile

~~~
# cat /etc/kudu/conf/master.gflagfile
# Do not modify these two lines. If you wish to change these variables,
# modify them in /etc/default/kudu-master.
--fromenv=rpc_bind_addresses
--fromenv=log_dir
--fs_wal_dir=/export/servers/kudu/master
--fs_data_dirs=/export/servers/kudu/master
--master_addresses=node03:7051     若为集node01:7051,node02:7051,node03:7051 若为单节点 则这句注释掉
若为单节点 且没注释掉 则启动报错

~~~

### 1.2 修改tserver.gflagfile

~~~
# Do not modify these two lines. If you wish to change these variables,
# modify them in /etc/default/kudu-tserver.
--fromenv=rpc_bind_addresses
--fromenv=log_dir
--fs_wal_dir=/export/servers/kudu/tserver
--fs_data_dirs=/export/servers/kudu/tserver
--tserver_master_addrs=node03:7051  若为集node01:7051,node02:7051,node03:7051  

~~~

### 1.3修改 /etc/default/kudu-master

~~~
export FLAGS_log_dir=/var/log/kudu
#每台机器的master地址要与主机名一致,这里是在node03上
export FLAGS_rpc_bind_addresses=node03:7051

~~~

### 1.4修改 /etc/default/kudu-tserver

~~~
export FLAGS_log_dir=/var/log/kudu
#每台机器的tserver地址要与主机名一致，这里是在node03上
export FLAGS_rpc_bind_addresses=node03:7050

~~~

kudu默认用户就是KUDU，所以需要将/export/servers/kudu权限修改成kudu：

```
mkdir /export/servers/kudu
chown -R kudu:kudu /export/servers/kudu
```

(如果使用的是普通的用户，那么最好配置sudo权限)/etc/sudoers文件中添加：

**kudu集群的启动与关闭**

1 ntp服务的安装

启动的时候要注意时间同步

安装ntp服务

```
yum -y install ntp
```

设置开机启动

```
service ntpd start 
chkconfig ntpd on
```

可以在每台服务器执行

```
/etc/init.d/ntpd restart
```

~~~
启动
service kudu-master start
service kudu-tserver start
关闭
service kudu-master stop
service kudu-tserver stop

~~~

~~~~
kudu的web管理界面。http://master主机名:8051

可以查看每个机器上master相关信息。http://node03:8051/masters    一定为8051 不为7051

tserver 的web地址  http://node03:8051/tablet-servers
~~~~

安装属于事项

### 1.1给普通用户授予sudo出错

```
sudo: /etc/sudoers is world writable
解决方式：``pkexec chmod 555 /etc/sudoers
```

### 1.2 启动kudu的时候报错

~~~
Failed to start Kudu Master Server. Return value: 1 [FAILED]
去日志文件中查看：
Service unavailable: Cannot initialize clock: Errorreading clock. Clock considered
unsynchronized
解决：
第一步：首先检查是否有安装ntp：如果没有安装则使用以下命令安装：
yum -y install ntp
第二步：设置随机启动：
service ntpd start
chkconfig ntpd on

~~~

### 1.3 启动过程中报错

~~~
Invalid argument: Unable to initialize catalog manager: Failed to initialize sys
tables
async: on-disk master list
解决：
（1）：停掉master和tserver
（2）：删除掉之前所有的/export/servers/kudu/master/*和/export/servers/kudu/tserver/*

~~~

### 1.4 启动过程中报错

~~~
error: Could not create new FS layout: unable to create file system roots: unable to
write instance metadata: Call to mkstemp() failed on name template
/export/servers/kudu/master/instance.kudutmp.XXXXXX: Permission denied (error 13)
这是因为kudu默认使用kudu权限进行执行，可能遇到文件夹的权限不一致情况，更改文件夹权限即可

~~~

## 四 Java操作kudu

###1 创建maven工程 导入依赖

~~~
<dependencies>  
   <dependency>
      <groupId>org.apache.kudu</groupId>
      <artifactId>kudu-client</artifactId>
      <version>1.6.0</version>
    </dependency>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.12</version>
    </dependency>
</dependencies>

~~~

### 2 初始化方法

~~~
public class TestKudu {

    //声明全局变量 KuduClient后期通过它来操作kudu表
    private KuduClient kuduClient;
    //指定kuduMaster地址
    private String kuduMaster;
    //指定表名
    private String tableName;

    @Before
    public void init(){
        //初始化操作
        kuduMaster="node03:7051";
        //指定表名
        tableName="student";
        KuduClient.KuduClientBuilder kuduClientBuilder = new KuduClient.KuduClientBuilder(kuduMaster);
		//设置客户端与kudu集群socket超时时间
        kuduClientBuilder.defaultSocketReadTimeoutMs(10000);
        kuduClient=kuduClientBuilder.build();
    } 

~~~

### 3 创建表

~~~
    /**
     * 创建表
     */
    @Test
    public void createTable() throws KuduException {
        //判断表是否存在，不存在就构建
        if(!kuduClient.tableExists(tableName)){

            //构建创建表的schema信息-----就是表的字段和类型
            ArrayList<ColumnSchema> columnSchemas = new ArrayList<ColumnSchema>();
            columnSchemas.add(new ColumnSchema.ColumnSchemaBuilder("id", Type.INT32).key(true).build());
            columnSchemas.add(new ColumnSchema.ColumnSchemaBuilder("name", Type.STRING).build());
            columnSchemas.add(new ColumnSchema.ColumnSchemaBuilder("age", Type.INT32).build());
            columnSchemas.add(new ColumnSchema.ColumnSchemaBuilder("sex", Type.INT32).build());
            Schema schema = new Schema(columnSchemas);

            //指定创建表的相关属性
            CreateTableOptions options = new CreateTableOptions();
            ArrayList<String> partitionList = new ArrayList<String>();
            //指定kudu表的分区字段是什么
            partitionList.add("id");    //  按照 id.hashcode % 分区数 = 分区号
            options.addHashPartitions(partitionList,6);

            kuduClient.createTable(tableName,schema,options);
        }
    }

~~~

### 4 插入数据

~~~
    /**
     * 向表加载数据
     */
    @Test
    public void insertTable() throws KuduException {
        //向表加载数据需要一个kuduSession对象
        KuduSession kuduSession = kuduClient.newSession();
        //设置提交数据为自动flush
        kuduSession.setFlushMode(SessionConfiguration.FlushMode.AUTO_FLUSH_SYNC);

        //需要使用kuduTable来构建Operation的子类实例对象  就是打开本次操作的表名
        KuduTable kuduTable = kuduClient.openTable(tableName);
		//此处需要kudutable 来构建operation的子类实例对象  此处 为insert
        for(int i=1;i<=10;i++){
            Insert insert = kuduTable.newInsert();
            PartialRow row = insert.getRow();
            row.addInt("id",i);
            row.addString("name","zhangsan-"+i);
            row.addInt("age",20+i);
            row.addInt("sex",i%2);

            kuduSession.apply(insert);//最后实现执行数据的加载操作
        }
    }

~~~

### 5 查询数据

~~~
/**
     * 查询表的数据结果
     */
    @Test
    public void queryData() throws KuduException {

        //构建一个查询的扫描器 在扫描器中指定 表名
        KuduScanner.KuduScannerBuilder kuduScannerBuilder = kuduClient.newScannerBuilder(kuduClient.openTable(tableName));
        //创建集合 用于存储扫描的字段信息
        ArrayList<String> columnsList = new ArrayList<String>();
        columnsList.add("id");
        columnsList.add("name");
        columnsList.add("age");
        columnsList.add("sex");
        kuduScannerBuilder.setProjectedColumnNames(columnsList);
        // 调用build方法执行扫描,返回结果集
        KuduScanner kuduScanner = kuduScannerBuilder.build();
        //遍历
        while (kuduScanner.hasMoreRows()){
            RowResultIterator rowResults = kuduScanner.nextRows();

             while (rowResults.hasNext()){
                 RowResult row = rowResults.next(); //拿到的是一行数据
                 int id = row.getInt("id");
                 String name = row.getString("name");
                 int age = row.getInt("age");
                 int sex = row.getInt("sex");

                 System.out.println("id="+id+"  name="+name+"  age="+age+"  sex="+sex);
             }
        }

    }

~~~

### 6 修改数据

~~~
/**
     * 修改表的数据
     */
    @Test
    public void updateData() throws KuduException {
        //修改表的数据需要一个kuduSession对象
        KuduSession kuduSession = kuduClient.newSession();
        //设置提交数据为自动flush
        kuduSession.setFlushMode(SessionConfiguration.FlushMode.AUTO_FLUSH_SYNC);

        //需要使用kuduTable来构建Operation的子类实例对象
        KuduTable kuduTable = kuduClient.openTable(tableName);

        //Update update = kuduTable.newUpdate(); //如果id不存在 ,就什么也不操作
        Upsert upsert = kuduTable.newUpsert(); //如果id存在就表示修改，不存在就新增
        PartialRow row = upsert.getRow();
        row.addInt("id",100);
        row.addString("name","zhangsan-100");
        row.addInt("age",100);
        row.addInt("sex",0);

        kuduSession.apply(upsert);//最后实现执行数据的修改操作
    }

~~~

### 7删除数据

~~~
/**
     * 删除数据
     */
    @Test
    public void deleteData() throws KuduException {
        //删除表的数据需要一个kuduSession对象
        KuduSession kuduSession = kuduClient.newSession();
        kuduSession.setFlushMode(SessionConfiguration.FlushMode.AUTO_FLUSH_SYNC);

        //需要使用kuduTable来构建Operation的子类实例对象
        KuduTable kuduTable = kuduClient.openTable(tableName);

        Delete delete = kuduTable.newDelete();
        PartialRow row = delete.getRow();
        row.addInt("id",100);


        kuduSession.apply(delete);//最后实现执行数据的删除操作
    }

~~~

### 8 删除表

~~~
    @Test
    public void dropTable() throws KuduException {

        if(kuduClient.tableExists(tableName)){
            kuduClient.deleteTable(tableName);
        }

    }

~~~

### 9 kudu的分区方式

为了提供可扩展性，Kudu 表被划分为称为 tablet 的单元，并分布在许多 tablet servers 上。行总是属于单个tablet 。将行分配给 tablet 的方法由在表创建期间设置的表的分区决定。 kudu提供了3种分区方式。

#### 1 范围分区Range Partitioning 

范围分区可以根据存入数据的数据量，均衡的存储到各个机器上，防止机器出现负载不均衡现象.

~~~
   /**
     * 测试分区：
     * RangePartition
     */
    @Test
    public void testRangePartition() throws KuduException {
        //设置表的schema
        LinkedList<ColumnSchema> columnSchemas = new LinkedList<ColumnSchema>();
        columnSchemas.add(new Column("id", Type.INT32,true));
        columnSchemas.add(new Column("WorkId", Type.INT32,false));
        columnSchemas.add(new Column("Name", Type.STRING,false));
        columnSchemas.add(new Column("Gender", Type.STRING,false));
        columnSchemas.add(new Column("Photo", Type.STRING,false));

        //创建schema
        Schema schema = new Schema(columnSchemas);

        //创建表时提供的所有选项
        CreateTableOptions tableOptions = new CreateTableOptions();
        //设置副本数
        //tableOptions.setNumReplicas(1);
        //设置范围分区的规则
        LinkedList<String> parcols = new LinkedList<String>();
        parcols.add("id");
        //设置按照那个字段进行range分区
        tableOptions.setRangePartitionColumns(parcols);

        /**
         * range
         *  0 < value < 10
         * 10 <= value < 20
         * 20 <= value < 30
         * ........
         * 80 <= value < 90
         * */
        int count=0;
        for(int i =0;i<10;i++){
            //范围开始
            PartialRow lower = schema.newPartialRow();
            lower.addInt("id",count);

            //范围结束
            PartialRow upper = schema.newPartialRow();
            count +=10;
            upper.addInt("id",count);

            //设置每一个分区的范围
            tableOptions.addRangePartition(lower,upper);
        }

        try {
            kuduClient.createTable("t-range-partition",schema,tableOptions);
        } catch (KuduException e) {
            e.printStackTrace();
        }
         kuduClient.close();

    }

~~~

#### 2 哈希分区Hash Partitioning 为kudu的默认分区

哈希分区通过哈希值将行分配到许多 buckets (存储桶 )之一； 哈希分区是一种有效的策略，当不需要对表进行有序访问时。哈希分区对于在 tablet 之间随机散布这些功能是有效的，这有助于减轻热点和 tablet 大小不均匀。

~~~
/**
     * 测试分区：
     * hash分区
     */
    @Test
    public void testHashPartition() throws KuduException {
        //设置表的schema
        LinkedList<ColumnSchema> columnSchemas = new LinkedList<ColumnSchema>();
        columnSchemas.add(new Column("id", Type.INT32,true));
        columnSchemas.add(new Column("WorkId", Type.INT32,false));
        columnSchemas.add(new Column("Name", Type.STRING,false));
        columnSchemas.add(new Column("Gender", Type.STRING,false));
        columnSchemas.add(new Column("Photo", Type.STRING,false));

        //创建schema
        Schema schema = new Schema(columnSchemas);

        //创建表时提供的所有选项
        CreateTableOptions tableOptions = new CreateTableOptions();
        //设置副本数
        tableOptions.setNumReplicas(1);
        //设置范围分区的规则
        LinkedList<String> parcols = new LinkedList<String>();
        parcols.add("id");
        //设置按照那个字段进行range分区
        tableOptions.addHashPartitions(parcols,6);
        try {
            kuduClient.createTable("t-hash-partition",schema,tableOptions);
        } catch (KuduException e) {
            e.printStackTrace();
        }

        kuduClient.close();
    }

~~~

#### 3 多级分区Multilevel Partitioning 

Kudu 允许一个表在单个表上组合多级分区。 当正确使用时，多级分区可以保留各个分区类型的优点，同时减少每个分区的缺点

如 范围分区  为5个  hash分区为3个   则多级分区为15个  (即在范围分区里面有进行了hash分区)

~~~
/**
     * 测试分区：
     * 多级分区
     * Multilevel Partition
     * 混合使用hash分区和range分区
     *
     * 哈希分区有利于提高写入数据的吞吐量，而范围分区可以避免tablet无限增长问题，
     * hash分区和range分区结合，可以极大的提升kudu的性能
     */
    @Test
    public void testMultilevelPartition() throws KuduException {
        //设置表的schema
        LinkedList<ColumnSchema> columnSchemas = new LinkedList<ColumnSchema>();
        columnSchemas.add(new Column("id", Type.INT32,true));
        columnSchemas.add(new Column("WorkId", Type.INT32,false));
        columnSchemas.add(new Column("Name", Type.STRING,false));
        columnSchemas.add(new Column("Gender", Type.STRING,false));
        columnSchemas.add(new Column("Photo", Type.STRING,false));

        //创建schema
        Schema schema = new Schema(columnSchemas);
        
        //创建表时提供的所有选项
        CreateTableOptions tableOptions = new CreateTableOptions();
        //设置副本数
        //tableOptions.setNumReplicas(1);
        //设置范围分区的规则
        LinkedList<String> parcols = new LinkedList<String>();
        parcols.add("id");

        //hash分区
        tableOptions.addHashPartitions(parcols,5);

        //range分区
        int count=0;
        for(int i=0;i<10;i++){
        //指定上界
            PartialRow lower = schema.newPartialRow();
            lower.addInt("id",count);
            count+=10;
			//指定下界
            PartialRow upper = schema.newPartialRow();
            upper.addInt("id",count);
            tableOptions.addRangePartition(lower,upper);
        }

        try {
            kuduClient.createTable("t-multilevel-partition",schema,tableOptions);
        } catch (KuduException e) {
            e.printStackTrace();
        }
        kuduClient.close();
    }

~~~



## 五 kudu集成impala

### 1 impala的配置修改

可选项 若配置以后写的时候指定 master地址即可

在每一个服务器的impala的配置文件中添加如下配置。

```
vim /etc/default/impala
```

在IMPALA_SERVER_ARGS下添加：

```
-kudu_master_hosts=node-1:7051,node-2:7051,node-3:7051
```

### 2 创建kudu表

需要先启动hdfs、hive、kudu、impala。使用impala的shell控制台。

~~~
impala-shell
~~~

内部表:impala中删除,kudu里也会删除  

外部表: impala中删除,kudu中不会删除.

2.1 内部表

内部表由Impala管理，当您从Impala中删除时，数据和表确实被删除。当您使用Impala创建新表时，它通常是内部表。

~~~~
CREATE TABLE my_first_table
(
id BIGINT,
name STRING,
PRIMARY KEY(id)
)
PARTITION BY HASH PARTITIONS 16 
STORED AS KUDU
TBLPROPERTIES (
'kudu.master_addresses' = 'node1:7051,node2:7051,node3:7051',  //这种就是没有配置以上的 指定地址
'kudu.table_name' = 'my_first_table'
);
    
在 CREATE TABLE 语句中，必须首先列出构成主键的列。

~~~~

2.2 外部表

外部表（创建者CREATE EXTERNAL TABLE）不受Impala管理，并且删除此表不会将表从其源位置（此处为Kudu）丢弃。相反，它只会去除Impala和Kudu之间的映射。这是Kudu提供的用于将现有表映射到Impala的语法。

首先使用java创建kudu表：

~~~
public class CreateTable {
        private static ColumnSchema newColumn(String name, Type type, boolean iskey) {
                ColumnSchema.ColumnSchemaBuilder column = new
                    ColumnSchema.ColumnSchemaBuilder(name, type);
                column.key(iskey);
                return column.build();
        }
    public static void main(String[] args) throws KuduException {
        // master地址
        final String masteraddr = "node1,node2,node3";
        // 创建kudu的数据库链接
        KuduClient client = new
     KuduClient.KuduClientBuilder(masteraddr).defaultSocketReadTimeoutMs(6000).build();
        
        // 设置表的schema
        List<ColumnSchema> columns = new LinkedList<ColumnSchema>();
        columns.add(newColumn("CompanyId", Type.INT32, true));
        columns.add(newColumn("WorkId", Type.INT32, false));
        columns.add(newColumn("Name", Type.STRING, false));
        columns.add(newColumn("Gender", Type.STRING, false));
        columns.add(newColumn("Photo", Type.STRING, false));
        Schema schema = new Schema(columns);
    //创建表时提供的所有选项
    CreateTableOptions options = new CreateTableOptions();
        
    // 设置表的replica备份和分区规则
    List<String> parcols = new LinkedList<String>();
        
    parcols.add("CompanyId");
    //设置表的备份数
        options.setNumReplicas(1);
    //设置range分区
    options.setRangePartitionColumns(parcols);
        
    //设置hash分区和数量
    options.addHashPartitions(parcols, 3);
    try {
    client.createTable("person", schema, options);
    } catch (KuduException e) {
    e.printStackTrace();
    }
    client.close();
    }
}

~~~

使用impala创建外部表 ， 将kudu的表映射到impala上。

~~~
在impala-shell中执行

CREATE EXTERNAL TABLE `person` STORED AS KUDU
TBLPROPERTIES(
    'kudu.table_name' = 'person',
    'kudu.master_addresses' = 'node1:7051,node2:7051,node3:7051')

~~~

### 3 使用impala对kudu进行DML

1 插入数据

impala 允许使用标准 SQL 语句将数据插入 Kudu 。

首先建表：

~~~~
在impala-shell中
CREATE TABLE my_first_table1
(
id BIGINT,
name STRING,
PRIMARY KEY(id)
)
PARTITION BY HASH PARTITIONS 16
STORED AS KUDU 
TBLPROPERTIES(
    'kudu.table_name' = 'person1',
    'kudu.master_addresses' = 'node1:7051,node2:7051,node3:7051');

~~~~

此示例插入单个行：

```
INSERT INTO my_first_table VALUES (50, "zhangsan");
```

此示例插入3行：

```
INSERT INTO my_first_table VALUES (1, "john"), (2, "jane"), (3, "jim");
```

批量导入数据：

从 Impala 和 Kudu 的角度来看，通常表现最好的方法通常是使用 Impala 中的 SELECT FROM 语句导入数据。

```
INSERT INTO my_first_table SELECT * FROM temp1;
```

2 更新数据

```
UPDATE my_first_table SET name="xiaowang" where id =1 ;
```

3 删除数据

```
delete from my_first_table where id =2;
```

4 更改表属性

4.1重命名impala表

```
ALTER TABLE PERSON RENAME TO person_temp;
```

 4.2重新命名内部表的基础kudu表

  				4.1.1创建内部表

~~~
CREATE TABLE kudu_student
(
CompanyId INT,
WorkId INT,
Name STRING,
Gender STRING,
Photo STRING,
PRIMARY KEY(CompanyId)
)
PARTITION BY HASH PARTITIONS 16
STORED AS KUDU
TBLPROPERTIES (
'kudu.master_addresses' = 'node1:7051,node2:7051,node3:7051',
'kudu.table_name' = 'student'
);

~~~

​				4.1.2如果表是内部表，则可以通过更改 kudu.table_name 属性重命名底层的 Kudu 表。

```
ALTER TABLE kudu_student SET TBLPROPERTIES('kudu.table_name' = 'new_student');
```

4.3将外部表重新映射kudu表

如果用户在使用过程中发现其他应用程序重新命名了kudu表，那么此时的外部表需要重新映射到kudu上。

首先创建一个外部表：

~~~
CREATE EXTERNAL TABLE external_table
    STORED AS KUDU
    TBLPROPERTIES (
    'kudu.master_addresses' = 'node1:7051,node2:7051,node3:7051',
    'kudu.table_name' = 'person'
);

~~~

重新映射外部表，指向不同的kudu表：

~~~
ALTER TABLE external_table
SET TBLPROPERTIES('kudu.table_name' = 'hashTable')

~~~

上面的操作是：将external_table映射的PERSON表重新指向hashTable表。

4.4 更改kudu master地址

~~~
ALTER TABLE my_table
SET TBLPROPERTIES('kudu.master_addresses' = 'kudu-new-master.example.com:7051');

~~~

4.5 将内部表改为外部表

~~~
ALTER TABLE my_table SET TBLPROPERTIES('EXTERNAL' = 'TRUE');
~~~

##4 impala使用Java操作kudu

对于impala而言，开发人员是可以通过JDBC连接impala的，有了JDBC，开发人员可以通过impala来间接操作 kudu。

### 1 引入依赖

~~~~
       <!--impala的jdbc操作--> 
	   <dependency>
            <groupId>com.cloudera</groupId>
            <artifactId>ImpalaJDBC41</artifactId>
            <version>2.5.42</version>
        </dependency>

        <!--Caused by : ClassNotFound : thrift.protocol.TPro-->
        <dependency>
            <groupId>org.apache.thrift</groupId>
            <artifactId>libfb303</artifactId>
            <version>0.9.3</version>
            <type>pom</type>
        </dependency>

        <!--Caused by : ClassNotFound : thrift.protocol.TPro-->
        <dependency>
            <groupId>org.apache.thrift</groupId>
            <artifactId>libthrift</artifactId>
            <version>0.9.3</version>
            <type>pom</type>
        </dependency>
       
        <dependency>
            <groupId>org.apache.hive</groupId>
            <artifactId>hive-jdbc</artifactId>
            <exclusions>
                <exclusion>
                    <groupId>org.apache.hive</groupId>
                    <artifactId>hive-service-rpc</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.apache.hive</groupId>
                    <artifactId>hive-service</artifactId>
                </exclusion>
            </exclusions>
            <version>1.1.0</version>
        </dependency>

        <!--导入hive-->
        <dependency>
            <groupId>org.apache.hive</groupId>
            <artifactId>hive-service</artifactId>
            <version>1.1.0</version>
        </dependency>

~~~~

### 2 jdbc连接impala操作kudu

使用JDBC连接impala操作kudu，与JDBC连接mysql做更重增删改查基本一样。

创建实体类

~~~
package cn.itcast.impala.impala;

public class Person {
    private int companyId;
    private int workId;
    private  String name;
    private  String gender;
    private  String photo;

    public Person(int companyId, int workId, String name, String gender, String photo) {
        this.companyId = companyId;
        this.workId = workId;
        this.name = name;
        this.gender = gender;
        this.photo = photo;
    }

    public int getCompanyId() {
        return companyId;
    }

    public void setCompanyId(int companyId) {
        this.companyId = companyId;
    }

    public int getWorkId() {
        return workId;
    }

    public void setWorkId(int workId) {
        this.workId = workId;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getGender() {
        return gender;
    }

    public void setGender(String gender) {
        this.gender = gender;
    }

    public String getPhoto() {
        return photo;
    }

    public void setPhoto(String photo) {
        this.photo = photo;
    }
}

~~~

##### JDBC连接impala对kudu进行增删改查

~~~~java
package cn.itcast.impala.impala;

import java.sql.*;

public class Contants {
    private static String JDBC_DRIVER="com.cloudera.impala.jdbc41.Driver";
    private static  String CONNECTION_URL="jdbc:impala://node1:21050/default;auth=noSasl";
     //定义数据库连接
    static Connection conn=null;
    //定义PreparedStatement对象
    static PreparedStatement ps=null;
    //定义查询的结果集
    static ResultSet rs= null;


    //数据库连接
    public static Connection getConn(){
        try {
            Class.forName(JDBC_DRIVER);
            conn=DriverManager.getConnection(CONNECTION_URL);
        } catch (Exception e) {
            e.printStackTrace();
        }

        return  conn;
        
    }

    //创建一个表
    public static void createTable(){
        conn=getConn();
        String sql="CREATE TABLE impala_kudu_test" +
                "(" +
                "companyId BIGINT," +
                "workId BIGINT," +
                "name STRING," +
                "gender STRING," +
                "photo STRING," +
                "PRIMARY KEY(companyId)" +
                ")" +
                "PARTITION BY HASH PARTITIONS 16 " +
                "STORED AS KUDU " +
                "TBLPROPERTIES (" +
                "'kudu.master_addresses' = 'node1:7051,node2:7051,node3:7051'," +
                "'kudu.table_name' = 'impala_kudu_test'" +
                ");";

        try {
            ps = conn.prepareStatement(sql);
            ps.execute();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }


    //查询数据
    public static ResultSet queryRows(){
        try {
            //定义执行的sql语句
            String sql="select * from impala_kudu_test";
            ps = getConn().prepareStatement(sql);
            rs= ps.executeQuery();
        } catch (SQLException e) {
            e.printStackTrace();
        }

        return  rs;
    }

    //打印结果
    public  static void printRows(ResultSet rs){
        /**
         private int companyId;
         private int workId;
         private  String name;
         private  String gender;
         private  String photo;
         */

        try {
            while (rs.next()){
                //获取表的每一行字段信息
                int companyId = rs.getInt("companyId");
                int workId = rs.getInt("workId");
                String name = rs.getString("name");
                String gender = rs.getString("gender");
                String photo = rs.getString("photo");
                System.out.print("companyId:"+companyId+" ");
                System.out.print("workId:"+workId+" ");
                System.out.print("name:"+name+" ");
                System.out.print("gender:"+gender+" ");
                System.out.println("photo:"+photo);

            }
        } catch (SQLException e) {
            e.printStackTrace();
        }finally {
            if(ps!=null){
                try {
                    ps.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }

            if(conn !=null){
                try {
                    conn.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }
        }
    }


    //插入数据
    public static void insertRows(Person person){
        conn=getConn();
        String sql="insert into table impala_kudu_test(companyId,workId,name,gender,photo) values(?,?,?,?,?)";

        try {
            ps=conn.prepareStatement(sql);
            //给占位符？赋值
            ps.setInt(1,person.getCompanyId());
            ps.setInt(2,person.getWorkId());
            ps.setString(3,person.getName());
            ps.setString(4,person.getGender());
            ps.setString(5,person.getPhoto());
            ps.execute();

        } catch (SQLException e) {
            e.printStackTrace();
        }finally {
            if(ps !=null){
                try {
                    //关闭
                    ps.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }

            if(conn !=null){
                try {
                      //关闭
                    conn.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }
        }

    }

    //更新数据
    public static void updateRows(Person person){
       //定义执行的sql语句
        String sql="update impala_kudu_test set workId="+person.getWorkId()+
                ",name='"+person.getName()+"' ,"+"gender='"+person.getGender()+"' ,"+
                "photo='"+person.getPhoto()+"' where companyId="+person.getCompanyId();

        try {
            ps= getConn().prepareStatement(sql);
            ps.execute();
        } catch (SQLException e) {
            e.printStackTrace();
        }finally {
            if(ps !=null){
                try {
                      //关闭
                    ps.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }

            if(conn !=null){
                try {
                      //关闭
                    conn.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    //删除数据
    public   static void deleteRows(int companyId){
      
        //定义sql语句
        String sql="delete from impala_kudu_test where companyId="+companyId;
        try {
            ps =getConn().prepareStatement(sql);
            ps.execute();
        } catch (SQLException e) {
            e.printStackTrace();

        }
    }
    
   //删除表
    public static void dropTable() {
        String sql="drop table if exists impala_kudu_test";
        try {
            ps =getConn().prepareStatement(sql);
            ps.execute();
        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}

~~~~

##### 代码测试运行

~~~java
package cn.itcast.impala.impala;

import java.sql.Connection;

public class ImpalaJdbcClient {
    public static void main(String[] args) {
        Connection conn = Contants.getConn();

        //创建一个表
       Contants.createTable();

        //插入数据
       Contants.insertRows(new Person(1,100,"lisi","male","lisi-photo"));

        //查询表的数据
        ResultSet rs = Contants.queryRows();
        Contants.printRows(rs);

        //更新数据
        Contants.updateRows(new Person(1,200,"zhangsan","male","zhangsan-photo"));

        //删除数据
        Contants.deleteRows(1);

        //删除表
        Contants.dropTable();

    }
}

~~~

# 六 Apache KUDU的原理

## 1． table与schema

Kudu设计是面向结构化存储的，因此，Kudu的表需要用户在建表时定义它的Schema信息，这些Schema信息包含：列定义（含类型），Primary Key定义（用户指定的若干个列的有序组合）。数据的唯一性，依赖于用户所提供的Primary Key中的Column组合的值的唯一性。Kudu提供了Alter命令来增删列，但位于Primary Key中的列是不允许删除的。 

从用户角度来看，Kudu是一种存储结构化数据表的存储系统。在一个Kudu集群中可以定义任意数量的table，每个table都需要预先定义好schema。每个table的列数是确定的，每一列都需要有名字和类型，每个表中可以把其中一列或多列定义为主键。这么看来，Kudu更像关系型数据库，而不是像HBase、Cassandra和MongoDB这些NoSQL数据库。不过Kudu目前还不能像关系型数据一样支持二级索引。

Kudu使用确定的列类型，而不是类似于NoSQL的“everything is byte”。带来好处：确定的列类型使Kudu可以进行类型特有的编码,可以提供元数据给其他上层查询工具。

## 2 kudu底层数据模型

Kudu的底层数据文件的存储，未采用HDFS这样的较高抽象层次的分布式文件系统，而是自行开发了一套可基于Table/Tablet/Replica视图级别的底层存储系统。

这套实现基于如下的几个设计目标：

• 可提供快速的列式查询 

• 可支持快速的随机更新 

• 可提供更为稳定的查询性能保障

![img](/images/kudu/dcs.png)

一张table会分成若干个tablet，每个tablet包括MetaData元信息及若干个RowSet。

RowSet包含一个MemRowSet及若干个DiskRowSet，DiskRowSet中包含一个BloomFile、Ad_hoc Index、BaseData、DeltaMem及若干个RedoFile和UndoFile。

MemRowSet：用于新数据insert及已在MemRowSet中的数据的更新，一个MemRowSet写满后会将数据刷到磁盘形成若干个DiskRowSet。默认是1G或者或者120S。

DiskRowSet：用于老数据的变更，后台定期对DiskRowSet做compaction，以删除没用的数据及合并历史数据，减少查询过程中的IO开销。

BloomFile：根据一个DiskRowSet中的key生成一个bloom filter，用于快速模糊定位某个key是否在DiskRowSet中。

Ad_hocIndex：是主键的索引，用于定位key在DiskRowSet中的具体哪个偏移位置。

BaseData是MemRowSet flush下来的数据，按列存储，按主键有序。

UndoFile是基于BaseData之前时间的历史数据，通过在BaseData上apply UndoFile中的记录，可以获得历史数据。

RedoFile是基于BaseData之后时间的变更记录，通过在BaseData上apply RedoFile中的记录，可获得较新的数据。

DeltaMem用于DiskRowSet中数据的变更，先写到内存中，写满后flush到磁盘形成RedoFile。

REDO与UNDO与关系型数据库中的REDO与UNDO日志类似（在关系型数据库中，REDO日志记录了更新后的数据，可以用来恢复尚未写入Data File的已成功事务更新的数据。而UNDO日志用来记录事务更新之前的数据，可以用来在事务失败时进行回滚）

![img](/images/kudu/dcs2.png)

MemRowSets可以对比理解成HBase中的MemStore, 而DiskRowSets可理解成HBase中的HFile。

MemRowSets中的数据被Flush到磁盘之后，形成DiskRowSets。 DisRowSets中的数据，按照32MB大小为单位，按序划分为一个个的DiskRowSet。 DiskRowSet中的数据按照Column进行组织，与Parquet类似。

这是Kudu可支持一些分析性查询的基础。每一个Column的数据被存储在一个相邻的数据区域，而这个数据区域进一步被细分成一个个的小的Page单元，与HBase File中的Block类似，对每一个Column Page可采用一些Encoding算法，以及一些通用的Compression算法。 既然可对Column Page可采用Encoding以及Compression算法，那么，对单条记录的更改就会比较困难了。

前面提到了Kudu可支持单条记录级别的更新/删除，是如何做到的？

与HBase类似，也是通过增加一条新的记录来描述这次更新/删除操作的。DiskRowSet是不可修改了，那么 KUDU 要如何应对数据的更新呢？在KUDU中，把DiskRowSet分为了两部分：base data、delta stores。base data 负责存储基础数据，delta stores负责存储 base data 中的变更数据.

![img](/images/kudu/dcs3.png)

如上图所示，数据从MemRowSet 刷到磁盘后就形成了一份 DiskRowSet（只包含 base data），每份 DiskRowSet 在内存中都会有一个对应的 DeltaMemStore，负责记录此 DiskRowSet 后续的数据变更（更新、删除）。DeltaMemStore 内部维护一个 B-树索引，映射到每个 row_offset 对应的数据变更。DeltaMemStore 数据增长到一定程度后转化成二进制文件存储到磁盘，形成一个 DeltaFile，随着 base data 对应数据的不断变更，DeltaFile 逐渐增长。

## 3 tablet发现过程

当创建Kudu客户端时，其会从主服务器上获取tablet位置信息，然后直接与服务于该tablet的服务器进行交谈。

为了优化读取和写入路径，客户端将保留该信息的本地缓存，以防止他们在每个请求时需要查询主机的tablet位置信息。随着时间的推移，客户端的缓存可能会变得过时，并且当写入被发送到不再是tablet领导者的tablet服务器时，则将被拒绝。然后客户端将通过查询主服务器发现新领导者的位置来更新其缓存。

![img](/images/kudu/f.png)

## 4 kudu写流程

当 Client 请求写数据时，先根据主键从Master Server中获取要访问的目标 Tablets，然后到依次对应的Tablet获取数据。

因为KUDU表存在主键约束，所以需要进行主键是否已经存在的判断，这里就涉及到之前说的索引结构对读写的优化了。一个Tablet中存在很多个RowSets，为了提升性能，我们要尽可能地减少要扫描的RowSets数量。

首先，我们先通过每个 RowSet 中记录的主键的（最大最小）范围，过滤掉一批不存在目标主键的RowSets，然后在根据RowSet中的布隆过滤器，过滤掉确定不存在目标主键的 RowSets，最后再通过RowSets中的 B-树索引，精确定位目标主键是否存在。

如果主键已经存在，则报错（主键重复），否则就进行写数据（写 MemRowSet）。

![img](/images/kudu/w.png)

## 5kudu读流程

数据读取过程大致如下：先根据要扫描数据的主键范围，定位到目标的Tablets，然后读取Tablets 中的RowSets。

在读取每个RowSet时，先根据主键过滤要scan范围，然后加载范围内的base data，再找到对应的delta stores，应用所有变更，最后union上MemRowSet中的内容，返回数据给Client。

![img](/images/kudu/r.png)

## 6kudu更新流程

数据更新的核心是定位到待更新数据的位置，这块与写入的时候类似，就不展开了，等定位到具体位置后，然后将变更写到对应的delta store 中。

![img](/images/kudu/updata.png)