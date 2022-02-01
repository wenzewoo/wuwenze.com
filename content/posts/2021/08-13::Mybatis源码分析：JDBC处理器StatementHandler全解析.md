+++
title = "Mybatis源码分析：JDBC处理器StatementHandler全解析"
date = "2021-08-13 11:24:35"
url = "archives/1001"
tags = ["MyBatis"]
categories = ["后端","Mybatis源码阅读"]
featuredImage = "https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210923/e72cefb271904fa0b13e41272d9f95e0.jpg"
+++

Mybatis本质上是一个JDBC框架，但是之前分析的内容跟JDBC的关系还不是很明显，实际上JDBC相关的操作全部封装到了`StatementHandler`组件中，一个SQL请求经过会话，到达执行器，最终由StatmentHandler执行JDBC来操作数据库。

照例先来回顾一下基本的流程：

![a8853c06-243c-47d4-a52b-4e554adc86fa.png][]

关于执行器的相关原理，之前已经做过详细分析，参见以下两篇文章：

- Mybatis源码分析：深入认识Executor执行器: https://wuwenze.com/archives/918

- Mybatis源码分析：一级&二级缓存原理: https://wuwenze.com/archives/951

## 何时创建? ##

`StatementHandler`由`BaseExecutor`的子类负责创建，以`SimpleExecutor`为例，来看一下是如何创建的。

```java
public class SimpleExecutor extends BaseExecutor {
  // 省略其他代码...

  @Override
  public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();

      // 创建StatementHandler对象
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);

      // 构建Statement对象
      stmt = prepareStatement(handler, ms.getStatementLog());

      // 使用StatementHandler处理Statement，执行SQL
      return handler.update(stmt);
    } finally {
      closeStatement(stmt);
    }
  }

  @Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    // 与doUpdate类似，省略代码..
  }

 // 省略其他...
}
```

在`Executor`的`doQuery()` / `doUpdate()`等实现中，每次都通过`Configuration`对象构建了一个新的`StatementHandler`对象

```java
public class Configuration {
  // 省略其他代码...

  public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    // 通过参数构建RoutingStatementHandler对象（该类又是一个装饰器）
    StatementHandler statementHandler = new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);

    // 加入拦截器链，后续讲解
    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }

  // 省略其他代码...
}

// 动态装饰器
public class RoutingStatementHandler implements StatementHandler {

  private final StatementHandler delegate;

  public RoutingStatementHandler(Executor executor, MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    // 通过配置的StatementType类型（默认是Prepared），来构建不同的委托实现类（StatementHandler）
    switch (ms.getStatementType()) {
      case STATEMENT:
        delegate = new SimpleStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case PREPARED:
        delegate = new PreparedStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      case CALLABLE:
        delegate = new CallableStatementHandler(executor, ms, parameter, rowBounds, resultHandler, boundSql);
        break;
      default:
        throw new ExecutorException("Unknown statement type: " + ms.getStatementType());
    }
  }

  @Override
  public int update(Statement statement) throws SQLException {
    return delegate.update(statement);
  }

  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    return delegate.query(statement, resultHandler);
  }

  @Override
  public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    return delegate.prepare(connection, transactionTimeout);
  }
  // 其余实现同上，忽略，都是通过委托对象调用相关的实现方法。
}
```

`RoutingStatementHandler`的实现目前看起来有点迷，有点像是无意义的封装，其核心逻辑就是通过配置的`StatementType`来构建不同的`StatementHandler`，实现动态装饰器，可能是我的道行还比较浅，不太明白这样做的意义是什么，如果换我来实现，可能就是类似这样的：

```java
public class Configuration {
  // 省略其他代码...

  public StatementHandler newStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    StatementHandler statementHandler = null; //new RoutingStatementHandler(executor, mappedStatement, parameterObject, rowBounds, resultHandler, boundSql);

    switch (mappedStatement.getStatementType()) {
      case STATEMENT:
        statementHandler = new SimpleStatementHandler(executor, mappedStatement, parameter, rowBounds, resultHandler, boundSql);
        break;
      case PREPARED:
        statementHandler = new PreparedStatementHandler(executor, mappedStatement, parameter, rowBounds, resultHandler, boundSql);
        break;
      case CALLABLE:
        statementHandler = new CallableStatementHandler(executor, mappedStatement, parameter, rowBounds, resultHandler, boundSql);
        break;
      default:
        throw new ExecutorException("Unknown statement type: " + mappedStatement.getStatementType());
    }

    statementHandler = (StatementHandler) interceptorChain.pluginAll(statementHandler);
    return statementHandler;
  }

  // 省略其他代码...
}
```

## 类结构 ##

StatementHandler是一个顶层的接口，定义了JDBC相关的操作方法：

```java
public interface StatementHandler {

  // 声明Statement对象
  Statement prepare(Connection connection, Integer transactionTimeout)
      throws SQLException;

  // 为Statement处理参数
  void parameterize(Statement statement)
      throws SQLException;

  // 添加批处理，即：statement.addBatch()
  void batch(Statement statement)
      throws SQLException;

  // 执行update操作
  int update(Statement statement)
      throws SQLException;

  // 执行query操作
  <E> List<E> query(Statement statement, ResultHandler resultHandler)
      throws SQLException;

  <E> Cursor<E> queryCursor(Statement statement)
      throws SQLException;

  // 获取动态SQL
  BoundSql getBoundSql();

  // 获取参数处理器
  ParameterHandler getParameterHandler();

}
```

![ec5eb69c-b3d2-4a32-b6f6-2f5af2a9ee99.png][]

实现类很简单，其中`RoutingStatementHandler`咱们刚才已经看过了，其余3个分别对应JDBC中的`Statement`，`PreparedStatement`，`CallableStatement`对象。

![c4d705cf-5da8-4956-b0e9-899eff0b3dd6.png][]

大部分情况下，都是使用`PreparedStatementHandler`（默认值）如果需要使用其他类型的处理器，通过这样配置即可：

```java
public interface UserMapper {
    @Select("SELECT * FROM user WHERE id = ##{id}")
    @Options(statementType = StatementType.STATEMENT)
    User findById(@Param("id") Integer id);
}
```

## 执行流程 ##

接下来了解`StatementHandler`的完整执行流程（以`PreparedStatementHandler`为例）先来看下图

![b02ebdf6-03e8-4870-87e6-17c01d2928a0.png][]

总体来说，执行过程共分为了三个阶段：

 *  **预处理**：这里不仅仅是通过`Connection`创建`Statement`对象，同时还包含了参数的处理。
 *  **执行SQL**：包含执行SQL以及对数据库返回结果的映射。
 *  **关闭**：直接关闭`Statement`。

![d1f04d07-41f1-4fae-bff8-3c9356268fe3.png][]

首先是创建`StatementHandler`对象（由`RoutingStatementHandler`装饰`PreparedStatementHandler`对象）

![741537a9-9c30-43f3-8bed-a68678df2467.png][]

接着使用`handler.prepare()`创建`PreparedStatement`对象，同时调用`handler.parameterize()`进行参数预处理。

需要注意的是，由于配置了输出日志，所以`Connection`以及`PreparedStatement`都是分别由`ConnectionLogger`和`PreparedStatementLogger`两个类进行动态代理的，其目的就是为了实现日志输出。

如果将日志输出的配置取消，则上述的两个对象不会经过Mybatis的动态代理包装，而是直接实例化JDBC的原始对象。

```xml
<settings>
   <setting name="cacheEnabled" value="true"/>
<!--        <setting name="logImpl" value="STDOUT_LOGGING"/>-->
</settings>
```

![8cc153f2-c41f-4015-9778-d9e97ca866b6.png][]

关于日志输出的动态代理实现，后续有时间再深入分析，咱们再继续跟进`StatementHandler`的执行流程。

![b5de6e5e-2d4a-465a-8875-e50cf9bc2acf.png][]

继续调用`handler.query()`方法，执行JDBC的`PreparedStatement.execute()`方法，完成SQL调用，最后由`ResultSetHandler`处理执行结果，最终由最外层的`finally`代码块执行`closeStatement()`方法，则完成了全部的流程。

此外，上图中还有个小细节，在`query()`中定义了`resultHandler`参数，但是并没有传值和使用，而是直接使用的是`BaseStatementHandler`中定义的`DefaultResultSetHandler`，这是为什么呢？这个行参目前来看好像是没有作用的，还是比较迷惑的。

## 参数处理 ##

参数处理以及结果集封装，可能是整个流程中最为复杂和繁琐的部分了，所以分别用了`ParameterHandler`以及`ResultSetHandler`两个专门的组件来实现这些复杂的逻辑，接下来先来看一下参数处理的详细过程。

何为参数处理？换句话说，如何将`Mapper`中定义的参数赋值到SQL语句中？总的来说，要经过3个步骤，参数转换、参数映射、参数赋值。

![2b667aaf-b73f-463e-9c64-23b4e5a9e127.png][]

### 参数转换 ###

即将`Mapper`定义的方法中的普通参数，转换成`Map`，以便于`Map`中的`key`与SQL模板中引用相对应。

```java
public interface User2Mapper {
    // 未使用@Param注解，则SQL模板中使用变量age0,age1或者param1,param2...

    @Select("SELECT * FROM user WHERE name = ##{arg0} AND age = ##{arg1}")
    User findByNameAndAge(String name, Integer age);

    @Select("SELECT * FROM user WHERE name = ##{name} AND age = ##{id}")
    User findByNameAndId(@Param("name") String name, @Param("id") Integer id);

    @Select("SELECT * FROM user WHERE id = ##{id}")
    User findById(Integer id);
}
```

在调用`user2Mapper.findByNameAndAge()`函数后，经过`Mapper`动态代理，最终由`ParamNameResolver`转换器负责对参数进行转换，关于`Mapper`的动态代理，以后我会专门整理一篇文章来说明，这里先看一个简单的动态代理流程图。

![663fb842-97cc-454d-a190-8748baca2844.png][]

参数的转换处理将在`MapperMethod.MethodSignature`中触发，来看看关键的源代码

```java
public class MapperMethod {
  private final SqlCommand command;
  private final MethodSignature method;

  // 省略其他...

  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      case INSERT: {
        // 对Mapper定义的参数进行转换
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      case UPDATE: {

        // 对Mapper定义的参数进行转换
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.update(command.getName(), param));
        break;
      }
      case DELETE: {

          // 对Mapper定义的参数进行转换
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.delete(command.getName(), param));
        break;
      }
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {

          // 对Mapper定义的参数进行转换
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
          if (method.returnsOptional()
              && (result == null || !method.getReturnType().equals(result.getClass()))) {
            result = Optional.ofNullable(result);
          }
        }
        break;
      case FLUSH:
        result = sqlSession.flushStatements();
        break;
      default:
        throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName()
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
   // 省略其他代码...

  }


  public static class MethodSignature {

    // 省略其他..
    private final ParamNameResolver paramNameResolver;

    public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
      // 省略其他..
      this.paramNameResolver = new ParamNameResolver(configuration, method);
    }

    public Object convertArgsToSqlCommandParam(Object[] args) {
      // 调用ParamNameResolver.getNamedParams() 完成参数转换
      return paramNameResolver.getNamedParams(args);
    }

    // 省略其他..
  }

}
```

其核心逻辑最终在`ParamNameResolver.getNamedParams()`中完成，来看关键代码：

```java
public class ParamNameResolver {
  // 默认的参数名前缀
  public static final String GENERIC_NAME_PREFIX = "param";
  // 是否使用实际的参数名
  private final boolean useActualParamName;
  // 有顺序的参数列表
  private final SortedMap<Integer, String> names;
  // 是否有@Param注解
  private boolean hasParamAnnotation;

  public ParamNameResolver(Configuration config, Method method) {
    this.useActualParamName = config.isUseActualParamName();

    // 使用反射获取参数列表、参数定义的注解列表
    final Class<?>[] paramTypes = method.getParameterTypes();
    final Annotation[][] paramAnnotations = method.getParameterAnnotations();
    final SortedMap<Integer, String> map = new TreeMap<>();
    int paramCount = paramAnnotations.length;

    // 根据方法行参数定义的@Param初始化names
    for (int paramIndex = 0; paramIndex < paramCount; paramIndex++) {
      if (isSpecialParameter(paramTypes[paramIndex])) {
        // skip special parameters
        continue;
      }
      String name = null;
      for (Annotation annotation : paramAnnotations[paramIndex]) {
        if (annotation instanceof Param) {
          hasParamAnnotation = true;
          name = ((Param) annotation).value();
          break;
        }
      }
      if (name == null) {
        // @Param was not specified.
        if (useActualParamName) {
          name = getActualParamName(method, paramIndex);
        }
        if (name == null) {
          // use the parameter index as the name ("0", "1", ...)
          // gcode issue ##71
          name = String.valueOf(map.size());
        }
      }
      map.put(paramIndex, name);
    }
    names = Collections.unmodifiableSortedMap(map);
  }


  // 省略其他

  public Object getNamedParams(Object[] args) {
    final int paramCount = names.size();
    if (args == null || paramCount == 0) {
      return null; // 无参数，返回null
    } 
    // 没有@Param注解，且只有一个参数，直接取第一个
    else if (!hasParamAnnotation && paramCount == 1) {
      Object value = args[names.firstKey()];

      // 只有一个参数，判断是否为list或者array，如果是，再包装一层
      return wrapToMapIfCollection(value, useActualParamName ? names.get(0) : null);
    } else {

      // 多个参数，转换成map
      final Map<String, Object> param = new ParamMap<>();
      int i = 0;
      for (Map.Entry<Integer, String> entry : names.entrySet()) {
        // names有3种情况
        // 1. 未定义@Param注解，默认为arg0, age1
        // 2. 未定义@Param注解，JDK8以上，且开启了“-parameters”编译参数，则names为实际的参数名
        // 3. 定义了@Pram注解，则值为@Param定义的名字
        param.put(entry.getValue(), args[entry.getKey()]);

        // 除反射获取的名称外，再加入通用参数名（param1, param2...)
        final String genericParamName = GENERIC_NAME_PREFIX + (i + 1);
        if (!names.containsValue(genericParamName)) {
          param.put(genericParamName, args[entry.getKey()]);
        }
        i++;
      }
      return param;
    }
  }

  public static Object wrapToMapIfCollection(Object object, String actualParamName) {
    if (object instanceof Collection) {
      ParamMap<Object> map = new ParamMap<>();
      map.put("collection", object);
      if (object instanceof List) {
        map.put("list", object);
      }
      Optional.ofNullable(actualParamName).ifPresent(name -> map.put(name, object));
      return map;
    } else if (object != null && object.getClass().isArray()) {
      ParamMap<Object> map = new ParamMap<>();
      map.put("array", object);
      Optional.ofNullable(actualParamName).ifPresent(name -> map.put(name, object));
      return map;
    }
    return object;
  }

}
```

参数转换的核心逻辑就分析完了，来简单总结一下：

 *  **单个参数**：且没有设置`@Param`注解会直接转换，忽略SQL中的引用名称
 *  **多个参数**：如果有`@Param`注解，则使用注解中定义的名称，反之使用反射获取的参数名（`arg0, arg1...`）此外，再加入通用的参数名（`param1, param2..`.）
 *  可以通过指定JDK8编译参数（“`-parameters`”）来省略`@Param`注解，一般不建议使用。

### 参数映射 ###

那么在参数转换完成之后，将转换后的`parameterMap`交由`SqlSession`门面传递给`Executor`进行执行

```java

public class SimpleExecutor extends BaseExecutor {

  // 省略其他代码...

  @Override
  public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();

      // 构建StatementHandler
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);

      // SQL预编译
      stmt = prepareStatement(handler, ms.getStatementLog());

      // 执行查询
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }

  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());

    // 参数处理
    handler.parameterize(stmt);
    return stmt;
  }
}
```

在`StatementHandler.parameterize()`中，又将参数对象交由了`ParameterHandler`进行处理

```java
public class PreparedStatementHandler extends BaseStatementHandler {
  protected final ParameterHandler parameterHandler;
  protected final ResultSetHandler resultSetHandler;

  // 省略其他代码..

  protected BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {

    // 省略其他代码...   
    this.parameterHandler = configuration.newParameterHandler(mappedStatement, parameterObject, boundSql);
    this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
  }

  @Override
  public void parameterize(Statement statement) throws SQLException {

    // 交由parameterHandler处理参数
    parameterHandler.setParameters((PreparedStatement) statement);
  }
}


public class DefaultParameterHandler implements ParameterHandler {

  // 省略其他代码...

  @Override
  public void setParameters(PreparedStatement ps) {
    ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());
    
    // 获取boundSql中包装后的参数对象,
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {

      // 遍历参数映射、赋值
      for (int i = 0; i < parameterMappings.size(); i++) {
        ParameterMapping parameterMapping = parameterMappings.get(i);
        if (parameterMapping.getMode() != ParameterMode.OUT) {
          Object value;
          String propertyName = parameterMapping.getProperty();
          if (boundSql.hasAdditionalParameter(propertyName)) { // issue ##448 ask first for additional params
            value = boundSql.getAdditionalParameter(propertyName);
          } else if (parameterObject == null) {
            value = null;
          } 
          // 如果有自定义的参数转换器，则无需映射，后面由自定义的TypeHandler进行赋值
          else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
            value = parameterObject;
          } else {

            // 使用反射工具类进行参数转换，支持格式如：username, user.name等
            MetaObject metaObject = configuration.newMetaObject(parameterObject);
            value = metaObject.getValue(propertyName);
          }

          // 由TypeHandler进行参数赋值
          TypeHandler typeHandler = parameterMapping.getTypeHandler();
          JdbcType jdbcType = parameterMapping.getJdbcType();
          if (value == null && jdbcType == null) {
            jdbcType = configuration.getJdbcTypeForNull();
          }
          try {
            typeHandler.setParameter(ps, i + 1, value, jdbcType);
          } catch (TypeException | SQLException e) {
            throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
          }
        }
      }
    }
  }
}
```

由此可见，参数映射的核心关键，就是这个`MetaObject`的反射工具类，这也是Mybatis中相当重要的一个组件，这里暂时不去研究底层的实现细节，咱们只需要知道，该工具类提供强大的反射功能，可以根据传入的`propertyName`获取对应传入的参数值。

参数映射完成后，拿到了具体参数的值，比如：`user.name = "张三"`，接下来就是将其设置到SQL中，这时就轮到`TypeHandler`出场了。

### 参数赋值 ###

```java
public interface TypeHandler<T> {

  // 设置参数
  void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;


  // 获取结果集
  T getResult(ResultSet rs, String columnName) throws SQLException;
  T getResult(ResultSet rs, int columnIndex) throws SQLException;
  T getResult(CallableStatement cs, int columnIndex) throws SQLException;
}
```

`TypeHandler`定义了两种方法，设置参数以及获取结果集，再来看看具体的实现类都是怎么实现参数设置的。

![a51c91a0-1cc8-4754-a90a-f5acee470adf.png][]

从图中可以看到，`TypeHandler`有很多实现类，针对每种数据类型都会有其自己的实现类，来看最常用的`StringTypeHandler`实现，首先会经过所有`TypeHandler`的父类`BaseTypeHandler`，该类是一个典型的模板设计方法：

```java
public abstract class BaseTypeHandler<T> extends TypeReference<T> implements TypeHandler<T> {

  @Override
  public void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException {
    
    // 参数为空的情况
    if (parameter == null) {
      if (jdbcType == null) {
        throw new TypeException("JDBC requires that the JdbcType must be specified for all nullable parameters.");
      }
      try {
        ps.setNull(i, jdbcType.TYPE_CODE);
      } catch (SQLException e) {
        throw new TypeException("Error setting null for parameter ##" + i + " with JdbcType " + jdbcType + " . "
              + "Try setting a different JdbcType for this parameter or a different jdbcTypeForNull configuration property. "
              + "Cause: " + e, e);
      }
    } else {

      // 参数不为空时，交由具体的实现类实现，子类只需实现setNonNullParameter即可。
      try {
        setNonNullParameter(ps, i, parameter, jdbcType);
      } catch (Exception e) {
        throw new TypeException("Error setting non null for parameter ##" + i + " with JdbcType " + jdbcType + " . "
              + "Try setting a different JdbcType for this parameter or a different configuration property. "
              + "Cause: " + e, e);
      }
    }
  }
  // 省略结果集处理相关

  public abstract void setNonNullParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;
}

public class StringTypeHandler extends BaseTypeHandler<String> {

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType)
      throws SQLException {
    // 调用jdbc中的PreparedStatement.setString()完成参数赋值
    ps.setString(i, parameter);
  }

  // 省略结果集处理相关..
}
```

参数的处理到这里就完成了，经过一系列的跳转，多个组件的传递后，最终调用了JDBC的底层API，实现了参数赋值。

## 结果集处理 ##

回顾之前的代码，在`StatmentHandler`中调用`PreparedStatment.execute()`后，待SQL执行完成，就该处理结果了

```java
public class PreparedStatementHandler extends BaseStatementHandler {
  // 省略其他代码...

  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();

    // 使用resultSetHandler处理SQL返回结果
    return resultSetHandler.handleResultSets(ps);
  }
}
```

`resultSetHandler`的实例化在其`BaseStatementHandler`父类中完成，关于结果集的转换，99%的逻辑都在唯一的实现类`DefaultResultSetHandler`中完成。

![e5581c37-b359-4cdd-a0f7-2a6a629b667f.png][]

简单来说，整理的流程可以大致描述为：

 *  读取结果集
 *  遍历结果集中的当前行
 *  创建对象
 *  填充属性

该类的内容比较多，咱们用代码调试的方式，挑重要的部分来看

![bc50523b-9e8c-453c-9f04-f4752d25faef.png][]

首先是通过`mappedStatment.getResultMaps()`读取结果集，紧接着开始依次遍历，如图中所示，获取到一条数据，返回结果为`List<ResultMap>`，针对当前行的数据，再继续调用`handleResultSet()`方法。

![848189af-957f-443b-8185-8555f9a3e590.png][]

再接着调用`handleRowValues()`返回，处理当前行的`ResultMap`对象。

![661085a3-7e15-46d0-8578-ddfc71622840.png][]

这里又经过了2个分支，其中分别对应的是处理**嵌套结果**`handleRowValuesForNestedResultMap()`以及处理**简单结果**`handleRowValuesForSimpleResultMap()`

![c1a63c57-52c6-476d-8d83-dafa38a2d73a.png][]

通过调用`getRowValue()`来创建对象，再继续往下看

![1cde291e-87ac-43bc-ae9c-e03f09ed18b5.png][]

即先通过`createResultObject()`创建一个空对象（即通过无参构造函数），接着使用`MetaObject`对象对其进行属性的填充，主要分为两个部分，首先是`applyAutomaticMappings()`方法负责自动填充属性，然后再由`applyPropertyMappings()`负责填充手动映射的一些属性。此外，这里还涉及到一些懒加载代理相关的逻辑，就暂不研究了。


[a8853c06-243c-47d4-a52b-4e554adc86fa.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/a8853c06-243c-47d4-a52b-4e554adc86fa.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[ec5eb69c-b3d2-4a32-b6f6-2f5af2a9ee99.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/ec5eb69c-b3d2-4a32-b6f6-2f5af2a9ee99.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[c4d705cf-5da8-4956-b0e9-899eff0b3dd6.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/c4d705cf-5da8-4956-b0e9-899eff0b3dd6.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[b02ebdf6-03e8-4870-87e6-17c01d2928a0.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/b02ebdf6-03e8-4870-87e6-17c01d2928a0.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[d1f04d07-41f1-4fae-bff8-3c9356268fe3.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/d1f04d07-41f1-4fae-bff8-3c9356268fe3.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[741537a9-9c30-43f3-8bed-a68678df2467.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/741537a9-9c30-43f3-8bed-a68678df2467.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[8cc153f2-c41f-4015-9778-d9e97ca866b6.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/8cc153f2-c41f-4015-9778-d9e97ca866b6.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[b5de6e5e-2d4a-465a-8875-e50cf9bc2acf.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/b5de6e5e-2d4a-465a-8875-e50cf9bc2acf.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[2b667aaf-b73f-463e-9c64-23b4e5a9e127.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/2b667aaf-b73f-463e-9c64-23b4e5a9e127.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[663fb842-97cc-454d-a190-8748baca2844.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/663fb842-97cc-454d-a190-8748baca2844.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[a51c91a0-1cc8-4754-a90a-f5acee470adf.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/a51c91a0-1cc8-4754-a90a-f5acee470adf.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[e5581c37-b359-4cdd-a0f7-2a6a629b667f.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/e5581c37-b359-4cdd-a0f7-2a6a629b667f.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[bc50523b-9e8c-453c-9f04-f4752d25faef.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/bc50523b-9e8c-453c-9f04-f4752d25faef.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[848189af-957f-443b-8185-8555f9a3e590.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/848189af-957f-443b-8185-8555f9a3e590.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[661085a3-7e15-46d0-8578-ddfc71622840.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/661085a3-7e15-46d0-8578-ddfc71622840.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[c1a63c57-52c6-476d-8d83-dafa38a2d73a.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/c1a63c57-52c6-476d-8d83-dafa38a2d73a.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[1cde291e-87ac-43bc-ae9c-e03f09ed18b5.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210813/1cde291e-87ac-43bc-ae9c-e03f09ed18b5.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg