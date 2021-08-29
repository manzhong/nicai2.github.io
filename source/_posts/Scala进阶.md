---
title: Scala进阶
abbrlink: 34872
date: 2017-08-11 10:42:15
tags: Scala
categories: Scala
summary_img:
encrypt:
enc_pwd:
---

# Scala 进阶

# 一 类与对象

scala是支持面向对象的，也有类和对象的概念。

创建类和对象

- 使用`class`关键字来定义类
- 使用`var`/`val`来定义成员变量
- 使用`def`来定义成员方法
- 使用`new`来创建一个实例对象

1. var name:String = \_，\_表示使用默认值进行初始化

   例如：String类型默认值是null，Int类型默认值是0，Boolean类型默认值是false...

2. val变量不能使用_来进行初始化，因为val是不可变的，所以必须手动指定一个默认值

3. main方法必须要放在一个scala的`object`（单例对象）中才能执行

   ~~~~scala
   object Demo4 {
     class Pe{
       var name:String=_
       var age:Int =_
       def add(m:String) = print(m)
       private def a(a:String )= print(a)  //private不可被访问
     }

     def main(args: Array[String]): Unit = {
       val pe = new Pe
       pe.add("mm")
     }
   }
   ~~~~


   geter和setter方法:

1. scala会自动为成员变量生成scala语言的getter/setter
2. scala的getter为`字段名()`，setter为`字段名_=()`
3. 要生成Java的getter/setter，可以在成员变量上加一个`@BeanProperty`注解，这样将来去调用一些Java库的时候很有

~~~scala
  @BeanProperty
  var name:String = _             // 姓名

  @BeanProperty
  val registerDate = new Date()   // 注册时间
~~~







# 二构造器

## 1 主构造器

~~~scala
object Demo5 {
  class Per(var name:String ="",var sex:String= ""){
    print("构造")
  }

  def main(args: Array[String]): Unit = {
    val ni = new Per("ni","男")
    println(ni.name)
    println(ni.sex)

    val per = new Per()
    println(ni.sex)
    println(ni.name)

    println(new Per(sex = "女").sex)

  }
}
~~~



## 2 辅助构造器

与定义方法一样,且方法名一定为this

注意:

辅助构造器第一行必须调用主构造器或者其他辅助构造器

~~~scala
object Demo6 {

  class Cus(var name: String = "", var add: String = "") {
    //辅助构造器
    def this(fu: Array[String]) {
      this(fu(0), fu(1))
    }
    def this(name:String){
      this(name,"郑州")
    }
   /* def this (add:String){
      this("你猜",add)
    }*/
  }

  def main(args: Array[String]): Unit = {
    val cus = new Cus(Array[String]("aa","北京"))
    println(cus.add)
    val cus2 = new Cus("niu")
    println(cus2.add)//郑州
    print(cus2.name)//niu
  }

}

~~~

1. 主构造器直接在类名后面定义
2. 主构造器中的参数列表会自动定义为私有的成员变量
3. 一般在主构造器中直接使用val/var定义成员变量，这样看起来会更简洁
4. 在辅助构造器中必须调用其他构造器（主构造器、其他辅助构造器）

## 三 单例对象 (类似于java中的static)

scala要比Java更加面向对象，所以，scala中是没有Java中的静态成员的。如果将来我们需要用到static变量、static方法，就要用到scala中的单例对象——object。可以把object理解为全是包含静态字段、静态方法的class，object本质上也是一个class。

**定义object**

定义单例对象和定义类很像，就是把class换成object

~~~scala
定义一个工具类，用来格式化日期时间
object DateUtils {

  // 在object中定义的成员变量，相当于Java中定义一个静态变量
  // 定义一个SimpleDateFormat日期时间格式化对象
  val simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm")

  // 构造代码
  println("构造代码")
    
  // 相当于Java中定义一个静态方法
  def format(date:Date) = simpleDateFormat.format(date)

  // main是一个静态方法，所以必须要写在object中
  def main(args: Array[String]): Unit = {
    println { DateUtils.format(new Date()) };
  }
}
~~~

> 1. 使用`object 单例对象名`定义一个单例对象，可以用object作为工具类或者存放常量
> 2. 在单例对象中定义的变量，类似于Java中的static成员变量
> 3. 在单例对象中定义的方法，类似于Java中的static方法
> 4. object单例对象的构造代码可以直接写在花括号中
> 5. 调用单例对象的方法，直接使用`单例对象名.方法名`，访问单例对象的成员变量也是使用`单例对象名.变量名`
> 6. 单例对象只能有一个`无参的主构造器`，不能添加其他参数

~~~scala
object Demo7 {
  object Dog{
    val num=4;
  }

  def main(args: Array[String]): Unit = {
    print(Dog.num)
  }
}
~~~

~~~scala
//调用方法
object Demo8 {
  object Uf{
    def add(): Unit ={
      print("-" * 15)   //生成15 个-
      print("nicai")
    }
  }

  def main(args: Array[String]): Unit = {
    Uf.add()
  }
}

~~~

**伴生对象**

在Java中，经常会有一些类，同时有实例成员又有静态成员。如下：

~~~java
public class CustomerService {

    private static Logger logger = Logger.getLogger("customerService");

    public void save(String customer) {
        logger.info("添加一个用户");
        // 保存客户
    }

    public static void main(String[] args) {
        new CustomerService().save("客户");
    }
}
~~~

在scala中，可以使用`伴生对象`来实现。

**一个class和object具有同样的名字。这个object称为`伴生对象`，这个class称为`伴生类`**

注意:

半生类和伴生对象一样的名字

这两个要在同一个scala源文件中

这两个可以互相访问private属性

~~~scala
object Demo11 {
  //半生类
  class Ssm(){
    def fin(): Unit ={
        print(Ssm.name)
    }
  }
  //半生对象
  object Ssm{
    var name="nicai"
  }

  def main(args: Array[String]): Unit = {
    val ssm = new Ssm
    ssm.fin()
  }
}
~~~

#### private[this] 访问权限  表示只能在当前类中访问,伴生对象也不可访问

~~~scala
object Demo12 {
  class De(private[this] var name:String,var age:Int)

  object De{
    def p(d:De): Unit =print(d.name)   //直接报错
  }

  def main(args: Array[String]): Unit = {
    De.p(new De("aa",788))
  }
}

~~~

工具类案例:

需求:

编写一个工具类专门格式化日期时间

定义一个方法用于将日期转换为年月日字符串  2012-10-15

~~~scala
object Demo9 {
  object Forma{
    //java 中的类
    private val format = new SimpleDateFormat("yyyy-MM-dd")

    //定义方法
    def fm(data:Date)=format.format(data)
  }

  def main(args: Array[String]): Unit = {
    println(Forma.fm(new Date()))
  }
}

~~~

**apply方法**

**必须用在伴生对象中**

我们之前使用过这种方式来创建一个Array对象。

~~~scala 
var a=Array(1,2)
~~~

这种写法非常简便，不需要再写一个new，然后敲一个空格，再写类名。如何直接使用类名来创建对象呢？

答案就是：实现伴生对象的apply方法

伴生对象的apply方法用来快速地创建一个伴生类的对象。

~~~scala 
object Demo13 {
  class Person(var name:String, var age:Int) {

    override def toString = s"Person($name, $age)"
  }

  object Person {
    // 实现apply方法
    // 返回的是伴生类的对象
    def apply(name:String, age:Int): Person = new Person(name, age)

    // apply方法支持重载
    def apply(name:String):Person = new Person(name, 20)

    def apply(age:Int):Person = new Person("某某某", age)

    def apply():Person = new Person("某某某", 20)
  }


  def main(args: Array[String]): Unit = {
    println(Person("jjj").name)
  }
}

~~~

1. 当遇到`类名(参数1, 参数2...)`会自动调用apply方法，在apply方法中来创建对象
2. 定义apply时，如果参数列表是空，也不能省略括号()，否则引用的是伴生对象

**main方法**

scala和Java一样，如果要运行一个程序，必须有一个main方法。而在Java中main方法是静态的，而在scala中没有静态方法。在scala中，这个main方法必须放在一个object中。

实例:

~~~
object Main5 {
  def main(args:Array[String]) = {
    println("hello, scala")
  }
}
~~~

也可以继承自App Trait（特质），然后将需要编写在main方法中的代码，写在object的构造方法体内。

~~~scala 
object 	Main extends App{
println("heoo")
}
~~~

## 四继承

scala和Java一样，使用**extends**关键字来实现继承。可以在子类中定义父类中没有的字段和方法，或者重写父类的方法。

类和单例对象都可以从某个父类继承

~~~scala
object Demo14 {
  class Per(){
    var name=""
    def getName()=this.name
  }
  class Student extends Per

  def main(args: Array[String]): Unit = {
    val student = new Student
    student.name="ni"
    println(student.name)
  }
}

~~~

单例继承

~~~scala
object Demo15{
  class Per(){
    var name=""
    def a()=print("nicai")
  }
  object Stu extends Per

  def main(args: Array[String]): Unit = {
    Stu.name="ni"
    println(Stu.name)
  }
}

~~~

### override 和supper

------

- 如果子类要覆盖父类中的一个非抽象方法，必须要使用override关键字
- 可以使用override关键字来重写一个val字段
- 可以使用super关键字来访问父类的成员

~~~scala
object Demo16 {
  class Per(){
    val name:String ="22"
    var age:Int=0
    def getName()=this.name
  }
  class Stu extends Per{
    override val name: String = "nn"

    override def getName(): String = "ni"+super.getName()
  }

  def main(args: Array[String]): Unit = {
    val stu = new Stu
    println(stu.getName())  //ninn
    println(stu.name)    //nn
  }
}

~~~

### 类型的判断与转换

我们经常要在代码中进行类型的判断和类型的转换。在Java中，我们可以使用instanceof关键字、以及(类型)object来实现，在scala中如何实现呢？

scala中对象提供isInstanceOf和asInstanceOf方法。

- isInstanceOf判断对象是否为指定类的对象
- asInstanceOf将对象转换为指定类型

~~~scala
object Demo17 {
  class per
  class Stu extends per

  def main(args: Array[String]): Unit = {
    val stu = new Stu
    if(stu.isInstanceOf[Stu]){
      stu.asInstanceOf[Stu]
      print(stu)
    }else{
      print("不是stu类型")
    }
  }
}

~~~

### getClass和classOf

isInstanceOf 只能判断出对象是否为指定类以及其子类的对象，而不能精确的判断出，对象就是指定类的对象。如果要求精确地判断出对象就是指定类的对象，那么就只能使用 getClass 和 classOf 。

- p.getClass可以精确获取对象的类型
- classOf[x]可以精确获取类型
- 使用==操作符就可以直接比较

~~~scala
object Demo18 {
  class Per()
  class Stu extends Per

  def main(args: Array[String]): Unit = {
    val stu:Per=new Stu

    if(stu.isInstanceOf[Per]){
      println("stu是一个per类型")      ///y
    }else{
      println("stu 不是一个per类型")
    }

    if(stu.getClass == classOf[Per]){
      println("stu是一个per类型")
    }else{
      println("stu 不是一个per类型")     ////y
    }

    if(stu.getClass == classOf[Stu]){
      println("stu是一个STU类型")       //y
    }else{
      println("stu 不是一个stu类型")
    }

  }
}

~~~

**调用父类的constructor**

实例化子类对象，必须要调用父类的构造器，在scala中，只能在子类的`主构造器`中调用父类的构造器

步骤：

1. 创建一个Person类，编写带有一个可变的name字段的主构造器
2. 创建一个Student类，继承自Person类
   - 编写带有一个name参数、clazz班级字段的主构造器
   - 调用父类的构造器
3. 创建main方法，创建Student对象实例，并打印它的姓名、班级

~~~scala
     class Person5(var name:String)
    // 直接在父类的类名后面调用父类构造器
    class Student5(name:String, var clazz:String) extends Person5(name)

    object Main5 {
      def main(args: Array[String]): Unit = {
        val s1 = new Student5("张三", "三年二班")
        println(s"${s1.name} - ${s1.clazz}")
      }
    }
~~~

### 抽象类

如果类的某个成员在当前类中的定义是不包含完整的，它就是一个**抽象类**

不完整定义有两种情况：

1. 方法没有方法体
2. 变量没有初始化

没有方法体的方法称为**抽象方法**，没有初始化的变量称为**抽象字段**。定义抽象类和Java一样，在类前面加上**abstract**关键字就可以了

~~~scala
//抽象方法
object Demo19 {
   abstract class Sop(){
      def m():Double    //或者 def  m:Double
    }
  class z(l:Double ) extends Sop{
    override def m:Double  = l*l
  }
  class Cyc(r:Double) extends Sop{
    override def m:Double  =  Math.PI*r*r
  }
  class Ju(l:Double,k:Double) extends Sop{
    override def m: Double = l*k
  }

  def main(args: Array[String]): Unit = {
    val z = new z(12.3)
    val cyc = new Cyc(2)
    val ju = new Ju(1,3.0)
    println(z.m)
    println(cyc.m)
    println(ju.m)
  }
}
~~~

抽象字段:

~~~scala
// 定义一个人的抽象类
abstract class Person6 {
  // 没有初始化的val字段就是抽象字段
  val WHO_AM_I:String
}

class Student6 extends Person6 {
  override val WHO_AM_I: String = "学生"
}

class Policeman6 extends Person6 {
  override val WHO_AM_I: String = "警察"
}

object Main6 {
  def main(args: Array[String]): Unit = {
    val p1 = new Student6
    val p2 = new Policeman6

    println(p1.WHO_AM_I)
    println(p2.WHO_AM_I)
  }
}
~~~

## 匿名内部类

匿名内部类是没有名称的子类，直接用来创建实例对象。Spark的源代码中有大量使用到匿名内部类。

~~~~scala
object Demo20 {
  abstract class Per(){
    def say()
  }

  def main(args: Array[String]): Unit = {
    val per = new Per {
      override def say(): Unit = println("nicai")
    }
    per.say()
  }
}
~~~~

## 特质(trait)

scala中没有接口的概念 替代的就是特质

- 特质是scala中代码复用的基础单元
- 它可以将方法和字段定义封装起来，然后添加到类中
- 与类继承不一样的是，类继承要求每个类都只能继承`一个`超类，而一个类可以添加`任意数量`的特质。
- 特质的定义和抽象类的定义很像，但它是使用`trait`关键字

### 用法一  作为接口使用

- 使用`extends`来继承trait（scala不论是类还是特质，都是使用extends关键字）
- 如果要继承多个trait，则使用`with`关键字

**继承单个特质**

~~~scala
object Demo21 {
  trait Loger{
    def pr(msg:String)
  }
  class LogerE extends Loger{
    override def pr(msg: String): Unit = print(msg)
  }

  def main(args: Array[String]): Unit = {
    var e:Loger=new LogerE()
    e.pr("nicai")
  }
}
~~~

**继承多个特质**

~~~scala
object Demo22 {
  trait D1{
    def ms(msg:String)
  }
  trait D2{
    def ms():String
  }
  class D3 extends D1 with D2{
    override def ms(msg: String): Unit = println(msg)

    override def ms(): String = "nicai"
  }

  def main(args: Array[String]): Unit = {
    val d = new D3
    d.ms("ni")
    println(d.ms())
  }
}

~~~

**object 继承trait**

~~~scala
object Demo23 {
  trait D1{
    def m(msg:String)
  }
  object D2 extends D1{
    override def m(msg: String): Unit = print(msg)
  }

  def main(args: Array[String]): Unit = {
    D2.m("nicai")
  }
}
~~~

1. 在trait中可以定义抽象方法，不写方法体就是抽象方法
2. 和继承类一样，使用extends来继承trait
3. 继承多个trait，使用with关键字
4. 单例对象也可以继承trait

### 特质 定义具体的方法

和类一样，trait中还可以定义具体的方法。·

~~~scala
object Demo24 {
  trait D{
    def add(msg:String) = println(msg)
  }
   class  D2 extends D{
     def add2()= add("nicai")
   }

  def main(args: Array[String]): Unit = {
    val d = new D2
    d.add2()
  }
}
~~~

### trait中定义具体的字段和抽象字段

- 在trait中，可以混合使用具体方法和抽象方法
- 使用具体方法依赖于抽象方法，而抽象方法可以放到继承trait的子类中实现，这种设计方式也称为**模板模式**

~~~scala
object Demo25 {
  trait D{
  val s = new SimpleDateFormat("yyyy-MM-dd")
    var TYPE:String
    def pr(msg:String)
  }
  class D2 extends D{
    override var TYPE: String = "控制台消息"

    override def pr(msg: String): Unit = println(s"${TYPE}:${s.format(new Date)}:${msg}")
  }

  def main(args: Array[String]): Unit = {
    val d = new D2
    d.pr("nnicai")
  }
}

~~~

### 使用trait实现模板模式

在特质中,具体方法依赖于抽象方法,而抽象 方法可以放在继承trait中的子类中实现,这种方式为模板设计模式

~~~scala
object Demo26 {
    trait  Logger{
      def  log(msg:String)
      def info(msg:String) = log(msg)
      def exce(msg:String) = log(msg)
      def erro(msg:String) = log(msg)
    }
  class LogE extends Logger{
    override def log(msg: String): Unit = println(msg)
  }

  def main(args: Array[String]): Unit = {
    val e = new LogE
    e.info("info")
    e.exce("exec")
    e.erro("erro")
  }
}

~~~

### 对象混入trait

- trait还可以混入到`实例对象`中，给对象实例添加额外的行为
- 只有混入了trait的对象才具有trait中的方法，其他的类对象不具有trait中的行为
- 使用with将trait混入到实例对象中

语法:

~~~scala
var /val da=new 类 with 特质
~~~

~~~scala
object Demo27 {
  trait Logger{
    def log()=println("nicai")
  }
  class Aa

  def main(args: Array[String]): Unit = {
    val aa = new Aa with Logger
    aa.log()
  }
}
~~~

### trait 实现调用链模式

类继承了多个trait后，可以依次调用多个trait中的同一个方法，只要让多个trait中的同一个方法在最后都依次执行super关键字即可。类中调用多个tait中都有这个方法时，首先会从最右边的trait方法开始执行，然后依次往左执行，形成一个调用链条。

如支付连  等

说明  :  一个子类 继承多个父trait  且还有祖父trait  则 会先执行自己的 在执行从右到左的父trait(继承顺序相反),最后执行祖父trait

~~~scala
object Demo28 {
    trait Zf{
      def log(data:String ) = println("祖父")
    }
  trait Login extends Zf{
    override def log(data: String): Unit ={
      println("父1")
      super.log(data)
    }
  }
  trait Handle extends Zf{
    override def log(data: String): Unit = {
      println("父2")
      super.log(data)
    }
  }

  class Ser() extends Handle with Login {
    override def log(data: String): Unit = {
     println("子类")
      super.log(data)
    }

  }

  def main(args: Array[String]): Unit = {
    val ser = new Ser()
    ser.log("nn")
  }
}
//子类
//父1
//父2
//祖父
~~~

### trait的构造机制

- trait也有构造代码，但和类不一样，特质不能有构造器参数
- 每个特质只有**`一个无参数`**的构造器。
- 一个类继承另一个类、以及多个trait，当创建该类的实例时，它的构造顺序如下：
  1. 执行父类的构造器
  2. `从左到右`依次执行trait的构造器
  3. 如果trait有父trait，先构造父trait，如果多个trait有同样的父trait，则只初始化一次
  4. 执行子类构造器

~~~scala
class Person_One {
  println("执行Person构造器!")
}
trait Logger_One {
  println("执行Logger构造器!")
}
trait MyLogger_One extends Logger_One {
  println("执行MyLogger构造器!")
}
trait TimeLogger_One extends Logger_One {
  println("执行TimeLogger构造器!")
}
class Student_One extends Person_One with MyLogger_One with TimeLogger_One {
  println("执行Student构造器!")
  }
object exe_one {
  def main(args: Array[String]): Unit = {
    val student = new Student_One
  }
}

// 程序运行输出如下：
// 执行Person构造器!
// 执行Logger构造器!
// 执行MyLogger构造器!
// 执行TimeLogger构造器!
// 执行Student构造器!
~~~

**trait继承class**

- trait也可以继承class  会把class中的成员都继承下来
- 这个class就会成为所有该trait子类的超级父类。

~~~scala
class MyUtil {
  def printMsg(msg: String) = println(msg)
}
trait Logger_Two extends MyUtil {
  def log(msg: String) = this.printMsg("log: " + msg)
}
class Person_Three(val name: String) extends Logger_Two {
    def sayHello {
        this.log("Hi, I'm " + this.name)
        this.printMsg("Hello, I'm " + this.name)
  }
}
object Person_Three{
  def main(args: Array[String]) {
      val p=new Person_Three("Tom")
      p.sayHello
    //执行结果：
//      log: Hi, I'm Tom
//      Hello, I'm Tom
  }
}
~~~

