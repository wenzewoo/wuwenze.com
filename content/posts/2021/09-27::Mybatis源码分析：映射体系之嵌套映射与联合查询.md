+++
title = "Mybatis源码分析：映射体系之嵌套映射与联合查询"
date = "2021-09-27 12:51:18"
url = "archives/1053"
tags = ["MyBatis"]
categories = ["后端","Mybatis源码阅读"]
featuredImage = "https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210923/261e0f3824e0400e87ff201af417b769.jpg"
+++

在之前的文章中，已经大概总结了简单映射的实现原理，在实际使用Mybatis的过程中，往往对象格式是更加复杂的。


## 使用示例

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210927/846487b46a294b25896233969b8d6fad.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

以上的表结构是一个简单的博客文章模型，文章关联作者，评论关联文章和作者，为此，我们先完成对应的`JavaBean`对象封装。

```java
@Data
public class Author {

    private Integer id;
    private String name;
}

@Data
public class Comment {

    private Integer id;
    private String body;
    private Integer postId;
    private Author author;
}


@Data
public class Post {

    private Integer id;
    private String title;
    private Author author;

    private List<Comment> comments;
}
```

如果要查询一篇文章的所有信息，`ResultMapping`该如何配置呢？


### 编写SQL语句（使用JOIN的方式）

```sql
SELECT p.id    AS id,
       p.title AS title,
       a.id    AS author_id,
       a.name  AS author_name,
       c.id    AS comment_id,
       c.body  AS comment_body,
       ca.id   AS comment_author_id,
       ca.name AS comment_author_name
FROM post p
         LEFT JOIN author a ON a.id = p.author_id
         LEFT JOIN comment c ON c.post_id = p.id
         LEFT JOIN author ca ON ca.id = c.author_id
WHERE p.id = 1;
```

查询结果如下：

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210927/086c7138549c4806a57007de97df5ed3.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


### 配置`ResultMap`以及编写`PostMapper`:

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.wuwenze.mybatis.example2.mapper.PostMapper">

    <resultMap id="joinResultMap" type="com.wuwenze.mybatis.example2.entity.Post">
        <id property="id" column="id"/>
        <result property="title" column="title"/>

        <association property="author"
                     javaType="com.wuwenze.mybatis.example2.entity.Author">
            <id property="id" column="author_id"/>
            <result property="name" column="author_name"/>
        </association>

        <collection property="comments"
                    ofType="com.wuwenze.mybatis.example2.entity.Comment">
            <id property="id" column="comment_id"/>
            <result property="body" column="comment_body"/>

            <association property="author"
                         javaType="com.wuwenze.mybatis.example2.entity.Author">
                <id property="id" column="comment_author_id"/>
                <result property="name" column="comment_author_name"/>
            </association>
        </collection>
    </resultMap>


    <select id="findById" resultMap="joinResultMap">
        SELECT p.id    AS id,
               p.title AS title,
               a.id    AS author_id,
               a.name  AS author_name,
               c.id    AS comment_id,
               c.body  AS comment_body,
               ca.id   AS comment_author_id,
               ca.name AS comment_author_name
        FROM post p
                 LEFT JOIN author a ON a.id = p.author_id
                 LEFT JOIN comment c ON c.post_id = p.id
                 LEFT JOIN author ca ON ca.id = c.author_id
        WHERE p.id = #{id};
    </select>
</mapper>
```


### 编写单元测试，查看运行结果：

```java
public interface PostMapper {

    Post findById(@Param("id") Integer id);
}

@Test
public void test1() {
    final PostMapper postMapper = sqlSession.getMapper(PostMapper.class);
    final Post post = postMapper.findById(1);
    System.out.println(JSONUtil.toJsonPrettyStr(post));
}
```

```bash
==>  Preparing: SELECT p.id AS id, p.title AS title, a.id AS author_id, a.name AS author_name, c.id AS comment_id, c.body AS comment_body, ca.id AS comment_author_id, ca.name AS comment_author_name FROM post p LEFT JOIN author a ON a.id = p.author_id LEFT JOIN comment c ON c.post_id = p.id LEFT JOIN author ca ON ca.id = c.author_id WHERE p.id = ?;
==> Parameters: 1(Integer)
<==    Columns: id, title, author_id, author_name, comment_id, comment_body, comment_author_id, comment_author_name
<==        Row: 1, How to use mybatis?, 1, Zhang, 1, haha~, 2, Wang
<==        Row: 1, How to use mybatis?, 1, Zhang, 2, I don't know., 3, Tom
<==      Total: 2
{
    "comments": [
        {
            "author": {
                "name": "Wang",
                "id": 2
            },
            "body": "haha~",
            "id": 1
        },
        {
            "author": {
                "name": "Tom",
                "id": 3
            },
            "body": "I don't know.",
            "id": 2
        }
    ],
    "author": {
        "name": "Zhang",
        "id": 1
    },
    "title": "How to use mybatis?",
    "id": 1
}
```

## 实现原理
Mybatis是如何将查询结果的两条复合结果数据，填充为一个JavaBean对象的呢？从上面的测试来看，在嵌套映射时，对象呈树形结构，对之对应的`ResultMap`嵌套结构亦是如此：

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210927/00e29577f99d4b92b11d903d75318a59.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

除了以上演示的子映射配置方式外，还可以引入外部映射以及自动映射，至于其他的映射方式，这里就不再说明了，可以查阅官方文档。

有了关系映射之后，普通的单表查询是无法获取复合映射所需的结果的，这里就必须用到联合查询，然后将联合查询返回的数据列，拆分给不同的对象属性。

### 1对1嵌套映射
1对1的映射相对简单，我们通过一张图就能理清其基本原理，先来改造一下刚才的查询语句

```xml
<resultMap id="joinNotCommentsResultMap" type="com.wuwenze.mybatis.example2.entity.Post">
    <id property="id" column="id"/>
    <result property="title" column="title"/>
    
    <association property="author"
                 javaType="com.wuwenze.mybatis.example2.entity.Author">
        <id property="id" column="author_id"/>
        <result property="name" column="author_name"/>
    </association>
</resultMap>

<select id="findNotCommentsById" resultMap="joinNotCommentsResultMap">
    SELECT 
        p.id    AS id,
        p.title AS title,
        a.id    AS author_id,
        a.name  AS author_name
    FROM post p
        LEFT JOIN author a ON a.id = p.author_id
    WHERE p.id = #{id}
</select>
```

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210927/df5538501e7049df878715a0f84fb150.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

通过上述SQL语句联合查询，可以得到一条二维平面的结果数据，在结果行中，前两个字段对应的是`Post`对象，后两个字段对应的是`Author`对象，换句话说，每行都会产生两个Java对象，当然，这取决于`ResultMap`的配置。


### 1对多嵌套映射
```xml
<resultMap id="joinNotAuthorResultMap" type="com.wuwenze.mybatis.example2.entity.Post">
    <id property="id" column="id"/>
    <result property="title" column="title"/>

    <collection property="comments"
                ofType="com.wuwenze.mybatis.example2.entity.Comment">
        <id property="id" column="comment_id"/>
        <result property="body" column="comment_body"/>

        <association property="author"
                     javaType="com.wuwenze.mybatis.example2.entity.Author">
            <id property="id" column="comment_author_id"/>
            <result property="name" column="comment_author_name"/>
        </association>
    </collection>
</resultMap>

<select id="findNotAuthorById" resultMap="joinNotAuthorResultMap">
    SELECT p.id    AS id,
           p.title AS title,
           c.id    AS comment_id,
           c.body  AS comment_body,
           ca.id   AS comment_author_id,
           ca.name AS comment_author_name
    FROM post p
             LEFT JOIN comment c ON c.post_id = p.id
             LEFT JOIN author ca ON ca.id = c.author_id
    WHERE p.id = #{id};
</select>
```

```bash
==> Parameters: 1(Integer)
<==    Columns: id, title, comment_id, comment_body, comment_author_id, comment_author_name
<==        Row: 1, How to use mybatis?, 1, haha~, 2, Wang
<==        Row: 1, How to use mybatis?, 2, I don't know., 3, Tom
<==      Total: 2
{
    "comments": [
        {
            "author": {
                "name": "Wang",
                "id": 2
            },
            "body": "haha~",
            "id": 1
        },
        {
            "author": {
                "name": "Tom",
                "id": 3
            },
            "body": "I don't know.",
            "id": 2
        }
    ],
    "title": "How to use mybatis?",
    "id": 1
}
```

上述查询结果中可以得到2条数据，前两个字段对应的是`Post`对象，后四个字段对应的则是`Comment`字段(List)，与一对一不同的是，这两行数据指向的都是同一条`Post`数据，因为他们的ID都是同样的。

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210928/1057fb4b5cb448319ba0e4debd9f0fde.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


#### RowKey创建机制
一对多的关键点在于如何区分数据是否为同一条，换句话说，是基于什么原理对数据进行分组的，在一对多的映射过程中，是基于`RowKey`来进行对行数据进行分组的。


默认情况下，`RowKey`一般基于配置的`<id />`字段，但有时候往往没有配置，这时候它将采用其他所有已经配置的字段，具体规则如下：

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210928/fa59847e3fbf4234ac120a0a7258ca3d.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

### 结果集的解析流程
这里直接用一对多的情况进行总结，因为一对一其实就是一对多的简化版。

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210928/9bcc3cf67d7a47b1b302ccfc5331a7e2.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

### 源码分析
所有映射流程的解析都是在`DefaultResultSetHandler`当中完成。

首先调用`handleRowValues()`方法，进入嵌套映射分支。

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210928/db952c11dec448d3ac8c79efde5dfd74.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


#### `handleRowValuesForNestedResultMap()`

嵌套结果集解析入口，在这里会遍历结果集中所有行。并为每一行创建一个`RowKey`对象。然后调用`getRowValue()`获取解析结果对象。最后保存至`ResultHandler`中。

```java
// 暂存区的定义
private final Map<CacheKey, Object> nestedResultObjects = new HashMap<>();

private void handleRowValuesForNestedResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    final DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    ResultSet resultSet = rsw.getResultSet();
    skipRows(resultSet, rowBounds);
    Object rowValue = previousRowValue;
    
    // 进行结果行的遍历操作
    while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
      final ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
      
      // 为每一行创建一个RowKey对象
      final CacheKey rowKey = createRowKey(discriminatedResultMap, rsw, null);
      
      // 基于rowkey获取已经在暂存区的数据
      Object partialObject = nestedResultObjects.get(rowKey);
      // issue #577 && #542
      if (mappedStatement.isResultOrdered()) {
        if (partialObject == null && rowValue != null) {
          nestedResultObjects.clear();
          storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
        }
        
        // 调用getRowValue获取解析结果对象
        rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
      } else {
        rowValue = getRowValue(rsw, discriminatedResultMap, rowKey, null, partialObject);
        if (partialObject == null) {
            
          // 保存对象
          storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
        }
      }
    }
    if (rowValue != null && mappedStatement.isResultOrdered() && shouldProcessMoreRows(resultContext, rowBounds)) {
      storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
      previousRowValue = null;
    } else if (rowValue != null) {
      previousRowValue = rowValue;
    }
}

private void storeObject(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue, ResultMapping parentMapping, ResultSet rs) throws SQLException {
    if (parentMapping != null) {
        linkToParents(rs, parentMapping, rowValue);
    } else {
        callResultHandler(resultHandler, resultContext, rowValue);
    }
}

@SuppressWarnings("unchecked" /* because ResultHandler<?> is always ResultHandler<Object>*/)
private void callResultHandler(ResultHandler<?> resultHandler, DefaultResultContext<Object> resultContext, Object rowValue) {
    resultContext.nextResultObject(rowValue);
    ((ResultHandler<Object>) resultHandler).handleResult(resultContext);
}
```


注：调用`getRowValue`前会基于`RowKey`在暂存区获取已解析的对象，然后作为`partialObject`参数发给`getRowValue`

#### `getRowValue()`


该方法最终会基于当前行生成一个解析好对象，具体职责包括，
1. 创建对象
2. 填充普通属性填充嵌套属性。在解析嵌套属性时会以递归的方式在调用getRowValue获取子对象
3. 基于RowKey暂存当前解析对象

```java
private Object getRowValue(ResultSetWrapper rsw, ResultMap resultMap, CacheKey combinedKey, String columnPrefix, Object partialObject) throws SQLException {
    final String resultMapId = resultMap.getId();
    Object rowValue = partialObject;
    
    // 暂存区存在当前行
    if (rowValue != null) {
      // 直接填充复合属性
      
      final MetaObject metaObject = configuration.newMetaObject(rowValue);
      putAncestor(rowValue, resultMapId);
      applyNestedResultMappings(rsw, resultMap, metaObject, columnPrefix, combinedKey, false);
      ancestorObjects.remove(resultMapId);
    } 
    
    // 暂存区不存在当前行
    else {
      final ResultLoaderMap lazyLoader = new ResultLoaderMap();
      
      // 创建空对象
      rowValue = createResultObject(rsw, resultMap, lazyLoader, columnPrefix);
      if (rowValue != null && !hasTypeHandlerForResultObject(rsw, resultMap.getType())) {
        
        final MetaObject metaObject = configuration.newMetaObject(rowValue);
        boolean foundValues = this.useConstructorMappings;
        
        // 处理自动映射属性
        if (shouldApplyAutomaticMappings(resultMap, true)) {
          foundValues = applyAutomaticMappings(rsw, resultMap, metaObject, columnPrefix) || foundValues;
        }
        
        // 处理手动映射属性
        foundValues = applyPropertyMappings(rsw, resultMap, metaObject, lazyLoader, columnPrefix) || foundValues;
        putAncestor(rowValue, resultMapId);
        
        // 关键：处理嵌套映射结果集映射(即：填充复合属性)
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
```


#### `applyNestedResultMappings()`

解析并填充嵌套结果集映射，遍历所有嵌套映射,然后获取其嵌套`ResultMap`，接着创建RowKey 去获取暂存区的值。然后调用`getRowValue`获取属性对象，最后填充至父对象。

```java
private boolean applyNestedResultMappings(ResultSetWrapper rsw, ResultMap resultMap, MetaObject metaObject, String parentPrefix, CacheKey parentRowKey, boolean newObject) {
    boolean foundValues = false;
    
    // 遍历ResultMapping
    for (ResultMapping resultMapping : resultMap.getPropertyResultMappings()) {
      final String nestedResultMapId = resultMapping.getNestedResultMapId();
      
      // nestedResultMapId不为空(自动生成的), 则为嵌套映射, 其余的忽略
      // com.wuwenze.mybatis.example2.mapper.PostMapper.mapper_resultMap[joinNotAuthorResultMap]_collection[comments]
      if (nestedResultMapId != null && resultMapping.getResultSet() == null) {
        try {
          
          // 获取配置的属性前缀
          final String columnPrefix = getColumnPrefix(parentPrefix, resultMapping);
          
          // 获取嵌套映射的ResultMap对象
          final ResultMap nestedResultMap = getNestedResultMap(rsw.getResultSet(), nestedResultMapId, columnPrefix);
          if (resultMapping.getColumnPrefix() == null) {
            // try to fill circular reference only when columnPrefix
            // is not specified for the nested result map (issue #215)
            Object ancestorObject = ancestorObjects.get(nestedResultMapId);
            if (ancestorObject != null) {
              if (newObject) {
                linkObjects(metaObject, resultMapping, ancestorObject); // issue #385
              }
              continue;
            }
          }
          
          // 创建当前嵌套映射对象的RowKey，对应的是comments对象
          final CacheKey rowKey = createRowKey(nestedResultMap, rsw, columnPrefix);
          
          // 将当前嵌套映射对象的RowKey与父类RowKey进行合并
          final CacheKey combinedKey = combineKeys(rowKey, parentRowKey);
          
          // 从暂存区获取值
          Object rowValue = nestedResultObjects.get(combinedKey);
          boolean knownValue = rowValue != null;
          instantiateCollectionPropertyIfAppropriate(resultMapping, metaObject); // mandatory
          if (anyNotNullColumnHasValue(resultMapping, columnPrefix, rsw)) {
            
            // 用合并后的RowKey以及嵌套映射的ResultMap对象再走一遍getRowValue流程
            rowValue = getRowValue(rsw, nestedResultMap, combinedKey, columnPrefix, rowValue);
            if (rowValue != null && !knownValue) {
              
              // 拿到值，使用MetaObject赋值
              linkObjects(metaObject, resultMapping, rowValue);
              foundValues = true;
            }
          }
        } catch (SQLException e) {
          throw new ExecutorException("Error getting nested result map values for '" + resultMapping.getProperty() + "'.  Cause: " + e, e);
        }
      }
    }
    return foundValues;
}
```

如果通过`RowKey`能获取到属性对象，它还是会去调用`getRowValue`，因为有可能属性下还存在未解析的属性。


## 循环引用
了解了Mybatis复合属性的映射，那就不得不关注一个问题，如果涉及到循环引用，Mybatis是如何处理的？

举个例子，假设`Post`对象与`Comment`对象之间相互引用，就像下面这样：

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210928/0bb149d4922c4d3c9e17006561ad1bd3.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


这种情况会导致死循环吗？答案是不会，`DefaultResultSetHandler`在解析复合映射之前都会在上下文中填充当前解析对象（使用`resultMapId`做为Key）。如果子属性又映射引用了父映射ID，就可以直接获取不需要在去解析父对象。

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210928/e33de411019c48959912ac176cfa9f87.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

