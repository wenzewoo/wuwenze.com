+++
title = "Dataway::后端接口开发或将迎来新的变革？"
date = "2020-04-29 17:08:00"
url = "archives/691"
tags = ["Java","Dataway"]
categories = ["后端"]
+++

### 概述 ###

你是否厌烦了面向CRUD编程？近期逛Github又发现了一个神奇的开源项目\[[zycgit/hasor][zycgit_hasor]\]，其中有个模块甚是诱人，这里摘抄一段其官网的描述：

> Dataway 是基于 DataQL 服务聚合能力，为应用提供的一个接口配置工具。使得使用者无需开发任何代码就配置一个满足需求的接口。 整个接口配置、测试、冒烟、发布。一站式都通过 Dataway 提供的 UI 界面完成。UI 会以 Jar 包方式提供并集成到应用中并和应用共享同一个 http 端口，应用无需单独为 Dataway 开辟新的管理端口.

简单来说，在面向简单的数据库模型编程时，你几乎可以进行0开发即可进行发布接口，相当多的基于数据库的需求仅需要配置就可以完成，避免了从数据存取到前端接口之间的一系列开发任务，例如：Mapper、BO、VO、DO、DAO、Service、Controller 这些统统不在需要了！

很神奇吧？本文结合其官方文档，搭建一个helloworld试试看，文档地址：[https://www.hasor.net/web/dataway/about.html][https_www.hasor.net_web_dataway_about.html]

### 在SpringBoot项目中引入依赖 ###

```xml
<dependencies>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <scope>compile</scope>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>net.hasor</groupId>
        <artifactId>hasor-spring</artifactId>
        <version>4.1.3</version>
    </dependency>
    <dependency>
        <groupId>net.hasor</groupId>
        <artifactId>hasor-dataway</artifactId>
        <version>4.1.3-fix20200414</version>
    </dependency>
</dependencies>
```

如上所见，仅需引入 hasor 和 jdbc 相关的依赖即可，无需引入任何ORM框架。  
hasor-spring 负责 Spring 和 Hasor 框架之间的整合。hasor-dataway 是工作在 Hasor 之上，利用 hasor-spring 我们就可以使用 dataway了。

### 提供Dataway基础配置 ###

Dataway需要提供一个集成的WebUI，且需要连接咱们的业务数据库，所以需要做一些相关的配置以及初始化数据库。

修改application.yml

```yaml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/dataway
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver


### 是否启用 Dataway 功能（必选：默认false）
HASOR_DATAQL_DATAWAY: true
### 是否开启 Dataway 后台管理界面（必选：默认false）
HASOR_DATAQL_DATAWAY_ADMIN: true
### dataway  API工作路径（可选，默认：/api/）
HASOR_DATAQL_DATAWAY_API_URL: /api/
### dataway-ui 的工作路径（可选，默认：/interface-ui/）
HASOR_DATAQL_DATAWAY_UI_URL: /interface-ui/
### SQL执行器方言设置（可选，建议设置）
HASOR_DATAQL_FX_PAGE_DIALECT: mysql
```

如上可见，Dataway提供的配置并不是所有都需要配置的，很多是有默认值的，如果不需要修改，则无需配置。  
另外，Dataway需要两个数据库表来支撑运行，其SQL脚本可以在依赖jar包中 “META-INF/hasor-framework/mysql” 目录下面找到：

```sql
CREATE TABLE `interface_info` (
                                  `api_id`          int(11)      NOT NULL AUTO_INCREMENT   COMMENT 'ID',
                                  `api_method`      varchar(12)  NOT NULL                  COMMENT 'HttpMethod：GET、PUT、POST',
                                  `api_path`        varchar(512) NOT NULL                  COMMENT '拦截路径',
                                  `api_status`      int(2)       NOT NULL                  COMMENT '状态：0草稿，1发布，2有变更，3禁用',
                                  `api_comment`     varchar(255)     NULL                  COMMENT '注释',
                                  `api_type`        varchar(24)  NOT NULL                  COMMENT '脚本类型：SQL、DataQL',
                                  `api_script`      mediumtext   NOT NULL                  COMMENT '查询脚本：xxxxxxx',
                                  `api_schema`      mediumtext       NULL                  COMMENT '接口的请求/响应数据结构',
                                  `api_sample`      mediumtext       NULL                  COMMENT '请求/响应/请求头样本数据',
                                  `api_create_time` datetime     DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
                                  `api_gmt_time`    datetime     DEFAULT CURRENT_TIMESTAMP COMMENT '修改时间',
                                  PRIMARY KEY (`api_id`)
) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8mb4 COMMENT='Dataway 中的API';

CREATE TABLE `interface_release` (
                                     `pub_id`          int(11)      NOT NULL AUTO_INCREMENT   COMMENT 'Publish ID',
                                     `pub_api_id`      int(11)      NOT NULL                  COMMENT '所属API ID',
                                     `pub_method`      varchar(12)  NOT NULL                  COMMENT 'HttpMethod：GET、PUT、POST',
                                     `pub_path`        varchar(512) NOT NULL                  COMMENT '拦截路径',
                                     `pub_status`      int(2)       NOT NULL                  COMMENT '状态：0有效，1无效（可能被下线）',
                                     `pub_type`        varchar(24)  NOT NULL                  COMMENT '脚本类型：SQL、DataQL',
                                     `pub_script`      mediumtext   NOT NULL                  COMMENT '查询脚本：xxxxxxx',
                                     `pub_script_ori`  mediumtext   NOT NULL                  COMMENT '原始查询脚本，仅当类型为SQL时不同',
                                     `pub_schema`      mediumtext       NULL                  COMMENT '接口的请求/响应数据结构',
                                     `pub_sample`      mediumtext       NULL                  COMMENT '请求/响应/请求头样本数据',
                                     `pub_release_time`datetime     DEFAULT CURRENT_TIMESTAMP COMMENT '发布时间（下线不更新）',
                                     PRIMARY KEY (`pub_id`)
) ENGINE=InnoDB AUTO_INCREMENT=0 DEFAULT CHARSET=utf8mb4 COMMENT='Dataway API 发布历史。';

create index idx_interface_release on interface_release (pub_api_id);
```

![100425\_d3a5abac\_955503][100425_d3a5abac_955503]

另外，为了演示业务系统数据，这里我们建立一个简单的用户表，用于后面生成该数据库的对外API接口：

```sql
create table tb_user
(
    id int auto_increment,
    username VARCHAR(12) not null,
    password VARCHAR(32) not null,
    nickname VARCHAR(20) null,
    constraint tb_user_pk
        primary key (id)
);

create unique index tb_user_username_uindex
    on tb_user (username);

INSERT INTO tb_user (id, username, password, nickname) VALUES (1, 'admin', '123456', 'Administrator');
INSERT INTO tb_user (id, username, password, nickname) VALUES (2, 'guest', '123456', 'Guest User');
```

### 将Spring容器中的数据源安装到Hasor容器 ###

Spring Boot 和 Hasor本是两个独立的容器框架，我们做整合之后为了使用 Dataway 的能力需要把 Spring 中的数据源设置到 Hasor 中。

首先新建一个 Hasor 的 模块，并且将其交给 Spring 管理。然后把数据源通过 Spring 注入进来。

```java
package com.wuwenze.dataway.config;

import lombok.RequiredArgsConstructor;
import net.hasor.core.ApiBinder;
import net.hasor.core.DimModule;
import net.hasor.db.JdbcModule;
import net.hasor.db.Level;
import net.hasor.spring.SpringModule;
import org.springframework.stereotype.Component;

import javax.sql.DataSource;

@DimModule
@Component
@RequiredArgsConstructor
public class DatawayModule implements SpringModule {
    private final DataSource dataSource;

    @Override
    public void loadModule(final ApiBinder apiBinder) throws Throwable {
        apiBinder.installModule(new JdbcModule(Level.Full, this.dataSource));
    }
}
```

### 在SpringBoot启动器中启用Hasor ###

```java
package com.wuwenze.dataway;

import net.hasor.spring.boot.EnableHasor;
import net.hasor.spring.boot.EnableHasorWeb;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@EnableHasor
@EnableHasorWeb
@SpringBootApplication
public class DatawayExampleApplication {

    public static void main(final String[] args) {
        SpringApplication.run(DatawayExampleApplication.class, args);
    }

}
```

这一步非常简单，只需要在 Spring 启动类上增加两个注解即可。

### 项目结构 ###

到这里，咱们的开发工作就算是完成了！目前的项目结构如下，出乎意料的简单吧？

```bash
├── dataway-example
│   ├── pom.xml
│   ├── src
│   │   ├── main
│   │   │   ├── java
│   │   │   │   └── com
│   │   │   │       └── wuwenze
│   │   │   │           └── dataway
│   │   │   │               ├── DatawayExampleApplication.java
│   │   │   │               └── config
│   │   │   │                   └── DatawayModule.java
│   │   │   └── resources
│   │   │       └── application.yml
```

### 启动项目 ###

```bash
.   ____          _            __ _ _
 /\ / ___'_ __ _ _(_)_ __  __ _    
( ( )___ | '_ | '_| | '_ / _` |    
 \/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |___, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::        (v2.2.6.RELEASE)

2020-04-29 10:58:22.669  INFO 1961 --- [           main] c.w.dataway.DatawayExampleApplication    : Starting DatawayExampleApplication on macOS.local with PID 1961 (/Users/wuwenze/Development/other-projects/dataway-example/target/classes started by wuwenze in /Users/wuwenze/Development/other-projects/dataway-example)
2020-04-29 10:58:22.673  INFO 1961 --- [           main] c.w.dataway.DatawayExampleApplication    : No active profile set, falling back to default profiles: default
2020-04-29 10:58:23.868  INFO 1961 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2020-04-29 10:58:23.877  INFO 1961 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2020-04-29 10:58:23.877  INFO 1961 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.33]
2020-04-29 10:58:23.948  INFO 1961 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2020-04-29 10:58:23.948  INFO 1961 --- [           main] o.s.web.context.ContextLoader            : Root WebApplicationContext: initialization completed in 1212 ms
2020-04-29 10:58:24.002  INFO 1961 --- [           main] n.h.spring.boot.BasicHasorConfiguration  : 
 _    _                        ____              _
| |  | |                      |  _             | |
| |__| | __ _ ___  ___  _ __  | |_) | ___   ___ | |_
|  __  |/ _` / __|/ _ | '__| |  _ < / _  / _ | __|
| |  | | (_| __  (_) | |    | |_) | (_) | (_) | |_
|_|  |_|__,_|___/___/|_|    |____/ ___/ ___/ __|

2020-04-29 10:58:24.008  INFO 1961 --- [           main] net.hasor.core.Hasor                     : runMode at Full ,runPath at /Users/wuwenze/Development/other-projects/dataway-example
2020-04-29 10:58:24.013  INFO 1961 --- [           main] n.h.c.setting.StandardContextSettings    : addConfig '/META-INF/hasor-framework/core-hconfig.xml' in 'jar:file:/Users/wuwenze/.m2/repository/net/hasor/hasor-core/4.1.3/hasor-core-4.1.3.jar!/META-INF/hasor.schemas'
2020-04-29 10:58:24.015  INFO 1961 --- [           main] n.h.c.setting.StandardContextSettings    : addConfig '/META-INF/hasor-framework/dataway-hconfig.xml' in 'jar:file:/Users/wuwenze/.m2/repository/net/hasor/hasor-dataway/4.1.3-fix20200414/hasor-dataway-4.1.3-fix20200414.jar!/META-INF/hasor.schemas'
2020-04-29 10:58:24.015  INFO 1961 --- [           main] n.h.c.setting.StandardContextSettings    : addConfig '/META-INF/hasor-framework/web-hconfig.xml' in 'jar:file:/Users/wuwenze/.m2/repository/net/hasor/hasor-web/4.1.3/hasor-web-4.1.3.jar!/META-INF/hasor.schemas'
2020-04-29 10:58:24.016  INFO 1961 --- [           main] n.h.c.setting.StandardContextSettings    : addConfig '/META-INF/hasor-framework/db-hconfig.xml' in 'jar:file:/Users/wuwenze/.m2/repository/net/hasor/hasor-db/4.1.3/hasor-db-4.1.3.jar!/META-INF/hasor.schemas'
2020-04-29 10:58:24.016  INFO 1961 --- [           main] n.h.c.setting.StandardContextSettings    : addConfig '/META-INF/hasor-framework/dataql-hconfig.xml' in 'jar:file:/Users/wuwenze/.m2/repository/net/hasor/hasor-dataql/4.1.3/hasor-dataql-4.1.3.jar!/META-INF/hasor.schemas'
2020-04-29 10:58:24.017  INFO 1961 --- [           main] n.h.c.setting.StandardContextSettings    : addConfig '/META-INF/hasor-framework/dataql-fx-hconfig.xml' in 'jar:file:/Users/wuwenze/.m2/repository/net/hasor/hasor-dataql-fx/4.1.3/hasor-dataql-fx-4.1.3.jar!/META-INF/hasor.schemas'
2020-04-29 10:58:24.066  INFO 1961 --- [           main] n.h.c.environment.AbstractEnvironment    : loadPackages = com.*, net.*, net.hasor.*, net.hasor.dataql.*, net.hasor.dataql.fx.*, net.hasor.dataway.*, net.hasor.db.*, net.hasor.web.*, org.*
2020-04-29 10:58:24.186  INFO 1961 --- [           main] n.hasor.core.context.TemplateAppContext  : loadModule class net.hasor.dataway.config.DatawayModule
2020-04-29 10:58:24.187  INFO 1961 --- [           main] net.hasor.dataway.config.DatawayModule   : dataway api workAt /api/
2020-04-29 10:58:24.187  INFO 1961 --- [           main] n.h.c.environment.AbstractEnvironment    : var -> HASOR_DATAQL_DATAWAY_API_URL = /api/.
2020-04-29 10:58:24.198  INFO 1961 --- [           main] net.hasor.dataway.config.DatawayModule   : dataway admin workAt /interface-ui/
2020-04-29 10:58:24.212  INFO 1961 --- [           main] net.hasor.core.binder.ApiBinderWrap      : mapingTo[1c50731177244247ac1441f1f634a5a5] -> bindType ‘class net.hasor.dataway.web.ApiDetailController’ mappingTo: ‘[/interface-ui/api/api-detail]’.
2020-04-29 10:58:24.213  INFO 1961 --- [           main] net.hasor.core.binder.ApiBinderWrap      : mapingTo[d9afbf784e0c4f6595186b804d06d3bb] -> bindType ‘class net.hasor.dataway.web.ApiHistoryListController’ mappingTo: ‘[/interface-ui/api/api-history]’.
2020-04-29 10:58:24.214  INFO 1961 --- [           main] net.hasor.core.binder.ApiBinderWrap      : mapingTo[16129bae92034be895725980365dd4ac] -> bindType ‘class net.hasor.dataway.web.ApiInfoController’ mappingTo: ‘[/interface-ui/api/api-info]’.
2020-04-29 10:58:24.216  INFO 1961 --- [           main] net.hasor.core.binder.ApiBinderWrap      : mapingTo[4c3673447d5442de98d526014ae446e7] -> bindType ‘class net.hasor.dataway.web.ApiListController’ mappingTo: ‘[/interface-ui/api/api-list]’.
2020-04-29 10:58:24.217  INFO 1961 --- [           main] net.hasor.core.binder.ApiBinderWrap      : mapingTo[cc435c1893a542079775368b114be013] -> bindType ‘class net.hasor.dataway.web.ApiHistoryGetController’ mappingTo: ‘[/interface-ui/api/get-history]’.
2020-04-29 10:58:24.219  INFO 1961 --- [           main] net.hasor.core.binder.ApiBinderWrap      : mapingTo[e2f37f74e1c142c4a5cc1dae6c01c25f] -> bindType ‘class net.hasor.dataway.web.DisableController’ mappingTo: ‘[/interface-ui/api/disable]’.
2020-04-29 10:58:24.220  INFO 1961 --- [           main] net.hasor.core.binder.ApiBinderWrap      : mapingTo[f3fb338f50be4c9483032ab17662cab3] -> bindType ‘class net.hasor.dataway.web.SmokeController’ mappingTo: ‘[/interface-ui/api/smoke]’.
2020-04-29 10:58:24.221  INFO 1961 --- [           main] net.hasor.core.binder.ApiBinderWrap      : mapingTo[f5b3df846533478484769d7e7c651bd1] -> bindType ‘class net.hasor.dataway.web.SaveApiController’ mappingTo: ‘[/interface-ui/api/save-api]’.
2020-04-29 10:58:24.222  INFO 1961 --- [           main] net.hasor.core.binder.ApiBinderWrap      : mapingTo[f50231f5f9ee4786b72d26442d72dca6] -> bindType ‘class net.hasor.dataway.web.PublishController’ mappingTo: ‘[/interface-ui/api/publish]’.
2020-04-29 10:58:24.225  INFO 1961 --- [           main] net.hasor.core.binder.ApiBinderWrap      : mapingTo[8bd64d9c80c34df4b7e04c5ec396e63d] -> bindType ‘class net.hasor.dataway.web.PerformController’ mappingTo: ‘[/interface-ui/api/perform]’.
2020-04-29 10:58:24.226  INFO 1961 --- [           main] net.hasor.core.binder.ApiBinderWrap      : mapingTo[16101d6573984fdabf030a67f97b08a8] -> bindType ‘class net.hasor.dataway.web.DeleteController’ mappingTo: ‘[/interface-ui/api/delete]’.
2020-04-29 10:58:24.231  INFO 1961 --- [           main] n.hasor.core.context.TemplateAppContext  : loadModule class net.hasor.web.render.RenderWebPlugin
2020-04-29 10:58:24.233  INFO 1961 --- [           main] n.hasor.core.context.TemplateAppContext  : loadModule class net.hasor.core.exts.startup.StartupModule
2020-04-29 10:58:24.233  INFO 1961 --- [           main] n.hasor.core.context.TemplateAppContext  : loadModule class net.hasor.core.exts.aop.AopModule
2020-04-29 10:58:24.236  INFO 1961 --- [           main] n.hasor.core.context.TemplateAppContext  : loadModule class net.hasor.dataql.fx.FxModule
2020-04-29 10:58:24.240  INFO 1961 --- [           main] n.hasor.core.context.TemplateAppContext  : loadModule class com.wuwenze.dataway.config.DatawayModule
2020-04-29 10:58:24.242  INFO 1961 --- [           main] n.h.core.context.TemplateAppContext$1    : installModule ->net.hasor.db.JdbcModule@1886991b
2020-04-29 10:58:24.250  INFO 1961 --- [           main] n.hasor.core.context.TemplateAppContext  : loadModule class net.hasor.spring.boot.BasicHasorConfiguration$$Lambda$393/1859216983
2020-04-29 10:58:24.288  INFO 1961 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
2020-04-29 10:58:24.524  INFO 1961 --- [           main] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
2020-04-29 10:58:24.534  INFO 1961 --- [           main] net.hasor.dataway.config.DatawayModule   : dataway dbMapping MySQL to default
2020-04-29 10:58:24.537  INFO 1961 --- [           main] net.hasor.web.render.RenderWebPlugin     : RenderPlugin init -> useLayout=false, layoutPath=/layout, templatePath=/templates, placeholder=content_placeholder, defaultLayout=default.html
2020-04-29 10:58:24.576  INFO 1961 --- [sor-EventPool-1] n.hasor.core.context.TemplateAppContext  : Hasor StartCompleted!
2020-04-29 10:58:24.617  INFO 1961 --- [           main] net.hasor.web.startup.RuntimeListener    : ServletContext Attribut is net.hasor.core.AppContext
2020-04-29 10:58:24.622  INFO 1961 --- [           main] net.hasor.web.invoker.InvokerContext     : mapingTo -> type ‘class net.hasor.dataway.web.ApiDetailController’ mappingTo: ‘/interface-ui/api/api-detail’.
2020-04-29 10:58:24.622  INFO 1961 --- [           main] net.hasor.web.invoker.InvokerContext     : mapingTo -> type ‘class net.hasor.dataway.web.ApiHistoryListController’ mappingTo: ‘/interface-ui/api/api-history’.
2020-04-29 10:58:24.622  INFO 1961 --- [           main] net.hasor.web.invoker.InvokerContext     : mapingTo -> type ‘class net.hasor.dataway.web.ApiInfoController’ mappingTo: ‘/interface-ui/api/api-info’.
2020-04-29 10:58:24.622  INFO 1961 --- [           main] net.hasor.web.invoker.InvokerContext     : mapingTo -> type ‘class net.hasor.dataway.web.ApiListController’ mappingTo: ‘/interface-ui/api/api-list’.
2020-04-29 10:58:24.622  INFO 1961 --- [           main] net.hasor.web.invoker.InvokerContext     : mapingTo -> type ‘class net.hasor.dataway.web.ApiHistoryGetController’ mappingTo: ‘/interface-ui/api/get-history’.
2020-04-29 10:58:24.622  INFO 1961 --- [           main] net.hasor.web.invoker.InvokerContext     : mapingTo -> type ‘class net.hasor.dataway.web.DisableController’ mappingTo: ‘/interface-ui/api/disable’.
2020-04-29 10:58:24.622  INFO 1961 --- [           main] net.hasor.web.invoker.InvokerContext     : mapingTo -> type ‘class net.hasor.dataway.web.SmokeController’ mappingTo: ‘/interface-ui/api/smoke’.
2020-04-29 10:58:24.622  INFO 1961 --- [           main] net.hasor.web.invoker.InvokerContext     : mapingTo -> type ‘class net.hasor.dataway.web.SaveApiController’ mappingTo: ‘/interface-ui/api/save-api’.
2020-04-29 10:58:24.622  INFO 1961 --- [           main] net.hasor.web.invoker.InvokerContext     : mapingTo -> type ‘class net.hasor.dataway.web.PublishController’ mappingTo: ‘/interface-ui/api/publish’.
2020-04-29 10:58:24.622  INFO 1961 --- [           main] net.hasor.web.invoker.InvokerContext     : mapingTo -> type ‘class net.hasor.dataway.web.PerformController’ mappingTo: ‘/interface-ui/api/perform’.
2020-04-29 10:58:24.622  INFO 1961 --- [           main] net.hasor.web.invoker.InvokerContext     : mapingTo -> type ‘class net.hasor.dataway.web.DeleteController’ mappingTo: ‘/interface-ui/api/delete’.
2020-04-29 10:58:24.646  INFO 1961 --- [           main] net.hasor.web.startup.RuntimeFilter      : RuntimeFilter started, at Apache Tomcat/9.0.33
2020-04-29 10:58:24.926  INFO 1961 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2020-04-29 10:58:25.188  INFO 1961 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2020-04-29 10:58:25.192  INFO 1961 --- [           main] c.w.dataway.DatawayExampleApplication    : Started DatawayExampleApplication in 3.328 seconds (JVM running for 4.527)
```

应用在启动过程中会看到 Hasor Boot 的欢迎信息，这就算是启动成功了。

### 使用管理界面进行接口发布 ###

在浏览器中访问 “[http://127.0.0.1:8080/interface-ui/”][http_127.0.0.1_8080_interface-ui] 就可以看到期待已久的界面了。  
![110049\_a38f2d3a\_955503][110049_a38f2d3a_955503]

接下来，我们来新建一个接口，查询所有的用户列表：  
![112411\_07c3a25e\_955503][112411_07c3a25e_955503]  
如上图所示，我们通过SQL模式，新建了一个GET接口，/api/userList，使用查询语句：

```sql
select id,username,nickname from tb_user;
```

接下来我们依次点击，保存、测试、冒烟测试，通过后，点击发布按钮，该接口即发布成功了！  
浏览器访问 [http://127.0.0.1:8080/api/userList][http_127.0.0.1_8080_api_userList] 即可看到接口正常工作了，真是太方便了有没有呢？  
![112632\_6b9c6817\_955503][112632_6b9c6817_955503]

除了简单的SQL语句外，还支持DataQL，这种特定的查询语法需要单独学习，如果有必要，也可以去尝试一下：  
上面的接口，我们可以切换为DataQL模式，修改查询如下：

```bash
var query = @@sql()<%
    select id,username,nickname from tb_user
%>
return query()
```

其中 var query = @@sql()<% ... %> 是用来定义SQL外部代码块，并将这个定义存入 query 变量名中。<% %> 中间的就是 SQL 语句

更多的我们就不研究了，毕竟官网写得还是很详细且全部是中文文档（国产开源项目又争光了）

### 参考链接 ###

 *  Dataway 官方手册：[https://www.hasor.net/web/dataway/about.html][https_www.hasor.net_web_dataway_about.html]
 *  DataQL 手册地址：[https://www.hasor.net/web/dataql/what\_is\_dataql.html][https_www.hasor.net_web_dataql_what_is_dataql.html]
 *  Hasor 项目的首页：[https://www.hasor.net/web/index.html][https_www.hasor.net_web_index.html]

### 类似框架 ###

在Github看到这个项目时，关注的人并不是很多，虽然想法很新潮，但是可能需要更长的时间沉淀，才敢应用于生产环境  
除此之外，还有一些类似思想的实现框架，比如：

 *  GraphQL - [https://graphql.cn/][https_graphql.cn] 由Facebook维护，稳定性有保障。
 *  APIJSON - [http://apijson.org/][http_apijson.org] 与Dataway一样，属于国产框架，关注的人并不是很多，但值得研究。


[zycgit_hasor]: https://github.com/zycgit/hasor
[https_www.hasor.net_web_dataway_about.html]: https://www.hasor.net/web/dataway/about.html
[100425_d3a5abac_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20200429/f75fdcaa-ec11-4924-9eab-2dc6900b748c.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[http_127.0.0.1_8080_interface-ui]: http://127.0.0.1:8080/interface-ui/%E2%80%9D
[110049_a38f2d3a_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20200429/3cacac9f-d960-4046-879a-45b1e1b496d5.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[112411_07c3a25e_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20200429/9184d519-312d-4eaf-8b2e-40d588041fb4.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[http_127.0.0.1_8080_api_userList]: http://127.0.0.1:8080/api/userList
[112632_6b9c6817_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20200429/b9d1d9f5-463b-4ec4-a25b-92e31ac33046.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[https_www.hasor.net_web_dataql_what_is_dataql.html]: https://www.hasor.net/web/dataql/what_is_dataql.html
[https_www.hasor.net_web_index.html]: https://www.hasor.net/web/index.html
[https_graphql.cn]: https://graphql.cn/
[http_apijson.org]: http://apijson.org/