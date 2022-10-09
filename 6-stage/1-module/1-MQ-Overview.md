第一部分 消息中间件概述

# 1 分布式架构通信

## 1.1 分布式架构通信原理

### 1.1.1 SOA

根据实际业务，把系统拆分成合适的、独立部署的模块，模块之间相互独立。

优点：分布式，松耦合、扩展灵活，可重用。



SOA 架构系统中，使用 Dubbo 和 Zookeeper 进行服务间的远程通信。

优点：Dubbo使用自定义的 TCP 协议，可以让请求报文体积更小，或者使用 HTTP2 协议，也可以减少报文的体积，提高传输效率。

### 1.1.2 微服务架构

SpringCloud 中使用 Feign 解决服务之间远程通信的问题。

Feign：轻量级 Restful 的 HTTP 服务客户端，广泛应用于 Spring Cloud 中。符合面向接口化的编程习惯。

本质：封装了 HTTP 调用流程，类似 Dubbo 的服务调用。

多用于 同步远程调用



RPC 主要基于 TCP/UDP 协议，HTTP 协议是应用层协议，是构建在传输层协议 TCP 之上的，RPC 更高，RPC 长连接：不必每次通信都像 HTTP 一样三次握手，减少网络开销；

HTTP 服务开发迭代更快：在接口不多，系统与系统治安交互比较少的情况下，HTTP 就显得更加方便；相反，在接口比较多，系统与系统之间交互比较多的情况下，HTTP就没有 RPC 优势。



## 1.2 分布式同步通信的问题

电商项目中，如果后台添加商品信息，该信息放到数据库。

我们同时，需要更新搜索引擎的倒排序索引，同时，加入有商品页面静态化处理，也需要更新页面信息。

![image-20221009191046694](assest/image-20221009191046694.png)

怎么解决？

方式一：可以在后台添加商品的方法中，如果数据插入数据库成功 ，就调用更新倒排序索引的方法，接着调用更新静态化页面的方法。

代码应该是：

```java
Long goodsId = addGoods(goods); 
if (goodsId != null) {
	refreshInvertedIndex(goods);    
    refreshStaticPage(goods); 
}
```

问题：

假如更新倒排序索引失败，该怎么办？

假如更新静态页面失败怎么办？

解决方式：

如果更新倒排索引失败，重试

如果更新静态页面失败，重试

代码应该是这样：

```java
public Long saveGoods() {
	Long goodsId = addGoods(goods);    
    if (goodsId != null) {
		// 调用递归的方法，实现重试
		boolean indexFlag = refreshInvertedIndex(goods);       
        // 调用递归的方法，实现重试
		boolean pageFlag = refreshStaticPage(goods);  
    }
}
private boolean refreshInvertedIndex(Goods goods) {    
    // 调用服务的方法
	boolean flag = indexService.refreshIndex(goods);    
    if (!flag) {
		refreshInvertedIndex(goods);  
    }
}
private boolean refreshStaticPage(Goods goods) {    
    // 调用服务的方法
	boolean flag = staticPageService.refreshStaticPage(goods);    
    if (!flag) {
		refreshStaticPage(goods);  
    }
}
```



## 1.3 分布式异步通信模式

# 2 消息中间件简介

## 2.1 消息中间件概念

## 2.2 自定义消息中间件

## 2.3 主流消息中间件及选型

### 2.3.1 选取原则

### 2.3.2 RabbitMQ

### 2.3.3 RocketMQ

### 2.3.4 Kafka

## 2.4 消息中间件应用场景

# 3 JMS规范和AMQP协议

## 3.1 JMS经典模式详解

### 3.1.1 JMS消息

### 3.1.2 体系结构

### 3.1.3 对象模型

### 3.1.4 模式

### 3.1.5 传递方式

### 3.1.6 供应商

## 3.2 JMS在应用集群中的问题

## 3.3 AMQP协议剖析

### 3.3.1 协议架构

### 3.3.2 AMQP中的概念

### 3.3.3 AMQP传输层架构

#### 3.3.3.1 简要概述

#### 3.3.3.2 数据类型

#### 3.3.3.3 协议协商

#### 3.3.3.4 数据帧界定

### 3.3.4 AMQP客户端实现JMS客户端

