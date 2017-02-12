---
title: "Go学习（二）引用第三方包"
date: 2016-12-27 11:54:42
tags: [Go]
categories: [Go]
---

上一篇开发环境搭建已经简单的引用了第三方glog包，这里我们回顾一下

### 引用第三方包

若你在包的导入路径中包含了代码仓库的URL，go get就会自动地获取、构建并安装它：

```
$ go get github.com/golang/glog
```

若指定的包不在工作空间中，go get就会将会将它放到GOPATH指定的第一个工作空间内。（若该包已存在，go get就会跳过远程获取，其行为与go install相同）

在执行完上面的go get命令后，工作空间的目录树看起来应该是这样的：
			
```
workspace_go
     |
     |--- TestGo
     |      |
     |      |--- src
     |      |     |
     |      |     |--- github.com
     |      |     |      |
     |      |     |      |--- golang
     |      |     |             |
     |      |     |             |--- glog
     |      |     |                    |
     |      |     |                    |--- LICENSE
     |      |     |                    |
     |      |     |                    |--- README
     |      |     |                    |
     |      |     |                    |--- glog.go
     |      |     |                    |
     |      |     |                    |--- glog_file.go
     |      |     |                    |
     |      |     |                    |--- glog_test.go
     |      |     |
     |      |     |--- hello
     |      |     |      |
     |      |     |      |--- hello.go
     |      |     |
     |      |     |--- main
     |      |            |
     |      |            |--- main.go
     |      |
     |      |--- bin
     |      |     |
     |      |     |--- main
     |      |
     |      |
     |      |--- pkg
     |            |
     |            |--- github.com
     |            |      |
     |            |      |--- golang
     |            |             |
     |            |             |--- glog.a
     |            |
     |            |--- hello.a
     |
     |-- GoDemo
```

hello.go

```
package hello

import "fmt"
import "flag"
import "github.com/golang/glog"

func Hello() {
    flag.Parse()
    fmt.Println("Hello World!")
    glog.Info("glog Hello World!")
    glog.Flush()
}
```

注意事项:  

1. 因为是使用时依参数配置，所以需在main()中加上flag.Parse()
2. 需在结尾加上glog.Flush()
3. 依级别生成不同的日志文件，但级别高的日志信息会同时在级别低的日志文件中输出

```
$ go run main.go -log_dir=./
Hello World!
$ ll
lrwxr-xr-x  1 yunyu  staff   51 12 27 11:44 main.INFO -> main.localhost.yunyu.log.INFO.20161227-114425.19830
-rw-r--r--  1 yunyu  staff   72 12 22 14:39 main.go
-rw-r--r--  1 yunyu  staff  186 12 27 11:44 main.localhost.yunyu.log.INFO.20161227-114425.19830

$ vi main.INFO
Log file created at: 2016/12/27 11:48:08
Running on machine: localhost
Binary: Built with gc go1.7.3 for darwin/amd64
Log line format: [IWEF]mmdd hh:mm:ss.uuuuuu threadid file:line] msg
I1227 11:48:08.088596   19951 hello.go:10] glog Hello World!
```

### import用法

```
import(
    "fmt"
)
```

上面这个fmt是Go语言的标准库，他其实是去GOROOT下去加载该模块（先找GOROOT，如果GOROOT找不到在去GOPATH找），当然Go的import还支持如下两种方式来加载自己写的模块：

#### 相对路径

```
import "./model"
```

当前文件同一目录的model目录，但是不建议这种方式来import

#### 绝对路径

```
import "shorturl/model"
```

加载gopath/src/shorturl/model模块

#### 点操作

```
import( . "fmt" ) 
```

这个点操作的含义就是这个包导入之后在你调用这个包的函数时，你可以省略前缀的包名，
也就是前面你调用的fmt.Println("hello world")可以省略的写成Println("hello world")

#### 别名操作

别名操作顾名思义我们可以把包命名成另一个我们用起来容易记忆的名字

```
import( f "fmt" )
```

别名操作的话调用包函数时前缀变成了我们的前缀，即f.Println("hello world")

#### _操作

这个操作经常是让很多人费解的一个操作符，请看下面这个import

```
import ( "database/sql" _ "github.com/ziutek/mymysql/godrv" )
```

_操作其实是引入该包，而不直接使用包里面的函数，而是调用了该包里面的init函数。

要理解这个问题，需要看下面这个图，理解包是怎么按照顺序加载的：

程序的初始化和执行都起始于main包。如果main包还导入了其它的包，那么就会在编译时将它们依次导入。有时一个包会被多个包同时导入，那么它只会被导入一次（例如很多包可能都会用到fmt包，但它只会被导入一次，因为没有必要导入多次）。当一个包被导入时，如果该包还导入了其它的包，那么会先将其它包导入进来，然后再对这些包中的包级常量和变量进行初始化，接着执行init函数（如果有的话），依次类推。等所有被导入的包都加载完毕了，就会开始对main包中的包级常量和变量进行初始化，然后执行main包中的init函数（如果存在的话），最后执行main函数。下图详细地解释了整个执行过程：

![](http://static.open-open.com/lib/uploadImg/20131113/20131113222223_265.png)

通过上面的介绍我们了解了import的时候其实是执行了该包里面的init函数，初始化了里面的变量，_操作只是说该包引入了，我只初始化里面的 init函数和一些变量，但是往往这些init函数里面是注册自己包里面的引擎，让外部可以方便的使用，就很多实现database/sql的引起，在 init函数里面都是调用了sql.Register(name string, driver driver.Driver)注册自己，然后外部就可以使用了。


示例目录结构：

```
GOPATH
  |
  |--bin
  |
  |--pkg
  |
  |--src
      |
      |--main.go
      |
      |--hello
           |
           |--init.go
```

main.go

```
package main

import _ "hello"

func main() {
    // hello.Print() 编译报错，说：undefined: hello
}
```

init.go

```
package hello

import "fmt"

func init() {
    fmt.Println("hello-init() come here.")
}

func Print() {
    fmt.Println("Hello!")
}
```

输出结果：hello-init() come here.

### 总结

#### 命令安装第三方包

安装第三方包命令，这条命令它会把第三方包源代码，下载解压到你的GOPATH路径里面去

```
go get github.com/golang/glog
```

注意：首先必须设置环境变量GOPATH的路径，而且要安装了git客户端，否则go get命令不起作用

在代码中导入下载的那个第三方包

```
import (
    "github.com/golang/glog"
)
```

#### 手动安装第三方包

如果是手动下载压缩包，就解压到$GOPATH/src里面的路径，然后执行go install github.com/golang/glog安装这个包。


参考文章：

- http://www.cnblogs.com/wangqishu/p/5147108.html
- http://www.open-open.com/lib/view/open1384352577352.html
- http://studygolang.com/articles/4356