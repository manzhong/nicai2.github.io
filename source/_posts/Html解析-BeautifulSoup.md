---
title: Html解析-BeautifulSoup
tags: Python
categories: Python
abbrlink: 53377
date: 2021-12-26 22:45:36
summary_img:
encrypt:
enc_pwd:
---

### 一 简介

​	BeautifulSoup(美味汤)是 python 的一个库,最主要的功能是从网页抓取数据,(当然还有其他两种:正则表达式和Lxml),官方解释最为致命,如下所示:

​	Beautiful Soup 提供一些简单的、python 式的函数用来处理导航、搜索、修改分析树等功能。它是一个工具箱，通过解析文档为用户提供需要抓取的数据，因为简单，所以不需要多少代码就可以写出一个完整的应用程序。 Beautiful Soup 自动将输入文档转换为 Unicode 编码，输出文档转换为 utf-8 编码。你不需要考虑编码方式，除非文档没有指定一个编码方式，这时，Beautiful Soup 就不能自动识别编码方式了。然后，你仅仅需要说明一下原始编码方式就可以了。 Beautiful Soup 已成为和 lxml、html6lib 一样出色的 python 解释器，为用户灵活地提供不同的解析策略或强劲的速度。

#### 1.1 BeautifulSoup,正则,lxml对比

| html解析器       | 性能          | 易用性      | 提取数据方式             |
| ------------- | ----------- | -------- | ------------------ |
| 正则表达式         | 快           | 上手不易(较难) | 正则表达式              |
| BeautifulSoup | 快(使用lxml解析) | 简单       | Find,css选择器....    |
| lxml          | 快           | 简单       | XPath,CSS选择器,..... |

官网地址:https://beautifulsoup.readthedocs.io/zh_CN/latest/

**本文版本:**

```
v4.4.0
```

### 二 安装

​	Beautiful Soup 3 目前已经停止开发，推荐在现在的项目中使用 Beautiful Soup 4，不过它已经被移植到 BS4 了，也就是说导入时我们需要 import bs4 。所以这里我们用的版本是 Beautiful Soup 4.3.2 (简称 BS4)，另外据说 BS4 对 Python3 的支持不够好，不过我用的是 Python2.7.7，如果有小伙伴用的是 Python3 版本，可以考虑下载 BS3 版本。 可以利用 pip 或者 easy_install 来安装，以下两种方法均可

```python
easy_install beautifulsoup4
pip install beautifulsoup4
# 安装lxml
pip install lxml
# 另一个可供选择的解析器是纯 Python 实现的 html5lib , html5lib 的解析方式与浏览器相同，可以选择下列方法来安装 html5lib
pip install html5lib
```

| 解析器           | 使用方法                                     | 优势                                  | 劣势                                  |
| :------------ | :--------------------------------------- | :---------------------------------- | :---------------------------------- |
| Python标准库     | `BeautifulSoup(markup, "html.parser")`   | 1:Python的内置标准库2:执行速度适中3:文档容错能力强     | Python 2.7.3 or 3.2.2)前 的版本中文档容错能力差 |
| lxml HTML 解析器 | `BeautifulSoup(markup, "lxml")`          | 1:速度快2:文档容错能力强                      | 需要安装C语言库                            |
| lxml XML 解析器  | `BeautifulSoup(markup, ["lxml-xml"])` `BeautifulSoup(markup, "xml")` | 1:速度快2:唯一支持XML的解析器                  | 需要安装C语言库                            |
| html5lib      | `BeautifulSoup(markup, "html5lib")`      | 1:最好的容错性2:以浏览器的方式解析文档3:生成HTML5格式的文档 | 1:速度慢2:不依赖外部扩展                      |

### 三  使用

```python
#导入bs4库
from bs4 import BeautifulSoup
# html 的字符串
html = """
<html>
 <head>
  <title>
   The Dormouse's story
  </title>
 </head>
 <body>
  <p class="title" name="dromouse">
   <b>
    The Dormouse's story
   </b>
  </p>
  <p class="story">
   Once upon a time there were three little sisters; and their names were
   <a class="sister" href="http://example.com/elsie" id="link1">
    <!-- Elsie -->
   </a>
   ,
   <a class="sister" href="http://example.com/lacie" id="link2">
    Lacie
   </a>
   and
   <a class="sister" href="http://example.com/tillie" id="link3">
    Tillie
   </a>
   ;
and they lived at the bottom of a well.
  </p>
  <p class="story">
   ...
  </p>
 </body>
</html>
"""
```

创建beautifulsoup对象:

```python
# 创建beautifulsoup对象: features='lxml'
soup = BeautifulSoup(html,features='html.parser') 
# 另外，我们还可以用本地 HTML 文件来创建对象，例如
soup = BeautifulSoup(open('index.html'),features='html.parser')
```

查看html;

```python
print(soup.prettify())
```

#### 3.1 四大对象

Beautiful Soup 将复杂 HTML 文档转换成一个复杂的树形结构，每个节点都是 Python 对象，所有对象可以归纳为 4 种:

- Tag
- NavigableString
- BeautifulSoup
- Comment

##### 3.1.1 tag

**tag是html中的一个个标签**

```html
如: <title>The Dormouse's story</title>,<a class="sister" href="http://example.com/elsie" id="link1">Elsie</a>
上面的 title a 等等 HTML 标签加上里面包括的内容就是 Tag
```

```python
print(soup.title)
#<title>The Dormouse's story</title>
print(soup.head)
#<head><title>The Dormouse's story</title></head>
# 若有多个,默认获取第一个
print(soup.a)
#<a class="sister" href="http://example.com/elsie" id="link1"><!-- Elsie --></a>
```

我们可以利用 soup 加标签名轻松地获取这些标签的内容，是不是感觉比正则表达式方便多了？不过有一点是，它查找的是在所有内容中的第一个符合要求的标签，如果要查询所有的标签，比如: find_all()。 我们可以验证一下这些对象的类型

```python
print(type(soup.a))
#<class 'bs4.element.Tag'>
```

**tag有两个重要的属性name和attrs**

```python
#soup 对象本身比较特殊，它的 name 即为 [document]，对于其他内部标签，输出的值便为标签本身的名称
print(soup.name)
# [document]
print(soup.head.name)
# head

########################## attrs
print(soup.p.attrs)
# {'class': ['title'], 'name': 'dromouse'}
#在这里，我们把 p 标签的所有属性打印输出了出来，得到的类型是一个字典。 如果我们想要单独获取某个属性，可以这样，例如我们获取它的 class 叫什么
print(soup.p['class'])
#['title']
# 等价于 
print(soup.p.get('class'))
#['title']
```

**修改,添加,删除属性和内容**

```python
# 修改
soup.p['class']="newClass"
print(soup.p)
#<p class="newClass" name="dromouse"><b>The Dormouse's story</b></p>

# 删除
del soup.p['class']
print(soup.p)
#<p name="dromouse"><b>The Dormouse's story</b></p>

# 添加属性
soup.p['id'] = 1
print(soup.p.attrs)
# <blockquote class="verybold" id="1">Extremely bold</blockquote>
```

##### 3.1.2 **NavigableString**

既然我们已经得到了标签的内容，那么问题来了，我们要想获取标签内部的文字怎么办呢？很简单，用 .string 即可，例如

```python
print(soup.p.b.string)
#The Dormouse's story
```

这样我们就轻松获取到了标签里面的内容，想想如果用正则表达式要多麻烦。它的类型是一个 NavigableString，翻译过来叫 可以遍历的字符串，不过我们最好还是称它英文名字吧。 来检查一下它的类型

```python
print(type(soup.p.b.string))
#<class 'bs4.element.NavigableString'>
```

**内容不可编辑但可以替换**

```python
soup.p.b.string.replace_with("No longer bold")
print(soup.p.b.string)
```

##### 3.1.3 **BeautifulSoup**

BeautifulSoup 对象表示的是一个文档的全部内容。大部分时候，可以把它当作 Tag 对象，是一个特殊的 Tag，我们可以分别获取它的类型，名称，以及属性来感受一下

```python
print(type(soup.name))
#<type 'unicode'>
print(soup.name)
# [document]
print(soup.attrs)
#{} 空字典
```

##### 3.1.4 **Comment**

Comment 对象是一个特殊类型的 NavigableString 对象，其实输出的内容仍然不包括注释符号，但是如果不好好处理它，可能会对我们的文本处理造成意想不到的麻烦。 我们找一个带注释的标签

```python
markup = "<b><!--Hey, buddy. Want to buy a used parser?--></b>"
soup = BeautifulSoup(markup)
comment = soup.b.string
print(comment)
print(type(comment))
```

运行结果如下

```python
Hey, buddy. Want to buy a used parser?
<class 'bs4.element.Comment'>
```

a 标签里的内容实际上是注释，但是如果我们利用 .string 来输出它的内容，我们发现它已经把注释符号去掉了，所以这可能会给我们带来不必要的麻烦。 另外我们打印输出下它的类型，发现它是一个 Comment 类型，所以，我们在使用前最好做一下判断，判断代码如下

```python
if type(soup.a.string)==bs4.element.Comment:
    print(soup.a.string)
```

上面的代码中，我们首先判断了它的类型，是否为 Comment 类型，然后再进行其他操作，如打印输出。

#### 3.2 遍历文档树

**3.2.1 直接子节点**

​		一个Tag可能包含多个字符串或其它的Tag,这些都是这个Tag的子节点.Beautiful Soup提供了许多操作和遍历子节点的属性.注意: Beautiful Soup中字符串节点不支持这些属性,因为字符串没有子节点.

> **要点：.contents .children** **属性**

**.contents** tag 的 .content 属性可以将 tag 的子节点以列表的方式输出

```python
print(soup.head.contents)
#[<title>The Dormouse's story</title>]
```

输出方式为列表，我们可以用列表索引来获取它的某一个元素

```python
print(soup.head.contents[0])
#<title>The Dormouse's story</title>
```

**.children** 它返回的不是一个 list，不过我们可以通过遍历获取所有子节点。 我们打印输出 .children 看一下，可以发现它是一个 list 生成器对象

```python
print(soup.head.children)
#<listiterator object at 0x7f71457f5710>
```

我们怎样获得里面的内容呢？很简单，遍历一下就好了，代码及结果如下

```python
for child in  soup.body.children:
    print child
<p class="title" name="dromouse"><b>The Dormouse's story</b></p>

<p class="story">Once upon a time there were three little sisters; and their names were
<a class="sister" href="http://example.com/elsie" id="link1"><!-- Elsie --></a>,
<a class="sister" href="http://example.com/lacie" id="link2">Lacie</a> and
<a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>;
and they lived at the bottom of a well.</p>


<p class="story">...</p>
```

**3.2.2所有子孙节点**

> **知识点：.descendants** **属性**

**.descendants** .contents 和 .children 属性仅包含 tag 的直接子节点，.descendants 属性可以对所有 tag 的子孙节点进行递归循环，和 children 类似，我们也需要遍历获取其中的内容。

```python
for child in soup.descendants:
    print(child)
```

运行结果如下，可以发现，所有的节点都被打印出来了，先生最外层的 HTML 标签，其次从 head 标签一个个剥离，以此类推。

```html
<html><head><title>The Dormouse's story</title></head>
<body>
<p class="title" name="dromouse"><b>The Dormouse's story</b></p>
<p class="story">Once upon a time there were three little sisters; and their names were
<a class="sister" href="http://example.com/elsie" id="link1"><!-- Elsie --></a>,
<a class="sister" href="http://example.com/lacie" id="link2">Lacie</a> and
<a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>;
and they lived at the bottom of a well.</p>
<p class="story">...</p>
</body></html>
<head><title>The Dormouse's story</title></head>
<title>The Dormouse's story</title>
The Dormouse's story


<body>
<p class="title" name="dromouse"><b>The Dormouse's story</b></p>
<p class="story">Once upon a time there were three little sisters; and their names were
<a class="sister" href="http://example.com/elsie" id="link1"><!-- Elsie --></a>,
<a class="sister" href="http://example.com/lacie" id="link2">Lacie</a> and
<a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>;
and they lived at the bottom of a well.</p>
<p class="story">...</p>
</body>


<p class="title" name="dromouse"><b>The Dormouse's story</b></p>
<b>The Dormouse's story</b>
The Dormouse's story


<p class="story">Once upon a time there were three little sisters; and their names were
<a class="sister" href="http://example.com/elsie" id="link1"><!-- Elsie --></a>,
<a class="sister" href="http://example.com/lacie" id="link2">Lacie</a> and
<a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>;
and they lived at the bottom of a well.</p>
Once upon a time there were three little sisters; and their names were

<a class="sister" href="http://example.com/elsie" id="link1"><!-- Elsie --></a>
 Elsie 
,

<a class="sister" href="http://example.com/lacie" id="link2">Lacie</a>
Lacie
 and

<a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>
Tillie
;
and they lived at the bottom of a well.


<p class="story">...</p>
...
```

**3.2.3 节点内容**

**知识点：.string 属性**

如果 tag 只有一个 NavigableString 类型子节点，那么这个 tag 可以使用 .string 得到子节点。如果一个 tag 仅有一个子节点，那么这个 tag 也可以使用 .string 方法，输出结果与当前唯一子节点的 .string 结果相同。 通俗点说就是：如果一个标签里面没有标签了，那么 .string 就会返回标签里面的内容。如果标签里面只有唯一的一个标签了，那么 .string 也会返回最里面的内容。例如

```python
print(soup.head.string)
#The Dormouse's story
print(soup.title.string)
#The Dormouse's story
```

如果 tag 包含了多个子节点，tag 就无法确定，string 方法应该调用哪个子节点的内容，.string 的输出结果是 None

```python
print(soup.html.string)
# None
```

**3.2.4 多个内容**

> **知识点： .strings .stripped_strings 属性**

**.strings** 获取多个内容，不过需要遍历获取，比如下面的例子

```python
for string in soup.strings:
    print(repr(string))
    # u"The Dormouse's story"
    # u'\n\n'
    # u"The Dormouse's story"
    # u'\n\n'
    # u'Once upon a time there were three little sisters; and their names were\n'
    # u'Elsie'
    # u',\n'
    # u'Lacie'
    # u' and\n'
    # u'Tillie'
    # u';\nand they lived at the bottom of a well.'
    # u'\n\n'
    # u'...'
    # u'\n'
```

**.stripped_strings** 输出的字符串中可能包含了很多空格或空行，使用 .stripped_strings 可以去除多余空白内容

```python
for string in soup.stripped_strings:
    print(repr(string))
    # u"The Dormouse's story"
    # u"The Dormouse's story"
    # u'Once upon a time there were three little sisters; and their names were'
    # u'Elsie'
    # u','
    # u'Lacie'
    # u'and'
    # u'Tillie'
    # u';\nand they lived at the bottom of a well.'
    # u'...'
```

**3.2.5 父节点**

**知识点： .parent 属性**

```python
p = soup.p
print(p.parent.name)
#body
content = soup.head.title.string
print(content.parent.name)
#title
```

**3.2.6 全部父节点**

> **知识点：.parents 属性**

通过元素的 .parents 属性可以递归得到元素的所有父辈节点，例如

```python
content = soup.head.title.string
for parent in  content.parents:
    print(parent.name)
#title
#head
#html
#[document]
```

**3.2.7 兄弟节点**

> **知识点：.next_sibling .previous_sibling 属性**

兄弟节点可以理解为和本节点处在统一级的节点，.next_sibling 属性获取了该节点的下一个兄弟节点，.previous_sibling 则与之相反，如果节点不存在，则返回 None 注意：实际文档中的 tag 的 .next_sibling 和 .previous_sibling 属性通常是字符串或空白，因为空白或者换行也可以被视作一个节点，所以得到的结果可能是空白或者换行

```python
print(soup.p.next_sibling)
#       实际该处为空白
print(soup.p.prev_sibling)
#None   没有前一个兄弟节点，返回 None
print(soup.p.next_sibling.next_sibling)
#<p class="story">Once upon a time there were three little sisters; and their names were
#<a class="sister" href="http://example.com/elsie" id="link1"><!-- Elsie --></a>,
#<a class="sister" href="http://example.com/lacie" id="link2">Lacie</a> and
#<a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>;
#and they lived at the bottom of a well.</p>
#下一个节点的下一个兄弟节点是我们可以看到的节点
```

**3.2.8 全部兄弟节点**

> **知识点：.next_siblings .previous_siblings 属性**

通过 .next_siblings 和 .previous_siblings 属性可以对当前节点的兄弟节点迭代输出

```python
for sibling in soup.a.next_siblings:
    print(repr(sibling))
    # u',\n'
    # <a class="sister" href="http://example.com/lacie" id="link2">Lacie</a>
    # u' and\n'
    # <a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>
    # u'; and they lived at the bottom of a well.'
    # None
```

**3.2.9 前后节点**

> **知识点：.next_element .previous_element 属性**

与 .next_sibling .previous_sibling 不同，它并不是针对于兄弟节点，而是在所有节点，不分层次 比如 head 节点为

```html
<head><title>The Dormouse's story</title></head>
```

那么它的下一个节点便是 title，它是不分层次关系的

```python
print(soup.head.next_element)
#<title>The Dormouse's story</title>
```

**3.2.10 所有前后节点**

> **知识点：.next_elements .previous_elements 属性**

通过 .next_elements 和 .previous_elements 的迭代器就可以向前或向后访问文档的解析内容，就好像文档正在被解析一样

```python
for element in last_a_tag.next_elements:
    print(repr(element))
# u'Tillie'
# u';\nand they lived at the bottom of a well.'
# u'\n\n'
# <p class="story">...</p>
# u'...'
# u'\n'
# None
```

以上是遍历文档树的基本用法。

#### 3.3. 搜索文档树

常用的是find()` 和 `find_all()

##### 3.3.1 find_all()

​	find_all () 方法搜索当前 tag 的所有 tag 子节点，并判断是否符合过滤器的条件 **1）name 参数** name 参数可以查找所有名字为 name 的 tag, 字符串对象会被自动忽略掉 **A. 传字符串** 最简单的过滤器是字符串。在搜索方法中传入一个字符串参数，Beautiful Soup 会查找与字符串完整匹配的内容.

**查找文档中所有的标签**

```python
print(soup.find_all('b'))
# [<b>The Dormouse's story</b>]
print(soup.find_all('a'))
#[<a class="sister" href="http://example.com/elsie" id="link1"><!-- Elsie --></a>, <a class="sister" href="http://example.com/lacie" id="link2">Lacie</a>, <a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>]
```

传正则表达式 如果传入正则表达式作为参数，Beautiful Soup 会通过正则表达式的 match () 来匹配内容。下面例子中找出所有以 b 开头的标签，这表示和标签都应该被找到

```python
import re
for tag in soup.find_all(re.compile("^b")):
    print(tag.name)
# body
# b
```

**传列表** **如果传入列表参数，Beautiful Soup 会将与列表中任一元素匹配的内容返回。下面代码找到文档中所有**标签和**标签**

```python
print(soup.find_all(["a", "b"]) ) 
# [<b>The Dormouse's story</b>,
#  <a class="sister" href="http://example.com/elsie" id="link1">Elsie</a>,
#  <a class="sister" href="http://example.com/lacie" id="link2">Lacie</a>,
#  <a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>]
```

***\*传 True\** True 可以匹配任何值，下面代码查找到所有的 tag, 但是不会返回字符串节点**

```python
for tag in soup.find_all(True):
    print(tag.name)
# html
# head
# title
# body
# p
# b
# p
# a
# a
```

**传方法** 如果没有合适过滤器，那么还可以定义一个方法，方法只接受一个元素参数 [[4\]](http://www.crummy.com/software/BeautifulSoup/bs4/doc/index.zh.html#id85) **, 如果这个方法返回 True 表示当前元素匹配并且被找到，如果不是则反回 False 下面方法校验了当前元素，如果包含 class 属性却不包含 id 属性，那么将返回 True:**

```python
def has_class_but_no_id(tag):
    return tag.has_attr('class') and not tag.has_attr('id')

print(soup.find_all(has_class_but_no_id))
# [<p class="title"><b>The Dormouse's story</b></p>,
#  <p class="story">Once upon a time there were...</p>,
#  <p class="story">...</p>]
```

##### 3.3.2 keyword参数

```python
#如果一个指定名字的参数不是搜索内置的参数名，搜索时会把该参数当作指定名字 tag 的属性来搜索，如果包含一个名字为 id 的参数，Beautiful Soup 会搜索每个 tag 的”id” 属性
soup.find_all(id='link2')
#[<a class="sister" href="http://example.com/lacie" id="link2">Lacie</a>]

#如果传入 href 参数，Beautiful Soup 会搜索每个 tag 的”href” 属性
soup.find_all(href=re.compile("elsie"))
# [<a class="sister" href="http://example.com/elsie" id="link1">Elsie</a>]

#使用多个指定名字的参数可以同时过滤 tag 的多个属性
soup.find_all(href=re.compile("elsie"), id='link1')
# [<a class="sister" href="http://example.com/elsie" id="link1">three</a>]

#在这里我们想用 class 过滤，不过 class 是 python 的关键词，这怎么办？加个下划线就可以
soup.find_all("a", class_="sister")
# [<a class="sister" href="http://example.com/elsie" id="link1">Elsie</a>,
#  <a class="sister" href="http://example.com/lacie" id="link2">Lacie</a>,
#  <a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>]

#有些 tag 属性在搜索不能使用，比如 HTML5 中的 data-* 属性
data_soup = BeautifulSoup('<div data-foo="value">foo!</div>')
data_soup.find_all(data-foo="value")
# SyntaxError: keyword can't be an expression
#但是可以通过 find_all () 方法的 attrs 参数定义一个字典参数来搜索包含特殊属性的 tag
data_soup.find_all(attrs={"data-foo": "value"})
# [<div data-foo="value">foo!</div>]


# text 参数 通过 text 参数可以搜搜文档中的字符串内容。与 name 参数的可选值一样，text 参数接受 字符串，正则表达式，列表，True
soup.find_all(text="Elsie")
# [u'Elsie']
soup.find_all(text=["Tillie", "Elsie", "Lacie"])
# [u'Elsie', u'Lacie', u'Tillie']
soup.find_all(text=re.compile("Dormouse"))
#[u"The Dormouse's story", u"The Dormouse's story"]


#limit 参数 find_all () 方法返回全部的搜索结构，如果文档树很大那么搜索会很慢。如果我们不需要全部结果，可以使用 limit 参数限制返回结果的数量。效果与 SQL 中的 limit 关键字类似，当搜索到的结果数量达到 limit 的限制时，就停止搜索返回结果。文档树中有 3 个 tag 符合搜索条件，但结果只返回了 2 个，因为我们限制了返回数量
soup.find_all("a", limit=2)
# [<a class="sister" href="http://example.com/elsie" id="link1">Elsie</a>,
#  <a class="sister" href="http://example.com/lacie" id="link2">Lacie</a>]


#recursive 参数 调用 tag 的 find_all () 方法时，Beautiful Soup 会检索当前 tag 的所有子孙节点，如果只想搜索 tag 的直接子节点，可以使用参数 recursive=False
soup.html.find_all("title")
# [<title>The Dormouse's story</title>]
soup.html.find_all("title", recursive=False)
# []
```

此外还有

```python
find( name , attrs , recursive , text , **kwargs )
#它与 find_all () 方法唯一的区别是 find_all () 方法的返回结果是值包含一个元素的列表，而 find () 方法直接返回结果
find_parents() find_parent()
#ind_all () 和 find () 只搜索当前节点的所有子节点，孙子节点等. find_parents () 和 find_parent () 用来搜索当前节点的父辈节点，搜索方法与普通 tag 的搜索方法相同，搜索文档搜索文档包含的内容
find_next_siblings() find_next_sibling()
#这 2 个方法通过 .next_siblings 属性对当 tag 的所有后面解析的兄弟 tag 节点进行迭代，find_next_siblings () 方法返回所有符合条件的后面的兄弟节点，find_next_sibling () 只返回符合条件的后面的第一个 tag 节点
find_previous_siblings() find_previous_sibling()
#这 2 个方法通过 .previous_siblings 属性对当前 tag 的前面解析的兄弟 tag 节点进行迭代，find_previous_siblings () 方法返回所有符合条件的前面的兄弟节点，find_previous_sibling () 方法返回第一个符合条件的前面的兄弟节点
find_all_next() find_next()
#这 2 个方法通过 .next_elements 属性对当前 tag 的之后的 tag 和字符串进行迭代，find_all_next () 方法返回所有符合条件的节点，find_next () 方法返回第一个符合条件的节点
find_all_previous () 和 find_previous ()
#这 2 个方法通过 .previous_elements 属性对当前节点前面的 tag 和字符串进行迭代，find_all_previous () 方法返回所有符合条件的节点，find_previous () 方法返回第一个符合条件的节点
```

等,用法与find_all类似,不做具体说明

#### 3.4 CSS选择器

​		按照CSS类名搜索tag的功能非常实用,但标识CSS类名的关键字 `class` 在Python中是保留字,使用 `class` 做参数会导致语法错误.从Beautiful Soup的4.1.1版本开始,可以通过 `class_` 参数搜索有指定CSS类名的tag:

```python
soup.find_all("a", class_="sister")
# [<a class="sister" href="http://example.com/elsie" id="link1">Elsie</a>,
#  <a class="sister" href="http://example.com/lacie" id="link2">Lacie</a>,
#  <a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>]
```

**我们在写 CSS 时，标签名不加任何修饰，类名前加点，id 名前加 #，在这里我们也可以利用类似的方法来筛选元素，用到的方法是** **soup.select()，****返回类型是** **list**

```python
# 标签名
print(soup.select('title') )
# 类名 class
print(soup.select('.sister'))
## [<a class="sister" href="http://example.com/elsie" id="link1"><!-- Elsie --></a>, <a class="sister" href="http://example.com/lacie" id="link2">Lacie</a>, <a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>]
# 通过 id 名查找
print soup.select('#link1')


#组合查找
#组合查找即和写 class 文件时，标签名与类名、id 名进行的组合原理是一样的，例如查找 p 标签中，id 等于 link1 的内容，二者需要用空格分开
print(soup.select('p #link1'))
# 子标签查找
print soup.select("head > title")
```

**属性查找:**

```python
#查找时还可以加入属性元素，属性需要用中括号括起来，注意属性和标签属于同一节点，所以中间不能加空格，否则会无法匹配到。
print(soup.select('a[class="sister"]'))
#[<a class="sister" href="http://example.com/elsie" id="link1"><!-- Elsie --></a>, <a class="sister" href="http://example.com/lacie" id="link2">Lacie</a>, <a class="sister" href="http://example.com/tillie" id="link3">Tillie</a>]
print(soup.select('a[href="http://example.com/elsie"]'))
#[<a class="sister" href="http://example.com/elsie" id="link1"><!-- Elsie --></a>]


#同样，属性仍然可以与上述查找方式组合，不在同一节点的空格隔开，同一节点的不加空格
print(soup.select('p a[href="http://example.com/elsie"]'))
#[<a class="sister" href="http://example.com/elsie" id="link1"><!-- Elsie --></a>]

#以上的 select 方法返回的结果都是列表形式，可以遍历形式输出，然后用 get_text () 方法来获取它的内容。
soup = BeautifulSoup(html, 'lxml')
print(type(soup.select('title')))
print(soup.select('title')[0].get_text())

for title in soup.select('title'):
    print(title.get_text())
```



