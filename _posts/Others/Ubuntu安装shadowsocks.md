## Ubuntu安装Shadowsocks

### 安装Shadowsocks和Supervisor
```
sudo apt-get update
sudo apt-get install python-pip
sudo pip install --upgrade pip
sudo pip install shadowsocks
sudo apt-get install supervisor
```

### 添加Shadowsocks配置文件
```
$ vi /etc/shadowsocks.json

# 具体配置内容
{
    "server":"0.0.0.0",
    "server_port":443,
    "local_address":"127.0.0.1",
    "local_port":1080,
    "password":"123456",
    "timeout":500,
    "method":"aes-256-cfb",
    "fast_open":false
}

# 配置内容描述
server : 服务端监听的地址，服务端可填写 0.0.0.0
server_port : 服务端的端口
local_address : 本地端监听的地址
local_port : 本地端的端口
password : 用于加密的密码
timeout : 超时时间，单位秒
method : 默认"aes-256-cfb"，建议chacha20或者rc4-md5，因为这两个速度快
fast_open : 是否使用 TCP_FASTOPEN, true / false（后面优化部分会打开系统的 TCP_FASTOPEN，所以这里填 true，否则填 false)
```

### 启动或停止Shadowsocks
```
$ ssserver -c /etc/shadowsocks.json -d start
$ ssserver -c /etc/shadowsocks.json -d stop
```

### 检查Shadowsocks日志
```
$ tailf /var/log/shadowsocks.log
```

### 将shadowsocks加入开机启动
```
$ sudo vi /etc/rc.local
/usr/bin/python /usr/local/bin/ssserver -c /etc/shadowsocks.json -d start
```

### 添加supervisord配置文件
```
$ sudo vi /etc/supervisord.conf

# 添加配置
[program:shadowsocks]
command=ssserver -c /etc/shadowsocks.json
autostart=true
autorestart=true
user=root
stderr_logfile=/var/log/supervisor/supervisor.log
stopsignal=INT

[supervisord]
```

### 启动 supervisord
```
$ sudo supervisord -c /etc/supervisord.conf
```

### 检查supervisor日志
```
$ tailf /var/log/supervisor/supervisor.log
```

### 把 supervisor 加入开机启动进程
```
$ vi /etc/rc.local

# 添加配置
supervisord -c /etc/supervisord.conf
```
