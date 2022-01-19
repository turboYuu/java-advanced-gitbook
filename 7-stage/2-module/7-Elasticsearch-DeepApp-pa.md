第七部分 Elasticsearch深度应用及原理剖析

# 1 索引文档写入和近实时搜索原理

## 1.1 基本概念

### 1.1.1 Segment in Lucene

### 1.1.2 Commits in Lucene

### 1.1.3 Translog

### 1.1.4 Refresh in Elasticsearch

### 1.1.5 Flush in Elasticsearch

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