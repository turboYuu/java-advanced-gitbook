第二部分 Kafka高级特性解析

# 1 生产者

## 1.1 消息发送

### 1.1.1 数据生成流程解析

![image-20211123134418311](assest/image-20211123134418311.png)

1. Producer创建时，会创建一个Sender线程并设置为守护线程
2. 生产消息时，内部其实是异步流程；生产的消息先经过 拦截器 -> 序列化器 -> 分区器，然后消息缓存在缓冲区（该缓冲区也是在Producer创建时创建）。
3. 批次发送的条件为：缓冲区数据大小达到batch.size或者linger.ms 上限，哪个先达到就算哪个。
4. 批次发送后，发往指定分区，然后落盘到broker；如果生产者配置了retries参数大于0并且失败原因允许重试，那么客户端内部会对该消息进行重试。
5. 落盘到broker成功，返回生产元数据给生产者。
6. 元数据返回有两种方式：一种通过阻塞直接返回，另一种通过回调返回。



### 1.1.2 必要参数配置

1. 配置条目的使用方式

   ![image-20211123140044996](assest/image-20211123140044996.png)

2. 配置参数：

   | 属性                                        | 说明                                                         | 重要性 |
   | ------------------------------------------- | ------------------------------------------------------------ | ------ |
   | <font color='blue'>bootstrap.servers</font> | 生产者客户端与broker集群建立初始连接需要的broker地址列表，由该初始连接发现kafka集群中其他所有broker。该地址不需要写全部Kafka集群中broker的地址，但也不要写一个，以访该节点宕机的时候不可用。形式为：`host1:port1,host2:port2,...` | high   |
   | key.serializer                              | 实现了接口`org.apache.kafka.common.serialization.IntegerSerializer`的key序列化类。 |        |
   | value.serializer                            | 实现了接口`org.apache.kafka.common.serialization.StringSerializer`的value序列化类。 |        |
   | acks                                        | 该选项控制着以发送消息的持久化。<br>`acks=0`：生产者不等待broker的任何消息确认。只要将消息放到socket的缓冲区，就认为消息已发送。不能保证服务器是否收到该消息，`retries`设置也不起作用，因为客户端不关心消息是否发送失败。客户端收到的消息偏移量永远是-1。<br><br>`acks=1`：leader将记录写到它本地日志，就响应客户端确认消息，而不等待follower副本的确认。如果leader确认了消息就宕机，则可能会丢失消息，因为follower副本可能还没来得及同步该消息。<br><br>`acks=all`：leader等待所有同步的副本 |        |

   

#### 1.1.2.1 broker配置

### 1.1.3 序列化器

#### 1.1.3.1 自定义序列化器

### 1.1.4 分区器

### 1.1.5 拦截器

## 1.2 原理剖析

## 1.3 生产者参数配置补齐



# 2 消费者

# 3 主题

# 4 分区

# 5 物理存储

# 6 稳定性

# 7 延时队列

# 8 重试队列