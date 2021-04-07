---
title: Git系列-9-分支管理
date: 2016-09-20 18:52:17
categories: Git
---
### 分支管理

分支管理是VCS必备的技能，Git分支管理是他的一大特点，很快而且高效。
<!--more-->
#### 创建分支

```
$ git checkout -b dev
Switched to a new branch 'dev'

$ git branch
* dev
  master

$ echo "dev" >> README.md
$ git add README.md
$ git commit -m 'branch dev create'
[dev fb43350] branch dev create
 1 file changed, 1 insertion(+)

$ git push -u origin dev
Counting objects: 3, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 271 bytes | 0 bytes/s, done.
Total 3 (delta 1), reused 0 (delta 0)
remote:
remote: Create merge request for dev:
remote:   http://ubuntu/shuwei/learngit/merge_requests/new?merge_request%5Bsource_branch%5D=dev
remote:
To git@10.211.55.9:shuwei/learngit.git
 * [new branch]      dev -> dev
Branch dev set up to track remote branch dev from origin.
```

```
-b参数表示创建并切换
$ git branch dev
$ git checkout dev
```

`branch dev`->`checkout dev`->`modify`->`add`->`commit`->`push dev`

查看gitlab上是否已经增加了dev分支

![](https://cloud.githubusercontent.com/assets/3062921/17997327/87b7eab4-6ba0-11e6-8e46-24c6230f05e7.png)

确实，dev分支已经创建成功了。还可以比对分支与master之间的差异
![](https://cloud.githubusercontent.com/assets/3062921/17997389/cc7e0908-6ba0-11e6-9d85-95d175f8f2ea.png)

#### 合并分支
    Note：和并前必须先切换到你要合并到的目标分支

```
$ git checkout master
Switched to branch 'master'
Your branch is up-to-date with 'origin/master'.

$ git merge dev
Updating 0b032b6..fb43350
Fast-forward
 README.md | 1 +
 1 file changed, 1 insertion(+)
```

#### 删除分支
这样就完成了一次分支合并，如果需要可以在合并完成后删除dev分支`$ git branch -d dev 本地分支` `$ git push origin --delete dev 远程分支`

