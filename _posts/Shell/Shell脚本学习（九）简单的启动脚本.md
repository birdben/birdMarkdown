---
title: "Shell脚本学习（九）简单的启动脚本"
date: 2016-11-24 16:53:09
tags: [Shell]
categories: [Shell]
---

最近部署了日志收集服务，发现好多服务都要自己写启动脚本，刚刚开始学习Shell只能一边学一边写，就拿简单的Kibana服务来写个启动脚本吧。

一般服务的启动脚本，都需要实现下面的几点：

1. 需要设置服务的内存使用大小（特别是Java服务）
2. 后台启动（一些服务支持后台启动，例如：ElasticSearch）
3. 限制启动用户（最好限定启动服务的用户，防止其他用户误操作）
4. 设置PID文件目录（一些服务可以在配置文件中设置PID文件目录，例如：Kibana）
5. 设置服务的logs目录（一些服务可以在配置文件中设置logs目录，例如：ElasticSearch）

（下面的Kibana启动脚本可能不是最新的，因为目前一直在更新）

```
#!/bin/bash
# Kibana Issue : https://github.com/elastic/kibana/issues/5170
export NODE_OPTIONS="${NODE_OPTIONS:=--max-old-space-size=200}"

RETVAL=0
SYS_USER="elk"
SYS_USER_ID="501"

KB_NAME="kibana"
KB_DESC="Kibana 4.5.4"
KB_PID_FOLDER="/var/run/kibana"
KB_PID_FILE="${KB_PID_FOLDER}/${KB_NAME}.pid"
KB_LOG_FOLDER="${KB_HOME}/logs"

# 限制启动用户
if [ `id -u` -ne "${SYS_USER_ID}" ]; then
    echo "You need ${SYS_USER} privileges to run this script"
    exit 1
fi

# 启动服务
start() {
    status
    RETVAL=$?
    if [ $RETVAL -eq 0 ]; then
        echo "${KB_NAME} is already running"
        exit $RETVAL
    fi

    echo "Starting ${KB_DESC} : "
    if [ ! -d "${KB_PID_FOLDER}" ] ; then
        mkdir -p ${KB_PID_FOLDER}
    fi
    if [ ! -d "${KB_LOG_FOLDER}" ] ; then
        mkdir -p ${KB_LOG_FOLDER}
    fi

    cd ${KB_HOME}/bin
    nohup kibana 1>${KB_LOG_FOLDER}/${KB_NAME}.out 2>${KB_LOG_FOLDER}/${KB_NAME}.err &
    sleep 3
    PID=`cat "${KB_PID_FILE}"`
    echo "${KB_NAME} started. PID:$PID" 
    return 0 
}

# 停止服务
stop() {
    echo "Stopping ${KB_DESC} : "
    if status ; then
    PID=`cat "${KB_PID_FILE}"`
    echo "Killing ${KB_NAME} (PID:$PID) with SIGTERM"
    kill -TERM $PID >/dev/null 2>&1
    sleep 1
    if status && sleep 1 ; then
        echo "${KB_NAME} stop failed; still running. Will force kill ${KB_NAME}"
        kill -KILL $PID >/dev/null 2>&1
        sleep 1
        if status && sleep 1 ; then
        	echo "${KB_NAME} stop failed; still running." 
        else
            echo "${KB_NAME} stopped."
            rm -f ${KB_PID_FILE}
        fi
    else
        echo "${KB_NAME} stopped."
        rm -f ${KB_PID_FILE}
    fi
  fi
}

# 检查服务状态
status() {
    if [ -f "${KB_PID_FILE}" ] ; then
        PID=`cat "${KB_PID_FILE}"`
        if kill -0 $PID > /dev/null 2> /dev/null ; then
            return 0
        else
        	# program is dead but pid file exists
            return 2
        fi
    else
    	# program is not running
        return 3
    fi
}

case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    status)
        status
        RETVAL=$?
        if [ $RETVAL -eq 0 ] ; then
            PID=`cat "${KB_PID_FILE}"`
            echo "${KB_NAME} is running. PID:$PID"
        else
            echo "${KB_NAME} is not running"
        fi
        exit $RETVAL
        ;;
    restart)
        stop && start
        ;;
    *)
        # Invalid Arguments, print the following message.
        echo "Usage: $0 {start|stop|status|restart}" >&2
        exit 2
        ;;
esac
```