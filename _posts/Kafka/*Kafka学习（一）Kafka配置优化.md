---
title: "Kafka学习（一）Kafka配置优化"
date: 2016-12-15 11:16:39
tags: [Kafka]
categories: [MQ]
---


```
controlled.shutdown.enable					Enable controlled shutdown of the server	boolean	true		medium
controlled.shutdown.max.retries				Controlled shutdown can fail for multiple reasons. This determines the number of retries when such failure happens	int	3		medium
controlled.shutdown.retry.backoff.ms		Before each retry, the system needs time to recover from the state that caused the previous failure (Controller fail over, replica lag etc). This config determines the amount of time to wait before retrying.	long	5000		medium
```

参考文章：

- https://my.oschina.net/frankwu/blog/305010
- http://www.itdadao.com/articles/c15a32099p0.html
- https://community.hortonworks.com/articles/49789/kafka-best-practices.html
- http://www.cnblogs.com/yinchengzhe/p/5111635.html
- http://blog.csdn.net/guoyuqi0554/article/details/51253701
- https://my.oschina.net/infiniteSpace/blog/312890?p=1
- http://bbs.umeng.com/thread-12479-1-1.html
- http://www.cnblogs.com/xiaodf/p/6090722.html
- http://blog.csdn.net/huanggang028/article/details/47830529
