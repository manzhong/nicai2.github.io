---
title: next7.8版配置优化
tags: Blog
encrypt: 
enc_pwd: 
abbrlink: 30833
date: 2020-10-08 17:30:36
categories: Blog
summary_img:
---

### 前言:

   之前用的hexo+next的版本太老了,而新版本的有很多新的功能,就萌生了版本升级计划,本次是记录了本次升级过程

### 推动我更新的最大动力:

​	当时选择 NexT 主题就是看中它的维护者多、用户量大，基本一个月左右的时间就会有一个版本更新，直接跟着官方就可以用到最新的特性。

但由于我之前对 NexT 主题做了不少自定义修改，这也使得更新主题变得比较麻烦，没有办法通过 `git pull` 平滑更新，这也违背了我选择 NexT 主题的初衷。同时，现在可以通过数据文件（Data File）将配置与主题分离，同时也可以把自定义布局、样式放到数据文件中，不用再修改主题源码，便于后续主题更新。

因此，为了以后可以方便的更新，我准备花点时间把 NexT 升级一下。

~~~txt
更新后版本: Hexo-5.2.0、NexT-7.8.0
~~~

## 1 数据文件

自从 NexT-7.3.0 开始，官方推荐采用数据文件将配置与主题分离，这样我们可以在不修改主题源码的同时完成选项配置、自定义布局、自定义样式，便于后续 NexT 版本更新。

### 1.1 next.yml

我们可以将所有主题配置放在一个位置（`hexo/source/_data/next.yml`）。这样就无需编辑主题配置文件（`next/_config.yml`）。

具体步骤：

1. 在 `hexo/source/_data` 目录中创建 `next.yml`（如果`_data` 不存在，则创建目录）。
2. 在 `next.yml` 设置 `override` 选项为 true。
3. 将**所有 NexT 主题选项**从主题配置文件复制到 `hexo/source/_data/next.yml` 中。

然后我们只需要根据自己的需求配置 `next.yml` 即可。

### 1.2 languages.yml

我们原来是通过配置主题下的 `languages` 目录中的 `zh-CN.yml` 文件来对菜单等进行中文翻译的，现在我们可以通过在 `hexo/source/_data/` 下新建数据文件 `languages.yml`，配置如下：

```yml
zh-CN: 
  menu:
    home: 首页
    top: 热榜
    archives: 归档
    categories: 分类
    tags: 标签
    about: 关于
    links: 友情链接
    search: 搜索
    schedule: 日程表
    sitemap: 站点地图
    commonweal: 公益 404

  reward:
    donate: <i class="fa fa-qrcode fa-2x" style="line-height:35px;"></i>
    wechatpay: 微信支付
    alipay: 支付宝
    bitcoin: 比特币
```

### 1.3 styles.styl

我们只需要把原来的 `hexo/next/source/css/_custom/_custom.styl` 中的全部自定义样式放到 `hexo/source/_data/styles.styl` 即可。

然后在 NexT 的配置文件 `next.yml` 中取消 `styles.styl` 的注释：

```yml
custom_file_path:

-  #style: source/_data/styles.styl
+  style: source/_data/styles.styl
```

### 1.4 variables.styl

#### 圆角设置

在自定义样式文件 `variables.styl` 中添加：

```styl
// 圆角设置
$border-radius-inner     = 20px 20px 20px 20px;
$border-radius           = 20px;
```

然后在 NexT 的配置文件 `next.yml` 中取消 `variables.styl` 的注释：

```yml
custom_file_path:

-  #variables: source/_data/variables.styl
+  variables: source/_data/variables.styl
```

#### 中文字体设置

首先修改主题配置文件 `next.yml`：

```yml
font:
- enable: false
+ enable: true

  # Uri of fonts host. E.g. //fonts.googleapis.com (Default).
- host:
+ host: https://fonts.loli.net

  # Font options:
  # `external: true` will load this font family from `host` above.
  # `family: Times New Roman`. Without any quotes.
  # `size: xx`. Use `px` as unit.

  # Global font settings used for all elements in <body>.
  global:
    external: true
-   family:
+   family: Noto Serif SC
    size:
```

修改配置文件 `variables.styl`，增加如下代码：

```styl
// 中文字体
$font-family-monospace    = consolas, Menlo, monospace, $font-family-base;
$font-family-monospace    = 
get_font_family('codes'), consolas, Menlo, monospace, $font-family-base if get_font_family('codes');
```

### 1.5`body-end.swig`

#### 打字特效、鼠标点击特效

之前版本：[Hexo-NexT 添加打字特效、鼠标点击特效](https://tding.top/archives/58cff12b.html)中，以下代码是放在 `hexo/next/_layout/_custom/custom.swig` 文件中的，现在这部分代码需要放到 `hexo/source/_data/body-end.swig` 文件中：

```
{# 鼠标点击特效 #}
{% if theme.cursor_effect == "fireworks" %}
  <script async src="/js/cursor/fireworks.js"></script>
{% elseif theme.cursor_effect == "explosion" %}
  <canvas class="fireworks" style="position: fixed;left: 0;top: 0;z-index: 1; pointer-events: none;" ></canvas>
  <script src="//cdn.bootcss.com/animejs/2.2.0/anime.min.js"></script>
  <script async src="/js/cursor/explosion.min.js"></script>
{% elseif theme.cursor_effect == "love" %}
  <script async src="/js/cursor/love.min.js"></script>
{% elseif theme.cursor_effect == "text" %}
  <script async src="/js/cursor/text.js"></script>
{% endif %}

{# 打字特效 #}
{% if theme.typing_effect %}
  <script src="/js/activate-power-mode.min.js"></script>
  <script>
    POWERMODE.colorful = {{ theme.typing_effect.colorful }};
    POWERMODE.shake = {{ theme.typing_effect.shake }};
    document.body.addEventListener('input', POWERMODE);
  </script>
{% endif %}
```

然后在 NexT 的配置文件 `next.yml` 中取消 `body-end.swig` 的注释：

```
custom_file_path:

-  #bodyEnd: source/_data/body-end.swig
+  bodyEnd: source/_data/body-end.swig
```

然后我们在 `next.yml` 中增加如下配置项：

````
# 鼠标点击特效
# mouse click effect: fireworks | explosion | love | text
cursor_effect: fireworks

# 打字特效
# typing effect
typing_effect:
  colorful: true  # 礼花特效
  shake: false    # 震动特效
````

注意：上面所有特效的 JS 代码文件都放在站点的 source 目录下（即 `hexo/source/js/`）而不是主题目录下，如果没有 js 目录，则新建一个。

