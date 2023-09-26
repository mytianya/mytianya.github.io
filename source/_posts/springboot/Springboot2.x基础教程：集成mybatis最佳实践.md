
---
title: Springboot2.x基础教程：集成mybatis最佳实践
date: 2023-09-25 02:00:08
tags: springboot
categories: 
- java
---
>前面文章介绍过SpringBoot结合Jpa实现对数据库的操作。今天介绍下SprigBoot集成Mybatis的相关知识点。
>Mybatis作为一个半自动化的ORM框架,根据条件动态拼接SQL，是其一大优点。贴合原生SQL的写法，方便开发人员灵活的编写复杂的SQL语句。
>SpringBoot集成Mybatis的配置还是相当简单的，教程并且会给出常见针对Mysql数据CURD、分页、批量操作的写法。
## SpringBoot配置Mybatis
### 引入依赖
```xml
<!--mybatis依赖-->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.0.1</version>
</dependency>
<!--分页依赖-->
<dependency>
    <groupId>com.github.pagehelper</groupId>
    <artifactId>pagehelper</artifactId>
    <version>5.1.2</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
```
### 配置数据源
这里使用springboot默认的数据连接池hikari,号称最快的数据库链接池，allowMultiQueries=true开启批量更新。
```yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/spring?autoReconnect=true&useSSL=false&characterEncoding=utf-8&failOverReadOnly=false&allowMultiQueries=true
    driver-class-name: com.mysql.jdbc.Driver
    username: root
    password: 123456
    hikari:
      maximum-pool-size: 50
      connection-test-query: SELECT 1
      #1分钟后释放
      idle-timeout: 600000
      minimum-idle: 10
      auto-commit: true
      validation-timeout: 250
```

### 配置Mybatis参数
这里有两种配置一种直接在springboot默认的application.yml中配置，一种单独提取出来配置，本文采用第二种。
```yml
mybatis:
  #指定配置mybatis参数配置地址
  config-location: classpath:mybatis-config.xml
  #指定接口xml地址
  mapper-locations: classpath:mappers/*.xml
```
这里贴出mybatis-config.xml的配置:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!-- 全局参数 -->
    <settings>
        <!-- 使全局的映射器启用或禁用缓存。 -->
        <setting name="cacheEnabled" value="true"/>
        <!-- 全局启用或禁用延迟加载。当禁用时，所有关联对象都会即时加载。 -->
        <setting name="lazyLoadingEnabled" value="true"/>
        <!-- 当启用时，有延迟加载属性的对象在被调用时将会完全加载任意属性。否则，每种属性将会按需要加载。 -->
        <setting name="aggressiveLazyLoading" value="true"/>
        <!-- 是否允许单条sql 返回多个数据集  (取决于驱动的兼容性) default:true -->
        <setting name="multipleResultSetsEnabled" value="true"/>
        <!-- 是否可以使用列的别名 (取决于驱动的兼容性) default:true -->
        <setting name="useColumnLabel" value="true"/>
        <!-- 允许JDBC 生成主键。需要驱动器支持。如果设为了true，这个设置将强制使用被生成的主键，有一些驱动器不兼容不过仍然可以执行。  default:false  -->
        <setting name="useGeneratedKeys" value="true"/>
        <!-- 指定 MyBatis 如何自动映射 数据基表的列 NONE：不隐射　PARTIAL:部分  FULL:全部  -->
        <setting name="autoMappingBehavior" value="PARTIAL"/>
        <!-- 这是默认的执行类型  （SIMPLE: 简单； REUSE: 执行器可能重复使用prepared statements语句；BATCH: 执行器可以重复执行语句和批量更新）  -->
        <!-- 对于批量更新操作缓存SQL以提高性能 BATCH,SIMPLE -->
        <setting name="defaultExecutorType" value="SIMPLE"/>
        <!-- 数据库超过25000秒仍未响应则超时 -->
        <setting name="defaultStatementTimeout" value="25000"/>
        <!-- 使用驼峰命名法转换字段。 -->
        <setting name="mapUnderscoreToCamelCase" value="true"/>
        <!-- 设置本地缓存范围 session:就会有数据的共享  statement:语句范围 (这样就不会有数据的共享 ) defalut:session -->
        <setting name="localCacheScope" value="SESSION"/>
        <!-- 设置但JDBC类型为空时,某些驱动程序 要指定值,default:OTHER，插入空值时不需要指定类型 -->
        <setting name="jdbcTypeForNull" value="NULL"/>
        <!-- 设置关联对象加载的形态，此处为按需加载字段(加载字段由SQL指 定)，不会加载关联表的所有字段，以提高性能 -->
        <setting name="aggressiveLazyLoading" value="false"/>
        <setting name="logImpl" value="org.apache.ibatis.logging.stdout.StdOutImpl"/>
    </settings>
    <typeAliases>
        <!--批量别名，默认为别名的规范就是对应包装类的类名首字母变为小写-->
        <package name="vip.codehome.springboot.tutorials.entity"/>
    </typeAliases>
    <plugins>
        <!--pagehelper分页插件-->
        <plugin interceptor="com.github.pagehelper.PageInterceptor">
            <!-- 使用MySQL方言的分页 -->
            <property name="helperDialect" value="sqlserver"/><!--如果使用mysql，这里value为mysql-->
            <property name="pageSizeZero" value="true"/>
        </plugin>
    </plugins>
</configuration>
```
## 增删改查最优写法
针对这张表给出Mybatis常见动态SQL写法
![tb_user](/img/tb_user.png)
首先编辑Mapper层接口，注意要加上@Mapper注解

```java
@Mapper
@Repository
public interface UserMapper {
	//插入
    int insert(UserDO userDO);
	//分页查询
    List<UserDO> select(UserDO userDO);
	//更新
    int update(UserDO userDO);
	//删除
    int delete(UserDO userDO);
	//批量插入
    int insertBatch(List<UserDO> list);
	//批量更新
    int updateBatch(List<UserDO> list);
	//批量删除
    int deleteBatch(Long[] array);
}
```
前面配置了mapper-locations为classpath:mappers/*.xml，即mapper对应的xml文件放在src/main/resources/mappers目录下,
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="vip.codehome.springboot.tutorials.mapper.UserMapper">
    <sql id="base">
        id,name,account,age,forbidden,login_time,passwd
    </sql>
    <insert id="insert" parameterType="userDO">
        insert into tb_user(
        <trim suffixOverrides=",">
            id,
            account,
            passwd,
            name,
            <if test="age!=null">
                age,
            </if>
            <if test="forbidden!=null and forbidden!=''">
                forbidden,
            </if>
            <if test="loginTime!=null">
                login_time,
            </if>
        </trim>
        )values (
        <trim suffixOverrides=",">
            #{id},
            #{account},
            #{passwd},
            #{name},
            <if test="age!=null">
                #{age},
            </if>
            <if test="forbidden!=null and forbidden!=''">
                #{forbidden},
            </if>
            <if test="loginTime!=null">
                #{loginTime},
            </if>
        </trim>
        )
    </insert>
    <select id="select" resultType="userDO">
        select
        <include refid="base"/>
        from tb_user
        <where>
            <if test="account!=null and account!=''">
                and account=#{account}
            </if>
            <if test="name!=null and name!=''">
                and name like concat('%',#{name},'%')
            </if>
        </where>
    </select>
    <update id="update" parameterType="userDO">
        update tb_user
        <set>
            <if test="name!=null and name!=''">
                name=#{name},
            </if>
            <if test="age!=null">
                age=#{age}
            </if>
        </set>
        where id=#{id}
    </update>

    <delete id="delete" parameterType="userDO">
        delete from tb_user where id=#{id}
    </delete>
    <insert id="insertBatch" parameterType="list" useGeneratedKeys="true">
        <selectKey resultType="long" keyProperty="id" order="AFTER">
            SELECT
            LAST_INSERT_ID()
        </selectKey>
        insert into tb_user(
        account,passwd,name,age,forbidden,login_time
        )values
        <foreach collection="list" item="user" index="index" separator=",">
            (#{user.account},
            #{user.passwd},
            #{user.name},
            #{user.age},
            #{user.forbidden},
            #{user.loginTime})
        </foreach>

    </insert>
    <update id="updateBatch" parameterType="list">
        <foreach collection="list" item="user" index="index" open="" close="" separator=";">
            update tb_user
            <set>
                <if test="name!=null and name!=''">
                    name=#{name},
                </if>
                <if test="age!=null">
                    age=#{age}
                </if>
            </set>
            where id=#{id}
        </foreach>
    </update>
    <delete id="deleteBatch" parameterType="long">
        delete from tb_user
        where id in
        <foreach collection="array" item="id" open="(" separator="," close=")">
            #{id}
        </foreach>
    </delete>
</mapper>
```
### 单元测试用例
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class UserMapperTest {
    @Autowired
    UserMapper userMapper;
    //插入
    @Test
    public void testAdd(){
        UserDO userDO=new UserDO();
        userDO.setPasswd("codehome");
        userDO.setAccount("codehome");
        userDO.setName("name");
        userDO.setForbidden(true);
        userDO.setLoginTime(LocalDateTime.now());
        userMapper.insert(userDO);
    }
    //分页
    @Test
    public void page(){
        PageHelper.startPage(0,10);
        UserDO userDO=new UserDO();
        userDO.setName("n");
       PageInfo<UserDO> userDOPage= new PageInfo<>(userMapper.select(userDO));
    }
    //更新
    @Test
    public void update(){
        UserDO userDO=new UserDO();
        userDO.setId(1L);
        userDO.setName("编程之家");
        userMapper.update(userDO);
    }
    //删除
    @Test
    public void delete(){
        UserDO userDO=new UserDO();
        userDO.setId(1L);
        userMapper.delete(userDO);
    }
    //批量插入
    @Test
    public void testAddBatch(){
        UserDO userDO=new UserDO();
        userDO.setPasswd("codehome");
        userDO.setAccount("codehome");
        userDO.setName("name");
        userDO.setForbidden(true);
        userDO.setLoginTime(LocalDateTime.now());
        UserDO userDO1=new UserDO();
        userDO1.setPasswd("codehome");
        userDO1.setAccount("codehome");
        userDO1.setName("name");
        userDO1.setForbidden(true);
        userDO1.setLoginTime(LocalDateTime.now());
        userMapper.insertBatch(Arrays.asList(userDO,userDO1));
    }
    //批量更新
    @Test
    public void testUpdateBatch(){
        UserDO userDO=new UserDO();
        userDO.setPasswd("codehome1");
        userDO.setAccount("codehome1");
        userDO.setName("name1");
        userDO.setId(1L);
        userDO.setForbidden(true);
        userDO.setLoginTime(LocalDateTime.now());
        UserDO userDO1=new UserDO();
        userDO.setId(2L);
        userDO1.setPasswd("codehome2");
        userDO1.setAccount("codehome2");
        userDO1.setName("name2");
        userDO1.setForbidden(true);
        userDO1.setLoginTime(LocalDateTime.now());
        userMapper.insertBatch(Arrays.asList(userDO,userDO1));
    }
    @Test
    public void deleteBatch(){
        userMapper.deleteBatch(new Long[]{1L,2L});
    }
}
```
这里举出的例子使用了if、set、where、foreach、trim、sql等标签，更详细的解释可以参考官方[动态 SQL](https://mybatis.org/mybatis-3/zh/dynamic-sql.html "动态 SQL")
**以上就是对springboot集成Mybatis的配置，跟常见CURD与批量操作，更多的内容篇幅有限大家可以参考文中给出的链接。
千里之行，始于足下。这里是SpringBoot教程系列第十四篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**