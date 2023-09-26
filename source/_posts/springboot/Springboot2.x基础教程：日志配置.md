
---
title: Springboot2.x基础教程：日志配置
date: 2023-09-25 02:00:10
tags: springboot
categories: 
- java
---
>项目的开发过程中，开发人员对于日志一定不会陌生。日志能够记录程序运行的轨迹，输出软件运行中的关键信息，辅助我们排查与定位问题，优化程序运行性能，监控程序运行状态，不可不谓重要。
>SpringBoot项目的spring-boot-starter默认引用spring-boot-starter-logging,其中底层采用logback日志框架，默认零配置即可使用日志记录功能。
>在讲解springboot日志配置之前先简单谈谈JAVA日志有关的基础知识。

## 日志记录的时机
- 记录程序初始化有关启动的参数，判断程序的运行状态
- 代码抛出异常，记录程序异常状态
- 业务流程与预期结果不符，记录业务异常状态
- 系统核心业务，核心权限操作。比如登录、付款等操作记录，通常还会入库分析。

## Java日志框架
对于日志框架，我们通常会看到log4j、logback等名词，也会遇到自己项目与第三方jar的日志库冲突问题。
初次接触这些，可能有种云雾缭绕不知所云的感觉，下面简单介绍下Java日志框架的关系。更具体的历史缘由，细节部分。网上有几篇文章介绍的很好，给大家附上自行阅读理解:
1. [知乎上面有篇文章：java日志框架解析](https://zhuanlan.zhihu.com/p/24272450)
2. [博客上面有篇文章：Java常用日志框架介绍](https://www.cnblogs.com/chenhongliang/p/5312517.html)

看完以上文章简单的总结Java日志框架分为3类：
- Java日志框架的具体的实现：log4j1.x、JUL(Java Util Log)、Logback、log4j2-core
- Java日志框架的门面对象，只提供接口不提供具体实现：JCL(Commons Logging)、SLF4J(The Simple Logging Facade for Java)、log4j2-api
- Java日志框架之间的适配器，为了让不同日志框架互相转换:jcl-over-slf4j、slf4j-jcl、log4j-over-slf4j、slf4j-log4j12等等
![](https://pan.codehome.vip/images/I9nRhzurQmLXtoa.jpg)

关于日志框架的最佳实践（来源参考链接，这里只是摘出）：
1. 总是使用Log Facade，而不是具体Log Implementation
2. 只添加一个 Log Implementation依赖
3. 具体的日志实现依赖应该设置为optional和使用runtime scope
4. 如果有必要, 排除依赖的第三方库中的Log Impementation依赖
5. 避免输出不必要的日志，跟不必要的日志字段如行号影响程序性能

## SpringBoot日志配置
### 日志依赖
springboot默认使用SLF4J+Logback的组合记录日志，查看依赖可知，不用我们额外引入。
![java-log](/img/java-log.png)

### springboot日志配置
```yml
logging:
  level:
    #包的日志级别
    org.springframework.web: DEBUG
  #自定义log信息
  config: classpath:config/logback-spring.xml
  pattern:
    #控制台的日志输出格式
    console: '%d{yyyy/MM/dd-HH:mm:ss} [%thread] %-5level %logger- %msg%n'
    #文件的日志输出格式
    file: '%d{yyyy/MM/dd-HH:mm} [%thread] %-5level %logger- %msg%n'
  file:
    #日志名称
    name: app.log
    #存储的路径
    path: /var/log/
    #存储的最大值
    max-size: 50MB
    #保存时间
    max-history: 7
```
![image-20200924220454446](https://pan.codehome.vip/images/u5i6kveV2nxtdRQ.png)


```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!--获取变量名中关于日志存储的路径与存储名称-->
    <springProperty scope="context" name="logPath" source="logging.file.path"/>
    <springProperty scope="context" name="logName" source="logging.file.name"/>
    <!--输出到控制台的appender-->
    <appender name="Console"
              class="ch.qos.logback.core.ConsoleAppender">
        <layout class="ch.qos.logback.classic.PatternLayout">
            <Pattern>
                %black(%d{ISO8601}) %highlight(%-5level) [%blue(%t)] %yellow(%C{1.}): %msg%n%throwable
            </Pattern>
        </layout>
    </appender>
    <!--输出到文件的appender-->
    <appender name="RollingFile"
              class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${logPath}/${logName}</file>
        <encoder
                class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <Pattern>%d %p %C{1.} [%t] %m%n</Pattern>
        </encoder>

        <rollingPolicy
                class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- rollover daily and when the file reaches 10 MegaBytes -->
            <fileNamePattern>${LOGS}/archived/spring-boot-logger-%d{yyyy-MM-dd}.%i.log
            </fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy
                    class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>10MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
    </appender>

    <!--开发环境基本级别为DEBUG-->
    <springProfile name="dev">
        <root level="DEBUG">
            <appender-ref ref="Console"/>
        </root>
    </springProfile>
    <!--生产环境输入到文件中-->
    <springProfile name="prod">
        <root level="INFO">
            <appender-ref ref="RollingFile"/>
        </root>
    </springProfile>
</configuration>
```

**千里之行，始于足下。这里是SpringBoot教程系列第八篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**