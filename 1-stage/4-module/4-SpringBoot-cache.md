第四部分 SpringBoot缓存深入

# 1 JSR 107

JSR 是 Java Specification Requests 的缩写，Java 规范请求，顾名思义提交Java规范，JSR-107 就是关于如何使用缓存的规范，是 Java 提供的一个接口规范，类似于 JDBC 规范，没有具体的实现，具体的实现就是 Redis 等这些缓存。

## 1.1 JSR 107 核心接口

Java Caching（JSR-107）定义了5个核心接口，分别是 CahcingProvider 、CacheManager、Cache、Entry、Expiry。

- CachingProvider（缓存提供者）：创建、配置、获取、管理和控制多个 CacheManager
- CacheManager（缓存管理器）：创建、配置、获取、管理和控制多个唯一命名的Cache，Cache存在于 CacheManager的上下文中。一个 CacheManager仅对应一个 CachingProvider。
- Cache（缓存）：是由 CacheManager管理的，CacheManager管理Cache的生命周期，Cache存在于 CacheManager 的上下文中，是一个类似 map 的数据结构，并临时存储以 key 为索引的值，一个 Cache 仅被一个 CacheManager所拥有。
- Entry（缓存键值对）是一个存储在 Cache 中的 key-value对。
- Expiry（缓存时效）：每一个存储在 Cache 中的条目都有一个定义的有效期。一旦超过这个时间，条目就自动过期，过期后，条目将不可访问、更新和删除操作。缓存有效期可以通过ExpiryPolicy设置。



## 1.2 JSR 107 图示

![image-20220624164448656](assest/image-20220624164448656.png)

一个应用里面可以由多个缓存提供者（CachingProvider），一个缓存缓存提供者可以获取到多个缓存管理器（CacheManager），一个缓存管理器管理着不同的缓存（Cache），缓存中是一个个的缓存键值对（Entry），每个entry都有一个有效期（Expiry）。缓存管理器和缓存之间的关系有点类似于数据库中的连接池和连接的关系。

使用 JSR-107 需导入的依赖

```xml

```

在实际使用时，并不会使用 JSR-107 提供的接口，而是具体使用 Spring 提供的缓存抽象。

# 2 Spring 的缓存抽象

[Spring中cache的官方文档](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache)

## 2.1 缓存抽象定义

Spring从3.1开始定义了 org.springframework.cache.Cache 和 org.springframework.cache.CacheManager 接口来统一不同的缓存技术；并支持使用 Java Caching （JSR-107）注解简化缓存开发。

Spring Cache 只负责维护抽象层，具体的实现由自己的技术选型来决定。将缓存处理和缓存技术解除耦合。

每次调用需要缓存功能的方法时，Spring会检查指定参数的指定的目标方法是否已经被调用过，如果有直接从缓存中获取方法调用后的结果，如果没有就调用方法并缓存结果返回给用户。下次调用直接从缓存中获取。

使用Spring缓存抽象时我们需要关注以下两点：

1. 确定哪些方法需要被缓存
2. 缓存策略

## 2.2 重要接口

- Cache：缓存抽象的规范接口，缓存实现有：RedisCache、EhCache、ConcurrentMapCache等。
- CacheManager：缓存管理器，管理 Cache 的声明周期

# 3 Spring 缓存使用

## 3.1 重要概念 缓存注解

案例实践之前，先介绍 Spring 提供的重要缓存及几个重要概念。

| 概念/注解      | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| Cache          | 缓存接口、定义缓存操作。实现有：RedisCache、EhCacheCache、ConcurrentMapCache 等 |
| CacheManager   | 缓存管理器，管理各种缓存（Cache）组件                        |
| @Cacheable     | 主要针对方法配置，能够根据方法的请求参数对其结果进行缓存     |
| @CacheEvict    | 清空缓存                                                     |
| @CachePut      | 保证方法被调用，又希望结果被缓存                             |
| @EnableCaching | 开启基于注解的缓存                                           |
| keyGenerator   | 缓存数据时 key 生成策略                                      |
| serialize      | 缓存数据时 value 序列化策略                                  |

**说明**：

1. @Cacheable 标注在方法上，表示该方法的结果需要被缓存起来，缓存的键由 keyGenerator 的策略决定，缓存的值的形式则由 serialize 序列化策略决定（序列化还是 json 格式）；标注上该注解之后，在缓存时效内再次调用该方法时将不会调用方法本身而是直接从缓存换取结果。
2. @CachePut 也标注在方法上，和 @Cacheable 相似也会将该方法的返回值缓存起来，不同的是标注 @CachePut 的方法每次都会被调用，而且每次都会将结果缓存起来，适用于对象的更新。

## 3.2 环境搭建

1. 创建 SpringBoot应用，选中 Mysql、Mybatis、Web 模块
2. 创建数据库





# 4 缓存自动配置原理源码剖析

# 5 @Cacheable 源码分析

# 6 @CahcePut、@CacheEvict、@CacheConfig

# 7 基于Redis的缓存实现

# 8 自定义 RedisCacheManager