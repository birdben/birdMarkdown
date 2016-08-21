---
title: "Git常用命令"
date: 2016-08-20 23:25:38
tags: [Git命令]
categories: [Git]
---

### Git区域划分

在介绍git命令之前，我们先简单了解下git的区域划分，这样有帮助于理解git的命令，可以将git简单的分为三个区域

1. 工作区（working directry） 
2. 暂缓区（stage index） 
3. 历史记录区（history）   

![Git区域的理解](http://img.blog.csdn.net/20160820133106483?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![Git区域的理解1](http://img.blog.csdn.net/20160820133129026?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

### Git 基础命令

```
# 本地初始化一个空的本地Git repository in /.git目录
$ git init

# 查看当前文件的状态
$ git status

# 添加文件README.md到index暂缓区（就是告诉git要开始记录跟踪README.md文件的修改记录）
$ git add README.md
$ git add '*.md'

# index暂缓区的文件还没存储到本地的Git repository，需要commit提交index暂缓区的文件到本地的Git repository
$ git commit -m "init"

# 在github服务器上创建一个新的远程Git repository，通过git remote add命令给本地的Git repository添加远程Git repository的提交地址，然后git push把本地已经commit的代码推送到远程的Git repository
$ git remote add origin https://github.com/birdben/birdGit.git
# -u参数是告诉git记住参数，下次就可以使用git push命令来提交到远程Git repository
$ git push -u origin master

# 从远端的Git repository pull最新的代码到本地的Git repository
$ git pull origin master
```

### Git 比较相关命令

```
# 查看当前没有add的内容的修改
# 此命令比较的是工作目录(Working tree)和暂存区域快照(index)之间的差异
$ git diff

# 只查看哪些文件做了修改的简单结果，不查看具体的修改内容可以使用--stat参数
$ git diff --stat

# 查看已经add但没有commit的修改
# 查看已经暂存起来的文件(index)和上次提交时的快照之间(HEAD)的差异
$ git diff --cached
$ git diff --staged

# 上面两条的合并
# 显示工作版本(Working tree)和HEAD的差别
$ git diff HEAD

# 查看当前目录和另外一个分支的差别
$ git diff new_branch

# 指定两个版本比较src文件夹的修改
$ git diff SHA1 SHA2
```

### Git 查看历史相关命令

```
# 查看已经commit的历史记录
$ git log

commit a3f3ce4ff543e34fc36fb46d7f71ec444c18b9a3
Author: liuben <191654006@163.com>
Date:   Thu Aug 18 16:11:05 2016 +0800

添加推荐网址

commit 128da6083a1d10d540dced91732ea04165810bba
Author: liuben <191654006@163.com>
Date:   Thu Aug 18 15:17:57 2016 +0800

init

# 只查看最后一次的commit修改的注释
$ git log -n 1

# 只查看最后一次的commit修改的文件列表
$ git log -n 1 --stat

# 只查看最后一次的commit修改的文件细节
$ git log -n 1 -p
```

### Git 回滚相关

```
###### git checkout(working tree内的回滚) ######
# git checkout : 把index区域中的文件覆盖working tree中的文件

# 回滚单个文件
$ git checkout file1

# 回滚多个文件，中间用空格隔开即可
$ git checkout file1 file2 ... fileN

# 回滚当前目录一下的所有working tree内的修改，会递归扫描当前目录下的所有子目录
$ git checkout .


# git checkout HEAD : 会用HEAD指向的master分支中的全部或者部分文件替换index暂存区和以及working tree工作区中的文件。这个命令也是极具危险性的，因为不但会清除working tree工作区中未提交的改动，也会清除index暂存区中未提交的改动 

# 回滚单个文件
$ git checkout HEAD file1

# 回滚多个文件，中间用空格隔开即可
$ git checkout HEAD file1 file2 ... fileN

# 回滚当前目录一下的所有working tree内的修改，会递归扫描当前目录下的所有子目录
$ git checkout HEAD .


###### git reset(index内的回滚) ######
# git reset语法（<commit>必须是没有push到remote端的）
git reset [-q] [<commit>] [--] <paths>…
git reset (--patch | -p) [<commit>] [--] [<paths>…]
git reset (--soft | --mixed | --hard | --merge | --keep) [-q] [<commit>或HEAD]

# 将index区域中修改过的文件移除index，也就是恢复到working tree中
# 文件被恢复到working tree中，回滚操作就是上面提到的git checkout
# 回滚单个文件，相当于git add file1的反命令
$ git reset file1

# 回滚某一个目录，相当于git add .的反命令
$ git reset .


###### git reset(commit之后的回滚（HEAD内的回滚）) ######
# 修改前一次的提交，并且保持前一次的Change-Id不变（推荐使用）
# 直接本地修改好上一次commit的文件，再次add，使用commit --amend提交，修改注释即可修改上一次的提交
$ git commit --amend

# 回滚到某个版本，只回滚了HEAD的信息，不会恢复到index一级。如果还要提交未commit的文件，直接commit即可
# 参数是git log中每次commit的ID
$ git reset --soft cb0c40643afa791ea2c7905318cf17b4eac4bce5

# 此为默认方式，不带任何参数的git reset，即时这种方式，它回滚到某个版本，只保留working tree中文件的源码，回滚了HEAD和index的信息
# 参数是git log中每次commit的ID
$ git reset --mixed cb0c40643afa791ea2c7905318cf17b4eac4bce5

# 彻底回滚到某个版本，本地working tree的源码也会变为上一个版本的内容
# 参数是git log中每次commit的ID
$ git reset --hard cb0c40643afa791ea2c7905318cf17b4eac4bce5


###### merge回滚的文件和本地working tree文件 ######
$ git reset --merge cb0c40643afa791ea2c7905318cf17b4eac4bce5


# 下面是git help给出的提示，很清晰的写明会回滚的区域
--mixed               reset HEAD and index
--soft                reset only HEAD
--hard                reset HEAD, index and working tree
--merge               reset HEAD, index and working tree


###### git revert ######
# 回滚到某一次的commit，但是如果本地working tree有修改需要先merge，然后重新add到index区，再执行git revert --continue回滚到指定版本，同时编辑本地回滚的commit的注释，然后在查看git log就会发现多了一次commit信息，add的文件出现了之前版本的内容修改，需要重新merge在提交

# 回滚到某一次的commit
$ git revert cb0c40643afa791ea2c7905318cf17b4eac4bce5

# merge完本地working tree中的文件内容重新add并且revert continue
$ git add file1
$ git revert --continue

# 查看提交历史记录会发现多了一条revert的记录
$ git log


###### reset和revert的区别 ######
git reset : git reset是指向原地或者向前移动指针，之前的commit信息会被删除
git revert : git revert是创建一个commit来覆盖当前的commit，指针向后移动，之前的commit信息会被保留
```

### Git branch 分支

```
# 查看当前有哪些分支
$ git branch

* master

# 新建一个分支new_branch
$ git branch new_branch

# 切换到new_branch分支
$ git checkout new_branch

# 新建并且切换到该分支, 例: new_branch
$ git checkout -b new_branch

# 再次查看
$ git branch

* master 
  new_branch

# 添加一个文件testBranch到你的repo
$ git add testBranch

# commit一个文件
$ git commit -m "testBranch"

# 将本地分支提交到远程，如果remote_branch不存在则会自动创建分支
$ git push origin local_branch:remote_branch
$ git push origin new_branch:new_branch

# 查看远程分支
$ git branch -r

  remotes/origin/master
  remotes/origin/new_branch

# 查看本地和远程所有的分支
$ git branch -a

  master
* new_branch
  remotes/origin/master
  remotes/origin/new_branch

# 修改branch的名字
$ git branch -m new_branch rename_branch
$ git branch -a

  master
* rename_branch
  remotes/origin/master
  remotes/origin/new_branch
  
# 删除本地分支（删除new_branch之前必须要切换到别的分支上）
$ git branch -d new_branch
# 如果new_branch没有合并到当前master分支的内容，需要使用-D参数强制删除
$ git branch -D new_branch
$ git branch -a

* master
  remotes/origin/master
  remotes/origin/new_branch

# 删除远程分支new_branch
$ git push origin --delete new_branch
$ git push origin  :new_branch
$ git branch -a

* master
  remotes/origin/master
```


### Git 合并分支

首先切换到想要合并到的分支下，运行'git merge’命令 （例如本例中将test2分支合并到test3分支的话，进入test3分支运行git merge test2命令）如果合并顺利的话：

```
# 创建test2分支
$ git branch test2
# 在test2分支中创建test文件，并且提交到test2分支
$ git add test
$ git commit -m "提交test2"

# 创建test3分支
$ git branch test3
# 在test3分支中创建test文件，并且提交到test3分支
$ git add test
$ git commit -m "提交test3"

# 确保当前分支为test3
$ git status

$ git branch -a

  master
  test2
* test3
  remotes/origin/master

# merge分支，如果没有文件冲突就会自动合并成功，如果存在合并冲突需要手动merge处理
$ git merge test2

Already up-to-date. 

# 合并冲突处理：
Automatic merge failed; fix conflicts and then commit the result.

# 修改冲突的文件后，然后git add和git commit，这样就将本地的test2分支merge到test3分支了
$ git add test
$ git commit

# 然后将本地test3分支push到远程
$ git push origin test3:test3
$ git branch -a

  master
  test2
* test3
  remotes/origin/master
  remotes/origin/test3
  
# 本地的test2分支如果没有用就可以删除了
$ git branch -d test2

  master
* test3
  remotes/origin/master
  remotes/origin/test3
```

适合git初学者使用：

- https://try.github.io/levels/1/challenges/1
- http://rogerdudler.github.io/git-guide/index.zh.html

参考文章：

- http://jonyfang.com/blog/2015/11/12/git_command_and_git_branching_model/
- http://www.jianshu.com/p/08ad7e427fec
- http://blog.csdn.net/wirelessqa/article/category/1522507
- http://selfcontroller.iteye.com/blog/1786644