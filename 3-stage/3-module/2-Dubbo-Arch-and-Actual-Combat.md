> 第二部分 Dubbo架构与实战

# 1 Dubbo 架构概述

## 1.1 什么是Dubbo

Apache Dubbo 是一款 RPC 服务开发框架，用于解决微服务架构下的服务治理与通信问题。

## 1.2 Dubbo 的特性

参考官网 [Dubbo 核心特性](https://dubbo.apache.org/zh/overview/what/overview/#dubbo-%E6%A0%B8%E5%BF%83%E7%89%B9%E6%80%A7)

## 1.3 Dubbo 的服务治理

服务治理（SOA governance），企业为了确保项目顺利完成二实施的过程，包括最佳实践，架构原则、治理规程、规律以及其他决定性的因素。服务治理指的是用来管理 SOA 的采用和实现的过程。

参考官网 [服务治理](https://dubbo.apache.org/zh/docs/v2.7/user/preface/requirements/)

![image](assest/dubbo-service-governance.jpg)

# 2 Dubbo 处理流程

![dubbo-architucture](assest/dubbo-architecture.jpg)

节点角色说明：

| 节点      | 角色名称                                 |
| --------- | ---------------------------------------- |
| Provide   | 暴露服务的服务提供方                     |
| Consumer  | 调用远程服务的服务消费方                 |
| Registry  | 服务注册与发现的注册中心                 |
| Monitor   | 统计服务的调用次数 和 调用时间的监控中心 |
| Container | 服务运行容器                             |

**调用关系说明**

0. 服务容器负责启动，加载，运行服务提供者。
1. 服务提供者在启动时，向注册中心注册自己提供的服务。
2. 服务消费者在启动时，向注册中心订阅自己所需的服务。
3. 注册中心返回服务提供者地址列表给消费者，如有变更，注册中心将基于长连接推送变更数据给消费者。
4. 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
5. 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。

# 3 服务注册中心 Zookeeper



# 4 Dubbo 开发实战

# 5 Dubbo 管理控制台

# 6 Dubbo配置项说明