---
title: "Git配置多个SSH-Key"
date: 2016-07-05 00:10:21
tags: [Git命令]
categories: [Git]
---

之前周末在家使用github创建SSH-Key进行blog的提交，但是第二天在用公司，使用公司的GitLab提交代码时发现账号是我github的账号，我想肯定是github生成的SSH-Key把之前我公司GitLab的SSH-Key给覆盖了

##### 查看我所有SSH-Key

```
$ cd ~/.ssh/
$ ls
github_rsa.pub	
github_rsa
id_rsa.pub
id_rsa
known_hosts.old
known_hosts
```

这里一共有两个SSH-Key，一个github_rsa是我github的SSH-Key，id_rsa是我公司的GitLab的SSH-Key，因为我周末给自己的github博客创建了一个新的SSH-Key，直接使用的默认路径（~/.ssh/id_rsa.pub），所以就直接把我公司GitLab的SSH-Key给覆盖掉了

这次为了区分开我自己github和公司的GitLab的SSH-Key，在生成SSH-Key文件的时候，我用了不同的名称来区分

##### 公司的GitLab生成一个SSH-Key     

```
# 在~/.ssh/目录会生成gitlab_id-rsa和gitlab_id-rsa.pub私钥和公钥。我们将gitlab_id-rsa.pub中的内容粘帖到公司GitLab服务器的SSH-key的配置中。
$ ssh-keygen -t rsa -C "XXXXXX@XXX.com” -f ~/.ssh/gitlab_id-rsa
```

##### 公司的GitLab生成一个SSH-Key

```
# 在~/.ssh/目录会生成github_id-rsa和github_id-rsa.pub私钥和公钥。我们将github_id-rsa.pub中的内容粘帖到github服务器的SSH-key的配置中。
$ ssh-keygen -t rsa -C "191654006@163.com” -f ~/.ssh/github_id-rsa
```

##### 在~/.ssh目录下添加config配置文件用于区分多个SSH-Key
```
# 添加config配置文件
# vi ~/.ssh/config

# 文件内容如下：
# gitlab
Host gitlab.com
    HostName gitlab.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/gitlab_id-rsa
# github
Host github.com
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/github_id-rsa
    
# 配置文件参数
# Host : Host可以看作是一个你要识别的模式，对识别的模式，进行配置对应的的主机名和ssh文件
# HostName : 要登录主机的主机名
# User : 登录名
# IdentityFile : 指明上面User对应的identityFile路径
```

##### 再次查看目录结构
```
$ cd ~/.ssh/
$ ls
github_id-rsa.pub	
github_id-rsa
gitlab-id_rsa.pub
gitlab-id_rsa
known_hosts.old
known_hosts
```
再次执行git命令已经不再提示权限验证问题

OK，大功告成

参考文章：

- http://my.oschina.net/stefanzhlg/blog/529403