+++
title = "Mybatis源码分析：动态SQL全流程解析"
date = "2021-09-28 16:30:14"
url = "archives/1054"
tags = ["MyBatis"]
categories = ["后端","Mybatis源码阅读"]
featuredImage = "https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210923/bb73be20852d4c1392cecf5eb60fee7a.jpg"

+++

动态 SQL 是 MyBatis 的强大特性之一。如果你使用过 JDBC 或其它类似的框架，你应该能理解根据不同条件拼接 SQL 语句有多痛苦，例如拼接时要确保不能忘记添加必要的空格，还要注意去掉列表最后一个列名的逗号。利用动态 SQL，可以彻底摆脱这种痛苦。

使用动态 SQL 并非一件易事，但借助可用于任何 SQL 映射语句中的强大的动态 SQL 语言，MyBatis 显著地提升了这一特性的易用性。

如果你之前用过 JSTL 或任何基于类 XML 语言的文本处理器，你对动态 SQL 元素可能会感觉似曾相识。在 MyBatis 之前的版本中，需要花时间了解大量的元素。借助功能强大的基于 OGNL 的表达式，MyBatis 3 替换了之前的大部分元素，大大精简了元素种类，现在要学习的元素种类比原来的一半还要少。

> 以上摘抄至Mybatis的官方文档：https://mybatis.org/mybatis-3/zh/dynamic-sql.html，本文的目的是理解并阅读其动态SQL的核心实现原理，有关更多的使用方法，可以查询相关文档。


## 简单示例

使用动态SQL最常见情景是根据条件包含`WHERE`子句的一部分。比如：

```xml
<select id="findActiveBlogWithTitleLike" resultType="Blog">
    SELECT * FROM BLOG
    WHERE state = 'ACTIVE'
    <if test="title != null">
        AND title like #{title}
    </if>
    <if test="author != null and author.name != null">
        AND author_name like #{author.name}
    </if>
</select>
```

以上代码经常使用Mybatis的朋友们已经很熟练了，我们一向只知道怎么用，那么Mybatis究竟是如何巧妙的实现它的呢？

除了常见的`<if/>`标签，Mybatis还提供了以下几种：
- if
- choose (when, otherwise)
- trim (where, set)
- foreach

## 动态SQL脚本

每次执行SQL语句时，基于预先编写的脚本和参数，动态的构建可以被执行的SQL语句。

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210928/529d408c12e747f7be8a8ab769036f42.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

如图上所示，在Mybatis解析动态SQL的过程中，所有的XML元素都会被解析成若干个`SqlNode`，每个动态元素都会有一个与之对应的`SqlNode`实现类，来先大概的了解一下有哪些实现类：

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210928/5c3387791bb045e8b025ba7cd5030cee.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

从类名来看，就已经非常好理解了，`<if />`标签对应的则是`IfSqlNode`，以此类推，这里我们着重注意以下三个实现类：

- **StaticTextSqlNode**: 表示一段纯静态的文本，比如：`SELECT * FROM BLOG`
- **TextSqlNode**: 表示一段通过参数瓶装的文本，卢比：`WHERE title = ${title}`
- **MixedSqlNode**: 表示多个节点的集合

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210928/62fed85caeda43c4b80764c0864ec0f7.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

说这么多可能还是不太好理解，咱们来模拟使用一下`SqlNode`试试：

```java
@Test
public void test1() {
    // 我们假设XML配置的SQL如下：
    // SELECT * FROM user
    // WHERE 1 = 1
    // <if test="phone != null and phone != ''">
    //  AND phone = #{phone}
    // </if>
    // <if test="searchKey != null and searchKey != ''">
    //  AND name like %${searchKey}%
    // </if>

    // 那么拼装出来的SqlNode大致就是以下形式

    final StaticTextSqlNode staticTextSqlNode = new StaticTextSqlNode(
            "SELECT * FROM user\nWHERE 1 = 1");

    final IfSqlNode ifSqlNode = new IfSqlNode(
            new TextSqlNode("AND phone = #{phone}"),
            "phone != null and phone != ''"// if 标签中的OGNL表达式
    );

    final IfSqlNode ifSqlNode1 = new IfSqlNode(
            new TextSqlNode("AND name like %${searchKey}%"),
            "searchKey != null and searchKey != ''"
    );

    // 组装成MixedSqlNode
    final MixedSqlNode mixedSqlNode = new MixedSqlNode(
            Lists.newArrayList(staticTextSqlNode, ifSqlNode, ifSqlNode1));


    // 使用SqlSource获取可以执行的SQL.
    final Map<String, String> params = ImmutableMap.of(
            "phone", "17311111111", "searchKey", "Hello");
    
    final String sql = new DynamicSqlSource(
            new Configuration(), mixedSqlNode).getBoundSql(params).getSql();
    System.out.println(sql);

    // 执行结果:
    // SELECT * FROM user
    // WHERE 1 = 1 AND phone = ? AND name like %Hello%
    
    // #{}会被解析成？，因为这涉及到SQL注入的问题，后续由参数处理器填充再执行。
}
```

### 动态SQL脚本语法树

以上是我们模拟的`SqlNode`结构，较为简单，这次我们写一个真实的查询，将断点打在`DynamicSqlSource`类中，观察真实的语法树结构。

```xml
<select id="listBySearchKeyAndAuthorId" resultType="com.wuwenze.mybatis.example2.entity.Post">
    SELECT * FROM post
    <where>
        <if test="searchKey != null and searchKey != ''">
            AND title like '%${searchKey}%'
        </if>

        <if test="authorId != null">
            AND author_id = #{authorId}
        </if>
    </where>
</select>
```

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210929/b841e6a35a9142a0b01fee046f3370b6.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

可以看到，脚本之间是呈现嵌套关系的，比如`if`元素中会包含一个`MixedSqlNode`，而`MixedSqlNode`下又会包含1个或者多个其他节点元素，图中是包含了一个`TextSqlNode`元素。

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210929/f050c095a49f40e98d900aaa594752c2.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

### 语法树是如何构建的？

知道了语法树的结构，那么构建语法树将变得简单，无非就是解析XML脚本，再构建对应的`SqlNode`对象，其核心源码在`XMLScriptBuilder`类中，后面我们再详细分析。

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210929/62eac8dab2b140f3b9b0b45ce6abe6b7.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


### 语法树是如何变成SQL语句的？
`SqlNode`的接口非常简单，就只有一个`apply()`方法，该方法的作用就是执行当前脚本逻辑，并将结果应用到`DynamicContext`中去

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210929/ce6cdb1f9c13410ba6968cbd34ddf4f0.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

如`IfSqlNode`当中执行`apply()`时先计算If逻辑，如果通过就会继续去访问它的子节点。直到最后访问到`TextSqlNode`时把SQL文本添加至`DynamicContext`。 

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210929/fda2f051c31944baa3c827303f12c36d.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


话不多说，弄个单元测试来调试一下。
```java
@Test
public void test3() {
    final StaticTextSqlNode staticTextSqlNode = new StaticTextSqlNode(
            "SELECT * FROM user\nWHERE 1 = 1");

    final IfSqlNode ifSqlNode = new IfSqlNode(
            new TextSqlNode("AND phone = #{phone}"),
            "phone != null and phone != ''"// if 标签中的OGNL表达式
    );

    final IfSqlNode ifSqlNode1 = new IfSqlNode(
            new TextSqlNode("AND name like %${searchKey}%"),
            "searchKey != null and searchKey != ''"
    );
    final MixedSqlNode mixedSqlNode = new MixedSqlNode(
            Lists.newArrayList(staticTextSqlNode, ifSqlNode, ifSqlNode1));


    // 模拟手动解析SqlNode
    final Map<String, String> params = ImmutableMap.of("phone", "17311111111", "searchKey", "Hello");
    final DynamicContext dynamicContext = new DynamicContext(new Configuration(), params);


    if (mixedSqlNode.apply(dynamicContext)) {
        System.out.println(dynamicContext.getSql());
        System.out.println(dynamicContext.getBindings());
    }
}

// 运行结果：
// SELECT * FROM user
// WHERE 1 = 1 AND phone = #{phone} AND name like %Hello%
// {_parameter={phone=17311111111, searchKey=Hello}, _databaseId=null}
```

可以看到其关键点就在于这个`apply()`方法，跟进去看看都做了些什么。

```java
public class MixedSqlNode implements SqlNode {
  private final List<SqlNode> contents;

  public MixedSqlNode(List<SqlNode> contents) {
    this.contents = contents;
  }

  @Override
  public boolean apply(DynamicContext context) {
    contents.forEach(node -> node.apply(context));
    return true;
  }
}
```

首先是最外层的`MixedSqlNode`，里面包含了其所有的子元素，通过传入`DynamicContext`，循环调用其子元素自身的`apply()`方法，来实现不同的节点不同的处理逻辑。

来看一个比较简单的`IfSqlNode`节点处理方法：
```java
public class IfSqlNode implements SqlNode { 
    
  // OGNL表达式解析器
  private final ExpressionEvaluator evaluator;
  // OGNL表达式
  private final String test;
  // 子节点，if标签内部的StaticSqlNode或者TextSqlNode
  private final SqlNode contents;

  public IfSqlNode(SqlNode contents, String test) {
    this.test = test;
    this.contents = contents;
    this.evaluator = new ExpressionEvaluator();
  }

  @Override
  public boolean apply(DynamicContext context) {
      
    // phone != null and phone != ''
    // 计算OGNL表达式是否成立，其中getBindings()是用户传入的参数列表(key,val)
    if (evaluator.evaluateBoolean(test, context.getBindings())) {
      contents.apply(context); // 处理子节点 AND phone = #{phone}
      return true;
    }
    return false;
  }

}
```

接者再看对应的`TextSqlNode`子节点：
```java
public class TextSqlNode implements SqlNode {
  private final String text;
  private final Pattern injectionFilter;

  public TextSqlNode(String text) {
    this(text, null);
  }

  // 省略其他

  @Override
  public boolean apply(DynamicContext context) {
    // sql表达式解析器，针对${}包裹的变量，直接替换为值，反之#{}的变量则替换成?, 由以后的JDBC参数预处理器来处理
    GenericTokenParser parser = createParser(new BindingTokenParser(context, injectionFilter));
    
    // 将解析出来的SQL追加到DynamicContext中
    context.appendSql(parser.parse(text));
    return true;
  }
  
  // 省略其他
}
```

> 其他的标签实现，就不再详细的说明了，有兴趣的可以自己去读一读相关的源码。


通过这种类似递归方式`DynamicContext`就会访问到所有的的节点，并把最后最终符合条件的的SQL文本追加到`DynamicContext`中。

访问完所有节点之后，就会生成一个SQL字符串，但这个并不是可直接执行的SQL，因为里面的参数还是表达式的形式`#{name=name}`

就需要通过`SqlSourceBuilder`来构建可执行的SQL和参数映射`ParameterMapping`，最后将这些东西用`BoundSql`包装起来。

最终解析好的`BoundSql`放在了哪里呢？这里又有了一个新的概念`SqlSource`(SQL源)


## SqlSource（SQL源）

在上层定义上每个SQL映射（`MappedStatement`）中都会包含一个`SqlSource`用来获取可执行SQL（`BoundSql`）

`SqlSource`又分为原生SQL源与动态SQL源，以及第三方源。其关系如下图

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210928/86ac992e525a4e5c9ee31b76dcacd43c.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)


- **DynamicSqlSource**：动态SQL源包含了SQL脚本，每次获取SQL都会基于参数又及脚本，一般是加了逻辑判断的节点，则每次都需要动态创建`BoundSql`
- **RawSqlSource**：不包含任何动态元素，原生文本的SQL。但这个SQL是不能直接执行的，需要转换成`BoundSql`
- **StaticSqlSource**：包含可执行的SQL，以及参数映射，可直接生成`BoundSql`。前面的数据源都要先创建`StaticSqlSource`然后才创建BoundSql

`SqlSource`的定义如下：

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210929/00dc240a9a2f41588a1336310db2b07d.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

### SqlSource的解析过程

`SqlSource`是基于XML解析而来，解析的底层是使用Dom4j把XML解析成一个个子节点，再通过`XMLScriptBuilder`遍历这些子节点最后生成对应的`SqlNode`。看以下这个简易的流程图：

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210929/e1723e722acb4c9aa9e9a71b159c8154.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)

从图中可以看出这是一种递归式的访问，如果是文本节点就会直接创建`TextSqlNode`或`StaticSqlNode`。否则就会创建动态脚本节点如`IfSqlNode`等，最终输出`MixedSqlNode`对象。

这里每种动态节点都会对应的处理器(`NodeHandler`)来创建。创建好之后又会继续访问子节点，让递归继续下去。当然子节点所创建的`SqNode`也会作为当前所创建的元素的子节点而存在。


来看`XMLScriptBuilder`类的关键源码：
```java
public class XMLScriptBuilder extends BaseBuilder {

  private final XNode context; // XML节点
  private boolean isDynamic; // 是否为动态元素

  private final Class<?> parameterType; // 参数类型
  private final Map<String, NodeHandler> nodeHandlerMap = new HashMap<>();

  public XMLScriptBuilder(Configuration configuration, XNode context) {
    this(configuration, context, null);
  }

  public XMLScriptBuilder(Configuration configuration, XNode context, Class<?> parameterType) {
    super(configuration);
    this.context = context;
    this.parameterType = parameterType;
    initNodeHandlerMap(); // 初始化所有的NodeHandler
  }


  private void initNodeHandlerMap() {
    nodeHandlerMap.put("trim", new TrimHandler());
    nodeHandlerMap.put("where", new WhereHandler());
    nodeHandlerMap.put("set", new SetHandler());
    nodeHandlerMap.put("foreach", new ForEachHandler());
    nodeHandlerMap.put("if", new IfHandler());
    nodeHandlerMap.put("choose", new ChooseHandler());
    nodeHandlerMap.put("when", new IfHandler());
    nodeHandlerMap.put("otherwise", new OtherwiseHandler());
    nodeHandlerMap.put("bind", new BindHandler());
  }

  public SqlSource parseScriptNode() {
    // 构建 MixedSqlNode 树
    MixedSqlNode rootSqlNode = parseDynamicTags(context);

    // 构建 SqlSource 对象
    SqlSource sqlSource;
    if (isDynamic) {
      sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
    } else {
      sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
    }
    return sqlSource;
  }

  protected MixedSqlNode parseDynamicTags(XNode node) {
    List<SqlNode> contents = new ArrayList<>();
    NodeList children = node.getNode().getChildNodes();

    // 遍历XML节点
    for (int i = 0; i < children.getLength(); i++) {
      XNode child = node.newXNode(children.item(i));

      // 文本节点，创建TextSqlNode
      if (child.getNode().getNodeType() == Node.CDATA_SECTION_NODE || child.getNode().getNodeType() == Node.TEXT_NODE) {
        String data = child.getStringBody("");
        TextSqlNode textSqlNode = new TextSqlNode(data);
        if (textSqlNode.isDynamic()) {
          contents.add(textSqlNode);
          isDynamic = true;
        } else {
          contents.add(new StaticTextSqlNode(data));
        }
      } 
      // 逻辑节点，获取指定的NodeHandler来构建
      else if (child.getNode().getNodeType() == Node.ELEMENT_NODE) { // issue #628
        String nodeName = child.getNode().getNodeName();
        NodeHandler handler = nodeHandlerMap.get(nodeName);
        if (handler == null) {
          throw new BuilderException("Unknown element <" + nodeName + "> in SQL statement.");
        }
        handler.handleNode(child, contents);
        isDynamic = true;
      }
    }
    return new MixedSqlNode(contents);
  }


  // NodeHandler的定义
  private interface NodeHandler {
    void handleNode(XNode nodeToHandle, List<SqlNode> targetContents);
  }


  // IfHandler，将if标签处理成IfSqlNode
  private class IfHandler implements NodeHandler {
    public IfHandler() {
      // Prevent Synthetic Access
    }

    @Override
    public void handleNode(XNode nodeToHandle, List<SqlNode> targetContents) {
      MixedSqlNode mixedSqlNode = parseDynamicTags(nodeToHandle);
      String test = nodeToHandle.getStringAttribute("test");
      IfSqlNode ifSqlNode = new IfSqlNode(mixedSqlNode, test);
      targetContents.add(ifSqlNode);
    }
  }

  // 省略其他NodeHandler实现类，其实现大同小异，看一个IfHandler就够了。

}
```

![](https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210929/f77d909cb9af45518cfccf1b1ca8643d.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg)
