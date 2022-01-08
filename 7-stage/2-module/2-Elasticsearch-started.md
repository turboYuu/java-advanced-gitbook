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

Elasticsearch提供了Rest风格的API，即http请求接口，而且也提供了各种语言的客户端API。

- Rest风格API

  文档地址：https://www.elastic.co/guide/en/elasticsearch/reference/7.3/index.html

  ![image-20220108110623425](assest/image-20220108110623425.png)

- 客户端API

  Elasticsearch支持的语言客户端非常多：https://www.elastic.co/guide/en/elasticsearch/client/index.html，实战中将使用到Java客户端API。

  ![image-20220108110548425](assest/image-20220108110548425.png)

Elasticsearch没有自带图形化界面，可以通过安装Elasticsearch的图形化插件，完成图形化界面的效果，完成索引数据的查看，比如可视化插件Kibana。

# 3 安装配置Kibana

> 1.什么是Kibana

Kibana是一个基于Node.js的Elasticsearch索引库数据统计工具，可以利用Elasticsearch的聚合功能，生成各种图表、如柱状图、线状图、饼图等。

而且还提供了操作Elasticsearch索引数据的控制台，并且提供了一定的API提示，非常有利于学习Elasticsearch的语法。

![image-20220108111330898](assest/image-20220108111330898.png)

> 2.安装Kibana

**1）下载Kibana**

![image-20220108111709928](assest/image-20220108111709928.png)

[Kibana与操作系统](https://www.elastic.co/cn/support/matrix#matrix_os)

**2）安装Kibana**

1. 上传kibana-7.3.0-linux-x86_64.tar.gz，解压

   ```shell
   [root@node1 ~]# tar -xvf kibana-7.3.0-linux-x86_64.tar.gz -C /usr
   [root@node1 usr]# mv kibana-7.3.0-linux-x86_64 kibana
   ```

2. 修改kibana目录拥有者和权限

   ```shell
   # 改变kibana目录拥有者账号
   [root@node1 usr]# chown -R estest /usr/kibana/
   # 设置访问权限
   [root@node1 usr]# chmod -R 777 /usr/kibana/
   ```

3. 修改配置文件

   ```shell
   [root@node1 usr]# vim /usr/kibana/config/kibana.yml
   ```

   修改端口号，访问ip，elasticsearch服务ip

   ```yml
   server.port: 5601
   server.host: "0.0.0.0"
   # The URLs of the Elasticsearch instances to use for all your queries
   elasticsearch.hosts: ["http://192.168.31.71:9200"]
   
   ```

4. 启动kibana

   ```shell
   [root@node1 usr]# su estest
   [estest@node1 usr]$ cd kibana/bin/
   [estest@node1 bin]$ ./kibana
   ```

   没有error错误启动成功：

   ![image-20220108114134559](assest/image-20220108114134559.png)

   访问ip:5601，即可看到安装成功

   ![image-20220108114304281](assest/image-20220108114304281.png)

   已全部安装完成，然后可以接入数据使用了。

**3）kibana使用页面**

选择左侧的Dev Tools菜单，即可进入控制台页面：

![image-20220108114557514](assest/image-20220108114557514.png)

![image-20220108115115550](assest/image-20220108115115550.png)



**4）扩展kibana dev tools快捷键：**

`ctrl+enter` 提交请求

`ctrl+i` 自动缩进

# 4 Elasticsearch继承IK分词器

## 4.1 集成IK分词器

IKAnalyzer是一个开源的，基于Java语言开发的轻量级的中文分析工具包。从2006年12月退出1.0版开始，IKAnalyzer已经推出了3个大版本。最初，它是以开源项目Lucene为应用主体的，结合词典分词和文法分析算法的中文分词组件。新版本的IKAnalyzer 3.0 则发证为面向Java的公用分词组件，独立于Lucene项目，同时提供了对Lucene的默认优化实现。

IK分词器 3.0 的特性如下：

1. 采用了特有的“正向迭代最细粒度切分算法”，具有60万/秒的高速处理能力。
2. 采用了多子处理器分析模式，支持：英文字母（IP地址、Email、URL）、数字（日期，常用中文量词，罗马数字，科学计数法），中文词汇（姓名，地名处理）等分词处理。
3. 支持个人词条的优化的词典存储，更小的内存占用。
4. 支持用户词典扩展定义。
5. 针对Lucene全文检索优化的查询分析器IKQueryParser；采用歧义分析算法优化查询关键字的搜索排列组合，能极大的提高Lucene检索的命中率。

**下载地址**：

https://github.com/medcl/elasticsearch-analysis-ik/releases/tag/v7.3.0

> **下载插件并安装（安装方式一）**

1. 在elasticsearch的bin目录下执行以下命令，es插件管理器会自动帮我们安装，然后等待安装完成：

   ```
   
   ```

2. 下载完成后会提示 Continue with installation? 输入`y` 即可完成安装

3. 重启Elasticsearch 和 Kibana

> **上传安装包安装（安装方式二）**

1. 在elasticsearch安装目录的plugins目录下新建 `analysis-ik` 目录

   ```
   
   ```

2. 重启Elasticsearch 和 Kibana



> **测试案例**

IK分词器有两种分词模式：ik_max_word和 ik_smart模式。

1. ik_max_word（常用）

   会将文本做最细粒度的拆分

2. ik_smart

   会做最粗粒度的拆分

   

## 4.2 扩展词典使用



## 4.3 停用词典使用



## 4.4 同义词典使用



# 5 索引操作（创建、查看、删除）

## 5.1 创建索引库

## 5.2 判断索引是否存在

## 5.3 查看索引

## 5.4 打开索引

## 5.5 关闭索引

## 5.6 删除索引

# 6 映射操作

## 6.1 创建映射字段

## 6.2 映射属性详解

## 6.3 查看映射关系

## 6.4 一次性创建索引和映射

# 7 文档增删改查及局部更新

## 7.1 新建文档

## 7.2 查看单个文档

## 7.3 查看所有文档

## 7.4 _source定制返回结果

## 7.5 更新文档（全部更新）

## 7.6 更新文档（局部更新）

## 7.7 删除文档

## 7.8 文档的全量替换、强制创建