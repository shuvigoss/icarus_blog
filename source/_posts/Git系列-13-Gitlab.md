---
title: Git系列-13-Gitlab
date: 2016-09-20 18:52:46
categories: Git
---
### gitlab简介

详细内容可查看gitlab[官方文档](https://doc.gitlab.cc/ce/).这里我介绍一些很常用的功能。

<!--more-->
#### 权限管理
- root 可以做任何事情。可删除Admin 用户
- Admin 与root 类似
- normal 普通用户

**新增用户**
![](https://cloud.githubusercontent.com/assets/3062921/18001935/d66f7534-6bb7-11e6-8f42-2979de244020.png)
![](https://cloud.githubusercontent.com/assets/3062921/18001937/d6ed8352-6bb7-11e6-8a44-47b190f598e0.png)
### group & project

group是project的合集。一个group下会有多个project

**group 权限**

*  Public ：未登录用户可以clone
*  Internal ：只有登录用户可以clone
*  Private ：只有属于这个group的用户可见

**project 权限**

*  Public ：未登录用户可以clone
*  Internal ：只有登录用户可以clone
*  Private ：只有属于这个group的用户可见

**将用户添加到group**
![](https://cloud.githubusercontent.com/assets/3062921/18001936/d6e5dca6-6bb7-11e6-96fa-8a0b1fc39597.png)

总体来说gitlab权限管理比较简单，与github很相近。

#### 神器API
gitlab提供了大量的[api](https://doc.gitlab.cc/ce/api/README.html)，可以配合自己的系统进行对接操作。

* Award Emoji
* Branches
* Builds
* Build triggers
* Build Variables
* Commits
* Deploy Keys
* Groups
* Group Access Requests
* Group Members
* Issues
* Keys
* Labels
* Merge Requests
* Milestones
* Open source license templates
* Namespaces
* Notes (comments)
* Pipelines
* Projects including setting Webhooks
* Project Access Requests
* Project Members
* Project Snippets
* Repositories
* Repository Files
* Runners
* Services
* Session
* Settings
* Sidekiq metrics
* System Hooks
* Tags
* Users
* Todos

#### wiki
gitlab、github都内置了wiki
![](https://cloud.githubusercontent.com/assets/3062921/18002109/9b168378-6bb8-11e6-866c-c1ee0dc28acb.png)

支持Markdown，RDoc，AsciiDoc格式文档，非常方便。

#### 其他
> 可以使用runner做持续集成，发布等。 
> 
> SystemHooks 对于增加project或者group会有系统通知
> 
> Message 可以对全局用户发送通知
> 
> ... and so on



