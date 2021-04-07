---
title: Git系列-1-Git简介
date: 2016-09-20 18:50:59
categories: Git
---
### 简介

#### 什么是Git？

Git is a free and open source distributed version control system designed to handle everything from small to very large projects with speed and efficiency

Git是为了快速和高效处理任何大小工程而设计的免费开源的**分布式版本控制系统**。
<!--more-->
#### Git简史

同生活中的许多伟大事物一样，Git 诞生于一个极富纷争大举创新的年代。

Linux 内核开源项目有着为数众广的参与者。 绝大多数的 Linux 内核维护工作都花在了提交补丁和保存归档的繁琐事务上（1991－2002年间）。 到 2002 年，整个项目组开始启用一个专有的分布式版本控制系统 BitKeeper 来管理和维护代码。

到了 2005 年，开发 BitKeeper 的商业公司同 Linux 内核开源社区的合作关系结束，他们收回了 Linux 内核社区免费使用 BitKeeper 的权力。 这就迫使 Linux 开源社区（特别是 Linux 的缔造者 Linux Torvalds）基于使用 BitKcheper 时的经验教训，开发出自己的版本系统。 他们对新的系统制订了若干目标：

- **速度**
- **简单的设计**
- **对非线性开发模式的强力支持（允许成千上万个并行开发的分支）**
- **完全分布式**
- **有能力高效管理类似 Linux 内核一样的超大规模项目（速度和数据量）**

自诞生于 2005 年以来，Git 日臻成熟完善，在高度易用的同时，仍然保留着初期设定的目标。 它的速度飞快，极其适合管理大项目，有着令人难以置信的非线性分支管理系统。

[Git的诞生故事版](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/00137402760310626208b4f695940a49e5348b689d095fc000)

#### 集中式VS分布式

##### 集中式（SVN、CVS等）
![](/images/0.jpeg)

##### 分布式（Git）
![](/images/1.jpeg)

##### 分布式VS集中式

|    | 分布式 | 集中式
---- | ----- | -----
网络依赖性 | 低 | 高
系统稳定性 | 低 | 高
存储扩展性 | 低 | 高

