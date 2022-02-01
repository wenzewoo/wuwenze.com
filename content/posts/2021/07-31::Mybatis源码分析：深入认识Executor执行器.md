+++
title = "Mybatis源码分析：深入认识Executor执行器"
date = "2021-07-31 13:44:41"
url = "archives/918"
tags = ["MyBatis"]
categories = ["后端","Mybatis源码阅读"]
featuredImage = "https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210923/bb73be20852d4c1392cecf5eb60fee7a.jpg"
+++

本文的目的是疏通Mybatis的整体执行流程，并重点理解Executor在其中扮演的重要作用。

## JDBC执行过程 ##

在开始了解`mybatis`之前，有必要先回顾一下`JDBC`的整体流程，因为`mybatis`的底层实际上就是`JDBC`。

```java
final List<User> result = new ArrayList<>();
final String jdbcUrl = "jdbc:mysql://127.0.0.1:3306/test";

// 获取连接 & 预编译SQL
try (final Connection conn = DriverManager.getConnection(jdbcUrl, "root", "root");
     final PreparedStatement preparedStatement = conn.prepareStatement("SELECT id,name,age FROM user WHERE age > ?")) {
    preparedStatement.setInt(1, 18);

    // 执行 & 获取resultSet
    final ResultSet resultSet = preparedStatement.executeQuery();

    // 将结果转换为Bean
    while (resultSet.next()) {
        result.add(new User(
                resultSet.getInt("id"),
                resultSet.getString("name"),
                resultSet.getInt("age")));
    }

} catch (SQLException e) {
    e.printStackTrace();
}

System.out.println(result);
```

![b6b12f48-e972-4c19-b58b-b37ad7031737.png][]

这里着重说明一下`Statement`，通过该组件来实现SQL以及参数的设置，上图中`PreparedStatement`的实际上是`Statement`的其中一个子类，除此之外，常见的实现类如下：

 *  `Statement` \- 简单执行器
 *  `PreparedStatement` \- 预处理执行器
 *  `CallableStatement` \- 存储过程执行器

![e78f26b7-9019-4826-a8a5-aff7d560eac6.png][]

可以看到几个`Statement`之间存在着父子关系，即意味着预编译执行器以及存储过程拥有`Statement`的全部功能。

`Statement`中有几个功能对性能提升尤为重要，这在之后的mybatis源码中也有所体现：

 *  `addBatch();` 批处理操作，将多个SQL合并在一起，最后调用`executeBatch()`一起发送至数据库执行。
 *  `setFetchSize();` 设置从数据库每次读取的数量单位。该举措是为了防止一次性从数据库加载数据过多，导致内存溢出。

## Mybatis执行过程概览 ##

咱们知道Mybatis基于JDBC实现封装，那么Mybatis又是如何调用JDBC，来完成一些常规的增删改查操作的呢？

在此之前，咱们先来写一段最基础的mybatis使用示例（暂不考虑spring集成）参考链接：[mybatis – MyBatis 3 | 入门][mybatis _ MyBatis 3 _]

```java
public class MybatisExampleTest {
    private static SqlSessionFactory sqlSessionFactory;

    @BeforeClass
    public static void before() throws IOException {
        final InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
        sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
    }


    @Test
    public void test1() {
        try (final SqlSession sqlSession = sqlSessionFactory.openSession()) {
            final UserMapper mapper = sqlSession.getMapper(UserMapper.class);
            System.out.println(mapper.findById(1));
        }
    }
}
```

![525bfe8f-a081-4b42-8f2d-f513690d744f.png][]

这里先大致的了解一下基础组件的概念，后面咱们再深入的跟进源代码：

 *  `SqlSession：`提供增删改查API，其本身不作任何业务逻辑的处理，所有处理都交给执行器。这是一个典型的门面模式设计。
 *  `Executor：`处理SQL请求、事物管理、维护缓存以及批处理等 。执行器在的角色更像是一个管理员，接收SQL请求，然后根据缓存、批处理等逻辑来决定如何执行这个SQL请求。并交给JDBC处理器执行具体SQL。
 *  `StatementHandler：`通过JDBC具体处理SQL和参数的。在会话中每调用一次CRUD，JDBC处理器就会生成一个实例与之对应（命中缓存除外）

一个SQL请求通过会话到达执行器，然后交给对应的JDBC处理器进行处理（注意三个组件之间的对应关系是1：1：N）。此外所有的组件都不是线程安全的，不能跨线程使用。

### SqlSessionFactory ###

从名称上不难看出，`SqlSessionFactory`是`SqlSession`的工厂类，用于创建并初始化`SqlSession`对象。

1.  调用`openSession()`方法，创建`SqlSession`对象。
2.  `openSession()`方法有多个重载，可以指定`SqlSession`自动提交事务、指定`ExecutorType`，生成不同类型的`Executor`，指定不同的数据库事务隔离级别等等。

```java
public interface SqlSessionFactory {

  SqlSession openSession();

  SqlSession openSession(boolean autoCommit);

  SqlSession openSession(Connection connection);

  SqlSession openSession(TransactionIsolationLevel level);

  SqlSession openSession(ExecutorType execType);

  SqlSession openSession(ExecutorType execType, boolean autoCommit);

  SqlSession openSession(ExecutorType execType, TransactionIsolationLevel level);

  SqlSession openSession(ExecutorType execType, Connection connection);

  Configuration getConfiguration();

}
```

默认情况下，一般使用`sqlSessionFactory.openSession()`来创建`SqlSession`对象，在`SqlSessionFactory`的默认实现类`DefaultSqlSessionFactory`中可以看到具体的代码实现。

```java
public class DefaultSqlSessionFactory implements SqlSessionFactory {

  @Override
  public SqlSession openSession() {
     // executorType = ExecutorType.SIMPLE; 其声明在Configuration
     // autoCommit = false; 即不开启事务自动提交
     // level = null; 即使用默认的事务隔离级别

    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }


  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      // 事务相关
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);

      // 根据executorType创建Executor对象，不同的类型对应不同的Executor实现类
      final Executor executor = configuration.newExecutor(tx, execType);

      // 生成SqlSession实例
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
  

  // 省略若干代码...
}
```

### SqlSession ###

`SqlSession`可以理解为会话（非数据库连接）是一个接口，其定义了很多操作数据库的方法：

```java
public interface SqlSession extends Closeable {

  // 各种查询方法的重载
  <T> T selectOne(String statement);
  <T> T selectOne(String statement, Object parameter);
  <E> List<E> selectList(String statement);
  <E> List<E> selectList(String statement, Object parameter);
  <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds);
  <K, V> Map<K, V> selectMap(String statement, String mapKey);
  <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey);
  <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds);
  <T> Cursor<T> selectCursor(String statement);
  <T> Cursor<T> selectCursor(String statement, Object parameter);
  <T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds);
  void select(String statement, Object parameter, ResultHandler handler);
  void select(String statement, ResultHandler handler);
  void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler);

  // 插入方法的重载
  int insert(String statement);
  int insert(String statement, Object parameter);

  // 更新方法的重载
  int update(String statement);
  int update(String statement, Object parameter);

  // 删除方法的重载
  int delete(String statement);
  int delete(String statement, Object parameter);

  // 事务的提交与回滚
  void commit();
  void commit(boolean force);
  void rollback();
  void rollback(boolean force);

  // 关闭SqlSession
  void close();

  // 获取数据库连接
  Connection getConnection();

  // 清理一级缓存
  void clearCache();

  // 省略若干声明...
}
```

如上所示，SqlSession只是定义了一些执行SQL的API，默认情况下，具体的实现由子类（DefaultSqlSession）来完成。

![02480ee2-e0c4-438c-aa17-c80af99b4755.png][]

### Executor ###

在上文中，使用`SqlSessionFactory.openSession()`时，我们注意到创建了一个`Executor`对象，该对象是咱们本文的重点研究对象，先来看看具体的创建代码吧：

```java
// DefaultSqlSessionFactory#openSessionFromDataSource()
// 根据executorType创建Executor对象，不同的类型对应不同的Executor实现类
final Executor executor = configuration.newExecutor(tx, execType);

// Configuration#newExecutor()
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
        executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
        executor = new ReuseExecutor(this, transaction);
    } else {
        executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
        executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
}
```

`Executor`根据`executorType`的不同，分别实例化了不同的实现类，大致分为两类，缓存执行器与非缓存执行器（使用哪个执行器是通过配置文件中settings下的属性defaultExecutorType控制的，默认是SIMPLE），是否使用缓存执行器则是通过执行cacheEnabled控制的，默认是true。

```java
public interface Executor {
  // 更新
  int update(MappedStatement ms, Object parameter) throws SQLException;

  // 查询函数重载
  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;
  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;
  <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException;

  // 刷新批处理器
  List<BatchResult> flushStatements() throws SQLException;

  // 事务提交与回滚
  Transaction getTransaction();
  void commit(boolean required) throws SQLException;
  void rollback(boolean required) throws SQLException;

  // 一级缓存相关
  CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql);
  boolean isCached(MappedStatement ms, CacheKey key);
  void clearLocalCache();

  // 省略若干代码..
}
```

  
缓存执行器不是真正功能上独立的执行器，而是非缓存执行器的装饰器模式。

## BaseExecutor的3个实现类 ##

![a894a61e-2431-4345-a907-ec71f44b7433.png][]

从实现类的结构来看，大体分为2类，`BaseExecutor`和`CachingExecutor`，这里先重点研究`BaseExecutor`的3个实现类。

![b5294ae1-8028-44f1-9695-1dad034e4a9d.png][]

### 基础执行器：BaseExecutor ###

![a83d4c25-f5e2-4680-9336-40865daab49c.png][]

我们先来看下`BaseExecutor`的属性，从上述`BaseExecutor`的定义可以看出：

 *  执行器在特定的事务上下文下执行；
 *  具有本地缓存和本地出参缓存（任何时候，只要事务提交或者回滚或者执行update或者查询时设定了刷新缓存，都会清空本地缓存和本地出参缓存）；
 *  具有延迟加载任务；

除此之外，`BaseExecutor`还实现了大部分通用功能本地缓存管理、事务提交、回滚、超时设置、延迟加载等，但是将下列4个方法留给了具体的子类实现：

![b2a712c4-f1c3-496d-a374-c39a5f71045b.png][]

涉及到具体查询、更新等操作，均交由了子类自行实现（声明为抽象方法）

### 简单执行器：SimpleExecutor ###

简单执行器继承于`BaseExecutor`，其主要是实现了`BaseExecutor`上文提供的4个抽象方法，仅此而已。

![a9666b72-fa88-4222-9da6-8031d2815715.png][]

为方便进一步的比较后续3种执行器的区别，咱们尝试写一些单元测试来验证其执行的效果。

```java
package com.wuwenze.mybatis.example;

import org.apache.ibatis.executor.SimpleExecutor;
import org.apache.ibatis.io.Resources;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.session.Configuration;
import org.apache.ibatis.session.RowBounds;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;
import org.apache.ibatis.transaction.jdbc.JdbcTransaction;
import org.junit.Before;
import org.junit.Test;

import java.io.IOException;
import java.io.InputStream;
import java.sql.SQLException;
import java.util.List;

public class ExecutorTest {
    private static Configuration configuration;
    private static JdbcTransaction jdbcTransaction;


    @Before
    public void before() throws IOException {
        // 构建SqlSessionFactory
        final InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);

        configuration = sqlSessionFactory.getConfiguration();
        jdbcTransaction = new JdbcTransaction(sqlSessionFactory.openSession().getConnection());
    }

    @Test
    public void simpleExecutorTest() throws SQLException {
        // 构建SimpleExecutor对象
        final SimpleExecutor simpleExecutor = new SimpleExecutor(configuration, jdbcTransaction);

        // 执行doQuery查询，入参为：
        // - MappedStatement： Sql的映射对象
        // - parameter: SQL执行参数
        // - RowBounds: 分页查询对象
        // - ResultHandler: 结果处理器
        // - BoundSql: 动态SQL
        final MappedStatement mappedStatement = configuration.getMappedStatement("com.wuwenze.mybatis.example.UserMapper.findById");
        final List<Object> result = simpleExecutor.doQuery(
                mappedStatement, 1, RowBounds.DEFAULT, SimpleExecutor.NO_RESULT_HANDLER, null);
        System.out.println(result);


        // 再执行一次
        final List<Object> result2 = simpleExecutor.doQuery(
                mappedStatement, 1, RowBounds.DEFAULT, SimpleExecutor.NO_RESULT_HANDLER, null);
        System.out.println(result2);
    }
}
```

![ea1223a7-6f75-467b-bfc2-73a7013fbf01.png][]

从打印的日志来看，`SimpleExecutor`非常简单，完全相同的SQL语句皆进行了一次编译处理，这是不必要的性能浪费，如果SQL完全相同，我们只需要一次编译，多次执行即可，当然，这就需要`ReuseExecutor`（可重用执行器）出场了。

### 可重用执行器：ReuseExecutor ###

```java
public class ReuseExecutor extends BaseExecutor {

  // JDBC Statment 对象的缓存
  private final Map<String, Statement> statementMap = new HashMap<>();

  public ReuseExecutor(Configuration configuration, Transaction transaction) {
    super(configuration, transaction);
  }

  @Override
  public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    // 省略...
  }

  @Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Configuration configuration = ms.getConfiguration();
    StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);

    // 构建Statment对象（优先命中缓存）
    Statement stmt = prepareStatement(handler, ms.getStatementLog());
    return handler.query(stmt, resultHandler);
  }

  @Override
  protected <E> Cursor<E> doQueryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds, BoundSql boundSql) throws SQLException {
   // 省略...
  }

  @Override
  public List<BatchResult> doFlushStatements(boolean isRollback) {
    // 并非执行一次SQL关闭一次Statment对象，
    // 而是等SQL全部执行完成后触发doFlushStatements()再进行批量关闭
    for (Statement stmt : statementMap.values()) {
      closeStatement(stmt);
    }
    // 清空Statment缓存
    statementMap.clear();
    return Collections.emptyList();
  }

  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    BoundSql boundSql = handler.getBoundSql();
    String sql = boundSql.getSql();

    // 从缓存中读取
    if (hasStatementFor(sql)) {
      stmt = getStatement(sql);
      applyTransactionTimeout(stmt);
    } else {

      // 否则重新构建，并加入缓存
      Connection connection = getConnection(statementLog);
      stmt = handler.prepare(connection, transaction.getTimeout());
      putStatement(sql, stmt);
    }
    handler.parameterize(stmt);
    return stmt;
  }

  private boolean hasStatementFor(String sql) {
    try {
      Statement statement = statementMap.get(sql);
      return statement != null && !statement.getConnection().isClosed();
    } catch (SQLException e) {
      return false;
    }
  }

  private Statement getStatement(String s) {
    return statementMap.get(s);
  }

  private void putStatement(String sql, Statement stmt) {
    statementMap.put(sql, stmt);
  }

}
```

从实现可以看出，`ReuseExecutor`和`SimpleExecutor`在`doUpdate()`/`doQuery()`上有个差别，不再是每执行一个语句就`close()`掉了，而是尽可能的根据SQL文本进行缓存并重用。

当然，该缓存非常简单，基于`HashMap`实现，key为SQL语句，value则为`Statment`对象的实例（当然，缓存仅会存在于`SqlSession`的生命周期中）。其核心是利用JDBC的特性，来看一段JDBC的伪代码：

```java
final String sql = "SELECT * FROM user WHERE id = ?";
// 一次编译
PrepareStatment pstmt = conn.prepareStatment(sql);

// 第一次执行
pstmt.setInt(1, 1);
pstmt.executeQuery();

// 第二次执行
pstmt.setInt(1, 2);
pstmt.executeQuery();
```

为了验证这个实现原理，咱们继续新增单元测试

```java
@Test
public void reuseExecutorTest() throws SQLException {
    final ReuseExecutor reuseExecutor = new ReuseExecutor(configuration, jdbcTransaction);

    final MappedStatement mappedStatement = configuration.getMappedStatement("com.wuwenze.mybatis.example.UserMapper.findById");
    final List<Object> result = reuseExecutor.doQuery(
            mappedStatement, 1, RowBounds.DEFAULT, SimpleExecutor.NO_RESULT_HANDLER, null);
    System.out.println(result);

    // 再执行一次
    final List<Object> result2 = reuseExecutor.doQuery(
            mappedStatement, 1, RowBounds.DEFAULT, SimpleExecutor.NO_RESULT_HANDLER, null);
    System.out.println(result2);
}
```

![e2f0f652-5c56-4fba-963e-35c5ec3e8238.png][]

由此可见，所谓的可重用执行器，其实就是将`Stetament`缓存起来，实现了SQL的一次编译，多次执行。此外，需要特别注意的是，该缓存会随着`SqlSession`的生命周期结束而销毁，并非全局缓存。

### 批处理执行器：BatchExecutor ###

批处理执行器其实很好理解，将所有SQL请求集中起来，最后调用某个方法，一次性将所有请求发送至数据库。

对于批量的`INSERT` / `UPDATE`语句而言，能大幅的提升更新语句的性能（主要是减少的网络开销）在研究Mybatis的`BatchExecutor`实现之前，我们先来回顾一下JDBC的批处理实现，以下为伪代码。

```java
final String sql = "UPDATE user SET name = ? WHERE id = ?";
try (final PreparedStatement pstmt = conn.prepareStatement(sql)) {

    for (int i = 0; i < N; i++) {
        pstmt.setString(1, xxx);
        pstmt.setInt(2, xxx);

        // 加入批处理
        pstmt.addBatch();

        // 当达到1000条，则执行一次
        if ((i + 1) % 1000 == 0) {
            pstmt.executeBatch();
            pstmt.clearBatch();
        }
    }
    pstmt.executeBatch();
    pstmt.clearBatch();
}
```

那么`BatchExecutor`也是利用了`JDBC Statement.addBatch()`特性吗？不一定。

先来观察`BatchExecutor`的源码实现，由于批处理只针对`UPDATE` / `INSERT` / `DELETE`等更新语句，所以咱们只需要关注`BatchExecutor.doUpdate()`以及`doFlushStatements()`方法即可。

```java
public class BatchExecutor extends BaseExecutor {
  public static final int BATCH_UPDATE_RETURN_VALUE = Integer.MIN_VALUE + 1002;

  // 当前批处理执行器的Satement对象合集
  private final List<Statement> statementList = new ArrayList<>();

  // 与之对应的批处理执行结果集
  private final List<BatchResult> batchResultList = new ArrayList<>();

  // 当前的SQL语句
  private String currentSql;

  // 当前的MappedStatement对象
  private MappedStatement currentStatement;

  public BatchExecutor(Configuration configuration, Transaction transaction) {
    super(configuration, transaction);
  }

  @Override
  public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
    final Configuration configuration = ms.getConfiguration();
    final StatementHandler handler = configuration.newStatementHandler(this, ms, parameterObject, RowBounds.DEFAULT, null, null);
    final BoundSql boundSql = handler.getBoundSql();
    final String sql = boundSql.getSql();
    final Statement stmt;

    // 判断当前的SQL以及MappedStatement是否与记录的一致，为了确保连续的才可以进行批处理
    if (sql.equals(currentSql) && ms.equals(currentStatement)) {
      int last = statementList.size() - 1;
      stmt = statementList.get(last); // 获取列表中的最后一个statement
      applyTransactionTimeout(stmt);
      handler.parameterize(stmt);// fix Issues 322
      BatchResult batchResult = batchResultList.get(last);
      batchResult.addParameterObject(parameterObject);
    } else {
      // 不满足条件，则构建新的Statement
      Connection connection = getConnection(ms.getStatementLog());
      stmt = handler.prepare(connection, transaction.getTimeout());
      handler.parameterize(stmt);    // fix Issues 322

      // 设置当前SQL以及MappedStatement
      currentSql = sql;
      currentStatement = ms;
      statementList.add(stmt);
      batchResultList.add(new BatchResult(ms, sql, parameterObject));
    }
    // 不执行实际的update操作，而是通过StatementHandler去调用Statement.addBatch();
    handler.batch(stmt);

    // 返回值不再是受影响的行数，这里返回一个固定的值。
    return BATCH_UPDATE_RETURN_VALUE;
  }

  @Override
  public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
    try {
      List<BatchResult> results = new ArrayList<>();
      if (isRollback) { // 若事物回滚，则没必要执行提交，直接返回空
        return Collections.emptyList();
      }

      // 遍历当前批处理执行器的所有Statment，每个依次去执行pstmt.executeBatch()以及保存执行结果操作。
      for (int i = 0, n = statementList.size(); i < n; i++) {
        Statement stmt = statementList.get(i);
        applyTransactionTimeout(stmt);
        BatchResult batchResult = batchResultList.get(i);
        try {
          // 调用jdbc pstmt.executeBatch()方法，提交已经发送的SQL
          batchResult.setUpdateCounts(stmt.executeBatch());

          // 回写主键相关，具体细节暂时不关注
          MappedStatement ms = batchResult.getMappedStatement();
          List<Object> parameterObjects = batchResult.getParameterObjects();
          KeyGenerator keyGenerator = ms.getKeyGenerator();
          if (Jdbc3KeyGenerator.class.equals(keyGenerator.getClass())) {
            Jdbc3KeyGenerator jdbc3KeyGenerator = (Jdbc3KeyGenerator) keyGenerator;
            jdbc3KeyGenerator.processBatch(ms, stmt, parameterObjects);
          } else if (!NoKeyGenerator.class.equals(keyGenerator.getClass())) { //issue ##141
            for (Object parameter : parameterObjects) {
              keyGenerator.processAfter(this, ms, stmt, parameter);
            }
          }
          // 提交完毕后，关闭Statement对象
          // Close statement to close cursor ##1109
          closeStatement(stmt);
        } catch (BatchUpdateException e) {
          // 省略错误处理代码..
          throw new BatchExecutorException(message.toString(), e, results, batchResult);
        }
        results.add(batchResult);
      }
      return results;
    } finally {

      // 在finally中再关一次Statement
      for (Statement stmt : statementList) {
        closeStatement(stmt);
      }

      // 清理相关的缓存
      currentSql = null;
      statementList.clear();
      batchResultList.clear();
    }
  }
  
  // 省略其他代码..
}


public class PreparedStatementHandler extends BaseStatementHandler {

  @Override
  public void batch(Statement statement) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.addBatch();
  }

  // 省略其他代码...
}
```

#### 批处理执行器结论 ####

通过阅读相关的源码，我们可以做以下总结，在Mybatis中，能进行批处理的条件有3个：

 *  相同的SQL映射声明，即MappedStatement相同
 *  必须是连续的SQL
 *  相同的SQL语句

先来几个单元测试验证一下以上的结论

##### `MappedStatement`不同 #####

```java
// UserMapper
@Update("UPDATE user SET name = ##{name} WHERE id = ##{id}")
int updateNameById2(@Param("id") Integer id, @Param("name") String name);

@Update("UPDATE user SET name = ##{name} WHERE id = ##{id}")
int updateNameById3(@Param("id") Integer id, @Param("name") String name);

@Test
public void batchExecutorTest1() throws SQLException {
    // 为了方便测试，这里直接通过SqlSession的门面模式，指定ExecutorType的来构建BatchExecutor对象执行sql
    try (final SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH)) {
        final UserMapper mapper = sqlSession.getMapper(UserMapper.class);

        // 分别调用两个不同的MappedStatement
        mapper.updateNameById2(1, "张三1");
        mapper.updateNameById3(2, "李四1");

        final List<BatchResult> batchResults = sqlSession.flushStatements();
        batchResults.forEach(e -> System.out.println(Arrays.toString(e.getUpdateCounts())));
    }
//    以上代码等同于
//    final BatchExecutor batchExecutor = new BatchExecutor(configuration, jdbcTransaction);
//    final MappedStatement mappedStatement1 = configuration.getMappedStatement("com.wuwenze.mybatis.example.UserMapper.updateNameById2");
//    final Map<String, Object> params1 = new HashMap<>();
//    params1.put("id", 1);
//    params1.put("name", "张三1");
//    batchExecutor.doUpdate(mappedStatement1, params1);
//
//    final MappedStatement mappedStatement2 = configuration.getMappedStatement("com.wuwenze.mybatis.example.UserMapper.updateNameById3");
//    final Map<String, Object> params2 = new HashMap<>();
//    params1.put("id", 2);
//    params1.put("name", "李四1");
//    batchExecutor.doUpdate(mappedStatement2, params2);
//
//    final List<BatchResult> batchResults = batchExecutor.flushStatements();
//    batchResults.forEach(e -> System.out.println(Arrays.toString(e.getUpdateCounts())));
}
```

![32d81b15-cc3f-42cb-8ac7-7fe5c67e88f1.png][]

从执行结果可以看到，尽管SQL完全一致，但是由于声明的`MappedStatement`对象并不相同，批处理执行器并没有生效。

##### 必须是连续的SQL #####

```java
@Test
public void batchExecutorTest2() throws SQLException {
    try (final SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH)) {
        final UserMapper mapper = sqlSession.getMapper(UserMapper.class);


        mapper.updateNameById2(1, "张三1");
        mapper.updateNameById3(2, "李四1"); // updateNameById2并没有连续发送SQL，中间被updateNameById3打断
        mapper.updateNameById2(1,"张三111");

        final List<BatchResult> batchResults = sqlSession.flushStatements();
        batchResults.forEach(e -> System.out.println(Arrays.toString(e.getUpdateCounts())));
    }
}
```

![06b140bb-ac62-4cd0-a34f-01ee19013a96.png][]

可见相同的SQL连续被编译了3次，批处理执行器并没有生效。

##### 同一个`MappedStatement`，且连续的发送相同SQL #####

```java
@Test
public void batchExecutorTest3() throws SQLException {
    try (final SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH)) {
        final UserMapper mapper = sqlSession.getMapper(UserMapper.class);


        mapper.updateNameById2(1, "张三1");
        mapper.updateNameById2(2, "李四111");
        mapper.updateNameById2(1, "张三111");

        final List<BatchResult> batchResults = sqlSession.flushStatements();
        batchResults.forEach(e -> System.out.println(Arrays.toString(e.getUpdateCounts())));
    }
}
```

![9ee664eb-a795-41fb-bd68-b201dae79c89.png][]

这次执行得很完美，同时也验证了上面的结论。

## 一级&二级缓存 ##

刚才研究完了`BaseExecutor`的核心逻辑，现在回过头来回顾之前的`Executor`的实现类结构，其实还有另外一个与`BaseExecutor`同级的横向扩展实现类`CachingExecutor`，其主要目的的是为了实现二级缓存（一级缓存在`BaseExecutor`中实现）

由于篇幅原因，关于缓存的部分，在下一篇文章中再详细分析。


[b6b12f48-e972-4c19-b58b-b37ad7031737.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210731/b6b12f48-e972-4c19-b58b-b37ad7031737.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[e78f26b7-9019-4826-a8a5-aff7d560eac6.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210731/e78f26b7-9019-4826-a8a5-aff7d560eac6.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[mybatis _ MyBatis 3 _]: https://mybatis.org/mybatis-3/zh/getting-started.html
[525bfe8f-a081-4b42-8f2d-f513690d744f.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210731/525bfe8f-a081-4b42-8f2d-f513690d744f.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[02480ee2-e0c4-438c-aa17-c80af99b4755.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210731/02480ee2-e0c4-438c-aa17-c80af99b4755.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[a894a61e-2431-4345-a907-ec71f44b7433.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210731/a894a61e-2431-4345-a907-ec71f44b7433.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[b5294ae1-8028-44f1-9695-1dad034e4a9d.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210731/b5294ae1-8028-44f1-9695-1dad034e4a9d.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[a83d4c25-f5e2-4680-9336-40865daab49c.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210731/a83d4c25-f5e2-4680-9336-40865daab49c.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[b2a712c4-f1c3-496d-a374-c39a5f71045b.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210731/b2a712c4-f1c3-496d-a374-c39a5f71045b.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[a9666b72-fa88-4222-9da6-8031d2815715.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210731/a9666b72-fa88-4222-9da6-8031d2815715.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[ea1223a7-6f75-467b-bfc2-73a7013fbf01.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210731/ea1223a7-6f75-467b-bfc2-73a7013fbf01.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[e2f0f652-5c56-4fba-963e-35c5ec3e8238.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210731/e2f0f652-5c56-4fba-963e-35c5ec3e8238.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[32d81b15-cc3f-42cb-8ac7-7fe5c67e88f1.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210731/32d81b15-cc3f-42cb-8ac7-7fe5c67e88f1.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[06b140bb-ac62-4cd0-a34f-01ee19013a96.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210731/06b140bb-ac62-4cd0-a34f-01ee19013a96.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[9ee664eb-a795-41fb-bd68-b201dae79c89.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210731/9ee664eb-a795-41fb-bd68-b201dae79c89.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg