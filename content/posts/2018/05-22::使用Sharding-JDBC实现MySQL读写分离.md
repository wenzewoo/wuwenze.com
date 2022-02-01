+++
title = "使用Sharding-JDBC实现MySQL读写分离"
date = "2018-05-22 06:30:00"
url = "archives/370"
tags = ["Java","MySQL","Sharding-JDBC"]
categories = ["后端"]
+++

读写分离，简单来说，就是将DML交给主数据库去执行，将更新结果同步至各个从数据库保持主从数据一致，DQL分发给从数据库去查询，从数据库只提供读取查询操作。读写分离特别适用于读多写少的场景下，通过分散读写到不同的数据库实例上来提高性能，缓解单机数据库的压力:

| Name | Remark                            |
|------|-----------------------------------|
| DQL  | 数据查询语言，比如select查询语句               |
| DML  | 数据操纵语言，比如insert、delete、update更新语句 |
| DDL  | 数据定义语言，比如create/drop/alter等语句     |
| DCL  | 数据控制语言，比如grant/rollback/commit等语句 |
|      |                                   |


Sharding-JDBC是一个开源的分布式数据库中间件解决方案。它在Java的JDBC层以对业务应用零侵入的方式额外提供数据分片，读写分离，柔性事务和分布式治理能力。并在其基础上提供封装了MySQL协议的服务端版本，用于完成对异构语言的支持。

基于JDBC的客户端版本定位为轻量级Java框架，使用客户端直连数据库，以jar包形式提供服务，无需额外部署和依赖，可理解为增强版的JDBC驱动，完全兼容JDBC和各种ORM框架。

封装了MySQL协议的服务端版本定位为透明化的MySQL代理端，可以使用任何兼容MySQL协议的访问客户端(如：MySQL Command Client, MySQL Workbench等)操作数据，对DBA更加友好。

以上内容摘抄至Sharding-JDBC官网 ([http://shardingjdbc.io/document/legacy/2.x/cn/00-overview/][http_shardingjdbc.io_document_legacy_2.x_cn_00-overview])

本文主要探讨在SpringBoot环境下如何使用Sharding-JDBC提供的读写分离解决方案;

### 环境 ###

```bash
SpringBoot: 1.5.7.RELEASE,
MybatisPlus: 2.1.4,
Sharding-JDBC: 2.0.0.M2
```

### POM.xml ###

> Sharding-JDBC现已提供相关的Starter, 集成起来非常简单;
> 
> 在pom文件中加入(springboot & mysql & mybatis-plus & sharding-jdbc)相关依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>sharding-jdbc-example-with-spring-boot</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>sharding-jdbc-example-with-spring-boot</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.7.RELEASE</version>
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>1.7</maven.compiler.source>
        <maven.compiler.target>1.7</maven.compiler.target>
        <mybatisplus-spring-boot-starter.version>1.0.5</mybatisplus-spring-boot-starter.version>
        <mybatisplus.version>2.1.4</mybatisplus.version>
        <HikariCP.version>2.7.2</HikariCP.version>
        <fastjson.version>1.2.39</fastjson.version>
        <commons-dbcp.version>1.4</commons-dbcp.version>
        <mysql-connector-java.version>5.1.30</mysql-connector-java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-jetty</artifactId>
        </dependency>
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
        </dependency>
        <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
            <version>${HikariCP.version}</version>
        </dependency>

        <!-- mybatis-plus begin -->
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatisplus-spring-boot-starter</artifactId>
            <version>${mybatisplus-spring-boot-starter.version}</version>
        </dependency>
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus</artifactId>
            <version>${mybatisplus.version}</version>
        </dependency>
        <!-- mybatis-plus end -->

        <!-- JUnit test dependency -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>${fastjson.version}</version>
        </dependency>
        <!--sharding-jdbc-->
        <dependency>
            <groupId>io.shardingjdbc</groupId>
            <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
            <version>2.0.0.M2</version>
        </dependency>
        <dependency>
            <groupId>commons-dbcp</groupId>
            <artifactId>commons-dbcp</artifactId>
            <version>${commons-dbcp.version}</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql-connector-java.version}</version>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <!-- https://mvnrepository.com/artifact/cn.hutool/hutool-all -->
        <dependency>
            <groupId>cn.hutool</groupId>
            <artifactId>hutool-all</artifactId>
            <version>4.0.12</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>8</source>
                    <target>8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

### 多数据源配置 (application.yml) ###

```yaml
server:
  port: 10086

sharding:
  jdbc:
      datasource:
        names: ds_master_0,ds_slave_0_1,ds_slave_0_2
        ds_master_0:
          type: org.apache.commons.dbcp.BasicDataSource
          driverClassName: com.mysql.jdbc.Driver
          url: jdbc:mysql://127.0.01:3306/ds_master?useUnicode=true&characterEncoding=UTF-8&useSSL=false
          username: root
          password: root
        ds_slave_0_1:
          type: org.apache.commons.dbcp.BasicDataSource
          driverClassName: com.mysql.jdbc.Driver
          url: jdbc:mysql://127.0.01:3306/ds_slave_0?useUnicode=true&characterEncoding=UTF-8&useSSL=false
          username: root
          password: root
        ds_slave_0_2:
          type: org.apache.commons.dbcp.BasicDataSource
          driverClassName: com.mysql.jdbc.Driver
          url: jdbc:mysql://127.0.01:3306/ds_slave_1?useUnicode=true&characterEncoding=UTF-8&useSSL=false
          username: root
          password: root
      config:
        ### 主从策略
        masterslave:
          load-balance-algorithm-type: round_robin ### 负载策略
          name: ds_m_1_s_2
          master-data-source-name: ds_master_0
          slave-data-source-names: ds_slave_0_1,ds_slave_0_2
        sharding:
          props:
            sql:
              show: true

##mybatis
mybatis-plus:
  datasource: dataSource
  mapper-locations: classpath:/mapper/*Mapper.xml 
  ##实体扫描，多个package用逗号或者分号分隔 
  typeAliasesPackage: com.example.shardingjdbcexamplewithspringboot.entity 
  global-config: 
    ##主键类型 0:"数据库ID自增", 1:"用户输入ID",2:"全局唯一ID (数字类型唯一ID)", 3:"全局唯一ID UUID"; 
    id-type: 0 
    ##字段策略 0:"忽略判断",1:"非 NULL 判断"),2:"非空判断" 
    field-strategy: 2 
    ##驼峰下划线转换 
    db-column-underline: true 
    ##刷新mapper 调试神器 
    refresh-mapper: true 
    ##逻辑删除配置 
    logic-delete-value: 0 
    logic-not-delete-value: 1 
    configuration: 
        map-underscore-to-camel-case: true 
        cache-enabled: false
```

实际上到这一步就完成了最简单的配置, 为了测试效果, 生成相关的数据库和实体吧:

### init data.sql ###

建立3个数据库, 分别为主, 从1, 从2, 为了区分数据来源, 在from中指定了节点名称;

```sql
CREATE SCHEMA IF NOT EXISTS `ds_master`;

DROP TABLE IF EXISTS `ds_master`.`tb_employee`;
CREATE TABLE `ds_master`.`tb_employee` (
`id`  int NOT NULL AUTO_INCREMENT ,
`name`  varchar(255) NULL ,
`from`  varchar(255) NULL ,
PRIMARY KEY (`id`)
);

INSERT INTO `ds_master`.`tb_employee` VALUES(1,'name1', 'ds_master');
INSERT INTO `ds_master`.`tb_employee` VALUES(2,'name2', 'ds_master');
INSERT INTO `ds_master`.`tb_employee` VALUES(3,'name3', 'ds_master');

#####
CREATE SCHEMA IF NOT EXISTS `ds_slave_0`;

DROP TABLE IF EXISTS `ds_slave_0`.`tb_employee`;
CREATE TABLE `ds_slave_0`.`tb_employee` (
`id`  int NOT NULL AUTO_INCREMENT ,
`name`  varchar(255) NULL ,
`from`  varchar(255) NULL ,
PRIMARY KEY (`id`)
);

INSERT INTO `ds_slave_0`.`tb_employee` VALUES(1,'name1', 'ds_slave_0');
INSERT INTO `ds_slave_0`.`tb_employee` VALUES(2,'name2', 'ds_slave_0');
INSERT INTO `ds_slave_0`.`tb_employee` VALUES(3,'name3', 'ds_slave_0');


#####
CREATE SCHEMA IF NOT EXISTS `ds_slave_1`;

DROP TABLE IF EXISTS `ds_slave_1`.`tb_employee`;
CREATE TABLE `ds_slave_1`.`tb_employee` (
`id`  int NOT NULL AUTO_INCREMENT ,
`name`  varchar(255) NULL ,
`from`  varchar(255) NULL ,
PRIMARY KEY (`id`)
);

INSERT INTO `ds_slave_1`.`tb_employee` VALUES(1,'name1', 'ds_slave_1');
INSERT INTO `ds_slave_1`.`tb_employee` VALUES(2,'name2', 'ds_slave_1');
INSERT INTO `ds_slave_1`.`tb_employee` VALUES(3,'name3', 'ds_slave_1');
```

### Entity / Mapper / Service ###

```java
@Data
@Builder
@ToString
@NoArgsConstructor
@AllArgsConstructor
@TableName("tb_employee")
public class EmployeeEntity {

    @TableId(type = IdType.AUTO)
    private Integer id;

    @TableField
    private String name;

    @TableField
    private String from;
}


public interface EmployeeMapper extends BaseMapper {
}
```

### 单元测试 ###

```java
@RunWith(SpringRunner.class)
@SpringBootTest(classes = ShardingJdbcExampleWithSpringBootApplication.class)
public class ShardingJdbcExampleWithSpringBootApplicationTests {

    @Resource
    EmployeeMapper employeeMapper;


    @Test
    public void testMasterSlave() {
        // search slave db;
        Console.log("search slave db:");
        for (int i = 0; i  {
                Console.log(employeeMapper.selectById(1));
            }).run();
        }
        Console.log("==========================================n");

        EmployeeEntity employeeEntity = EmployeeEntity.builder().name("test").from("write master db").build();
        Integer insert = employeeMapper.insert(employeeEntity);
        Console.log("write master db: {}", insert > 0); // true
        Console.log("==========================================n");

        EmployeeEntity ret = employeeMapper.selectOne(employeeEntity);
        Console.log("search by "write master db": {}", ret); // null
        Console.log("==========================================n");

           // 强制路由,访问masterdb数据
        HintManager hintManager = HintManager.getInstance();
        hintManager.setMasterRouteOnly();
        ret = employeeMapper.selectOne(employeeEntity); //(id=9, name=test, from=write master db)
        Console.log("[HintManager]search by "write master db": {}", ret);
        hintManager.close();
    }
}
```

### 最终结果 ###

```bash
search slave db:
EmployeeEntity(id=1, name=name1, from=ds_slave_0)
EmployeeEntity(id=1, name=name1, from=ds_slave_0)
EmployeeEntity(id=1, name=name1, from=ds_slave_1)
EmployeeEntity(id=1, name=name1, from=ds_slave_0)
==========================================

write master db: true
==========================================

search by "write master db": null
==========================================

[HintManager]search by "write master db": EmployeeEntity(id=11, name=test, from=write master db)
```

![9b9482dd-8399-4db9-9dc2-620ff9dbf986.png][]![6c7d10e1-ad36-44ef-892c-71652b0de6d9.png][]

如测试效果一般, Sharding-JDBC可以帮你轻松的实现读写分离, 但是数据同步仍然是需要考虑的问题;


[http_shardingjdbc.io_document_legacy_2.x_cn_00-overview]: http://shardingjdbc.io/document/legacy/2.x/cn/00-overview/
[9b9482dd-8399-4db9-9dc2-620ff9dbf986.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180522/9b9482dd-8399-4db9-9dc2-620ff9dbf986.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[6c7d10e1-ad36-44ef-892c-71652b0de6d9.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20180522/6c7d10e1-ad36-44ef-892c-71652b0de6d9.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg