+++
title = "使用MockMvc在SpringBoot中进行单元测试"
date = "2018-08-11 07:09:00"
url = "archives/407"
tags = ["单元测试","Spring"]
categories = ["测试","后端"]
+++

在开发好常规的RESTful接口后，难免会依次进行单元测试，一般来说使用`Postman`即可, 但是依然是不太方便，有没有更方便，更优雅的方式呢？

### MockMvc ###

> `org.springframework.test.web.servlet.MockMvc`

MockMvc是由Spring提供的，作用是在单元测试代码中，伪造一套MVC环境，常见的方法如下：

| Method              | Remark                                                                                                                                                    |
|---------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------|
| perform             | 执行一个RequestBuilder请求，会自动执行SpringMVC的流程并映射到相应的控制器执行处理；                                                                                                     |
| get/post/delete/put | 声明发送一个get请求的方法。MockHttpServletRequestBuilder get(String urlTemplate, Object... urlVariables)：根据uri模板和uri变量值得到一个GET请求方式的。另外提供了其他的请求的方法，如：post、put、delete等。 |
| param               | 添加request的参数                                                                                                                                              |
| content             | 添加requestBody的参数                                                                                                                                          |
| contentType         | 设置contentType属性                                                                                                                                           |
| header              | 设置header属性                                                                                                                                                |
| andExpect           | 添加ResultMatcher验证规则，验证控制器执行完成后结果是否正确（对返回的数据进行的判断）；                                                                                                        |
| andDo               | 添加ResultHandler结果处理器，比如调试时打印结果到控制台（对返回的数据进行的判断）；                                                                                                          |
| andReturn           | 最后返回相应的MvcResult；然后进行自定义验证/进行下一步的异步处理（对返回的数据进行的判断）；                                                                                                       |


### 代码实现 ###

现有以下两个接口，/user/login, /user/logout

```java
@Slf4j
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private ApiContext apiContext;

    @Autowired
    private UserService userService;

    @Autowired
    private TokenService tokenService;

    @PostMapping("/login")
    private JSONEntity<?> login() {
        String requestBodyString = WebUtil.getBody();
        Assert.notEmpty(requestBodyString, "requestBody is null");
        String userJsonString = Base64.decodeStr(requestBodyString);
        if (!JSONUtil.isJsonObj(userJsonString)) {
            throw new BusinessException(JSONEntity.Error.ILLEGAL_ARGUMENT_ERROR);
        }
        User requestBody = JSON.parseObject(userJsonString, User.class);
        if (null == requestBody) {
            throw new BusinessException(JSONEntity.Error.JSON_SERIALIZATION_ERROR);
        }
        try {
            User userDb = userService.login(requestBody.getUsername(), requestBody.getPassword());
            Token token = tokenService.findOrCreateToken(userDb.getId());
            return JSONEntity.ok("token", token.getToken(), "expire", token.getExpire());
        } catch (AuthorityException e) {
            log.error("#sign_in fail. username={}, e=" + e, requestBody.getUsername());
            return JSONEntity.error(e.getMessage());
        }
    }

    @CheckAccessToken
    @DeleteMapping("/logout")
    private JSONEntity<?> logout() {
        Boolean flag = tokenService.removeToken(apiContext.getToken());
        return flag ? JSONEntity.ok() : JSONEntity.error("注销失败");
    }
}
```

创建单元测试

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = BlogApplication.class)
public class BaseTester {
    protected MockMvc mockMvc;

    @Autowired
    protected WebApplicationContext webApplicationContext;

    @Before
    public void setUp() throws Exception {
        mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext).build();
    }
}

@Slf4j
public class UserControllerTest extends BaseTester {

    @Test
    public void testLogin() throws Exception {
        Map<String, String> requestBodyMap = MapUtil.newHashMap(//
                "username", "admin",//
                "password", SecureUtil.md5("password@"));
        log.info("#testLogin requestBodyMap={}", requestBodyMap);
        String requestBody = Base64.encode(JSON.toJSONString(requestBodyMap));
        log.info("#testLogin requestBody={}", requestBody);
        mockMvc.perform(
                MockMvcRequestBuilders
                        .post("/user/login")
                        .contentType(MediaType.APPLICATION_JSON_UTF8)
                        .content(requestBody)
        ).andDo(mvcResult -> log.info("#testLogin mvcResult = {}", mvcResult.getResponse().getContentAsString()));
    }

    @Test
    public void testLogout() throws Exception {
        String token = "eb33ef4b50d04a288c3c531fab2a32ed";
        mockMvc.perform(
                MockMvcRequestBuilders
                        .delete("/user/logout")
                        .contentType(MediaType.APPLICATION_JSON_UTF8)
                        .header(Consts.ACCESS_TOKEN_HEADER_KEY, token)
        ).andDo(mvcResult -> log.info("#testLogout mvcResult = {}", mvcResult.getResponse().getContentAsString()));
    }
}
```

结果打印：

```bash
2018-08-10 11:34:38.461  INFO 18848 --- [           main] c.w.blog.controller.UserControllerTest   : ##testLogin requestBodyMap={password=321f39d6a69ad68534417d4205b7f52c, username=admin}
2018-08-10 11:34:38.469  INFO 18848 --- [           main] c.w.blog.controller.UserControllerTest   : ##testLogin requestBody=eyJwYXNzd29yZCI6IjMyMWYzOWQ2YTY5YWQ2ODUzNDQxN2Q0MjA1YjdmNTJjIiwidXNlcm5hbWUiOiJhZG1pbiJ9
2018-08-10 11:34:38.968  INFO 18848 --- [           main] c.w.blog.controller.UserControllerTest   : ##testLogin mvcResult = {
    "status":true,
    "result":{
        "expire":"2018/08/11 11:34:10",
        "token":"a6e71e9b8f0848c8a029934ee975cccc"
    },
    "timestamp":1533872078964
}
2018-08-10 11:34:38.972  INFO 18848 --- [           main] o.s.b.t.m.w.SpringBootMockServletContext : Initializing Spring FrameworkServlet ''
2018-08-10 11:34:38.973  INFO 18848 --- [           main] o.s.t.web.servlet.TestDispatcherServlet  : FrameworkServlet '': initialization started
2018-08-10 11:34:38.978  INFO 18848 --- [           main] o.s.t.web.servlet.TestDispatcherServlet  : FrameworkServlet '': initialization completed in 5 ms
2018-08-10 11:34:39.150  WARN 18848 --- [           main] com.wuwenze.blog.util.MapUtil            : MapUtil#newMap() keyValues is null.
2018-08-10 11:34:39.151  INFO 18848 --- [           main] c.w.blog.controller.UserControllerTest   : ##testLogout mvcResult = {
    "status":true,
    "timestamp":1533872079151
}
```

在以上的例子中，我使用了 `andDo` 结果处理器，通过打印接口返回值的方式，确定了接口是可以正常工作的。