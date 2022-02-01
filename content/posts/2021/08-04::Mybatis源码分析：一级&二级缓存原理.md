+++
title = "Mybatis源码分析：一级&二级缓存原理"
date = "2021-08-04 03:52:09"
url = "archives/951"
tags = ["MyBatis"]
categories = ["后端","Mybatis源码阅读"]
featuredImage = "https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210923/261e0f3824e0400e87ff201af417b769.jpg"
+++

通过分析`Executor`的源码，发现Mybatis的缓存逻辑都在执行器中实现，本文将继续探讨多级缓存命中场景以及其实现原理。

先来回顾一下`Executor`的结构：

![88f83e04-b284-4abc-b68c-0c013a1616eb.png][]

如果你还没有看过`Executor`执行器的源码分析，可以通过下面的链接阅读更多：

- Mybatis源码分析：深入认识Executor执行器: https://wuwenze.com/archives/918

## 概述 ##

在mybatis中，共存在二级缓存，分别在`BaseExecutor`和`CachingExcutor`中实现。

 *  一级缓存：会话级缓存，仅作用于当前会话，不可直接关闭（但是在和spring集成中是一般是失效的）
 *  二级缓存：应用级缓存，即生命周期作用于整个应用的生命周期，是可以直接跨线程使用的，默认是关闭的。

![856f9684-5c1a-4220-8f9a-b4b4949c9c82.png][]

## 一级缓存 ##

### 一级缓存命中场景 ###

一级缓存在特定的场景下才会命中，咱们先来进行一些单元测试，以便于总结：

```java
public interface UserMapper {

    @Select("SELECT * FROM user WHERE id = ##{id}")
    User findById1(@Param("id") Integer id);

    @Select("SELECT * FROM user WHERE id = ##{id}")
    User findById2(@Param("id") Integer id);

    @Select("SELECT * FROM user WHERE age >= ##{age}")
    List<User> listByAge(@Param("age") Integer age);
}


public class OneLevelCachedTest {
    // 省略...

    @Test
    public void test1() {
        final UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
        // SQL/MapperStatement/参数完全相同
        final User user1 = userMapper.findById1(1);
        final User user2 = userMapper.findById1(1);

        // 内存地址相同说明命中缓存
        assert user1 == user2;
    }


    @Test
    public void test2() {
        final UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
        // SQL/参数完全相同
        // MapperStatement不同（即调用的Mapper不同）

        // com.wuwenze.mybatis.example.UserMapper.findById1
        final User user1 = userMapper.findById1(1);
        // com.wuwenze.mybatis.example.UserMapper.findById2
        final User user2 = userMapper.findById2(1);

        // 内存地址相同说明命中缓存
        assert user1 == user2;
    }


    @Test
    public void test3() {
        // SQL/MapperStatement/参数完全相同
        // 但是不是同一个SqlSession
        final User user1 = sqlSession.getMapper(UserMapper.class).findById1(1);

        try (final SqlSession sqlSession2 = sqlSessionFactory.openSession()) {
            final User user2 = sqlSession2.getMapper(UserMapper.class).findById1(1);

            // 内存地址相同说明命中缓存
            assert user1 == user2;
        }
    }


    @Test
    public void test4() {
        // SQL/MapperStatement/参数/SqlSession完全相同
        // 但是RowBounds不同（分页参数）
        final String statementId = "com.wuwenze.mybatis.example.UserMapper.listByAge";

        final Map<String, Object> params = new HashMap<>();
        params.put("age", 3);

        final List<Object> result1 = sqlSession.selectList(statementId, params, RowBounds.DEFAULT);
        final List<Object> result2 = sqlSession.selectList(statementId, params, new RowBounds(0,10));
        assert result1 == result2;
    }
}
```

![d2573e6f-60d5-422f-aa5b-b235b5d4f9b2.png][]

上述的单元测试中，构造了四种不同的情况，按照执行结果来看，可以将缓存的命中条件总结为：

 *  在同一个`SqlSession`会话中
 *  调用同一个`StatementID`（即`MapperStatementId`相同）
 *  SQL与参数需完全相同（包括`RowBounds`分页参数）

### 一级缓存清空策略 ###

既然是缓存，那么一定有清理的策略，除了当前会话关闭，在会话中何时清空呢？继续看单元测试：

```java
void testClearCache(Consumer<SqlSession> consumer) {
    final UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    // SQL/MapperStatement/参数完全相同
    final User user1 = userMapper.findById1(1);

    consumer.accept(sqlSession); // 执行清理缓存的操作

    final User user2 = userMapper.findById1(1);
    // 内存地址不同，则说明清理缓存成功
    assert user1 != user2;
}

@Test
public void testClearCache1() {
    // 手动清空缓存
    testClearCache(SqlSession::clearCache);
}

@Test
public void testClearCache2() {
    testClearCache(session -> {
        // 使用当前session执行一次更新操作（非当前查询的ID）
        session.getMapper(UserMapper.class).updateNameById(20, "张三");
    });
}

@Test
public void testClearCache3() {
    // 将全局配置中的"localCacheScope"修改为STATEMENT，此时一级缓存都不会生效
    final Configuration configuration = sqlSessionFactory.getConfiguration();
    configuration.setLocalCacheScope(LocalCacheScope.STATEMENT);

    testClearCache(session -> {
        // Do nothing..
    });
}

@Test
public void testClearCache4() {
    final UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    // 中间无任何操作，但是该findById3()配置了flushCache=true
    //    @Select("SELECT * FROM user WHERE id = ##{id}")
    //    @Options(flushCache = Options.FlushCachePolicy.TRUE)
    //    User findById3(@Param("id") Integer id);
    final User user1 = userMapper.findById3(1);
    final User user2 = userMapper.findById3(1);
    // 内存地址不同，则说明清理缓存成功
    assert user1 != user2;
}
```

![98bc8b42-8963-4a7b-acc3-f0cf5fd2fa03.png][]

通过运行单元测试的结果，列举的几种方式都已成功清理缓存，总结如下：

 *  手动调用`sqlSession.clearCache()`
 *  执行事务回滚
 *  执行`update`操作（任意）
 *  配置当前`flushCache=true`
 *  全局配置缓存作用域`localCacheScope=STATEMENT`

![751b1566-a573-436d-b641-468e9245674e.png][]

### 一级缓存实现原理 ###

一级缓存在`BaseExecutor`中实现，来看看具体的实现逻辑吧，这里只关注`query()`相关的逻辑即可。

```java
public abstract class BaseExecutor implements Executor {
  // 省略其他代码...

  // 一级缓存定义
  protected PerpetualCache localCache;

  // 存储过程Out参数的缓存定义
  protected PerpetualCache localOutputParameterCache;

  protected BaseExecutor(Configuration configuration, Transaction transaction) {
    // 初始化缓存容器
    this.localCache = new PerpetualCache("LocalCache");
    this.localOutputParameterCache = new PerpetualCache("LocalOutputParameterCache");
    // 省略其他..
  }

  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);

    // 构建缓存的Key，几个参数决定了命中缓存的条件
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
  }

  @SuppressWarnings("unchecked")
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    // queryStack 是嵌套查询相关的逻辑，这里暂时不研究
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;

      // 尝试从localCache中获取缓存
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        // 处理存储过程Out参数相关
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {

        // 未命中缓存，从数据库中查询数据
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue ##601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue ##482
        clearLocalCache();
      }
    }
    return list;
  }

  protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
      throws SQLException;

  private void handleLocallyCachedOutputParameters(MappedStatement ms, CacheKey key, Object parameter, BoundSql boundSql) {
    // 仅存储过程需要处理
    if (ms.getStatementType() == StatementType.CALLABLE) {
      // 暂时省略..
    }
  }

 @Override
  public void clearLocalCache() {
    if (!closed) { 
      // 清理缓存实现
      localCache.clear();
      localOutputParameterCache.clear();
    }
  }

  private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    // 填充缓存占位
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {

      // 调用子类的doQuery方法，查询数据库
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }

    // put查询结果到缓存中，下次直接命中
    localCache.putObject(key, list);

    // 若查询类型是存储过程，同上
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }

  @Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }

    // 构建缓存Key
    CacheKey cacheKey = new CacheKey();

    // StatementId
    cacheKey.update(ms.getId());

    // RowBounds
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());

    // Raw sql
    cacheKey.update(boundSql.getSql());

    // Parameters.
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        cacheKey.update(value);
      }
    }

    // environment
    if (configuration.getEnvironment() != null) {
      // issue ##176
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }

  // 省略其他代码...
}
```

实现原理非常简单，没有什么高深莫测的骚操作，无非就是对`PerpetualCache`类的操作，再来看看`PerpetualCache`的定义：

```java
public class PerpetualCache implements Cache {

  private final String id;

  private final Map<Object, Object> cache = new HashMap<>();

  public PerpetualCache(String id) {
    this.id = id;
  }

  @Override
  public String getId() {
    return id;
  }

  @Override
  public int getSize() {
    return cache.size();
  }

  @Override
  public void putObject(Object key, Object value) {
    cache.put(key, value);
  }

  @Override
  public Object getObject(Object key) {
    return cache.get(key);
  }

  @Override
  public Object removeObject(Object key) {
    return cache.remove(key);
  }

  @Override
  public void clear() {
    cache.clear();
  }

  //...
}
```

底层就是一个`HashMap`，这里也完全不需要考虑线程安全问题，因为整个`SqlSession`生命周期都不是线程安全的。

继续跟踪代码，通过`Find useage`功能，看看什么地方会调用清理缓存呢？

![063fd9c9-09fc-4993-aac8-f1fd4b2166d3.png][]

清理缓存也相当的简单粗暴，在更新/提交/回滚中直接清理当前会话的全部缓存，没有各种花里胡哨的判断与逻辑，虽然会导致一级缓存命中率不高，但是很好的保证了缓存一致性问题。

![ea1df496-5c5b-484d-8e40-e00f454504ef.png][]

### 一级缓存在Spring中不生效 ###

其实在日常使用中，不会单独使用Mybatis框架，一般都是与Spring进行集成，但是集成后会惊奇的发现，Mybatis的一级缓存好像并没有什么鸟用，这是为什么呢？先看看一段测试。

> Mybatis与Spring集成是使用了另外一个子项目`mybatis-spring`，这里我们集成到我们的测试项目中来，弄个简单的Spring环境。

```java
public class SpringTest {

    @ComponentScan
    static class SpringApp {

        @Bean
        public DataSource dataSource() {
            final MysqlConnectionPoolDataSource dataSource = new MysqlConnectionPoolDataSource();
            dataSource.setUrl("jdbc:mysql://127.0.0.1:3306/mybatis-study");
            dataSource.setUser("root");
            dataSource.setPassword("123456");
            return dataSource;
        }

        @Bean
        public SqlSessionFactory sqlSessionFactory() throws Exception {
            final SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
            sqlSessionFactoryBean.setDataSource(dataSource());
            sqlSessionFactoryBean.setConfigLocation(
                    new PathMatchingResourcePatternResolver().getResource("mybatis-config.xml"));
            return sqlSessionFactoryBean.getObject();
        }

        @Bean
        public UserMapper userMapper() throws Exception {
            SqlSessionTemplate sqlSessionTemplate = new SqlSessionTemplate(sqlSessionFactory());
            return sqlSessionTemplate.getMapper(UserMapper.class);
        }
    }


    public static void main(String[] args) {
        final ApplicationContext ctx = new AnnotationConfigApplicationContext(SpringApp.class);
        final UserMapper userMapper = ctx.getBean(UserMapper.class);
        final User user1 = userMapper.findById1(1);
        final User user2 = userMapper.findById1(1);
        System.out.println(user1 == user2);
    }
}
```

![db83c1ab-6152-4ff7-b025-726020f59122.png][]

从运行日志中很容易就看出一些端倪，由Spring驱动的Mybatis，每次查询都会创建一个新的`SqlSession`对象，自然也就不能命中一级缓存了。

那么如何让Spring集成后也能使用一级缓存呢？也许我们可以尝试自行控制SqlSession对象的创建，比如像这样：

```java
public static void main(String[] args) {
    final ApplicationContext ctx = new AnnotationConfigApplicationContext(SpringApp.class);

    final SqlSessionFactory factory = ctx.getBean(SqlSessionFactory.class);
    try (final SqlSession sqlSession = factory.openSession()) {
        final UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
        final User user1 = userMapper.findById1(1);
        final User user2 = userMapper.findById1(1);
        System.out.println(user1 == user2);
    }
}
```

虽然可行，但并非良策，我都自己获取`SqlSessionFactory`了，那我还要这Spring有用？其实还有一种方法，将连续执行的SQL查询语句用事务控制起来，看代码：

```java
@ComponentScan
static class SpringApp {

    @Bean
    public DataSourceTransactionManager transactionManager() {
        return new DataSourceTransactionManager(dataSource());
    }

    @Bean
    public TransactionTemplate transactionTemplate() {
        return new TransactionTemplate(transactionManager());
    }

    // 省略其他配置...
}


public static void main(String[] args) {
    final ApplicationContext ctx = new AnnotationConfigApplicationContext(SpringApp.class);

    final UserMapper userMapper = ctx.getBean(UserMapper.class);
    final TransactionTemplate tx = ctx.getBean(TransactionTemplate.class);
    tx.execute(transactionStatus ->  {
        final User user1 = userMapper.findById1(1);
        final User user2 = userMapper.findById1(1);
        System.out.println(user1 == user2);

        return null;
    });
}
```

> 这里为了简单起见，用的是编程式事务，实际开发中用声明式事务更为妥当（AOP+TX）

![5ca3520b-601f-4c95-b719-aabf8c412def.png][]

为什么会这样呢？Spring又做了一些什么样的骚操作？其实这是`mybatis-spring`项目的实现，先来看集成后的流程：

![5c84a8c4-3fce-4aa8-b9d3-1b1f20d420bd.png][]

这一系列的封装，为的就是集成Spring后的事务控制，在`SqlSessionInterceptor`（会话拦截器）会去判断两次查询是否在同一个事务中，如果是则会对`SqlSession`进行复用，反之亦然，来看看核心类SqlSessionTemplate的源代码：

```java
// SqlSessionTempate 实现SqlSession，具备SqlSession的功能（由SqlSession代理类完成）
public class SqlSessionTemplate implements SqlSession, DisposableBean {

  // 包装sqlSessionFactory，方便后续使用
  private final SqlSessionFactory sqlSessionFactory;

  // 执行器类型
  private final ExecutorType executorType;

  // SqlSession的代理对象
  private final SqlSession sqlSessionProxy;

  // 异常转换器
  private final PersistenceExceptionTranslator exceptionTranslator;

  public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
    this(sqlSessionFactory, sqlSessionFactory.getConfiguration().getDefaultExecutorType());
  }
  public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType) {
    this(sqlSessionFactory, executorType,
        new MyBatisExceptionTranslator(sqlSessionFactory.getConfiguration().getEnvironment().getDataSource(), true));
  }
  public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sqlSessionFactory, "Property 'sqlSessionFactory' is required");
    notNull(executorType, "Property 'executorType' is required");

    this.sqlSessionFactory = sqlSessionFactory;
    this.executorType = executorType;
    this.exceptionTranslator = exceptionTranslator;

    // 创建SqlSession的代理对象，核心逻辑见SqlSessionInterceptor
    this.sqlSessionProxy = (SqlSession) newProxyInstance(SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class }, new SqlSessionInterceptor());
  }


  // 用SqlSession的代理对象实现SqlSession接口的核心功能。
  @Override
  public <T> T selectOne(String statement) {
    return this.sqlSessionProxy.selectOne(statement);
  }

  // 省略若干实现..


  // 关于事务的提交、回滚以及会话的关闭接口，这里都不再实现，即不可通过手动调用的方式，已经交由spring进行管理。
  @Override
  public void commit() {
    throw new UnsupportedOperationException("Manual commit is not allowed over a Spring managed SqlSession");
  }

  // 省略 rollback / close 等...


  // 针对SqlSession的动态代理拦截器实现
  private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      // 获取SqlSession对象(有事务的情况下，会获取到同一个SqlSession对象，否则重新创建)
      SqlSession sqlSession = SqlSessionUtils.getSqlSession(SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType, SqlSessionTemplate.this.exceptionTranslator);
      try {
        // 调用代理对象的方法
        Object result = method.invoke(sqlSession, args);
        
        // 不在事务中，则执行完毕马上commit
        if (!SqlSessionUtils.isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        // 省略异常处理...
      } finally {
        if (sqlSession != null) {
          SqlSessionUtils.closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }

}


public final class SqlSessionUtils {

  // 省略若干代码...

  public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType,
      PersistenceExceptionTranslator exceptionTranslator) {

    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);
    notNull(executorType, NO_EXECUTOR_TYPE_SPECIFIED);

    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    // 若在事务中，则获取到的SqlSession为同一个对象，具体实现细节不再研究
    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
      return session;
    }

    // 否则创建一个新的SqlSession
    LOGGER.debug(() -> "Creating a new SqlSession");
    session = sessionFactory.openSession(executorType);
    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);
    return session;
  }

  // 判断是否在事务中
  public static boolean isSqlSessionTransactional(SqlSession session, SqlSessionFactory sessionFactory) {
    notNull(session, NO_SQL_SESSION_SPECIFIED);
    notNull(sessionFactory, NO_SQL_SESSION_FACTORY_SPECIFIED);

    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    return (holder != null) && (holder.getSqlSession() == session);
  }

  // 省略若干代码...
}
```

## 二级缓存 ##

二级缓存也被称之为应用级缓存，与一级缓存不同的是，他的作用范围是整个应用的生命周期，且是线程安全的，所以其拥有更高的命中率，相对于一级缓存来说，显得不是那么鸡肋，在Mybatis的缓存体系中，首先是访问二级缓存（如果开启）再访问一级缓存，最后再查询数据库。

### 二级缓存使用示例 ###

二级缓存默认是不开启的，在使用前，需要做一些简单的配置

```xml
// mybatis-config.xml
<settings>
   <setting name="cacheEnabled" value="true"/>
</settings>
// 该配置默认是开启的，因此可以不做配置
```

接下来，需要在Mapper文件中，申明缓存空间（`<cache>`）

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.wuwenze.mybatis.example.UserMapper">
    <cache></cache>
</mapper>
```

除了在Mapper文件中配置XML标签外，亦可通过在Mapper接口上申明`@CacheNamesapce`注解来完成。

```java
@CacheNamespace
public interface UserMapper {
  // ...
}
```

> 关于`<cache/>`标签或者`@CacheNamspce`，还有很多参数，这里不再详细说明，来看第一个单元测试。

```java
public class TwoLevelCacheTest {
    private static SqlSession sqlSession;
    private static SqlSessionFactory sqlSessionFactory;

    @Before
    public void before() throws IOException {
        // 构建SqlSessionFactory
        final InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
        sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        sqlSession = sqlSessionFactory.openSession();
    }

    @After
    public void after() {
        if (null != sqlSession) sqlSession.close();
    }


    @Test
    public void test1() {
        final User user1 = sqlSession.getMapper(UserMapper.class).findById(1);
        System.out.println("user1 = " + user1);
        sqlSession.commit(); // 必须先提交，将暂存区的缓存转义至全局缓存

        // 新开sqlSession，验证是否命中缓存
        try (final SqlSession sqlSession = sqlSessionFactory.openSession()) {
            final User user2 = sqlSession.getMapper(UserMapper.class).findById(1);
            System.out.println("user2 = " + user2);
        }
    }
}
```

![e4743289-debd-4dc2-9de9-bc371f660910.png][]

### 二级缓存的命中场景 ###

![8967d424-6a5b-48a7-b14b-79c610ec0b1a.png][]

二级缓存的命中场景与一级缓存类似（因为`CacheKey`是同一个）但是区别在与二级缓存可以跨`SqlSession`使用，此外，二级缓存在更新或写入后，需要提交会话（即`sqlSession.commit()`），才能被其他会话命中，这是为什么呢？

![0dc8785b-826e-4136-a9c5-b95e896f56e8.png][]

在上图中，我们假设二级缓存由各个会话自行操作，更新完成后，直接修改二级缓存中的数据，那么会话2在修改完数据并填充到二级缓存后，这时会话1获取到的是修改后的数据，而会话2紧接着又回滚了数据，这就导致了会话1获取到的数据是明显的脏数据，遇到这样的情况，非但不好排查，还会导致严重的数据一致性问题。Mybatis对这一块的逻辑封装甚多，咱们稍后再做分析（详见下文**缓存事务管理器**）

### 二级缓存核心功能 ###

Mybatis中的二级缓存是一个完整的解决方案，包含了哪些功能呢？

 *  **存储**

一个成熟的二级缓存框架，绝不局限于将缓存数据存储在内存中，Mybatis默认提供的实现将缓存存储在内存中，除此之外，还提供了无限的扩展可能，你可以将数据存储在硬盘、亦或是第三方中间件（如redis等）

 *  **溢出淘汰机制**

无论是哪种存储，都需要一种机制，当容量快满的时候要进行数据清除，这就是所谓的溢出淘汰算法，目前Mybatis实现了`FIFO（先进先出）`、以及`LRU（最近最少使用）`两种机制。

 *  其他

缓存命中统计、线程安全机制、过期清理机制、写安全机制（保证拿到缓存数据后，可以对其进行修改，而不影响原本的缓存对象实例，通常采用的做法是对象深拷贝）

### 二级缓存责任链设计 ###

如此多的功能，如何设计，才能同时具备简单、灵活、扩展性呢？这点Mybatis设计得很是精妙，来让我们先来跟踪一下代码。

通过前面的内容，我们已经知道了Mybatis的二级缓存是通过`CachingExecutor`装饰`BaseExecutor`的模式来完成的，所以咱们直接跟踪`CachingExecutor.query()`代码。

```java
public class CachingExecutor implements Executor {

  // BaseExcutor委托, (CachingExecutor装饰BaseExecutor，实现BaseExecutor功能的横向扩展)
  private final Executor delegate;

  // 缓存事务管理器（后续再说）
  private final TransactionalCacheManager tcm = new TransactionalCacheManager();

  // 省略若干代码

  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);

    // 构建缓存key，通过委托对象去构建，实际上等于：baseExecutor.createCacheKey()， 即与一级缓存的key一致
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

  @Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    return delegate.createCacheKey(ms, parameterObject, rowBounds, boundSql);
  }

  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {

    // 获取Mapper的缓存空间（@CacheNamespace)
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")

        // 从缓存事务管理器从获取缓存
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {

          // 若无法命中缓存，委托查询数据库结果，并写入缓存事务管理器
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue ##578 and ##116
        }
        return list;
      }
    }

    // 获取不到缓存命名空间，直接委托查询。
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

  @Override
  public void commit(boolean required) throws SQLException {
    delegate.commit(required);
    tcm.commit(); // 提交缓存事务管理器中的数据。
  }
}
```

怎么样，够简单吧？真是奇了怪了，之前提到的那么多功能究竟在哪儿实现的呢？这就不得不赞叹设计模式的强大之处了，继续看。

![df16eb8b-1b25-4ca0-b526-3b2222d26b3c.png][]

关键代码就在`Cache`对象中，这无疑是一个精妙绝伦的设计，通过堆栈分析程序结构，这妥妥的是一个装饰器+责任链模式啊！

![6367e72f-ea82-4a46-a641-183a50a3e14b.png][]![276e4fce-f04e-43f9-9b50-3fa38841391d.png][]

通过这些实现类的类名，咱们看出了一些端倪，每个实现类对应着不同的功能，那么是如何串联起来使用的呢？先来看一张简单的图：

![daec14bc-eeb7-4aea-806a-91fb775caca6.png][]

为了隐藏更多的实现细节，并方便扩展，Mybatis抽象出了`Cache`接口，其中只定义了对缓存的一些基本操作功能，然后上述的每一个功能都会对应一个组件类，并基于装饰者+责任链设计模式，将各个组件进行串联，在执行缓存的基本功能时，每个组件的缓存扩展逻辑，会沿着这个组装好的责任链依次往下执行。

为了便于理解，咱们先来写一个小小的demo，来熟悉这种设计模式的终极用法：

```java
public class ChainTest {

    static interface MessageHandler {
        void handler(String message);
    }

    @RequiredArgsConstructor
    static class CtoMessageHandler implements MessageHandler {
        private final MessageHandler delegate;

        @Override
        public void handler(String message) {
            System.out.println("Cto处理消息：" + message);
            delegate.handler(message);
        }
    }

    @RequiredArgsConstructor
    static class LeaderMessageHandler implements MessageHandler {
        private final MessageHandler delegate;

        @Override
        public void handler(String message) {
            System.out.println("Leader处理消息：" + message);
            delegate.handler(message);
        }
    }

    static class GroupLeaderMessageHandler implements MessageHandler {

        @Override
        public void handler(String message) {
            System.out.println("GroupLeader处理消息：" + message);
        }
    }


    public static void main(String[] args) {
        // 构建责任链
        final MessageHandler handler =
                new CtoMessageHandler(
                        new LeaderMessageHandler(
                                new GroupLeaderMessageHandler()));

        handler.handler("Hello World.");
    }
}


// 运行结果
Cto处理消息：Hello World.
Leader处理消息：Hello World.
GroupLeader处理消息：Hello World.
```

Mybatis这样设计，有以下几个优点：

 *  职责单一：各个组件只负责自己的逻辑，不需要关系其他的组件。
 *  扩展性强：可根据需要扩展节点、删除节点，同时也可以调整各自的顺序来保证灵活性。
 *  松藕合：各组件之间不强制依赖其他节点，而是通过顶层的`Cache`接口来进行串联。
 *  隐藏实现细节：对于外部来说，只需关注`Cache`接口即可，代码结构简单清晰。

了解了基本的原理之后，咱们来看几个核心的二级`Cache`组件，看看都是如何实现其负责的功能的。

#### BlockingCache ####

该组件的为每个`CacheKey`的访问添加阻塞锁，防止缓存被击穿。

```java
public class BlockingCache implements Cache {

  private long timeout;
  // 委托对象，下一个二级缓存处理组件
  private final Cache delegate; 
  private final ConcurrentHashMap<Object, CountDownLatch> locks;

  // 省略其他代码

  @Override
  public void putObject(Object key, Object value) {
    try {
      delegate.putObject(key, value);
    } finally {
      // 释放锁
      releaseLock(key);
    }
  }

  @Override
  public Object getObject(Object key) {
    acquireLock(key); // 尝试获取锁(阻塞状态)
    Object value = delegate.getObject(key);
    if (value != null) {
      releaseLock(key); // 释放锁
    }
    return value;
  }

  // 省略其他代码...

  private void acquireLock(Object key) {
    // 阻塞获取锁，直到抢占到锁或超时（timeout属性）
    CountDownLatch newLatch = new CountDownLatch(1);
    while (true) {
      CountDownLatch latch = locks.putIfAbsent(key, newLatch);
      if (latch == null) {
        break; // 没有锁，直接返回
      }
      try {
        if (timeout > 0) {
          // 获取，直到超时
          boolean acquired = latch.await(timeout, TimeUnit.MILLISECONDS);
          if (!acquired) {
            throw new CacheException(
                "Couldn't get a lock in " + timeout + " for the key " + key + " at the cache " + delegate.getId());
          }
        } else {
          latch.await(); // if timout = -1, 死等
        }
      } catch (InterruptedException e) {
        throw new CacheException("Got interrupted while trying to acquire lock for key " + key, e);
      }
    }
  }

  private void releaseLock(Object key) {
    CountDownLatch latch = locks.remove(key);
    if (latch == null) {
      throw new IllegalStateException("Detected an attempt at releasing unacquired lock. This should never happen.");
    }
    latch.countDown();
  }
}
```

#### SynchronizedCache ####

![29f8a1c7-0580-4d1b-8a56-c47a5a333522.png][]

给核心的增删方法强制加上了`synchronized`关键字保证操作缓存时的安全性，简单且粗暴，由于每个Mapper对应一个`Cache`对象，因此没有必要进行其他花里胡哨的操作。

#### LoggingCache ####

该类顾名思义，对缓存的命中操作进行日志输出，代码非常的简单

![7a50fbbc-e229-43ab-9b7e-95ffaabafc63.png][]

#### LruCache ####

```java
public class LruCache implements Cache {

  private final Cache delegate;
  private Map<Object, Object> keyMap;
  private Object eldestKey;

  public LruCache(Cache delegate) {
    this.delegate = delegate;
    setSize(1024); // 默认缓存1024个对象
  }

  public void setSize(final int size) {
    // 利用LinkeHashMap特性，来实现最少使用移除（关键参数accessOrder=true)
    keyMap = new LinkedHashMap<Object, Object>(size, .75F, true) {
      private static final long serialVersionUID = 4267176411845948333L;
      @Override
      protected boolean removeEldestEntry(Map.Entry<Object, Object> eldest) {
        boolean tooBig = size() > size;
        if (tooBig) {
      // 如果当前map里面的元素个数大于了缓存最大容量，则删除顶端元素
          eldestKey = eldest.getKey();
        }
        return tooBig;
      }
    };
  }

  @Override
  public void putObject(Object key, Object value) {
    delegate.putObject(key, value);
    cycleKeyList(key); // 每次写入缓存，需要将key写入keyMap，并且移除顶端最少使用的元素
  }

  @Override
  public Object getObject(Object key) {
    keyMap.get(key); // 同理，每次访问缓存时，也需要访问一下缓存的key
    return delegate.getObject(key);
  }

  // 省略其他代码...

  private void cycleKeyList(Object key) {
    keyMap.put(key, key); // 将CacheKey写入，记数
    if (eldestKey != null) {
      // 删除顶端元素
      delegate.removeObject(eldestKey);
      eldestKey = null;
    }
  }

}
```

关于`LRU`算法的实现，这个有点意思，咱们再来单独测试一下，以便更好的掌握。

```java
public static void main(String[] args) {
    final int capacity = 3; // 缓存的最大容量
    final Map<String, String> cacheMap = new LinkedHashMap<String, String>
            (capacity, 0.75F, true) {
        // 如果map里面的元素个数大于了缓存最大容量，则删除顶端元素
        @Override
        protected boolean removeEldestEntry(Map.Entry<String, String> eldest) {
            return size() > capacity;
        }
    };

    // 初始化3个缓存
    cacheMap.put("key1", "value1");
    cacheMap.put("key2", "value2");
    cacheMap.put("key3", "value3");

    // {key1=value1, key2=value2, key3=value3}
    System.out.println(cacheMap);

    // 访问3次key1, 则key1排到最后
    for (int i = 0; i < 3; i++) {
        cacheMap.get("key1");
    }
    // {key2=value2, key3=value3, key1=value1}
    System.out.println(cacheMap);

    // 再加入1个key4，则栈顶对象（即最少使用的key2）被移除
    cacheMap.put("key4","value4");
    // {key3=value3, key1=value1, key4=value4}
    System.out.println(cacheMap);
}
```

#### FifoCache ####

与`LRU`算法对应的，则是`FIFO`（先进先出），Mybatis默认是使用`LRU`算法的，可以通过配置`eviction`属性来切换

```java
@CacheNamespace(eviction = FifoCache.class)
public interface UserMapper {
  //...
}
```

![fe3e85d3-6ba3-4374-9088-ac8f8f41b2c6.png][]

除此之外，还有一些其他的二级缓存组件，由于篇幅原因，这里就不再详细说明了。

最终责任链会到`PerpetualCache`来实现内存缓存（与一级缓存是同一个对象）当然，这也是可以进行替换的，通过`implementation`属性来配置责任链的最终实现类。

```java
@CacheNamespace(
        eviction = FifoCache.class, 
        implementation = YourStoreCache.class 
)
public interface UserMapper {
    //....
}
```

### 二级缓存事务管理器（暂存区） ###

为了保证多会话的情况下，缓存的一致性问题，Mybatis为每个会话都设立了若干个暂存区，当前会话对指定缓存空间的变更操作，都会临时存放在暂存区内，只有会话被提交了之后才对更新对应的全局缓存空间，为了管理这些暂存区，因此每个会话内部都会有一个唯一的事务缓存管理器（即：`TransactionalCacheManager`）

![9a22cc00-d75a-487c-b4a1-39b433dc8721.png][]

先来一张图了解一下会话、暂存区、二级缓存空间之间的关系：

![002324d0-44f3-430d-84ff-95d7ba4c1461.png][]

每个`CacheingExecutor`中，都有一个`TransactionalCacheManager`对象，对二级缓存的操作，实际上是操作暂存区的缓存，来看代码：

```java
public class CachingExecutor implements Executor {

  private final Executor delegate;

  // 创建事务缓存管理器
  private final TransactionalCacheManager tcm = new TransactionalCacheManager();


  @Override
  public int update(MappedStatement ms, Object parameterObject) throws SQLException {
    flushCacheIfRequired(ms); // 每次更新前，先清理缓存（配置flushCache后）
    return delegate.update(ms, parameterObject);
  }

  // 省略若干代码...

  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")

        // 从暂存区获取缓存
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);

          // 写入缓存到暂存区
          tcm.putObject(cache, key, list); // issue ##578 and ##116
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }


  @Override
  public void commit(boolean required) throws SQLException {
    delegate.commit(required);
    tcm.commit(); // 手动提交暂存区的缓存
  }


  @Override
  public void close(boolean forceRollback) {
    try {
      // 会话关闭时，提交或者回滚暂存区的缓存
      // issues ##499, ##524 and ##573
      if (forceRollback) {
        tcm.rollback();
      } else {
        tcm.commit();
      }
    } finally {
      delegate.close(forceRollback);
    }
  }
}
```

对二级缓存的操作，都经过了事务管理器的包装，再来看看事务管理器的内部代码：

```java
public class TransactionalCacheManager {

  // 这就是所谓的暂存区，每个缓存对象对应一个TransactionalCache
  private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<>();


  // 从暂存区获取对应的TransactionalCache对象，没有则新建
  private TransactionalCache getTransactionalCache(Cache cache) {
    return MapUtil.computeIfAbsent(transactionalCaches, cache, TransactionalCache::new);
  }

  // 获取、设值、清空、提交、回滚等操作

  public void clear(Cache cache) {
    getTransactionalCache(cache).clear();
  }

  public Object getObject(Cache cache, CacheKey key) {
    return getTransactionalCache(cache).getObject(key);
  }

  public void putObject(Cache cache, CacheKey key, Object value) {
    getTransactionalCache(cache).putObject(key, value);
  }

  public void commit() {

    // 暂存区存在多个命名空间的缓存，所以要循环提交
    for (TransactionalCache txCache : transactionalCaches.values()) {
      txCache.commit();
    }
  }

  public void rollback() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
      txCache.rollback();
    }
  }


}


// TransactionalCache实现自Cache接口，拥有全部行为，同时也是二级缓存责任链中的一个组件
public class TransactionalCache implements Cache {

  private static final Log log = LogFactory.getLog(TransactionalCache.class);

  private final Cache delegate;
  private boolean clearOnCommit;

  // 调用commit时需要添加的对象(即暂存的缓存对象，key，value)
  private final Map<Object, Object> entriesToAddOnCommit;

  // 未命中缓存时的key（说明二级缓存空间中不存在该数据）
  private final Set<Object> entriesMissedInCache;

  // 省略若干代码..

  @Override
  public Object getObject(Object key) {
    // issue ##116

    // 获取缓存时，直接用委托对象获取二级缓存空间中的真实值（无需操作暂存区）
    Object object = delegate.getObject(key);
    if (object == null) {
      
      // 若未命中，记录key备用
      entriesMissedInCache.add(key);
    }
    // issue ##146
    if (clearOnCommit) {
      return null;
    } else {
      return object;
    }
  }

  @Override
  public void putObject(Object key, Object object) {
    entriesToAddOnCommit.put(key, object); // 加入暂存
  }

  @Override
  public Object removeObject(Object key) {
    return null;
  }

  @Override
  public void clear() {
    clearOnCommit = true;
    entriesToAddOnCommit.clear();
  }

  public void commit() {
    if (clearOnCommit) {
      delegate.clear();
    }

    // 提交时，处理暂存区的数据
    flushPendingEntries();

    // 重置暂存区
    reset();
  }

  private void flushPendingEntries() {

    // 将暂存区数据put到真实的二级缓存空间（使用委托缓存对象）
    for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
      delegate.putObject(entry.getKey(), entry.getValue());
    }

    // 未命中记录的部分，提交时也要再进行清空值
    for (Object entry : entriesMissedInCache) {
      if (!entriesToAddOnCommit.containsKey(entry)) {
        delegate.putObject(entry, null);
      }
    }
  }

  private void unlockMissedEntries() {
    // 移除真实的二级缓存空间中的指定key
    for (Object entry : entriesMissedInCache) {
      try {
        delegate.removeObject(entry);
      } catch (Exception e) {
        log.warn("Unexpected exception while notifying a rollback to the cache adapter. "
            + "Consider upgrading your cache adapter to the latest version. Cause: " + e);
      }
    }
  }

  public void rollback() {
    unlockMissedEntries();
    reset();
  }

  private void reset() {
    clearOnCommit = false;
    entriesToAddOnCommit.clear();
    entriesMissedInCache.clear();
  }

}
```

为了保证二级缓存的一致性，Mybatis封装了很多逻辑，其本质上就是将每个会话的二级缓存操作加上事物包装。

### 总结：二级缓存执行流程 ###

最后，对二级缓存部分，再来做一个简单的总结，在没有使用二级缓存之前，会话是通过`BaseExecutor`去实现SQL查询调用，当开启二级缓存后，则基于装饰器模式使用`CachingExecutor`对`BaseExecutor`的调用逻辑进行拦截处理，嵌入了所有的二级缓存逻辑。

![2b6e573a-52ce-4c01-ad57-d772536bbdfe.png][]

 *  查询操作

当会话调用`query()`时，生成相关的`CacheKey`，然后尝试从二级缓存中读取数据，读取到就直接返回，防止则调用被装饰的`Executor`去查询数据库，然后再填充到对应的暂存区（请注意，这里的查询是实时从二级缓存空间读取的）

 *  更新操作

当会话调用`update()`时，同样会生成相关的`CacheKey`，然后再执行update之前清空缓存（只针对暂存区）同时计入清空的标记，以便于会话提交之时，依据该标记去清理二级缓存空间的数据（另外，如果在查询操作配置了`flushCache`属性的话，也会执行相同的操作）

 *  提交操作

当会话调用`commit()`操作时，会将该会话下的所有暂存区的变更，更新到对应的二级缓存空间中去。


[88f83e04-b284-4abc-b68c-0c013a1616eb.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/88f83e04-b284-4abc-b68c-0c013a1616eb.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[856f9684-5c1a-4220-8f9a-b4b4949c9c82.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/856f9684-5c1a-4220-8f9a-b4b4949c9c82.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[d2573e6f-60d5-422f-aa5b-b235b5d4f9b2.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/d2573e6f-60d5-422f-aa5b-b235b5d4f9b2.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[98bc8b42-8963-4a7b-acc3-f0cf5fd2fa03.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/98bc8b42-8963-4a7b-acc3-f0cf5fd2fa03.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[751b1566-a573-436d-b641-468e9245674e.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/751b1566-a573-436d-b641-468e9245674e.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[063fd9c9-09fc-4993-aac8-f1fd4b2166d3.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/063fd9c9-09fc-4993-aac8-f1fd4b2166d3.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[ea1df496-5c5b-484d-8e40-e00f454504ef.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/ea1df496-5c5b-484d-8e40-e00f454504ef.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[db83c1ab-6152-4ff7-b025-726020f59122.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/db83c1ab-6152-4ff7-b025-726020f59122.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[5ca3520b-601f-4c95-b719-aabf8c412def.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/5ca3520b-601f-4c95-b719-aabf8c412def.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[5c84a8c4-3fce-4aa8-b9d3-1b1f20d420bd.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/5c84a8c4-3fce-4aa8-b9d3-1b1f20d420bd.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[e4743289-debd-4dc2-9de9-bc371f660910.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/e4743289-debd-4dc2-9de9-bc371f660910.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[8967d424-6a5b-48a7-b14b-79c610ec0b1a.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/8967d424-6a5b-48a7-b14b-79c610ec0b1a.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[0dc8785b-826e-4136-a9c5-b95e896f56e8.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/0dc8785b-826e-4136-a9c5-b95e896f56e8.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[df16eb8b-1b25-4ca0-b526-3b2222d26b3c.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/df16eb8b-1b25-4ca0-b526-3b2222d26b3c.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[6367e72f-ea82-4a46-a641-183a50a3e14b.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/6367e72f-ea82-4a46-a641-183a50a3e14b.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[276e4fce-f04e-43f9-9b50-3fa38841391d.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/276e4fce-f04e-43f9-9b50-3fa38841391d.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[daec14bc-eeb7-4aea-806a-91fb775caca6.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/daec14bc-eeb7-4aea-806a-91fb775caca6.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[29f8a1c7-0580-4d1b-8a56-c47a5a333522.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/29f8a1c7-0580-4d1b-8a56-c47a5a333522.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[7a50fbbc-e229-43ab-9b7e-95ffaabafc63.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/7a50fbbc-e229-43ab-9b7e-95ffaabafc63.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[fe3e85d3-6ba3-4374-9088-ac8f8f41b2c6.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/fe3e85d3-6ba3-4374-9088-ac8f8f41b2c6.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[9a22cc00-d75a-487c-b4a1-39b433dc8721.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/9a22cc00-d75a-487c-b4a1-39b433dc8721.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[002324d0-44f3-430d-84ff-95d7ba4c1461.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/002324d0-44f3-430d-84ff-95d7ba4c1461.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2b6e573a-52ce-4c01-ad57-d772536bbdfe.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210804/2b6e573a-52ce-4c01-ad57-d772536bbdfe.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg