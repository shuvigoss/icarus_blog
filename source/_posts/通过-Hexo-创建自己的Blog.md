---
title: 通过 Hexo 创建自己的Blog
date: 2016-09-20 15:55:11
categories: Hexo
---

### 选型

很早以前就想搭建一个自己的Blog，记录一些工作中沉淀下来的东西。最近有些时间,So...动起来吧！

<!--more-->
#### 从jekyll到hexo
刚开始选择使用[jekyll](https://jekyllrb.com/),搞了半天各种不顺,再查阅资料的过程中接触到了[hexo](https://hexo.io/zh-cn/),而且[对比](http://www.jianshu.com/p/ce1619874d34)两者,我也选择了hexo.

#### 安装
```
$ sudo npm install -g hexo-cli
```
```
$ hexo init my-blog
$ cd my-blog
$ npm install
$ git clone https://github.com/tufu9441/maupassant-hexo.git themes/maupassant --depth 1
$ npm install hexo-renderer-jade --save
$ npm install hexo-renderer-sass --save
```
由于hexo完全基于NodeJs,所以比较简单.这里我选择了[tufu9441](https://github.com/tufu9441/maupassant-hexo)的Theme变种,感觉不错.

`_config.yml(根目录)`

```
# Hexo Configuration
## Docs: https://hexo.io/docs/configuration.html
## Source: https://github.com/hexojs/hexo/

# Site
title: Shuvigoss'Blog
subtitle: 好记性不如烂笔头
description: 记录一些总是会忘掉的东西
author: Shuvigoss
language: zh-CN
timezone:

# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: http://yoursite.com
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:

# Directory
source_dir: source
public_dir: public
tag_dir: tags
archive_dir: archives
category_dir: categories
code_dir: downloads/code
i18n_dir: :lang
skip_render:

# Writing
new_post_name: :title.md # File name of new posts
default_layout: post
titlecase: false # Transform title into titlecase
external_link: true # Open external links in new tab
filename_case: 0
render_drafts: false
post_asset_folder: false
relative_link: false
future: true
highlight:
  enable: true
  line_number: true
  auto_detect: false
  tab_replace:

# Category & Tag
default_category: uncategorized
category_map:
  hexo:hexo
  java:java
tag_map:

# Date / Time format
## Hexo uses Moment.js to parse and display date
## You can customize the date format as defined in
## http://momentjs.com/docs/#/displaying/format/
date_format: YYYY-MM-DD
time_format: HH:mm:ss

# Pagination
## Set per_page to 0 to disable pagination
per_page: 10
pagination_dir: page

# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: maupassant

# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type:

```

`_config.yml(themes/maupassant/_config.yml)`

```
fancybox: true ## If you want to use fancybox please set the value to true.
duoshuo: ## Your duoshuo_shortname, e.g. username
disqus: ## Your disqus_shortname, e.g. username
google_search: true ## Use Google search, true/false.
baidu_search: ## Use Baidu search, true/false.
swiftype: ## Your swiftype_key, e.g. m7b11ZrsT8Me7gzApciT
tinysou: ## Your tinysou_key, e.g. 4ac092ad8d749fdc6293
self_search: ## Use a jQuery-based local search engine, true/false.
google_analytics: ## Your Google Analytics tracking id, e.g. UA-42425684-2
baidu_analytics: ## Your Baidu Analytics tracking id, e.g. 8006843039519956000
show_category_count: false ## If you want to show the count of categories in the sidebar widget please set the value to true.
shareto: true ## If you want to use the share button please set the value to true.
busuanzi: true ## If you want to use Busuanzi page views please set the value to true.
widgets_on_small_screens: false ## Set to true to enable widgets on small screens.

menu:
  - page: home
    directory: .
    icon: fa-home
  - page: archive
    directory: archives/
    icon: fa-archive
  # - page: about
  #   directory: about/
  #   icon: fa-user
  # - page: rss
  #   directory: atom.xml
  #   icon: fa-rss

widgets: ## Six widgets in sidebar provided: search, category, tag, recent_posts, rencent_comments and links.
  - search
  - category
  - recent_posts

# Static files
js: js
css: css

# Theme version
version: 0.0.0

```
这里边没有改太多,后续再做优化吧.

```
$ hexo server
```

通过访问`http://localhost:4000/`就可以看到页面了.

#### 发布到Github
* 通过[https://pages.github.com/](https://pages.github.com/)创建自己的Pages
* 在blog下安装`npm install hexo-deployer-git --save`,用于发布到github
* 修改根目录下的`_config.yml`
```
deploy:
  type: git
  repo: git@github.com:shuvigoss/shuvigoss.github.io.git
  branch: master
```
* `hexo g` & `hexo d` 进行生成和发布.

#### 域名绑定
* 在[阿里云](https://wanwang.aliyun.com)注册了个[shuvigoss.win](http://www.shuvigoss.win)的域名(10年50块,挺便宜:)
* `$ ping shuvigoss.github.io` 获得该域名IP
* 在阿里云上绑定IP即可

### 参考文章
[https://www.haomwei.com/technology/maupassant-hexo.html](https://www.haomwei.com/technology/maupassant-hexo.html)
[https://hexo.io/zh-cn/docs/setup.html](https://hexo.io/zh-cn/docs/setup.html)
[http://www.jianshu.com/p/ce1619874d34](http://www.jianshu.com/p/ce1619874d34)
[https://segmentfault.com/a/1190000002398039](https://segmentfault.com/a/1190000002398039)
[https://pages.github.com/](https://pages.github.com/)
[http://www.jianshu.com/p/35e197cb1273](http://www.jianshu.com/p/35e197cb1273)