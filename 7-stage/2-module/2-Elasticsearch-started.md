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
   # root用户启动
   [root@node1 ~]# /usr/kibana/bin/kibana --allow-root
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

   ```shell
   [root@node1 ~]# /usr/elasticsearch/bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v7.3.0/elasticsearch-analysis-ik-7.3.0.zip
   ```

2. 下载完成后会提示 Continue with installation? 输入`y` 即可完成安装

3. 重启Elasticsearch 和 Kibana

![image-20220108134715299](assest/image-20220108134715299.png)

> **上传安装包安装（安装方式二）**

1. 在elasticsearch安装目录的plugins目录下新建 `analysis-ik` 目录

   ```shell
   #新建analysis-ik文件夹 
   mkdir analysis-ik
   #切换至analysis-ik文件夹下 
   cd analysis-ik
   #上传资料中的elasticsearch-analysis-ik-7.3.0.zip 
   #解压
   unzip elasticsearch-analysis-ik-7.3.3.zip
   #解压完成后删除zip
   rm -rf elasticsearch-analysis-ik-7.3.0.zip
   ```

2. 重启Elasticsearch 和 Kibana



> **测试案例**

IK分词器有两种分词模式：ik_max_word和 ik_smart模式。

1. ik_max_word（常用）

   会将文本做最细粒度的拆分

2. ik_smart

   会做最粗粒度的拆分



先不管语法，先在kibana测试一波，输入下面的请求：

```json
POST _analyze 
{
 "analyzer": "ik_max_word",
 "text": "南京市长江大桥"
}
```

ik_max_word 分析模式运行得到结果

```json
{
  "tokens" : [
    {
      "token" : "南京市",
      "start_offset" : 0,
      "end_offset" : 3,
      "type" : "CN_WORD",
      "position" : 0
    },
    {
      "token" : "南京",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "CN_WORD",
      "position" : 1
    },
    {
      "token" : "市长",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "长江大桥",
      "start_offset" : 3,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 3
    },
    {
      "token" : "长江",
      "start_offset" : 3,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 4
    },
    {
      "token" : "大桥",
      "start_offset" : 5,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 5
    }
  ]
}

```



```json
POST _analyze 
{
 "analyzer": "ik_smart", 
 "text": "南京市长江大桥" 
}
```

ik_smart分词模式运行得到结果：

```json
{
  "tokens" : [
    {
      "token" : "南京市",
      "start_offset" : 0,
      "end_offset" : 3,
      "type" : "CN_WORD",
      "position" : 0
    },
    {
      "token" : "长江大桥",
      "start_offset" : 3,
      "end_offset" : 7,
      "type" : "CN_WORD",
      "position" : 1
    }
  ]
}
```

如果现在假如**江大桥**是一个人名，是南京市市长，那么上面的分词显然是不合理的，该怎么办？



## 4.2 扩展词典使用

**扩展词**：就是不想让哪些词被分开，让他们分成一个词。比如上面的**江大桥**

**自定义扩展词库**

1. 进入到config/analysis-ik/(**插件命令安装**) 或 plugins/analysis-ik/config (**安装包安装方式**) 目录下，新增自定义词典

   ```shell
   [root@node1 analysis-ik]# vim turbo_ext_dict.dic
   ```

   输入：江大桥

2. 将我们自定义的扩展词典文件添加到 IKAnalyzer.cfg.xml 配置中

   vim IKAnalyzer.cfg.xml

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
   <properties>
           <comment>IK Analyzer 扩展配置</comment>
           <!--用户可以在这里配置自己的扩展字典 -->
           <entry key="ext_dict">turbo_ext_dict.dic</entry>
            <!--用户可以在这里配置自己的扩展停止词字典-->
           <entry key="ext_stopwords"></entry>
           <!--用户可以在这里配置远程扩展字典 -->
           <!-- <entry key="remote_ext_dict">words_location</entry> -->
           <!--用户可以在这里配置远程扩展停止词字典-->
           <!-- <entry key="remote_ext_stopwords">words_location</entry> -->
   </properties>
   ```

3. 重启Elasticsearch

   ik_max_word 分析模式运行得到结果中会出现：江大桥



## 4.3 停用词典使用

**停用词**：有些词在文本中出现的频率非常高。但对文本的语义产生不了多大的影响。例如英文的a、an、the、of等，或中文 的、了、呢  等。这样的词称为停用词。停用词经常被过滤掉，不会进行索引。在检索过程中，如果用户的查询中含有停用词，系统会自动过滤掉。停用词可以加快索引的速度，减少索引库文件的大小。

**自定义停用词库**

1. 进入到config/analysis-ik/(**插件命令安装**) 或 plugins/analysis-ik/config (**安装包安装方式**) 目录下，新增自定义词典

   ```shell
   vim turbo_stop_dict.dic
   ```

   输入

   ```xml
   的
   了
   啊
   ```

2. 将自定义的停用词典文件添加到IKAnalyzer.cfg.xml配置中

3. 重启Elasticsearch



## 4.4 同义词典使用

很多相同意思的词，称之为同义词。在搜索时，输入“番茄”，但是应该把含有“西红柿”的数据也查出来，这种情况叫 同义词查询。

注意：扩展词和停用词是在索引的时候使用，而同义词是检索时候使用。

**配置IK同义词**

Elasticsearch 自带一个名为 synonym 的同义词filter。为了能让 IK 和 synonym 同时工作，需要定义新的analyzer，用 IK 做 tokenizer，synonym 做 filter。实际只需要加一段配置。

1. 创建 /config/analysis-ik/synonym.txt 文件，输入一些同义词并存为 utf-8（linux默认格式就是 utf-8） 格式。例如

   ```tex
   turbo,涡轮
   china,中国
   ```

2. 创建索引时，使用同义词配置，示例模板如下

   ```json
   PUT /索引名称
   {
     "settings": {
       "analysis": {
         "filter": {
           "word_sync": {
             "type": "synonym",
             "synonyms_path": "analysis-ik/synonym.txt"
           }
         },
         "analyzer": {
           "ik_sync_max_word": {
             "filter": [
               "word_sync"
             ],
             "type": "custom",
             "tokenizer": "ik_max_word"
           },
           "ik_sync_smart": {
             "filter": [
               "word_sync"
             ],
             "type": "custom",
             "tokenizer": "ik_smart"
           }
         }
       }
     },
     "mappings": {
       "properties": {
         "字段名": {
           "type": "字段类型",
           "analyzer": "ik_sync_smart",
           "search_analyzer": "ik_sync_smart"
         }
       }
     }
   }
   ```

   以上配置定义了ik_sync_max_word 和 ik_sync_smart 这两个新的 analyzer，对应 IK 的 ik_max_word 和 ik_smart 两种分词策略。ik_sync_max_word 和 ik_sync_smart 都会使用 synonym filter 实现同义词转换。

3. 到此，索引创建模板中同义词配置完成，搜索时指定分词器为 ik_sync_max_work 或 ik_sync_smart。

4. 案例

   ```json
   PUT /turbo-es-synonym
   {
     "settings": {
       "analysis": {
         "filter": {
           "word_sync": {
             "type": "synonym",
             "synonyms_path": "analysis-ik/synonym.txt"
           }
         },
         "analyzer": {
           "ik_sync_max_word": {
             "filter": [
               "word_sync"
             ],
             "type": "custom",
             "tokenizer": "ik_max_word"
           },
           "ik_sync_smart": {
             "filter": [
               "word_sync"
             ],
             "type": "custom",
             "tokenizer": "ik_smart"
           }
         }
       }
     },
     "mappings": {
       "properties": {
         "name": {
           "type": "text",
           "analyzer": "ik_sync_smart",
           "search_analyzer": "ik_sync_smart"
         }
       }
     }
   }
   ```

   插入数据

   ```json
   POST /turbo-es-synonym/_doc/1
   {
     "name": "涡轮是一种将流动工质的能量转换为机械功的旋转式动力机械"
   }
   ```

   使用同义词 "turbo" 或者 "涡轮" 进行搜索

   ```json
   POST /turbo-es-synonym/_doc/_search
   {
     "query": {
       "match": {
         "name": "turbo"
       }
     }
   }
   ```

   ![image-20220108150743482](assest/image-20220108150743482.png)



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