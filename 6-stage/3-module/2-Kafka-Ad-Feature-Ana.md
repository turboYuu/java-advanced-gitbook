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

> 1.配置条目的使用方式

![image-20211123140044996](assest/image-20211123140044996.png)

> 2.配置参数：

| 属性                                        | 说明                                                         | 重要性 |
| ------------------------------------------- | ------------------------------------------------------------ | ------ |
| <font color='blue'>bootstrap.servers</font> | 生产者客户端与broker集群建立初始连接需要的broker地址列表，由该初始连接发现kafka集群中其他所有broker。该地址不需要写全部Kafka集群中broker的地址，但也不要写一个，以访该节点宕机的时候不可用。形式为：`host1:port1,host2:port2,...` | high   |
| key.serializer                              | 实现了接口`org.apache.kafka.common.serialization.Serializer`的key序列化类。 | high   |
| value.serializer                            | 实现了接口`org.apache.kafka.common.serialization.Serializer`的value序列化类。 | high   |
| acks                                        | 该选项控制着已发送消息的持久化。<br>`acks=0`：生产者不等待broker的任何消息确认。只要将消息放到socket的缓冲区，就认为消息已发送。不能保证服务器是否收到该消息，`retries`设置也不起作用，因为客户端不关心消息是否发送失败。客户端收到的消息偏移量永远是-1。<br><br>`acks=1`：leader将记录写到它本地日志，就响应客户端确认消息，而不等待follower副本的确认。如果leader确认了消息就宕机，则可能会丢失消息，因为follower副本可能还没来得及同步该消息。<br><br>`acks=all`：leader等待所有同步的副本确认消息。保证了只要由一个副本存在，消息就不会丢失，这是最强的可用性保证。（all指的是***ISR***）等价于`acks=-1`。<br>默认值为1，字符串。可选值：[all,-1,0,1] | high   |
| compression.type                            | 生产者生成数据的压缩格式。默认是none（没有压缩）。允许的值`none`，`gzip`，`snappy`和`lz4`。压缩是对整个消息批次来讲的。消息批的效率也影响压缩比例。消息批越大，压缩效率越好。字符串类型的值。默认是none。 | high   |
| retries                                     | 设置该属性为一个大于1的值，将在消息发送失败的时候重新发送消息。该重试与客户端收到异常重新发送并无二至。允许重试但是不设置`max.in.flight.requests.per.connection`为1，存在消息乱序的可能，因为如果两个批次发送到同一个分区，第一个失败了重试，第二个成功了，则第一个消息批在第二个消息批后。int类型的值，默认：0，可选值：[0,.....,2147483647] | high   |



### 1.1.3 序列化器

![image-20211123143904301](assest/image-20211123143904301.png)

由于Kafka中的数据都是**字节数组**，在将消息发送到kafka之前需要先将数据序列化为字节数组。

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



**系统提供了该接口的子接口以及实现类**：

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

案例：https://gitee.com/turboYuu/kafka-6-3/tree/master/lab/kafka-demos/demo-05-kafka-customSerializer

1. 实体类

2. 自定义序列化类 UserSerializer

3. 生产者中使用

   ![image-20211126171200541](assest/image-20211126171200541.png)

   

### 1.1.4 分区器

![image-20211123151952813](assest/image-20211123151952813.png)

默认（DefaultPartitioner）分区计算：

1. 如果record提供了分区号，则使用record提供的分区号
2. 如果record没有提供分区号，则使用key的序列化后的值的hash值对分区数量取模
3. 如果record没有提供分区号，也没有提供key，则使用轮询的方式分配区号。
   - 会首先在可用的分区中分配分区号
   - 如果没有可用的分区，则在该主题所有分区中分配分区号

![image-20211124105641765](assest/image-20211124105641765.png)

![image-20211124105840767](assest/image-20211124105840767.png)

#### 1.1.4.1 自定义分区器

如果要自定义分区器，则需要

1. 首先开发`org.apache.kafka.clients.producer.Partitioner`接口的实现类
2. 在kafkaProducer中进行设置：config.put("partitioner.class","xxx.xx.Xxx.Class")



位于`org.apache.kafka.clients.producer.Partitioner`中的分区器接口

```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements. See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License. You may obtain a copy of the License at
 *
 *    http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.apache.kafka.clients.producer;

import org.apache.kafka.common.Configurable;
import org.apache.kafka.common.Cluster;

import java.io.Closeable;

/**
 * 分区器接口
 */

public interface Partitioner extends Configurable, Closeable {

    /**
     * 为指定的消息记录计算分区值
     *
     * @param topic 主题名称
     * @param key 根据该key的值进行分区计算，如果没有则为null
     * @param keyBytes key的序列化字节数组，根据该数组进行分区计算。如果没有key，则为null
     * @param value 根据value值进行分区计算，如果没有，则为null
     * @param valueBytes value的序列化字节数组，根据此值进行分区计算。如果没有，则为null
     * @param cluster 当前集群的元数据
     */
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster);

    /**
     * 关闭分区器的时候调用该方法
     */
    public void close();

}
```

包 `org.apache.kafka.clients.producer.internals`中分区器的默认实现：

```java
package org.apache.kafka.clients.producer.internals;

import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import java.util.concurrent.ThreadLocalRandom;
import java.util.concurrent.atomic.AtomicInteger;

import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;
import org.apache.kafka.common.PartitionInfo;
import org.apache.kafka.common.utils.Utils;

/**
 * 默认分区策略:
 * 
 * 如果在记录中指定了分区，则使用指定的分区
 * 如果没有指定分区，但是有key，则使用key值得散列值计算分区
 * 如果没有指定分区也没有key的值，则使用轮询的方式选择一个分区
 */
public class DefaultPartitioner implements Partitioner {

    private final ConcurrentMap<String, AtomicInteger> topicCounterMap = new ConcurrentHashMap<>();

    public void configure(Map<String, ?> configs) {}

    /**
     * 为指定的消息记录分区值
     *
     * @param topic 主题名称
     * @param key 根据该key的值进行分区计算，如果没有则为null
     * @param keyBytes key的序列化字节数组，根据该数据进行分区计算。如果没有key，则为null
     * @param value 根据value值进行分区计算，如果没有，则为null
     * @param valueBytes value的序列化字节数组，根据此值进行分区计算。如果没有，则为null
     * @param cluster 当前集群的元数据
     */
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        // 获取指定主题的所有分区信息
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        // 分区数量
        int numPartitions = partitions.size();
        // 如果没有提供key
        if (keyBytes == null) {
            int nextValue = nextValue(topic);
            List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
            if (availablePartitions.size() > 0) {
                int part = Utils.toPositive(nextValue) % availablePartitions.size();
                return availablePartitions.get(part).partition();
            } else {
                // no partitions are available, give a non-available partition
                return Utils.toPositive(nextValue) % numPartitions;
            }
        } else {
            // hash the keyBytes to choose a partition
            // 如果有，就计算keyBytes的哈希值，然后对当前主题的个数取模
            return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
        }
    }

    private int nextValue(String topic) {
        AtomicInteger counter = topicCounterMap.get(topic);
        if (null == counter) {
            counter = new AtomicInteger(ThreadLocalRandom.current().nextInt());
            AtomicInteger currentCounter = topicCounterMap.putIfAbsent(topic, counter);
            if (currentCounter != null) {
                counter = currentCounter;
            }
        }
        return counter.getAndIncrement();
    }

    public void close() {}

}

```

![image-20211124113242334](assest/image-20211124113242334.png)



案例：https://gitee.com/turboYuu/kafka-6-3/tree/master/lab/kafka-demos/demo-06-kafka-customPartitioner

可以实现`Partitioner`接口自定义分区器：

```java
package com.turbo.kafka.demo.partitioner;

import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;

import java.util.Map;


/**
 * 自定义分区器
 */
public class MyPartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
        // 此处可以计算分区的数字
        // 直接返回2
        return 2;
    }


    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> configs) {

    }
}

```



然后在生产者中配置：

```java
// 指定自定义的分区器
configs.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, MyPartitioner.class);
```



### 1.1.5 拦截器

![image-20211123152018506](assest/image-20211123152018506.png)

Producer拦截器（interceptor）和Cosumer端的Interceptor是在Kafka 0.10版本被引入的，主要用于实现Clienr端的定制化控制逻辑。

对于Producer而言，Interceptor使得用户在**消息发送前**以及**Producer回调逻辑前**有机会对消息做一些定制化需求，比如修改消息等。同时，Producer允许用户指定多个Interceptor按序作用于同一条消息从而形成一个拦截器链（interceptor chain）(1,2,3,4 进入，1,2,3,4出来)。Interceptor的实现接口是`org.apache.kafka.clients.producer.ProducerInterceptor`，其定义的方法包括 ：

- onSend(ProducerRecord<K, V> record)：该方法封装进KafkaProducer.send方法中，即运行在用户主线程中。Producer确保在消息被序列化以计算分区前调用该方法。用户可以在该方法中对消息做任何操作，但最好不要修改消息所属的topic和分区，否则会影响目标分区的计算。
- onAcknowledgement(RecordMetadata, Exception)：该方法会在消息被应答之前或消息发送失败时调用，并且通常都是放在Producer回调逻辑触发之前。onAcknowledgement运行在Producer的IO线程中，因此不要在该方法中放入很重的逻辑，否则会拖慢Producer的消息发送效率。
- close：关闭Interceptor，主要用于执行一些资源清理工作。

如前所述，Interceptor可能被运行在多个线程中，因此在具体实现时，用户需要**自行确保线程安全**。另外倘若指定了多个Interceptor，则Producer将按照指定顺序调用它们，并仅仅是捕获每个Interceptor可能排除的异常记录到错误日志中而非在向上传递。这在使用过程中要特别留意。

#### 1.1.5.1 自定义拦截器

自定义拦截器：

1. 实现`org.apache.kafka.clients.producer.ProducerInterceptor`接口
2. 在KafkaProducer的设置中设置自定义的拦截器



![image-20211124144732078](assest/image-20211124144732078.png)

案例：https://gitee.com/turboYuu/kafka-6-3/tree/master/lab/kafka-demos/demo-07-kafka-customInterceptor

1. 自定义拦截器1 `InterceptorOne`
2. 自定义拦截器2 `InterceptorTwo`
3. 自定义拦截器3 `InterceptorThree`
4. 生产者
5. 执行结果

![image-20211124144821557](assest/image-20211124144821557.png)

## 1.2 原理剖析

![image-20211125123145443](assest/image-20211125123145443.png)

由上图可以看出：KafkaProducer有两个基本线程：

- 主线程：负责消息创建，拦截器，序列化器，分区器等操作，并将消息追加到消息收集器RecordAccumulator中：
  - 消息收集器RecordAccumulator为每个分区维护了一个Deque< ProducerBatch>类型的双端队列。
  - ProducerBatch 可以理解为是 ProducerRecord的集合，批量发送有利于提升吞吐量，降低网络影响。
  - 由于生产者客户端使用java.io.ByteBuffer在发送消息之前进行消息保存，并维护了一个BufferPool实现ByteBuffer的复用；该缓冲池只针对特定大小（batch.size指定）的ByteBuffer进行管理，对于消息过大的缓存，不能做到重复利用。
  - 每次追加一条ProducerRecord消息，会寻找/新建对应的双端队列，从其尾部获取一个ProducerBatch，判断当前消息的大小是否可以写入该批次中。若可以写入则写入；<br>若不可以写入，则新建一个ProducerBatch，判断该消息大小是否超过客户端参数配置batch.size的值，<br>**不超过**，则以batch.size建立新的ProducerBatch，这样方便进行缓存重复利用；<br>**若超过**，则计算消息的大小，建立对应的ProducerBatch，缺点就是该内存不能被复用。
- Sender线程：
  - 该线程从消息收集器获取缓存的消息，将其处理为<Node,List< ProducerBacth>>的形式，Node表示集群的broker节点。
  - 进一步将<Node,List< ProducerBacth>>转化为<Node, Request>形式，此时才可以向服务端发送数据。
  - 在发送之前，Sender线程将消息以Map<NodeId,Deque< Request>>的形式保存到 InFlightRequests 中进行缓存，可以通过其获取leastLoadedNode，即当前Node中负载压力最小的一个，以实现消息的尽快发出。

## 1.3 生产者参数配置补齐

> 1.参数设置方式

![image-20211123140044996](assest/image-20211123140044996.png)

![image-20211124161128223](assest/image-20211124161128223.png)

> 2.补充参数：

| 参数名称                                    | 描述                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| retry.backoff.ms                            | <font style="font-size:92%">在向一个执行的主题分区重发消息的时候，重试之间的等待时间。<br>比如3次重试，每次重试之后等待该时间长度，再接着重试。在一些失败的场景，避免了密集循环的重新发送请求.<br>long类型，默认100，可选值：[0,...]</font> |
| retries                                     | <font style="font-size:92%">retries重试次数。<br>当消息发送出现错误的时候，系统会重发消息。<br>跟客户端收到错误时重发一样。<br>如果设置了重试，还想保证消息的有序性，需要设置MAX_IN_FLIGHT_REQUEST_PER_CONNECTION=1<br>否则在重试此失败消息的时候，其他的消息可能发送成功了。</font> |
| request.timeout.ms                          | <font style="font-size:92%">客户端等待请求响应的最大时长。如果服务端响应超时，则会重发请求，除非达到重试次数。该设置应该比`replica.lag.time.max.ms`(a broker configuration)要大，以免在服务器延迟时间内重发消息。int类型值，默认：3000，可选值：[0,...]</font> |
| interceptor.classes                         | <font style="font-size:92%">在生产者接收到该消息，向Kafka集群传输之前，由序列化处理处理之前，可以通过拦截器对消息进行处理。<br>要求拦截器必须实现`org.apache.kafka.clients.producer.ProducerInterceptor`接口。<br>默认没有拦截器。<br>Map<String,Object> configs中通过List集合配置多个拦截器类名</font> |
| acks                                        | <font style="font-size:92%">当生产者发送消息之后，如何确认消息已经发送成功了。<br>支持的值：<br><br>acks=0：如果设置为0，表示生产者不会等待Broker对消息的确认，只要将消息放到缓冲区，就认为消息已经发送完成。该情形下不能保证broker是否真的收到了消息，retries配置也不会生效，因为客户端不需要知道消息是否发送成功。发送的消息的返回的消息偏移量永远是-1。<br><br>acks=1：表示消息只需要写道主分区即可，然后就响应客户端，而不等待副本分区的确认。<br>在该情形下，如果主分区收到消息确认之后就宕机了，而副本还没来得及同步消息，则该消息丢失。<br><br>acks=all：首领分区等待所有的***ISR***副本分区确认记录。<br>该处理保证了只要有一个ISR副本分区存储，消息就不会丢失。<br>这就是Kafka最强的可靠性保证，等效于`acks=-1`。</font> |
| batch.size                                  | 当多个消息发送到同一个分区的时候，生产者尝试将多个记录作为一个批次来处理。批处理提高了客户端和服务器的处理效率。<br>该配置项以字节为单位控制默认批的大小。<br>所有的批小于等于该值。<br>发送给broker的请求将包含多个批次，每个分区一个，并包含可发送的数据。<br>如果该值设置得比较小，会限制吞吐量（设置为0会完全禁用批处理）。如果设置的很大，又有一点浪费内存，因为Kafka会永远分配这么大的内存来参与到消息的批整合中。 |
| client.id                                   | 生产者发送请求的时候传递给broker的id字符串。<br>用于在broker的请求日志中追踪什么应用发送了什么消息。<br>一般该id是跟业务有关的字符串。 |
| compression.type                            | 生产设发送的所有数据的压缩方式。默认是`none`，也就是不压缩。<br>支持的值：none，gzip，snappy和lz4。<br>压缩是对整个批次来讲的，所以批处理的效率也就会影响到压缩的比例 |
| send.buffer.bytes                           | TCP发送数据的时候使用的缓冲区（SO_SNDBUF）大小。如果设置为0，则使用操作系统默认的。 |
| buffer.memory                               | 生产者可以用来缓存等待发送到服务器的记录的总内存字节。如果记录的发送速度超过了将记录发送到服务器的速度，则生产者将阻塞`max.block.ms`的时间，此后它将引发异常。<br>此设置应大致对应于生产者将使用的总内存，但并非生产者使用的所有内存都用于缓冲。<br>一些额外的内存将用于压缩（如果启用了压缩）以及维护运行中的请求。<br>long类型，默认值：33554432（32M），可选值：[0,...] |
| connections.max.idle.ms                     | 当连接空闲时间达到这个值，就关闭连接。long类型数据，默认：540000 |
| linger.ms                                   | <font style="font-size:92%">生产者在发送请求传输间隔回对需要发送的消息进行累加，然后作为一个批次发送。<br>一般情况是消息的发送速度比消息累积的速度慢。有时客户端需要减少请求的次数，即使在发送负载不大的情况下。<br>该配置设置了一个延迟，生产者不会立即将消息发送到broker，而是等待这么一段时间以累积消息，然后将这段时间之内的消息作为一个批次发送。<br>该设置是批处理的另一个上限：一旦批处理达到`batch.size`指定的值，消息批会立即发送，如果积累的消息字节数达不到`batch.size`的值，可以设置该毫秒值，等待这么长时间之后，也会发送消息批。该属性默认值是0（没有延迟）。如果设置`linger.ms=5`，则在一个请求发送之前先等待5ms。<br>long类型，默认：0，可选值：[0,...]</font> |
| max.block.ms                                | 控制`KafkaProducer.send()`和`KafkaProducer.partitionsFor`阻塞的时长。当缓存满了或元数据不可用的时候，这些方法阻塞。在用户提供的序列化器和分区器的阻塞时间不计入。long类型，默认：60000，可选值：[0,....] |
| max.request.size                            | 单个请求的最大字节数。该设置会限制单个请求中消息批的消息个数，以免单个请求发送太多的数据。服务器有自己的限制批大小的设置，于该配置可能不一样。<br>int类型，默认值：1048576，可选值：[0,....] |
| partitioner.class                           | 实现了接口`org.apache.kafka.clients.producer.Partitioner`的分区实现类。默认值为：`org.apache.kafka.clients.producer.internals.DefaultPartitioner` |
| receive.buffer.bytes                        | TCP接收缓存（SO_RCVBUF），如果设置为-1，则使用操作系统默认的值。<br>int类型，默认值：32768，可选值：[-1,...] |
| <font color='blue'>security.protocol</font> | 跟broker通信协议：PLAINTEXT，SSL，SASL_PLAINTEXT，SASL_SSL<br>String类型，默认：PLAINTEXT |
| max.in.flight.requests.per.connection       | 单个连接上未确认请求的最大数量。达到这个数量，客户端阻塞。如果该值大于1，且存在失败的请求，在重试的时候消息顺序不能保证。<br>int类型，默认5，可选值：[1,....] |
| reconnect.backoff.max.ms                    | 对于每个连续的连接失败，每台主机的退避将成倍增加，直至达到此最大值。在计算退避增量之后，添加20%的随机抖动以避免连接风暴。<br>long型值，默认1000，可选值：[0,...] |
| reconnect.backoff.ms                        | 尝试重连指定主机的基础等待时间。避免了到该主机的密集重连。该瑞比时间应用于该客户端到broker的所有连接。<br>long类型，默认值：50，可选值：[0,...] |



# 2 消费者

## 2.1 概念

### 2.1.1 消费者、消费组

消费者从订阅的主题消费消息，消费消息的偏移量保存在kafka的名字是`_consumer_offsets主题中。`

消费者还可以将自己的偏移量存储到Zookeeper，需要设置`offset.storage=zookeeper`。

推荐使用Kafka存储消费者的偏移量。因为<font color='red'>Zookeeper不适合高并发</font>。



多个从同一个主题消费的消费者可以加入到一个消费组中。

消费组中的消费者共享group_id。

`configs.put(";group.id","xxx");`



group_id一般设置为应用的逻辑名称。比如多个订单处理程序组成一个消费组，可以设置group_id为"order_process"。

group_id通过消费者的配置指定：`group.id=xxx`

消费组均衡地给消费者分配分区，每个分区只能由消费组中的一个消费者消费。



![image-20211125123650766](assest/image-20211125123650766.png)

一个拥有四个分区的主题，包含一个消费者的消费组。

此时，消费组中的消费者消费主题中的所有分区，并且没有重复的可能。



如果在消费组中添加一个消费者2，则每个消费者分别从两个分区接收消息。

![image-20211125124005883](assest/image-20211125124005883.png)



如果消费组有四个消费者，则每个消费者可以分配到一个分区。

![image-20211125124158098](assest/image-20211125124158098.png)



如果向消费组中添加更多的消费者，超过主题分区数量，则有一部分消费者就会闲置，不会接收任何消息。

![image-20211125124354403](assest/image-20211125124354403.png)



向消费组添加消费者是横向扩展消费能力的主要方式。

必要时，需要为主题创建大量分区，在负载增长时可以加入更多的消费者，但是不要让消费者的数量超过主题分区的数量。



![image-20211125124812401](assest/image-20211125124812401.png)

除了通过增加消费者来横向扩展单个应用的消费能力之外，经常出现多个应用程序从同一个主题消费的情况。

此时，每个应用都可以获取到所有的消息。只要保证每个应用都有自己的消费组，就可以让他们获取到主题所有的消息。

横向扩展消费者和消费组不会对性能造成负面影响。



为每个需求获取一个或多个主题全部消息的应用创建一个消费组，然后向消费组添加消费者来横向扩展消费能力和应用的处理能力，则每个消费者只处理一部分消息。

### 2.1.2 心跳机制

![image-20211125125647683](assest/image-20211125125647683.png)

消费者宕机，退出消费组，触发再平衡，重新给消费组中的消费者分配分区。

![image-20211125125856627](assest/image-20211125125856627.png)



由于broker宕机，主题X的分区3宕机，此时分区3没有Leader副本，触发再平衡，消费者4没有对应的主题分区，则消费者4闲置。

![image-20211125132522037](assest/image-20211125132522037.png)



Kafka的心跳是Kafka Consumer 和 Broker之间的健康检查，只有当Broker Coordinator正常时，Consumer才会发送心跳。

Consumer 和 Rebalance 相关的2个配置参数：

| 参数                 | 字段                              |
| -------------------- | --------------------------------- |
| session.timeout.ms   | MemberMetadata.sessionTimeoutMs   |
| max.poll.interval.ms | MemberMetadata.rebalanceTimeoutMs |



> broker端，sessionTimeoutMs参数

broker处理心跳的逻辑在`GroupCoordinator`类中：如果心跳超期，broker coordinate会把消费者从group中移除，并触发rebalance。





> consumer 端：sessionTimeoutMs，rebalanceTimeoutMs参数

如果客户端发现心跳超期，客户端会标记coordinator为不可用，并阻塞心跳线程；如果超过poll消息的间隔超过了rebalanceTimeoutms，则consumer告知broker主动离开消费组，也会触发rebalance。

`org.apache.kafka.clients.consumer.internals.AbstractCoordinator.HeartbeatThread`

```java
if (coordinatorUnknown()) {
    if (findCoordinatorFuture != null || lookupCoordinator().failed())
        // the immediate future check ensures that we backoff properly in the case that no
        // brokers are available to connect to.
        AbstractCoordinator.this.wait(retryBackoffMs);
} else if (heartbeat.sessionTimeoutExpired(now)) {
    // the session timeout has expired without seeing a successful heartbeat, so we should
    // probably make sure the coordinator is still healthy.
    markCoordinatorUnknown();
} else if (heartbeat.pollTimeoutExpired(now)) {
    // the poll timeout has expired, which means that the foreground thread has stalled
    // in between calls to poll(), so we explicitly leave the group.
    maybeLeaveGroup();
} else if (!heartbeat.shouldHeartbeat(now)) {
    // poll again after waiting for the retry backoff in case the heartbeat failed or the
    // coordinator disconnected
    AbstractCoordinator.this.wait(retryBackoffMs);
} else {
    heartbeat.sentHeartbeat(now);

    sendHeartbeatRequest().addListener(new RequestFutureListener<Void>() {
        @Override
        public void onSuccess(Void value) {
            synchronized (AbstractCoordinator.this) {
                heartbeat.receiveHeartbeat(time.milliseconds());
            }
        }

        @Override
        public void onFailure(RuntimeException e) {
            synchronized (AbstractCoordinator.this) {
                if (e instanceof RebalanceInProgressException) {
                    // it is valid to continue heartbeating while the group is rebalancing. This
                    // ensures that the coordinator keeps the member in the group for as long
                    // as the duration of the rebalance timeout. If we stop sending heartbeats,
                    // however, then the session timeout may expire before we can rejoin.
                    heartbeat.receiveHeartbeat(time.milliseconds());
                } else {
                    heartbeat.failHeartbeat();

                    // wake up the thread if it's sleeping to reschedule the heartbeat
                    AbstractCoordinator.this.notify();
                }
            }
        }
    });
}
```



## 2.2 消息接收

### 2.2.1 必要参数配置

| 参数                                        | 说明                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| <font color='blue'>bootstrap.servers</font> | <font style="font-size:93%">向Kafka集群建立初始连接用到host/port列表。<br>客户端使用这里列出的所有服务器进行集群其他服务器的发现，而不管是否指定哪个服务器用作引导。<br>这个列表仅影响用来发现集群所有服务器的初始主机。<br>字符串形式：`host1:port1,host2:port2,...`<br>由于这组服务器仅用于建立初始连接，然后发现集群中的所有服务器，因此没有必要将集群中的所有地址写在这里。一般最好两台，以访其中一台宕机。</font> |
| key.deserializer                            | key的反序列化类，该类需要实现`org.apache.kafka.common.serialization.Deserializer`接口。 |
| value.deserializer                          | <font style="font-size:93%">实现了`org.apache.kafka.common.serialization.Deserializer`的接口的反序列化器，用于对消息的value进行反序列化。</font> |
| client.id                                   | 当从服务器消费消息的时候向服务器发送的id字符串。在ip/port基础上提供应用的逻辑名称，记录在服务端的请求日志中，用于追踪请求的源。 |
| group.id                                    | 用于唯一标示当前消费者所属的消费组的字符串。<br>如果消费者使用管理功能如subscribe(topic)或使用基于kafka的偏移量管理策略，该项必须设置。 |
| auto.offset.reset                           | 当Kafka中没有初始偏移量或当前偏移量在服务器中不存在（如，数据被删除了），该如何处理？<br>earliest：自动重置偏移量到最早的偏移量；<br>latest：自动重置偏移量为最新的偏移量；<br>none：如果消费组原来的（previous）偏移量不存在，则向消费者抛异常；<br>anything：向消费者抛异常。 |
| enable.auto.commit                          | 如果设置为true，消费者会自动周期性的向服务器提交偏移量。     |



### 2.2.2 订阅

#### 2.2.2.1 主题和分区

- **Topic**，Kafka用于分类管理消息的逻辑单元，类似于MySQL分库分表中的逻辑表。
- **Partition**，是Kafka下数据存储的基本单元，这是个物理上的概念。**同一个Topic的数据，会被分散的存储到多个Partition中**，这些Partition可以在同一台机器上，也可以在多台机器上。优势在于：有利于水平扩展，避免单台机器在磁盘空间和性能上的限制，同时可以通过复制来增加数据冗余性，提高容灾能力。为了做到均匀分布，通常Partition的数量是Broker Server数量的整数倍。
- **Consumer Group**，同样是逻辑上的概念，是**Kafka实现单播和广播的两种消息模型的手段**。保证一个消费组获取到特定主题的全部消息。在消费组内部，若干个消费者消费主题分区的消息，消费组可以保证一个主题的每个分区只被消费组中的一个消费者消费。



![image-20211125155405380](assest/image-20211125155405380.png)

consumer采用pull模式从broker中读取数据。

采用pull模式，consumer可自主控制消息的速率，可以自己控制消费方式（批量消费/逐条消费），还可以选择不同的提交方式从而实现不同的传输语义。



### 2.2.3 反序列化

Kafka的broker中所有的消息都是字节数组，消费者获取到消息之后，需要先对消息进行反序列化，然后才能交给用户程序消费处理。

消费者的反序列化器包括key的和value的反序列化器。

> key.deserializer
>
> value.deserializer

需要实现`org.apache.kafka.common.serialization.Deserializer`接口。

消费者从订阅的主题拉取消息：consumer.poll(3_000);

在Fetcher类中，对拉取到的消息首先进行反序列化处理。

![image-20211125170451945](assest/image-20211125170451945.png)

Kafka默认提供了几个反序列化的实现：

`org.apache.kafka.common.serialization.Deserializer`包下包含了这几个实现：

`org.apache.kafka.common.serialization.ByteArrayDeserializer`

`org.apache.kafka.common.serialization.ByteBufferDeserializer`

`org.apache.kafka.common.serialization.BytesDeserializer`

`org.apache.kafka.common.serialization.DoubleDeserializer`

`org.apache.kafka.common.serialization.FloatDeserializer`

`org.apache.kafka.common.serialization.IntegerDeserializer`

`org.apache.kafka.common.serialization.LongDeserializer`

`org.apache.kafka.common.serialization.ShortDeserializer`

`org.apache.kafka.common.serialization.StringDeserializer`



#### 2.2.3.1 自定义反序列化

自定义反序列化器，需要实现`org.apache.kafka.common.serialization.Deserializer`接口。

https://gitee.com/turboYuu/kafka-6-3/tree/master/lab/kafka-demos/demo-08-kafka-customDeserializer

1. 自定义反序列化器`UserDeserializer`
2. 消费中配置反序列化器`config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, UserDeserializer.class);`

### 2.2.4 位移提交

1. Consumer需要向kafka记录自己的位移数据，这个汇报过程称为`提交位移(Committing Offsets)`
2. Consumer需要为分配给它的每个分区提交各自的位移数据
3. 位移提交的由Consumer端负责，Kafka只负责保管。`_consumer_offset`
4. 位移提交分为自动提交和手动提交
5. 位移提交(手动提交)分为同步提交和异步提交

#### 2.2.4.1 自动提交

Kafka Consumer 后台提交

- 开启自动提交：`enable.auto.commit=true`
- 配置自动提交间隔：Consumer端：`auto.commit.interval.ms`，默认5s

```java
Map<String,Object> configs = new HashMap<>();
// node1对应192.168.31.61 ，windows的hosts文件中手动配置域名解析
configs.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,"node1:9092");
// 使用常量代理手写字符串，配置key的反序列化器
configs.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, IntegerDeserializer.class);
// 使用常量代理手写字符串，配置value的反序列化器
configs.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
// 配置消费组ID
configs.put(ConsumerConfig.GROUP_ID_CONFIG,"consumer_demo2");
// 如果找不到消费者有效偏移量，则自动配置到最开始
// latest：表示直接重置到消息偏移量最后一个
configs.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG,"earliest");

// enable.auto.commit 设置自定提交。自动提交是默认值。这里做示例
configs.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG,"true");
// auto.commit.interval.ms 偏移量自动提交的时间间隔
configs.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG,"3000");

KafkaConsumer<Integer,String> consumer = new KafkaConsumer<Integer, String>(configs);

// 先订阅 再消费
consumer.subscribe(Arrays.asList("topic_1"));

while (true){
    // 批量从主题的分区拉取消息
    final ConsumerRecords<Integer, String> consumerRecords = consumer.poll(3_000);

    // 遍历本次从主题的分区拉取的批量消息
    consumerRecords.forEach(new Consumer<ConsumerRecord<Integer, String>>() {
        @Override
        public void accept(ConsumerRecord<Integer, String> record) {
            System.out.println(record.topic() +"\t"
                               + record.partition() +"\t"
                               + record.offset() + "\t"
                               + record.key() + "\t"
                               + record.value() );
        }
    });
}
```



- 自动提交位移的顺序
  - 配置 `enable.auto.commit=true`
  - Kafka会保证在开始调用poll方法时，提交上次poll返回的所有信息
  - 因此自动提交不会出现消息丢失，但会`重复消费`
- 重复消费举例
  - Consumer 每 5s 提交 offset
  - 假设提交 offset 后 3s 发生了 Rebalance
  - Rebalance 之后的所有 Consumer 从上次提交（3s前）的 offset 处继续消费
  - 因此 Rebalance 发生前 3s 的消息会被重复消费



#### 2.2.4.2 同步提交

- 使用KafkaConsumer#commitSync()，会提交KafkaConsumer#poll() 返回的最新 offset

- 该方法为同步操作，等待直到 offset 被成功提交才返回

  ```
  
  ```

- commitSync 在处理完所有消息之后



#### 2.2.4.3 异步提交



### 2.2.5 消费者位移管理

### 2.2.6 再均衡

### 2.2.7 消费者拦截器

### 2.2.8 消费者参数补齐

## 2.3 消费组管理

# 3 主题

# 4 分区

# 5 物理存储

# 6 稳定性

# 7 延时队列

# 8 重试队列