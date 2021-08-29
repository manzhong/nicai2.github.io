---
title: Hue
abbrlink: 44390
date: 2017-07-13 09:27:33
tags: Hue
categories: Hue
summary_img:
encrypt:
enc_pwd:
---

# 										Hue

## 一概述

HUE=**Hadoop User Experience**

Hue是一个开源的Apache Hadoop UI系统，由Cloudera Desktop演化而来，最后Cloudera公司将其贡献给Apache基金会的Hadoop社区，它是基于Python Web框架Django实现的。

用于与其他各个框架做整合,提供一个web 界面可以供我们去操作其他的大数据框架.

其实就是一个与其他大数据框架做整合的工具,他本身并不提供任何功能,这些功能由他整合的框架完成

## 二架构

第一个UI界面：主要是提供我们web界面供我们使用的
第二个hue  server：就是一个tomcat的服务
第三个hue  DB: hue的数据库，主要用于保存一起我们提交的任务

核心功能:

·        SQL编辑器，支持Hive, Impala, MySQL, Oracle, PostgreSQL, SparkSQL, Solr SQL, Phoenix…

·        搜索引擎Solr的各种图表

·        Spark和Hadoop的友好界面支持

·        支持调度系统Apache Oozie，可进行workflow的编辑、查看

HUE提供的这些功能相比Hadoop生态各组件提供的界面更加友好，但是一些需要debug的场景可能还是需要使用原生系统才能更加深入的找到错误的原因。

HUE中查看Oozie workflow时，也可以很方便的看到整个workflow的DAG图，不过在最新版本中已经将DAG图去掉了，只能看到workflow中的action列表和他们之间的跳转关系，想要看DAG图的仍然可以使用oozie原生的界面系统查看。

支持的框架:

1，访问HDFS和文件浏览 

2，通过web调试和开发hive以及数据结果展示 

3，查询solr和结果展示，报表生成 

4，通过web调试和开发impala交互式SQL Query 

5，spark调试和开发 

7，oozie任务的开发，监控，和工作流协调调度 

8，Hbase数据查询和修改，数据展示 

9，Hive的元数据（metastore）查询 

10，MapReduce任务进度查看，日志追踪 

11，创建和提交MapReduce，Streaming，Java job任务 

12，Sqoop2的开发和调试 

13，Zookeeper的浏览和编辑 

14，数据库（，，，）的查询和展示

Hue是一个友好的界面集成框架，可以集成我们各种学习过的以及将要学习的框架，一个界面就可以做到查看以及执行所有的框架.

## 三 hue 的安装

安装方式:

1 .RPM

2.tar.gz 安装

3.clouder mananger 安装

这里使用tar.gz安装

1 下载解压

~~~shell
tar -zxvf hue-3.9.0-cdh5.14.0.tar.gz -C ../servers/
~~~

2 编译安装启动

2.1  linux安装依赖包

~~~shell
yum install ant asciidoc cyrus-sasl-devel cyrus-sasl-gssapi cyrus-sasl-plain gcc gcc-c++ krb5-devel libffi-devel libxml2-devel libxslt-devel make  mysql mysql-devel openldap-devel python-devel sqlite-devel gmp-devel
~~~

2.2 配值 hue

~~~shell
cd /hue/desktop/conf
vim hue.ini
~~~

通用配置: 在标签 [desktop]

~~~~properties
secret_key=jFE93j;2[290-eiw.KEiwN2s3['d;/.q[eIW^y#e=+Iei*@Mn<qW5o
http_host=node03.hadoop.com
is_hue_4=true
time_zone=Asia/Shanghai
server_user=root
server_group=root
default_user=root
default_hdfs_superuser=root
~~~~

\#配置使用mysql作为hue的存储数据库,大概在hue.ini的587行左右 [database]

~~~properties
engine=mysql
host=node03.hadoop.com
port=3306
user=root
password=123456
name=hue
~~~

2.3 配置mysql数据库

~~~sql
create database hue default character set utf8 default collate utf8_general_ci;
~~~

**注意实际工作中需要为hue 这个数据库创建对应的用户并分配权限,此处不用创建**

~~~sql
grant all on hue.* to 'hue'@'%' identified by 'hue';
~~~

2.4 准备编译

~~~shell
cd /hue
make apps
~~~

2.5 linux 添加普通用户

~~~shell
useradd hue
passwd hue
~~~

2.6 启动hue

~~~shell
cd /hue
bulid/env/bin/supervisor
~~~

2.7 页面访问

http://node03:8888

第一次访问的时候，需要设置管理员用户和密码

我们这里的管理员的用户名与密码尽量保持与我们安装hadoop的用户名和密码一致，

我们安装hadoop的用户名与密码分别是root  ***

初次登录使用root用户，密码为****

## 四 hue与其他框架的集成

## 1hue 与hadoop的集成

### 1.1 hue 与hdfs的集成

注意修改完HDFS相关配置后，需要把配置scp给集群中每台机器，重启hdfs集群。

修改 hadoop 中的core-site.xml

~~~~
<!--允许通过httpfs方式访问hdfs的主机名 -->
<property>
<name>hadoop.proxyuser.root.hosts</name>
<value>*</value>
</property>
<!—允许通过httpfs方式访问hdfs的用户组 -->
<property>
<name>hadoop.proxyuser.root.groups</name>
<value>*</value>
</property>

~~~~

修改hadoop 中的hdfs-site.xml

~~~
<property>
	  <name>dfs.webhdfs.enabled</name>
	  <value>true</value>
</property>

~~~

修改hue.ini文件

```
cd /export/servers/hue-3.9.0-cdh5.14.0/desktop/conf
vim hue.ini

[[hdfs_clusters]]
    [[[default]]]
fs_defaultfs=hdfs://node-1:9000
webhdfs_url=http://node-1:50070/webhdfs/v1
hadoop_hdfs_home= /export/servers/hadoop-2.7.5
hadoop_bin=/export/servers/hadoop-2.7.5/bin
hadoop_conf_dir=/export/servers/hadoop-2.7.5/etc/hadoop

```

重启hdfs与hue

### 2 与yarn集成

修改hue.ini 文件

~~~
[[yarn_clusters]]
    [[[default]]]
      resourcemanager_host=node-1
      resourcemanager_port=8032
      submit_to=True
      resourcemanager_api_url=http://node-1:8088
      history_server_api_url=http://node-1:19888

~~~

开启yarn日志聚集服务

修改Hadoop中的 yarn-site.xml

~~~~
<property>  ##是否启用日志聚集功能。
<name>yarn.log-aggregation-enable</name>
<value>true</value>
</property>
<property>  ##设置日志保留时间，单位是秒。
<name>yarn.log-aggregation.retain-seconds</name>
<value>106800</value>
</property>

~~~~

重启yarn 与hue

## 2 hue集成hive

如果需要配置hue与hive的集成，我们需要启动hive的metastore服务以及hiveserver2服务（impala需要hive的metastore服务，hue需要hvie的hiveserver2服务）。

1 修改hue.ini文件

~~~
[beeswax]
  hive_server_host=node-1
  hive_server_port=10000
  hive_conf_dir=/export/servers/hive/conf
  server_conn_timeout=120
  auth_username=root
  auth_password=123456

[metastore]
  #允许使用hive创建数据库表等操作
  enable_new_create_table=true

~~~

2 启动hIive 和hue

去node-1机器上启动hive的metastore以及hiveserver2服务

```
cd /export/servers/hive
nohup bin/hive --service metastore &
nohup bin/hive --service hiveserver2 &
```

重新启动hue。 

```
cd /export/servers/hue-3.9.0-cdh5.14.0/
build/env/bin/supervisor
```

## 3 hue 集成MySQL

1 修改hue.ini文件

需要把mysql的注释给去掉。 大概位于1546行

~~~
[[[mysql]]]
      nice_name="My SQL DB"
      engine=mysql
      host=node-1
      port=3306
      user=root
      password=hadoop

~~~

2重启hue

## 4 hue集成Oozie

1修改hue.ini 文件

~~~
[liboozie]
  # The URL where the Oozie service runs on. This is required in order for
  # users to submit jobs. Empty value disables the config check.
  oozie_url=http://node-1:11000/oozie

  # Requires FQDN in oozie_url if enabled
  ## security_enabled=false

  # Location on HDFS where the workflows/coordinator are deployed when submitted.
  remote_deployement_dir=/user/root/oozie_works

~~~

~~~~
[oozie]
  # Location on local FS where the examples are stored.
  # local_data_dir=/export/servers/oozie-4.1.0-cdh5.14.0/examples/apps

  # Location on local FS where the data for the examples is stored.
  # sample_data_dir=/export/servers/oozie-4.1.0-cdh5.14.0/examples/input-data

  # Location on HDFS where the oozie examples and workflows are stored.
  # Parameters are $TIME and $USER, e.g. /user/$USER/hue/workspaces/workflow-$TIME
  # remote_data_dir=/user/root/oozie_works/examples/apps

  # Maximum of Oozie workflows or coodinators to retrieve in one API call.
  oozie_jobs_count=100

  # Use Cron format for defining the frequency of a Coordinator instead of the old frequency number/unit.
  enable_cron_scheduling=true

  # Flag to enable the saved Editor queries to be dragged and dropped into a workflow.
  enable_document_action=true

  # Flag to enable Oozie backend filtering instead of doing it at the page level in Javascript. Requires Oozie 4.3+.
  enable_oozie_backend_filtering=true

  # Flag to enable the Impala action.
  enable_impala_action=true

~~~~

~~~
[filebrowser]
  # Location on local filesystem where the uploaded archives are temporary stored.
  archive_upload_tempdir=/tmp

  # Show Download Button for HDFS file browser.
  show_download_button=true

  # Show Upload Button for HDFS file browser.
  show_upload_button=true

  # Flag to enable the extraction of a uploaded archive in HDFS.
  enable_extract_uploaded_archive=true

~~~

启动hue和oozie

启动hue进程

```
cd /export/servers/hue-3.9.0-cdh5.14.0
build/env/bin/supervisor
```

启动oozie进程

```
cd /export/servers/oozie-4.1.0-cdh5.14.0
bin/oozied.sh start
```

## 5 hue集成impala

修改配置文件

~~~
[impala]
  server_host=node-3
  server_port=21050
  impala_conf_dir=/etc/impala/conf

~~~

启动impala 和hue

