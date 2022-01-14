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

## 2.2 分层

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