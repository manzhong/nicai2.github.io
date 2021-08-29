---
title: Oozie
abbrlink: 30855
date: 2017-07-12 08:50:24
tags: Oozie
categories: Oozie
summary_img:
encrypt:
enc_pwd:
---

# 										Oozie

## 一   概述

- 是一个工作流调度软件  本身属于cloudera  后来贡献给了apache
- oozie目的根据一个定义DAG（有向无环图）执行工作流程
- oozie本身的配置是一种xml格式的配置文件   oozie跟hue配合使用将会很方便
- oozie特点：顺序执行 周期重复定时  可视化  追踪结果

# 二 架构

**Oozie Client**：提供命令行、java api、rest等方式，对Oozie的工作流流程的提交、启动、运行等操作；

**Oozie WebApp**：即 Oozie Server,本质是一个java应用。可以使用内置的web容器，也可以使用外置的web容器；

**Hadoop Cluster**：底层执行Oozie编排流程的各个hadoop生态圈组件；launcher Job(single map task  no reduce task)   和actual program(mr hive pig sqark etc)   **oozie各种类型任务提交底层依赖于mr程序 首先启动一个没有reducetak的mr  通过这个mr把各个不同类型的任务提交到具体的集群上执行**

# 三 原理

​			Oozie对工作流的编排，是基于workflow.xml文件来完成的。用户预先将工作流执行规则定制于workflow.xml文件中，并在job.properties配置相关的参数，然后由Oozie Server向MR提交job来启动工作流。

## 1 流程节点

**Control Flow Nodes**：控制工作流执行路径，包括start，end，kill，decision判断的节点，fork 插值 分开执行不同的任务 并行执行,join  合并。

**Action Nodes**：决定每个操作执行的任务类型，包括MapReduce、java、hive、shell等。

![img](/images/Oozie/lc.png)

上述两种类型结合起来 就可以描绘出一个工作流的DAG图。

## 2 工作流类型

- - workflow  基本类型的工作流  只会按照定义顺序执行 无定时触发和条件触发

  - ![img](/images/Oozie/lx1.png)

- coordinator   Coordinator将多个工作流Job组织起来，称为Coordinator
    Job，并指定触发时间和频率，还可以配置数据集、并发数等，类似于在工作流外部增加了一个协调器来管理这些工作流的工作流Job的运行。

  - ![img](/images/Oozie/lx2.png)

  - bundle  针对coordinator的批处理工作流。Bundle将多个Coordinator管理起来，这样我们只需要一个Bundle提交即可。

    ![img](/images/Oozie/lx3.png)

# 四 Apache Oozie 安装

版本问题：Apache官方提供的是源码包 需要自己结合hadoop生态圈软件环境进行编译  兼容性问题特别难以处理  因此可以使用第三方商业公司编译好  Cloudera（CDH）

## 1  修改Hadoop的配置

### 1.1 配置httpfs服务

修改Hadoop的配置文件core-site.xml：

~~~
<property>
        <name>hadoop.proxyuser.root.hosts</name>
        <value>*</value>
</property>
<property>
        <name>hadoop.proxyuser.root.groups</name>
        <value>*</value>
</property>

~~~

`hadoop.proxyuser.root.hosts `允许通过httpfs方式访问hdfs的主机名、域名；

`hadoop.proxyuser.root.groups`允许访问的客户端的用户组

### 1.2 配置jobhistory服务

修改Hadoop的配置文件mapred-site.xml:

~~~~
<property>
  <name>mapreduce.jobhistory.address</name>
  <value>node01:10020</value>
  <description>MapReduce JobHistory Server IPC host:port</description>
</property>

<property>
  <name>mapreduce.jobhistory.webapp.address</name>
  <value>node01:19888</value>
  <description>MapReduce JobHistory Server Web UI host:port</description>
</property>
<!-- 配置运行过的日志存放在hdfs上的存放路径 -->
<property>
    <name>mapreduce.jobhistory.done-dir</name>
    <value>/export/data/history/done</value>
</property>

<!-- 配置正在运行中的日志在hdfs上的存放路径 -->
<property>
    <name>mapreduce.jobhistory.intermediate-done-dir</name>
    <value>/export/data/history/done_intermediate</value>
</property>

~~~~

### 1.3 重启hadoop集群相关服务

启动history-server

mr-jobhistory-daemon.sh start historyserver

停止history-server

mr-jobhistory-daemon.sh stop historyserver

通过浏览器访问Hadoop Jobhistory的WEBUI

~~~
http://node01:19888
~~~

## 2 解压 Oozie的安装包

例如：

~~~
oozie的安装包上传到/export/softwares
tar -zxvf oozie-4.1.0-cdh5.14.0.tar.gz -C ../servers/

解压hadooplibs到与oozie平行的目录
cd /export/servers/oozie-4.1.0-cdh5.14.0
tar -zxvf oozie-hadooplibs-4.1.0-cdh5.14.0.tar.gz -C ../
~~~

## 3 添加相关依赖

~~~
oozie的安装路径下创建libext目录
cd /export/servers/oozie-4.1.0-cdh5.14.0
mkdir -p libext

拷贝hadoop依赖包到libext
cd /export/servers/oozie-4.1.0-cdh5.14.0
cp -ra hadooplibs/hadooplib-2.6.0-cdh5.14.0.oozie-4.1.0-cdh5.14.0/* libext/

上传mysql的驱动包到libext
mysql-connector-java-5.1.32.jar
添加ext-2.2.zip压缩包到libext
ext-2.2.zip
~~~

## 4 修改oozie-site.xml文件

oozie默认使用的是UTC的时区，需要在oozie-site.xml当中配置时区为**GMT+0800**时区

~~~
cd /export/servers/oozie-4.1.0-cdh5.14.0/conf
vim oozie-site.xml

	<property>
        <name>oozie.service.JPAService.jdbc.driver</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
	<property>
        <name>oozie.service.JPAService.jdbc.url</name>
        <value>jdbc:mysql://node03:3306/oozie</value>
    </property>
	<property>
		<name>oozie.service.JPAService.jdbc.username</name>
		<value>root</value>
	</property>
    <property>
        <name>oozie.service.JPAService.jdbc.password</name>
        <value>123456</value>
    </property>
	<property>
			<name>oozie.processing.timezone</name>
			<value>GMT+0800</value>
	</property>

	<property>
        <name>oozie.service.coord.check.maximum.frequency</name>
		<value>false</value>
    </property>     

	<property>
		<name>oozie.service.HadoopAccessorService.hadoop.configurations</name>
        <value>*=/export/servers/hadoop-2.7.5/etc/hadoop</value>
    </property>

~~~

## 5 初始化MySQL相关信息

上传oozie的解压后目录的下的yarn.tar.gz到hdfs目录

```
bin/oozie-setup.sh  sharelib create -fs hdfs://node01:8020 -locallib oozie-sharelib-4.1.0-cdh5.14.0-yarn.tar.gz
```

本质上就是将这些jar包解压到了hdfs上面的路径下面去

创建mysql数据库

```
mysql -uroot -p
create database oozie;
```

初始化创建oozie的数据库表

```
cd /export/servers/oozie-4.1.0-cdh5.14.0
bin/oozie-setup.sh  db create -run -sqlfile oozie.sql
```

## 6 打包项目生成war包

```
cd /export/servers/oozie-4.1.0-cdh5.14.0
bin/oozie-setup.sh  prepare-war
```

## 7 配置oozie 环境变量

~~~~
vim /etc/profile

export OOZIE_HOME=/export/servers/oozie-4.1.0-cdh5.14.0
export OOZIE_URL=http://node01.hadoop.com:11000/oozie
export PATH=$PATH:$OOZIE_HOME/bin

source /etc/profile

~~~~

## 8 启动关闭oozie服务

~~~
启动命令
cd /export/servers/oozie-4.1.0-cdh5.14.0
bin/oozied.sh start 
关闭命令
bin/oozied.sh stop

~~~

注意：

~~~
oozie-4.1.0-cdh5.14.0/oozie-server/temp/oozie.pid
~~~

启动的时候产生的 pid文件，如果是kill方式关闭进程 则需要删除该文件重新启动，否则再次启动会报错。启动的时候产生的 pid文件，如果是kill方式关闭进程 则需要删除该文件重新启动，否则再次启动会报错。

## 9 浏览器web ui 页面

~~~
http://node01:11000/oozie
~~~

## 10 解决oozie 页面显示异常

页面访问的时候，发现oozie使用的还是GMT的时区，与我们现在的时区相差一定的时间，所以需要调整一个js的获取时区的方法，将其改成我们现在的时区。

修改js当中的时区问题

```
cd  oozie-server/webapps/oozie
vim oozie-console.js
```

~~~
function getTimeZone() {
    Ext.state.Manager.setProvider(new Ext.state.CookieProvider());
    return Ext.state.Manager.get("TimezoneId","GMT+0800");
}
~~~

重启oozie服务

~~~
cd /export/servers/oozie-4.1.0-cdh5.14.0
bin/oozied.sh stop
bin/oozied.sh start
~~~

# 五 Oozie的实战

oozie安装好了之后，需要测试oozie的功能是否完整好使，官方已经给自带带了各种测试案例，可以通过官方提供的各种案例来学习oozie的使用，后续也可以把这些案例作为模板在企业实际中使用。

先把官方提供的各种案例给解压出来

```
cd /export/servers/oozie-4.1.0-cdh5.14.0
tar -zxvf oozie-examples.tar.gz
 
```

创建统一的工作目录，便于集中管理oozie。企业中可任意指定路径。这里直接在oozie的安装目录下面创建工作目录

```
cd /export/servers/oozie-4.1.0-cdh5.14.0
mkdir oozie_works
```

## 1 ． 优化更新hadoop相关配置

1.1 yarn 容器资源	分配属性

yarn-site.xml

~~~
<!—节点最大可用内存，结合实际物理内存调整 -->
<property>
        <name>yarn.nodemanager.resource.memory-mb</name>
        <value>3072</value>
</property>
<!—每个容器可以申请内存资源的最小值，最大值 -->
<property>
        <name>yarn.scheduler.minimum-allocation-mb</name>
        <value>1024</value>
</property>
<property>
        <name>yarn.scheduler.maximum-allocation-mb</name>
        <value>3072</value>
</property>

<!—修改为Fair公平调度，动态调整资源，避免yarn上任务等待（多线程执行） -->
<property>
 <name>yarn.resourcemanager.scheduler.class</name>
 <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler</value>
</property>
<!—Fair调度时候是否开启抢占功能 -->
<property>
        <name>yarn.scheduler.fair.preemption</name>
        <value>true</value>
</property>
<!—超过多少开始抢占，默认0.8-->
    <property>
        <name>yarn.scheduler.fair.preemption.cluster-utilization-threshold</name>
        <value>1.0</value>
    </property>

~~~

1.2mapreduce资源申请配置

设置mapreduce.map.memory.mb和mapreduce.reduce.memory.mb配置

否则Oozie读取的默认配置 -1, 提交给yarn的时候会抛异常*Invalid resource request, requested memory < 0, or requested memory > max configured, requestedMemory=-1, maxMemory=8192*

**mapred-site.xml**

~~~
<!—单个maptask、reducetask可申请内存大小 -->
<property>
        <name>mapreduce.map.memory.mb</name>
        <value>1024</value>
</property>
<property>
        <name>mapreduce.reduce.memory.mb</name>
        <value>1024</value>
</property>

~~~

scp给其他机器  重启集群 （hadoop ）  oozie



##1． Oozie调度shell脚本

把shell的任务模板拷贝到oozie的工作目录当中去

```
cd /export/servers/oozie-4.1.0-cdh5.14.0
cp -r examples/apps/shell/ oozie_works/
 
```

准备待调度的shell脚本文件

```
cd /export/servers/oozie-4.1.0-cdh5.14.0
vim oozie_works/shell/hello.sh
```

**注意**：这个脚本一定要是在我们oozie工作路径下的shell路径下的位置

\#!/bin/bash

echo "hello world" >> /export/servers/hello_oozie.txt

修改job.properties

```
cd /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works/shell
vim job.properties

nameNode=hdfs://node01:8020
jobTracker=node01:8032
queueName=default
examplesRoot=oozie_works
oozie.wf.application.path=${nameNode}/user/${user.name}/${examplesRoot}/shell
EXEC=hello.sh

```

**jobTracker**：在hadoop2当中，jobTracker这种角色已经没有了，只有resourceManager，这里给定resourceManager的IP及端口即可。

**queueName**：提交mr任务的队列名；

**examplesRoot**：指定oozie的工作目录；

**oozie.wf.application.path**：指定oozie调度资源存储于hdfs的工作路径；

**EXEC**：指定执行任务的名称。

修改workflow.xml

~~~
<workflow-app xmlns="uri:oozie:workflow:0.4" name="shell-wf">
<start to="shell-node"/>
<action name="shell-node">
    <shell xmlns="uri:oozie:shell-action:0.2">
        <job-tracker>${jobTracker}</job-tracker>
        <name-node>${nameNode}</name-node>
        <configuration>
            <property>
                <name>mapred.job.queue.name</name>
                <value>${queueName}</value>
            </property>
        </configuration>
        <exec>${EXEC}</exec>
        <file>/user/root/oozie_works/shell/${EXEC}#${EXEC}</file>
        <capture-output/>
    </shell>
    <ok to="end"/>
    <error to="fail"/>
</action>
<decision name="check-output">
    <switch>
        <case to="end">
            ${wf:actionData('shell-node')['my_output'] eq 'Hello Oozie'}
        </case>
        <default to="fail-output"/>
    </switch>
</decision>
<kill name="fail">
    <message>Shell action failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
</kill>
<kill name="fail-output">
    <message>Incorrect output, expected [Hello Oozie] but was [${wf:actionData('shell-node')['my_output']}]</message>
</kill>
<end name="end"/>
</workflow-app>

~~~

### 1.1． 上传调度任务到hdfs

注意：上传的hdfs目录为/user/root，因为hadoop启动的时候使用的是root用户，如果hadoop启动的是其他用户，那么就上传到/user/其他用户

```
cd /export/servers/oozie-4.1.0-cdh5.14.0
hdfs dfs -put oozie_works/ /user/root
```

通过oozie的命令来执行调度任务

~~~
cd /export/servers/oozie-4.1.0-cdh5.14.0

bin/oozie job -oozie http://node01:11000/oozie -config oozie_works/shell/job.properties  -run
~~~

杀死任务流:

~~~
bin/oozie job -oozie http://node01:11000/oozie -kill job的ID
~~~



##2． Oozie调度hive

### 2.1 准备模板

```
cd /export/servers/oozie-4.1.0-cdh5.14.0
cp -ra examples/apps/hive2/ oozie_works/
```

2.2 修改模板

~~~
修改job.properties
cd /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works/hive2
vim job.properties


nameNode=hdfs://node01:8020
jobTracker=node01:8032
queueName=default
jdbcURL=jdbc:hive2://node01:10000/default
examplesRoot=oozie_works

oozie.use.system.libpath=true
# 配置我们文件上传到hdfs的保存路径 实际上就是在hdfs 的/user/root/oozie_works/hive2这个路径下
oozie.wf.application.path=${nameNode}/user/${user.name}/${examplesRoot}/hive2

~~~

修改workflow.xml（实际上无修改）

~~~
<?xml version="1.0" encoding="UTF-8"?>
<workflow-app xmlns="uri:oozie:workflow:0.5" name="hive2-wf">
    <start to="hive2-node"/>

    <action name="hive2-node">
        <hive2 xmlns="uri:oozie:hive2-action:0.1">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <prepare>
                <delete path="${nameNode}/user/${wf:user()}/${examplesRoot}/output-data/hive2"/>
                <mkdir path="${nameNode}/user/${wf:user()}/${examplesRoot}/output-data"/>
            </prepare>
            <configuration>
                <property>
                    <name>mapred.job.queue.name</name>
                    <value>${queueName}</value>
                </property>
            </configuration>
            <jdbc-url>${jdbcURL}</jdbc-url>
            <script>script.q</script>
            <param>INPUT=/user/${wf:user()}/${examplesRoot}/input-data/table</param>
            <param>OUTPUT=/user/${wf:user()}/${examplesRoot}/output-data/hive2</param>
        </hive2>
        <ok to="end"/>
        <error to="fail"/>
    </action>

    <kill name="fail">
        <message>Hive2 (Beeline) action failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
    </kill>
    <end name="end"/>
</workflow-app>

~~~

编辑hivesql文件

```
vim script.q
```

~~~
DROP TABLE IF EXISTS test;
CREATE EXTERNAL TABLE test (a INT) STORED AS TEXTFILE LOCATION '${INPUT}';
insert into test values(10);
insert into test values(20);
insert into test values(30);

~~~

上传调度任务到hdfs

```
cd /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works
hdfs dfs -put hive2/ /user/root/oozie_works/
```

执行调度任务

oozie调度hive脚本

- 首先必须保证hiveserve2服务是启动正常的，如果配置metastore服务，要首先启动metastore

  ~~~
  在hive目录下执行
  nohup hive --service metastore &
  nohup hive --service hiveserver2 &
  ~~~

  测试 hive2

  ~~~
  beeline
  ! connect jdbc:hive2//node01:10000    看能否进去  前提node01上的hiveserve2服务是启动正常的
  ~~~


~~~
cd /export/servers/oozie-4.1.0-cdh5.14.0
bin/oozie job -oozie http://node01:11000/oozie -config oozie_works/hive2/job.properties  -run

~~~

## 3 oozie 调度 MapReduce

1准备模板

准备mr程序的待处理数据。用hadoop自带的MR程序来运行wordcount。

准备数据上传到HDFS的/oozie/input路径下去

```
hdfs dfs -mkdir -p /oozie/input
hdfs dfs -put wordcount.txt /oozie/input
```

拷贝MR的任务模板

```
cd /export/servers/oozie-4.1.0-cdh5.14.0
cp -ra examples/apps/map-reduce/ oozie_works/
```

删掉MR任务模板lib目录下自带的jar包

```
cd /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works/map-reduce/lib
rm -rf oozie-examples-4.1.0-cdh5.14.0.jar
 
```

拷贝官方自带mr程序jar包到对应目录

```
cp /export/servers/hadoop-2.7.5/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.5.jar 
 /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works/map-reduce/lib/
```

2 修改模板

修改job.properties

```
cd /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works/map-reduce
vim job.properties

nameNode=hdfs://node01:8020
jobTracker=node01:8032
queueName=default
examplesRoot=oozie_works
oozie.wf.application.path=${nameNode}/user/${user.name}/${examplesRoot}/map-reduce/workflow.xml
outputDir=/oozie/output
inputdir=/oozie/input

```

修改workflow.xml

```
cd /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works/map-reduce
vim workflow.xml

<?xml version="1.0" encoding="UTF-8"?>
<workflow-app xmlns="uri:oozie:workflow:0.5" name="map-reduce-wf">
    <start to="mr-node"/>
    <action name="mr-node">
        <map-reduce>
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <prepare>
                <delete path="${nameNode}/${outputDir}"/>
            </prepare>
            <configuration>
                <property>
                    <name>mapred.job.queue.name</name>
                    <value>${queueName}</value>
                </property>
				<!--  
                <property>
                    <name>mapred.mapper.class</name>
                    <value>org.apache.oozie.example.SampleMapper</value>
                </property>
                <property>
                    <name>mapred.reducer.class</name>
                    <value>org.apache.oozie.example.SampleReducer</value>
                </property>
                <property>
                    <name>mapred.map.tasks</name>
                    <value>1</value>
                </property>
                <property>
                    <name>mapred.input.dir</name>
                    <value>/user/${wf:user()}/${examplesRoot}/input-data/text</value>
                </property>
                <property>
                    <name>mapred.output.dir</name>
                    <value>/user/${wf:user()}/${examplesRoot}/output-data/${outputDir}</value>
                </property>
				-->
				
				   <!-- 开启使用新的API来进行配置 -->
                <property>
                    <name>mapred.mapper.new-api</name>
                    <value>true</value>
                </property>

                <property>
                    <name>mapred.reducer.new-api</name>
                    <value>true</value>
                </property>

                <!-- 指定MR的输出key的类型 -->
                <property>
                    <name>mapreduce.job.output.key.class</name>
                    <value>org.apache.hadoop.io.Text</value>
                </property>

                <!-- 指定MR的输出的value的类型-->
                <property>
                    <name>mapreduce.job.output.value.class</name>
                    <value>org.apache.hadoop.io.IntWritable</value>
                </property>

                <!-- 指定输入路径 -->
                <property>
                    <name>mapred.input.dir</name>
                    <value>${nameNode}/${inputdir}</value>
                </property>

                <!-- 指定输出路径 -->
                <property>
                    <name>mapred.output.dir</name>
                    <value>${nameNode}/${outputDir}</value>
                </property>

                <!-- 指定执行的map类 -->
                <property>
                    <name>mapreduce.job.map.class</name>
                    <value>org.apache.hadoop.examples.WordCount$TokenizerMapper</value>
                </property>

                <!-- 指定执行的reduce类 -->
                <property>
                    <name>mapreduce.job.reduce.class</name>
                    <value>org.apache.hadoop.examples.WordCount$IntSumReducer</value>
                </property>
				<!--  配置map task的个数 -->
                <property>
                    <name>mapred.map.tasks</name>
                    <value>1</value>
                </property>

            </configuration>
        </map-reduce>
        <ok to="end"/>
        <error to="fail"/>
    </action>
    <kill name="fail">
        <message>Map/Reduce failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
    </kill>
    <end name="end"/>
</workflow-app>

```

3 上传调度任务到hdfs 

```
cd /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works hdfs dfs -put map-reduce/ /user/root/oozie_works/
```

4 执行任务

```
cd /export/servers/oozie-4.1.0-cdh5.14.0
bin/oozie job -oozie http://node01:11000/oozie -config oozie_works/map-reduce/job.properties -run
```

注意:

需要在workflow.xml中开启使用新版的 api  hadoop2.x

## 4 oozie 任务串联

在实际工作当中，肯定会存在多个任务需要执行，并且存在上一个任务的输出结果作为下一个任务的输入数据这样的情况，所以我们需要在workflow.xml配置文件当中配置多个action，实现多个任务之间的相互依赖关系。

需求：首先执行一个shell脚本，执行完了之后再执行一个MR的程序，最后再执行一个hive的程序。

1 准备工作目录

~~~
cd /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works
mkdir -p sereval-actions

~~~

2 准备调度文件

将之前的hive，shell， MR的执行，进行串联成到一个workflow当中。

```
cd /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works
cp hive2/script.q sereval-actions/
cp shell/hello.sh sereval-actions/
cp -ra map-reduce/lib sereval-actions/
```

3修改配置模板

```
cd /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works/sereval-actions
vim workflow.xml

<workflow-app xmlns="uri:oozie:workflow:0.4" name="shell-wf">
<start to="shell-node"/>
<action name="shell-node">
    <shell xmlns="uri:oozie:shell-action:0.2">
        <job-tracker>${jobTracker}</job-tracker>
        <name-node>${nameNode}</name-node>
        <configuration>
            <property>
                <name>mapred.job.queue.name</name>
                <value>${queueName}</value>
            </property>
        </configuration>
        <exec>${EXEC}</exec>
        <!-- <argument>my_output=Hello Oozie</argument> -->
        <file>/user/root/oozie_works/sereval-actions/${EXEC}#${EXEC}</file>

        <capture-output/>
    </shell>
    <ok to="mr-node"/>
    <error to="mr-node"/>
</action>

<action name="mr-node">
        <map-reduce>
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <prepare>
                <delete path="${nameNode}/${outputDir}"/>
            </prepare>
            <configuration>
                <property>
                    <name>mapred.job.queue.name</name>
                    <value>${queueName}</value>
                </property>
				<!--  
                <property>
                    <name>mapred.mapper.class</name>
                    <value>org.apache.oozie.example.SampleMapper</value>
                </property>
                <property>
                    <name>mapred.reducer.class</name>
                    <value>org.apache.oozie.example.SampleReducer</value>
                </property>
                <property>
                    <name>mapred.map.tasks</name>
                    <value>1</value>
                </property>
                <property>
                    <name>mapred.input.dir</name>
                    <value>/user/${wf:user()}/${examplesRoot}/input-data/text</value>
                </property>
                <property>
                    <name>mapred.output.dir</name>
                    <value>/user/${wf:user()}/${examplesRoot}/output-data/${outputDir}</value>
                </property>
				-->
				
				   <!-- 开启使用新的API来进行配置 -->
                <property>
                    <name>mapred.mapper.new-api</name>
                    <value>true</value>
                </property>

                <property>
                    <name>mapred.reducer.new-api</name>
                    <value>true</value>
                </property>

                <!-- 指定MR的输出key的类型 -->
                <property>
                    <name>mapreduce.job.output.key.class</name>
                    <value>org.apache.hadoop.io.Text</value>
                </property>

                <!-- 指定MR的输出的value的类型-->
                <property>
                    <name>mapreduce.job.output.value.class</name>
                    <value>org.apache.hadoop.io.IntWritable</value>
                </property>

                <!-- 指定输入路径 -->
                <property>
                    <name>mapred.input.dir</name>
                    <value>${nameNode}/${inputdir}</value>
                </property>

                <!-- 指定输出路径 -->
                <property>
                    <name>mapred.output.dir</name>
                    <value>${nameNode}/${outputDir}</value>
                </property>

                <!-- 指定执行的map类 -->
                <property>
                    <name>mapreduce.job.map.class</name>
                    <value>org.apache.hadoop.examples.WordCount$TokenizerMapper</value>
                </property>

                <!-- 指定执行的reduce类 -->
                <property>
                    <name>mapreduce.job.reduce.class</name>
                    <value>org.apache.hadoop.examples.WordCount$IntSumReducer</value>
                </property>
				<!--  配置map task的个数 -->
                <property>
                    <name>mapred.map.tasks</name>
                    <value>1</value>
                </property>

            </configuration>
        </map-reduce>
        <ok to="hive2-node"/>
        <error to="fail"/>
    </action>






 <action name="hive2-node">
        <hive2 xmlns="uri:oozie:hive2-action:0.1">
            <job-tracker>${jobTracker}</job-tracker>
            <name-node>${nameNode}</name-node>
            <prepare>
                <delete path="${nameNode}/user/${wf:user()}/${examplesRoot}/output-data/hive2"/>
                <mkdir path="${nameNode}/user/${wf:user()}/${examplesRoot}/output-data"/>
            </prepare>
            <configuration>
                <property>
                    <name>mapred.job.queue.name</name>
                    <value>${queueName}</value>
                </property>
            </configuration>
            <jdbc-url>${jdbcURL}</jdbc-url>
            <script>script.q</script>
            <param>INPUT=/user/${wf:user()}/${examplesRoot}/input-data/table</param>
            <param>OUTPUT=/user/${wf:user()}/${examplesRoot}/output-data/hive2</param>
        </hive2>
        <ok to="end"/>
        <error to="fail"/>
    </action>
<decision name="check-output">
    <switch>
        <case to="end">
            ${wf:actionData('shell-node')['my_output'] eq 'Hello Oozie'}
        </case>
        <default to="fail-output"/>
    </switch>
</decision>
<kill name="fail">
    <message>Shell action failed, error message[${wf:errorMessage(wf:lastErrorNode())}]</message>
</kill>
<kill name="fail-output">
    <message>Incorrect output, expected [Hello Oozie] but was [${wf:actionData('shell-node')['my_output']}]</message>
</kill>
<end name="end"/>
</workflow-app>

```

job.properties配置文件

~~~
nameNode=hdfs://node01:8020
jobTracker=node01:8032
queueName=default
examplesRoot=oozie_works
EXEC=hello.sh
outputDir=/oozie/output
inputdir=/oozie/input
jdbcURL=jdbc:hive2://node01:10000/default
oozie.use.system.libpath=true
# 配置我们文件上传到hdfs的保存路径 实际上就是在hdfs 的/user/root/oozie_works/sereval-actions这个路径下
oozie.wf.application.path=${nameNode}/user/${user.name}/${examplesRoot}/sereval-actions/workflow.xml

~~~

4 上传任务调度到hdfs

```
cd /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works/
hdfs dfs -put sereval-actions/ /user/root/oozie_works/
```

5执行任务调度

```
cd /export/servers/oozie-4.1.0-cdh5.14.0/
bin/oozie job -oozie http://node01:11000/oozie -config oozie_works/sereval-actions/job.properties -run
```

通过action节点 成功失败控制执行的流程

如果上一个action成功  跳转到下一个action 这样就可以变成首尾相连的串联任务

## 5 定时调度

在oozie当中，主要是通过Coordinator 来实现任务的定时调度， Coordinator 模块主要通过xml来进行配置即可。

Coordinator 的调度主要可以有两种实现方式

第一种：基于时间的定时任务调度：

oozie基于时间的调度主要需要指定三个参数，第一个起始时间，第二个结束时间，第三个调度频率；

第二种：基于数据的任务调度， 这种是基于数据的调度，只要在有了数据才会触发调度任务。

###1 准备配置模板

第一步：拷贝定时任务的调度模板

```
cd /export/servers/oozie-4.1.0-cdh5.14.0
cp -r examples/apps/cron oozie_works/cron-job
```

 

第二步：拷贝我们的hello.sh脚本

```
cd /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works
cp shell/hello.sh  cron-job/
```

### 2修改配制模板

修改job.properties

```
cd /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works/cron-job
vim job.properties

nameNode=hdfs://node01:8020
jobTracker=node01:8032
queueName=default
examplesRoot=oozie_works

oozie.coord.application.path=${nameNode}/user/${user.name}/${examplesRoot}/cron-job/coordinator.xml
#start：必须设置为未来时间，否则任务失败
start=2019-05-22T19:20+0800
end=2019-08-22T19:20+0800
EXEC=hello.sh
workflowAppUri=${nameNode}/user/${user.name}/${examplesRoot}/cron-job/workflow.xml

```

修改coordinator.xml

```
vim coordinator.xml

<!--
	oozie的frequency 可以支持很多表达式，其中可以通过定时每分，或者每小时，或者每天，或者每月进行执行，也支持可以通过与linux的crontab表达式类似的写法来进行定时任务的执行
	例如frequency 也可以写成以下方式
	frequency="10 9 * * *"  每天上午的09:10:00开始执行任务
	frequency="0 1 * * *"  每天凌晨的01:00开始执行任务
 -->
<coordinator-app name="cron-job" frequency="${coord:minutes(1)}" start="${start}" end="${end}" timezone="GMT+0800"
                 xmlns="uri:oozie:coordinator:0.4">
        <action>
        <workflow>
            <app-path>${workflowAppUri}</app-path>
            <configuration>
                <property>
                    <name>jobTracker</name>
                    <value>${jobTracker}</value>
                </property>
                <property>
                    <name>nameNode</name>
                    <value>${nameNode}</value>
                </property>
                <property>
                    <name>queueName</name>
                    <value>${queueName}</value>
                </property>
            </configuration>
        </workflow>
    </action>
</coordinator-app>

```

修改workflow.xml

```
vim workflow.xml


	<workflow-app xmlns="uri:oozie:workflow:0.5" name="one-op-wf">
    <start to="action1"/>
    <action name="action1">
    <shell xmlns="uri:oozie:shell-action:0.2">
        <job-tracker>${jobTracker}</job-tracker>
        <name-node>${nameNode}</name-node>
        <configuration>
            <property>
                <name>mapred.job.queue.name</name>
                <value>${queueName}</value>
            </property>
        </configuration>
        <exec>${EXEC}</exec>
        <!-- <argument>my_output=Hello Oozie</argument> -->
        <file>/user/root/oozie_works/cron-job/${EXEC}#${EXEC}</file>

        <capture-output/>
    </shell>
    <ok to="end"/>
    <error to="end"/>
</action>
    <end name="end"/>
</workflow-app>

```

### 3上传调度任务到hdfs

```
cd /export/servers/oozie-4.1.0-cdh5.14.0/oozie_works
hdfs dfs -put cron-job/ /user/root/oozie_works/
```

### 4 执行调度

```
cd /export/servers/oozie-4.1.0-cdh5.14.0
bin/oozie job -oozie http://node01:11000/oozie -config oozie_works/cron-job/job.properties –run
```

- oozie基于时间的定时

  主要是需要coordinator来配合workflow进行周期性的触发执行

  需要注意时间的格式问题  时区的问题

