---
title: Linux常用命令
abbrlink: 48230
date: 2017-07-03 22:51:13
tags: Linux
categories: Linux
summary_img:
encrypt:
enc_pwd:
---

# 一 目录结构

![img](/images/linux/linux目录结构.png)

Linux系统它是文件系统。

它的根目录 是”/”,是以树型结构来管理。

root用户登录后，显示一个~ ，表示root目录

root目录 管理员的

其他用户的在home目录中

# 二常用命令

## 1． 切换目录命令cd：

~~~
使用cd app  切换到app目录
cd ..       切换到上一层目录
cd /        切换到系统根目录
cd~         切换到root用户主目录
cd -        切换到上一个所在目录
~~~

## 2． 列出文件列表：ls ll dir(*****)

ls(list)是一个非常有用的命令，用来显示当前目录下的内容。配合参数的使用，能以不同的方式显示目录内容。    格式：ls[参数] [路径或文件名] 

常用：

~~~
在linux中以 . 开头的文件都是隐藏的文件
* ls
* ls -a  显示所有文件或目录（包含隐藏的文件）
* ls -l  缩写成ll

~~~

## 3． 创建目录和移除目录：mkdir rmdir

~~~
mkdir(make directory)命令可用来创建子目录。
mkdir app   在当前目录下创建app目录
mkdir –p app2/test  级联创建aap2以及test目  递归创建

rmdir(remove directory)命令可用来删除“空”的子目录：
rmdir app  删除app目录 
~~~

## 4． 浏览文件

【cat、more、less】

~~~
cat操作：展示所有内容—适合查看小文件
cat a.txt
~~~

~~~
more与less用法类似 ： 适合查看大文件—翻页查看
more a.log
less b.log
空格 ：查看下一页
Enter ：查看下一行
Q    ：退出
Less 可以使用上下箭头查看行。

~~~

~~~shell 
tail命令是在实际使用过程中使用非常多的一个命令，它的功能是：用于显示文件后几行的内容。
用法:
tail -10 /etc/passwd  查看后10行数据
tail -f catalina.log  动态查看日志(*****)

ctrl+c  
~~~

~~~~
Ctrl+c 与ctrl+z 的区别

两者都为中断进程
Ctrl+c  是强制中断程序的执行
ctrl+z  是将任务中断,但任务并未结束,他任然在进程中只是挂起了,用户可以使用fg/bg操作继续前台或后台的任务,fg命令重新启动前台被中断的任务,bg命令把被中断的任务放在后台执行.
~~~~

## 5． 文件操作：

cp是copy操作

mv它是move相当于剪切

~~~
cp(copy)命令可以将文件从一处复制到另一处。一般在使用cp命令时将一个文件复制成另一个文件或复制到某目录时，需要指定源文件名与目标文件名或目录。
cp a.txt b.txt    将a.txt复制为b.txt文件
cp a.txt ../    将a.txt文件复制到上一层目录中


mv 移动或者重命名
mv a.txt ../    将a.txt文件移动到上一层目录中
mv a.txt b.txt    将a.txt文件重命名为b.txt
mv a.txt /b/b.txt  将a.txt移动到根目录下的b目录下的b.txt

~~~

tar 命令

~~~
tar命令位于/bin目录下，它能够将用户所指定的文件或目录打包成一个文件，但不做压缩。一般Linux上常用的压缩方式是选用tar将许多文件打包成一个文件，再以gzip压缩命令压缩成xxx.tar.gz(或称为xxx.tgz)的文件。常用参数：-c：创建一个新tar文件-v：显示运行过程的信息-f：指定文件名-z：调用gzip压缩命令进行压缩-t：查看压缩文件的内容-x：解开tar文件

打包：
tar –cvf xxx.tar ./*
打包并且压缩：
tar –zcvf xxx.tar.gz ./* 

解压 
     tar –xvf xxx.tar
tar -xvf xxx.tar.gz -C /usr/aaa
~~~

find  命令

~~~
find指令用于查找符合条件的文件
示例：
find / -name “ins*” 查找文件名称是以ins开头的文件
find / -name “ins*” –ls 
find / –user itcast –ls 查找用户itcast的文件
find / –user itcast –type d –ls 查找用户itcast的目录
find /-perm -777 –type d-ls 查找权限是777的文件
~~~

grep 命令  常与|命令一起使用

~~~
查找文件里符合条件的字符串。
用法: grep [选项]... PATTERN [FILE]...示例：
grep lang anaconda-ks.cfg  在文件中查找lang
grep lang anaconda-ks.cfg –color 高亮显示
~~~



## 6． 其他常用命令

【pwd】

显示当前所在目录

【touch】

创建一个空文件

\* touch a.txt

【ll -h】

友好显示文件大小

【wget】

下载资料

\* wget http://nginx.org/download/nginx-1.9.12.tar.gz



## 7vi 和vim 编辑器

~~~
在Linux下一般使用vi编辑器来编辑文件。vi既可以查看文件也可以编辑文件。三种模式：命令行、插入、底行模式。
切换到命令行模式：按Esc键；
切换到插入模式：按 i 、o、a键；
    i 在当前位置前插入
    I 在当前行首插入
    a 在当前位置后插入
    A 在当前行尾插入
    o 在当前行之后插入一行
    O 在当前行之前插入一行

切换到底行模式：按 :（冒号）；更多详细用法，查询文档《Vim命令合集.docx》和《vi使用方法详细介绍.docx》

打开文件：vim file
退出：esc  :q
修改文件：输入i进入插入模式
保存并退出：esc:wq

不保存退出：esc:q!

三种进入插入模式：
i:在当前的光标所在处插入
o:在当前光标所在的行的下一行插入
a:在光标所在的下一个字符插入

快捷键：
dd – 快速删除一行
yy - 复制当前行
nyy - 从当前行向后复制几行
p - 粘贴
R – 替换

~~~

## 8 重定向输出>  >>

~~~
>  重定向输出，覆盖原有内容；>> 重定向输出，又追加功能；示例：
cat /etc/passwd > a.txt  将输出定向到a.txt中
cat /etc/passwd >> a.txt  输出并且追加

ifconfig > ifconfig.txt

~~~

## 9 管道

~~~
管道是Linux命令中重要的一个概念，其作用是将一个命令的输出用作另一个命令的输入。示例
ls --help | more  分页查询帮助信息
ps –ef | grep java  查询名称中包含java的进程

ifconfig | more
cat index.html | more
ps –ef | grep aio

~~~

## 10 &&命令执行控制

~~~
命令之间使用 && 连接，实现逻辑与的功能。 
只有在 && 左边的命令返回真（命令返回值 $? == 0），&& 右边的命令才会被执行。 
只要有一个命令返回假（命令返回值 $? == 1），后面的命令就不会被执行。

mkdir test && cd test

~~~

## 11 系统管理命令

~~~
date 显示或设置系统时间
date  显示当前系统时间
date -s “2014-01-01 10:10:10“  设置系统时间df 显示磁盘信息
df –h  友好显示大小free 显示内存状态
free –m 以mb单位显示内存组昂头top 显示，管理执行中的程序

clear 清屏幕
ps 正在运行的某个进程的状态
ps –ef  查看所有进程
ps –ef | grep ssh 查找某一进程
kill 杀掉某一进程
kill 2868  杀掉2868编号的进程
kill -9 2868  强制杀死进程

du 显示目录或文件的大小。
du –h 显示当前目录的大小
who 显示目前登入系统的用户信息。
uname 显示系统信息。
uname -a 显示本机详细信息。依次为：内核名称(类别)，主机名，内核版本号，内核版本，内核编译日期，硬件名，处理器类型，硬件平台类型，操作系统名称

~~~

## 12 用户和组

1用户

~~~
useradd 添加一个用户
useradd test 添加test用户
useradd test -d /home/t1  指定用户home目录
passwd  设置、修改密码
passwd test  为test用户设置密码

切换登录：
ssh -l test -p 22 192.168.19.128

su – 用户名
userdel 删除一个用户
userdel test 删除test用户(不会删除home目录)
userdel –r test  删除用户以及home目录

~~~

2 组

~~~
当在创建一个新用户user时，若没有指定他所属于的组，就建立一个和该用户同名的私有组
创建用户时也可以指定所在组
groupadd  创建组
groupadd public  创建一个名为public的组
useradd u1 –g public  创建用户指定组groupdel 删除组，如果该组有用户成员，必须先删除用户才能删除组。
groupdel public

~~~

3 ID su 命令

~~~
功能：查看一个用户的UID和GID用法：id [选项]... [用户名]
直接使用id
直接使用id 用户名

~~~

~~~~
功能：切换用户。用法：su [选项]... [-] [用户 [参数]... ]示例：
su u1  切换到u1用户
su - u1 切换到u1用户，并且将环境也切换到u1用户的环境（推荐使用）

~~~~

~~~
/etc/passwd  用户文件/etc/shadow  密码文件/etc/group  组信息文件

用户文件

root:x:0:0:root:/root:/bin/bash账号名称：		在系统中是唯一的用户密码：		此字段存放加密口令用户标识码(User ID)：  系统内部用它来标示用户组标识码(Group ID)：   系统内部用它来标识用户属性用户相关信息：		例如用户全名等用户目录：		用户登录系统后所进入的目录用户环境:		用户工作的环境

密码文件

shadow文件中每条记录用冒号间隔的9个字段组成.用户名：用户登录到系统时使用的名字，而且是惟一的口令：  存放加密的口令最后一次修改时间:  标识从某一时刻起到用户最后一次修改时间最大时间间隔:  口令保持有效的最大天数，即多少天后必须修改口令最小时间间隔：	再次修改口令之间的最小天数警告时间：从系统开始警告到口令正式失效的天数不活动时间：	口令过期少天后，该账号被禁用失效时间：指示口令失效的绝对天数(从1970年1月1日开始计算)标志：未使用

组文件

root:x:0:组名：用户所属组组口令：一般不用GID：组ID用户列表：属于该组的所有用户
~~~

## 13 权限命令

~~~
drwxr-xr-x.
d 文件类型   rwx 
r:对文件是指可读取内容 对目录是可以ls
w:对文件是指可修改文件内容，对目录 是指可以在其中创建或删除子节点(目录或文件)
x:对文件是指是否可以运行这个文件，对目录是指是否可以cd进入这个目录

rwx 属主
r-x  属组
r-x 其他用户

~~~

~~~
普通文件： 包括文本文件、数据文件、可执行的二进制程序文件等。 
目录文件： Linux系统把目录看成是一种特殊的文件，利用它构成文件系统的树型结构。  
设备文件： Linux系统把每一个设备都看成是一个文件
普通文件（-）目录（d）符号链接（l）
~~~

~~~~
权限管理
chmod 变更文件或目录的权限。
chmod 755 a.txt 
chmod u=rwx,g=rx,o=rx a.txt
chmod 000 a.txt  / chmod 777 a.txtchown 变更文件或目录改文件所属用户和组
chown u1:public a.txt	：变更当前的目录或文件的所属用户和组
chown -R u1:public dir	：变更目录中的所有的子目录及文件的所属用户和组

~~~~

## 14 网络操作

~~~
主机名
hostname 查看主机名
hostname xxx 修改主机名 重启后无效
如果想要永久生效，可以修改/etc/sysconfig/network文件
~~~

~~~
ip地址

setup设置ip地址
ifconfig 查看(修改)ip地址(重启后无效)
ifconfig eth0 192.168.12.22 修改ip地址
如果想要永久生效
修改 /etc/sysconfig/network-scripts/ifcfg-eth0文件

~~~

~~~
域名映射
/etc/hosts文件用于在通过主机名进行访问时做ip地址解析之用

~~~

~~~
网络服务管理

service network status 查看网络的状态
service network stop 停止网络服务
service network start 启动网络服务
service network restart 重启网络服务

service --status–all 查看系统中所有后台服务
netstat –nltp 查看系统中网络进程的端口监听情况

防火墙设置
防火墙根据配置文件/etc/sysconfig/iptables来控制本机的”出”、”入”网络访问行为。
service iptables status 查看防火墙状态
service iptables stop 关闭防火墙
service iptables start 启动防火墙
chkconfig  iptables off 禁止防火墙自启


~~~

~~~
mysql服务打开、关闭、查看状态

service mysqld start、stop、status

~~~

## 15 其他

yum 安装

- yum install -y telnet

- 测试机器之间能否通信

  - ping 192.122...

- 测试能否与某个应用（比如mysql）通信

  - telnet 192.123..  3306

- 查看进程

  - ps -ef


- 过滤相关信息

  - grep
  - netstat -nltp | grep 3306 查看端口
  - jps | grep NameNode
  - cat | grep -v "#"

- 查看文件

  - cat filename
  - more filename
  - tail -f/-F/-300f filename 查看文件后300行
  - head [-number]filename查看文件头

- 节点传送文件

  - scp -r /export/servers/hadoop node02:/export/servers
  - scp -r /export/servers/hadoop node02:$PWD (发送到当前同级目录)
  - scp -r /export/servers/hadoop user@node02:/export/servers

- 查看日期

  - date
  - date +"%Y-%m-%d %H:%M:%S"
  - date -d "-1 day" +"%Y-%m-%d %H:%M:%S"

- 创建文本

  - while true; do echo 1 >> /root/a.txt ; sleep  1;done



### 2、用户管理

- 添加用户
  - useradd username
- 更改用户密码
  - password username
- 删除用户
  - userdel username 删除用户（不删除用户数据
  - userdel -r username 删除用户和用户数据



### 3、压缩包管理

- gz压缩包
  - tar czf file.tar.gz file 制作file的压缩包
  - tar zxvf file.tar.gz -C /directory 解压缩包
- zip压缩包
  - zip file.zip file 将file制成名为file.zip
  - unzip file.zip 解压缩

### 4、查看属性

- 查看磁盘大小

  - df -h

- 查看内存大小

  - free -h

- 查看文件大小

  - du -h

- 清理缓存

  - echo 1 > /proc/sys/vm/drop_caches

  ~~~~
  Ctrl+c 与ctrl+z 的区别

  两者都为中断进程
  Ctrl+c  是强制中断程序的执行
  ctrl+z  是将任务中断,但任务并未结束,他任然在进程中只是挂起了,用户可以使用fg/bg操作继续前台或后台的任务,fg命令重新启动前台被中断的任务,bg命令把被中断的任务放在后台执行.
  ~~~~

  ~~~
  /etc/hosts   修改主机名与IP映射
  /etc/sysconfig/network  修改主机名
  visudo    设置其他用户权限
  /etc/profile  配置环境变量
  /etc/udev/rules.d/70-persistent-net.rules  更改mac地址
  /etc/sysconfig/network-scripts/ifcfg-eth0  更改IP地址
  ~~~

  ~~~
  三台虚拟机免密登录
  1,三台机器生成公钥与私钥
  ssh -keygen -t rsa
  2.三台机器都把 公钥拷到同一台机器
  ssh-copy-id node01.hadoop.com
  3.复制第一台机器(有三个公钥的那个)的认证到其他两台机器
  scp /root/.ssh/authorized_keys node02.hadoop.com:$PWD
  scp /root/.ssh/authorized_keys node03.hadoop.com:/root/.ssh
  ~~~

  ~~~
  虚拟机的时钟同步
    通过网络连接外网进行时钟同步,必须保证虚拟机连上外网
   ntpdate us.pool.ntp.org
    阿里云时钟同步服务器
    ntpdate ntp4.aliyun.com
  定时任务  定时同步时钟
  crontab  -e   
  */1 * * * * /usr/sbin/ntpdate ntp4.aliyun.com;
  */1 * * * * /usr/sbin/ntpdate us.pool.ntp.org;  二选一
  ~~~

