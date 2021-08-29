---
title: python
tags:
  - Python
categories: Python
encrypt: 
enc_pwd: 
abbrlink: 51872
date: 2020-05-02 22:31:31
summary_img:
---

## 一 简介

​	一种面向对象，面向函数的解释型计算机程序设计语言,Python语法简洁清晰，特色之一是强制用空白符(white space)作为语句缩进

## 二 安装

### 1、Python的下载

         1、python网址：[https://www.python.org/](https://www.python.org/)

         2、anaconda网址：[https://www.anaconda.com/](https://www.anaconda.com/)

注意：Anaconda指的是一个开源的[Python](https://baike.baidu.com/item/Python)发行版本，其包含了conda、Python等180多个科学包及其依赖项。

## 三 python的基础语法

### 1 注释及乱码

1:单行注释：以#开头，#右边的所有东西当做说明，而不是真正要执行的程序，起辅助说明作用

2:多行注释：’’’多行注释’’’可以写多行的功能说明

3:Python乱码问题

            由于Python源代码也是一个文本文件，所以，当你的源代码中包含中文的时候，在保存源代码时，就需要务必指定保存为UTF-8编码。当Python解释器读取源代码时，为了让它按UTF-8编码读取，我们通常在文件开头写上这两行：

```python
# -*- coding:utf-8 -*-
# coding=utf-8
```

### 2 变量

变量三要素：变量的名称，变量的类型，变量的值

**变量类型**

为了更充分的利用内存空间以及更有效率的管理内存，变量是有不同的类型，如图所示

```
                                         object
          Internal对象      Mapping对象    Numeric对象  Sequence对象  Fundamental对象
            1function         dict			布尔			String		Type
            2code						   浮点           List
            3frame						   整形		   tuple
            4module
            5method
```

python中无null但有none,且不等于0

注意:

​	1:布尔值和布尔代数的表示完全一致，一个布尔值只有True、False两种值，要么是True，要么是False，在Python中，可以直接用True、False表示布尔值（请注意大小写），也可以通过布尔运算计算出来

布尔值可以用and、or和not运算。

and运算是与运算，只有所有都为True，and运算结果才是True：

​	2:空值是Python里一个特殊的值，用None表示。None不能理解为0，因为0是有意义的，而None是一个特殊的空值

​	3:在python中，只要定义了一个变量，而且它有数据，那么它的类型就已经确定了，不需要咱们开发者主动的去说明它的类型，系统会自动辨别

可以使用type(变量的名字)，来查看变量的类型

**常见的数据类型转换**

| 函数                     | 说明                            |
| ---------------------- | ----------------------------- |
| int(x [,base ])        | 将x转换为一个整数                     |
| long(x [,base ])       | 将x转换为一个长整数                    |
| float(x )              | 将x转换到一个浮点数                    |
| complex(real [,imag ]) | 创建一个复数                        |
| str(x )                | 将对象 x 转换为字符串                  |
| repr(x )               | 将对象 x 转换为表达式字符串               |
| eval(str )             | 用来计算在字符串中的有效Python表达式,并返回一个对象 |
| tuple(s )              | 将序列 s 转换为一个元组                 |
| list(s )               | 将序列 s 转换为一个列表                 |
| chr(x )                | 将一个整数转换为一个字符                  |
| unichr(x )             | 将一个整数转换为Unicode字符             |
| ord(x )                | 将一个字符转换为它的整数值                 |
| hex(x )                | 将一个整数转换为一个十六进制字符串             |
| oct(x )                | 将一个整数转换为一个八进制字符串              |

4:格式化输出:

我们经常会输出类似'亲爱的xxx你好！你xx月的话费是xx，余额是xx'之类的字符串，而xxx的内容都是根据变量变化的，所以，需要一种简便的格式化字符串的方式。

 在Python中，采用的格式化方式和C语言是一致的，用%实现，举例如下：

```python
>>> 'Hello, %s' % 'world'

'Hello, world'

>>> 'Hi, %s, you have $%d.' % ('Michael', 1000000)

'Hi, Michael, you have $1000000.'
```

你可能猜到了，%运算符就是用来格式化字符串的。在字符串内部，%s表示用字符串替换，%d表示用整数替换，有几个%?占位符，后面就跟几个变量或者值，顺序要对应好。如果只有一个%?，括号可以省略。

常见的占位符有：

**%d ****整数**

**%f ****浮点数**

**%s ****字符串**

**%x ****十六进制整数**

其中，格式化整数和浮点数还可以指定是否补0和整数与小数的位数：

```python
>>> '%2d-%02d' % (3, 1)

' 3-01'

>>>'%.2f' % 3.1415926
'3.14'
```



如果你不太确定应该用什么，%s永远起作用，它会把任何数据类型转换为字符串：

\>>> 'Age: %s. Gender: %s' % (25, True)

'Age: 25. Gender: True'

有些时候，字符串里面的%是一个普通字符怎么办？这个时候就需要转义，用%%来表示一个%：

```python
>>> 'growth rate: %d %%' % 7
'growth rate: 7 %'
```

### 3 流程判断

条件判断:

```
elif是else if的缩写，完全可以有多个elif，所以if语句的完整形式就是：
if <条件判断1>:
    <执行1>
elif <条件判断2>:
    <执行2>
elif <条件判断3>:
    <执行3>
else:
    <执行4>
```

循环:

```
while 条件:
        条件满足时，做的事情1
        条件满足时，做的事情2
        条件满足时，做的事情3
        ...(省略)...
```

````
for 临时变量 in 列表或者字符串等:
        循环满足条件时执行的代码
    else:
        循环不满足条件时执行的代码

````

## 四 集合







