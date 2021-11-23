第二部分 Kafka高级特性解析

# 1 生产者

## 1.1 消息发送

### 1.1.1 数据生成流程解析

![image-20211123134418311](assest/image-20211123134418311.png)

1. Producer创建时，会创建一个Sender线程并设置为守护线程
2. 生产消息时，内部其实是异步流程；生产的消息先经过 拦截器 -> 序列化器 -> 分区器，然后消息缓存在缓冲区（该缓冲区也是在Producer创建时创建）。
3. 批次发送的条件为：缓冲区数据大小达到batch.size或者linger.ms(等待时间) 上限，哪个先达到就算哪个。
4. 批次发送后，发往指定分区，然后落盘到broker；如果生产者配置了retries参数大于0并且失败原因允许重试，那么客户端内部会对该消息进行重试。
5. 落盘到broker成功，返回生产元数据给生产者。
6. 元数据返回有两种方式：一种通过阻塞直接返回，另一种通过回调返回。



### 1.1.2 必要参数配置

#### 1.1.2.1 broker配置

1. 配置条目的使用方式

   ![image-20211123140044996](assest/image-20211123140044996.png)

2. 配置参数：

   | 属性                                        | 说明                                                         | 重要性 |
   | ------------------------------------------- | ------------------------------------------------------------ | ------ |
   | <font color='blue'>bootstrap.servers</font> | 生产者客户端与broker集群建立初始连接需要的broker地址列表，由该初始连接发现kafka集群中其他所有broker。该地址不需要写全部Kafka集群中broker的地址，但也不要写一个，以访该节点宕机的时候不可用。形式为：`host1:port1,host2:port2,...` | high   |
   | key.serializer                              | 实现了接口`org.apache.kafka.common.serialization.Serializer`的key序列化类。 | high   |
   | value.serializer                            | 实现了接口`org.apache.kafka.common.serialization.Serializer`的value序列化类。 | high   |
   | acks                                        | 该选项控制着已发送消息的持久化。<br>`acks=0`：生产者不等待broker的任何消息确认。只要将消息放到socket的缓冲区，就认为消息已发送。不能保证服务器是否收到该消息，`retries`设置也不起作用，因为客户端不关心消息是否发送失败。客户端收到的消息偏移量永远是-1。<br><br>`acks=1`：leader将记录写到它本地日志，就响应客户端确认消息，而不等待follower副本的确认。如果leader确认了消息就宕机，则可能会丢失消息，因为follower副本可能还没来得及同步该消息。<br><br>`acks=all`：leader等待所有同步的副本确认消息。保证了只要由一个副本存在，消息就不会丢失，这是最强的可用性保证。等价于`acks=-1`。<br>默认值为1，字符串。可选值：[all,-1,0,1] | high   |
   | compression.type                            | 生产者生成数据的压缩格式。默认是none（没有压缩）。允许的值`none`，`gzip`，`snappy`和`lz4`。压缩是对整个消息批次来讲的。消息批的效率也影响压缩比例。消息批越大，压缩效率越好。字符串类型的值。默认是none。 | high   |
   | retries                                     | 设置该属性为一个大于1的值，将在消息发送失败的时候重新发送消息。该重试与客户端收到异常重新发送并无二至。允许重试但是不设置`max.in.flight.requests.per.connection`为1，存在消息乱序的可能，因为如果两个批次发送到同一个分区，第一个失败了重试，第二个成功了，则第一个消息批在第二个消息批后。int类型的值，默认：0，可选值：[0,.....,2147483647] | high   |

   

### 1.1.3 序列化器

![image-20211123143904301](assest/image-20211123143904301.png)

由于Kafka中的数据都是字节数组，在将消息发送到kafka之前需要先将数据序列化为字节数组。

序列化器的作用就是用于序列化要发送的消息的。



Kafka使用`org.apache.kafka.common.serialization.Serializer`接口用于定义序列化器，将泛型指定类型的数据转换为字节数组。

```java
package org.apache.kafka.common.serialization;

import java.io.Closeable;
import java.util.Map;

/**
 * 将对象转换为byte数组的接口
 * 该接口的实现类需要提供无参构造器
 * @param <T> 从哪个类型转换
 */
public interface Serializer<T> extends Closeable {

    /**
     * 类的配置信息.
     * @param configs configs in key/value pairs
     * @param isKey key的序列化还是value得序列化
     */
    void configure(Map<String, ?> configs, boolean isKey);

    /**
     * 将对象转换为字节数组
     * @param topic 主题名称
     * @param data 需要转换得对象
     * @return 序列化的字节数组
     */
    byte[] serialize(String topic, T data);

    /**
     * 关闭序列化
     * 该方法需要提供幂等性，因为可能调用多次
       This method must be idempotent as it may be called multiple times.
     */
    @Override
    void close();
}

```



系统提供了该接口的子接口以及实现类：

`org.apache.kafka.common.serialization.ByteArraySerializer`

![image-20211123150656186](assest/image-20211123150656186.png)



`org.apache.kafka.common.serialization.ByteBufferSerializer`

![image-20211123150908992](assest/image-20211123150908992.png)



`org.apache.kafka.common.serialization.BytesSerializer`

![image-20211123151047664](assest/image-20211123151047664.png)



`org.apache.kafka.common.serialization.DoubleSerializer`

![image-20211123151157246](assest/image-20211123151157246.png)



`org.apache.kafka.common.serialization.FloatSerializer`

![image-20211123151305544](assest/image-20211123151305544.png)



`org.apache.kafka.common.serialization.IntegerSerializer`

![image-20211123151356447](assest/image-20211123151356447.png)



`org.apache.kafka.common.serialization.StringSerializer`

![image-20211123151502742](assest/image-20211123151502742.png)



`org.apache.kafka.common.serialization.LongSerializer`

![image-20211123151618534](assest/image-20211123151618534.png)



`org.apache.kafka.common.serialization.ShortSerializer`

![image-20211123151703157](assest/image-20211123151703157.png)







#### 1.1.3.1 自定义序列化器

数据的序列化一般生产中使用avro。

自定义序列化器需要实现`org.apache.kafka.common.serialization.Serializer`接口，并实现其中的`serialize`方法。



案例：



### 1.1.4 分区器

![image-20211123151952813](assest/image-20211123151952813.png)

### 1.1.5 拦截器

![image-20211123152018506](assest/image-20211123152018506.png)

## 1.2 原理剖析

## 1.3 生产者参数配置补齐



# 2 消费者

# 3 主题

# 4 分区

# 5 物理存储

# 6 稳定性

# 7 延时队列

# 8 重试队列