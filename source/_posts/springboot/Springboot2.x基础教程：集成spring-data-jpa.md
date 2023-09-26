
---
title: Springboot2.x基础教程：集成spring-data-jpa
date: 2023-09-25 02:00:09
tags: springboot
categories: 
- java
---
>Spring Data是Spring 社区的一个子项目，主要用于简化数据（关系型&非关系型）访问，其主要目标是使得数据库的访问变得方便快捷。
>目前支持的关系型与非关系型数据有Spring data JPA、Mongodb、Redis、JDBC、Elasticsearch....具体可查看[Spring官网](https://spring.io/projects/spring-data "Spring官网")
>JPA全称为Java Persistence API（Java持久层API），它是Sun公司在JavaEE 5中提出的Java持久化规范。它为Java开发人员提供了一种对象/关联映射工具，来管理Java应用中的关系数据，JPA吸取了目前Java持久化技术的优点，旨在规范、简化Java对象的持久化工作。很多ORM框架都是实现了JPA的规范，如：Hibernate、EclipseLink。
>Spring Data JPA 是 Spring 基于 ORM 框架、JPA 规范的基础上封装的一套 JPA 应用框架，底层使用了 Hibernate 的 JPA 技术实现，可使开发者用极简的代码即可实现对数据的访问和操作。它提供了包括增删改查等在内的常用功能，且易于扩展！学习并使用 Spring Data JPA 可以极大提高开发效率。

## SpringBoot集成Spring-data-jpa
### 引入JPA依赖
```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<!--以操作Mysql数据库为例，引入Mysql数据库驱动-->
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <scope>runtime</scope>
</dependency>
```
### jpa与Mysql配置
spring.hibernate.ddl-auto配置介绍：
1. create:每次删除上次的表，根据model重新生成新表，一般用于测试，会丢失数据。[删除-创建-操作]
2. create-drop：每次加载 hibernate 时根据 model 类生成表，但是 sessionFactory 一关闭，表就自动删除。[删除-创建-操作-再删除]
3. update: 最常用的属性，第一次加载 hibernate 时根据 model 类会自动建立起表的结构（前提是先建立好数据库），以后加载 hibernate 时根据 model 类自动更新表结构，即使表结构改变了，但表中的行仍然存在，不会删除以前的行。要注意的是当部署到服务器后，表结构是不会被马上建立起来的，是要等应用第一次运行起来后才会。[没表-创建-操作 | 有表-更新没有的属性列-操作]
4. validate: 每次加载 hibernate 时，验证创建数据库表结构，只会和数据库中的表进行比较，不会创建新表，但是会插入新值。[启动验证表结构，验证不成功，项目启动失败]

```yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mytestdb?serverTimezone=UTC&useSSL=false&allowPublicKeyRetrieval=true
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: create
    #设置数据库方言
    database-platform: org.hibernate.dialect.MySQL5InnoDBDialect
    #打印sql
    show-sql: true
```
### 关于数据库表映射的注解
1. @Entity: 指定映射的表名
2. @Id：指定主键
3. @GeneratedValue：主键生成策略
4. @Column: 指定映射的列名
5. @Temporal: 关于时间的精度
6. @Transient: 实体类忽略映射的字段
```java
@Entity(name = "tb_user")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserDO implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(name = "name",nullable = false)
    String userName;
    String account;
    String passwd;
    int age;
    boolean forbidden;
    @Temporal(value = TemporalType.TIMESTAMP)
    Date loginTime;
    @Transient
    String token;
}
```
### 增删改查
是不是很简单??? 只要继承JpaRepository就能通过UserRepository实现对一个实体的CURD操作。
```java
@Repository
public interface UserRepository extends JpaRepository<UserDO,Long> {
}
```
### 分页、排序、自定义语句查询
```java
@Repository
public interface UserRepository extends JpaRepository<UserDO,Long> {
    @Query("from tb_user u where u.userName like :userName")
    Page<UserDO> findUserDOByUserName(@Param("userName")String userName, Pageable pageable);
    Page<UserDO> findAll(Pageable pageable);
}
//使用方法
Pageable pageable= PageRequest.of(0,10);
Page<UserDO> pageUsers=userRepository.findAll(pageable,Sort.by("age").ascending().and(Sort.by("userName").descending()));
List<UserDO> users=pageUsers.getContent();
int totalPages= pageUsers.getTotalPages();
```
### 方法命名规则查询
```java
@Repository
public interface UserRepository extends JpaRepository<UserDO,Long> {
    List<UserDO> findUserDOByAccountAndAgeAndUserNameLike(String account,int age,String userName);
}
```
更多使用方法：
![](/img/jpa.png)
**Jpa官方文档地址：https://docs.spring.io/spring-data/jpa/docs/2.3.2.RELEASE/reference/html/#reference
千里之行，始于足下。这里是SpringBoot教程系列第六篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**