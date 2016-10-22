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
# 文件表达式
-e filename：如果filename存在（不论filename是文件名还是目录名），则为真
-d filename：如果filename存在（filename必须是目录名），则为真
-f filename：如果filename存在（filename必须是文件名），则为真
-s filename：如果filename内容不为空（filename必须是文件名），则为真

# 字符串变量表达式
if [ -z $var ]：如果var的值为空，则为真
if [ -n $var ]：如果var的值为非空，则为真
if [ $var ]：如果var的值为非空，则为真
```

下面是一些常用的例子

```
filePath=/Users/yunyu/Downloads/test
if [ -e $filePath ]; then
	echo "文件/目录存在"
else 
	echo "文件/目录不存在"
fi

if [ -d $filePath ]; then
	echo "目录存在"
else 
	echo "目录不存在"
fi

if [ -f $filePath ]; then
	echo "文件存在"
else 
	echo "文件不存在"
fi

if [ -s $filePath ]; then
	echo "文件内容不为空"
else 
	echo "文件内容为空"
fi

#var="test";
var="";
# -z 判断一个变量的值是否为空
if [ -z "$var" ]; then
	echo "var is empty"
else
	echo "var is $var"
fi

# -n 判断一个变量是否为非空
if [ -n "$var" ]; then
	echo "var is not empty, value is $var"
else
	echo "var is empty"
fi

# ! -n 和 -z 用法一样
if [ ! -n "$var" ]; then
	echo "var is empty"
else
	echo "var is $var"
fi

if [ "$var" ]; then
	echo "if var is $var"
else
	echo "if var is blank"
fi

var1="1";
var2="2";
# 判断两个变量是否相等
if [ "$var1" = "$var2" ]; then
	echo "$var1 equals $var2"
else
	echo "$var1 not equals $var2"
fi

# 判断两个变量是否不相等
if [ "$var1" != "$var2" ]; then
	echo "$var1 not equals $var2"
else
	echo "$var1 equals $var2"
fi

num1=1;
num2=2;
# 算术比较运算符
if [ "$num1" -eq "$num2" ]; then
	echo "$num1 equals $num2"
elif [ "$num1" -lt "$num2" ]; then
	echo "$num1 less than $num2"
elif [ "$num1" -le "$num2" ]; then
	echo "$num1 less than equals $num2"
elif [ "$num1" -gt "$num2" ]; then
	echo "$num1 great than $num2"
elif [ "$num1" -ge "$num2" ]; then
	echo "$num1 great than equals $num2"
elif [ "$num1" -ne "$num2" ]; then
	echo "$num1 not equals $num2"
fi

# 判断变量是否模糊匹配
base_version='2.2.0'
if [[ $base_version =~ 1.* ]]; then
	base_version='1.x'
fi
if [[ $base_version =~ 2.* ]]; then
	base_version='2.x'
fi
echo $base_version
```

参考文章：

- http://blog.csdn.net/caoyongjunjava/article/details/44560849