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

提交（Commiting）一个新的段到磁盘需要一个 `fsync` 来确保段被物理性的写入磁盘，这样在断电的时候就不会丢失数据。但是`fsync`操作代价很大；如果每次索引一个文档都去执行一次的话会造成很大的性能问题。

我们需要的是一个更轻量的方式来使一个文档可被搜索，这意味着 `fsync`要从整个过程中被移除。

在Elasticsearch 和 磁盘之间是文件系统缓存。像之前描述的一样，在内存索引缓冲区中的文档会被写入到一个新的段中。但是这里新段会被先写入到文件系统缓存——这一步代价会比较低，稍后再被刷新到磁盘——这一步代价比较高。不过只要文件已经存在系统缓存中，就可以像其他文件一样被打开和读取了。

**在内存缓冲区中包含了新文档的Lucene索引**：

![In-memory buffer](assest/edf18db788d94ded9c7ce427beab0376.png)

Lucene 允许新段被写入和打开——使其包含的文档在未进行一次完整提交时便对搜索可见。这种方式比进行一次提交代价要小得多，并且在不影响性能的前提下可以被频繁的执行。

**缓冲区的内容已经被写入一个可被搜索的段中，但还没有进行提交**：

![uncommit](assest/35f09ecb74a149a99809e3e4e2867532.png)

### 1.2.1 原理

下图表示是 es 写操作流程，当一个写请求发送到 es 后，es将数据写入 `memory buffer`中，并添加事务日志（`translog`）。如果每次一条数据写入内存后立即写到硬盘文件上，由于写入的数据肯定是离散的，因此写入因公安的操作也就是随机写入。硬盘随机写入的效率相当低，会严重降低 es 的性能。

因此 es 在设计时`memory buffer`和硬盘间加入了 Linux 的高速缓存（`File system cache`）来提高 es 的写效率。

当写请求发送到 es 后，es将数据暂时写入`memory buffer`中，此时写入的数据还不能被查询到。默认设置下，es 每1秒钟 将`memory buffer`中的数据 `refresh`到 Linux的`File system cache`，并清空`memory buffer`，此时写入的数据就可以被查询到了。

![es写操作流程](assest/image-20220119192815723.png)



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