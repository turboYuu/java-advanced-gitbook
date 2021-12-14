第三部分 Kafka集群与运维

# 1 集群应用场景

1. 消息传递

2. 网站活动路由

3. 监控指标

4. 日志汇总

5. 流处理

6. 活动采集

7. 提交日志

总结：

1. 横向扩展，提高Kafka的处理能力
2. 镜像、副本，提供高可用

# 2 集群搭建

1. 搭建设计

   ![image-20211214135122729](assest/image-20211214135122729.png)

2. 分配三台Linux，用于安装又有三个节点的kafka集群。

   - node2(192.168.31.62)
   - node3(192.168.31.63)
   - node4(192.168.31.64)

   以上三台主机的/etc/hosts配置

   ```xml
   192.168.31.61 node1
   192.168.31.62 node2
   192.168.31.63 node3
   192.168.31.64 node4
   ```

## 2.1 Zookeeper集群搭建

1. Linux安装JDK，三台都安装

   ```shell
   [root@node2 ~]# rpm -ivh jdk-8u261-linux-x64.rpm
   
   [root@node2 ~]# vim /etc/profile
   # 文件最后添加两行
   export JAVA_HOME=/usr/java/jdk1.8.0_261-amd64
   export PATH=$PATH:$JAVA_HOME/bin
   
   # 环境变量生效
   [root@node2 ~]# . /etc/profile
   # 验证
   [root@node2 ~]# java -version
   java version "1.8.0_261"
   Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
   Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
   ```

2. 安装Zookeeper，三台都安装，搭建Zookeeper集群

   - node2配置

     ```shell
     # 解压到 /opt 目录
     [root@node2 ~]# tar -zxf zookeeper-3.4.14.tar.gz -C /opt
     # 配置
     [root@node2 ~]# cd /opt/zookeeper-3.4.14/conf/
     [root@node2 conf]# cp zoo_sample.cfg zoo.cfg
     
     dataDir=/var/turbo/zookeeper/data
     
     server.1=node2:2881:3881
     server.2=node3:2881:3881
     server.3=node4:2881:3881
     
     [root@node2 conf]# mkdir -p /var/turbo/zookeeper/data
     [root@node2 conf]# echo 1 > /var/turbo/zookeeper/data/myid
     
     # 环境变量
     [root@node2 conf]# vim /etc/profile
     
     export ZOOKEEPER_PREFIX=/opt/zookeeper-3.4.14
     export PATH=$PATH:$ZOOKEEPER_PREFIX/bin
     export ZOO_LOG_DIR=/var/turbo/zookeeper/log
     # 生效环境变量
     [root@node2 conf]#. /etc/profile
     
     [root@node2 ~]# scp -r /opt/zookeeper-3.4.14/ node3:/opt
     [root@node2 ~]# scp -r /opt/zookeeper-3.4.14/ node4:/opt
     ```

   - node3 配置

     ```shell
     # 配置环境变量
     [root@node3 ~]# vim /etc/profile
     
     export ZOOKEEPER_PREFIX=/opt/zookeeper-3.4.14
     export PATH=$PATH:$ZOOKEEPER_PREFIX/bin
     export ZOO_LOG_DIR=/var/turbo/zookeeper/log
     
     # 生效
     [root@node3 ~]# . /etc/profile
     
     [root@node3 conf]# mkdir -p /var/turbo/zookeeper/data
     [root@node3 conf]# echo 2 > /var/turbo/zookeeper/data/myid
     ```

   - node4 配置

     ```shell
     # 配置环境变量
     [root@node4 ~]# vim /etc/profile
     
     export ZOOKEEPER_PREFIX=/opt/zookeeper-3.4.14
     export PATH=$PATH:$ZOOKEEPER_PREFIX/bin
     export ZOO_LOG_DIR=/var/turbo/zookeeper/log
     
     # 生效
     [root@node4 ~]# . /etc/profile
     
     [root@node4 conf]# mkdir -p /var/turbo/zookeeper/data
     [root@node4 conf]# echo 3 > /var/turbo/zookeeper/data/myid
     ```

   - 启动zookeeper

     关闭防火墙

     ```
     systemctl stop firewalld
     systemctl disable firewalld.service
     ```

     

     ```shell
     # 在三台Linux上启动Zookeeper
     [root@node2 ~]# zkServer.sh start
     [root@node3 ~]# zkServer.sh start
     [root@node4 ~]# zkServer.sh start
     
     # 在三台Linux上查看Zookeeper的状态
     [root@node2 ~]# zkServer.sh status
     ZooKeeper JMX enabled by default
     Using config: /opt/zookeeper-3.4.14/bin/../conf/zoo.cfg
     Mode: follower
     
     [root@node3 ~]# zkServer.sh status
     ZooKeeper JMX enabled by default
     Using config: /opt/zookeeper-3.4.14/bin/../conf/zoo.cfg
     Mode: leader
     
     [root@node4 log]# zkServer.sh status
     ZooKeeper JMX enabled by default
     Using config: /opt/zookeeper-3.4.14/bin/../conf/zoo.cfg
     Mode: follower
     ```

     

## 2.2 Kafka集群搭建

1. 安装Kafka

   - 上传解压

     ```shell
     # 解压
     [root@node2 ~]# tar -zxf kafka_2.12-1.0.2.tgz -C /opt
     
     # 复制到node3和node4
     [root@node2 ~]# scp -r /opt/kafka_2.12-1.0.2/ node3:/opt
     [root@node2 ~]# scp -r /opt/kafka_2.12-1.0.2/ node4:/opt
     ```

   - 配置kafka

     ```shell
     # 三台机器 配置环境变量
     vim /etc/profile
     
     # 添加以下内容
     export KAFKA_HOME=/opt/kafka_2.12-1.0.2
     export PATH=$PATH:$KAFKA_HOME/bin
     
     # 配置生效
     source /etc/profile
     
     
     # node2 配置
     [root@node2 ~]# vim /opt/kafka_2.12-1.0.2/config/server.properties
     
     broker.id=0
     listeners=PLAINTEXT://:9092
     advertised.listeners=PLAINTEXT://node2:9092 
     log.dirs=/var/turbo/kafka/kafka-logs
     zookeeper.connect=node2:2181,node3:2181,node4:2181/myKafka
     #其他使用默认配置
     
     # node3 配置
     [root@node3 ~]# vim /opt/kafka_2.12-1.0.2/config/server.properties
     
     broker.id=1
     listeners=PLAINTEXT://:9092
     advertised.listeners=PLAINTEXT://node3:9092 
     log.dirs=/var/turbo/kafka/kafka-logs
     zookeeper.connect=node2:2181,node3:2181,node4:2181/myKafka
     #其他使用默认配置
     
     
     # node4 配置
     [root@node4 ~]# vim /opt/kafka_2.12-1.0.2/config/server.properties
     
     broker.id=2
     listeners=PLAINTEXT://:9092
     advertised.listeners=PLAINTEXT://node4:9092 
     log.dirs=/var/turbo/kafka/kafka-logs
     zookeeper.connect=node2:2181,node3:2181,node4:2181/myKafka
     #其他使用默认配置
     
     ```

   - 启动Kafka

     ```shell
     [root@node2 ~]# kafka-server-start.sh /opt/kafka_2.12-1.0.2/config/server.properties
     [root@node3 ~]# kafka-server-start.sh /opt/kafka_2.12-1.0.2/config/server.properties
     [root@node4 ~]# kafka-server-start.sh /opt/kafka_2.12-1.0.2/config/server.properties
     ```

     node2节点的Cluster ID：

     ![image-20211214162700705](assest/image-20211214162700705.png)

     node3节点的Cluster ID：

     ![image-20211214162719054](assest/image-20211214162719054.png)

     node4节点的Cluster ID：

     ![image-20211214162747698](assest/image-20211214162747698.png)

   - 验证Kafka

     1. Cluster ID是一个位移的不可变的标识符，用于唯一标志一个Kafka集群；
     2. 该Id最多可以有22个字符组成，字符对应于URL-safe Base64；
     3. Kafka 0.10.1版本之后的版本中，在集群第一次启动的时候，Broker从Zookeeper的<Kafka_ROOT>/cluster/id节点获取，如果该id不存在，就自动生成一个新的。

     ```shell
     zkCli.sh
     # 查看每个Broker的信息
     get /myKafka/brokers/ids/0
     get /myKafka/brokers/ids/1
     get /myKafka/brokers/ids/2
     ```

     ![image-20211214163707722](assest/image-20211214163707722.png)

     ![image-20211214165834189](assest/image-20211214165834189.png)

     node2节点在Zookeeper上的信息：

     ![image-20211214163915941](assest/image-20211214163915941.png)

     node3节点在Zookeeper上的信息：

     ![image-20211214163948462](assest/image-20211214163948462.png)

     node4节点在Zookeeper上的信息：

     ![image-20211214164005064](assest/image-20211214164005064.png)

# 3 集群监控

## 3.1 监控度量指标

Kafka使用Yammer Metrics在服务器和Scala客户端中报告指标。Java客户端使用Kafka Metrics，它是一个内置的度量标准注册表。可最大程度地减少拉入客户端应用程序地传递依赖项。两者都通过JMX公开指标，并且可以配置为使用可插拔的统计报告统计信息，以连接到你的监视系统。

具体监控指标可以查看：[官方文档](http://kafka.apache.org/10/documentation.html#monitoring)。

### 3.1.1 JMX

#### 3.1.1.1 Kafka开启JMX端口

```shell
vim /opt/kafka_2.12-1.0.2/bin/kafka-server-start.sh
```

![image-20211214173147192](assest/image-20211214173147192.png)

#### 3.1.1.2 验证JMX开启

### 3.1.2 使用JConsole连接JMX端口

### 3.1.3 编程手段来获取监控指标

## 3.2 监控工具 Kafka Eagle