---
title: Springboot2.x基础教程：jsr303接口参数校验,结合统一异常拦截
date: 2023-09-23 21:56:08
tags: springboot
categories: 
- java
---
>JSR是Java Specification Requests的缩写，意思是Java 规范提案。是指向JCP(Java Community Process)提出新增一个标准化技术规范的正式请求。任何人都可以提交JSR，以向Java平台增添新的API和服务。JSR已成为Java界的一个重要标准。
>JSR-303 是JAVA EE 6 中的一项子规范，叫做Bean Validation，Hibernate Validator 是 Bean Validation 的参考实现 . Hibernate Validator 提供了 JSR 303 规范中所有内置 constraint 的实现，除此之外还有一些附加的 constraint。如果想了解更多有关 Hibernate Validator 的信息，请查看 http://www.hibernate.org/subprojects/validator.html
>SpringBoot项目开发的后端接口，常常会对参数是否为空、参数长度进行基本的校验，这里SpringBoot能够很方便的集成JSR303，通过注解很方便的验证接口参数，并且可以自定义验证规则。

## JSR303注解介绍
### JSR303自带约束
![jsr303-annotation](/img/jsr303-annotation.png)
### Hibernate Validator附加的约束
![hibernate-annotation](/img/hibernate-annotation.jpg)
## SpringBoot项目集成
### 项目引入依赖
```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
```
### Hibernate Validator介绍
Hibernate Validator的校验模式有两种情况
1. 快速失败（验证失败一个，快速返回错误信息）
2. 普通模式（默认，会校验完所有的属性，然后返回所有的验证失败信息）
我们一般使用第一种，SpringBoot需要开启配置
```java
/**
 * 配置Jsr303 hibernate validator快速失败模式
 */
@Configuration
public class Jsr303Config {
    @Bean
    public Validator validator(){
        ValidatorFactory validatorFactory = Validation
                .byProvider( HibernateValidator.class )
                .configure()
                .failFast( true )
                .buildValidatorFactory();
        return validatorFactory.getValidator();
    }
}
```
### 基本使用
在方法上使用@Valid注解开启验证，使用BindingResult接收验证结果
```java
@Data
public class LoginDTO {
    @Size(min = 8,message = "账号长度大于8")
    String account;
    @NotBlank(message = "密码不能为空")
    String passwd;
}
    @PostMapping("/logon")
    public R logon(@Valid LoginDTO loginDTO, BindingResult result){
        check(result);
        return R.ok("");
    }
    public static void check(BindingResult result){
        StringBuffer sb=new StringBuffer();
        if(result.hasErrors()){
            for (ObjectError error : result.getAllErrors()) {
                sb.append(error.getDefaultMessage());
            }
            ExUtil.throwBusException(sb.toString());
        }
    }
```
### Jsr303配合统一异常使用
在项目中我通常要定义属于业务的统一异常类，这里验证失败后组装自定义的验证信息，抛出自定义的BusinessException异常。
```java
public class BusinessException extends RuntimeException {

    private static final long serialVersionUID = 1L;
    private int code = ApiCommonCodeEnum.FAIL.getCode();
    public BusinessException(String message) {
        super(message);
    }
	//防止篇幅过长省略其他代码
}
//异常工具类
public class ExUtil {
    public static void throwBusException(String msg){
        throw new BusinessException(msg);
    }
	//......
}
```
**然后被统一异常拦截，返回前面定义的R接口,失败信息包含在msg字段中。
贴出统一异常拦截部分代码**
```java
@Component
@Slf4j
public class ExceptionResolver implements HandlerExceptionResolver {
    @Override
    public ModelAndView resolveException(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Object o, Exception ex) {
        if(ex instanceof BusinessException){
            R result= R.fill(((BusinessException) ex).getCode(),null,ex.getMessage());
            outputJSON(httpServletResponse, Charsets.UTF_8.toString(), JsonUtil.toJson(result));
            return null;
        }else {
            R result=R.failed(ex.getMessage());
            outputJSON(httpServletResponse, Charsets.UTF_8.toString(), JsonUtil.toJson(result));
            return null;
        }
    }
    private void outputJSON(HttpServletResponse response, String charset, String jsonStr) {
        PrintWriter out = null;
        try {
            if (response != null) {
                response.setCharacterEncoding(charset);
                response.setContentType("text/html;charset=" + charset);
                response.setHeader("Pragma", "No-cache");
                response.setHeader("Cache-Control", "no-cache");
                response.setDateHeader("Expires", 0);
                out = response.getWriter();
                out.print(jsonStr);
            }
        } catch (Exception e) {
            log.error(ExUtil.getSimpleMessage(e));
        } finally {
            if (out != null) {
                out.close();
            }
        }
    }
}
```
### 接口测试效果
![jsr-test1](/img/jsr-test1.jpg)
**千里之行，始于足下。这里是SpringBoot教程系列第四篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**