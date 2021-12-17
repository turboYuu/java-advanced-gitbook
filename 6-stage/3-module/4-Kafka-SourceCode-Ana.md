第四部分 Kafka源码剖析

# 1 源码阅读环境搭建

首先下载源码：https://archive.apache.org/dist/kafka/1.0.2/kafka-1.0.2-src.tgz

gradle-4.8.1 下载地址：https://services.gradle.org/distributions/gradle-4.8.1-bin.zip

Scala-2.12.12 下载地址：https://downloads.lightbend.com/scala/2.12.12/scala-2.12.12.msi

## 1.1 安装Gradle

![image-20211215134711958](assest/image-20211215134711958.png)

![image-20211215134757138](assest/image-20211215134757138.png)

![image-20211215134851449](assest/image-20211215134851449.png)

进入GRADLE_USER_HOME目录，添加init.gradle，配置gradle的源：

init.gradle内容

```json
allprojects {
    repositories {
        maven { url 'https://maven.aliyun.com/repository/public/' }
        maven { url 'https://maven.aliyun.com/nexus/content/repositories/google' }
        maven { url 'https://maven.aliyun.com/nexus/content/groups/public/' }
        maven { url 'https://maven.aliyun.com/nexus/content/repositories/jcenter'}

        all { ArtifactRepository repo ->
            if (repo instanceof MavenArtifactRepository) {
                def url = repo.url.toString()

                if (url.startsWith('https://repo.maven.apache.org/maven2/') || url.startsWith('https://repo.maven.org/maven2') || url.startsWith('https://repo1.maven.org/maven2') || url.startsWith('https://jcenter.bintray.com/')) {
                    //project.logger.lifecycle "Repository ${repo.url} replaced by $REPOSITORY_URL."
                    remove repo
                }
            }
        }
    }

    buildscript {

        repositories {

            maven { url 'https://maven.aliyun.com/repository/public/'}
            maven { url 'https://maven.aliyun.com/nexus/content/repositories/google' }
            maven { url 'https://maven.aliyun.com/nexus/content/groups/public/' }
            maven { url 'https://maven.aliyun.com/nexus/content/repositories/jcenter'}
            all { ArtifactRepository repo ->
                if (repo instanceof MavenArtifactRepository) {
                    def url = repo.url.toString()
                    if (url.startsWith('https://repo1.maven.org/maven2') || url.startsWith('https://jcenter.bintray.com/')) {
                        //project.logger.lifecycle "Repository ${repo.url} replaced by $REPOSITORY_URL."
                        remove repo
                    }
                }
            }
        }
    }
}
```

保存并退出，打开cmd，运行：

![image-20211215135356914](assest/image-20211215135356914.png)

设置成功。



## 1.2 Scala安装和配置

双击安装

![image-20211215135440537](assest/image-20211215135440537.png)

![image-20211215135723903](assest/image-20211215135723903.png)

![image-20211215135746718](assest/image-20211215135746718.png)

![image-20211215135849395](assest/image-20211215135849395.png)

![image-20211215135927038](assest/image-20211215135927038.png)

![image-20211215140022055](assest/image-20211215140022055.png)

![image-20211215140221777](assest/image-20211215140221777.png)

![image-20211215140354376](assest/image-20211215140354376.png)

![image-20211215140529307](assest/image-20211215140529307.png)

## 1.3 Idea配置

![image-20211215142015876](assest/image-20211215142015876.png)

## 1.4 源码操作

解压源码

打开cmd ，进入源码根目录，执行：gradle

![image-20211215142412360](assest/image-20211215142412360.png)

结束后，执行 gradle idea（注意不要使用生成的gradlew.bat执行操作）。

![image-20211215143121837](assest/image-20211215143121837.png)

idea导入源码：

![image-20211215143516186](assest/image-20211215143516186.png)

![image-20211215143612610](assest/image-20211215143612610.png)

在Idea中配置gradle user home和gradle home，否则会重新开始下载gradle -4.8。

# 2 Broker启动流程



# 3 Topic创建流程

## 3.1 Topic创建

有两种创建方式：自动创建、手动创建。在server.properties中配置`auto.create.topics.enable=true`时，Kafka在发现该topic不存在的时候会按照默认配置自动创建topic，触发自动创建topic有以下两种情况：

1. Producer向某个不存在的Topic写入消息
2. Consumer从某个不存在的Topic读取消息

## 3.2 手动创建

当`auto.create.topics.enable=false`时，需要手动创建topic，否则消息会发送失败。手动创建topic的方式如下：

```shell
kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 10 --topic kafka_test
```

--replication-factor：副本数目

--partitions：分区数

--topic：topic名字

## 3.3 查看Topic入口

查看脚本文件`kafka-topics.sh`

```sh
exec $(dirname $0)/kafka-run-class.sh kafka.admin.TopicCommand "$@"
```

最终还是调用的`TopicCommand`类：首先判断参数是否为空，并且create、list、alter、describe、delete只允许存在一个，进行参数验证，创建`zookeeper`连接，如果参数中包含`create`则开始创建topic，其他情况类似。

```scala
// kafka.admin.TopicCommand#main
def main(args: Array[String]): Unit = {

    // 负责解析参数
    val opts = new TopicCommandOptions(args)

    if(args.length == 0)
      CommandLineUtils.printUsageAndDie(opts.parser, "Create, delete, describe, or change a topic.")

    // should have exactly one action
    val actions = Seq(opts.createOpt, opts.listOpt, opts.alterOpt, opts.describeOpt, opts.deleteOpt).count(opts.options.has _)
    if(actions != 1)
      CommandLineUtils.printUsageAndDie(opts.parser, "Command must include exactly one action: --list, --describe, --create, --alter or --delete")

    opts.checkArgs()

    val zkUtils = ZkUtils(opts.options.valueOf(opts.zkConnectOpt),
                          30000,
                          30000,
                          JaasUtils.isZkSecurityEnabled())
    var exitCode = 0
    try {
      if(opts.options.has(opts.createOpt))
        // 创建topic
        createTopic(zkUtils, opts)
      else if(opts.options.has(opts.alterOpt))
        alterTopic(zkUtils, opts)
      else if(opts.options.has(opts.listOpt))
        listTopics(zkUtils, opts)
      else if(opts.options.has(opts.describeOpt))
        describeTopic(zkUtils, opts)
      else if(opts.options.has(opts.deleteOpt))
        deleteTopic(zkUtils, opts)
    } catch {
      case e: Throwable =>
        println("Error while executing topic command : " + e.getMessage)
        error(Utils.stackTrace(e))
        exitCode = 1
    } finally {
      zkUtils.close()
      Exit.exit(exitCode)
    }

  }
```



## 3.4 创建Topic

下面主要看一下`createTopic`的执行过程：

```scala
// kafka.admin.TopicCommand#createTopic
def createTopic(zkUtils: ZkUtils, opts: TopicCommandOptions) {
    // 获取参数中指定的主题名称
    val topic = opts.options.valueOf(opts.topicOpt)
    // 获取参数中给当前要创建的主题指定的参数 --config max.message.bytes = 1048576 --config segment.bytes=104857600
    val configs = parseTopicConfigsToBeAdded(opts)
    // --if-not-exists选项，判断有无该选项
    val ifNotExists = opts.options.has(opts.ifNotExistsOpt)
    if (Topic.hasCollisionChars(topic))
      println("WARNING: Due to limitations in metric names, topics with a period ('.') or underscore ('_') could collide. To avoid issues it is best to use either, but not both.")
    try {
      // 看有没有手动指定当前要创建的主题副本分区的分配
      if (opts.options.has(opts.replicaAssignmentOpt)) {
        // 解析用户手动指定的副本分区分配，
        val assignment = parseReplicaAssignment(opts.options.valueOf(opts.replicaAssignmentOpt))
        // 将该副本分区的分配写到zookeeper中
        AdminUtils.createOrUpdateTopicPartitionAssignmentPathInZK(zkUtils, topic, assignment, configs, update = false)
      } else {
        // 检查有没有指定必要的参数：解析器，选项，分区个数，副本因子
        CommandLineUtils.checkRequiredArgs(opts.parser, opts.options, opts.partitionsOpt, opts.replicationFactorOpt)
        // 从选项中获取分区个数
        val partitions = opts.options.valueOf(opts.partitionsOpt).intValue
        // 获取分区副本因子
        val replicas = opts.options.valueOf(opts.replicationFactorOpt).intValue
        // 是否机架敏感的
        val rackAwareMode = if (opts.options.has(opts.disableRackAware)) RackAwareMode.Disabled
                            else RackAwareMode.Enforced
        // 创建主题
        AdminUtils.createTopic(zkUtils, topic, partitions, replicas, configs, rackAwareMode)
      }
      println("Created topic \"%s\".".format(topic))
    } catch  {
      case e: TopicExistsException => if (!ifNotExists) throw e
    }
  }
```

1. 如果客户端指定了topic的partition的replicas分配情况，则直接把所有topic的元数据信息持久化写入zookeeper，topic的properties写入到`/<Kafka_ROOT>/config/topics/[topic]`目录，topic的PartitionAssignment写入`<kafka_root>/brokers/topics/[topic]/partitions/[partition]/state`目录。

2. 根据分区数量、副本集、是否指定机架来自动生成topic分区数据

3. 下面继续来看`AdminUtils.createTopic`方法

   ```scala
   def createTopic(zkUtils: ZkUtils,
                     topic: String,
                     partitions: Int,
                     replicationFactor: Int,
                     topicConfig: Properties = new Properties,
                     rackAwareMode: RackAwareMode = RackAwareMode.Enforced) {
       // 获取broker元数据
       val brokerMetadatas = getBrokerMetadatas(zkUtils, rackAwareMode)
       // 将副本分区分配给broker
       val replicaAssignment = AdminUtils.assignReplicasToBrokers(brokerMetadatas, partitions, replicationFactor)
       // 将主题的分区分配情况xi写到zookeeper中
       AdminUtils.createOrUpdateTopicPartitionAssignmentPathInZK(zkUtils, topic, replicaAssignment, topicConfig)
       // 到此创建主题的过程结束
     }
   ```

4. 继续看`AdminUtils.assignReplicasToBrokers`方法

# 4 Producer生产者流程

## 4.1 Producer示例

下面代码展示`KafkaProducer`的使用方法，在下面示例中，使用`KafkaProducer`实现向Kafka发送消息的功能。

```java
public class MyProducer3 {
    public static void main(String[] args) throws InterruptedException {

        Properties props = new Properties();
        // 客户端id
        props.put(ProducerConfig.CLIENT_ID_CONFIG,"kafkaProducerDemo");
        // kafka地址，列表格式为host1:port1,host2:port2,…，
        // ⽆需添加所有的集群地址，kafka会根据 提供的地址发现其他的地址（建议多提供⼏个，以防提供的服务器关闭）
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,"node1:9092");
        // 发送返回应答⽅式
        // 0:Producer 往集群发送数据不需要等到集群的返回，不确保消息发送成功。安全性最低但是效率最 ⾼。
        // 1:Producer 往集群发送数据只要 Leader 应答就可以发送下⼀条，只确保Leader接收成功。
        // -1或者all：Producer 往集群发送数据需要所有的ISR Follower都完成从Leader的同步才会发送下⼀条，
        // 确保Leader发送成功和所有的副本都成功接收。安全性最⾼，但是效率最低。
        props.put(ProducerConfig.ACKS_CONFIG,"all");
        // 重试次数
        props.put("retries",0);
        // 重试间隔时间
        props.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG,0);
        // 批量发送的⼤⼩
        props.put(ProducerConfig.BATCH_SIZE_CONFIG,16384);
        // ⼀个Batch被创建之后，最多过多久，不管这个Batch有没有写满，都必须发送出去
        props.put(ProducerConfig.LINGER_MS_CONFIG,10);
        // 缓冲区⼤⼩
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG,33554432);
        // key序列化⽅式
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        // value序列化⽅式
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,StringSerializer.class);
        // topic
        String topic = "turbine_grp";

        Producer<String,String> producer = new KafkaProducer<String, String>(props);
        AtomicInteger count = new AtomicInteger();
        while (true){
            int num = count.get();
            String key = Integer.toString(num);
            String value = Integer.toString(num);
            ProducerRecord<String,String> record = new ProducerRecord<>(topic,key,value);
            if(num%2==0){
                // 偶数异步发送
                // 第一个参数record封装了topic、key、value
                // 第二个参数是一个callback对象，当生产者收到kafka发来的ACK确认消息时，会调用此CallBack对象的onComplete方法
                producer.send(record,(recordMetadata,e)->{
                    System.out.println("num:"+num
                                +"\t topic:"+recordMetadata.topic()
                                +"\t offset:"+recordMetadata.offset());
                });
            }else {
                // 同步发送
                // KafkaProducer.send方法返回的类型是Future<RecordMetadata>，通过get方法阻塞当前线程，等待Kafka服务端ack响应
                final RecordMetadata recordMetadata = producer.send(record).get();
            }
            count.incrementAndGet();
            TimeUnit.MILLISECONDS.sleep(100);
        }
    }
}
```

### 4.1.1 同步发送

1. KafkaProducer.send方法返回的类型是Future<RecordMetadata_>，通过get方法阻塞当前线程，等待Kafka服务端ACK响应。

   ```java
   producer.send(record).get();
   ```

### 4.1.2 异步发送

1. 第一个参数record封装了topic、key、value
2. 第二个参数是一个callback对象，当生产者收到kafka发来的ACK确认消息时，会调用此CallBack对象的onComplete方法

```java
producer.send(record,(recordMetadata,e)->{
                    System.out.println("num:"+num
                                +"\t topic:"+recordMetadata.topic()
                                +"\t offset:"+recordMetadata.offset());
                });
```

## 4.2 KafkaProducer实例化

了解`KafkaProducer`的基本使用，开始深入了解KafkaProducer原理和实现，先看一下构造方法核心逻辑

org.apache.kafka.clients.producer.KafkaProducer#KafkaProducer(org.apache.kafka.clients.producer.ProducerConfig, org.apache.kafka.common.serialization.Serializer<K>, org.apache.kafka.common.serialization.Serializer<V>)

## 4.3 消息发送过程

### 4.3.1 拦截器

### 4.3.2 拦截器核心逻辑

### 4.3.3 发送五步骤

### 4.3.4 MetaData更新机制

# 5 Consumer消费者流程

## 5.1 Consumer示例

KafkaConsumer

消费者的根本目的是从Kafka服务端拉取消息，并交给业务逻辑进行处理。

开发人员不必关心与Kafka服务端之间网络连接的管理、心跳检测、请求超时重试等底层操作，也不必关心订阅Topic的分区数量、分区Leader副本的网络拓扑以及消费组的Rebalance等细节，另外还提供了自动提交offset的功能。

```java
public class MyConsumer3 {
    public static void main(String[] args) throws InterruptedException {
        // 是否⾃动提交
        Boolean autoCommit = false;
        // 是否异步提交
        Boolean isSync = true;
        Properties props = new Properties();
        // kafka地址,列表格式为host1:port1,host2:port2,…，⽆需添加所有的集群地址，kafka会根据 提供的地址发现其他的地址（建议多提供⼏个，以防提供的服务器关闭）
        props.put("bootstrap.servers", "localhost:9092");
        // 消费组
        props.put("group.id", "test");
        // 开启⾃动提交offset
        props.put("enable.auto.commit", autoCommit.toString());
        // 1s⾃动提交
        props.put("auto.commit.interval.ms", "1000");
        // 消费者和群组协调器的最⼤⼼跳时间，如果超过该时间则认为该消费者已经死亡或者故障，需要踢出 消费者组
        props.put("session.timeout.ms", "60000");
        // ⼀次poll间隔最⼤时间
        props.put("max.poll.interval.ms", "1000");
        // 当消费者读取偏移量⽆效的情况下，需要重置消费起始位置，默认为latest（从消费者启动后⽣成的 记录），另外⼀个选项值是 earliest，将从有效的最⼩位移位置开始消费
        props.put("auto.offset.reset", "latest");
        // consumer端⼀次拉取数据的最⼤字节数
        props.put("fetch.max.bytes", "1024000");
        // key序列化⽅式
        props.put("key.deserializer",
                "org.apache.kafka.common.serialization.StringDeserializer");
        // value序列化⽅式
        props.put("value.deserializer",
                "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
        String topic = "turbine_grp";
        // 订阅topic列表
        consumer.subscribe(Arrays.asList(topic));
        while (true) {
            // 消息拉取
            ConsumerRecords<String, String> records = consumer.poll(100);

            for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("offset = %d, key = %s, value = %s%n",
                            record.offset(), record.key(), record.value());
            }
            if (!autoCommit) {
                if (isSync) {
                    // 处理完成单次消息以后，提交当前的offset，如果失败会⼀直重试直⾄成功
                    consumer.commitSync();
                } else {
                    // 异步提交
                    consumer.commitAsync((offsets, exception) -> {
                        exception.printStackTrace();
                        System.out.println(offsets.size());
                    });
                }
            }
            TimeUnit.SECONDS.sleep(3);
        }
    }
}
```



## 5.2 KafkaConsumer实例化



# 6 消息存储机制

log.dirs/<topic_name>-<partition_no>/{.index, .timeindex, .log}

首先查看Kafka如何处理产生的消息：

![image-20211216210202892](assest/image-20211216210202892.png)

调用副本管理器，将记录追加到分区的副本中。

![image-20211216210012189](assest/image-20211216210012189.png)

将数据追加到本地Log日志中：

![image-20211216210644748](assest/image-20211216210644748.png)

追加消息的实现：

![image-20211216210950993](assest/image-20211216210950993.png)

# 7 SocketServer



# 8 KafkaRequestHandlerPool



# 9 LogManager



# 10 ReplicaManager



# 11 OffsetManager



# 12 KafkaApis



# 13 KafkaController



# 14 KafkaHealthcheck



# 15 DynamicConfigManager



# 16 分区消费模式



# 17 组消费模式



# 18 同步发送模式



# 19 异步发送模式





