第二部分 RocketMQ高级特性及原理

# 1 消息发送

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

# 12 延迟消息

# 13 顺序消息

# 14 事务消息

# 15 消息查询

# 16 消息优先级

# 17 底层网络通信 - Netty高性能之道

# 18 限流






