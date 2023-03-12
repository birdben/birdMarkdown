---
title: "Python学习（一）开发环境搭建"
date: 2017-02-17 15:58:17
tags: [Python]
categories: [Python]
---

我这里是Mac系统，不同的操作系统安装略有不同，请知晓

这里我使用的是Python3，后续所有的文章不做特殊说明都默认是Python3

### Python安装
```
# 我这里直接使用homebrew安装Python，非常的简单

# Mac安装Python2
$ brew install python

# Ubuntu安装Python2
$ apt-get install python

# Mac安装Python3
$ brew install python3

# Ubuntu安装Python3
$ apt-get install python3

# 查看安装后的Python执行文件
$ ll /usr/local/bin/python*
lrwxr-xr-x  1 yunyu  admin    34B  2 16 17:13 /usr/local/bin/python -> ../Cellar/python/2.7.13/bin/python
lrwxr-xr-x  1 yunyu  admin    41B  2 16 17:13 /usr/local/bin/python-config -> ../Cellar/python/2.7.13/bin/python-config
lrwxr-xr-x  1 yunyu  admin    35B  2 16 17:13 /usr/local/bin/python2 -> ../Cellar/python/2.7.13/bin/python2
lrwxr-xr-x  1 yunyu  admin    42B  2 16 17:13 /usr/local/bin/python2-config -> ../Cellar/python/2.7.13/bin/python2-config
lrwxr-xr-x  1 yunyu  admin    37B  2 16 17:13 /usr/local/bin/python2.7 -> ../Cellar/python/2.7.13/bin/python2.7
lrwxr-xr-x  1 yunyu  admin    44B  2 16 17:13 /usr/local/bin/python2.7-config -> ../Cellar/python/2.7.13/bin/python2.7-config
lrwxr-xr-x  1 yunyu  admin    35B  2 16 17:14 /usr/local/bin/python3 -> ../Cellar/python3/3.6.0/bin/python3
lrwxr-xr-x  1 yunyu  admin    42B  2 16 17:14 /usr/local/bin/python3-config -> ../Cellar/python3/3.6.0/bin/python3-config
lrwxr-xr-x  1 yunyu  admin    37B  2 16 17:14 /usr/local/bin/python3.6 -> ../Cellar/python3/3.6.0/bin/python3.6
lrwxr-xr-x  1 yunyu  admin    44B  2 16 17:14 /usr/local/bin/python3.6-config -> ../Cellar/python3/3.6.0/bin/python3.6-config
lrwxr-xr-x  1 yunyu  admin    38B  2 16 17:14 /usr/local/bin/python3.6m -> ../Cellar/python3/3.6.0/bin/python3.6m
lrwxr-xr-x  1 yunyu  admin    45B  2 16 17:14 /usr/local/bin/python3.6m-config -> ../Cellar/python3/3.6.0/bin/python3.6m-config
lrwxr-xr-x  1 yunyu  admin    35B  2 16 17:13 /usr/local/bin/pythonw -> ../Cellar/python/2.7.13/bin/pythonw
lrwxr-xr-x  1 yunyu  admin    36B  2 16 17:13 /usr/local/bin/pythonw2 -> ../Cellar/python/2.7.13/bin/pythonw2
lrwxr-xr-x  1 yunyu  admin    38B  2 16 17:13 /usr/local/bin/pythonw2.7 -> ../Cellar/python/2.7.13/bin/pythonw2.7
```

### 配置环境变量
```
PYTHON2_HOME=/usr/local/bin/python2
PYTHON3_HOME=/usr/local/bin/python3

PATH=$PYTHON2_HOME/bin:$PYTHON3_HOME/bin:$PATH

export PYTHON2_HOME
export PYTHON3_HOME
export PATH
```

这里我们把Python2和Python3的环境变量都配置好了，配置好了检查一下Python的版本

```
$ python2 --version
Python 2.7.13

$ python3 --version
Python 3.6.0
```

### Helloworld

#### 命令行用法

Python2的Helloworld示例

```
$ python2
Python 2.7.13 (default, Dec 17 2016, 23:03:43)
[GCC 4.2.1 Compatible Apple LLVM 8.0.0 (clang-800.0.42.1)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>>
>>> print "helloworld"
helloworld
>>>
>>> exit()
```

Python3的Helloworld示例

```
$ python3
Python 3.6.0 (default, Dec 24 2016, 00:01:50)
[GCC 4.2.1 Compatible Apple LLVM 8.0.0 (clang-800.0.42.1)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>>
>>> print("helloworld")
helloworld
>>>
>>> exit()
```

#### 程序用法

hello.py写一段小程序

```
print('Hello World!')
```

在终端调用python脚本

```
$ python3 hello.py
```

#### 脚本用法

修改上面写的hello.py小程序，在脚本文件上面加上使用python执行

```
#!/usr/local/bin/python3
print('Hello World!')
```

给hello.py脚本可执行权限，然后执行hello.py脚本

```
$ chmod +x hello.py
$ ./hello.py
Hello World!
```

### pip安装问题

```
# Mac如果是使用brew安装的Python默认是已经安装好了pip，所以不需要额外安装了

# Ubuntu安装Python2的pip
$ sudo apt-get install python-pip

# Ubuntu安装Python3的pip
$ sudo apt-get install python3-pip

# 查看Python2的pip版本
$ pip --version
pip 9.0.1 from /usr/local/lib/python2.7/site-packages (python 2.7)

# 查看Python3的pip版本
$ pip3 --version
pip 9.0.1 from /usr/local/lib/python3.6/site-packages (python 3.6)

# 查看Python2安装的pip包路径
$ ll /usr/local/Frameworks/Python.framework/Versions/3.6/lib/python3.6/site-packages/

# 查看Python3安装的pip包路径
$ ll /usr/local/Frameworks/Python.framework/Versions/2.7/lib/python2.7/site-packages/

# pip安装卸载elasticsearch包
$ pip install elasticsearch
$ pip uninstall elasticsearch

# pip3安装卸载elasticsearch包
$ pip3 install elasticsearch
$ pip3 uninstall elasticsearch
```