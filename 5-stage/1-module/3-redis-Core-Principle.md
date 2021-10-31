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

为了解决这个问题Redis增加了一个AOF重写缓存，这个缓存在fork出子进程之后开始启用，Redis主进程在接到新的命令之后，除了会将这个写命令的内容追加到现有的AOF文件之外，还会追加到这个缓存中。

![image-20211029124817252](assest/image-20211029124817252.png)

> 重写过程分析（整个重写操作是绝对安全的）

Redis在创建新AOF文件的过程中，会继续将命令追加到现有的AOF文件里面，即使重写过程中发生停机，现有的AOF文件也不会丢失。而一旦新AOF文件创建完毕，Redis就会从旧AOF文件切换到新AOF文件，并开始对新AOF文件进行追加操作。

当子进程在执行AOF重写时，主进程需要执行以下三个工作：

- 处理命令请求
- 将写命令追加到现有的AOF文件中。
- 将写命令追加到AOF重写缓存中

这样一来可以保证：

现有的AOF功能继续执行，即使在AOF重写期间发生停机，也不会有任何数据丢失。所有对数据库进行修改的命令都会被记录到AOF重写缓存中。当子进程完成AOF重写之后，它会向父进程发送一个完成信号，父进程在接收到完成信号之后，会调用一个信号处理函数，并完成以下工作：

- 将AOF重写缓存中的内容全部写入到新AOF文件中；

- 对新的AOF文件进行改名，覆盖原有的AOF文件。

Redis数据库里的 + AOF重写过程中的命令 ---> 新的AOF文件 ---> 覆盖老的AOF文件

当步骤一执行完毕后，现有AOF文件、新AOF文件和数据库三者的状态旧完全一致了；

当步骤二执行完毕后，程序就完成了新旧两个AOF文件的交替。

这个信号处理函数执行完毕后，主进程就可以继续向往常一样接受命令请求了。在整个AOF后台重写过程中，只有**最后的写入缓存**和**改名操作**会造成主进程阻塞，其他时候，AOF后台重写都不会对主进程造成阻塞，这将AOF重写行性能造成的影响降到了最低。

以上就是AOF后台重写，也就是BGREWRITEAOF命令（AOF重写的工作原理）。

![image-20211029135232087](assest/image-20211029135232087.png)

> 触发方式

1. 配置触发

   在redis.conf中配置

   ```properties
   # 表示当前aof文件大小超过上一次aof文件大小的百分之多少的时候会进行重写。如果之前没有重写过，以启动时aof文件大小为准
   auto-aof-rewrite-percentage 100
   
   # 限制允许重写最小aof文件大小，也就是文件大小小于64mb的时候，不需要进行优化
   auto-aof-rewrite-min-size 64mb
   ```

   

2. 执行bgrewriteaof命令

   ```shell
   127.0.0.1:6379> bgrewriteaof
   Background append only file rewriting started
   127.0.0.1:6379> 
   ```

> 混合持久化

RDB和AOF各有优缺点，Redis 4.0开始支持rdb和aof的混合持久化。如果把混合持久化打开，aof rewrite的时候就直接把rdb的内容写到aof文件开头。

RDB的头 + AOF的身体 ---> appendonly.aof

开启混合持久化

```properties
aof-use-rdb-preamble yes
```

![image-20211029142427632](assest/image-20211029142427632.png)

可以看到该AOF文件是rdb文件开头和aof格式的内容，在加载时，首先会识别AOF文件是否以REDIS字符串开头，如果是就是按RDB格式加载，加载完RDB后继续按AOF格式加载剩余部分。

#### 10.3.2.6 AOF 文件在载入与数据还原

因为AOF文件里面包含了重建数据库状态所需的所有写命令，所以服务器只需要读入并重新执行一边AOF文件里面保存的写命令，就可以还原服务器关闭之前的数据库状态

Redis读取AOF文件并还原数据库状态的详细步骤如下：

1. 创建一个不带网络连接的伪客户端（fake client）：因为Redis的命令只能在客户端上下文中执行，而载入AOF文件时所使用的命令直接来源于AOF文件，而不是网络连接。所以服务器使用一个没有网络连接的伪客户端来执行AOF文件保存的写命令，伪客户端执行命令的效果和带网络连接的客户端执行命令的效果完全一样。
2. 从AOF文件中分析并读取出一条写命令
3. 使用伪客户端执行被读出的写命令
4. 一直执行步骤2和步骤3，直到AOF文件中的所有写命令都被执行完毕为止

当完成以上步骤后，AOF文件所保存的数据库状态就会被完整的还原出来，整个过程如下：

![image-20211029144945368](assest/image-20211029144945368.png)

## 10.4 RDB与AOF对比

1. RDB存某个时刻的数据快照，采用二进制压缩存储；AOF存储操作命令，采用文本存储（混合）
2. RDB性能高、AOF性能较低
3. RDB在配置触发状态会丢失最后一次快照以后更改的所有数据；AOF设置为每秒保存一次，则最多丢2秒的数据
4. Redis以主服务器模式运行，RDB不会保存过期键值对数据；Redis以从服务器模式运行，RDB会保存过期键值对，当主服务器向从服务器同步时，在清空过期键值对。

AOF写入文件时，对过期的key会追加一条del命令，当执行AOF重写时，会忽略过期key和del命令。

![image-20211029153556946](assest/image-20211029153556946.png)

## 10.5 应用场景

内存数据库	rdb + aof	数据不易丢失

有原始数据源：每次启动时，都从原始数据源中初始化，则不用开启持久化（数据量较小）

缓存服务器	rdb	一般开启，性能高（数量大的时候，fork时会降低性能）



在数据还原时

有rdb + aof 则还原aof，因为RDB会造成文件的丢失，AOF相对数据要完整。

只有rdb，则还原rdb





# 11 底层数据结构

Redis作为Key - Value存储系统，数据结构如下：

![image-20211029155556905](assest/image-20211029155556905.png)

Redis没有表的概念，Redis实例所对应的db以编号区分，db本身就是key的命名空间。

比如：user:1000作为key值，表示在user这个命名空间下id为1000的元素，类似于user表的id=1000的行。

## 11.1 RedisDB结构

Redis中存在“数据库”的概念，该结构由redis.h中的redisDb定义。

当redis服务器初始化时，会预先分配16个数据库

所有数据库保存到结构redisServer的一个成员 redisServer.db数组中

redisClient中存在一个名为db的指针指向当前使用的数据库

RedisDB结构体源码：

```c
typedef struct redisDb {	
   int id;			//id是数据库序号，为0-15（默认Redis有16个数据库）
   long  avg_ttl;  	//存储的数据库对象的平均ttl（time to live），用于统计
   dict *dict;     	//存储数据库所有的key-value
   dict *expires;  	//存储key的过期时间
   dict *blocking_keys;	//blpop 存储阻塞key和客户端对象
   dict *ready_keys;	//阻塞后push 响应阻塞客户端  存储阻塞后push的key和客户端对象
   dict *watched_keys;	//存储watch监控的的key和客户端对象
} redisDb;
```

> **id**

id是数据库号，为0-15（默认Redis有16个数据库）

> **dict**

存储数据库所有的key-value

> **expires**

存储key的过期时间



## 11.2 RedisObject结构

Value是一个对象

包含字符串对象，列表对象，哈希对象，集合对象和有序集合对象

### 11.2.1 结构信息概览

```c
typedef struct redisObject {
	unsigned type:4;	//类型     对象类型 
    unsigned encoding:4;//编码
	void *ptr;		//指向底层实现数据结构的指针 
    //...
	int refcount;	//引用计数 
    //...
	unsigned lru:LRU_BITS; //LRU_BITS为24bit 记录最后一次被命令程序访问的时间 
    //...
}robj;
```

#### 11.2.1.1 4位type

type字段表示对象的类型，占4位；

REDIS_STRING(字符串)、REDIS_LIST(列表)、REDIS_HASH(哈希)、REDIS_SET(集合)、REDIS_ZSET(有序集合)

当执行type命令时，便是通过读取RedisObject的type字段获取对象的类型

```
127.0.0.1:6379> type age
string
```

#### 11.2.1.2 4位encoding

encoding表示对象的内部编码，占4位，

每个对象有不同的实现编码

Redis可以根据不同的使用场景来为对象设置不同的编码，大大提高了Redis的灵活性和效率。

通过object encoding命令，可以查看对象采用的编码方式

```
127.0.0.1:6379> object encoding a1
"int"
```

#### 11.2.1.3 24 位LRU

LRU（Least Recently Used）记录的是最后一次被命令程序访问的时间，（4.0版本占24位，2.6版本占22位）。

高16位存储一个分钟级别的时间戳，低8位存储访问计数（lfu：最近访问次数）

lru ---> 高16位：最后被访问的时间

lfu ---> 低8位：最近访问次数



#### 11.2.1.4 refcount

refcount记录的是该对象被引用的次数，类型为整形。

refcount的作用，主要在于对象的引用计数和内存回收。

当对象的refcount > 1时，称为共享对象

Redis为了节省内存，当有一些对象重复出现时，新的程序不会创建新的对象，而是仍然使用原来的对象。

#### 11.2.1.5 ptr

ptr指针指向具体的数据，比如：set hello world，ptr指向包含字符串world的SDS。

### 11.2.2 7种type

#### 11.2.2.1 字符串对象

C语言：字符数组	以"\0"结尾

Redis使用了SDS（Simple Dynamic String）。用于存储字符串和整型数据。

![image-20211029164750716](assest/image-20211029164750716.png)

```c
struct sdshdr{
   	//记录buf数组中已使用字节的数量    
    int len;
   	//记录     buf 数组中未使用字节的数量    
    int free;
   	//字符数组，用于保存字符串    
    char buf[];
}
```

buf[]的长度 = len + free + 1

SDS的优势：

1. SDS在C字符串的基础上加入了free和len字段，获取字符串长度：SDS是O(1)，C字符串是O(n)。
2. SDS由于记录了长度，在可能造成缓冲区溢出时会自动重新分配内存，杜绝了缓冲区溢出
3. 可以存取二进制数据，以字符串长度len作为结束标识



使用场景：

SDS的主要应用在：存储字符串和整型数据、存储key、AOF缓冲和用户输入缓冲。



#### 11.2.2.2 跳跃表（重点）

跳跃表是有序集合（sorted-set）的底层实现，效率高，实现简单。

跳跃表的基本思想：

**将有序链表中的部分节点分层，每一层都是一个有序链表**

> 查找

在查找时优先从最高层开始向后查找，当达到某个节点时，如果next节点值大于要查找的值或next指针指向null，则从当前节点下降一层继续向后查找。

举例：

![image-20211029180026323](assest/image-20211029180026323.png)

查找元素9，按道理需要从头节点开始遍历，一共遍历8个节点才能找到元素9。

**第一次分层**：

遍历5次找到元素9（红色线为查找路径）

![image-20211029180534205](assest/image-20211029180534205.png)

**第二次分层**：

遍历四次找到元素9

![image-20211029180628388](assest/image-20211029180628388.png)

**第三次分层**：

遍历4次找到元素9

![image-20211029180809462](assest/image-20211029180809462.png)

这种数据结构，就是跳跃表，它具有二分查找的功能。

插入与删除

上面例子中，9个节点，一共4层，是理想的跳跃表。

通过抛硬币（概率1/2）的方式来决定新插入节点跨越的层数：

> **删除**

找到指定元素并删除每层的该元素即可

跳跃表特点：

- 每层都是一个有序链表

- 查找次数近似于层数（1/2）

- 底层包含所有原色
- 空间复杂度O(n)扩充了一倍

> **Redis跳跃表的实现**

```c
//跳跃表节点
typedef struct zskiplistNode {
	sds ele; /* 存储字符串类型数据     redis3.0版本中使用robj类型表示，
				但是在redis4.0.1中直接使用sds类型表示     */
	double score;//存储排序的分值
    struct zskiplistNode *backward;//后退指针，指向当前节点最底层的前一个节点     
    /*
		层，柔性数组，随机生成1-64的值     
	*/
    struct zskiplistLevel {
    	struct zskiplistNode *forward; //指向本层下一个节点          
        unsigned int span;//本层下个节点到本节点的元素个数
     } level[]; 
} zskiplistNode;

//链表
typedef struct zskiplist{     
	//表头节点和表尾节点
    structz skiplistNode *header, *tail;     
    //表中节点的数量
    unsigned long length;     
    //表中层数最大的节点的层数     
    int level;
}zskiplist;
```

完整的跳跃表结构体：

![image-20211029181553854](assest/image-20211029181553854.png)

**跳跃表的优势**：

1. 可以快速查找到需要的节点O(logn)
2. 可以在O(1)的时间复杂度下，快速获得跳跃表的头节点、尾节点、长度和高度。

应用场景：有序集合的实现



#### 11.2.2.3 字典（重点 + 难点）

字典dict又称散列表（hash），是用来存储键值对的一种数据结构。

Redis整个数据库使用字典来存储的。（K-V结构）

对Redis进行CURD操作，其实就是对字典中的数据进行CURD操作。

> 数组

数组：用来存储数据的容器，采用头指针 + 偏移量的方式能够以O(1)的时间复杂度定位到数据所在的内存地址。

Redis快速海量存储

> Hash函数

Hash（散列），作用是把任意长度的输入通过散列算法转换成固定类型，固定长度的散列值。

hash函数可以把Redis里的key：包括字符串、整数、浮点数 统一转换成整数。

```
Redis-cli : time 33(hash算法)
RedisServer: siphash(hash算法)
```

数组下标 = hash(key)%数组容量(hash值%数组容量得到的余数)

> Hash冲突

不同的key经过计算后出现数组下标一致，称为Hash冲突。

采用单链表在相同的下标为止出存储原始key和value

当根据key找Value时，找到数组下标，遍历单链表可以找出key相同的value

![image-20211029190550556](assest/image-20211029190550556.png)

> Redis字典的实现

Redis字典实现包括：字典（dict）、Hash表（dictht）、Hash表节点（dicEntry）。

![image-20211029190713888](assest/image-20211029190713888.png)

> Hash表

```c
typedef struct dictht {
	dictEntry **table;             // 哈希表数组
	unsigned long size;            // 哈希表数组的大小
	unsigned long sizemask;        // 用于映射位置的掩码，值永远等于(size-1)
	unsigned long used;            // 哈希表已有节点的数量,包含next单链表数据 
} dictht;
```

1. hash表的数组初始容量为4，醉着k-v存储量的增加需要对hash表数组进行扩容，新扩容量为当前量的一倍，即 4、8、16、32
2. 索引值 = Hash值&掩码值（Hash值与Hash表容量取余）

> Hash表节点

```c
typedef struct dictEntry {
	void *key;                  // 键
	union {                     // 值v的类型可以是以下4种类型        
		void *val;
       	uint64_t u64;        
        int64_t s64;        
        double d;
   } v;
   struct dictEntry *next;     // 指向下一个哈希表节点，形成单向链表     解决hash冲突 
} dictEntry;
```

key字段存储的是键值对中的键

v字段是个联合体，存储的是键值对中的值。

next指向下一个哈希表节点，用于解决hash冲突

![image-20211029192041586](assest/image-20211029192041586.png)

> dict字典

```c
typedef struct dict {
    dictType *type;		// 该字典对应的特定操作函数    
    void *privdata;     // 上述类型函数对应的可选参数
    dictht ht[2];       /* 两张哈希表，存储键值对数据，ht[0]为原生 哈希表，
							ht[1]为rehash 哈希表     */
    long rehashidx;		/*rehash标识     当等于-1时表示没有在 rehash，
						否则表示正在进行rehash操作，存储的值表示hash表
                        ht[0]的rehash进行到哪个索引值			
						(数组下标)*/
    int iterators;		// 当前运行的迭代器数量 
} dict;
```

type字段，指向dicType结构体，里面包括了对字典操作的函数指针

```c
typedef struct dictType {    
    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);                                          
    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);                          
    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);                                        
    // 比较键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);                  
    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);                                        
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);                          
} dictType;
```

Redis字典除了主数据库的K-V数据存储以外，还可以用于：散列表对象、哨兵模式中的主从节点管理等在不同的应用中，字典的形态都可能不同，dicType是为了实现各种形态的字典而抽象出来的操作函数（多态）。

完整的Redis字典数据结构：

![image-20211030111014139](assest/image-20211030111014139.png)

> 字典扩容

字典达到存储上限（阈值0.75），需要rehash（扩容）

扩容流程：

![image-20211030111629925](assest/image-20211030111629925.png)

说明：

1. 初次申请默认容量为4个dicEntry，非初次申请为当前hash表容量的一倍。
2. rehashidx=0表示要进行rehash操作。
3. 新增加的数据在新的hash表h[1]
4. 修改、删除、查询在老hash表h[0]、新hash表h[1]中（rehash中）
5. 将老的hash表h[0]的数据重新计算索引值后全部迁移到新的hash表h[1]中，这个过程称为rehash。



> 渐进式rehash

当数据量巨大时rehash的过程是非常缓慢的，所以需要进行优化。

服务器忙，则只对一个节点进行rehash

服务器闲，可批量rehash（100节点）

应用场景：

1. 主数据库的k-v数据存储
2. 散列表对象（hash）
3. 哨兵模式中的主从节点管理



#### 11.2.2.4 压缩列表

压缩列表（ziplist）是由一系列特殊编码的连续内存块组成的顺序型数据结构

节省内存

是一个字节数组，可以包含多个节点（entry）。每个节点可以保存一个字节数组或一个整数。

压缩列表的数据结构如下

![image-20211030112545264](assest/image-20211030112545264.png)

- zlbytes：压缩列表的字节长度

- zltail：压缩列表尾元素相对于压缩列表起始地址的偏移量

- zllen：压缩列表的元素个数

- entry1..entryX：压缩列表的各个节点

- zlend：压缩列表的结尾，占一个字节，恒为0xFF（255）



entryX元素的编码结构：

![image-20211030132834641](assest/image-20211030132834641.png)

- previous_entry_length：前一个元素的字节长度
- encoding：表示当前元素的编码
- content：数据内容



压缩列表（ziplist）结构体如下：

```c
struct ziplist<T>{
    unsigned int zlbytes;		// ziplist的长度字节数，包含头部、所有entry和zipend。
    unsigned int zloffset;		// 从ziplist的头指针到指向最后一个entry的偏移量，用于快速反 向查询
    unsigned short int zllength; // entry元素个数    
    T[] entry;              	// 元素值
    unsigned char zlend;   		// ziplist结束符，值固定为0xFF 
}
typedef struct zlentry {
    unsigned int prevrawlensize;  //previous_entry_length字段的长度
    unsigned int prevrawlen;      //previous_entry_length字段存储的内容
    unsigned int lensize;         //encoding字段的长度                            
    unsigned int len;             //数据内容长度
    
    //当前元素的首部长度，即previous_entry_length字段长度与encoding字段长度之和。
    unsigned int headersize;
    
    unsigned char encoding;     //数据类型

    unsigned char *p;           //当前元素首地址 
} zlentry;
```

应用场景：

sorted-set和hash元素个数少且是小整数或短字符（直接使用）

list用快速链表（quicklist）数据结构存储，而快速链表是双向列表与压缩列表的组合。（间接使用）



#### 11.2.2.5 整数集合

整数集合（intset）是一个有序的（整数升序）、存储整数的连续存储结构。

当Redis集合类型的元素都是整数并且都处在64位有符号整数范围内（2^64），使用该结构体存储。

```shell
127.0.0.1:6379> sadd set:001 1 3 5 7 9
(integer) 5
127.0.0.1:6379> object encoding set:001
"intset"
127.0.0.1:6379> sadd set:004 1 10000000000000000000000000000 9999999999
(integer) 3
127.0.0.1:6379> object encoding set:004
"hashtable"
127.0.0.1:6379> type set:001
set
127.0.0.1:6379> type set:004
set
```

intset的结构图如下：

![image-20211030135643211](assest/image-20211030135643211.png)

```c
typedef struct intset{     
    //编码方式
    uint32_t encoding;     
    //集合包含的元素数量     
    uint32_t length;     
    //保存元素的数组
    int8_t contents[]; 
}intset;
```

应用场景：

可以保存类型为 int16_t、int32_t或者int64_t的整数值，并且保证集合中不会出现重复元素。

#### 11.2.2.6 快速列表（重要）

快速列表（quicklist）是Redis底层最重要的数据结构。是列表的底层实现。【在Redis3.2之前，Redis采用双向链表（adlist）和压缩列表（ziplist）实现】。在Redis3.2以后结合adlist和ziplist的优势Redis设计出了quicklist。

```shell
127.0.0.1:6379> lpush list:001 1 2 3 5 4
(integer) 5
127.0.0.1:6379> object encoding list:001
"quicklist"
127.0.0.1:6379> type list:001
list
```

> 双向链表（adlist）

![image-20211030141138289](assest/image-20211030141138289.png)

双向链表优势：

1. 双向：链表具有前置节点和后置节点的引用，获取这两个节点的时间复杂度都为O(1)。

   *普通链表（单链表）：节点类保留下一节点的引用。链表类只保留头节点的引用，只能从头节点插入删除*

2. 无环：表头节点的prev指针和表尾节点的next指针都指向NULL，对链表的访问都是以NULL结束。

   *环状：头的前一个节点指向尾节点*

3. 带链表长度计数器：通过len属性获取链表长度的时间复杂度为O(1)。

4. 多态：链表节点使用void*指针来保存节点值，可以保存各种不同类型的值。



> 快速列表

quicklist是一个双向链表，链表中的每个节点是一个ziplist结构。quicklist中的每个节点ziplist都能够存储多个数据元素。

![image-20211030142434095](assest/image-20211030142434095.png)

quicklist的结构定义如下：

```c
typedef struct quicklist {
    quicklistNode *head;        // 指向quicklist的头部
    quicklistNode *tail;        // 指向quicklist的尾部
    unsigned long count;        // 列表中所有数据项的个数总和
    unsigned int len;           // quicklist节点的个数，即ziplist的个数
    int fill : 16;              // ziplist大小限定，由list-max-ziplist-size给定 (Redis设定)
    unsigned int compress : 16; // 节点压缩深度设置，由list-compress-depth给定 (Redis设定)
} quicklist;
```

quicklistNode的结构定义如下：

```c
typedef struct quicklistNode {
    struct quicklistNode *prev;  // 指向上一个ziplist节点    
    struct quicklistNode *next;  // 指向下一个ziplist节点
    unsigned char *zl;           // 数据指针，如果没有被压缩，就指向ziplist结构，反之指向quicklistLZF结构
    
    unsigned int sz;             // 表示指向ziplist结构的总长度(内存占用长度)
    unsigned int count : 16;     // 表示ziplist中的数据项个数
    unsigned int encoding : 2;   // 编码方式，1--ziplist，2--quicklistLZF
    unsigned int container : 2;  // 预留字段，存放数据的方式，1--NONE，2--ziplist
    
    // 解压标记，当查看一个被压缩的数据时，需要暂时解压，标 记此参数为1，之后再重新进行压缩
    unsigned int recompress : 1; 
    unsigned int attempted_compress : 1; // 测试相关    
    unsigned int extra : 10; // 扩展字段，暂时没用 
} quicklistNode;
```

> 数据压缩

quicklist每个节点的时机数据存储结构为ziplist，这种结构的优势在于节省存储空间。为了进一步降低ziplist的存储空间，还可以对ziplist进行压缩。Redis采用的压缩算法是LZF。其基本思想是：数据与前面重复的记录  重复位置及长度，不重复的记录原始数据。

压缩后的数据可以分成多个片段，每个片段有两个部分：解释字段和数据字段。quicklistLZF的结构体如下：

```c
typedef struct quicklistLZF {
    unsigned int sz; // LZF压缩后占用的字节数   
    char compressed[]; // 柔性数组，指向数据部分 
} quicklistLZF;
```

应用场景：

列表（list）的底层实现、发布与订阅、慢查询、监视器等功能。

#### 11.2.2.7 流对象

stream主要由：消息、生产者、消费者和消费组 构成

![image-20211030144429235](assest/image-20211030144429235.png)

Redis stream的底层主要使用了listpack（紧凑列表）和Rax（基数树）。

> listpack

listpack表示一个字符串列表的序列化，listpack可用于存储字符串或整数。用于存储stream的消息内容。

结构如下图：

![image-20211030144701476](assest/image-20211030144701476.png)

> Rax树

Rax是一个有序字典树（基数树Radix Tree），按照key的字典序排列，支持快速地定位，插入和删除操作。

![image-20211030144833590](assest/image-20211030144833590.png)

Rax被用在Redis Stream结构里面用于存储消息队列，在stream里面消息ID的前缀是时间戳 + 序号，这样的消息可以理解为时间序列消息。使用Rax结构 进行存储就可以快速地根据消息ID定位到具体的消息，然后继续遍历指定消息之后的所有消息。

![image-20211030145117693](assest/image-20211030145117693.png)

应用场景：

stream的底层实现。

### 11.2.3 10种encoding

encoding表示对象的内部编码，占4位。

Redis通过encoding属性为对象设置不同的编码。

对于`少的`和`小的`数据，Redis采用小的和压缩的存储方式，体现Redis的灵活性

大大提高了Redis的存储量和执行效率

**比如Set对象**：

intset：元素是64位以内的整数

hashtable：元素是64位以外的整数

如下所示：

```shell
127.0.0.1:6379> sadd set:001 1 3 5 7 9
(integer) 5
127.0.0.1:6379> object encoding set:001
"intset"
127.0.0.1:6379> sadd set:004 1 10000000000000000000000000000 9999999999
(integer) 3
127.0.0.1:6379> object encoding set:004
"hashtable"
```



#### 11.2.3.1 String

int、raw、embstr

> int

REDIS_ENCODING_INT（int类型的整数）

```shell
127.0.0.1:6379> set n1 123
OK
127.0.0.1:6379> object encoding n1
"int"
```

> embstr

REDIS_ENCODING_EMBSTR（编码的简单动态字符串）

小字符串 长度小于44个字节

```shell
127.0.0.1:6379> set name:001 zhangfei
OK
127.0.0.1:6379> object encoding name:001
"embstr"
```

> raw

REDIS_ENCODING_RAW（简单动态字符串）

大字符串 长度大于44个字节

```shell
127.0.0.1:6379> set address:001 asdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdasdaadasdasadasdasaadaad
OK
127.0.0.1:6379> object encoding address:001
"raw"
```



#### 11.2.3.2 list

列表的编码是quicklist。

REDIS_ENCODING_QUICKLIST（快速列表）

```shell
127.0.0.1:6379> lpush list:001 1 2 3 5 4
(integer) 5
127.0.0.1:6379> object encoding list:001
"quicklist"
```



#### 11.2.3.3 hash

散列的编码是**字典**和**压缩列表**

> dict

REDIS_ENCODING_HT（字典）

当散列表元素的个数比较多或元素不是 小整数或短字符串时。

```shell
127.0.0.1:6379> hmset user:003 username11111111111111111111111111111111111111111111111111111111111111111111111111111111 zhangfei password 111 num 230000000000000000000000000000000000000000000000000
OK
127.0.0.1:6379> object encoding user:003
"hashtable"
127.0.0.1:6379> type user:003
hash
```

> ziplist

REDIS_ENCODING_ZIPLIST（压缩列表）

当散列表元素的个数比较少，且元素都是小整数或短字符串时。

```shell
127.0.0.1:6379> hmset user:001 username password 11 age sex m 
OK
127.0.0.1:6379> object encoding user:001
"ziplist"
127.0.0.1:6379> type user:001
hash
```



#### 11.2.3.4 set

集合的编码是整型集合和字典

> intset

REDIS_ENCODING_INTSET（整数集合）

当Redis集合类型的元素都是整数并且都处在64位有符号整数范围内（< 18446744073709551616）

```shell
127.0.0.1:6379> sadd set:001 1 3 5 7 9
(integer) 5
127.0.0.1:6379> object encoding set:001
"intset"
127.0.0.1:6379> type set:001
set
```

> dict

REDIS_ENCODING_HT（字典）

当Redis集合类型的元素是非整数或都处于64位有符号整数范围外（> 18446744073709551616）

```shell
127.0.0.1:6379> sadd set:004 1 10000000000000000000000000000 9999999999
(integer) 3
127.0.0.1:6379> object encoding set:004
"hashtable"
127.0.0.1:6379> type set:004
set
```



#### 11.2.3.5 zset

有序集合的编码是**压缩列表**和**跳跃表+字典**

> ziplist

REDIS_ENCODING_ZIPLIST（压缩列表）

当元素的个数比较少，且元素都是小整数或短字符串时。

```shell
127.0.0.1:6379> zadd hit:1 100 item1 20 item2 45 itme3
(integer) 3
127.0.0.1:6379> object encoding hit:1
"ziplist"
```

> skiplist + dict

REDIS_ENCODING_SKIPLIST（跳跃表 + 字典）

当元素的个数比较多或元素不是小整数或短字符串时。

```shell
127.0.0.1:6379> zadd hit:2 100 item1111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111111 20 item2 45 item3
(integer) 3
127.0.0.1:6379> object encoding hit:2
"skiplist"
```



# 12 缓存过期和淘汰策略

Redis的性能高，官方数据：读（110000次/s），写（81000次/s）。

长期使用，key会不断增加，Redis作为缓存使用，物理内存会满。内存与硬盘交换（swap）虚拟内存，频繁IO性能急剧下降。

## 12.1 maxmemory

> 不设置的场景

Redis的key是固定的，不会增加

Redis作为DB使用，保证数据的完整性，不能淘汰，可以做集群，横向扩展

缓存淘汰策略：禁止驱逐（默认）

> 设置的场景

```
Redis是作为缓存使用，不断增加key
maxmemory：默认为0 不限制
问题：达到物理内存后性能急剧下降，甚至崩溃
内存与硬盘交换（swap）虚拟内存，频繁IO性能急剧下降
```

设置多少？与业务有关

1个Redis实例，保证系统运行1G，剩下的都可以设置Redis。物理内存的3/4

如果需要slaver：还需要留出一定的内存

在redis.conf中

```basic
maxmemory	1024mb
```

命令：获得maxmemory数

```basic
127.0.0.1:6379> config get maxmemory
1) "maxmemory"
2) "0"
```

设置maxmemory后，当趋近maxmemory时，通过缓存淘汰策略，从内存中删除对象



不设置maxmemory	无最大内存限制	maxmemory-policy noeviction（禁止驱逐）不淘汰

设置maxmemory	maxmemory-policy  要配置



## 12.2 expire数据结构

在Redis中可以使用expire命令设置一个键的存活时间（ttl：time to live），过了这段时间，这键就会自动被删除。

### 12.2.1 expire的使用

expire命令的使用方法如下：

expire key ttl(单位秒)

```shell
127.0.0.1:6379> expire name 2	#2秒失效
(integer) 0
127.0.0.1:6379> get name
(nil)
127.0.0.1:6379> set name turbo
OK
127.0.0.1:6379> ttl name	#永久有效
(integer) -1
127.0.0.1:6379> expire name 30	#30秒失效
(integer) 1
127.0.0.1:6379> ttl name	#还有26秒
(integer) 26
127.0.0.1:6379> ttl name	#失效
(integer) -2
```



### 12.2.2 expire原理

```c
typedef struct redisDb {
    dict *dict;  -- key Value
    dict *expires; -- key ttl
    dict *blocking_keys;
    dict *ready_keys;
    dict *watched_keys;    
    int id;
} redisDb;
```

上面的代码是Redis中关于数据库的结构体定义，这个结构体定义中出来id以外都是指向字典的指针，其中我们只看dict和expires。

dict用来维护一个Redis数据库中包含的所有key-Value键值对，expires则用于维护一个Redis数据库中设置了失效时间的键（即key与实现时间的映射）。

当我们使用expire命令设置一个key的失效时间时，Redis首先到dict这个字典表中查找要设置的key是否存在，如果存在就将这个key和失效时间添加到expire这个字典表。

当我们使用setex命令向系统插入数据时，Redis首先将Key和Value添加到dict这个字典表中，然后将Key和失效时间添加到expires这个字典表中。

简单地总结来说就是，设置了失效时间的key和具体的失效时间全部都维护在expires这个字典表中。



## 12.3 删除策略

Redis的数据删除有定时删除、惰性删除和主动删除三种方式。

Redis目前采用惰性删除 + 主动删除的方式。

### 12.3.1 定时删除

在设置键的过期时间的同时，创建一个定时器，让定时器在键的过期时间来临时，立即执行对键的删除操作。

需要创建定时器，而且消耗CPU，**一般不推荐使用**。

### 12.3.2 惰性删除

在key被访问时如果发现它已经失效，那么就删除它。

调用expireIfNeeded函数，该函数的意义是：读取数据之前先检查一下它有没有失效，如果失效了就删除它。

```c
int expireIfNeeded(redisDb *db, robj *key) {    
    //获取主键的失效时间       get当前时间-创建时间>ttl    
    long long when = getExpire(db,key);
    //假如失效时间为负数，说明该主键未设置失效时间（失效时间默认为-1），直接返回0    
    if (when < 0) return 0;
    //假如Redis服务器正在从RDB文件中加载数据，暂时不进行失效主键的删除，直接返回0    
    if (server.loading) return 0;
    ...
    //如果以上条件都不满足，就将主键的失效时间与当前时间进行对比，如果发现指定的主键    
    //还未失效就直接返回0
    if (mstime() <= when) return 0;
    //如果发现主键确实已经失效了，那么首先更新关于失效主键的统计个数，然后将该主键失    
    //效的信息进行广播，最后将该主键从数据库中删除
    server.stat_expiredkeys++;    
    propagateExpire(db,key);    
    return dbDelete(db,key);
}
```



### 12.3.3 主动删除

在redis.conf文件中可以配置主动删除策略，默认是no-enviction（不删除）

```basic
maxmemory-policy allkeys-lru
```



#### 12.3.3.1 LRU

LRU（Least recently used）最近最少使用，算法根据数据的历史访问记录来进行淘汰数据，其核心思想是【如果数据最近被访问过，那么将来被访问的记录也更高】。

最常见的实现是使用一个链表保存缓存数据，详细算法实现如下：

1. 新数据插入到链表头部；
2. 每当缓存命中（即缓存数据被访问），则将数据移到链表头部；
3. 当链表满的时候，将链表尾部的数据丢失。
4. 在Java中可以使用**LinkHashMap**（哈希链表）实现LRU。

以用户信息的需求为例，演示一下LRU算法的基本思路：

> 1.假设我们使用哈希链表来缓存用户信息，目前缓存了4个用户，这4个用户是按照时间顺序依次从链表右端插入的。

![image-20211030173637221](assest/image-20211030173637221.png)

> 2.此时，业务访问用户5，由于哈希链表中没有用户5的数据，我们从数据库中读取出来，插入到缓存当中。这时候，链表中最右端是最新访问到的用户5，最左端是最近最少访问的用户1。

![image-20211030174952851](assest/image-20211030174952851.png)

> 3.接下来，业务方访问用户2，哈希链表中存在用户2的数据，我们怎么做呢？我们把用户2从它的前驱节点和后继节点之间移除，重新插入到链表最右端。这时候，链表中最右端变成了最新访问到的用户2，最左端仍然是最近最少访问的用户1。

![image-20211030180555282](assest/image-20211030180555282.png)

![image-20211030180608792](assest/image-20211030180608792.png)

> 4.接下来，业务方请求修改用户4的信息。同样道理，我们把用户4从原来的位置移动到链表最右侧，并把用户信息的值更新。这时候，链表中最右端是最新访问到的用户4，最左端仍然是最近最少访问的用户1。

![image-20211030180940431](assest/image-20211030180940431.png)

![image-20211030180949031](assest/image-20211030180949031.png)



> 5.业务访问用户6，用户6在缓存中没有，需要插入到哈希链表。假设这时，缓存容量达到上限，必须先删除最近最少访问的数据，那么位于哈希链表最左端的用户1就会被删除，然后再把用户6插入到最右端。

![image-20211030181522013](assest/image-20211030181522013.png)

![image-20211030181529514](assest/image-20211030181529514.png)



> Redis的LRU数据淘汰机制

在服务器配置中保存了lru计数器server.lrulock，会定时（redis定时程序serverCorn()）更新，server.lrulock的值是根据server.unixtime计算出来的。

另外，从struct redisObject中可以发现，每一个redis对象都会设置相应的lru。可以想象的是，每一次访问数据的时候，会更新redisObject.lru。

LRU数据淘汰机制是这样的：在数据集中最忌挑选几个键值对，取出其中lru最大的键值对淘汰。

不可能遍历key	用当前时间-最近访问	越大说明	访问间隔时间越长

**volatile-lru** 从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰

**allkeys-lru** 从数据集（server.db[i].dict）中挑选最近最少使用的数据淘汰



#### 12.3.3.2 LFU

LFU（Least frequently used）最不经常使用，如果一个数据在最近一段时间内使用次数很少，那么在将来一段时间内被使用的可能性也很小。

volatile-lfu

allkeys-lfu

#### 12.3.3.3 random

随机

**volatile-random** 从以设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰

**allkeys-random** 从数据集（server.db[i].dict）中任意选择数据淘汰



#### 12.3.3.4 ttl

**volatile-ttl** 从以设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰

redis数据集结构中保存了键值对过期时间的表，即redisDb.expires。

TTL数据淘汰机制：从过期时间的表中随机挑选几个键值对，取出其中ttl最小的键值对淘汰

#### 12.3.3.5 noenviction

禁止驱逐数据，不删除默认



#### 12.3.3.6 缓存淘汰策略的选择

- allkeys-lru：在不确定时一般采用策略。冷热数据交换
- volatile-lru：比allkeys-lru性能差，需要存过期时间
- allkeys-random：希望请求符合平均分布（每个元素以相同的概率被访问）
- 自己控制：volatile-ttl（把过期时间设置小一点）

# 13 通讯协议及事件处理机制

## 13.1 通信协议

Redis是单进程单线程的。

应用系统和Redis通过Redis协议（RESP）进行交互。

### 13.1.1 请求响应模式

Redis协议位于TCP层之上，即客户端和Redis实例保持双工的连接。

![image-20211030191008541](assest/image-20211030191008541.png)

#### 13.1.1.1 串行的请求响应模式（ping-pong）

串行化是最简单的模式，客户端与服务器端简历长连接

连接通过心跳机制检测（ping-pong）ack应答

客户端发送请求，服务端响应，客户端收到响应后，再发起第二个请求，服务端再响应。

![image-20211031142234388](assest/image-20211031142234388.png)

telnet和redis-cli发出的命令，都属于该种模式

特点：

- 有问有答
- 耗时在网络传输命令
- 性能较低



#### 13.1.1.2 双工的请求响应模式（pipline）

批量请求，批量响应

请求响应交叉进行，不会混淆（TCP双工）

![image-20211031142439231](assest/image-20211031142439231.png)

- pipeline的作用是将一批命令进行打包，然后发送给服务器，服务器执行完按顺序打包返回。
- 通过pipeline，一次pipeline（n条命令） = 一次网络时间 + n次命令时间

通过Jedis可以很方便的使用pipeline

```java
Jedis redis = new Jedis("192.168.1.111", 6379);
redis.auth("12345678");//授权密码     对应redis.conf的requirepass密码 
Pipeline pipe = jedis.pipelined();
for (int i = 0; i <50000; i++) {
    pipe.set("key_"+String.valueOf(i),String.valueOf(i));      
}
//将封装后的PIPE一次性发给redis        
pipe.sync();
```



#### 13.1.1.3 原子化的批量请求响应模式（事务）

Redis可以利用事务机制批量执行命令。

#### 13.1.1.4 发布订阅模式（pub/sub）

发布订阅模式：一个客户端触发，多个客户端被动接收，通过服务器中转。

#### 13.1.1.5 脚本化的批量执行（lua）

客户端向服务器端提交一个lua脚本，服务器执行该脚本。

### 13.1.2 请求数据格式



### 13.1.3 命令处理流程

### 13.1.4 协议相应格式

### 13.1.5 协议解析及处理

## 13.2 时间处理机制

### 13.2.1 文件事件

### 13.2.2 时间事件

### 13.2.3 aeEventLoop

### 13.2.4 aeMain

### 13.2.5 aeProcessEvent





























