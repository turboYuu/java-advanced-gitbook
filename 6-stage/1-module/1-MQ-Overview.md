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

以上代码在执行中的问题：

1. 如果相应的更新一直失败，岂不是一直死循环直到调用栈崩溃？
2. 如果相应的更新一直在重试，在重试期间，添加商品的方法调用是不是一直阻塞中？
3. 如果添加商品的时候并发量很大，效率岂不是很低？



## 1.3 分布式异步通信模式

比较典型的 “生产者消费者模式”，可以跨平台、支持异构系统，通常借助消息中间件来完成。

优点：系统间解耦，并具有一定的可回复性，支持异构系统，下游通常可并发执行，系统具备弹性。服务解耦，流量削峰填谷等。

缺点：消息中间件存在一些瓶颈和一致性问题，对于开发来讲不直观不易调试，有额外成本。

使用异步消息模式需要注意的问题：

1. 哪些业务需要同步处理，哪些业务可以异步处理？
2. 如何保证消息的安全？消息是否丢失，是否会重复？
3. 请求的延迟如何能减少？
4. 消息接收的顺序是否会影响到业务流程的正常执行？
5. 消息处理失败后是否需要重发？如果重发如何保证幂等性？



# 2 消息中间件简介

## 2.1 消息中间件概念

维基百科对消息中间件的解释：面向消息的系统（消息中间件）是在分布式系统中完成消息的发送和接收的基础软件。

消息中间件也可称 消息队列，是指用高效可靠的消息传递机制进行与平台无关的数据交流，并基于数据通信来进行分布式系统的集成。通过提供消息传递和消息队列模型，可以在分布式环境下扩展进程的通信。

消息中间件就是在通信的上下游之间截断：break it，Broker

然后利用中间件解耦、异步的特性、构建弹性、可靠、稳定的系统。

体会一下：“必有歹人从中作梗，定有贵人从中相助”

**异步处理、流量削峰**、限流、缓冲、排队、**最终一致性、消息驱动** 等需求的场景都可以使用消息中间件。

![image-20221010112318474](assest/image-20221010112318474.png)

## 2.2 自定义消息中间件

并发比那还曾领域经典面试题：**请使用 java 代码来实现 “生产者消费者模式”**。

BlockingQueue（阻塞队列）是 java 中常见的容器，在多线程编程中被广泛使用。

当队列容器已满时 生产者线程被阻塞，直到队列未满后才可以继续 put；<br>当队列容器为空时，消费者线程被阻塞，直到队列非空时才可以继续 take。

![image-20221010150053911](assest/image-20221010150053911.png)

```java
package com.turbo.demo;

public class KouZhao  {

    private Integer id;
    private String type;

    @Override
    public String toString() {
        return "KouZhao{" +
                "id=" + id +
                ", type='" + type + '\'' +
                '}';
    }

    public Integer getId() {
        return id;
    }
    public void setId(Integer id) {
        this.id = id;
    }
    public String getType() {
        return type;
    }
    public void setType(String type) {
        this.type = type;
    }
}
```

```java
package com.turbo.demo;

import java.util.concurrent.BlockingQueue;

public class Producer implements Runnable {

    private BlockingQueue<KouZhao> queue;
    public Producer(BlockingQueue<KouZhao> queue) {
        this.queue = queue;
    }
    private Integer index = 0;

    @Override
    public void run() {
        while (true){
            try {
                Thread.sleep(200);

                if(queue.remainingCapacity() <= 0){
                    System.out.println("口罩已经堆积如山了...");
                }else{
                    KouZhao kouZhao = new KouZhao();
                    kouZhao.setType("N95");
                    kouZhao.setId(index++);
                    System.out.println("正在生产第"+(index-1)+"个口罩。");
                    queue.put(kouZhao);
                    System.out.println("已经生产了口罩："+queue.size()+"个。");
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
package com.turbo.demo;

import java.util.concurrent.BlockingQueue;

public class Consumer implements Runnable {

    private BlockingQueue<KouZhao> queue;
    public Consumer(BlockingQueue<KouZhao> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        while (true){
            try {
                Thread.sleep(100);
                System.out.println("正在准备买口罩...");
                final KouZhao kouZhao = queue.take();
                System.out.println("买到了口罩："+kouZhao.getId()+" "+kouZhao.getType()+"口罩。");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
package com.turbo.demo;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.BlockingQueue;

public class App {

    public static void main(String[] args) {
        BlockingQueue<KouZhao> queue = new ArrayBlockingQueue<>(20);

        new Thread(new Producer(queue)).start();
        new Thread(new Consumer(queue)).start();
    }
}
```



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

