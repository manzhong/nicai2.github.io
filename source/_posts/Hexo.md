---
title: Hexo
tags:
  - Hexo
categories:
  - Hexo
encrypt: 
enc_pwd: 
abbrlink: 37733
date: 2017-05-17 21:12:52
summary_img:
---





# 一 Hexo搭建博客

写本篇博客的理由:

- 你可以参考别人的技术方案，集众所长，亲自实践，然后融入自己的思考写出一篇新文章

- 即使并没有做出创新性的贡献，自己重新归纳一遍也有助于梳理流程，深化理解

- 否则久而久之就是所谓的“代码搬运工” ！

  ​

## 1 安装node.js

在 [官方下载网站](https://nodejs.org/en/download/) 下载源代码，选择最后一项 `Source Code`
解压到某一目录, 然后进入此目录,依次执行以下 3 条命令

```
$ ./configure
$ make
$ sudo make install
```

安装完后查看`node.js`版本，检验是否安装成功

```
$ node -v
```

## 2 安装hexo

在命令行中通过 **npm** 来安装 hexo：

```
$ npm install -g hexo-cli
```

## 3本地启动hexo

创建一个博客目录（例如 `/my-blog`），在此目录下，执行初始化命令

```
$ mkdir -p my-blog
$ cd my-blog
$ hexo init
```

执行完毕后，将会生成以下文件结构：

```
.
|-- node_modules       //依赖安装目录
|-- scaffolds          //模板文件夹，新建的文章将会从此目录下的文件中继承格式
|-- source             //资源文件夹，用于放置图片、数据、文章等资源
|   |-- _posts          //文章目录
|-- themes             //主题文件夹
|   |-- landscape      //默认主题
|-- .gitignore         //指定不纳入git版本控制的文件
|-- _config.yml        //站点配置文件
|-- db.json
|-- package.json
`-- package-lock.json
```

在根目录下执行如下命令启动**hexo**内置的web容器

```
$ hexo generate(g)     # 生成静态文件
$ hexo server(s)       # 在本地服务器运行
```

在浏览器输入IP地址 [http://localhost:4000](http://localhost:4000/) 就可以看到我们熟悉的** Hello Word **了。

## 4 常用命令简化和组合

```
$ hexo g    # 等同于hexo generate
$ hexo s    # 等同于hexo server
$ hexo p    # 等同于hexo port 
$ hexo d    # 等同于hexo deploy
```

当本地不想使用默认的4000端口时（比如在服务器上，默认使用80端口），可以使用 port 命令更改启动端口
另外，**hexo**支持命令合并，比方说 生成静态文件 → 本地启动80端口，我们可以执行

```
$ hexo s -g -p 80
```

## 5 安装next主题

hexo 安装主题的方式非常简单, 只需几个简单的命令即可。
将NexT主题文件拷贝至**themes**目录下，然后修改 站点配置文件 _config.yml 中的 `theme`字段为`next`即可。

cd 到博客的根目录下执行以下命令下载主题文件：

```
$ cd my-blog
$ git clone https://github.com/theme-next/hexo-theme-next.git themes/next

$ vim _config.yml
theme: next
```

清除 **hexo**缓存，重启服务

```
$ hexo clean
$ hexo s -g
```

大部分的设定都能在 [NexT官方文档](http://theme-next.iissnan.com/getting-started.html) 里找到, 如主题设定、侧栏、头像、友情链接、打赏等等，在此就不多讲了，照着文档走就行了。





## 二 目录结构

### 1 hexo目录结构

```
.
├── .deploy
├── public
├── scaffolds
├── scripts
├── source
|   ├── _drafts
|   └── _posts
├── themes
├── _config.yml
└── package.json
```

- deploy：执行hexo deploy命令部署到GitHub上的内容目录

- public：执行hexo generate命令，输出的静态网页内容目录

- scaffolds：layout模板文件目录，其中的md文件可以添加编辑

- scripts：扩展脚本目录，这里可以自定义一些javascript脚本

- source：文章源码目录，该目录下的markdown和html文件均会被hexo处理。该页面对应repo的根目录，404文件、favicon.ico文件，CNAME文件等都应该放这里，该目录下可新建页面目录。

  ​

  - drafts：草稿文章
  - posts：发布文章

- themes：主题文件目录

- _config.yml：全局配置文件，大多数的设置都在这里

- package.json：应用程序数据，指明hexo的版本等信息，类似于一般软件中的关于按钮

### 2 next主题结构

```text
├── .github            #git信息
├── languages          #多语言
|   ├── default.yml    #默认语言
|   └── zh-Hans.yml      #简体中文
|   └── zh-tw.yml      #繁体中文
├── layout             #布局，根目录下的*.ejs文件是对主页，分页，存档等的控制
|   ├── _custom        #可以自己修改的模板，覆盖原有模板
|   |   ├── _header.swig    #头部样式
|   |   ├── _sidebar.swig   #侧边栏样式
|   ├── _macro        #可以自己修改的模板，覆盖原有模板
|   |   ├── post.swig    #文章模板
|   |   ├── reward.swig    #打赏模板
|   |   ├── sidebar.swig   #侧边栏模板
|   ├── _partial       #局部的布局
|   |   ├── head       #头部模板
|   |   ├── search     #搜索模板
|   |   ├── share      #分享模板
|   ├── _script        #局部的布局
|   ├── _third-party   #第三方模板
|   ├── _layout.swig   #主页面模板
|   ├── index.swig     #主页面模板
|   ├── page           #页面模板
|   └── tag.swig       #tag模板
├── scripts            #script源码
|   ├── tags           #tags的script源码
|   ├── marge.js       #页面模板
├── source             #源码
|   ├── css            #css源码
|   |   ├── _common    #*.styl基础css
|   |   ├── _custom    #*.styl局部css
|   |   └── _mixins    #mixins的css
|   ├── fonts          #字体
|   ├── images         #图片
|   ├── uploads        #添加的文件
|   └── js             #javascript源代码
├── _config.yml        #主题配置文件
└── README.md          #用GitHub的都知道
```



# 三 next主题优化

## 1 文章加密

首先输入命令：

首先输入命令：
`npm install hexo-encrypt --save`

首先输入命令：
`npm install hexo-encrypt --save`
等待安装完成后，修改**博客**配置文件`_config.yml`。

首先输入命令：
`npm install hexo-encrypt --save`
等待安装完成后，修改**博客**配置文件`_config.yml`。
在末尾添加：

```
encrypt: 
  password: 123456
```

这里的`123456`为**默认密码**，即若文章加密并且未声明**独立密码**即可通过**默认密码**解锁文章。

然后在每一篇文章的开头加入：

```
encrypt: true
enc_pwd: 123456
```

直接修改模板:

scaffolds/post.md

```
title: {{ title }}
date: {{ date }}
tags:
categories:
summary_img:
encrypt:
enc_pwd:
```

## 2  



