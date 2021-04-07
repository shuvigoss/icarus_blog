---
title: Git系列-6-修改与撤销
date: 2016-09-20 18:51:53
categories: Git
---
### 修改

#### 管理修改
Git是面向修改的，这个特性是有别于其他的VCS，怎么来理解Git的是面向修改的呢？通过以下这个例子你就明白了。
<!--more-->
```
$ echo "fourth blood" >> README.md
$ git add README.md
$ echo "fifth blood" >> README.md
$ git commit -m 'fourth and fifth commit'
[master cc9cccf] fourth and fifth commit
 1 file changed, 1 insertion(+)

$ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

    modified:   README.md

no changes added to commit (use "git add" and/or "git commit -a")
```

我们可以发现add fourth blood 后执行了一次add，再没有执行commit之前又增加了fifth blood，最终执行commit后发现当前工作区的状态并没有把fifth blood 的修改提交上去。

所以我们要记住一次commit是将暂存区提交代码而不会提交工作区。

#### 撤销修改

**有2种修改需要撤销：**

- add
- no add

针对add后的reset我们前边已经做过了`git reset HEAD <file>...`可以完成对add的撤销。

如果当前工作区中没有add modify，可以直接使用`git checkout -- <file>...`来完成对工作区的清理。

```
$ echo "a" >> README.md
$ more README.md
first blood
second blood
third blood
fourth blood
fifth blood
a

$ git checkout -- README.md
$ more README.md
first blood
second blood
third blood
fourth blood
fifth blood
```

#### 删除文件
```
$ touch test.txt
$ git add test.txt
$ git commit -m 'add test.txt'
[master e530961] add test.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 test.txt


$ git rm test.txt
rm 'test.txt'
$ git commit -m 'rm test.txt'
[master 00ddca9] rm test.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 delete mode 100644 test.txt
```
