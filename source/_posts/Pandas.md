---
title: Pandas
tags: 数据分析
categories: 数据分析
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

```python
import pandas as pd
import numpy as np
pd.read_csv()
pd.read_excel()
pd.read_html() # 可以简单的爬虫 爬取表格等
```

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

## 四 常用

创建数据集

```python
import numpy as np
import pandas as pd

boolean=[True,False]
gender=["男","女"]
color=["white","black","yellow"]
data2=pd.DataFrame({
    "height":np.random.randint(150,190,100),
    "weight":np.random.randint(40,90,100),
    "smoker":[boolean[x] for x in np.random.randint(0,2,100)],
    "gender":[gender[x] for x in np.random.randint(0,2,100)],
    "age":np.random.randint(15,90,100),
    "color":[color[x] for x in np.random.randint(0,len(color),100) ]
}
)
data2
```

#### 1 新加列/行

```python
#新加列
data2['bmi'] = (data2['weight'] / ((data2['height']/100) *(data2['height']/100))).map(int)
data2
#新加行
new_index=pd.DataFrame({'height':178,'weight':74,'smoker':False,'gender':'女','age':17,'color':'white','bmi':23},index=['e'])
data=data.append(new_index,ignore_index=True) #ignore_index忽略元行索引,新从0开始
data
```

#### 2 map函数

如将gender列的男:1,女:0

```python
#1  lanbda表达式
data2['gender']=data2['gender'].map(lambda x:1 if x == "男" else 0)
data2
#2
data2["gender"] = data2["gender"].map({"男":1, "女":0})
#3
def gender_map(x):
    gender = 1 if x == "男" else 0
    return gender
data["gender"] = data["gender"].map(gender_map)
```

#### 3 apply函数

传入函数,进行计算

```python
# 1 新加列计算bmi
def BMI(series):
    weight = series["weight"]
    height = series["height"]/100
    BMI = weight/height**2
    return BMI
data2["BMI"] = data2.apply(BMI,axis=1) # axis=1 行  axis=0 列

# 2 计算列的和
data2[['whight','age']].apply(np.sum,axis=0)
weight     6608
age        5122
dtype: int64

#全部保留两位小数
df.applymap(lambda x:"%.2f" % x)
```



#### 4 行列重命名

**axis=0 :列  axis=1 :行**

**inplace : 是否映射会原dataframe**

```python
# 列重命名
data2.rename({0:'h'},axis=0,inplace=True)
data2
# 行重命名
data2.rename({'height':'h'},axis=1,inplace=True)
data2
```

#### 5 删除列

```python
data2.drop(["bmi"],axis=1,inplace=True)
```

#### 6 分组groupby 和agg,transform,apply函数结合使用

```python
group = data2.groupby('color')
list(group)
```

转换成列表的形式后，可以看到，列表由三个元组组成，每个元组中，第一个元素是组别（这里是按照`color`进行分组，所以最后分为了"white","black","yellow"）

`groupby`的过程就是将原有的`DataFrame`按照`groupby`的字段（这里是`color`），划分为若干个`分组DataFrame`，被分为多少个组就有多少个`分组DataFrame`。**所以说，在`groupby`之后的一系列操作（如`agg`、`apply`等），均是基于`子DataFrame`的操作**

**agg**

聚合操作是`groupby`后非常常见的操作,列出了常用的

```
min,max,sum,mean,median,std,var,count,.....
```

```python
# 求均值
group.agg('mean').reset_index()
#针对不同的列求不同的值
group.agg({'weight':'max','smoker':'count'}).reset_index()

color	weight	smoker
black	87	28
white	85	26
yellow	88	46
```

**transfrom**

**就类似于窗口函数**

```python
#会对每一条数据都附加一列 值为改数据所在组的平均值
group['weight'].transform('mean')

0     64.730769
1     66.326087
2     66.326087
3     66.326087
4     66.326087
        ...    
95    64.730769
96    66.928571
97    66.928571
98    64.730769
99    66.326087

#每个组产生一条,
group['weight'].agg('mean')

color
black     66.928571
white     64.730769
yellow    66.326087
```

**应用可以像窗口函数似的给每一条数据附加一个聚合值**

```python
data2['avg_weight'] = group['weight'].transform('mean')
data2

	h	weight	smoker	gender	age	color	avg_weight
0	178	74	True	0	28	white	64.730769
1	157	81	True	0	33	yellow	66.326087
2	169	77	True	0	37	yellow	66.326087
3	174	60	False	1	88	yellow	66.326087
4	174	88	False	0	65	yellow	66.326087
...	...	...	...	...	...	...	...
95	173	75	True	0	58	white	64.730769
96	150	54	True	0	73	black	66.928571
97	178	82	False	0	45	black	66.928571
98	184	62	False	0	37	white	64.730769
99	166	75	False	0	62	yellow	66.326087

# 添加值2
maxage_dict=group['age'].max().to_dict()
data2['max_age'] = data2['color'].map(maxage_dict)
data2
```

**apply**

对于`groupby`后的`apply`，以分组后的`子DataFrame`作为参数传入指定函数的，基本操作单位是`DataFrame`，而未分组的`apply`的基本操作单位是`Series`;

```python
# 如获取每个颜色中年龄最大的那一条数据
def get_oldest_staff(x):
    df = x.sort_values(by = 'age',ascending=True)
    return df.iloc[-1,:]
oldest_staff = group.apply(get_oldest_staff)
oldest_staff.drop(['color'],axis=1,inplace=True)
oldest_staff.reset_index()

color	h	weight	smoker	gender	age	avg_weight
black	150	60	True	0	87	66.928571
white	153	72	False	1	87	64.730769
yellow	170	82	False	0	88	66.326087
```

#### 7 数据拼接

merge相当于sql中的join

merge有4中类型:

```
inner  left  right  outer
```

```python
# inner 内连接 取交集
df_1.merge(df_2,how='inner',on='userid')
# left  right 以左表或右表为基准,两个可以相互转化
df_1.merge(df_2,how='left',on='userid')
# outer 取并集 对于没有匹配到的地方，使用缺失值NaN进行填充
df_1.merge(df_2,how='outer',on='userid')

# pd形式
newdf=pd.merge(df[['a', 'b', 'c', 'd']]
        ,df2
        ,how='inner'
        ,on=['user_id','cal_dt'])
```

#### 8 zip

将数据组合为元祖

```python
a = [1,2,3,4]
b = [5,6,7,8]
zz= zip(a,b)

print(list(zz))
zz2= zip(a,b)
print(list(zip(*zz2)))

[(1, 5), (2, 6), (3, 7), (4, 8)]
[(1, 2, 3, 4), (5, 6, 7, 8)]
```

```python
# 计算百分比
a = [1,2,3,4]
b = [5,6,7,8]
zz= zip(a,b)
aa=list(map(lambda x: {"value":x[0],"percent":x[0]/(x[0]+x[1])}, zip(a,b)))
aa

[{'value': 1, 'percent': 0.16666666666666666},
 {'value': 2, 'percent': 0.25},
 {'value': 3, 'percent': 0.3},
 {'value': 4, 'percent': 0.3333333333333333}]
```

#### 9 行列转换

**9.1 行转列**

```
A  a 12
A  b 13
A  c 14
B  a 34
B  b 56
B  c 67
C  a 90
C  b 80
C  c 95
转为
	 a   b  c
A  12  13 14
B  34  56 67
C  90  80 95
```

```python
data=pd.DataFrame([['A','a',1],['A','b',2],['A','c',3], ['B','a',4],['B','b',5],['B','c',6],  ['C','a',7],['C','b',8],['A','c',9]]
                 ,columns=['姓名','科目','分数']
                 )
data
# 变为有2级索引的series 1级索引姓名 2级索引 科目
data_new=data.set_index(['姓名','科目'])["分数"]
# 调用具有二级索引的Series的unstack, 会得到一个DataFrame
# 并会自动把一级索引变成DataFrame的索引, 二级索引变成DataFrame的列
data_new=data_new.unstack()
#可以通过 rename_axis(index=, columns=) 来给坐标轴重命名
data_new=data_new.rename_axis(columns=None)
# 给索引重命名
data_new.reset_index()
```

**调用unstack默认是将一级索引变成DataFrame的索引，二级索引变成DataFrame的列。更准确的说，unstack是将最后一级的索引变成DataFrame的列，前面的索引变成DataFrame的索引。比如有一个具有八级索引的Series，它在调用unstack的时候，默认是将最后一级索引变成DataFrame的列，前面七个索引整体作为DataFrame的索引。**

**只不过索引一般很少有超过二级的，所以这里就用二级举例了。因此问题来了，那么可不可以将一级索引`(这里的"姓名")`变成DataFrame的列，二级索引`(这里的"科目")`变成DataFrame的行呢？答案是可以的，在unstack中指定一个参数即可。**

```python
# 变为有2级索引的series 1级索引姓名 2级索引 科目
data_new=data.set_index(['姓名','科目'])["分数"]
# 这里的level默认是-1, 表示将最后一级的索引变成列
# 这里我们指定为0(注意: 索引从0开始), 告诉pandas, 把第一级索引变成列
data_new=data_new.unstack(level=0)
#可以通过 rename_axis(index=, columns=) 来给坐标轴重命名
data_new=data_new.rename_axis(columns=None)
# 给索引重命名
data_new.reset_index()
```

```python
#还可以使用pivot进行行专列
print(pd.pivot(data, index="姓名", columns="科目", values="分数").rename_axis(columns=None).reset_index())
```

**9.2 一行转多行**

Explode 和hive中的一致

```python
#### 一行转多行
data3=pd.DataFrame([['A','a','二,那你'],['B','b','二'],['C','c','二,那你,看看']],columns=['姓名','别名','组别'])
data3
data3["组别"] = data3["组别"].str.split(",")
data3=data3.explode("组别")
data3
```

**9.3 根据字典拆分多列**

```python
df = pd.DataFrame({"id": ["001", "002", "003"],
                   "info": [{"姓名": "琪亚娜·卡斯兰娜", "生日": "12月7日", "外号": "草履虫"},
                            {"姓名": "布洛妮娅·扎伊切克", "生日": "8月18日", "外号": "板鸭"},
                            {"姓名": "德丽莎·阿波卡利斯", "生日": "3月28日", "外号": "德丽傻", "武器": "犹大的誓约"}]
                  })
tmp = df["info"].apply(pd.Series)
tmp
```

**使用apply(pd.Series)的时候，对应的列里面的值必须是一个字典，不能是字典格式的字符串**

```python
import pandas as pd

df = pd.DataFrame({"id": ["001", "002", "003"],
                   "info": [str({"姓名": "琪亚娜·卡斯兰娜", "生日": "12月7日"}),
                            str({"姓名": "布洛妮娅·扎伊切克", "生日": "8月18日"}),
                            str({"姓名": "德丽莎·阿波卡利斯", "生日": "3月28日"})]
                   })

# 显然"info"字段的所有值都是一个字符串
tmp = df["info"].apply(pd.Series)
print(tmp)
"""
                                    0
0   {'姓名': '琪亚娜·卡斯兰娜', '生日': '12月7日'}
1  {'姓名': '布洛妮娅·扎伊切克', '生日': '8月18日'}
2  {'姓名': '德丽莎·阿波卡利斯', '生日': '3月28日'}
"""
# 我们看到此时再对info字段使用apply(pd.Series)得到的就不是我们希望的结果了, 因为它不是一个字典
# 这个时候, 我们可以eval一下, 将其变成一个字典
tmp = df["info"].map(eval).apply(pd.Series)
print(tmp)
"""
          姓名       生日
0   琪亚娜·卡斯兰娜  12月7日
1  布洛妮娅·扎伊切克  8月18日
2  德丽莎·阿波卡利斯  3月28日
"""
# 此时就完成啦
```

**不过在生产环境中还会有一个问题，我们知道Python中的eval是将一个字符串里面的内容当成值，或者你理解为就是把字符串周围的引号给剥掉。比如说：**

- `a = "123", 那么eval(a)得到的就是整型123`
- `a = "[1, 2, 3]", 那么eval(a)得到的就是列表[1, 2, 3]`
- `a = "{'a': 1, 'b': 'xxx'}", 那么eval(a)得到的就是字典{'a': 1, 'b': 'xxx'}`
- `a = "name", 那么eval(a)得到的就是变量name指向的值, 而如果不存在name这个变量, 则会抛出一个NameError`
- `a = "'name'", 那么eval(a)得到的就是'name'这个字符串; 同理a = '"name"', 那么eval(a)得到的依旧是'name'这个字符串`

**9.4 列转行 melt**

```
姓名      水果    星期一    星期二   星期三    
 古明地觉     草莓    70斤     72斤     60斤  
雾雨魔理沙    樱桃    61斤     60斤     81斤   
  琪露诺     西瓜    103斤    116斤    153斤 
  
  转为
  
  姓名     水果       日期     销量
   古明地觉    草莓      星期一    70斤
  雾雨魔理沙   樱桃      星期一    61斤
    琪露诺     西瓜      星期一    103斤
   古明地觉    草莓      星期二    72斤
  雾雨魔理沙   樱桃      星期二    60斤
    琪露诺    西瓜       星期二   116斤
   古明地觉    草莓      星期三    60斤
  雾雨魔理沙   樱桃      星期三    81斤
    琪露诺     西瓜      星期三   153斤
```

```python
import pandas as pd

df = pd.DataFrame({"姓名": ["古明地觉", "雾雨魔理沙", "琪露诺"],
                   "水果": ["草莓", "樱桃", "西瓜"],
                   "星期一": ["70斤", "61斤", "103斤"],
                   "星期二": ["72斤", "60斤", "116斤"],
                   "星期三": ["60斤", "81斤", "153斤"],
                   })



print(pd.melt(df, id_vars=["姓名", "水果"], value_vars=["星期一", "星期二", "星期三"]))

# 但是默认起得字段名叫做variable和value, 我们可以在结果的基础之上手动rename, 也可以直接在参数中指定
print(pd.melt(df, id_vars=["姓名", "水果"],
              value_vars=["星期一", "星期二", "星期三"],
              var_name="星期几?",
              value_name="销量"))
```



#### 10 读取数据速度优化

针对需要反复读取的文件有效,首次无效

pandas读取pkl和hdf的文件格式的速度要快于excel和csv,excel是最慢的,pkl最快

可以将csv和xls的保存为pkl文件,以后再次读取会变快

```python
import pandas as pd
 #读取csv
 df = pd.read_csv('xxx.csv')
 #pkl格式
 df.to_pickle('xxx.pkl') #格式另存
 df = pd.read_pickle('xxx.pkl') #读取
 #hdf格式
df.to_hdf('xxx.hdf','df') #格式另存
df = pd.read_hdf('xxx.pkl','df') #读取
```

#### 11 减小df的内存开销

数据量大时可用来减小内存开销。

```python
def reduce_mem_usage(df):
    start_mem = df.memory_usage().sum() / 1024**2
    numerics = ['int16', 'int32', 'int64', 'float16', 'float32', 'float64']

    for col in df.columns:
        col_type = df[col].dtypes
        if col_type in numerics:
            c_min = df[col].min()
            c_max = df[col].max()
            if str(col_type)[:3] == 'int':
                if c_min > np.iinfo(np.int8).min and c_max < np.iinfo(np.int8).max:
                    df[col] = df[col].astype(np.int8)
                elif c_min > np.iinfo(np.int16).min and c_max < np.iinfo(np.int16).max:
                    df[col] = df[col].astype(np.int16)
                elif c_min > np.iinfo(np.int32).min and c_max < np.iinfo(np.int32).max:
                    df[col] = df[col].astype(np.int32)
                elif c_min > np.iinfo(np.int64).min and c_max < np.iinfo(np.int64).max:
                    df[col] = df[col].astype(np.int64)
            else:
                if c_min > np.finfo(np.float16).min and c_max < np.finfo(np.float16).max:
                    df[col] = df[col].astype(np.float16)
                elif c_min > np.finfo(np.float32).min and c_max < np.finfo(np.float32).max:
                    df[col] = df[col].astype(np.float32)
                else:
                    df[col] = df[col].astype(np.float64)
                    
    end_mem = df.memory_usage().sum() / 1024**2
    print('Memory usage after optimization is: {:.2f} MB'.format(end_mem))
    print('Decreased by {:.1f}%'.format(100 * (start_mem - end_mem) / start_mem))
    return df
```

#### 常用方法

```python
# 排序
.sort_values()
data2.sort_values(by='weight',ascending=True)
# 填充nan值
.fillna()
data.fillna('B') # nan填充为B
data.fillna(method='bfill') # 使用空值后的值填充,{‘backfill’, ‘bfill’, ‘pad’, ‘ffill’, None}, default None

# 删掉含有缺失值的数据
.dropna()

# 查看是否有缺失值 true 有
.isna()
#大多数情况下数据量较大，不可能直接isna()后一个一个看是否是缺失值。any()和isna()结合使用可以判断某一列是否有缺失值。
.any()
data.isna().any()


#统计分类变量中每个类的数量
.value_counts()
data2['color'].value_counts()
data2['color'].value_counts(normalize=True,ascending=True) # 返回占比和升序

# 生成描述性统计汇总，包括数据的计数和百分位数，有助于了解大致的数据分布
data2.describe()

#打印所用数据的一些基本信息，包括索引和列的数据类型和占用的内存大小。
data2.info()


# 修改字段的数据类型，数据量大的情况下可用于减小数据占用的内存，多用于Series。
data["age"] = data["age"].astype(int)

#将DataFrame中的某一（多）个字段设置为索引
.set_index()
data2.set_index('height',inplace=True)

#重置索引，默认重置后的索引为0~len(df)-1
.reset_index()

# 去除重复值
.drop_duplicates()
data2['color'].drop_duplicates()


# 常用于构建布尔索引，对DataFrame的数据进行条件筛选
.isin()
# 晒出red和green的
data.loc[data['color'].isin(['red','green'])]

#将连续变量离散化，比如将人的年龄划分为各个区间
pd.cut()
#把薪水分成5个区间
pd.cut(data.salary,bins = 5)
# 自行指定间断点
pd.cut(data.salary,bins = [0,10,20,30,40,50])
# 指定区间的标签 
pd.cut(data.salary,bins = [0,10,20,30,40,50],labels = ['低','中下','中','中上','高'])

#将连续变量离散化，区别于pd.cut()用具体数值划分，pd.qcut()使用分位数进行区间划分
pd.qcut()
# 按照0-33.33%，33.33%-66.67%，66.67%-100%百分位进行划分
pd.qcut(data.salary,q = 3)

#将不符合条件的值替换掉成指定值，相当于执行了一个if-else
.where()
# 若salary<=40，则保持原来的值不变
# 若salary大于40，则设置为40
data['salary'].where(data.salary<=40,40)

# 拼接df 将多个Series或DataFrame拼起来（横拼或者竖拼都可以）
pd.concat()
pd.concat([data1,data2],ignore_index = False)

#对DataFrame进行数据透视，相当于Excel中的数据透视表
.pivot_table()
# 从公司和性别两个维度对薪水进行数据透视
# 看看这两个维度下的平均薪资水平
data.pivot_table(values = 'salary',index = 'company',
                          columns = 'gender',aggfunc=np.mean)
```





