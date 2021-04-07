---
title: Git系列-3-hello world
date: 2016-09-20 18:51:24
categories: Git
---
### 创建仓库

#### init
```

$ mkdir learngit
$ cd learngit
$ git init
Initialized empty Git repository in /Users/shuvigoss/gitrepository/learngit/.git/

```

Git仓库已经创建好了，我们看下文件夹里边都有什么内容。
<!--more-->
```
$ ls -all

drwxr-xr-x  10 shuvigoss  staff   340B Aug 25 15:19 .git

$ cd .git/
$ ls -all

-rw-r--r--   1 shuvigoss  staff   23 Aug 25 15:18 HEAD
drwxr-xr-x   2 shuvigoss  staff   68 Aug 25 15:18 branches
-rw-r--r--   1 shuvigoss  staff  137 Aug 25 15:18 config
-rw-r--r--   1 shuvigoss  staff   73 Aug 25 15:18 description
drwxr-xr-x  11 shuvigoss  staff  374 Aug 25 15:18 hooks
drwxr-xr-x   3 shuvigoss  staff  102 Aug 25 15:18 info
drwxr-xr-x   4 shuvigoss  staff  136 Aug 25 15:18 objects
drwxr-xr-x   4 shuvigoss  staff  136 Aug 25 15:18 refs
```

先看一下里边的内容，这些构成了这个仓库所有功能，包括版本、头指针、config等等，这里可以先不管，如果有时间可以[详细看看](http://blog.jobbole.com/98634/)

#### add & commit & status
```
$ echo "first blood" > README.md
$ git status
On branch master

Initial commit

Untracked files:
  (use "git add <file>..." to include in what will be committed)

    README.md

nothing added to commit but untracked files present (use "git add" to track)
```
```
$ git add README.md
$ git status
On branch master

Initial commit

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)

    new file:   README.md
```
```
$ git commit -m 'first commit'
[master (root-commit) 3d78e6b] first commit
 1 file changed, 1 insertion(+)
 create mode 100644 README.md

$ git status
On branch master
nothing to commit, working directory clean
```


通过`add` `commit` 可以实现网仓库提交修改。

    Note：在提交时一定要加 -m 参数并且写上你提交内容的注释
        比如在一次提交中你增加了个功能A，半年后你想看当时增加的功能A有哪些改动，通过注释就可以很直白的知道。


`status` 命令可以查看当前工作区的状态，是一个非常重要的命令。

`add` 命令不要被他的字面意思迷惑，他的意思是*add modify*而不是单纯的增加文件。从下面的例子就能很直白的明白了。

```
$ echo "second blood" >> README.md
$ more README.md
first blood
second blood

$ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

    modified:   README.md

no changes added to commit (use "git add" and/or "git commit -a")

$ git add *.md 
$ git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

    modified:   README.md

$ git commit -m 'second commit'
[master afef6dd] second commit
 1 file changed, 1 insertion(+)

$ git status
On branch master
nothing to commit, working directory clean
```

其中`add`命令可以使用类似`add *.txt`这种模糊匹配、`add a.txt b.txt`多文件、`add -A`等多种形式。

#### reset
    reset命令是回退命令，功能很多。

```
$ touch a.txt
$ git add a.txt
$ git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

    new file:   a.txt

$ git reset HEAD a.txt
$ rm -rf a.txt
$ git status
On branch master
nothing to commit, working directory clean
```


    (use "git reset HEAD <file>..." to unstage) 
    在add之后，它会提示你使用reset可以进行add 的撤销

#### diff
    diff命令用户比较difference，使用的是Unix通用的diff格式

```
$ echo "third blood" >> README.md
$ git diff README.md
```
```
diff --git a/README.md b/README.md
index 09cd50b..6ffccbe 100644
--- a/README.md
+++ b/README.md
@@ -1,2 +1,3 @@
 first blood
 second blood
+third blood
```

可以看到文件变化了，在文本最后增加了一行"third blood"，通过`diff`命令可以了解到更多文件变更的信息，为提交代码前做一次检查。



