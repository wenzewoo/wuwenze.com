+++
title = "Swagger2在SpringBoot中的集成指南"
date = "2019-01-28 07:12:00"
url = "archives/410"
tags = ["Spring","Swagger"]
categories = ["后端"]
+++

#### 引入依赖Swagger2及Swagger2 UI ####

```xml
<!-- swagger -->
<dependency>
  <groupId>io.springfox</groupId>
  <artifactId>springfox-swagger2</artifactId>
  <version>2.7.0</version>
</dependency>
<!-- swagger-ui -->
<dependency>
  <groupId>io.springfox</groupId>
  <artifactId>springfox-swagger-ui</artifactId>
  <version>2.7.0</version>
</dependency>
```

#### 构建配置文件（基于JavaConfig） ####

```java
package com.wuwenze.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.service.Contact;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;

/**
 * @author wuwenze
 * @date 2019/1/28
 */
@Configuration
@EnableSwagger2
public class SwaggerConfig {

  @Bean
  public Docket createRestApi() {
    return new Docket(DocumentationType.SWAGGER_2)
        .apiInfo(apiInfo())
        .select()
        .apis(RequestHandlerSelectors.basePackage("com.wuwenze.api"))
        .paths(PathSelectors.any())
        .build();
  }

  private ApiInfo apiInfo() {
    return new ApiInfoBuilder()
        .title("某电商平台在线API文档")
        .description("这是详细描述详细描述详细描述详细描述详细描述详细描述")
        .contact(new Contact("吴汶泽", "https://wuwenze.com",
            "wenzewoo@gmail.com"))
        .version("1.0")
        .build();
  }
}
```

#### 定义需要发布文档接口 ####

```java
package com.wuwenze.api;

import cn.hutool.core.util.RandomUtil;
import com.wuwenze.entity.Token;
import com.wuwenze.support.mvc.entity.ResultBody;
import io.swagger.annotations.Api;
import io.swagger.annotations.ApiImplicitParam;
import io.swagger.annotations.ApiImplicitParams;
import io.swagger.annotations.ApiOperation;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RestController;

/**
 * @author wuwenze
 * @date 2019/1/28
 */
@RestController
@RequestMapping("/UserApi")
@Api(value = "UserApi", description = "用户相关接口")
public class UserApi {

  @ApiOperation(value = "用户登陆", notes = "登陆成功后返回Token")
  @ApiImplicitParams({
      @ApiImplicitParam(name = "username", value = "用户名", required = true, paramType = "query", dataType = "String"),
      @ApiImplicitParam(name = "password", value = "密码", required = true, paramType = "query", dataType = "String")
  })
  @RequestMapping(value = "/login", method = RequestMethod.POST)
  ResultBody<Token> login(String username, String password) {
    return ResultBody.ok(
        Token.builder().token(RandomUtil.simpleUUID()).build()
    );
  }
}
```

### 国际化实现 ###

 *  国际化文件位于springfox-swagger-ui-2.7.0.jar/META-INF/resources/webjars/springfox-swagger-ui/lang目录中
 *  实现国际化(用自己的swagger-ui.html 代替原生swagger-ui.html)
    
     *  新建 src/main/resources/META-INF/resources/swagger-ui.html 文件
     *  把swaager-ui jar包中 /META-INF/resources/swagger-ui.html的内容复制进去
     *  在新建的swagger-ui.html补充插入以下js：

```html
<!--国际化操作：选择中文版 -->
<script src='webjars/springfox-swagger-ui/lang/translator.js' type='text/javascript'></script>
<script src='webjars/springfox-swagger-ui/lang/zh-cn.js' type='text/javascript'></script>
```

 *  访问[http://localhost:8080/swagger-ui.html][http_localhost_8080_swagger-ui.html] 就是开启中文的UI界面

![33732ee0-3fa4-4a8f-b2d1-9c2f4d1b1b8d.jpeg][]


[http_localhost_8080_swagger-ui.html]: http://localhost:8080/swagger-ui.html
[33732ee0-3fa4-4a8f-b2d1-9c2f4d1b1b8d.jpeg]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20190128/33732ee0-3fa4-4a8f-b2d1-9c2f4d1b1b8d.jpeg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg