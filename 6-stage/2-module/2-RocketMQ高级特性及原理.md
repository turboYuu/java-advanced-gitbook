第二部分 RocketMQ高级特性及原理

# 1 消息发送

生产者向队列里写入消息，不同的业务场景需要生产者采用不同的写入策略。比如同步发送、异步发送、Oneway发送、延迟发送、发送事务消息等。默认使用的是DefaultMQProducer类，发送消息要经过五个步骤：

> 1. 设置producer的GroupName。
> 2. 设置InstanceName，当一个Jvm需要启动多个Producer的时候，设置不同的InstanceName来区分，不设置的话系统使用默认名称"DEFAULT"。
> 3. 设置发送失败重试次数，当网络出现异常的时候，这个次数影响消息的重复投递次数。像保证不丢失消息，可以设置多重试几次。
> 4. 设置NameServer地址。
> 5. 组装消息并发送。

消息发送返回状态（SendResult#SendStatus）有如下四种：

每个状态进行说明：

- **SEND_OK**

  消息发送成功。要注意的是消息发送成功也不意味着它是可靠的。要确保不会丢失任何消息，，还应启用同步Master服务器或同步刷盘，即SYNC_MASTER或SYNC_FLUSH。

- **FLUSH_DISK_TIMEOUT**

  消息发送成功但是服务器刷盘超时。此时消息已经进入服务器队列（内存），只有服务器宕机，消息才会丢失。消息存储配置参数中可以设置刷盘方式和同步刷盘时间长度。

  如果Broker服务器设置了刷盘方式为同步刷盘，即FlushDiskType=**SYNC_FLUSH**（默认为异步刷盘方式），当Broker服务器为在同步刷盘时间内（**默认5s**）完成刷盘，则将返回该状态——刷盘超时。

- **FLUSH_SLAVE_TIMEOUT**

  消息发送成功，但是服务器同步到Slave超时。此时消息已经进入服务器队列，只有服务器宕机，消息才会丢失。

  如果Broker服务器的角色是同步Master，即SYNC_MASTER（默认是异步Master即ASYNC_MASTER），并且从Broker服务器未在同步刷盘时间（默认5s）内完成与主服务器的同步，则将返回该状态——数据同步到Slave服务器超时。

- **SLAVE_NOT_AVAILABLE**

  消息发送成功，但此时Slave不可用。

  如果Broker服务器角色是同步Master，即**SYNC_MASTER**（默认是ASYNC_MASTER），但是没有配置slave broker服务器，则将返回该状态——无slave服务器可用。

![image-20210913193804194](assest/image-20210913193804194.png)



# 2 消息消费

# 3 消息存储

# 4 过滤消息

![image-20210911185236958](assest/image-20210911185236958.png)

# 5 零拷贝原理

# 6 同步复制和异步复制

# 7 高可用机制

# 8 刷盘机制

# 9 负载均衡



```
mqbroker -n localhost:9876 -c /opt/rocket/conf/broker.conf
```



```
mqadmin updateTopic -n localhost:9876 -b localhost:10911 -t tp_demo_06 -w 6
```



![image-20210911204410922](assest/image-20210911204410922.png)

```
mqadmin topicList -n localhost:9876
```

![image-20210911204815706](assest/image-20210911204815706.png)



# 10 消息重试

## 10.1 顺序消息的重试

## 10.2 无序消息的重试

### 10.2.1 重试次数

### 10.2.2 配置方式

# 11 死信队列

https://github.com/apache/rocketmq-externals/archive/rocketmq-console-1.0.0.zip

![image-20210912154323541](assest/image-20210912154323541.png)

![image-20210912154236524](assest/image-20210912154236524.png)



![image-20210912155817316](assest/image-20210912155817316.png)



![image-20210912160130031](assest/image-20210912160130031.png)



# 12 延迟消息

![image-20210912162424025](assest/image-20210912162424025.png)

# 13 顺序消息

## 13.1 部分有序

```
[root@node1 ~]# mqadmin updateTopic -b node1:10911 -n localhost:9876 -r 8 -t tp_demo_11 -w 8

[root@node1 ~]# mqadmin topicStatus -n localhost:9876 -t tp_demo_11

```

![image-20210912163952113](assest/image-20210912163952113.png)



## 13.2 全局有序

```
[root@node1 ~]# mqadmin updateTopic -b node1:10911 -n localhost:9876 -r 1 -t tp_demo_11_01 -w 1

[root@node1 ~]# mqadmin topicStatus -n localhost:9876 -t tp_demo_11_01
```



![image-20210912170322280](assest/image-20210912170322280.png)





# 14 事务消息

# 15 消息查询

区别于消息消费	



# 16 消息优先级

# 17 底层网络通信 - Netty高性能之道

# 18 限流

https://github.com/alibaba/Sentinel/wiki/Sentinel-%E4%B8%BA-RocketMQ-%E4%BF%9D%E9%A9%BE%E6%8A%A4%E8%88%AA






