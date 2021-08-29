---
title: Impala
abbrlink: 26116
date: 2017-07-11 01:42:24
tags: Impala
categories: Impala
summary_img:
encrypt:
enc_pwd:
---

## 一 简介

​		impala来自于cloudera，后来贡献给了apache

​	  **impala**是cloudera提供的一款高效率的sql查询工具，提供实时的查询效果，官方测试性能比hive快10到100倍，其sql查询比sparkSQL还要更加快速，号称是当前大数据领域最快的查询sql工具，

​	  impala是参照谷歌的新三篇论文（Caffeine--网络搜索引擎、Pregel--分布式图计算、Dremel--交互式分析工具）当中的Dremel实现而来，其中旧三篇论文分别是（BigTable，GFS，MapReduce）分别对应我们即将学的HBase和已经学过的HDFS以及MapReduce。

​		**impala是基于hive并使用内存进行计算**，兼顾数据仓库，具有实时，批处理，多并发等优点。

### 1impala与hive的关系:

**impala工作底层执行依赖于hive  与hive共用一套元数据存储。在使用impala的时候，必须保证hive服务是正常可靠的，至少metastore开启。**

impala是基于hive的大数据分析查询引擎，直接使用hive的元数据库metadata，意味着impala元数据都存储在hive的metastore当中，并且impala兼容hive的绝大多数sql语法。所以需要安装impala的话，必须先安装hive，保证hive安装成功，并且还需要启动hive的metastore服务。

Hive元数据包含用Hive创建的database、table等元信息。元数据存储在关系型数据库中，如Derby、MySQL等。

客户端连接metastore服务，metastore再去连接MySQL数据库来存取元数据。有了metastore服务，就可以有多个客户端同时连接，而且这些客户端不需要知道MySQL数据库的用户名和密码，只需要连接metastore 服务即可。

**Hive适合于长时间的批处理查询分析，而Impala适合于实时交互式SQL查询。可以先使用hive进行数据转换处理，之后使用Impala在Hive处理后的结果数据集上进行快速的数据分析。**

### 2 impala与hive的异同

相同:

数据表元数据、ODBC/JDBC驱动、SQL语法、灵活的文件格式、存储资源池等

不同:

**impala最大的跟hive的不同在于 不在把sql编译成mr程序执行 编译成执行计划树**

**impala的运行分为前端和后端  前端为java实现  后端为c++实现**

### impala的优化技术:

使用LLVM产生运行代码，针对特定查询生成特定代码，同时使用Inline的方式减少函数调用的开销，加快执行效率。(C++特性)

充分利用可用的硬件指令（SSE4.2）。

更好的IO调度，Impala知道数据块所在的磁盘位置能够更好的利用多磁盘的优势，同时Impala支持直接数据块读取和本地代码计算checksum。

通过选择合适数据存储格式可以得到最好性能（Impala支持多种存储格式）。

最大使用内存，中间结果不写磁盘，及时通过网络以stream的方式传递。

### 执行计划:

**Hive**: 依赖于MapReduce执行框架，执行计划分成 map->shuffle->reduce->map->shuffle->reduce…的模型。如果一个Query会 被编译成多轮MapReduce，则会有更多的写中间结果。由于MapReduce执行框架本身的特点，过多的中间过程会增加整个Query的执行时间。

**Impala**: 把执行计划表现为一棵完整的执行计划树，可以更自然地分发执行计划到各个Impalad执行查询，而不用像Hive那样把它组合成管道型的 map->reduce模式，以此保证Impala有更好的并发性和避免不必要的中间sort与shuffle。

### 数据流:

**Hive**: 采用推的方式，每一个计算节点计算完成后将数据主动推给后续节点。

**Impala**: 采用拉的方式，后续节点通过getNext主动向前面节点要数据，以此方式数据可以流式的返回给客户端，且只要有1条数据被处理完，就可以立即展现出来，而不用等到全部处理完成，更符合SQL交互式查询使用。

### 内存使用:

**Hive**: 在执行过程中如果内存放不下所有数据，则会使用外存，以保证Query能顺序执行完。每一轮MapReduce结束，中间结果也会写入HDFS中，同样由于MapReduce执行架构的特性，shuffle过程也会有写本地磁盘的操作。

**Impala**: 1.0.1，而不会利用外存，以后版本应该会进行改进。这使用得目前处理会受到一定的限制，最好还是与配合使用

### 调度:

**Hive**: 任务调度依赖于Hadoop的调度策略。

**Impala**: 调度由自己完成，目前只有一种调度器simple-schedule，它会尽量满足数据的局部性，扫描数据的进程尽量靠近数据本身所在的物理机器。调度器 目前还比较简单，在SimpleScheduler::GetBackend中可以看到，现在还没有考虑负载，网络IO状况等因素进行调度。但目前 Impala已经有对执行过程的性能统计分析，应该以后版本会利用这些统计信息进行调度吧。

### 容错:

**Hive**: 依赖于Hadoop的容错能力。

**Impala**: 在查询过程中，没有容错逻辑，如果在执行过程中发生故障，则直接返回错误（**这与Impala的设计有关，因为Impala定位于实时查询，一次查询失败， 再查一次就好了，再查一次的成本很低**）。

### 使用地:

**Hive**: 复杂的批处理查询任务，数据转换任务。

**Impala**：实时数据分析，因为不支持UDF，能处理的问题域有一定的限制，与Hive配合使用,对Hive的结果数据集进行实时分析。

### 架构:

Impala主要由Impalad、 State Store、Catalogd和CLI组成。

- impala  可以集群部署
  - Impalad(impala server):可以部署多个不同机器上，通常与datanode部署在同一个节点 方便数据本地计算，负责具体执行本次查询sql的impalad称之为Coordinator。每个impala server都可以对外提供服务。
  - impala state store:主要是保存impalad的状态信息 监视其健康状态
  - impala catalogd :metastore维护的网关 负责跟hive 的metastore进行交互  同步hive的元数据到impala自己的元数据中。
  - CLI:用户操作impala的方式（impala shell、jdbc、hue）
- impala 查询处理流程
  - impalad分为java前端（接受解析sql编译成执行计划树），c++后端（负责具体的执行计划树操作）
  - impala sql---->impalad（Coordinator）---->调用java前端编译sql成计划树------>以Thrift数据格式返回给C++后端------>根据执行计划树、数据位于路径（libhdfs和hdfs交互）、impalad状态分配执行计划 查询----->汇总查询结果----->返回给java前端---->用户cli
  - 跟hive不同就在于整个执行中已经没有了mapreduce程序的存在

## 二安装部署

前提集群提前安装好Hadoop和hive

hive安装包scp在所有需要安装impala的节点上，因为impala需要引用hive的依赖包。

hadoop框架需要支持C程序访问接口，Hadoop/native，如果有该路径下有这么文件，就证明支持C接口。

1 下载安装包和依赖包

由于impala没有提供tar包进行安装，只提供了rpm包。因此在安装impala的时候，需要使用rpm包来进行安装。rpm包只有cloudera公司提供了，所以去cloudera公司网站进行下载rpm包即可。

但是另外一个问题，impala的rpm包依赖非常多的其他的rpm包，可以一个个的将依赖找出来，也可以将所有的rpm包下载下来，制作成我们本地yum源来进行安装。这里就选择制作本地的yum源来进行安装。

所以首先需要下载到所有的rpm包，下载地址如下

```
http://archive.cloudera.com/cdh5/repo-as-tarball/5.14.0/cdh5.14.0-centos6.tar.gz
```

2 配置本地yam源

### 1.1． 上传安装包解压

使用sftp的方式把安装包大文件上传到服务器/cloudera_data目录下。

```
cd /cloudera_data
tar -zxvf cdh5.14.0-centos6.tar.gz
```

### 1.2． 配置本地yum源信息

安装Apache Server服务器

```
yum  -y install httpd
service httpd start
chkconfig httpd on
```

 

配置本地yum源的文件

```
cd /etc/yum.repos.d
vim localimp.repo 


[localimp]
name=localimp
baseurl=http://node-3/cdh5.14.0/
gpgcheck=0
enabled=1

```



创建apache  httpd的读取链接

```
ln -s /cloudera_data/cdh/5.14.0 /var/www/html/cdh5.14.0
```

**确保****linux****的****Selinux****关闭**



 

通过浏览器访问本地yum源，如果出现下述页面则成功。

```
node03/cdh5.14.0
```



将本地yum源配置文件localimp.repo发放到所有需要安装impala的节点。

```
cd /etc/yum.repos.d/
scp localimp.repo  node02:$PWD
scp localimp.repo  node01:$PWD
```

3 安装 impala

节点规划

| 服务名称                   | 从节点    | 从节点    | 主节点    |
| ---------------------- | ------ | ------ | ------ |
| impala-catalog         |        |        | Node-3 |
| impala-state-store     |        |        | Node-3 |
| impala-server(impalad) | Node-1 | Node-2 | Node-3 |

### 1.1． 主节点安装

在规划的**主节点****node-3**执行以下命令进行安装：

```
  yum install   -y impala impala-server impala-state-store impala-catalog impala-shell   
```



### 1.2． 从节点安装

在规划的**从节点****node-1****、****node-2**执行以下命令进行安装：

```
yum install -y impala-server
```

配置hive和Hadoop

1 hive

```
vim /export/servers/hive/conf/hive-site.xml
<configuration> 
  <property> 
    <name>javax.jdo.option.ConnectionURL</name>  
    <value>jdbc:mysql://node-1:3306/hive?createDatabaseIfNotExist=true</value> 
  </property>  
  <property> 
    <name>javax.jdo.option.ConnectionDriverName</name>  
    <value>com.mysql.jdbc.Driver</value> 
  </property>  
  <property> 
    <name>javax.jdo.option.ConnectionUserName</name>  
    <value>root</value> 
  </property>  
  <property> 
    <name>javax.jdo.option.ConnectionPassword</name>  
    <value>hadoop</value> 
  </property>  
  <property> 
    <name>hive.cli.print.current.db</name>  
    <value>true</value> 
  </property>  
  <property> 
    <name>hive.cli.print.header</name>  
    <value>true</value> 
  </property>  
  <!-- 绑定运行hiveServer2的主机host,默认localhost -->  
  <property> 
    <name>hive.server2.thrift.bind.host</name>  
    <value>node-1</value> 
  </property>  
  <!-- 指定hive metastore服务请求的uri地址 -->  
  <property> 
    <name>hive.metastore.uris</name>  
    <value>thrift://node-1:9083</value> 
  </property>  
  <property> 
    <name>hive.metastore.client.socket.timeout</name>  
    <value>3600</value> 
  </property> 
</configuration>

```

将hive安装包cp给其他两个机器。

```
cd /export/servers/
scp -r hive/ node-2:$PWD
scp -r hive/ node-3:$PWD
```

### 1.1． 修改hadoop配置

所有节点创建下述文件夹

```
mkdir -p /var/run/hdfs-sockets
 
```

修改所有节点的hdfs-site.xml添加以下配置，修改完之后重启hdfs集群生效

```
vim   etc/hadoop/hdfs-site.xml
```



dfs.client.read.shortcircuit 打开DFSClient本地读取数据的控制，

dfs.domain.socket.path``是``Datanode``和``DFSClient``之间沟通的``Socket的本地路径。

```
 
 
```

把更新hadoop的配置文件，scp给其他机器。

```
cd /export/servers/hadoop-2.7.5/etc/hadoop
scp -r hdfs-site.xml node-2:$PWD
scp -r hdfs-site.xml node-3:$PWD 
```

注意：root用户不需要下面操作，普通用户需要这一步操作。

给这个文件夹赋予权限，如果用的是普通用户hadoop，那就直接赋予普通用户的权限，例如：

```
chown  -R  hadoop:hadoop   /var/run/hdfs-sockets/
```

因为这里直接用的root用户，所以不需要赋权限了。

### 1.1． 重启hadoop、hive

在node-1上执行下述命令分别启动hive metastore服务和hadoop。

```
cd  /export/servers/hive
nohup bin/hive --service metastore &
nohup bin/hive --service hiveserver2 &

清空日志文件：cat /dev/null > nohup.out

启动hiveserver2 的服务  
nohup  bin/hive --service  hiveserver2 &

再启动metastore服务
nohup  bin/hive --service  metastore &


如果先启动hiveserver2成功再启动metastore报错怎么办？？
先把hiveserver2的服务杀死，然后先启动metastore  再启动hiveserver2 


如果先启动metastore成功再启动hiveserver2报错怎么办？？
先把metastore的服务杀死，然后先启动hiveserver2  再启动metastore
 
cd /export/servers/hadoop-2.7.5/
sbin/stop-dfs.sh  |  sbin/start-dfs.sh
```

### 1.2． 复制hadoop、hive配置文件

impala的配置目录为/etc/impala/conf，这个路径下面需要把core-site.xml，hdfs-site.xml以及hive-site.xml。

所有节点执行以下命令

```
cp -r /export/servers/hadoop-2.7.5/etc/hadoop/core-site.xml /etc/impala/conf/core-site.xml
cp -r /export/servers/hadoop-2.7.5/etc/hadoop/hdfs-site.xml /etc/impala/conf/hdfs-site.xml
cp -r /export/servers/hive/conf/hive-site.xml /etc/impala/conf/hive-site.xml
```

## 1． 修改impala配置

### 1.1． 修改impala默认配置

所有节点更改impala默认配置文件

```
vim /etc/default/impala
IMPALA_CATALOG_SERVICE_HOST=node-3
IMPALA_STATE_STORE_HOST=node-3
```

### 1.2． 添加mysql驱动

通过配置/etc/default/impala中可以发现已经指定了mysql驱动的位置名字。

​                                                  

使用软链接指向该路径即可（3台机器都需要执行）

```
ln -s /export/servers/hive/lib/mysql-connector-java-5.1.32.jar /usr/share/java/mysql-connector-java.jar
```

### 1.3． 修改bigtop配置

修改bigtop的java_home路径（3台机器）

```
vim /etc/default/bigtop-utils
export JAVA_HOME=/export/servers/jdk1.8.0_65
```

## 1． 启动、关闭impala服务

主节点node-3启动以下三个服务进程

```
service impala-state-store start
service impala-catalog start
service impala-server start
```

 

从节点启动node-1与node-2启动impala-server

```
service  impala-server  start
 
```

查看impala进程是否存在

```
ps -ef | grep impala
``
启动之后所有关于impala的**日志默认都在****/var/log/impala** 

如果需要关闭impala服务 把命令中的start该成stop即可。注意如果关闭之后进程依然驻留，可以采取下述方式删除。正常情况下是随着关闭消失的。

解决方式：

![im](D:\blog\myblog\source\images\impala\im.png)

访问impalad的管理界面node03:25000

访问statestored的管理界面node03:25010





## 四 impala的shell参数


```