---
title: Git系列-4-版本管理
date: 2016-09-20 18:51:34
categories: Git
---
### 版本管理

#### 版本回退
按照之前的操作，我们已经有3次commit，如何查看呢？
<!--more-->
```
$ git log
commit a83beff902f6e37a0b51218d2fba3d911c4d1487
Author: shuwei <shuvigoss@gmail.com>
Date:   Thu Aug 25 16:22:07 2016 +0800

    third commit

commit afef6dd1fc89eb977a1c51de9f1c5a7de7ae5dd1
Author: shuwei <shuvigoss@gmail.com>
Date:   Thu Aug 25 15:48:32 2016 +0800

    second commit

commit 3d78e6b751d368c72d020ab8621c74aa97a1ff4e
Author: shuwei <shuvigoss@gmail.com>
Date:   Thu Aug 25 15:33:39 2016 +0800

    first commit
```

每一次的commit都有记录，包括时间、提交的描述、谁提交的。其中第一行的commit后跟的是每一次提交生成的SHA1串，我们可以管它叫commit id(版本号)，这个版本号是用来做版本回退的核心。比如我想将版本回退到第一次提交也就是`first commit`，那么如何做呢？

```
$ git reset --hard HEAD^^
HEAD is now at 3d78e6b first commit
```

上面又用到了reset命令，其中`HEAD`这个参数就是你要将版本回退到当前HEAD往前的几个版本。比如：

    HEAD^   回退到当前版本的前一版本
    HEAD^^  回退到当前版本的前前一版本
    HEAD~10 回退到当前版本前10个版本

接下来查看当前工作区是否已经回退到我们需要的版本呢？

```
$ more README.md
first blood
```

确实已经是我们第一次提交的内容。

    每个commit id 会有它的简写 比如原始commit id 为 3d78e6b751d368c72d020ab8621c74aa97a1ff4e
    那么它可以使用前七位来表示3d78e6b

那么我们再使用`git log`查看当前的commit log会是什么样的呢？

```
commit 3d78e6b751d368c72d020ab8621c74aa97a1ff4e
Author: shuwei <shuvigoss@gmail.com>
Date:   Thu Aug 25 15:33:39 2016 +0800

    first commit
```

我靠！我后边的内容丢了！完蛋了，回退错了~~
不急，因为我们前边已经有最后一个版本的版本号了，那么再回退回去就好了。

```
$ git reset --hard a83beff
HEAD is now at a83beff third commit
$ git log
commit a83beff902f6e37a0b51218d2fba3d911c4d1487
Author: shuwei <shuvigoss@gmail.com>
Date:   Thu Aug 25 16:22:07 2016 +0800

    third commit

commit afef6dd1fc89eb977a1c51de9f1c5a7de7ae5dd1
Author: shuwei <shuvigoss@gmail.com>
Date:   Thu Aug 25 15:48:32 2016 +0800

    second commit

commit 3d78e6b751d368c72d020ab8621c74aa97a1ff4e
Author: shuwei <shuvigoss@gmail.com>
Date:   Thu Aug 25 15:33:39 2016 +0800

    first commit
```

看，HEAD又回到最后一次提交的版本了。
我们可以把上边2次reset操作通过下面这个图表来表示。

| | first commit | second commit | third commit  |
| ---| :--------- |:-------------:| -----:|
| commit id     | 3d78e6b | afef6dd | 3d78e6b |
| origin      |  |  | HEAD |
| reset1 | HEAD |   |   |
| reset2 |   |   | HEAD |

所以版本的回退Git做的就是将HEAD指针移动到相应的commit id，所以说Git的速度快就体现在这里。

假如你没有记录之前的版本内容，如何获取呢？

```
$ git reflog
a83beff HEAD@{0}: reset: moving to a83beff
3d78e6b HEAD@{1}: reset: moving to HEAD^^
a83beff HEAD@{2}: commit: third commit
afef6dd HEAD@{3}: commit: second commit
3d78e6b HEAD@{4}: commit (initial): first commit
```

`reflog`命令可以记录你每一次的命令。
