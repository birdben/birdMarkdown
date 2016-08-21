---
title: "Shell脚本学习（六）if语句的格式问题"
date: 2016-08-21 14:44:03
tags: [Shell]
categories: [Shell]
---

今天在写Shell启动脚本的时候遇到了个比较奇葩的问题，因为对Shell了解的不够，不是很清楚if语句的格式，if语句不按照指定的格式写可能不会得到你想要的结果。先简单描述下我的问题吧

一个简单的判断字符串的脚本，根据base_version变量的值来输出内容

```
#!/bin/bash
base_version='2.x'

if [ $base_version='1.x' ]; then
  echo "当前安装的是1.x版本"
fi
if [ $base_version='2.x' ]; then
  echo "当前安装的是2.x版本"
fi
```

看完上面的脚本之后，我想大家的第一答案应该都是"当前安装的是2.x版本"，但是运行结果却是"当前安装的是1.x版本"和"当前安装的是2.x版本"都输出了。

规范有点严格

```
if空格[空格"xx"空格=空格"xx"空格];空格then
	echo "if"
elif空格[空格"xx"空格=空格"xx"空格];空格then
	echo "elseif"
else
	echo "else"
fi
```

我的脚本在$base_version='1.x'等号左右没有加空格，修改之后如下之后，运行结果是"当前安装的是2.x版本"正确的了。

```
#!/bin/bash
base_version='2.x'

if [ $base_version = '1.x' ]; then
  echo "当前安装的是1.x版本"
fi
if [ $base_version = '2.x' ]; then
  echo "当前安装的是2.x版本"
fi
```

接下来介绍一些if语句常用的判断语句

```
－d：判断指定的变量是否为目录
－z：判断指定的变量是否存在值
－f：判断指定的变量是否为文件
－L：判断指定的变量是否为符号链接（软链接）
－r：判断指定的变量是否可读
－w：判断指定的变量是否可写
－x：判断存在的对象是否可以执行
!：测试条件的否定符号 
```


```
#!/bin/bash

# 判断文件夹是否存在
if [ -d $folder ]; then
	echo "目录存在！"
else
	echo "目录不存在！"
fi

# 如果文件夹不存在，创建文件夹
if [ ! -d "/test/folder" ]; then
  mkdir /test/folder
fi

# shell判断文件,目录是否存在或者具有权限
folder="/software/es/"
file="/software/es/config/elasticsearch.yml"

# -x 参数判断 $folder 是否存在并且是否具有可执行权限
if [ ! -x "$folder"]; then
  mkdir "$folder"
fi

# -d 参数判断 $folder 是否存在
if [ ! -d "$folder"]; then
  mkdir "$folder"
fi

# -f 参数判断 $file 是否存在
if [ ! -f "$file" ]; then
  touch "$file"
fi

# -n 判断一个变量是否有值
if [ ! -n "$var" ]; then
  echo "$var is empty"
fi

# 判断两个变量是否相等
if [ "$var1" = "$var2" ]; then
  echo '$var1 equals $var2'
else
  echo '$var1 not equals $var2'
fi
```

参考文章：

- http://blog.csdn.net/caoyongjunjava/article/details/44560849