---
title: Pandas的可视化
tags: 数据分析
categories: 数据分析
abbrlink: 9864
date: 2021-03-06 19:52:51
summary_img:
encrypt:
enc_pwd:
---

## 一 简介

pandas自身依赖于matplotlib,自己构造了一套数据可视化的api,主要有以下几个包

```
1: pandas.DataFrame.plot.* (area,bar,barh,box,hist,line,pie....)
2: pandas.DataFrame.hist/boxplot
3: pandas.plotting.* (table,scatter_matrix,boxplot....)
```

## 二 使用

### 2.1 DataFrame自带的绘图

目前自带的绘图只有两个直方图和箱型图

**1 hist图**

**用于查看dataframe的每个数值属性的直方图**

```python
import pandas as pd
%matplotlib inline
import matplotlib.pyplot as plt
# 绘制df的每一列的直方图分布
# bins 直方图的箱数
df.hist(bins=20,figsize=(20,15))
plt.show()

#new_pbf同一幅图里 分sex指标的直方图
#newdf_sub.groupby("sex")["new_pbf"].hist(bins=100,figsize=(10,6),alpha=0.5)
# 按性别分两个图
#newdf_sub["newb"].hist(bins=100,figsize=(10,6),alpha=0.5,by="sex")
```

**2 boxplot**

**用于查看df的每一列的箱型图**

```python
import pandas as pd
import numpy as np
%matplotlib inline
import matplotlib.pyplot as plt

np.random.seed(1234)
df = pd.DataFrame(np.random.randn(10, 4),
                  columns=['Col1', 'Col2', 'Col3', 'Col4'])
boxplot = df.boxplot(column=['Col1', 'Col2', 'Col3']) 
```

### 2.2 dataframe 的plot下的绘图

#### 1 plot

可以绘制多类型图

```python
import numpy as np
np.random.seed(1234)
df = pd.DataFrame(np.random.randn(10, 4),
                  columns=['Col1', 'Col2', 'Col3', 'Col4']) 
# kind
#‘line’ : line plot (default)
#‘bar’ : vertical bar plot
#‘barh’ : horizontal bar plot
#‘hist’ : histogram
#‘box’ : boxplot
#‘kde’ : Kernel Density Estimation plot
#‘density’ : same as ‘kde’
#‘area’ : area plot
#‘pie’ : pie plot
#‘scatter’ : scatter plot (DataFrame only)
#‘hexbin’ : hexbin plot (DataFrame only)
df.plot(kind="line",x="Col1",y="Col2")
```

#### 2 bar/barh

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({'lab':['A', 'B', 'C'], 'val':[10, 30, 20]})
ax = df.plot.bar(x='lab', y='val', rot=0)
# 竖直条形图
axh = df.plot.barh(x='lab', y='val') 
```

#### 3 box

```python
data = np.random.randn(25, 4)
df = pd.DataFrame(data, columns=list('ABCD'))
ax = df.plot.box()
```

#### 4 density 核密度估计图

核密度图可以看作是概率密度图,其纵轴可以粗略看做是数据出现的次数，与横轴围成的面积是一.

```python
df = pd.DataFrame({
    'x': [1, 2, 2.5, 3, 3.5, 4, 5],
    'y': [4, 4, 4.5, 5, 5.5, 6, 6],
})
ax = df.plot.kde()
ax2 = df.plot.kde(bw_method=0.3)
```

#### 5 hist

```python
df = pd.DataFrame(
    np.random.randint(1, 7, 6000),
    columns = ['one'])
df['two'] = df['one'] + np.random.randint(1, 7, 6000)
ax = df.plot.hist(bins=12, alpha=0.5)
```

by参数分组直方图

```python
age_list = [8, 10, 12, 14, 72, 74, 76, 78, 20, 25, 30, 35, 60, 85]
df = pd.DataFrame({"gender": list("MMMMMMMMFFFFFF"), "age": age_list})
ax = df.plot.hist(column=["age"], by="gender", figsize=(10, 8))
```

#### 5 line

```python
df = pd.DataFrame({
   'pig': [20, 18, 489, 675, 1776],
   'horse': [4, 25, 281, 600, 1900]
   }, index=[1990, 1997, 2003, 2009, 2014])
lines = df.plot.line()
```

分组子图

```python
axes = df.plot.line(subplots=True)
```

指定xy

```python
lines = df.plot.line(x='pig', y='horse')
```

#### 6 pie

```python
df = pd.DataFrame({'mass': [0.330, 4.87 , 5.97],
                   'radius': [2439.7, 6051.8, 6378.1]},
                  index=['Mercury', 'Venus', 'Earth'])
plot = df.plot.pie(y='mass', figsize=(5, 5))
```

分组子图

```python
plot = df.plot.pie(subplots=True, figsize=(11, 6))
```

#### 7 scatter

```python
df = pd.DataFrame([[5.1, 3.5, 0], [4.9, 3.0, 0], [7.0, 3.2, 1],
                   [6.4, 3.2, 1], [5.9, 3.0, 2]],
                  columns=['length', 'width', 'species'])
ax1 = df.plot.scatter(x='length',
                      y='width',
                      c='DarkBlue')
```

控制颜色

```python
ax2 = df.plot.scatter(x='length',
                      y='width',
                      c='species',
                      colormap='viridis')
```



### 2.3 pandas下的plotting

#### 1 pandas.plotting.scatter_matrix

**监测属性间相关性的散点直方图**

```python
from pandas.plotting import scatter_matrix
df = pd.DataFrame(np.random.randn(1000, 4), columns=['A','B','C','D'])
#pd.plotting.scatter_matrix(df, alpha=0.2)
scatter_matrix(df, alpha=0.2,figsize=(20,15))
```

#### 2 pandas.plotting.radviz

**绘制聚类图,将 N 维数据集投影到 2D 空间中，其中每个维度的影响可以解释为所有维度影响之间的平衡**

```python
df = pd.DataFrame(
    {
        'SepalLength': [6.5, 7.7, 5.1, 5.8, 7.6, 5.0, 5.4, 4.6, 6.7, 4.6],
        'SepalWidth': [3.0, 3.8, 3.8, 2.7, 3.0, 2.3, 3.0, 3.2, 3.3, 3.6],
        'PetalLength': [5.5, 6.7, 1.9, 5.1, 6.6, 3.3, 4.5, 1.4, 5.7, 1.0],
        'PetalWidth': [1.8, 2.2, 0.4, 1.9, 2.1, 1.0, 1.5, 0.2, 2.1, 0.2],
        'Category': [
            'virginica',
            'virginica',
            'setosa',
            'virginica',
            'virginica',
            'versicolor',
            'versicolor',
            'setosa',
            'virginica',
            'setosa'
        ]
    }
)
pd.plotting.radviz(df, 'Category')
```

#### 3 pandas.plotting.parallel_coordinates

平行坐标绘图

```python
dfiris = pd.read_csv(
    'https://raw.github.com/pandas-dev/'
    'pandas/main/pandas/tests/io/data/csv/iris.csv'
)
pd.plotting.parallel_coordinates(
    df, 'Name', color=('#556270', '#4ECDC4', '#C7F464')
)
```

#### 4 pandas.plotting.lag_plot

时间序列的滞后图,滞后图是用时间序列和相应的滞后阶数序列做出的散点图。可以用于观测自相关性。

```python
# 给定序列
np.random.seed(5)
x = np.cumsum(np.random.normal(loc=1, scale=5, size=50))
s = pd.Series(x)
s.plot()

#lag=1 的回报滞后图
pd.plotting.lag_plot(s, lag=1)
```

#### 5 pandas.plotting.boxplot

**从 DataFrame 列制作箱形图**

```python
np.random.seed(1234)
df = pd.DataFrame(np.random.randn(10, 4),
                  columns=['Col1', 'Col2', 'Col3', 'Col4'])
boxplot = df.boxplot(column=['Col1', 'Col2', 'Col3']) 
```

**分组**

```python
df = pd.DataFrame(np.random.randn(10, 2),
                  columns=['Col1', 'Col2'])
df['X'] = pd.Series(['A', 'A', 'A', 'A', 'A',
                     'B', 'B', 'B', 'B', 'B'])
boxplot = df.boxplot(by='X')
```

可以将字符串列表（即）传递给箱线图，以便通过 x 轴中的变量组合对数据进行分组：`['X', 'Y']`

```python
df = pd.DataFrame(np.random.randn(10, 3),
                  columns=['Col1', 'Col2', 'Col3'])
df['X'] = pd.Series(['A', 'A', 'A', 'A', 'A',
                     'B', 'B', 'B', 'B', 'B'])
df['Y'] = pd.Series(['A', 'B', 'A', 'B', 'A',
                     'B', 'A', 'B', 'A', 'B'])
boxplot = df.boxplot(column=['Col1', 'Col2'], by=['X', 'Y'])
```

```python
#可以对箱线图进行其他格式化，例如抑制网格 ( grid=False)、在 x 轴上旋转标签 (ie rot=45) 或更改字体大小 (ie fontsize=15)
boxplot = df.boxplot(grid=False, rot=45, fontsize=15)  
```

#### 6 pandas.plotting.bootstrap_plot

均值、中值和中间范围统计数据的引导图。

```python
s = pd.Series(np.random.uniform(size=100))
pd.plotting.bootstrap_plot(s)
```

#### 7 pandas.plotting.autocorrelation_plot

时间序列的自相关图。

```python
spacing = np.linspace(-9 * np.pi, 9 * np.pi, num=1000)
s = pd.Series(0.7 * np.random.rand(1000) + 0.3 * np.sin(spacing))
# 图中的水平线对应于 95% 和 99% 的置信带。虚线是 99% 置信带。
pd.plotting.autocorrelation_plot(s)
```

#### 8 pandas.plotting.andrews_curves(调和曲线)

生成 Andrews 曲线的 matplotlib 图，用于可视化多变量数据的集群。一般用户聚类分析

两个样品点之间的欧式距离越近，其Andrews曲线也会越近，往往彼此纠缠在一起。因此Andrews曲线常用于反映多元样品数据的结构，以预估各样品的聚类情况。

调和曲线图由Andrews于1972年提出，因此又叫Andrews plots或Andrews curve，是将多元数据以二维曲线展现的一种统计图，常用于表示多元数据的结构

```python
df = pd.read_csv(
    'https://raw.github.com/pandas-dev/'
    'pandas/main/pandas/tests/io/data/csv/iris.csv'
)
pd.plotting.andrews_curves(df, 'Name')
```