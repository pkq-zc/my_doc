# springboot管理脚本

## 简介

在linux中部署springboot应用经常需要执行各种命令,但是频繁的启动和停止应用比较麻烦.于是就准备自己写一个简单的shell脚本来管理springboot应用.  
我自己也不是太懂如何写shell脚本,但是通过查看shell脚本教程写一个简单的脚本还是比较容易的,主要麻烦的地方是不会调试只能一个个试.下面就是我自己  
写的一个简单的脚本,我自己使用过后没有什么太大问题.如果有问题,还请大家在留言处指出,我会加以修改.

## 脚本内容

```shell
#!/bin/bash
# 使用时需要使用'chmod u+x 脚本名称'添加执行权限.例如:chmod u+x boot.sh
#获取脚本名称
SCRIPT=$0
#获取进程名称,必须为完整程序名,否则可能会误操作其他进程
APP_NAME=$1
#获取操作符
OPERATOR=$2

usage() {
    echo "Usage: sh $SCRIPT [app_name] [start|stop|restart|status]"
    exit 1
}

#判断是否输入了两个参数
if [ $# != 2 ]; then
    usage
fi

is_exist(){
  # ps -ef 查看进程
  # | 代表管道,把上一个命令的内容输出到管道
  # grep 过滤字符,例如 grep tomcat 代表过滤内容中的 tomcat 字符串. -v 表示显示不包含指定的字符串
  # awk 用来处理文本 $2 代表第二栏内容

  # 获取进程的pid
  pid=`ps -ef|grep $APP_NAME|grep -v grep|grep -v $SCRIPT|awk '{print $2}'`
  if [ -z "${pid}" ]; then
  return 1
  else
    return 0
  fi
}

# 启动应用
start(){
  is_exist
  if [ $? -eq "0" ]; then
    echo "${APP_NAME} is already running. pid=${pid} ."
  else
    # 执行命令启动java应用.该命令可以根据自己的需求修改
    nohup java -jar $APP_NAME > "${APP_NAME}.log" 2>&1 &
  fi
}

# 停止应用
stop(){
  is_exist
  if [ $? -eq "0" ]; then
    kill -9 $pid
  else
    echo "${APP_NAME} is not running"
  fi
}

# 查看当前应用的状态
status(){
  is_exist
  if [ $? -eq "0" ]; then
    echo "${APP_NAME} is running. Pid is ${pid}"
  else
    echo "${APP_NAME} is NOT running."
  fi
}

# 重新启动
restart(){
  stop
  start
}

case "$OPERATOR" in
  "start")
    start ;;
  "stop")
    stop ;;
  "status")
    status ;;
  "restart")
    restart ;;
  *)
    usage ;;

```