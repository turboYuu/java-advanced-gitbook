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

**Tomcat 调试优化**

- 修改conf/server.xml 连接
  - 使用线程池
  - 修改 I/O，使用 NIO
  - 动态的修改垃圾回收参数
- 动静分离

# 6 Nginx

**Nginx 进程模式示意图**：

![image-20220707141355864](assest/image-20220707141355864.png)

# 7 多线程

线程状态转换

![image-20220413160602425](assest/image-20220413160602425.png)

# 8 JMM

JMM：Java 虚拟机规范中定义了 Java 内存模型（Java Memory Model，JMM），用于屏蔽掉各种硬件和操作系统的内存访问差异，以实现让 Java 程序在各种平台下都能达到一致的并发效果。JMM 规范了 Java 虚拟机与计算机内存是如何协同工作的：规定了一个线程如何 和 何时 可以看到由其他线程修改后的共享变量的值，以及在必须时如何同步的访问共享变量。

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



# 14 Zookeeper

Zookeeper 是一个开源的分布式协调服务

![ZooKeeper Service](assest/zkservice.jpg)

所有有关事务的请求都有 Leader 来处理。





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