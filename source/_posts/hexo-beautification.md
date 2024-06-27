---
title: "Hexo美化"
date: 2024-06-26 22:15:20
tags: 
 - hexo
 - tutorial
---

参考： https://hexo.fluid-dev.com/docs/guide/

# 1. fluid主题安装

命令行输入：

```powershell
PS C:\Users> npm install --save hexo-theme-fluid

added 1 package, and audited 229 packages in 14s

23 packages are looking for funding
  run `npm fund` for details

found 0 vulnerabilities
```

博客目录下创建 `_config.fluid.yml`，将主题的 [_config.yml](https://github.com/fluid-dev/hexo-theme-fluid/blob/master/_config.yml)内容复制过去。

# 2. 指定主题

如下修改 Hexo 博客目录中的 `_config.yml`：

```yaml
theme: fluid  # 指定主题

language: zh-CN  # 指定语言，会影响主题显示的语言，按需修改
```

# 3. 创建「关于页」

首次使用主题的「关于页」需要手动创建：

```bash
hexo new page about
```

创建成功后修改 `/source/about/index.md`，添加 `layout` 属性。

修改后的文件示例如下：

```yaml
---
title: 标题
layout: about
---

这里写关于页的正文，支持 Markdown, HTML
```

*此设置为了显示头像

# 4.启用主题

```powershell
hexo clean
hexo g
hexo s
```

# 5.页面顶部大图设置

```powershell
  # 首页 Banner 头图，可以是相对路径或绝对路径，以下相同
  # 实际图片对应存放在 /source/img/bg/xxx.jpg
  # 修改主题配置文件 _config.fluid.yml
  # Path of Banner image, can be a relative path or an absolute path, the same on other pages
  banner_img: /img/bg/joshua-earle-9idqIGrLuTE-unsplash.jpg

```

# 6. 博客标题

页面左上角的博客标题，可在**主题配置**中设置：

```yaml
navbar:
  blog_title: oharuのblog
```

![image-20240627003413942](hexo-beautification/image-20240627003413942.png)

# 7. 网页统计

目前支持多种统计网站，开启后按需填入 Key 或 ID 即可。

```yaml
  # Google Analytics 4 的媒体资源 ID
  # Google Analytics 4 MEASUREMENT_ID
  # See: https://support.google.com/analytics/answer/9744165#zippy=%2Cin-this-article
  google:
    measurement_id: G-8Q835Q3Z38
```

# 8. 展示 PV 与 UV 统计

页脚可以展示 PV 与 UV 统计数据，目前支持两种数据来源：[不蒜子 ](http://busuanzi.ibruce.info/)。

```yaml
statistics:
    enable: true
```

# 9. Slogan副标题设置

```yaml
  # 首页副标题的独立设置
  # Independent config of home page subtitle
  slogan:
    enable: true

    # 为空则按 hexo config.subtitle 显示
    # If empty, text based on `subtitle` in hexo config
    text: "An elegant Material-Design theme for Hexo"

    # 通过 API 接口作为首页副标题的内容，必须返回的是 JSON 格式，如果请求失败则按 text 字段显示，该功能必须先开启 typing 打字机功能
    # Subtitle of the homepage through the API, must be returned a JSON. If the request fails, it will be displayed in `text` value. This feature must first enable the typing animation
    api:
      enable: true

      # 请求地址
      # Request url
      url: "https://international.v1.hitokoto.cn/"

      # 请求方法
      # Request method
      # Available: GET | POST | PUT
      method: "GET"

      # 请求头
      # Request headers
      headers: {}

      # 从请求结果获取字符串的取值字段，最终必须是一个字符串，例如返回结果为 {"data": {"author": "fluid", "content": "An elegant theme"}}, 则取值字段为 ['data', 'content']；如果返回是列表则自动选择第一项
      # The value field of the string obtained from the response. For example, the response content is {"data": {"author": "fluid", "content": "An elegant theme"}}, the expected `keys: ['data','content']`; if the return is a list, the first item is automatically selected
      keys: ["hitokoto"]

# 一些好玩的功能
# Some fun features
fun_features:
  # 为 subtitle 添加打字机效果
  # Typing animation for subtitle
  typing:
    enable: true

    # 打印速度，数字越大越慢
    # Typing speed, the larger the number, the slower
    typeSpeed: 120

    # 游标字符
    # Cursor character
    cursorChar: "|"

    # 是否循环播放效果
    # If true, loop animation
    loop: true

    # 在指定页面开启，不填则在所有页面开启
    # Enable in specified page, all pages by default
    # Options: home | post | tag | category | about | links | page | 404
    scope: []

```

# 10.文章跳转方式

```yaml
  # 打开文章的标签方式
  # The browser tag to open the post
  # Available: _blank | _self
  post_url_target: _blank
```

# 【TODO】11.文章排序

如果想手动将某些文章固定在首页靠前的位置，可以在安装 `hexo-generator-index` >= 2.0.0 版本的情况下，在文章开头 [front-matter](https://hexo.io/zh-cn/docs/front-matter)中配置 `sticky` 属性：

```yaml
---
title: 文章标题
index_img: /img/example.jpg
date: 2019-10-10 10:00:00
sticky: 100
---
以下是文章内容
```

`sticky` 数值越大，该文章越靠前，达到类似于置顶的效果，其他未设置的文章依然按默认排序。

当文章设置了 `sticky` 后，主题会默认在首页文章标题前增加一个图标，来标识这是一个置顶文章，你可以通过**主题配置**去关闭或修改这个功能：

```yaml
index:
  post_sticky:
    enable: true
    icon: 'iconfont icon-top'
```

`icon` 可以通过[自定义图标](https://hexo.fluid-dev.com/docs/icon/)修改为其他图标

# 【TODO】12.文章在首页的封面图

对于单篇文章，在文章开头 [front-matter (opens new window)](https://hexo.io/zh-cn/docs/front-matter)中配置 `index_img` 属性。

```yaml
---
title: 文章标题
tags: [Hexo, Fluid]
index_img: /img/example.jpg
date: 2019-10-10 10:00:00
---
以下是文章内容
```

和 Banner 配置相同，`/img/example.jpg` 对应的是存放在 `/source/img/example.jpg` 目录下的图片（目录也可自定义，但必须在 source 目录下）。

