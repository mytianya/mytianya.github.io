
---
title: Springboot2.x基础教程：@Scheduled开启定时任务及源码分析
date: 2023-09-24 11:54:08
tags: springboot
categories: 
- java
---
> 在项目开发过程中，我们经常需要执行具有周期性的任务，通过定时任务可以很好的帮助我们实现。
> 常见的定时任务有JDK自带的TimeTask,ScheduledExecutorService，第三方的quartz框架，elastic-job等。
> 今天要给大家介绍的是SpringBoot自带的定时任务框架，通过@Scheduled注解就能很方便的开启一个定时任务。
> Spring Schedule框架功能完善，简单易用。对于中小型项目需求，Spring Schedule是完全可以胜任的。
## TimeTask,Spring-Schedule,Quartz对比
![task-vs](/img/task-vs.png)
## SpringBoot配置定时任务
SpringBoot开启一个定时任务非常简单,在方法上加上@Scheduled注解跟配合@EnableScheduling注解开启就能够开启一个定时任务。
```java
@Component
@EnableScheduling
@Slf4j
public class ScheduledTask {
    @Scheduled(cron = "*/1 * * * * ?")
    public void cronTask1(){
        log.info("CronTask-当方法的执行时间超过任务调度频率时，调度器会在下个周期执行");
        try {
            Thread.sleep(5100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    @Scheduled(fixedRate = 1000)
    public void cronTask2(){
        log.info("fixedRate--固定频率执行，当前执行任务如果超时，调度器会在当前方法执行完成后立即执行");
    }
    @Scheduled(fixedDelay = 1000)
    public void cronTask3(){
        log.info("fixedDelay---固定间隔执行，从上一次执行任务的结束时间开始算-------");
    }
}

```
## 配置定时任务线程池
在实际项目中，我们一个系统可能会定义多个定时任务。那么多个定时任务之间是可以相互独立且可以并行执行的。Spring Sheduld默认会创建一个单线程池执行定时任务。
这样对于我们的多任务调度可能会是致命的，当多个任务并发（或需要在同一时间）执行时，任务调度器就会出现时间漂移，任务执行时间将不确定。所以我们通常给定时任务自定义配置一个线程池。
```java
@EnableScheduling
@Configuration
public class ScheduledTaskConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar scheduledTaskRegistrar) {
        scheduledTaskRegistrar.setScheduler(taskExecutor());
    }

    public ThreadPoolTaskScheduler  taskExecutor() {
        ThreadPoolTaskScheduler scheduler=new ThreadPoolTaskScheduler();
        // 设置核心线程数
        scheduler.setPoolSize(8);
        // 设置默认线程名称
        scheduler.setThreadNamePrefix("CodehomeScheduledTask-");
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.initialize();
        return scheduler;
    }
}
```
## 定时任务源码解析
SpringBoot加上@EnableScheduling注解配合@Scheduled就能生成定时任务，背后的源码带大家分析一遍。
### 扫描定时任务
@EnableScheduling注解引入了SchedulingConfiguration类
```
@Import({SchedulingConfiguration.class})
public @interface EnableScheduling {
}

```
SchedulingConfiguration又引入了ScheduledAnnotationBeanPostProcessor类，核心的流程在该类中。
```
@Configuration
public class SchedulingConfiguration {
    @Bean(
        name = {"org.springframework.context.annotation.internalScheduledAnnotationProcessor"}
    )
    public ScheduledAnnotationBeanPostProcessor scheduledAnnotationProcessor() {
        return new ScheduledAnnotationBeanPostProcessor();
    }
}
```
![ScheduledAnnotationBeanPostProcessor](/img/ScheduledAnnotationBeanPostProcessor.png)

该类继承了BeanPostProcessor接口是Spring IOC容器给我们提供的一个扩展接口。

```
public interface BeanPostProcessor {
    //bean初始化方法调用前被调用
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    //bean初始化方法调用后被调用
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```
因此在该类初始化bean后执行postProcessAfterInitialization方法,扫描容器中Bean带有@scheduled注解的方法，并通过processScheduled解析。
![processScheduled](/img/processScheduled.png)
processScheduled会解析前面提到的cron、fixedDelay、fixedRate等属性，创建不同的类型的Task，加入scheduledTasks变量中。
![task-class](/img/task-class.png)

### 触发定时任务
1. 在容器发出ContextRefreshedEvent事件时，事件监听方法会调用finishRegistration()方法
2. finishRegistration()会调用registrar.afterPropertiesSet()方法
3. afterPropertiesSet()方法调用scheduleTasks();
![task-class](/img/task-invoke.png)
4. scheduleTasks方法依次调用各个定时任务，这里也验证了不配置线程池，默认是单线程执行。
![task-class](/img/task-invoke2.png)
**以上就是对springboot定时任务简单的分析，看完后发现还是很简单的。我们学习springboot源码学习的是优秀的设计方法，通过理解、模仿、从而创新，自己的编程功底才能进一步提升。
千里之行，始于足下。这里是SpringBoot教程系列第十三篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**

