---
title: "Git的SSH-Key用法"
date: 2016-09-11 16:51:32
tags: [Git命令]
categories: [Git]
---

之前GitHub提交代码的时候总是不知道该使用SSH方式还是Https方式，后来看了GitHub官网上建议使用Https方式，但Https方式有些麻烦，因为每次使用Https方式提交代码的时候都需要输入用户名和密码，而用SSH方式就有免密码登录的方式，只是需要在GitHub服务器添加我们本地的公钥就可以了。但是之前有一点令我一直都不解，虽然我用Https方式提交代码但是并没有让我输入用户名密码，后来找了好久原因才发现是因为我用的Mac笔记本，Mac系统有个钥匙串记录的功能，会将GitHub的用户名密码保存下来。


```
# 首先在本地生成公钥/私钥的键值对
$ ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
# 输入公钥/私钥的文件路径
Enter a file in which to save the key (/Users/you/.ssh/id_rsa): [Press enter]
Enter passphrase (empty for no passphrase): [Type a passphrase]
Enter same passphrase again: [Type passphrase again]
# 复制公钥文件中的内容，并且添加到GitHub上即可
$ cat ~/.ssh/id_dsa.pub
```

同样的道理，如果我们想在本地免密码登录测试服务器，那么我们也可以用这样的方式来设置

```
# 首先在本地生成公钥/私钥的键值对
$ ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
$ cat ~/.ssh/id_dsa.pub

# 将本地的公钥上传到测试服务器
$ scp root@LocalServer:~/.ssh/id_dsa.pub  ~/.ssh/master_dsa.pub
# 将上传的本地公钥添加到authorized_keys中
$ cat ~/.ssh/master_dsa.pub >> ~/.ssh/authorized_keys
```

参考文章：

- https://help.github.com/articles/which-remote-url-should-i-use/
- http://www.jianshu.com/p/1ac06bcd8ab5