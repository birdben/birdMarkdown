---
title: "Kibana学习（三）查看Kibana的Request"
date: 2016-11-22 15:54:06
tags: [Kibana]
categories: [Log]
---

如果查看Kibana发送给ES的request请求，请看下面的图示例，哈哈

![Kibana_request1](http://img.blog.csdn.net/20161122155733354?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

![Kibana_request2](http://img.blog.csdn.net/20161122155832431?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

这里额外说一下使用curl -I获取Kibana Response的Header问题，Kibana要求Request中的Header必须带有kbn-version这个参数，否则获取到的Response中的Header Status Code就是400，而不是200。这个问题我是使用阿里云的健康检查发现的，因为阿里云要求所有服务的Response Status Code必须是200才认为该服务是健康状态。

阿里云的健康检查使用的命令如下，如果返回非2XX、3XX状态，定义为健康检查异常。

```
echo -e "HEAD /test.html HTTP/1.0\r\n\r\n" |nc -t LAN_IP 80
```

```
$ curl -I 'http://localhost:5601' -H 'Host: localhost:5601'HTTP/1.1 400 Bad Requestkbn-name: kibanakbn-version: 4.5.4content-type: application/json; charset=utf-8cache-control: no-cacheDate: Thu, 01 Dec 2016 10:54:47 GMTConnection: keep-alive$ curl -I 'http://localhost:5601' -H 'Host: localhost:5601' -H 'kbn-version:4.5.4'HTTP/1.1 200 OKkbn-name: kibanakbn-version: 4.5.4cache-control: no-cacheDate: Thu, 01 Dec 2016 11:01:59 GMTConnection: keep-alive
```

还有就是阿里云的健康检查异常的情况下，我们检查elasticsearch的运行状态会发现会有很多10和100段的IP地址和我们ES服务器有通信，这是由于负载均衡系统进行健康检查引起的。由于阿里云的健康检查异常，所以就会出现很多tcp连接是TIME_WAIT状态。但是最后我们没有办法在header中添加'kbn-version:4.5.4'参数，所以只好关闭阿里云的健康检查了。

参考文章：

- https://github.com/elastic/kibana/issues/3580
- https://github.com/elastic/kibana/issues/1370
- https://discuss.elastic.co/t/curl-request-returns-missing-kbn-version-header/50591
- https://helpcdn.aliyun.com/document_detail/27674.html
- https://helpcdn.aliyun.com/document_detail/27660.html