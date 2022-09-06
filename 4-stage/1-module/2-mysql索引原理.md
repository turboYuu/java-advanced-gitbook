> 第二部分 MySQL索引原理

# 1 索引类型

索引可以提升查询速度，会影响 where 查询，以及 order by 排序。MySQL 索引类型如下：

- 从索引存储结构划分：B Tree 索引、Hash索引、FULLTEXT 全文索引、R Tree 索引
- 从应用层次划分：普通索引、唯一索引、主键索引、复合索引
- 从索引键值类型划分：主键索引、辅助索引（二级索引）
- 从数据存储和索引键值逻辑关系划分：聚集索引（聚簇索引）、非聚集索引（非聚簇索引）

## 1.1 普通索引

这是最基本的索引类型，基于普通字段建立的索引，没有任何限制。

创建普通索引的方法如下：

- CREATE INDEX <索引的名字> ON tablename (字段名);
- ALTER TABLE tablename ADD INDEX[索引的名字] (字段名);
- CREATE TABLE tablename ([...], INDEX[索引的名字] (字段名));

![image-20220906191904447](assest/image-20220906191904447.png)

## 1.2 唯一索引

与 “普通索引” 类似，不同的就是：索引字段的值必须唯一，但允许有空值。在创建或修改时追加唯一约束，就会自动创建对应的唯一索引。

创建唯一索引的方法如下：

- CREATE UNIQUE INDEX <索引的名字> ON tablename (字段名);
- ALTER TABLE tablename ADD UNIQUE INDEX [索引的名字] (字段名);
- CREATE TABLE tablename ([...], UNIQUE[索引的名字] (字段名))



## 1.3 主键索引

它是一种特殊的唯一索引，不允许有空值。在创建或修改表时追加主键约束即可，每个表只能有一个主键。

创建主键索引的方法如下：

- CREATE TABLE tablename ([...], PRIMARY KEY (字段名));
- ALTER TABLE tablename ADD PRIMARY KEY (字段名);

## 1.4 复合索引

单一索引是指索引列为一列的情况，即新建索引的语句只能实施在一列；用户可以在多个列上建立索引，这种索引叫做组复合索引（组合索引）。复合索引可以代替多个单一索引，相比多个单一索引，复合索引所需的开销更小。

索引同时有两个概念叫做 **窄索引** 和 **宽索引** ，窄索引 是指索引列为 1-2 列的索引；宽索引 也就是 索引列超过 2 列的索引。设计索引的一个重要原则就是能用窄索引 就不用 宽索引，因为窄索引往往比组合索引更有效。

创建组合索引的方法如下：

- CREATE INDEX <索引的名字> ON tablename (字段名1, 字段名2, ....);
- ALTER TABLE tablename ADD INDEX [索引的名字] (字段名1, 字段名2, ...);
- CREATE TABLE tablename ([...], INDEX[索引的名字] (字段名1, 字段名2, ...));

复合索引使用注意事项：

- 何时使用复合索引，要根据 where 条件建索引，注意不要过多使用索引，过多使用会对更新操作效率有很大影响。
- 如果表已经建立了 (col1, col2)，就没有必要再单独建立 (col1)；如果现在有 (col1) 索引，如果查询需要 col1 和 col2 条件，可以建立 (col1,col2) 复合索引，对于查询有一定提高。

## 1.5 全文索引

查询操作在数据量比较少时，可以使用 like 模糊查询，但是对于大量的文版数据检索，效率很低。如果使用全文索引，查询速度会比 like 快很多倍。在 MySQL 5.6 以前的版本，只有 MyISAM 存储引擎支持全文索引，从 MySQL 5.6 开始 MyISAM 和 InnoDB 存储引擎均支持。

创建全文索引的方法如下：

- CREATE FULLTEXT INDEX <索引的名字> ON tablename (字段名);
- ALTER TABLE tablename ADD FULLTEXT [索引的名字] (字段名);
- CREATE TABLE tablename ([...], FULLTEXT KEY [索引的名字] (字段名));

和常用的 like 模糊查询不同，全文索引有自己的语法格式，使用 match 和 against 关键字，比如

```sql
select * from user where match(name) against('aaa');
```



# 2 索引原理

# 3 索引分析与优化

# 4 查询优化