
---
title: Springboot2.x基础教程：@Async开启异步任务
date: 2023-09-24 10:54:08
tags: springboot
categories: 
- java
---
>在开发项目中通常我们有场景需要开启异步任务。比如在用户注册成功时，需要发放一些优惠券。此时为了不让这些额外的操作影响用户的注册流程，我们通常开启一个线程异步去执行发放优惠券逻辑。
>通常我们需要自己定义一个线程池，开启一个线程任务。在Springboot中对其进行了简化处理，自动配置一个 org.springframework.core.task.TaskExecutor类型任务线程池，当我们开启@EnableAsync注解时，在需要执行异步任务的方法添加@Async注解时，该方法自动会开启一个线程去执行。
>在讲解Springboot如何配置开启异步任务时，我们先简单说明下Java的线程池有关的基本知识。
## 线程池基本知识
### 使用线程池场景
通常高并发，任务执行时间短的业务创建线程的时间是T1,执行任务的时间是T2，销毁线程的时间是T3,当T1+T2的时间远远大于T2则可以采用线程池，线程池的作用核心在于复用线程，减少线程的创建与销毁的时间开销。
方便的统计任务的执行信息，例如完成的任务数量。

### Executor线程池的顶级接口
Jdk1.5版本以后，便提供线程池供开发人员方便的创建自己的多线程任务。java.util.concurrent.Executor接口为线程池的顶级接口。
该接口execute接收一个Runnable线程实例用来执行一个任务。
```
public interface Executor {
    void execute(Runnable command);
}
```
### ExecutorService接口详解
```
public interface ExecutorService extends Executor {

    /**关闭线程池不会接收新的任务，内部正在跑的任务和队列里等待的任务，会执行完
     */
    void shutdown();

    /**
     * 关闭线程池，不会接收新的任务
	 * 尝试将正在跑的任务interrupt中断
	 * 忽略队列里等待的任务
	 * 返回未执行的任务列表
     */
    List<Runnable> shutdownNow();

    /**
     * 返回线程池是否被关闭
     */
    boolean isShutdown();

    /**
     *线程池的任务线程是否被中断
     */
    boolean isTerminated();

    /**
	*接收timeout和TimeUnit两个参数，用于设定超时时间及单位。
	* 当等待超过设定时间时，会监测ExecutorService是否已经关闭，若关闭则返回true，否则返回false。和shutdown方法组合使用。
     */
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
	 * 提交任务会获取线程的执行结果
     */
    <T> Future<T> submit(Callable<T> task);

    /**
	* 提交任务会获取线程的T类型的执行结果
     */
    <T> Future<T> submit(Runnable task, T result);

    /**
     * 提交任务
     */
    Future<?> submit(Runnable task);

    /**
     * 批量提交返回任务执行结果
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    /**
     * 批量提交返回任务执行结果
	 *增加了超时时间控制，这里的超时时间是针对的所有tasks，而不是单个task的超时时间。
	 *如果超时，会取消没有执行完的所有任务，并抛出超时异常
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
     * 取得第一个方法的返回值,当第一个任务结束后，会调用interrupt方法中断其它任务
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    /**
     * 取得第一个方法的返回值,当第一个任务结束后，会调用interrupt方法中断其它任务
	 * 任务执行超时抛出异常
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

### ThreadPoolExecutor参数解析
ThreadPoolExecutor 是用来处理异步任务的一个接口，可以将其理解成为一个线程池和一个任务队列，提交到 ExecutorService 对象的任务会被放入任务队或者直接被线程池中的线程执行并且ThreadPoolExecutor 支持通过调整构造参数来配置不同的任务处理策略。
下面贴出ThreadPoolExecutor线程池构造函数代码:
```
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
ThreadPoolExecutor每个入参含义：
![thread-pool-params](/img/thread-pool-params.png)
参数之间的关系：

1. 从线程池中获取可用线程执行任务，如果没有可用线程则使用ThreadFactory创建新的线程，直到线程数达到corePoolSize限制
2. 线程池线程数达到corePoolSize以后，新的任务将被放入队列，直到队列不能再容纳更多的任务
3. 当队列不能再容纳更多的任务以后，会创建新的线程，直到线程数达到maxinumPoolSize限制
4. 线程数达到maxinumPoolSize限制以后新任务会被拒绝执行，调用 RejectedExecutionHandler 进行处理
[![thread-pool-work](/img/thread-pool-work.png)
### RejectedExecutionHandler拒绝策略 
1、AbortPolicy：直接抛出异常，默认策略；
2、CallerRunsPolicy：用调用者所在的线程来执行任务；
3、DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；
4、DiscardPolicy：直接丢弃任务；

### ThreadPoolExecutor使用要点
1. 合理配置线程池的大小：如果是CPU密集的任务，线程池数量设置为CPU核心数+1即可，如果是IO密集任务，则CPU空闲较多，线程池数量设置为CPU核心数*2
2. 任务队列：任务队列一般使用LinkedBlockingQueue指定大小，配合拒绝策略使用，默认是Integer.Max_VALUE容易内存溢出。
3. 任务异常： 线程池中任务注意上加上try{}catch{}捕获异常，方便排查问题

## SpringBoot线程池
org.springframework.core.task.TaskExecutor为Spring异步线程池的接口类，其实质是java.util.concurrent.Executor。
![executor](/img/executor.png)
![ThreadPoolTaskExecutor](/img/ThreadPoolTaskExecutor.png)
springboot2.3.x默认使用的线程池就是ThreadPoolTaskExecutor，可以通过TaskExecutionProperties进行简单的参数调整

## SpringBoot定义自己的异步线程池
```java
@EnableAsync
@Component
@Slf4j
public class AsyncConfig implements AsyncConfigurer {
    public TaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        // 设置核心线程数
        executor.setCorePoolSize(8);
        // 设置最大线程数
        executor.setMaxPoolSize(16);
        // 设置队列容量
        executor.setQueueCapacity(50);
        // 设置线程活跃时间（秒）
        executor.setKeepAliveSeconds(60);
        //设置线程池中任务的等待时间，如果超过这个时候还没有销毁就强制销毁，以确保应用最后能够被关闭，而不是阻塞住
        executor.setAwaitTerminationSeconds(60);
        // 设置默认线程名称
        executor.setThreadNamePrefix("CodehomeAsyncTask-");
        // 设置拒绝策略
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // 等待所有任务结束后再关闭线程池
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.initialize();
        return executor;
    }
    public Executor getAsyncExecutor() {
        return taskExecutor();
    }
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new MyAsyncExceptionHandler();
    }

    /**
     * 自定义异常处理类
     */
    class MyAsyncExceptionHandler implements AsyncUncaughtExceptionHandler {

        @Override
        public void handleUncaughtException(Throwable throwable, Method method, Object... objects) {
            log.info("Exception message - " + throwable.getMessage());
            log.info("Method name - " + method.getName());
            for (Object param : objects) {
                log.info("Parameter value - " + param);
            }
        }
    }
}
```
### 两种使用方法
```java

@Component
@Slf4j
public class UserServiceSyncTask {
	//不带返回值
    @Async
    public void sendEmail(){
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.info(Thread.currentThread().getName());
    }
	//带返回值
    @Async
    public Future<String> echo(String msg){
        try {
            Thread.sleep(5000);
            return new AsyncResult<String>(Thread.currentThread().getName()+"hello world !!!!");
        } catch (InterruptedException e) {
            //
        }
        return null;
    }
}
```
**千里之行，始于足下。这里是SpringBoot教程系列第十篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**

