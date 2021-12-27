第一部分 全文所有引擎 Elasticsearch基础

# 1 Elasticsearch是什么

Elasticsearch简称ES，是一个开源的可扩展的分布式的**全文搜索引擎**，它可以近乎实时的存储、检索数据。本身扩展性很好，可扩展到上百台服务器，处理**PB**级别的数据。ES使用Java开发并使用Lucene作为核心来实现索引和搜索的功能，但是它通过简单的**RestfulAPI**和**javaAPI**来隐藏Lucene的复杂性，从而让全文搜索变得简单。

Elasticsearch官网：https://www.elastic.co/cn/elasticsearch/

![image-20211227171646490](assest/image-20211227171646490.png)

起源：Shay Banon，2004年失业，陪wife去伦敦学习厨师。事业在家帮wife写一个菜谱搜索引擎。封装了lucene，做出了开源项目compass。找到工作后，做分布式高性能项目，再封装compass，写出来elasticsearch，使得lucene支持分布式。现在Elasticsearch创建人，兼Elastic首席执行官。

# 2 Elasticsearch的功能

# 3 Elasticsearch的特点

# 4 Elasticsearch企业使用场景

## 4.1 常见场景

## 4.2 常见案例

# 5 主流全文搜索方案对比

# 6 Elasticsearch的版本

## 6.1 Elasticsearch版本介绍

## 6.2 Elasticsearch与其他软件兼容

# 7 Elasticsearch Single-Node Mode 快速部署

## 7.1 虚拟机环境准备

## 7.2 Elasticsearch single-Node Mode 部署