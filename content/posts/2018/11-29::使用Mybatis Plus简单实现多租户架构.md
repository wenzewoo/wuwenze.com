+++
title = "使用Mybatis Plus简单实现多租户架构"
date = "2018-11-29 07:51:00"
url = "archives/258"
tags = ["MyBatis","SaaS"]
categories = ["后端"]
+++

在进行多租户架构`(Multi-tenancy)`实现之前，先了解一下相关的定义吧：

### 什么是多租户 ###

多租户技术或称多重租赁技术，简称`SaaS`，是一种软件架构技术，是实现如何在多用户环境下（此处的多用户一般是面向企业用户）共用相同的系统或程序组件，并且可确保各用户间数据的隔离性。

简单讲：在一台服务器上运行单个应用实例，它为多个租户（客户）提供服务。从定义中我们可以理解：多租户是一种架构，目的是为了让多用户环境下使用同一套程序，且保证用户间数据隔离。那么重点就很浅显易懂了，多租户的重点就是同一套程序下实现多用户数据的隔离。

### 数据隔离方案 ###

多租户在数据存储上存在三种主要的方案，分别是：

#### 独立数据库 ####

即一个租户一个数据库，这种方案的用户数据隔离级别最高，安全性最好，但成本较高。

 *  优点：为不同的租户提供独立的数据库，有助于简化数据模型的扩展设计，满足不同租户的独特需求；如果出现故障，恢复数据比较简单。
 *  缺点：增多了数据库的安装数量，随之带来维护成本和购置成本的增加。

#### 共享数据库，独立 Schema ####

多个或所有租户共享Database，但是每个租户一个Schema（也可叫做一个user）。底层库比如是：DB2、ORACLE等，一个数据库下可以有多个SCHEMA。

 *  优点：为安全性要求较高的租户提供了一定程度的逻辑数据隔离，并不是完全隔离；每个数据库可支持更多的租户数量。
 *  缺点：如果出现故障，数据恢复比较困难，因为恢复数据库将牵涉到其他租户的数据；

#### 共享数据库，共享 Schema，共享数据表 ####

即租户共享同一个Database、同一个Schema，但在表中增加TenantID多租户的数据字段。这是共享程度最高、隔离级别最低的模式。

简单来讲，即每插入一条数据时都需要有一个客户的标识。这样才能在同一张表中区分出不同客户的数据，这也是我们系统目前用到的(provider\_id)

 *  优点：三种方案比较，第三种方案的维护和购置成本最低，允许每个数据库支持的租户数量最多。
 *  缺点：隔离级别最低，安全性最低，需要在设计开发时加大对安全的开发量； 数据备份和恢复最困难，需要逐表逐条备份和还原。

### 利用MybatisPlus实现 ###

这里我们选用了第三种方案`（共享数据库，共享 Schema，共享数据表）`来实现，也就意味着，每个数据表都需要有一个租户标识`(provider_id)`

--------------------

现在有数据库表`(user)`如下：

| 字段名         | 字段类型        | 描述    |
|-------------|-------------|-------|
| id          | BIGINT(20)  | 主键    |
| provider_id | BIGINT(20)  | 服务商ID |
| name        | VARCHAR(30) | 姓名    |


将`provider_id`视为租户ID，用来隔离租户与租户之间的数据，如果要查询当前服务商的用户，SQL大致如下：

```sql
SELECT * FROM user t WHERE t.name LIKE '%Tom%' AND t.provider_id = 1;
```

试想一下，除了一些系统共用的表以外，其他租户相关的表，我们都需要不厌其烦的加上`AND t.provider_id = ?`查询条件，稍不注意就会导致数据越界，数据安全问题让人担忧。

好在有了MybatisPlus这个神器，可以极为方便的实现`多租户SQL解析器`，官方文档如下：

[http://mp.baomidou.com/guide/tenant.html][http_mp.baomidou.com_guide_tenant.html]

> 这里终于进入了正题，开始搭建一个极为简单的开发环境吧!

#### 新建SpringBoot环境 ####

> POM核心依赖如下，主要集成了MybatisPlus以及H2数据库(方便测试)

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.0.5</version>
</dependency>
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus</artifactId>
    <version>3.0.5</version>
</dependency>
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-generator</artifactId>
    <version>3.0.5</version>
</dependency>

<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
</dependency>
```

> 数据源配置(application.yml)

```yaml
spring:
  datasource:
    driver-class-name: org.h2.Driver
    schema: classpath:db/schema.sql
    data: classpath:db/data.sql
    url: jdbc:h2:mem:test
    username: root
    password: test

logging:
  level:
    com.wuwenze.mybatisplusmultitenancy: debug
```

> 对应的H2数据库初始化schema文件

```sql
##schema.sql
DROP TABLE IF EXISTS user;
CREATE TABLE user
(
    id BIGINT(20) NOT NULL COMMENT '主键',
    provider_id BIGINT(20) NOT NULL COMMENT '服务商ID',
    name VARCHAR(30) NULL DEFAULT NULL COMMENT '姓名',
    PRIMARY KEY (id)
);


##data.sql
INSERT INTO user (id, provider_id, name) VALUES (1, 1, 'Tony老师');
INSERT INTO user (id, provider_id, name) VALUES (2, 1, 'William老师');
INSERT INTO user (id, provider_id, name) VALUES (3, 2, '路人甲');
INSERT INTO user (id, provider_id, name) VALUES (4, 2, '路人乙');
INSERT INTO user (id, provider_id, name) VALUES (5, 2, '路人丙');
INSERT INTO user (id, provider_id, name) VALUES (6, 2, '路人丁');
```

#### MybatisPlus Config ####

> 基础环境搭建完成，现在开始配置MybatisPlus多租户相关的实现。

1.  核心配置：TenantSqlParser

    
```java
@Configuration
@MapperScan("com.wuwenze.mybatisplusmultitenancy.mapper")
public class MybatisPlusConfig {

    private static final String SYSTEM_TENANT_ID = "provider_id";
    private static final List<String> IGNORE_TENANT_TABLES = Lists.newArrayList("provider");

    @Autowired
    private ApiContext apiContext;

    @Bean
    public PaginationInterceptor paginationInterceptor() {
        PaginationInterceptor paginationInterceptor = new PaginationInterceptor();

        // SQL解析处理拦截：增加租户处理回调。
        TenantSqlParser tenantSqlParser = new TenantSqlParser()
                .setTenantHandler(new TenantHandler() {

                    @Override
                    public Expression getTenantId() {
                        // 从当前系统上下文中取出当前请求的服务商ID，通过解析器注入到SQL中。
                        Long currentProviderId = apiContext.getCurrentProviderId();
                        if (null == currentProviderId) {
                            throw new RuntimeException("#1129 getCurrentProviderId error.");
                        }
                        return new LongValue(currentProviderId);
                    }

                    @Override
                    public String getTenantIdColumn() {
                        return SYSTEM_TENANT_ID;
                    }

                    @Override
                    public boolean doTableFilter(String tableName) {
                        // 忽略掉一些表：如租户表（provider）本身不需要执行这样的处理。
                        return IGNORE_TENANT_TABLES.stream().anyMatch((e) -> e.equalsIgnoreCase(tableName));
                    }
                });
        paginationInterceptor.setSqlParserList(Lists.newArrayList(tenantSqlParser));
        return paginationInterceptor;
    }

    @Bean(name = "performanceInterceptor")
    public PerformanceInterceptor performanceInterceptor() {
        return new PerformanceInterceptor();
    }
}
```

1.  ApiContext

> 注意，这里为了方便，简单用Map做演示，正式代码可千万别这么干啊

```java
@Component
public class ApiContext {
    private static final String KEY_CURRENT_PROVIDER_ID = "KEY_CURRENT_PROVIDER_ID";
    private static final Map<String, Object> mContext = Maps.newConcurrentMap();

    public void setCurrentProviderId(Long providerId) {
        mContext.put(KEY_CURRENT_PROVIDER_ID, providerId);
    }

    public Long getCurrentProviderId() {
        return (Long) mContext.get(KEY_CURRENT_PROVIDER_ID);
    }
}
```

1.  Entity、Mapper

    
```java
@Data
@ToString
@Accessors(chain = true)
public class User {
    private Long id;
    private Long providerId;
    private String name;
}

public interface UserMapper extends BaseMapper<User> {

}
```

#### 单元测试 ####

> com.wuwenze.mybatisplusmultitenancy.MybatisPlusMultiTenancyApplicationTests

```java
@Slf4j
@RunWith(SpringRunner.class)
@FixMethodOrder(MethodSorters.JVM)
@SpringBootTest(classes = MybatisPlusMultiTenancyApplication.class)
public class MybatisPlusMultiTenancyApplicationTests {


    @Autowired
    private ApiContext apiContext;

    @Autowired
    private UserMapper userMapper;

    @Before
    public void before() {
        // 在上下文中设置当前服务商的ID
        apiContext.setCurrentProviderId(1L);
    }

    @Test
    public void insert() {
        User user = new User().setName("新来的Tom老师");
        Assert.assertTrue(userMapper.insert(user) > 0);

        user = userMapper.selectById(user.getId());
        log.info("#insert user={}", user);

        // 检查插入的数据是否自动填充了租户ID
        Assert.assertEquals(apiContext.getCurrentProviderId(), user.getProviderId());
    }

    @Test
    public void selectList() {
        userMapper.selectList(null).forEach((e) -> {
            log.info("#selectList, e={}", e);
            // 验证查询的数据是否超出范围
            Assert.assertEquals(apiContext.getCurrentProviderId(), e.getProviderId());
        });
    }
}
```

> 运行结果

```bash
2018-11-29 21:07:14.262  INFO 18688 --- [ main] .MybatisPlusMultiTenancyApplicationTests : Started MybatisPlusMultiTenancyApplicationTests in 2.629 seconds (JVM running for 3.904)
2018-11-29 21:07:14.554 DEBUG 18688 --- [ main] c.w.m.mapper.UserMapper.insert : ==> Preparing: INSERT INTO user (id, name, provider_id) VALUES (?, ?, 1)
2018-11-29 21:07:14.577 DEBUG 18688 --- [ main] c.w.m.mapper.UserMapper.insert : ==> Parameters: 1068129257418178562(Long), 新来的Tom老师(String)
2018-11-29 21:07:14.577 DEBUG 18688 --- [ main] c.w.m.mapper.UserMapper.insert : <== Updates: 1
 Time：0 ms - ID：com.wuwenze.mybatisplusmultitenancy.mapper.UserMapper.insert
Execute SQL：INSERT INTO user (id, name, provider_id) VALUES (?, ?, 1) {1: 1068129257418178562, 2: STRINGDECODE('u65b0u6765u7684Tomu8001u5e08')}

2018-11-29 21:07:14.585 DEBUG 18688 --- [ main] c.w.m.mapper.UserMapper.selectById : ==> Preparing: SELECT id, provider_id, name FROM user WHERE user.provider_id = 1 AND id = ?
2018-11-29 21:07:14.595 DEBUG 18688 --- [ main] c.w.m.mapper.UserMapper.selectById : ==> Parameters: 1068129257418178562(Long)
2018-11-29 21:07:14.614 DEBUG 18688 --- [ main] c.w.m.mapper.UserMapper.selectById : <== Total: 1
2018-11-29 21:07:14.615  INFO 18688 --- [ main] .MybatisPlusMultiTenancyApplicationTests : ##insert user=User(id=1068129257418178562, providerId=1, name=新来的Tom老师)
 Time：19 ms - ID：com.wuwenze.mybatisplusmultitenancy.mapper.UserMapper.selectById
Execute SQL：SELECT id, provider_id, name FROM user WHERE user.provider_id = 1 AND id = ? {1: 1068129257418178562}

2018-11-29 21:07:14.626 DEBUG 18688 --- [ main] c.w.m.mapper.UserMapper.selectList : ==> Preparing: SELECT id, provider_id, name FROM user WHERE user.provider_id = 1
 Time：0 ms - ID：com.wuwenze.mybatisplusmultitenancy.mapper.UserMapper.selectList
Execute SQL：SELECT id, provider_id, name FROM user WHERE user.provider_id = 1

2018-11-29 21:07:14.629 DEBUG 18688 --- [ main] c.w.m.mapper.UserMapper.selectList : ==> Parameters:
2018-11-29 21:07:14.630 DEBUG 18688 --- [ main] c.w.m.mapper.UserMapper.selectList : <== Total: 3
2018-11-29 21:07:14.632  INFO 18688 --- [ main] .MybatisPlusMultiTenancyApplicationTests : ##selectList, e=User(id=1, providerId=1, name=Tony老师)
2018-11-29 21:07:14.632  INFO 18688 --- [ main] .MybatisPlusMultiTenancyApplicationTests : ##selectList, e=User(id=2, providerId=1, name=William老师)
2018-11-29 21:07:14.632  INFO 18688 --- [ main] .MybatisPlusMultiTenancyApplicationTests : ##selectList, e=User(id=1068129257418178562, providerId=1, name=新来的Tom老师)
```

从打印的日志不难看出，这个方案相当完美，仅需简单的配置，让开发者完全忽略了(provider\_id)字段的存在，同时又最大程度的保证了数据的安全性，可谓是一举两得！


[http_mp.baomidou.com_guide_tenant.html]: http://mp.baomidou.com/guide/tenant.html