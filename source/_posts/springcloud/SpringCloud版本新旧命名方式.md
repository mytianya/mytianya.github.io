
---
title: SpringCloud版本新旧命名方式
date: 2023-09-26 02:00:18
tags: springcloud
categories: 
- java
---
## 看看SpringCloud已发布版本

当前已发布SpringCloud稳定版本见下图，在线查看地址:[https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-dependencies](https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-dependencies)

![springcloud-version](/img/springcloud-version.png)

## SpringCloud新旧命名方式

- ~~采用版本名+版本号，其中版本名采用伦敦地铁站命名，其中按照地铁首字母A-Z依次命令如Hoxton.SR9~~。但是现在已更改为主版本号.次版本号.修订号如2020.0.0

- 旧版本命名方式中,开发的快照版本(BUILD-SNAPSHOT)到里程碑版本(M),开发的差不多到会发布的候选发布版(RELEASE),最后到正式版(SR)版本。
- 新版本命名是`YYYY.MINOR.MICRO[-MODIFIER]`，拿**2020.0.1-SNAPSHOT** 这个版本来说，其中YYYY为年份全称、MINOR为辅助版本号、MICRO为补丁版本号。MODIFIER同上述修饰关键节点，BUILD-SNAPSHOT、里程碑M等。

 ## SpringCloud与SpringBoot版本对应关系

浏览器访问https://start.spring.io/actuator/info，在返回的数据中找到spring-cloud即可以查看SpringCloud与SpringBoot版本对应关系。

![springcloud-springboot-version](/img/springcloud-springboot-version.png)

## SpringCloud版本选择

- 根据上面的原则首先选择SpringCloud要与SpringBoot版本能够对应的上
- SpringCloud应该优先选择SR的版本
- 采用官网推荐版本，每一个SpringCloud的文档均有推荐SpringBoot版本。如https://docs.spring.io/spring-cloud/docs/2020.0.0/reference/html/
- 本人公司的SpringBoot框架使用的2.1.0.RELEASE,SpringCloud选择是Greenwich.SR4(有历史原因)。本系列教程**SpringCloud采用Hoxton.SR9**,**SpringBoot采用 2.3.5.RELEASE。**

**Springcloud企业项目实战问题汇总系列教程**,所有资料和源码均在本人GitHub上，[https://github.com/mytianya/springcloud-tutorials](https://github.com/mytianya/springcloud-tutorials)