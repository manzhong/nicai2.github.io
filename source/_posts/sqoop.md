---
title: Sqoop
abbrlink: 33510
date: 2017-07-09 20:45:25
tags: Sqoop
categories: Sqoop
summary_img:
encrypt:
enc_pwd:
---

# 											Sqoop

# 一 简介

Apache Sqoop是在Hadoop生态体系和RDBMS体系之间传送数据的一种工具。

​		Sqoop工作机制是将导入或导出命令翻译成mapreduce程序来实现。在翻译出的mapreduce中主要是对inputformat和outputformat进行定制。

Hadoop生态系统包括：HDFS、Hive、Hbase等

RDBMS体系包括(关系型数据库)：Mysql、Oracle、DB2等

Sqoop也可以理解为：“SQL 到 Hadoop 和 Hadoop 到SQL”。

站在Apache的立场数据可以分为导入和导出:

Import：数据导入。RDBMS----->Hadoop

Export：数据导出。Hadoop---->RDBMS

## 安装

前提: 安装过java和Hadoop

版本 1.4.6

解压及安装

配置sqoop中的conf中的

mv sqoop-env-template.sh sqoop-env.sh

vi sqoop-env.sh

~~~properties
export HADOOP_COMMON_HOME= /export/servers/hadoop-2.7.5 
export HADOOP_MAPRED_HOME= /export/servers/hadoop-2.7.5
export HIVE_HOME= /export/servers/hive
##还可以配置hbase等
~~~

把数据库的驱动加入 sqoop的lib中

测试:

~~~shell
bin/sqoop list-databases --connect jdbc:mysql://node03:3306/ --username root --password 123456
~~~

本命令会列出所有(mysql)(orole)等的数据库。

# 二  sqoop的导入

## 1 从数据库导入hdfs

- mysql的地址尽量不要使用localhost  请使用ip或者host
- 如果不指定  导入到hdfs默认分隔符是  ","
- 可以通过-- fields-terminated-by '\ t‘ 指定具体的分隔符
- 如果表的数据比较大 可以并行启动多个maptask执行导入操作，如果表没有主键，请指定根据哪个字段进行切分

全量导入

**若是换行 每行结尾必须加 \ 否则报错**

~~~shell
bin/sqoop import \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--target-dir /sqoopresult1 \   ##指定存在hdfs的目录
--table emp --m 1   ###指定要导入的表  并指定要运行几个MapTask --m 1
~~~

指定分隔符导入 sqoop的默认分隔符为  ","

--fields-terminated-by '\t' \  ##指定存在hdfs 上的文件的分隔符

~~~shell
bin/sqoop import \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--target-dir /sqoopresout2 \
--fields-terminated-by '\t' \
--table emp --m 1
~~~

指定分割字段和启动几个MapReduce

sqoop命令中，--split-by
id通常配合-m 10参数使用。用于指定根据哪个字段进行划分并启动多少个maptask。

~~~shell
bin/sqoop import \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--target-dir /sqoopresult214 \
--fields-terminated-by '\t' \
--split-by id \                 ##指定分割字段
--table emp --m 2               ##--m 2 启动两个maptask
~~~

##2从数据库导入hive

全量导入

### 1将表结构复制到hive中

hive 中的数据库为test 必须存在  emp_add_sp表 可以不存在

~~~shell
bin/sqoop create-hive-table \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--table emp_add \
--hive-table test.emp_add_sp

~~~

从关系数据库导入文件到hive中

~~~shell
bin/sqoop import \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--table emp_add \
--hive-table test.emp_add_sp \
--hive-import \
--m 1
~~~

### 2直接从数据库导入数据到hive中 包括表结构和数据

不用指定hive的表名,会根据数据库的表名自动创建  若test库不存在会在root下创建表

~~~~shell
bin/sqoop import \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--table emp_conn \
--hive-import \
--m 1 \
--hive-database test
~~~~

##3导入表数据子集(导入hdfs)

### 1where条件过滤导入

--where可以指定从关系数据库导入数据时的查询条件。它执行在数据库服务器相应的SQL查询，并将结果存储在HDFS的目标目录。

~~~
bin/sqoop import \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--where "city='sec-bad'" \
--target-dir /wherequery \
--table emp_add \
--m 1
~~~

###2query查询过滤导入

使用 query sql 语句来进行查找不能加参数--table ;
并且必须要添加 where 条件;
并且 where 条件后面必须带一个$CONDITIONS 这个字符串;
并且这个 sql 语句必须用单引号，不能用双引号;

~~~shell
bin/sqoop import \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--target-dir /query1 \
--query 'select id,name,deg from emp where id>1203 and $CONDITIONS' \
--split-by id \
--fields-terminated-by '\001' \
--m 2
~~~

sqoop命令中–split-by id通常配合-m 10参数使用。
首先sqoop会向关系型数据库比如mysql发送一个命令:select max(id),min(id) from test。
然后会把max、min之间的区间平均分为10分，最后10个并行的map去找数据库，导数据就正式开始。

## 4增量导入

### 1 Append,模式

就是追加导入,在原有的基础上 **根据数值类型字段进行追加导入  大于指定的last-value**

例子

先把数据库数据导入

~~~shell
bin/sqoop import \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--target-dir /appendresult \
--table emp --m 1
~~~

在数据库emp 中在插入两条数据

~~~sql
insert into `userdb`.`emp` (`id`, `name`, `deg`, `salary`, `dept`) values ('1206', 'allen', 'admin', '30000', 'tp');
insert into `userdb`.`emp` (`id`, `name`, `deg`, `salary`, `dept`) values ('1207', 'woon', 'admin', '40000', 'tp');
~~~

执行追加导入

--incremental append \   增量导入的模式
--check-column id \         数值类型字段进行追加导入
--last-value 1205            最后字段值

~~~shell
bin/sqoop import \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--table emp --m 1 \
--target-dir /appendresult \
--incremental append \
--check-column id \
--last-value 1205
~~~

### 3 lastmodified模式

1append模式(附加)

lastmodified 根据时间戳类型字段进行追加  **大于等于**指定的last-value

- 注意在lastmodified 模式下 还分为两种情形：append  merge-key

关于lastmodified 中的两种模式：

- append 只会追加增量数据到一个新的文件中  并且会产生数据的重复问题

  因为默认是从指定的last-value 大于等于其值的数据开始导入

- merge-key 把增量的数据合并到一个文件中  处理追加增量数据之外 如果之前的数据有变化修改

  也可以进行修改操作 底层相当于进行了一次完整的mr作业。数据不会重复。

数据库建表:

~~~sql
create table customertest(id int,name varchar(20),last_mod timestamp default current_timestamp on update current_timestamp);
此处的时间戳设置为在数据的产生和更新时都会发生改变.
~~~

插入数据(一个一个插,可以保证last_mod 字段不一样)

~~~sql
insert into customertest(id,name) values(1,'neil');
insert into customertest(id,name) values(2,'jack');
insert into customertest(id,name) values(3,'martin');
insert into customertest(id,name) values(4,'tony');
insert into customertest(id,name) values(5,'eric');
~~~

执行命令导入hdfs:

~~~shell
bin/sqoop import \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--target-dir /lastmodifiedresult \
--table customertest --m 1
~~~

在mysql中在插入一条数据

~~~sql
insert into customertest(id,name) values(6,'james')
~~~

使用增量导入:

三兄弟:两种导入都要写

--check-column last_mod \
--incremental lastmodified \
--last-value "2019-06-05 15:52:58" \

~~~shell
bin/sqoop import \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--table customertest \
--target-dir /lastmodifiedresult \
--check-column last_mod \
--incremental lastmodified \
--last-value "2019-06-05 15:52:58" \
--m 1 \
--append    #追加方式  merge-key 和append 
~~~

查看结果发现:

此处已经会导入我们最后插入的一条记录,但是我们却发现此处插入了2条数据，这是为什么呢？ 
这是因为采用lastmodified模式去处理增量时，会将大于等于last-value值的数据当做增量插入

注意: 
**使用lastmodified模式进行增量处理要指定增量数据是以append模式(附加)还是merge-key(合并)模式添加**

2merge-key模式(合并)

数据库操作:

~~~sql
update customertest set name = 'Neil' where id = 1;
~~~

增量导入:

~~~shell
bin/sqoop import \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--table customertest \
--target-dir /lastmodifiedresult \
--check-column last_mod \
--incremental lastmodified \
--last-value "2019-06-05 15:52:58" \
--m 1 \
--merge-key id 
~~~

由于merge-key模式是进行了一次完整的mapreduce操作，因此最终我们在lastmodifiedresult文件夹下可以看到生成的为part-r-00000这样的文件，会发现id=1的name已经得到修改，同时新增了id=6的数据。

# 三sqoop导出

将数据从Hadoop生态体系导出到RDBMS数据库导出前，目标表必须存在于目标数据库中。

export有三种模式：

默认操作:   是从将文件中的数据使用INSERT语句插入到表中。若为空表底层为insert一条一条的插入

更新模式：Sqoop将生成UPDATE替换数据库中现有记录的语句。底层为updata

调用模式：Sqoop将为每条记录创建一个存储过程调用。

配置参数:

- 导出文件的分隔符  如果不指定 默认以“,”去切割读取数据文件   --input-fields-terminated-by
- 如果文件的字段顺序和表中顺序不一致 需要--columns 指定 多个字段之间以","
- 导出的时候需要指定导出数据的目的 export-dir 和导出到目标的表名或者存储过程名
- 针对空字符串类型和非字符串类型的转换  “\n”

## 1 默认模式导出HDFS数据到mysql

默认情况下，sqoop export将每行输入记录转换成一条INSERT语句，添加到目标数据库表中。如果数据库中的表具有约束条件（例如，其值必须唯一的主键列）并且已有数据存在，则必须注意避免插入违反这些约束条件的记录。如果INSERT语句失败，导出过程将失败。**此模式主要用于将记录导出到可以接收这些结果的空表中**。通常用于全表数据导出。

导出时可以是将Hive表中的全部记录或者HDFS数据（可以是全部字段也可以部分字段）导出到Mysql目标表。

1准备hdfs数据

 在HDFS文件系统中“/emp/”目录的下创建一个文件emp_data.txt：

~~~properties
1201,gopal,manager,50000,TP
1202,manisha,preader,50000,TP
1203,kalil,php dev,30000,AC
1204,prasanth,php dev,30000,AC
1205,kranthi,admin,20000,TP
1206,satishp,grpdes,20000,GR
~~~

2手动创建数据库中的表

~~~sql
mysql> USE userdb;
mysql> CREATE TABLE employee ( 
   id INT NOT NULL PRIMARY KEY, 
   name VARCHAR(20), 
   deg VARCHAR(20),
   salary INT,
   dept VARCHAR(10));
~~~

3执行导出命令

~~~shell
bin/sqoop export \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--table employee \
--export-dir /emp/
~~~

若是数据库中字段与emp_data.txt文件中字段类型一致,上述做法可以 若不一致

当导出数据文件和目标表字段列顺序完全一致的时候上述做法可以 若不一致。以逗号为间隔选择和排列各个列。加一下配置

--columns id,name,deg,salary,dept \    指定emp_data.txt 中个字段的名字

~~~shell
bin/sqoop export \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--table employee1 \
--columns id,name,deg,salary,dept \
--export-dir /emp/
~~~

--input-fields-terminated-by '\t'  

指定文件中的分隔符

--columns 

选择列并控制它们的排序。当导出数据文件和目标表字段列顺序完全一致的时候可以不写。否则以逗号为间隔选择和排列各个列。没有被包含在–columns后面列名或字段要么具备默认值，要么就允许插入空值。否则数据库会拒绝接受sqoop导出的数据，导致Sqoop作业失败

--export-dir 导出目录，在执行导出的时候，必须指定这个参数，同时需要具备--table或--call参数两者之一，--table是指的导出数据库当中对应的表，

--call是指的某个存储过程。

--input-null-string --input-null-non-string

如果没有指定第一个参数，对于字符串类型的列来说，“NULL”这个字符串就回被翻译成空值，如果没有使用第二个参数，无论是“NULL”字符串还是说空字符串也好，对于非字符串类型的字段来说，这两个类型的空串都会被翻译成空值。比如：

--input-null-string "\\N" --input-null-non-string "\\N"

## 2更新导出（updateonly模式）

-- update-key，更新标识，即根据某个字段进行更新，例如id，可以指定多个更新标识的字段，多个字段之间用逗号分隔。

-- updatemod，指定updateonly（默认模式），**仅仅更新已存在的数据记录，不会插入新纪录。**

在HDFS文件系统中“/updateonly_1/”目录的下创建一个文件updateonly_1.txt：
1201,gopal,manager,50000
1202,manisha,preader,50000
1203,kalil,php dev,30000

手动创建mysql中的目标表

~~~sql
mysql> USE userdb;
mysql> CREATE TABLE updateonly ( 
   id INT NOT NULL PRIMARY KEY, 
   name VARCHAR(20), 
   deg VARCHAR(20),
   salary INT);
~~~



先执行全部导出操作：

~~~shell
bin/sqoop export \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--table updateonly \
--export-dir /updateonly_1/
~~~



新增一个文件updateonly_2.txt：修改了前三条数据并且新增了一条记录
1201,gopal,manager,1212
1202,manisha,preader,1313
1203,kalil,php dev,1414
1204,allen,java,1515


hadoop fs -put updateonly_2.txt /updateonly_2

执行更新导出：

~~~shell
bin/sqoop export \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--table updateonly \
--export-dir /updateonly_2/ \
--update-key id \
--update-mode updateonly
~~~



## 3 更新导出（allowinsert模式）

-- update-key，更新标识，即根据某个字段进行更新，例如id，可以指定多个更新标识的字段，多个字段之间用逗号分隔。

-- updatemod，指定allowinsert，更新已存在的数据记录，同时插入新纪录。**实质上是一个insert & update的操作。**

在HDFS “/allowinsert_1/”目录的下创建一个文件allowinsert_1.txt：
1201,gopal,manager,50000
1202,manisha,preader,50000
1203,kalil,php dev,30000

手动创建mysql中的目标表

~~~
mysql> USE userdb;
mysql> CREATE TABLE allowinsert ( 
   id INT NOT NULL PRIMARY KEY, 
   name VARCHAR(20), 
   deg VARCHAR(20),
   salary INT);
~~~



先执行全部导出操作

~~~shell
bin/sqoop export \
--connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--table allowinsert \
--export-dir /allowinsert_1/
~~~



allowinsert_2.txt。修改了前三条数据并且新增了一条记录。上传至/ allowinsert_2/目录下：
1201,gopal,manager,1212
1202,manisha,preader,1313
1203,kalil,php dev,1414
1204,allen,java,1515



执行更新导出

~~~shell
bin/sqoop export \
--connect jdbc:mysql://node03:3306/userdb \
--username root --password 123456 \
--table allowinsert \
--export-dir /allowinsert_2/ \
--update-key id \
--update-mode allowinsert
~~~





# 四sqoop job 作业

## 1 job 语法

~~~
$ sqoop job (generic-args) (job-args)
   [-- [subtool-name] (subtool-args)]

$ sqoop-job (generic-args) (job-args)
   [-- [subtool-name] (subtool-args)]

~~~

## 2 创建job

在这里，我们创建一个名为jobtest，这可以从RDBMS表的数据导入到HDFS作业。

下面的命令用于创建一个从DB数据库的emp表导入到HDFS文件的作业。

~~~shell
bin/sqoop job --create jobtest -- import --connect jdbc:mysql://node03:3306/userdb \
--username root \
--password 123456 \
--target-dir /sqoopresult333 \
--table emp --m 1

注意import前要有空格

~~~

## 3 验证job

**--list’** 参数是用来验证保存的作业。下面的命令用来验证保存Sqoop作业的列表。

~~~
bin/sqoop job --list
~~~

## 4检查job

**‘--show’** 参数用于检查或验证特定的工作，及其详细信息。以下命令和样本输出用来验证一个名为jobtest的作业。

~~~
bin/sqoop job --show jobtest
~~~

## 5 执行job

**‘--exec’** 选项用于执行保存的作业。下面的命令用于执行保存的作业称为jobtest。

~~~
bin/sqoop job --exec jobtest
~~~

## 6免密执行job

sqoop在创建job时，使用--password-file参数，可以避免输入mysql密码，如果使用--password将出现警告，并且每次都要手动输入密码才能执行job，sqoop规定密码文件必须存放在HDFS上，并且权限必须是400。

并且检查sqoop的sqoop-site.xml是否存在如下配置：

~~~properties
<property>

    <name>sqoop.metastore.client.record.password</name>

    <value>true</value>

    <description>If true, allow saved passwords in the metastore.

    </description>

</property>

~~~

~~~~
bin/sqoop job --create jobtest -- import --connect jdbc:mysql://node03:3306/userdb \
--username root \
--password-file /input/sqoop/pwd/mysqltest.pwd \
--target-dir /sqoopresult333 \
--table emp --m 1

~~~~

# 五 sqoop1与sqoop2的区别

## 1 版本号对比

两代之间是两个完全不同的版本，不兼容 

两代之间是两个完全不同的版本，不兼容 
sqoop1：1.4.x 

sqoop2：1.99.x

## 2 sqoop2的改进

(1) 引入sqoop server，集中化管理connector等 
(2) 多种访问方式：CLI,Web UI，REST API 
(3) 引入基于角色 的安全机制\

## 3 功能性对比

| 功能                         | sqoop1                                   | sqoop2                                   |
| -------------------------- | ---------------------------------------- | ---------------------------------------- |
| 用于所有主要 RDBMS 的连接器          | 支持                                       | 不支持解决办法： 使用已在以下数据库上执行测试的通用 JDBC 连接器： Microsoft SQL Server 、 PostgreSQL 、 MySQL 和 Oracle 。 |
| Kerberos 安全集成              | 支持                                       | 不支持                                      |
| 数据从 RDBMS 传输至 Hive 或 HBase | 支持                                       | 不支持 解决办法： 按照此两步方法操作。 将数据从 RDBMS 导入 HDFS 在 Hive 中使用相应的工具和命令（例如 LOAD DATA 语句），手动将数据载入 Hive 或 HBase |
| 数据从 Hive 或 HBase 传输至 RDBMS | 不支持 解决办法： 按照此两步方法操作。 从 Hive 或 HBase 将数据提取至 HDFS （作为文本或 Avro 文件） 使用 Sqoop 将上一步的输出导出至 RDBMS | 不支持 按照与 Sqoop 1 相同的解决方法操作                |

## 4 架构对比

### 1 sqoop1

在架构上：sqoop1使用sqoop客户端直接提交的方式 
访问方式：CLI控制台方式进行访问 
安全性：命令或脚本中指定用户数据库名及密码

### 2 sqoop2

版本号为1.99x为sqoop2 
在架构上：sqoop2引入了sqoop server，对connector实现了集中的管理 
访问方式：REST API、 JAVA API、 WEB UI以及CLI控制台方式进行访问 

CLI方式访问，会通过交互过程界面，输入的密码信息丌被看到，同时Sqoop2引入基亍角色的安全机制，Sqoop2比Sqoop多了一个Server端。

## 5 两者的优缺点比较

sqoop1:

```
sqoop1优点架构部署简单 
 sqoop1的缺点命令行方式容易出错，格式紧耦合，无法支持所有数据类型，安全机制不够完善，例如密码暴漏， 
安装需要root权限，connector必须符合JDBC模型 
```

sqoop2

```
sqoop2的优点多种交互方式，命令行，web UI，rest API，conncetor集中化管理，所有的链接安装在sqoop server上，完善权限管理机制，connector规范化，仅仅负责数据的读写。 
它引入了基于角色的安全机制，管理员可以在sqoopServer上， 配置不同的角色s
sqoop2的缺点，架构稍复杂，配置部署更繁琐。
```





