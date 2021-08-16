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
	show dbs;
切换数据库 如果没有对应的数据库则创建
	use 数据库名;
创建集合
	db.createcollection("集合名");
查看集合
	show tables;
	show collections;
删除集合
	db.集合名.drop();
删除当前数据库
	db.dropDatabase();
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

### 2.3.2 MongoDB聚合操作分类

### 2.3.3 单目的聚合操作

### 2.3.4 聚合管道（Aggregation Pipeline）

![image-20210815182827697](assest/image-20210815182827697.png)

![image-20210815183032744](assest/image-20210815183032744.png)

### 2.3.5 MapReduce编程模型

```
db.lg_resume_preview.mapReduce(
      function () {emit(this.city, this.expectSalary)}, //mapFunction
      function(key, values){return Array.avg(values)},//reduceFunction
      {
         query:{expectSalary:{$gt:15000}},
         finalize:function(key,value){
             return value+5000
         },verbose:true,
         out: "citySalary"
      })
```



# 第三部分 MongoDB索引Index

## 3.1 什么是索引

> 索引是一种单独的、物理的对数据库表中一列或多列的值进行排序的一种存储结构，它是某个表中一列或若干列值的集合和相应的指向表中物理标识这些值的数据页

## 3.2 索引类型

### 3.2.1 单键索引（Single Field）



过期索引或删除索引上的数据



### 3.2.2 复合索引（Compound Index）

https://docs.mongodb.com/manual/core/index-compound/

### 3.2.3 多键索引（Multikey indexes）

### 3.2.4 地理空间索引（Geospatial Index）

### 3.2.5 全文索引

### 3.2.6 哈希索引 Hashed Index

https://docs.mongodb.com/manual/core/index-hashed/

## 3.3 索引和explain

### 3.3.1 索引管理



### 3.3.2 explain分析



## 3.4 慢查询分析

## 3.5 MongoDB索引底层实现原理



