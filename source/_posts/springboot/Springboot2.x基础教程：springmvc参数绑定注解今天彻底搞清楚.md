
---
title: Springboot2.x基础教程：springmvc参数绑定注解今天彻底搞清楚
date: 2023-09-23 21:58:08
tags: springboot
categories: 
- java
---
>在编写SpringBoot项目中我们通常在Controller层使用@RequestParam、@RequestBody等注解接收前端请求参数。
>我们应该怎么使用各种注解，这片文章带大家把springmvc参数绑定使用彻底搞清楚。
## Http请求报文
HTTP协议定义Web客户端如何从Web服务器请求Web页面，以及服务器如何把Web页面传送给客户端。
客户端向服务器发送一个请求报文，请求报文包含：
![http_head](
/img/http_head.png)

### 请求方法
1. 在Springboot中请求方法最常用的是GET或者POST方法，其他的HEAD、PUT一般不会使用
2. GET通常用于查询某个资源，POST通常用于新增、更新、删除资源操作。
3. GET一般利用URL传递参数，POST利用请求体。
4. URL的一般浏览器设定最大长度为1k,POST利用的请求体的无数据限制
5. **在Http协议中没有规定GET不能利用请求体发送数据，实际上我们是可以这样做的。但是因为有些服务器实现可能直接把GET的请求体内容忽略掉，所以一般我们不会这样使用**

### 请求URL
URL,全称是UniformResourceLocator, 中文叫统一资源定位符,是互联网上用来标识某一处资源的地址。
```
scheme:[//[user[:password]@]host[:port]][/path][?query][#fragment]
```
```
http://www.example.com:80/path/to/myfile.html?key1=value1&key2=value2#SomewhereInTheDocument
```
![url-format](/img/url_format.png)

### 请求头部
请求头部由关键字/值对组成，每行一对，关键字和值用英文冒号“:”分隔。请求头部通知服务器有关于客户端请求的信息。
```
:authority: www.cnblogs.com
:method: GET
content-type: application/json; charset=utf-8
cookie: _ga=GA1.2.1704734765.1586779730; _gid=GA1.2.846324607.1598838524
pragma: no-cache
referer: https://www.cnblogs.com/
sec-fetch-dest: empty
sec-fetch-mode: cors
sec-fetch-site: same-origin
user-agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.83 Safari/537.36
x-requested-with: XMLHttpRequest
```
### 请求数据
协议规定post提交的数据，必须包含在消息主体中entity-body中，但是协议并没有规定数据使用什么编码方式。开发者可以自己决定消息主体的格式,服务端会根据content-type字段来获取参数是怎么编码的，然后对应去解码
#### 常见的ContentType
##### 1、application/x-www-form-urlencoded
浏览器的原生form表单提交的数据按照 key1=val1&key2=val2 的方式进行编码，key和val都进行了URL转码
##### 2、multipart/form-data
常见的 POST 数据提交的方式。我们使用表单上传文件时，必须让 form 的 enctype 等于这个值。
##### 3、application/json
消息主体是序列化后的 JSON 字符串
##### 4、text/xml
使用 HTTP 作为传输协议，XML 作为编码方式的远程调用规范

## SpringMVC参数绑定
### @PathVariable注解详解
@PathVariable 是用来获得请求url中的动态参数的，可以将URL中的变量映射到功能处理方法的参数上，其中URL 中的 {xxx} 占位符可以通过@PathVariable(“xxx“) 绑定到操作方法的入参中。
```
@GetMapping("/user/{id}")
@ResponseBody
public R userInfo(@PathVariable("id")String id){
    return  R.ok(id);
}
```
![image-20210530224844709](/img/springmvc1.png)

### @RequestHeader注解详解
@RequestHeader 注解，可以把Request请求header部分的值绑定到方法的参数上。
```
@GetMapping("/useragent")
@ResponseBody
public R getHeader(@RequestHeader("User-Agent")String userAgent){
   //...省略
}
```
![image-20210530224900794](/img/springmvc2.png)

### @CookieValue注解详解
@CookieValue 可以把Request header中关于cookie的值绑定到方法的参数上。
```
@GetMapping("/cookie")
@ResponseBody
public R getCookie(@CookieValue("token")String token){
    //...省略
}
```
![image-20210530224922785](
/img/cookie-value.png)

### @RequestParam详解
@RequestParam注解用来处理Content-Type: 为 application/x-www-form-urlencoded编码的内容，提交方式为get或post。
@RequestParam注解实质是将Request.getParameter()中的Key-Value参数Map利用Spring的转化机制ConversionService配置，转化成参数接收对象或字段，RequestParam可以获取的到**get方式中queryString的值，和post方式中body data的值**。
```
@RequestMapping("/reqparam")
@ResponseBody
public R requsetParam(@RequestParam Map<String,Object> params) {
    return R.ok(params);
}
```
#### get请求@RequestParam获取到了url上的参数：
![image-20210530224949318](
/img/springmvc-get.png)

#### post请求@RequestParam获取到了url上的参数和请求实体的参数:
![springmvc-post](/img/springmvc-post.png)
#### @RequestParam同样可以用来处理Content-Type：为form/data的内容，通常用于文件上传
```
@RequestMapping("/upload")
@ResponseBody
public R requsetParam(@RequestParam("files") MultipartFile file,@RequestParam Map<String,Object> params) {
    params.put("files",file.getOriginalFilename());
    return R.ok(params);
}
```
![springmvc-file](/img/springmvc-file.png)

### @RequestBody详解
@RequestBody注解用来处理HttpEntity（请求体）传递过来的数据，一般用来处理非Content-Type: application/x-www-form-urlencoded编码格式的数据
#### @RequestBody获取Json数据
```
@RequestMapping("/json")
@ResponseBody
public R json(@RequestBody UserDO userDO){
    return R.ok(userDO);
}
```
![req-body](/img/req-body.png)

#### @RequestBody获取xml数据
```
@RequestMapping(value = "/xml",consumes = "application/xml",produces ="application/xml",method = RequestMethod.POST)
@ResponseBody
public UserDO xml(@RequestBody UserDO userDO){
    return userDO;
}
```
![req-xml](/img/req-xml.png)
**千里之行，始于足下。这里是SpringBoot教程系列第十六篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**