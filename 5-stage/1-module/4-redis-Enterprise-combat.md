第四部分 Redis企业实战

# 14 架构设计

## 14.1 组件选择/多级

缓存的设计要分多个层次，在不同的层次上选择不同的缓存，包括JVM缓存、文件缓存和Redis缓存

### 14.1.1 JVM缓存

JVM缓存就是本地缓存，设计在应用服务器中（tomcat）。

通过可以采用Ehcache和Guava Cache，在互联网中，由于要处理高并发，通常选择Guava Cache。

适用本地（JVM）缓存的场景：

1. 对性能有非常高的要求。
2. 不经常变化
3. 占用内存不大
4. 有访问整个集合的需求
5. 数据允许不实时一致

### 14.1.2 文件缓存

这里的文件缓存是基于http协议的文件缓存，一般放在nginx中。

因为静态文件（比如css，js，图片）中，很多都是不经常更新的。nginx适用proxy_cache将用户的请求缓存到本地一个目录。下一个相同请求可以直接调取缓存文件，就不用去请求服务器了。

```properties
server {
    listen       80 default_server;        
    server_name  localhost;
    root /mnt/blog/;        
    location / {
    }
    #要缓存文件的后缀，可以在以下设置。
    location ~ .*\.(gif|jpg|png|css|js)(.*) {
        proxy_pass http://ip地址:90;
        proxy_redirect off;
        proxy_set_header Host $host;
        proxy_cache cache_one;
        proxy_cache_valid 200 302 24h;
        proxy_cache_valid 301 30d;
        proxy_cache_valid any 5m;                
        expires 90d;
        add_header wall  "hello lagou.";      
    }
 }
```



### 14.1.3 Redis缓存

分布式缓存，采用主从 + 哨兵或RedisCluster的方式缓存数据库的数据。

在实际开发中，作为数据库适用，数据要完整；作为缓存适用，作为Mybatis的二级缓存适用；

## 14.2 缓存大小

GuavaCache的缓存设置方式：

```java
CacheBuilder.newBuilder().maximumSize(num)	//超过num会按LRU算法来移除缓存
```

Nginx的缓存设置方式：

```properties
http {
    ...
    proxy_cache_path /path/to/cache levels=1:2 keys_zone=my_cache:10m max_size=10g 
    inactive=60m use_temp_path=off;
    
    server {
        proxy_cache mycache; 
        location / {
            proxy_pass http://localhost:8000; 
        }
    } 
}
```

Redis 缓存设置

```properties
maxmemory=num	#最大缓存容量	一般为内存的3/4
maxmemory-policy allkeys-lru
```

### 14.3.1 缓存淘汰策略的选择

- allkeys-lru：在不确定时一般采用策略。冷热数据交换
- volatile-lru：比allkeys-lru性能差，需要存过期时间
- allkeys-random：希望请求符合平均分布（每个元素以相同的概率被访问）
- 自己控制：volatile-ttl（把过期时间设置小一点）

## 14.3 key数量

官方说Redis单例能处理key：2.5亿个

一个key或是value大小 最大是512M

## 14.4 读写峰值

Redis采用的是基于内存的，采用的是单进程单线程模型的KV数据库，又C语言编码，官方提供的数据是可以达到110000+的QPS（每秒内查询次数）。80000的写



## 14.5 命中率

## 14.6 过期策略

## 14.7 性能监控指标

## 14.8 缓存预热

# 15 缓存问题

## 15.1 缓存穿透

## 15.2 缓存雪崩

## 15.3 缓存击穿

## 15.4 数据不一致

## 15.5 数据并发竞争

## 15.6 Hot Key

## 15.7 Big Key

# 16 缓存与数据库一致性

## 16.1 缓存更新策略

## 16.2 不同策略之间的优缺点

## 16.3 与Mybatis整合

https://gitee.com/turboYuu/spring-boot-source-code/tree/master/spring-boot-2.2.9.RELEASE/springboot_04_cache

# 17 分布式锁

## 17.1 watch

### 17.1.1 利用Watch实现Redis乐观锁



## 17.2 setnx

### 17.2.1 实现原理

### 17.2.2 实现方式

### 17.2.3 存在问题

### 17.2.4 本质分析

## 17.3 Redisson分布式锁的使用

## 17.4 分布式锁特征

## 17.5 分布式锁的实际应用

## 17.6 Zookeeper分布式锁的对比

# 18 分布式集群架构中的session分离

# 19 阿里Redis使用手册