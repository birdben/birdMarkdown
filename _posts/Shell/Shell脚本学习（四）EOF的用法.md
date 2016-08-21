---
title: "Shell脚本学习（四）EOF的用法"
date: 2016-08-15 00:15:18
tags: [Shell]
categories: [Shell]
---

#### Shell脚本的EOF

EOF通常与<<结合使用，<<EOF表示后续的输入作为子命令或子shell的输入参数，直到遇到EOF，再次返回到主shell中。EOF可将其理解为分界符（delimiter），它的作用就是将两个 delimiter 之间的内容传递给交互式程序(cmd命令)作为输入参数。

其使用形式如下：

```
交互式程序(cmd命令)<<EOF
command1
command2
...
EOF
```
- 注意，最后的EOF必须单独占一行。


EOF一般常和cat命令连用。一般有以下两种形式

```
1.将内容直接输出到标准输出（屏幕）
cat <<EOF

2.将标准输出进行重定向，将本应输出到屏幕的内容重定向到文件
cat <<EOF > filename或者cat <<EOF >> filename
```

```
# 将下面的脚本内容输出到helloworld.sh文件中
$ cat << EOF > helloworld.sh
> echo "hello"
> echo "world"
> EOF

$ sh helloworld.sh
```