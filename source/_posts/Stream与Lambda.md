---
title: Java的Stream
abbrlink: 33510
date: 2019-08-09 20:45:25
tags: Java
categories: Java
summary_img:
encrypt:
enc_pwd:
---

stream  流
  jdk1.8 操作集合和数组  

集合 
list.stream()
数组
Stream.of(数组)
特点 
只能使用一次 后流关闭

方法有两种  延迟方法   终结方法(count() forEach())除了终结  都是延迟

方法(全支持 lambda表达式)
forEach()遍历
filter() 过滤
map() 映射  如string数组  映射为integer数组  也可映射对象
count() 获取个数
limit() 获取前几个  传入 long
skip() 跳过 几个参数 传入 long
concat()  合并流  传入 两个流

常用的函数式接口
Supplier
Consumer
Predicate
Function

# Lambda 表达式

## 一函数式编程思想

面向对象的思想:
做一件事情,找一个能解决这个事情的对象,调用对象的方法,完成事情.
函数式编程思想:
只要能获取到结果,谁去做的,怎么做的都不重要,重视的是结果,不重视过程

## 二 Lambda 转换线程写法

java原始写法匿名内部类

~~~java
public class Demo01Runnable {
	public static void main(String[] args) {
		// 匿名内部类
		Runnable task = new Runnable() {
			@Override
			public void run() { // 覆盖重写抽象方法
				System.out.println("多线程任务执行！");
			}
		};
		new Thread(task).start(); // 启动线程
    }
}
~~~

代码分析:

对于Runnable 的匿名内部类用法，可以分析出几点内容：
        Thread 类需要Runnable 接口作为参数，其中的抽象run 方法是用来指定线程任务内容的核心；

​       为了指定run 的方法体，不得不需要Runnable 接口的实现类；

​       为了省去定义一个RunnableImpl 实现类的麻烦，不得不使用匿名内部类；
​       必须覆盖重写抽象run 方法，所以方法名称、方法参数、方法返回值不得不再写一遍，且不能写错；
​       而实际上，似乎只有方法体才是关键所在。

### 思想转换

用lambda

~~~java
public class Demo02LambdaRunnable {
	public static void main(String[] args) {
		new Thread(() ‐> System.out.println("多线程任务执行！")).start(); // 启动线程
	}
}
~~~

## 匿名内部类分析

匿名内部类的好处与弊端
		一方面，匿名内部类可以帮我们省去实现类的定义；另一方面，匿名内部类的语法——确实太复杂了！
语义分析
仔细分析该代码中的语义， Runnable 接口只有一个run 方法的定义：
		public abstract void run();
即制定了一种做事情的方案（其实就是一个函数）：
		无参数：不需要任何条件即可执行该方案。
		无返回值：该方案不产生任何结果。
		代码块（方法体）：该方案的具体执行步骤。
同样的语义体现在Lambda 语法中，要更加简单：

~~~
() ‐> System.out.println("多线程任务执行！")
~~~

​		前面的一对小括号即run 方法的参数（无），代表不需要任何条件；
​		中间的一个箭头代表将前面的参数传递给后面的代码；
​		后面的输出语句即业务逻辑代码。

## lambda 格式

Lambda省去面向对象的条条框框，格式由3个部分组成：
		一些参数
		一个箭头
		一段代码
Lambda表达式的标准格式为：

~~~~
(参数类型 参数名称) ‐> { 代码语句 }
~~~~

格式说明：
		小括号内的语法与传统方法参数列表一致：无参数则留空；多个参数则用逗号分隔。
		-> 是新引入的语法格式，代表指向动作。
		大括号内的语法与传统方法体要求基本一致。

