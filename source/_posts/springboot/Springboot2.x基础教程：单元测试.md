---
title: Springboot2.x基础教程：单元测试
date: 2023-09-23 21:56:08
tags: springboot
categories: 
- java
---
>单元测试用于测试单个代码组件，并确保代码按预期方式工作。单元测试由开发人员编写和执行。大多数情况下，会使用JUnit或TestNG这样的测试框架。测试用例通常在方法级别编写，并通过自动化执行。
>Spring Boot提供了一些注解和工具去帮助开发者测试他们的应用。
>在讲springboot单元测试之前，先简单介绍下软件测试的类型(从开发角度来说)，跟如何写好一个单元测试。
## 软件测试类型
1. 单元测试：用于测试单个代码组件，并确保代码按预期方式工作。单元测试由开发人员编写和执行。
2. 集成测试: 检查整个系统是否工作正常。集成测试也是由开发人员完成的，但它不是测试单个组件，而是旨在跨组件进行测试。系统由许多单独的组件组成，如代码、数据库、Web服务器等。集成测试能够发现组件的连接、网络访问、数据库问题等问题。
3. 功能测试: 通过将给定输入的结果与规范进行比较来检查每个特性是否正确实现。这个阶段通常由公司专门的测试团队来负责。
4. 验收测试：验收测试是保证软件的准备就绪，按照项目合同、任务书、双方约定的验收依据文档，向软件的购买者展示该软件的原始的需求。

## 单元测试要点
1. 测试的粒度是方法级的。
2. 测试的用例的结果应该是稳定的。
3. 测试的用例应该尽量少写逻辑或者不写测试逻辑。
4. 测试用例应该有很高的覆盖率，把基本的输入输出覆盖,一些边界也要覆盖到。
5. 使用断言而不是输出打印的语句。
6. 单元测试用例应该有个好名字，例如test_MethodName()

## SpringBoot集成单元测试
### 引入依赖
```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-test</artifactId>
	<scope>test</scope>
</dependency>
```
### 依赖关系
1.	JUnit — The de-facto standard for unit testing Java applications.
2.	Spring Test & Spring Boot Test — Utilities and integration test support for Spring Boot applications.
3.	AssertJ — A fluent assertion library.
4.	Hamcrest — A library of matcher objects (also known as constraints or predicates).
5.	Mockito — A Java mocking framework.
6.  JsonPath — XPath for JSON

![](/img/junit-test.jpg)

### 常用注解说明
- @RunWith(SpringRunner.class)
JUnit运行使用Spring的测试支持。SpringRunner是SpringJUnit4ClassRunner的新名字，这样做的目的仅仅是为了让名字看起来更简单一点。
- @SpringBootTest
用于Spring Boot应用测试，它默认会根据包名逐级往上找，一直找到Spring Boot主程序，通过类注解是否包含@SpringBootApplication来判断是否主程序，并在测试的时候启动该类来创建Spring上下文环境。
- @BeforeClass
针对所有测试，只执行一次，且必须为static void
- @BeforeEach
初始化方法，执行当前测试类的每个测试方法前执行
- @Test
测试方法，在这里可以测试期望异常和超时时间
- @AfterEach
释放资源，执行当前测试类的每个测试方法后执行
- @AfterClass
针对所有测试，只执行一次，且必须为static void
- @Ignore
忽略的测试方法

### 断言常见使用方法
1. assertNotNull("message",A) //判断A不为空
2. assertFalse("message",A) //判断条件A不为真
3. assertTure("message",A) //判断条件A为真
4. assertEquals("message",A,B) // 判断A.equals(B)
5. assertSame("message",A,B) //判断A==B
### 测试方法
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class UserRepositoryTest  {
    @Autowired
    UserRepository userRepository;
    @Test
    @Ignore
    public void testFindAll(){
        Page<UserDO> userDOS= userRepository.findAll(PageRequest.of(1,10));
        Assert.assertNotNull(userDOS.getContent());
    }
    @Test(expected = RuntimeException.class)
    public void testNullPointerException(){
        throw new RuntimeException();
    }
}
```
### 测试API接口
- MockMvcRequestBuilders
构造请求的路径，请求的参数等信息
- andExpect
添加断言判断结果是否达到预期
- andDo
添加结果处理器，比如示例中的打印
- andReturn
返回验证成功后的MvcResult,用于自定义验证/下一步的异步处理。
```java
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {
    @Autowired
    MockMvc mockMvc;
    UserDO userDO;
    MultiValueMap<String,String> params;
    @BeforeEach
    public void setUp()throws Exception{
        userDO=new UserDO();
        userDO.setPasswd("123456");
        params=new LinkedMultiValueMap<>();
        params.add("name","codehome");
    }
	//测试get接口
    @Test
   public  void queryUser() throws Exception {
      String result= mockMvc.perform(MockMvcRequestBuilders.get("/user/query")
            .contentType(MediaType.APPLICATION_FORM_URLENCODED)
                .params(params)
        ).andExpect(MockMvcResultMatchers.status().is2xxSuccessful())
        .andDo(MockMvcResultHandlers.print())
                .andReturn().getResponse()
                .getContentAsString();
        Assert.assertEquals("调用成功","codehome",result);
    }
	//测试post接口
    @Test
    void addUser() throws Exception {
        mockMvc.perform(MockMvcRequestBuilders.post("/user/add")
            .contentType(MediaType.APPLICATION_JSON)
                .content(JsonUtil.toJson(userDO))
                .accept(MediaType.APPLICATION_JSON)
        ).andExpect(MockMvcResultMatchers.status().is2xxSuccessful())
                .andDo(MockMvcResultHandlers.print())
                .andExpect(MockMvcResultMatchers.jsonPath("$.data.passwd").value("123456"));
    }
	//测试cookie
    @Test
    void testCookie()throws Exception{
       String token= mockMvc.perform(MockMvcRequestBuilders.get("/user/cookie")
        .cookie(new Cookie("token","123456")))
                .andDo(MockMvcResultHandlers.print())
                .andReturn().getResponse()
                .getContentAsString();
       Assert.assertEquals("token从cookie中获取成功","123456",token);
    }
}
//测试的接口类
@RestController
@RequestMapping("/user")
public class UserController {
    @GetMapping("/query")
    public String queryUser(String name){
        return name;
    }
    @PostMapping("/add")
    public R addUser(@RequestBody UserDO userDO){
        return R.ok(userDO);
    }
    @GetMapping("/cookie")
    public String testCookie(@CookieValue("token") String token){
        return token;
    }
}
```
**千里之行，始于足下。这里是SpringBoot教程系列第七篇，所有项目源码均可以在我的[GitHub](https://github.com/mytianya/springboot-tutorials "GitHub")上面下载源码。**