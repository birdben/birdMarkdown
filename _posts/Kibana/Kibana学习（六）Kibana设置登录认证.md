---
title: "Kibana学习（六）Kibana设置登录认证"
date: 2017-02-08 19:00:35
tags: [Kibana]
categories: [Log]
---

前阵子MongoDB和Elasticsearch被黑客利用漏洞入侵并删除数据的事情闹得沸沸扬扬，我们公司使用Elasticsearch也是受害者之一，幸好我们使用Elasticsearch只是应用日志数据，不涉及到业务数据，所以被删除的日志数据对我们影响不是很大。这里我总结一下我们遇到的问题，和处理措施。

Elasticsearch被黑客入侵的事情发生在2017年1月份，当时Elasticsearch技术群有人遇到了索引数据无故丢失的现象，然后出现非法的.warning索引，正好当天我也发现我们Elasticsearch的索引也少了几天的日志索引，打开.warning索引之后发现类似如下信息。

![Warning_Index](http://img.blog.csdn.net/20170212154312823?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

索引的描述信息说明你的数据已经被backup到黑客的服务器上，如果想要恢复需要给他们0.5比特币。

看来这回是真的中招了，开始分析黑客能够入侵的原因。首先想到的就是先把ES的http端口9200禁止对外开放，以前是为了维护ES索引方便，所以把9200端口对外网开放了，虽然做了外网IP的访问控制，但是仍然被入侵进来，说明对外开放ES的9200端口并不安全。然后开始检查服务器的防火墙，对外网开放的IP过滤情况。这里我检查了服务器本身的防火墙并没有问题，然后也咨询过运维的同学ES服务器做了外网IP访问的限制。

但是为什么还会出现这种情况呢？我又仔细的检查了一遍全部的服务器配置，这里我们的服务器使用的是阿里云的服务器，访问的流程是：client客户端 -> SLB服务器 -> Kibana/ES服务器，通过和运维的同学一起排查，终于找到了问题的原因，原因是运维同学的马虎，只对ES服务器进行了外网IP的过滤，没有对SLB服务器进行外网IP的访问限制，而且我们的ES服务器也使用的默认9200端口，所以直接访问SLB的9200即可直接操作我们的ES的索引数据。

问题的原因：

1. ES的http服务端口9200对外网开放
2. 使用了ES的9200端口，极容易被黑客攻击
3. SLB服务器并没有对外网IP进行访问限制
4. 访问Kibana也没有用户名和密码限制，任何人都可以通过公网IP直接看到我们的数据

处理措施：

1. 关闭ES的http服务端口9200对外网开放
2. 在阿里云的SLB代理层做端口的映射，SLB使用19200端口映射到ES服务的9200端口，15601端口映射到Kibana服务的5601端口
3. SLB服务器和ES服务器都设置对外网IP访问限制
4. 在访问Kibana和ES之前加一层Nginx的代理，并且设置基础的用户登录认证

问题的原因就分析到这里，下面我们说明一下如何在Nginx中设置基础的用户登录认证。

主要就是利用Nginx接管所有Kibana请求，通过Nginx配置将Kibana的访问加上权限控制，简单常见的方式可以使用如下两种方式：

方式一：使用Nginx的用户认证模块（ngx_http_auth_basic_module模块，Nginxg一般已经默认安装），用户访问时会直接弹出登录提示，要求输入用户名及密码后登录。
此方案需要依赖htpasswd命令生成密码文件。

方式二：利用Nginx的IP访问控制，限定允许和禁止访问的IP，非允许的IP访问后网页会显示403 Forbidden信息。

这里我们在本地可以尝试方式二，因为我们的线上环境使用了阿里云的SLB代理，SLB已经提供了类似的功能，所以就不在自己的Nginx中进行IP访问设置了。

```
# 需要先安装htpasswd
$ apt-get -y install apache2-utils

# 生成passwd.db文件，接下来需要输入访问Nginx的密码
$ htpasswd -c -d /usr/local/nginx-1.8.0/passwd.db elk_user
New password:
Re-type new password:
Adding password for user elk_user
```

nginx.conf配置文件

```
# 如果是大规模集群环境，此处配置多台Kibana服务器即可（访问Nginx的15601端口后，会轮询访问下面的多台Kibana服务器）
upstream kibana_server {
  server 192.168.99.128:5601;
}

server {
        listen       15601;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            # 权限的描述
            auth_basic "Protect Kibana";
            # 指定我们刚才生成好的passwd.db文件
            auth_basic_user_file /usr/local/nginx-1.8.0/passwd.db;
            root   html;
            index  index.html index.htm;
            # 代理的访问地址是Kibana的地址
            # proxy_pass http://localhost:5601;
            # 如果有多台服务器建议使用upstream配置方式进行访问的负载均衡
            proxy_pass http://kibana_server$request_uri;
            
            # 只允许公司的外网IP，局域网IP可以访问
            # allow   123.**.**.***;
            # allow   192.168.99.0/255;
            # deny    all;
        }
        
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
}

# 如果是大规模集群环境，此处配置多台ES服务器即可（访问Nginx的19200端口后，会轮询访问下面的多台ES服务器）
upstream es_server {
  server 192.168.99.128:9200;
  server 192.168.99.212:9200;
  server 192.168.99.213:9200;
}

server {
        listen       19200;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            # 权限的描述
            auth_basic "Protect Elasticsearch";
            # 指定我们刚才生成好的passwd.db文件
            auth_basic_user_file /usr/local/nginx-1.8.0/passwd.db;
            root   html;
            index  index.html index.htm;
            # 代理的访问地址是Elasticsearch的地址
            # proxy_pass http://localhost:9200;
            # 如果有多台服务器建议使用upstream配置方式进行访问的负载均衡
            proxy_pass http://es_server$request_uri;
            
            # 只允许公司的外网IP，局域网IP可以访问
            # allow   123.**.**.***;
            # allow   192.168.99.0/255;
            # deny    all;
        }
        
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
}
```

配置好Nginx的配置文件之后，需要重新加载

```
# 重新加载Nginx配置文件
$ ./nginx -s reload
```

通过上面的配置访问http://localhost:15601和http://localhost:19200就会弹出用户名密码的验证框，如果输入正确就会跳转到对应http://localhost:5601和http://localhost:9200的地址，也就是Kibana和ES的访问地址。

![ES Protected](http://img.blog.csdn.net/20170212165640900?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYmlyZGJlbg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

也可以通过curl命令来进行测试

```
# elk_user是用户名和密码
# 192.168.99.128:19200是访问Nginx的IP地址，和Nginx监听的端口号
# 如果upstream配置多台ES服务器进行轮询访问，那么下面的name可能每访问一次都会不同
$ curl -i elk_user:elk_user@192.168.99.128:19200
HTTP/1.1 200 OK
Server: nginx/1.4.6 (Ubuntu)
Date: Sun, 12 Feb 2017 08:59:37 GMT
Content-Type: application/json; charset=UTF-8
Content-Length: 308
Connection: keep-alive

{
  "name" : "node-1",
  "cluster_name" : "ben-es",
  "version" : {
    "number" : "2.3.5",
    "build_hash" : "90f439ff60a3c0f497f91663701e64ccd01edbb4",
    "build_timestamp" : "2016-07-27T10:36:52Z",
    "build_snapshot" : false,
    "lucene_version" : "5.5.0"
  },
  "tagline" : "You Know, for Search"
}
```

更多关于ES的安全防护措施请阅读参考文章

参考文章：

- http://elasticsearch.cn/article/129
- https://sanwen8.cn/p/2382AEa.html
- http://blog.csdn.net/yangwenbo214/article/details/54576747