
---
title: 贡献一个springboot项目linux shell启动脚本
date: 2023-09-25 02:00:18
tags: springboot
categories: 
- java
---
>springboot打好的包放在/usr/local/app目录下，如App.jar改名为mv App.jar App
>springboot配置外提为App.yml也放在当前目录下,日志生成为App.log
```shell
#!/bin/bash 
#这里可替换为你自己的执行程序,其他代码无需更改 
APP_NAME=$1
JAR_NAME=/usr/local/app/${APP_NAME} 
JVM="-server -Xms2048m -Xmx2048m -XX:PermSize=1024M -XX:MaxNewSize=512m -XX:MaxPermSize=2048m -Djava.awt.headless=true -XX:+CMSClassUnloadingEnabled -XX:+CMSPermGenSweepingEnabled"
APPFILE_PATH="-Dspring.config.location=/usr/local/app/${APP_NAME}.yml 
#使用说明,用来提示输入参数 
usage() { 
	echo "Usage: sh 执行脚本.sh app_name [start|stop|restart|status]" 
	exit 1 
} 
#检查程序是否在运行 
is_exist(){ 
pid=`ps -ef|grep java |grep $APP_NAME|grep -v grep|awk '{print $2}' ` 
echo $pid
#如果不存在返回1,存在返回0 
if [ -z "${pid}" ]; then 
	return 1 
	else 
	return 0 
	fi 
} 
#启动方法 
start(){ 
is_exist 
if [ $? -eq "0" ]; then 
	echo "${APP_NAME} is already running. pid=${pid} ." 
	else 
	nohup java $JVM -jar $APPFILE_PATH $JAR_NAME > ${APP_NAME}.log 2>&1 &
	fi
} 
#停止方法 
stop(){ 
is_exist 
if [ $? -eq "0" ]; then 
	kill -9 $pid 
	else 
	echo "${APP_NAME} is not running" 
	fi 
} 
#输出运行状态 
status(){ 
is_exist 
if [ $? -eq "0" ]; then 
	echo "${APP_NAME} is running. Pid is ${pid}" 
	else 
	echo "${APP_NAME} is NOT running." 
	fi 
} 
#重启 
restart(){ 
	stop 
	start 
} 
#根据输入参数,选择执行对应方法,不输入则执行使用说明 
case "$2" in 
	"start") 
	start 
;; 
"stop") 
	stop 
;; 
"status") 
	status 
;; 
"restart") 
	restart 
;; 
*) 
usage 
;; 
esac
```

## 开机自启，注册成Systemd系统服务

使用root权限在linux系统的/lib/systemd/system下编辑需要启动服务如app.service

```shell
[Unit]
Description=eureka service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/java -server -Xms2048m -Xmx2048m -XX:PermSize=1024M -XX:MaxNewSize=512m -XX:MaxPermSize=2048m -Djava.awt.headless=true -XX:+CMSClassUnloadingEnabled -XX:+CMSPermGenSweepingEnabled  -Dcom.sun.management.jmxremote -Djava.rmi.server.hostname=<服务器IP> -jar -Dspring.config.location=<外置的配置文件地址>  <Jar包地址>
PrivateTmp=true
User=<启动服务的用户>
Restart=on-failure
RestartSec=10
[Install]
WantedBy=multi-user.target
```

接下来要做的就是启动服务，首先重新加载systemd

```shell
systemctl daemon-reload
```

启动该服务,并查看app.service状态

```shell
systemctl start app.service
systemctl status app.service
```

配置成开机自启

```shell
systemctl enable app.service
systemctl stop app.service #关闭
systemctl disable app.service #移除开启自启
```

