---
title: mysql的常用函数
date: 2021-02-10 22:52:47
tags: Mysql
categories: Mysql
summary_img:
encrypt:
enc_pwd:
---

### 1 定义变量排序行号

```sql
 --1 
 select
    if(province_name='unknow','未知地区',province_name) as province_name
    ,(@rowNum := @rowNum + 1) as pro_rank
    ,user_cnt
 from rpt_user_device_province_cnt_d ,(select (@rowNum := 0)) b
 where hp_cal_dt='2022-01-10'
 order by user_cnt desc
 --2
 select
    province_name
    ,user_cnt
    ,@rowNum := @rowNum+1
 from
 (select
    if(province_name='unknow','未知地区',province_name) as province_name
    ,user_cnt
    ,@rowNum := 0 as rr
 from rpt_user_device_province_cnt_d  b
 where hp_cal_dt='2022-01-10'
) a order by user_cnt desc
;
--3 
set @rowNum = 0;
select ......
 
```

### 2 字符串分割

```sql
--分割
SELECT SUBSTRING_INDEX(SUBSTRING_INDEX('7654,7698,7782,7788',',',0+2),',',-1) AS num 
,SUBSTRING_INDEX('7654,7698,7782,7788',',',2)
--7698   7654,7698

--获取字符串,分割后的长度
SELECT
LENGTH('7654,7698,7782,7788')-LENGTH(REPLACE('7654,7698,7782,7788',',',''))+1
--4
```

### 3 mysql的自定义函数(类似于udf)

自定义函数是一种与存储过程十分相似的过程式数据库对象。它与存储过程一样，都是由 SQL 语句和过程式语句组成的代码片段，并且可以被应用程序和其他 SQL 语句调用。

自定义函数与存储过程之间存在几点区别：

- 自定义函数不能拥有输出参数，这是因为自定义函数自身就是输出参数；而存储过程可以拥有输出参数。
- 自定义函数中必须包含一条 RETURN 语句，而这条特殊的 SQL 语句不允许包含于存储过程中。
- 可以直接对自定义函数进行调用而不需要使用 CALL 语句，而对存储过程的调用需要使用 CALL 语句。

**语法:**

```sql
CREATE FUNCTION <函数名> ( [ <参数1> <类型1> [ , <参数2> <类型2>] ] … )
  RETURNS <类型>
  <函数主体>
```

若不能执行,函数中开头加DELIMITER // 和 END结尾加 // ,这是告诉mysql解析器让他执行一整段的sql 语句。（一般sql解析器会遇到分行就执行相应的语句）

**删除函数语句&查看构建函数的语句:**

```sql
--删除函数
drop function function_name;
--查看构建函数是的语句和相关信息
show create function function_name;
```

#### 3.1 函数控制语句

**if语句**

```sql
IF search_condition THEN statement_list
    [ELSEIF search_condition THEN statement_list] ...
    [ELSE statement_list]
END IF
```

IF实现了一个基本的条件构造。如果search_condition求值为真，相应的SQL语句列表被执行。如果没有search_condition匹配，在ELSE子句里的语句列表被执行。statement_list可以包括一个或多个语句。

**case语句**

```sql
CASE case_value
    WHEN when_value THEN statement_list
    [WHEN when_value THEN statement_list] ...
    [ELSE statement_list]
END CASE
--或
CASE
    WHEN search_condition THEN statement_list
    [WHEN search_condition THEN statement_list] ...
    [ELSE statement_list]
END CASE
```

存储程序的CASE语句实现一个复杂的条件构造。如果search_condition 求值为真，相应的SQL被执行。如果没有搜索条件匹配，在ELSE子句里的语句被执行。

例子: 根据传入一个表示函数名称的字符串与一个待处理的数字，对出入的数字执行不同的操作

```sql
DELIMITER //
create function caseTest(str varchar(5),num int)
returns int 
begin 
case str
when 'power' then set @result=power(num,2);
when 'ceil' then set @result=ceil(num);
when 'floor' then set @result=floor(num);
when 'round' then set @result=round(num);
else set @result=0;
end case;
return (select @result);
end//
DELIMITER;
```

结果:

```sql
select
`caseTest`('power',2)
-- 4
```

**LOOP语句**

```sql
[begin_label:] LOOP
    statement_list
END LOOP [end_label]
```

LOOP允许某特定语句或语句群的重复执行，实现一个简单的循环构造。在循环内的语句一直重复直到循环被退出，退出通常伴随着一个LEAVE 语句。

LOOP语句可以被标注。除非begin_label存在，否则end_label不能被给出，并且如果两者都出现，它们必须是同样的。

- LEAVE语句

```sql
LEAVE label1
```

这个语句被用来退出任何被标注的流程控制构造。它和BEGIN … END或循环一起被使用。

- ITERATE语句

```sql
ITERATE label
```

ITERATE只可以出现在LOOP, REPEAT, 和WHILE语句内。ITERATE意思为:再次循环.

例如:传入值,返回值

```sql
DELIMITER //
CREATE FUNCTION yhf_add(p1 INT)
RETURNS int
BEGIN
  label1: LOOP
    SET p1 = p1 + 1;
    IF p1 < 10 THEN ITERATE label1; 
    END IF;
    LEAVE label1;
  END LOOP label1;
  RETURN(p1);
END//
DELIMITER;
```

调用;

```sql
select
doiterate(1)
--10
```

**REPEAT语句**

```sql
[begin_label:] REPEAT
    statement_list
UNTIL search_condition
END REPEAT [end_label]
```

REPEAT语句内的语句或语句群被重复，直至search_condition 为真。

REPEAT 语句可以被标注。 除非begin_label也存在，end_label才能被用，如果两者都存在，它们必须是一样的。

使用repeat来实现上面的程序，程序如下：

```sql
DELIMITER //
create function doRepeat(p1 int)
returns int
begin
 repeat set p1 = p1 + 1;
 until p1>10 
 end repeat;
return p1;
end//
DELIMITER;
```

```sql
select
doRepeat(6)
--11
```

**while语句**

```sql
[begin_label:] WHILE search_condition DO
    statement_list
END WHILE [end_label]
```

WHILE语句内的语句或语句群被重复，直至search_condition 为真。

WHILE语句可以被标注。 除非begin_label也存在，end_label才能被用，如果两者都存在，它们必须是一样的。

DECLARE仅被用在BEGIN … END复合语句里，并且必须在复合语句的开头，在任何其它语句之前。用于声明一个局部变量，该变量在函数外不可访问，如果想要访问必须将数值返回，此时应该用create function而不可以用create procedure，因为只有create function才可以有返回值

```sql
DELIMITER //
CREATE function dowhile()
RETURNS int
BEGIN
  DECLARE v1 INT DEFAULT 5;
  WHILE v1 > 2 DO
    SET v1 = v1 - 1;
  END WHILE;
RETURN v1;
END//
DELIMITER;
```

```sql
select
dowhile()
--2
```



#### 3.2字符串分割并取出对应值的

```sql
DELIMITER //
CREATE  FUNCTION `yhf_split`(`num` int,`sep` varchar(200),`str` varchar(5000))
/*
	函数功能: 字符串根据指定分割符分割,并取出索引为num的值
	参数: num:索引值,以0开始
		 sep:分割符
		 str:要分割的字符串
*/
/*定义返回值类型 */
RETURNS varchar(100)
BEGIN
	RETURN(SUBSTRING(
         SUBSTRING_INDEX(str, sep, num + 1),
         CASE num
     WHEN 0 THEN
         CHAR_LENGTH(
             SUBSTRING_INDEX(str, sep, num)
         ) + 1
     ELSE
         CHAR_LENGTH(
             SUBSTRING_INDEX(str, sep, num)
         ) + 2
     END,
     CASE num
 WHEN 0 THEN
     CHAR_LENGTH(
         SUBSTRING_INDEX(str, sep, num + 1)
     ) - CHAR_LENGTH(
         SUBSTRING_INDEX(str, sep, num)
     )
 ELSE
     CHAR_LENGTH(
         SUBSTRING_INDEX(str, sep, num + 1)
     ) - CHAR_LENGTH(
         SUBSTRING_INDEX(str, sep, num)
     ) - 1
 END
     ) );
END //
DELIMITER ;

----使用1
select
yhf_split(1,',',"aa,ss,cc,rr")
--ss

----使用2
select
yhf_split(1,':',"aa:ss,cc,rr")
--ss,cc,rr
```



### 4 创建存储过程

存储过程是一组为了完成特定功能的 SQL 语句集合

使用存储过程的目的是将常用或复杂的工作预先用 SQL 语句写好并用一个指定名称存储起来，这个过程经编译和优化后存储在数据库服务器中，因此称为存储过程

创建:

```sql
CREATE PROCEDURE <过程名> ( [过程参数[,…] ] ) <过程体>
[过程参数[,…] ] 格式
[ IN | OUT | INOUT ] <参数名> <类型>
```

删除:

```sql
drop PROCEDURE name;
```

**其创建方式使用的是create procedure而不是create function，这是因为procedure不需要returns 与return字段，而function必须有returns与return字段：**

根据传入的int值,设置自定义变量

```sql
DELIMITER //
CREATE PROCEDURE doiterate(p1 INT)
BEGIN
  label1: LOOP
    SET p1 = p1 + 1;
    IF p1 < 10 THEN ITERATE label1; 
    END IF;
    LEAVE label1;
  END LOOP label1;
  SET @x = p1;
END//
DELIMITER;
```

调用:

```sql
--create procedure 是创建存储过程 实际就是打包了sql语句 由call调用 里面可以写入常用的查询等等
call
doiterate(1)

--调用变量 
select
@x
--10
```

### 4 mysql 自定义用户变量

- 可以先在用户变量中保存值然后在以后引用它；这样可以将值从一个语句传递到另一个语句。用户变量与连接有关。也就是说，一个客户端定义的变量不能被其它客户端看到或使用。当客户端退出时，该客户端连接的所有变量将自动释放。
- 用户变量的形式为@var_name，其中变量名var_name可以由当前字符集的文字数字字符、‘.’、‘_’和‘$’组成。 默认字符集是cp1252 (Latin1)。可以用mysqld的–default-character-set选项更改字符集。用户变量名对大小写不敏感。
  设置用户变量的一个途径是执行SET语句：

```sql
set @var_name = expr 
```

对于SET，可以使用=或:=作为分配符。分配给每个变量的expr可以为整数、实数、字符串或者NULL值。

也可以用语句select代替SET来为用户变量分配一个值。在这种情况下，分配符必须为:=而不能用=，因为在非SET语句中=被视为一个比较 操作符：

```sql
set @a1 =1 ,@a2 = 2,@a3 = 3;
select @a1 := (@a2:=1)+@a3:=4,@a1,@a2,@a3
 -- 5 5 1 4
```

### 5 mysql的行列转换

**行转列**

```sql
create table t_score(
id int primary key auto_increment,
name varchar(20) not null,  
Subject varchar(10) not null,
Fraction double default 0  
)ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='';

INSERT INTO `t_score`(name,Subject,Fraction) VALUES
         ('王海', '语文', 86),
        ('王海', '数学', 83),
        ('王海', '英语', 93),
        ('陶俊', '语文', 88),
        ('陶俊', '数学', 84),
        ('陶俊', '英语', 94),
        ('刘可', '语文', 80),
        ('刘可', '数学', 86),
        ('刘可', '英语', 88),
        ('李春', '语文', 89),
        ('李春', '数学', 80),
        ('李春', '英语', 87);
```

```sql
--1
select 
        ifnull(name,'TOll') name,
        sum(if(Subject='语文',Fraction,0)) as 语文,
       sum(if(Subject='英语',Fraction,0)) as 英语,
       sum(if(Subject='数学',Fraction,0))as 数学,
       sum(Fraction) 总分
        from t_score group by name with rollup
 --2 case when 可以换if
 select  name as Name,
sum(case when Subject = '语文' then Fraction end) as Chinese,
sum(case when Subject = '数学' then Fraction end) as Math,
sum(case when Subject = '英语' then Fraction end) as English,
sum(fraction)as score
from t_score group by name
UNION ALL
select  name as Name,sum(Chinese) as Chinese,sum(Math) as Math,sum(English) as English,sum(score) as score from(
select 'TOTAL' as name,
sum(case when Subject = '语文' then Fraction end) as Chinese,
sum(case when Subject = '数学' then Fraction end) as Math,
sum(case when Subject = '英语' then Fraction end) as English,
sum(fraction)as score
from t_score group by Subject) t
```



