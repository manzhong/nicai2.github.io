---
title: Azkaban
abbrlink: 2409
date: 2017-07-10 09:00:17
tags: Azkaban
categories: Azkaban
summary_img:
encrypt:
enc_pwd:
---

																						# 												Azkaban

## 一 概述

是由领英推出的一款开源免费的工作流调度器软件.

特点:

- 功能强大  可以调度几乎所有软件的执行（command）
- 配置简单  job配置文件
- 提供了web页面使用
- 提供模块化和可插拔的插件机制，原生支持command、Java、Hive、Pig、Hadoop
- java语言开发 源码清晰可见  可以进行二次开发

工作流概述:

​		工作流（Workflow），指“业务过程的部分或整体在计算机应用环境下的**自动化**”。

​		一个完整的数据分析系统通常都是由多个前后依赖的模块组合构成的：数据采集、数据预处理、数据分析、数据展示等。各个模块单元之间存在时间先后依赖关系，且存在着周期性重复。为了很好地组织起这样的复杂执行计划，需要一个工作流调度系统来调度执行。

实现方式:

​			简单的任务调度：直接使用linux的crontab来定义,但是缺点也是比较明显，无法设置依赖。

​			复杂的任务调度：自主开发调度平台，使用开源调度系统，比如**azkaban**、Apache Oozie、Cascading、Hamake等。其中知名度比较高的是Apache Oozie，但是其配置工作流的过程是编写大量的XML配置，而且代码复杂度比较高，不易于二次开发。

工作流之间的对比:

| 特性             | Hamake               | Oozie             | Azkaban                        | Cascading |
| -------------- | -------------------- | ----------------- | ------------------------------ | --------- |
| 工作流描述语言        | XML                  | XML (xPDL based)  | text file with key/value pairs | Java API  |
| 依赖机制           | data-driven          | explicit          | explicit                       | explicit  |
| 是否要web容器       | No                   | Yes               | Yes                            | No        |
| 进度跟踪           | console/log messages | web page          | web page                       | Java API  |
| Hadoop job调度支持 | no                   | yes               | yes                            | yes       |
| 运行模式           | command line utility | daemon            | daemon                         | API       |
| Pig支持          | yes                  | yes               | yes                            | yes       |
| 事件通知           | no                   | no                | no                             | yes       |
| 需要安装           | no                   | yes               | yes                            | no        |
| 支持的hadoop版本    | 0.18+                | 0.20+             | currently unknown              | 0.18+     |
| 重试支持           | no                   | workflownode evel | yes                            | yes       |
| 运行任意命令         | yes                  | yes               | yes                            | yes       |
| Amazon EMR支持   | yes                  | no                | currently unknown              | yes       |

##二 Azkaban调度器

###1原理架构:

mysql服务器: 存储元数据，如项目名称、项目描述、项目权限、任务状态、SLA规则等

AzkabanWebServer: 对外提供web服务，使用户可以通过web页面管理。职责包括项目管理、权限授权、任务调度、监控executor。

AzkabanExecutorServer: 负责具体的工作流的提交、执行。

### 2部署方式(3种)

- 单节点模式：web、executor在同一个进程  适用于测试体验 使用自带的H2数据库
- two-server: web、executor在不同的进程中  **可以使用第三方数据库**
- mutil-executor-server:  web、executor在不同的机器上 **可以部署多个executor服务器** 可使用第三方数据库

### 3源码编译

1.Azkaban3.x在安装前需要自己编译成二进制包。并且提前安装好Maven,Ant,Node等软件

2 编译环境

~~~shell
yum install –y git

yum install –y gcc-c++
~~~

3 下载源码解析

~~~shell
wget https://github.com/azkaban/azkaban/archive/3.51.0.tar.gz
tar -zxvf 3.51.0.tar.gz 
cd ./azkaban-3.51.0/
~~~

4 编译源码

~~~
./gradlew build installDist -x test
~~~

Gradle是一个基于ApacheAnt和ApacheMaven的项目自动化构建工具。-x test 跳过测试。（注意联网下载jar可能会失败、慢）

5 编译后安装包路径

solo-server模式安装包路径

~~~
azkaban-solo-server/build/distributions/
~~~

two-server模式和multiple-executor模式web-server安装包路径

~~~
azkaban-web-server/build/distributions/
~~~

two-server模式和multiple-executor模式exec-server安装包路径

~~~
azkaban-exec-server/build/distributions/
~~~

数据库相关安装包路径

~~~
azkaban-db/build/distributions/
~~~

##三 安装部署

### 1单节点模式

~~~~shell
mkdir /export/servers/azkaban
tar -zxvf azkaban-solo-server-0.1.0-SNAPSHOT.tar.gz –C /export/servers/azkaban/

vim conf/azkaban.properties
default.timezone.id=Asia/Shanghai #修改时区

vim  plugins/jobtypes/commonprivate.properties
添加：memCheck.enabled=false
azkaban默认需要3G的内存，剩余内存不足则会报异常

~~~~

启动验证:

~~~shell
cd azkaban-solo-server-0.1.0-SNAPSHOT/
bin/start-solo.sh
注:启动/关闭必须进到azkaban-solo-server-0.1.0-SNAPSHOT/目录下。
~~~

AzkabanSingleServer(对于Azkaban solo‐server模式，Exec Server和Web Server在同一个进程中)

登录页面:

~~~properties
http://node01:8081/
默认密码和用户名为  azkaban
~~~

测试:

创建两个文件one.job ,two.job,内容如下，打包成zip包。

~~~properties
##one.job
cat one.job 
    type=command                        ##命令类型
    command=echo "this is job one"

##two.job 
cat two.job 
    type=command
    dependencies=one                     ## 依赖于one.job one.job 执行完后 在执行自己的
    command=echo "this is job two"

~~~

Create Project=>Upload zip包=>execute flow执行一步步操作即可。

- 上传zip压缩包
- 选择调度schduler或者立即执行executor工程。

### 2 two server 模式

节点规划:

~~~
node03        mysql
node02        web-server和exec-server不同进程
~~~

node03 上mysql配置初始化:

~~~
tar -zxvf azkaban-db-0.1.0-SNAPSHOT.tar.gz –C /export/servers/azkaban/

Mysql上创建对应的库、增加权限、创建表
mysql> CREATE DATABASE azkaban_two_server; #创建数据库
mysql> use azkaban_two_server;
mysql> source /export/servers/azkaban/azkaban-db-0.1.0-SNAPSHOT/create-all-sql-0.1.0-SNAPSHOT.sql;
#加载初始化sql创建表

~~~

node02上 配置

~~~
tar -zxvf azkaban-web-server-0.1.0-SNAPSHOT.tar.gz –C /export/servers/azkaban/
tar -zxvf azkaban-exec-server-0.1.0-SNAPSHOT.tar.gz –C /export/servers/azkaban/
~~~

###2.1web配置:

生成ssl证书：

**服务器证书,将ssl证书安装在网站服务器上可以实现网站身份和数据加密传输双重功能**

~~~
keytool -keystore keystore -alias jetty -genkey -keyalg RSA
~~~

运行此命令后,会提示输入当前生成keystore的密码及相应信息,输入的密码请记住(**所有密码统一以123456输入**)。

完成上述工作后,将在当前目录生成keystore证书文件,将keystore拷贝到 azkaban web服务器根目录中。

配置 web服务器的 conf/azkaban.properties：

~~~properties
# Azkaban Personalization Settings
azkaban.name=Test
azkaban.label=My Local Azkaban
azkaban.color=#FF3601
azkaban.default.servlet.path=/index
web.resource.dir=web/
# 1要修改的地方
default.timezone.id=Asia/Shanghai
# Azkaban UserManager class
user.manager.class=azkaban.user.XmlUserManager
user.manager.xml.file=conf/azkaban-users.xml
# Loader for projects
executor.global.properties=conf/global.properties
azkaban.project.dir=projects
# Velocity dev mode
velocity.dev.mode=false
# Azkaban Jetty server properties.
# 2要修改的地方
jetty.use.ssl=true
jetty.ssl.port=8443
jetty.maxThreads=25
jetty.port=8081
# Azkaban Executor settings
#3要修改的地方
executor.host=localhost
executor.port=12321

#  KeyStore for SSL ssl相关配置  注意密码和证书路径
# 4要修改的地方
jetty.keystore=keystore
jetty.password=123456
jetty.keypassword=123456
jetty.truststore=keystore
jetty.trustpassword=123456
# mail settings
mail.sender=
mail.host=
# User facing web server configurations used to construct the user facing server URLs. They are useful when there is a reverse proxy between Azkaban web servers and users.
# enduser -> myazkabanhost:443 -> proxy -> localhost:8081
# when this parameters set then these parameters are used to generate email links.
# if these parameters are not set then jetty.hostname, and jetty.port(if ssl configured jetty.ssl.port) are used.
# azkaban.webserver.external_hostname=myazkabanhost.com
# azkaban.webserver.external_ssl_port=443
# azkaban.webserver.external_port=8081
job.failure.email=
job.success.email=
lockdown.create.projects=false
cache.directory=cache
# JMX stats
jetty.connector.stats=true
executor.connector.stats=true
# Azkaban mysql settings by default. Users should configure their own username and password.
#5要修改的地方
database.type=mysql
mysql.port=3306
mysql.host=node03
mysql.database=azkaban_two_server
mysql.user=root
mysql.password=123456
mysql.numconnections=100
#Multiple Executor
azkaban.use.multiple.executors=true
# 6修改的地方 注释掉这一行 放弃检查内存
#azkaban.executorselector.filters=StaticRemainingFlowSize,MinimumFreeMemory,CpuStatus
azkaban.executorselector.comparator.NumberOfAssignedFlowComparator=1
azkaban.executorselector.comparator.Memory=1
azkaban.executorselector.comparator.LastDispatched=1
azkaban.executorselector.comparator.CpuUsage=1

~~~

在web 的根目录下创建 目录:

~~~
mkdir -p plugins/jobtypes
~~~

在这个目录下:

~~~
vim commonprivate.properties
#本地库
azkaban.native.lib=false

execute.as.user=false
#内存检测
memCheck.enabled=false
~~~

### 2.2 exec配置

exec服务器下:conf/azkaban.properties：

~~~~properties
# Azkaban Personalization Settings
azkaban.name=Test
azkaban.label=My Local Azkaban
azkaban.color=#FF3601
azkaban.default.servlet.path=/index
web.resource.dir=web/
# 1要修改
default.timezone.id=Asia/Shanghai
# Azkaban UserManager class
user.manager.class=azkaban.user.XmlUserManager
user.manager.xml.file=conf/azkaban-users.xml
# Loader for projects
executor.global.properties=conf/global.properties
azkaban.project.dir=projects
# Velocity dev mode
velocity.dev.mode=false
# Azkaban Jetty server properties.
jetty.use.ssl=false
jetty.maxThreads=25
jetty.port=8081
# Where the Azkaban web server is located 
# 2要修改的地方
azkaban.webserver.url=https://node02:8443
# mail settings
mail.sender=
mail.host=
# User facing web server configurations used to construct the user facing server URLs. They are useful when there is a reverse proxy between Azkaban web servers and users.
# enduser -> myazkabanhost:443 -> proxy -> localhost:8081
# when this parameters set then these parameters are used to generate email links.
# if these parameters are not set then jetty.hostname, and jetty.port(if ssl configured jetty.ssl.port) are used.
# azkaban.webserver.external_hostname=myazkabanhost.com
# azkaban.webserver.external_ssl_port=443
# azkaban.webserver.external_port=8081
job.failure.email=
job.success.email=
lockdown.create.projects=false
cache.directory=cache
# JMX stats
jetty.connector.stats=true
executor.connector.stats=true
# Azkaban plugin settings
azkaban.jobtype.plugin.dir=plugins/jobtypes
# Azkaban mysql settings by default. Users should configure their own username and password.
# 3要修改的地方
database.type=mysql
mysql.port=3306
mysql.host=node03
mysql.database=azkaban_two_server
mysql.user=root
mysql.password=123456
mysql.numconnections=100
# Azkaban Executor settings
executor.maxThreads=50
# 4要修改的地方 添加
executor.port=12321
executor.flow.threads=30


~~~~

### 2.3启动:

先启动exec-server  在根目录下 bin/start-exec.sh

再启动web-server。 在根目录下 bin/start-web.sh

启动webServer之后进程失败消失，可通过安装包根目录下对应启动日志进行排查。

解决:

需要手动激活executor

到exec服务器的根目录下

向executor.port 发送一个executor 将其action改为activate

~~~shell
curl -G "node02:$(<./executor.port)/executor?action=activate" && echo
~~~

重启web即可

即可在页面测试

**测试阶段可能一直处于running状态**

解决:

在exec服务器下 cd 到 plugins/jobtypes

~~~shell
vim commonprivate.properties
加入以下配置:
memCheck.enabled=false
~~~

重启exec和web  并重新向executor.port 发送一个executor 将其action改为activate

```shell
curl -G "node02:$(<./executor.port)/executor?action=activate" && echo
```



特点:

- 该模式的特点是web服务器和executor服务器分别位于不同的进程中
- 使用第三方的数据库进行数据的保存 ：mysql

注意事项:

- 先对mysql进行初始化操作
- 配置azkaban.properties 注意时区 mysql相关  ssl 
- 启动时候注意需要自己手动的激活executor服务器  在根目录下启动
- 如果启动出错  通过安装包根目录下的日志进行判断
- 访问的页面https

### 3multiple-executor模式部署

multiple-executor模式是多个executor Server分布在不同服务器上，只需要将azkaban-exec-server安装包拷贝到不同机器上即可组成分布式。

节点配置:

~~~properties
node03       mysql
node02       web-server和 exec-server
node01       exec-server
~~~

1  scp executor server安装包到node01

前提 node01 和node02 有相同的目录结构(元数据存在的路径)才可以使用 pwd   否则回传到root路径下

~~~
scp -r azkaban-exec-server-0.1.0-SNAPSHOT/ node01:$PWD
~~~

启动:

先启动exec 分别启动node01 和node02上的exec 在向executor.port 发送一个executor 将其action改为activate

~~~
curl -G "node02:$(<./executor.port)/executor?action=activate" && echo
~~~

~~~
curl -G "node01:$(<./executor.port)/executor?action=activate" && echo
~~~

在启动web即可

- 所谓的  multiple-executor指的是可以在多个机器上分别部署executor服务器

  相当于做了一个负载均衡

- 特别注意：executor启动（包括重启）的时候 默认不会激活 需要自己手动激活

  对应的mysql中的表executors  active ：0 表示未激活 1表示激活

  可以自己手动修改数据提交激活 也可以使用官方的命令请求激活

  ~~~
  curl -G "node01:$(<./executor.port)/executor?action=activate" && echo
  ~~~


## 五 实战

- 理论上任何一款软件，只有可以通过shell command执行 都可以转化成为azkaban的调度执行
- type=command   command = sh xxx.sh

## 1． shell command调度

 创建job描述文件

vi command.job

~~~
#command.job
type=command
command=sh hello.sh
~~~

~~~
hello.sh 内容
#!/bin/bash
date  > /root/hello.txt
~~~

将hello.sh和command.job一起打包为zip

上传azkaban

## 2． job依赖调度

~~~~
# foo.job
type=command
command=echo foo
~~~~

~~~
# bar.job
type=command
dependencies=foo
command=echo bar
~~~

~~~
type=command
dependencies=bar
command=echo baidu
~~~

资源打包 zip上传:

若前一个未成功  后一个不会执行

## 3 hdfs的调度

~~~
# fs.job
type=command
command=sh hdfs.sh
~~~



~~~
#!/bin/bash
/export/servers/hadoop-2.7.5/bin/hadoop fs -mkdir /azaz666
~~~

打包压缩  zip

通过azkaban的web管理平台创建project并上传job压缩包  启动执行该job

## 4 MAPREDUCE任务调度

~~~
# mrwc.job
type=command
command=/home/hadoop/apps/hadoop-2.6.1/bin/hadoop  jar hadoop-mapreduce-examples-2.6.1.jar wordcount /wordcount/input /wordcount/azout

~~~

将jar包和job文件打包zip压缩

上传

## 5hive

hive脚本 test.sql

~~~
use default;
drop table aztest;
create table aztest(id int,name string) row format delimited fields terminated by ',';
load data inpath '/aztest/hiveinput' into table aztest;
create table azres as select * from aztest;
insert overwrite directory '/aztest/hiveoutput' select count(1) from aztest; 

~~~



~~~
# hivef.job
type=command
command=/home/hadoop/apps/hive/bin/hive -f 'test.sql'

~~~

## 6定时任务调度

页面中选择schedule进行设置