---
title: Pandas
tags:
  - Python
  - 数据分析与可视化
categories: 数据分析与可视化
encrypt: 
enc_pwd: 
abbrlink: 55940
date: 2020-11-22 20:50:44
summary_img:
---

## 一 pandas概述

​	**Pandas** 是 [Python](https://www.python.org/) 的核心数据分析支持库，提供了快速、灵活、明确的数据结构，旨在简单、直观地处理关系型、标记型数据,

   Pandas 适用于处理以下类型的数据：

- 与 SQL 或 Excel 表类似的，含异构列的表格数据;
- 有序和无序（非固定频率）的时间序列数据;
- 带行列标签的矩阵数据，包括同构或异构型数据;
- 任意其它形式的观测、统计数据集, 数据转入 Pandas 数据结构时不必事先标记。

​     Pandas 的主要数据结构是 [Series](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.Series.html#pandas.Series)（一维数据）与 [DataFrame](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.DataFrame.html#pandas.DataFrame)（二维数据），这两种数据结构足以处理金融、统计、社会科学、工程等领域里的大多数典型用例

   Pandas 基于 [NumPy](https://www.numpy.org/) 开发(所以有些numpy的操作pandas同样适用)，可以与其它第三方科学计算支持库完美集成。

优点:

```
1.处理浮点与非浮点数据里的缺失数据，表示为 NaN；
2.大小可变：插入或删除 DataFrame 等多维对象的列；
3.自动、显式数据对齐：显式地将对象与一组标签对齐，也可以忽略标签，在 Series、DataFrame 计算时自动与数据对齐；
4.强大、灵活的分组（group by）功能：拆分-应用-组合数据集，聚合、转换数据；
5.把 Python 和 NumPy 数据结构里不规则、不同索引的数据轻松地转换为 DataFrame 对象；
6.基于智能标签，对大型数据集进行切片、花式索引、子集分解等操作；
7.直观地合并（merge）、**连接（join）**数据集；
8.灵活地重塑（reshape）、**透视（pivot）**数据集；
9.轴支持结构化标签：一个刻度支持多个标签；
10.成熟的 IO 工具：读取文本文件（CSV 等支持分隔符的文件）、Excel 文件、数据库等来源的数据，利用超快的 HDF5 格式保存 / 加载数据；
11.时间序列：支持日期范围生成、频率转换、移动窗口统计、移动窗口线性回归、日期位移等时间序列功能。
```

处理数据一般分为几个阶段：数据整理与清洗、数据分析与建模、数据可视化与制表，Pandas 是处理数据的理想工具。

## 二 数据结构

| 维数   | 名称        | 描述                |
| ---- | --------- | ----------------- |
| 1    | Series    | 带标签的一维同构数组        |
| 2    | DataFrame | 带标签的，大小可变的，二维异构表格 |

​	Pandas 数据结构就像是低维数据的容器。比如，DataFrame 是 Series 的容器，Series 则是标量的容器。使用这种方式，可以在容器中以字典的形式插入或删除对象。

​	此外，通用 API 函数的默认操作要顾及时间序列与截面数据集的方向。多维数组存储二维或三维数据时，编写函数要注意数据集的方向，这对用户来说是一种负担；如果不考虑 C 或 Fortran 中连续性对性能的影响，一般情况下，不同的轴在程序里其实没有什么区别。Pandas 里，轴的概念主要是为了给数据赋予更直观的语义，即用“更恰当”的方式表示数据集的方向。这样做可以让用户编写数据转换函数时，少费点脑子。

​	处理 DataFrame 等表格数据时，**index**（行）或 **columns**（列）比 **axis 0** 和 **axis 1** 更直观。用这种方式迭代 DataFrame 的列，代码更易读易懂：

```python
for col in df.columns:
    series = df[col]
    # do something with series
```

Pandas 所有数据结构的值都是可变的，但数据结构的大小并非都是可变的，比如，Series 的长度不可改变，但 DataFrame 里就可以插入列。

Pandas 里，绝大多数方法都不改变原始的输入数据，而是复制数据，生成新的对象。 一般来说，原始输入数据**不变**更稳妥。

## 三 基本使用

### 1 Series

有些操作与numpy一致

```python
import pandas as pd
import numpy as np
pd.Series(np.arange(4),index=list("abcd")) #idex的长度和Series的长度保持一致  index 设置索引的格式
dict_a={"name":"fei","age":10,"te":123} #字典
pd.Series(dict_a)
#name    fei
#age      10
#te      123
#dtype: object
```

series本质是由两个数组构成,一个数组构成建(index),一个构成数组的值(values) 建-->值

```python
import pandas as pd
df=pd.read_csv("user_info.csv").to_numpy()
print(type(df))
t = df[0]
pd.Series(t)
```

### 2 DataFrame

DF既有行索引(index或axis=0),又有列索引(column或axis=1)

```python
import pandas as pd 
import numpy as np
pd.DataFrame(np.arange(12).reshape((3,4)),index=list("abc"),columns=list("qwer"))
#传入字典
dict_a={"name":"fei","age":10,"te":123}
pd.DataFrame(dict_a) #报错
######
dict_a={"name":["fei"],"age":[10],"te":[123]}
pd.DataFrame(dict_a) #true
```

#### 2.1**基础属性**

```python
df.shape #行列
df.dtypes #列数据类型
df.ndim #数据维度
df.index #行索引
		
df.columns #列索引
df.values # 值
```

#### 2.2**df的整体情况查看**

```python
df.head(5) #头部几行 默认5
df.tail(3) #尾部几行 默认5
df.info() #df的相关信息
df.describe() #快速统计结果:计数,均值,标准差,分位数,极值等
```

#### 2.3**df的函数**

```python
#排序
df.sort_values(by="列名",ascending=False) #ascending:升降序
#字符串操作
df["name"].str.split()
....
#数组的合并 join 默认行行索引相同的数据合并在一起
df.join(df2) #
#数组的合并 2 merge  按照指定的列把数据按一定方式合并在一起
df.merge(df2,on="o",how="inner|outer|left|right")
#where条件
df=pd.read_csv("user_info.csv")
df2=df.where(df["user_id"]>462176) #会将<462176 的行数据变为nan
df2.dropna() #删除nan
#分组聚合
# 分组
dfgb=df.groupby(by="column_name") #返回DataFrameGroupBy对象 这个对象可以做聚合操作
for i in dfgb:
    print(i) #打印每个分组的所有数据为元组
for i,j in dfgb:
    print(i) #组信息 0-24 40-等
    print(j) #组的数据    #这样将每一个分组组成一个df
 # 多分组
df.groupby(by=[df[""],df[""],....])
df.groupby(by=["age","user_id"]) #两者同
# 聚合
dfgb.count() # sum() max() min() .....
# 取某一列聚合
dfgb["user_id"].count() #等效于dfgb.count()["user_id"]
```

#### 2.4 获取行列数据

**获取行列数据实例**

```python
#取行列数据
#方括号写数字 表示对行操作,方括号写字符串表示对列操作
df[:20]["user_id"] #取user_id列前20行
#取行列数据2 ---loc和iloc
# loc 通过index(标签索引)获取行数据
# iloc 通过行号(位置)获取数据
####################################################实例1################################################

# 利用loc、iloc提取行数据
import numpy as np
import pandas as pd
#创建一个Dataframe
data=pd.DataFrame(np.arange(16).reshape(4,4),index=list('abcd'),columns=list('ABCD'))
In[1]: data
Out[1]: 
    A   B   C   D
a   0   1   2   3
b   4   5   6   7
c   8   9  10  11
d  12  13  14  15
 
#取索引为'a'的行
In[2]: data.loc['a']
Out[2]:
A    0
B    1
C    2
D    3
#取第一行数据，索引为'a'的行就是第一行，所以结果相同
In[3]: data.iloc[0]
Out[3]:
A    0
B    1
C    2
D    3
####################################################实例2################################################
#利用loc、iloc提取列数据
In[4]:data.loc[:,['A']] #取'A'列所有行，多取几列格式为 data.loc[:,['A','B']]
Out[4]: 
    A
a   0
b   4
c   8
d  12
In[5]:data.iloc[:,[0]] #取第0列所有行，多取几列格式为 data.iloc[:,[0,1]]
Out[5]: 
    A
a   0
b   4
c   8
d  12
####################################################实例3################################################
#利用loc、iloc提取指定行、指定列数据
In[6]:data.loc[['a','b'],['A','B']] #提取index为'a','b',列名为'A','B'中的数据
Out[6]: 
   A  B
a  0  1
b  4  5
In[7]:data.iloc[[0,1],[0,1]] #提取第0、1行，第0、1列中的数据
Out[7]: 
   A  B
a  0  1
b  4  5
####################################################实例4################################################
#利用loc、iloc提取所有数据
In[8]:data.loc[:,:] #取A,B,C,D列的所有行
Out[8]: 
    A   B   C   D
a   0   1   2   3
b   4   5   6   7
c   8   9  10  11
d  12  13  14  15
In[9]:data.iloc[:,:] #取第0,1,2,3列的所有行
Out[9]: 
    A   B   C   D
a   0   1   2   3
b   4   5   6   7
c   8   9  10  11
d  12  13  14  15
####################################################实例5################################################
#利用loc函数，根据某个数据来提取数据所在的行
In[10]: data.loc[data['A']==0] #提取data数据(筛选条件: A列中数字为0所在的行数据)
Out[10]: 
   A  B  C  D
a  0  1  2  3
```

**布尔索引**

```python
#根据条件获取数据
df[df["userid"]>0]
```

#### 2.5 缺失值处理

```python
#nan或0
pd.isnull(df)
pd.notnull(df)
#处理
dropna(axis=0,how='any',inplace=False) #删除
t.fillna(t.mean) #填充
#计算平均值nan不会参与计算但0会
#处理为0的数据
t[t==0]=np.nan
```

#### 2.6 索引和复合索引

```python
#简单索引
df.index # 获取索引
df.index=['x','y'] #指定索引
df.reindex(list("abcd") #重设索引
df.set_index("column_name",drop=False)  #指定某一列为索引
df.set_index("column_name").index.unique() #返回index的唯一值
#复合索引
 df=pd.DataFrame({'a':range(7),'b':range(7,0,-1),'c':['one','one','one','two','two','two','two'],'d':list("hjklmno")})
df.set_index(['c','d']) 
#		   a	b
# c	 d		
#one h	0	7
#		 j	1	6
#	k	 2	5
#two l	3	4
#		 m	4	3
#		 n	5	2
#		 o	6	1 
# 交换索引的位置
df=pd.DataFrame({'a':range(7),'b':range(7,0,-1),'c':['one','one','one','two','two','two','two'],'d':list("hjklmno")})
d=df.set_index(['d','c'])['a']
d.index
d.swaplevel()
# 从复合索引取值
  # series
    s1['a']['b'] or s1["a","b"]
  # df
    df.loc["a"].loc["b"]
```

### 3 读取外部数据源

```python
import pandas as pd
df=pd.read_csv("user_info.csv")  #返回都是DataFrame
#pd.read_sql(sql_sentence,connection) sql_sentence:sql语句,connection:连接
#pd.read_excel()
.....
```

### 4 时间序列

#### 4.1 生成时间范围

```python
import pandas as pd
pd.date_range(start=,end=,periods=,freq='d')  # end 和periods 不可共存
# start,end和freq配合能生成范围内频率为freq 的一组时间索引
#start,periods和freq配合能生成从start开始频率为freq 的periods个时间索引
#start end 格式2021-03-01,20210301,2021/03/01 ...
###### 天
import pandas as pd
pd.date_range(start='2021-02-01',end='2021-03-01',freq='d') # freq :10d,d,m,w...
######
import pandas as pd
pd.date_range(start='2021/02/01',periods=10,freq='m') # periods 生成时间的个数
##########
pd.to_datetime(df["timeStamp"],format="") # python的时间格式化
#example
pd.to_datetime("20210304",format="%Y-%m-%d %H:%M:%S") 
#Timestamp('2021-03-04 00:00:00')
```

**freq**

| 别名    | 说明        |      |
| ----- | --------- | ---- |
| D     | 日         |      |
| B     | 工作日       |      |
| H     | 小时        |      |
| T或MIN | 分         |      |
| S     | 秒         |      |
| L或MS  | 毫秒        |      |
| U     | 微秒        |      |
| M     | 月的最后一个日期  |      |
| BM    | 每月最后一个工作日 |      |
| MS    | 每月第一日     |      |
| BMS   | 每月第一个工作日  |      |

#### 4.2 df中使用时间序列

```python
import pandas as pd
import numpy as np
index=pd.date_range(start='2021/02/01',periods=10,freq='m') 
pd.DataFrame(np.random.rand(10),index=index)
```

#### 4.3 重采样

是将时间序列从一个频率转化为另一个频率进行处理的过程,将高频率转化为低频率的为降采样,低频率转化为高频率为升采样

```python
# resamole
import pandas as pd
import numpy as np
index=pd.date_range(start='2021/02/01',periods=100,freq='d') 
df=pd.DataFrame(np.random.rand(100),index=index)s
df.resample("m").mean()
```



