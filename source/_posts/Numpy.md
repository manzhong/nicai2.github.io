---
title: Numpy
tags:
  - Ai
  - Python
  - 数据分析与可视化
categories: 数据分析与可视化
abbrlink: 64775
date: 2020-11-23 21:35:45
summary_img:
encrypt:
enc_pwd:
---

## 一 简介

​	NumPy(Numerical Python) 是 Python 语言的一个扩展程序库，支持大量的维度数组与矩阵运算，此外也针对数组运算提供大量的数学函数库

**安装:**

```shell
pip3 install --user numpy scipy matplotlib
```

--user 选项可以设置只安装在当前的用户下，而不是写入到系统目录。默认情况使用国外线路，国外太慢，我们使用清华的镜像就可以:

```shell
pip3 install numpy scipy matplotlib -i https://pypi.tuna.tsinghua.edu.cn/simple
```

验证:

```python
>>> from numpy import *
>>> eye(4)
array([[1., 0., 0., 0.],
       [0., 1., 0., 0.],
       [0., 0., 1., 0.],
       [0., 0., 0., 1.]])
```

**from numpy import ** 为导入 numpy 库。**eye(4)** 生成对角矩阵。

## 二 使用

### 1 NumPy Ndarray 对象

NumPy 最重要的一个特点是其 N 维数组对象 ndarray，它是一系列同类型数据的集合，以 0 下标为开始进行集合中元素的索引。

ndarray 对象是用于存放同类型元素的多维数组。

ndarray 中的每个元素在内存中都有相同存储大小的区域。

ndarray 内部由以下内容组成：

- 一个指向数据（内存或内存映射文件中的一块数据）的指针。
- 数据类型或 dtype，描述在数组中的固定大小值的格子。
- 一个表示数组形状（shape）的元组，表示各维度大小的元组。
- 一个跨度元组（stride），其中的整数指的是为了前进到当前维度下一个元素需要"跨过"的字节数。

创建一个 ndarray 只需调用 NumPy 的 array 函数即可：

```python
import numpy as np 
np.array(object, dtype = None, copy = True, order = None, subok = False, ndmin = 0)
```

参数说明:

| 名称     | 描述                             |
| :----- | :----------------------------- |
| object | 数组或嵌套的数列                       |
| dtype  | 数组元素的数据类型，可选                   |
| copy   | 对象是否需要复制，可选                    |
| order  | 创建数组的样式，C为行方向，F为列方向，A为任意方向（默认） |
| subok  | 默认返回一个与基类类型一致的数组               |
| ndmin  | 指定生成数组的最小维度                    |

实例:

```python
##实例一
	import numpy as np 
	a = np.array([1,2,3])  
	print (a)
##实例二
	# 多于一个维度  
	import numpy as np 
	a = np.array([[1,  2],  [3,  4]])  
	print (a)
##实例三
	import numpy as np 
	a = np.array([[1,2,3],[4,5,6]], ndmin =  3)  
	print (a)
	#结果:
	  [[[1 2 3]
	  [4 5 6]]]
##实例四
# dtype 参数  
import numpy as np 
a = np.array([1,  2,  3], dtype = complex)  
print (a)
result:[ 1.+0.j,  2.+0.j,  3.+0.j]
```

#### 1.1**创建数组的方式**

###### 1.1.1empty

| shape | 数组形状                                     |
| ----- | ---------------------------------------- |
| dtype | 数据类型，可选                                  |
| order | 有"C"和"F"两个选项,分别代表，行优先和列优先，在计算机内存中的存储元素的顺序。 |

```python
import numpy as np 
import numpy as np 
x = np.empty([3,2], dtype = int) 
print (x)#空的 里面是随机值
```

###### 1.1.2 zeros

| 参数    | 描述                                   |
| :---- | :----------------------------------- |
| shape | 数组形状                                 |
| dtype | 数据类型，可选                              |
| order | 'C' 用于 C 的行数组，或者 'F' 用于 FORTRAN 的列数组 |

```python
import numpy as np
# 默认为浮点数
x = np.zeros(5) 
print(x)
# 设置类型为整数
y = np.zeros((5,), dtype = np.int) 
print(y)
# 自定义类型
z = np.zeros((2,2), dtype = [('x', 'i4'), ('y', 'i4')])  
print(z)
```

###### 1.1.3ones

```python
import numpy as np
# 默认为浮点数
x = np.ones(5) 
print(x)
# 自定义类型
x = np.ones([2,2], dtype = int)
print(x)
```

###### 1.1.4 asarray

```python
#从已有的数组创建数组。
import numpy as np 
x =  [1,2,3] 
a = np.asarray(x)  
print (a)
####
x =  (1,2,3) 
a = np.asarray(x)  
print (a)
####
x =  [(1,2,3),(4,5)] 
a = np.asarray(x)  
print (a)
```

###### 1.1.5 frombuffer

| 参数     | 描述                    |
| :----- | :-------------------- |
| buffer | 可以是任意对象，会以流的形式读入。     |
| dtype  | 返回数组的数据类型，可选          |
| count  | 读取的数据数量，默认为-1，读取所有数据。 |
| offset | 读取的起始位置，默认为0。         |

```python
#用于实现动态数组。
#buffer 是字符串的时候，Python3 默认 str 是 Unicode 类型，所以要转成 bytestring 在原 str 前加上 b。
import numpy as np 
s =  b'I Love Wmy' 
a = np.frombuffer(s, dtype =  'S1')  
print (a)
```

###### 1.1.6 fromiter

fromiter 方法从可迭代对象中建立 ndarray 对象，返回一维数组。

```python
import numpy as np 
# 使用 range 函数创建列表对象  
list=range(5)
it=iter(list)
# 使用迭代器创建 ndarray 
x=np.fromiter(it, dtype=float)
print(x)
```

###### 1.1.7 arange

| `start` | 起始值，默认为`0`                           |
| ------- | ------------------------------------ |
| `stop`  | 终止值（不包含）                             |
| `step`  | 步长，默认为`1`                            |
| `dtype` | 返回`ndarray`的数据类型，如果没有提供，则会使用输入数据的类型。 |

```python
#从数值范围创建数组
import numpy as np
x = np.arange(5)  
print (x)
###
import numpy as np
x = np.arange(10,20,2)  
print (x)
```

###### 1.1.8 linspace

linspace 函数用于创建一个一维数组，数组是一个等差数列构成的.

| 参数         | 描述                                       |
| :--------- | :--------------------------------------- |
| `start`    | 序列的起始值                                   |
| `stop`     | 序列的终止值，如果`endpoint`为`true`，该值包含于数列中      |
| `num`      | 要生成的等步长的样本数量，默认为`50`                     |
| `endpoint` | 该值为 `true` 时，数列中包含`stop`值，反之不包含，默认是True。 |
| `retstep`  | 如果为 True 时，生成的数组中会显示间距，反之不显示。            |
| `dtype`    | `ndarray` 的数据类型                          |

```python
import numpy as np
a = np.linspace(1,10,10)
print(a)

###
import numpy as np
a = np.linspace(10, 20,  5, endpoint =  False)  
print(a)
```

###### 1.1.9 logspace

logspace 函数用于创建一个于等比数列

| 参数         | 描述                                       |
| :--------- | :--------------------------------------- |
| `start`    | 序列的起始值为：base ** start                    |
| `stop`     | 序列的终止值为：base ** stop。如果`endpoint`为`true`，该值包含于数列中 |
| `num`      | 要生成的等步长的样本数量，默认为`50`                     |
| `endpoint` | 该值为 `true` 时，数列中中包含`stop`值，反之不包含，默认是True。 |
| `base`     | 对数 log 的底数。                              |
| `dtype`    | `ndarray` 的数据类型                          |

```python
import numpy as np
# 默认底数是 10
a = np.logspace(1.0,  2.0, num =  10)  
print (a)
###
```

### 2 numpy的数据类型

   numpy 支持的数据类型比 Python 内置的类型要多很多，基本上可以和 C 语言的数据类型对应上，其中部分类型对应为 Python 内置的类型。下表列举了常用 NumPy 基本类型。

| bool_      | 布尔型数据类型（True 或者 False）                   |
| ---------- | ---------------------------------------- |
| int_       | 默认的整数类型（类似于 C 语言中的 long，int32 或 int64）   |
| intc       | 与 C 的 int 类型一样，一般是 int32 或 int 64        |
| intp       | 用于索引的整数类型（类似于 C 的 ssize_t，一般情况下仍然是 int32 或 int64） |
| int8       | 字节（-128 to 127）                          |
| int16      | 整数（-32768 to 32767）                      |
| int32      | 整数（-2147483648 to 2147483647）            |
| int64      | 整数（-9223372036854775808 to 9223372036854775807） |
| uint8      | 无符号整数（0 to 255）                          |
| uint16     | 无符号整数（0 to 65535）                        |
| uint32     | 无符号整数（0 to 4294967295）                   |
| uint64     | 无符号整数（0 to 18446744073709551615）         |
| float_     | float64 类型的简写                            |
| float16    | 半精度浮点数，包括：1 个符号位，5 个指数位，10 个尾数位          |
| float32    | 单精度浮点数，包括：1 个符号位，8 个指数位，23 个尾数位          |
| float64    | 双精度浮点数，包括：1 个符号位，11 个指数位，52 个尾数位         |
| complex_   | complex128 类型的简写，即 128 位复数               |
| complex64  | 复数，表示双 32 位浮点数（实数部分和虚数部分）                |
| complex128 | 复数，表示双 64 位浮点数（实数部分和虚数部分）                |

numpy 的数值类型实际上是 dtype 对象的实例，并对应唯一的字符，包括 np.bool_，np.int32，np.float32，等等。

**数据类型对象**

数据类型对象（numpy.dtype 类的实例）用来描述与数组对应的内存区域是如何使用，它描述了数据的以下几个方面：：

- 数据的类型（整数，浮点数或者 Python 对象）
- 数据的大小（例如， 整数使用多少个字节存储）
- 数据的字节顺序（小端法或大端法）
- 在结构化类型的情况下，字段的名称、每个字段的数据类型和每个字段所取的内存块的部分
- 如果数据类型是子数组，那么它的形状和数据类型是什么。

字节顺序是通过对数据类型预先设定 **<** 或 **>** 来决定的。 **<** 意味着小端法(最小值存储在最小的地址，即低位组放在最前面)。**>** 意味着大端法(最重要的字节存储在最小的地址，即高位组放在最前面)。

```python
numpy.dtype(object, align, copy)
```

- object - 要转换为的数据类型对象
- align - 如果为 true，填充字段使其类似 C 的结构体。
- copy - 复制 dtype 对象 ，如果为 false，则是对内置数据类型对象的引用

```python
## 例子一
		import numpy as np
		# 使用标量类型
		dt = np.dtype(np.int32)
		print(dt)  ##int32
## 例子二
		import numpy as np
		# int8, int16, int32, int64 四种数据类型可以使用字符串 'i1', 'i2','i4','i8' 代替
		dt = np.dtype('i4')
		print(dt)  ##int32
## 例子三
	import numpy as np
	# 字节顺序标注
	dt = np.dtype('<i4')
	print(dt) ## int32
```

下面实例展示结构化数据类型的使用，类型字段和对应的实际类型将被创建。

```python
实例一
# 首先创建结构化数据类型
import numpy as np
dt = np.dtype([('age',np.int8)]) 
print(dt) ## [('age', 'i1')]
实例二
# 将数据类型应用于 ndarray 对象
import numpy as np
dt = np.dtype([('age',np.int8)]) 
a = np.array([(10,),(20,),(30,)], dtype = dt) 
print(a)## [(10,) (20,) (30,)]
print(type(a))  ## <class 'numpy.ndarray'>
print(a.dtype) #[('age', 'i1')]

实例三 
#修改数组的数据类型
dt = np.dtype([('age',np.int8)]) 
a = np.array([(10,),(20,),(30,)], dtype = dt) 
a.astype(np.int8)  ##array([10, 20, 30], dtype=int8)
```

numpy中的小数

```python
import numpy as np
import random
a=np.array([random.random() for i in range(10)])
print(a) #[0.465556   0.77481502 0.79937197 0.30163108 0.50223941 0.28052332
 0.22233766 0.05811927 0.1378844  0.31182252]
print(a.dtype) #float64
#保留两位
np.round(a,2) #array([0.47, 0.77, 0.8 , 0.3 , 0.5 , 0.28, 0.22, 0.06, 0.14, 0.31])
```

### 3 numpy的数组属性

| 属性               | 说明                                       |
| :--------------- | :--------------------------------------- |
| ndarray.ndim     | 秩，即轴的数量或维度的数量                            |
| ndarray.shape    | 数组的维度，对于矩阵，n 行 m 列                       |
| ndarray.size     | 数组元素的总个数，相当于 .shape 中 n*m 的值             |
| ndarray.dtype    | ndarray 对象的元素类型                          |
| ndarray.itemsize | ndarray 对象中每个元素的大小，以字节为单位                |
| ndarray.flags    | ndarray 对象的内存信息                          |
| ndarray.real     | ndarray元素的实部                             |
| ndarray.imag     | ndarray 元素的虚部                            |
| ndarray.data     | 包含实际数组元素的缓冲区，由于一般通过数组的索引获取元素，所以通常不需要使用这个属性。 |

#### 3.1 数组的形状

```python
import numpy as np
a=np.array([[1,2,3,4,5,6],[2,3,4,5,6,7]])
a.shape #(2, 6)
#改变数组的形状 是创建一个新的数组
b=a.reshape(3,4)
print(b) 
 #[[1 2 3 4]
 #[5 6 2 3]
 #[4 5 6 7]]
b.shape #(3,4)
a.shape #(2, 6)
#变为1维
c=a.reshape(12,) # array([1, 2, 3, 4, 5, 6, 2, 3, 4, 5, 6, 7])
#要变为1维 就要统计原数组的所有个数
d=a.reshape(a.shape[0]*a.shape[1],) #array([1, 2, 3, 4, 5, 6, 2, 3, 4, 5, 6, 7])
#变为1维 方式3
#flatten() 多维数组按行展开
np.arange(24).reshape(2,3,4).flatten()
```

reshape方法里面的数字必须是原数组大小的分解因式:

```python
import numpy as np
np.arange(24).reshape(2,3,5)  #直接报错  cannot reshape array of size 24 into shape (2,3,5)
np.arange(24).reshape(2,3,4) # 2最外层里面有两个数组,3次外层里有三个数组,4最里层有4个元素  
#array([[[ 0,  1,  2,  3],
#        [ 4,  5,  6,  7],
#        [ 8,  9, 10, 11]],
#
#       [[12, 13, 14, 15],
#        [16, 17, 18, 19],
#        [20, 21, 22, 23]]])
np.arange(24).reshape(2,3,4).ndim #3 3维 矩阵的秩为3
############################################################
np.arange(24).reshape(2,2,2,3) # 2最外层里面两个数组,2次外层里面有2个数组,2次次外层里面有2个数组,3最里层里面3个元素
#array([[[[ 0,  1,  2],
#         [ 3,  4,  5]],
#
#        [[ 6,  7,  8],
#         [ 9, 10, 11]]],
#
#
#       [[[12, 13, 14],
#         [15, 16, 17]],
#
#        [[18, 19, 20],
#         [21, 22, 23]]]])
#格式化
#[
#  [ 
# 	[    [ 0,  1,  2],
#         [ 3,  4,  5]
#    ],
#
#    [    [ 6,  7,  8],
#         [ 9, 10, 11]
#    ]
#  ],
#
#
#  [ [    [12, 13, 14],
#         [15, 16, 17]
#    ],
#
#    [    [18, 19, 20],
#         [21, 22, 23]
#    ]
#  ]
#]
np.arange(24).reshape(2,2,2,3).ndim #4 秩  
```

### 4 数组的计算

#### 4.1 数组与数字计算

```python
#+-*/
a=np.array([[0,2,3,4,5,6],[2,3,4,5,6,7]])
a+2 #每一个值加2
a-2 #每一个值-2
a*2#每一个值*2
a/2 #每一个值/2
a/0 # nan不是一个数字,inf无穷大
#[[nan, inf, inf, inf, inf, inf],
 #  [inf, inf, inf, inf, inf, inf]]
```

#### 4.2数组间的计算

**1:形状相同的数组之间的运算就是在对应位做运算**

**2:在NumPy中如果遇到大小不一致的数组运算，就会触发广播机制。**

 **3:numpy的广播机制**

- 让所有输入数组都向其中形状最长的数组看齐，形状中不足的部分都通过在前面加 1 补齐。

- 输出数组的形状是输入数组形状的各个维度上的最大值。

- 如果输入数组的某个维度和输出数组的对应维度的长度相同或者其长度为 1 时，这个数组能够用来计算，否则出错。

- 当输入数组的某个维度的长度为 1 时，沿着此维度运算时都用此维度上的第一组值。

  简单理解:对两个数组，分别比较他们的每一个维度（若其中一个数组没有当前维度则忽略），满足：

  - 数组拥有相同形状。
  - 当前维度的值相等。
  - 当前维度的值有一个是 1。

```python
#数组间的运算+-*/
a=np.array([[0,2,3,4,5,6],[2,3,4,5,6,7]])
b=np.array([[0,2,3,4,5,6],[2,3,4,5,6,7]])
c=np.array([[0,2,3,4,5,6]])
d=np.array([[0,2,3,4,5]])
e=np.array([[0],[1]])
print(a.shape)
a+b # 对应位置相加
a+c # 按列加
#array([[ 0,  4,  6,  8, 10, 12],
#       [ 2,  5,  7,  9, 11, 13]])
#a+d  a*d a/d a-d # 都报错
a+e # 按行加
```

```python
#如果两个数组的后缘维度(即从末尾开始算起的维度,)的轴长度相符或其中一方长度为1,则认为他们是广播兼容的,广播会在缺失或长度为1 的维度上进行
a=np.arange(6).reshape(1,2,3)
b=np.arange(3).reshape(1,3)
print(a)
print(b)
a*b #可以
########
a=np.arange(6).reshape(1,2,3)
b=np.arange(2).reshape(1,2)
print(a)
print(b)
a*b #不可以
#######
a=np.arange(6).reshape(1,2,3)
b=np.arange(4).reshape(2,2)
print(a)
print(b)
a*b #不可以
#######
a=np.arange(6).reshape(1,2,3)
b=np.arange(2).reshape(2,3) (2,1) (1,1) 都可以
print(a)
print(b)
a*b #可以
# 总结 两个数组都从右到左看能对应(位置对应)几个是几个,对应上的要不数字相等,要不其中一方为1,才能计算,否则全部报错
		(2,2,3):(2,3),(1,3),(1,1),(2,1),(1,2,3),(2,2,1),... #可以
    (2,2,3):(2,2),(2,2,4),(1,2,4),(2,3,4),...#不可以
```

### 5 numpy读取数据

Numpy 可以读写磁盘上的文本数据或二进制数据。

NumPy 为 ndarray 对象引入了一个简单的文件格式：**npy**。

npy 文件用于存储重建 ndarray 所需的数据、图形、dtype 和其他信息。

常用的 IO 函数有：

- load() 和 save() 函数是读写文件数组数据的两个主要函数，默认情况下，数组是以未压缩的原始二进制格式保存在扩展名为 .npy 的文件中。

- savez() 函数用于将多个数组写入文件，默认情况下，数组是以未压缩的原始二进制格式保存在扩展名为 .npz 的文件中。

- loadtxt() 和 savetxt() 函数处理正常的文本文件(.txt .csv等)

  ```python
  savetxt(fileName,data)
  #参数：
  #fileName:保存文件路径和名称   
  #data:需要保存的数据 
  ```

```python
  #loadtxt()
  import numpy as np
  #读取csv文件
  data = np.loadtxt('./data/data.csv',delimiter=',',skiprows=1, usecols=(2,3))
  print(data)
  print(data.shape)
```

参数:

| Frame      | 被读取的文件名（文件的相对地址或者绝对地址） |
| ---------- | ---------------------- |
| dtype      | 指定读取后数据的数据类型           |
| comments   | 跳过文件中指定参数开头的行（即不读取）    |
| delimiter  | 指定读取文件中数据的分割符          |
| converters | 对读取的数据进行预处理            |
| skiprows   | 选择跳过的行数                |
| usecols    | 指定需要读取的列               |
| unpack     | 选择是否将数据进行向量输出          |
| encoding   | 对读取的文件进行预编码            |
|            |                        |

### 6 索引与切片

**行和列的下标都从0开始**

**1:**ndarray对象的内容可以通过索引或切片来访问和修改，与 Python 中 list 的切片操作一样

**2:**ndarray 数组可以基于 0 - n 的下标进行索引，切片对象可以通过内置的 slice 函数，并设置 start, stop 及 step 参数进行，从原数组中切割出一个新数组。

#### 6.1 单维数组

```python
import numpy as np
a = np.arange(10)
s = slice(2,7,2)   # 从索引 2 开始到索引 7 停止，间隔为2
print (a[s])
############
#我们也可以通过冒号分隔切片参数 start:stop:step 来进行切片操作：
import numpy as np
a = np.arange(10)  
b = a[2:7:2]   # 从索引 2 开始到索引 7 停止，间隔为 2
print(b)
#冒号 : 的解释：如果只放置一个参数，如 [2]，将返回与该索引相对应的单个元素。如果为 [2:]，表示从该索引开始以后的所有项都将被提取。如果使用了两个参数，如 [2:7]，那么则提取两个索引(不包括停止索引)之间的项。
######################
import numpy as np
a = np.arange(10)  # [0 1 2 3 4 5 6 7 8 9]
b = a[0] 
print(b) # 0
```

#### 6.2 多维数组索引提取

```python
import numpy as np
a = np.array([[1,2,3],[3,4,5],[4,5,6]])
print(a)
# 从某个索引处开始切割
print('从数组索引 a[1:] 处开始切割')
print(a[1:])
#[[1 2 3]
# [3 4 5]
# [4 5 6]]
#从数组索引 a[1:] 处开始切割
#[[3 4 5]
# [4 5 6]]
#######################################################################

#切片还可以包括省略号 …，来使选择元组的长度与数组的维度相同。 如果在行位置使用省略号，它将返回包含行中元素的 ndarray。等效于:
import numpy as np
a = np.array([[1,2,3],[3,4,5],[4,5,6]])  
print (a[...,1])   # 第2列元素 [2 4 5]
print (a[1,...])   # 第2行元素 [3 4 5]
print (a[...,1:])  # 第2列及剩下的所有元素
#[[2 3]
# [4 5]
# [5 6]]
print(a[:,[0,2]])  # 取不连续的列 print(a[...,[0,2]]) 结果同
#取行和列 1行,2列  
print(a[0,1]) # 2
#取多行多列,取第1行到2行,第1列到第2列
print(a[0:2,0:2]) #1行的下标0,...
#[[1 2]
# [3 4]]

# 取多个不相邻的点
print(a[[0,2],[0,2]]) # [1 6] 1对应(0,0) 6对应(2,2)
```

### 7 numpy 中的数值的修改

#### 7.1 修改行列的值

```python
#修改1行2列的值
import numpy as np
a = np.array([[1,2,3],[3,4,5],[4,5,6]])  
a[0,1]=11
print(a)
#[[ 1 11  3]
# [ 3  4  5]
# [ 4  5  6]]
```

#### 7.2 修改带条件的值

```python
# 将数组中小于5的值改为3
import numpy as np
a = np.array([[1,2,3],[3,4,5],[4,5,6]]) 
a[a<5]=3 #布尔索引
print(a) #修改成功
a<5 # 变为bool值
#array([[False, False, False],
#       [False, False, False],
#       [False, False, False]])

####################分割#####################
#将大于5的改为6,小于5的改为4
import numpy as np
a = np.array([[1,2,3],[3,4,5],[4,5,6]]) 
b=np.where(a<5,4,6)
print(a)
print(b)
#[[1 2 3]
# [3 4 5]
# [4 5 6]]
#[[4 4 4]
# [4 4 6]
# [4 6 6]]
######################分割#####################
#将小于4的替换为4,大于6的改为6
import numpy as np
a = np.array([[1,2,3],[3,4,5],[4,5,6]]) 
a.clip(4,6)
```

### 8 数组的拼接与分割与行列交换

#### 8.1 拼接与分割

**拼接和分割是相反的**

**拼接前应注意每一行和每一列代表的意义应该相同**

```python
#竖直拼接
np.vstack(t1,t2)
#水平拼接
np.hstack(t1,t2)
```

#### 8.2 行列交换

```python
#行交换 第1,2行交换
import numpy as np
a = np.array([[1,2,3],[3,4,5],[4,5,6]]) 
a[[0,1],:]=a[[1,0],:]
print(a)
#列交换 第1,2列交换
import numpy as np
a = np.array([[1,2,3],[3,4,5],[4,5,6]]) 
a[:,[0,1]]=a[:,[1,0]]
print(a)
```

### 9 数组的迭代

#### 9.1 内部循环

```python
for x in np.nditer(a, order='F'):Fortran order，即是列序优先；
for x in np.nditer(a.T, order='C'):C order，即是行序优先；
 # a.T 数组的转置
```

#### 9.2 外部循环

```python
import numpy as np 
a = np.arange(0,60,5) 
a = a.reshape(3,4)  
print ('原始数组是：')
print (a)
print ('\n')
print ('修改后的数组是：')
for x in np.nditer(a, flags =  ['external_loop'], order =  'F'):  
   print (x, end=", " )
```

flags参数

| 参数              | 描述                      |
| :-------------- | :---------------------- |
| `c_index`       | 可以跟踪 C 顺序的索引            |
| `f_index`       | 可以跟踪 Fortran 顺序的索引      |
| `multi-index`   | 每次迭代可以跟踪一种索引类型          |
| `external_loop` | 给出的值是具有多个值的一维数组，而不是零维数组 |

#### 9.3 广播迭代

若是两个数组是可广播的,则可以同时迭代他们

```python
import numpy as np 
 
a = np.arange(0,60,5) 
a = a.reshape(3,4)  
print  ('第一个数组为：')
print (a)
print  ('\n')
print ('第二个数组为：')
b = np.array([1,  2,  3,  4], dtype =  int)  
print (b)
print ('\n')
print ('修改后的数组为：')
for x,y in np.nditer([a,b]):  
    print ("%d:%d"  %  (x,y), end=", " )
```

### 10 数组的操作

```text
1.修改数组形状
2.翻转数组
3.修改数组维度
4.连接数组
5.分割数组
6.数组元素的添加与删除
```

#### 10.1 修改数组的形状

| 函数        | 描述                        |
| :-------- | :------------------------ |
| `reshape` | 不改变数据的条件下修改形状             |
| `flat`    | 数组元素迭代器                   |
| `flatten` | 返回一份数组拷贝，对拷贝所做的修改不会影响原始数组 |
| `ravel`   | 返回展开数组                    |

**reshape**

numpy.reshape 函数可以在不改变数据的条件下修改形状，格式如下： numpy.reshape(arr, newshape, order='C')

- `arr`：要修改形状的数组
- `newshape`：整数或者整数数组，新的形状应当兼容原有形状
- order：'C' -- 按行，'F' -- 按列，'A' -- 原顺序，'k' -- 元素在内存中的出现顺序。

**flat**

数组元素迭代器

```python
import numpy as np
 
a = np.arange(9).reshape(3,3) 
print ('原始数组：')
for row in a:
    print (row)
 
#对数组中每个元素都进行处理，可以使用flat属性，该属性是一个数组元素迭代器：
print ('迭代后的数组：')
for element in a.flat:
    print (element)
```

**flatten**

flatten 返回一份数组拷贝，对拷贝所做的修改不会影响原始数组,降维操作

```
ndarray.flatten(order='C')
```

参数说明：

- order：'C' -- 按行，'F' -- 按列，'A' -- 原顺序，'K' -- 元素在内存中的出现顺序。

```python
import numpy as np
 
a = np.arange(8).reshape(2,4)
 
print ('原数组：')
print (a)
print ('\n')
# 默认按行
 
print ('展开的数组：')
print (a.flatten())
print ('\n')
 
print ('以 F 风格顺序展开的数组：')
print (a.flatten(order = 'F'))
```

**ravel**

numpy.ravel() 展平的数组元素，顺序通常是"C风格"，返回的是数组视图（view，有点类似 C/C++引用reference的意味），修改会影响原始数组。

该函数接收两个参数：

```
numpy.ravel(a, order='C')
```

参数说明：

- order：'C' -- 按行，'F' -- 按列，'A' -- 原顺序，'K' -- 元素在内存中的出现顺序。

```python
import numpy as np
 
a = np.arange(8).reshape(2,4)
 
print ('原数组：')
print (a)
print ('\n')
 
print ('调用 ravel 函数之后：')
print (a.ravel())
print ('\n')
 
print ('以 F 风格顺序调用 ravel 函数之后：')
print (a.ravel(order = 'F'))
```

#### 10.2 翻转数组

```
1:transpose	对换数组的维度
2:ndarray.T	和 self.transpose() 相同
3:rollaxis	向后滚动指定的轴
4:swapaxes	对换数组的两个轴
```

**numpy.transpose**

numpy.transpose 函数用于对换数组的维度，格式如下：

```
numpy.transpose(arr, axes)
```

参数说明:

- `arr`：要操作的数组

- `axes`：整数列表，对应维度，通常所有维度都会对换。

  ```python
  import numpy as np
   
  a = np.arange(12).reshape(3,4)
   
  print ('原数组：')
  print (a )
  print ('\n')
   
  print ('对换数组：')
  print (np.transpose(a))
  ```

  **numpy.rollaxis**

  numpy.rollaxis 函数向后滚动特定的轴到一个特定位置，格式如下：

  ```
  numpy.rollaxis(arr, axis, start)
  ```

  参数说明：

  - `arr`：数组
  - `axis`：要向后滚动的轴，其它轴的相对位置不会改变
  - `start`：默认为零，表示完整的滚动。会滚动到特定位置。

```python
import numpy as np
 
# 创建了三维的 ndarray
a = np.arange(8).reshape(2,2,2)
print ('原数组：')
print (a)
print ('获取数组中一个值：')
print(np.where(a==6))   
print(a[1,1,0])  # 为 6
print ('\n')
# 将轴 2 滚动到轴 0（宽度到深度）
print ('调用 rollaxis 函数：')
b = np.rollaxis(a,2,0)
print (b)
# 查看元素 a[1,1,0]，即 6 的坐标，变成 [0, 1, 1]
# 最后一个 0 移动到最前面
print(np.where(b==6))   
print ('\n')
 
# 将轴 2 滚动到轴 1：（宽度到高度）
 
print ('调用 rollaxis 函数：')
c = np.rollaxis(a,2,1)
print (c)
# 查看元素 a[1,1,0]，即 6 的坐标，变成 [1, 0, 1]
# 最后的 0 和 它前面的 1 对换位置
print(np.where(c==6))   
print ('\n')
```

**numpy.swapaxes**

numpy.swapaxes 函数用于交换数组的两个轴，格式如下：

```
numpy.swapaxes(arr, axis1, axis2)
```

- `arr`：输入的数组
- `axis1`：对应第一个轴的整数
- `axis2`：对应第二个轴的整数

#### 10.3 修改数组维度

| 维度             | 描述            |
| :------------- | :------------ |
| `broadcast`    | 产生模仿广播的对象     |
| `broadcast_to` | 将数组广播到新形状     |
| `expand_dims`  | 扩展数组的形状       |
| `squeeze`      | 从数组的形状中删除一维条目 |

#### 10.4连接数组

| 函数            | 描述              |
| :------------ | :-------------- |
| `concatenate` | 连接沿现有轴的数组序列     |
| `stack`       | 沿着新的轴加入一系列数组。   |
| `hstack`      | 水平堆叠序列中的数组（列方向） |
| `vstack`      | 竖直堆叠序列中的数组（行方向） |

#### 10.5 分割数组

| 函数       | 数组及操作               |
| :------- | :------------------ |
| `split`  | 将一个数组分割为多个子数组       |
| `hsplit` | 将一个数组水平分割为多个子数组（按列） |
| `vsplit` | 将一个数组垂直分割为多个子数组（按行） |

#### 10.6 数组元素的添加和删除

| 函数       | 元素及描述                |
| :------- | :------------------- |
| `resize` | 返回指定形状的新数组           |
| `append` | 将值添加到数组末尾            |
| `insert` | 沿指定轴将值插入到指定下标之前      |
| `delete` | 删掉某个轴的子数组，并返回删除后的新数组 |
| `unique` | 查找数组内的唯一元素           |

#### 10.7 其他方法

```python
#获取极值
np.argmax(t,axis=0)
np.argmin(t,axis=1)
#创建全为0的数组
np.zeros((3,4))
#创建全为1的数组
np.ones((3,4))
#创建一个对角线为1的正方形数组(方阵)
np.eye(3)
#生成随机数
np.random.rand() #均匀分布的随机数
np.randn() #标准正态分布的随机数,浮点数,均值0,标准差1
randint(low,high,(shape)) #从给定的上下限范围内取随机整数,形状shape
uniform(low,high,(size)) #产生具有均匀分布的数组
normall(loc,scale,(size)) #指定正态分布中随机抽取样本,分布中心loc(概率分布均值),标准差(scale),形状是size
seed(s) #随机数种子,s是给定值,可以设定相同的随机数种子,可以每次生成相同的随机数
#判定数组中NAN(NOT A NUMBER) 的个数
np.count_nansero(t!=t)
#判定一个数字是否为nan 并将nan赋值为0 nan和任何值计算都为nan
t[np.isnan(t)]=0
```

### 11 numpy 的函数

#### 11.1 统计函数

```python
#和:t.sum()
#均值:t.mean()
#中值:t.median()
#大:t.max()
#小:t.min()
#极值:np.ptp(t,axis=None)
#标准差:t.std()  :::;std = sqrt(mean((x - x.mean())**2))
#方差:np.var()  标准差是方差的平方根
#分位:np.percentile(a, q, axis) a: 输入数组 q: 要计算的百分位数，在 0 ~ 100 之间 axis: 沿着它计算百分位数的轴
#numpy.amin() 用于计算数组中的元素沿指定轴的最小值。
#numpy.amax() 用于计算数组中的元素沿指定轴的最大值。
#加权平均值:np.average(t)
		函数根据在另一个数组中给出的各自的权重计算数组中元素的加权平均值。
		该函数可以接受一个轴参数。 如果没有指定轴，则数组会被展开。
		加权平均值即将各数值乘以相应的权数，然后加总求和得到总体值，再除以总的单位数。
		考虑数组[1,2,3,4]和相应的权重[4,3,2,1]，通过将相应元素的乘积相加，并将和除以权重的和，来计算加权平均值。
    加权平均值 = (1*4+2*3+3*2+4*1)/(4+3+2+1)
```

```python
#加权平均值
import numpy as np 
a = np.array([1,2,3,4])  
print ('我们的数组是：')
print (a)
print ('\n')
print ('调用 average() 函数：')
print (np.average(a))
print ('\n')
# 不指定权重时相当于 mean 函数
wts = np.array([4,3,2,1])  
print ('再次调用 average() 函数：')
print (np.average(a,weights = wts))
print ('\n')
# 如果 returned 参数设为 true，则返回权重的和  
print ('权重的和：')
print (np.average([1,2,3,  4],weights =  [4,3,2,1], returned =  True))
```

### 12 numpy的线性代数

NumPy 提供了线性代数函数库 **linalg**，该库包含了线性代数所需的所有功能，可以看看下面的说明：

| 函数            | 描述               |
| :------------ | :--------------- |
| `dot`         | 两个数组的点积，即元素对应相乘。 |
| `vdot`        | 两个向量的点积          |
| `inner`       | 两个数组的内积          |
| `matmul`      | 两个数组的矩阵积         |
| `determinant` | 数组的行列式           |
| `solve`       | 求解线性矩阵方程         |
| `inv`         | 计算矩阵的乘法逆矩阵       |