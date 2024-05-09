
---
title: 记一次破解Springboot Java项目授权过期问题
date: 2024-04-15 10:54:08
tags: 
- recaf
- jd-gui
categories:  
- 逆向工程
---

> 从网上下载的一个商业的MQ程序启动报错授权过期，简单破解了下绕过了授权认证。
使用到工具与命令，jd-gui、recaf、jdk自带的jar命令。
## 破解过程
1. 启动了下该MQ的startup.sh文件，启动日志报错。可以看出是com.primeton.ext.common.l72.Imprimatur.validate()验证失败了。
![](/img/springboot-license-outdate-log.png)
2. 使用从startup.sh文件中找到该MQ的jar包，使用jd-gui.exe打开了该jar包，可以查看目录知道jar包是springboot技术开发并打包的。找了在lib目录的一个jar下面。
![alt text](/img/jd-gui-view-jar-1.PNG)

3. 使用jdk自带的jar命令解压包。
```shell
jar -xvf mq.jar
```
4. 使用recaf在打开BOOT-INF/lib/验证jar包，破解方法很简单，修改jar包中的com.xxx.ext.common.l72.Imprimatur.validate()方法，直接使用recaf工具，选中validate方法右键使用编译汇编代码功能。在该校验方法中加入下面ICONST_1,IRETURN方法，相对于直接返回return true。
![](/img/recaf-recompile.PNG)
5. 修改后使用recaf导出功能，把改好的class文件重新生成jar包，然后替换原来的jar包。
6. 使用jar重新打包未springboot可执行程序。打包成功后可以再通过jd-gui.exe反编译查看下，是否都修改成功了。
```shell
jar -cfM0 mq.jar BOOT-INF/ META-INF/ org/
```

7. 重新执行启动MQ程序的startup.sh启动脚本，查看日志输出，发现已绕过授权成功启动。
