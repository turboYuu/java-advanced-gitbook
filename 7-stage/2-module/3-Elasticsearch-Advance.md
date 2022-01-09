第三部分 Elasticsearch高级应用

# 1 映射高级

## 1.1 地理坐标点数据类型

## 1.2 动态映射

## 1.3 自定义动态映射

### 1.3.1 日期检测

### 1.3.2 dynamic templates

# 2 Query SQL

## 2.1 查询所有（match_all_query）

## 2.2 全文搜索（full-text query）

### 2.2.1 匹配搜索（match query）

### 2.2.2 短语搜索（match phrase query）

### 2.2.3 query_string 查询

### 2.2.4 多字段匹配搜索（mutilation match query）

## 2.3 词条级搜索（term-level queries）

### 2.3.1 词条搜索（term query）

### 2.3.2 词条集合搜索（terms query）

### 2.3.3 范围搜索（range query）

### 2.3.4 不为空搜索（exists query）

### 2.3.5 词项前缀搜索（prefix query）

### 2.3.6 通配符搜索（wildcard query）

### 2.3.7 正则搜索（regexp query）

### 2.3.8 模糊搜索（fuzzy query）

### 2.3.9 ids搜索（id集合查询）



## 2.4 复合搜索（compound query）

## 2.5 排序

## 2.6 分页

## 2.7 高亮

## 2.8 文档批量操作（bulk 和 mget）

### 2.8.1 mget 批量查询

### 2.8.2 bulk 批量增删改

# 3 Filter DSL

# 4 定位非法搜索及原因

# 5 聚合分析

## 5.1 聚合介绍

## 5.2 指标聚合

## 5.3 桶聚合

# 6 Elasticsearch零停机索引重建

## 6.1 说明

## 6.2 方案一：外部数据导入方案

## 6.3 方案二：基于scroll+bulk+索引别名方案

## 6.4 方案三：Reindex API 方案

## 6.5 小结

# 7 Elasticsearch Suggester 智能搜索建议

# 8 Elasticsearch Java Client

## 8.1 说明

## 8.2 SpringBoot 中使用 RestClient