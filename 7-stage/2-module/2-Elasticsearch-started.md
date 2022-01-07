第二部分 Elasticsearch入门使用

Elasticsearch是基于Lucene的全文检索引擎，本质也是存储和检索数据。ES种的很多概念与MySQL类似。

# 1 核心概念

- **索引（index）**

  类似的数据放在一个索引，非类似的数据放不同索引，一个索引也可以理解成一个关系型数据库。

- **类型（type）**

  代表document属于index中的哪个类别（type）也有一种说法：一种type就像是数据库的表。

  注意ES每个大版本之间的区别很大：

  ES 5.x 中的一个index可以有多种type；

  ES 6.x 中的一个index只能有一种type；

  ES 7.x 以后要逐渐移除type这个概念。

- **映射（mapping）**

  mapping定义了每个字段的类型等信息。相等于关系型数据库中的表结构。

  常用数据类型：text、keyword、number、array、range、bolean、date、go_point、ip、nested、object

  https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html#_multi_fields_2

  | 关系型数据库（如MySQL） | 非关系型数据库（Elasticsearch） |
  | ----------------------- | ------------------------------- |
  | 数据库Database          | 索引 index                      |
  | 表 Table                | 索引 index类型（原为Type）      |
  | 数据行 Row              | 文档 Document                   |
  | 数据列 Column           | 字段 Field                      |
  | 约束 Schema             | 映射 Mapping                    |

  

# 2 Elasticsearch API 介绍



# 3 安装配置Kibana



# 4 Elasticsearch继承IK分词器



# 5 索引操作（创建、查看、删除）



# 6 映射操作



# 7 文档增删改查及局部更新