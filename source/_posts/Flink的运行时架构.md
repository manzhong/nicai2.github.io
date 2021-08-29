---
title: Flink的运行时架构
tags: Flink
categories: Flink
abbrlink: 5098
date: 2020-09-18 21:21:29
summary_img:
encrypt:
enc_pwd:
---

## 一 Flink运行时的组件

```
1:JObManger 作业管理器
2:TaskManger 任务管理器
3:ResourceManger 资源管理器
4:Dispacher 分发器
5:JObClient  作业客户端
```

### 1 任务完整流程

​	Flink程序需要提交给Job Client。 然后，Job Client将作业提交给Job Manager。 Job Manager负责协调资源分配和作业执行。 它首先要做的是分配所需的资源。 资源分配完成后，任务将提交给相应的Task Manager。 在接收任务时，Task Manager启动一个线程以开始执行。 执行到位时，Task Manager会继续向Job Manager报告状态更改。 可以有各种状态，例如开始执行，正在进行或已完成。 作业执行完成后，结果将发送回客户端（Job Client）。

### 2 JobManger

```
1:控制一个应用程序执行的主进程，每个应用程序都会被一个不同的JM所控制执行
2：jm会先接受要执行的应用程序，这个程序包含：作业图（JobGraph）,逻辑数据流图（Logical dataflow graph）和打包了所有类，库和其他资源的jar包
3：jm会把JobGraph转换为一个物理层面的数据流图，这个图被称为执行图（ExecutionGraph）,包含了所有可以并发执行的任务
4：jm会向资源管理器（ResourceManger）请求执行任务必要的资源，也就是任务管理器（TaskManger）上的插槽（slot），一旦获取到了足够的资源，就会将执行图分发到真正运行他们的TaskManger上，而在运行过程中，Jobmanger会负责所有需要中央协调的操作，如检查点（checkpoint）的协调等
```

​	master 进程（也称为作业管理器）协调和管理程序的执行。 他们的主要职责包括调度任务，管理检查点，故障恢复等。可以有多个Masters 并行运行并分担职责。 这有助于实现高可用性。 其中一个master需要成为leader。 如果leader 节点发生故障，master 节点（备用节点）将被选为领导者。

作业管理器包含以下重要组件：

- Actor system:Flink内部使用Akka actor系统在作业管理器和任务管理器之间进行通信,spark的通信原先使用akka,后改为netty

  ```
  在Flink中，actor是具有状态和行为的容器。 actor的线程依次继续处理它将在其邮箱中接收的消息。 状态和行为由它收到的消息决定
  Actor使用消息传递系统相互通信。 每个actor都有自己的邮箱，从中读取所有邮件。 如果actor是本地的，则消息通过共享内存共享，但如果actor是远程的，则通过RPC调用传递消息。
  ```

- Scheduler

- Check pointing

### 3 TaskManger

```
1: Flink中的工作进程，通常Flink中会有多个Taskmanger运行，每一个Tm都包含一定数量的插槽（slots）,插槽的数量限制了tm能够执行的任务数量（tm的数量*slot的数量=所有的slot的数量 这个数量代表了任务的总并行执行能力（不是并行度））
2：启动后，tm会向资源管理器注册他的插槽，收到资源管理器的指令后。tm就会将一个或多个插槽提供给Jobmanger调用，jm就可以向插槽分配任务（tasks）来执行,允许多个task共享slot
3：在执行过程中，一个tm可以跟其他运行的同一应用程序的tm交换数据
```

### 4 ResourceManger

```
1: 主要负责管理tm的插槽（slot），tm的插槽是flink中定义的处理资源的单位
2：Flink为不同的环境和资源管理工具提供了不同的资源管理器，如yarn,mesos,k8s和standalone部署
3：当jm申请插槽资源时，rm会将有空闲插槽的tm分配给jm，如果rm没有足够的插槽来满足jm的请求，他还可以向资源提供平台发起会话，以提供启动tm进程的容器
```

### 5 Dispatcher

```
1:可以跨作业运行，为应用提供rest的接口
2:当一个应用被提交，分发器会启动并将应用移交给一个jm
3：dis会启动一个webui，用来方便展示和监控作业执行的信息
4：dis在架构中并不是必需的，这取决于应用提交运行的方式（yarn无）
```

## 二 任务的提交流程

standalone的模式4和5 在集群启动的时候就已经完成

![1552262880777](/images/flink/submit.png)

### 2 yarn模式(YARN-Cluster)(per_job)

图中的resourcemanger是yarn的resourcemanger,flink自己的resourcemanger在applicationMaster中。实际中flink先向自己的rm申请资源，flink的rm在向yarn的rm申请资源

![1552262880777](/images/flink/yarnsub.png)

## 三 任务调度原理

![1552262880777](/images/flink/run1.png)

```
编写代码，代码经过编译打包，直接会生成一个dataFlow graph（数据流图），经过接口（client)发送期间会把能合并的数据操作进行合并，合并后得到job graph，提交给jobmanger,分析job流图，判断并行度，任务有几个子任务，需要的资源等，然后向resourcemanger申请资源，然后向worker分配任务
```

```
问题：
	1：怎么实现并行计算？
	2：并行任务，需要占用多少slot？
	3：一个留任务到底包含多少个子任务？
```



### 3.1 并行度

![1552262880777](/images/flink/bx.png)

```
一个特定算子的子任务（subtask）的个数被称之为并行度（parallelism）（一般一个子任务占用一个slot）.
一般情况下，一个stream的并行度，可以认为是其所有算子中最大的并行度
```

### 3.2 Taskmanger和slots

![1552262880777](/images/flink/ts.png)

```
1: flink中每一个taskmanger都是一个jvm进程，它可能会在独立的线程上执行一个或多个子任务
2：为了控制一个taskmanger能接受多少个task，tm通过task slot来进行控制，一个tm至少一个slot(每一个slot可以执行一个独立的任务)，实际就是一个tm的内存划分为多少分（slot的个数），slot的个数代表了tm的并行静态资源，这静态资源能用多少取决于并行度设置多少
3：slot就是一块独立的资源，主要是内存的隔离
```

![1552262880777](/images/flink/ts2.png)

```
1:默认情况下，flink允许子任务共享slot,即使他们是不同任务的子任务，这样的结果是，一个slot可以保存作业的整个管道
2：tsak slot是静态的概念，是指tm具有的并发执行的能力
```

### 3.3 并行子任务的分配

![1552262880777](/images/flink/bxf.png)

```
左侧的jobgraph的最大并行度是4，所以至少需要4个slot,具体分配a的并行度是4，四个slot依次都有，b并行度4，则4个slot都有，c的并行度为2，....(至于c和e具体分配到哪，这个可以配置，可以分配到不同的tm上，flink默认是随机）
```

**具体的实例**

![1552262880777](/images/flink/bf3.png)

![1552262880777](/images/flink/bf4.png)

```
配置文件中的tm的slot的个数是3，例子1：类似于词频统计...不细说看图
```

### 3.4 程序与数据流

**任务划分为几个子任务，有些情况会把任务合并成一个任务，有时不合，现在讨论这个问题**

![1552262880777](/images/flink/coded.png)

```
1:所有的flink程序是由三部分组成：Source,Transformation,sink
2：Source读取数据源，transformation利用算子进行处理加工，sink负责输出
```

![1552262880777](/images/flink/dd.png)

```
1：在运行时，fkink上运行的程序会被映射成逻辑数据流，包含三部分（source,transformation,sink）
2：每个dataflow(df)，以一个或多个source开始，以一个或多个sink结束，df类似于任意的有向无环图
3：大部分情况下，程序中的转换算子跟df中的算子（operator）是一一对应的关系，不划分stage
```

##### 3.4.1 执行图

```
1:flink中的执行图可以划分为四层：StreamGraph->jobgraph->executionGraph->物理执行图
StreamGraph（sg）：是根据用户通过streamapi编写的代码生成最初的图，用来表示程序的拓补结构
jobgraph(jg)：sg经过优化后生成了jg,提交给jobmanger的数据结构，主要优化为将多个符合条件的节点chain在一起作为一个节点(webui上看到）
executionGraph(eg）：jobmanger根据jg生成eg,eg是jg的并行化版本，是调度最核心的数据结构
物理执行图：jobmanger根据eg对job进行的调度后，在各个taskmanger上部署task后形成的图，并不是一个具体的数据结构
```

![1552262880777](/images/flink/zxt.png)

##### 3.4.2 数据传输格式

```
1:一个程序中，不同的算子可能具有不同的并行度
2：算子之间传输数据的形式可以是one-to-one(forwarding)的模式，也可以是redistributing的，模式。具体是哪一种，取决于算子的种类和并行度的设置
one-to-one:stream维护着分区以及元素的顺序（如source与map之间），这意味着map算子的子任务看到的元素的个数以及顺序跟source算子的子任务生产的元素个数，顺序相同，map,filter,flatmap等算子都是one-to-one的对应关系
Redistribution:stream的分区会发生改变，每一个算子的子任务依据所选择的转换发送数据到不同的目标任务，例如，keyby基于hashcode重分区，而broadcast和rebalance会随机重新分区，这些算子都会引起redistribute过程，而redistribution过程类似于spark中的shuffle过程
```

##### 3.4.3 任务链

```
1: flink采用了一种称为任务链的优化技术，可以在特定条件下减少本地通信的开销。为了满足任务链的要求，必须将两个或多个算子设为相同的并行度，并通过 本地转发（local forward）的方式进行 链接
2：相同并行度的onetoone操作，flink这样相连的算子链接在一起形成一个task,原来的算子成为里面的subtask
3: 并行度相同，且是one-to-one的操作，两个条件缺一不可（子任务合并）
```

![1552262880777](/images/flink/rwl.png)

```
1：flink的算子有个方法disableChaining(),则这个算子不参与任务链合并，前后都断开:
2：chain的切断，在想切断的算子后方法 startNewChain() 开启一个新的链
拆开是拆开了，但是若是slot是共享的，这个拆开意义就不大了，也可以设置拆开后放在不同的slot执行，如flatmap(...).slotSharingGroup("a"),
a组：是从当前算子开始，后面的算子没有划分新的组，则后面的所有算子共享这个slot组，就和之前的分开了，属于不同的组，（在同一个组内可以共享slot）
如flatmap(/...).disableChaining("a").map(....).startNewChain()
ps：若是不指定，则默认所有的算子在一个共享组内为default,自己指定是不要命名为default
```

