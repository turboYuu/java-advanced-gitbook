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

了解了`KafkaConsumer`的基本使用，开始深入了解`KafkaConsumer`原理和实现，先看一下构造方法核心逻辑：

```java
org.apache.kafka.clients.consumer.KafkaConsumer#KafkaConsumer(org.apache.kafka.clients.consumer.ConsumerConfig, org.apache.kafka.common.serialization.Deserializer<K>, org.apache.kafka.common.serialization.Deserializer<V>)
```



```java
private KafkaConsumer(ConsumerConfig config,
                          Deserializer<K> keyDeserializer,
                          Deserializer<V> valueDeserializer) {
        try {
            // 获取客户端id，如果没有设置，则生成一个
            String clientId = config.getString(ConsumerConfig.CLIENT_ID_CONFIG);
            if (clientId.isEmpty())
                clientId = "consumer-" + CONSUMER_CLIENT_ID_SEQUENCE.getAndIncrement();
            this.clientId = clientId;
            // 获取消费组id
            String groupId = config.getString(ConsumerConfig.GROUP_ID_CONFIG);

            LogContext logContext = new LogContext("[Consumer clientId=" + clientId + ", groupId=" + groupId + "] ");
            this.log = logContext.logger(getClass());

            log.debug("Initializing the Kafka consumer");
            this.requestTimeoutMs = config.getInt(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG);
            int sessionTimeOutMs = config.getInt(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG);
            int fetchMaxWaitMs = config.getInt(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG);
            if (this.requestTimeoutMs <= sessionTimeOutMs || this.requestTimeoutMs <= fetchMaxWaitMs)
                throw new ConfigException(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG + " should be greater than " + ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG + " and " + ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG);
            this.time = Time.SYSTEM;

            Map<String, String> metricsTags = Collections.singletonMap("client-id", clientId);
            MetricConfig metricConfig = new MetricConfig().samples(config.getInt(ConsumerConfig.METRICS_NUM_SAMPLES_CONFIG))
                    .timeWindow(config.getLong(ConsumerConfig.METRICS_SAMPLE_WINDOW_MS_CONFIG), TimeUnit.MILLISECONDS)
                    .recordLevel(Sensor.RecordingLevel.forName(config.getString(ConsumerConfig.METRICS_RECORDING_LEVEL_CONFIG)))
                    .tags(metricsTags);
            List<MetricsReporter> reporters = config.getConfiguredInstances(ConsumerConfig.METRIC_REPORTER_CLASSES_CONFIG,
                    MetricsReporter.class);
            reporters.add(new JmxReporter(JMX_PREFIX));
            this.metrics = new Metrics(metricConfig, reporters, time);
            this.retryBackoffMs = config.getLong(ConsumerConfig.RETRY_BACKOFF_MS_CONFIG);

            // load interceptors and make sure they get clientId
            // 获取用户设置的参数
            Map<String, Object> userProvidedConfigs = config.originals();
            // 设置客户端id
            userProvidedConfigs.put(ConsumerConfig.CLIENT_ID_CONFIG, clientId);
            // 设置拦截器
            List<ConsumerInterceptor<K, V>> interceptorList = (List) (new ConsumerConfig(userProvidedConfigs, false)).getConfiguredInstances(ConsumerConfig.INTERCEPTOR_CLASSES_CONFIG,
                    ConsumerInterceptor.class);
            // 如果没有设置拦截器，就是null，否则设置拦截器
            this.interceptors = interceptorList.isEmpty() ? null : new ConsumerInterceptors<>(interceptorList);
            // key的反序列化
            if (keyDeserializer == null) {
                this.keyDeserializer = config.getConfiguredInstance(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                        Deserializer.class);
                this.keyDeserializer.configure(config.originals(), true);
            } else {
                config.ignore(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG);
                this.keyDeserializer = keyDeserializer;
            }
            // value的反序列化
            if (valueDeserializer == null) {
                this.valueDeserializer = config.getConfiguredInstance(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                        Deserializer.class);
                this.valueDeserializer.configure(config.originals(), false);
            } else {
                config.ignore(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG);
                this.valueDeserializer = valueDeserializer;
            }
            ClusterResourceListeners clusterResourceListeners = configureClusterResourceListeners(keyDeserializer, valueDeserializer, reporters, interceptorList);
            this.metadata = new Metadata(retryBackoffMs, config.getLong(ConsumerConfig.METADATA_MAX_AGE_CONFIG),
                    true, false, clusterResourceListeners);
            List<InetSocketAddress> addresses = ClientUtils.parseAndValidateAddresses(config.getList(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG));
            // 更新集群元数据
            this.metadata.update(Cluster.bootstrap(addresses), Collections.<String>emptySet(), 0);
            String metricGrpPrefix = "consumer";
            ConsumerMetrics metricsRegistry = new ConsumerMetrics(metricsTags.keySet(), "consumer");
            // 创建Channel的构建起
            ChannelBuilder channelBuilder = ClientUtils.createChannelBuilder(config);

            // 设置隔离级别
            IsolationLevel isolationLevel = IsolationLevel.valueOf(
                    config.getString(ConsumerConfig.ISOLATION_LEVEL_CONFIG).toUpperCase(Locale.ROOT));
            Sensor throttleTimeSensor = Fetcher.throttleTimeSensor(metrics, metricsRegistry.fetcherMetrics);

            // 实例化网络客户端
            NetworkClient netClient = new NetworkClient(
                    new Selector(config.getLong(ConsumerConfig.CONNECTIONS_MAX_IDLE_MS_CONFIG), metrics, time, metricGrpPrefix, channelBuilder, logContext),
                    this.metadata,
                    clientId,
                    100, // a fixed large enough value will suffice for max in-flight requests
                    config.getLong(ConsumerConfig.RECONNECT_BACKOFF_MS_CONFIG),
                    config.getLong(ConsumerConfig.RECONNECT_BACKOFF_MAX_MS_CONFIG),
                    config.getInt(ConsumerConfig.SEND_BUFFER_CONFIG),
                    config.getInt(ConsumerConfig.RECEIVE_BUFFER_CONFIG),
                    config.getInt(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG),
                    time,
                    true,
                    new ApiVersions(),
                    throttleTimeSensor,
                    logContext);
            // 实例化消费者网络客户端
            this.client = new ConsumerNetworkClient(
                    logContext,
                    netClient,
                    metadata,
                    time,
                    retryBackoffMs,
                    config.getInt(ConsumerConfig.REQUEST_TIMEOUT_MS_CONFIG));
            OffsetResetStrategy offsetResetStrategy = OffsetResetStrategy.valueOf(config.getString(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG).toUpperCase(Locale.ROOT));
            // 创建订阅的对象，用于封装订阅信息
            this.subscriptions = new SubscriptionState(offsetResetStrategy);
            // 消费组和主题分区分配器
            this.assignors = config.getConfiguredInstances(
                    ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
                    PartitionAssignor.class);
            // 协调器，协调再平衡的协调器
            this.coordinator = new ConsumerCoordinator(logContext,
                    this.client,
                    groupId,
                    config.getInt(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG),
                    config.getInt(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG),
                    config.getInt(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG),
                    assignors,
                    this.metadata,
                    this.subscriptions,
                    metrics,
                    metricGrpPrefix,
                    this.time,
                    retryBackoffMs,
                    config.getBoolean(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG),
                    config.getInt(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG),
                    this.interceptors,
                    config.getBoolean(ConsumerConfig.EXCLUDE_INTERNAL_TOPICS_CONFIG),
                    config.getBoolean(ConsumerConfig.LEAVE_GROUP_ON_CLOSE_CONFIG));
            // 通过网络获取消息的对象
            this.fetcher = new Fetcher<>(
                    logContext,
                    this.client,
                    config.getInt(ConsumerConfig.FETCH_MIN_BYTES_CONFIG),
                    config.getInt(ConsumerConfig.FETCH_MAX_BYTES_CONFIG),
                    config.getInt(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG),
                    config.getInt(ConsumerConfig.MAX_PARTITION_FETCH_BYTES_CONFIG),
                    config.getInt(ConsumerConfig.MAX_POLL_RECORDS_CONFIG),
                    config.getBoolean(ConsumerConfig.CHECK_CRCS_CONFIG),
                    this.keyDeserializer,
                    this.valueDeserializer,
                    this.metadata,
                    this.subscriptions,
                    metrics,
                    metricsRegistry.fetcherMetrics,
                    this.time,
                    this.retryBackoffMs,
                    isolationLevel);

            config.logUnused();
            AppInfoParser.registerAppInfo(JMX_PREFIX, clientId, metrics);

            log.debug("Kafka consumer initialized");
        } catch (Throwable t) {
            // call close methods if internal objects are already constructed
            // this is to prevent resource leak. see KAFKA-2121
            close(0, true);
            // now propagate the exception
            throw new KafkaException("Failed to construct kafka consumer", t);
        }
    }
```

1. 初始化参数配置

   client.id、group.id、消费者拦截器、key/value序列化、事务隔离级别

2. 初始化网络客户端`NetworkClient`

3. 初始化消费者网络客户端`ConsumerNetworkClient`

4. 初始化offset提交策略，默认自动提交

5. 初始化消费者协调器`ConsumerCoordinator`

6. 初始化拉取器`Fetcher`

## 5.3 订阅Topic

下面先看一下subscribe方法都有哪些逻辑：

```java
public void subscribe(Collection<String> topics, ConsumerRebalanceListener listener) {
    // 轻量级锁
    acquireAndEnsureOpen();
    try {
        if (topics == null) {
            throw new IllegalArgumentException("Topic collection to subscribe to cannot be null");
        } else if (topics.isEmpty()) {
            // topic为空，则开始取消订阅的逻辑
            // treat subscribing to empty topic list as the same as unsubscribing
            this.unsubscribe();
        } else {
            // topic合法性判断，包含null或者空字符串直接抛异常
            for (String topic : topics) {
                if (topic == null || topic.trim().isEmpty())
                    throw new IllegalArgumentException("Topic collection to subscribe to cannot contain null or empty topic");
            }
			// 如果没有消费协调者直接抛异常
            throwIfNoAssignorsConfigured();
            log.debug("Subscribed to topic(s): {}", Utils.join(topics, ", "));
            // 开始订阅
            this.subscriptions.subscribe(new HashSet<>(topics), listener);
            // 更新元数据，如果metadata当前不包括所有的topics则标记更新
            metadata.setTopics(subscriptions.groupSubscription());
        }
    } finally {
        release();
    }
}
```

![image-20211220114455175](assest/image-20211220114455175.png)

![image-20211220115422383](assest/image-20211220115422383.png)

1. KafkaConsumer不是线程安全类，开启轻量级锁，topics为空抛异常，topics是空集合开始取消订阅，再次判断topics集合中是否有非法数据，判断消费者协调者是否为空。开始订阅对应topic。listener默认为`NoOpConsumerRebalanceListener`，一个空操作

   > 轻量级锁：分别记录了当前使用KafkaConsumer的线程id和重入次数，KafkaConsumer的acquire()和release()方法实现了一个"轻量级锁"，它并非真正的锁，紧时检测是否有多线程并发操作KafkaConsumer而已。

2. 每一个KafkaConsumer实例内部都拥有一个`SubscriptionState`对象，subscribe内部调用了subscribe方法，subscribe方法订阅信息记录到`SubscriptionState`，多次订阅会覆盖旧数据。

3. 更新metadata，判断如果metadata中不包含当前`groupSubscription`，开始标记更新（后面会有更新的逻辑，并且消费者侧的topic不会过期）



## 5.4 消息消费过程

### 5.4.1 poll



### 5.4.2 pollOnce

#### 5.4.2.1 coordinator.poll()



#### 5.4.2.2 updateFetchPositions()



#### 5.4.2.3 fetcher.fetchdRecords()



#### 5.4.2.4 fetcher.sendFecthes()



#### 5.4.2.5 client.poll()



#### 5.4.2.6 coordinator.needRejoin()



## 5.5 自动提交



## 5.6 手动提交



### 5.6.1 同步提交



### 5.6.2 异步提交



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

遍历需要追加的每个主题分区的消息：

![image-20211220140210216](assest/image-20211220140210216.png)

调用partition的方法将记录追加到该分区的leader分区中：

![image-20211220140545538](assest/image-20211220140545538.png)

如果在本地找到了该分区的Leader：

![image-20211220141148628](assest/image-20211220141148628.png)

执行下述逻辑将消息追加到leader分区：

```scala
// 获取该分区的log
val log = leaderReplica.log.get
// 获取最小ISR副本数
val minIsr = log.config.minInSyncReplicas
// 计算同步副本的个数
val inSyncSize = inSyncReplicas.size

// Avoid writing to leader if there are not enough insync replicas to make it safe
// 如果同步副本的个数小于要求的最小副本数，并且acks设置的是-1，则不追加消息
if (inSyncSize < minIsr && requiredAcks == -1) {
    throw new NotEnoughReplicasException("Number of insync replicas for partition %s is [%d], below required minimum [%d]"
              .format(topicPartition, inSyncSize, minIsr))
}

// 追加消息到leader
val info = log.appendAsLeader(records, leaderEpoch = this.leaderEpoch, isFromClient)
// probably unblock some follower fetch requests since log end offset has been updated
// 尝试锁定follower获取消息的请求，因为此时leader正在更新 LEO
replicaManager.tryCompleteDelayedFetch(TopicPartitionOperationKey(this.topic, this.partitionId))
// we may need to increment high watermark since ISR could be down to 1
// 如果ISR只有一个元素的话，需要 HW+1
(info, maybeIncrementLeaderHW(leaderReplica))
```

`log.appendAsLeader(records, leaderEpoch = this.leaderEpoch, isFromClient)`的实现：

![image-20211220142440246](assest/image-20211220142440246.png)

![image-20211220143306548](assest/image-20211220143306548.png)



# 7 SocketServer

线程模型：

1. 当前broker上配置了多少个listener，就有多少个Acceptor，用于新建连接
2. 每个Acceptor对应N个线程的处理器（Processor），用于接收客户端请求
3. 每个处理器对应M个线程的处理程序（Handler），处理用户请求，并将响应发送给等待客户写响应的处理器线程。



在启动KafkaServer的startup方法中启动SocketServer：

![image-20211220160100880](assest/image-20211220160100880.png)

每个listener就是一个端点，每个端点创建多个处理程序。

![image-20211220160627766](assest/image-20211220160627766.png)

究竟启动多少了处理程序？

processor个数为numProcessorThreads个。上图中for循环为从`processorBeginIndex`到`processorEndIndex`（不包括）。

numProcessorThread为：

![image-20211220161146693](assest/image-20211220161146693.png)

![image-20211220161255389](assest/image-20211220161255389.png)

![image-20211220161347959](assest/image-20211220161347959.png)





accptor的启动过程：

![image-20211220161532030](assest/image-20211220161532030.png)

KafkaThread：

![image-20211220161957902](assest/image-20211220161957902.png)

![image-20211220162112370](assest/image-20211220162112370.png)

调用Thread的构造器：

![image-20211220162219106](assest/image-20211220162219106.png)

![image-20211220162303073](assest/image-20211220162303073.png)

KafkaThread的start方法即是 Thread的start方法，此时调用的是acceptor的run方法：

```scala
/**
   * Accept loop that checks for new connection attempts
    * 使用Java的NIO
    * 循环检查是否有新的连接尝试
    * 轮询的方式将请求交给各个processor来处理
   */
  def run() {
    serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT)
    startupComplete()
    try {
      var currentProcessor = 0
      while (isRunning) {
        try {
          val ready = nioSelector.select(500)
          if (ready > 0) {
            val keys = nioSelector.selectedKeys()
            val iter = keys.iterator()
            while (iter.hasNext && isRunning) {
              try {
                val key = iter.next
                iter.remove()
                if (key.isAcceptable)
                  // 指定一个processor处理请求
                  accept(key, processors(currentProcessor))
                else
                  throw new IllegalStateException("Unrecognized key state for acceptor thread.")

                // round robin to the next processor thread
                // 通过轮询的方式找到下一个processor线程
                currentProcessor = (currentProcessor + 1) % processors.length
              } catch {
                case e: Throwable => error("Error while accepting connection", e)
              }
            }
          }
        }
        catch {
          // We catch all the throwables to prevent the acceptor thread from exiting on exceptions due
          // to a select operation on a specific channel or a bad request. We don't want
          // the broker to stop responding to requests from other clients in these scenarios.
          case e: ControlThrowable => throw e
          case e: Throwable => error("Error occurred", e)
        }
      }
    } finally {
      debug("Closing server socket and selector.")
      swallowError(serverChannel.close())
      swallowError(nioSelector.close())
      shutdownComplete()
    }
  }
```

Acceptor建立建立，处理请求：

```scala
/*
   * Accept a new connection
   * 建立一个连接
   */
  def accept(key: SelectionKey, processor: Processor) {
    // 服务端
    val serverSocketChannel = key.channel().asInstanceOf[ServerSocketChannel]
    // 客户端
    val socketChannel = serverSocketChannel.accept()
    try {
      connectionQuotas.inc(socketChannel.socket().getInetAddress)
      // 非阻塞
      socketChannel.configureBlocking(false)
      socketChannel.socket().setTcpNoDelay(true)
      socketChannel.socket().setKeepAlive(true)
      // 设置发送缓冲大小
      if (sendBufferSize != Selectable.USE_DEFAULT_BUFFER_SIZE)
        socketChannel.socket().setSendBufferSize(sendBufferSize)

      debug("Accepted connection from %s on %s and assigned it to processor %d, sendBufferSize [actual|requested]: [%d|%d] recvBufferSize [actual|requested]: [%d|%d]"
            .format(socketChannel.socket.getRemoteSocketAddress, socketChannel.socket.getLocalSocketAddress, processor.id,
                  socketChannel.socket.getSendBufferSize, sendBufferSize,
                  socketChannel.socket.getReceiveBufferSize, recvBufferSize))
      // 调用Processor的accept方法，由processor处理请求
      processor.accept(socketChannel)
    } catch {
      case e: TooManyConnectionsException =>
        info("Rejected connection from %s, address already has the configured maximum of %d connections.".format(e.ip, e.count))
        close(socketChannel)
    }
  }
```

Processor将连接加入缓冲队列，同时唤醒处理线程：

![image-20211220165451778](assest/image-20211220165451778.png)

Processor的run方法从newConnections中取出请求的channel，解析封装请求，交给handler处理：

![image-20211220170804551](assest/image-20211220170804551.png)

![image-20211220170933509](assest/image-20211220170933509.png)

将请求消息放到请求队列中：

在KafkaServer的startup方法中实例化KafkaRequestHandlerPool，该类会立即初始化numIoThreads个线程用于执行KafkaRequestHandler处理请求的逻辑。

![image-20211220171702337](assest/image-20211220171702337.png)

KafkaRequestHandlerPool以多线程的方式启动多个KafkaRequesthandler：

![image-20211220172535362](assest/image-20211220172535362.png)

KafkaRequestHandler的run方法中，receiveRequest方法从请求队列获取请求：

![image-20211220172836171](assest/image-20211220172836171.png)

具体实现：

![image-20211220173007154](assest/image-20211220173007154.png)

KafkaRequestHandler的run方法中使用模式匹配：

![image-20211220173154268](assest/image-20211220173154268.png)

上图中，apis的handler方法处理请求：

```scala
/**
   * Top-level method that handles all requests and multiplexes to the right api
    * 处理所有请求的顶级方法，使用模式匹配，交给具体的api来处理
   */
  def handle(request: RequestChannel.Request) {
    try {
      trace(s"Handling request:${request.requestDesc(true)} from connection ${request.context.connectionId};" +
        s"securityProtocol:${request.context.securityProtocol},principal:${request.context.principal}")
      request.header.apiKey match {
          // 消息生产请求的处理逻辑
        case ApiKeys.PRODUCE => handleProduceRequest(request)
        // 处理
        case ApiKeys.FETCH => handleFetchRequest(request)
        case ApiKeys.LIST_OFFSETS => handleListOffsetRequest(request)
        case ApiKeys.METADATA => handleTopicMetadataRequest(request)
        case ApiKeys.LEADER_AND_ISR => handleLeaderAndIsrRequest(request)
        case ApiKeys.STOP_REPLICA => handleStopReplicaRequest(request)
        case ApiKeys.UPDATE_METADATA => handleUpdateMetadataRequest(request)
        case ApiKeys.CONTROLLED_SHUTDOWN => handleControlledShutdownRequest(request)
        case ApiKeys.OFFSET_COMMIT => handleOffsetCommitRequest(request)
        case ApiKeys.OFFSET_FETCH => handleOffsetFetchRequest(request)
        case ApiKeys.FIND_COORDINATOR => handleFindCoordinatorRequest(request)
        case ApiKeys.JOIN_GROUP => handleJoinGroupRequest(request)
        case ApiKeys.HEARTBEAT => handleHeartbeatRequest(request)
        case ApiKeys.LEAVE_GROUP => handleLeaveGroupRequest(request)
        case ApiKeys.SYNC_GROUP => handleSyncGroupRequest(request)
        case ApiKeys.DESCRIBE_GROUPS => handleDescribeGroupRequest(request)
        case ApiKeys.LIST_GROUPS => handleListGroupsRequest(request)
        case ApiKeys.SASL_HANDSHAKE => handleSaslHandshakeRequest(request)
        case ApiKeys.API_VERSIONS => handleApiVersionsRequest(request)
        case ApiKeys.CREATE_TOPICS => handleCreateTopicsRequest(request)
        case ApiKeys.DELETE_TOPICS => handleDeleteTopicsRequest(request)
        case ApiKeys.DELETE_RECORDS => handleDeleteRecordsRequest(request)
        case ApiKeys.INIT_PRODUCER_ID => handleInitProducerIdRequest(request)
        case ApiKeys.OFFSET_FOR_LEADER_EPOCH => handleOffsetForLeaderEpochRequest(request)
        case ApiKeys.ADD_PARTITIONS_TO_TXN => handleAddPartitionToTxnRequest(request)
        case ApiKeys.ADD_OFFSETS_TO_TXN => handleAddOffsetsToTxnRequest(request)
        case ApiKeys.END_TXN => handleEndTxnRequest(request)
        case ApiKeys.WRITE_TXN_MARKERS => handleWriteTxnMarkersRequest(request)
        case ApiKeys.TXN_OFFSET_COMMIT => handleTxnOffsetCommitRequest(request)
        case ApiKeys.DESCRIBE_ACLS => handleDescribeAcls(request)
        case ApiKeys.CREATE_ACLS => handleCreateAcls(request)
        case ApiKeys.DELETE_ACLS => handleDeleteAcls(request)
        case ApiKeys.ALTER_CONFIGS => handleAlterConfigsRequest(request)
        case ApiKeys.DESCRIBE_CONFIGS => handleDescribeConfigsRequest(request)
        case ApiKeys.ALTER_REPLICA_LOG_DIRS => handleAlterReplicaLogDirsRequest(request)
        case ApiKeys.DESCRIBE_LOG_DIRS => handleDescribeLogDirsRequest(request)
        case ApiKeys.SASL_AUTHENTICATE => handleSaslAuthenticateRequest(request)
        case ApiKeys.CREATE_PARTITIONS => handleCreatePartitionsRequest(request)
      }
    } catch {
      case e: FatalExitError => throw e
      case e: Throwable => handleError(request, e)
    } finally {
      request.apiLocalCompleteTimeNanos = time.nanoseconds
    }
  }
```



# 8 KafkaRequestHandlerPool

KafkaRequestHandlerPool的作用是创建

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





