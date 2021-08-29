---
title: linux开发环境搭建
abbrlink: 36836
date: 2017-07-03 22:30:33
tags: Linux
categories: Linux
summary_img:
encrypt:
enc_pwd:
---

# linux常用软件安装

如有需要,自行参考

## 一 mysql的安装

第一步：在线安装mysql相关的软件包

~~~shell
yum  install  mysql  mysql-server  mysql-devel
~~~

第二步：启动mysql的服务

    /etc/init.d/mysqld start

第三步：通过mysql安装自带脚本进行设置

    /usr/bin/mysql_secure_installation

第四步：进入mysql的客户端然后进行授权

~~~shell
 grant all privileges on *.* to 'root'@'%' identified by '123456' with grant option;
flush privileges;
~~~

yum 安装的卸载

**一、使用以下命令查看当前安装mysql情况，查找以前是否装有mysql**

```
`rpm -qa|``grep` `-i mysql`
```

显示之前安装了：

​     MySQL-client-5.5.25a-1.rhel5

​     MySQL-server-5.5.25a-1.rhel5

**2、停止mysql服务、删除之前安装的mysql**

service mysqld stop

删除命令：`rpm -e –nodeps 包名`

```
`rpm -ev MySQL-client-5.5.25a-1.rhel5 ``rpm -ev MySQL-server-5.5.25a-1.rhel5`
```

如果提示依赖包错误，则使用以下命令尝试

```
`rpm -ev MySQL-client-5.5.25a-1.rhel5 --nodeps`
```

如果提示错误：`error: %preun(xxxxxx) scriptlet failed, exit status 1`

则用以下命令尝试：

```
`rpm -e --noscripts MySQL-client-5.5.25a-1.rhel5`
```

**3、查找之前老版本mysql的目录、并且删除老版本mysql的文件和库**

```
`find` `/ -name mysql`
```

查找结果如下：

```
`find` `/ -name mysql ` `/var/lib/mysql``/var/lib/mysql/mysql``/usr/lib64/mysql`
```

删除对应的mysql目录

```
`rm` `-rf ``/var/lib/mysql``rm` `-rf ``/var/lib/mysql``rm` `-rf ``/usr/lib64/mysql`
```

**注意：**卸载后/etc/my.cnf不会删除，需要进行手工删除

```
`rm` `-rf ``/etc/my``.cnf`
```

**4、再次查找机器是否安装mysql**

```
`rpm -qa|``grep` `-i mysql`
```

## 二 jdk的安装

1.1 查看自带的openjdk并卸载

    查询
    rpm -qa | grep java
    卸载
    rpm -e java-1.6.0-openjdk-1.6.0.41-1.13.13.1.el6_8.x86_64 tzdata-java-2016j-1.el6.noarch java-1.7.0-openjdk-1.7.0.131-2.6.9.0.el6_8.x86_64 --nodeps
--nodeps  不管依赖直接删

 2上传解压

~~~
 tar -zxvf jdk-8u141-linux-x64.tar.gz
~~~

3配置环境变量

~~~properties
 vim /etc/profile
 
 添加
 export JAVA_HOME=/export/servers/jdk1.8.0_141
 export PATH=:$JAVA_HOME/bin:$PATH
~~~

4 生效

~~~
source /etc/profile
~~~



# zookeeper的安装



# Hadoop的安装

# hive的安装

1上传压塑包并解压

~~~
tar -zxvf apache-hive-2.1.1-bin.tar.gz
~~~

2 解压后可重命名

~~~
mv apache-hive-2.1.1-bin hive
~~~

3 安装mysql省略

4修改配置文件

~~~
cp hive-env.sh.template hive-env.sh

配置
HADOOP_HOME=/export/servers/hadoop-2.7.5
export HIVE_CONF_DIR=/export/servers/hive/conf
~~~

创建文件:

~~~xml
vim hive-site.xml

<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
<property>
      <name>javax.jdo.option.ConnectionUserName</name>
      <value>root</value>
  </property>
  <property>
      <name>javax.jdo.option.ConnectionPassword</name>
      <value>123456</value>
  </property>
  <property>
      <name>javax.jdo.option.ConnectionURL</name>
      <value>jdbc:mysql://node03:3306/hive?createDatabaseIfNotExist=true&amp;useSSL=false</value>
  </property>
  <property>
      <name>javax.jdo.option.ConnectionDriverName</name>
      <value>com.mysql.jdbc.Driver</value>
  </property>
  <property>
      <name>hive.metastore.schema.verification</name>
      <value>false</value>
  </property>
  <property>
    <name>datanucleus.schema.autoCreateAll</name>
    <value>true</value>
 </property>
 <property>
		<name>hive.server2.thrift.bind.host</name>
		<value>node03</value>
   </property>
</configuration>
~~~

5将数据库驱动加入hive下的lib目录下

6配置hive环境变量

~~~xml
sudo vim /etc/profile

export HIVE_HOME=/export/servers/hive
export PATH=:$HIVE_HOME/bin:$PATH

配后:
source /etc/profile
~~~



# flume的安装



# sqoop的安装

安装sqoop的前提是已经具备java和hadoop的环境

上传解压后:

配置文件修改:

conf下的:

~~~
mv sqoop-env-template.sh sqoop-env.sh
vi sqoop-env.sh
export HADOOP_COMMON_HOME= /export/servers/hadoop-2.7.5 
export HADOOP_MAPRED_HOME= /export/servers/hadoop-2.7.5
export HIVE_HOME= /export/servers/hive
~~~

把数据库驱动加入lib目录下

测试:

~~~
bin/sqoop list-databases \
--connect jdbc:mysql://node01:3306/ \
--username root \
--password 123456
~~~

列出数据库中所有的数据库

# azkaban的安装





# telnet 安装

~~~
yum list telnet*              列出telnet相关的安装包
yum install telnet-server          安装telnet服务
yum install telnet.*           安装telnet客户端
~~~

