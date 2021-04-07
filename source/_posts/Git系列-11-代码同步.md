---
title: Git系列-11-代码同步
date: 2016-09-20 18:52:31
categories: Git
---
### 代码更新

#### git pull & git fetch

* `git pull` = `git fetch` + `git merge`

`$ git fetch <远程主机名> <分支名>`
<!--more-->

```
$ git fetch
取回所有内容，包括分支，tag等

$ git fetch origin master
将origin(upstream) 的master分支取回本地
```


`git pull <远程主机名> <远程分支名>:<本地分支名>`

```
$ git pull origin next:master
将origin的next fetch下来与本地master分支合并。
```

大部分情况下，可以直接使用`pull`命令进行fetch+merge操作，如果你本地代码会有一些测试分支等建议使用`fetch`手动进行合并，否则可能会覆盖掉你本地的一些内容。

