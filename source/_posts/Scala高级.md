---
title: Scala高级
abbrlink: 29104
date: 2017-08-13 21:37:29
tags: Scala
categories: Scala
summary_img:
encrypt:
enc_pwd:
---

# Scala高级

## 一 高阶函数

scala 混合了面向对象和函数式的特性，在函数式编程语言中，函数是“头等公民”，它和Int、String、Class等其他类型处于同等的地位，可以像其他类型的变量一样被传递和操作。

高阶函数包含

- 作为值的函数
- 匿名函数
- 闭包
- 柯里化等等

### 1作为值得函数

在scala中，函数就像和数字、字符串一样，可以将函数传递给一个方法。我们可以对算法进行封装，然后将具体的动作传递给方法，这种特性很有用。

我们之前学习过List的map方法，它就可以接收一个函数，完成List的转换。

**例子**

**示例说明**

将一个整数列表中的每个元素转换为对应个数的小星星

```html
List(1, 2, 3...) => *, **, ***
```

**步骤**

1. 创建一个函数，用于将数字转换为指定个数的小星星
2. 创建一个列表，调用map方法
3. 打印转换为的列表

**参考代码**

```scala
package com.nicai.highlevel

object Demo01 {
  def main(args: Array[String]): Unit = {
    val fun: Int => String = (num:Int) => "*" * num
    val strings = (1 to 10).map(fun)
    println(strings)
  }

}

```

  ### 2 	匿名函数

**定义**

上面的代码，给(num:Int) => "*" * num函数赋值给了一个变量，但是这种写法有一些啰嗦。在scala中，可以不需要给函数赋值给变量，没有赋值给变量的函数就是*匿名函数

```scala
val list = List(1, 2, 3, 4)

// 字符串*方法，表示生成指定数量的字符串
val func_num2star = (num:Int) => "*" * num

print(list.map(func_num2star))
```

使用匿名函数优化上述代码

**参考代码**

```scala

object Demo02 {
  def main(args: Array[String]): Unit = {
    val strings = (1 to 10).map(x => "*" * x)

    println(strings)
// 因为此处num变量只使用了一次，而且只是进行简单的计算，所以可以省略参数列表，使用_替代参数
    val strings2 = (1 to 10 ).map("*" * _)
    println(strings2)
  }
}

```

### 3 柯里化

在scala和spark的源代码中，大量使用到了柯里化。为了后续方便阅读源代码，我们需要来了解下柯里化。  

定义:

柯里化（Currying）是指将原先接受多个参数的方法转换为多个只有一个参数的参数列表的过程。

![img](/images/scala/klh.png)

**柯里化过程解析**

![img](/images/scala/klhgcjx.png)	

**例子**

**示例说明**

- 编写一个方法，用来完成两个Int类型数字的计算
- 具体如何计算封装到函数中
- 使用柯里化来实现上述操作

**参考代码**

```scala
// 柯里化：实现对两个数进行计算
package com.nicai.highlevel

object Demo3 {
  def add(a:Int,b:Int)(cala:(Int,Int) => Int)={
    cala(a,b)
  }

  def main(args: Array[String]): Unit = {
    println(add(1, 2)(_ + _))
    println(add(1, 2)(_ * _))

  }
}

```

### 闭包

闭包其实就是一个函数，只不过这个函数的返回值依赖于声明在函数外部的变量。

可以简单认为，就是可以访问不在当前作用域范围的一个函数。

**例子一**

定义一个闭包

```scala
package com.nicai.highlevel
//闭包
object Demo4 {
  var x=4
  val add=(y:Int) => x+y

  def main(args: Array[String]): Unit = {
    println(add(5))
  }
}

```

add函数就是一个闭包

**例子二**

柯里化就是一个闭包

```scala
  def add(x:Int)(y:Int) = {
    x + y
  }
```

上述代码相当于

```scala
  def add(x:Int) = {
    (y:Int) => x + y
  }

总的:
package com.nicai.highlevel
//闭包二  柯里化就是一个闭包
object Demo5 {
  def add(a:Int)(b:Int): Int ={
    a+b
  }
  def add2(x:Int)={
    (y:Int) => x+y
  }

  def main(args: Array[String]): Unit = {
    println(add(5)(6))  //11
    println(add2(5)(6))//11
  }
}

```

## 二 隐式转换与隐式参数

隐式转换和隐式参数是scala非常有特色的功能，也是Java等其他编程语言没有的功能。我们可以很方便地利用隐式转换来丰富现有类的功能。后面在编写Akka并发编程、Spark SQL、Flink都会看到隐式转换和隐式参数的身影。

### 1 使用隐式转换

**定义:**

所谓**隐式转换**，是指以implicit关键字声明的带有**单个参数**的方法。它是**自动被调用**的，自动将某种类型转换为另外一种类型。

**使用步骤**

1. 在object中定义隐式转换方法（使用implicit）
2. 在需要用到隐式转换的地方，引入隐式转换（使用import）
3. 自动调用隐式转化后的方法

**例子**

**示例说明**

使用隐式转换，让File具备有read功能——实现将文本中的内容以字符串形式读取出来

**步骤**

1. ```
   创建RichFile类，提供一个read方法，用于将文件内容读取为字符串
   ```

2. 定义一个隐式转换方法，将File隐式转换为RichFile对象

3. 创建一个File，导入隐式转换，调用File的read方法

**参考代码**

```scala
package com.nicai.yinshizhuanhuan

import java.io.File

import scala.io.Source
//隐式转换
/*1. 创建RichFile类，提供一个read方法，用于将文件内容读取为字符串
2. 定义一个隐式转换方法，将File隐式转换为RichFile对象
3. 创建一个File，导入隐式转换，调用File的read方法*/

object Demo6 {
    class RichFile(f:File){
      def read()={
        Source.fromFile(f).mkString
      }
    }

  object Im{
    implicit def fileToRichFile(file:File) =new RichFile(file)
  }

  def main(args: Array[String]): Unit = {
  	val file = new File("day23Scala4/data/a.txt")
    import Im.fileToRichFile
    println(file.read())
  }
}

```

  **隐式转换的时机**

- 当对象调用类中不存在的方法或者成员时，编译器会自动将对象进行隐式转换
- 当方法中的参数的类型与目标类型不一致时

### 2 自动导入隐式转化方法

前面，我们手动使用了import来导入隐式转换。是否可以不手动import呢？

在scala中，如果在当前作用域中有隐式转换方法，会自动导入隐式转换。

示例：将隐式转换方法定义在main所在的object中

```scala
package com.nicai.yinshizhuanhuan

import java.io.File

import scala.io.Source

//自动导入隐式转换
object Demo7 {
  class RichFile(f:File){
    def read()={
      Source.fromFile(f).mkString
    }
  }

  def main(args: Array[String]): Unit = {
    val file = new File("day23Scala4/data/a.txt")
    implicit def fileToRichFile(file:File) =new RichFile(file)
    println(file.read())
  }
}

```

### 3 隐式参数

方法可以带有一个标记为implicit的参数列表。这种情况，编译器会查找缺省值，提供给该方法。

**定义**

1. 在方法后面添加一个参数列表，参数使用implicit修饰
2. 在object中定义implicit修饰的隐式值
3. 调用方法，可以不传入implicit修饰的参数列表，编译器会自动查找缺省值


  注意:

1. 和隐式转换一样，可以使用import手动导入隐式参数
2. 如果在当前作用域定义了隐式值，会自动进行导入

**例子**

**示例说明**

- 定义一个方法，可将传入的值，使用一个分隔符前缀、后缀包括起来
- 使用隐式参数定义分隔符
- 调用该方法，并打印测试

**参考代码**

```scala
package com.nicai.yinshizhuanhuan
//隐式参数
/*- 定义一个方法，可将传入的值，使用一个分隔符前缀、后缀包括起来
- 使用隐式参数定义分隔符
- 调用该方法，并打印测试*/
//与java中的动态代理  作用有点类似
object Demo8 {
    def qu(str:String)(implicit im:(String,String)) ={
      im._1+str+im._2
    }

  //定义隐式参数
  object Im{
    implicit val delim= ("<<",">>")
  }

  def main(args: Array[String]): Unit = {
    import Im.delim
    println(qu("aa"))
  }
}

```

## 三 Akka 并发编程

### 1 介绍

Akka是一个用于构建高并发、分布式和可扩展的基于事件驱动的应用的工具包。Akka是使用scala开发的库，同时可以使用scala和Java语言来开发基于Akka的应用程序。


 ### 2 特性

- 提供基于异步非阻塞、高性能的事件驱动编程模型
- 内置容错机制，允许Actor在出错时进行恢复或者重置操作
- 超级轻量级的事件处理（每GB堆内存几百万Actor）
- 使用Akka可以在单机上构建高并发程序，也可以在网络中构建分布式程序。

### 3 Akka通信过程

以下图片说明了Akka Actor的并发编程模型的基本流程：

1. 学生创建一个ActorSystem
2. 通过ActorSystem来创建一个ActorRef（老师的引用），并将消息发送给ActorRef
3. ActorRef将消息发送给Message Dispatcher（消息分发器）
4. Message Dispatcher将消息按照顺序保存到目标Actor的MailBox中
5. Message Dispatcher将MailBox放到一个线程中
6. MailBox按照顺序取出消息，最终将它递给TeacherActor接受的方法中

  ![img](/images/scala/akka.png)

### 4 入门案例

基于Akka创建两个Actor，Actor之间可以互相发送消息。

![img](/images/scala/akkabf.png)

## 实现步骤

1. 创建Maven模块
2. 创建并加载Actor
3. 发送/接收消息

**1创建Maven模块**

使用Akka需要导入Akka库，我们这里使用Maven来管理项目

1. 创建Maven模块
2. 打开pom.xml文件，导入akka Maven依赖和插件

~~~xml
 <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <encoding>UTF-8</encoding>
        <scala.version>2.11.8</scala.version>
        <scala.compat.version>2.11</scala.compat.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>${scala.version}</version>
        </dependency>

        <dependency>
            <groupId>com.typesafe.akka</groupId>
            <artifactId>akka-actor_2.11</artifactId>
            <version>2.3.14</version>
        </dependency>

        <dependency>
            <groupId>com.typesafe.akka</groupId>
            <artifactId>akka-remote_2.11</artifactId>
            <version>2.3.14</version>
        </dependency>

    </dependencies>

    <build>
        <sourceDirectory>src/main/scala</sourceDirectory>
        <testSourceDirectory>src/test/scala</testSourceDirectory>
        <plugins>
            <plugin>
                <groupId>net.alchim31.maven</groupId>
                <artifactId>scala-maven-plugin</artifactId>
                <version>3.2.2</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>testCompile</goal>
                        </goals>
                        <configuration>
                            <args>
                                <arg>-dependencyfile</arg>
                                <arg>${project.build.directory}/.scala_dependencies</arg>
                            </args>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>2.4.3</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                            <transformers>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                    <resource>reference.conf</resource>
                                </transformer>
                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                    <mainClass></mainClass>
                                </transformer>
                            </transformers>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
~~~

**2创建并加载Actor**

创建两个Actor

- SenderActor：用来发送消息

  ~~~scala
  package com.nicai.akkademo

  import akka.actor.Actor
  //发送消息
  object SenderActor extends Actor{
    //不在使用loop+react了 在akka中直接在receive中编写偏函数直接处理消息就可以持续接受消息
    override def receive: Receive = {
      case x => println(x)
    }
  }

  ~~~


- ReceiveActor：用来接收，回复消息

  ~~~~scala
  package com.nicai.akkademo

  import akka.actor.Actor
  //接收  回复消息
  object ReceiveActor extends Actor{
    override def receive: Receive = {
      case x => println(x)
    }
  }

  ~~~~


**创建Actor**

1. 创建ActorSystem

2. 创建自定义Actor

3. ActorSystem加载Actor

   ~~~scala
   package com.nicai.akkademo

   import akka.actor.{ActorSystem, Props}
   import com.typesafe.config.ConfigFactory

   object Entrance {
     def main(args: Array[String]): Unit = {
       //创建ActorSystem
       val actorSystem = ActorSystem("actorSystem", ConfigFactory.load())
       //加载Actor
       val senderActor = actorSystem.actorOf(Props(SenderActor),"senderActor")
       val receiveActor = actorSystem.actorOf(Props(ReceiveActor),"receiveActor")
     }
   }

   ~~~


**3发送/接收消息**

- 使用样例类封装消息
- SubmitTaskMessage——提交任务消息
- SuccessSubmitTaskMessage——任务提交成功消息
- 使用类似于之前学习的Actor方式，使用`!`发送异步消息

**参考代码**

```scala
完整版
//sender发送消息
case class MessageSub(msg:String)
//receive回复消息
case class MsgSuccess(msg:String)
............................................................................
import akka.actor.Actor
//发送消息
object SenderActor extends Actor{
  //不在使用loop+react了 在akka中直接在receive中编写偏函数直接处理消息就可以持续接受消息
  override def receive: Receive = {
    //匹配 entrance 的消息
    case MessageSub("start") => {
      println("收到消息")
        //格式akka://actorSystem的名字/user(固定)/receiveActor的名字  若是远程连接则加端口号等
      val receiveActor = context.actorSelection("akka://actorSystem/user/receiveActor")
      //向receive发送消息
      receiveActor ! MessageSub("nicai")
    }
      //接收receive的消息
    case MsgSuccess(name) =>{
      println(name)
    }
  }
}
...............................................................................
import akka.actor.Actor
//接收  回复消息
object ReceiveActor extends Actor{
  override def receive: Receive = {
      //匹配消息
    case MessageSub(name) => {
      println(name)
        //回复消息
      sender  ! MsgSuccess("我不猜")
    }
    case _ => println("未匹配的消息类型")
  }
}
............................................................................
import akka.actor.{ActorSystem, Props}
import com.typesafe.config.ConfigFactory
object Entrance {
  def main(args: Array[String]): Unit = {
    //创建ActorSystem
    val actorSystem = ActorSystem("actorSystem", ConfigFactory.load())
    //加载Actor
    val senderActor = actorSystem.actorOf(Props(SenderActor),"senderActor")
    val receiveActor = actorSystem.actorOf(Props(ReceiveActor),"receiveActor")

    //向senderActor发送消息
    senderActor ! MessageSub("start")
  }
}

```

程序输出：

```text
收到消息
nicai
我不猜
```

### 5 	Akka定时任务

如果我们想要使用Akka框架定时的执行一些任务，该如何处理呢？

使用方式:


  Akka中，提供一个**scheduler**对象来实现定时调度功能。使用ActorSystem.scheduler.schedule方法，可以启动一个定时任务。

schedule方法针对scala提供两种使用形式：

**第一种：发送消息**

```scala
def schedule(
    initialDelay: FiniteDuration,        // 延迟多久后启动定时任务
    interval: FiniteDuration,            // 每隔多久执行一次
    receiver: ActorRef,                    // 给哪个Actor发送消息
    message: Any)                        // 要发送的消息
(implicit executor: ExecutionContext)    // 隐式参数：需要手动导入
```



**第二种：自定义实现**

```scala
def schedule(
    initialDelay: FiniteDuration,            // 延迟多久后启动定时任务
    interval: FiniteDuration                // 每隔多久执行一次
)(f: ⇒ Unit)                                // 定期要执行的函数，可以将逻辑写在这里
(implicit executor: ExecutionContext)        // 隐式参数：需要手动导入
```

## 示例一

**示例说明**

- 定义一个Actor，每1秒发送一个消息给Actor，Actor收到后打印消息
- 使用发送消息方式实现

**参考代码**

```scala
import akka.actor.{Actor, ActorSystem, Props}
import com.typesafe.config.ConfigFactory

object Demo1 {
//创建actor
  object ReceiveActor extends Actor {
    override def receive: Receive = {
      case x => println(x)
    }
  }

  def main(args: Array[String]): Unit = {
    val actorSystem = ActorSystem("actorSystem",ConfigFactory.load())
    val receiveActor = actorSystem.actorOf(Props(ReceiveActor))

    //导入 隐式转换
    import  scala.concurrent.duration._
    //导入隐式参数
    import actorSystem.dispatcher

    actorSystem.scheduler.schedule(0 seconds, //延迟后多久启动定时任务
      1 seconds,   //每隔多久执行一次
      receiveActor,        //给那个actor发送消息
      "hello"    //消息正文
    )
  }
}

```

## 示例二

**示例说明**

- 定义一个Actor，每1秒发送一个消息给Actor，Actor收到后打印消息
- 使用自定义方式实现

**参考代码**

```scala

import akka.actor.{Actor, ActorSystem, Props}
import com.typesafe.config.ConfigFactory

object Demo2 {
    object ReceiceActor extends Actor{
      override def receive: Receive = {
        case x => println(x)
      }
    }

  def main(args: Array[String]): Unit = {
    val actorSystem = ActorSystem("actorSystem",ConfigFactory.load())
    val receiveActor = actorSystem.actorOf(Props(ReceiceActor))

    //导入隐式装换 不到无法使用 0 seconds
    import scala.concurrent.duration._
    //导入隐式参数
    import actorSystem.dispatcher

    actorSystem.scheduler.schedule(0 seconds,1 seconds)(
      receiveActor ! " iac"
    )
  }
}

```

 注意:

1. 需要导入隐式转换`import scala.concurrent.duration._`才能调用0 seconds方法
2. 需要导入隐式参数`import actorSystem.dispatcher`才能启动定时任务

### 6 实现两个进程间的通信  master实现

基于Akka实现在两个**进程**间发送、接收消息。Worker启动后去连接Master，并发送消息，Master接收到消息后，再回复Worker消息。 

![img](/images/scala/jctx.png)

## 1. Worker实现

**步骤**

1. 创建一个Maven模块，导入依赖和配置文件

2. 创建配置文件

   ~~~properties
   application.conf

   akka.actor.provider = "akka.remote.RemoteActorRefProvider"
   akka.remote.netty.tcp.hostname = "127.0.0.1"
   akka.remote.netty.tcp.port = "8888"
   ~~~


1. 创建启动WorkerActor

2. 发送"setup"消息给WorkerActor，WorkerActor接收打印消息

3. 启动测试

**参考代码**

Worker.scala

```scala
import akka.actor.{Actor, ActorSystem, Props}
import com.typesafe.config.ConfigFactory

object Worker {
  def main(args: Array[String]): Unit = {
    val actorSystem = ActorSystem("actorSystem",ConfigFactory.load())
    val worker = actorSystem.actorOf(Props(WorkerActor))
    //发送消息
    worker ! "nicai"
  }
}

```

WorkerActor.scala

```scala
import akka.actor.Actor

object WorkerActor extends Actor{
  override def receive: Receive = {
    case x => println(x)
  }
}

```

## 2. Master实现

**步骤**

1. 创建Maven模块，导入依赖和配置文件
2. 创建启动MasterActor
3. WorkerActor发送"connect"消息给MasterActor
4. MasterActor回复"success"消息给WorkerActor
5. WorkerActor接收并打印接收到的消息
6. 启动Master、Worker测试

**参考代码**

Master.scala

```scala
import akka.actor.{ActorSystem, Props}
import com.typesafe.config.ConfigFactory

object Master {
  def main(args: Array[String]): Unit = {
    val actorSystem = ActorSystem("actorSystem",ConfigFactory.load())
    val masterActor = actorSystem.actorOf(Props(MasterActor),"masterActor")
  }
}

```

MasterActor.scala

```scala
import akka.actor.Actor

object MasterActor  extends Actor{
  override def receive: Receive = {
    case "connect" => {
      println("worker连接成功")
      sender ! "success"
    }
  }
}

```

WorkerActor.scala

```scala
import akka.actor.Actor

object WorkerActor extends Actor{
  override def receive: Receive = {
    case "start" => {
      println("start")
        //设置连接
      val masterActor = context.actorSelection("akka.tcp://actorSystem@127.0.0.1:9999/user/masterActor")
      masterActor ! "connect"

    }
    case "success" => {
      println("连接master成功")
    }
  }
}

```

## 四 简易Spark通信框架案例

**案例介绍**

模拟Spark的Master与Worker通信

- 一个Master
  - 管理Worker
- 若干个Worker（Worker可以按需添加）
  - 注册
  - 发送心跳

![img](/images/scala/spark.png)

**实现思路**

1. 构建Master、Worker阶段
   - 构建Master ActorSystem、Actor
   - 构建Worker ActorSystem、Actor
2. Worker注册阶段
   - Worker进程向Master注册（将自己的ID、CPU核数、内存大小(M)发送给Master）
3. Worker定时发送心跳阶段
   - Worker定期向Master发送心跳消息
4. Master定时心跳检测阶段
   - Master定期检查Worker心跳，将一些超时的Worker移除，并对Worker按照内存进行倒序排序
5. 多个Worker测试阶段
   - 启动多个Worker，查看是否能够注册成功，并停止某个Worker查看是否能够正确移除

**工程搭建**

项目使用Maven搭建工程

1. 分别搭建几下几个项目

| 工程名               | 说明            |
| :---------------- | :------------ |
| spark-demo-common | 存放公共的消息、实体类   |
| spark-demo-master | Akka Master节点 |
| spark-demo-worker | Akka Worker节点 |

1. 导入依赖

   - master/worker添加common依赖,其余同上

   ~~~xml
     common依赖
     
      <properties>
             <maven.compiler.source>1.8</maven.compiler.source>
             <maven.compiler.target>1.8</maven.compiler.target>
             <encoding>UTF-8</encoding>
             <scala.version>2.11.8</scala.version>
             <scala.compat.version>2.11</scala.compat.version>
         </properties>
     
         <dependencies>
             <dependency>
                 <groupId>org.scala-lang</groupId>
                 <artifactId>scala-library</artifactId>
                 <version>${scala.version}</version>
             </dependency>
         </dependencies>
     
     <build>
             <sourceDirectory>src/main/scala</sourceDirectory>
             <testSourceDirectory>src/test/scala</testSourceDirectory>
             <plugins>
                 <plugin>
                     <groupId>net.alchim31.maven</groupId>
                     <artifactId>scala-maven-plugin</artifactId>
                     <version>3.2.2</version>
                     <executions>
                         <execution>
                             <goals>
                                 <goal>compile</goal>
                                 <goal>testCompile</goal>
                             </goals>
                             <configuration>
                                 <args>
                                     <arg>-dependencyfile</arg>
                                     <arg>${project.build.directory}/.scala_dependencies</arg>
                                 </args>
                             </configuration>
                         </execution>
                     </executions>
                 </plugin>
     
                 <plugin>
                     <groupId>org.apache.maven.plugins</groupId>
                     <artifactId>maven-shade-plugin</artifactId>
                     <version>2.4.3</version>
                     <executions>
                         <execution>
                             <phase>package</phase>
                             <goals>
                                 <goal>shade</goal>
                             </goals>
                             <configuration>
                                 <filters>
                                     <filter>
                                         <artifact>*:*</artifact>
                                         <excludes>
                                             <exclude>META-INF/*.SF</exclude>
                                             <exclude>META-INF/*.DSA</exclude>
                                             <exclude>META-INF/*.RSA</exclude>
                                         </excludes>
                                     </filter>
                                 </filters>
                                 <transformers>
                                     <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                         <resource>reference.conf</resource>
                                     </transformer>
                                     <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                         <mainClass></mainClass>
                                     </transformer>
                                 </transformers>
                             </configuration>
                         </execution>
                     </executions>
             </plugins>
         </build>
   ~~~

2. 导入配置文件(同上)

   - 修改Master的端口为7000(或者自定义)
   - 修改Worker的端口为7100(或者自定义 最好 8000以上)

**构建master和worker**

master和masterActor     worker和workerActor

同上:

~~~~scala 
//MasterActor
import akka.actor.Actor
object MasterActor extends Actor{
  override def receive: Receive = {
    case x => println(x)
  }
}
........................................
//MasterMain
import akka.actor.{ActorSystem, Props}
import com.typesafe.config.ConfigFactory

object MasterMain {
  def main(args: Array[String]): Unit = {
    val actorSystem = ActorSystem("actorSystem",ConfigFactory.load())
    val masterActor = actorSystem.actorOf(Props(MasterActor),"masterActor")
  }
}
.............................................................
//WorkerActor 
import akka.actor.Actor

object WorkerActor extends Actor{
  override def receive: Receive = {
    case x => println(x)
  }
}
..............................
//WorkerMain 
import akka.actor.{ActorSystem, Props}
import com.typesafe.config.ConfigFactory

object WorkerMain {
  def main(args: Array[String]): Unit = {
    val actorSystem = ActorSystem("actorSystem",ConfigFactory.load())
    val workerActor = actorSystem.actorOf(Props(WorkerActor),"workerActor")
  }
}

~~~~

**worker注册实现**

在Worker启动时，发送注册消息给Master

**步骤**

1. Worker向Master发送注册消息（workerid、cpu核数、内存大小）
   - 随机生成CPU核（1、2、3、4、6、8）
   - 随机生成内存大小（512、1024、2048、4096）（单位M）
2. Master保存Worker信息，并给Worker回复注册成功消息
3. 启动测试

**参考代码**

MasterActor.scala

```scala

import akka.actor.Actor
import com.nicai.common.{MsgRegin, MsgSuccess, Pojo}

import scala.collection.mutable

object MasterActor extends Actor{
  //保存消息
  private var stringToPojo: mutable.Map[String, Pojo] = collection.mutable.Map[String,Pojo]()

  override def receive: Receive = {
    case MsgRegin(workid,cpu,mem) => {
      println("收到注册消息"+workid+"-"+cpu+"-"+mem)
      //保存消息到实体类
      stringToPojo += workid -> Pojo(workid,cpu,mem)
      //回复消息
      sender ! MsgSuccess("success")
    }
  }
}

```

Pojo.scala

```scala
//实体类保存 worker的注册信息
case class Pojo(wokid:String,cpu:Int,mem:Int)
```

MsgPackage.scala

```scala
//注册消息
case class MsgRegin(workid:String,cpu:Int,mem:Int)
//注册成功回复消息
case class MsgSuccess(success:String)
```

WorkerActor.scala

```scala
import java.util.UUID

import akka.actor.{Actor, ActorSelection, ActorSystem}
import com.nicai.common.{MsgRegin, MsgSuccess}

import scala.util.Random


object WorkerActor extends Actor{
  private var actorSelection:ActorSelection=_
  private var CPU_lIST:Int=_
  private var MEM_LIST:Int=_
  private var cpuList=List(1,2,4,8)
  private var memList=List(128,256,512,1024,2048)
  //在actor启动前要做的事
 override def preStart()={
    //1 获取发送的对象
   actorSelection = context.system.actorSelection("akka.tcp://actorSystem@127.0.0.1:7000/user/masterActor")
    //2  封装消息
    val workerid:String=UUID.randomUUID().toString
    var a=new Random()
    CPU_lIST=cpuList(a.nextInt(cpuList.length))
    MEM_LIST=memList(a.nextInt(memList.length))
    val regin = MsgRegin(workerid,CPU_lIST,MEM_LIST)
    //3 发送消息
    actorSelection ! regin
  }

  override def receive: Receive = {
    case MsgSuccess(name) => {
      println("注册后的回复"+name)
    }
  }
}

```

**worker定时发送心跳**

Worker接收到Master返回注册成功后，发送心跳消息。而Master收到Worker发送的心跳消息后，需要更新对应Worker的最后心跳时间。

**步骤**

1. 编写工具类读取心跳发送时间间隔
2. 创建心跳消息
3. Worker接收到注册成功后，定时发送心跳消息
4. Master收到心跳消息，更新Worker最后心跳时间
5. 启动测试

**参考代码**

修改配置文件:

~~~
添加 workerActor中的
//定时发送消息
worker.heartbeat.interval=5
~~~

在workerActor中ConfUtil.scala

```scala
import com.typesafe.config.{Config, ConfigFactory}

object ConfUtil{
  private val config: Config = ConfigFactory.load()

  val `worker.heartbeat.interval` = config.getInt("worker.heartbeat.interval")
}

```

MsgPackage.scala

```scala
//心跳信息
case class MsgHeartBeat(workid:String,cpu:Int,mem:Int)
```

WorkerActor.scala

```scala

import java.util.UUID
import akka.actor.{Actor, ActorSelection, ActorSystem}
import com.nicai.common.{MsgHeartBeat, MsgRegin, MsgSuccess}
import scala.util.Random
object WorkerActor extends Actor {
  private var actorSelection: ActorSelection = _
  private var workerid: String = _
  private var CPU_lIST: Int = _
  private var MEM_LIST: Int = _
  private var cpuList = List(1, 2, 4, 8)
  private var memList = List(128, 256, 512, 1024, 2048)

  //在actor启动前要做的事
  override def preStart() = {
    //1 获取发送的对象
    actorSelection = context.system.actorSelection("akka.tcp://actorSystem@127.0.0.1:7000/user/masterActor")
    //2  封装消息
    workerid = UUID.randomUUID().toString
    var a = new Random()
    CPU_lIST = cpuList(a.nextInt(cpuList.length))
    MEM_LIST = memList(a.nextInt(memList.length))
    val regin = MsgRegin(workerid, CPU_lIST, MEM_LIST)
    //3 发送消息
    actorSelection ! regin
  }

  override def receive: Receive = {
    case MsgSuccess(name) => {
      println("注册后的回复" + name)
      //导入隐式转换
      import scala.concurrent.duration._
      //导入隐式参数
      import context.dispatcher
      //心跳发送
      context.system.scheduler.schedule(0 seconds,
        ConfUtil.`worker.heartbeat.interval` seconds
      ) {
        actorSelection ! MsgHeartBeat(workerid,CPU_lIST,MEM_LIST)
      }
    }
  }
}

```

MasterActor.scala

```scala
import java.util.Date

import akka.actor.Actor
import com.nicai.common.{MsgHeartBeat, MsgRegin, MsgSuccess, Pojo}

import scala.collection.mutable

object MasterActor extends Actor{
  //保存消息
  private var stringToPojo: mutable.Map[String, Pojo] = collection.mutable.Map[String,Pojo]()

  override def receive: Receive = {
    case MsgRegin(workid,cpu,mem) => {
      println("收到注册消息"+workid+"-"+cpu+"-"+mem)
      //保存消息到实体类
      stringToPojo += workid -> Pojo(workid,cpu,mem,new Date().getTime)
      //回复消息
      sender ! MsgSuccess("success")
    }
      //心跳
    case MsgHeartBeat(workid,cpu,mem)=>{
      println("接收到心跳")
      //更新消息
      stringToPojo += workid -> Pojo(workid,cpu,mem,new Date().getTime)
      println(stringToPojo)
    }
  }
}

```

**master定时心跳检测**

如果某个worker超过一段时间没有发送心跳，Master需要将该worker从当前的Worker集合中移除。可以通过Akka的定时任务，来实现心跳超时检查。

**步骤**

1. 编写工具类，读取检查心跳间隔时间间隔、超时时间
2. 定时检查心跳，过滤出来大于超时时间的Worker
3. 移除超时的Worker
4. 对现有Worker按照内存进行降序排序，打印可用Worker

**参考代码**

~~~
修改 master的配置文件

//检查worker心跳的时间周期
master.heartbeat.check.interval=6
//配置worker的心跳超时时间
master.heartbeat.check.timeout=15
~~~



ConfigUtil.scala

```scala
import com.typesafe.config.{Config, ConfigFactory}
object ConfigUtil {
  private val config: Config = ConfigFactory.load()

  // 心跳检查时间间隔
  val `master.heartbeat.check.interval` = config.getInt("master.heartbeat.check.interval")
  // 心跳超时时间
  val `master.heartbeat.check.timeout` = config.getInt("master.heartbeat.check.timeout")
}

```

MasterActor.scala

```scala
import java.util.Date
import akka.actor.Actor
import com.nicai.common.{MsgHeartBeat, MsgRegin, MsgSuccess, Pojo}

import scala.collection.mutable

object MasterActor extends Actor{
  //保存消息
  private var stringToPojo: mutable.Map[String, Pojo] = collection.mutable.Map[String,Pojo]()
  override def preStart(): Unit = {
    // 导入时间单位隐式转换
    import scala.concurrent.duration._
    // 导入隐式参数
    import context.dispatcher

    // 1. 启动定时任务
    context.system.scheduler.schedule(0 seconds,
      ConfigUtil.`master.heartbeat.check.interval` seconds){
      // 2. 过滤大于超时时间的Worker
      val timeoutWorkerMap = stringToPojo.filter {
        keyval =>
          // 获取最后一次心跳更新时间
          val lastHeartBeatTime = keyval._2.heartBeat
          // 当前系统时间 - 最后一次心跳更新时间 > 超时时间（配置文件） * 1000，返回true，否则返回false
          if (new Date().getTime - lastHeartBeatTime > ConfigUtil.`master.heartbeat.check.timeout` * 1000) {
            true
          }
          else {
            false
          }
      }

      // 3. 移除超时Worker
      if(!timeoutWorkerMap.isEmpty) {
        stringToPojo --= timeoutWorkerMap.map(_._1)

        // 4. 对Worker按照内存进行降序排序，打印Worker
        val workerList = stringToPojo.map(_._2).toList
        val sortedWorkerList = workerList.sortBy(_.mem).reverse
        println("按照内存降序排序后的Worker列表：")
        println(sortedWorkerList)
      }
    }
  }

  override def receive: Receive = {
    case MsgRegin(workid,cpu,mem) => {
      println("收到注册消息"+workid+"-"+cpu+"-"+mem)
      //保存消息到实体类
      stringToPojo += workid -> Pojo(workid,cpu,mem,new Date().getTime)
      //回复消息
      sender ! MsgSuccess("success")
    }
      //心跳
    case MsgHeartBeat(workid,cpu,mem)=>{
      println("接收到心跳")
      //更新消息
      stringToPojo += workid -> Pojo(workid,cpu,mem,new Date().getTime)
      println(stringToPojo)
    }
  }
}

```

**多个worker测试**

修改配置文件，启动多个worker进行测试。

**步骤**

1. 测试启动新的Worker是否能够注册成功 (修改worker的端口号即可)
2. 停止Worker，测试是否能够从现有列表删除