---
title: Git系列-7-代码托管
date: 2016-09-20 18:52:01
categories: Git
---
### 安装代码托管服务

> 如果做开源项目:首选肯定是[github](https://github.com)。但是由于github服务器在国外，也遇到过被墙的情况，备选可以将代码托管给[开源中国](http://git.oschina.net/)、[coding](https://coding.net/git)， 同时他们也支持付费托管私有仓库。
<!--more-->
当然，做私有代码托管最好的方式还是搭建自己的Git代码托管服务，目前较好的服务有2个。

1. [gitlab](https://about.gitlab.com)
2. [gogs](https://gogs.io)

gogs是新起的开源代码托管服务。是用go语言写的，支持跨平台，最主要是中国人主导的开源项目，维护以及更新非常及时。缺点是功能并没有太多，如果纯粹做一些代码管理是足够的。

gitlab是一个老牌的也是使用人数最多的开源代码托管服务，是用ruby on rails写的，稳定性和功能非常强大，几乎与github不相上下。

这里我选择使用gitlab做我的远程代码托管服务。gitlab的安装就不在这儿说了，需要的可自行在gitlab官网下载并安装。