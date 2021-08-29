---
title: Hadoop
tags:
  - Hadoop
abbrlink: 31153
date: 2017-07-05 16:17:10
categories: Hadoop
summary_img:
encrypt:
enc_pwd:
---

# Hadoop

## 一概述

狭义上来说Hadoop就是:

- HDFS    ：分布式文件系统
- MapReduce : 分布式计算系统
- Yarn：分布式样集群资源管理 

广义上来说:

Hadoop指代大数据的一个生态圈,包括很多其他软件:

![img](/images/hadoop/1558225014064.png)

### 历史版本与发行公司

 2.1 Hadoop历史版本

1.x版本系列：hadoop版本当中的第二代开源版本，主要修复0.x版本的一些bug等

2.x版本系列：架构产生重大变化，引入了yarn平台等许多新特性

3.x版本系列:  加入多namenoode新特性

##### 2.2 Hadoop三大发行版公司

- 免费开源版本apache:

<http://hadoop.apache.org/>

优点：拥有全世界的开源贡献者，代码更新迭代版本比较快，

缺点：版本的升级，版本的维护，版本的兼容性，版本的补丁都可能考虑不太周到，

apache所有软件的下载地址（包括各种历史版本）：

<http://archive.apache.org/dist/>

- 免费开源版本hortonWorks：

<https://hortonworks.com/>

hortonworks主要是雅虎主导Hadoop开发的副总裁，带领二十几个核心成员成立Hortonworks，核心产品软件HDP（ambari），HDF免费开源，并且提供一整套的web管理界面，供我们可以通过web界面管理我们的集群状态，web管理界面软件HDF网址（<http://ambari.apache.org/>）

- 软件收费版本ClouderaManager:

<https://www.cloudera.com/>

cloudera主要是美国一家大数据公司在apache开源hadoop的版本上，通过自己公司内部的各种补丁，实现版本之间的稳定运行，大数据生态圈的各个版本的软件都提供了对应的版本，解决了版本的升级困难，版本兼容性等各种问题

## 二架构

#### 1.x的版本架构模型介绍

![img](/images/hadoop/1558232908565.png)	

文件系统核心模块：

NameNode：集群当中的主节点，管理元数据(文件的大小，文件的位置，文件的权限)，主要用于管理集群当中的各种数据

secondaryNameNode：主要能用于hadoop当中元数据信息的辅助管理

DataNode：集群当中的从节点，主要用于存储集群当中的各种数据

数据计算核心模块：

JobTracker：接收用户的计算请求任务，并分配任务给从节点

TaskTracker：负责执行主节点JobTracker分配的任务

#### 2.x的版本架构模型介绍

引入了yarn,其中MapReduce运行在yarn中

**第一种：NameNode与ResourceManager单节点架构模型**

![img](/images/hadoop/1558232924095.png)	

文件系统核心模块：

NameNode：集群当中的主节点，主要用于管理集群当中的各种数据

secondaryNameNode：主要能用于hadoop当中元数据信息的辅助管理                                                                                                

DataNode：集群当中的从节点，主要用于存储集群当中的各种数据

数据计算核心模块：

ResourceManager：接收用户的计算请求任务，并负责集群的资源分配

NodeManager：负责执行主节点APPmaster分配的任务

**第二种：NameNode单节点与ResourceManager高可用架构模型**

![img](/images/hadoop/1558232966712.png)	

文件系统核心模块：

NameNode：集群当中的主节点，主要用于管理集群当中的各种数据

secondaryNameNode：主要能用于hadoop当中元数据信息的辅助管理

DataNode：集群当中的从节点，主要用于存储集群当中的各种数据

数据计算核心模块：

ResourceManager：接收用户的计算请求任务，并负责集群的资源分配，以及计算任务的划分，通过zookeeper实现ResourceManager的高可用

NodeManager：负责执行主节点ResourceManager分配的任务

**第三种：NameNode高可用与ResourceManager单节点架构模型**

![img](/images/hadoop/1558232980575.png)	

文件系统核心模块：

NameNode：集群当中的主节点，主要用于管理集群当中的各种数据，其中nameNode可以有两个，形成高可用状态

DataNode：集群当中的从节点，主要用于存储集群当中的各种数据

JournalNode：文件系统元数据信息管理

数据计算核心模块：

ResourceManager：接收用户的计算请求任务，并负责集群的资源分配，以及计算任务的划分

NodeManager：负责执行主节点ResourceManager分配的任务

**第四种：NameNode高可用与ResourceManager高可用架构模型**

![img](/images/hadoop/1558232995675.png)	

文件系统核心模块：

NameNode：集群当中的主节点，主要用于管理集群当中的各种数据，一般都是使用两个，实现HA高可用

JournalNode：元数据信息管理进程，一般都是奇数个

DataNode：从节点，用于数据的存储

数据计算核心模块：

ResourceManager：Yarn平台的主节点，主要用于接收各种任务，通过两个，构建成高可用

NodeManager：Yarn平台的从节点，主要用于处理ResourceManager分配的任务

## 三Apache hadoop 编译



## 四 安装Apache Hadoop

例如 以三台服务为例:

**节点规划:**

| 服务器IP             | 192.168.174.*** | 192.168.174.*** | 192.168.174.*** |
| ----------------- | --------------- | --------------- | --------------- |
| 主机名               | node01          | node02          | node03          |
| NameNode          | 是               | 否               | 否               |
| SecondaryNameNode | 是               | 否               | 否               |
| dataNode          | 是               | 是               | 是               |
| ResourceManager   | 是               | 否               | 否               |
| NodeManager       | 是               | 是               | 是               |

 解压:

~~~
tar -zxvf hadoop-2.7.5.tar.gz -C ../servers/
~~~

### 2 修改配置文件

配置文件位置:

~~~~
cd  /export/servers/hadoop-2.7.5/etc/hadoop
vim  core-site.xml
~~~~



修改core-site.xml文件

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

<!-- Put site-specific property overrides in this file. -->

<configuration>
	<!--  指定集群的文件系统类型:分布式文件系统 -->
	<property>
		<name>fs.default.name</name>
		<value>hdfs://node01:8020</value>
	</property>
	<!--  指定临时文件存储目录 -->
	<property>
		<name>hadoop.tmp.dir</name>
		<value>/export/servers/hadoop-2.7.5/hadoopDatas/tempDatas</value>
	</property>
	<!--  缓冲区大小，实际工作中根据服务器性能动态调整 -->
	<property>
		<name>io.file.buffer.size</name>
		<value>4096</value>
	</property>

	<!--  开启hdfs的垃圾桶机制，删除掉的数据可以从垃圾桶中回收，单位分钟 -->
	<property>
		<name>fs.trash.interval</name>
		<value>10080</value>
	</property>
</configuration>

~~~

修改hdfs-site.xml

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->

<!-- Put site-specific property overrides in this file. -->

<configuration>

	 <property>
			<name>dfs.namenode.secondary.http-address</name>
			<value>node01:50090</value>
	</property>

	<!-- 指定namenode的访问地址和端口 -->
	<property>
		<name>dfs.namenode.http-address</name>
		<value>node01:50070</value>
	</property>
	<!-- 指定namenode元数据的存放位置 -->
	<property>
		<name>dfs.namenode.name.dir</name>
		<value>file:///export/servers/hadoop-2.7.5/hadoopDatas/namenodeDatas,file:///export/servers/hadoop-2.7.5/hadoopDatas/namenodeDatas2</value>
	</property>
	<!--  定义dataNode数据存储的节点位置，实际工作中，一般先确定磁盘的挂载目录，然后多个目录用，进行分割  -->
	<property>
		<name>dfs.datanode.data.dir</name>
		<value>file:///export/servers/hadoop-2.7.5/hadoopDatas/datanodeDatas,file:///export/servers/hadoop-2.7.5/hadoopDatas/datanodeDatas2</value>
	</property>
	
	<!-- 指定namenode日志文件的存放目录 -->
	<property>
		<name>dfs.namenode.edits.dir</name>
		<value>file:///export/servers/hadoop-2.7.5/hadoopDatas/nn/edits</value>
	</property>
	

	<property>
		<name>dfs.namenode.checkpoint.dir</name>
		<value>file:///export/servers/hadoop-2.7.5/hadoopDatas/snn/name</value>
	</property>
	<property>
		<name>dfs.namenode.checkpoint.edits.dir</name>
		<value>file:///export/servers/hadoop-2.7.5/hadoopDatas/dfs/snn/edits</value>
	</property>
	<!-- 文件切片的副本个数-->
	<property>
		<name>dfs.replication</name>
		<value>3</value>
	</property>

	<!-- 设置HDFS的文件权限-->
	<property>
		<name>dfs.permissions</name>
		<value>true</value>
	</property>

	<!-- 设置一个文件切片的大小：128M-->
	<property>
		<name>dfs.blocksize</name>
		<value>134217728</value>
	</property>
</configuration>

~~~

修改hadoop-env.sh

~~~xml
export JAVA_HOME=/export/servers/jdk1.8.0_141
~~~

修改mapred-site.xml

~~~xml
<configuration>
    
     <!-- 如果启用了该功能，则会将一个“小的application”的所有子task在同一个JVM里面执行，达到JVM重用的目的。这个JVM便是负责该application的ApplicationMaster所用的JVM -->
  <!--mapreduce.job.ubertask.maxmaps | 9 | map任务数的阀值，如果一个application包含的map数小于该值的定义，那么该application就会被认为是一个小的application-->
  <!-- mapreduce.job.ubertask.maxreduces | 1 | reduce任务数的阀值，如果一个application包含的reduce数小于该值的定义，那么该application就会被认为是一个小的application。不过目前Yarn不支持该值大于1的情况 -->
  <!-- mapreduce.job.ubertask.maxbytes | | application的输入大小的阀值。默认为dfs.block.size的值。当实际的输入大小部超过该值的设定，便会认为该application为一个小的application。 -->
    
	<!-- 开启MapReduce小任务模式 -->
	<property>
		<name>mapreduce.job.ubertask.enable</name>
		<value>true</value>
	</property>
	
	<!-- 设置历史任务的主机和端口 -->
	<property>
		<name>mapreduce.jobhistory.address</name>
		<value>node01:10020</value>
	</property>

	<!-- 设置网页访问历史任务的主机和端口 -->
	<property>
		<name>mapreduce.jobhistory.webapp.address</name>
		<value>node01:19888</value>
	</property>
    
    <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
	</property>

</configuration>

~~~

修改yarn-site.xml

~~~xml
<configuration>
	<!-- 配置yarn主节点的位置 -->
	<property>
		<name>yarn.resourcemanager.hostname</name>
		<value>node01</value>
	</property>
	<property>
		<name>yarn.nodemanager.aux-services</name>
		<value>mapreduce_shuffle</value>
	</property>
	
	<!-- 开启日志聚合功能 -->
	<property>
		<name>yarn.log-aggregation-enable</name>
		<value>true</value>
	</property>
	<!-- 设置聚合日志在hdfs上的保存时间 -->
	<property>
		<name>yarn.log-aggregation.retain-seconds</name>
		<value>604800</value>
	</property>
	<!-- 设置yarn集群的内存分配方案 -->
	<property>    
		<name>yarn.nodemanager.resource.memory-mb</name>    
		<value>20480</value>
	</property>

	<property>  
        	 <name>yarn.scheduler.minimum-allocation-mb</name>
         	<value>2048</value>
	</property>
	<property>
		<name>yarn.nodemanager.vmem-pmem-ratio</name>
		<value>2.1</value>
	</property>

</configuration>

~~~

修改mapred-env.sh

~~~sh
export JAVA_HOME=/export/servers/jdk1.8.0_141
~~~

修改slaves

~~~~XML
node01
node02
node03
~~~~

创建目录:

~~~~shell
mkdir -p /export/servers/hadoop-2.7.5/hadoopDatas/tempDatas
mkdir -p /export/servers/hadoop-2.7.5/hadoopDatas/namenodeDatas
mkdir -p /export/servers/hadoop-2.7.5/hadoopDatas/namenodeDatas2
mkdir -p /export/servers/hadoop-2.7.5/hadoopDatas/datanodeDatas
mkdir -p /export/servers/hadoop-2.7.5/hadoopDatas/datanodeDatas2
mkdir -p /export/servers/hadoop-2.7.5/hadoopDatas/nn/edits
mkdir -p /export/servers/hadoop-2.7.5/hadoopDatas/snn/name
mkdir -p /export/servers/hadoop-2.7.5/hadoopDatas/dfs/snn/edits
~~~~

安装包分发:

~~~
scp -r hadoop-2.7.5 node02:$PWD
scp -r hadoop-2.7.5 node03:$PWD
~~~

配置Hadoop环境变量

~~~
vim  /etc/profile
export HADOOP_HOME=/export/servers/hadoop-2.7.5
export PATH=:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
source /etc/profile
~~~

#### 第四步：启动集群

要启动 Hadoop 集群，需要启动 HDFS 和 YARN 两个模块。
注意： 首次启动 HDFS 时，必须对其进行格式化操作。 本质上是一些清理和
准备工作，因为此时的 HDFS 在物理上还是不存在的。

hdfs namenode -format 或者 hadoop namenode –format

准备启动

第一台机器执行以下命令

```shell
cd  /export/servers/hadoop-2.7.5/
bin/hdfs namenode -format
sbin/start-dfs.sh
sbin/start-yarn.sh
sbin/mr-jobhistory-daemon.sh start historyserver
```

三个端口查看界面

[http://node01:50070/explorer.html#/](#/)  查看hdfs

<http://node01:8088/cluster>   查看yarn集群

<http://node01:19888/jobhistory>  查看历史完成的任务