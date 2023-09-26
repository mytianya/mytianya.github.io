---
title: Springboot2.x基础教程：配置文件详解
date: 2023-09-23 21:53:08
tags: springboot
categories: 
- java
---
>当使用Spring Initializr构建springboot项目时，会自动在src/main/resources下生产application.properties文件。今天我们就来聊聊SpringBoot的配置文件。

## 配置文件的作用
SpringBoot采用“习惯优于配置”的理念，项目中存在大量的配置，采用默认配置，让你无需手动配置。
SpringBoot能够识别properties格式与yml格式的配置文件（我们一般使用yml格式更多）。当需要**对默认配置进行修改或者自定义配置**时可用通过修改配置文件达到目的。

## 配置文件的基本使用
这是使用yml格式的配置举例，说明application.yml中如何配置，以及代码中如何获取。
### 数字，字符串，布尔获取
配置文件写法：
```yml
version: 1.0
author: codhome.vip
flag: true
```
使用@Value获取:
```
@Value("${version}")
float version;
@Value("${author}")
String author;
//这里的true为默认值
@Value("${flag:true}")
boolean flag;
```

### 对象、Map写法与获取
配置文件写法：
```yml
user:
  userName: codehome
  age: 18
  forbidden: true
```
使用@ConfigurationProperties获取:
```
@Configuration
@ConfigurationProperties(prefix = "user")
@Data
public class UserProperties {
    String userName;
    int age;
    boolean forbidden;
}
//注入使用
@Autowired
UserProperties userProperties;
```
## List、Set、Array获取
配置文件写法：
```yml
random: 10,20,30
```
使用@Value获取:
```
@Value("#{'${random}'.split(',')}")
int[] randoms; //List<Integer>也可以
```
配置文件写法：
```yml
random1:
  users:
    - zhangsao
    - lisi
    - wangwu
```
使用@ConfigurationProperties获取:

```
@Configuration
@ConfigurationProperties(prefix = "random1")
@Data
public class UserProperties {
    List<String> users;
}
```
## 总结下两种注解区别
![configuration-vs-value](/img/configuration-vs-value.png)

## 多环境配置
我们在主配置文件编写的时候，文件名可以是 application-{profile}.properties/yml,使用spring.profiles.active激活使用哪一个配置文件
```yml
spring:
  profiles:
    active: dev
---

#配置开发环境
spring:
  profiles: dev
server:
  port: 9000

---

#配置生产环境
spring:
  profiles: prod
server:
  port: 9100

---

#配置测试环境
spring:
  profiles: dev
server:
  port: 8000
```

## 配置文件优先级
### 项目内部配置文件
1. application.properties 与application.yml同一个目录共存时，properties配置优先级更高
2. ConfigFileApplicationListener中默认配置文件加载顺序
项目根目录config文件夹>根目录配置文件>resources下config文件夹>resources下配置文件
3. 当多个配置文件属性不冲突时，配置是互补的
![](/img/config-load-order.jpg)
4. 也可以指定配置文件地址
```shell
java -jar run-0.0.1-SNAPSHOT.jar --spring.config.location=D:/application.properties
```
### 外部配置

[官方参考链接](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-external-config)

1. [Devtools global settings properties](https://docs.spring.io/spring-boot/docs/current/reference/html/using-spring-boot.html#using-boot-devtools-globalsettings) in the `$HOME/.config/spring-boot` directory when devtools is active.
2. [`@TestPropertySource`](https://docs.spring.io/spring/docs/5.2.9.RELEASE/javadoc-api/org/springframework/test/context/TestPropertySource.html) annotations on your tests.
3. `properties` attribute on your tests. Available on [`@SpringBootTest`](https://docs.spring.io/spring-boot/docs/2.3.4.RELEASE/api/org/springframework/boot/test/context/SpringBootTest.html) and the [test annotations for testing a particular slice of your application](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-testing-spring-boot-applications-testing-autoconfigured-tests).
4. Command line arguments.
5. Properties from `SPRING_APPLICATION_JSON` (inline JSON embedded in an environment variable or system property).
6. `ServletConfig` init parameters.
7. `ServletContext` init parameters.
8. JNDI attributes from `java:comp/env`.
9. Java System properties (`System.getProperties()`).
10. OS environment variables.
11. A `RandomValuePropertySource` that has properties only in `random.*`.
12. [Profile-specific application properties](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-external-config-profile-specific-properties) outside of your packaged jar (`application-{profile}.properties` and YAML variants).
13. [Profile-specific application properties](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-external-config-profile-specific-properties) packaged inside your jar (`application-{profile}.properties` and YAML variants).
14. [Application properties](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-external-config-application-property-files) outside of your packaged jar (`application.properties` and YAML variants).
15. [Application properties](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-external-config-application-property-files) packaged inside your jar (`application.properties` and YAML variants).
16. [`@PropertySource`](https://docs.spring.io/spring/docs/5.2.9.RELEASE/javadoc-api/org/springframework/context/annotation/PropertySource.html) annotations on your `@Configuration` classes. Please note that such property sources are not added to the `Environment` until the application context is being refreshed. This is too late to configure certain properties such as `logging.*` and `spring.main.*` which are read before refresh begins.
17. Default properties (specified by setting `SpringApplication.setDefaultProperties`).

**千里之行，始于足下。这里是SpringBoot教程系列第五篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**