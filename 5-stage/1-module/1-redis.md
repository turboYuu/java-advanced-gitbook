# 第一部分 Redis快速实战

## 1.1 缓存原理与设计

### 1.1.1 缓存基本思想

#### 1.1.1.1 缓存的使用场景

##### DB缓存，减轻DB服务器压力

##### 提高系统响应

### 1.2.1 缓存的读写模式

#### Cache Aside pattern(常用)

#### Read/Write Through Pattern 

#### Write Behind Caching Pattern

## 1.2 Redis简介和安装

### 1.2.1 Redis简介

### 1.2.2 Redis单机版安装和使用

使用5.0.5稳定版

#### Redis下载



#### Redis安装环境



#### Redis安装

1.安装c语言需要的GCC环境

```
yum install -y gcc-c++
yum install -y wget
```

2.下载并解压解压Redis源码压缩包

```
wget http://download.redis.io/releases/redis-5.0.5.tar.gz 
tar -zxf redis-5.0.5.tar.gz  
```

3.编译Redis源码，进入redis-5.0.5目录，执行编译命令

```
cd redis-5.0.5/src 
make 
```

4.安装Redis，需要通过PREFIX指定安装路径

```
mkdir /usr/redis -p
make install PREFIX=/usr/redis  
```

#### Redis 启动

##### 前端启动

- 启动命令：`redis-server`，直接运行`./bin/redis-server`将以前端模式启动

  ```
  ./bin/redis-server
  ```

- 关闭命令：`ctrl+c`

- 启动缺点：客户端关闭则redis-server程序结束，不推荐

![image-20210824162959478](assest/image-20210824162959478.png)

##### 后端启动（守护进程启动）

1.拷贝redis-5.0.5/redis/redis.conf配置文件到Redis安装目录的bin目录

```
cp  redis.conf /usr/redis/bin/
```

2.修改redis.conf

```
# 将`daemonize`由`no`改为`yes`
daemonize yes

# 默认绑定的是回环地址，默认不能被其他机器访问 
# bind 127.0.0.1

# 是否开启保护模式，由yes该为no
protected-mode no  
```

3.启动服务

```
./redis-server redis.conf
```

![image-20210825161946150](assest/image-20210825161946150.png)

##### 后端启动的关闭方式

```shell
./redis-cli shutdown
```

##### 命令说明

![image-20210824163749902](assest/image-20210824163749902.png)

- `redis-server`：启动`redis`服务
- `redis-cli`：进入redis命令客户端
- `redis-benchmark`：性能测试的工具
- `redis-check-aof`：`aof`文件进行检查的工具
- `redis-check-dump`：`rdb`文件进行检查的工具
- `redis-sentinel`：启动哨兵监控服务

#### Redis命令行客户端

- 命令格式

  ```shell
  ./redis-cli -h 127.0.0.1 -p 6379
  ```

- 参数说明

  ```
  -h：redis服务器的ip地址 
  -p：redis实例的端口号
  ```

- 默认方式

  如果不指定主机和端口号也可以

  默认主机地址：127.0.0.1

  默认端口号：6379

  ```
  ./redis-cli
  ```



## 1.3 Redis客户端访问

### 1.3.1 java程序访问Redis

采用jedis API进行访问即可

https://gitee.com/turboYuu/redis5.1/tree/master/lab/jedis_demo

1.关闭防火墙

```
systemctl stop firewalld（默认）
systemctl disable firewalld.service（设置开启不启动）
```

2.新建maven项目后导入jedis包

pom.xml

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.9.0</version>
</dependency>
```

3.程序

```java
import org.junit.Test;
import redis.clients.jedis.Jedis;

public class TestRedis {

    @Test
    public void test(){
        Jedis jedis = new Jedis("192.168.1.135",6379);
        jedis.set("name","zhangsan");
        System.out.println(jedis.get("name"));

        jedis.lpush("list1","1","2","3","4","5");
        System.out.println(jedis.llen("list1"));
    }
}
```

### 1.3.2 Spring访问Redis

https://gitee.com/turboYuu/redis5.1/tree/master/lab/spring_redis

**1.新建Maven项目，引入Spring依赖**

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-beans</artifactId>
    <version>5.2.5.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-core</artifactId>
    <version>5.2.5.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.2.5.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <version>5.2.5.RELEASE</version>
</dependency>
<dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.12</version>
    <scope>test</scope>
</dependency>

```

**2.添加redis依赖**

```
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-redis</artifactId>
    <version>1.0.3.RELEASE</version>
</dependency>
```

**3.添加Spring配置文件**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="propertyConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="locations">
            <list>
                <value>classpath:redis.properties</value>
            </list>
        </property>
    </bean>

    <!--redis config-->
    <bean id="jedisPoolConfig" class="redis.clients.jedis.JedisPoolConfig">
        <property name="maxActive" value="${redis.pool.maxActive}"/>
        <property name="maxIdle" value="${redis.pool.maxIdle}"/>
        <property name="maxWait" value="${redis.pool.maxWait}"/>
        <property name="testOnBorrow" value="${redis.pool.testOnBorrow}"/>
    </bean>

    <bean id="jedisConnectionFactory" class="org.springframework.data.redis.connection.jedis.JedisConnectionFactory">
        <property name="hostName" value="${redis.server}"/>
        <property name="port" value="${redis.port}"/>
        <property name="timeout" value="${redis.timeout}" />
        <property name="poolConfig" ref="jedisPoolConfig" />
    </bean>


    <bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
        <property name="connectionFactory" ref="jedisConnectionFactory"/>
        <property name="keySerializer">
            <bean class="org.springframework.data.redis.serializer.StringRedisSerializer"></bean>
        </property>
        <property name="valueSerializer">
            <bean class="org.springframework.data.redis.serializer.StringRedisSerializer"></bean>
        </property>
    </bean>

</beans>
```

**4.添加redis.properties**

```
redis.pool.maxActive=100
redis.pool.maxIdle=50
redis.pool.maxWait=1000
redis.pool.testOnBorrow=true

redis.timeout=50000
redis.server=192.168.1.135
redis.port=6379
```

5.编写测试用例

```java
import org.junit.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.test.context.ContextConfiguration;
import org.springframework.test.context.junit4.AbstractJUnit4SpringContextTests;

@ContextConfiguration("classpath:redis.xml")
public class TestRedis extends AbstractJUnit4SpringContextTests {

    @Autowired
    RedisTemplate<String,String> redisTemplate;

    @Test
    public void testConn(){
        redisTemplate.opsForValue().set("name-s","lisi");
        System.out.println(redisTemplate.opsForValue().get("name-s"));
    }
}
```

### 1.3.3 SpringBoot访问Redis

https://gitee.com/turboYuu/redis5.1/tree/master/lab/springboot_redis

**1.新建springboot项目**

选择Spring Web依赖，添加redis依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

**2.添加配置文件application.yaml**

```yaml
spring:
  redis:
    host: 192.168.1.135
    port: 6379
    jedis:
      pool:
        min-idle: 0
        max-active: 80
        max-wait: 30000
        max-idle: 8
        timeout: 3000
```

**3.添加配置类RedisConfig**

```java
package com.turbo.sbr.cache;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.StringRedisSerializer;

/**
 * 配置类
 */
@Configuration
public class RedisConfig {

    @Autowired
    RedisConnectionFactory factory;

    @Bean
    public RedisTemplate<String,Object> redisTemplate(){
        RedisTemplate<String,Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new StringRedisSerializer());
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashValueSerializer(new StringRedisSerializer());
        redisTemplate.setConnectionFactory(factory);
        return redisTemplate;
    }
}
```

**4.添加RedisController**

```java
ackage com.turbo.sbr.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import java.util.concurrent.TimeUnit;

@RestController
@RequestMapping(value = "/redis")
public class RedisController {

    @Autowired
    RedisTemplate redisTemplate;

    @GetMapping("/put")
    public String put(@RequestParam(required = true) String key, 
    	@RequestParam(required = true) String value){
        redisTemplate.opsForValue().set(key,value, 20,TimeUnit.SECONDS);
        return "suc";
    }

    @GetMapping("/get")
    public String get(@RequestParam(required = true) String key){
        return (String) redisTemplate.opsForValue().get(key);
    }
}
```

**5.修改Application并运行**

```java
package com.turbo.sbr;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cache.annotation.EnableCaching;

@SpringBootApplication
@EnableCaching
public class SpringbootRedisApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringbootRedisApplication.class, args);
    }
}
```



## 1.4 Redis数据类型选择和应用场景

Redis是一个Key-Value的存储系统，使用ANSI C语言编写。

key的类型是字符串。

value的数据类型有：

- 常用的：string字符串类型、list列表类型、set集合类型、sortedset（zset）有序集合类型、hash类型。
- 不常用的：bitmap位图类型，geo地理位置类型。

Redis5.0新增一种：stream类型

注意：Redis中命令忽略大小写，key不忽略大小写。

### 1.4.1 Redis的key的设计



### 1.4.2 string字符串类型

Redis的string能表达3中值的类型：字符串、整数、浮点数100.01是个六位数的串

常见命令：

| 命令名称 |                      | 命令描述                                                     |
| -------- | -------------------- | ------------------------------------------------------------ |
| set      | set key value        | 赋值                                                         |
| get      | get key              |                                                              |
| getset   |                      |                                                              |
| setnx    | setnx key value      | 当key不存在时才用赋值<br/>set key value NX PX 3000 原子操作，px 设置毫秒数 |
| append   | append key value     |                                                              |
| strlen   | strlen key           |                                                              |
| incr     | incr key             |                                                              |
| incrby   | incrby key increment |                                                              |
| decr     | decr key             |                                                              |
| decrby   | decrby key decrement |                                                              |

应用场景：

1.key和命令是字符串

2.普通的赋值

3.incr用于乐观锁

incr：递增数字，可用于实现乐观锁watch(事务)

4.setnx用于分布式锁

当key不存在时采用赋值，可用于实现分布式锁



举例：

setnx:

```
127.0.0.1:6379> setnx name zhangf       #如果name不存在赋值      
(integer) 1
127.0.0.1:6379> setnx name zhaoyun      #再次赋值失败 
(integer) 0
127.0.0.1:6379> get name "zhangf"
```

set

```
127.0.0.1:6379> set age 18 NX PX 10000  #如果不存在赋值 有效期10秒
OK
127.0.0.1:6379> set age 20 NX           #赋值失败 
(nil)
127.0.0.1:6379> get age                 #age失效 
(nil)
127.0.0.1:6379> set age 30 NX PX 10000  #赋值成功
OK
127.0.0.1:6379> get age 
"30"
```

### 1.4.3 list列表类型

list列表类型可以存储有序，可重复的元素

获取头部或尾部附件的记录是极快的

list的元素个数最多为2^32-1个（40亿）

常见命令：

| 命令名称   | 命令格式 | 描述 |
| ---------- | -------- | ---- |
| lpush      |          |      |
| lpop       |          |      |
| rpush      |          |      |
| rpop       |          |      |
| lpushx     |          |      |
| rpushx     |          |      |
| blpop      |          |      |
| brpop      |          |      |
| llen       |          |      |
| lindex     |          |      |
| lrange     |          |      |
| lrem       |          |      |
| lset       |          |      |
| ltrim      |          |      |
| rpoplpush  |          |      |
| brpoplpush |          |      |
| linsert    |          |      |

应用场景：

1.作为栈或队列使用

列表有序可以作为栈和队列使用

2.可用于各种列表



举例：

```
127.0.0.1:6379> lpush list:1 1 2 3 4 5 3
(integer) 6
127.0.0.1:6379> lrange list:1 0 -1
1) "3"
2) "5"
3) "4"
4) "3"
5) "2"
6) "1"
127.0.0.1:6379> lpop list:1
"3"
127.0.0.1:6379> rpop list:1
"1"
127.0.0.1:6379> lindex list:1 1
"4"
127.0.0.1:6379> lrange list:1 0 -1
1) "5"
2) "4"
3) "3"
4) "2"
127.0.0.1:6379> rpoplpush list:1 list:2
"2"
127.0.0.1:6379> lrange list:1 0 -1
1) "5"
2) "4"
3) "3"
127.0.0.1:6379> 

```

### 1.4.4 set集合类型

Set：无需、唯一元素

集合中最大的成员数为2^32-1

常见操作列表如下：

| 命令名称    | 命令格式 | 描述 |
| ----------- | -------- | ---- |
| sadd        |          |      |
| srem        |          |      |
| smembers    |          |      |
| spop        |          |      |
| srandmember |          |      |
| scard       |          |      |
| sismember   |          |      |
| sinter      |          |      |
| sdiff       |          |      |
| sunion      |          |      |

应用场景：

适用于不能重复的且不需要顺序的数据结构



举例：

```
127.0.0.1:6379> sadd set:1 a b c d a
(integer) 4
127.0.0.1:6379> smembers set:1
1) "c"
2) "a"
3) "b"
4) "d"
127.0.0.1:6379> srandmember set:1
"a"
127.0.0.1:6379> smembers set:1
1) "a"
2) "b"
3) "d"
4) "c"
127.0.0.1:6379> spop set:1
"b"
127.0.0.1:6379> smembers set:1
1) "d"
2) "c"
3) "a"
127.0.0.1:6379> sadd set:2 b c r f
(integer) 4
127.0.0.1:6379> sinter set:1 set:2
1) "c"
127.0.0.1:6379> 

```

### 1.4.5 sortedset有序集合类型

SortedSet(ZSet)有序集合：元素本身是无序不重复的

每个元素关联一个分数（score）

可按分数排序，分数可重复

常见命令：

| 命令名称  | 命令格式 | 描述 |
| --------- | -------- | ---- |
| zadd      |          |      |
| zrem      |          |      |
| zcard     |          |      |
| zcount    |          |      |
| zincrby   |          |      |
| zscore    |          |      |
| zrank     |          |      |
| zrevrank  |          |      |
| zrange    |          |      |
| zrevrange |          |      |

应用场景：

由于可以按照分值排序，所以适合各种排行榜。



举例：

```
127.0.0.1:6379> zadd hit:1 100 item1 20 item2 45 item3
(integer) 3
127.0.0.1:6379> zcard hit:1
(integer) 3
127.0.0.1:6379> zscore hit:1 item3
"45"
127.0.0.1:6379> zrevrange hit:1 0 -1
1) "item1"
2) "item3"
3) "item2"
127.0.0.1:6379> 

```

### 1.4.6 hash类型（散列表）

Redis hash是一个string类型的field和value的映射表，它提供了字段和字段值的映射。

每个hash可以存储2^32-1键值对（40多亿）。

![image-20210825171929115](assest/image-20210825171929115.png)

常见操作命令：

| 命令名称 | 命令格式 | 描述 |
| -------- | -------- | ---- |
| hset     |          |      |
| hmset    |          |      |
| hsetnx   |          |      |
| hexists  |          |      |
| hget     |          |      |
| hmget    |          |      |
| hgetall  |          |      |
| hdel     |          |      |
| hincrby  |          |      |
| hlen     |          |      |

应用场景：

对象的存储，表数据的映射

举例：

```shell
127.0.0.1:6379> hmset user:001 username zhangfei password 111 age 23 sex M
OK
127.0.0.1:6379> hgetall user:001
1) "username"
2) "zhangfei"
3) "password"
4) "111"
5) "age"
6) "23"
7) "sex"
8) "M"
127.0.0.1:6379> hget user:001 username
"zhangfei"
127.0.0.1:6379> hincrby user:001 age 1
(integer) 24
127.0.0.1:6379> hlen user:001
(integer) 4

```

### 1.4.7 bitmap位图类型

bitmap是进行位操作的

通过一个bit为来表示某个元素对应的值或状态，其中的key就是对应元素本身。

bitmap本身会极大的节省空间。

常见操作命令：

| 命令名称 | 命令格式 | 描述 |
| -------- | -------- | ---- |
| setbit   |          |      |
| getbit   |          |      |
| bitcount |          |      |
| bitpos   |          |      |
| bitop    |          |      |

应用场景

1.用户每月签到，用户id为key，日期作为偏移量，1表示签到

2.统计活跃用户，日期为key，用户id为偏移量，1表示活跃

3.查询用户状态，日期为key，用户id为偏移量，1表示在线

举例：

```
127.0.0.1:6379> setbit user:sign:1000 20200101 1
(integer) 0
127.0.0.1:6379> setbit user:sign:1000 20200103 1
(integer) 0
127.0.0.1:6379> getbit user:sign:1000 20200101
(integer) 1
127.0.0.1:6379> getbit user:sign:1000 20200102
(integer) 0
127.0.0.1:6379> bitcount user:sign:1000
(integer) 2
127.0.0.1:6379> bitpos user:sign:1000 1
(integer) 20200101
127.0.0.1:6379> setbit 20200201 1000 1
(integer) 0
127.0.0.1:6379> setbit 20200202 1001 1
(integer) 0
127.0.0.1:6379> setbit 20200201 1002 1
(integer) 0
127.0.0.1:6379> bitcount 20200201 
(integer) 2
127.0.0.1:6379> bitop or desk1 20200201 20200202
(integer) 126
127.0.0.1:6379> bitcount desk1 
(integer) 3

```

### 1.4.8 geo地理位置类型

geo是Redis用来处理位置信息的。在Redis3.2中使用。主要是利用了Z阶曲线、Base32编码和geohash算法

**Z阶曲线**

**Base32编码**

**geohash算法**

常见操作命令：

| 命令名称          | 命令格式 | 描述 |
| ----------------- | -------- | ---- |
| geoadd            |          |      |
| geohash           |          |      |
| geopos            |          |      |
| geodist           |          |      |
| georadiusbymember |          |      |

应用场景：

1.记录地理位置

2.计算距离

3.查找“附近的人”



举例：

```
127.0.0.1:6379> geoadd user:addr 116.31 40.05 zhangf 116.38 39.88 zhaoyun 116.47 40.00 diaochan
(integer) 3
127.0.0.1:6379> geohash user:addr zhangf diaochan
1) "wx4eydyk5m0"
2) "wx4gd3fbgs0"
127.0.0.1:6379> geopos user:addr zhaoyun
1) 1) "116.38000041246414185"
   2) "39.88000114172373145"
127.0.0.1:6379> geodist user:addr zhangf diaochan
"14718.6972"
127.0.0.1:6379> geodist user:addr zhangf diaochan km
"14.7187"
127.0.0.1:6379> georadiusbymember user:addr zhangf 20 km withcoord withdist
1) 1) "zhangf"
   2) "0.0000"
   3) 1) "116.31000012159347534"
      2) "40.04999982043828055"
2) 1) "zhaoyun"
   2) "19.8276"
   3) 1) "116.38000041246414185"
      2) "39.88000114172373145"
3) 1) "diaochan"
   2) "14.7187"
   3) 1) "116.46999925374984741"
      2) "39.99999991084916218"
127.0.0.1:6379> georadiusbymember user:addr zhangf 20 km withcoord withdist count 3 asc
1) 1) "zhangf"
   2) "0.0000"
   3) 1) "116.31000012159347534"
      2) "40.04999982043828055"
2) 1) "diaochan"
   2) "14.7187"
   3) 1) "116.46999925374984741"
      2) "39.99999991084916218"
3) 1) "zhaoyun"
   2) "19.8276"
   3) 1) "116.38000041246414185"
      2) "39.88000114172373145"
      
      
```

### 1.4.9 stream数据流类型

stream是Redis5.0后新增的数据结构，用于可持久化的消息队列。

几乎满足了消息队列具备的全部内容，包括：

- 消息ID的序列化生成
- 消息遍历
- 消息的阻塞和非阻塞读取
- 消息的分组消息
- 未完成消息的处理
- 消息队列监控

每个Stream都有唯一的名称，它就是Redis的key，首次使用xadd指令最佳消息时自动创建。

常见操作命令：

| 命令名称   | 命令格式 | 描述 |
| ---------- | -------- | ---- |
| xadd       |          |      |
| xread      |          |      |
| xrange     |          |      |
| xrevrange  |          |      |
| xdel       |          |      |
| xgroup     |          |      |
| xgroup     |          |      |
| xgroup     |          |      |
| xgroup     |          |      |
| xreadgroup |          |      |

应用场景：消息队列的使用

举例：

```properties
127.0.0.1:6379> xadd topic:001 * name zhangfei age 23
"1629825153821-0"
127.0.0.1:6379> xadd topic:001 * name zhaoyun age 24 name diaochan age 16
"1629825177188-0"
127.0.0.1:6379> xrange topic:001 - +
1) 1) "1629825153821-0"
   2) 1) "name"
      2) "zhangfei"
      3) "age"
      4) "23"
2) 1) "1629825177188-0"
   2) 1) "name"
      2) "zhaoyun"
      3) "age"
      4) "24"
      5) "name"
      6) "diaochan"
      7) "age"
      8) "16"
127.0.0.1:6379> xread COUNT 1 streams topic:001 0
1) 1) "topic:001"
   2) 1) 1) "1629825153821-0"
         2) 1) "name"
            2) "zhangfei"
            3) "age"
            4) "23"
127.0.0.1:6379> xgroup create topic:001 group1 0
OK
#消费第一条
127.0.0.1:6379> xreadgroup group group1 cus1 count 1 streams topic:001 >
1) 1) "topic:001"
   2) 1) 1) "1629825153821-0"
         2) 1) "name"
            2) "zhangfei"
            3) "age"
            4) "23"
127.0.0.1:6379> xreadgroup group group1 cus1 count 1 streams topic:001 >
1) 1) "topic:001"
   2) 1) 1) "1629825177188-0"
         2) 1) "name"
            2) "zhaoyun"
            3) "age"
            4) "24"
            5) "name"
            6) "diaochan"
            7) "age"
            8) "16"
127.0.0.1:6379> xreadgroup group group1 cus1 count 1 streams topic:001 >
(nil)

```



# 第二部分 Redis扩展功能

## 2.1 发布与订阅

### 2.1.1 频道/模式的订阅与退订



### 2.1.2 发布订阅的机制



### 2.1.3 使用场景 哨兵模式、Redisson框架使用

## 2.2 事务

### 2.2.1 ACID回顾

### 2.2.2 Redis事务

### 2.2.3 事务命令

### 2.2.4 事务机制

#### 事务的执行

#### Watch的执行

#### Redis的弱事务性

## 2.3 Lua脚本

### 2.3.1 创建并修改lua环境

### 2.3.2 lau环境协作组件

### 2.3.3 EVAL/EVALSHA命令实现

### 2.3.4 EVAL命令

### 2.3.5 EVALSHA命令

### 2.3.6 SCRIPT命令

### 2.3.7 脚本管理命令实现

### 2.3.8 脚本复制

## 2.4 慢查询日志

### 2.4.1 慢查询日志

### 2.4.2 慢查询记录的保存

### 2.4.3 慢查询日志的阅览和删除

### 2.4.4 添加日志实现

### 2.4.5 慢查询定位和处理

## 2.5 监视器

### 2.5.1 实现监视器

### 2.5.2 想监视器发送命令信息

### 2.5.3 Redis监控平台

# 第三部分 Redis核心原理



# 第四部分 Redis企业实战



# 第五部分 Redis高可用方案