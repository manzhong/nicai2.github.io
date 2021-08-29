---
title: scala入门
abbrlink: 33510
date: 2017-08-10 09:07:51
tags: Scala
categories: Scala
summary_img:
encrypt:
enc_pwd:
---

# Scala入门

## 一 什么是Scala

​           scala是运行在 JVM 上的多范式编程语言，同时支持面向对象和面向函数编程 (如java是面向对象的也是面向接口的,懂得自然懂)

早期，scala刚出现的时候，并没有怎么引起重视，随着Spark和Kafka这样基于scala的大数据框架
的兴起，scala逐步进入大数据开发者的眼帘。scala的主要优势是它的表达性。

### 1 使用场景:

开发大数据应用程序（Spark程序、Flink程序）
表达能力强，一行代码抵得上Java多行，开发速度快
兼容Java，可以访问庞大的Java类库，例如：操作mysql、redis、freemarker、activemq等等

### 2 scala与java的简单对比

scala定义三个实体类:

~~~scala
case class User(var name:String,var orders:List[Order])   //用户实体类
case class Order(var id:Int,var products:List[Product])   //订单实体类
case class Product(var id:Int,var category:String)          //商品实体类
~~~

讲一个列表中的字符串(数字类型) 转为数字列表:

~~~scala
val ints = list.map(s => s.toInt)
~~~

## 二 开发环境搭建

### 1 java与scala的编译流程

java

java源代码> (javac编译)>java字节码和java类库>(加载)>jvm>(解释执行)>操作系统

scala:

scala源代码>(scalac编译)>java字节码和java类库和scala类库>(加载)>jvm>(解释执行)>操作系统

scala程序运行需要依赖于Java类库，必须要有Java运行环境，scala才能正确执行.scala源文件也是编译为class文件

根据上述流程图，要编译运行scala程序，需要
jdk（jvm）
scala编译器（scala SDK）

### 2jdk安装

略

### 3安装SDK

下载安装即可.

idea安装scala插件

## 三scala的解释器

后续我们会使用scala解释器来学习scala基本语法，scala解释器像Linux命令一样，执行一条代
码，马上就可以让我们看到执行结果，用来测试比较方便。

启动:

win+r 后输入scala

退出:    

~~~
:quit
~~~

## 四 语法

### 1 定义变量

格式:

~~~
var/val 变量标识:变量类型 = 初始值 
~~~

val 定义的是不可重新赋值的变量
var 定义的是可重新赋值的变量

注意:  句尾不用写分号.

~~~
重心赋值:
name = "ni"   若为val 则会报错  var 不会报错
~~~

**使用类型推断来定义变量**

~~~
var name = "nicai"
~~~

scala可以自动根据变量的值来自动推断变量的类型，这样编写代码更加简洁。

**惰性赋值**

在企业的大数据开发中，有时候会编写非常复杂的SQL语句，这些SQL语句可能有几百行甚至上千
行。这些SQL语句，如果直接加载到JVM中，会有很大的内存开销。如何解决？

当有一些变量保存的数据较大时，但是不需要马上加载到JVM内存。可以使用惰性赋值来提高效
率。
语法格式：

~~~
lazy var/val 变量标书:变量类型 = 值
或者
lazy var/val 变量标识 = 值
~~~

示例
在程序中需要执行一条以下复杂的SQL语句，我们希望只有用到这个SQL语句才加载它。

~~~~scala
"""insert overwrite table adm.itcast_adm_personas
select
a.user_id,
a.user_name,
a.user_sex,
a.user_birthday,
a.user_age,
a.constellation,
a.province,
a.city,
a.city_level,
a.hex_mail,
a.op_mail,
a.hex_phone,
a.fore_phone,
a.figure_model,
a.stature_model,
b.first_order_time,
b.last_order_time,
...
d.month1_hour025_cnt,
d.month1_hour627_cnt,
d.month1_hour829_cnt,
d.month1_hour10212_cnt,
d.month1_hour13214_cnt,
d.month1_hour15217_cnt,
d.month1_hour18219_cnt,
d.month1_hour20221_cnt,
d.month1_hour22223_cnt
from gdm.itcast_gdm_user_basic a
left join gdm.itcast_gdm_user_consume_order b on a.user_id=b.user_id
left join gdm.itcast_gdm_user_buy_category c on a.user_id=c.user_id
left join gdm.itcast_gdm_user_visit d on a.user_id=d.user_id;"""
参考代码
scala> lazy val sql = """insert overwrite table adm.itcast_adm_personas
略

~~~~

### 2 字符串

多种定义字符串;

使用双引号
使用插值表达式
使用三引号

1使用双引号

~~~
var/val name= "值"     name.length  长度
~~~

2 使用插值表达式

scala中，可以使用插值表达式来定义字符串，有效避免大量字符串的拼接。

~~~
var name="n"
var age=123
var sex="man"
插值拼接:
var info = s"name=${name},age=${age},sex=${sex}"
~~~

3 使用三引号

如果有大段的文本需要保存，就可以使用三引号来定义字符串。例如：保存一大段的SQL语句。三
个引号中间的所有字符串都将作为字符串的值。(包括空行,空格等)

~~~
val/var 变量名 = """字符串1
字符串2"""
~~~

~~~
val sql = """select
| *
| from
| t_user
| where
| name = "zhangsan""""

println(sql)
~~~

## 五 数据类型与操作符

scala中的类型以及操作符绝大多数和Java一样，
数据类型:

~~~
基础类型                                   类型说明
Byte                                       8位带符号整数
Short                                      16位带符号整数
Int                                          32位带符号整数
Long                                      64位带符号整数
Char                                      16位无符号Unicode字符
String                                     Char类型的序列（字符串）
Float                                      32位单精度浮点数
Double                                   64位双精度浮点数
Boolean                                 true或false
~~~

与java的区别:

1scala中所有的类型都使用大写字母开头

2 整形使用 Int 而不是Integer

3 scala中定义变量可以不写类型，让scala编译器自动推断

**运算符**

~~~
类别                                      操作符

算术运算符                     +、-、*、/

关系运算符                     >、<、==、!=、>=、<=

逻辑运算符                     &&、||、!

位运算符                       &、||、^、<<、>>
~~~

~~~
与java的异同:

scala中没有，++、--运算符
== equals 都表示比较值
eq 表示比较地址值是否相等
~~~

例子:

~~~
var a="aa"
var b=a + ""

a == b   true
a.equals(b)   true
a.eq(b)        false
~~~

**scala的类型层次结构**

![img](/images/scala/lxc.jpg)

~~~properties
类型                                       说明

Any             所有类型的父类，,它有两个子类AnyRef与AnyVal

AnyVal        所有数值类型的父类

AnyRef        所有对象类型（引用类型）的父类

Unit             表示空，Unit是AnyVal的子类，它只有一个的实例()

它类似于Java中的void，但scala要比Java更加面向对象

Null             Null是AnyRef的子类，也就是说它是所有引用类型的子类。它的实例是null

可以将null赋值给任何对象类型

Nothing

所有类型的子类
不能直接创建该类型实例，某个方法抛出异常时，返回的就是Nothing类型，因为
Nothing是所有类的子类，那么它可以赋值为任何类型
~~~

noting 例子:

~~~scala
def main(args: Array[String]): Unit = {
    val c = m3(1,0)
}
def m3(x:Int, y:Int):Int = {
  if(y == 0) throw new Exception("这是一个异常")
  x / y
}
~~~

~~~scala
val b:Int = null   会报错   null不属于Int的子类
~~~

## 六 条件,循环表达式

### 1 if语句

语法与java一样

不一样的是:

在scala中，**条件表达式也是有返回值的**
在scala中，**没有三元表达式，**可以使用if表达式替代三元表达式

~~~scala
val sex = "male"
val result = if(sex == "male") 1 else 0
~~~

### 2 块表达式

scala中，使用{}表示一个块表达式
和if表达式一样，块表达式也是有值的
**值就是最后一个表达式的值**

~~~
var a={
println("55")
1+1
} 
输出结果微博2
~~~

### 3 循环

语法:

~~~
for(i <- 表达式/数组/集合) {
// 表达式
}
~~~

循环打印1到10

~~~
var nums=1.to(10)
for(i <- nums) {
println(i)
}
或者:
for(i <- 1 to 10)println(i)

~~~

嵌套循环:

~~~
for (i <- 1 to 10;j <- 1 to 3){
print("*");
if(j==3){
println()
}
}
~~~

守卫:

for表达式中，可以添加if判断语句，这个if判断就称之为守卫。我们可以使用守卫让for表达式更简
洁。

语法:

~~~
for(i <- 表达式/数组/集合 if 表达式) {
// 表达式
}
~~~

~~~
// 添加守卫，打印能够整除3的数字
for(i <- 1 to 10; if i % 3 == 0) println(i)
~~~

**for推导式**:

**将来可以使用for推导式生成一个新的集合（一组数据）**
在for循环体中，可以使用yield表达式构建出一个集合，我们把使用yield的for表达式称之为推
导式

生成一个10,20.....100的集合

~~~
// for推导式：for表达式中以yield开始，该for表达式会构建出一个集合
val v = for(i <- 1 to 10) yield i * 10
print(v)
~~~

**while循环**

语法与java中一致

打印1到10 

~~~scala 
scala> var i = 1
i: Int = 1
scala> while(i <= 10) {
| println(i)
| i = i+1
| }
~~~

## 七 方法

一个类可以有自己的方法，scala中的方法和Java方法类似。但scala与Java定义方法的语法是不一
样的，而且scala支持多种调用方式。

### 1 定义方法

~~~
def  add(参数名:参数类型,参数名:参数类型): [返回值类型 return type] ={
方法体
}
~~~

参数列表的参数类型不能省略
返回值类型可以省略
返回值可以不写return，默认就是{}块表达式的值

定方法:实现两数相加:

~~~
def add(a:Int,b:Int):Int ={
a+b
}

调用 add(1,2)
~~~

### 2 方法参数:

scala中的方法参数，使用比较灵活。它支持以下几种类型的参数：
默认参数
带名参数
变长参数

**默认参数**

在定义方法时可以给参数定义一个默认值。

~~~
def add(a:Int =1,b:Int = 2): Int = a+b
~~~

**带名参数**

在调用方法时，可以指定参数的名称来进行调用。

~~~
def add(a:Int =1,b:Int = 2): Int = a+b

调用时:
add(a=2)
~~~

**变长参数**

如果方法的参数是不固定的，可以定义一个方法的参数是变长参数。

~~~
def add (num:Int*)=num.sum
add(1,2,3)
~~~

在参数类型后面加一个 * 号，表示参数可以是0个或者多个

### 3 方法返回值类型推断

scala定义方法可以省略返回值，由scala自动推断返回值类型。这样方法定义后更加简洁。

~~~
def add(x:Int, y:Int) = x + y
~~~

**定义递归返回，不能省略返回值**

~~~

~~~

### 4 方法调用方式

在scala中，有以下几种方法调用方式，
	后缀调用法
	中缀调用法
	花括号调用法
	无括号调用法
在spark、flink程序时，使用到这些方法。
后缀调用:

~~~
对象名.方法名(参数)
Math.abs(-1)
~~~

中缀调用

~~~
对象名 方法名 参数   若有多个参数用括号
Math abs -1
~~~

**操作符就是方法**

~~~
1 + 1   与中缀
~~~

在scala中，+ - * / %等这些操作符和Java一样，但在scala中，
所有的操作符都是方法
操作符是一个方法名字是符号的方法

**花括号调用发**

语法

~~~
Math.abs{
// 表达式1
// 表达式2 最后一个表达式 的是传入abs 的参数
}
~~~

~~~~
Math.abs{
print("nn")
-10}
~~~~

**无括号调用发**

	如果方法没有参数，可以省略方法名后面的括号
示例
定义一个无参数的方法，打印"hello"
使用无括号调用法调用该方法
参考代码

~~~
def m3()=println("hello")
~~~

## 八 函数

scala支持函数式编程，将来编写Spark/Flink程序中，会大量经常使用到函数

语法:

~~~
val 函数变量名 = (参数名:参数类型, 参数名:参数类型....) => 函数体

注意:
函数是一个对象（变量）
类似于方法，函数也有输入参数和返回值
函数定义不需要使用 def 定义
无需指定返回值类型
~~~

~~~~
scala> val add = (x:Int, y:Int) => x + y
add: (Int, Int) => Int = <function2>
scala> add(1,2)
res3: Int = 3
~~~~

~~~~
方法和函数的区别

方法是隶属于类或者对象的，在运行时，它是加载到JVM的方法区中
可以将函数对象赋值给一个变量，在运行时，它是加载到JVM的堆内存中
函数是一个对象，继承自FunctionN，函数对象有apply，curried，toString，tupled这些方
法。方法则没有
~~~~

方法转换为函数
有时候需要将方法转换为函数，作为变量传递，就需要将方法转换为函数
使用 _ 即可将方法转换为函数

~~~
scala> def add(x:Int,y:Int)=x+y
add: (x: Int, y: Int)Int
scala> val a = add _
a: (Int, Int) => Int = <function2>
43
~~~

# 九数组

scala中数组的概念是和Java类似，可以用数组来存放一组数据。scala中，有两种数组，一种是定
长数组，另一种是变长数组

定长数组
定长数组指的是数组的长度是不允许改变的

~~~
// 通过指定长度定义数组
val/var 变量名 = new Array[元素类型](数组长度)
// 用元素直接初始化数组
val/var 变量名 = Array(元素1, 元素2, 元素3...)

在scala中，数组的泛型使用 [] 来指定
使用 () 来获取元素
~~~

~~~
scala> val a = new Array[Int](100)
a: Array[Int] = Array(0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0
, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,
0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)
scala> a(0) = 110
scala> println(a(0))
110
~~~

~~~
// 定义包含jave、scala、python三个元素的数组
scala> val a = Array("java", "scala", "python")
a: Array[String] = Array(java, scala, python)
scala> a.length
res17: Int = 3
~~~

变长数组
变长数组指的是数组的长度是可变的，可以往数组中添加、删除元素

定义变长数组
语法
创建空的ArrayBuffer变长数组，语法结构：

~~~
val/var a = ArrayBuffer[元素类型]()
~~~

创建带有初始元素的ArrayBuffer

~~~
val/var a = ArrayBuffer(元素1，元素2，元素3....)
~~~

注意:

~~~
创建变长数组，需要提前导入ArrayBuffer类 import scala.collection.mutable.ArrayBuffer
~~~

~~~~
val a = ArrayBuffer[Int]()   长度为0
~~~~

~~~
scala> val a = ArrayBuffer("hadoop", "storm", "spark")
a: scala.collection.mutable.ArrayBuffer[String] = ArrayBuffer(hadoop, storm, spark)
~~~

添加/修改/删除元素
使用 += 添加元素
使用 -= 删除元素
使用 ++= 追加一个数组到变长数组
示例
1. 定义一个变长数组，包含以下元素: "hadoop", "spark", "flink"
2. 往该变长数组添加一个"flume"元素
3. 从该变长数组删除"hadoop"元素
4. 再将一个数组，该数组包含"hive", "sqoop"追加到变长数组中

~~~
// 定义变长数组
scala> val a = ArrayBuffer("hadoop", "spark", "flink")
a: scala.collection.mutable.ArrayBuffer[String] = ArrayBuffer(hadoop, spark, flink)
// 追加一个元素
scala> a += "flume"
res10: a.type = ArrayBuffer(hadoop, spark, flink, flume)
// 删除一个元素
scala> a -= "hadoop"
res11: a.type = ArrayBuffer(spark, flink, flume)
// 追加一个数组
scala> a ++= Array("hive", "sqoop")
res12: a.type = ArrayBuffer(spark, flink, flume, hive, sqoop)
~~~

遍历数组
可以使用以下两种方式来遍历数组：
使用 for表达式 直接遍历数组中的元素
使用 索引 遍历数组中的元素

~~~
scala> val a = Array(1,2,3,4,5)
a: Array[Int] = Array(1, 2, 3, 4, 5)
scala> for(i<-a) println(i)
1
2
3
4
5
~~~

~~~~
scala> val a = Array(1,2,3,4,5)
a: Array[Int] = Array(1, 2, 3, 4, 5)
scala> for(i <- 0 to a.length - 1) println(a(i))
1
2
3
4
5
scala> for(i <- 0 until a.length) println(a(i))
1
2
3
4
5
~~~~

注意:

~~~
0 until n——生成一系列的数字，包含0，不包含n
0 to n ——包含0，也包含n
~~~

### 数组常用算法:

scala中的数组封装了丰富的计算操作，将来在对数据处理的时候，不需要我们自己再重新实现。
以下为常用的几个算法：
求和——sum方法
求最大值——max方法
求最小值——min方法
排序——sorted方法

~~~~
scala> val a = Array(1,2,3,4)
a: Array[Int] = Array(1, 2, 3, 4)
求和   a.sum
最大值  a.max
最小值  a.min
排序   a.sorted
降序  a.sorted.reverse

~~~~

## 十  元组

元组
元组可以用来包含一组不同类型的值。例如：姓名，年龄，性别，出生年月。元组的元素是不可变
的。

语法:

~~~
使用括号来定义元组
val/var 元组 = (元素1, 元素2, 元素3....)
使用尽头来定义元素（元组只有两个元素）
val/var 元组 = 元素1->元素2
~~~

~~~~
例子
// 可以直接使用括号来定义一个元组
scala> val a = (1, "张三", 20, "北京市")
a: (Int, String, Int, String) = (1,张三,20,北京市)
例子二
scala> val a = 1->2
a: (Int, Int) = (1,2)
~~~~

访问元组
使用_1,2,3...来访问元祖中的元素，1表示访问第一个元素，一次类推


~~~~
// 可以直接使用括号来定义一个元组
scala> val a = (1, "张三", 20, "北京市")
a: (Int, String, Int, String) = (1,张三,20,北京市)
// 获取第一个元素
scala> a._1
res57: Int = 1
// 获取第二个元素
scala> a._2
res58: String = 张三
// 不能修改元组中的值
scala> a._1 = 2
<console>:13: error: reassignment to val
a._1 = 2
^
54
~~~~

## 十一列表

列表
List是scala中最重要的、也是最常用的数据结构。List具备以下性质：
可以保存重复的值
有先后顺序
在scala中，也有两种列表，一种是不可变列表、另一种是可变列表

### 1 不可变列表

~~~
不可变列表就是列表的元素、长度都是不可变的。
语法
使用 List(元素1, 元素2, 元素3, ...) 来创建一个不可变列表，语法格式：
val/var 变量名 = List(元素1, 元素2, 元素3...)
使用 Nil 创建一个不可变的空列表
val/var 变量名 = Nil
使用::拼接方式来创建列表，必须在最后添加一个Nil
使用 :: 方法创建一个不可变列表
val/var 变量名 = 元素1 :: 元素2 :: Nil

~~~

例子:

~~~~
示例一
创建一个不可变列表，存放以下几个元素（1,2,3,4）
参考代码
scala> val a = List(1,2,3,4)
a: List[Int] = List(1, 2, 3, 4)
示例二
使用Nil创建一个不可变的空列表
参考代码
scala> val a = Nil
a: scala.collection.immutable.Nil.type = List()
示例三
使用 :: 方法创建列表，包含-2、-1两个元素
参考代码
scala> val a = -2 :: -1 :: Nil
a: List[Int] = List(-2, -1)
~~~~

### 2 可变列表

可变列表就是列表的元素、长度都是可变的。
要使用可变列表，先要导入 import scala.collection.mutable.ListBuffer

注意:

可变集合都在 mutable 包中
不可变集合都在 immutable 包中（默认导入）

~~~~
初始化列表
语法
使用ListBuffer[元素类型]()创建空的可变列表，语法结构：
val/var 变量名 = ListBuffer[Int]()
使用ListBuffer(元素1, 元素2, 元素3...)创建可变列表，语法结构：
val/var 变量名 = ListBuffer(元素1，元素2，元素3...)
~~~~

~~~
例子:
scala> val a = ListBuffer[Int]()
a: scala.collection.mutable.ListBuffer[Int] = ListBuffer()
例子2:
scala> val a = ListBuffer(1,2,3,4)
a: scala.collection.mutable.ListBuffer[Int] = ListBuffer(1, 2, 3, 4)
~~~

**列表操作**
获取元素（使用括号访问 (索引值) ）
添加元素（ += ）
追加一个列表（ ++= ）
更改元素（ 使用括号获取元素，然后进行赋值 ）
删除元素（ -= ）
转换为List（ toList ）
转换为Array（ toArray ）

~~~
// 导入不可变列表
scala> import scala.collection.mutable.ListBuffer
import scala.collection.mutable.ListBuffer
// 创建不可变列表
scala> val a = ListBuffer(1,2,3)
a: scala.collection.mutable.ListBuffer[Int] = ListBuffer(1, 2, 3)
// 获取第一个元素
scala> a(0)
res19: Int = 1
// 追加一个元素
scala> a += 4
res20: a.type = ListBuffer(1, 2, 3, 4)
// 追加一个列表
scala> a ++= List(5,6,7)
res21: a.type = ListBuffer(1, 2, 3, 4, 5, 6, 7)
// 删除元素
scala> a -= 7
res22: a.type = ListBuffer(1, 2, 3, 4, 5, 6)
// 转换为不可变列表
scala> a.toList
res23: List[Int] = List(1, 2, 3, 4, 5, 6)
// 转换为数组
scala> a.toArray
res24: Array[Int] = Array(1, 2, 3, 4, 5, 6
~~~

**列表操作**

以下是列表常用的操作
判断列表是否为空（ isEmpty ）
拼接两个列表（ ++ ）
获取列表的首个元素（ head ）和剩余部分( tail )
反转列表（ reverse ）
获取前缀（ take ）、获取后缀（ drop ）
扁平化（ flaten ）

​         扁平化表示将列表中的列表中的所有元素放到一个列表中。

拉链（ zip ）和拉开（ unzip ）
转换字符串（ toString ）
生成字符串（ mkString ）
并集（ union ）
交集（ intersect ）
差集（ diff ）

例子:

~~~scala 
a.isEmpty
 a ++ b
 a.head            //获取列表的首个元素（ head ）和剩余部分( tail )
 a.tail
 a.reverse
 a.take(3)    //获取前缀（ take ）、获取后缀（ drop ）
 a.drop(3)
~~~

~~~~scala 
扁平化表示将列表中的列表中的所有元素放到一个列表中。
scala> val a = List(List(1,2), List(3), List(4,5))
a: List[List[Int]] = List(List(1, 2), List(3), List(4, 5))

scala> a.flatten
res0: List[Int] = List(1, 2, 3, 4, 5)
~~~~

~~~~scala 
拉链：使用zip将两个列表，组合成一个元素为元组的列表
拉开：将一个包含元组的列表，解开成包含两个列表的元组

scala> val a = List("zhangsan", "lisi", "wangwu")
a: List[String] = List(zhangsan, lisi, wangwu)

scala> val b = List(19, 20, 21)
b: List[Int] = List(19, 20, 21)

scala> a.zip(b)
res1: List[(String, Int)] = List((zhangsan,19), (lisi,20), (wangwu,21))

拉开
scala> res1.unzip
res2: (List[String], List[Int]) = (List(zhangsan, lisi, wangwu),List(19, 20, 21))
~~~~

~~~scala
转换字符串（ toString ）
生成字符串（ mkString ）

a.toString
a.mkString  a.mkString(":")
~~~

~~~~scala 
并集
union表示对两个列表取并集，不去重
 a1.union(a2)
 a1.union(a2).distinct  去重
~~~~

~~~scala 
交集
intersect表示对两个列表取交集
scala> val a1 = List(1,2,3,4)
a1: List[Int] = List(1, 2, 3, 4)

scala> val a2 = List(3,4,5,6)
a2: List[Int] = List(3, 4, 5, 6)

scala> a1.intersect(a2)
res19: List[Int] = List(3, 4)
~~~

~~~scala
差集
diff表示对两个列表取差集，例如： a1.diff(a2)，表示获取a1在a2中不存在的元素
scala> val a1 = List(1,2,3,4)
a1: List[Int] = List(1, 2, 3, 4)

scala> val a2 = List(3,4,5,6)
a2: List[Int] = List(3, 4, 5, 6)

scala> a1.diff(a2)
res24: List[Int] = List(1, 2)
~~~

## 集 set

Set(集)是代表没有重复元素的集合。Set具备以下性质：

1. 元素不重复
2. 不保证插入顺序

scala中的集也分为两种，一种是不可变集，另一种是可变集。

### 不可变集

语法:

~~~
val/var 变量名 = Set[类型]()
~~~

​	给定元素来创建一个不可变集，语法格式：

~~~
val/var 变量名 = Set(元素1, 元素2, 元素3...)
~~~

操作:

~~~
获取集的大小（size）  a.size
遍历集（和遍历数组一致）   for(i <- a) println(i)
添加一个元素，生成一个Set（+）     a - 1  不是下标 
拼接两个集，生成一个Set（++）      a ++ Set(6,7,8)
拼接集和列表，生成一个Set（++）      a ++ List(6,7,8,9)
~~~

### 可变集

可变集合不可变集的创建方式一致，只不过需要提前导入一个可变集类。

~~~
import scala.collection.mutable.Set
~~~

~~~scala
scala> val a = Set(1,2,3,4)
a: scala.collection.mutable.Set[Int] = Set(1, 2, 3, 4)                          

// 添加元素
scala> a += 5
res25: a.type = Set(1, 5, 2, 3, 4)

// 删除元素
scala> a -= 1
res26: a.type = Set(5, 2, 3, 4)
~~~

## 映射

Map可以称之为映射。它是由键值对组成的集合。在scala中，Map也分为不可变Map和可变Map。


  ### 不可变Map

语法:

~~~scala
val/var map = Map(键->值, 键->值, 键->值...)    // 推荐，可读性更好
val/var map = Map((键, 值), (键, 值), (键, 值), (键, 值)...)

例
val map = Map("zhangsan"->30, "lisi"->40)
val map = Map(("zhangsan", 30), ("lisi", 30))
map("zhangsan")// 根据key获取value
~~~

### 可变Map

定义语法与不可变Map一致。但定义可变Map需要手动导入`import scala.collection.mutable.Map`

~~~scala 
val map = Map("zhangsan"->30, "lisi"->40)

// 修改value
scala> map("zhangsan") = 20
~~~

## interator 迭代器

scala针对每一类集合都提供了一个迭代器（iterator）用来迭代访问集合

使用迭代器遍历集合

- 使用`iterator`方法可以从集合获取一个迭代器
- 迭代器的两个基本操作
  - hasNext——查询容器中是否有下一个元素
  - next——返回迭代器的下一个元素，如果没有，抛出NoSuchElementException
- 每一个迭代器都是有状态的
  - 迭代完后保留在最后一个元素的位置
  - 再次使用则抛出NoSuchElementException
- 可以使用while或者for来逐个返回元素

例子:

~~~scala
val ite = a.iterator
 while(ite.hasNext) {
     | println(ite.next)
     | }
~~~

## 函数式编程

我们将来使用Spark/Flink的大量业务代码都会使用到函数式编程。下面的这些操作是学习的重点。

- 遍历（`foreach`）

- 映射（`map`）

- 映射扁平化（`flatmap`）

- 过滤（`filter`）

- 是否存在（`exists`）

- 排序（`sorted`、`sortBy`、`sortWith`）

- 分组（`groupBy`）

- 聚合计算（`reduce`）

- 折叠（`fold`）


可以使用类型推断简化函数定义

- scala可以自动来推断出来集合中每个元素参数的类型
- 创建函数时，可以省略其参数列表的类型

使用下划线简化函数定义

​        函数参数，只在函数体中出现一次，而且函数体没有嵌套调用时，可以使用下划线来简化函数定义		

​		如果方法参数是函数，如果出现了下划线，scala编译器会自动将代码封装到一个函数中

​		参数列表也是由scala编译器自动处理

### 1 遍历

~~~scala 
val a = List(1,2,3,4)
a.foreach((x:Int)=>println(x))

使用类型推断简化
a.foreach(x=>println(x))
使用下划线简化
a.foreach(println(_))
~~~

### 2 映射

集合的映射操作是将来在编写Spark/Flink用得最多的操作，是我们必须要掌握的。因为进行数据计算的时候，就是一个将一种数据类型转换为另外一种数据类型的过程。

map方法接收一个函数，将这个函数应用到每一个元素，返回一个新的列表

用法:

~~~scala 
def map[B](f: (A) ⇒ B): TraversableOnce[B]
~~~

| map方法 | API                | 说明                                    |
| :---- | :----------------- | :------------------------------------ |
| 泛型    | [B]                | 指定map方法最终返回的集合泛型                      |
| 参数    | f: (A) ⇒ B         | 传入一个函数对象 该函数接收一个类型A（要转换的列表元素），返回值为类型B |
| 返回值   | TraversableOnce[B] | B类型的集合                                |

例:

~~~scala
var a=List(1,2,3,4)
scala> a.map(x=>x+1)
res4: List[Int] = List(2, 3, 4, 5)
~~~

~~~scala 
val a = List(1,2,3,4)
a.map(_ + 1)
~~~

### 3 扁平化映射

扁平化映射也是将来用得非常多的操作，也是必须要掌握的。

定义

~~~
可以把flatMap，理解为先map，然后再flatten
map是将列表中的元素转换为一个List
flatten再将整个列表进行扁平化

~~~

方法签名:

~~~scala
def flatMap[B](f: (A) ⇒ GenTraversableOnce[B]): TraversableOnce[B]
~~~

| flatmap方法 | API                            | 说明                               |
| :-------- | :----------------------------- | :------------------------------- |
| 泛型        | [B]                            | 最终要转换的集合元素类型                     |
| 参数        | f: (A) ⇒ GenTraversableOnce[B] | 传入一个函数对象 函数的参数是集合的元素 函数的返回值是一个集合 |
| 返回值       | TraversableOnce[B]             | B类型的集合                           |

**案例说明**

1. 有一个包含了若干个文本行的列表："hadoop hive spark flink flume", "kudu hbase sqoop storm"
2. 获取到文本行中的每一个单词，并将每一个单词都放到列表中

~~~~scala
// 定义文本行列表
scala> val a = List("hadoop hive spark flink flume", "kudu hbase sqoop storm")
a: List[String] = List(hadoop hive spark flink flume, kudu hbase sqoop storm)

// 使用map将文本行转换为单词数组
scala> a.map(x=>x.split(" "))
res5: List[Array[String]] = List(Array(hadoop, hive, spark, flink, flume), Array(kudu, hbase, sqoop, storm))

// 扁平化，将数组中的
scala> a.map(x=>x.split(" ")).flatten
res6: List[String] = List(hadoop, hive, spark, flink, flume, kudu, hbase, sqoop, storm)
~~~~

**使用flatMap简化操作**

~~~scala
scala>  val a = List("hadoop hive spark flink flume", "kudu hbase sqoop storm")
a: List[String] = List(hadoop hive spark flink flume, kudu hbase sqoop storm)

scala> a.flatMap(_.split(" "))
res7: List[String] = List(hadoop, hive, spark, flink, flume, kudu, hbase, sqoop, storm)
~~~

### 4 过滤

过滤符合一定条件的元素

~~~scala
def filter(p: (A) ⇒ Boolean): TraversableOnce[A]
~~~

| filter方法 | API                | 说明                                       |
| :------- | :----------------- | :--------------------------------------- |
| 参数       | p: (A) ⇒ Boolean   | 传入一个函数对象 接收一个集合类型的参数 返回布尔类型，满足条件返回true, 不满足返回false |
| 返回值      | TraversableOnce[A] | 列表                                       |

~~~scala
//过滤偶数   结果得到的权威偶数
List(1,2,3,4,5,6,7,8,9).filter(_ % 2 == 0)
~~~

### 5 排序

在scala集合中，可以使用以下几种方式来进行排序

- sorted默认排序
- sortBy指定字段排序
- sortWith自定义排序

~~~scala
//默认排序
List(3,1,2,9,7).sorted
~~~

指定字段排序

~~~scala
def sortBy[B](f: (A) ⇒ B): List[A]
~~~

| sortBy方法 | API        | 说明                                |
| :------- | :--------- | :-------------------------------- |
| 泛型       | [B]        | 按照什么类型来进行排序                       |
| 参数       | f: (A) ⇒ B | 传入函数对象 接收一个集合类型的元素参数 返回B类型的元素进行排序 |
| 返回值      | List[A]    | 返回排序后的列表                          |



```scala
val a = List("01 hadoop", "02 flume", "03 hive", "04 spark")
 a.sortBy(_.split(" ")(1))
res8: List[String] = List(02 flume, 01 hadoop, 03 hive, 04 spark)  //按照单词字母进行排序
```

  ### 自定义排序

自定义排序，根据一个函数来进行自定义排序

方法签名

~~~scala
def sortWith(lt: (A, A) ⇒ Boolean): List[A]
~~~

| sortWith方法 | API                  | 说明                                       |
| :--------- | :------------------- | :--------------------------------------- |
| 参数         | lt: (A, A) ⇒ Boolean | 传入一个比较大小的函数对象 接收两个集合类型的元素参数 返回两个元素大小，小于返回true，大于返回false |
| 返回值        | List[A]              | 返回排序后的列表                                 |

~~~scala
scala> val a = List(2,3,1,6,4,5)
a: List[Int] = List(2, 3, 1, 6, 4, 5)

scala> a.sortWith((x,y) => if(x<y)true else false)  
res15: List[Int] = List(1, 2, 3, 4, 5, 6)

scala> res15.reverse
res18: List[Int] = List(6, 5, 4, 3, 2, 1)   
~~~

使用下划线简写上述案例

~~~scala
scala> val a = List(2,3,1,6,4,5)
a: List[Int] = List(2, 3, 1, 6, 4, 5)

// 函数参数只在函数中出现一次，可以使用下划线代替
scala> a.sortWith(_ < _).reverse
res19: List[Int] = List(6, 5, 4, 3, 2, 1)
~~~

## 分组

我们如果要将数据按照分组来进行统计分析，就需要使用到分组方法

groupBy表示按照函数将列表分成不同的组

~~~scala
//方法签名
def groupBy[K](f: (A) ⇒ K): Map[K, List[A]]
~~~

| groupBy方法 | API             | 说明                                       |
| :-------- | :-------------- | :--------------------------------------- |
| 泛型        | [K]             | 分组字段的类型                                  |
| 参数        | f: (A) ⇒ K      | 传入一个函数对象 接收集合元素类型的参数 返回一个K类型的key，这个key会用来进行分组，相同的key放在一组中 |
| 返回值       | Map[K, List[A]] | 返回一个映射，K为分组字段，List为这个分组字段对应的一组数据         |

1. 有一个列表，包含了学生的姓名和性别:

   ```scala
    "张三", "男"
    "李四", "女"
    "王五", "男"
   ```

2. 请按照性别进行分组，统计不同性别的学生人数

~~~scala
scala> val a = List("张三"->"男", "李四"->"女", "王五"->"男")
a: List[(String, String)] = List((张三,男), (李四,女), (王五,男))

// 按照性别分组
scala> a.groupBy(_._2)
res0: scala.collection.immutable.Map[String,List[(String, String)]] = Map(男 -> List((张三,男), (王五,男)),
女 -> List((李四,女)))

// 将分组后的映射转换为性别/人数元组列表
scala> res0.map(x => x._1 -> x._2.size)
res3: scala.collection.immutable.Map[String,Int] = Map(男 -> 2, 女 -> 1)
~~~

## 聚合

聚合操作，可以将一个列表中的数据合并为一个。这种操作经常用来统计分析中

reduce表示将列表，传入一个函数进行聚合计算

方法签名

~~~scala
def reduce[A1 >: A](op: (A1, A1) ⇒ A1): A1
~~~

| reduce方法 | API               | 说明                                       |
| :------- | :---------------- | :--------------------------------------- |
| 泛型       | [A1 >: A]         | （下界）A1必须是集合元素类型的父类                       |
| 参数       | op: (A1, A1) ⇒ A1 | 传入函数对象，用来不断进行聚合操作 第一个A1类型参数为：当前聚合后的变量 第二个A1类型参数为：当前要进行聚合的元素 |
| 返回值      | A1                | 列表最终聚合为一个元素                              |


   注意:

- reduce和reduceLeft效果一致，表示从左到右计算
- reduceRight表示从右到左计算

1. 定义一个列表，包含以下元素：1,2,3,4,5,6,7,8,9,10
2. 使用reduce计算所有元素的和

```scala
scala> val a = List(1,2,3,4,5,6,7,8,9,10)
a: List[Int] = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

scala> a.reduce((x,y) => x + y)
res5: Int = 55

// 第一个下划线表示第一个参数，就是历史的聚合数据结果
// 第二个下划线表示第二个参数，就是当前要聚合的数据元素
scala> a.reduce(_ + _)
res53: Int = 55

// 与reduce一样，从左往右计算
scala> a.reduceLeft(_ + _)
res0: Int = 55

// 从右往左聚合计算
scala> a.reduceRight(_ + _)
res1: Int = 55
```

  ## 折叠

fold与reduce很像，但是多了一个指定初始值参数

方法签名

~~~scala
def fold[A1 >: A](z: A1)(op: (A1, A1) ⇒ A1): A1
~~~

| reduce方法 | API               | 说明                                       |
| :------- | :---------------- | :--------------------------------------- |
| 泛型       | [A1 >: A]         | （下界）A1必须是集合元素类型的父类                       |
| 参数1      | z: A1             | 初始值                                      |
| 参数2      | op: (A1, A1) ⇒ A1 | 传入函数对象，用来不断进行折叠操作 第一个A1类型参数为：当前折叠后的变量 第二个A1类型参数为：当前要进行折叠的元素 |
| 返回值      | A1                | 列表最终折叠为一个元素                              |

 注意:

- fold和foldLet效果一致，表示从左往右计算
- foldRight表示从右往左计算

1. 定义一个列表，包含以下元素：1,2,3,4,5,6,7,8,9,10
2. 使用fold方法计算所有元素的和

~~~scala
scala> val a = List(1,2,3,4,5,6,7,8,9,10)
a: List[Int] = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

scala> a.fold(0)(_ + _)
res4: Int = 155
~~~

