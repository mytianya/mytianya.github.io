
---
title: springboot项目打包瘦身
date: 2023-09-25 02:00:58
tags: springboot
categories: 
- java
---
>   默认情况下，**Spring Boot** 项目发布时会将项目代码和项目的所有依赖文件一起打成一个可执行的 **jar** 包。但如果项目的依赖包很多，那么这个文件就会非常大。这样每次即使只改动一点东西，就需要将整个项目重新打包部署，我们将依赖 **lib** 从项目分离出来，这样每次部署只需要发布项目源码即可。

## 瘦身打包配置

springboot默认使用**spring-boot-maven-plugin** 来打包，这个插件会将项目所有的依赖打入项目**jar** 包里面，将打包插件替换为 **maven-jar-plugin**，并拷贝依赖到 **jar** 到外面的 **lib** 目录。

```xml
<build>
    <plugins>
        <!-- 指定启动类，将依赖打成外部jar包 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <configuration>
                <archive>
                    <!-- 生成的jar中，不要包含pom.xml和pom.properties这两个文件 -->
                    <addMavenDescriptor>false</addMavenDescriptor>
                    <manifest>
                        <!-- 是否要把第三方jar加入到类构建路径 -->
                        <addClasspath>true</addClasspath>
                        <!-- 外部依赖jar包的最终位置 -->
                        <classpathPrefix>lib/</classpathPrefix>
                        <!-- 项目启动类 -->                       <mainClass>vip.codehome.springboot.tutorials.SpringbootTutorialsApplication</mainClass>
                    </manifest>
                </archive>
            </configuration>
        </plugin>
        <!--拷贝依赖到jar外面的lib目录-->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-dependency-plugin</artifactId>
            <executions>
                <execution>
                    <id>copy-lib</id>
                    <phase>package</phase>
                    <goals>
                        <goal>copy-dependencies</goal>
                    </goals>
                    <configuration>
                        <outputDirectory>target/lib</outputDirectory>
                        <excludeTransitive>false</excludeTransitive>
                        <stripVersion>false</stripVersion>
                        <includeScope>runtime</includeScope>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

​	项目打包时会在target目录生成lib依赖包跟项目jar包，部署时将项目 **jar** 包以及 **lib** 文件夹上传到服务器上，使用java -jar 命令启动即可。如果后续仅仅修改了项目代码，只需上传替换项目 **jar** 包。

![spring-package](/img/spring-package.png)

**千里之行，始于足下。这里是SpringBoot教程系列第十八篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**