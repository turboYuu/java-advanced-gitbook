第三部分 Redis核心原理

# 10 Redis持久化

## 10.1 为什么要持久化

Redis是内存数据库，宕机后数据会消失。Redis重启后快速恢复数据，要提供持久化机制。Redis持久化是为了快速恢复数据，而不是为了存储数据。

Redis有两种持久化方式：RDB和AOF

注意：Redis持久化不保证数据的完整性。

当Redis用作DB时，DB数据要完整，所以一定要有一个完整的数据源（文件、mysql）

在系统启动时，从这个完整的数据源中将数据load到Redis中，数据量较小，不易改变，比如：字典库（xml、Table）

通过info命令可以查看关于持久化的信息：

```shell
[root@localhost bin]# ./redis-cli 
127.0.0.1:6379> info
...

# Persistence
loading:0
rdb_changes_since_last_save:17
rdb_bgsave_in_progress:0
rdb_last_save_time:1635304292
rdb_last_bgsave_status:ok
rdb_last_bgsave_time_sec:-1
rdb_current_bgsave_time_sec:-1
rdb_last_cow_size:0
aof_enabled:1
aof_rewrite_in_progress:0
aof_rewrite_scheduled:0
aof_last_rewrite_time_sec:-1
aof_current_rewrite_time_sec:-1
aof_last_bgrewrite_status:ok
aof_last_write_status:ok
aof_last_cow_size:0
aof_current_size:649
aof_base_size:649
aof_pending_rewrite:0
aof_buffer_length:0
aof_rewrite_buffer_length:0
aof_pending_bio_fsync:0
aof_delayed_fsync:0

...
```



## 10.2 RDB

RDB（Redis DataBase），是Redis默认的存储方式，RDB方式是通过快照（`snapshotting`）完成的。关注这一刻的数据，不关注过程。

### 10.2.1 触发快照的方式

1. 符合自定义配置的快照规则
2. 执行save或者bgsave命令
3. 执行flushall命令
4. 执行主从复制操作（第一次）

### 10.2.2 配置参数定期执行

在redis.conf中配置：save	多少秒内数据变了多少

![image-20211027112306847](assest/image-20211027112306847.png)

```shell
save ""		#不使用RDB	不能主从

save 900 1		#表示15分钟（900秒）内至少1个键被更改则进行快照。
save 300 10		#表示5分钟（300秒）内至少10个键被更改则进行快照。
save 60 10000	#表示1分钟（60秒）内至少10000个键被更改则进行快照。
```

漏斗设计 提供性能

### 10.2.3 命令显示触发

在客户端输入bgsave命令。

```shell
[root@localhost bin]# ./redis-cli -h 127.0.0.1 -p 6379
127.0.0.1:6379> bgsave
Background saving started
```



### 10.2.4 RDB执行流程（原理）

![image-20211027112948081](assest/image-20211027112948081.png)

1. Redis父进程首先判断：当前是否在执行save、或bgsave/bgrewriteaof（aof文件重写命令）的子进程，如果在执行则bgsave命令直接返回。
2. 父进程执行fork（调用OS函数复制主进程）操作创建子进程，这个复制过程中父进程是阻塞的，Redis不能执行来自客户端的任何命令。
3. 父进程fork之后，bgsave命令返回“Background saving started”信息并不再阻塞父进程，并可以响应其他命令。
4. 子进程创建RDB文件，根据父进程内存快照生成临时快照文件，完成后对原有文件进行原子替换。（RDB始终完整）
5. 子进程发送信号给父进程表示完成，父进程更新统计信息。
6. 父进程fork子进程后，继续工作。



### 10.2.5 RDB文件结构

![image-20211027140841637](assest/image-20211027140841637.png)

1. 头部5字节固定为“REDIS”字符串

2. 4字节“RDB”版本号（不是Redis版本号），当前为9，填充后为0009

3. 辅助字段，以key-value的形式

   | 字段名     | 字段值     | 字段名         | 字段值      |
   | ---------- | ---------- | -------------- | ----------- |
   | redis-ver  | 5.0.5      | aof-preamble   | 是否开启aof |
   | redis-bits | 64/32      | repl-stream-db | 主从复制    |
   | ctime      | 当前时间戳 | repl-id        | 主从复制    |
   | used-mem   | 使用内存   | repl-offset    | 主从复制    |

4. 存储数据库号码

5. 字典大小

6. 过期key

7. 主要数据，以key-value的形式存储

8. 结束标志

9. 校验和，就是文件是否损坏，或者是否被修改。

可使用winhex打开dump.rdb文件查看。

![image-20211027142312770](assest/image-20211027142312770.png)

### 10.2.6 RDB的优缺点

优点：

- RDB是二进制文件，占用空间小，便于传输（传给slaver）
- 主进程fork子进程，可以最大化Redis性能，主进程不能太大，Redis的数据量不能太大，复制过程中主进程阻塞

缺点：

- 不保证数据完整性，会丢失最后一次快照之后更改的所有数据。

## 10.3 AOF

AOF（append only file）是Redis的另一种持久化方式。Redis默认情况下是不开启的。

开启AOF持久化后，Redis将所有对数据库进行过的**写入命令（及其参数）**（RESP）记录到AOF文件，以此达到记录数据库的目的，这样当Redis重启后只要按顺序回放这些命名就会恢复到原始状态了。

AOF会记录过程，RDB只记录结果。

### 10.3.1 AOF持久化实现

配置redis.conf

```shell
# 可以通过修改redis.conf配置文件中的appendonly参数开启
appendonly yes  

# AOF文件的保存位置和RDB文件的位置相同，都是通过dir参数设置的。
dir   ./  

# 默认的文件名是appendonly.aof，可以通过appendfilename参数修改
appendfilename   appendonly.aof
```

### 10.3.2 AOF原理

AOF文件中存储的是redis的命令，同步命令到AOF文件的整个过程可以分为三个阶段：

1. 命令传播：Redis将执行完的命令、命令的参数、命令的参数个数等信息发送到AOF程序中。
2. 缓存追加：AOF程序根据接收到的命令数据，将命令转换为网络通讯协议的格式，然后将协议内容追加到服务器的AOF缓存中。
3. 文件写入和保存：AOF缓存中的内容被写入到AOF文件末尾，如果设定的AOF保存条件被满足的话，fsync函数挥着fdatasync函数被调用，将写入的内容真正地保存到磁盘中。

#### 10.3.2.1 命令传播

当一个Redis客户端需要执行命令时，它通过网络连接，将协议文本发送给Redis服务器。服务器在接收到客户端地请求之后，它会根据协议文本地内容，选择适当地命令函数，并将各个参数从字符串文本转换为Redis字符串对象（`StringObject`）。每当命令函数成功执行之后，命令参数都会被传播到AOF程序。

#### 10.3.2.2 缓存追加

当命令被传播到AOF程序之后，程序会很据命令以及命令的参数，将命令从字符串对象转换回原来的协议文本。协议文本生成之后，它会被追加到`redis.h/redisServer`的结构`aof_buf`末尾。

`redisServer`结构维持着Redis服务器的状态，`aof_buf`域则保存着所有等待写入到AOF文件的协议文本（RESP）。

#### 10.3.2.3 文件写入和保存

每当服务器常规任务函数被执行、或者时间处理器被执行时，aof.c/flushAppendOnlyFile函数都会被调用，这个函数执行以下两个工作：

- WRITE：根据条件，将aof_buf中的缓存写入到AOF文件。
- SAVE：根据条件，调用fsync或fdatasync函数，将AOF文件保存到磁盘中。

#### 10.3.2.4 AOF保存模式

Redis目前支持三种AOF保存模式，它们分别是：

AOF_FSYNC_NO：不保存

AOF_FSYNC_EVERYSEC：每秒钟保存一次（默认）

AOF_FSYNC_ALWAYS：每执行一个命令保存一次（不推荐）

以下三个小结将分别讨论这三种保护模式。

##### 10.3.2.4.1 不保存

在这种模式下，每次调用flushAppendOnlyFile函数，WRITE都会被执行，但SAVE会被略过。

在这种模式下，SAVE只会在以下任意一种情况下被执行：

- Redis被关闭

- AOF功能被关闭

- 系统的写缓存被刷新（可能是缓存已经被写满，或者定期保存操作被执行）

这三种情况下的SAVE操作都会引起Redis主进程阻塞。

##### 10.3.2.4.2 每秒钟保存一次（默认）

在这种模式下，SAVE原则上每隔一秒钟就会执行一次，因为SAVE操作是由后台子线程（fork）调用的，所以他不会引起服务器主进程阻塞。

##### 10.3.2.4.3 每执行一个命令保存一次（不推荐）

在这种模式下，每次执行完一个命令之后，WRITE和SAVE都会被执行。另外，因为SAVE是由Redis主进程执行的，所以在SAVE执行期间，主进程会被阻塞，不能接收命令请求。

AOF保存模式对性能和安全性的影响：对于三种AOF保存模式，它们对服务器的阻塞情况如下：

![image-20211029111936342](assest/image-20211029111936342.png)

#### 10.3.2.5 AOF重写、触发方式、混合持久化

AOF记录数据的变化过程，越来越大，需要重写“瘦身”

Redis可以在AOF体积变得过大时，自动地在后台（Fork子进程）对AOF进行重写。重写后的新AOF文件包含了恢复当前数据集所需的最小命令集合。所谓的“重写”其实就是一个有歧义的词语，实际上，AOF重写并不需要对原有的AOF文件进行任何写入和读取，它针对的是数据库中键的当前值。

eg:

```
set s1 11
set s1 22
set s1 33
```

没有优化

```
set s1 11
set s1 22
set s1 33
```

优化后

```
set s1 33
```

Redis不希望AOF重写造成服务器无法处理请求，所以Redis决定将AOF重写程序放到（后台）子进程里执行，这样处理的最大好处：

1. 子进程进行AOF重写期间，主进程可以继续处理命令请求。
2. 子进程带有主进程的数据副本，使用子进程而不是线程，可以避免锁的情况下，保证数据的安全性。

不过，使用子进程也有一个问题需要解决：因为子进程在进程AOF重写期间，主进程还需要继续处理命令，而新的命令可能对现有的数据进行修改，这会让当前数据库的数据和重写后的AOF文件中的数据不一致。





# 11 底层数据结构

# 12 缓存过期和淘汰策略

Redis的性能高，官方数据：读（110000次/s），写（81000次/s）。长期使用，key会不断增加，Redis作为缓存使用，物理内存会满。内存与硬盘交换（swap）虚拟内存，频繁IO性能急剧下降。

## 12.1 maxmemory

# 13 通讯协议及事件处理机制

































