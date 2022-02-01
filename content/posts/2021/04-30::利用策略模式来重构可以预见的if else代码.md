+++
title = "利用策略模式来重构可以预见的if else代码"
date = "2021-04-30 09:49:52"
url = "archives/755"
tags = ["Java","设计模式"]
categories = ["后端"]
+++

相信大家一定见过自己项目中的满屏`if else`魔鬼代码，对于可以预见的逻辑分支增长，在前期代码设计阶段是有必要精心设计一番的。

### 案例 ###

假设有一个需求如下，**A系统**的某个业务操作完成后，会发送一个`webhook通知`给**B系统**，`webhook消息格式`如下：

```bash
{
 "type": "type1", // 目前已知的type为type1,type2
 "data": {
    //...
  }
}
```

现在你作为**B系统**的开发人员，需要接收这个`webhook消息`，并针对不同的`webhook.type`做出不同的业务逻辑处理，你一想，这简单啊！很快，你写完了下面的代码，交了差，并且项目运行得非常完美。

```java
@RestController
@RequiredArgsConstructor
public class WebhookHandlerController {

    @PostMapping("/webhook_handler")
    public Boolean webhookHandler(
        @RequestBody WebhookRequest request) {

        final String type = request.getType();
        final Object data = request.getData();
        
        
        if("type1".equals(type)) {

            // 100 行逻辑代码处理
        } else if ("type2".equals(type)) {

            // 10行逻辑代码处理
        }


        return Boolean.TRUE;
    }

    @Data
    static class WebhookRequest {
        private String type;
        private Object data;
    }
}
```

这天小明接手了这个项目，被告知需要再动一下这块的逻辑，然而小明傻眼了，他看到的是这样的代码：

![4daa5139-0b15-42e8-94fa-77a3bef9f5af.png][]

经过漫长的岁月，随着甲方需求的不断变更以及开发人员的一波又一波更换，这段在当初看起来那么无懈可击的代码早已被形形色色的人改得面目全非（图中展示的效果远没有真实项目来得恐怖）这就是传说中的**祖传·屎山代码**

### 使用策略模式优化 ###

上面的案例中，如果你作为第一个开发人员，在一开始就精心设计一番，可能后面的结局却是截然不同。

#### 定义注解和枚举 ####

```java
public enum WebhookType {
    TYPE1, TYPE2
}

@Documented
@Inherited
@Target({ ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
public @interface WebhookHandler {

    WebhookType type();
}
```

将`if else`的条件涉及的关键属性，抽象为注解和枚举，方便后续使用

#### 定义抽象handler类 ####

```java
public abstract class AbstractWebhookHandler {

    public abstract Boolean handler(WebhookRequest request);
}
```

#### 定义策略Context ####

```java
@RequiredArgsConstructor
public class WebhookHandlerContext {
    private final Map<WebhookType, Class<? extends AbstractWebhookHandler>> mHandlerMap;

    public AbstractWebhookHandler getHandler(WebhookType type) {
        final Class<? extends AbstractWebhookHandler> handlerClass = this.mHandlerMap.get(type);

        if (null != handlerClass) 
            return ReflectUtil.newInstance(handlerClass);

        throw new IllegalArgumentException("Not found webhook handler, type=" + type);
    }
}
```

这个`WebhookHandlerContext`很简单，为了方便演示，实现了最简逻辑，可能并不是很严谨，不过无关紧要，这个类的核心是想把每种类型的处理程序，都装在`mHandlerMap`中，以条件作为`map的key`，处理程序作为`map的value`值。

那么如何设置这个`mHanderMap`的值呢？需要手动的一个个的注册吗？当然不必，请继续往下看。

#### 将Context注册到Spring容器中 ####

```java
@Component
public class WebhookHandlerContextPostProcessor implements BeanFactoryPostProcessor {

    private final static String HANDLER_PACKAGE_NAME = "com.example.springbootexamples.webhook";

    @Override
    @SuppressWarnings("unchecked")
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {

        final Map<WebhookType, Class<? extends AbstractWebhookHandler>> handlerMap = new HashMap<>();
        ClassScanner.scanPackage(HANDLER_PACKAGE_NAME).forEach(handlerClass -> {

            final WebhookHandler handler = handlerClass.getAnnotation(WebhookHandler.class);

            if (null != handler)
                handlerMap.put(handler.type(), (Class<? extends AbstractWebhookHandler>) handlerClass);
        });

        // 将WebhookHandlerContext注册到Spring容器中
        beanFactory.registerSingleton(WebhookHandlerContext.class.getName(), new WebhookHandlerContext(handlerMap));
    }
}
```

利用`Spring`提供的`BeanFactoryPostProcessor`接口，可以很轻松的操作`IoC容器`，将`Context`注册到其中

通过扫描指定的包（`com.example.springbootexamples.webhook`）下的类文件，判断是否包含`@WebhookHandler`注解，然后将其设置到`WebhookHandlerContext.mHandlerMap`中，如此一来，咱们就不必手动的一个个的设置啦。

#### 使用 ####

上面的封装已经完成，现在就可以使用了，先来改造一下之前满屏的`if else`代码。

```java
@RestController
@RequiredArgsConstructor
public class WebhookHandlerController {
    private final WebhookHandlerContext webhookHandlerContext;

    @PostMapping("/webhook_handler")
    public Boolean webhookHandler(@RequestBody WebhookRequest request) {
        final AbstractWebhookHandler webhookHandler = webhookHandlerContext.getHandler(request.getType());
        return webhookHandler.handler(request); // 此处不必判空，getHandler()内部已做判断
    }
}
```

`Controller`中的代码目前变成了两行，将来大概率也不会快速增长了，接下来咱们就来实现具体的业务逻辑了。

在`com.example.springbootexamples.webhook`包下定义实现的业务逻辑类，代码如下：

```java
@WebhookHandler(type = WebhookType.TYPE1)
public class Type1WebhookHandler extends AbstractWebhookHandler {

    @Override
    public Boolean handler(WebhookRequest request) {
        // 1000 行逻辑处理代码
        return Boolean.TRUE;
    }
}

@WebhookHandler(type = WebhookType.TYPE2)
public class Type2WebhookHandler extends AbstractWebhookHandler {

    @Override
    public Boolean handler(WebhookRequest request) {
        // 100行逻辑代码
        return Boolean.TRUE;
    }
}
```

![611c8670-7566-43c6-b502-97e7b99994e5.png][]

改造到这里就结束了，将处理逻辑完美的解耦了出来，不再混杂到一起，而`Handler`之间还可以继续设计抽象父类，达到代码复用的目的，代码层次显得非常清晰。

#### 说明 ####

本文只起一个抛砖引玉的目的，文中实现的封装逻辑可能并不严谨，对于条件的策略处理也不太灵活，有需要的借鉴其思路即可。


[4daa5139-0b15-42e8-94fa-77a3bef9f5af.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210430/4daa5139-0b15-42e8-94fa-77a3bef9f5af.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg
[611c8670-7566-43c6-b502-97e7b99994e5.png]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20210430/611c8670-7566-43c6-b502-97e7b99994e5.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg