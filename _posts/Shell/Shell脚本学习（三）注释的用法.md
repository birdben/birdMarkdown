---
title: "Shell脚本学习（三）注释的用法"
date: 2016-08-11 13:06:18
tags: [Shell]
categories: [Shell]
---

#### Shell脚本的注释

Shell脚本单行注释用#，这个我想大家应该都知道。如果要把一段代码全部注释掉，可以用如下方法

```
#!bin/bash

echo "我不是单行注释"
# echo "我是单行注释，你看不到我"

echo "我不是多行注释"
:<<COMMENT
echo "我是多行注释1，你看不到我"
echo "我是多行注释2，你看不到我"
COMMENT
echo "我没有看到多行注释1和多行注释2"
```

其实COMMENT可以随意命名，只要别跟中间的注释内容相同即可。当Shell脚本执行遇到:<<COMMENT，就不执行脚本了，一直到再碰到COMMENT后才重新开始执行脚本。如果忘记写COMMENT或者写错(由于已经不执行脚本了，所以即使写错也不会报错)，则:<<COMMENT之后的脚步将都不会执行。

参考文章：

- http://blog.csdn.net/xyqzki/article/details/8762035

#### Shell脚本的文件注释模板

```
#!bin/bash
# ----------------------------------------------------------------------
# name:			login.sh
# version:		1.0
# createTime:	2016-06-22
# description:	shell脚本的功能描述
# author:		birdben
# email:		191654006@163.com
# github:		https://github.com/birdben
# ----------------------------------------------------------------------
```

这里推荐一个比较好的Shell代码规范

- http://blog.csdn.net/wirelessqa/article/details/18863403