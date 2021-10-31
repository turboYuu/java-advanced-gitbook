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

命中：可以直接通过缓存获取到需要的数据。

不命中：无法直接通过缓存获取到想要的数据，需要再次查询数据库或者执行其他的操作。其原因可能是由于缓存中根本不存在，挥着缓存已经过期。

通常来讲，缓存的命中率越高则表示适用缓存的收益越高，应用的性能越好（响应时间越短，吞吐量越高），抗并发的能力越强。

由此可见，在高并发的互联网系统中，缓存的命中率是至关重要的指标。

通过info命令可以监控服务器状态：

```shell
127.0.0.1:6379> info 
# Server
redis_version:5.0.5
redis_git_sha1:00000000
redis_git_dirty:0
redis_build_id:e188a39ce7a16352
redis_mode:standalone
os:Linux 3.10.0-229.el7.x86_64 x86_64 arch_bits:64
#缓存命中
keyspace_hits:1000  
#缓存未命中
keyspace_misses:20 
used_memory:433264648  
expired_keys:1333536  
evicted_keys:1547380
```



命中率 = 1000/(1000+20) = 83%

一个缓存失效的机制，和过期时间设计良好的系统，命中率可以做到95%以上。

影响命中率的因素：

1. 缓存的数量越少命中率越高，比如缓存单个对象的命中率要高于缓存集合
2. 过期时间越长命中率越高
3. 缓存越大，缓存的对象越多，则命中的越多

## 14.6 过期策略

参考上一部分的**删除策略**

## 14.7 性能监控指标

利用info命名就可以了解Redis的状态了，主要监控指标有：

```yaml
connected_clients:68				#连接的客户端数量
used_memory_rss_human:847.62M 		#系统给redis分配的内存
used_memory_peak_human:794.42M  	#内存使用的峰值大小
total_connections_received:619104 	#服务器已接受的连接请求数量
instantaneous_ops_per_sec:1159 		#服务器每秒钟执行的命令数量     qps
instantaneous_input_kbps:55.85 		#redis网络入口kps
instantaneous_output_kbps:3553.89 	#redis网络出口kps
rejected_connections:0 				#因为最大客户端数量限制而被拒绝的连接请求数量 
expired_keys:0 						#因为过期而被自动删除的数据库键数量
evicted_keys:0 						#因为最大内存容量限制而被驱逐（evict）的键数量 
keyspace_hits:0 					#查找数据库键成功的次数
keyspace_misses:0 					#查找数据库键失败的次数
```

Redis监控平台：

grafana、prometheus以及redis_exporter。

## 14.8 缓存预热

缓存预热就是系统启动前，提前将相关的缓存数据直接加载到缓存系统。避免在用户请求的时候，先查询数据，然后再将数据缓存的问题！用户直接查询事先被预热的缓存数据。

加载缓存思路：

- 数据量不大，可以在项目启动的时候自动进行加载
- 利用定时任务刷新缓存，将数据库的数据刷新到缓存中

# 15 缓存问题

## 15.1 缓存穿透

一般的缓存西永，都是按照key去缓存查询，如果不存在对应的value，就应该去后端系统查找（比如DB）。

缓存穿透是指在高并发下查询key不存在的数据（不存在的key），会穿过缓存查询数据库。导致数据库压力过大而宕机。

解决方案：

- 对查询结果为空的情况也进行缓存，缓存时间（ttl）设置短一点，或者该key对应的数据insert了之后清理缓存。

  问题：*缓存太多，空值占用了更多的空间*

- 使用布隆过滤器。在缓存之前再加一层布隆过滤器，在其查询的时候先去布隆过滤器查询key是否存在，如果不存在就直接返回，存在再查询缓存和DB。



![image-20211031155241798](assest/image-20211031155241798.png)

布隆过滤器（Bloom Filter）是1970年由布隆提出的，它实际上是一个很长的二进制和一系列随机hash映射函数。

布隆过滤器可以用于检索一个元素是否在一个集合中。它的优点是空间效率和查询时间都远远超过一般的算法。

![image-20211031155521064](assest/image-20211031155521064.png)

![image-20211031155535344](assest/image-20211031155535344.png)

布隆过滤器的原理是，当一个元素被加入集合时，通过K个Hash函数将这个元素映射成一个数组中的K个点，把它们置为1。检索时，只需要看看这些点是不是都是1就大约知道集合中有没有它了：如果这些点有任何一个0，则被检元素一定不在；如果都是1，则被检元素很可能存在。这就是布隆过滤器的基本思想。

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