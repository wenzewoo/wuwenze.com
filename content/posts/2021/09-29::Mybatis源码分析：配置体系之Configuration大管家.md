+++
title = "Mybatis源码分析：配置体系之Configuration大管家"
date = "2021-09-29 14:44:06"
url = "archives/1055"
tags = ["MyBatis"]
categories = ["后端","Mybatis源码阅读"]
featuredImage = "https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210923/261e0f3824e0400e87ff201af417b769.jpg"
+++


`Configuration`类是Mybatis所有组件之中的集中管理中心，前面我们分析过的`Executor`，`StatementHandler`，`Cache`，`MetaObject`等绝大部分组件都是由他直接或者间接的创建和管理，

此外，除了组件本身由其进行维护管理，影响这些组件的属性配置，也是由他进行管理和保存的，比如常见的`cacheEnabled`，`lazyLoadingEnabled`等配置项。所以说`Configuration`是Mybatis的大管家是当之无愧的。


## 核心作用总结

- 存储全局配置信息，其来源于settings（设置） 
- 初始化并维护全局基础组件
  - `typeAliases`（类型别名）
  - `typeHandlers`（类型处理器）
  - `plugins`（插件）
  - `environments`（环境配置）
  - `cache` (二级缓存空间)
- 初始化并维护`MappedStatement`
- 各大组件的构造器 
  - `newExecutor`（执行器）
  - `newStatementHandler`（JDBC处理器）
  - `newResultSetHandler`（结果集处理器）
  - `newParameterHandler`（参数处理器）

## 配置来源

`Configuration` 配置来源有三项：
- **mybatis-config.xml** 启动文件，全局配置、全局组件都是来源于此。
- **Mapper.xml** SQL映射(`MappedStatement`) \结果集映射(`ResultMapper`)都来源于此。
- **@Annotation** SQL映射与结果集映射的另一种表达形式。


![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210929/72224317d1634fb4ae97a0ae02e94d31.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


## 配置元素

`Configuration` 配置信息来源于XML或者注解，每个文件和注解都是由若干个配置元素组成，并呈现出嵌套关系，总体的关系如下：

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20211012/dcee72faf4f149e4bbff9b2732384fbd.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

上图中列举了部分配置元素之间的关系，关于更多的配置项以及使用方式，可以参考官方文档：[https://mybatis.org/mybatis-3/zh/configuration.html#properties](https://mybatis.org/mybatis-3/zh/configuration.html#properties)

## 元素承载

无论是XML配置的也好，还是由注解配置的元素，最终都会转换成为Java属性配置或者对象组件来承载，其对应的关系如下：

- 全局配置(`mybatis-config.xml`)由`Configuration`对象属性承载  
- SQL映射(`<select>` or `@Select`)由`MappedStatement`对象承载    
- 缓存配置(`<cache` or `@CacheNamespace`)由`Cache`对象承载  
- 结果集映射(`<resultMap` or `@ResultMap`)由`ResultMap`对象承载  

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20211012/07067ec95c604497be97b83ff9d06e4b.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


## 配置文件解析

关于XML配置文件的解析，可以用一句话概括，其底层就是使用`dom4j`将XML文件解析成一个树节点，然后根据不同的节点类型去匹配不同的解析器，最终解析成特定的组件。


### 解析器结构

解析器的基类为`BaseBuilder`，其内部包含全局的`Configuration`对象，最终所有解析出来的组件都要集中归属至`Configuration`对象。

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20211012/a49bef54e5e2429494e8ee6969fb8037.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


先来了解一下各个解析器的基本作用： 

- **XMLConfigBuilder**: 解析`mybatis-config.xml`文件，会直接创建一个`Configuration`对象。 
- **XMLMapperBuilder**: 解析`Mapper.xml`文件。 
- **MapperBuilderAssistant**: `Mapper.xml`解析的辅助器。 
- **XMLStatementBuilder**: SQL映射解析器，即将`<insert|update>`等解析成`MapperStatement`对象
- **SqlSourceBuilder**: SQL源解析器，将声明的SQL解析成可以执行的SQL。
- **XMLScriptBuilder**: 解析动态SQL源中所设置的`SqlNode`集。


# 未完待续
