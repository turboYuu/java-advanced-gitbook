第七部分 Elasticsearch深度应用及原理剖析

[Elasticsearch: 权威指南](https://www.elastic.co/guide/cn/elasticsearch/guide/current/index.html)

# 1 索引文档写入和近实时搜索原理

## 1.1 基本概念

### 1.1.1 Segment in Lucene

众所周知，Elasticsearch 存储的基本单元是`shard`，ES 中一个 Index 可能分为多个 shard，事实上每个shard都是一个 Lucene的 Index ，并且每个 Lucene Index 由多个`segment`组成，每个Segment事实上是一些倒排序索引的集合，每次创建一个新的`Document`，都会归属于一个新的 Segment，而不会去修改原来的Segment。且每次的文档删除操作，会仅仅标记 Segment 中该文档为删除状态，而不会真正的立马物理删除，所以说 ES 的 index 可以理解为一个抽象的概念，就像下图所示：

![Segment-in-Lucene](assest/f4372b8b83b643b18c2ee6ec1118fe19.png)



### 1.1.2 Commits in Lucene

Commit 操作意味着将Segment 合并，并写入磁盘。保证内存数据尽量不丢失。但是刷盘是很重的 IO 操作，索引为了机器性能和近实时搜索，并不会刷盘那么及时。

### 1.1.3 Translog

新文档被索引意味着文档会被首先写入内存 buffer 和 translog 文件。每个 shard 都对应一个 translog 文件。

![translog](assest/9cdea21e54c94f84b942b7a111f48ec1.png)

### 1.1.4 Refresh in Elasticsearch

在Elasticsearch中，`_refresh`操作默认每秒执行一次，意味着将内存 buffer 的数据写入到一个新的 Segment 中，这个时候索引变成了可被检索的。写入新Segment后，会清空内存buffer。

![Refresh-in-Elasticsearch](assest/0ce7a70dede942f7a465b528fa9ead99.png)

### 1.1.5 Flush in Elasticsearch

`Flush`操作意味着将内存 buffer 的数据全部都写入新的 Segments 中，并将内存中所有的 Segments 全部刷盘，并且清空 translog 日志的过程。

![Flush-in-Elasticsearch](assest/30e1d93490a94ed6a8a52b7193c726a8.png)

## 1.2 近实时搜索

[近实时搜索-官网参考](https://www.elastic.co/guide/cn/elasticsearch/guide/current/near-real-time.html)

提交（Commiting）一个新的段到磁盘需要一个 `fsync` 来确保段被物理性的写入磁盘，这样在断电的时候就不会丢失数据。但是`fsync`操作代价很大；如果每次索引一个文档都去执行一次的话会造成很大的性能问题。

我们需要的是一个更轻量的方式来使一个文档可被搜索，这意味着 `fsync` 要从整个过程中被移除。

在Elasticsearch 和 磁盘之间是文件系统缓存。像之前描述的一样，在内存索引缓冲区中的文档会被写入到一个**新的段**（文件系统缓存中的结构）中。**但是这里新段会被先写入到文件系统缓存**——这一步代价会比较低，稍后再被刷新到磁盘——这一步代价比较高。不过只要文件已经在系统缓存中，就可以像其他文件一样被打开和读取了。

**在内存缓冲区中包含了新文档的Lucene索引**：

![In-memory buffer](assest/edf18db788d94ded9c7ce427beab0376.png)

Lucene 允许新段被写入和打开——使其包含的文档在未进行一次完整提交时便对搜索可见。这种方式比进行一次提交代价要小得多，并且在不影响性能的前提下可以被频繁的执行。

**缓冲区的内容已经被写入一个可被搜索的段中，但还没有进行提交**：

![uncommit](assest/35f09ecb74a149a99809e3e4e2867532.png)

### 1.2.1 原理

下图表示是 es 写操作流程，当一个写请求发送到 es 后，es将数据写入 `memory buffer`中，并添加事务日志（`translog`）。如果每次一条数据写入内存后立即写到硬盘文件上，由于写入的数据肯定是离散的，因此写入硬盘的操作也就是随机写入。硬盘随机写入的效率相当低，会严重降低 es 的性能。

因此 es 在设计时`memory buffer`和硬盘间加入了 Linux 的高速缓存（`File system cache`）来提高 es 的写效率。

当写请求发送到 es 后，es将数据暂时写入`memory buffer`中，此时写入的数据还不能被查询到。默认设置下，es 每1秒钟 将 `memory buffer`中的数据 `refresh`到 Linux的 `File system cache`，并清空`memory buffer`，此时写入的数据就可以被查询到了。

![es写操作流程](assest/image-20220119192815723.png)



### 1.2.2  refresh API

在Elasticsearch中，写入和打开一个新段的轻量的过程叫做*refresh*。默认情况下每个分片每秒自动刷新一次。这就是为什么我们说 Elasticsearch 是 **近**实时搜索：文档的变化并不是立即对搜索可见，但会在一秒之内变为可见。

这性行为可能会对新用户造成困惑：他们索引了一个文档然后尝试搜索它，但是没有搜到。这个问题的解决办法使用`refresh` API执行一次手动刷新：

```yaml
1. POST  /_refresh
2. POST  /my_blogs/_refresh
3. PUT   /my_blogs/_doc/1?refresh 
   {"test": "test"}
   PUT /test/_doc/2?refresh=true 
   {"test": "test"}
```

1. 刷新（Refresh）所有的索引
2. 只刷新（Refresh）`my_blogs`索引
3. 只刷新 文档

并不是所有的情况都需要每秒刷新。可能你正在使用 Elasticsearch 索引大量的日志文件，你可能想优化索引速度而不是近实时搜索，可以通过设置`refresh_interval`，降低每个索引的刷新频率。

```yaml
PUT /my_logs
{
  "settings": {
    "refresh_interval": "30s" 
  }
}
```

`refresh_interval`可以在既存索引上进行动态更新。在生产环境中，当你正在建立一个大的新索引时，可以先关闭自动刷新，待开始使用索引时，再把它们调回来：

```yaml
PUT /my_logs/_settings
{ "refresh_interval": -1 } 

PUT /my_logs/_settings
{ "refresh_interval": "1s" } 
```



## 1.3 持久化变更

[持久化变更-官网参考](https://www.elastic.co/guide/cn/elasticsearch/guide/current/translog.html)

### 1.3.1 原理

如果没有用`fsync`把数据从文件系统缓存 刷（flush）到硬盘，不能保证数据在断电甚至是程序正常退出之后依然存在。为了保证 Elasticsearch 的可靠性，需要确保数据变化被持久化到磁盘。

在 [动态更新索引]()，我们说一次完整的提交会将段刷到磁盘，并写入一个包含所有段列表的提交点。Elasticsearch 在启动或重新打开一个索引的过程中 ***使用这个提交点来判断哪些段隶属于当前分片***。

即使通过每秒刷新（refresh）实现了近实时搜索，我们仍然需要经常进行完整提交来确保能从失败中恢复。但是两次提交之间变化的文档怎么办？我们也不希望丢失这些数据。

Elasticsearch 增加了一个 *translog*，或者叫事务日志，在每一次对 Elasticsearch 进行操作时均进行了日志记录。通过 translog，整个流程看起来是下面这样：

1. 一个文档被索引之后，就会被添加到内存缓冲区，并且追加到了 translog，如下图：

   ![New documents are added to the in-memory buffer and appended to the transaction log](assest/elas_1106.png)

2. 刷新（refresh）使分片处于 ”刷新（refresh）完成后，缓存被清空但是事务日志不会“ 的状态，分片每秒被刷新（refresh）一次：

   - 这些在内存缓冲区的文档被写入到一个新的段中，且没有进行`fsync`操作。
   - 这个段被打开，使其可被搜索
   - 内存缓冲区被清空

   ![After a refresh, the buffer is cleared but the transaction log is not](assest/elas_1107.png)

3. 这个进程继续工作，更多的文档被添加到内存缓冲区和追加到事务日志（见下图：事务日志不断积累文档）。

   ![The transaction log keeps accumulating documents](assest/elas_1108.png)

4. 每隔一段时间——例如 translog 变得越大——索引被刷新（flush）；一个新的 translog 被创建，并且一个全量提交被执行（见下图：在刷新（flush）之后，段被全量提交，并且事务日志被清空）：

   - 所有在内存缓冲区的文档都被写入一个新的阶段
   - 缓冲区被清空
   - 一个提交点被写入硬盘
   - 文件系统缓存通过`fsync`被刷新（flush）
   - 老的 translog 被删除

   translog 提供所有还没有被刷到磁盘的操作的一个持久化记录。当Elasticsearch启动的时候，它会从磁盘中使用最后一个提交点去恢复已知的段，并且会重放 translog 中所有在最后一次提交后发生的变更操作

   translog 也被用来提供实时 CRUD。当你试着通过 ID 查询、更新、删除一个文档，它会在尝试从相应的段中检索之前，首先检查 translog 任何最近的变更。这意味着它总是能够实时地获取到文档的最新版本。

   ***在刷新（flush）之后，段被全量提交，并且事务日志被清空***

   ![After a flush, the segments are fully commited and the transaction log is cleared](assest/elas_1109.png)

### 1.3.2 flush API

这个 执行一个提交并且截断 translog 的行为在 Elasticsearch 被称为一次 *flush*。分片每30分钟被自动刷新（flush），或者在 translog 太大的时候也会被刷新。

`flush` API 可以被用来执行一个手工的刷新（flush）：

```yaml
1. POST /blogs/_flush
2. POST /_flush?wait_for_ongoin
```

1. 刷新（flush）`blogs`索引
2. 刷新（flush）所有索引并且等待所有刷新在返回前完成

很少需要自己动手执行`flush` 操作；通常情况下，自动刷新就够了。

这就是说，在重启节点或关闭索引之前执行 <font color='blue'>flush</font> 有益于你的索引。当 Elasticsearch 尝试恢复或重新打开一个索引，它需要重放 translog 中的所有操作，如果日志越短，恢复越快。

> **Translog 有多安全？**
>
> translog 的目的是保证操作不会丢失。这引出了这个问题：Translog 有多安全？
>
> 在文件被 `fsync` 到磁盘前，被写入的文件在重启之后就丢失了。默认 translog 是每5秒被 `fsync` 刷新到硬盘，或者在每次写请求完成之后执行（e.g. index, delete, update, bulk）。这个过程在主分片和复制分片都会发生。最终，基本上，这意味着在整个请求被 `fsync` 到主分片和复制分片的 translog 之前，你的客户端不会得到一个 200 OK 响应。
>
> 在每次请求后都执行一个 fsync 会带来一些性能损失，尽管实践表明这种损失相对较小（特别是 bulk 导入，它在一次请求中平摊了大量文档的开销）。
>
> 但是对于一些大容量的偶尔丢失几秒数据问题也并不严重的集群，使用异步的 fsync 还是比较有益的。比如，写入的数据被缓存到内存中，再每5秒执行一次 `fsync`。
>
> 这个行为可以通过设置`durability` 参数为 `async` 来启用：
>
> ```yaml
> PUT /my_index/_settings 
> {
>  "index.translog.durability":  "async", 
>  "index.translog.sync_interval":  "5s"
> }
> ```
>
> 这个选项可以针对索引单独设置，并且可以动态进行修改。如果你决定使用异步 translog 的话，你需要保证在发生 crash 时，丢掉 `sync_interval`时间段的数据也无所谓。请在决定前知晓这个特性。
>
> 如果你不确定这个行为的后果，最好使用默认参数（`"index.translog.sync_interval" : "request"`）来避免数据丢失。

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