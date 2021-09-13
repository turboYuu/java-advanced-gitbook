第三部分 RocketMQ高级实战

# 1 生产者

## 1.1 Tags的使用

一个应用尽可能用一个Topic，而消息子类型则可以用tags来标识。tags可以由应用自由设置，只有生产者在发送消息设置了tags，消费方在订阅消息时才能利用tags通过broker做消息过滤：

```java
// tag用于标记一类消息
message.setTags("tag1");
```



## 1.2 Keys的使用

每个消息在业务层面的唯一标识码要设置到keys字段，方便将来定位消息丢失问题。服务器会为每个消息创建索引（哈希索引），应用可以通过topic、key来查询消息内容，以及消息被谁消费。由于是哈希索引，请务必保证key尽可能唯一，这样可以避免潜在的哈希冲突。

```java
// key用于建立索引的时候，hash取模将消息的索引放到SlotTable的一个Slot链表中
message.setKeys("ord_2021_11_12");
```



## 1.3 日志的打印

消息发送成功或者失败要打印消息日志，务必要打印SendResult和Key字段。send消息方法只要不抛异常，就代表发送成功。发送成功会有多个状态，在sendResult里定义。

![image-20210913193804194](assest/image-20210913193804194.png)

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

## 1.4 消息发送失败处理方式

Producer的send方法本身支持内部重试，重试逻辑如下：

- 至多重试2次（同步发送为2次，异步发送为0次）。
- 如果发送失败，则轮转到下一个Broker。这个方法的总耗时时间不超过sendMsgTimeout设置的值，默认10s。
- 如果本身向Broker发送消息产生超时异常，就不会再重试。

以上策略也是在一定程度上保证了消息可以发送成功。如果业务对消息可靠性要求比较高，建议应用增加相应的重试逻辑，比如同步send失败，将消息存储到db，由后台线程定时重试。

上述db重试方式没有集成到MQ客户端内部，基于以下几点考虑：

1. MQ的客户端设计为无状态模式，方便任意水平扩展，且对机器资源的消耗仅仅是CPU，内存、网络。
2. 如果MQ客户端内部集成了一个KV存储模块，那么数据只有同步落盘才比较可靠，而同步落盘本身性能开销较大，所以通常会采用异步落盘，又由于应用关闭过程不受MQ运维人员控制，可能经常会发生kill -9 这样暴力方式关闭，造成数据没有即使落盘而丢失。
3. Producer所在机器的可靠性较低，一般为虚拟机，不适合存储重要数据。综上，建议重试过程交由应用来控制。

## 1.5 选择oneway形式发送

通常消息的发送是这样一个过程：

- 客户端发送请求到服务器
- 服务器处理请求
- 服务器向哭护短返回应答

所以，一次消息发送的耗时时间是上述三个步骤的总和，而某些场景要求耗时非常短，但是对可靠性要求不高，可以采用oneway形式调用，**oneway形式只发送请求不等待应答**，而发送请求在客户端实现层面仅仅是一个操作系统调用的开销，**即将数据写入客户端的socket缓冲区**，此过程耗时通常在**微秒级**。

# 2 消费者

## 2.1 消费过程幂等

RocketMQ无法避免消息重复（Exactly-Once），所以如果业务对消费重复非常敏感，务必要在业务层面进行去重处理。

可以借助关系数据库唯一约束等进行处理。

msgId一定是全局唯一标识符，但是实际使用中，可能会存在相同消息有两个不同msgId的情况（客户端重投机制导致等），这种情况就需要业务层面处理。

## 2.2 消费速度慢的处理方式

**提高消费并行度**

绝大部分消费行为都属于IO密集型，即可能是操作数据库，或者调用RPC，这类消费行为消费速度在于后端数据库或者外系统的吞吐量。

通过增加消费并行度，可以提高消费吞吐量，但是并行度增加到一定程度，**反而会下降**。

所以，应用必须要设置合理的并行度，如下有几种修改消费并行度的方法：

- 同一个ConsumerGroup下，通过增加Consumer实例数量来提高并行度（需要注意的是超过订阅队列数的Consumer实例无效）。可以通过增加机器，或者在已有机器启动多个进程方式。
- 提高单个Consumer的消费并行线程，通过修改参数consumeThreadMin、consumeThreadMax实现。
- 丢弃部分不重要的消息

**批量方式消费**

业务如果支持批量消费，则可以很大程度上提高消费吞吐量。

**跳过非重要消息**

如果消费速度一直追不上发送速度，如果业务对数据要求不高的话，可以选择丢弃不重要的消息。

```java
public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, 
	ConsumeConcurrentlyContext context) {
   long offset = msgs.get(0).getQueueOffset();    
   String maxOffset = msgs.get(0).getProperty(Message.PROPERTY_MAX_OFFSET);    
   long diff = Long.parseLong(maxOffset) - offset;    
   if (diff > 100000) {
       // TODO 消息堆积情况的特殊处理
       return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;  
    }
   	// TODO 正常消费过程
   	return ConsumeConcurrentlyStatus.CONSUME_SUCCESS; 
} 
```



## 2.3 优化每条消息消费过程

减少与DB的交互，可以把DB部署到SSD硬盘。

## 2.4 消费打印日志

如果消息量较少，建议在消费入口方法打印消息，消费耗时等，方便后续排查问题。

```java
public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, 
	ConsumeConcurrentlyContext context) {
   log.info("RECEIVE_MSG_BEGIN: " + msgs.toString());    
   // TODO 正常消费过程
   return ConsumeConcurrentlyStatus.CONSUME_SUCCESS; 
}
```

如果能打印每条消息消费时间，那么在排查消费慢等线上问题时，会更方便。

## 2.5 其他消费建议

1. 关于消费者和订阅

   第一件需要注意的是，不同消费组可以独立消费一些topic，并且每个消费组都有自己的消费偏移量。

   确保同一组内的**每个消费者订阅信息保持一致**。

2. 关于有序消息

   消费者将锁定每个消费队列，以确保它们被逐个消费，虽然这将会导致性能下降，但是当关心消息顺序的时候会非常有用。

   不建议抛出异常，可以返回ConsumeOrderlyStatus.SUSPEND_CURRENT_QUEUE_A_MOMENT作为代替。

3. 关于并发消费

   消费者将并发消费这些消息，建议使用它来获得良好性能，不建议抛出异常，可以返回ConsumeConcurrentlyStatus.RECONSUME_LATER作为代替。

4. 关于消费状态

   对于并发的消费监听，可以返回RECONSUME_LATER来通知broker现在不能消费这条消息，并且希望可以稍后重新消费它。然后，可以继续消费其他消息。对于有序的消息监听器，因为关系它的顺序，所以不能跳过消息，但是可以返回SUSPEND_CURRENT_QUEUE_A_MOMENT告诉broker等待片刻。

5. 关于Blocking

   不建议阻塞监听器，因为它会阻塞线程池，并且最终可能会终止消费进程

6. 关于线程数设置

   消费者使用ThreadPoolExecutor在内部对消息进行消费，所以可以通过设置setConsumeThreadMin或setConsumeThreadMax来改变它。

7. 关于消费位点

   当建立一个新的消费组时，需要决定是否需要消费已经存在与Broker中的历史消息。

   `CONSUME_FROM_LAST_OFFSET`将忽略历史消息，并消费之后生成的任何消息

   `CONSUME_FROM_FIRST_OFFSET`将消费每个存在于Broker中的信息

   也可以使用`CONSUME_FROM_TIMESTAMP`来消费在指定时间戳后产生的消息。

   ```java
   public static void main(String[] args) throws MQClientException {    
   	DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("consumer_grp_15_01");    				consumer.setNamesrvAddr("node1:9886");    
   	consumer.subscribe("tp_demo_15", "*");
      	// 以下三个选一个使用，如果是根据时间戳进行消费，则需要设置时间戳   
      
      	// 从第一个消息开始消费，从头开始
    	consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);    
    	// 从最后一个消息开始消费，不消费历史消息
   	consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);    
   	
   	// 从指定的时间戳开始消费
      consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_TIMESTAMP);    
      // 指定时间戳的值
      consumer.setConsumeTimestamp("");
      consumer.setMessageListener(new MessageListenerConcurrently() {        
      		@Override
          	public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, 
          			ConsumeConcurrentlyContext context) {
              	// TODO 处理消息的业务逻辑
              	return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
   //                return ConsumeConcurrentlyStatus.RECONSUME_LATER;
        	}    
   	});
      	consumer.start(); 
   }
   ```

# 3 Broker

## 3.1 Broker角色

Broker角色分为ASYNC_MASTER（异步主机），SYNC_MASTER（同步主机）以及SLAVE（从机）。如果对消息的可靠性要求比较严格，可以采用SYNC_MASTER + SLAVE的部署方式，如果对消息可靠性要求不高，可以采用AYSNC_MASTER + SLAVE的部署方式。如果只是测试方便可以选择仅ASYNC_MASTER或仅SYNC_MASTER的部署方式。

## 3.2 FlushDiskType

SYNC_FLUSH（同步刷盘）相比于ASYNC_FLUSH（异步处理）会损失很多性能，但是也更可靠，所以需要根据实际的业务场景做好权衡。

## 3.3 Broker配置

| 参数名                  | 默认值                    | 说明                                                         |
| ----------------------- | ------------------------- | ------------------------------------------------------------ |
| listenPort              | 10911                     | 接收客户端连接的监听端口                                     |
| namesrvAddr             | null                      | nameServer地址                                               |
| brokerIP1               | 网卡的InteAddress         | 当前broker监听的IP                                           |
| brokerIP2               | 跟 brokerIP1 一样         | 存在主从 broker 时，<br>如果在 broker 主节点上配置 了 brokerIP2 属性，<br>broker 从节点会连接主节点配置的 brokerIP2进行同步 |
| brokerName              | null                      | broker 的名称                                                |
| brokerClusterName       | DefaultCluster            | 本  broker 所属的  Cluser 名称                               |
| brokerId                | 0                         | broker id, 0 表示  master, 其他的正整数表示  slave           |
| storePathCommitLog      | $HOME/store/commitlog/    | 存储  commit log 的路径                                      |
| storePathConsumerQueue  | $HOME/store/consumequeue/ | 存储  consume queue 的路径                                   |
| mappedFileSizeCommitLog | 1024 * 1024 * 1024(1G)    | commit log 的映射文件大小                                    |
| deleteWhen              | 04                        | 在每天的什么时间删除已经超过文件保留时间的commit log         |
| fileReservedTime        | 72                        | 以小时计算的文件保留时间                                     |
| brokerRole              | ASYNC_MASTER              | SYNC_MASTER/ASYNC_MASTER/SLAVE                               |
| flushDiskType           | ASYNC_FLUSH               | SYNC_FLUSH/ASYNC_FLUSH <br>SYNC_FLUSH 模式下的broker保证在收到确认生产者之前将消息刷盘。<br>ASYNC_FLUSH模式下的broker则异步刷盘一组消息的模式，可以取得更好的性能。 |



# 4 NameServer

![image-20210913102818808](assest/image-20210913102818808.png)

NameServer的设计：

1. NameServer互相独立

# 5 客户端配置

-Drocketmq.namesrv.addr=192.168.31.101:9876

![image-20210913111521309](assest/image-20210913111521309.png)

![image-20210913111620092](assest/image-20210913111620092.png)







![image-20210913112829326](assest/image-20210913112829326.png)



HTTP静态服务器寻址（默认）



![image-20210913125839562](assest/image-20210913125839562.png)

![image-20210913130403387](assest/image-20210913130403387.png)

![image-20210913125627540](assest/image-20210913125627540.png)



![image-20210913130313212](assest/image-20210913130313212.png)

## 5.1 客户端寻址方式

## 5.2 客户端的公共配置

## 5.3 Producer配置

## 5.4 PushConsumer配置

## 5.5 PullConsumer配置

## 5.6 Message数据结构

# 6 系统配置

## 6.1 JVM选项

## 6.2 Linux内核参数

# 7 动态扩缩容

## 7.1 动态增减Namesrv机器

```
mqbroker -n 'node1:9876;node2:9876'
```



![image-20210913145003090](assest/image-20210913145003090.png)

## 7.2 动态增减Broker机器

```shell
[root@node2 ~]# mqadmin topicStatus -n node2:9876 -t tp_demo_09

[root@node2 ~]# mqadmin updateTopic -b node2:10911 -n node2:9876 -r 4 -t tp_demo_09 -w 4

[root@node2 ~]# mqadmin topicStatus -n node2:9876 -t tp_demo_09
```

![image-20210913152021383](assest/image-20210913152021383.png)

![image-20210913151804066](assest/image-20210913151804066.png)

![image-20210913151846465](assest/image-20210913151846465.png)





减少Broker





# 8 各种故障对消息的影响

