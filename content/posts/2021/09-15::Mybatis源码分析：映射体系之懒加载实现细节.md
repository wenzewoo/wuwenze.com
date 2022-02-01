+++
title = "Mybatis源码分析：映射体系之懒加载实现细节"
date = "2021-09-15 10:30:43"
url = "archives/1052"
tags = ["MyBatis"]
categories = ["后端","Mybatis源码阅读"]
featuredImage = "https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210923/e72cefb271904fa0b13e41272d9f95e0.jpg"
+++

懒加载是为了改善在映射结果集解析对象属性时，大量的嵌套子查询的并发效率问题，当设置懒加载后，只有在使用指定属性时才会触发子查询，从而实现分散SQL请求的目的。

按照惯例，依然是先来看一个基础的使用示例：


## 使用示例
```java
public interface User2Mapper {

    User findWithLazyById(Integer id);
}
```

```bash
<resultMap id="lazyResultMap" type="com.wuwenze.mybatis.example.User">
    <id property="id" column="id"/>
    <result property="name" column="name"/>
    <result property="age" column="age"/>

    <!-- 通过fetchType="lazy"配置当前子查询为懒加载 -->
    <association property="userGroup" fetchType="lazy" column="user_group_id" select="findUserGroupById" />
</resultMap>

<select id="findUserGroupById" parameterType="integer" resultType="com.wuwenze.mybatis.example.UserGroup">
    SELECT * FROM user_group WHERE id = #{id}
</select>


<select id="findWithLazyById" parameterType="integer" resultMap="lazyResultMap">
    SELECT * FROM user WHERE id = #{id}
</select>
```

```bash
@Test
public void test1() {
    final User user = user2Mapper.findWithLazyById(1);
    System.out.printf("获取基本属性：%s\n", user.getName());
    System.out.printf("获取配置了懒加载的属性：%s\n", user.getUserGroup());
}

//Opening JDBC Connection
//Created connection 482082765.
//Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1cbbffcd]
//==>  Preparing: SELECT * FROM user WHERE id = ?
//==> Parameters: 1(Integer)
//<==    Columns: id, name, age, user_group_id
//<==        Row: 1, Zhang, 30, 1
//<==      Total: 1
//获取基本属性：Zhang
//==>  Preparing: SELECT * FROM user_group WHERE id = ?
//==> Parameters: 1(Integer)
//<==    Columns: id, name
//<==        Row: 1, Administrators
//<==      Total: 1
//获取配置了懒加载的属性：UserGroup(id=1, name=Administrators)
//Resetting autocommit to true on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1cbbffcd]
//Closing JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@1cbbffcd]
//Returned connection 482082765 to pool.
```

从运行结果可以很清晰的看到，只有在使用`user.getUserGroup()`对象时，才会触发对应的子查询。

在嵌套子查询中，通过手动指定`fetchType="lazy"`即可设置懒加载，除了懒加载的属性本身，在调用`equals()`, `clone()`, `hashCode()`, `toString()`等函数时均会触发当前对象所有未执行的懒加载SQL。

此外，除了手动控制以外，亦可以通过全局mybatis参数进行控制。

- **`lazyLoadingEnabled`**: 全局懒加载开关 (默认false)
- **`aggressiveLazyLoading`**: 任意方法触发加载 (默认false)

```xml
<settings>
    <setting name="lazyLoadingEnabled" value="true"/>
    <setting name="aggressiveLazyLoading" value="false"/>
</settings>
```


## 懒加载的原理及底层结构

通过对Bean的动态代理（基于`javaassist`)，重新所有属性的`Getter`方法，在获取属性前，先判断属性是否需要懒加载，如果需要，则加载之。

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210926/237f4bcc55f24d24b9e9e065d94fec4b.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


如上图所示，在被动态代理之后的Bean中
- 包含一个`MethodHandler`, 其内部包含一个`ResultLoaderMap`用于存放待执行懒加载的属性，在触发该属性的`Getter`方法时，执行懒加载之前进行移除（确保不会执行多次查询）
- `LoadPair`用于针对反序列化的Bean准备执行环境。
- `ResultLoader`用于执行加载操作，如果在执行前，原来的`Executor`已经被关闭，则会创建一个新的。


依然是上面的懒加载示例代码，加个断点来跟踪一下相关的核心逻辑。
![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210926/c9944e2a9c084437bc025af372daaa0d.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

在调试加载断点观察user对象的过程中，IDEA会调用其`toString()`方法，导致懒加载被提前触发（如上图所示）这会影响我们后续的判断，那么如何避免这种情况发生呢？

前文中提到，在调用对象的`equals()`, `clone()`, `hashCode()`, `toString()`等函数时均会触发当前对象所有未执行的懒加载SQL，这其实是可以进行配置的，来看`Configuration`的源码：

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210926/916c0f43e09d41f694d62e28fd457e83.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

很幸运的是，该属性`lazyLoadTriggerMethods`提供了`Setter`方法，这使得我们对其值进行覆盖变得非常容易，我们只需要加上以下配置：

```xml
<settings>
    <!--全局懒加载开关-->
    <setting name="lazyLoadingEnabled" value="true"/>
    <!--将积极加载改即按需加载-->
    <setting name="aggressiveLazyLoading" value="false"/>

    <!--默认equals,clone,hashCode,toString触发懒加载，移除toString()（仅供调试）-->
    <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode"/>
</settings>
```

这种方式能解决问题，~~但是不太推荐~~，若在调试完成后，忘记删除而上传到生产环境，容易引发问题，我更推荐使用下面的方式。

由于在`IDEA Debugger`模式下，在`Variables`窗口中观察属性，IDEA默认是会调用对象的`toString()`方法的，在设置中一通查找后，终于找到了关闭的方式：
![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210926/61c4945d5f11468c9132107a5a067fad.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


解决了这个问题，咱们再重新执行单元测试，观察被动态代理的`User`对象：
![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210926/3cba72d8f49c42969ebe8bbdb0b7db74.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)
从图中可以清晰的看到`BeanProxy`，`MethodHandler`，`ResultLoaderMap`，`LoadPair`以及`ResultLoader`之间的关系。


## 懒加载的触发流程
接着上面运行的代码，继续执行`user.getUserGroup()`方法，来进行触发`UserGroup`的子查询。

> 在上面的图中，我们观察到`MethodHandler`由`JavassistProxyFactory#EnhancedResultObjectProxyImpl`进行动态代理，所以我们只需要关注其动态代理实现的`invoke()`方法即可。

### 何时创建代理对象的？
代理过程发生在结果集解析创建对象之后(`DefaultResultSetHandler#createResultObject()`)，
如果对应的属性设置了懒加载，则会通过`ProxyFactory`创建代理对象，该对象继承自原来的对象，然后将对象的值全部拷贝到代理对象，并创建对应的`MethodHandler`对象。
![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210926/64526d2027e74e72922bbd5bbd7f9468.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

执行`configuration.getProxyFactory().createProxy()`后，由`ProxyFactory`的实现类`JavassistProxyFactory`实现代理的创建，最终来到了`EnhancedResultObjectProxyImpl#createProxy()`工具方法。

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210926/5e6d29c7ea0d495197f8a3ddc98b4ac8.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


```java
private static class EnhancedResultObjectProxyImpl implements MethodHandler {

    // 省略其他

    public static Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      final Class<?> type = target.getClass();
      EnhancedResultObjectProxyImpl callback = new EnhancedResultObjectProxyImpl(type, lazyLoader, configuration, objectFactory, constructorArgTypes, constructorArgs);

      // 创建user对象的动态代理，并将EnhancedResultObjectProxyImpl(也就是MethodHandler)设置进去
      Object enhanced = crateProxy(type, callback, constructorArgTypes, constructorArgs);

      // 拷贝原有的属性
      PropertyCopier.copyBeanProperties(type, target, enhanced);
      return enhanced;
    }
    
    // 省略其他

}
```

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210926/79c3e44687d44907bd567e837dbd8c1f.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

### MethodHandler做了些什么？

在其内部，实现了对`User`对象所有`Method`调用的拦截处理，来看`EnhancedResultObjectProxyImpl#invoke()`的核心代码。


```java
private static class EnhancedResultObjectProxyImpl implements MethodHandler {

    // com.wuwenze.mybatis.example.User
    private final Class<?> type;
    // 需要懒加载的属性 USERGROUP
    private final ResultLoaderMap lazyLoader;
    // 全局配置中的aggressiveLazyLoading配置(任意方法触发加载)
    private final boolean aggressive;
    // 全局配置中的lazyLoadTriggerMethods配置(默认equals,clone,hashCode,toString) 指定方法触发懒加载
    private final Set<String> lazyLoadTriggerMethods;

    // 创建对象实例的工厂、构建函数等
    private final ObjectFactory objectFactory;
    private final List<Class<?>> constructorArgTypes;
    private final List<Object> constructorArgs;

    private EnhancedResultObjectProxyImpl(Class<?> type, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      // 省略赋值初始化逻辑
    }


   // 创建该类代理的工具方法
   // 在Mybatis处理结果时调用DefaultResultSetHandler#createResultObject()
    public static Object createProxy(Object target, ResultLoaderMap lazyLoader, Configuration configuration, ObjectFactory objectFactory, List<Class<?>> constructorArgTypes, List<Object> constructorArgs) {
      // 省略，见上文。
    }


    // 对代理的user对象每个method的调用进行拦截处理。
    @Override
    public Object invoke(Object enhanced, Method method, Method methodProxy, Object[] args) throws Throwable {
      final String methodName = method.getName();
      try {
        synchronized (lazyLoader) {

          // 如果是writeReplace()方法 (序列化相关的函数，暂不关注)
          if (WRITE_REPLACE_METHOD.equals(methodName)) {
            Object original;
            if (constructorArgTypes.isEmpty()) {
              original = objectFactory.create(type);
            } else {
              original = objectFactory.create(type, constructorArgTypes, constructorArgs);
            }
            PropertyCopier.copyBeanProperties(type, enhanced, original);
            if (lazyLoader.size() > 0) {
              return new JavassistSerialStateHolder(original, lazyLoader.getProperties(), objectFactory, constructorArgTypes, constructorArgs);
            } else {
              return original;
            }
          } else {
            if (lazyLoader.size() > 0 && !FINALIZE_METHOD.equals(methodName)) {

              // 任意方法触发 或者 equals,clone,hashCode,toString等方法？
              if (aggressive || lazyLoadTriggerMethods.contains(methodName)) {
                lazyLoader.loadAll(); // 加载所有
              } 
              // 如果是Setter方法（如setUserGroup()），则移除ResultLoaderMap中对应的属性，不会触发懒加载(避免覆盖手动设置的值）。
              else if (PropertyNamer.isSetter(methodName)) {
                final String property = PropertyNamer.methodToProperty(methodName);
                lazyLoader.remove(property);
              } 
              
              // 如果是Getter方法，
              else if (PropertyNamer.isGetter(methodName)) {
                final String property = PropertyNamer.methodToProperty(methodName);
                
                // 判断ResultLoaderMap中是待加载的属性是否包含，如果是，这触发该属性的懒加载。
                if (lazyLoader.hasLoader(property)) {
                  lazyLoader.load(property);
                }
              }
            }
          }
        }
        return methodProxy.invoke(enhanced, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
  }
```

在`EnhancedResultObjectProxyImpl#invoke()`中，更多的是一些逻辑判断，而真正的逻辑，还得继续看下一层`ResultLoaderMap`内部实现的封装

```java
public class ResultLoaderMap {

  // 需要懒加载且还未加载的属性列表
  private final Map<String, LoadPair> loaderMap = new HashMap<>();

  // 省略其他代码

  public boolean hasLoader(String property) {
    // 判断是否需要懒加载
    return loaderMap.containsKey(property.toUpperCase(Locale.ENGLISH));
  }

  // 触发单个属性的懒加载逻辑
  public boolean load(String property) throws SQLException {

    // 无论如何，先移除掉该属性，避免下次再次触发懒加载（结合hasLoader()）来看
    LoadPair pair = loaderMap.remove(property.toUpperCase(Locale.ENGLISH));
    if (pair != null) {
      pair.load(); // 更近一层的封装，见LoadPair
      return true;
    }
    return false;
  }

  public void remove(String property) {
    loaderMap.remove(property.toUpperCase(Locale.ENGLISH));
  }

  // 所有属性的全部加载
  public void loadAll() throws SQLException {
    final Set<String> methodNameSet = loaderMap.keySet();
    String[] methodNames = methodNameSet.toArray(new String[methodNameSet.size()]);
    for (String methodName : methodNames) {
      load(methodName);
    }
  }

  /** Property which was not loaded yet. */
  public static class LoadPair implements Serializable {

    private final transient Object serializationCheck = new Object();
    /** Meta object which sets loaded properties. */
    private transient MetaObject metaResultObject;
    /** Result loader which loads unread properties. */
    private transient ResultLoader resultLoader;

    // 省略其他

    private LoadPair(
        final String property, MetaObject metaResultObject, ResultLoader resultLoader) {
      // 暂时省略构造函数
    }

    // 此方法为加载的入口
    public void load() throws SQLException {
      /* These field should not be null unless the loadpair was serialized.
       * Yet in that case this method should not be called. */
      if (this.metaResultObject == null) {
        throw new IllegalArgumentException("metaResultObject is null");
      }
      if (this.resultLoader == null) {
        throw new IllegalArgumentException("resultLoader is null");
      }

      this.load(null);
    }

    public void load(final Object userObject) throws SQLException {

      // 这两个属性经过transient修饰，如果是反序列化后再触发懒加载，则这两个属性为空，以此来判断是否是反序列化操作。
      if (this.metaResultObject == null || this.resultLoader == null) {
        if (this.mappedParameter == null) {
          throw new ExecutorException(
              "Property ["
                  + this.property
                  + "] cannot be loaded because "
                  + "required parameter of mapped statement ["
                  + this.mappedStatement
                  + "] is not serializable.");
        }

        // 重新获取Configuration等对象，重新构建MetaObject以及ResultLoader对象
        final Configuration config = this.getConfiguration();
        final MappedStatement ms = config.getMappedStatement(this.mappedStatement);
        if (ms == null) {
          throw new ExecutorException(
              "Cannot lazy load property ["
                  + this.property
                  + "] of deserialized object ["
                  + userObject.getClass()
                  + "] because configuration does not contain statement ["
                  + this.mappedStatement
                  + "]");
        }

        this.metaResultObject = config.newMetaObject(userObject);

        // 反序列化回来的对象，重新构建ResultLoader，指定执行器为ClosedExecutor，表示该执行器已经不能重用了，需要重新构建Executor对象。
        this.resultLoader =
            new ResultLoader(
                config,
                new ClosedExecutor(),
                ms,
                this.mappedParameter,
                metaResultObject.getSetterType(this.property),
                null,
                null);
      }

      /* We are using a new executor because we may be (and likely are) on a new thread
       * and executors aren't thread safe. (Is this sufficient?)
       *
       * A better approach would be making executors thread safe. */

      // 双重检查
      if (this.serializationCheck == null) {
        final ResultLoader old = this.resultLoader;
        this.resultLoader =
            new ResultLoader(
                old.configuration,
                new ClosedExecutor(),
                old.mappedStatement,
                old.parameterObject,
                old.targetType,
                old.cacheKey,
                old.boundSql);
      }

      // 使用metaObject反射工具类设置值，获取值见对应的resultLoader.loadResult()
      this.metaResultObject.setValue(property, this.resultLoader.loadResult());
    }

    private static final class ClosedExecutor extends BaseExecutor {

      public ClosedExecutor() {
        super(null, null);
      }

      @Override
      public boolean isClosed() {
        return true; // 表示该执行器已经不能重用了，需要重新构建Executor对象。
      }

      @Override
      protected int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
        throw new UnsupportedOperationException("Not supported.");
      }

      @Override
      protected List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
        throw new UnsupportedOperationException("Not supported.");
      }

      @Override
      protected <E> List<E> doQuery(
          MappedStatement ms,
          Object parameter,
          RowBounds rowBounds,
          ResultHandler resultHandler,
          BoundSql boundSql)
          throws SQLException {
        throw new UnsupportedOperationException("Not supported.");
      }

      @Override
      protected <E> Cursor<E> doQueryCursor(
          MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql)
          throws SQLException {
        throw new UnsupportedOperationException("Not supported.");
      }
    }
  }
}
```

在上面兜了一圈之后，我们发现最终触发查询的代码逻辑又跑到`ResultLoader.loadResult()`去了，好吧继续跟踪源代码。


```java
public class ResultLoader {

  // 省略其他代码

  public Object loadResult() throws SQLException {
    List<Object> list = selectList(); // 执行查询操作
    resultObject = resultExtractor.extractObjectFromList(list, targetType);
    return resultObject;
  }

  private <E> List<E> selectList() throws SQLException {
    Executor localExecutor = executor;

    // 是否为同一个线程，或者executor已经被关闭？(即ClosedExectutor)
    if (Thread.currentThread().getId() != this.creatorThreadId || localExecutor.isClosed()) {
      localExecutor = newExecutor(); // 重新构建Executor对象
    }
    try {

      // 执行查询操作。
      return localExecutor.query(mappedStatement, parameterObject, RowBounds.DEFAULT, Executor.NO_RESULT_HANDLER, cacheKey, boundSql);
    } finally {
      if (localExecutor != executor) {
        localExecutor.close(false);
      }
    }
  }

  private Executor newExecutor() {
    final Environment environment = configuration.getEnvironment();
    if (environment == null) {
      throw new ExecutorException("ResultLoader could not load lazily.  Environment was not configured.");
    }
    final DataSource ds = environment.getDataSource();
    if (ds == null) {
      throw new ExecutorException("ResultLoader could not load lazily.  DataSource was not configured.");
    }

    // 构造一个全新的SimpleExecutor对象，用于执行SQL查询。
    final TransactionFactory transactionFactory = environment.getTransactionFactory();
    final Transaction tx = transactionFactory.newTransaction(ds, null, false);
    return configuration.newExecutor(tx, ExecutorType.SIMPLE);
  }
}
```

至此，关于Mybatis的属性懒加载流程，就分析完毕了。