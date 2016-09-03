
---
title: "Docker实战（二十）Docker镜像的导入导出"
date: 2016-09-04 11:30:19
tags: [Docker命令, Dockerfile]
categories: [Docker]
---

## Docker镜像导入导出方式

最近公司需要做Docker私有化部署，需要将本地安装好的Docker容器部署到客户的环境，这里遇到了一些问题客户的服务器不能连接外网，无法在线做Docker镜像的构建，所以需要只能通过导入导出镜像的方式来做。下面是我总结的Docker镜像导入导出方式。

Docker提供了两种方式的导入导出：

- load/save方式导入导出镜像
	* docker save：来导出本地镜像库中指定的镜像存储成文件
	* docker load：来导入镜像存储文件到本地镜像库
- import/export方式导入导出容器
	* docker export：来导出一个容器快照到本地文件
	* docker import：来导入一个容器快照文件到本地镜像库
- 区别：容器快照文件将丢弃所有的历史记录和元数据信息（即仅保存容器当时的快照状态），而镜像存储文件将保存完整记录，体积也要大。此外，从容器快照文件导入时可以重新指定标签等元数据信息。我个人比较推荐用load/save方式，这样所有之前的镜像都会存在，只是比较占用空间。

### Docker镜像save/load方式

```
$ sudo docker imagesREPOSITORY                  TAG                 IMAGE ID            CREATED             VIRTUAL SIZEbirdben/zookeeper           v1                  20e4011b9286        2 minutes ago       1.658 GBubuntu                      latest              37b164bb431e        7 days ago          126.6 MBcentos                      7                   d83a55af4e75        5 weeks ago         196.7 MBcentos                      latest              d83a55af4e75        5 weeks ago         196.7 MBbirdben/jdk7                v1                  25c2f0e69206        8 months ago        583.4 MB

# 导出birdben/zookeeper:v1镜像到zookeeper_image.tar文件
$ sudo docker save birdben/zookeeper:v1 > zookeeper_image.tar

# 删除之前的birdben/zookeeper:v1镜像
$ sudo docker rmi "birdben/zookeeper:v1"

# 导入zookeeper_image.tar镜像文件
$ sudo docker load < zookeeper_image.tar

# 再次查看所有的镜像，可以看到birdben/zookeeper:v1又回来了
# 注意：这里import回来的ImageID也和原来是一样的
$ sudo docker imagesREPOSITORY                  TAG                 IMAGE ID            CREATED             VIRTUAL SIZEbirdben/zookeeper           v1                  20e4011b9286        6 minutes ago       1.658 GBubuntu                      latest              37b164bb431e        7 days ago          126.6 MBcentos                      7                   d83a55af4e75        5 weeks ago         196.7 MBcentos                      latest              d83a55af4e75        5 weeks ago         196.7 MBbirdben/jdk7                v1                  25c2f0e69206        8 months ago        583.4 MB

# 这时候在查看birdben/zookeeper:v1镜像的tree结构，发现之前所有的镜像历史都在
$ sudo docker images --tree
├─3690474eb5b4 Virtual Size: 0 B│ └─b200b2d33d98 Virtual Size: 196.7 MB│   └─3fbd5972aaac Virtual Size: 196.7 MB│     └─d83a55af4e75 Virtual Size: 196.7 MB Tags: centos:7, centos:latest│       └─1df8e9ff4de7 Virtual Size: 196.7 MB│         └─b37af9ce019a Virtual Size: 196.7 MB│           └─7858b8d134c6 Virtual Size: 403.3 MB│             └─c872974343d2 Virtual Size: 403.3 MB│               └─d4c0e59dc712 Virtual Size: 403.3 MB│                 └─30c3076be68f Virtual Size: 556.8 MB│                   └─0e66c066e1de Virtual Size: 571 MB│                     └─69f8ec0b7932 Virtual Size: 889.1 MB│                       └─7cfcd6d4c6e7 Virtual Size: 911.4 MB│                         └─c2bc26e11781 Virtual Size: 911.4 MB│                           └─31d728531f9a Virtual Size: 911.4 MB│                             └─6434457046ec Virtual Size: 911.4 MB│                               └─651290e3ddef Virtual Size: 911.4 MB│                                 └─d99d028fca92 Virtual Size: 911.6 MB│                                   └─5d4d89731a7d Virtual Size: 911.6 MB│                                     └─a530df3b220c Virtual Size: 925.6 MB│                                       └─39381e27bf53 Virtual Size: 1.232 GB│                                         └─cda80cbe8e7f Virtual Size: 1.276 GB│                                           └─287a8cf1090c Virtual Size: 1.289 GB│                                             └─d5672fcec9a4 Virtual Size: 1.289 GB│                                               └─e63cb61422e1 Virtual Size: 1.289 GB│                                                 └─aa8f6ecc78ca Virtual Size: 1.303 GB│                                                   └─b44f1969877f Virtual Size: 1.303 GB│                                                     └─d17184db904f Virtual Size: 1.303 GB│                                                       └─7df628e7fd36 Virtual Size: 1.303 GB│                                                         └─dfe01b409095 Virtual Size: 1.303 GB│                                                           └─238718f45aa6 Virtual Size: 1.303 GB│                                                             └─a678149d4c34 Virtual Size: 1.303 GB│                                                               └─d3f7fb8e3bc2 Virtual Size: 1.658 GB│                                                                 └─ff152402a43c Virtual Size: 1.658 GB│                                                                   └─fdf82aa49b89 Virtual Size: 1.658 GB│                                                                     └─0dbfccd66315 Virtual Size: 1.658 GB│                                                                       └─4cae49ef2ecb Virtual Size: 1.658 GB Tags: birdben/zookeeper:v1
```

### Docker镜像import/export方式

```
$ sudo docker images
REPOSITORY                  TAG                 IMAGE ID            CREATED             VIRTUAL SIZEbirdben/zookeeper           v1       	        20e4011b9286        11 seconds ago      1.658 GBubuntu                      latest              37b164bb431e        7 days ago          126.6 MBcentos                      7                   d83a55af4e75        5 weeks ago         196.7 MBcentos                      latest              d83a55af4e75        5 weeks ago         196.7 MBbirdben/jdk7                v1                  25c2f0e69206        8 months ago        583.4 MB

$ sudo docker ps -aCONTAINER ID        IMAGE                 COMMAND             CREATED             STATUS              PORTS                                            NAMESf99771de17b0        20e4011b9286:latest   "/bin/bash"         6 seconds ago       Up 5 seconds        0.0.0.0:3306->3306/tcp, 0.0.0.0:8080->8080/tcp   birdben/zookeeper:v1

# 导出容器ID为f99771de17b0的Docker容器
$ sudo docker export f99771de17b0 > container.tar.gz

# 删除之前的镜像birdben/zookeeper:v1
$ sudo docker rmi "birdben/zookeeper:v1"

# 导入容器文件container.tar.gz
$ cat container.tar.gz | sudo docker import - birdben/zookeeper:v1

# 注意：这里import回来的ImageID也和原来不一样了
$ sudo docker imagesREPOSITORY                  TAG                 IMAGE ID            CREATED             VIRTUAL SIZEbirdben/zookeeper           v1                  e80c1046dc12        9 minutes ago       1.119 GBubuntu                      latest              37b164bb431e        7 days ago          126.6 MBcentos                      7                   d83a55af4e75        5 weeks ago         196.7 MBcentos                      latest              d83a55af4e75        5 weeks ago         196.7 MBbirdben/jdk7                v1                  25c2f0e69206        8 months ago        583.4 MB

# 这时候在查看birdben/zookeeper:v1镜像的tree结构，发现只有最有的镜像，没有以前的历史镜像
$ sudo docker images --treeWarning: '--tree' is deprecated, it will be removed soon. See usage.├─e80c1046dc12 Virtual Size: 1.119 GB Tags: birdben/zookeeper:v1├─f1b49dd0c243 Virtual Size: 126.6 MB│ └─008ecf8686ec Virtual Size: 126.6 MB│   └─fd74137ff5ae Virtual Size: 126.6 MB│     └─35371c8124e2 Virtual Size: 126.6 MB│       └─99dc4d8f603d Virtual Size: 126.6 MB│         └─37b164bb431e Virtual Size: 126.6 MB Tags: ubuntu:latest
```

参考文章：

- https://segmentfault.com/a/1190000000586840
- http://www.sxt.cn/u/756/blog/5339