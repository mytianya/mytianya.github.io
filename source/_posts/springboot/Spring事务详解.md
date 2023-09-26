
---
title: Spring事务详解
date: 2023-09-25 02:00:14
tags: springboot
categories: 
- java
---
> 事务是逻辑上的一组操作，要么都执行，要么都不执行。

## 事务特性(ACID)

- Atomicity（原子性）：一个事务（transaction）中的所有操作，或者全部完成，或者全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被[回滚](https://zh.wikipedia.org/wiki/回滚_(数据管理))（Rollback）到事务开始前的状态，就像这个事务从来没有执行过一样。即，事务不可分割、不可约简。[[1\]](https://zh.wikipedia.org/wiki/ACID#cite_note-acid-1)
- Consistency（一致性）：在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设[约束](https://zh.wikipedia.org/wiki/数据完整性)、[触发器](https://zh.wikipedia.org/wiki/触发器_(数据库))、[级联回滚](https://zh.wikipedia.org/wiki/级联回滚)等。[[1\]](https://zh.wikipedia.org/wiki/ACID#cite_note-acid-1)
- Isolation（隔离性）：数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致。事务隔离分为不同级别，包括未提交读（Read uncommitted）、提交读（read committed）、可重复读（repeatable read）和串行化（Serializable）。[[1\]](https://zh.wikipedia.org/wiki/ACID#cite_note-acid-1)
- Durability（持久性）：事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。

## Spring事务管理接口

​	**Spring并不直接管理事务，而是提供了多种事务管理器** ，他们将事务管理的职责委托给Hibernate或者JTA等持久化机制所提供的相关平台框架的事务来实现。 Spring事务管理器的接口是： **org.springframework.transaction.PlatformTransactionManager** ，通过这个接口，Spring为各个平台如JDBC、Hibernate等都提供了对应的事务管理器，但是具体的实现就是各个平台自己的事情了。

![PlatformTransactionManager](/img/PlatformTransactionManager.png)

### PlatformTransactionManager核心方法

```java
public interface PlatformTransactionManager extends TransactionManager {
	//根据指定的传播行为，返回当前活动的事务或创建一个新事务
	TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
			throws TransactionException;
	//使用事务目前的状态提交事务
	void commit(TransactionStatus status) throws TransactionException;
	//对执行的事务进行回滚
	void rollback(TransactionStatus status) throws TransactionException;
}

```

### TransactionDefinition事务定义

- 事务的传播行为
- 事务隔离级别
- 事务名称
- 事务超时时间
- 是否为[只读事务](https://blog.csdn.net/andyzhaojianhui/article/details/51984157)

```java
public interface TransactionDefinition {
    // 返回事务的传播行为
    int getPropagationBehavior(); 
    // 返回事务的隔离级别，事务管理器根据它来控制另外一个事务可以看到本事务内的哪些数据
    int getIsolationLevel(); 
    // 返回事务必须在多少秒内完成
    //返回事务的名字
    String getName()；
    int getTimeout();  
    // 返回是否优化为只读事务。
    boolean isReadOnly();
} 

```

### 事务的传播行为

简单的来说，多个事务方法相互调用时,事务如何在这些方法间传播。

> 举个栗子，方法A是一个事务的方法，方法A执行过程中调用了方法B，那么方法B有无事务以及方法B对事务的要求不同都会对方法A的事务具体执行造成影响，同时方法A的事务对方法B的事务执行也有影响，这种影响具体是什么就由两个方法所定义的事务传播类型所决定。

##### 支持外层事务情况

- Required: 当前外层存在事务，加入当前事务。没有自己创建新一个事务。
- Supports: 当前外层存在事务，加入当前事务。没有自己以非事务方式运行。
- Mandatory: 当前外层存在事务，加入当前事务。没有当前事务，抛出异常。

#### 不支持外层事务情况

- Required_NEW: 创建一个新事务，当前外层存在事务，则挂起当前事务。
- Not_Supported: 以非事务方式运行。当前外层存在事务，则挂起当前事务。
- Never: 以非事务方式运行。当前外层存在事务，则抛出异常。

#### 其他情况
- Nested: 当前外层存在事务，则创建一个嵌套事务作为当前事务的嵌套事务来运行。没有等于Required。

#### 事务的隔离级别

定义了一个事务可能受其他并发事务影响的程度。

**TransactionDefinition.ISOLATION_DEFAULT:**	使用后端数据库默认的隔离级别，Mysql 默认采用的 REPEATABLE_READ隔离级别 Oracle 默认采用的 READ_COMMITTED隔离级别.

**TransactionDefinition.ISOLATION_READ_UNCOMMITTED:** 最低的隔离级别，允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**

**TransactionDefinition.ISOLATION_READ_COMMITTED:** 	允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**

**TransactionDefinition.ISOLATION_REPEATABLE_READ:** 	对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生。**

**TransactionDefinition.ISOLATION_SERIALIZABLE:** 	最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

## Spring事务集成

### 声明式事务

通过在方法或类或接口上添加注解进行包装，无侵入地实现事务，更方便，但粒度更大。

```java
@Transactional(propagation = Propagation.REQUIRED,timeout = 3000,rollbackFor = Exception.class)
public Integer save(Integer id,String name){
    String insertSql="insert into `tb_user`(`id`,`name`)values(?,?);";
    jdbcTemplate.update(insertSql,id,name);
    return jdbcTemplate.queryForObject("select  count(*) from tb_user",Integer.class);
}
```



### 编程式事务

通过编码的方式手动启用、提交或回滚事务，粒度更细，但更麻烦。

```java
  public Integer save(Integer id,String name){
    Integer num=transactionTemplate.<Integer>execute((TransactionStatus status)->{
      try{
        String insertSql="insert into `tb_user`(`id`,`name`)values(?,?);";
        jdbcTemplate.update(insertSql,id,name);
      }catch (Exception e){
        e.printStackTrace();
        status.setRollbackOnly();
      }
      return jdbcTemplate.queryForObject("select  count(*) from tb_user",Integer.class);
    });
    return num;
  }
//或者
  public Integer save2(Integer id,String name){
    TransactionStatus transactionStatus=transactionManager.getTransaction(new DefaultTransactionDefinition());
    try{
      String insertSql="insert into `tb_user`(`id`,`name`)values(?,?);";
      jdbcTemplate.update(insertSql,id,name);
        transactionManager.commit(transactionStatus);
      }catch (Exception e){
        e.printStackTrace();
        transactionManager.rollback(transactionStatus);
      }
    return jdbcTemplate.queryForObject("select  count(*) from tb_user",Integer.class);
  }
```



### AOP全局事务

一般在项目中使用等待全局性事务，拦截特定包下面的特定方法，不推荐使用。

```java
@Aspect
@Component //事务依然生效
@ConditionalOnBean(DataSource.class)
public class TxAdviceInterceptor {

  private Logger logger = LoggerFactory.getLogger(this.getClass());
  @Value("${tx.timeout:5}")
  private int TX_METHOD_TIMEOUT = 5;
  private String AOP_POINTCUT_EXPRESSION = "execution(* codehome.vip.*.service.*.*(..)) ";
  @Autowired
  private PlatformTransactionManager transactionManager;

  @Bean
  public TransactionInterceptor txAdvice() {
    NameMatchTransactionAttributeSource source = new NameMatchTransactionAttributeSource();
    /*只读事务，不做更新操作*/
    RuleBasedTransactionAttribute readOnlyTx = new RuleBasedTransactionAttribute();
    readOnlyTx.setReadOnly(true);
    readOnlyTx.setPropagationBehavior(TransactionDefinition.PROPAGATION_NOT_SUPPORTED);
    /*当前存在事务就使用当前事务，当前不存在事务就创建一个新的事务*/
    RuleBasedTransactionAttribute requiredTx = new RuleBasedTransactionAttribute();
    requiredTx.setRollbackRules(
        Collections.singletonList(new RollbackRuleAttribute(Exception.class)));
    requiredTx.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
    requiredTx.setTimeout(TX_METHOD_TIMEOUT);
    Map<String, TransactionAttribute> txMap = new HashMap<>();
    txMap.put("save*", requiredTx);
    txMap.put("update*", requiredTx);
    txMap.put("remove*", requiredTx);
    txMap.put("*", readOnlyTx);
    source.setNameMap(txMap);
    TransactionInterceptor txAdvice = new TransactionInterceptor(transactionManager, source);
    if (logger.isInfoEnabled()) {
      logger.info("事务管理器启动成功！");
    }
    return txAdvice;
  }

  @Bean
  public Advisor txAdviceAdvisor() {
    AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
    pointcut.setExpression(AOP_POINTCUT_EXPRESSION);
    return new DefaultPointcutAdvisor(pointcut, txAdvice());
  }
}
```

