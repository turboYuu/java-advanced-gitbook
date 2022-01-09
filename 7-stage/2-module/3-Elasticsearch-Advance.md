第三部分 Elasticsearch高级应用

# 1 映射高级

## 1.1 地理坐标点数据类型

- 地理坐标点

  地理坐标点是指地球表面可以用经纬度描述的一个点。地理坐标点可以用来计算两个坐标间的距离，还可以判断一个坐标是否在一个区域中。地理坐标点需要显式声明对应字段类型为`geo_point`:

  > 示例

  ```json
  PUT /company-locations
  {
    "mappings": {
      "properties": {
        "name": {
          "type": "text"
        },
        "location": {
          "type": "geo_point"
        }
      }
    }
  }
  ```

  ![image-20220109135810566](assest/image-20220109135810566.png)

- 经纬度坐标格式

  如上例，`location`字段被声明为`geo_point`后，我们就可以索引包含了经纬度信息的文档了。经纬度信息的形式可以是字符串、数组或者对象。

  ```json
  # 字符串形式
  PUT /company-locations/_doc/1
  {
    "name": "NetEase",
    "location": "40.715,74.011"
  }
  # 对象形式
  PUT /company-locations/_doc/2
  {
    "name": "Sina",
    "location": {
      "lat": 40.722,
      "lon": 73.989
    }
  }
  # 数组形式
  PUT /company-locations/_doc/3
  {
    "name": "Baidu",
    "location": [
      73.983,
      40.719
    ]
  }
  ```

  ![image-20220109142310560](assest/image-20220109142310560.png)

  **注意**

  字符串形式以保健逗号分隔，如："lat,lon"

  对象形式显式命名为 lat 和 lon

  数组形式表示为 [lon,lat]

- 通过地理坐标点过滤

  有四种地理坐标点相关的过滤器，可以用来选中或者排除文档

  | 过滤器             | 作用                                                         |
  | ------------------ | ------------------------------------------------------------ |
  | geo_bounding_box   | 找出落在指定矩形框中的点                                     |
  | geo_distance       | 找出与指定位置在给定距离内的点                               |
  | geo_distance_range | 找出与指定距离在给定最小距离和最大距离之间的点               |
  | geo_polygon        | 找出落在多边形中的点，**这个过滤器使用代价很大**，<br>当你觉得自己需要使用它，最好先看看[geo-shapes](https://www.elastic.co/guide/cn/elasticsearch/guide/current/geo-shapes.html) |

- geo_bounding_box 查询

  这是目前位置最有效的地理坐标过滤器，因为它计算起来非常简单。你指定一个矩形的顶部，底部，左边界和右边界。然后过滤器只需要判断坐标的经度是否在左右边界之间，纬度是否在上下边界之间

  然后可以使用`geo_bounding_box`过滤器执行以下查询

  ```json
  GET /company-locations/_search
  {
    "query": {
      "bool": {
        "must": {
          "match_all": {}
        },
        "filter": {
          "geo_bounding_box": {
            "location": {
              "top_left": {
                "lat": 40.73,
                "lon": 71.12
              },
              "bottom_right": {
                "lat": 40.01,
                "lon": 74.1
              }
            }
          }
        }
      }
    }
  }
  ```

  location这些坐标可以用 bottom_left 和 top_right 来表示。

- geo_distance

  过滤仅包含与地理位置相距特定距离内的匹配文档。假设一下映射和索引文档

  然后可以使用`geo_distance`过滤器执行以下查询

  ```json
  GET /company-locations/_search
  {
    "query": {
      "bool": {
        "must": {
          "match_all": {}
        },
        "filter": {
          "geo_distance": {
            "distance": "400km",
            "location": {
              "lat": 40,
              "lon": 70
            }
          }
        }
      }
    }
  }
  ```
  
  
  
  

## 1.2 动态映射

Elasticsearch在遇到文档中以前未遇到的字段，可以使用dynamic mapping（动态映射机制）来确定字段的数据类型并自动把新的字段添加到类型映射。

Elastic的动态映射机制可以进行开关控制，通过设置mappings的dynamic属性，dynamic有如下设置项

- true：遇到陌生字段就执行dynamic mapping处理机制
- false：遇到陌生字段就忽略
- strict：遇到陌生字段就报错

```json
# 设置为报错
PUT /user
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 0
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "name": {
        "type": "text"
      },
      "address": {
        "type": "object",
        "dynamic": true
      }
    }
  }
}

# 插入以下文档，将会报错
# user索引层设置 dynamic是strict，在user层内设置age将报错
# 在address层设置dynamic是true，将动态映射生成字段
PUT /user/_doc/1
{
  "name": "lisi",
  "age": "20",
  "address": {
    "province": "beijing",
    "city": "beijing"
  }
}

PUT /user
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 0
  },
  "mappings": {
    "dynamic": "true",
    "properties": {
      "name": {
        "type": "text"
      },
      "address": {
        "type": "object",
        "dynamic": true
      }
    }
  }
}
```

![image-20220109165629193](assest/image-20220109165629193.png)

## 1.3 自定义动态映射

如果你想在运行时增加新的字段，你可能会启用动态映射。然而，有时候，动态映射规则 可能不太智能。幸运的是，可以通过设置系定义这些规则，以便更好的适用于你的数据。

### 1.3.1 日期检测

当Elasticsearch遇到一个新的字符串字段时，它会检测这个字段是否包含一个可识别的日期，比如 2014-0-01如果它像日期，这个字段就会被作为date类型添加。否则，它会被作为string类型添加。这些时候这个行为可能导致一些问题。假如，你有如下这样的一个文档：<br>{"note":"2014-01-01"}<br>假设这是第一次识别note 字段，它会被添加为 date 字段。但是如果下一个文档像这样：<br>{"note":"Logged out"}<br>这显然不是一个日期，但为时已晚。这个字段已经是一个日期类型，这个不合法的日期，将会造成一个异常。

日期检测可以通过在根对象上设置 date_detection 为 false 来关闭

```

```





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