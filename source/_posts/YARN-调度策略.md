---
title: YARN-调度策略
tags:
  - Hadoop
categories: Hadoop
encrypt: 
enc_pwd: 
abbrlink: 30491
date: 2020-06-07 18:51:11
summary_img:
---

# 一 简介

YARN总共有三种调度策略：**FIFO、Capacity Scheduler、Fair Scheduler**。FIFO就是先进先出，最简单，实际中用的也比较少，这里就不再赘述了。Capacity Scheduler比Fair Scheduler出现的早，但随着慢慢的发展和改进，这二者的差异也越来越小了（个人觉得以后这两个合并为一个也是有可能的）。使用情况的话目前CDH（版本为5.8.2）默认使用Fair Scheduler，HDP（版本为2.6）默认使用Capacity Scheduler。下面我们分别介绍。

# 二 使用说明

## 2.1 Capacity Scheduler

Capacity Scheduler主要是为了解决多租户（multiple-tenants）的问题，也就是为了做资源隔离，让不同的组织使用各自的资源，不相互影响，同时提高整个集群的利用率和吞吐量。Capacity Scheduler里面的核心概念或者资源就是**队列（queue）**。概念理解上很简单，分多个队列，每个队列分配一部分资源，不同组织的应用运行在各自的队列里面，从而做到资源隔离。但为了提高资源（主要是CPU和内存）利用率和系统吞吐量，在情况允许的情况下，允许队列之间的资源抢占。

开启Capacity Scheduler调度策略的方法是在**yarn-site.xml**中进行如下设置：

```
yarn.resourcemanager.scheduler.class=org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler
yarn.resourcemanager.scheduler.class=org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduleryarn.resourcemanager.scheduler.class=org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduleryarn.resourcemanager.scheduler.class=org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduleryarn.resourcemanager.scheduler.class=org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduleryarn.resourcemanager.scheduler.class=org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler
```

该策略的配置文件为*$HADOOP_HOME/etc/hadoop/capacity-scheduler.xml*（目录可能变更）。

下面看一下官方给的Capacity Scheduler的一些特点：

**Hierarchical Queues**：支持分层队列，或者说嵌套队列。在YARN中，队列的组织有点像文件系统的路径，内置了一个根*root*和一个默认的队列*default*。现在假设我们的生产（需分配40%的资源）和开发（需分配60%的资源）共用一个集群，而开发又分为team1和team2两个组（两个组资源对半分），那我们可以这样划分队列：

```
root
    - prod
    - dev
        - team1
        - team2
```

对应的配置文件为（部分内容）：

```XML
<property>
    <name>yarn.scheduler.capacity.root.queues</name>
    <value>prod,dev</value>
    <description>for production and development.</description>
</property>

<property>
    <name>yarn.scheduler.capacity.root.dev.queues</name>
    <value>team1,team2</value>
    <description>queue for team1 and team2.</description>
</property>

<property>
    <name>yarn.scheduler.capacity.root.prod.capacity</name>
    <value>40</value>
    <description>40% of all resource.</description>                
</property>

<property>
    <name>yarn.scheduler.capacity.root.dev.capacity</name>
    <value>60</value>
    <description>40% of all resource.</description>                
</property>

<property>
    <name>yarn.scheduler.capacity.root.dev.team1.capacity</name>
    <value>50</value>
    <description>half of dev.</description>        
</property>

<property>
    <name>yarn.scheduler.capacity.root.dev.team2.capacity</name>
    <value>50</value>
    <description>half of dev.</description>                
</property>
```

这里只简单分了队列，并设置了队列所绑定的资源量，还有很多其他的配置。通过这种嵌套的多层队列机制，我们可以根据需求非常灵活的设置队列。

- **Capacity Guarantees**：容量保证，这个很简单，像上面我们已经给每个队列绑定了一定的容量。当然还有很多针对容量的配置项。
- **Security**：给以给不同的队列设置不同的权限，甚至每个队列都可以设置一个管理员。总之，权限都是可以针对不同队列去设置的，所以可以提供了相当出色的安全控制手段。
- **Elasticity**：弹性。为了提高系统利用率和吞吐量，队列之间支持资源抢占。继续以上面的例子来说：假设刚开始prod队列是空闲的，而dev队列已经满了，那dev里面的应用是可以从prod里面抢占资源的，也就是说dev队列所使用的资源是可以超过预先分配的60%的，但当prod里面的应用页需要资源的时候，dev抢占的部分就需要归还给prod了，正常是需要dev里面的应用释放资源后，prod里面的应用才可以拿回资源。YARN提供了preemption机制，可以强制拿回资源，后面再介绍。
- **Multi-tenancy**：多租户很容易理解了，通过不同队列实现资源隔离，从而实现多租户支持。
- **Operability**：可操作性主要体现在两个方面：(1)修改队列的很多属性（比如增加队列、修改队列的容量、ACL等）是不需要重启的，直接在应用运行期间就可以直接修改生效。(2)队列可以启停，比如我们可以STOP一个队列，STOP之后，队列里面已有的应用会继续运行完，但新的应用是无法提交到该队列中来的。之后我们也可以重新启动该队列。操作非常方便。
- **Resource-based Scheduling**：我们可以给一个应用指定它所需要的资源量，这个量可以比它真正使用的多，目前只支持指定内存。
- **Queue Mapping based on User or Group**：我们可以实现指定一个用户/用户组到队列的映射，这样后续可以根据用户/用户组来判断一个应用该提交到哪个队列里面去。比如下面的示例：

```XML
<property>
<name>yarn.scheduler.capacity.queue-mappings</name>
<value>u:user1:queue1,g:group1:queue2,u:%user:%user,u:user2:%primary_group</value>
<description>
Here, <user1> is mapped to <queue1>, <group1> is mapped to <queue2>, 
maps users to queues with the same name as user, <user2> is mapped 
to queue name same as <primary group> respectively. The mappings will be 
evaluated from left to right, and the first valid mapping will be used.
</description>
   </property>
```

​		映射语法为：*[u or g]:[name]:[queue_name]*. u表示用户，g表示组。name表示用户名或组名，queue_name表示队列		名。*u:%user:%user*表示队列名与提交应用的用户名相同，*u:user2:%primary_group*表示队列名与*user2*这个用户所		属的主组组名相同。

- **Priority Scheduling**：提交应用的时候可以指定优先级，数字越大优先级越高。目前只有当队列内使用FIFO策略时才支持该特性。
- **Absolute Resource Configuration**：上面分配资源量的时候我们使用的是百分比，也支持直接使用指定具体的资源量，比如*memory=10240,vcores=12*。
- **Dynamic Auto-Creation and Management of Leaf Queues**：这个特性和上面用户组映射到队列的特性对应，当映射的队列不存在时，可以动态的自动创建。目前只支持创建叶子队列。

**这里虽然介绍的是Capacity Scheduler调度策略的特点，但文章刚开始的时候我们就说过Capacity Scheduler和Fair Scheduler已经非常类似了，所以上面的列的这些特性其实在较新的YARN版本里面Fair Scheduler也都是具备的。**

我们再讲一下Capacity Scheduler和Fair Scheduler的**Preemption**。细心的人在看前面关于**Elasticity**特点的讲解时可能就已经有个疑问了：

***如果dev队列抢占了prod队列的所有资源，而其运行的应用都是非常耗时的那种，那prod里面应用岂不是将一直“拿不回”资源？\***

是的，针对这种问题YARN提供了两种解决方案，Preemption就是其中之一。其实原理也很简单，当设置了Preemption之后，prod不会等待dev里面的应用主动释放资源，而是会再等待一定时间之后强制拿回资源。这个强制就是dev里面的一些应用会被杀掉以释放抢占的prod的资源。Capacity Scheduler和Fair Scheduler都支持Preemption，但设置稍有不同，对于Capacity Scheduler策略，开启Preemption需要做做如下配置：

```XML
# yarn-site.xml中

yarn.resourcemanager.scheduler.monitor.enable=true
yarn.resourcemanager.scheduler.monitor.policies=org.apache.hadoop.yarn.server.resourcemanager.monitor.capacity.ProportionalCapacityPreemptionPolicy

# capacity-scheduler.xml中
yarn.scheduler.capacity.<queue-path>.disable_preemption=false
```

除了Preemption之外，还有另外一种解决方案：我们可以设置一个队列最大可分配的资源。对应的两个配置项全局配置项（在yarn-site.xml中设置）为*yarn.scheduler.maximum-allocation-mb*和*yarn.scheduler.maximum-allocation-vcores*，Capacity Scheduler自己的配置项为（在capacity-scheduler.xml中配置，可覆盖全局配合）*yarn.scheduler.capacity..maximum-allocation-mb*和*yarn.scheduler.capacity..maximum-allocation-vcores*，前者用于限制内存，后者用于限制CPU，都既支持百分比，也支持具体的值。当我们设置了这两个值以后，队列所能拥有的资源就有了一个上限，当达到这个上限之后，即使其它队列有空闲资源，你也无法再抢占了。

更多Capacity Scheduler配置选项请查看[CapacityScheduler Configuration](https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/CapacityScheduler.html).

## 2.2 FAIR SCHEDULER

其实上面讲的除了具体的配置项Capacity Scheduler和Fair Scheduler不一样以为，特性基本是相同的。相比于Capacity Scheduler设计的初衷是为了支持多租户，Fair Scheduler的设计目标是为了更好的更公平的分配资源。举个例子，在Fair Scheduler调度策略下，假设刚开始系统中有两个队列Q1和Q2，刚开始我们往Q1中提交了一个应用A，此时整个系统中只有A一个应用，所以它可以使用**系统的**全部资源；然后我们再往Q2中提交一个应用B，此时B将从A那里拿去50%的资源（没有开Preemption的情况下，需要等A内的应用释放后，B才能拿回）。接着，我们再往Q2中提交一个应用C，那C将从B那里再拿去50%的资源，此时，A占系统一半的资源，B、C分别占四分之一的资源。这就是所谓的Fair。

不过，现在Capacity Scheduler和Fair Scheduler相互同化，已经不再那么纯粹了。在Fair Scheduler中也是queue（有时称为pool，即资源池）。特性就不再介绍了，基本和Capacity Scheduler相同，这里主要介绍一下Fair Scheduler比较重要的一些配置项。

开启Fair Scheduler调度策略的方法是在**yarn-site.xml**中进行如下设置:

```
yarn.resourcemanager.scheduler.class=org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler
```

该策略的配置文件为*$HADOOP_HOME/etc/hadoop/capacity-scheduler.xml*（目录可能变更）。

假设现在我们要配置如下的一个队列：

```
root
    - default
    - q
        - sub-q1
        - sub-q2
```

*default*是系统默认的，我们只需要配置*q、sub-q1、sub-q2*即可，示例的配置文件capacity-scheduler.xml如下：

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<allocations>
    <queue name="root">
        <weight>1.0</weight>
        <schedulingPolicy>drf</schedulingPolicy>
        <aclSubmitApps> </aclSubmitApps>
        <aclAdministerApps>*</aclAdministerApps>
        <queue name="default">
            <minResources>512 mb, 1 vcores</minResources>
            <maxResources>24576 mb, 24 vcores</maxResources>
            <weight>1.0</weight>
            <fairSharePreemptionTimeout>60</fairSharePreemptionTimeout>
            <schedulingPolicy>drf</schedulingPolicy>
            <aclSubmitApps>*</aclSubmitApps>
            <aclAdministerApps>*</aclAdministerApps>
            <maxAMShare>0.6</maxAMShare>
        </queue>
        <queue name="q">
            <minResources>512 mb, 1 vcores</minResources>
            <maxResources>55296 mb, 54 vcores</maxResources>
            <weight>4.0</weight>
            <fairSharePreemptionTimeout>60</fairSharePreemptionTimeout>
            <schedulingPolicy>drf</schedulingPolicy>
            <aclSubmitApps>*</aclSubmitApps>
            <aclAdministerApps>*</aclAdministerApps>
            <maxAMShare>-1.0</maxAMShare>
            <queue name="sub-q1">
                <minResources>512 mb, 1 vcores</minResources>
                <maxResources>45056 mb, 44 vcores</maxResources>
                <weight>3.0</weight>
                <fairSharePreemptionTimeout>60</fairSharePreemptionTimeout>
                <schedulingPolicy>drf</schedulingPolicy>
                <aclSubmitApps>*</aclSubmitApps>
                <aclAdministerApps>*</aclAdministerApps>
                <maxAMShare>0.5</maxAMShare>
            </queue>
            <queue name="sub-q2">
                <minResources>512 mb, 1 vcores</minResources>
                <maxResources>16384 mb, 16 vcores</maxResources>
                <weight>1.0</weight>
                <fairSharePreemptionTimeout>60</fairSharePreemptionTimeout>
                <schedulingPolicy>drf</schedulingPolicy>
                <aclSubmitApps>*</aclSubmitApps>
                <aclAdministerApps>*</aclAdministerApps>
                <maxAMShare>0.6</maxAMShare>
            </queue>
        </queue>
    </queue>
    <defaultQueueSchedulingPolicy>drf</defaultQueueSchedulingPolicy>
    <queuePlacementPolicy>
        <rule name="specified" create="false"/>
        <rule name="default"/>
    </queuePlacementPolicy>
</allocations>
```

这里我们介绍几个重点配置项：

- **weight**：在Fair Scheduler里面不使用百分比，而是用weight，它和百分比很像，但不是百分比。系统默认的*default*队列的weight默认是1. 如果我们再增加一个q队列，weight设为3，那*default*将占系统总资源的1/(1+3)，*q*占3/(1+3)。
- **schedulingPolicy**：只一个队列内的调度策略，目前支持*fifo、fair(默认)、drf*，前两者很好理解，这里主要说一下DRF(Dominant Resource Fairness)。fifo先进先出，fair策略仅以内存作为资源判定标准，而DRF高级一些会根据应用对CPU和内存的需求来判断该应用是内存型（首要资源是内存）的还是CPU型（首要资源是CPU）的，然后根据首要资源来公平调度。举个例子比如系统总资源为100个CPU，10TB内存，应用A需要2个CPU、300GB内存，应用B需要6个CPU，100GB内存，和系统总资源求一下百分比，则应用A需要的资源为2%的CPU和3%的内存，3% > 2%，所以应用A是内存型的，同理B需要的资源为6%的CPU和1%的内存，所以B是CPU型的。根据DRF的调度规则，应用A的首要资源为3%，应用B的首要资源为6%，所以给B分配资源比给A分配的资源多一倍。
- **minResources**：其实这个最小资源不是用来做限制的，而主要是用来做调度的。举个例子，当若干个队列的资源都被别人抢占，都低于这个最小值时，那如果有新释放的资源，会优先给差的最多（和队列的minResources相比）的队列分配。配置格式为：*X mb, Y vcores*。
- **maxResources**：队列所能拥有的资源的最大值，即使抢占别的队列也不能超过该值。和Capacity Scheduler中的*yarn.scheduler.capacity..maximum-allocation-mb*与*yarn.scheduler.capacity..maximum-allocation-vcores*作用相同。
- **maxAMShare**：Capacity Scheduler中类似的设置项。我们知道YARN里面的应用运行的时候首先会拉起来一个ApplicationMaster(AM)，而这个选项就是设置我们给队列分配的资源中有多少可以用来创建AM。该选项设置百分比，默认为0.5，表示队列中有一半的资源可以用来创建AM。设置为-1禁用该选项，表示不检查AM占用的资源，实质也就是不限制。该选项可以用来限制队列内的应用数目。
- **maxRunningApps**：允许队列内同时运行的应用个数。
- **queuePlacementPolicy**：配置应用放置到哪个队列的策略，里面写一系列规则，逐个匹配，类似于编程语言里面的switch语句。合法的rule有：*specified，user，primaryGroup，secondaryGroupExistingQueue，nestedUserQueue，default，reject*。每个rule都支持 *create*，表示当该条rule匹配成功之后，如果队列不存在，是否创建，默认为true。如果设为false，当队列不存在时，则匹配不成功，接着往下匹配。一定要保证最后一条规则是肯定可以成功的，类似于switch要有一个default一样。

其他更多选项说明请查看[FairScheduler Configuration](https://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/FairScheduler.html).

PS: 配置为队列后，我们可以在Resource Manager的Web界面上的schedule一栏中查看这些队列的资源及使用情况。

### 参考:

```http
https://niyanchun.com/yarn-scheduler.html
```



