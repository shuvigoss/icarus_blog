---
title: Git系列-8-远程仓库
date: 2016-09-20 18:52:09
categories: Git
---
### 远程仓库

    http://10.211.55.9/ 本地gitlab地址


#### 将本地learngit代码托管到gitlab
1.首先先在gitlab上创建自己的project
![](https://cloud.githubusercontent.com/assets/3062921/17994900/7ce787e0-6b8f-11e6-8c8d-adbfdc83e8bd.png)
2.确认创建信息。
![](https://cloud.githubusercontent.com/assets/3062921/17994967/031b2b78-6b90-11e6-9917-48d3cc86d89c.png)
<!--more-->
##### http协议
回到learngit工作区。
```
$ git remote add origin http://10.211.55.9/shuwei/learngit.git
$ git push -u origin master
Username for 'http://10.211.55.9': shuwei
Password for 'http://shuwei@10.211.55.9':
Counting objects: 19, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (10/10), done.
Writing objects: 100% (19/19), 1.45 KiB | 0 bytes/s, done.
Total 19 (delta 2), reused 0 (delta 0)
To http://10.211.55.9/shuwei/learngit.git
 * [new branch]      master -> master
Branch master set up to track remote branch master from origin.

$ git remote -v
origin  http://10.211.55.9/shuwei/learngit.git (fetch)
origin  http://10.211.55.9/shuwei/learngit.git (push)
```
通过http协议托管到gitlab时需要输入用户名密码，这个就是你gitlab登录用户的用户名密码。
如果觉得输入账号密码麻烦，可以使用git协议。

##### git协议
在完成[将本地learngit代码托管到gitlab](#user-content-将本地learngit代码托管到gitlab)后，可以生成本地公钥。

在用户主目录下，看看有没有.ssh目录，如果有，再看看这个目录下有没有id_rsa和id_rsa.pub这两个文件，如果已经有了，可直接跳到下一步。如果没有，打开Shell（Windows下打开Git Bash），创建SSH Key：

```
$ ssh-keygen -t rsa -C "shuvigoss@gmail.com"
一路回车
cat ~/.ssh/id_rsa.pub
```
可以看到公钥信息，copy

打开用户Profile setting
![](https://cloud.githubusercontent.com/assets/3062921/17995381/c1a236f6-6b93-11e6-918e-d3e324fc5fe6.png)

将公钥配置上
![](https://cloud.githubusercontent.com/assets/3062921/17995408/0de5acf0-6b94-11e6-8ff7-92e077555cf1.png)

```
$ git remote add origin git@10.211.55.9:shuwei/learngit.git
$ git push -u origin master
Counting objects: 19, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (10/10), done.
Writing objects: 100% (19/19), 1.45 KiB | 0 bytes/s, done.
Total 19 (delta 2), reused 0 (delta 0)
To git@10.211.55.9:shuwei/learngit.git
 * [new branch]      master -> master
Branch master set up to track remote branch master from origin.
```

这样免密的推送就完成了。


#### clone远程仓库到本地

```
$ mkdir learngit1
$ git clone git@10.211.55.9:shuwei/learngit.git learngit1/
Cloning into 'learngit1'...
remote: Counting objects: 19, done.
remote: Compressing objects: 100% (10/10), done.
remote: Total 19 (delta 2), reused 0 (delta 0)
Receiving objects: 100% (19/19), done.
Resolving deltas: 100% (2/2), done.
Checking connectivity... done.
```

`clone`命令可以指定clone到哪个文件夹
