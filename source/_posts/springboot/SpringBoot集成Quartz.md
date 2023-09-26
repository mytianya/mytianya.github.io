
---
title: Springboot2.x基础教程：SpringBoot集成Quartz
date: 2023-09-25 02:00:38
tags: springboot
categories: 
- java
---
## 引入依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
```

## Quartz配置

```yaml
spring:
  quartz:
    properties:
      org:
        quartz:
          scheduler:
            instanceName: clusteredScheduler #调度器实例名称
            instanceId: AUTO #调度器实例编号自动生成
          jobStore:
            class: org.quartz.impl.jdbcjobstore.JobStoreTX #持久化方式配置
            driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate #持久化方式配置数据驱动，MySQL数据库
            tablePrefix: qrtz_ #quartz相关数据表前缀名
            isClustered: true #开启分布式部署
            clusterCheckinInterval: 10000 #分布式节点有效性检查时间间隔，单位：毫秒
            useProperties: false #配置是否使用
          threadPool:
            class: org.quartz.simpl.SimpleThreadPool #线程池实现类
            threadCount: 10 #执行最大并发线程数量
            threadPriority: 5 #线程优先级
            threadsInheritContextClassLoaderOfInitializingThread: true #配置是否启动自动加载数据库内的定时任务，默认true
    job-store-type: jdbc
    overwrite-existing-jobs: true
```

## Quartz SpringBoot使用

​	DataCheckJob继承 Job重写execute方法，这里**与springboot2.x集成,需要注入DataMapper Bean,直接使用@Autowired强制注入,或者使用构造函数注入即可，其他低版本的则会出现bean注入不了的情况。需要自己实现AdaptableJobFactory注入bean **

```java
@Slf4j
public class DataCheckJob implements QuartzJobBean {
  @Autowired
  DataMapper dataMapper;
  @Override
  public void execute(JobExecutionContext jobExecutionContext) throws JobExecutionException {
    dataMapper.check();
  }
}

```
配置DataCheckJob与任务的触发器。
```java
@Configuration
public class JobConfig {
  @Bean
  public JobDetail dataCheckJobDetail(){
    return JobBuilder.newJob(DataCheckJob.class).storeDurably().build();
  }
  @Bean
  public Trigger dataCheckTrigger(){
    return TriggerBuilder.newTrigger().forJob(dataCheckJobDetail()).withIdentity("DataCheckJob")
        .withSchedule(CronScheduleBuilder.cronSchedule("0 37 11 ? * *")).build();
  }
}
```





## Quartz表说明

### qrtz_blob_triggers

​	自定义的triggers使用blog类型进行存储，非自定义的triggers不会存放在此表中，Quartz提供的triggers包括：CronTrigger，CalendarIntervalTrigger，
DailyTimeIntervalTrigger以及SimpleTrigger，这几个trigger信息会保存在后面的几张表中；

### qrtz_cron_triggers

存放CronTrigger类型的触发器实例

![qrtz_cron_triggers](/img/qrtz_cron_triggers.png)

## qrtz_simple_triggers

存储SimpleTrigger

### qrtz_simprop_triggers

存储CalendarIntervalTrigger和DailyTimeIntervalTrigger两种类型的触发器

### qrtz_fired_triggers

​	存储已经触发的trigger相关信息，trigger随着时间的推移状态发生变化，直到最后trigger执行完成，从表中被删除。

相同的trigger和task，每触发一次都会创建一个实例；从刚被创建的ACQUIRED状态，到EXECUTING状态，最后执行完从数据库中删除。

![qrtz_fired_triggers](/img/qrtz_fired_triggers.png)

### qrtz_triggers

存储定义的trigger，和qrtz_fired_triggers存放的不一样，不管trigger触发了多少次都只有一条记录，TRIGGER_STATE用来标识当前trigger的状态

![qrtz_triggers](/img/qrtz_triggers.png)

### qrtz_job_details

存储jobDetails信息，相关信息在定义的时候指定。

![](/img/qrtz_job_details.png)

### qrtz_calendars

​	Quartz为我们提供了日历的功能，可以自己定义一个时间段，可以控制触发器在这个时间段内触发或者不触发；现在提供6种类型：AnnualCalendar，CronCalendar，DailyCalendar，HolidayCalendar，MonthlyCalendar，WeeklyCalendar；

### qrtz_paused_trigger_grps

存放暂停的触发器

### qrtz_scheduler_state

存储所有节点的scheduler，会定期检查scheduler是否失效，启动多个scheduler。

![](/img/qrtz_scheduler_state.png)

### qrtz_locks

Quartz提供的锁表，为多个节点调度提供分布式锁，实现分布式调度，默认有2个锁。

​	/img/
​	TRIGGER_ACCESS主要用在TRIGGER被调度的时候，保证只有一个节点去执行调度；

## Quartz表手动生成

实际在使用Quartz过程中数据库中的表通常因为权限不能够自动生成，通常配置initialize-schema为never。

```properties
spring.quartz.jdbc.initialize-schema=never|always
```

在quartz的jar的org/quartz/impl/jdbcjobstore包中有对应各种关系数据库的数据库生成脚本。有时候默认的quartz-sql对于最新的数据库，比如oracle19c不能很好兼容，需要我们自己修改对应SQL语句。

![quartz-table](/img/quartz-table.png)

