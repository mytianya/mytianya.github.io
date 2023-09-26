
---
title: Springboot2.x基础教程：动手制作一个starter包
date: 2023-09-25 02:00:09
tags: springboot
categories: 
- java
---
> 上一篇博客介绍了springboot自动装配的原理。springboot本身有丰富的spring-boot-starter-xx集成组件，这一篇趁热打铁加深理解，我们利用springboot自动装配的机制，制作一个属于自己的starter包。

## 制作一个starter包思路

​	这一篇博客我制作一个上传图片第三方图床的starter，集成常见的第三方图床sm.ms、imgur、github图床等。

​	本教程不会具体的讲解图床上传相关的代码，而是主要分析封装此starter的思路。

1. 首先安装springboot第三方的starter规范命名：xx-spring-boot-starter,我们项目取名为imghost-spring-boot-starter。
2. 对于图床相关的配置项，我们同样准备建立一个ImgHostProperties配置类存放。
3. 同样我们也需要一个ImgHostAutoConfiguration,并且加上条件注解在某些情况下才会注入我们的工具类到IOC容器中。
4. 按照规范在我们项目的META-INF/spring.factories文件下，指定我们starter的自动装配类。
 ## 项目结构一览

![](https://www.codehome.vip/upload/spring-boot-start1.png)

## Starter开发实例

### 引入必要的依赖

​	这里主要引入spring-boot-starter包，spring-boot-configuration-processor其他依赖主要为上传到第三方图床发送Http请求依赖包。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>vip.codehome</groupId>
	<artifactId>imghost-spring-boot-starter</artifactId>
	<version>1.0-SNAPSHOT</version>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.1.0.RELEASE</version>
	</parent>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>
		      <dependency>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>1.18.12</version>
                <scope>compile</scope>
            </dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-configuration-processor</artifactId>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-json</artifactId>
		</dependency>
		<!-- https://mvnrepository.com/artifact/com.squareup.okhttp3/okhttp -->
		<dependency>
			<groupId>com.squareup.okhttp3</groupId>
			<artifactId>okhttp</artifactId>
			<version>3.14.9</version>
		</dependency>
	</dependencies>
    <build>
        <resources>
            <resource>
                <directory>src/main/java</directory>
                <includes>
                    <include>**/spring.factories</include>
                </includes>
                <filtering>false</filtering>
            </resource>
        </resources>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.5.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <!--把注释源码也打入基础包中-->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-source-plugin</artifactId>
                <version>3.0.1</version>
                <executions>
                    <execution>
                        <phase>compile</phase>
                        <goals>
                            <goal>jar-no-fork</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>

```



### 定义一个图床参数上传的配置类

​	上传到SM.MS的API需要上传的token，在sm.ms网站注册获取个人的私钥，后面如果上传到imgur同样可以在此类中加入对应的配置类。

```java
@Data
@ConfigurationProperties(prefix = "imghost")
public class ImgHostProperties {
    SMMS smms;
    @Data
    public static class SMMS{
        String token;
    }
}
```

### 定义上传服务AutoConfiguration类

当imghost.smms.token使用者配置时，我们生成一个SMMSImgHostService的图床上传服务类。

```java
@Configuration
@EnableConfigurationProperties(ImgHostProperties.class)
@ConditionalOnProperty(prefix = "imghost",name = "enabled",havingValue = "true",matchIfMissing = true)
public class ImgHostAutoConfiguration {
	private final ImgHostProperties imgHostProperties;

	public ImgHostAutoConfiguration(ImgHostProperties imgHostProperties) {
		this.imgHostProperties = imgHostProperties;
	}

	@ConditionalOnMissingBean
	@ConditionalOnProperty(prefix="imghost.smms",name="token")
	@Bean
	public SMMSImgHostService imgHostService() {
		return new SMMSImgHostService(imgHostProperties);
	}
}
```

### 编写spring.factories

​	最后在项目的src/main/resource上加入META-INF/spring.factories中引入我们自定义的ImgHostAutoConfiguration配置类。

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=vip.codehome.imghost.ImgHostAutoConfiguration
```

### 如何使用

1. 在使用的项目中引入我们的imghost-spring-boot-starter。

```
<dependency>
	<groupId>vip.codehome</groupId>
	<artifactId>imghost-spring-boot-starter</artifactId>
	<version>1.0-SNAPSHOT</version>
</dependency>
```

2. 在springboot项目中加入如下配置
![](http://www.codehome.vip/upload/spring-boot-starter2.png)

3. 项目使用
```java
	@Autowired
	SMMSImgHostService smms;
	public void upload() {
		System.out.println(smms.upload(newFile("D:\\test.jpg")));
	}
```

## 总结

​	**千里之行，始于足下。这里是SpringBoot教程系列第十八篇。以上就是我们自己动手制作一个starter包的全过程，是不是很简单。此项目在[github可下载源码](https://github.com/mytianya/imghost-spring-boot-starter)**

​	**当前只是实现了上传到SM.MS图床，后期会逐渐迭代一个上传到sm.ms,imgur，github各种图床的通用工具类，敬请期待。如果觉得不错，点赞、评论、关注三连击**