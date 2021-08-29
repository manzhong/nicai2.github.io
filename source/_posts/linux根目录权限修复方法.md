---
title: linux根目录权限修复方法
abbrlink: 27094
date: 2017-07-03 08:32:21
tags: Linux
categories: Linux
summary_img:
encrypt:
enc_pwd:
---

今天由于权限问题一般把/这个目录用chmod -R 777执行了一遍，结果各种问题出现，su root就报su:鉴定故障的错误。然后上网找教程很多都要求在root权限下操作来修复，真是悔不当初，哭都哭不出来，只想剁手。幸好最好予以解决了，不然就真得重装系统了，在此把解决方案记录下来，希望能给踩到坑的朋友抢救一下。

## step1

新建一个.c文件，在这里我命名为chmodfix.c，把如下内容写到这个.c文件中

~~~~c

#include <stdio.h>                                                                   #include <stdlib.h>
#include <string.h>
#include <sys/stat.h>
#include <ftw.h>
int list(const char *name, const struct stat *status, int type)
{
    if(type == FTW_NS)
        return 0;
    printf("%s 0%3o\n", name, status->st_mode & 07777);
    return 0;
}
int main(int argc, char *argv[])
{
    if(argc == 1)
        ftw(".", list, 1); 
    else
    ftw(argv[1], list, 2); 
    exit(0);
}
~~~~

然后在终端命令行下使用gcc编译得到可执行文件chmodfix.com

~~~
gcc chmodfix.c -o chmodfix.com
~~~

## step2

新建一个.sh文件，在这里我命名为chmodfix.sh，把如下内容写到这个.sh文件中

~~~shell

#!/bin/sh                                                                            if [ $# != 1 ] 
then
    echo Usage : $0 \<filename\>
    exit
fi
PERMFILE=$1
cat $PERMFILE | while read LINE
do
    FILE=`echo $LINE | awk '{print $1}'`
    PERM=`echo $LINE | awk '{print $2}'`    
    chmod $PERM $FILE
    #echo "chmod $PERM $FILE"
done
echo "change perm finished! "
~~~

## step3

找到另一台装有centos7并且系统权限正常的电脑，利用step1中得到的chmodfix.com从这台电脑上获取被你损坏的目录下所有文件的正常权限

~~~

# 假设原电脑上权限损坏的目录为/usr/bin
./chmodfix.com /usr/bin >> chmodfix.txt
~~~

输出文件chmodfix.txt的内容形式如下

~~~
/usr/bin 0755 /usr/bin/cp 0755
/usr/bin/lua 0755
/usr/bin/captoinfo 0755
/usr/bin/csplit 0755
/usr/bin/clear 0755
/usr/bin/cut 0755
~~~

将得到的权限文件chmodfix.txt复制到权限受损的电脑上

## step4

权限受损电脑进入单用户模式:

CentOS6.x版本
单用户模式，就是你现在站在这台机器面前能干的活，再通俗点就是你能够接触到这个物理设备。

一般干这个活的话，基本上是系统出现严重故障或者其他的root密码忘记等等，单用户模式就非常有用了；

1、在开机启动的时候能看到引导目录，用上下方向键选择你忘记密码的那个系统，然后按“e”；

[![CentOS6/CentOS7进入单用户模式 - 第1张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517153944.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517153944.png)

2、接下来你可以看到如下图所示的画面，然后你再用上下键选择最新的内核，然后在按“e”；

[![CentOS6/CentOS7进入单用户模式 - 第2张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517154016.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517154016.png)

[![CentOS6/CentOS7进入单用户模式 - 第3张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517154040.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517154040.png)

3、执行完上步操作后 在rhgb quiet最后加“空格”，然后键入“single”，或者直接输入数字的“1”并回车确定；

[![CentOS6/CentOS7进入单用户模式 - 第4张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517154102.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517154102.png)

[![CentOS6/CentOS7进入单用户模式 - 第5张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517154155.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517154155.png)

4、按“b”键，重新引导系统；

[![CentOS6/CentOS7进入单用户模式 - 第6张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517154216.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517154216.png)

5、然后就进入了单用户模式下，你就可以使用root功能的东西了，改完你要改的文件后reboot即可。

[![CentOS6/CentOS7进入单用户模式 - 第7张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517154249.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517154249.png)

[![CentOS6/CentOS7进入单用户模式 - 第8张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517154615.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517154615.png)

 

centos7版本采用的是grub2，和centos6.x进入单用户的方法不同。

init方法

1、centos7的grub2界面会有两个入口，正常系统入口和救援模式；

2、修改grub2引导

在正常系统入口上按下”e”，会进入edit模式，搜寻ro那一行，以linux16开头的；

[![CentOS6/CentOS7进入单用户模式 - 第9张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517145321.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517145321.png)

[![CentOS6/CentOS7进入单用户模式 - 第10张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517145440.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517145440.png)

把只读更改成可写
把ro更改成rw

指定shell环境
增加init=/sysroot/bin/sh
或init=/sysroot/bin/bash

[![CentOS6/CentOS7进入单用户模式 - 第11张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517145703.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517145703.png)

[![CentOS6/CentOS7进入单用户模式 - 第12张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517145737.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517145737.png)

按下ctrl+x来启动系统。

[![CentOS6/CentOS7进入单用户模式 - 第13张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517150057.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517150057.png)

3、进入系统以后将/sysroot/设置为根

```
chroot /sysroot/
```

4、做相应的系统维护工作

如：修改密码

```
passwd
```

5、系统启用了selinux，必须运行以下命令，否则将无法正常启动系统：

```
touch /.autorelabel
```

6、退出并重启系统
退出之前设置的根

```
exit
```

重启系统

```
reboot
```

 

另外还有一种rd.break方法

1、启动的时候，在启动界面，相应启动项，内核名称上按“e”；

[![CentOS6/CentOS7进入单用户模式 - 第14张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517150940.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517150940.png)

2、进入后，找到linux16开头的地方，按“end”键到最后，输入rd.break，按ctrl+x进入；

[![CentOS6/CentOS7进入单用户模式 - 第15张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517151010.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517151010.png)

[![CentOS6/CentOS7进入单用户模式 - 第16张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517151045.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517151045.png)

3、进去后输入命令mount，发现根为/sysroot/，并且不能写，只有ro=readonly权限；

[![CentOS6/CentOS7进入单用户模式 - 第17张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517151129.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517151129.png)

[![CentOS6/CentOS7进入单用户模式 - 第18张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517151232.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517151232.png)

4、重新挂载，之后mount，发现有了r,w权限；

```
mount -o remount,rw /sysroot/
```

[![CentOS6/CentOS7进入单用户模式 - 第19张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517151527.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517151527.png)

5、改变根

```
chroot /sysroot/
```

在/tmp/下创建一个aaa的目录

```
mkdir /tmp/aaa
```

6、系统启用了selinux，必须运行以下命令，否则将无法正常启动系统：

```
touch /.autorelabel
```

7、退出之前设置的根

```
exit
```

[![CentOS6/CentOS7进入单用户模式 - 第20张  | 劳福喜博客-专注Linux服务器运维技术](http://laofuxi.com/wp-content/uploads/2016/05/QQ%E5%9B%BE%E7%89%8720160517151625.png)](http://laofuxi.com/wp-content/uploads/2016/05/QQ图片20160517151625.png)

8、重启系统

```
reboot
```



# 以上结束后

然后进入到chmodfix.sh和chmodfix.txt所存放的文件夹下，执行chmodfix.sh以根据chmodfix.txt恢复受损文件的正确权限

~~~
bash chmodfix.sh chmodfix.txt
~~~

