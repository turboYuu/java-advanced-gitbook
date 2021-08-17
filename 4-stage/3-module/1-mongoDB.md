MongoDB是一款高性能的NoSQL（Not Only SQL 不仅仅SQL）数据库

# 第一部分 MongoDB体系结构

## 1.1 NoSQL 和 MongoDB

> NoSQL = Not Only SQL，支持类似SQL的功能，与Relational Database相辅相成。其性能较高，不使用SQL意味着没有结构化的存储要求（SQL为结构化的查询语句），没有约束之后架构更加灵活。
>
> NoSQl数据库四大家族：列存储Hbase，键值(Key-Value)存储 Redis，图像存储Neo4j，文档存储MongoDB。
>
> MongoDB是一个基于分布式文件存储的数据库，由C++编写，可以为WEB应用提供可扩展、高性能、易部署的数据存储解决方案。
>
> MongoDB是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库中功能最丰富、最像关系数据库的。在高负载的情况下，通过添加更多的节点，可以保证服务器性能。

## 1.2 MongoDB 体系结构

![image-20210814164948465](assest/image-20210814164948465.png)

## 1.3 MongoDB和RDBMS(关系型数据库)对比

| RDBMS                             | MongoDB                                       |
| --------------------------------- | --------------------------------------------- |
| database(数据库)                  | database(数据库)                              |
| table（表）                       | collection（集合）                            |
| row（行）                         | document（BSON文档）                          |
| column（列）                      | field（字段）                                 |
| index（唯一索引，主键索引）       | index（支持地理位置索引，全文索引、哈希索引） |
| join（主外键关联）                | embedded Document（嵌套文档）                 |
| primary key（指定1至N个列做主键） | primary key（指定_id field作为主键）          |

## 1.4 BSON

BSON是一类json的一种二进制形式的存储格式，简称Binary JSON，和JSON一样，支持内嵌的文档的数组对象，但是BSON有JSON没有的一些数据类型，如Date和Binary Data类型。BSON可以做为网络数据交换的一种存储形式，是一种schema-less的存储形式，优点是灵活性高，但是缺点是空间利用率不是很理想。

{key:value,key2:value2}这时一个BSON的例子，其中key是字符串类型，后面的value值的类型一般是字符串，double，Array，ISODate等类型。

BSON有三个特点：轻量性，可便利性，高效性

## 1.5 BSON在MongoDB中的使用

MongoDB使用了BSON结构来存储数据和网络数据交换。把这种格式转化成一文档这个概念（Document），这里的一个Document也可以理解成关系数据库中的一条记录（Record），只是这里的Document的变化更丰富一些，如Document可以嵌套。

MongoDB中的Document中可以出现的数据类型

![image-20210814212027572](assest/image-20210814212027572.png)

## 1.6 MongoDB在Linux的安装

```
1.下载社区版 mongoDB 4.1.3，上传至linux
2.解压 tar -xvf mongodb-linux-x86_64-4.1.3.tgz
3.启动
	./bin/mongod
4.指定配置文件方式启动
	./bin/mongod -f mongo.conf
```

配置文件样例

```
dbpath=/data/mongo/
port=27017
bind_ip=0.0.0.0
fork=true
logpath = /data/mongo/MongoDB.log
logappend = true
auth=false
```

![image-20210814171109592](assest/image-20210814171109592.png)

![image-20210814171202492](assest/image-20210814171202492.png)

指定配置文件启动

![image-20210814172345492](assest/image-20210814172345492.png)

![image-20210814172408212](assest/image-20210814172408212.png)

## 1.7 MongoDB启动和参数说明

```
参数			说明
dbpath		  数据库目录，默认/data/db
port		  监听端口，默认27017
bind_ip		  监听IP地址，默认全部可以访问
fork		  是否已后台启动方式登录
logpath       日志路径
logappend     是否追加日志
auth          是否开启用户密码登录
config		  指定配置文件
```

## 1.8 mongo shell的启动

```
启动mongo shell
	./bin/mongo
指定主机和端口的方式启动
	./bin/mongo --host=主机ip --port=端口
```

![image-20210814174207990](assest/image-20210814174207990.png)

## 1.9 MongoDB GUI工具

### 1.9.1 MongoDB Compass Community

### 1.9.2 NoSQLBooster（mongobooster）



# 第二部分 MongoDB命令

## 2.1 MongoDB的基本操作

```
查看数据库
	show dbs
切换数据库 如果没有对应的数据库则创建（数据库中要创建集合，才会真正创建出数据库）
	use 数据库名
创建集合
	db.createcollection("集合名")
查看集合
	show tables
	show collections
删除集合
	db.集合名.drop()
删除当前数据库
	db.dropDatabase()
```



## 2.2 MongoDB集合数据操作（CURD）

https://docs.mongodb.com/guides/

### 2.2.1 数据添加

1）插入单条数据db.集合名.insert(文档)

文档的数据结构和JSON基本一致。所有存储在集合中的数据都是BSON格式。BSON是一种类json的一种二进制形式的存储格式，简称Binary JSON。

2）例如：

db.lg_resume_preview.insert({name:"张晓峰",birthday:new ISODate("2000-07-01"),expectSalary:45000,city:'beijing'})

没有指定_id这个字段 系统会自动生成一个12字节BSON类型数据，有以下格式：

前4个字节表示时间戳 ObjectId("对象Id字符串").getTimestamp()来获取

接下来的3个字节是机器标识码

紧接的两个字节由进程id组成（PID）

最后三个字节是随机数。

3）插入多条数据

db.集合名.insert([文档,文档])

![image-20210815012306191](assest/image-20210815012306191.png)

### 2.2.2 数据查询

**比较条件查询**

db.集合名.find(条件)

| 操作   | 条件格式           | 例子 | RDBMS |
| ------ | ------------------ | ---- | ----- |
| 等于   | {key:value}        |      |       |
| 大于   | {key:{$gt:value}}  |      |       |
| 小于   | {key:{$lt:value}}  |      |       |
| 大于等 | {key:{$gte:value}} |      |       |
| 小于等 | {key:{$lte:value}} |      |       |
| 不等于 | {key:{$ne:value}}  |      |       |

```
db.lg_resume_preview.find({})
db.lg_resume_preview.find()
db.lg_resume_preview.find({name:"李四"})
db.lg_resume_preview.find({expectSalary:45000)
db.lg_resume_preview.find({expectSalary:{$eq: 45000}})
db.lg_resume_preview.find({expectSalary:{$gt: 45000}})
db.lg_resume_preview.find({expectSalary:{$lt: 45000}})
```

**逻辑条件查询**

```
andt条件
MongoDB的find()方法可以传入多个键(key)，每个键(key)以逗号隔开，即常规SQL的AND条件
	db.集合名.find({key1:value1,key2:value2}).pretty()
	
or条件
	db.集合名.find({$or:[{key1:value1},{key2:value2}]}).pretty()
	
not条件
	db.集合名.find({key:{$not:{$操作符:value}}}).pretty()
```

```
db.lg_resume_preview.find({city:"shanghai",expectSalary:45000})
db.lg_resume_preview.find({$and: [{city:"shanghai"},{expectSalary:45000}]})

db.lg_resume_preview.find({$or: [{city:"shanghai"},{expectSalary:45000}]})
db.lg_resume_preview.find({city: {$not: {$eq: "beijing"}}})
```

**分页查询**

db.集合名.find({条件}).sort({排序字段:排序方式}).skip(跳过的行数).limit(一页显示多少数据)

```
db.lg_resume_preview.find({expectSalary:45000).sort({ city:-1 }).skip(4).limit(2)
```

### 2.2.3 数据更新 

```
$set:设置字段值
$unset:删除指定字段
$inc:对修改的值进行自增
db.集合名.update(
	<query>,
	<update>,
	{
		upsert:<boolean>,
		multi:<boolean>,
		writeConcern:<document>
	}
)

参数说明：
query：update的查询条件，类似sql update查询内where后面的。
update：update的对象和一些更新的操作符（如$set,$inc...）等，也可以理解为sql update中set后面的
upsert：可选，这个参数的意思是，如果不存在update的记录，是否插入objNew，true为插入，默认false,不插入
multi：可选，默认false,只更新找到的第一条记录，如果为true,就把按条件查询出来的多条记录全部更新。
writeConcern：可选，用来指定mongoDB对写操作的回执行为比如写的行为是否需要确认。

举例：
db.集合名.update({条件},{$set:{字段名:值}},{multi:true})
```

```
writeConcern 包括以下字段：
{w:<value>,j:<boolean>,wtimeout:<number>}
	w:指定写操作传播到的成员数量
比如：

```

```
-- 数据更新
/** 更新 张三的工资为40000*/
db.lg_resume_preview.update({name:"张三"}, 
    {$set:{expectSalary:40000}}, 
    { multi: false, upsert: false}
)

db.lg_resume_preview.update({expectSalary:45000}, 
    {$set:{expectSalary:49000}}, 
    { multi: true, upsert: true}
)
/** inc*/
db.lg_resume_preview.update({name:"李四"}, 
    {$inc:{expectSalary:-900}}, 
    { multi: false, upsert: false}
)
/** 删除字段*/
db.lg_resume_preview.update({name:"李四"}, 
    {$unset:{expectSalary:""}}, 
    { multi: false, upsert: false}
)
/**没有操作符，会删除其他字段，只留expectSalary*/
db.lg_resume_preview.update({name:"李四"}, 
    {expectSalary:18000}, 
    { multi: false, upsert: false}
)
```

### 2.2.4 数据删除

```
db.collection.remove(
	<query>,
	{
		justOne:<boolean>,
		writeConcern:<document>
	}
)

参数说明：
query：可选，删除的文档的条件
justOne:可选，如果设置为true或1,则只删除一个文档，如果不设置该参数，使用默认值false,则删除所有匹配条件的文档。
writeConcern:可选，用来指定mongod对写操作的回执行为。
```

```
-- 删除数据
db.lg_resume_preview.remove({expectSalary:18000}, {justOne: true})
db.lg_resume_preview.find({city:"beijing"})
db.lg_resume_preview.remove({city:"beijing"}, {justOne: true})
db.lg_resume_preview.remove({})
```

## 2.3 MongoDB聚合操作

### 2.3.1 聚合操作简介

聚合是MongoDB的高级查询语言，它允许我们通过转化合并由多个文档的数据来生成新的在单个文档中不存在的文档信息。一般都是将记录按条件分组之后进行一系列求最大值、最小值、平均值的简单操作，也可以对记录进行复杂的数据统计，数据挖掘等操作。聚合操作的输入是集中的文档，输出可以是一个文档也可以是多个文档。

### 2.3.2 MongoDB聚合操作分类

- 单目的聚合操作（Single Purpose Aggregation Operation）
- 聚合管道（Aggregation Pipeline）
- MapReduce编程模型

### 2.3.3 单目的聚合操作

单目的聚合命令常用的有：count()和distinct()

```
db.lg_resume_preview.find().count()
db.lg_resume_preview.count()
db.lg_resume_preview.distinct("city")
```

### 2.3.4 聚合管道（Aggregation Pipeline）

```
db.collection.aggregater(AGGREGATE_OPERATION)
如：
db.lg_resume_preview.aggregate([{$group: { _id: "$city",city_count:{$sum: 1}}}])
```

MongoDB中聚合（aggregate）主要用于统计数据（诸如统计平均值，求和等），并返回计算后的数据结果。

表达式：处理输入文档并输出。表达式只能用于计算当前聚合管道的文档，不能处理其他的文档。

| 表达式    | 描述                                         |
| --------- | -------------------------------------------- |
| $sum      | 计算总和                                     |
| $avg      | 计算平均值                                   |
| $min      | 最小值                                       |
| $max      | 最大值                                       |
| $push     | 在结果文档中插入值到一个数组中               |
| $addToSet | 在结果文档中插入值到一个数组中，但数据不重复 |
| $first    | 根据资源文档的排序获取第一个文档数据         |
| $last     | 根据资源文档的排序获取最后一个文档数据       |

MongoDB中使用`db.COLLECTION_NAME.aggregate([{},...])`方法来构建和使用聚合管道，内阁文档通过一个由一个或多个阶段（stage）组成的管道，经过一系列的处理，输出相应的结果。

MongoDB的聚合管道将MongoDB文档在一个管道处理完毕后将结果传递给下一个管道处理。管道操作可以重复。

常用的几个操作：

- $group:将集合中的文档分组，可用于统计结果。
- $project:修改输入文档的结构。可以用来重命名，增加或删除域，也可以用于创建计算结果以及嵌套文档。
- $match:用于过滤数据，只输出符合条件的文档。$match使用MongoDB的标准查询操作。
- $limit: 用来限制MongoDB聚合管道返回的文档数。
- $skip:将输入文档排序后输出。
- $geoNear:输出接近某一地理位置的有序文档。

```
/**统计city出现个数*/
db.lg_resume_preview.aggregate([{$group: { _id: "$city",city_count:{$sum: 1}}}])

/**统计city平均薪资*/
db.lg_resume_preview.aggregate([{$group: { _id: "$city",avgSalary:{$avg: "$expectSalary"}}}])

/**统计city min薪资*/
db.lg_resume_preview.aggregate([{$group: { _id: "$city",expectSalary:{$min: "$expectSalary"}}}])

/**统计city max薪资*/
db.lg_resume_preview.aggregate([{$group: { _id: "$city",expectSalary:{$max: "$expectSalary"}}}])

db.lg_resume_preview.aggregate([{$group: { _id: "$city",name:{$first: "$name"}}}])
db.lg_resume_preview.aggregate([{$group: { _id: "$city",city_name:{$push: "$city"}}])
db.lg_resume_preview.aggregate([{$group: { _id: "$city",city_name:{$addToSet: "$city"}}}])

db.lg_resume_preview.aggregate([{$group: { _id: "$city",avgSalary:{$avg: "$expectSalary"}}},
{$project: {city:"$city",sal:"$avgSalary"}}])

db.lg_resume_preview.aggregate([{$group: { _id: "$city",city_count:{$sum: 1}}},
{$match: {city_count:{$gt:2}}}])
```



![image-20210815182827697](assest/image-20210815182827697.png)

![image-20210815183032744](assest/image-20210815183032744.png)

### 2.3.5 MapReduce编程模型

Pipeline查询速度快于MapReduce，但是MapReduce的强大之处在于能够在多台Server上并行执行复杂的聚合逻辑。MongoDB不允许Pipeline的单个聚合操作占用过多的系统内存，如果一个聚合操作消耗20%以上的内存，那么MongoDB直接停止操作，并向客户端输出错误消息。

**MapReduce是一种计算模型，简单说就是将大批量的工作（数据）分解（MAP）执行，然后再将结果合并成最终结果（REDUCE）。**

```
db.lg_resume_preview.mapReduce(
      function () {emit(this.city, this.expectSalary)}, //mapFunction
      function(key, values){return Array.avg(values)},//reduceFunction
      {
         query:{expectSalary:{$gt:15000}},
         finalize:function(key,value){
             return value+5000
         },verbose:false,
         out: "citySalary"
      })
```

使用MapReduce要实现两个函数Map函数和Reduce函数，Map函数调用emit(key,value)，遍历collection中所有的记录，将key与value传递给Reduce函数进行处理。

**参数说明：**

- map:是JavaScript函数，负责将每一个输入文档转换为零或多个文档，生成键值对序列，作为reduce函数参数
- reduce：是JavaScript函数，对Map操作的输出做合并的化简操作（将key-value变成key-values，也就是把values数组变成一个单一的值value）
- out：统计结果存放集合
- query: 一个筛选条件，只有满足条件的文档才会调用map函数。
- sort：和limit结合的sort排序参数（也是在发往map函数前给文档排序），可以优化分组机制
- limit: 发往map函数的文档数量上线（没有limit,单独使用sort的用处不大）
- finalize:可以对reduce输出结果再一次修改
- verbose:是否包括信息中的时间信息，默认为false

![image-20210816161510897](assest/image-20210816161510897.png)

# 第三部分 MongoDB索引Index

## 3.1 什么是索引

> 索引是一种单独的、物理的对数据库表中一列或多列的值进行排序的一种存储结构，它是某个表中一列或若干列值的集合和相应的指向表中物理标识这些值的数据页的逻辑指针清单。默认情况下Mongo在一个集合（collection）创建时，自动地对集合地_id创建了唯一索引。

## 3.2 索引类型

### 3.2.1 单键索引（Single Field）

MongoDB支持所有数据类型中地单个字段索引，并且可以在文档地任何字段上定义。

对于单个字段索引，索引键的排序顺序无关紧要，因为MongoDB可以在任意方向读取索引。

单个例上创建索引：

```
db.集合名.createIndex({"字段名":排序方式})

db.lg_resume_preview.createIndex({name:1})
db.lg_resume_preview.getIndexes()
```

**特殊单键索引 过期索引TTL (Time To Live)**

TTL索引是MongoDB中一种特殊的索引，可以支持文档在一定时间之后自动过期删除，目前TTL索引只能在单字段上建立，**并且字段类型必须是日期类型。**

```
db.集合名.createIndex({"日期字段":排序方式},{expireAfterSeconds:秒数})
```

过期索引或删除索引上的数据

```
db.lg_index_test.insert({name:"test1",salary:38000,birthday:new ISODate("2020-08-12")})

db.lg_index_test.createIndex({birthday:1},{expireAfterSeconds:5})
```

### 3.2.2 复合索引（Compound Index）

官网参考：https://docs.mongodb.com/manual/core/index-compound/

在多个字段的基础上搜索表，需要在MongoDB中创建复合索引。复合索引支持基于多个字段的索引，这种扩展了索引的概念并将它们扩展到索引中的更大域。

创建复合索引的注意事项：字段顺序、索引方向。

```
db.集合名.createIndex({"字段名1":排序方式,"字段名2":排序方式})
```

### 3.2.3 多键索引（Multikey indexes）

针对属性包括数组数据的情况，MongoDB支持针对数组中每一个element创建索引，Multikey indexes支持Strings，numbers和nested documents

### 3.2.4 地理空间索引（Geospatial Index）

针对地理空间坐标数据创建索引。

2dsphere索引，用于存储和查找球面上的点

2d索引，用于存储和查找平面上的点

```
db.company.insert(
   {
     loc : { type: "Point", coordinates: [ 116.482451, 39.914176 ] },
     name: "大望路地铁",
     category : "Parks"
   }
)
db.company.ensureIndex({loc:"2dsphere"})
db.company.getIndexes()
db.company.find({
  "loc" : {
      "$geoWithin" : {
       "$center":[[116.482451,39.914176],0.05] 
      }
  } 
})

/** 计算中心点最近的三个点 */
db.company.aggregate([
   {
     $geoNear: {
        near: { type: "Point", coordinates: [116.482451,39.914176 ] },
        key: "loc",
        distanceField: "dist.calculated"
     }
   },
   { $limit: 3 }
])
```

### 3.2.5 全文索引

MongoDB提供了针对string内容的文本查询，Text Index支持任意属性值为string或string数组元素的索引查询。注意：**一个集合仅支持最多一个Text Index**，中文分词不理想，推荐ES。

```
db.集合名.createIndex({"字段":"text"})
db.集合名.find({"$text":{"$search":"two"}})

db.textTextIndex.insert({id:4,name:"test4",description:"three world three dream",city:"dj"})
db.textTextIndex.find()

db.textTextIndex.createIndex({description:"text"})
db.textTextIndex.find({"$text":{"$search":"two"}})
```

### 3.2.6 哈希索引 Hashed Index

官网参考：https://docs.mongodb.com/manual/core/index-hashed/

针对属性的哈希值进行索引查询，当要使用hashed index时，MongoDB能够自动计算hash值，无需程序计算hash值。注：hash index仅支持等于查询，不支持范围查询。

```
db.集合名.createIndex({"字段":"hashed"})
```

## 3.3 索引和explain

### 3.3.1 索引管理

创建索引并在后台运行

```
db.COLLECTION_NAME.createIndex({"字段":1},{background:true})

db.textTextIndex.createIndex({name:1},{background:true})
```

获取针对某个集合的索引

```
db.textTextIndex.getIndexes()
```

索引大小

```
db.textTextIndex.totalIndexSize()
```

索引重建

```
db.textTextIndex.reIndex()
```

索引删除

```
db.COLLECTION_NAME.dropIndex("INDEX-NAME") 
db.COLLECTION_NAME.dropIndexes()
注意: _id 对应的索引是删除不了的
```

### 3.3.2 explain分析

循环插入100万条数据，不使用索引字段查询查看执行计划，然后给某个字段建立索引，使用索引字段作为查询条件，再查看执行计划进行分析。

explain()接收不同的参数，通过设置不同的参数，可以查看更详细的查询计划。

- queryPlanner：queryPlanner是默认参数，具体执行计划信息参考下面表格》
- executionStats：executionStats会返回执行计划的一些统计信息（有些版本中和allPlansExecution等同）。
- allPlansExecution：allPlansExecution用来获取所有执行计划，结果参数基本与上文相同。

**1、queryPlanner默认参数**

| 参数                           | 含义                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| plannerVersion                 | 查询计划版本                                                 |
| namespace                      | 要查询的集合（该值返回的是该query所查询的表）数据库.集合     |
| indexFilterSet                 | 针对该query是否有indexFilter                                 |
| parsedQuery                    | 查询条件                                                     |
| winningPlan                    | 被选中的执行计划                                             |
| winningPlan.stage              | 被选中执行计划的stage(查询方式)，常见的有：COLLSCAN/全表扫描：（应该知道就是CollectionScan，就是所谓的“集合扫描”， 和mysql中table scan/heap scan类似，这个就是所谓的性能最烂最无奈的由来）、IXSCAN/索引扫描：（是IndexScan，这就说明 我们已经命中索引了）、FETCH/根据索引去检索文档、 SHARD_MERGE/合并分片结果、IDHACK/针对_id进行查询等 |
| winningPlan.inputStage         | 用来描述子stage，并且为其父stage提供文档和索引关键字。       |
| winningPlan.stage的child stage | 如果此处是IXSCAN，表示进行的是index scanning。               |
| winningPlan.keyPattern         | 所扫描的index内容                                            |
| winningPlan.indexName          | winning plan所选用的index。                                  |
| winningPlan.isMultiKey         | 是否是MultiKey，此处返回是false，如果索引建立再array上，此处将是true。 |
| winningPlan.direction          | 此query的查询顺序，此处是forward，如果用了.sort({字段:-1}) 将显示backward。 |
| filter                         | 过滤条件                                                     |
| winningPlan.indexBounds        | winningplan所扫描的索引范围,如果没有制定范围就是[MaxKey, MinKey]，这主要是直接定位到mongodb的chunck中去查找数据，加快数据读取。 |
| rejectedPlans                  | 被拒绝的执行计划的详细返回，其中具体信息与winningPlan的返回中意义相同，故不在此赘述） |
| serverInfo                     | MongoDB服务器信息                                            |

**2、executionStats参数**

| 参数                                   | 含义                                                         |
| -------------------------------------- | ------------------------------------------------------------ |
| executionSuccess                       | 是否执行成功                                                 |
| nReturned                              | 返回的文档数                                                 |
| executionTimeMillis                    | 执行耗时                                                     |
| totalKeysExamined                      | 索引扫描次数                                                 |
| totalDocsExamined                      | 文档扫描次数                                                 |
| executionStages                        | 这个分类下描述执行的状态                                     |
| stage                                  | 扫描方式，具体可选值与上文的相同                             |
| nReturned                              | 查询结果数量                                                 |
| executionTimeMillisEstimate            | 检索document获得数据的时间                                   |
| inputStage.executionTimeMillisEstimate | 该查询扫面文档index所用时间                                  |
| works                                  | 工作单元数，一个查询会分解成小的工作单元                     |
| advanced                               | 优先返回的结果数                                             |
| docsExamined                           | 文档检查数目，与totalDocsExamined一致。检查了总共的document 个数，而从返回上面的nReturned数量 |

**3、executionStats返回逐层分析**

第一层，executionTimeMillis 最为直观explain返回值是executionTimeMillis值，指的是这条语句的执行时间，这个值当然是希望越少越好。

其中有3个executionTimeMillis，分别是：

executionStats.executionTimeMillis 该query的整体查询时间。

executionStats.executionStages.executionTimeMillisEstimate 该查询检索document获取数据的时间。

executionStats.executionStages.inputStage.exexutionTimeMillisEstimate 该查询扫描文档index所用时间。

第二层，index与document扫描数与查询返回条目数，主要讨论3个返回项 nReturned、totalKeysExamined、totalDocsExamined，分别代表该条查询返回的条目，索引扫描条目、文档扫描条目。这些都是直观地影响到executionTimeMillis，扫描地越少速度越快。对于一个查询最理想地状态是：nReturded=totalKeysExamined=totalDocsExamined。

第三层，stage状态分析，那又是什么影响到了totalKeysExamined和totalDocsExamined？是stage的类型。

类型列举如下：

COLLSCAN：全表扫描

IXSCAN：索引扫描

TETCH：根据索引去检索指定document

SHARD_MERGE：将各个分片返回数据进行merge

SORT：表明在内存中进行了排序

LIMIT：使用limit限制返回数

SKIP：使用skip进行跳过

IDHACK：针对_id进行查询

SHARDING_FILTER：通过mongos对分片数据进行查询

COUNT：利用db.coll.explain().count()之类及逆行count运算

TEXT：使用全文索引进行查询时候的stage返回

PROJECTION：限定返回字段时stage的返回

```
/**限定返回字段,去掉_id*/
db.lg_resume.find({name:{$gt:"test222333"}},{_id:0}).explain("executionStats")
```

对于普通查询，希望看到stage的组合（查询的时候尽可能用上索引）：

Fetch+IDHACK

Tetch+IXSCAN

Limit+(Fetch+IXSCAN)

PROJECTION+IXSCAN

不希望看到包含如下的stage：

COLLSCAN

SORT

COUNT

**4、allPlansExecution参数**

```
queryPlanner 参数 和 executionStats的拼接
```



![image-20210816181656466](assest/image-20210816181656466.png)

```
示例：
db.lg_resume.find({name:"test11011"}).explain("executionStats")
db.lg_resume.find({id:{$gt:222333}}).explain()
/**限定返回字段*/
db.lg_resume.find({name:{$gt:"test222333"}},{_id:0}).explain("executionStats")
```

## 3.4 慢查询分析

1、开启内置的查询分析器，记录读写操作效率

```
db.setProfilingLevel(n,m) n的取值可选0,1,2

0 表示不记录
1 表示记录慢速操作，如果值为1，m必须赋值，单位为ms，用于定于慢速查询时间阈值
2 表示记录所有的读写操作
```

2、查询监控结果

```
db.system.profile.find().sort({millis:-1}).limit(3)
```

3、分析慢速查询

应用程序设计不合理，不正确的数据模型，硬件配置问题，缺少索引等

4、解读explain结果 确定是否缺少索引



## 3.5 MongoDB索引底层实现原理

MongoDB时文档型的数据库，它使用BSON格式保存数据，比关系型数据库存储更方便。MySQL是关系型数据库，数据的关联性非常强，底层索引租住数据使用B+树，B+树由于数据全部存储在叶子节点，并且通过指针串联在一起，这样就容易进行区间遍历甚至全部遍历。**MongoDB使用B-树**，所有节点都有Data域，只要找到指定索引就可以进行访问，单次查询从结构上来看要快于MySQL。

B-树是一种自平衡的搜索树，形式简单：

![image-20210817012020299](assest/image-20210817012020299.png)

B-树的特点：

- 多路非二叉树
- 每个节点 即保存数据 又保存索引
- 搜索时 相当于二分查找

B+树是B-树的变种

![image-20210817012159726](assest/image-20210817012159726.png)

B+树的特点：

- 多路非二叉树
- 只有叶子节点保存数据
- 搜索时 也相当于二分查找
- 增加了 相邻节点指针

从上面可以看出最核心的区别有2个：一个是数据保存位置，一个是相邻节点的指向。就是这两个造成MongoDB和MySQL的差别。

- B+树相邻节点的指针可以大大增加区间访问性，可使用在范围查询等，而B-树每个节点key和data在一起适合随机读写，而区间查找效率很差。
- B+树更适合外部存储，也就是磁盘存储，使用B-结构的话，每次磁盘预读中的很多数据是用不上的数据。因此，它没能利用好磁盘预读提供的数据。B+数 由于节点内无data域，每个节点能索引的范围更大更精确。
- 注意这个区别相当重要，是基于（1）（2）的，**B-数每个节点即保存数据又保存索引，树的深度小，所以磁盘IO次数少**，B+树只有叶子节点保存数据，较B-树而言深度大磁盘IO多，但是区间访问比较好。

# 第四部分 MongoDB应用实战

## 4.1 MongoDB的适用场景

> - 网站数据
> - 缓存
> - 大尺寸、低价值的数据
> - 高伸缩性的场景
> - 用于对象及JSON数据的存储

## 4.2 MongoDB的行业具体应用场景

- 游戏场景
- 物流场景
- 社交场景
- 物联网场景
- 直播

## 4.3 如何抉择是否使用MongoDB

| 应用特征                                           | Yes/No  |
| -------------------------------------------------- | ------- |
| 应用不需要事务及复杂 join 支持                     | 必须Yes |
| 新应用，需求会变，数据模型无法确定，想快速迭代开发 | ?       |
| 应用需要2000-3000以上的读写QPS（更高也可以）       | ?       |
| 应用需要TB甚至 PB 级别数据存储                     | ?       |
| 应用发展迅速，需要能快速水平扩展                   | ?       |
| 应用要求存储的数据不丢失                           | ?       |
| 应用需要99.999%高可用                              | ?       |
| 应用需要大量的地理位置查询、文本查询               | ?       |

## 4.4 Java访问MongoDB



## 4.5 Spring 访问MongoDB



## 4.6 Spring Boot访问MongoDB

### 4.6.1 MomgoTemplate的方式

### 4.6.2 MongoRepository



# 第五部分 MongoDB架构

## 5.1 MongoDB逻辑结构



## 5.2 MongoDB的数据模型

### 5.2.1 描述数据模型

#### 内嵌

#### 引用

### 5.2.2 如何选择数据模型

## 5.3 MongoDB的存储引擎

### 5.3.1 存储引擎概述



### 5.3.2 WiredTiger存储引擎优势



### 5.3.3 WiredTiger引擎包含的文件和作用



### 5.3.4 WiredTiger存储引擎实现原理

#### 写请求

#### checkpoint流程

#### Journaling





# 第六部分 MongoDB集群高可用



# 第七部分 MongoDB安全认证









