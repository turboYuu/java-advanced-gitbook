> 第五部分 Redis高可用方案

"高可用性"（High Availability）通常来描述一个系统经过专门的设计，从而减少停工时间，而保持其服务的高度可用性。

单机的 Redis 是无法保证高可用性的，当 Redis 服务器宕机后，即使在有持久化的机制下也无法保证不丢失数据。

所以我们采用 Redis 多机和集群的方式来保证 Redis 的高可用性。

# 1 主从复制

Redis支持主从复制功能，可以通过执行slaveof（Redis 5 以后改成replicaof）或者配置文件中设置slaveof（Redis 5以后改成replicaof）来开启复制功能。

![image-20220927171020446](assest/image-20220927171020446.png)

![image-20220927171052334](assest/image-20220927171052334.png)

![image-20220927171122688](assest/image-20220927171122688.png)

- 主对外 从对内，主可写，从不可写
- 主挂了，从不可为主

## 1.1 主从配置

### 1.1.1 主Redis配置

无需特殊配置

### 1.1.2 从Redis配置

（安装后的redis 使用  `cp -r redis redis-slave-6380` 复制出一个redis）

修改从服务器上的`redis.conf`文件：

```properties
# replicaof <masterip> <masterport>
# 表示当前【从服务器】对应的【主服务器】的ip是192.168.31.135，端口号是6379
replicaof 127.0.0.1 6379
```

```bash
[root@localhost bin]# ./redis-cli -p 6380
127.0.0.1:6380> set lock 20
(error) READONLY You can't write against a read only replica.
```

## 1.2 作用

### 1.2.1 读写分离

- 一主多从，主从同步。

- 主负责写，从负责读。

- 提升Redis的性能和吞吐量。

- 主从的数据一致性问题。

### 1.2.2 数据容灾

- 从机是主机的备份。
- 主机宕机，从机可读不可写。
- 默认情况下主机宕机后，从机不可为主机。
- 利用**哨兵**可以实现主从切换，做到高可用。

## 1.3 原理与实现

### 1.3.1 复制流程

#### 1.3.1.1 保存主节点信息

当客户端向从服务器发送 slaveof(replicaof) 主机地址（127.0.0.1）端口（6379）时：从服务器将主机IP（127.0.0.1）和 端口（6379）保存到 redisServer 的 masterhost 和 masterprot 中。

```c
Struct redisServer{
	char *masterhost; 	//主服务器ip   
	int  masterport; 	//主服务器端口 
};
```

从服务器将向发送 SLAVEOF  命令的客户端返回 OK，表示复制指令已经被接收，而实际上复制工作是在 OK 返回之后进行。

#### 1.3.1.2 建立 socket 链接

slaver 与 master 建立 socket 链接。

slaver 关联文件事件处理器。

该处理器接收 RDB 文件（全量复制）、接收 Master 传播来的写命令（增量复制）

![image-20220419135444484](assest/image-20220419135444484.png)

主服务器 accept 从服务器 socket 连接后，创建相应的客户端状态。相当于从服务器是主服务器的 Client 端。

![image-20220419135939187](assest/image-20220419135939187.png)

#### 1.3.1.3 发送 ping 命令

**Slaver 向 Master 发送 ping 命令**

1. 检测 socket 的读写状态
2. 检测 Master 能否正常处理

**Master 的响应**：

1. 发送 “pong” ，说明正常
2. 返回错误，说明 Master 不正常
3. timeout，说明网络超时

![image-20220419140544467](assest/image-20220419140544467.png)

#### 1.3.1.4 权限验证

主从正常连接后，进行权限验证

**主**未设置密码（requirepass=""），**从** 也不用设置密码（masterauth=""）

**主**设置密码（requirepass!=""）， **从**需要设置密码（masterauth=主的requirepass的值）

或者 **从** 通过 auth 命令向 **主** 发送密码。

![image-20220927172428077](assest/image-20220927172428077.png)

![image-20220419143333929](assest/image-20220419143333929.png)

#### 1.3.1.5 发送端口信息

在身份验证步骤之后，从服务器将执行命令 REPLCONF listening-port，向主服务器发送从服务器的监听端口号。

![image-20220419143625737](assest/image-20220419143625737.png)

#### 1.3.1.6 同步数据

Redis 2.8 之后分为全量同步 和 增量同步，具体的后面详细讲解。

#### 1.3.1.7 命令传播

当同步数据完成后，主从服务器就会进入命令传播阶段，主服务器只要将自己执行的写命令发送给从服务器，而从服务器只要一直执行并接收主服务器发来的写命令。

### 1.3.2 同步数据集

Redis 2.8 以前使用 SYNC 命令同步复制

Redis 2.8 之后使用 PSYNC 命令替换 SYNC

#### 1.3.2.1 旧版本

Redis 2.8 以前

##### 1.3.2.1.1 实现方式

Redis 的同步功能分为 同步（sync）和 命令传播（command propagate）

###### 1.3.2.1.1.1 同步操作

1. 通过从服务器发送到 SYNC 命令给主服务器
2. 主服务器生成 RDB 文件并发送给从服务器，同时发送保存所有写命令给从服务器
3. 从服务器清空之前数据并执行解析 RDB 文件。
4. 保持数据一致（还需要命令传播过程才能保持一致）

![image-20220419154438228](assest/image-20220419154438228.png)

###### 1.3.2.1.1.2 命令传播操作

同步操作完成后，主服务器执行写命令，该命令发送给从服务器并执行，使主从保持一致。

##### 1.3.2.1.2 缺陷

没有 全量同步 和 增量同步 的概念，从服务器在同步时，会清空所有数据。

主从服务器断线后重新复制，主服务器会重新生成 RDB 文件和重新记录缓冲区的所有写命令，并全量同步到从服务器上。



#### 1.3.2.2 新版本

Redis 2.8 以后

##### 1.3.2.2.1 实现方式

在 Redis 2.8 之后使用 PSYNC 命令，具备 完整重同步 和 部分重同步 模式。

- Redis 的主从同步，分为 **全量同步** 和 **增量同步**。

- 只有从服务器第一次连接上主机是 **全量同步**。

- 断线重连有可能触发 **全量同步** 也有可能是 **增量同步**（`master` 判断 `runid` 是否一致）。

  ![image-20220419162333073](assest/image-20220419162333073.png)

- 除此之外的情况都是 **增量同步**。

###### 1.3.2.2.1.1 全量同步

Redis 的 全量同步过程主要分为三个阶段：

- **同步快照阶段**：Master 创建并发送 **快照** RDB 给 Slave，Slave 载入并解析快照。Master 同时将此阶段所产生的新的写命令存储到缓冲区。
- **同步写缓冲阶段**：Master 向 Slave 同步存储在缓冲区的写操作命令。
- **同步增量阶段**：Master 向 Slave 同步写操作命令。

![image-20220419165046745](assest/image-20220419165046745.png)



###### 1.3.2.2.1.2 增量同步

- Redis 增量同步主要指 Slave 完成初始化后开始正常工作时，Master 发生的写操作同步到 Slave 的过程。
- 通常情况下，Master 每执行一个写命令就会向 Slave 发送相同的 **写命令**，然后 Slave 接收并执行。

### 1.3.3 心跳检测

在命令传播阶段，从服务器默认会以每秒一次的频率向主服务器发送命令：

```properties
replconf ack <replication_offset>
    
#ack :应答
#replication_offset：从服务器当前的复制偏移量
```

主要作用有三个：

1. 检测主从的连接状态

   检测主从服务器的网络连接状态

   通过向主服务器发送 INFO replication 命令，可以列出从服务器列表，可以看出最后一次向主发送命令距离现在过了多少秒。lag 的值应该在 0 或 1 之间跳动，如果超过 1 则说明主从之间的连接有故障。

   ![image-20220927173711796](assest/image-20220927173711796.png)

2. 辅助实现 min-slaves

   ![image-20220927173303392](assest/image-20220927173303392.png)

   Redis 可以通过配置防止主服务器在不安全的情况下执行写命令

   min-replicas-to-write 3 （min-replicas-to-write 3）

   min-replicas-max-lag 10 （min-replicas-max-lag 10）

   上面的配置表示：从服务器的数量少于 3 个，或者3个从服务器的延迟（lag）值都大于或等于10秒时，主服务器将拒绝执行写命令。这里的延迟值就是上面 `info replication` 命令的 lag 值。

3. 检测命令丢失

   如果因为网络故障，主服务器传播给从服务器的写命令在半路丢失，那么当从服务器向主服务器发送 REPLCONF ACK 命令时，主服务器将发觉从服务器当前的复制偏移量少于自己的复制偏移量，然后主服务器就会根据从服务器提交的复制偏移量，在复制积压缓冲区里面找到从服务器缺少的数据，并将这些数据重新发送给从服务器。（补发：网络不断）。

   增量同步：网断了，再次连接

# 2 哨兵模式

哨兵（Sentinel）是 Redis 高可用性（High Availability）的解决方案：

由一个或多个 sentinel 实例组成 sentient 集群可以监视一个或多个主服务器和多个从服务器。

当主服务器进行下线状态时，sentinel 可以将主服务器下的某一从服务器升级为主服务器继续提供服务，从而保证 redis 的高可用。

## 2.1 部署方案

![image-20220927180025725](assest/image-20220927180025725.png)



## 2.2 搭建配置

这里在一台机器上采用伪分布式的方式不是。（生产环境应该是多台）

根据上面的部署方案搭建如下：

Redis-Master：127.0.0.1 6379，采用安装的方式，正常安装和配置。

```bash
# 1 安装 redis 5.0
mkdir -p /usr/redis-ms/redis-master
cd redis-5.0.5/src
make install PREFIX=/usr/redis-ms/redis-master
cp /root/redis-5.0.5/redis.conf /usr/redis-ms/redis-master/bin/

# 2 修改 redis.conf
	# 将`daemonize`由`no`改为`yes`
	daemonize yes
	
	# 默认绑定的是回环地址，默认不能被其他机器访问 
	# bind 127.0.0.1
	
	# 是否开启保护模式，由yes该为no
	protected-mode no  
```

Redis-Slaver1：127.0.0.1 6380

```bash
# 在 /usr/redis-ms 目录下
cp -r redis-master redis-slaver1
# 修改配置文件
vim redis-slave1/bin/redis.conf
	port 6380
	replicaof 127.0.0.1 6379
```

Redis-Slaver2：127.0.0.1 6381

```bash
# 在 /usr/redis-ms 目录下
cp -r redis-master redis-slaver2
# 修改配置文件
vim redis-slave2/bin/redis.conf
	port 6381
	replicaof 127.0.0.1 6379
```



Redis-Sentinel1：127.0.0.1 26379

```bash
# 在 /usr/redis-ms 目录下
cp -r redis-master redis-sentinel1
# 拷贝 sentinel.conf 配置文件并修改
cp /root/redis-5.0.5/sentinel.conf /usr/redis-ms/redis-sentinel1/bin/
vim redis-sentinel1/bin/sentinel.conf

	# 哨兵sentinel实例运行的端口，默认 26379
	port 26379
	
	# 将`daemonize`由`no`改为`yes`
	daemonize yes
	
	# 哨兵 sentinel 监控的redis主节点的 ip port
	# master-name 可以自己命名的主节点名字，只能由字母 A-z、数字0-9、这三个字符“.-_” 组成。
	# quorum 当这些 quorum 个数 sentinel哨兵认为 master 主节点失联，那么这时、客观上认为主节点失联了
	# sentinel monitor <master-name> <ip> <redis-port> <quorum>
	sentinel monitor mymaster 127.0.0.1 6379 2
	
	# 当在 Redis 实例中开启了requirepass foobared 授权密码，这样所有连接Redis实例的客户端都要提供密码
	# 设置哨兵sentinel连接主从的密码，注意必须为主从设置一样的验证密码
	# sentinel auth-pass <master-name> <password>
	sentinel auth-pass mymaster MySUPER--secret-0123passw0rd
	
	# 执行多少毫秒之后，主节点没有应答哨兵sentinel此时，哨兵主观上认为主节点下线，默认30秒，改为3秒
	# sentinel down-after-milliseconds <master-name> <milliseconds>
	sentinel down-after-milliseconds mymaster 30000
	
	# 这个配置项指定了在发生failover主备切换时最多可以有多少个slave同时对新的master进行同步。
	# 这个数字越小，完成failover所需的时间就越长，
	# 但是如果这个数字越大，就意味着，多的slave因为replication而不可用
	# 可以通过将这个值设置为1，来保证每次只有一个slave处于不能处理命令请求的状态
	# sentinel parallel-syncs <master-name> <numreplicas>
	sentinel parallel-syncs mymaster 1
	
	# 故障转移的超时时间，failover-timeout 可以用在以下这些方面：
	# 1. 同一个sentinel对同一个master两次failover之间的间隔时间。
	# 2. 当一个slave从一个错误的master那里同步数据开始计算时间，直到slave被纠正为正确的master那里同步数据时。
	# 3. 当想要取消一个正在进行的failover所需的时间
	# 4. 当进行faliover时，配置所有slaves指向新的master所需多的最大时间。不过，即使过了这个超时，slaves依然会被正确配置为指向master，但是就不按parallel-syncs所配置的规则来了
	# sentinel failover-timeout <master-name> <milliseconds>
	sentinel failover-timeout mymaster 180000
```



Redis-Sentinel2：127.0.0.1 26380

```bash
cp -r redis-sentinel1/ redis-sentinel2
# 修改 sentinel.conf 配置文件
vim redis-sentinel2/bin/sentinel.conf
	port 26380
```



Redis-Sentinel3：127.0.0.1 26381

```bash
cp -r redis-sentinel1/ redis-sentinel3
# 修改 sentinel.conf 配置文件
vim redis-sentinel3/bin/sentinel.conf
	port 26381
```

配置好依次执行 redis-master、redis-slaver1、redis-slaver2、redis-sentinel1、redis-sentinel2、redis-sentinel3。

这里编写一个 start.sh 可执行文件，`chmod 77 start.sh`

```bash
vim 
cd redis-master/bin
./redis-server redis.conf
cd ..
cd ..

cd redis-slaver1/bin
./redis-server redis.conf
cd ..
cd ..

cd redis-slaver2/bin
./redis-server redis.conf
cd ..
cd ..

cd redis-sentinel1/bin
./redis-sentinel sentinel.conf
cd ..
cd ..

cd redis-sentinel2/bin
./redis-sentinel sentinel.conf
cd ..
cd ..

cd redis-sentinel3/bin
./redis-sentinel sentinel.conf
cd ..
cd ..
```

执行 start.sh 文件

```bash
[root@localhost redis-ms]# ps -ef|grep redis
root     16136     1  0 19:21 ?        00:00:00 ./redis-server *:6379
root     16141     1  0 19:21 ?        00:00:00 ./redis-server *:6380
root     16147     1  0 19:21 ?        00:00:00 ./redis-server *:6381
root     16152     1  0 19:21 ?        00:00:00 ./redis-sentinel *:26379 [sentinel]
root     16157     1  0 19:21 ?        00:00:00 ./redis-sentinel *:26380 [sentinel]
root     16159     1  0 19:21 ?        00:00:00 ./redis-sentinel *:26381 [sentinel]
root     16172  1581  0 19:21 pts/0    00:00:00 grep --color=auto redis
```

## 2.3 执行流程

### 2.3.1 启动并初始化Sentinel

### 2.3.2 获取主服务器信息

### 2.3.3 获取从服务器信息

### 2.3.4 项主服务器和从服务器发送消息（以订阅的方式）

### 2.3.5 接收来自主服务器和从服务器的频道信息

### 2.3.6 检测主观下线状态

### 2.3.7 检测客观下线状态

### 2.3.8 选举Leader Sentinel

## 2.4 哨兵 leader 选举

## 2.5 主服务器的选择

# 3 集群与分区

## 3.1 分区的意义

## 3.2 分区的方式

## 3.3 client 端分区

## 3.4 proxy 端分区

## 3.5 官方 cluster 分区