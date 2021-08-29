---
title: Scala进阶2
abbrlink: 33510
date: 2017-08-12 10:11:17
tags: Scala
categories: Scala
summary_img:
encrypt:
enc_pwd:
---

#一   样例类

样例类是一种特殊类，它可以用来快速定义一个用于**保存数据**的类（类似于Java POJO类），而且它会自动生成apply方法，允许我们快速地创建样例类实例对象。后面，在并发编程和spark、flink这些框架也都会经常使用它。

1. 样例类可以使用**类名(参数1, 参数2...)**快速创建实例对象
2. 定义样例类成员变量时，可以指定var类型，表示可变。默认是不可变的 val  可省略
3. 样例类自动生成了toString、equals、hashCode、copy方法
4. 样例对象没有主构造器，可以使用样例对象来创建枚举、或者标识一类没有任何数据的消息

语法:

~~~~scala
case class 样例类名(成员变量名1:类型1, 成员变量名2:类型2, 成员变量名3:类型3)
~~~~

**定义样例类**

~~~scala
object Demo1 {
  case class Per(name:String, var age:Int)

  def main(args: Array[String]): Unit = {
    val per = Per("你猜",25)
    //per.name="njj"   会报错
    per.age=25  //正常可以修改
    println(per)
  }
}
~~~

### 样例类的方法

定义样例类编译器自动帮我们实现了一下几个方法

~~~
apply   快速的用类名创建对象
toString   与java同
equals      比较两个样例类的成员变量是否相等  与==类似
hashCode    如两个样例类的所有的成员变量的值相等 则hash值相等 否则 只要一个不同 则hash值就不等
copy     样例类的克隆
~~~

~~~~scala
object Demo2 {
  case class Per(name:String,age:Int)

  def main(args: Array[String]): Unit = {
    val nicai = Per("nicai",55)
    val nn = nicai.copy("nn")
      //可以修改成员变量的值
    println(nn)
  }
}
~~~~

### 样例对象

使用case object可以创建样例对象。样例对象是单例的，而且它**没有主构造器**。样例对象是可序列化的。格式：

~~~scala
case object 样例对象名
~~~

它主要用在两个地方：

1. 定义枚举
2. 作为没有任何参数的消息传递（后面Akka编程会讲到）

~~~~SCALA 
object Demo3 {
  //定义一个枚举
  trait Sex
  case object Man extends Sex
  case object Wonmn extends Sex

 case class Per(name:String,sex: Sex)

  def main(args: Array[String]): Unit = {
    val per = Per("小明",Man)
    val p = Per("小红",Wonmn)
    println(per)
    println(p)
  }

}

~~~~

定义消息

~~~scala
case class StartSpeakingMessage(textToSpeak: String)
// 消息如果没有任何参数，就可以定义为样例对象
case object StopSpeakingMessage
case object PauseSpeakingMessage
case object ResumeSpeakingMessage
~~~

# 二模板匹配

scala中有一个非常强大的模式匹配机制，可以应用在很多场景：

- switch语句
- 类型查询
- 以及快速获取数据

### 简单模式匹配

相当于java中的switch语句

语法

~~~scala
变量 match {
    case "常量1" => 表达式1
    case "常量2" => 表达式2
    case "常量3" => 表达式3
    case _ => 表达式4		// 默认匹配
}
~~~

~~~scala
object Demo1 {
  def main(args: Array[String]): Unit = {
    val str = StdIn.readLine()   //从键盘录入

    val unit = str match {
      case "hadoop" => "nicai"
      case "spaker" => "分布式计算框架"
      case _ => "未匹配"
    }
    println(unit)
  }

}
~~~

### 匹配类型

根据不同的数据类型进行匹配

~~~scala
变量 match {
    case 类型1变量名: 类型1 => 表达式1
    case 类型2变量名: 类型2 => 表达式2
    case 类型3变量名: 类型3 => 表达式3
    ...
    case _ => 表达式4
}
~~~

~~~scala
object Demo2 {
  var  a:Any="hadoop"

  def main(args: Array[String]): Unit = {
    val unit = a match {
      case x: String => s"${x}字符串"   //若后面没有用到这个变量 可以写为 _:String
      case x: Int => "整形"
      case x: Double => "浮点型"
      case _ => "没匹配"
    }
    println(unit)
  }
}

~~~

### 守卫

在Java中，只能简单地添加多个case标签，例如：要匹配0-7，就需要写出来8个case语句。例如：

~~~scala
int a = 0;
switch(a) {
    case 0: a += 1;
    case 1: a += 1;
    case 2: a += 1;
    case 3: a += 1;
    case 4: a += 2;
    case 5: a += 2;
    case 6: a += 2;
    case 7: a += 2;
    default: a = 0;
}
~~~

在scala中，可以使用守卫来简化上述代码——也就是在**case语句中添加if条件判断**。

~~~scala
object Demo3 {
    private val i: Int = StdIn.readInt()

  def main(args: Array[String]): Unit = {
    val unit = i match {
      case x if x > 0 && x < 3 => "0-3"   
      case x if x > 3 && x < 10 => println("3-10")  //若这样鞋  则0-3 之后会打印()
      case x if x > 10 && x < 13 => println("10-13")
      case _ => println("weipi")
    }
    println(unit)
  }
}
~~~

### 匹配样例类

scala可以使用模式匹配来匹配样例类，从而可以快速获取样例类中的成员数据。后续，我们在开发Akka案例时，还会用到。

~~~scala
object Demo4 {
  case class Per(name:String,age:Int)
  case class Stu(name:String,age:Int)

  def main(args: Array[String]): Unit = {
    val ni:Any = Per("ni",55)              //若不为any则会报错
    val unit = ni match {
      case Per(name, age) => s"${name}:${age}per"
      case Stu(name, age) => s"${name}:${age}stu"
      case _ => "未匹配"
    }
    println(unit)
  }
}

~~~

### 匹配集合

1 匹配数组

- 依次修改代码定义以下三个数组

  ```scala
    Array(1,x,y)   // 以1开头，后续的两个元素不固定
    Array(0)       // 只匹配一个0元素的元素
    Array(0, ...)  // 可以任意数量，但是以0开头
  ```

- 使用模式匹配上述数组

**参考代码**

```scala
val arr = Array(1, 3, 5)
arr match {
    case Array(1, x, y) => println(x + " " + y)
    case Array(0) => println("only 0")
    case Array(0, _*) => println("0 ...")
    case _ => println("something else")
}
```

2匹配列表

- 依次修改代码定义以下三个列表

  ```scala
    List(0)                // 只保存0一个元素的列表
    List(0,...)           // 以0开头的列表，数量不固定
    List(x,y)               // 只包含两个元素的列表
  ```

- 使用模式匹配上述列表

**参考代码**

```scala
val list = List(0, 1, 2)

list match {
    case 0 :: Nil => println("只有0的列表")
    case 0 :: tail => println("0开头的列表")
    case x :: y :: Nil => println(s"只有另两个元素${x}, ${y}的列表")
    case _ => println("未匹配")
}
```

3 匹配元组

- 依次修改代码定义以下两个元组

  ```scala
    (1, x, y)        // 以1开头的、一共三个元素的元组
    (x, y, 5)   // 一共有三个元素，最后一个元素为5的元组
  ```

- 使用模式匹配上述元素

**参考代码**

```scala
val tuple = (2, 2, 5)

tuple match {
    case (1, x, y) => println(s"三个元素，1开头的元组：1, ${x}, ${y}")
    case (x, y, 5) => println(s"三个元素，5结尾的元组：${x}, ${y}, 5")
    case _ => println("未匹配")
}
```

### 变量声明中的模式匹配

在定义变量的时候，可以使用模式匹配快速获取数据。

1 获取数组中的元素

- 生成包含0-10数字的数组，使用模式匹配分别获取第二个、第三个、第四个元素

**参考代码**

```scala
package com.nicai.demo.matchdemo
//变量声明中的模式匹配
object Demo8 {
  def main(args: Array[String]): Unit = {
    var array=(0 to 10).toArray
    var Array(_,x,y,z,_*)=array
    println(x)  //1
    println(y)  //2
    println(z)   //3
  }

}

```

2 获取列表中的数据

- 生成包含0-10数字的列表，使用模式匹配分别获取第一个、第二个元素

**参考代码**

```scala
package com.nicai.demo.matchdemo
//变量声明中的模式匹配
object Demo9 {
  def main(args: Array[String]): Unit = {
    var a= (0 to 10).toList
    var x :: y :: tail =a

    println(x) //0
    println(y)  //1
  }

}

```

### option类型

scala中，Option类型来表示可选值。这种类型的数据有两种形式：

Some(x)：表示实际的值

None：表示没有值

使用Option类型，可以用来有效避免空引用(null)异常。也就是说，将来我们返回某些数据时，可以返回一个Option类型来替代。

getOrElse方法

使用getOrElse方法，当Option对应的实例是None时，可以指定一个默认值，从而避免空指针异常

1. scala鼓励使用Option类型来封装数据，可以有效减少，在代码中判断某个值是否为null
2. 可以使用getOrElse方法来针对None返回一个默认值

例子一

- 定义一个两个数相除的方法，使用Option类型来封装结果
- 然后使用模式匹配来打印结果
  - 不是除零，打印结果
  - 除零打印异常错误

**参考代码**

```scala
 package com.nicai.demo.matchdemo

object Demo10 {
  def div(a:Double,b:Int): Option[Double] ={
    if (b != 0){
      Some(a/b)
    }else {
      None
    }
  }

  def main(args: Array[String]): Unit = {
    val option = div(15.2,2)
    val unit = option match {
      case Some(x) => x
      case None => "除数不可为0"
    }
    println(unit)  //7.6
  }
}

```

例子二

- 重写上述案例，使用getOrElse方法，当除零时，或者默认值为0

**参考代码**

```scala
package com.nicai.demo.matchdemo

object Demo11 {
  def div(a: Double, b: Int):Option[Double]= {
    if( b != 0){
      Some(a/b)
    }else{
      None
    }
  }

  def main(args: Array[String]): Unit = {
    val d = div(15.6,0).getOrElse(0)
    println(d)
  }
}

```

  

### 偏函数

偏函数可以提供了简洁的语法，可以简化函数的定义。配合集合的函数式编程，可以让代码更加优雅。

## 定义

- 偏函数被包在花括号内没有match的一组case语句是一个偏函数
- 偏函数是PartialFunction[A, B]的一个实例
  - A代表输入参数类型
  - B代表返回结果类型

可以理解为：偏函数是一个参数和一个返回值的函数。

~~~scala
package com.nicai.demo.PartialFunctionDemo

object Demo12 {

  private val value: PartialFunction[Int, String] = {
    case 1 => "一"
    case 2 => "二"
    case _ => "其他"
  }

  def main(args: Array[String]): Unit = {
    println(value(1))
  }
}

~~~

- 定义一个列表，包含1-10的数字
- 请将1-3的数字都转换为[1-3]
- 请将4-8的数字都转换为[4-8]
- 将其他的数字转换为(8-星]

**参考代码**

```scala
val list = (1 to 10).toList

val list2 = list.map{
    case x if x >= 1 && x <= 3 => "[1-3]"
    case x if x >= 4 && x <= 8 => "[4-8]"
    case x if x > 8 => "(8-*]"
}

println(list2)
```

### 正则表达式

在scala中，可以很方便地使用正则表达式来匹配数据。

scala中提供了Regex类来定义正则表达式，要构造一个RegEx对象，直接使用String类的r方法即可。

建议使用三个双引号来表示正则表达式，不然就得对正则中的反斜杠来进行转义。

~~~scala
val regEx = """正则表达式""".r
~~~

findAllMatchIn方法

使用findAllMatchIn方法可以获取所有正则匹配到的字符串

~~~~scala
示例说明

定义一个正则表达式，来匹配邮箱是否合法
合法邮箱测试：qq12344@163.com
不合法邮箱测试：qq12344@.com
val r = """.+@.+\..+""".r

val eml1 = "qq12344@163.com"
val eml2 = "qq12344@.com"

if(r.findAllMatchIn(eml1).size > 0) {  //z\size  为0 没有匹配上  大于0 为匹配上
    println(eml1 + "邮箱合法")
}
else {
    println(eml1 + "邮箱不合法")
}

if(r.findAllMatchIn(eml2).size > 0) {
    println(eml2 + "邮箱合法")
}
else {
    println(eml2 + "邮箱不合法")
}
~~~~

找出以下列表中的所有不合法的邮箱

```html
"38123845@qq.com", "a1da88123f@gmail.com", "zhansan@163.com", "123afadff.com"
```

~~~~scala
package com.nicai.demo.zhengzebiaodashi
//匹配多个邮箱
object Demo15 {
  def main(args: Array[String]): Unit = {
    var a= List("38123845@qq.com", "a1da88123f@gmail.com", "zhansan@163.com", "123afadff.com")

    val r=""".+@.+\.com""".r
    val strings = a.filter {
      //过滤出不合法的
      case x if r.findAllMatchIn(x).size == 0 => true
      case _ => false
    }
    println(strings)
  }

}

~~~~

- 有以下邮箱列表

  ```scala
    "38123845@qq.com", "a1da88123f@gmail.com", "zhansan@163.com", "123afadff.com"
  ```

- 使用正则表达式进行模式匹配，匹配出来邮箱运营商的名字。例如：邮箱zhansan@163.com，需要将163匹配出来

  - 使用括号来匹配分组

- 打印匹配到的邮箱以及运营商

~~~scala
package com.nicai.demo.zhengzebiaodashi

object Demo16 {
  def main(args: Array[String]): Unit = {
    //括号为分组
    val re =""".+@(.+)\.com""".r   //此处必为 val
    var li=List("38123845@qq.com", "a1da88123f@gmail.com", "zhansan@163.com", "123afadff.com")
    val strings = li.map {
      //company为分组的名字  就是分组的字段
      case x@re(company) => s"${x} -> ${company}"
      case x => s"${x} + 未知"
    }
    println(strings)
  }
}

~~~

## 异常处理

来看看下面一段代码。

```scala
  def main(args: Array[String]): Unit = {
   val i = 10 / 0

    println("你好！")
  }

Exception in thread "main" java.lang.ArithmeticException: / by zero
    at ForDemo$.main(ForDemo.scala:3)
    at ForDemo.main(ForDemo.scala)
```

执行程序，可以看到scala抛出了异常，而且没有打印出来"你好"。说明程序出现错误后就终止了。

那怎么解决该问题呢？

在scala中，可以使用异常处理来解决这个问题

### 捕获异常

**语法格式**

```scala
try {
    // 代码
}
catch {
    case ex:异常类型1 => // 代码
    case ex:异常类型2 => // 代码
}
finally {
    // 代码
}
```

- try中的代码是我们编写的业务处理代码
- 在catch中表示当出现某个异常时，需要执行的代码
- 在finally中，是不管是否出现异常都会执行的代码

## 示例

**示例说明**

- 使用try..catch来捕获除零异常

**参考代码**

```scala
package com.nicai.demo.exceptionDemo

object Demo17 {
  def main(args: Array[String]): Unit = {
    try{
      var a= 4/0
    }catch {
      case ex:Exception => println(ex.getMessage)
    }

  }
}
```

###抛出异常

我们也可以在一个方法中，抛出异常。语法格式和Java类似，使用`throw new Exception...`

例子:

- 在main方法中抛出一个异常

**参考代码**

```scala
 package com.nicai.demo.exceptionDemo

object Demo18 {
  def main(args: Array[String]): Unit = {
    throw new Exception("这是一个异常")
  }

}


Exception in thread "main" java.lang.Exception: 这是一个异常
    at ForDemo$.main(ForDemo.scala:3)
    at ForDemo.main(ForDemo.scala)
```

- scala不需要在方法上声明要抛出的异常，它已经解决了再Java中被认为是设计失败的检查型异常。

下面是Java代码

```java
public static void main(String[] args) throws Exception {
    throw new Exception("这是一个异常");
}
```


## 提取器

我们之前已经使用过scala中非常强大的模式匹配功能了，通过模式匹配，我们可以快速匹配样例类中的成员变量.  

那是不是所有的类都可以进行这样的模式匹配呢？答案是：

`不可以`的。要支持模式匹配，必须要实现一个**提取器**。

**样例类自动实现了apply、unapply方法**

##  定义提取器

之前我们学习过了，实现一个类的伴生对象中的apply方法，可以用类名来快速构建一个对象。伴生对象中，还有一个unapply方法。与apply相反，unapply是将该类的对象，拆解为一个个的元素。

要实现一个类的提取器，只需要在该类的伴生对象中实现一个unapply方法即可

**语法格式**

```scala
def unapply(stu:Student):Option[(类型1, 类型2, 类型3...)] = {
    if(stu != null) {
        Some((变量1, 变量2, 变量3...))
    }
    else {
        None
    }
}
```

**示例说明**

- 创建一个Student类，包含姓名年龄两个字段
- 实现一个类的解构器，并使用match表达式进行模式匹配，提取类中的字段。

**参考代码**

```scala
package com.nicai.demo.tiquqi

object Demo19 {
  class Stu(var name:String,var age:Int)
  object Stu{
    def apply(name: String, age: Int): Stu = new Stu(name, age)

    def unapply(stu :Stu) = {
      val tuple =(stu.name,stu.age)
      Some(tuple)
    }
  }

  def main(args: Array[String]): Unit = {
    val nicai = Stu("nicai",55)
    val unit = nicai match {
      case Stu(name, age) => s"${name}:${age}"
    }
    println(unit)
  }
}

```

## 泛型

scala和Java一样，类和特质、方法都可以支持泛型。我们在学习集合的时候，一般都会涉及到泛型。

## 定义一个泛型方法

在scala中，使用方括号来定义类型参数。

**语法格式**

```scala
def 方法名[泛型名称](..) = {
    //...
}
```

**示例说明**

- 用一个方法来获取任意类型数组的中间的元素
  - 不考虑泛型直接实现（基于Array[Int]实现）
  - 加入泛型支持

**参考代码**

不考虑泛型的实现

```scala
  def getMiddle(arr:Array[Int]) = arr(arr.length / 2)

  def main(args: Array[String]): Unit = {
    val arr1 = Array(1,2,3,4,5)

    println(getMiddle(arr1))
  }
```

加入泛型支持

```scala
package com.nicai.demo.fanxing

object Demo20 {
  def getMid[T](array: Array[T])= array(array.length/2)

  def main(args: Array[String]): Unit = {
    println(getMid(Array(1, 2, 3)))
    println(getMid(Array("dd", "uu", "sss")))
  }
}

```

##泛型类

scala的类也可以定义泛型。接下来，我们来学习如何定义scala的泛型类

## 定义

**语法格式**

```scala
class 类[T](val 变量名: T)
```

- 定义一个泛型类，直接在类名后面加上方括号，指定要使用的泛型参数
- 指定类对应的泛型参数后，就使用这些类型参数来定义变量了

## 示例

**参考代码**

```scala
package com.nicai.demo.fanxing

object Demo21 {
  case class Per[y] (name:y,age:y)

  def main(args: Array[String]): Unit = {
    val list = List(
      Per("NJJS", 45),
      Per("jsjj", 789),
      Per(56456, "SSS")
    )
    println(list)
  }
}

```

## 上下界

需求：

我们在定义方法/类的泛型时，限定必须从哪个类继承、或者必须是哪个类的父类。此时，就需要使用到上下界。    

**上界定义:**

使用`<: 类型名`表示给类型添加一个**上界**，表示泛型参数必须要从该类（或本身）继承

**语法格式**

```scala
[T <: 类型]
```

**示例说明**

**参考代码**

```scala
package com.nicai.demo.fanxing

object Demo22 {
   //上界
  class  Per
  class Stu extends Per
  class Man extends Stu
  def m[t <: Stu](a:Array[t]) = println(a)   //Per 本身及其子类

  def main(args: Array[String]): Unit = {
    // 编译报错
   // m(Array(new Per))
    m(Array(new Stu))
  }
}

```

**下界**

上界是要求必须是某个类的子类，或者必须从某个类继承，而下界是必须是**某个类的父类**（或本身）

**语法格式**

```scala
[T >: 类型]
```

注意:

如果类既有上界、又有下界。下界写在前面，上界写在后面    (同时又上下界,可能会守不住,即范围之外的也可以)

**示例说明**

**参考代码**

```scala
package com.nicai.demo.fanxing


object Demo23 {
//下界

  class  Per
  class Stu extends Per
  class Man extends Stu
  def m[T >: Stu](a:Array[T])= println(a)  //Stu  本身及其父类

  def main(args: Array[String]): Unit = {
    m(Array(new Stu))
    m(Array(new Per))
    //会报错
    //m(Array(new Man))
  }
}

```


  ## 协变 逆变 非变

spark的源代码中大量使用到了协变、逆变、非变，学习该知识点对我们将来阅读spark源代码很有帮助。

来看一个类型转换的问题：

```scala
class Pair[T]

object Pair {
  def main(args: Array[String]): Unit = {
    val p1 = Pair("hello")
    // 编译报错，无法将p1转换为p2
    val p2:Pair[AnyRef] = p1

    println(p2)
  }
}
```

如何让带有泛型的类支持类型转换呢？

**非变**

**语法格式**

```scala
class Pair[T]{}
```

- 默认泛型类是非变的
- 类型B是A的子类型，Pair[A]和Pair[B]没有任何从属关系
- Java是一样的

![img](/images/scala/fxn.png)

### 协变

**语法格式**

```scala
class Pair[+T]
```

- 类型B是A的子类型，Pair[B]可以认为是Pair[A]的子类型
- 参数化类型的方向和类型的方向是一致的。

### 逆变

**语法格式**

```scala
class Pair[-T]
```

- 类型B是A的子类型，Pair[A]反过来可以认为是Pair[B]的子类型
- 参数化类型的方向和类型的方向是相反的

**参考代码**

```scala
class Super
class Sub extends Super

class Temp1[T]
class Temp2[+T]
class Temp3[-T]

def main(args: Array[String]): Unit = {
    val a:Temp1[Sub] = new Temp1[Sub]
    // 编译报错
    // 非变
    //val b:Temp1[Super] = a

    // 协变
    val c: Temp2[Sub] = new Temp2[Sub]
    val d: Temp2[Super] = c

    // 逆变
    val e: Temp3[Super] = new Temp3[Super]
    val f: Temp3[Sub] = e
}
```

  ## Actor并发编程

scala的Actor并发编程模型可以用来开发比Java线程效率更高的并发程序。我们学习scala Actor的目的主要是为后续学习Akka做准备。

## Java并发编程的问题

在Java并发编程中，每个对象都有一个逻辑监视器（monitor），可以用来控制对象的多线程访问。我们添加sychronized关键字来标记，需要进行同步加锁访问。这样，通过加锁的机制来确保同一时间只有一个线程访问共享数据。但这种方式存在资源争夺、以及死锁问题，程序越大问题越麻烦。

思索问题

![img](/images/scala/xcss.png)

例子:

~~~java
package com.nicai.Demo;

public class MyLock {
    public static Object obja = new Object();
    public static Object objb = new Object();
}

class DieLock extends Thread {
    private boolean flag;

    public DieLock(boolean flag) {
         this.flag=flag;
    }

    @Override
    public void run() {
        if(flag){
            synchronized (MyLock.obja){
                System.out.println("a");
                synchronized (MyLock.objb){
                    System.out.println("b");
                }
            }
        }else {
            synchronized (MyLock.objb){
                System.out.println("bb");
                synchronized (MyLock.obja){
                    System.out.println("aa");
                }
            }
        }
    }
    public static void main(String[] args){
        DieLock lock1 = new DieLock(true);
        DieLock lock2 = new DieLock(false);
        lock1.start();
        lock2.start();
    }
}
~~~

### Artor并发编程模型

Actor并发编程模型，是scala提供给程序员的一种与Java并发编程完全不一样的并发编程模型，是一种基于事件模型的并发机制。Actor并发编程模型是一种不共享数据，依赖消息传递的一种并发编程模式，有效避免资源争夺、死锁等情况。

![img](/images/scala/bf.png)

### java 并发编程 与Actor并发编程对比

| Java内置线程模型                       | scala Actor模型            |
| :------------------------------- | :----------------------- |
| "共享数据-锁"模型 (share data and lock) | share nothing            |
| 每个object有一个monitor，监视线程对共享数据的访问  | 不共享数据，Actor之间通过Message通讯 |
| 加锁代码使用synchronized标识             |                          |
| 死锁问题                             |                          |
| 每个线程内部是顺序执行的                     | 每个Actor内部是顺序执行的          |

 **注意**

scala在2.11.x版本中加入了Akka并发编程框架，老版本已经废弃。Actor的编程模型和Akka很像，我们这里学习Actor的目的是为学习Akka做准备。

### 创建Actor

创建Actor的方式和Java中创建线程很类似，也是通过继承来创建。


  使用方式

1. 定义class或object继承Actor特质
2. 重写act方法
3. 调用Actor的start方法执行Actor

类似于Java线程，这里的每个Actor是并行执行的


**示例说明**

创建两个Actor，一个Actor打印1-10，另一个Actor打印11-20

- 使用class继承Actor创建（如果需要在程序中创建多个相同的Actor）
- 使用object继承Actor创建（如果在程序中只创建一个Actor）

**参考代码**

使用class继承Actor创建

```scala
object _05ActorDemo {
  class Actor1 extends Actor {
    override def act(): Unit = (1 to 10).foreach(println(_))
  }

  class Actor2 extends Actor {
    override def act(): Unit = (11 to 20).foreach(println(_))
  }

  def main(args: Array[String]): Unit = {
    new Actor1().start()
    new Actor2().start()
  }
}
```

使用object继承Actor创建

```scala
package com.nicai.demo.actorDemo

import scala.actors.Actor

object Demo26 {
  object  A1 extends Actor{
    override def act(): Unit = (1 to 10).foreach(println(_)+",")
  }
  object A2 extends Actor {
    override def act(): Unit = (11 to 20).foreach(print(_)+",")
  }

  def main(args: Array[String]): Unit = {
    A1.start()
     A2.start()
  }
}
```

## Actor程序运行流程

1. 调用start()方法启动Actor
2. 自动执行**act**()方法
3. 向Actor发送消息
4. act方法执行完成后，程序会调用**exit()**方法

### 发送消息 与接收消息

我们之前介绍Actor的时候，说过Actor是基于事件（消息）的并发编程模型，那么Actor是如何发送消息和接收消息的呢？

#### 使用方式

**发送消息**

我们可以使用三种方式来发送消息：

| **！**  | **发送异步消息，没有返回值**           |
| :----- | :------------------------- |
| **!?** | **发送同步消息，等待返回值**           |
| **!!** | **发送异步消息，返回值是Future[Any]** |

例如：

要给actor1发送一个异步字符串消息，使用以下代码：

```scala
actor1 ! "你好!"
```

**接收消息**

Actor中使用receive方法来接收消息，需要给receive方法传入一个偏函数

```scala
{
    case 变量名1:消息类型1 => 业务处理1,
    case 变量名2:消息类型2 => 业务处理2,
    ...
}
```

 注意:

receive方法只接收一次消息，接收完后继续执行act方法  

**示例说明**

- 创建两个Actor（ActorSender、ActorReceiver）
- ActorSender发送一个异步字符串消息给ActorReceiver
- ActorReceive接收到该消息后，打印出来

~~~scala
package com.nicai.demo.actorDemo

import java.util.concurrent.TimeUnit

import scala.actors.Actor

object Demo27 {
//发送消息 与 接收消息
//发送
  object  MsgSender extends Actor{
    override def act(): Unit = {
      MsgReceiver ! "nicai"    //给谁发消息
      TimeUnit.SECONDS.sleep(3)
    }
  }
  //接收
  object MsgReceiver extends Actor{
    override def act(): Unit = {
      receive{
        case msg: String => println(msg)
      }
    }
  }

  def main(args: Array[String]): Unit = {
    MsgSender.start()
    MsgReceiver.start()
  }
}

~~~

### 持续接收消息

通过上一个案例，ActorReceiver调用receive来接收消息，但接收一次后，Actor就退出了。

我们希望ActorReceiver能够一直接收消息，怎么实现呢？

——我们只需要使用一个while(true)循环，不停地调用receive来接收消息就可以啦

~~~scala
package com.nicai.demo.actorDemo

import java.util.concurrent.TimeUnit

import scala.actors.Actor

object Demo27 {
//发送消息 与 接收消息
//发送
  object  MsgSender extends Actor{
    override def act(): Unit = {
      MsgReceiver ! "nicai"
      TimeUnit.SECONDS.sleep(3)
    }
  }
  //接收
  object MsgReceiver extends Actor{
    override def act(): Unit = {
      receive{
        case msg: String => println(msg)
      }
    }
  }

  def main(args: Array[String]): Unit = {
    MsgSender.start()
    MsgReceiver.start()
  }
}

~~~

### 使用loop和react 优化接收消息

上述代码，使用while循环来不断接收消息。

- 如果当前Actor没有接收到消息，线程就会处于阻塞状态
- 如果有很多的Actor，就有可能会导致很多线程都是处于阻塞状态
- 每次有新的消息来时，重新创建线程来处理
- 频繁的线程创建、销毁和切换，会影响运行效率

在scala中，可以使用loop + react来复用线程。比while + receive更高效

使用loop + react重写上述案例

**参考代码**

```scala
// 持续接收消息
loop {
    react {
        case msg:String => println("接收到消息：" + msg)
    }
}

改写:
package com.nicai.demo.actorDemo

import java.util.concurrent.TimeUnit


import scala.actors.Actor

object Demo29 {
  object MsgSender extends Actor{
    override def act(): Unit = {
      while(true){
        MsgReceice ! "NICAII"
        TimeUnit.SECONDS.sleep(3)
      }
    }
  }
  object MsgReceice extends Actor{
    override def act(): Unit = {
      loop{
        react{
          case msg :String => println(msg)
        }
      }
    }
  }

  def main(args: Array[String]): Unit = {
    MsgReceice.start()
    MsgSender.start()
  }
}

```

### 发送和接收自定义消息

我们前面发送的消息是字符串类型，Actor中也支持发送自定义消息，常见的如：使用样例类封装消息，然后进行发送处理。

**例子1**

**示例说明**

- 创建一个MsgActor，并向它发送一个同步消息，该消息包含两个字段（id、message）
- MsgActor回复一个消息，该消息包含两个字段（message、name）
- 打印回复消息

注意:

- 使用`!?`来发送同步消息
- 在Actor的act方法中，可以使用sender获取发送者的Actor引用

```scala
//同步的方式
import scala.actors.Actor

//发送和接收自定义消息
object Demo30 {
 //封装发送消息
 case  class Msg(name:String,Age:Int)
  //封装回复消息
 case class ReplyMsg(name:String,addres:String)
  //接收消息
  object MsgActor extends Actor{
    override def act(): Unit = {
      loop{
        react{
          case Msg(name,age) =>{
            println("收到消息"+s"${name}:${age}")
            //获取发送者队象 并回复消息
            sender ! ReplyMsg("wobucai","bbb")
          }
        }
      }
    }
  }

  def main(args: Array[String]): Unit = {
    MsgActor.start()
//发送消息 并获取返回的消息
    val unit:Any = MsgActor !? Msg("nicai",22)
    //转换 消息类型
    if(unit.isInstanceOf[ReplyMsg]){
      println("回复消息"+unit.asInstanceOf[ReplyMsg])
    }
  }
}

```

**实例2**

- 创建一个MsgActor，并向它发送一个异步无返回消息，该消息包含两个字段（message, company）
- 使用`!`发送异步无返回消息

```scala
//异步无返回值
import com.nicai.demo.actorDemo.Demo30.Msg
import scala.actors.Actor
object Demo31 {
  //封装 消息
  case class Mag(name:String,age:Int)

  object  MsgActor extends Actor{
    override def act(): Unit = {
      loop{
        react{
          case Msg(name,age) => {
            println(s"${name}:${age}")
          }
        }
      }

    }
  }
  def main(args: Array[String]): Unit = {
    MsgActor.start()
    //发送消息
    MsgActor ! Msg("你猜",55)
  }
}

```

**例子3**

- 创建一个MsgActor，并向它发送一个异步有返回消息，该消息包含两个字段（id、message）
- MsgActor回复一个消息，该消息包含两个字段（message、name）
- 打印回复消息

 注意:

- 使用`!!`发送异步有返回消息
- 发送后，返回类型为Future[Any]的对象
- Future表示异步返回数据的封装，虽获取到Future的返回值，但不一定有值，可能在将来某一时刻才会返回消息
- Future的isSet()可检查是否已经收到返回消息，apply()方法可获取返回数据

```scala
//异步有返回值
package com.nicai.demo.actorDemo

import scala.actors.Actor

object Demo32 {
    //封装发送消息
  case class Msg(name:String ,age: Int)
  //封装返回消息
  case class ReMsg(name:String ,age: Int)

  //设置接收消息
  object  MsgActor extends Actor{
    override def act(): Unit = {

        loop{
          react{
            case Msg(name,age) =>{
              println(s"${name}:${age}")
              //返回消息
              sender ! ReMsg("NICAI",4564)
            }
          }
        }

    }
  }

  def main(args: Array[String]): Unit = {
    MsgActor.start()

    val unit = MsgActor !! Msg("温暖你的空间",777)
    //if(unit.isInstanceOf[ReMsg]){
      //检查是否已经收到返回消息  apply()方法可获取返回数据
      // 等待所有结果都已返回
    while(!unit.isSet){ }
println(unit.apply().asInstanceOf[ReMsg])
    //}
  }
}

```

##WordCount案例

我们要使用Actor并发编程模型实现多文件的单词统计

需求:

给定几个文本文件（文本文件都是以空格分隔的），使用Actor并发编程来统计单词的数量

![img](/images/scala/wc.png)

**实现思路**

1. MainActor获取要进行单词统计的文件
2. 根据文件数量创建对应的WordCountActor
3. 将文件名封装为消息发送给WordCountActor
4. WordCountActor接收消息，并统计单个文件的单词计数
5. 将单词计数结果发送给MainActor
6. MainActor等待所有的WordCountActor都已经成功返回消息，然后进行结果合并

## 步骤1 | 获取文件列表

**实现思路**

在main方法中读取指定目录(${project_root_dir}/data/)下的所有文件，并打印所有的文件名

**实现步骤**

1. 创建用于测试的数据文件
2. 加载工程根目录，获取到所有文件
3. 将每一个文件名，添加目录路径
4. 打印所有文件名

**参考代码**

```scala
//获取文件目录
   // val DIR="G:\\develop\\bigdatas\\BigData\\day22Scala3\\data/"
    val DIR="day22Scala3/data/"   //当为maven的子工程时 不可使用 "./data/"
    //获取文件流
    val list = new File(DIR).list().toList
    //把每个文件加上前缀 形成完整路径
    val pathAll = list.map(DIR + _)
    println(pathAll)
```

## 步骤2 | 创建WordCountActor

**实现思路**

根据文件数量创建WordCountActor，为了方便后续发送消息给Actor，将每个Actor与文件名关联在一起

**实现步骤**

1. 创建WordCountActor
2. 将文件列表转换为WordCountActor
3. 为了后续方便发送消息给Actor，将Actor列表和文件列表拉链到一起
4. 打印测试

**参考代码**

MainActor.scala

```scala
   //获取wordcountList
    val wordCountList = list.map {
      fileNmae => new WordCountActor()
    }
    //每个 文件路径与 wordcount建立连接
    val tuplesList = wordCountList.zip(pathAll)
    println(tuplesList)

```

WordCountActor.scala

```scala
class WordCountActor extends Actor{
  override def act(): Unit = {
      
  }    
}
```

## 步骤3 | 启动Actor/发送/接收任务消息

**实现思路**

启动所有WordCountActor，并发送单词统计任务消息给每个WordCountActor

 **注意**

此处应发送异步有返回消息

**实现步骤**

1. 创建一个WordCountTask样例类消息，封装要进行单词计数的文件名
2. 启动所有WordCountTask，并发送异步有返回消息
3. 获取到所有的WordCount中获取到的消息（封装到一个Future列表中）
4. 在WordCountActor中接收并打印消息

**参考代码**

MainActor.scala

```scala
 //启动 actor /发送和接收消息
    tuplesList.map{
      actorFileName =>{
        val actor = actorFileName._1
        actor.start()
        val future = actor !! Msg(actorFileName._2)
        future
      }
    }

```

MessagePackage.scala

```scala
/**
  * 单词统计任务消息
  * @param fileName 文件名
  */
case class Msg(name:String)

```

WordCountActor.scala

```scala
class WordCountActor extends Actor{
  override def act(): Unit = {
    loop{
      react{
        //获取消息
        case Msg(fileName) => println("对"+fileName+"进行单词统计")
      }
    }
  }
}

```

## 步骤4 | 消息统计文件单词计数

**实现思路**

读取文件文本，并统计出来单词的数量。例如：

```html
(hadoop, 3), (spark, 1)...
```



**实现步骤**

1. 读取文件内容，并转换为列表
2. 按照空格切割文本，并转换为一个一个的单词
3. 为了方便进行计数，将单词转换为元组
4. 按照单词进行分组，然后再进行聚合统计
5. 打印聚合统计结果

**参考代码**

WordCountActor.scala

```scala
class WordCountActor extends Actor{
  override def act(): Unit = {
    loop{
      react{
        //获取消息
        case Msg(fileName) => println("对"+fileName+"进行单词统计")
          //一 读取文件 获取列表  hadoop sqoop hadoop
        val wordLineList = Source.fromFile(fileName).getLines().toList
          //二 切割字符串,转换为一个一个的单词[hadoop, sqoop, hadoop]
        val wordList = wordLineList.flatMap(_.split(" "))
          //三将单词转换为元组  [<hadoop,1>, <sqoop,1>, <hadoop,1>]
        val wordAndCountList = wordList.map(_ -> 1)
          // 四 对其进行分组 聚合计算
        //4.1 分组 {hadoop->List(<hadoop,1>,<hadoop,1>), sqoop->List(<sqoop,1>)}
        val wordGroubList = wordAndCountList.groupBy(_._1)
          //4.2 聚合  {hadoop->2, sqoop->1}
        var wordSum=wordGroubList.map{
          keyValue =>
            keyValue._1 -> keyValue._2.map(_._2).sum
        }
          println(wordSum)
      }
    }
  }
}

```

## 步骤5 | 封装单词计数结果回复给MainActor

**实现思路**

- 将单词计数的结果封装为一个样例类消息，并发送给MainActor
- MainActor等待所有WordCount均已返回后获取到每个WordCountActor单词计算后的结果



**实现步骤**

1. 定义一个样例类封装单词计数结果
2. 将单词计数结果发送给MainActor
3. MainActor中检测所有WordActor是否均已返回，如果均已返回，则获取并转换结果
4. 打印结果

**参考代码**

MessagePackage.scala

```scala
/**
  * 单词统计结果
  * @param wordCount 单词计数
  */
//封装单词统计结果
case class WordCountResult(wordSum:Map[String,Int])

```

WordCountActor.scala

```scala
 //6将结果封装到样例类中,发送给WcMain
 sender ! WordCountResult(wordSum)

```

MainActor.scala

```scala
 // 编写一个while循环来等待所有的Actor都已经返回数据
      while (futureList.filter(!_.isSet).size!=0){}
    // 获取Future中封装的数据
      val wordCountResultList = futureList.map(_.apply().asInstanceOf[WordCountResult])
    // 获取样例类中封装的单词统计结果
    val stringToInts = wordCountResultList.map(_.wordSum)
    println(stringToInts)

```

## 步骤6 | 结果合并

**实现思路**

对接收到的所有单词计数进行合并。因为该部分已经在WordCountActor已经编写过，所以抽取这部分一样的代码到一个工具类中，再调用合并得到最终结果

**实现步骤**

1. 创建一个用于单词合并的工具类
2. 抽取重复代码为一个方法
3. 在MainActor调用该合并方法，计算得到最终结果，并打印

**参考代码**

WordCountUtil.scala

```scala
  /**
    * 单词分组统计
    * @param wordCountList 单词计数列表
    * @return 分组聚合结果
    */
  def reduce(wordCountList:List[(String, Int)]) = {
    // 按照单词进行分组
    // [单词分组] = {hadoop->List(hadoop->1, hadoop->1, hadoop->1), spark->List(spark ->1)}
    val grouped: Map[String, List[(String, Int)]] = wordCountList.groupBy(_._1)
    // 将分组内的数据进行聚合
    // [单词计数] = (hadoop, 3), (spark, 1)
    val wordCount: Map[String, Int] = grouped.map {
      tuple =>
        // 单词
        val word = tuple._1
        // 进行计数
        // 获取到所有的单词数量，然后进行累加
        val total = tuple._2.map(_._2).sum
        word -> total
    }
    wordCount
  }

```

MainActor.scala

```scala
// 扁平化后再聚合计算
val result: Map[String, Int] = WordCountUtil.reduce(resultList.flatten)

println("最终结果:" + result)
```