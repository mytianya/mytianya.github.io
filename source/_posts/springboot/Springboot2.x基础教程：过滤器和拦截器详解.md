
---
title: Springboot2.x基础教程：过滤器和拦截器详解
date: 2023-09-23 22:00:08
tags: springboot
categories: 
- java
---
>在springboot web项目开发过程中，我们通常需要对请求与响应的内容请求拦截处理，如进行请求日志记录、UA检查、用户权限验证、非法内容过滤等功能，这时候过滤器与拦截器就派上用场。
>本文带大家讲解springboot如何使用过滤器与拦截器以及两者之间的区别。
## 过滤器
Servlet 过滤器可以动态地拦截请求和响应，以变换或使用包含在请求或响应中的信息。过滤器是一个实现了 javax.servlet.Filter 接口的 Java 类。javax.servlet.Filter 接口定义了三个方法：
```java
public interface Filter {
    default void init(FilterConfig filterConfig) throws ServletException {
    }
    void doFilter(ServletRequest var1, ServletResponse var2, FilterChain var3) throws IOException, ServletException;
    default void destroy() {
    }
}
```

| 序号 | 方法&描叙                                                    |
| ---- | ------------------------------------------------------------ |
| 1    | **doFilter**<br/>该方法完成实际的过滤操作，当客户端请求方法与过滤器设置匹配的URL时，Servlet容器将先调用过滤器的doFilter方法。FilterChain用户访问后续过滤器。 |
| 2    | **init**<br/>web 应用程序启动时，web 服务器将创建Filter 的实例对象，并调用其init方法，读取web.xml配置，完成对象的初始化功能，从而为后续的用户请求作好拦截的准备工作（filter对象只会创建一次，init方法也只会执行一次）。开发人员通过init方法的参数，可获得代表当前filter配置信息的FilterConfig对象。 |
| 3    | **destroy**<br/>Servlet容器在销毁过滤器实例前调用该方法，在该方法中释放Servlet过滤器占用的资源。 |

## SpringBoot使用过滤器
定义了一个简单的过滤器
```java
@Slf4j
public class LogFilter implements Filter {
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
         HttpServletRequest req=(HttpServletRequest)servletRequest;
         log.info(req.getRequestURI());
         filterChain.doFilter(servletRequest,servletResponse);
    }

    @Override
    public void destroy() {

    }
}
```
使用FilterRegistrationBean注册过滤器
```java
@Configuration
public class LogFilterConfiguration {
    @Bean
    public FilterRegistrationBean registrationBean(){
        FilterRegistrationBean registrationBean=new FilterRegistrationBean();
        registrationBean.setFilter(new LogFilter());
        //匹配的过滤器
        registrationBean.addUrlPatterns("/*");
        //过滤器名称
        registrationBean.setName("logFilter");
        //过滤器顺序
        registrationBean.setOrder(1);
        return registrationBean;
    }
}
```

使用Servlet3.0注解定义过滤器

```java
@WebFilter(urlPatterns = "/*",filterName = "authFiler")
@Slf4j
public class AuthFilter implements Filter {
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain chain) throws IOException, ServletException {
        log.info("进行权限校验.........");
        chain.doFilter(servletRequest,servletResponse);
    }

}
```



两种方式的区别：

1. WebFilter这个注解并没有指定执行顺序的属性，其执行顺序依赖于Filter的名称，是根据Filter类名（注意不是配置的filter的名字）的字母顺序倒序排列
2. @WebFilter指定的过滤器优先级都高于FilterRegistrationBean配置的过滤器
3. FilterRegistrationBean方式可以注入SpringBoot IOC容器中的Bean

## 拦截器

SpringBoot拦截器Interceptor类似 **面向切面编程** 中的 **切面** 和 **通知**，我们通过 **动态代理** 对一个 `service()` 方法添加 **通知** 进行功能增强。比如说在方法执行前进行 **初始化处理**，在方法执行后进行 **后置处理**。**拦截器** 的思想和 `AOP` 类似，区别就是 **拦截器** 只能对 `Controller` 的 `HTTP` 请求进行拦截。

```java
public interface HandlerInterceptor {
    default boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        return true;
    }

    default void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable ModelAndView modelAndView) throws Exception {
    }

    default void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable Exception ex) throws Exception {
    }
}
```
| 序号 | 方法&描叙                                                    |
| ---- | ------------------------------------------------------------ |
| 1    | **preHandle**<br/>该方法在controller接收请求处理request之前执行，返回值为 boolean，返回值为 true 时接着执行 postHandle() 和 afterCompletion() 方法；如果返回false则中断执行。 |
| 2    | **postHandle**<br/> 在 controller 处理请求之后， ModelAndView 处理前执行，可以对 响应结果 进行修改。 |
| 3    | **afterCompletion**<br/>在 DispatchServlet 对本次请求处理完成，即生成 ModelAndView 之后执行。 |

定义一个简单的拦截器
```java
@Slf4j
public class LogHandler implements HandlerInterceptor {
    private NamedThreadLocal<Long> startTimeThreadLocal = new NamedThreadLocal<>("StopWatch-StartTime");

    public LogHandler() {
        super();
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        long beginTime = System.currentTimeMillis();//1、开始时间      
        startTimeThreadLocal.set(beginTime);//线程绑定变量（该数据只有当前请求的线程可见）
        return true;//继续流程
    }

    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, @Nullable ModelAndView modelAndView) throws Exception {
    }

    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        long endTime = System.currentTimeMillis();
        long beginTime = startTimeThreadLocal.get();//得到线程绑定的局部变量（开始时间）  
        long consumeTime = endTime - beginTime;
        //3、消耗的时间         
        log.info(String.format("%s consume %d millis", request.getRequestURI(), consumeTime));
    }    
}
```
注册拦截器
```java
@Configuration
public class HandlerConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new LogHandler());
    }
}
```
## 过滤器与拦截器区别
| 序号 | 区别                                                         |
| ---- | ------------------------------------------------------------ |
| 1    | Filter是servlet规范，使用范围是web程序，拦截器不限于web程序，也可以用于Application、Swing程序中 |
| 2    | Filter是servlet规范中定义，是servlet容器支持的。拦截器是Spring容器内，是spring框架支持的 |
| 3    | 拦截器是Spring的一个组件，能够使用spring中对象，如Service对象、数据源、事务管理、通过IOC注入容器即可，filter则不能 |
| 4    | filter在servlet前后起作用，拦截器能够深入方法的前后，异常抛出前后。 |
| 5    | 在springboot项目中一般优先使用拦截器                         |

**千里之行，始于足下。这里是SpringBoot教程系列第十二篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**