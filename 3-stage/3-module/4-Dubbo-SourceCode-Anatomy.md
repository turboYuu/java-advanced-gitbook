> 第四部分 Dubbo源码剖析

# 1 源码下载和编译

1. dubbo 的项目在 github 中的地址为 ：https://github.com/apache/dubbo
2. 进行需要进行下载的目录，执行 `git clone https://github.com/apache/dubbo.git`
3. 为了防止 master 中的代码不稳定，进入 dubbo 项目 `cd dubbo` 可以及切换到 2.7.6 分支 `git checkout 2.7.6`
4. 进行本地编译，进入 dubbo 项目 `cd dubbo`，进行编译操作 `mvn clean install -DskipTests`
5. 使用 IDE 引入项目。



![image-20220803124139668](assest/image-20220803124139668.png)

# 2 架构整体设计

[Dubbo 框架设计-官网说明](https://dubbo.apache.org/zh/docsv2.7/dev/design/)

## 2.1 Dubbo 调用关系说明

![/dev-guide/images/dubbo-relation.jpg](assest/dubbo-relation.jpg)

在这里主要由四部分组成：

1. Provider：暴露服务的服务提供方
   - Protocol：负责提供者 和 消费者 之间协议交互数据。
   - Service：真实的业务服务信息，可以理解成接口 和 实现。
   - Container：Dubbo 的运行环境。
2. Consumer：调用远程服务的服务消费方
   - Protocol：负责提供者 和 消费者 之间协议交互数据。
   - Cluster：感知提供者端的列表信息。
   - Proxy：可以理解成 提供者的服务调用代理类，由它接管 Consumer 中的接口调用逻辑。
3. Register：注册中心。用于作为服务发现 和 路由配置等工作，提供者和消费者都会在这里进行注册。
4. Monitot：用于提供者和消费者中的数据统计，比如调用频次，成功失败次数等信息。



**启动 和 执行流程说明**：



## 2.2 整体调用链路

## 2.3 Dubbo源码整体设计

# 3 服务注册与消费源码剖析

# 4 Dubbo 扩展 SPI 源码剖析

# 5 集群容错源码剖析

# 6 网络通信原理剖析