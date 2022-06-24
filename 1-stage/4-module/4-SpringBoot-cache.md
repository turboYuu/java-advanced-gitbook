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

# 2 Spring 的缓存抽象

# 3 Spring 缓存使用

# 4 缓存自动配置原理源码剖析

# 5 @Cacheable 源码分析

# 6 @CahcePut、@CacheEvict、@CacheConfig

# 7 基于Redis的缓存实现

# 8 自定义 RedisCacheManager