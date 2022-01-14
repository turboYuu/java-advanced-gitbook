第四部分 Elasticsearch企业级高可用分布式集群

# 1 核心概念

## 1.1 集群（Cluster）

一个Elasticsearch集群由多个节点（Node）组成，每个集群都有一个共同的集群名称作为标识

## 1.2 节点（Node）

- 一个Elasticsearch实例即一个Node，一台机器可以有多个实例，正常使用下每个实例都应该会部署在不同的机器上。Elasticsearch的配置文件中可以通过node.master、node.data来设置节点类型。
- node.master：表示节点是否具有称为主节点的资格
  - true：代表的是有资格竞选主节点
  - false：代表的时没有资格竞选主节点
- node.data：表示节点是否存储数据

- Node 节点组合

  - 主节点 + 数据节点（master + data）默认

    - 节点既有成为主节点的，又存储数据

      ```properties
      node.mater: true
      node.data: true
      ```

    - 数据节点（data）

      节点没有成为主节点的资格，不参与选举，只会存储数据

      ```properties
      node.master: false
      node.data: true
      ```

    - 客户端节点（client）

      不会成为主节点，也不会存储数据，主要是神对海量请求的时候可以进行负载均衡

      ```properties
      node.master: false
      node.data: false
      ```



## 1.3 分片

每个索引有一个或多个分片，每个分片存储不同的数据。分片可分为主分片（primary shard）和复制分片（replica shard），复制分片是主分片的拷贝。默认每个主分片有一个复制分片，每个索引的复制分片数量可以动态调整，复制分片从不与它的主分片在同一个节点上

## 1.4 副本

这里指主分片的副本分片（主分片的拷贝）

```
提高恢复能力：当主分片挂掉时，某个复制分片可以变成主分片
提高性能：get 和 search 请求既可以又主分片又可以由复制分片处理
```



# 2 Elasticsearch分布式架构

Elasticsearch 的架构遵循其基本概念：一个采用Restful API 标准的高扩展性和高可用性的实时数据分析的全文搜索引擎。

## 2.1 特性

- 高扩展性：体现在Elasticsearch添加节点非常简单，新节点无需做复杂的配置，只需要配置好集群信息将会被集群自动发现。
- 高可用性：因为Elasticsearch是分布式的，每个节点都会有备份，所以宕机一两个节点也不会出现问题，集群会通过备份进行自动复盘
- 实时性：使用倒排序索引来建立存储结构，搜索时常在百毫秒内就可完成。

## 2.2 分层

![es系统架构](assest/es系统架构.png)

> 第一层 Gateway

Elasticsearch支持的索引快照存储格式，ES 默认是先把索引存放在内存中，当内存满了之后再持久化到本地磁盘。gateway对索引快照进行存储，当Elasticsearch关闭再启动的时候，它就会从这个gateway里面读取索引数据；支持的格式有：本地的 Local FileSystem、分布式的Shared FileSystem、Hadoop的文件系统 HDFS、Amazon（亚马逊）的S3。

> 第二层 Lucene框架

Elasticsearch 基于 Lucene（基于Java开发）框架。

> 第三层 Elasticsearch数据的加工处理方式

Index Module（创建Index模块）、Search Module（搜索模块）、Mapping（映射）、River代表 ES 的一个数据源（运行在Elasticsearch 集群内存的一个插件，主要用来从外部获取异构数据，然后在Elasticsearch里面创建索引；常见的插件有 RabbitMQ River、Twitter River）。

> 第四层 Elasticsearch发现机制、脚本

Discovery 是 Elasticsearch 自动发现节点的机制模块，Zen Discovery 和 EC2 discovery。EC2：亚马逊弹性计算云 EC2 discovery主要在亚马逊云平台中使用。Zen Discovery作用就相当于 solrcloud 中的 zookeeper。<br>zen discovery 从功能上可以分为两部分，<br>第一部分是集群刚启动时的选主，或是新加入集群的节点发现当前集群的Master。<br>第二部分是选主完成后，Master和Follower的相互探活。

Scripting 是脚本执行功能，有这个功能能很方便对查询出来的数据进行加工处理。

3rd Plugins 表示 Elasticsearch支持安装很多第三方的插件，例如 elasticsearch-ik分词插件、elasticsearch-sql sql插件。

> 第五层 Elasticsearch的交互方式

有Thrift、Memcached、Http三种协议，默认的是用Http协议传输

> 第六层 Elasticsearch 的 API 支持模式

Restful Style API 风格的API接口标准是当下十分流行的。Elasticsearch作为分布式集群，客户端到服务端，节点与节点间通信有TCP和Http通信协议，底层实现为Netty框架



## 2.3 解析Elasticsearch的分布式架构

## 2.4 分布式架构的透明隐藏特性

## 2.5 主节点

## 2.6 节点对等

# 3 集群环境搭建



# 4 集群规划

## 4.1 需要多大规模的集群

## 4.2 集群中的节点角色如何分配

## 4.3 如何避免脑裂问题

## 4.4 索引应该设置多少个分片

## 4.5 分片应该设置几个副本

# 5 分布式集群调优策略

## 5.1 Index(写)调优

## 5.3 Search(读)调优