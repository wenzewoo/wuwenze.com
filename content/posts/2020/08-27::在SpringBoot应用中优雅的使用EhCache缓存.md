+++
title = "在SpringBoot应用中优雅的使用EhCache缓存"
date = "2020-08-27 17:11:00"
url = "archives/699"
tags = ["Spring","Kotlin","EhCache"]
categories = ["后端"]
+++

SpringBoot家族提供的`spring-boot-starter-cache`使用`JCache（JSR-107）`注解统一了不同的缓存技术的使用，很是方便，本文主要说说集成`EhCache`的一种较为优雅的方案。

### 引入依赖 ###

```kotlin
implementation("net.sf.ehcache:ehcache")
implementation("org.springframework.boot:spring-boot-starter-cache")
```

### 开启缓存自动装配 ###

```kotlin
@EnableCaching // 开启
@SpringBootApplication
class MyApplication

fun main(args: Array<String>) {
    runApplication<MyApplication>(*args)
}
```

### 使用缓存 ###

ehcache的缓存使用较为麻烦（涉及到XML配置文件）本文的目的也是为了简化ehcache的使用，先来看看正常的使用流程。

#### 在classpath下新建ehcache.xml配置 ####

```xml
<ehcache>
    <cache 
        name="AttachmentService:findByIdOrKey" 
        timeToLiveSeconds="1200" 
        timeToIdleSeconds="1200" 
        maxElementsInMemory="500">
    </cache>
</ehcache>
```

> 最简配置，有关ehcache的更多配置，请查阅文档，本文只做演示。

#### 使用JCache（JSR-107）注解标记要缓存的方法 ####

```kotlin
@Service
class AttachmentService(
        private val attachmentMapper: AttachmentMapper) {

    companion object {
        // 缓存的命名空间，需要先在ehcache.xml定义
        const val CacheNameWithFindByIdOrKey = "AttachmentService:findByIdOrKey"
    }

    // 标记要缓存的方法
    @Cacheable(value = [CacheNameWithFindByIdOrKey], key = "#idOrKey")
    fun findByIdOrKey(idOrKey: String): Attachment {
        return attachmentMapper.findByIdOrKey(idOrKey)
                ?: throw BusinessException(Errors.NOT_FOUND.build("没有找到文件"))
    }

    // 方法调用后，自动删除指定key的缓存
    @CacheEvict(value = [CacheNameWithFindByIdOrKey], key = "#idOrKey")
    fun deleteFile(idOrKey: String, userId: Int): Boolean {
        
        return true
    }
}
```

到这一步，咱们的工作就做完了，目前来说，比较麻烦的是，每次定义缓存都需要先到`ehcache.xml`中先定义命名空间，设置过期时间等，如果我们需要不同粒度的缓存过期时间，那配置起来将十分麻烦，且不方便管理，接下来咱们进行改造。

### 改造方案 ###

`cacheManager`其实在管理的Spring的容器中的，那么可以合理的利用Spring的特性，在启动阶段动态的注入`ehcache.xml`配置文件，而不需要手动配置。

改造流程简要说明一下：

1.  定义注解：`@EhCacheConfig`，其中包含`ehcache.xml`中需要的常用配置项，后面将用来与`@Cacheable`配合使用。
2.  利用`FactoryBean`特性，在Spring容器创建`EhCacheManagerFactoryBean`时，根据`@EhCacheConfig`注解配置，动态生成`ehcache.xml`配置文件。
3.  将`EhCacheManagerFactoryBean.object`作为`CacheManager`的实例，注入Spring容器中。

#### 定义注解 ####

```kotlin
@Target(AnnotationTarget.FUNCTION, AnnotationTarget.PROPERTY_GETTER, AnnotationTarget.PROPERTY_SETTER)
@kotlin.annotation.Retention(AnnotationRetention.RUNTIME)
annotation class EhCacheConfig(
        /** * 缓存最大个数（内存） */
        val maxElementsInMemory: Int = 2500,
        /** * 缓存自最后一次访问时起的失效间隔时间 */
        val timeToIdleSeconds: Int = 10 * 60,
        /** * 缓存自创建时起的失效间隔时间 */
        val timeToLiveSeconds: Int = 6 * 60
)
```

> 注解中只举例说明了几个常用的属性，如果需要，可以加入更多。

#### 利用FactoryBean动态生成ehcache.xml文件 ####

```kotlin
@Component
class EhCacheDefinitionBeanFactoryProcessor : BeanFactoryPostProcessor {
    val beanDefinitionSet: MutableSet<BeanDefinition> = Sets.newHashSet()

    @Throws(BeansException::class)
    override fun postProcessBeanFactory(configurableListableBeanFactory: ConfigurableListableBeanFactory) {
        configurableListableBeanFactory
                .beanNamesIterator.forEachRemaining { e: String? ->
                    try {
                        beanDefinitionSet.add(configurableListableBeanFactory.getBeanDefinition(e!!))
                    } catch (ignored: Throwable) {
                        // Ignore this exception
                    }
                }
    }
}


@Component
class EhCacheConfigResourceFactoryBean(
        private val ehCacheDefinitionBeanFactoryProcessor: EhCacheDefinitionBeanFactoryProcessor) : FactoryBean<Resource> {
    private val log: Logger = LoggerFactory.getLogger(this.javaClass)

    override fun getObject(): Resource {
        var ehCacheConfig = "<?xml version="1.0" encoding="UTF-8"?>n<ehcache>n"
        val ehCacheConfigItems: MutableList<String> = arrayListOf()
        ehCacheDefinitionBeanFactoryProcessor.beanDefinitionSet.forEach(
                Consumer forEach@{ e: BeanDefinition ->
                    val beanClassName = e.beanClassName
                    if (beanClassName.isNullOrBlank()) {
                        return@forEach   // continue
                    }

                    // Filter out annotated functions
                    val methods = getMethods(beanClassName,
                            Filter { method: Method ->
                                (null != method.getAnnotation(Cacheable::class.java)
                                        && null != method.getAnnotation(EhCacheConfig::class.java))
                            })

                    // Building XML configuration items
                    Arrays.stream(methods).forEach { method: Method? -> ehCacheConfigItems.add(buildEhCacheConfigItem(method)) }
                })

        // Build ehcache.xml
        ehCacheConfig += """
            ${CollectionUtil.join(ehCacheConfigItems.iterator(), "n")}
            </ehcache>
            """.trimIndent()
        log.info("Build ehcache.xml: n{}", ehCacheConfig)
        return ByteArrayResource(ehCacheConfig.toByteArray())
    }

    override fun getObjectType(): Class<*> {
        return Resource::class.java
    }

    companion object {
        private fun buildEhCacheConfigItem(method: Method?): String {
            val cacheable = method!!.getAnnotation(Cacheable::class.java)
            val ehCacheConfig = method.getAnnotation(EhCacheConfig::class.java)
            var cacheName = ""
            if (cacheable.value.isNotEmpty()) {
                cacheName = cacheable.value.get(0)
            } else if (cacheable.cacheNames.isNotEmpty()) {
                cacheName = cacheable.cacheNames.get(0)
            }
            if (cacheName.isBlank()) {
                cacheName = method.declaringClass.simpleName + "_" + method.name
            }
            return """
                <cache 
                    name="$cacheName" 
                    timeToLiveSeconds="${ehCacheConfig.timeToLiveSeconds}" 
                    timeToIdleSeconds="${ehCacheConfig.timeToIdleSeconds}" 
                    maxElementsInMemory="${ehCacheConfig.maxElementsInMemory}">
                </cache>"""
        }

        private fun getMethods(beanClassName: String?, filter: Filter<Method>): Array<Method?> {
            return try {
                val clazz = Class.forName(beanClassName)
                ReflectUtil.getMethods(clazz, filter)
            } catch (e: Throwable) {
                arrayOfNulls(0)
            }
        }
    }

}
```

> 这里其实还需要考虑classpath已经存在ehcache.xml配置的情况，为了简单起见，直接生成全新的，如果有此需求，则自行完善。

#### 注入容器 ####

```kotlin
@Configuration
class CacheConfiguration(
        val ehCacheConfigResourceFactoryBean: EhCacheConfigResourceFactoryBean) {
    @Bean
    fun ehCacheManagerFactoryBean(): EhCacheManagerFactoryBean {
        val cacheManagerFactoryBean = EhCacheManagerFactoryBean()
        val configResource = ehCacheConfigResourceFactoryBean.getObject()
        // Use dynamically generated ehcache.xml
        cacheManagerFactoryBean.setConfigLocation(configResource)
        cacheManagerFactoryBean.setShared(true)
        return cacheManagerFactoryBean
    }

    @Bean
    fun cacheManager(): CacheManager {
        val ehCacheCacheManager = EhCacheCacheManager()
        ehCacheCacheManager.cacheManager = ehCacheManagerFactoryBean().`object`;
        return ehCacheCacheManager
    }

}
```

#### 最简使用 ####

到这就处理完成了，咱们来看看优化后的使用方式。

```kotlin
@Service
class AttachmentService(
        private val attachmentMapper: AttachmentMapper) {

    companion object {
        // 缓存的命名空间，无需ehcache.xml配置文件，自动生成。
        const val CacheNameWithFindByIdOrKey = "AttachmentService:findByIdOrKey"
    }

    // 标记要缓存的方法
    @Cacheable(value = [CacheNameWithFindByIdOrKey], key = "#idOrKey")
    // 定义缓存规则
    @EhCacheConfig(maxElementsInMemory = 500, timeToLiveSeconds = 20 * 60, timeToIdleSeconds = 20 * 60)
    fun findByIdOrKey(idOrKey: String): Attachment {
        return attachmentMapper.findByIdOrKey(idOrKey)
                ?: throw BusinessException(Errors.NOT_FOUND.build("没有找到文件"))
    }

    // 方法调用后，自动删除指定key的缓存
    @CacheEvict(value = [CacheNameWithFindByIdOrKey], key = "#idOrKey")
    fun deleteFile(idOrKey: String, userId: Int): Boolean {
        
        return true
    }
}
```

### 启动项目 ###

启动项目后，观察日志，发现已经自动生成了ehcache.xml文件  
![114455\_34ede666\_955503][114455_34ede666_955503]

### 总结 ###

优化后，咱们直接省略了`ehcache.xml`配置文件，通过注解的方式自动生成，简化了配置，进一步避免了难以管理的问题。

### 需要优化的点 ###

 *  兼容已有ehcache.xml配置文件的情况
 *  分布式系统下的ehcache数据同步


[114455_34ede666_955503]: https://wenzewoo-cdn.oss-cn-chengdu.aliyuncs.com/images/20200827/dfa11fe7-c762-4b38-9a80-674c79b22205.png?x-oss-process=image/auto-orient,1/interlace,1/quality,q_70/format,jpg