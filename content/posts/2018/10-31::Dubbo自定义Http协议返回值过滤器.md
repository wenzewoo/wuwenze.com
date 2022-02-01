+++
title = "Dubbo自定义Http协议返回值过滤器"
date = "2018-10-31 05:58:00"
url = "archives/83"
tags = ["Dubbo"]
categories = ["后端"]
+++

公司内部的Dubbo又封装了一层，通过注解直接暴露Service接口，对外提供Http服务，在序列化返回结果时，简单粗暴的将实体的所有非空属性全部序列化出来了，接口的返回体相当庞大，很是浪费资源。

### 核心实现 ###

使用FastJson的`com.alibaba.fastjson.serializer.PropertyFilter`，在序列化时，排除相关的属性，核心代码如下：

```java
PropertyFilter profilter = new PropertyFilter(){
    @Override
    public boolean apply(Object object, String name, Object value) {
        if(name.equalsIgnoreCase("password")){
            return false;  // 排除password 属性
        }
        return true;
    }
};
serializer.addFilter(profilter);
```

V2接口基于Dubbo相关协议封装

其核心序列化逻辑在`org.budo.dubbo.protocol.http.view.render.ViewRender`的`com.ewei.common.dubbo.http.view.render.EweiDubboHttpApiJsonViewRender`实现类中，见下图：

![2019-08-19-073707][]

现只需要在序列化之前，添加相关的PropertyFilter即可。

```java
serializer.addFilter(new XxxPropertyFilter());
```

#### @ApiResponseFilter注解定义 ####

考虑到通用性和灵活性，每个接口的PropertyFilter的定义显然是不一样的，可以通过注解配置在接口方法上，然后在使用时读取注解配置，动态生成PropertyFilter对象。

```java
/**
 * 注解在V2接口上，规定接口返回结果JSON的序列化规则。
 * @author wwz
 * @version 1 (2018/10/23)
 * @since Java7
 */
@Target({ ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface ApiResponseFilter {

    /**
     * 指定配置的序列化规则作用的实体类
     * 当配置includeFields时，该属性必填
     * @return
     */
    Class<?> clazz() default EmptyClazz.class;

    /**
     * 要排除的字段，多个用","分割 (优先级0)
     * @return
     */
    String ignoreFields() default "";

    /**
     * 要序列化的字段，多个用","分割 (优先级1)
     * @return
     */
    String includeFields() default "";

    class EmptyClazz {}
}
```

一般来说，一个@ApiResponseFilter，对应一个clazz，然而一个复杂的返回对象中，往往不止一个实体对象（存在嵌套关系），这时候就需要配置一组@ApiResponseFilter规则了。

下面通过组合注解的方式，将@ApiResponseFilter进行包装：

```java
/**
 * @author wwz
 * @version 1 (2018/10/23)
 * @since Java7
 * @see ApiResponseFilter
 */
@Target({ ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface ApiResponseFilters {

    ApiResponseFilter[] value();
}
```

#### 通用PropertyFilter定义 ####

```java
/**
 * 自定义实现：指定序列化时只能包含哪些属性
 * @author wwz
 * @version 1 (2018/10/24)
 * @since Java7
 */
public class BudoJsonIncludeFieldFilter implements PropertyFilter {
    private final static String[] PUBLIC_INCLUDE_FIELDS =//
            {"result", "status", "error", "error_code", "error_description", "empty", "list", "recordCount"};
    private Class<?> clazz;
    private String[] includeFields;

    public BudoJsonIncludeFieldFilter(Class<?> clazz, String... includeFields) {
        this.clazz = clazz;
        this.includeFields = includeFields;
        this.assertionOfParameters();
    }

    public BudoJsonIncludeFieldFilter(Class<?> clazz, String includeFields) {
        this.clazz = clazz;
        this.includeFields = includeFields.split(",");
        this.assertionOfParameters();
    }

    private void assertionOfParameters() {
        if (null == clazz) {
            throw new IllegalArgumentException("#1024 clazz is null.");
        }
        if (null == includeFields || includeFields.length == 0) {
            throw new IllegalArgumentException("#1024 includeFields is null.");
        }
    }

    @Override
    public boolean apply(Object obj, String field, Object value) {
        if (!StringUtil.equals(obj.getClass().getName(), clazz.getName())) {
            return true;
        }
        for (String includeField : includeFields) {
            includeField = StringUtil.trim(includeField);
            if (StringUtil.equalsIgnoreCase(includeField, field)) {
                return true;
            }
        }
        return false;
    }
}

/**
 * 自定义实现：指定序列化时需要排除哪些属性
 * @author wwz
 * @version 1 (2018/10/24)
 * @since Java7
 */
public class BudoJsonIgnoreFieldFilter implements PropertyFilter {
    private Class<?> clazz;
    private String[] ignoreFieldNames;

    public BudoJsonIgnoreFieldFilter(String... ignoreFieldNames) {
        this(null, ignoreFieldNames);
    }

    public BudoJsonIgnoreFieldFilter(Class<?> clazz, String... ignoreFieldNames) {
        this.clazz = clazz;
        this.ignoreFieldNames = ignoreFieldNames;
        this.assertionOfParameters();
    }

    public BudoJsonIgnoreFieldFilter(String ignoreFieldNames) {
        this(null, ignoreFieldNames);
    }

    public BudoJsonIgnoreFieldFilter(Class<?> clazz, String ignoreFieldNames) {
        this.clazz = clazz;
        this.ignoreFieldNames = ignoreFieldNames.split(",");
        this.assertionOfParameters();
    }

    private void assertionOfParameters() {
        if (null == ignoreFieldNames || ignoreFieldNames.length == 0) {
            throw new IllegalArgumentException("#1024 ignoreFieldNames is null.");
        }
    }

    @Override
    public boolean apply(Object obj, String field, Object value) {
        String objClassName = obj.getClass().getName();
        if (null != clazz &&//
                !StringUtil.equals(objClassName, clazz.getName())) {
            return true;
        }

        for (String ignoreField : ignoreFieldNames) {
            ignoreField = StringUtil.trim(ignoreField);
            if (StringUtil.equalsIgnoreCase(ignoreField, field)) {
                return false;
            }
        }
        return true;
    }
}
```

#### ViewRender重构 ####

ViewRender用于视图的渲染，是一个顶层的接口，其主要实现类有以下：

![0b8ab4a8-cd64-4e1d-bf3f-3416669a34b6.png][]

由于在渲染JSON数据时，需要获取到当前接口的配置信息（注解），目前的ViewRender还满足不了，重新定义后如下：

```java
public interface ViewRender {
    void renderView(ProtocolRequest request, ProtocolResponse response, Object result) throws Throwable;
}

// 添加Invocation参数
public interface ViewRender {
    void renderView(ProtocolRequest request, ProtocolResponse response, Invocation invocation, Object result) throws Throwable;
}
```

`com.alibaba.dubbo.rpc.Invocation`中包含RPC调用相关参数信息：

![980285b2-92cc-4c70-a808-fcaa7ea79b81.png][]

可以通过getInvoker().getInterface()、getMethodName()、getParameterTypes()等信息获取到被调用接口的相关对象，从而反射获取@ApiResponseFilter配置。

顶级ViewRender被重新定义后，相关的实现类也做响应的重构操作（这里不详细描述了）

#### EweiDubboHttpApiJsonViewRender实现 ####

```java
@Override
protected void doWriteResponse(ProtocolRequest protocolRequest, ProtocolResponse protocolResponse, Invocation invocation, Object result) throws IOException {
    String json = this.serializeJsonString(invocation, result);
    protocolResponse.write(new ByteArrayInputStream(json.getBytes()));
}

private String serializeJsonString(Invocation invocation, Object result) {
    SerializeWriter serializeWriter = new SerializeWriter();

    try {
        JSONSerializer serializer = new JSONSerializer(serializeWriter);

        if (FastJsonHibernatePropertyFilter.HAS_HIBERNATE) {
            serializer.addFilter(FastJsonHibernatePropertyFilter.INSTANCE);
        }
        // 根据调用接口配置的@ApiResponseFilter注解，动态生成PropertyFilter
        this.builderSerializerFilter(serializer, invocation);
        serializer.setDateFormat(DATE_FORMAT);
        serializer.config(SerializerFeature.WriteDateUseDateFormat, true);

        serializer.write(result);

        return serializeWriter.toString();
    } finally {
        serializeWriter.close();
    }
}

private void builderSerializerFilter(JSONSerializer serializer, Invocation invocation) {
    if (null == invocation) {
        return;
    }
    List<ApiResponseFilter> apiResponseFilters = this.getMethodAnnotationWithApiResponseFilter(invocation);
    if (CollectionUtil.isEmpty(apiResponseFilters)) {
        return;
    }
    for (ApiResponseFilter apiResponseFilter : apiResponseFilters) {
        Class<?> clazz = apiResponseFilter.clazz();
        if (StringUtil.equals(clazz.getName(), ApiResponseFilter.EmptyClazz.class.getName())) {
            clazz = null;
        }
        String ignoreFields = apiResponseFilter.ignoreFields();
        String includeFields = apiResponseFilter.includeFields();

        if (!StringUtil.isEmpty(ignoreFields)) {
            serializer.addFilter(new BudoJsonIgnoreFieldFilter(clazz, ignoreFields));
        }
        if (!StringUtil.isEmpty(includeFields)) {
            serializer.addFilter(new BudoJsonIncludeFieldFilter(clazz, includeFields));
        }
    }
}

private List<ApiResponseFilter> getMethodAnnotationWithApiResponseFilter(Invocation invocation) {
    List<ApiResponseFilter> apiResponseFilterList = new ArrayList<>();

    Invoker<?> invoker = invocation.getInvoker();
    ApiResponseFilters apiResponseFilters = BudoReflectionUtil.getMethodAnnotation(invoker.getInterface(), //
        invocation.getMethodName(), //
        invocation.getParameterTypes(), //
        ApiResponseFilters.class);
    if (null != apiResponseFilters) {
        ApiResponseFilter[] filters = apiResponseFilters.value();
        if (!ArrayUtil.isEmpty(filters)) {
            for (ApiResponseFilter filter : filters) {
                apiResponseFilterList.add(filter);
            }
        }
    }

    ApiResponseFilter apiResponseFilter = BudoReflectionUtil.getMethodAnnotation(invoker.getInterface(), //
        invocation.getMethodName(), //
        invocation.getParameterTypes(), //
        ApiResponseFilter.class);
    if (null != apiResponseFilter) {
        apiResponseFilterList.add(apiResponseFilter);
    }
    return apiResponseFilterList;
}
```

### 测试 ###

定义一个测试用的JavaBean:

```java
public class TestEntity {

    private Integer id;
    private String name;
    private String key;
    private TestEntity1 testEntity1;

    //...
}

public class TestEntity1 {

    private String name;
    private String value;

    //...
}
```

初始化数据：

```java
private TestEntity build() {
    TestEntity testEntity = new TestEntity();
    testEntity.setId(1);
    testEntity.setName("name");
    testEntity.setKey("key");

    TestEntity1 testEntity1 = new TestEntity1();
    testEntity1.setName("name1");
    testEntity1.setValue("value1");
    testEntity.setTestEntity1(testEntity1);
    return testEntity;
}
```

按照Open V2的方式定义测试接口，并测试@ApiResponseFilter注解

```java
public interface OpenTestApiResponseFilterApi {

    /**
     * 不做任何限制
     * {
     *  "result": {
     *    "id": 1,
     *    "key": "key",
     *    "name": "name",
     *    "testEntity1": {
     *      "name": "name1",
     *      "value": "value1"
     *    }
     *  },
     *  "status": 0
     * }
     * @return
     */
    public TestEntity test();

    /**
     * 限制返回结果中，TestEntity只能包含id,testEntity1两个字段
     * {
     *  "result": {
     *    "id": 1,
     *    "testEntity1": {
     *      "name": "name1",
     *      "value": "value1"
     *    }
     *  },
     *  "status": 0
     * }
     * @return
     */
    @ApiResponseFilter(clazz = TestEntity.class, includeFields = "id,testEntity1")
    public TestEntity test1();

    /**
     * 需要忽略的字段（不限制实体类）
     * {
     *  "result": {
     *    "id": 1,
     *    "name": "name",
     *    "testEntity1": {
     *      "name": "name1"
     *    }
     *  },
     *  "status": 0
     * }
     * @return
     */
    @ApiResponseFilter(ignoreFields = "key,value")
    public TestEntity test2();

    /**
     * 需要忽略的字段（TestEntity中）
     * {
     *  "result": {
     *    "id": 1,
     *    "name": "name",
     *    "testEntity1": {
     *      "name": "name1",
     *      "value": "value1"
     *    }
     *  },
     *  "status": 0
     * }
     * @return
     */
    @ApiResponseFilter(clazz = TestEntity.class, ignoreFields = "key,value")
    public TestEntity test3();

    /**
     * 同时配置多个规则，将按照配置顺序依次添加相关的PropertyFilter
     * {
     * "result": {
     *    "key": "key",
     *    "name": "name",
     *    "testEntity1": {
     *      "value": "value1"
     *    }
     *  },
     *  "status": 0
     * }
     * @return
     */
    @ApiResponseFilters({ //
        @ApiResponseFilter(clazz = TestEntity.class, ignoreFields = "id"), //
        @ApiResponseFilter(clazz = TestEntity1.class, ignoreFields = "name")
    })
    public TestEntity test4();
}
```


[2019-08-19-073707]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181031/98c16527-9b41-4708-8f11-825fdd965af2.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[0b8ab4a8-cd64-4e1d-bf3f-3416669a34b6.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181031/0b8ab4a8-cd64-4e1d-bf3f-3416669a34b6.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[980285b2-92cc-4c70-a808-fcaa7ea79b81.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20181031/980285b2-92cc-4c70-a808-fcaa7ea79b81.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg