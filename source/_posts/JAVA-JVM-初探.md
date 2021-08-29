---
title: JAVA-JVM-初探
abbrlink: 58431
date: 2017-07-02 20:42:30
tags: Jvm
categories: Jvm
summary_img:
encrypt:
enc_pwd:
---

## 一   Java -version

1 hotspot  热点探测 jvm核心  jdk1.5 之后有 为执行引擎

  优化class文件 因为CPU不会直接执行class文件  要转为二进制才可以

2 jdk 有client 和server默认启动为client  可在jre/lib/i386 里吧两者顺序调整 前面的先起

当大型项目时 可改为server

在jdk1.8之后都默认为server 且client 为ignore

## 二   Jvm结构

 ![img](/images/jvm/jvm结构图.jpg)                

 ![img](/images/jvm/jvm简介.jpg)

 ![img](/images/jvm/jvm简介2.jpg)

## 三 堆结构图及分代

Jvm 根据对象的存活周期不同 把堆内存分为 新生代,老年代,和永久代

目的  提高对象内存分配和垃圾回收效率

经过多次回收荏苒存活的对象放在老年代,静态属性和类信息放在永久代,

新生代中对象存活时间短 在新生代中做垃圾回收 老年代垃圾回收频率少

永久代一般不进行垃圾回收,可根据不同的代用不同的;垃圾回收算法.

堆有五个区(细)

新生代(3个) Eden from to  eden 回收最频繁 =幸存的放在from和to的幸存区在此幸存放在 老年去

Old区

永久区

1新生代

新创建的对象放在新生代,这里边存活率很低,一次回收可回收百分之八十到九十五,  Eden from to 比例8:1:1

 在jdk1.8和之后没有老年代 老年代与方法区作用类似;

## 三   垃圾回收

### 3.1 引用计数法Reference Counting

### 3.2 标记-清除法Mark-Sweep

### 4.3、复制算法（Copying）

### 4.4、标记-压缩算法（Mark-Compact）

### 4.5、增量算法（Incremental Collecting）

### 4.6、分代（Generational Collecting）



 

