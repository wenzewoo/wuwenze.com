+++
title = "Mybatis源码分析：映射体系之ResultMap原理"
date = "2021-09-13 14:21:52"
url = "archives/1051"
tags = ["MyBatis"]
categories = ["后端","Mybatis源码阅读"]
featuredImage = "https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210923/261e0f3824e0400e87ff201af417b769.jpg"
+++

在Mybatis中，映射是指返回查询结果的`ResultSet`列与`JavaBean`属性之间的对应关系，通过`ResultMapping`进行映射描述，再用`ResultMap`封装成一个整体。

按照惯例，先来看一个基本的使用例子：


```bash
<resultMap id="fullResultMap" type="com.wuwenze.mybatis.example.User">

    <!--主键字段-->
    <id property="id" column="id"/>

    <!--普通字段-->
    <result property="name" column="name"/>
    <result property="age" column="age"/>

    <!--关联字段(Has One)-->
    <association property="userGroup"
                 column="user_group_id"
                 javaType="com.wuwenze.mybatis.example.UserGroup">

        <id property="id" column="user_group_id"/>
        <result property="name" column="user_group_name"/>
    </association>

    <!--关联字段(Has Many)-->
    <collection property="tags" ofType="com.wuwenze.mybatis.example.Tag">
        <id property="id" column="tag_id"/>
        <result property="name" column="tag_name"/>
    </collection>
</resultMap>

<select id="findResultMapById" parameterType="integer" resultMap="fullResultMap">
    SELECT u.*,
        ug.name AS user_group_name,
        t.id AS tag_id,
        t.name AS tag_name
    FROM user u
        LEFT JOIN user_group ug ON ug.id = u.user_group_id
        LEFT JOIN user_r_tag urt ON urt.user_id = u.id
        LEFT JOIN tag t ON t.id = urt.tag_id
    WHERE u.id = #{id}
</select>
```


```bash
@Test
public void test1(){
    final User user=user2Mapper.findResultMapById(1);
    System.out.println(user);
}

// User(id=1, name=Zhang, age=30, userGroup=UserGroup(id=1, name=Administrators), tags=[Tag(id=1, name=Tag1), Tag(id=2, name=Tag2), Tag(id=3, name=Tag3)])
```


### 手动映射设置

在一个`ResultMap`中，包含多个`ResultMapping`，其中(`<id/>`, `<result/>`)主要的属性有以下几个：

- **`property`**: 映射到javabean中的的属性名（必填）
- **`column`**: SQL中返回的列名（必填）
- **`jdbcType`**：对应的JDBC列名的数据类型（可自动推导）
- **`javaType`**: 对应的javabean属性名的数据类型（可自动推导）
- **`typeHandler`**: 类型处理器

通常情况下，`ResultMapping`还有多种的表达形式：

- **`<constructor/>`**: 构造参数
- **`<id />`**: 主键字段
- **`<result />`**: 普通结构集字段
- **`<association />`**: 1对1关联对象字段
- **`<collection />`**: 1对M关联对象集合字段



### 映射关系

通过上述总结，整理`ResultMap`映射体系之间的关系图如下：

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210914/16fd07b3d8f1448fad1ec381722b2fe5.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


### 源码解析

在`Mapper.xml`文件中，每个`ResultMap`节点都会被解析成对应的`ResultMap`对象，与之对应的子节点，则被解析为`ResultMapping`对象，即一个`ResultMap`包含的是`ResultMapping`
的集合。

先来看两者对应的对象的声明：

```java
public class ResultMap {
    // ID，每个ResultMap唯一（namespace.id)
    private String id;
    // 该resultMap对应的Javabean类型
    private Class<?> type;
    // 除了<discriminator />节点外的其他所有节点
    private List<ResultMapping> resultMappings;
    // <id /> 主键的映射集合
    private List<ResultMapping> idResultMappings;
    // <constructor /> 构造函数节点的集合
    private List<ResultMapping> constructorResultMappings;
    // <result /> 基础属性的集合
    private List<ResultMapping> propertyResultMappings;
    // column集合
    private Set<String> mappedColumns;
    // <discriminator /> 节点（鉴别器映射）
    private Discriminator discriminator;
    private boolean hasNestedResultMaps;
    private boolean hasNestedQueries;

    // 是否开启自动映射
    private Boolean autoMapping;
}
```

```java
public class ResultMapping {
    private Configuration configuration;
    private String property;
    private String column;
    private Class<?> javaType;
    private JdbcType jdbcType;
    private TypeHandler<?> typeHandler;
    // 对应的是 resultMap 属性，通过id来引用其他的resultMap
    private String nestedResultMapId;
    // 对应的是 select 属性，通过id来引用其他的select节点的定义
    private String nestedQueryId;
    private Set<String> notNullColumns;
    private String columnPrefix;
    // 处理后的标志，标志有两个 id和constructor
    private List<ResultFlag> flags;
    // 对应节点的column属性拆分后生成的结果，composites.size()>0会使column为null
    private List<ResultMapping> composites;
    private String resultSet;
    private String foreignColumn;
    private boolean lazy;
}
```

由`Mapper.xml`转换为`ResultMap`的过程就暂时不分析了，其核心无非就是解析XML的`DOM节点`，这里我们重点分析在SQL执行完毕后，是如何利用配置好的`ResultMap`转换为Java对象结果的。

其实这部分逻辑之前在分析`StatementHandler`源码时已经初见一些端倪

- JDBC处理器StatementHandler全解析: https://wuwenze.com/archives/1001/

如果需要了解更详细的流程，可以通过上面的链接查看，这里我们只看其关键的转换逻辑：

```java
public class DefaultResultSetHandler implements ResultSetHandler {

    // 省略其他代码..


    // 从简单行中获取值
    private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, String columnPrefix) throws SQLException {
        final ResultLoaderMap lazyLoader = new ResultLoaderMap();
        Object rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
        if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
            final MetaObject metaObject = configuration.newMetaObject(rowValue);
            boolean foundValues = this.useConstructorMappings;

            // 如果开启自动映射，则先进行自动映射（后文详解）
            if (shouldApplyAutomaticMappings(resultMap, false)) {
                foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
            }

            // 再通过手动配置的ResultMap进行映射
            foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
            foundValues = lazyLoader.size() > 0 || foundValues;
            rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
        }
        return rowValue;
    }


    // 从嵌套行中获取值
    private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, CacheKey combinedKey, String columnPrefix, Object partialObject) throws SQLException {
        final String resultMapId = resultMap.getId();
        Object rowValue = partialObject;
        if (rowValue != null) {
            final MetaObject metaObject = configuration.newMetaObject(rowValue);
            putAncestor(rowValue, resultMapId);
            applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, false);
            ancestorObjects.remove(resultMapId);
        } else {
            final ResultLoaderMap lazyLoader = new ResultLoaderMap();
            rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
            if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
                final MetaObject metaObject = configuration.newMetaObject(rowValue);
                boolean foundValues = this.useConstructorMappings;
                if (shouldApplyAutomaticMappings(resultMap, true)) {
                    foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
                }
                foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
                putAncestor(rowValue, resultMapId);
                foundValues = applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, true) || foundValues;
                ancestorObjects.remove(resultMapId);
                foundValues = lazyLoader.size() > 0 || foundValues;
                rowValue = foundValues || configuration.isReturnInstanceForEmptyRow() ? rowValue : null;
            }
            if (combinedKey != CacheKey.NULL_CACHE_KEY) {
                nestedResultObjects.put(combinedKey, rowValue);
            }
        }
        return rowValue;
    }

    // 是否使用自动映射
    private boolean shouldApplyAutomaticMappings(ResultMap resultMap, boolean isNested) {
        if (resultMap.getAutoMapping() != null) {
            return resultMap.getAutoMapping(); // 当前resultMap是否开启autoMapping属性
        } else {
            // 检查全局配置autoMappingBehavior
            if (isNested) {
                return AutoMappingBehavior.FULL == configuration.getAutoMappingBehavior();
            } else {
                return AutoMappingBehavior.NONE != configuration.getAutoMappingBehavior();
            }
        }
    }


    // 属性的手动映射
    private boolean applyPropertyMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, ResultLoaderMap lazyLoader, String columnPrefix)
            throws SQLException {
        // 从查询结果中，获取已经映射的列名： ID, NAME, AGE, USER_GROUP_ID
        final List<String> mappedColumnNames = rsw.getMappedColumnNames(resultMap, columnPrefix);
        boolean foundValues = false;

        // 获取所有配置的ResultMapping对象进行遍历
        final List<ResultMapping> propertyMappings = resultMap.getPropertyResultMappings();
        for (ResultMapping propertyMapping : propertyMappings) {

            // 格式化配置的列名（如果有前缀）
            String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);

            if (propertyMapping.getNestedResultMapId() != null) {
                // the user added a column attribute to a nested result map, ignore it
                column = null;
            }

            // 是复合结果 or 已映射的列名中包含 or ResultSet不为空
            if (propertyMapping.isCompositeResult()
                    || (column != null && mappedColumnNames.contains(column.toUpperCase(Locale.ENGLISH)))
                    || propertyMapping.getResultSet() != null) {

                // 从ResultSet中获取当前列的值
                Object value = getPropertyMappingValue(rsw.getResultSet(), metaObject, propertyMapping, lazyLoader, columnPrefix);
                // issue #541 make property optional
                final String property = propertyMapping.getProperty();
                if (property == null) {
                    continue;
                } else if (value == DEFERRED) {
                    foundValues = true;
                    continue;
                }
                if (value != null) {
                    foundValues = true;
                }
                if (value != null || (configuration.isCallSettersOnNulls() && !metaObject.getSetterType(property).isPrimitive())) {
                    // gcode issue #377, call setter on nulls (value is not 'found')
                    metaObject.setValue(property, value); // 将映射对应的结果的值，通过MetaObject工具写入对象
                }
            }
        }
        return foundValues;
    }

    // 省略其他代码...
}
```


### 自动映射

手动映射还是相当麻烦的，尤其是当表中的字段非常多之后，在分析刚才的`getRowValue()`代码时，我们其实发现mybatis还支持一些自动属性的映射的，自动映射也需要一定的条件，可以总结为以下几类：

- 开启`autoMapping`或者全局`autoMappingBehavior`（默认开启）
- 数据库查询列名和Java属性名同时存在（忽略大小写）
- 当前列没有手动配置相关的`ResultMapping`
- 当前列的数据类型，有对应的`TypeHandler`处理器
- 只支持简单属性的自动映射

当满足以上条件后，可以自动映射的属性，将被转换为`UnMappedColumnAutoMapping`对象集合，为了方便演示，再来写一段不带有`ResultMap`配置的测试代码。

```java
public interface User2Mapper {

    @Select("SELECT * FROM user WHERE id = #{id}")
    User findById(Integer id);
}
```

```bash
@Test
public void test2() {
    final User user = user2Mapper.findById(1);
    System.out.println(user);
}


//==>  Preparing: SELECT * FROM user WHERE id = ?
//==> Parameters: 1(Integer)
//<==    Columns: id, name, age, user_group_id
//<==        Row: 1, Zhang, 30, 1
//<==      Total: 1
//User(id=1, name=Zhang, age=30, userGroup=null, tags=null)
```

上面的代码，并未配置`ResultMap`，从运行结果来看，是只支持简单的属性的。

`UnMappedColumnAutoMapping`的定义：

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210914/5dd68ff7e06d41b1bf67ca347fd775d0.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


```java
private static class UnMappedColumnAutoMapping {
    private final String column;
    private final String property;
    private final TypeHandler<?> typeHandler; // 类型转换器
    private final boolean primitive; // 是否是基础属性类型
}
```

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210914/1d8411334fd8436094750d9202a25f3b.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

