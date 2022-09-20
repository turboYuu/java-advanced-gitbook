> 第二部分 Redis扩展功能

# 1 发布与订阅

[Redis Pub/Sub](https://redis.io/docs/manual/pubsub/)

Redis 提供了发布订阅功能，可以用于消息的传输

Redis 的发布订阅机制包括三个部分，publisher，subscriber 和 Channel。

![image-20220919120505810](assest/image-20220919120505810.png)

发布者 和 订阅者 都是 Redis 客户端，Channel 则为 Redis 服务端。

发布者将消息发送到某个频道，订阅了这个频道的订阅者就能接收到这条消息。

## 1.1 频道/模式的订阅与退订

**subscribe**：订阅 subscribe channel1 channel2 ...

Redis 客户端1 订阅 频道1 和 频道2

```bash
127.0.0.1:6379> subscribe ch1 ch2
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "ch1"
3) (integer) 1
1) "subscribe"
2) "ch2"
3) (integer) 2
```

**publish**：发布消息 publish channel message

Redis 客户端2 将消息发布在 频道1 和 频道2 上

```bash
127.0.0.1:6379> publish ch1 hello
(integer) 1
127.0.0.1:6379> publish ch2 world
(integer) 1
```

 Redis 客户端1 接收到 频道1 和 频道2 的消息

```bash
1) "message"
2) "ch1"
3) "hello"
1) "message"
2) "ch2"
3) "world"
```



**unsubscribe**：退订 channel

Redis 客户端1 退订频道1

```bash
127.0.0.1:6379> unsubscribe ch1
1) "unsubscribe"
2) "ch1"
3) (integer) 0
```



**psubscribe**：模式匹配 psubscribe + 模式

Redis 客户端1 订阅 所有 以 ch 开头的 频道

```bash
127.0.0.1:6379> psubscribe ch*
Reading messages... (press Ctrl-C to quit)
1) "psubscribe"
2) "ch*"
3) (integer) 1
```

Redis 客户端2 发布信息在 频道5 上

```bash
127.0.0.1:6379> publish ch5 helloworld
(integer) 1
```

Redis 客户端1 收到频道5 的信息

```bash
1) "pmessage"
2) "ch*"
3) "ch5"
4) "helloworld"
```

**punsubscribe** 退订模式

```bash
127.0.0.1:6379> punsubscribe ch*
1) "punsubscribe"
2) "ch*"
3) (integer) 0
```



## 1.2 发布订阅的机制

订阅某个频道或模式

- 客户端（client）

  属性为 pubsub_channels，该属性表明了该客户端订阅的所有频道。

  属性为 pubsub_patterns，该属性表示该客户端订阅的所有模式。

- 服务器端（RedisServer）

  属性为 pubsub_channels，该服务器端中的所有频道以及订阅了这个频道的客户端。

  属性为 pubsub_patterns，该服务器端中的所有模式和订阅了这些模式的客户端。

```c
typedef struct redisClient {
	...
	dict *pubsub_channels;  //该client订阅的channels，以channel为key用dict的方式组织    
    list *pubsub_patterns;  //该client订阅的pattern，以list的方式组织
	...
} redisClient; 

struct redisServer {
	...
	dict *pubsub_channels;  //redis server进程中维护的channel dict，它以channel为key，订阅								channel的client list为value
	list *pubsub_patterns;  //redis server进程中维护的pattern list    
    int notify_keyspace_events;
	...
};
```

当客户端向某个频道发送消息时，Redis 首先在 RedisServer 中的 pubsub_channels 中找出键为该频道的节点，遍历该节点的值，即遍历订阅了该频道的所有客户端，将消息发送给这些客户端。

然后，遍历结构体 redisServer 中的 pubsub_patterns，找出包含该频道的模式的节点，将消息发送给订阅了该模式的客户端。

## 1.3 使用场景：哨兵模式，Redisson框架使用

在 Redis 哨兵模式中，哨兵通过发布 与 订阅 的方式 实现 Redis 主服务器 和 Redis 从服务器进行通信。（后面章节中讲解）。

Redisson 是一个分布式锁框架，在 Redisson 分布式锁释放的时候，是使用发布与订阅的方式通知的。（后面章节中讲解）。

# 2 事务

[Redis-Transactions](https://redis.io/docs/manual/transactions/)

## 2.1 ACID

## 2.2 Redis 事务

- Redis 事务是 通过 multi、exec、discard 和 watch 这四个命令来完成的。
- Redis 的单个命令都是原子性的，所以这里需要确保事务性的对象是命令集合。
- Redis 将命令集合序列化并确保处于同一事务的命令集合连续且不被打断执行。
- Redis 不支持回滚操作。

## 2.3 事务命令

- multi：用于标记事务块的开始，Redis 会将后续的命令逐个放入队列中，然后使用 exec 原子化地 执行这个命令队列。
- exec：执行命令队列
- discard：清除命令队列
- watch：监视 key
- unwatch：清除监视 key

![image-20220919151141781](assest/image-20220919151141781.png)

```bash
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set s1 222
QUEUED
127.0.0.1:6379> hset set1 name zhangfei
QUEUED
127.0.0.1:6379> exec 
1) OK
2) (integer) 1
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set s2 333
QUEUED
127.0.0.1:6379> hset set2 age 23
QUEUED
127.0.0.1:6379> discard
OK
127.0.0.1:6379> exec
(error) ERR EXEC without MULTI
127.0.0.1:6379> watch s1    
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set s1 555
QUEUED
127.0.0.1:6379> exec 	# 此时在没有exec之前，通过另一个命令窗口对监控的s1字段进行修改
(nil)
127.0.0.1:6379> get s1
222
127.0.0.1:6379> unwatch
OK
```

## 2.4 事务机制

### 2.4.1 事务的执行

1. 事务开始

   在 RedisClient 中，有属性 flags，用来表示是否在事务中。flags=REDIS_MULTI

2. 命令入队

   RedisClient 将命令存放在事务队列中 （EXEC, DISCARD, WATCH, MULTI 除外）。

3. 事务队列

   multiCmd * commands 用于存放命令

4. 执行事务

   RedisClient 向服务器端发送 exec 命令，RedisServer 会遍历事务队列，执行队列中的命令，最后将执行的结果一次性返回给客户端。

如果某条命令在入队过程中发生错误，redisClient 将 flags 置为 REDIS_DIRTY_EXEC，EXEC 命令将会失败返回。

![image-20220919154405378](assest/image-20220919154405378.png)

```c
typedef struct redisClient{    
	// flags
   	int flags //状态    
   	// 事务状态
   	multiState mstate;
   	// ..... 
}redisClient;

// 事务状态
typedef struct multiState{    
	// 事务队列,FIFO顺序
   	// 是一个数组,先入队的命令在前,后入队在后    
   	multiCmd *commands;
   	// 已入队命令数    
   	int count; 
}multiState;

// 事务队列
typedef struct multiCmd{    
	// 参数
   	robj **argv;    
   	// 参数数量    
   	int argc;    
   	// 命令指针
   	struct redisCommand *cmd; 
}multiCmd;
```



### 2.4.2 Watch的执行

使用 WATCH 命令监视数据库键。

redisDb 有一个 watched_keys 字典，key 是某个被监视的数据的 key，值是一个链表，记录了所有监视这个数据的客户端。

监视机制的触发：当修改数据后，监视这个数据的客户端的 flags 置为 REDIS_DIRTY_CAS

事务执行：RedisClient 向服务器端发送 exec 命令，服务器判断 RedisClient 的 flags，如果为 REDIS_DIRTY_CAS，则清空事务队列。

![image-20220919155119287](assest/image-20220919155119287.png)

```c
typedef struct redisDb{
   // .....
   // 正在被WATCH命令监视的键    
   dict *watched_keys;
   // ..... 
}redisDb;
```



### 2.4.3 Redis 的弱事务性

- Redis语法错误

  整个事务的命令在队列里都清除

  ```bash
  127.0.0.1:6379> multi
  OK
  127.0.0.1:6379> sets m1 44
  (error) ERR unknown command `sets`, with args beginning with: `m1`, `44`, 
  127.0.0.1:6379> set m2 55
  QUEUED
  127.0.0.1:6379> exec
  (error) EXECABORT Transaction discarded because of previous errors.
  127.0.0.1:6379> get m1
  "22"
  ```

  flags=REDIS_DIRTY_EXEC

- Redis 运行错误

  在队列里正确的命令可以执行（弱事务性）

  弱事务性：

  - 在队列里正确的命令可以执行（非原子操作）
  - 不支持回滚

  ```bash
  127.0.0.1:6379> multi
  OK
  127.0.0.1:6379> set m1 55
  QUEUED
  127.0.0.1:6379> lpush m1 1 2 3 # 不是语法错误
  QUEUED
  127.0.0.1:6379> exec
  1) OK
  2) (error) WRONGTYPE Operation against a key holding the wrong kind of value
  127.0.0.1:6379> get m1
  "55"
  ```

- Redis 不支持事务回滚（为什么）

  - 大多数事务失败是因为**语法错误或者类型错误**，这两种错误，在开发阶段都是可以预见的。
  - Redis 为了**性能方面**就忽略的事务回滚。（回滚需要记录历史版本）。

# 3 Lua脚本

lua 是一种清凉小巧的脚本语言，用标准 **C语言** 编写并以源代码形式开放，其设计目的是为了嵌入应用程序中，从而为应用程序提供灵活的扩展和定制功能。



# 4 慢日志查询

```yaml
# 执行时间超过多少微秒的命令请求会被记录到日志上 0 :全记录 <0 不记录 
slowlog-log-slower-than  10000
#slowlog-max-len 存储慢查询日志条数 
slowlog-max-len 128
```

临时设置

config set

```shell
127.0.0.1:6379> config set slowlog-log-slower-than 0
OK
127.0.0.1:6379> config set slowlog-max-len 5
OK
127.0.0.1:6379> slowlog get 1
1) 1) (integer) 2
   2) (integer) 1636018486
   3) (integer) 3
   4) 1) "slowlog"
      2) "get"
      3) "[1]"
   5) "127.0.0.1:55990"
   6) ""
127.0.0.1:6379> slowlog get 3
1) 1) (integer) 3
   2) (integer) 1636018500
   3) (integer) 16
   4) 1) "slowlog"
      2) "get"
      3) "1"
   5) "127.0.0.1:55990"
   6) ""
2) 1) (integer) 2
   2) (integer) 1636018486
   3) (integer) 3
   4) 1) "slowlog"
      2) "get"
      3) "[1]"
   5) "127.0.0.1:55990"
   6) ""
3) 1) (integer) 1
   2) (integer) 1636018472
   3) (integer) 4
   4) 1) "config"
      2) "set"
      3) "slowlog-max-len"
      4) "5"
   5) "127.0.0.1:55990"
   6) ""
127.0.0.1:6379> 
```



# 5 监视器

Redis客户端1

```shell
127.0.0.1:6379> monitor
OK
1636019353.766249 [0 127.0.0.1:55992] "COMMAND"
1636019375.414681 [0 127.0.0.1:55992] "set" "name" "turbo"
1636019380.011170 [0 127.0.0.1:55992] "set" "name" "echo"
```

Redis客户端2

```shell
127.0.0.1:6379> set name turbo
OK
127.0.0.1:6379> set name echo
OK
```

