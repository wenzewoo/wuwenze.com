+++
title = "MockMvc WebFilter不生效问题解决"
date = "2019-06-23 16:40:00"
url = "archives/624"
tags = ["单元测试","Spring","MockMvc"]
categories = ["后端"]
+++

在SpringBoot项目中，配置了一个`@WebFilter`，正常启动没问题，但是通过`MockMvc`进行单元测试死活不生效

```kotlin
@Configuration
@Order(1)
@WebFilter(urlPatterns = ["/**"])
class HttpServletRequestWrapperFilter : Filter {
    override fun doFilter(req: ServletRequest?, resp: ServletResponse?, chain: FilterChain?) {
        when (req) {
            is HttpServletRequest -> chain?.doFilter(MyHttpServletRequestWrapper(req), resp)
            else -> chain?.doFilter(req, resp)
        }
    }
}
```

后来搞了半天，原来在构建`MockMvc`对象时，需要手动添加过滤器，这坑爹的玩意儿。

```kotlin
@Before
fun setUp() {
    mockMvc = MockMvcBuilders.webAppContextSetup(webApplicationContext)
        .addFilters<DefaultMockMvcBuilder>(HttpServletRequestWrapperFilter())
        .build()
}
```