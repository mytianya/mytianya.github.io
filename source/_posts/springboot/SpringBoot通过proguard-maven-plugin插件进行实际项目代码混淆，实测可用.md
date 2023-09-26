
---
title: SpringBoot通过proguard-maven-plugin插件进行实际项目代码混淆，实测可用
date: 2023-09-25 02:00:20
tags: springboot
categories: 
- java
---
>本文主要研究下如何使用proguard-maven-plugin插件混淆springboot代码。工程代码是实际跑在线上的Springboot2.x项目，踩过N个坑，最后实测成功。
## 先说贴出成功的配置

```xml
<build>
    <finalName>spring</finalName>
    <resources>
        <resource>
            <directory>${basedir}/src/main/java</directory>
            <includes>
                <include>**/*.xml</include>
            </includes>
        </resource>
        <resource>
            <directory>${basedir}/src/main/resources</directory>
            <filtering>true</filtering>
            <excludes>
                <exclude>**/application-*.yml</exclude>
            </excludes>
        </resource>
    </resources>
    <plugins>
        <!--代码混淆-->
        <plugin>
            <groupId>com.github.wvengen</groupId>
            <artifactId>proguard-maven-plugin</artifactId>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals><goal>proguard</goal></goals>
                </execution>
            </executions>
            <configuration>
                <proguardVersion>6.0.3</proguardVersion>
                <injar>${project.build.finalName}.jar</injar>
                <!-- <injar>classes</injar> -->
                <outjar>${project.build.finalName}.jar</outjar>
                <obfuscate>true</obfuscate>
                <options>
                    <!--  ##默认是开启的，这里关闭shrink，即不删除没有使用的类/成员-->
                    <option>-dontshrink</option>
                    <!-- ##默认是开启的，这里关闭字节码级别的优化-->
                    <option>-dontoptimize</option>
                    <!--##对于类成员的命名的混淆采取唯一策略-->
                    <option>-useuniqueclassmembernames</option>
                    <!--- 混淆类名之后，对使用Class.forName('className')之类的地方进行相应替代-->
                    <option>-adaptclassstrings </option>
                    <option>-ignorewarnings</option>
                    <!-- 混淆时不生成大小写混合的类名，默认是可以大小写混合-->
                    <option>-dontusemixedcaseclassnames</option>
                    <!-- This option will replace all strings in reflections method invocations with new class names.
                         For example, invokes Class.forName('className')-->
                    <!-- <option>-adaptclassstrings</option> -->
                    <!-- This option will save all original annotations and etc. Otherwise all we be removed from files.-->
                    <!-- 不混淆所有特殊的类-->
                    <option>-keepattributes Exceptions,InnerClasses,Signature,Deprecated,
                        SourceFile,LineNumberTable,*Annotation*,EnclosingMethod</option>
                    <!-- This option will save all original names in interfaces (without obfuscate).-->
                    <option>-keepnames interface **</option>
                    <!-- This option will save all original methods parameters in files defined in -keep sections,
                         otherwise all parameter names will be obfuscate.-->
                    <option>-keepparameternames</option>
                    <!-- <option>-libraryjars **</option> -->
                    <!-- This option will save all original class files (without obfuscate) but obfuscate all in domain package.-->
                    <!--<option>-keep class !com.slm.proguard.example.spring.boot.domain.** { *; }</option>-->
                    <option>-keep class !com.dsys.project.** { *; }</option>
                    <option>-keep class com.dsys.project.App { *; }</option>
                    <option>-keep class com.dsys.project.config.** { *; }</option>
                    <!--保留不然Mybatis报错-->
                    <option>-keep class com.dsys.project.entity.** { *; }</option>
                    <option>-keep class com.dsys.project.utils.PageRes { *; }</option>
                    <option>-keep class com.dsys.project.controller.** { *; }</option>
                    <option>-keep class com.dsys.project.mp.controller.** { *; }</option>
                    <option>-keep class com.dsys.project.mp.config.** { *; }</option>
                    <option>-keep class com.dsys.project.dto.** { *; }</option>
                    <option>-keep class * implements java.io.Serializable </option>
                    <!-- This option will save all original class files (without obfuscate) in service package-->
                    <!--<option>-keep class com.slm.proguard.example.spring.boot.service { *; }</option>-->
                    <!-- This option will save all original interfaces files (without obfuscate) in all packages.-->
                    <option>-keep interface * extends * { *; }</option>
                    <!-- <option>-keep @org.springframework.stereotype.Service class *</option> -->
                    <!-- This option will save all original defined annotations in all class in all packages.-->
                    <option>-keepclassmembers class * {
                        <!-- @org.springframework.beans.factory.annotation.Autowired *; -->
                        @org.springframework.beans.factory.annotation.Value *;
                        }
                    </option>
                </options>
                <libs>
                    <!-- Include main JAVA library required.-->
                    <lib>${java.home}/lib/rt.jar</lib>
                </libs>
            </configuration>
            <dependencies>
                <dependency>
                    <groupId>net.sf.proguard</groupId>
                    <artifactId>proguard-base</artifactId>
                    <version>6.2.2</version>
                </dependency>
            </dependencies>
        </plugin>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <executions>
                <execution>
                    <!-- <phase>none</phase> -->
                    <goals>
                        <goal>repackage</goal>
                    </goals>
                    <configuration>
                        <mainClass>com.dsys.project.App</mainClass>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

## 主要的坑,springboot项目配置注意项

### 启动类不能混淆

```java
//混淆会把Bean的名称重复，这里要求SpringBoot生成唯一的BeanName
@SpringBootApplication
@MapperScan("com.dsys.project.dao")
@ComponentScan("com.dsys")
@ServletComponentScan("com.dsys")
@EnableCaching
@EnableAsync
public class App {
    public static class CustomGenerator implements BeanNameGenerator {

        @Override
        public String generateBeanName(BeanDefinition definition, BeanDefinitionRegistry registry) {
            return definition.getBeanClassName();
        }
    }

    /***
     * 由于proguard混淆貌似不能指定在basePackages下面类名混淆后唯一，不同包名经常有a.class，b.class,c.class之类重复的类名，因此spring容器初始化bean的时候会报错。
     *庆幸的是，我们可以通过改变spring的bean的命名策略来解决这个问题，把包名带上，就唯一了
     * @param args
     */
    public static void main(String[] args) {
        new SpringApplicationBuilder(App.class)
                .beanNameGenerator(new CustomGenerator())
                .run(args);
    }
}
```

### 实体类一定要保留

```
解释：这里的实体类包括各种Entity,Dto等。保留的原因有：
1. Mybatis的XML的ResultType需要实体类的全路径
2. Jackson需要序列化，字段混淆前端会找不到
```

### Controller一定要保留

```
解释：Controller混淆了前端找不到请求路径，模板引擎例如thymeleaf找不到路径
```

### SpringBoot JavaConfig配置不能混淆

```java
//解释：例如下面的JavaConfig配置，混淆后配置出错
@Data
@ConfigurationProperties(prefix = "wx.mp")
public class WxMpProperties {
    private List<MpConfig> configs;

    @Data
    public static class MpConfig {
        private String appId;
  }
}
```

**其他配置参考以上的注释，其他照抄修改成自己的项目对应路径即可，使用JD-GUI反编译查看效果。**