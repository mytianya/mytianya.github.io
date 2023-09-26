---
title: Springboot2.x基础教程：接口实现统一格式返回
date: 2023-09-23 21:53:09
tags: springboot
categories: 
- java
---
>在SpringBoot项目中经常会涉及到前后端数据的交互，目前比较流行的是基于 json 格式的数据交互。但是 json 只是消息的格式，其中的内容还需要我们自行设计。不管是 HTTP 接口还是 RPC 接口保持返回值格式统一很重要，这将大大降低 前后端联调的成本。

## 定义的接口具体格式
```
{
	#返回状态码
	"code":integer类型,
	#返回信息描述
	"msg":string,
	#返回值
	"data":object
}
```

## 提供的R类型工具类
```java
@Data
public class R<T> {
    private int code;
    private T data;
    private String msg;
    public R() {
    }
    public static <T> R<T> ok(T data) {
        return fill(data,ApiCommonCodeEnum.OK);
    }
    public static <T> R<T> failed(String msg) {
        return fill( null, ApiCommonCodeEnum.FAIL);
    }
    public static <T> R<T> failed(ApiCommonCodeEnum apiEnum) {
        return fill( null, apiEnum);
    }
    public static <T> R<T> fill(T data, ApiCommonCodeEnum apiEnum) {
        return fill(apiEnum.getCode(),data,apiEnum.getMsg());
    }
    public static <T> R<T> fill(int code,T data,String msg) {
        R R = new R();
        R.setCode(code);
        R.setData(data);
        R.setMsg(msg);
        return R;
    }
}
//使用举例
@PostMapping("/login")
public R<UserInfoVO> login(@RequestBody LoginDTO loginDTO){
	UserInfoVO userInfoVO=new UserInfoVO();
	userInfoVO.setNickname("编程之家");
	userInfoVO.setToken("xxx");
	return R.ok(userInfoVO);
}
```

## 状态码设计
这里我们可以参考Http状态码设计，每种业务类型定义一个状态码区间，抽出放在枚举类中

![](/img/http_status_code.jpg)

```java
public enum  ApiCommonCodeEnum {
    FAIL(1,"调用出错"),
    OK(0,"调用成功"),
	USER_EXISIT(100,"用户不存在"),
	USER_PASSWORD_ERROR(101,"用户密码错误"),
	//....
	NO_PERM(300,"无权限");
	//....
    int code;
    String msg;
    ApiCommonCodeEnum(int code,String msg){
        this.code=code;
        this.msg=msg;
    }
    public int getCode() {
        return code;
    }
    private void setCode(int code) {
        this.code = code;
    }
    public String getMsg() {
        return msg;
    }
    private void setMsg(String msg) {
        this.msg = msg;
    }
}
```
**千里之行，始于足下。这里是SpringBoot教程系列第二篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**