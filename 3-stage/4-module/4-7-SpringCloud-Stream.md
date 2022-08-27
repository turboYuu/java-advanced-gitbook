> 4-7 Spring Cloud Stream 消息驱动组件

Spring Cloud Stream 消息驱动组件帮助我们更快速，更方便，更友好的去构建**消息驱动**微服务的。

定时任务和消息驱动的一个对比。（消息驱动：基于消息机制做一些事情）

MQ：消息队列/消息中间件/消息代理，产品有很多，ActiveMQ、RabbitMQ、RocketMQ、Kafka

# 1 Stream 解决的痛点问题

MQ 消息中间件广泛应用在 应用解耦、异步消息处理、流量削峰等场景中。

不同的 MQ 消息中间件内部机制包括使用方式都会有所不同，比如 RabbitMQ 中有 Exchange（交换机/交换器）这一概念，Kafka 有 Topic、Partition 分区这些概念，MQ 消息中间件的差异性不利于我们上层的开发应用，当我们的系统希望从原有的 RabbitMQ 切换到 Kafka 时，我们会发现比较困难，很多要操作重来（**因为应用程序和具体的某一款 MQ 消息中间件耦合在一起了**）。

Spring Cloud Stream 进行了很好的**上层抽象**，可以让我们与具体消息中间件解耦，屏蔽了底层具体 MQ 消息中间件的细节差异，就像 HIbernate 屏蔽掉了具体数据库（Mysql/Oracle一样）。如此一来，学习、开发、维护 MQ 都会变得轻松。目前 Spring Cloud Stream 支持 RabbitMQ 和 Kafka。

本质：**屏蔽掉了底层不同MQ消息中间件之间的差异，统一了MQ的编程模型，降低了学习、开发、维护 MQ的成本**



# 2 Stream 重要概念

Spring Cloud Stream 是一个构建消息驱动微服务的框架。应用程序通过 inputs（相当于消息消费者 consumer）或者 outputs（相当于消息生产者 producer）来与 Spring Cloud Stream 中的 binder 对象交互，而 Binder 对象式用来屏蔽底层 MQ 细节的，它负责与具体的消息中间件交互。

**说白了：对于我们来说，只需要知道如何使用 Spring Cloud Stream 与 Binder 对象交互即可**

![SCSt with binder](assest/SCSt-with-binder.png)

![image-20220826182133498](assest/image-20220826182133498.png)

**Binder 绑定器**

Binder 绑定器是 Spring Cloud Stream 中非常核心的概念，就是通过它来屏蔽底层不同 MQ 消息中间件的细节差异，当需要更换为其他消息中间件时，我们需要做的就是 **更换对应的 Binder 绑定器** 而不需要修改任何逻辑（Binder 绑定器的实现是框架内置的，Spring Cloud Stream 目前支持 Rabbit、Kafka 两种消息队列）

# 3 传统 MQ 模型与Stream消息驱动模型

![image-20220826184330759](assest/image-20220826184330759.png)

![img](assest/SCSt-overview.png)

# 4 Stream 消息通信方式及编程模型

## 4.1 Stream 消息通信方式

Stream 中的消息通信方式遵循了发布 — 订阅模式。

在 Spring Cloud Stream 中的消息通信方式遵循了发布-订阅模式，当一条消息被投递到消息中间件之后，它会通过共享的 Topic 主题进行广播，消息消费者在订阅的主题中收到它并触发自身的业务逻辑处理。这里所提到的 Topic 主题是 Spring Cloud Stream 中的一个抽象概念，用来代表发布共享消息给消费者的地方。在不同的消息中间件中，Topic 可能对应着不同的概念，比如：在 RabbitMQ 中的它对应了 Exchange 、在 Kafka 中则对应了 Kafka 中的 Topic。

## 4.2 Stream 编程注解

**如下的注解无非在做一件事，把我们结构图中那些组成部分上下关联起来，打通通道（这样的话生产者的 message 数据才能进入 mq，mq中数据才能进入消费者工程）**。

| 注解    | 描述 |
| ------- | ---- |
| @Input  |      |
| @Output |      |
|         |      |
|         |      |



# 5 Stream 高级之自定义消息通道

# 6 Stream 高级之消息分组