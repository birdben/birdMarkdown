## Linux常用命令（四）

### jar命令 

##### jar 替换jar包指定的文件
```
# 方式一：解压缩工具来解压，替换，重新打包

# 方式二：用Java jar 工具来替换
# 这样会直接把test.class直接添加到jar包的根目录。
$ jar uvf test.jar test.class
# 这样就可以替换相应目录的class文件了。
$ jar uvf test.jar com/test/test.class

# 这里值得注意的是test.class必须放在com/test文件下，要和jar的路径对应起来。不然会说没有这个文件或目录jar包和com文件夹的上级在同一个目录。
```


### jps
### jstat
### jstack
### jinfo
### jmap
### jconsole


### jstack命令

```
$jstack [ option ] pid

$jstack [ option ] executable core

$jstack [ option ] [server-id@]remote-hostname-or-IP

参数说明:
pid : java应用程序的进程号,一般可以通过jps来获得
executable : 产生core dump的java可执行程序
core : 打印出的core文件
remote-hostname-or-ip : 远程debug服务器的名称或IP
server-id : 唯一id,假如一台主机上多个远程debug服务
```

参考文章：

- http://blog.csdn.net/giianhui/article/details/10085145

JMX

- http://blog.csdn.net/xuefeng0707/article/details/7907945

JVM

- http://www.cnblogs.com/chengxin1982/p/3949008.html

jconsole

- http://limaoyuan.iteye.com/blog/1541745
- http://jiajun.iteye.com/blog/810150
- http://blog.csdn.net/lifuxiangcaohui/article/details/36896199
- http://jingyan.baidu.com/article/acf728fd3c568af8e410a37a.html