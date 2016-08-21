---
title: "Linux常用命令（三）"
date: 2016-06-26 05:18:31
tags: [Linux命令]
categories: [Linux]
---

### curl命令

以下的案例都使用的百度开源API进行的测试，具体地址如下：
https://www.juhe.cn/docs/api/id/39

- 接口地址：http://v.juhe.cn/weather/index
- 支持格式：json/xml
- 请求方式：get
- 请求示例：http://v.juhe.cn/weather/index?format=2&cityname=%E8%8B%8F%E5%B7%9E&key=您申请的KEY
- 请求参数说明：

|名称	|类型	|必填	|说明
|------|----|-----|----
|cityname	|string	|Y	|城市名或城市ID，如："苏州"，需要utf8 urlencode
|dtype		|string	|N	|返回数据格式：json或xml,默认json
|format	|int		|N	|未来6天预报(future)两种返回格式，1或2，默认1
|key		|string	|Y	|你申请的key


- 接口地址：http://v.juhe.cn/toutiao/index
- 支持格式：json
- 请求方式：get/post
- 请求示例：http://v.juhe.cn/toutiao/index?type=top&key=APPKEY
- 接口备注：返回头条，社会，国内，娱乐，体育，军事，科技，财经，时尚等新闻信息
- 请求参数说明：

|名称	|类型	|必填	|说明
|------|------|------|----
|key	|string	|是	|应用APPKEY
|type	|string	|否	|类型：top(头条，默认),shehui(社会),guonei(国内),guoji(国际),yule(娱乐),tiyu(体育)junshi(军事),keji(科技),caijing(财经),shishang(时尚)

##### curl -d 传递请求数据
```
# 参数：
# -G:指定GET方式发送请求
# --data/-d:指定使用POST方式传递数据
```

##### curl 发送GET请求
```
# 方式一：curl 以?方式传递参数发送GET请求
# 默认curl使用POST方式请求数据，这种方式下直接通过URL传递数据，也就是以?方式添加到url后面
# 格式：curl "https://hostname.com:8080/getMethod?param1=value1&param2=value2"

# 方式二：curl -G -d 发送GET请求
# 但是如果有-d参数，也可以加上-G参数指定使用GET方式请求数据，如果不加上-G则默认使用的POST方式请求数据
# 格式：curl -G -d "param1=value1&param2=value2" "http://hostname.com:8080/getMethod"

# 使用?方式传递参数，不使用-d方式
$ curl "http://v.juhe.cn/weather/index?format=2&cityname=%E8%8B%8F%E5%B7%9E&key=您申请的KEY"

# 也可以使用-d方式传递参数，但是必须指定-G来指定GET方式请求数据，因为该接口只支持GET方式请求，如果不加-G参数，使用-d参数就默认为POST方式请求
$ curl -G -d "format=2&cityname=%E8%8B%8F%E5%B7%9E&key=您申请的KEY" "http://v.juhe.cn/weather/index"

# 以下的方式就会报错，因为该接口只支持GET方式请求，不支持POST方式请求，如果不加-G参数，使用-d参数就默认为POST方式请求
$ curl -d "format=2&cityname=%E8%8B%8F%E5%B7%9E&key=您申请的KEY" "http://v.juhe.cn/weather/index"
# 返回的结果是error_code:203901，获取不到cityname参数
{"resultcode":"201","reason":"Error Cityname!","result":null,"error_code":203901}
```

##### curl 发送POST请求
```
# 格式：curl -d/--data "param1=value1&param2=value2" "http://hostname.com:8080/html/postMethod"
$ curl -d "type=tiyu&key=申请的KEY" "http://v.juhe.cn/toutiao/index"
$ curl --data "type=tiyu&key=申请的KEY" "http://v.juhe.cn/toutiao/index"

# POST提交多个同名参数，paramList是一个数组参数
$ curl -XPOST 'http://hostname.com/postMethod' -d 'userName=birdben&paramList=["aa", "bb"]'

# form表单上传文件
# 可以通过 --form/-F 方式指定使用POST方式传递数据
# --form/-F 参数相当于设置form表单的method="POST"和enctype="multipart/form-data"两个属性
# filename为文本框中的name元素对应的属性值。<type="text" name="filename">
$ curl --form "filename=@filename.txt" "http://hostname.com:8080/html/uploadFile"
$ curl -F "filename=@filename.txt;type=text/plain"  "http://hostname.com:8080/html/uploadFile"
```

##### curl 特殊字符处理
```
# 默认情况下，通过GET/POST方式传递过去的数据中若有特殊字符（或者中文字符），首先需要将特殊字符转义在传递给服务器端，如"value 1"值中包含有空格，则需要先将空格转换成%20，如：
$ curl -d "value%201" "http://hostname.com"

# 在新版本的curl中，提供了新的选项 --data-urlencode，通过该选项提供的参数会自动转义特殊字符。
$ curl --data-urlencode "value 1" "http://hostname.com"
```

##### curl 指定其它协议
```
# 除了使用GET和POST协议外，还可以通过 -X 选项指定其它协议，一般用过ES删除索引的时候用的比较多，如：
$ curl -X DELETE "https://hostname.com"
```

##### curl 设置http请求头信息
```
# 以下操作可以挂代理抓包查看并且验证
# -v 通过使用 -v 和 -trace获取更多的链接信息
# 访问https://www.github.com/下面返回的信息会提示该域名已经被重定向到https://github.com/
$ curl -v "https://www.github.com/"

* Rebuilt URL to: https://www.github.com/
* Hostname was NOT found in DNS cache
*   Trying 192.30.252.130...
* Connected to www.github.com (192.30.252.130) port 443 (#0)
* TLS 1.2 connection using TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
* Server certificate: github.com
* Server certificate: DigiCert SHA2 Extended Validation Server CA
* Server certificate: DigiCert High Assurance EV Root CA
> GET / HTTP/1.1
> User-Agent: curl/7.37.1
> Host: www.github.com
> Accept: */*
>
< HTTP/1.1 301 Moved Permanently
< Content-length: 0
< Location: https://github.com/
< Connection: close
<
* Closing connection 0

# -A 设置http请求头User-Agent，可以指定自己这次访问所模拟的浏览器信息
# 前一个直接用curl命令的，User-Agent: curl/7.37.1
> User-Agent: curl/7.37.1
# 使用-A参数重新指定浏览器信息后，User-Agent: Mozilla/5.0 Firefox/21.0
> User-Agent: Mozilla/5.0 Firefox/21.0
$ curl -v -A "Mozilla/5.0 Firefox/21.0" "https://www.github.com/"

 Hostname was NOT found in DNS cache
*   Trying 192.30.252.120...
* Connected to www.github.com (192.30.252.120) port 443 (#0)
* TLS 1.2 connection using TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
* Server certificate: github.com
* Server certificate: DigiCert SHA2 Extended Validation Server CA
* Server certificate: DigiCert High Assurance EV Root CA
> GET / HTTP/1.1
> User-Agent: Mozilla/5.0 Firefox/21.0
> Host: www.github.com
> Accept: */*
>
< HTTP/1.1 301 Moved Permanently
< Content-length: 0
< Location: https://github.com/
< Connection: close
<
* Closing connection 0

# -e 设置http请求头Referer，解释：检查http访问的Referer是服务器端常用的限制方法。比如你先访问网站的首页，再访问里面所指定的下载页，这第二次访问的referer地址就是第一次访问成功后的页面地址。这样，服务器端只要发现对下载页面某次访问的referer地址不是首页的地址，就可以断定那是个盗链了。所以这里我们可以自己修改我们的Referer链接地址。
$ curl -v -e "http://birdben.com/" "http://birdben.com/download.html"

* Hostname was NOT found in DNS cache
*   Trying 50.63.202.46...
* Connected to birdben.com (50.63.202.46) port 80 (#0)
> GET /download.html HTTP/1.1
> User-Agent: curl/7.37.1
> Host: birdben.com
> Accept: */*
> Referer: http://birdben.com/
>
< HTTP/1.1 302 Found
< Connection: close
< Pragma: no-cache
< cache-control: no-cache
< Location: /download.html
<
* Closing connection 0

# --header/-H 设置header
$ curl -H "Connection:keep-alive" -H "Host:127.0.0.1" -H "Accept-Language:es" -H "Cookie:ID=1234" "https://www.github.com/"

* Hostname was NOT found in DNS cache
*   Trying 192.30.252.129...
* Connected to www.github.com (192.30.252.129) port 443 (#0)
* TLS 1.2 connection using TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
* Server certificate: github.com
* Server certificate: DigiCert SHA2 Extended Validation Server CA
* Server certificate: DigiCert High Assurance EV Root CA
> GET / HTTP/1.1
> User-Agent: curl/7.37.1
> Accept: */*
> Connection:keep-alive
> Host:127.0.0.1
> Accept-Language:es
> Cookie:ID=1234
>
< HTTP/1.1 301 Moved Permanently
< Content-length: 0
< Location: https://github.com/
< Connection: close
<
* Closing connection 0

# -I 设置http响应头处理，仅仅返回header
$ curl -I "https://github.com"

# --dump-header/-D 将http header保存到/tmp/header文件
$ curl -D /tmp/header "https://github.com"
```

##### curl -o/-O 下载
```
# 通过-o/-O选项保存下载的文件到指定的文件中：
-o：将文件保存为命令行中指定的文件名的文件中
-O：使用URL中默认的文件名保存文件到本地

# 将文件下载到本地并命名为test.html
$ curl -o test.html "http://apistore.baidu.com/apiworks/servicedetail/478.html"

# 将文件保存到本地并命名为478.html
$ curl -O "http://apistore.baidu.com/apiworks/servicedetail/478.html"
```

##### curl -L 重定向
```
# 默认情况下CURL不会发送HTTP Location headers(重定向).当一个被请求页面移动到另一个站点时，会发送一个HTTP Loaction header作为请求，然后将请求重定向到新的地址上。

# 例如：浏览器访问京东以前的域名www.360buy.com，会自动将地址重定向到www.jd.com上。通过curl命令就会出现如下信息
$ curl "www.360buy.com"
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html>
<head><title>301 Moved Permanently</title></head>
<body bgcolor="white">
<h1>301 Moved Permanently</h1>
<p>The requested resource has been assigned a new permanent URI.</p>
<hr/>Server: JDWS</body>
</html>

# 这时可以通过使用-L选项进行强制重定向，让curl使用地址重定向，此时就会返回京东首页的信息
$ curl -L "www.360buy.com"
```

##### curl -O 从FTP服务器下载文件
```
# curl同样支持FTP下载，若在url中指定的是某个文件路径而非具体的某个要下载的文件名， curl则会列出该目录下的所有文件名而并非下载该目录下的所有文件

# 列出public_html下的所有文件夹和文件
$ curl -u user:pass -O "ftp://ftp_server/html/"
# 下载index.css文件
$ curl -u user:pass -O "ftp://ftp_server/css/index.css"

# 从FTP服务器下载文件的另一种写法
$ curl "ftp://user:pass@ftp_server:port/path/file"
```

##### curl -T 上传文件到FTP服务器

```
# 通过 -T 选项可将指定的本地文件上传到FTP服务器上
# 将myfile.txt文件上传到服务器
$ curl -u user:pass -T myfile.txt "ftp://ftp.testserver.com:port/path/"

# 同时上传多个文件
$ curl -u user:pass -T "{file1,file2}" "ftp://ftp.testserver.com"
```

##### curl -u 授权（没有测试过）
```
# 在访问需要授权的页面时，可通过-u选项提供用户名和密码进行授权
$ curl -u username:password URL

# 通常的做法是在命令行只输入用户名，之后会提示输入密码，这样可以保证在查看历史记录时不会将密码泄露
$ curl -u username URL
```

##### curl -x 设置代理
```
# -x 选项可以为CURL添加代理功能
# 指定代理主机和端口
$ curl -x proxysever.com:8888 -U user:pass "http://google.co.jp"
```


参考文章：

- https://linux.cn/article-4957-1.html
- http://www.cnblogs.com/gbyukg/p/3326825.html