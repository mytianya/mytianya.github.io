
---
title: SpringBoot2.x基础教程：连接池配置
date: 2023-09-25 02:00:48
tags: springboot
categories: 
- java
---
## HikariCP

## 连接池基本配置

```properties
#数据源驱动名称
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
#连接池类型
spring.datasource.type=com.zaxxer.hikari.HikariDataSource
#数据源地址
spring.datasource.url=jdbc:mysql://localhost:3306/test
#数据源连接名
spring.datasource.username=root
#数据源地址
spring.datasource.password=123456
```

## HikariCP配置

```properties
#最小空闲连接
spring.datasource.hikari.mininum-idle=10
#最大连接数
spring.datasource.hikari.maximun-pool-size=10
#空闲超时时间，这个属性用来控制空闲连接允许保留在连接池中的最大时间，这个属性只有在minimumIdle（最小空闲连接数）小于maximumPoolSize(最大连接数)时才会生效,空闲连接断开会有15s-30s的延迟变动时间.在这个超时时间之前空闲连接永远不会断开,当连接池达到minimumIdle,连接将永远不会断开，即使处于闲置状态.值为0表示空闲连接将永远不会从连接池中移除，最小值为10000ms （10s),默认值为600000(10min)
spring.datasource.idle-timeout=300000
#最大生命周期.这个属性用来控制连接池中连接的最大生命周期,一个使用中的连接永远不会被断开,只有当它处于关闭状态然后才会被移除.推荐设置比任何数据库或基础设施规定的连接时间限制少至少30秒。 值为0表示没有最大寿命（无限寿命）， 默认：1800000（30分钟）,由于HikariCP的housekeeper默认每30s运行一次,以维护minimumIdle最小空闲连接数，它可能添加新连接或者断开空闲连接，所以你必须设置maxLifetime属性比（mysql)wait_timeout时间少一些来避免 broken connection / exceptions.意思就是说比如mysql wait_timeout为10min,此时有一个连接由于达到超时时间，mysql主动断开了连接，而HakariCP仍然持有此连接，如果再使用此连接去请求数据库则会发生异常,设置maxLifetime最大生命周期比wait_timeout少30s后,就能确保再housekeeper运行期间提前断开此连接，避免发生异常.
spring.datasource.hikari.max-lifetime=600000
#连接池名称
spring.datasource.hikari.poolName=CDHikariPool
```

## Druid

### 引入依赖

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.1.17</version>
</dependency>
```

### Druid配置

```yaml
spring:
  datasource:
    driver-class-name: oracle.jdbc.OracleDriver
    maxActive: 20
    type: com.alibaba.druid.pool.DruidDataSource
    url: jdbc:oracle:thin:@192.28.4.21:1521:fdp1
    username: imf
    password: ca2804
    druid:
      filters: stat,wall,slf4j
      # 通过 connectProperties 属性来打开 mergeSql 功能；慢 SQL 记录
      connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000
      initial-size: 20
      max-active: 20
      min-idle: 20
      max-wait: 60000
      #是否缓存preparedStatement，也就是PSCache。PSCache对支持游标的数据库性能提升巨大，比如说oracle。在mysql下建议关闭
      pool-prepared-statements: true
      #要启用PSCache，必须配置大于0，当大于0时，poolPreparedStatements自动触发修改为true。在Druid中，不会存在Oracle下PSCache占用内存过多的问题，可以把这个数值配置大一些，比如说100
      max-pool-prepared-statement-per-connection-size: 100
      #测试连接
      validation-query: select 'x' from dual
      validation-query-timeout: 10
      #申请连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能。
      test-on-borrow: true
      test-while-idle: true
      #归还连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能
      test-on-return: false
      web-stat-filter:
        enabled: true
        url-pattern: "/*"
        exclusions: "*.js,*.gif,*.jpg,*.bmp,*.png,*.css,*.ico,/druid/*"
      stat-view-servlet:
        url-pattern: "/druid/*"
        login-username: atc
        login-password: atc4
        reset-enable: false
        enabled: true
```

### Druid监控页面

浏览器访问应用地址http://<ip>:<port>/druid/login.html即可以访问监控页面。

![image-20210706162306080](/img/druid-monitor.png)