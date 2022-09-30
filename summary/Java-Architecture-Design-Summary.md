> Java后端各种架构图汇总



# 1 Mybatis 架构

[MyBatis](https://mybatis.org/mybatis-3/zh/index.html) 是一款优秀的持久层框架，它支持自定义 SQL、存储过程以及高级映射。

Mybatis 中比较重要的几个类 SqlSessionFactoryBuilder、SqlSessionFactory、SqlSession、Executor。

Mybatis架构设计

![image-20220426111724390](assest/image-20220426111724390.png)

Mybatis 层次结构 和 执行流程

![image-20220426130653547](assest/image-20220426130653547.png)

# 2 Spring Framework

[Spring](https://spring.io/projects/spring-framework) 是分层的 full-stack（全栈）轻量级开源框架，以 `IoC` 和 `AOP` 为内核，提供了展现层 SpringMVC 和 业务层事务管理等 众多的企业级应用技术，还整合开源世界众多著名的第三方框架和类库，已经成为使用最多的 Java EE 企业应用开源框架。

![image-20211019185543778](assest/image-20211019185543778.png)



![image-20220812173614259](assest/image-20220812173614259.png)

**SpringBean 的生命周期**

![image-20220812173817455](assest/image-20220812173817455.png)

如果 该bean中有注入其他的bean，那么会在 BeanPostProcessor 的 postProcessBeforeInitialization()、 postProcessAfterInitialization() 方法之前进行。（循环依赖也就在这里实现）

Spring AOP 生成代理对象是在 postProcessAfterInitialization() 中完成的。

**Spring IoC 的循环依赖**：

![image-20220812175613829](assest/image-20220812175613829.png)

三级缓存中，一级缓存存放完全初始化好的bean，二级缓存和三级缓存中存放的是提前暴露的bean，从三级缓存到二级缓存可以做一些扩展的功能，同时通过三级缓存的名字可以看出是一个工厂，可生成一些代理的bean。这就是为什么Spring IoC循环依赖需要三级缓存，而不是二级缓存。





# 3 Spring MVC

Spring MVC 是 Spring 给我们提供的一个用于简化 Web 开发的框架。

**MVC 设计模式**

![image-20220407163821263](assest/image-20220407163821263.png)

**Spring MVC 请求处理流程**

![image-20220407125553523](assest/image-20220407125553523.png)



**监听器、过滤器 和 拦截器的区别**

![image-20220409143203202](assest/image-20220409143203202.png)

# 4 SpringBoot 

SpringBoot 启动类的 SpringApplication.run 方法解析



# 5 Tomcat

Http 请求处理过程

![image-20220629142802249](assest/image-20220629142802249.png)

**Tomcat Servlet 容器处理过程**

![image-20220629153948180](assest/image-20220629153948180.png)

Tomcat的两个重要身份：

1. http服务器；
2. Tomcat是一个 Servlet 容器。

**Tomcat 调试优化**

- 修改conf/server.xml 连接
  - 使用线程池
  - 修改 I/O，使用 NIO
  - 动态的修改垃圾回收参数
- 动静分离

# 6 Nginx

## 6.1 Nginx 配置文件

- 全局块

  ![image-20220707092749993](assest/image-20220707092749993.png)

- events 块

  ![image-20220707094707593](assest/image-20220707094707593.png)

- http 块

  ![image-20220707095506426](assest/image-20220707095506426.png)

  ![image-20220707095611629](assest/image-20220707095611629.png)

  ![image-20220707095650457](assest/image-20220707095650457.png)

## 6.2 Nginx 进程模式示意图

![image-20220707141355864](assest/image-20220707141355864.png)

# 7 多线程

线程状态转换

![image-20220413160602425](assest/image-20220413160602425.png)

# 8 JMM

JMM：Java 虚拟机规范中定义了 Java 内存模型（Java Memory Model，JMM），用于屏蔽掉各种硬件和操作系统的内存访问差异，以实现让 Java 程序在各种平台下都能达到一致的并发效果。**JMM 规范了 Java 虚拟机与计算机内存是如何协同工作的**：规定了一个线程如何 和 何时 可以看到由其他线程修改后的共享变量的值，以及在必须时如何同步的访问共享变量。

![image-20210925185512917](assest/image-20210925185512917.png)





# 9 线程池

线程池状态

![image-20210923133944932](assest/image-20210923133944932.png)

# 10 JVM

![image-20211008140248375](assest/image-20211008140248375.png)



![image-20211008124440250](assest/image-20211008124440250.png)



**类加载的过程**

![image-20211014165304036](assest/image-20211014165304036.png)



# 11 NIO

Java NIO全称 Java non-blocking IO，是指JDK提供的新API。从JDK1.4 开始，Java提供了一系列改进输入/输出的新特性，被统称为NIO（New I/O）同步非阻塞。

一张图描述NIO的Selector、Channel、Buffer的 三大核心原理示意图 关系

![image-20211105185416652](assest/image-20211105185416652.png)



# 12 Netty

Netty是由 JBOSS 提供的一个Java开源框架。**Netty 提供异步的、基于事件驱动的网络应用程序框架，用以快速开发高性能、高可靠的网络 IO 程序。**

详细版 Netty 线程模型

![image-20211108183953465](assest/image-20211108183953465.png)

# 13 集群、分布式、SOA、微服务

集群：不同服务器部署同一企应用服务对外提供访问，实现服务的负载均衡或者互备(热备，主从等)，指同一种组
件的多个实例，形成的逻镇上的整体。单个节点可以提供完整服务。**集群是物理形态**

分布式：服务的不同模块部署在不同的服务器上，单个节点不能提供完整服务，需要多节点协调提供服务(也可以
是相同组件部署在不同节点． 但节点间通过交换信息协作提供服务)，**分布式强调的是工作方式**

SOA：面向服务的架构，一种设计方法，其中包含多个服务，服务之间通过相互依赖最终提供一系列的功能。一个服务通常以独立的形式存在于操作系统进程中。各个服务之间通过网络调用。

- 中心化实现：
  ESB(企业服务总线)，各服务通过ESB进行交互，解决异构系统之间的连通性，通过协议转换、消
  息解析、消息路由把服务提供者的数据传送到服务消费者。很重，有一定的逻辑，可以解决一些公用逻辑的问
  题。
- 去中心化实现：微服务

微服务：在 SOA 上做的开华，微服务架构强调的一个重点是业务需要彻底的组件化和服务化，原有的单个业务系
统会拆分为多个可以独立开发．设计，运行的小应用。这些小应用之间通过服务完成交互和集成

服务单一职责
轻量级通信：去掉ESB总线 来用 restful (PRC、restTemplate)通信

# 14 Zookeeper

Zookeeper 是一个开源的分布式协调服务

![ZooKeeper Service](assest/zkservice.jpg)

所有有关事务的请求都有 Leader 来处理。

ZAB 协议（崩溃恢复、消息广播），Zookeeper 集群的Leade选举。



# 15 Dubbo 架构

[Apache Dubbo](https://dubbo.apache.org/zh/) 是一款微服务框架，为大规模微服务实践提供高性能 RPC 通信、流量治理、可观测性等解决方案。

![/dev-guide/images/dubbo-framework.jpg](assest/dubbo-framework.jpg)

理解这十层，每层的作用，可以作为自定义实现 RPC 框架的思路。

# 16 Spring Cloud 

第一代 Spring Cloud Netflix

## 16.1 Spring Cloud Eureka

服务注册 与 发现

![image-20220818120417439](assest/image-20220818120417439.png)

## 16.2 Spring Cloud Ribbon

负载均衡

**Ribbon工作原理**

![image-20220819183930133](assest/image-20220819183930133.png)

**Ribbon细节结构图**

![image-20220819184552853](assest/image-20220819184552853.png)



## 16.3 Eureka 服务发现慢的原因

主要有两个：一部分是因为服务缓存导致的，另一部分是客户端缓存导致的。

# 17 MySQL

## 17.1 MySQL 体系结构

MySQL Server 架构自顶向下大致可以分为网络连接层、服务层、存储引擎层 和 系统文件层。

![preview](assest/view)

## 17.2 MySQL 运行机制

五个部分

![image-20210715133857845](assest/image-20210715133857845.png)

## 17.3 InnoDB 存储结构

![InnoDB architecture diagram showing in-memory and on-disk structures.](assest/innodb-architecture.png)



## 17.4 InnoDB 中 LogBuffer 刷盘方式

innodb_flush_log_at_trx_commit参数控制日志刷新行为，可通过 innodb_flush_log_at_trx_commit 设置：

- 0：每秒提交 LogBuffer -> OS cache -> flush cache to disk，可能丢失一秒内的事务数据。由后台 master 线程每隔 1 秒 执行一次操作。
- 1（默认值）：每次事务提交执行 LogBuffer -> OS cache -> flush cache to disk，最安全，性能最差的方式。
- 2：每次事务提交执行 LogBuffer -> OS cache ，然后由后台 Master 线程再每隔1秒执行 OS cache -> flush cache to disk 的操作。

一般建议选择 2，因为 MySQL 挂了数据没有损失，整个服务器挂了才会损失1秒的事务提交数据。

![image-20220905184101688](assest/image-20220905184101688.png)

# 18 Redis



redis 常用数据类型 对应的底层数据结构（所有的数据类型都会使用的 type：数据字典）

| 数据类型   | 底层数据结构 type | RedisObject-encoding                                         |
| ---------- | ----------------- | ------------------------------------------------------------ |
| String     | 字符串对象        | int、row、embstr                                             |
| List       | 快速链表          | quicklist                                                    |
| Hash       | 压缩列表          | dict（当散列表元素的个数比较多 或 元素不是小整数 或 短字符串时）<br>ziplist（当散列表元素的个数比较少，且元素都是小整数 或 短字符串时） |
| Set        |                   | intset（整数集合）、dict（字典）                             |
| Sorted set | 跳跃表，压缩列表  | ziplist（压缩列表）、skiplist+dict（跳跃表+字典）            |

## 18.1 Redis 高可用

|               | 节点                               |      |
| ------------- | ---------------------------------- | ---- |
| Redis 主从    | 主节点：可读写<br>从节点：只可以读 |      |
| 哨兵          |                                    |      |
| Redis Cluster |                                    |      |







# 各种服务集群总结

| 服务           | 节点类型                   | 各节点的功能区别                                             | 主节点的选举的触发条件和策略 |
| -------------- | -------------------------- | ------------------------------------------------------------ | ---------------------------- |
| Zookeeper      | Leader、Follower、Observer | Leader：事务请求的唯一调度者，集群内各服务器的调度者<br>Follower：非事务请求，转发事务请求给 Leader；参与事务请求 Proposal 投票；参与Leader选举。<br>Observer：和Follower基本一致，但不参与任何投票。 |                              |
| MySQL 主从复制 | Master、Slave              | Master：可以写和读（主要是写）<br>Slave：只提供读（从Master同步数据）<br>同步的方式 有：异步同步，半同步复制，并行同步（基于组提交的并行复制）<br>分库分表的话 需要一个 中间件 在应用和数据库之间 将读和写指向不同的从、主库，以及数据分片策略、主键策略。 |                              |
| Redis          | 主从                       |                                                              |                              |



# 如何健壮你的后端服务？

## 1 怀疑第三方

### 1.1 有兜底，制定好业务降级方案

### 1.2 遵循快速失败原则，一定要设置超时时间

### 1.3 适当保护第三方，慎重选择重试机制

## 2 防备使用方

### 2.1 设计一个好的 api（RPC、Restful）避免误用

1. 遵循接口暴露最少原则
2. 不要让使用方做接口可以做的事情
3. 避免长时间执行的接口
4. 参数启用原则
5. 异常

### 2.2 流量控制，按服务分配流量，避免滥用

## 3 做好自己

### 3.1 单一职责原则

### 3.2 控制资源的使用

#### 3.2.1 CPU资源怎么限制

#### 3.2.2 内存资源怎么限制