---
title: Git系列-12-tag管理
date: 2016-09-20 18:52:38
categories: Git
---
### 标签管理

标签就是对`commit id`加上一个别名。比如上次提交的`commit id`为`6224937`06ab447b6bb37e4e2a2f276a20fed2ab4，如果我想把这次版本进行发布，后期又能很清楚的知道这次版本发布了什么。那么使用Tag就很简单了。

<!--more-->
#### 给当前版本打Tag
```
$ git tag v1.0
$ git tag
v1.0

$ git show v1.0
commit a44ba6aa022fea773e3a157ee683abc25e255d8d
Author: shuwei <shuvigoss@gmail.com>
Date:   Fri Aug 26 16:02:31 2016 +0800

    mod test.txt

diff --git a/test.txt b/test.txt
index 8b13789..6b673e8 100644
--- a/test.txt
+++ b/test.txt
@@ -1 +1,2 @@

推送tag到upstream
$ git push origin --tags
Total 0 (delta 0), reused 0 (delta 0)
To git@10.211.55.9:shuwei/learngit.git
 * [new tag]         v1.0 -> v1.0
```

#### 给历史版本打Tag

```
$ git log --pretty=oneline --abbrev-commit
a44ba6a mod test.txt
093885d add test.txt
f25f3a7 add 2
6714338 add 1
fb43350 branch dev create
0b032b6 rm test.txt
e530961 add test.txt
2458334 fourth and fifth commit
cc9cccf fourth and fifth commit
a83beff third commit
afef6dd second commit
3d78e6b first commit

$ git tag v0.9 2458334
$ git tag
v0.9
v1.0

推送tag到upstream
$ git push origin --tags
Total 0 (delta 0), reused 0 (delta 0)
To git@10.211.55.9:shuwei/learngit.git
 * [new tag]         v0.9 -> v0.9
```


这两次的版本在gitlab上都可以体现。
![](https://cloud.githubusercontent.com/assets/3062921/17999952/51c5a276-6bae-11e6-9e4f-15e5af2f3377.png)
