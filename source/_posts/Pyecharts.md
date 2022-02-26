---
title: Pyecharts
tags: 数据可视化
categories: 数据可视化
abbrlink: 5055
date: 2021-12-19 18:04:19
summary_img:
encrypt:
enc_pwd:
---

## 一 简介

### 1.1 概述

[Echarts](https://github.com/ecomfe/echarts) 是一个由百度开源的数据可视化，凭借着良好的交互性，精巧的图表设计，得到了众多开发者的认可。而 Python 是一门富有表达力的语言，很适合用于数据处理。当数据分析遇上数据可视化时，[pyecharts](https://github.com/pyecharts/pyecharts) 诞生了。

### 1.2 特性

```text
简洁的 API 设计，使用如丝滑般流畅，支持链式调用
囊括了 30+ 种常见图表，应有尽有
支持主流 Notebook 环境，Jupyter Notebook 和 JupyterLab
可轻松集成至 Flask，Django 等主流 Web 框架
高度灵活的配置项，可轻松搭配出精美的图表
详细的文档和示例，帮助开发者更快的上手项目
多达 400+ 地图文件以及原生的百度地图，为地理数据可视化提供强有力的支持
```

### 1.3 版本

pyecharts 分为 v0.5.X 和 v1 两个大版本，v0.5.X 和 v1 间不兼容，v1 是一个全新的版本

v0.5.X :支持 Python2.7，3.4+

v1: 仅支持 Python3.6+

**本文版本:**

```
1.9.1
```

### 1.4  安装

```
1: 查看pyecharts的安装版本
	import pyecharts
	print(pyecharts.__version__)
2:安装
	pip(3) install pyecharts
```

**官方文档(不做版本管理，您所看到的当前文档为最新版文档，若文档与您使用的版本出现不一致情况，请及时更新 pyecharts)**

```url
https://pyecharts.org/#/zh-cn/quickstart
https://gallery.pyecharts.org/#/Table/table_base
```

### 1.5 图表展示形式

**1: 保存为图片**

```python
from pyecharts.charts import Bar
from pyecharts.faker import Faker
from pyecharts.globals import ThemeType
def bar_ch() -> Bar:
    c = (
        Bar({"theme": ThemeType.MACARONS})
        .add_xaxis(Faker.choose())
        .add_yaxis("商家A", Faker.values())
        .add_yaxis("商家B", Faker.values())
        .set_global_opts(
            title_opts={"text": "Bar-通过 dict 进行配置", "subtext": "我也是通过 dict 进行配置的"}
        )
    )
    return c
 #保存为图片
make_snapshot(snapshot, bar_ch().render(), "bar.png")
```

保存为图片需要安装:

```shell
pip install snapshot-selenium
```

使用selenium 需要配置browser driver(mac版)

```
1: 下载chromedriver  需要和chrome浏览器的版本对应
	http://chromedriver.storage.googleapis.com/index.html
2: 下载后解压将文件移动到 /usr/local/bin 目录下
3: 配置环境变量
```

**2: 生成html 文件**

```python
from pyecharts.charts import Bar
from pyecharts.faker import Faker
from pyecharts.globals import ThemeType
def bar_ch() -> Bar:
    c = (
        Bar({"theme": ThemeType.MACARONS})
        .add_xaxis(Faker.choose())
        .add_yaxis("商家A", Faker.values())
        .add_yaxis("商家B", Faker.values())
        .set_global_opts(
            title_opts={"text": "Bar-通过 dict 进行配置", "subtext": "我也是通过 dict 进行配置的"}
        )
    )
    return c
 #html
 bar_ch().render("path/bar_test.html")
```

**3: jupyter notebook, Zeppelin等直接展示图(动态的)**

当配置为ZEPPELIN 但在jupyternotebook上运行时,会把html的源文件打印出来(方便查看修改html)

```python
# 默认JUPYTER_NOTEBOOK一般不需配置(配置是必须在引用pyecharts.charts前) 
#可选值:JUPYTER_LAB,NTERACT,ZEPPELIN(写的时候 %python)
from pyecharts.globals import CurrentConfig,NotebookType
CurrentConfig.NOTEBOOK_TYPE = NotebookType.JUPYTER_NOTEBOOK  

from pyecharts.charts import Bar
from pyecharts.faker import Faker
from pyecharts.globals import ThemeType
def bar_ch() -> Bar:
    c = (
        Bar({"theme": ThemeType.MACARONS})
        .add_xaxis(Faker.choose())
        .add_yaxis("商家A", Faker.values())
        .add_yaxis("商家B", Faker.values())
        .set_global_opts(
            title_opts={"text": "Bar-通过 dict 进行配置", "subtext": "我也是通过 dict 进行配置的"}
        )
    )
    return c
#直接展示
bar_ch().render_notebook()
```

## 二 配置

#### 2.1 全局配置

​	**全局配置项可通过 `set_global_opts` 方法设置**

​	如图所示:

```python
from pyecharts.globals import CurrentConfig,NotebookType
CurrentConfig.NOTEBOOK_TYPE = NotebookType.JUPYTER_NOTEBOOK

import pyecharts.options as opts
from pyecharts.charts import Bar
from pyecharts.faker import Faker
from pyecharts.globals import ThemeType
from pyecharts.commons.utils import JsCode

# 常用配置说明 具体各项的参数未写全,只写了常用的,需要更加细致的图表请查阅官网,在各个配置项下补全即可
#1.初始化配置项 配置在图表名称中 在set_global_opts 配置会报错
echart=Bar(init_opts=opts.InitOpts(
            		#背景色 当下载图片为全黑时可以修改该值
                bg_color='white' 
            		# 图表主题 可选项: 默认为WHITE ,其他:LIGHT,DARK,CHALK,ESSOS,INFOGRAPHIC,MACARONS
            							# PURPLE_PASSION,ROMA,ROMANTIC,SHINE,VINTAGE,WALDEN,WESTEROS
            							# WONDERLAND, 也可自定义主题具体参考官网
                , theme=ThemeType.LIGHT
  							# 动画配置项
            		animation_opts=opts.AnimationOpts(
                  animation=True,
                ),
           ))
echart.set_global_opts(
          #2.工具栏
          toolbox_opts=opts.ToolboxOpts(
                    is_show=True
                    # 工具栏 icon 的布局朝向 可选：'horizontal', 'vertical'
                    ,orient="horizontal"
                    #位置
#                     ,pos_left='center'
#                     ,pos_right='center'
#                     ,pos_top='top'
#                     ,pos_bottom='top'
                    #各工具配置项
                    ,feature=opts.ToolBoxFeatureOpts(
                        #保存为图片的格式
                        save_as_image=opts.ToolBoxFeatureSaveAsImageOpts(type_="jpeg",is_show=True)
                         #数据视图
                        ,data_view = opts.ToolBoxFeatureDataViewOpts(is_show=True)
                        # 数据区域缩放。（目前只支持直角坐标系的缩放）
                        ,data_zoom=opts.ToolBoxFeatureDataZoomOpts(is_show=False)
                        #动态类型切换。
                        ,magic_type=opts.ToolBoxFeatureMagicTypeOpts(is_show=True)
                        #选框组件的控制按钮。
                        ,brush=opts.ToolBoxFeatureBrushOpts(type_="clear")
                        #还原
                        ,restore = opts.ToolBoxFeatureRestoreOpts()
                    )
          ),
          #3.提示框 点击图标时
          tooltip_opts=opts.TooltipOpts(
                    is_show=True
                    # 触发类型 
                    # 'item': 数据项图形触发，主要在散点图，饼图等无类目轴的图表中使用。
                    # 'axis': 坐标轴触发，主要在柱状图，折线图等会使用类目轴的图表中使用。
                    # 'none': 什么都不触发
                    ,trigger="axis"
                    #提示框触发的条件
                    # 'mousemove': 鼠标移动时触发。
                    # 'click': 鼠标点击时触发。
                    # 'mousemove|click': 同时鼠标移动和点击时触发。
                    # 'none': 不在 'mousemove' 或 'click' 时触发，
                    ,trigger_on="mousemove"
                    #指示器类型
                    # 'line'：直线指示器
                    # 'shadow'：阴影指示器
                    # 'none'：无指示器
                    # 'cross'：十字准星指示器。其实是种简写，表示启用两个正交的轴的 axisPointer。
                    ,axis_pointer_type="cross"
                    #提示框浮层的边框颜色。
                    ,border_color="#FF0000"
                    #提示框浮层的边框宽
                    ,border_width=5
                    # 标签内容格式器
                    # {a}：系列名。
                    # {b}：数据名。
                    # {c}：数据值。
                    ,formatter="{a}:{b}:{c}" + '%'
          ),
          # 4.AxisOpts：坐标轴配置项 x和y
           xaxis_opts=opts.AxisOpts(type_="category",is_show=True,name="类别"),
           yaxis_opts=opts.AxisOpts(
                    name="y轴名称",
                    is_show=True,
                    # 坐标轴类型 'value': 数值轴，适用于连续数据。
                    #'category': 类目轴，适用于离散的类目数据，为该类型时必须通过 data 设置类目数据
                    #'time': 时间轴，适用于连续的时序数据，与数值轴相比时间轴带有时间的格式化，在刻度计算上也有所不同，
                    #'log' 对数轴。适用于对数数据。
                    type_="value",
                    # min_=0,
                    # max_=10000,
                    # interval=500,
                    # axislabel_opts=opts.LabelOpts(formatter="{value}"),
               
                    #坐标轴刻度 默认显示 可不配置
                    axistick_opts=opts.AxisTickOpts(
                        is_show=True
                        ,is_align_with_label=True
                    ),
                    # 分割线配置项 搭配x的分割线可以组成网格
                    splitline_opts=opts.SplitLineOpts(
                        is_show=True
                        
                    ),
                    # 坐标轴刻度线配置项
                    axisline_opts=opts.AxisLineOpts(
                        is_show=True
                        #显示箭头 arrow 
                        ,symbol=None
                    ),
                    #添加y轴刻度图例说明 如 10% 20% ....
                    axislabel_opts=opts.LabelOpts(formatter="{value}"+f""),
                    # 坐标轴名字反转 一般用于x轴 填入反转度数
                    name_rotate=45,
                       
            ),
            # 5.图例配置项
           legend_opts=opts.LegendOpts(
                  is_show=True,
                   #图例位置 left_right:'left', 'center', 'right'
                   #图例位置 top_bottom:'top', 'middle', 'bottom' 而且也可天20 像素值,或'20%'容器高宽的百分比
                  pos_left='right',
                  pos_right='right',
                  pos_top='middle',
                  pos_bottom='middle',
                   #图例布局朝向 vertical:竖向 horizontal:横向
                  orient='vertical',
                   #图例icon 'circle', 'rect', 'roundRect', 'triangle', 'diamond', 'pin', 'arrow', 'none'
                  legend_icon="roundRect",
            ),
            # 6.标题配置项
            title_opts=opts.TitleOpts(
                   title="测试-主标题"
                   ,subtitle="测试-副标题"
                   ,pos_left='center'
                   ,pos_right='center'
                   ,pos_top='top'
                   ,pos_bottom='top'
                   ,title_textstyle_opts=opts.TextStyleOpts(
                       color="Green"
                       #文字字体的风格 'normal'，'italic'，'oblique'
                       ,font_style="italic"
                       #主标题文字字体的粗细 'normal'，'bold'，'bolder'，'lighter'
                       ,font_weight='bold'
                       #文字的字体大小
                       ,font_size=30
                   )
                   ,subtitle_textstyle_opts=opts.TextStyleOpts(
                       color="Red"
                       ,font_style="oblique"
                       ,font_weight='lighter'
                   )
            ),
            
            # 7. 区域缩放配置 对于如饼图等 无缩放的不可用
            datazoom_opts=opts.DataZoomOpts(
                is_show=True
                # 组件类别 可选 有筛选框:"slider", 无筛选框依赖鼠标滑动:"inside"
                ,type_="slider"
                # 水平或竖直 水平:'horizontal', 竖直:'vertical' 水平控制x轴 竖直控制y轴
                ,orient="horizontal"
           
            ),
            # 8 视觉映射配置项 一般用于散点图 根据数据大小显示大小不同的点
            visualmap_opts=opts.VisualMapOpts(
                is_show=True
                # 映射过渡类型，可选，"color", "size"
                ,type_="color"
                # 水平:horizontal 竖直:vertical
                ,orient="vertical"
                #位置
                ,pos_left='left'
                ,pos_right='left'
                ,pos_top='middle'
                ,pos_bottom='middle'
              	# 视觉范围
                ,range_color=["#D7DA8B", "#E15457"]
            ),
        
    )
```

#### 2.2 系列配置项

可以通过**set_series_opts** 配置

```python
# 系列配置项 具体需要参考各个图的参数 直角坐标系图和饼图等有些是不太一致的 有时会导致图片运行不出来,此时根据图表删掉一些配置即可
echart.set_series_opts(
		#标签配置项 折线图 柱状图等 每个点的值
        label_opts=opts.LabelOpts(
            is_show=True,
            position="inside",
            color="black",
            # 配置柱状堆积图的百分比 需要加载js文件 
          #数据格式y值2 list2 = [{"value": 12, "percent": 12 / (12 + 3)},...]
#             formatter=JsCode(
#                 "function(x){return Number(x.data.value).toFixed() +'&' + Number(x.data.percent * 100).toFixed() + '%';}"
#             ),
            # 常规 
            # 折线（区域）图、柱状（条形）图、K线图 : {a}（系列名称），{b}（类目值），{c}（数值）, {d}（无）
            # 散点图（气泡）图 : {a}（系列名称），{b}（数据名称），{c}（数值数组）, {d}（无）
            # 地图 : {a}（系列名称），{b}（区域名称），{c}（合并数值）, {d}（无）
            # 饼图、仪表盘、漏斗图: {a}（系列名称），{b}（数据项名称），{c}（数值）, {d}（百分比）
            # 示例：formatter: '{b}: {@score}'
            formatter="{b}:{c}" + '%'
        ),
        #标记点配置项
#         markpoint_opts=opts.MarkPointOpts(
#      			# 'min' 最大值。
#   			  # 'max' 最大值。
#   			  # 'average' 平均值。 可以填写多个
#             data=[opts.MarkPointItem(type_="max",name="最大值")]
#         ),
        # 标记线配置项
#         markline_opts=opts.MarkLineOpts(
#             data=[
#                 opts.MarkLineItem(type_="min", name="最小值"),
#                 opts.MarkLineItem(type_="max", name="最大值"),
#                 opts.MarkLineItem(type_="average", name="平均值"),
#             ]
#         ),
        #涟漪特效 一般用户散点图
        effect_opts=opts.EffectOpts(
            is_show=False
        ),
  			# 区域填充样式配置项 如折线图 变面积图
  			areastyle_opts=opts.AreaStyleOpts(
          #图形透明度。支持从 0 到 1 的数字，为 0 时不绘制该图形。
          opacity=0.5
        ),
)
```

这一块要配合具体的图表解释说明,系列配置项可以先参考官网:https://pyecharts.org/#/zh-cn/series_options

### 三 图表

**概述**:目前只编写了自己工作用到的图形,上面的全局和系列配置项基本可以共用,但某些特殊图形如3d图等需要特定函数修饰,该系列特殊项请参考官网,或后期本文更新会补全

#### 3.1 直角坐标系图表

```tex
Bar(柱状图)
,Boxplot(箱型图)
,EffectScatter(涟漪特效散点图)
,HeatMap(热力图)
,Kline/Candlestick(K线图)
,Line(折线/面积图)
,PictorialBar(象形柱状图)
,Scatter(散点图) 
注释: 多图层叠:Overlap
```

##### 3.1.1 公有方法

直角坐标系图表继承自 `RectChart` 都拥有以下方法

```python
# 扩展x/y轴
def extend_axis(
    # 扩展 X 坐标轴数据项
    xaxis_data: Sequence = None,
    # 扩展 X 坐标轴配置项，参考 `global_options.AxisOpts`
    xaxis: Union[opts.AxisOpts, dict, None] = None,
    # 新增 Y 坐标轴配置项，参考 `global_options.AxisOpts` 具体参考上面全局配置的y轴配置
    yaxis: Union[opts.AxisOpts, dict, None] = None,
)
# 添加x轴
def add_xaxis(
    # X 轴数据项
    xaxis_data: Sequence
)
#反转x/y轴 如竖柱状图 转 横柱状图
def reversal_axis():
# 层叠多图
def overlap(
    # chart 为直角坐标系类型图表
    chart: Base
)
# 添加dataset组件
def add_dataset(
    # 原始数据。一般来说，原始数据表达的是二维表。
    source: types.Union[types.Sequence, types.JSFunc] = None,

    # 使用 dimensions 定义 series.data 或者 dataset.source 的每个维度的信息。
    dimensions: types.Optional[types.Sequence] = None,

    # dataset.source 第一行/列是否是 维度名 信息。可选值：
    # null/undefine（对应 Python 的 None 值）：默认，自动探测。
    # true：第一行/列是维度名信息。
    # false：第一行/列直接开始是数据。
    source_header: types.Optional[bool] = None,
)
```

```python
# 添加数据集的操作 dataset  支持直接添加数据集 进行绘图
# 1 柱状图
from pyecharts import options as opts
from pyecharts.charts import Bar
c = (
    Bar()
    .add_dataset(
        source=[
            ["score", "amount", "product"],
            [89.3, 58212, "Matcha Latte"],
            [57.1, 78254, "Milk Tea"],
            [74.4, 41032, "Cheese Cocoa"],
            [50.1, 12755, "Cheese Brownie"],
            [89.7, 20145, "Matcha Cocoa"],
            [68.1, 79146, "Tea"],
            [19.6, 91852, "Orange Juice"],
            [10.6, 101852, "Lemon Juice"],
            [32.7, 20112, "Walnut Brownie"],
        ]
    )
    .add_yaxis(
        series_name="",
        y_axis=[],
        encode={"x":"amount","y":"product"},
        label_opts=opts.LabelOpts(is_show=False),
    )
    .set_global_opts(
        title_opts=opts.TitleOpts(title="Dataset normal bar example"),
        #xaxis_opts=opts.AxisOpts(name="amount"),
        yaxis_opts=opts.AxisOpts(type_="category"),
        visualmap_opts=opts.VisualMapOpts(
            orient="horizontal",
            pos_left="center",
            min_=10,
            max_=100,
            range_text=["High Score", "Low Score"],
            dimension=0,
            range_color=["#D7DA8B", "#E15457"],
        ),
    )
)
c.render_notebook()

## 2  柱图2

from pyecharts import options as opts
from pyecharts.charts import Bar

c = (
    Bar()
    .add_dataset(
        source=[
            ["product", "2015", "2016", "2017"],
            ["Matcha Latte", 43.3, 85.8, 93.7],
            ["Milk Tea", 83.1, 73.4, 55.1],
            ["Cheese Cocoa", 86.4, 65.2, 82.5],
            ["Walnut Brownie", 72.4, 53.9, 39.1],
        ]
    )
    .add_yaxis(series_name="2015", y_axis=[])
    .add_yaxis(series_name="2016", y_axis=[])
    .add_yaxis(series_name="2017", y_axis=[])
    .set_global_opts(
        title_opts=opts.TitleOpts(title="Dataset simple bar example"),
        xaxis_opts=opts.AxisOpts(type_="category"),
    )
)
c.render_notebook()

## 3  饼图
from pyecharts import options as opts
from pyecharts.charts import Pie

c = (
    Pie()
    .add_dataset(
        source=[
            ["product", "2012", "2013", "2014", "2015", "2016", "2017"],
            ["Matcha Latte", 41.1, 30.4, 65.1, 53.3, 83.8, 98.7],
            ["Milk Tea", 86.5, 92.1, 85.7, 83.1, 73.4, 55.1],
            ["Cheese Cocoa", 24.1, 67.2, 79.5, 86.4, 65.2, 82.5],
            ["Walnut Brownie", 55.2, 67.1, 69.2, 72.4, 53.9, 39.1],
        ]
    )
    .add(
        series_name="Matcha Latte",
        data_pair=[],
        radius=60,
        center=["25%", "30%"],
        encode={"itemName": "product", "value": "2012"},
    )
    .add(
        series_name="Milk Tea",
        data_pair=[],
        radius=60,
        center=["75%", "30%"],
        encode={"itemName": "product", "value": "2013"},
    )
    .add(
        series_name="Cheese Cocoa",
        data_pair=[],
        radius=60,
        center=["25%", "75%"],
        encode={"itemName": "product", "value": "2014"},
    )
    .add(
        series_name="Walnut Brownie",
        data_pair=[],
        radius=60,
        center=["75%", "75%"],
        encode={"itemName": "product", "value": "2015"},
    )
    .set_global_opts(
        title_opts=opts.TitleOpts(title="Dataset simple pie example"),
        legend_opts=opts.LegendOpts(pos_left="30%", pos_top="2%"),
    )
    
)
c.render_notebook()
```



##### 3.1.2 柱状图

```python
class Bar(
    # 初始化配置项，参考 `global_options.InitOpts`
    init_opts: opts.InitOpts = opts.InitOpts()
)
# 函数
def add_yaxis(
    # 系列名称，用于 tooltip 的显示，legend 的图例筛选。
    series_name: str,
    # 系列数据
    y_axis: Sequence[Numeric, opts.BarItem, dict],
    # 是否选中图例
    is_selected: bool = True,
    # 使用的 x 轴的 index，在单个图表实例中存在多个 x 轴的时候有用。
    xaxis_index: Optional[Numeric] = None,
    # 使用的 y 轴的 index，在单个图表实例中存在多个 y 轴的时候有用。
    yaxis_index: Optional[Numeric] = None,
    # 是否启用图例 hover 时的联动高亮
    is_legend_hover_link: bool = True,
    # 系列 label 颜色
    color: Optional[str] = None,
    # 是否显示柱条的背景色。通过 backgroundStyle 配置背景样式。
    is_show_background: bool = False,
    # 每一个柱条的背景样式。需要将 showBackground 设置为 true 时才有效。
    background_style: types.Union[types.BarBackground, dict, None] = None,
    # 数据堆叠，同个类目轴上系列配置相同的　stack　值可以堆叠放置。
    stack: Optional[str] = None,
    # 柱条的宽度，不设时自适应。
    # 可以是绝对值例如 40 或者百分数例如 '60%'。百分数基于自动计算出的每一类目的宽度。
    # 在同一坐标系上，此属性会被多个 'bar' 系列共享。此属性应设置于此坐标系中最后一个 'bar' 系列上才会生效，并且是对此坐标系中所有 'bar' 系列生效。
    bar_width: types.Union[types.Numeric, str] = None,
    # 柱条的最大宽度。比 barWidth 优先级高。
    bar_max_width: types.Union[types.Numeric, str] = None,
    # 柱条的最小宽度。在直角坐标系中，默认值是 1。否则默认值是 null。比 barWidth 优先级高。
    bar_min_width: types.Union[types.Numeric, str] = None,
    # 柱条最小高度，可用于防止某数据项的值过小而影响交互。
    bar_min_height: types.Numeric = 0,
    # 同一系列的柱间距离，默认为类目间距的 20%，可设固定值
    category_gap: Union[Numeric, str] = "20%",
    # 不同系列的柱间距离，为百分比（如 '30%'，表示柱子宽度的 30%）。
    # 如果想要两个系列的柱子重叠，可以设置 gap 为 '-100%'。这在用柱子做背景的时候有用。
    gap: Optional[str] = "30%",
    # 是否开启大数据量优化，在数据图形特别多而出现卡顿时候可以开启。
    # 开启后配合 largeThreshold 在数据量大于指定阈值的时候对绘制进行优化。
    # 缺点：优化后不能自定义设置单个数据项的样式。
    is_large: bool = False,
    # 开启绘制优化的阈值。
    large_threshold: types.Numeric = 400,
    # 使用 dimensions 定义 series.data 或者 dataset.source 的每个维度的信息。
    # 注意：如果使用了 dataset，那么可以在 dataset.source 的第一行/列中给出 dimension 名称。
    # 于是就不用在这里指定 dimension。
    # 但是，如果在这里指定了 dimensions，那么 ECharts 不再会自动从 dataset.source 的第一行/列中获取维度信息。
    dimensions: types.Union[types.Sequence, None] = None,
    # 当使用 dataset 时，seriesLayoutBy 指定了 dataset 中用行还是列对应到系列上，也就是说，系列“排布”到 dataset 的行还是列上。可取值：
    # 'column'：默认，dataset 的列对应于系列，从而 dataset 中每一列是一个维度（dimension）。
    # 'row'：dataset 的行对应于系列，从而 dataset 中每一行是一个维度（dimension）。
    series_layout_by: str = "column",
    # 如果 series.data 没有指定，并且 dataset 存在，那么就会使用 dataset。
    # datasetIndex 指定本系列使用那个 dataset。
    dataset_index: types.Numeric = 0,
    # 是否裁剪超出坐标系部分的图形。柱状图：裁掉所有超出坐标系的部分，但是依然保留柱子的宽度
    is_clip: bool = True,
    # 柱状图所有图形的 zlevel 值。
    z_level: types.Numeric = 0,
    # 柱状图组件的所有图形的z值。控制图形的前后顺序。
    # z值小的图形会被z值大的图形覆盖。
    # z相比zlevel优先级更低，而且不会创建新的 Canvas。
    z: types.Numeric = 2,
    # 标签配置项，参考 `series_options.LabelOpts`
    label_opts: Union[opts.LabelOpts, dict] = opts.LabelOpts(),

    # 标记点配置项，参考 `series_options.MarkPointOpts`
    markpoint_opts: Union[opts.MarkPointOpts, dict, None] = None,

    # 标记线配置项，参考 `series_options.MarkLineOpts`
    markline_opts: Union[opts.MarkLineOpts, dict, None] = None,

    # 提示框组件配置项，参考 `series_options.TooltipOpts`
    tooltip_opts: Union[opts.TooltipOpts, dict, None] = None,

    # 图元样式配置项，参考 `series_options.ItemStyleOpts`
    itemstyle_opts: Union[opts.ItemStyleOpts, dict, None] = None,

    # 可以定义 data 的哪个维度被编码成什么。
    encode: types.Union[types.JSFunc, dict, None] = None,
)
```

**例子**

```python
from pyecharts.globals import CurrentConfig,NotebookType
CurrentConfig.NOTEBOOK_TYPE = NotebookType.JUPYTER_NOTEBOOK

import pyecharts.options as opts
from pyecharts.charts import Bar
from pyecharts.faker import Faker
from pyecharts.globals import ThemeType
from pyecharts.commons.utils import JsCode

list2 = [
    {"value": 12, "percent": 12 / (12 + 3)},
    {"value": 23, "percent": 23 / (23 + 21)},
    {"value": 33, "percent": 33 / (33 + 5)},
    {"value": 3, "percent": 3 / (3 + 52)},
    {"value": 33, "percent": 33 / (33 + 43)},
]

list3 = [
    {"value": 3, "percent": 3 / (12 + 3)},
    {"value": 21, "percent": 21 / (23 + 21)},
    {"value": 5, "percent": 5 / (33 + 5)},
    {"value": 52, "percent": 52 / (3 + 52)},
    {"value": 43, "percent": 43 / (33 + 43)},
]

bar = (
   Bar(init_opts=opts.InitOpts(
                bg_color='white'
                ,theme=ThemeType.LIGHT
                ,animation_opts=opts.AnimationOpts(
                  animation=True,
                ),
        ))
   .add_xaxis(Faker.choose())
   .add_yaxis("商家A", Faker.values())
   .add_yaxis("商家B", Faker.values())
#    .add_xaxis([1,2,3,4.5])
#    .add_yaxis("商家A", list2,stack="stack")
#    .add_yaxis("商家B", list3,stack="stack")
   .set_global_opts(
          #2.工具栏
          toolbox_opts=opts.ToolboxOpts(
                    is_show=True
                    # 工具栏 icon 的布局朝向 可选：'horizontal', 'vertical'
                    ,orient="horizontal"
                    #位置
#                     ,pos_left='center'
#                     ,pos_right='center'
#                     ,pos_top='top'
#                     ,pos_bottom='top'
                    #各工具配置项
                    ,feature=opts.ToolBoxFeatureOpts(
                        #保存为图片的格式
                        save_as_image=opts.ToolBoxFeatureSaveAsImageOpts(type_="jpeg",is_show=True)
                         #数据视图
                        ,data_view = opts.ToolBoxFeatureDataViewOpts(is_show=True)
                        # 数据区域缩放。（目前只支持直角坐标系的缩放）
                        ,data_zoom=opts.ToolBoxFeatureDataZoomOpts(is_show=False)
                        #动态类型切换。
                        ,magic_type=opts.ToolBoxFeatureMagicTypeOpts(is_show=True)
                        #选框组件的控制按钮。
                        ,brush=opts.ToolBoxFeatureBrushOpts(type_="clear")
                        #还原
                        ,restore = opts.ToolBoxFeatureRestoreOpts()
                    )
          ),
          #3.提示框 点击图标时
          tooltip_opts=opts.TooltipOpts(
                    is_show=True
                    # 触发类型 
                    # 'item': 数据项图形触发，主要在散点图，饼图等无类目轴的图表中使用。
                    # 'axis': 坐标轴触发，主要在柱状图，折线图等会使用类目轴的图表中使用。
                    # 'none': 什么都不触发
                    ,trigger=None
                    #提示框触发的条件
                    # 'mousemove': 鼠标移动时触发。
                    # 'click': 鼠标点击时触发。
                    # 'mousemove|click': 同时鼠标移动和点击时触发。
                    # 'none': 不在 'mousemove' 或 'click' 时触发，
                    ,trigger_on="mousemove"
                    #指示器类型
                    # 'line'：直线指示器
                    # 'shadow'：阴影指示器
                    # 'none'：无指示器
                    # 'cross'：十字准星指示器。其实是种简写，表示启用两个正交的轴的 axisPointer。
                    ,axis_pointer_type="cross"
                    #提示框浮层的边框颜色。
                    ,border_color="#FF0000"
                    #提示框浮层的边框宽
                    ,border_width=5
                    # 标签内容格式器
                    # {a}：系列名。
                    # {b}：数据名。
                    # {c}：数据值。
                    ,formatter="{a}:{b}:{c}" + '%'
          ),
          # 4.AxisOpts：坐标轴配置项 x和y
           xaxis_opts=opts.AxisOpts(type_="category",is_show=True,name="类别"),
           yaxis_opts=opts.AxisOpts(
                    name="y轴名称",
                    is_show=True,
                    # 坐标轴类型 'value': 数值轴，适用于连续数据。
                    #'category': 类目轴，适用于离散的类目数据，为该类型时必须通过 data 设置类目数据
                    #'time': 时间轴，适用于连续的时序数据，与数值轴相比时间轴带有时间的格式化，在刻度计算上也有所不同，
                    #'log' 对数轴。适用于对数数据。
                    type_="value",
                    # min_=0,
                    # max_=10000,
                    # interval=500,
                    # axislabel_opts=opts.LabelOpts(formatter="{value}"),
               
                    #坐标轴刻度 默认显示 可不配置
                    axistick_opts=opts.AxisTickOpts(
                        is_show=True
                        ,is_align_with_label=True
                    ),
                    # 分割线配置项 搭配x的分割线可以组成网格
                    splitline_opts=opts.SplitLineOpts(
                        is_show=True
                        
                    ),
                    # 坐标轴刻度线配置项
                    axisline_opts=opts.AxisLineOpts(
                        is_show=True
                        #显示箭头 arrow 
                        ,symbol=None
                    ),
                    #添加y轴刻度图例说明 如 10% 20% ....
                    axislabel_opts=opts.LabelOpts(formatter="{value}"+f""),
                    # 坐标轴名字反转 一般用于x轴 填入反转度数
                    name_rotate=45,
                       
            ),
            # 5.图例配置项
           legend_opts=opts.LegendOpts(
                  is_show=True,
                   #图例位置 left_right:'left', 'center', 'right'
                   #图例位置 top_bottom:'top', 'middle', 'bottom' 而且也可天20 像素值,或'20%'容器高宽的百分比
                  pos_left='right',
                  pos_right='right',
                  pos_top='middle',
                  pos_bottom='middle',
                   #图例布局朝向 vertical:竖向 horizontal:横向
                  orient='vertical',
                   #图例icon 'circle', 'rect', 'roundRect', 'triangle', 'diamond', 'pin', 'arrow', 'none'
                  legend_icon="roundRect",
            ),
            # 6.标题配置项
            title_opts=opts.TitleOpts(
                   title="测试-主标题"
                   ,subtitle="测试-副标题"
                   ,pos_left='center'
                   ,pos_right='center'
                   ,pos_top='top'
                   ,pos_bottom='top'
                   ,title_textstyle_opts=opts.TextStyleOpts(
                       color="Green"
                       #文字字体的风格 'normal'，'italic'，'oblique'
                       ,font_style="italic"
                       #主标题文字字体的粗细 'normal'，'bold'，'bolder'，'lighter'
                       ,font_weight='bold'
                       #文字的字体大小
                       ,font_size=30
                   )
                   ,subtitle_textstyle_opts=opts.TextStyleOpts(
                       color="Red"
                       ,font_style="oblique"
                       ,font_weight='lighter'
                   )
            ),
            
            # 7. 区域缩放配置
            datazoom_opts=opts.DataZoomOpts(
                is_show=True
                # 组件类别 可选 有筛选框:"slider", 无筛选框依赖鼠标滑动:"inside"
                ,type_="slider"
                # 水平或竖直 水平:'horizontal', 竖直:'vertical' 水平控制x轴 竖直控制y轴
                ,orient="horizontal"
           
            ),
            # 8 视觉映射配置项
            visualmap_opts=opts.VisualMapOpts(
                is_show=True
                # 映射过渡类型，可选，"color", "size"
                ,type_="black"
                # 水平:horizontal 竖直:vertical
                ,orient="vertical"
                #位置
                ,pos_left='left'
                ,pos_right='left'
                ,pos_top='middle'
                ,pos_bottom='middle'
            ),
        
    )
    .set_series_opts(
        #标签配置项 折线图 柱状图等 每个点的值
        label_opts=opts.LabelOpts(
            is_show=True,
            position="inside",
            color="black",
            # 配置柱状堆积图的百分比 需要加载js文件
#             formatter=JsCode(
#                 "function(x){return Number(x.data.value).toFixed() +'&' + Number(x.data.percent * 100).toFixed() + '%';}"
#             ),
            # 常规 
            # 折线（区域）图、柱状（条形）图、K线图 : {a}（系列名称），{b}（类目值），{c}（数值）, {d}（无）
            # 散点图（气泡）图 : {a}（系列名称），{b}（数据名称），{c}（数值数组）, {d}（无）
            # 地图 : {a}（系列名称），{b}（区域名称），{c}（合并数值）, {d}（无）
            # 饼图、仪表盘、漏斗图: {a}（系列名称），{b}（数据项名称），{c}（数值）, {d}（百分比）
            # 示例：formatter: '{b}: {@score}'
            formatter="{a}:{c}" + '%'
        ),
        #标记点配置项
#         markpoint_opts=opts.MarkPointOpts(
#             data=[opts.MarkPointItem(type_="max",name="最大值")]
#         ),
        # 标记线配置项
#         markline_opts=opts.MarkLineOpts(
#             data=[
#                 opts.MarkLineItem(type_="min", name="最小值"),
#                 opts.MarkLineItem(type_="max", name="最大值"),
#                 opts.MarkLineItem(type_="average", name="平均值"),
#             ]
#         ),
        #涟漪特效 一般用户散点图
        effect_opts=opts.EffectOpts(
            is_show=False
        )
    )
)
bar.add_js_funcs("console.log('hello world')")
#bar.render("js.html")
bar.render_notebook()
#bar.get_options()
# bar.dump_options()
# bar.dump_options_with_quotes()
```

##### 3.1.3 折线图

```python
class Line(
		# 初始化配置项，参考 `global_options.InitOpts`
    init_opts: opts.InitOpts = opts.InitOpts()
)

def add_yaxis(
    # 系列名称，用于 tooltip 的显示，legend 的图例筛选。
    series_name: str,

    # 系列数据
    y_axis: types.Sequence[types.Union[opts.LineItem, dict]],

    # 是否选中图例
    is_selected: bool = True,

    # 是否连接空数据，空数据使用 `None` 填充
    is_connect_nones: bool = False,

    # 使用的 x 轴的 index，在单个图表实例中存在多个 x 轴的时候有用。
    xaxis_index: Optional[Numeric] = None,

    # 使用的 y 轴的 index，在单个图表实例中存在多个 y 轴的时候有用。
    yaxis_index: Optional[Numeric] = None,

    # 系列 label 颜色
    color: Optional[str] = None,
    # 是否显示 symbol, 如果 false 则只有在 tooltip hover 的时候显示。
    is_symbol_show: bool = True,
    # 标记的图形。
    # ECharts 提供的标记类型包括 'circle', 'rect', 'roundRect', 'triangle', 
    # 'diamond', 'pin', 'arrow', 'none'
    # 可以通过 'image://url' 设置为图片，其中 URL 为图片的链接，或者 dataURI。
    symbol: Optional[str] = None,
    # 标记的大小，可以设置成诸如 10 这样单一的数字，也可以用数组分开表示宽和高，
    # 例如 [20, 10] 表示标记宽为 20，高为 10。
    symbol_size: Union[Numeric, Sequence] = 4,
    # 数据堆叠，同个类目轴上系列配置相同的　stack　值可以堆叠放置。
    stack: Optional[str] = None,
    # 是否平滑曲线
    is_smooth: bool = False,
    # 是否裁剪超出坐标系部分的图形。折线图：裁掉所有超出坐标系的折线部分，拐点图形的逻辑按照散点图处理
    is_clip: bool = True,
    # 是否显示成阶梯图
    is_step: bool = False,
    # 是否开启 hover 在拐点标志上的提示动画效果。
    is_hover_animation: bool = True,
    # 折线图所有图形的 zlevel 值。
    # zlevel用于 Canvas 分层，不同zlevel值的图形会放置在不同的 Canvas 中，Canvas 分层是一种常见的优化手段。
    # zlevel 大的 Canvas 会放在 zlevel 小的 Canvas 的上面。
    z_level: types.Numeric = 0,
    # 折线图组件的所有图形的z值。控制图形的前后顺序。z值小的图形会被z值大的图形覆盖。
    # z 相比 zlevel 优先级更低，而且不会创建新的 Canvas。
    z: types.Numeric = 0,
    # 标记点配置项，参考 `series_options.MarkPointOpts`
    markpoint_opts: Union[opts.MarkPointOpts, dict, None] = None,
    # 标记线配置项，参考 `series_options.MarkLineOpts`
    markline_opts: Union[opts.MarkLineOpts, dict, None] = None,
    # 提示框组件配置项，参考 `series_options.TooltipOpts`
    tooltip_opts: Union[opts.TooltipOpts, dict, None] = None,
    # 标签配置项，参考 `series_options.LabelOpts`
    label_opts: Union[opts.LabelOpts, dict] = opts.LabelOpts(),
    # 线样式配置项，参考 `series_options.LineStyleOpts`
    linestyle_opts: Union[opts.LineStyleOpts, dict] = opts.LineStyleOpts(),
    # 填充区域配置项，参考 `series_options.AreaStyleOpts`
    areastyle_opts: Union[opts.AreaStyleOpts, dict] = opts.AreaStyleOpts(),
    # 图元样式配置项，参考 `series_options.ItemStyleOpts`
    itemstyle_opts: Union[opts.ItemStyleOpts, dict, None] = None,
)
```

**例子:**

```python
import pyecharts.options as opts
from pyecharts.charts import Line
from pyecharts.faker import Faker

c = (
    Line()
    .add_xaxis(Faker.choose())
    .add_yaxis("商家A", Faker.values(), is_smooth=True)
    .add_yaxis("商家B", Faker.values(), is_smooth=True)
    .set_series_opts(
        areastyle_opts=opts.AreaStyleOpts(opacity=0.5),
        label_opts=opts.LabelOpts(is_show=False),
    )
    .set_global_opts(
        title_opts=opts.TitleOpts(title="Line-面积图（紧贴 Y 轴）"),
        xaxis_opts=opts.AxisOpts(
            axistick_opts=opts.AxisTickOpts(is_align_with_label=True),
            is_scale=False,
            boundary_gap=False,
        ),
    )
    
)
c.render_notebook()
```



##### 3.1.4 箱型图

```python
class Boxplot(
		# 初始化配置项，参考 `global_options.InitOpts`
    init_opts: opts.InitOpts = opts.InitOpts()
)
def add_yaxis(
    # 系列名称，用于 tooltip 的显示，legend 的图例筛选。
    series_name: str,
    # 系列数据
    y_axis: types.Sequence[types.Union[opts.BoxplotItem, dict]],
    # 是否选中图例
    is_selected: bool = True,
    # 使用的 x 轴的 index，在单个图表实例中存在多个 x 轴的时候有用。
    xaxis_index: Optional[Numeric] = None,
    # 使用的 y 轴的 index，在单个图表实例中存在多个 y 轴的时候有用。
    yaxis_index: Optional[Numeric] = None,
    # 标签配置项，参考 `series_options.LabelOpts`
    label_opts: Union[opts.LabelOpts, dict] = opts.LabelOpts(),
    # 标记点配置项，参考 `series_options.MarkPointOpts`
    markpoint_opts: Union[opts.MarkPointOpts, dict] = opts.MarkPointOpts(),
    # 标记线配置项，参考 `series_options.MarkLineOpts`
    markline_opts: Union[opts.MarkLineOpts, dict] = opts.MarkLineOpts(),
    # 提示框组件配置项，参考 `series_options.TooltipOpts`
    tooltip_opts: Union[opts.TooltipOpts, dict, None] = None,
    # 图元样式配置项，参考 `series_options.ItemStyleOpts`
    itemstyle_opts: Union[opts.ItemStyleOpts, dict, None] = None,
)
```

**例子:**

```python
from pyecharts import options as opts
from pyecharts.charts import Boxplot

v1 = [
    [850, 740, 900, 1070, 930, 850, 950, 980, 980, 880, 1000, 980],
    [960, 940, 960, 940, 880, 800, 850, 880, 900, 840, 830, 790],
]
v2 = [
    [890, 810, 810, 820, 800, 770, 760, 740, 750, 760, 910, 920],
    [890, 840, 780, 810, 760, 810, 790, 810, 820, 850, 870, 870],
]
c = Boxplot()
c.add_xaxis(["expr1", "expr2"])
c.add_yaxis("A", c.prepare_data(v1))
c.add_yaxis("B", c.prepare_data(v2))
c.set_global_opts(title_opts=opts.TitleOpts(title="BoxPlot-基本示例"))
c.render_notebook()
```



##### 3.1.6 热力图

```python
class HeatMap(
    # 初始化配置项，参考 `global_options.InitOpts`
    init_opts: opts.InitOpts = opts.InitOpts()
)
def add_yaxis(
    # 系列名称，用于 tooltip 的显示，legend 的图例筛选。
    series_name: str,
    # Y 坐标轴数据
    yaxis_data: types.Sequence[types.Union[opts.HeatMapItem, dict]],
    # 系列数据项
    value: types.Sequence[types.Union[opts.HeatMapItem, dict]],
    # 是否选中图例
    is_selected: bool = True,
    # 使用的 x 轴的 index，在单个图表实例中存在多个 x 轴的时候有用。
    xaxis_index: Optional[Numeric] = None,
    # 使用的 y 轴的 index，在单个图表实例中存在多个 y 轴的时候有用。
    yaxis_index: Optional[Numeric] = None,
    # 标签配置项，参考 `series_options.LabelOpts`
    label_opts: Union[opts.LabelOpts, dict] = opts.LabelOpts(),
    # 标记点配置项，参考 `series_options.MarkPointOpts`
    markpoint_opts: Union[opts.MarkPointOpts, dict, None] = None,
    # 标记线配置项，参考 `series_options.MarkLineOpts`
    markline_opts: Union[opts.MarkLineOpts, dict, None] = None,
    # 提示框组件配置项，参考 `series_options.TooltipOpts`
    tooltip_opts: Union[opts.TooltipOpts, dict, None] = None,
    # 图元样式配置项，参考 `series_options.ItemStyleOpts`
    itemstyle_opts: Union[opts.ItemStyleOpts, dict, None] = None,
)
```

**例子:**

```python
import random

from pyecharts import options as opts
from pyecharts.charts import HeatMap
from pyecharts.faker import Faker

value = [[i, j, random.randint(0, 50)] for i in range(24) for j in range(7)]
c = (
    HeatMap()
    .add_xaxis(Faker.clock)
    .add_yaxis(
        "series0",
        Faker.week,
        value,
        label_opts=opts.LabelOpts(is_show=True, position="inside"),
    )
    .set_global_opts(
        title_opts=opts.TitleOpts(title="HeatMap-Label 显示"),
        visualmap_opts=opts.VisualMapOpts(),
    )
)
c.render_notebook()
```

##### 3.1.7 散点图

```python
class Scatter(
    # 初始化配置项，参考 `global_options.InitOpts`
    init_opts: opts.InitOpts = opts.InitOpts()
)
```

**例子:**

```python
from pyecharts import options as opts
from pyecharts.charts import Scatter
from pyecharts.faker import Faker

c = (
    Scatter()
    .add_xaxis(Faker.choose())
    .add_yaxis("商家A", Faker.values())
    .set_global_opts(
        title_opts=opts.TitleOpts(title="Scatter-VisualMap(Color)"),
        visualmap_opts=opts.VisualMapOpts(max_=150),
    )    
)
c.render_notebook()
```

##### 3.1.8 层叠多图

```python
from pyecharts import options as opts
from pyecharts.charts import Bar, Line
from pyecharts.faker import Faker

v1 = [2.0, 4.9, 7.0, 23.2, 25.6, 76.7, 135.6, 162.2, 32.6, 20.0, 6.4, 3.3]
v2 = [2.6, 5.9, 9.0, 26.4, 28.7, 70.7, 175.6, 182.2, 48.7, 18.8, 6.0, 2.3]
v3 = [2.0, 2.2, 3.3, 4.5, 6.3, 10.2, 20.3, 23.4, 23.0, 16.5, 12.0, 6.2]


bar = (
    Bar()
    .add_xaxis(Faker.months)
    .add_yaxis("蒸发量", v1)
    .add_yaxis("降水量", v2)
    .extend_axis(
        yaxis=opts.AxisOpts(
            axislabel_opts=opts.LabelOpts(formatter="{value} °C"), interval=5
        )
    )
    .set_series_opts(label_opts=opts.LabelOpts(is_show=False))
    .set_global_opts(
        title_opts=opts.TitleOpts(title="Overlap-bar+line"),
        yaxis_opts=opts.AxisOpts(axislabel_opts=opts.LabelOpts(formatter="{value} ml")),
    )
)

line = Line().add_xaxis(Faker.months).add_yaxis("平均温度", v3, yaxis_index=1)
bar.overlap(line)
bar.render_notebook()
```

#### 3.2 基本图表

**跟直角坐标系图区别无 add_yaxis等相关方法**

```tex
Calendar：日历图
Funnel：漏斗图
Gauge：仪表盘
Graph：关系图
Liquid：水球图
Parallel：平行坐标系
Pie：饼图
Polar：极坐标系
Radar：雷达图
Sankey：桑基图
Sunburst：旭日图
ThemeRiver：主题河流图
WordCloud：词云图
```

##### 3.2.1 饼图

```python
class Pie(
    # 初始化配置项，参考 `global_options.InitOpts`
    init_opts: opts.InitOpts = opts.InitOpts()
)

def add(
    # 系列名称，用于 tooltip 的显示，legend 的图例筛选。
    series_name: str,
    # 系列数据项，格式为 [(key1, value1), (key2, value2)]
    data_pair: types.Sequence[types.Union[types.Sequence, opts.PieItem, dict]],
    # 系列 label 颜色
    color: Optional[str] = None,
    # 饼图的半径，数组的第一项是内半径，第二项是外半径
    # 默认设置成百分比，相对于容器高宽中较小的一项的一半
    radius: Optional[Sequence] = None,
    # 饼图的中心（圆心）坐标，数组的第一项是横坐标，第二项是纵坐标
    # 默认设置成百分比，设置成百分比时第一项是相对于容器宽度，第二项是相对于容器高度
    center: Optional[Sequence] = None,
    # 是否展示成南丁格尔图，通过半径区分数据大小，有'radius'和'area'两种模式。
    # radius：扇区圆心角展现数据的百分比，半径展现数据的大小
    # area：所有扇区圆心角相同，仅通过半径展现数据大小
    rosetype: Optional[str] = None,
    # 饼图的扇区是否是顺时针排布。
    is_clockwise: bool = True,
    # 标签配置项，参考 `series_options.LabelOpts`
    label_opts: Union[opts.LabelOpts, dict] = opts.LabelOpts(),
    # 提示框组件配置项，参考 `series_options.TooltipOpts`
    tooltip_opts: Union[opts.TooltipOpts, dict, None] = None,
    # 图元样式配置项，参考 `series_options.ItemStyleOpts`
    itemstyle_opts: Union[opts.ItemStyleOpts, dict, None] = None,
    # 可以定义 data 的哪个维度被编码成什么。
    encode: types.Union[types.JSFunc, dict, None] = None,
)
```

**例子:**

```python
from pyecharts.globals import CurrentConfig,NotebookType
CurrentConfig.NOTEBOOK_TYPE = NotebookType.JUPYTER_NOTEBOOK

import pyecharts.options as opts
from pyecharts.charts import Bar
from pyecharts.faker import Faker
from pyecharts.globals import ThemeType
from pyecharts.commons.utils import JsCode
from pyecharts.charts import Pie

c = (
    Pie()
    .add("", [list(z) for z in zip(Faker.choose(), Faker.values())])
    .set_colors(["blue", "green", "yellow", "red", "pink", "orange", "purple"])
    
    .set_series_opts(label_opts=opts.LabelOpts(formatter="{b}: {c}"))
    .set_global_opts(
          #2.工具栏
          toolbox_opts=opts.ToolboxOpts(
                    is_show=True
                    # 工具栏 icon 的布局朝向 可选：'horizontal', 'vertical'
                    ,orient="horizontal"
                    #位置
#                     ,pos_left='center'
#                     ,pos_right='center'
#                     ,pos_top='top'
#                     ,pos_bottom='top'
                    #各工具配置项
                    ,feature=opts.ToolBoxFeatureOpts(
                        #保存为图片的格式
                        save_as_image=opts.ToolBoxFeatureSaveAsImageOpts(type_="jpeg",is_show=True)
                         #数据视图
                        ,data_view = opts.ToolBoxFeatureDataViewOpts(is_show=True)
                        # 数据区域缩放。（目前只支持直角坐标系的缩放）
                        ,data_zoom=opts.ToolBoxFeatureDataZoomOpts(is_show=False)
                        #动态类型切换。
                        ,magic_type=opts.ToolBoxFeatureMagicTypeOpts(is_show=True)
                        #选框组件的控制按钮。
                        ,brush=opts.ToolBoxFeatureBrushOpts(type_="clear")
                        #还原
                        ,restore = opts.ToolBoxFeatureRestoreOpts()
                    )
          ),
          #3.提示框 点击图标时
          tooltip_opts=opts.TooltipOpts(
                    is_show=True
                    # 触发类型 
                    # 'item': 数据项图形触发，主要在散点图，饼图等无类目轴的图表中使用。
                    # 'axis': 坐标轴触发，主要在柱状图，折线图等会使用类目轴的图表中使用。
                    # 'none': 什么都不触发
                    ,trigger="axis"
                    #提示框触发的条件
                    # 'mousemove': 鼠标移动时触发。
                    # 'click': 鼠标点击时触发。
                    # 'mousemove|click': 同时鼠标移动和点击时触发。
                    # 'none': 不在 'mousemove' 或 'click' 时触发，
                    ,trigger_on="mousemove"
                    #指示器类型
                    # 'line'：直线指示器
                    # 'shadow'：阴影指示器
                    # 'none'：无指示器
                    # 'cross'：十字准星指示器。其实是种简写，表示启用两个正交的轴的 axisPointer。
                    ,axis_pointer_type="cross"
                    #提示框浮层的边框颜色。
                    ,border_color="#FF0000"
                    #提示框浮层的边框宽
                    ,border_width=5
          ),
#         
            # 5.图例配置项
           legend_opts=opts.LegendOpts(
                  is_show=True,
                   #图例位置 left_right:'left', 'center', 'right'
                   #图例位置 top_bottom:'top', 'middle', 'bottom' 而且也可天20 像素值,或'20%'容器高宽的百分比
                  pos_left='right',
                  pos_right='right',
                  pos_top='middle',
                  pos_bottom='middle',
                   #图例布局朝向 vertical:竖向 horizontal:横向
                  orient='vertical',
                   #图例icon 'circle', 'rect', 'roundRect', 'triangle', 'diamond', 'pin', 'arrow', 'none'
                  legend_icon="roundRect",
            ),
            # 6.标题配置项
            title_opts=opts.TitleOpts(
                   title="测试-主标题"
                   ,subtitle="测试-副标题"
                   ,pos_left='center'
                   ,pos_right='center'
                   ,pos_top='top'
                   ,pos_bottom='top'
                   ,title_textstyle_opts=opts.TextStyleOpts(
                       color="Green"
                       #文字字体的风格 'normal'，'italic'，'oblique'
                       ,font_style="italic"
                       #主标题文字字体的粗细 'normal'，'bold'，'bolder'，'lighter'
                       ,font_weight='bold'
                       #文字的字体大小
                       ,font_size=30
                   )
                   ,subtitle_textstyle_opts=opts.TextStyleOpts(
                       color="Red"
                       ,font_style="oblique"
                       ,font_weight='lighter'
                   )
            ),
            

            # 8 视觉映射配置项
            visualmap_opts=opts.VisualMapOpts(
                is_show=True
                # 映射过渡类型，可选，"color", "size"
                ,type_="color"
                # 水平:horizontal 竖直:vertical
                ,orient="vertical"
                #位置
                ,pos_left='left'
                ,pos_right='left'
                ,pos_top='middle'
                ,pos_bottom='middle'
            ),
        
    )
)
c.render_notebook()
```

##### 3.2.2 漏斗图

```python
class Funnel(
    # 初始化配置项，参考 `global_options.InitOpts`
    init_opts: opts.InitOpts = opts.InitOpts()
)
def add(
    # 系列名称，用于 tooltip 的显示，legend 的图例筛选。
    series_name: str,
    # 系列数据项，格式为 [(key1, value1), (key2, value2)]
    data_pair: Sequence,
    # 是否选中图例
    is_selected: bool = True,
    # 系列 label 颜色
    color: Optional[str] = None,
    # 数据排序， 可以取 'ascending'，'descending'，'none'（表示按 data 顺序）
    sort_: str = "descending",
    # 数据图形间距
    gap: Numeric = 0,
    # 标签配置项，参考 `series_options.LabelOpts`
    label_opts: Union[opts.LabelOpts, dict] = opts.LabelOpts(),
    # 提示框组件配置项，参考 `series_options.TooltipOpts`
    tooltip_opts: Union[opts.TooltipOpts, dict, None] = None,
    # 图元样式配置项，参考 `series_options.ItemStyleOpts`
    itemstyle_opts: Union[opts.ItemStyleOpts, dict, None] = None,
)
```

**例子:**

```python
from pyecharts import options as opts
from pyecharts.charts import Funnel
from pyecharts.faker import Faker

c = (
    Funnel()
    .add("商品", [list(z) for z in zip(Faker.choose(), Faker.values())])
    .set_global_opts(title_opts=opts.TitleOpts(title="Funnel-基本示例"))
)
c.render_notebook()
```





#### 3.3 树图

```tex
Tree：树图
TreeMap：矩形树图
```





#### 3.4 3D图

```tex
Bar3D：3D柱状图
Line3D：3D折线图
Scatter3D：3D散点图
Surface3D：3D曲面图
Map3D: 三维地图
```





#### 3.5 地理图

```tex
Geo：地理坐标系
Map：地图
BMap：百度地图
```



#### 3.6 组合图表

```tex
Grid：并行多图
Page：顺序多图
Tab：选项卡多图
Timeline：时间线轮播多图
```

##### 3.6.1 并行多图Grid

```python
class Grid(
    # 初始化配置项，参考 `global_options.InitOpts`
    init_opts: opts.InitOpts = opts.InitOpts()
)
def add(
    # 图表实例，仅 `Chart` 类或者其子类
    chart: Chart,
    # 直角坐标系网格配置项，参见 `GridOpts`
    grid_opts: Union[opts.GridOpts, dict],
    # 直角坐标系网格索引
    grid_index: int = 0,
    # 是否由自己控制 Axis 索引
    is_control_axis_index: bool = False,
)

class GridOpts(
    # 是否显示直角坐标系网格。
    is_show: bool = False,
    # 所有图形的 zlevel 值。
    z_level: Numeric = 0,
    # 组件的所有图形的z值。
    z: Numeric = 2,
    # grid 组件离容器左侧的距离。
    # left 的值可以是像 20 这样的具体像素值，可以是像 '20%' 这样相对于容器高宽的百分比，
    # 也可以是 'left', 'center', 'right'。
    # 如果 left 的值为'left', 'center', 'right'，组件会根据相应的位置自动对齐。
    pos_left: Union[Numeric, str, None] = None,
    # grid 组件离容器上侧的距离。
    # top 的值可以是像 20 这样的具体像素值，可以是像 '20%' 这样相对于容器高宽的百分比，
    # 也可以是 'top', 'middle', 'bottom'。
    # 如果 top 的值为'top', 'middle', 'bottom'，组件会根据相应的位置自动对齐。
    pos_top: Union[Numeric, str, None]  = None,
    # grid 组件离容器右侧的距离。
    # right 的值可以是像 20 这样的具体像素值，可以是像 '20%' 这样相对于容器高宽的百分比。
    pos_right: Union[Numeric, str, None]  = None,
    # grid 组件离容器下侧的距离。
    # bottom 的值可以是像 20 这样的具体像素值，可以是像 '20%' 这样相对于容器高宽的百分比。
    pos_bottom: Union[Numeric, str, None]  = None,
    # grid 组件的宽度。默认自适应。
    width: Union[Numeric, str, None] = None,
    # grid 组件的高度。默认自适应。
    height: Union[Numeric, str, None] = None,
    # grid 区域是否包含坐标轴的刻度标签。
    is_contain_label: bool = False,
    # 网格背景色，默认透明。
    background_color: str = "transparent",
    # 网格的边框颜色。支持的颜色格式同 backgroundColor。
    border_color: str = "#ccc",
    # 网格的边框线宽。
    border_width: Numeric = 1,
    # 本坐标系特定的 tooltip 设定。
    tooltip_opts: Union[TooltipOpts, dict, None] = None,
)
```

**例子:**

```python
from pyecharts import options as opts
from pyecharts.charts import Bar, Grid, Line
from pyecharts.faker import Faker

bar = (
    Bar()
    .add_xaxis(Faker.choose())
    .add_yaxis("商家A", Faker.values())
    .add_yaxis("商家B", Faker.values())
    .set_global_opts(title_opts=opts.TitleOpts(title="Grid-Bar"))
)
line = (
    Line()
    .add_xaxis(Faker.choose())
    .add_yaxis("商家A", Faker.values())
    .add_yaxis("商家B", Faker.values())
    .set_global_opts(
        title_opts=opts.TitleOpts(title="Grid-Line", pos_top="48%"),
        legend_opts=opts.LegendOpts(pos_top="48%"),
    )
)

grid = (
    Grid()
    .add(bar, grid_opts=opts.GridOpts(pos_bottom="60%"))
    .add(line, grid_opts=opts.GridOpts(pos_top="60%"))
    
)
grid.render_notebook()
```

##### 3.6.2 顺序多图Page

```python
class Page(
    # HTML 标题
    page_title: str = "Awesome-pyecharts",
    # 远程 HOST，默认为 "https://assets.pyecharts.org/assets/"
    js_host: str = "",
    # 每个图例之间的间隔
    interval: int = 1,
    # 布局配置项，参考 `PageLayoutOpts`
    layout: Union[PageLayoutOpts, dict] = PageLayoutOpts()
)

def add(*charts)    # charts: 任意图表实例

# 需要自己写css 
class PageLayoutOpts(
    # 配置均为原生 CSS 样式
    justify_content: Optional[str] = None,
    margin: Optional[str] = None,
    display: Optional[str] = None,
    flex_wrap: Optional[str] = None,
)

#用于 DraggablePageLayout 布局重新渲染图表
def save_resize_html(
    # Page 第一次渲染后的 html 文件
    source: str = "render.html",
    *,
    # 布局配置文件
    cfg_file: types.Optional[str] = None,

    # 布局配置 dict
    cfg_dict: types.Optional[list] = None,

    # 重新生成的 .html 存放路径
    dest: str = "resize_render.html",
) -> str

#Page 内置了以下布局
SimplePageLayout
DraggablePageLayout
```

**例子:**

```python
from pyecharts import options as opts
from pyecharts.charts import Bar, Grid, Line, Liquid, Page, Pie
from pyecharts.commons.utils import JsCode
from pyecharts.components import Table
from pyecharts.faker import Faker


def bar_datazoom_slider() -> Bar:
    c = (
        Bar()
        .add_xaxis(Faker.days_attrs)
        .add_yaxis("商家A", Faker.days_values)
        .set_global_opts(
            title_opts=opts.TitleOpts(title="Bar-DataZoom（slider-水平）"),
            datazoom_opts=[opts.DataZoomOpts()],
        )
    )
    return c


def line_markpoint() -> Line:
    c = (
        Line()
        .add_xaxis(Faker.choose())
        .add_yaxis(
            "商家A",
            Faker.values(),
            markpoint_opts=opts.MarkPointOpts(data=[opts.MarkPointItem(type_="min")]),
        )
        .add_yaxis(
            "商家B",
            Faker.values(),
            markpoint_opts=opts.MarkPointOpts(data=[opts.MarkPointItem(type_="max")]),
        )
        .set_global_opts(title_opts=opts.TitleOpts(title="Line-MarkPoint"))
    )
    return c


def pie_rosetype() -> Pie:
    v = Faker.choose()
    c = (
        Pie()
        .add(
            "",
            [list(z) for z in zip(v, Faker.values())],
            radius=["30%", "75%"],
            center=["25%", "50%"],
            rosetype="radius",
            label_opts=opts.LabelOpts(is_show=False),
        )
        .add(
            "",
            [list(z) for z in zip(v, Faker.values())],
            radius=["30%", "75%"],
            center=["75%", "50%"],
            rosetype="area",
        )
        .set_global_opts(title_opts=opts.TitleOpts(title="Pie-玫瑰图示例"))
    )
    return c


def grid_mutil_yaxis() -> Grid:
    x_data = ["{}月".format(i) for i in range(1, 13)]
    bar = (
        Bar()
        .add_xaxis(x_data)
        .add_yaxis(
            "蒸发量",
            [2.0, 4.9, 7.0, 23.2, 25.6, 76.7, 135.6, 162.2, 32.6, 20.0, 6.4, 3.3],
            yaxis_index=0,
            color="#d14a61",
        )
        .add_yaxis(
            "降水量",
            [2.6, 5.9, 9.0, 26.4, 28.7, 70.7, 175.6, 182.2, 48.7, 18.8, 6.0, 2.3],
            yaxis_index=1,
            color="#5793f3",
        )
        .extend_axis(
            yaxis=opts.AxisOpts(
                name="蒸发量",
                type_="value",
                min_=0,
                max_=250,
                position="right",
                axisline_opts=opts.AxisLineOpts(
                    linestyle_opts=opts.LineStyleOpts(color="#d14a61")
                ),
                axislabel_opts=opts.LabelOpts(formatter="{value} ml"),
            )
        )
        .extend_axis(
            yaxis=opts.AxisOpts(
                type_="value",
                name="温度",
                min_=0,
                max_=25,
                position="left",
                axisline_opts=opts.AxisLineOpts(
                    linestyle_opts=opts.LineStyleOpts(color="#675bba")
                ),
                axislabel_opts=opts.LabelOpts(formatter="{value} °C"),
                splitline_opts=opts.SplitLineOpts(
                    is_show=True, linestyle_opts=opts.LineStyleOpts(opacity=1)
                ),
            )
        )
        .set_global_opts(
            yaxis_opts=opts.AxisOpts(
                name="降水量",
                min_=0,
                max_=250,
                position="right",
                offset=80,
                axisline_opts=opts.AxisLineOpts(
                    linestyle_opts=opts.LineStyleOpts(color="#5793f3")
                ),
                axislabel_opts=opts.LabelOpts(formatter="{value} ml"),
            ),
            title_opts=opts.TitleOpts(title="Grid-多 Y 轴示例"),
            tooltip_opts=opts.TooltipOpts(trigger="axis", axis_pointer_type="cross"),
        )
    )

    line = (
        Line()
        .add_xaxis(x_data)
        .add_yaxis(
            "平均温度",
            [2.0, 2.2, 3.3, 4.5, 6.3, 10.2, 20.3, 23.4, 23.0, 16.5, 12.0, 6.2],
            yaxis_index=2,
            color="#675bba",
            label_opts=opts.LabelOpts(is_show=False),
        )
    )

    bar.overlap(line)
    return Grid().add(
        bar, opts.GridOpts(pos_left="5%", pos_right="20%"), is_control_axis_index=True
    )


def liquid_data_precision() -> Liquid:
    c = (
        Liquid()
        .add(
            "lq",
            [0.3254],
            label_opts=opts.LabelOpts(
                font_size=50,
                formatter=JsCode(
                    """function (param) {
                        return (Math.floor(param.value * 10000) / 100) + '%';
                    }"""
                ),
                position="inside",
            ),
        )
        .set_global_opts(title_opts=opts.TitleOpts(title="Liquid-数据精度"))
    )
    return c


def table_base() -> Table:
    table = Table()

    headers = ["City name", "Area", "Population", "Annual Rainfall"]
    rows = [
        ["Brisbane", 5905, 1857594, 1146.4],
        ["Adelaide", 1295, 1158259, 600.5],
        ["Darwin", 112, 120900, 1714.7],
        ["Hobart", 1357, 205556, 619.5],
        ["Sydney", 2058, 4336374, 1214.8],
        ["Melbourne", 1566, 3806092, 646.9],
        ["Perth", 5386, 1554769, 869.4],
    ]
    table.add(headers, rows).set_global_opts(
        title_opts=opts.ComponentTitleOpts(title="Table")
    )
    return table


def page_simple_layout() -> Page:
    page = Page(layout=Page.SimplePageLayout)
    page.add(
        bar_datazoom_slider(),
        line_markpoint(),
        pie_rosetype(),
        grid_mutil_yaxis(),
        liquid_data_precision(),
        table_base(),
    )
    return page
   
page_simple_layout().render_notebook()
```

##### 3.6.3 选项卡多图Tab

```python
class Tab(
    # HTML 标题
    page_title: str = "Awesome-pyecharts",
    # 远程 HOST，默认为 "https://assets.pyecharts.org/assets/"
    js_host: str = ""
)
def add(
    # 任意图表类型
    chart,
    # 标签名称
    tab_name
):
```

**例子:**

```python
from pyecharts import options as opts
from pyecharts.charts import Bar, Grid, Line, Pie, Tab
from pyecharts.faker import Faker


def bar_datazoom_slider() -> Bar:
    c = (
        Bar()
        .add_xaxis(Faker.days_attrs)
        .add_yaxis("商家A", Faker.days_values)
        .set_global_opts(
            title_opts=opts.TitleOpts(title="Bar-DataZoom（slider-水平）"),
            datazoom_opts=[opts.DataZoomOpts()],
        )
    )
    return c


def line_markpoint() -> Line:
    c = (
        Line()
        .add_xaxis(Faker.choose())
        .add_yaxis(
            "商家A",
            Faker.values(),
            markpoint_opts=opts.MarkPointOpts(data=[opts.MarkPointItem(type_="min")]),
        )
        .add_yaxis(
            "商家B",
            Faker.values(),
            markpoint_opts=opts.MarkPointOpts(data=[opts.MarkPointItem(type_="max")]),
        )
        .set_global_opts(title_opts=opts.TitleOpts(title="Line-MarkPoint"))
    )
    return c


def pie_rosetype() -> Pie:
    v = Faker.choose()
    c = (
        Pie()
        .add(
            "",
            [list(z) for z in zip(v, Faker.values())],
            radius=["30%", "75%"],
            center=["25%", "50%"],
            rosetype="radius",
            label_opts=opts.LabelOpts(is_show=False),
        )
        .add(
            "",
            [list(z) for z in zip(v, Faker.values())],
            radius=["30%", "75%"],
            center=["75%", "50%"],
            rosetype="area",
        )
        .set_global_opts(title_opts=opts.TitleOpts(title="Pie-玫瑰图示例"))
    )
    return c


def grid_mutil_yaxis() -> Grid:
    x_data = ["{}月".format(i) for i in range(1, 13)]
    bar = (
        Bar()
        .add_xaxis(x_data)
        .add_yaxis(
            "蒸发量",
            [2.0, 4.9, 7.0, 23.2, 25.6, 76.7, 135.6, 162.2, 32.6, 20.0, 6.4, 3.3],
            yaxis_index=0,
            color="#d14a61",
        )
        .add_yaxis(
            "降水量",
            [2.6, 5.9, 9.0, 26.4, 28.7, 70.7, 175.6, 182.2, 48.7, 18.8, 6.0, 2.3],
            yaxis_index=1,
            color="#5793f3",
        )
        .extend_axis(
            yaxis=opts.AxisOpts(
                name="蒸发量",
                type_="value",
                min_=0,
                max_=250,
                position="right",
                axisline_opts=opts.AxisLineOpts(
                    linestyle_opts=opts.LineStyleOpts(color="#d14a61")
                ),
                axislabel_opts=opts.LabelOpts(formatter="{value} ml"),
            )
        )
        .extend_axis(
            yaxis=opts.AxisOpts(
                type_="value",
                name="温度",
                min_=0,
                max_=25,
                position="left",
                axisline_opts=opts.AxisLineOpts(
                    linestyle_opts=opts.LineStyleOpts(color="#675bba")
                ),
                axislabel_opts=opts.LabelOpts(formatter="{value} °C"),
                splitline_opts=opts.SplitLineOpts(
                    is_show=True, linestyle_opts=opts.LineStyleOpts(opacity=1)
                ),
            )
        )
        .set_global_opts(
            yaxis_opts=opts.AxisOpts(
                name="降水量",
                min_=0,
                max_=250,
                position="right",
                offset=80,
                axisline_opts=opts.AxisLineOpts(
                    linestyle_opts=opts.LineStyleOpts(color="#5793f3")
                ),
                axislabel_opts=opts.LabelOpts(formatter="{value} ml"),
            ),
            title_opts=opts.TitleOpts(title="Grid-多 Y 轴示例"),
            tooltip_opts=opts.TooltipOpts(trigger="axis", axis_pointer_type="cross"),
        )
    )

    line = (
        Line()
        .add_xaxis(x_data)
        .add_yaxis(
            "平均温度",
            [2.0, 2.2, 3.3, 4.5, 6.3, 10.2, 20.3, 23.4, 23.0, 16.5, 12.0, 6.2],
            yaxis_index=2,
            color="#675bba",
            label_opts=opts.LabelOpts(is_show=False),
        )
    )

    bar.overlap(line)
    return Grid().add(
        bar, opts.GridOpts(pos_left="5%", pos_right="20%"), is_control_axis_index=True
    )


tab = Tab()
tab.add(bar_datazoom_slider(), "bar-example")
tab.add(line_markpoint(), "line-example")
tab.add(pie_rosetype(), "pie-example")
tab.add(grid_mutil_yaxis(), "grid-example")
tab.render_notebook()
```

##### 3.6.4 时间线轮播多图Timeline

```python
class Timeline(
    # 初始化配置项，参考 `global_options.InitOpts`
    init_opts: opts.InitOpts = opts.InitOpts()
)
def add(
    # 图表实例
    chart: Base, 

    # 时间点
    time_point: str
)
### 样式调整  个人感觉默认的就够用了 个人没有调节更细的参数 如有需要请参考官网
def add_schema(....)
### 时间轴 checkpoint 样式配置
class TimelineCheckPointerStyle(....)
### 时间轴控制按钮样式
class TimelineControlStyle(....)
```

**例子:**

```python
from pyecharts import options as opts
from pyecharts.charts import Bar, Timeline
from pyecharts.faker import Faker

tl = Timeline()
for i in range(2015, 2020):
    bar = (
        Bar()
        .add_xaxis(Faker.choose())
        .add_yaxis("商家A", Faker.values(), label_opts=opts.LabelOpts(position="right"))
        .add_yaxis("商家B", Faker.values(), label_opts=opts.LabelOpts(position="right"))
        .reversal_axis()
        .set_global_opts(
            title_opts=opts.TitleOpts("Timeline-Bar-Reversal (时间: {} 年)".format(i))
        )
    )
    tl.add(bar, "{}年".format(i))
tl.render_notebook()
```

#### 补充 用table画cohort图

**调整留存图的视觉效果,需要自行调节css样式,本人不擅于前段,只是粗调,有兴趣的可以自行研究**

```python
from pyecharts.globals import CurrentConfig,NotebookType
CurrentConfig.NOTEBOOK_TYPE = NotebookType.JUPYTER_NOTEBOOK   #ZEPPELIN 看html源文件
from pyecharts.components import Table
from pyecharts.options import ComponentTitleOpts

table = Table()
headers = ["week", "actuv","visit1w", "visit2w", "visit3w","visit4w", "visit5w", "visit6w", "visit7w"]
rows = [
    ["2021-11-01", 5905, "56%", "34%","30%","25%","24%","24%","23%"],
    ["2021-11-08", 4800, "56%", "34%","30%","25%","24%","24%",''],
    ["2021-11-15", 3523, "56%", "34%","30%","25%","24%",'',''],
    ["2021-11-22", 3935, "56%", "34%","30%","25%",'','',''],
    ["2021-11-29", 4230, "56%", "34%","30%",'','','',''],
    ["2021-12-06", 4678, "56%", "34%",'','','','',''],
    ["2021-11-13", 5245, "45%", '','','','','',''],
]
## 调节css 样式
# "style":"color:green;text-align:center","border":"1px"
table.add(headers, rows,{"class":"fl-table tr","style":"color:green;text-align:center"})

table.set_global_opts(
    title_opts=ComponentTitleOpts(title="留存", subtitle="留存测试")
)
table.render("table_base.html")
table.render_notebook()
```