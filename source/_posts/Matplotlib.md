---
title: Matplotlib
tags: 数据可视化
categories: 数据可视化
encrypt: 
enc_pwd: 
abbrlink: 40424
date: 2020-11-24 19:35:22
summary_img:
---

## 一简介

​         matplotlib是受MATLAB的启发构建的。MATLAB是数据绘图领域广泛使用的语言和工具。MATLAB语言是面向过程的。利用函数的调用，MATLAB中可以轻松的利用一行命令来绘制直线，然后再用一系列的函数调整结果。

​        matplotlib有一套完全仿照MATLAB的函数形式的绘图接口，在matplotlib.pyplot模块中。这套函数接口方便MATLAB用户过度到matplotlib包

官网:[http://matplotlib.org/](https://links.jianshu.com/go?to=http%3A%2F%2Fmatplotlib.org%2F)

官网学习例子:

- http://matplotlib.org/examples/index.html
- http://matplotlib.org/gallery.html

## 二 使用

```python
#列表:
    list1 = ['physics', 'chemistry', 1997, 2000]
    list.append('Google')  #加元素
#元组
tup1 = ('physics', 'chemistry', 1997, 2000)
tup2 = (1,)
tup3 = "a", "b", "c", "d"
tup1 = ()
#字典
dict = {'a': 1, 'b': 2, 'b': '3'}
```

**图例汉化**

```python
1:乱码原因:
	matplotlib 初始化时首先要加载一个配置文件，字体设置也在这个配置文件中。之所有无法正常显示中文是因为 这个配置文件中没有加入中文字体，解决的办法是我们需要在这个配置文件中指定一个可用的中文字体.
2:找到配置文件(下面代码可以找到)
	import matplotlib as mp
  mp.matplotlib_fname() # /Users/../.pyenv/versions/3.7.5/lib/python3.7/site-packages/matplotlib/mpl-data/matplotlibrc
  打开配置文件所在的路径，可以看到配置文件matplotlibrc。同时，我们还能看到字体文件夹fonts，这里放置matplotlib使用的字体。# fonts		images		matplotlibrc	sample_data	stylelib
3:下载中文字体
  在网站上下载 黑体SimHei 字体（http://www.fontpalace.com/font-details/SimHei/） ，该字体即有Windows字体也有Mac字体
  mac电脑：下载字体后直接双击安装就可以。
  window电脑：复制下载的字体安装文件到matplotlib字体文件夹fonts中的ttf文件夹中，然后双击该字体进行安装。
4:修改配置文件
  修改之前最好先备份下这个配置文件（matplotlibrc），万一修改错，还可以用备份文件还原。我一般比较喜欢将要修改的文件复制一份，并手动给他加上后缀.bak，表示备份backup。如果我修改原文件错误，想恢复到原来的状态，那我就可以将备份文件的后缀.bak删除，就能恢复了。 使用sublime软件打开 matplotlib的配置文件，如果你没有sublime这个软件，最好下载一个，因为这个软件可以打开各种格式的文件。 打开文件后，你可以使用查找功能找到 font.family 和 font.sans-serif这两行，去掉最前面的注释'#'，并在font.sans-serif这一行值中添加我们刚才安装的黑体SimHei, 有的坐标轴的负号显示不正常，我们还要找到axes.unicode_minus这一行，去掉最前面的注释'#'，同时将值修改为False。
5:删除缓存
  在路径下（windows路径C:\Users\你的用户名\.matplotlib，mac路径/Users/你的用户名/.matplotlib ）找到一个叫\.matplotlib的文件夹，这是matplotlib的缓存目录，删除这个文件夹。 注意，mac需要同时删除tex.cache和fontList.json这两个文件
6:重启jupyter notebook
7:额外
	如果上面操作完，中文还是乱码，试下这个解决办法:
    方法1：改了配置之后并不会生效，需要重新加载字体，在notebook中运行如下代码即可： from matplotlib.font_manager import _rebuild _rebuild() #重新加载一下
    方法2:重启大法(重启电脑)
  若是图例正常显示中文,但有报错 如:RuntimeWarning: Glyph xxxxx missing from current font.
      设置一下参数:
        plt.rcParams['font.sans-serif']=['SimHei'] 
        plt.rcParams['axes.unicode_minus']=False
```

### 0 查看各个图的参数

```python
import numpy as np
#如散点图:
np.info(plt.scatter)
```

### 1 折线图

#### 1.1 折线图基本

```python
from matplotlib import pyplot as plt 
x = range(2,26,2)
y_t = range(2,100,10)
y = [2,3,55,
    37,5,51
     ,8,4,54
     ,6,7,8]
y2 = [1,2,3
     ,4,5,6
     ,7,8,9
     ,10,11,12]
#设置图像大小 图像模糊dpi可以是图片清晰
fig=plt.figure(figsize=(5,4),dpi=80)
#设置x,y轴的刻度值
#x = [i/5 for i in x]
plt.xticks(x)
plt.yticks(y_t)
#设置x轴和y轴的图例
plt.xlabel('一天的时间')
plt.ylabel('温度')
#图名称
plt.title('折线图')
#label线名,c线的颜色支持16进制,linewidth线的宽度,linestyle线的格式,alpha=0.5透明度,
#plt.plot( )返回的是一个二元组值，若要获取实例，必须用x, = plt.plot( )才能取出来实例对象
l1,=plt.plot(x, y, label='折线',c='red',linewidth='1.5',linestyle='--')
l2,=plt.plot(x, y2, label='线',c='blue')
# 给图像加上图例,不加的话 上方的label参数不生效 
#plt.legend()
plt.legend(handles=[l1,l2],labels=['z','l'],loc='best')
#保存图片 可以保存为svg这种矢量图,放大不会有锯齿
plt.savefig('/Users/manzhong/Desktop/fig_t.png')
plt.show()
```

```
legend 里面参数主要有以下三种
		handles需要传入你所画线条的实例对象，[l1,l2]
		labels是图例的名称（能够覆盖在plt.plot( )中label参数值）
		loc代表了图例在整个坐标轴平面中的位置（一般选取'best'这个参数值）
loc详解:
	1:loc='best' 图中最合适的位置(机器判断)
	2:loc='xxx' 范围(center right/left,center,upper,upper left/right,lower,lower left/right)9种
	3:loc=(x,y)表示图例左下角的位置，这是最灵活的一种放置图例的方法，慢慢调整，总会找到你想要的放置图例的位置
		 当使用loc = (x, y)时，x, y并不是轴域中实际的x, y的值，而是将x轴, y轴分别看成1, 即：
			( x/(x_max-x_min) , y/(y_max-y_min) )（即进行归一化处理）;
			那么，在绘制图表时，若用到坐标轴的范围限制，如xlim=(0, 16), ylim=(0, 9)。在此基础上，如果要将图例放置	     			到点(2, 2)上，loc实际传入的参数应该为：
		 loc = ( 2/(16-0) , 2/(9-0) )
		 即 loc = (2/16, 2/9)
```

#### 1.2:x轴刻度

```python
from matplotlib import pyplot as plt 
import random
#展示10点到12点之间每一分钟的值
x=range(0,120)
y = [random.randint(20,35) for i in range(120)]
plt.figure(figsize=(20,8))
plt.plot(x,y)
#调整x轴的刻度
_xt= ["10点{}分".format(i) for i in range(60)] 
_xt += ["11点{}分".format(i) for i in range(60)]
# 取步长 数字与字符串一一对应,数据长度一样 rotation旋转
plt.xticks(list(x)[::3],_xt[::3],rotation=90)
# 网格 这个网格和x,y轴的各自间距有关 alpha网格透明度
plt.grid(alpha=0.9)
plt.show()
```

#### 1.3 画多个图

```python
# 1 subplot
from matplotlib import pyplot as plt 
x = range(2,26,2)
y_t = range(2,100,10)
y = [2,3,55,
    37,5,51
     ,8,4,54
     ,6,7,8]
y2 = [1,2,3
     ,4,5,6
     ,7,8,9
     ,10,11,12]
plt.figure(figsize=(8,5))
# 第1行2列 第1列
plt.subplot(1,2,1,frameon=True)
#plt.xticks(range(1,8)) 设置x轴
#plt.yticks(range(1,8)) 设置y轴
plt.plot(x,y)
# 第1行2列 第2列
plt.subplot(1,2,2)
#plt.xticks(range(1,8)) 设置x轴
#plt.yticks(range(1,8))
plt.bar(x,y2)
plt.show()
```

```python
# 2 subplots
from matplotlib import pyplot as plt 
x = range(2,26,2)
y_t = range(2,100,10)
y = [2,3,55,
    37,5,51
     ,8,4,54
     ,6,7,8]
y2 = [1,2,3
     ,4,5,6
     ,7,8,9
     ,10,11,12]
# sharex=True 共享x轴,sharey=True共享y轴
fig,ax=plt.subplots(2,2,sharex=True,sharey=True)
#fig,((ax, ax2), (ax3, ax4)) =plt.subplots(2,2,figsize=(8,6))
ax[0][0].plot(x,y)
#ax[0][0].set_xlabel('日期') 设置x的说明
#ax[0][0].tick_params(rotation=90) 设置x的轴值 相当于xticks
ax[0][1].bar(x,y)
ax[1][0].scatter(x,y,marker='*')
ax[1][1].hist(y)
```

```python
# 3 subplot2grid 
plt.figure()
# 整个图有3行3列,从（0，0）点开始画，行跨度colspan为3，列跨度rowspan为1
ax1 = plt.subplot2grid((3, 3), (0, 0), colspan=3, rowspan=1)
ax1.plot([1, 2], [1, 2]) #两个点
ax1.set_title('ax1_title')
# ax2 从（1，0）点开始画，下标从0开始
ax2 = plt.subplot2grid((3, 3), (1, 0), colspan=2)
ax3 = plt.subplot2grid((3, 3), (1, 2), colspan=1,rowspan=2)
ax4 = plt.subplot2grid((3, 3), (2, 0))  # colspan,rowspan 默认取值为1
ax5 = plt.subplot2grid((3, 3), (2, 1))
 
ax2.plot(np.linspace(-5, 5, 100), np.sin(np.linspace(-5, 5, 100)))
ax3.plot(np.linspace(-3, 3, 50), np.tan(np.linspace(-3, 3, 50)))
ax4.plot(np.linspace(-3, 3, 50), np.cos(np.linspace(-3, 3, 50)))
ax5.plot(np.linspace(-3, 3, 50), np.tanh(np.linspace(-3, 3, 50)))
plt.show()
```

```python
# 4 gridspec 
plt.figure()
gs = gridspec.GridSpec(3, 3)  # 3行3列的GridSpec
ax1 = plt.subplot(gs[0, :])  #ax1在GridSpec中是第0行，所有列
ax2 = plt.subplot(gs[1, :2])  #ax2在GridSpec中是第1行，第0、1列
ax3 = plt.subplot(gs[1:, 2])
ax4 = plt.subplot(gs[2, 0])
ax5 = plt.subplot(gs[2, 1])
plt.show()
```



#### 1.4 一个图多个坐标轴(twinx与twiny)

```python
# 主次坐标轴，共享x轴，y轴数据不同
x = np.arange(0, 10, 0.1)
y1 = 0.05*x**2
y2 = -1*y1
 
fig, ax1 = plt.subplots()
ax2 = ax1.twinx()    # y坐标轴twin
# ax2 = ax1.twiny()  # x坐标轴twin
ax1.plot(x, y1, 'g-')
ax2.plot(x, y2, 'b-')
 
ax1.set_xlabel('x')
ax1.set_ylabel('y1', color='g')
ax2.set_ylabel('y2', color='b')
```

#### 1.5图中图

```python
# 图中图
fig = plt.figure()
x = [1, 2, 3, 4, 5, 6, 7]
y = [1, 3, 4, 2, 5, 8, 6]
 
left, bottom, width, height = 0.1, 0.1, 0.8, 0.8
ax1 = fig.add_axes([left, bottom, width, height])  # 位置 宽度 高度（值是百分比）
ax1.plot(x, y, 'r')
ax1.set_title('title')
ax1.set_xlabel('x')
ax1.set_ylabel('y')
 
left, bottom, width, height = 0.2, 0.6, 0.25, 0.25
ax2 = fig.add_axes([left, bottom, width, height])  # 位置 宽度 高度（值是百分比）
ax2.plot([1, 2], [2, 5], 'b')
ax2.set_title('title inside1')
ax2.set_xlabel('x')
ax2.set_ylabel('y')
 
plt.axes([0.6, 0.2, 0.25, 0.25])
plt.plot(y[::-1], x, 'g')
plt.title('title inside2')
plt.xlabel('x')
plt.ylabel('y')
 
plt.show()
```

### 2 散点图

```python
# 函数 scatter()
from matplotlib import pyplot as plt 
y_3=[11,17,12,6,7,8,9,10,17,18,19,20,22,24,25,12,11,14,17,20]
y_10=[26,27,22,26,27,28,29,20,17,28,18,16,12,14,15,12,11,14,17,10]
x_3=range(1,21)
x_10=range(40,60)
#设置x刻度
_x=list(x_3)+list(x_10)
_xt=["3月{}日".format(i) for i in x_3] 
_xt += ["10月{}日".format(i-40) for i in x_10]
#_x[:;3] 取步长
plt.xticks(_x[::3],_xt,rotation=90)

plt.scatter(x_3,y_3,label='三月')
plt.scatter(x_10,y_10,label='十月')
plt.legend()
```

#### 2.1 散点图中的气泡图

```python
#只需调整scatter中参数
x = np.arange(0,10,1)
y = x*x
# s 大小 c颜色
plt.scatter(x,y,s=y*10,c=y*10,linewidths=2) #颜色、大小都用数组表示
plt.ylim(0,100)
plt.xlim(0,12)
plt.show()
```

#### 2.2散点图拟合函数

```python
import numpy as np
from scipy.optimize import leastsq
import matplotlib.pyplot as plt
#训练数据
# Xi = np.array([8.19,2.72,6.39,8.71,4.7,2.66,3.78])
# Yi = np.array([7.01,2.78,6.47,6.71,4.1,4.23,4.05])
Xi = np.array([8.19,2.72,6.39,8.71,4.7,2.66,3.78,12.19,8.72,9.39,10.71,11.7,12.66,12.78])
Yi = np.array([7.01,2.78,6.47,6.71,4.1,4.23,4.05,12.01,9.78,8.67,10.2,12.8,12.9,12.09])
#定义拟合函数形式
def func(p,x):
    k,b = p
    return k*x+b
#定义误差函数
def error(p,x,y,s):
    print(s)
    return func(p,x)-y

#随机给出参数的初始值
p = [10,2]

#使用leastsq()函数进行参数估计
s = '参数估计次数'
Para = leastsq(error,p,args=(Xi,Yi,s))
k,b = Para[0]
print('k=',k,'\n','b=',b)

#图形可视化
plt.figure(figsize = (10,8))
#绘制训练数据的散点图
plt.scatter(Xi,Yi,color='r',label='Sample Point',linewidths = 3)
plt.xlabel('x')
plt.ylabel('y')
x = np.linspace(0,10,1000)
y = k*x+b
plt.plot(x,y,color= 'orange',label = 'Fitting Line',linewidth = 2)
plt.legend()
plt.show()
```

### 3.条形图

#### 3.1竖条形图

```python
# plt.bar(x,y)
from matplotlib import pyplot as plt 
y_10=[26,27,22,26,27,28,29,20,17,28,18,16,12,14,15,12,11,14,17,10]
# width 宽度 hatch="\\" 图案
plt.bar(range(len(y_10)),y_10,width=0.3)
plt.show()
```

#### 3.2 横条形图

```python
from matplotlib import pyplot as plt 
y_10=[26,27,22,26,27,28,29,20,17,28,18,16,12,14,15,12,11,14,17,10]
plt.grid()
#height 线条粗细  和竖着的width做区别
plt.barh(range(len(y_10)),y_10)
```

#### 3.3 绘制多条

```python
from matplotlib import pyplot as plt 
#对比 abcd 最近三天的趋势
a=["a","b","c","d"]
b15=[123,456,789,123]
b16=[456,123,456,234]
b17=[789,789,123,345]
#图形间间距
bar_width=0.2
x_15= list(range(len(a)))
x_16 = [i+bar_width for i in x_15]
x_17 = [i+bar_width*2 for i in x_15]

plt.bar(range(len(a)),b15,width=bar_width,label='15日')
plt.bar(x_16,b16,width=bar_width,label='16日')
plt.bar(x_17,b17,width=bar_width,label='17日')
#x刻度
plt.xticks(x_16,a)
plt.legend()
plt.show()
```

### 4 直方图

#### 4.1 频数直方图

```python 
#如你获取了10个电影的时长数据,希望统计电影时长的分布(如100到120分钟的电影数量)
#将数据分为多少组进行统计?组数要适当,太少有较大的误差,太多规律不明显
# 组数:数据量在100以内一般5-12组
# 组距:每个小组两个端点的距离
# 组数=极差/组距
from matplotlib import pyplot as plt 
a=[180,100,90,100,110,120,80,70,150,140,140,150,70,90,85,60,80,100,90,90]
#组距
d=10
#计算组数
num=(max(a)-min(a))//d
#设置x轴
plt.xticks(range(min(a),max(a)+d,d))
plt.hist(a,num)

#以上是组距均匀的 下面看一下组距不均匀的
from matplotlib import pyplot as plt 
a=[183,100,90,100,110,120,80,70,150,140,140,150,70,90,85,60,80,100,90,90]
#组距
d=10
#计算组数
num=(max(a)-min(a))//d
#设置x轴
plt.xticks(range(min(a),max(a)+d,d))
plt.grid()
plt.hist(a,num)

#组距不均匀 一般是组距和x轴间距不一致导致 合理设置组距即可
```

#### 4.2 频率直方图

```python
from matplotlib import pyplot as plt 
import math
a=[100,90,100,110,120,80,70,150,180,140,140,150,70,90,85,60,80,100,90,90]
#组距
d=10
#计算组数
num=(max(a)-min(a))//d
#设置x轴
plt.xticks(range(min(a),max(a)+d,d)) 
plt.grid()
#density=1 或True 变频率直方图
plt.hist(a,num,density=1)
```

#### 4.3 直方图参数说明

| orientation | 直方图竖直或者是水平显示 值: horizontal(横) 默认竖 |
| ----------- | --------------------------------- |
| align       | 直方图坐落的位置                          |
| density     | 频数换算成频率                           |
| bottom      | 设定y轴的起始位置                         |
| range       | 设定随机变量统计范围                        |
| bins        | 分组个数                              |

### 5 饼图

```python
from matplotlib import pyplot as plt 
import numpy as np
import time 
import pandas as pd 
#此处必须添加此句代码方可改变标题字体大小 暂未生效
#plt.rcParams.update({"font.size":20})

shapes = ['Cross', 'Cone', 'Egg', 'Teardrop', 'Chevron', 'Diamond', 'Cylinder',
       'Rectangle', 'Flash', 'Cigar', 'Changing', 'Formation', 'Oval', 'Disk',
       'Sphere', 'Fireball', 'Triangle', 'Circle', 'Light']
values = [  287,   383,   842,   866,  1187,  1405,  1495,  1620,  1717,
        2313,  2378,  3070,  4332,  5841,  6482,  7785,  9358,  9818, 20254]
#数据组合
s = pd.Series(values, index=shapes)

labels = s.index
sizes = s.values
#设置每一部分离开中心点的距离
explode = (0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0)  

fig1, ax1 = plt.subplots(figsize=(10,10),dpi=80)

#绘制饼图
#labels：设置各类的说明文字
#autopct:第二个百分号表示转义字符，将第三个%直接显示出来,1表示小数点后面一位
#colors：设置为各部分染色列表
#explode:每一部分离开中心点的距离 ,元素数目与x相同且一一对应
#shadow:在饼图下面画一个阴影，3D效果
patches, texts, autotexts = ax1.pie(sizes, explode=explode, labels=labels, autopct='%1.0f%%',
        shadow=True, startangle=170)
#相等的长宽比可确保将饼图绘制为圆形
ax1.axis('equal')  # Equal aspect ratio ensures that pie is drawn as a circle.
plt.legend(labels, loc=(1.2,0.2))
plt.show()
```

### 6 堆积图

```python
#柱状堆积图
import matplotlib as mpl
import matplotlib.pyplot as plt

plt.figure(figsize=(10,10),dpi=80)

x=[1,2,3,4,5]
y=[6,10,4,5,1]
y1=[2,6,3,8,5]

plt.bar(x,y,align="center",color="#66c2a5",tick_label=["A","B","C","D","E"],label="班级A")
plt.bar(x,y1,align="center",bottom=y,color="#8da0cb",label="班级B")

plt.xlabel("测试难度")
plt.ylabel("试卷份数")

plt.legend(loc='best')
plt.show()
```

### 7 面积图

```python
#折线面积图
import matplotlib.pyplot as plt
import numpy as np

x = np.arange(1,6,1)
y = [0,4,3,5,6]
y1 = [1,3,4,2,7]
y2 = [3,4,1,6,5]
labels = ["BluePlanet","BrownPlanet","GreenPlanet"]
colors = ["#8da0cb","#fc8d62","#66c2a5"]
plt.stackplot(x,y,y1,y2,labels=labels,colors=colors)
plt.legend(loc="upper left")
plt.show()
```

### 8 热力图

```python
# 读取数据
Sales = pd.read_excel('/Users/.../Desktop/Jupyter/Sales.xlsx')
# 根据交易日期，衍生出年份和月份字段
Sales['year'] = Sales.Date.dt.year
Sales['month'] = Sales.Date.dt.month
# 统计每年各月份的销售总额（绘制热力图之前，必须将数据转换为交叉表形式）
Summary = Sales.pivot_table(index = 'month', columns = 'year', values = 'Sales', aggfunc = np.sum)
# 绘制热力图
sns.heatmap(data = Summary, # 指定绘图数据
 cmap = 'PuBuGn', # 指定填充色
 linewidths = .1, # 设置每个单元格边框的宽度
 annot = True, # 显示数值
 fmt = '.1e' # 以科学计算法显示数据
 ) #添加标题
plt.title('每年各月份销售总额热力图')
# 显示图形
plt.show()
```

```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

y = np.random.randint(1,100,40)
y = y.reshape((5,8))
df = pd.DataFrame(y,columns=[x for x in 'abcdefgh'])
sns.heatmap(df,annot=True)
plt.show()
```

### 9 箱型图

#### 9.1 说明

```
        o          异常值
    ---------      上限
    		|
    		|
    ---------		   上4分位数
    |				|
    |				|
    |-------|      中位数
    |				|
    ---------      下4分位数
        |
        |
    ---------      下限
    
```

#### 9.2 分类

```python
# 1 画箱型图的另一种方法，参数较少，而且只接受dataframe，不常用
dataframe.boxplot()
# 2 
plt.boxplot()
```

**plt.boxplot()的参数**

| 参数           | 说明             | 参数           | 说明                 |
| ------------ | -------------- | ------------ | ------------------ |
| x            | 指定要绘制箱线图的数据；   | showcaps     | 是否显示箱线图顶端和末端的两条线   |
| notch        | 是否是凹口的形式展现箱线图  | showbox      | 是否显示箱线图的箱体         |
| sym          | 指定异常点的形状       | showfliers   | 是否显示异常值            |
| vert         | 是否需要将箱线图垂直摆放   | boxprops     | 设置箱体的属性，如边框色，填充色等； |
| whis         | 指定上下须与上下四分位的距离 | labels       | 为箱线图添加标签           |
| positions    | 指定箱线图的位置       | filerprops   | 设置异常值的属性           |
| widths       | 指定箱线图的宽度       | medianprops  | 设置中位数的属性           |
| patch_artist | 是否填充箱体的颜色；     | meanprops    | 设置均值的属性            |
| meanline     | 是否用线的形式表示均值    | capprops     | 设置箱线图顶端和末端线条的属性    |
| showmeans    | 是否显示均值         | whiskerprops | 设置须的属性             |

**只接收DataFrame的 boxplot () 语法：**

| 参数           | 接收值    | 说明                        | 默认值   |
| ------------ | ------ | ------------------------- | ----- |
| column       | list   | 指定要进行箱型图分析的列；             | 全部列   |
| showmeans    | bool   | 是否显示均值；                   | FALSE |
| notch        | bool   | 是否是凹口的形式展现箱线图；            | FALSE |
| patch_artist | bool   | 是否填充箱体的颜色，若为true，则默认蓝色；   | FALSE |
| grid         | bool   | 箱型图网格线是否显示；               | TRUE  |
| vert         | bool   | 竖立箱型图（True）/水平箱型图（False）； | TRUE  |
| sym          | string | 指定异常点的形状；                 | o     |

```python
import pandas as pd
import matplotlib.pyplot as plt
from pandas.core.frame import DataFrame
#csv格式
#   hp_cal_dt	  user_id	  	stay_time
#0	2021-03-01	11430293		192575
#1	2021-03-01	16111935		41829
#2	2021-03-02	20234033		34217
#3	2021-03-02	22830367		3680
#4	2021-03-03	23807166		4874
path = 'stay_box.csv'
data = pd.read_csv(path)
df=data.sort_values(by='hp_cal_dt').groupby(data['hp_cal_dt'])
c={}
listx=[]
listy=[]
fig=plt.figure(figsize=(50,20),dpi=80)
for i,j in df:
    c[i]=j['stay_time']
    listx.append(i)
    listy.append(j['stay_time']/1000)
plt.xticks(range(1,4000,20),rotation=90)
plt.boxplot(listy,labels=listx,vert=False,showmeans=True)
plt.savefig('/Users/manzhong/Desktop/box.png')
```

```python
import pandas as pd
import matplotlib.pyplot as plt
from pandas.core.frame import DataFrame
#csv格式
#   hp_cal_dt	  user_id	  	stay_time
#0	2021-03-01	11430293		192575
#1	2021-03-01	16111935		41829
#2	2021-03-02	20234033		34217
#3	2021-03-02	22830367		3680
#4	2021-03-03	23807166		4874
path = 'stay_box.csv'
data = pd.read_csv(path)
data.boxplot(column='stay_time',by='hp_cal_dt')
```



