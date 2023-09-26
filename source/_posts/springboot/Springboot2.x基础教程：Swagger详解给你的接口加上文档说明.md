---
title: Springboot2.x基础教程：Swagger详解给你的接口加上文档说明
date: 2023-09-23 21:54:08
tags: springboot
categories: 
- java
---
> 相信无论是前端还是后端开发，都或多或少地被接口文档折磨过。前端经常抱怨后端给的接口文档与实际情况不一致。后端又觉得编写及维护接口文档会耗费不少精力，经常来不及更新。其实无论是前端调用后端，还是后端调用后端，都期望有一个好的接口文档。
> SpringBoot集成Swagger能够通过很简单的注解把接口描述清楚，生成可视化文档页面。
> 原生的Swagger-ui界面很粗糙，这里用[knife4j-spring-ui](https://doc.xiaominfo.com/knife4j/ "knife4j-spring-ui")替代。

## 一个好的HTTP接口文档描述

1. 写清楚接口的请求路径:  QueryPath: /user/login
2. 写清楚接口的请求方法类型： GET/POST/DELETE/PUT
3. 写清楚接口的业务含义,使用场景
4. 写清楚接口的入参：参数描述、参数类型、参数结构、参数是否必传
5. 写清楚接口的返回类型：返回的数据结构，异常状况

## SpringBoot集成Swagger

### 项目引入依赖

```xml
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger2</artifactId>
            <version>2.9.2</version>
        </dependency>
        <dependency>
            <groupId>io.springfox</groupId>
            <artifactId>springfox-swagger-ui</artifactId>
            <version>2.9.2</version>
        </dependency>
        <dependency>
            <groupId>com.github.xiaoymin</groupId>
            <artifactId>knife4j-spring-ui</artifactId>
            <version>2.0.4</version>
        </dependency>
```

### SpringBoot关于Swagger配置

把此Swagger配置粘入项目即可

```java
@EnableSwagger2
@Configuration
public class SwaggerConfig implements WebMvcConfigurer {
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                //这里改成自己的接口包名
                .apis(RequestHandlerSelectors.basePackage("vip.codehome.springboot.tutorials.controller"))
                .paths(PathSelectors.any())
                .build();
    }
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("SpringBoot教程接口文档")//标题
                .description("使用swagger文档管理接口")//描述
                .contact(new Contact("codehome", "", "dsyslove@163.com"))//作者信息
                .version("1.0.0")//版本号
                .build();
    }
    //解决doc.html，swagger-ui.html 404问题
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/**").addResourceLocations(
                "classpath:/static/");
        registry.addResourceHandler("swagger-ui.html").addResourceLocations(
                "classpath:/META-INF/resources/");
        registry.addResourceHandler("doc.html").addResourceLocations(
                "classpath:/META-INF/resources/");
        registry.addResourceHandler("/webjars/**").addResourceLocations(
                "classpath:/META-INF/resources/webjars/");

    }
}
```

### Swagger的具体使用

#### 各个注解的作用

1. @Api 放在类上介绍类的作用
2. @ApiOperation 放在方法上介绍方法的作用
3. @ApiImplicitParam介绍入参说明
4. @ApiResponse介绍返回状态
5. @ApiModel、@ApiModelProperty介绍入参是对象，返回是对象字段说明

#### 代码示例

```java
@RestController
@Api(tags = "Swagger注解测试类")
public class SwaggerUserController {
    @ApiOperation(value = "这是一个echo接口")
    @ApiImplicitParams({
            @ApiImplicitParam(name = "msg",value = "请求的msg参数",required = true,paramType = "query"),
            @ApiImplicitParam(name = "token",value = "请求的token",required = false,paramType ="header" )
    })
    @ApiResponses({
            @ApiResponse(code=200,message = "请求成功"),
            @ApiResponse(code=400,message="请求无权限")
    })
    @GetMapping("/echo")
    public String echo(String msg,@RequestHeader(name = "token") String token){
        return msg;
    }
    @PostMapping("/login")
    public R<UserInfoVO> login(@RequestBody LoginDTO loginDTO){
        UserInfoVO userInfoVO=new UserInfoVO();
        userInfoVO.setNickname("编程之家");
        userInfoVO.setToken("xxx");
        return R.ok(userInfoVO);
    }
}
@Data
@ApiModel
public class LoginDTO {
    @ApiModelProperty(value = "用户账号或者邮箱")
    String account;
    @ApiModelProperty(value = "用户密码")
    String passwd;
    @ApiModelProperty(value = "用户密码")
    String verifyCode;
}
```

#### 接口文档效果

这里访问的是http://localhost:8080/doc.html,knife4j-spring-ui还有相比原生还有更多强大的功能，大家自行发现。
**千里之行，始于足下。这里是SpringBoot教程系列第二篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**

![swagger-ui](/img/swagger-ui.png)