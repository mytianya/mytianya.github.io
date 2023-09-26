---
title: Springboot2.x基础教程：SpringCache缓存抽象详解与Ehcache、Redis缓存配置实战
date: 2023-09-25 02:00:11
tags: springboot
categories: 
- java
---
> 在计算机发展史中一台计算机只需要外部存储器就能运行，但是在实际中磁盘的读取数据的速度往往跟不上CPU的运算速度，因此引入的内存作为CPU和外部存储器之间的缓冲区域。
> 在项目开发过程数据库数据的查询速度远远比不上数据在内存中的访问速度，因此我们通常使用缓存来提高热点数据的访问速度，缓存可谓是计算机科学中最伟大的发明。
## 缓存基本知识
### 缓存命中率
```
命中率=从缓存中读取的速度/总读取次数(从缓存读取次数+从慢速设备读取次数)
Miss率=没有从缓存中读取的速度/总读取次数(从缓存读取次数+从慢速设备读取次数)
```
这是一个缓存非常重要的一个性能指标，往往命中率越高表示缓存作用越大

### 缓存策略
移除策略，即如果缓存满了，如何从缓存中移除数据的策略
1. FIFO(First in First Out):先进先出算法，即先放入缓存的数据先被移除
2. LRU(Least Recently used)：最久未使用的数据优先被有限移除
3. LFU(Least Frequently used): 最少被使用的数据有限被移除

### 缓存设置参数
1. TTL(Time To live): 存活期，即从缓存中创建时间点开始直到它到期的一个时间段（不管在这个时间段内有没有访问都将过期）
2. TTI(Time To Idle): 空闲期，即一个数据多久没被访问将从缓存中移除的时间。
3. 最多被缓存的元素

## Spring注解缓存
Spring 3.1之后，引入了注解缓存技术，其本质上不是一个具体的缓存实现方案，而是一个对缓存使用的抽象，通过在既有代码中添加少量自定义的各种annotation，即能够达到使用缓存对象和缓存方法的返回对象的效果。Spring的缓存技术具备相当的灵活性，不仅能够使用SpEL（Spring Expression Language）来定义缓存的key和各种condition，还提供开箱即用的缓存临时存储方案，也支持和主流的专业缓存集成。其特点总结如下：
1. 少量的配置annotation注释即可使得既有代码支持缓存；
2. 支持开箱即用，不用安装和部署额外的第三方组件即可使用缓存；
3. 支持Spring Express Language（SpEL），能使用对象的任何属性或者方法来定义缓存的key和使用规则条件；
4. 支持自定义key和自定义缓存管理者，具有相当的灵活性和可扩展性。

### 缓存相关注解
| 注解        | 说明                                                         | 配置参数                                                     |
| ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| @Cacheable  | 根据方法的请求参数对其结果进行缓存                           | value：缓存的名称；key：缓存的 key，可以为空，也可按照SpEL 表达式编写，如果不指定，则默认按照方法的所有参数进行组合； condition：缓存的条件，可以为空，使用 SpEL 编写，返回 true 或者 false，只有为 true 才进行缓存 |
| @CachePut   | 根据方法的请求参数对其结果进行缓存，和 @Cacheable 不同的是，它每次都会触发真实方法的调用 | value：缓存的名称；key：缓存的 key，可以为空，也可按照SpEL 表达式编写，如果不指定，则默认按照方法的所有参数进行组合； condition：缓存的条件，可以为空，使用 SpEL 编写，返回 true 或者 false，只有为 true 才进行缓存 |
| @CacheEvict | 够根据一定的条件对缓存进行清空                               | value：缓存的名称；key：缓存的 key，可以为空，也可按照SpEL 表达式编写，如果不指定，则默认按照方法的所有参数进行组合；condition：缓存的条件，可以为空，使用 SpEL 编写，返回 true 或者 false，只有为 true 才进行缓存； allEntries：是否清空所有缓存内容，默认为 false，如果指定为 true，则方法调用后将立即清空所有缓存； beforeInvocation：是否在方法执行前就清空，默认为 false，如果指定为 true，则在方法还没有执行的时候就清空缓存，默认情况下，如果方法执行抛出异常，则不会清空缓存 |

### 集成Ehcache缓存
EhCache是一个比较成熟的Java缓存框架，最早从hibernate发展而来， 是进程中的缓存系统，它提供了用内存，磁盘文件存储，以及分布式存储方式等多种灵活的cache管理方案，快速简单。
Springboot对ehcache的使用非常支持，所以在Springboot中只需做些配置就可使用，且使用方式也简易。
#### 引入依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache</artifactId>
</dependency>
```

#### 配置Ehcache缓存配置
```xml
<ehcache xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:noNamespaceSchemaLocation="../config/ehcache.xsd">
    <!--磁盘路径，内存满了存放位置-->
    <diskStore path="caches"/>
    <cache name="users"
 	<!-- 最多存储的元素-->
	maxElementsInMemory="10000"
	<!--缓存中对象是否永久有效-->
    eternal="false"
	<!--缓存数据在失效前的允许闲置时间(单位:秒)，仅当eternal=false时使用,默认值是0表示可闲置时间无穷大,若超过这个时间没有访问此Cache中的某个元素,那么此元素将被从Cache中清除-->
    timeToIdleSeconds="120"
	<!--缓存数据的总的存活时间（单位：秒），仅当eternal=false时使用，从创建开始计时，失效结束。-->
    timeToLiveSeconds="120"
	<!--磁盘缓存中最多可以存放的元素数量,0表示无穷大-->
    maxElementsOnDisk="10000000"
    diskExpiryThreadIntervalSeconds="120"
	<!--缓存策略-->
    memoryStoreEvictionPolicy="LRU">
        <persistence strategy="localTempSwap"/>
    </cache>
</ehcache>
```
applicatin.yml配置ehcache地址，在springboot启动类加上@EnableCaching注解
```xml
spring:
  cache:
    ehcache:
      config: classpath:ehcache.xml
    type: ehcache
```
### 配置Redis缓存配置
#### 引入依赖
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
    <version>2.6.2</version>
</dependency>
```
#### Redis配置
```java
@Configuration
@EnableCaching//开启缓存
public class RedisConfig extends CachingConfigurerSupport {

    private final static Logger log= LoggerFactory.getLogger(RedisConfig.class);

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {

        return new RedisCacheManager(
                RedisCacheWriter.nonLockingRedisCacheWriter(redisConnectionFactory),
                this.getRedisCacheConfigurationWithTtl(60*60*24), // 默认策略，未配置的 key 会使用这个
                this.getRedisCacheConfigurationMap() // 指定 key 策略
        );
    }

    private Map<String, RedisCacheConfiguration> getRedisCacheConfigurationMap() {
        Map<String, RedisCacheConfiguration> redisCacheConfigurationMap = new HashMap<>();
        //可以进行过期时间配置
        redisCacheConfigurationMap.put("24h", this.getRedisCacheConfigurationWithTtl(60*60*24));
        redisCacheConfigurationMap.put("30d", this.getRedisCacheConfigurationWithTtl(60*60*24*30));
        return redisCacheConfigurationMap;
    }

    private RedisCacheConfiguration getRedisCacheConfigurationWithTtl(Integer seconds) {
        Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<>(Object.class);
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);

        RedisCacheConfiguration redisCacheConfiguration = RedisCacheConfiguration.defaultCacheConfig();

        redisCacheConfiguration = redisCacheConfiguration.serializeValuesWith(
                RedisSerializationContext
                        .SerializationPair
                        .fromSerializer(jackson2JsonRedisSerializer)
        ).entryTtl(Duration.ofSeconds(seconds));

        //自定义前缀 默认为中间两个：
        redisCacheConfiguration = redisCacheConfiguration.computePrefixWith(myKeyPrefix());

        return redisCacheConfiguration;
    }

    /**
     * 缓存前缀（追加一个冒号 : ）
     * @return
     */
    private CacheKeyPrefix myKeyPrefix(){
        return (name) -> {
            return name +":";
        };
    }

    /**
     * 生成Key规则
     * @return
     */
    @Bean
    public KeyGenerator wiselyKeyGenerator() {
        return new KeyGenerator() {
            @Override
            public Object generate(Object target, Method method, Object... params) {
                StringBuilder sb = new StringBuilder();
                sb.append(target.getClass().getName());
                sb.append("." + method.getName());
                if(params==null||params.length==0||params[0]==null){
                    return null;
                }
                String join = String.join("&", Arrays.stream(params).map(Object::toString).collect(Collectors.toList()));
                String format = String.format("%s{%s}", sb.toString(), join);
                //log.info("缓存key：" + format);
                return format;
            }
        };
    }

    @Bean
    public RedisTemplate<String, String> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, String> redisTemplate = new RedisTemplate<String, String>();
        redisTemplate.setConnectionFactory(factory);
        return redisTemplate;
    }


    /**
     * 缓存异常处理
     * @return
     */
    @Bean
    @Override
    public CacheErrorHandler errorHandler() {
        CacheErrorHandler cacheErrorHandler = new CacheErrorHandler() {
            @Override
            public void handleCacheGetError(RuntimeException e, Cache cache, Object key) {
                log.info("redis缓存获取异常："+ key);
            }

            @Override
            public void handleCachePutError(RuntimeException e, Cache cache, Object key, Object value) {
                log.info("redis缓存添加异常："+ key);
            }

            @Override
            public void handleCacheEvictError(RuntimeException e, Cache cache, Object key) {
                log.info("redis缓存删除异常："+ key);
            }

            @Override
            public void handleCacheClearError(RuntimeException e, Cache cache) {
                log.info("redis缓存清理异常");
            }
        };
        return cacheErrorHandler;
    }
}
```
applicatin.yml配置Redis连接信息
```yml
spring:
  redis:
    timeout: 5000ms
    lettuce:
      pool:
        max-active: 5
        max-wait: -1
        max-idle: 10
  cache:
    type: redis
```

## 缓存注解使用举例
对用户的查询、新增、删除使用缓存，以及更新缓存
```java
public interface UserService {
    @Cacheable(value = "users",key = "#userDO.id")
    List<UserDO> queryUsers(UserDO userDO);
    @CachePut(value = "users",key ="#userDO.id" )
    void saveUser(UserDO userDO);
    @CacheEvict(value = "users",key = "#userDO.id")
    void removeUser(UserDO userDO);
}
```
**千里之行，始于足下。这里是SpringBoot教程系列第十五篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**