---
title: Git系列-2-Git安装
date: 2016-09-20 18:51:14
categories: Git
---
### Git安装

    Note:这里的安装是安装Command-Line，并非Git工具。

**如果需要安装Git工具可在[这里](https://git-scm.com/download/gui/mac)找到**
<!--more-->
#### Linux Fedora
```

sudo yum install git

```
#### Linux Debian
```

sudo apt-get install git

```
#### Mac OSX
[官网下载](http://git-scm.com/download/mac)
#### Windows
[官网下载](http://git-scm.com/download/win)
#### 通过Github安装（Windows、Mac OSX）
[Github](https://desktop.github.com)

#### 测试安装是否成功
```

git --version

```

如果出现版本信息说明已经安装成功了。

### Git配置
安装按成后，可以进行Git的配置。

1. `/etc/gitconfig` **--system**
2. `~/.gitconfig 或 ~/.config/git/config` **--global**
3. `.git/config` **repository(当前仓库)**

> repository > golbal > system

**逐级的配置会覆盖上一层配置**


```

git config --global user.name shuwei
git config --global user.email shuvigoss@gmail.com
#查看配置信息
git config --list 

```

    这个配置很重要，因为在每次提交时都会带上你的姓名和邮箱。


