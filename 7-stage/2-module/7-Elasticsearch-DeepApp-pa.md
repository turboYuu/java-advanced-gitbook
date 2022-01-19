第七部分 Elasticsearch深度应用及原理剖析

# 1 索引文档写入和近实时搜索原理

## 1.1 基本概念

### 1.1.1 Segment in Lucene

众所周知，Elasticsearch 存储的基本单元是`shard`，ES 中一个 Index 可能分为多个 shard，事实上每个shard都是一个 Lucene的 Index ，并且每个 Lucene Index 由多个`segment`组成，每个Segment事实上是一些倒排序索引的集合，每次创建一个新的`Document`，都会归属于一个新的 Segment，而不会去修改原来的Segment。且每次的文档删除操作，会仅仅标记 Segment 中该文档为删除状态，而不会真正的立马物理删除，所以说 ES 的 index 可以理解为一个抽象的概念，就像下图所示：

![Segment-in-Lucene](assest/f4372b8b83b643b18c2ee6ec1118fe19.png)



### 1.1.2 Commits in Lucene

Commit 操作意味着将Segment 合并，并写入磁盘。保证内存数据尽量不丢失。但是刷盘是很重要的 IO 操作，索引为了机器性能和近实时搜索，并不会刷盘那么及时。

### 1.1.3 Translog

新文档被索引意味着文档会被首先写入内存 buffer 和 translog 文件。每个 shard 都对应一个 translog 文件。

![translog](assest/9cdea21e54c94f84b942b7a111f48ec1.png)

### 1.1.4 Refresh in Elasticsearch

在Elasticsearch中，`_refresh`操作默认每秒执行一次，意味着将内存 buffer 的数据写入到一个新的 Segment 中，这个时候索引变成了可被检索的。写入新Segment后，会清空内存buffer。

![Refresh-in-Elasticsearc](assest/0ce7a70dede942f7a465b528fa9ead99.png)

### 1.1.5 Flush in Elasticsearch

`Flush`操作意味着将内存 buffer 的数据全部都写入新的 Segments 中，并将内存中所有的 Segments 全部刷盘，并且清空 translog 日志的过程。

![Flush-in-Elasticsearch](assest/30e1d93490a94ed6a8a52b7193c726a8.png)

## 1.2 近实时搜索

### 1.2.1 原理

### 1.2.2  refresh API

## 1.3 持久化变更

### 1.3.1 原理

### 1.3.2 flush API

# 2 索引文档存储段合并机制（segment merge、policy、optimize）

# 3 并发冲突处理机制剖析

# 4 分布式数据一致性如何保证？quorum及timeout机制的原理

# 5 Query文档搜索机制剖析

# 6 文档增删改查和搜索请求过程

# 7 相关性评分算法BM25

# 8 排序那点事之内核级DocValues机制大揭秘

# 9 Filter过滤机制剖析（bitset机制与caching机制）

# 10 控制搜索精准度 - 基于boost的细粒度搜索的条件权重控制

# 11 控制搜索精准度 - 基于 dis_max 实现 best fields 策略

# 12 控制搜索精准度 - 基于 function_score 自定义相关度分数算法

# 13 bulk操作的api json格式与底层性能优化的关系

# 14 deep paging 性能问题 和 解决方案