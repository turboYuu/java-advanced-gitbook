第六部分 Elasticsearch搜索实战

# 1 案例需求

MySQL中的数据批量导入到ES中，然后进行搜索职位信息，展示出职位信息

# 2 代码实现

[源码地址](https://gitee.com/turboYuu/elastic-stack-7-2/tree/master/lab/turbo-es-project)

## 2.1 项目准备

1. 数据库准备，执行[position.sql](https://gitee.com/turboYuu/elastic-stack-7-2/blob/master/lab/position.sql)

2. pom.xml

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <project xmlns="http://maven.apache.org/POM/4.0.0"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
       <modelVersion>4.0.0</modelVersion>
   
       <parent>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-parent</artifactId>
           <version>2.1.0.RELEASE</version>
           <relativePath/>
       </parent>
       <groupId>com.turbo</groupId>
       <artifactId>turbo-es-project</artifactId>
       <version>1.0-SNAPSHOT</version>
       <name>turbo-es-project</name>
       <description>Demo project for Spring Boot</description>
   
       <properties>
           <elasticsearch.version>7.3.0</elasticsearch.version>
       </properties>
       <dependencies>
           <dependency>
               <groupId>org.elasticsearch.client</groupId>
               <artifactId>elasticsearch-rest-high-level-client</artifactId>
               <version>${elasticsearch.version}</version>
               <exclusions>
                   <exclusion>
                       <groupId>org.elasticsearch</groupId>
                       <artifactId>elasticsearch</artifactId>
                   </exclusion>
               </exclusions>
           </dependency>
           <dependency>
               <groupId>org.elasticsearch</groupId>
               <artifactId>elasticsearch</artifactId>
               <version>${elasticsearch.version}</version>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-thymeleaf</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-web</artifactId>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-devtools</artifactId>
               <scope>runtime</scope>
               <optional>true</optional>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-configuration-processor</artifactId>
               <optional>true</optional>
           </dependency>
           <dependency>
               <groupId>org.projectlombok</groupId>
               <artifactId>lombok</artifactId>
               <optional>true</optional>
           </dependency>
           <dependency>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-starter-test</artifactId>
               <scope>test</scope>
               <exclusions>
                   <exclusion>
                       <groupId>org.junit.vintage</groupId>
                       <artifactId>junit-vintage-engine</artifactId>
                   </exclusion>
               </exclusions>
           </dependency>
           <!--httpClient-->
           <dependency>
               <groupId>org.apache.httpcomponents</groupId>
               <artifactId>httpclient</artifactId>
               <version>4.5.3</version>
           </dependency>
           <dependency>
               <groupId>com.alibaba</groupId>
               <artifactId>fastjson</artifactId>
               <version>1.2.58</version>
           </dependency>
           <dependency>
               <groupId>mysql</groupId>
               <artifactId>mysql-connector-java</artifactId>
               <scope>runtime</scope>
           </dependency>
           <dependency>
               <groupId>org.apache.commons</groupId>
               <artifactId>commons-lang3</artifactId>
               <version>3.9</version>
           </dependency>
           <dependency>
               <groupId>junit</groupId>
               <artifactId>junit</artifactId>
               <version>4.12</version>
               <scope>test</scope>
           </dependency>
   
       </dependencies>
   
       <build>
           <plugins>
               <plugin>
                   <groupId>org.springframework.boot</groupId>
                   <artifactId>spring-boot-maven-plugin</artifactId>
               </plugin>
           </plugins>
       </build>
   </project>
   ```
   
   

## 2.2 代码编写

### 2.2.1 application.yml

```yaml
spring:
  devtools:
    restart:
      enabled: true # 设置开启热部署
      additional-paths: src/main/java #重启目录
      exclude: WEB-INF/**
    freemarker:
      cache: false # 页面不加载缓存，修改即时生效

  elasticsearch:
    rest:
      uris: 192.168.31.72:9200,192.168.31.72:9201,192.168.31.72:9202

server:
  port: 8083

logging:
  level:
    root: info
    com.xdclass.search: debug
```

### 2.2.2 Model

```java
package com.turbo.es.model;

import com.fasterxml.jackson.annotation.JsonFormat;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.Date;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class Position {
    //主键
    private String id;

    //公司名称
    private String companyName;

    //职位名称
    private String positionName;

    //职位诱惑
    private String positionAdvantage;

    //薪资
    private String salary;

    //薪资下限
    private int salaryMin;

    //薪资上限
    private int salaryMax;

    //学历
    private String education;

    //工作年限
    private String workYear;

    //发布时间
    private String publishTime;

    //工作城市
    private String city;

    //工作地点
    private String workAddress;

    //发布时间
    @JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
    private Date createTime;

    //工作模式
    private String jobNature;
}
```



### 2.2.3 ES配置类

```java
package com.turbo.es.config;

import org.apache.http.HttpHost;
import org.elasticsearch.client.RestClient;
import org.elasticsearch.client.RestHighLevelClient;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class EsConfig {

    @Value("${spring.elasticsearch.rest.uris}")
    private String hostlists;

    @Bean
    public RestHighLevelClient client(){
        // 解析hostlist配置信息
        final String[] split = hostlists.split(",");
        // 创建httphost数组，其中存放es主机和端口号配置信息
        HttpHost[] httpHostArray = new HttpHost[split.length];
        for (int i = 0; i < split.length; i++) {
            String item = split[i];
            System.out.println(item);
            httpHostArray[i] = new HttpHost(item.split(":")[0],
                    Integer.parseInt(item.split(":")[1]),
                    "http");
        }
        //创建 RestHighLevelClient客户端
        return new RestHighLevelClient(RestClient.builder(httpHostArray));
    }
}
```



### 2.2.4 连接mysql的工具类DBHelper

```java
package com.turbo.es.util;

import java.sql.Connection;
import java.sql.DriverManager;

public class DBHelper {

    public static final String url = "jdbc:mysql://152.136.177.192:3306/turbo_position?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai";
    public static final String name = "com.mysql.cj.jdbc.Driver";
    public static final String user = "root";
    public static final String password = "123456";

    public static Connection conn = null;

    public static Connection getConn(){
        try {
            Class.forName(name);
            // 获取连接
            conn = DriverManager.getConnection(url,user,password);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return conn;
    }
}
```



### 2.2.5 Service 接口和实现类

```java
package com.turbo.es.service;

import java.io.IOException;
import java.util.List;
import java.util.Map;

public interface PositionService {
    /**
     * 分页查询
     */
    public List<Map<String,Object>> searchPos(String keyword,int pageNo,int pageSize) throws IOException;
    /**
      * 导入数据    
     */
    void importAll() throws IOException;
}
```

PositionServiceImpl

```java
package com.turbo.es.service.impl;

import com.turbo.es.service.PositionService;
import com.turbo.es.util.DBHelper;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import org.elasticsearch.action.ActionListener;
import org.elasticsearch.action.bulk.BackoffPolicy;
import org.elasticsearch.action.bulk.BulkProcessor;
import org.elasticsearch.action.bulk.BulkRequest;
import org.elasticsearch.action.bulk.BulkResponse;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.common.unit.ByteSizeUnit;
import org.elasticsearch.common.unit.ByteSizeValue;
import org.elasticsearch.common.unit.TimeValue;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.sql.*;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.TimeUnit;
import java.util.function.BiConsumer;

@Service
public class PositionServiceImpl implements PositionService {

    private static final Logger logger = LogManager.getLogger(PositionServiceImpl.class);
    @Autowired
    private RestHighLevelClient client;
    private static final String POSITIOIN_INDEX = "position";

    /**
     * 分页查询
     *
     * @param keyword
     * @param pageNo
     * @param pageSize
     */
    @Override
    public List<Map<String, Object>> searchPos(String keyword, int pageNo, int pageSize) throws IOException {
        if(pageNo <= 1){
            pageNo = 1;
        }
        SearchRequest searchRequest = new SearchRequest(POSITIOIN_INDEX);
        SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
        searchSourceBuilder.from((pageNo-1)*pageSize);
        searchSourceBuilder.size(pageSize);
        searchSourceBuilder.query(QueryBuilders.matchQuery("positionName",keyword));
        searchSourceBuilder.timeout(new TimeValue(60,TimeUnit.SECONDS));

        // 执行搜索
        searchRequest.source(searchSourceBuilder);
        final SearchResponse searchResponse = client.search(searchRequest, RequestOptions.DEFAULT);
        ArrayList<Map<String,Object>> list = new ArrayList<>();
        final SearchHit[] searchHits = searchResponse.getHits().getHits();
        System.out.println(searchHits.length);

        for (SearchHit searchHit : searchHits) {
            list.add(searchHit.getSourceAsMap());
        }
        return list;
    }

    /**
     * 导入数据
     */
    @Override
    public void importAll() throws IOException {
        writeMysqlDataToES(POSITIOIN_INDEX);
    }

    /**
     * 将数据批量写入ES中
     *
     * @param tableName
     */
    private void writeMysqlDataToES(String tableName) {
        BulkProcessor bulkProcessor = getBulkProcessor(client);
        Connection connection = null;
        PreparedStatement ps = null;
        ResultSet rs = null;
        try {
            connection = DBHelper.getConn();
            logger.info("start handle Data :" + tableName);
            String sql = "select * from " + tableName;
            ps = connection.prepareStatement(sql, 
                                             ResultSet.TYPE_FORWARD_ONLY, 
                                             ResultSet.CONCUR_READ_ONLY);
            // 根据自己需要设置批处理 fetchSize 大小
            ps.setFetchSize(20);
            rs = ps.executeQuery();
            ResultSetMetaData colData = rs.getMetaData();
            ArrayList<HashMap<String, String>> dataList = new ArrayList<>();
            HashMap<String, String> map = null;
            int count = 0;
            // c 就是列的名字
            String c = null;
            // v就是列对应的值
            String v = null;
            while (rs.next()) {
                count++;
                map = new HashMap<>(128);
                for (int i = 1; i < colData.getColumnCount(); i++) {
                    c = colData.getColumnName(i);
                    v = rs.getString(c);
                    map.put(c, v);
                }
                dataList.add(map);
                // 每 10,000 条 写一次，不足一万条的批次数据，最后一次提交处理
                if (count % 10000 == 0) {
                    logger.info("mysql handle data number:" + count);
                    // 将数据添加到bulkProcessor
                    for (HashMap<String, String> hashMap2 : dataList) {
                        bulkProcessor.add(new IndexRequest(POSITIOIN_INDEX).source(hashMap2));
                    }
                    map.clear();
                    dataList.clear();
                }
            }
            // 处理未提交的数据
            for (HashMap<String, String> hashMap2 : dataList) {
                bulkProcessor.add(new IndexRequest(POSITIOIN_INDEX).source(hashMap2));
                System.out.println(hashMap2);
            }
            logger.info("-------------------------- Finally insert number total : " + count);
            bulkProcessor.flush();
        } catch (SQLException e) {
            e.printStackTrace();
        } finally {
            try {
                rs.close();
                ps.close();
                connection.close();
                boolean terminatedFlag = bulkProcessor.awaitClose(150L,TimeUnit.SECONDS);
                logger.info(terminatedFlag);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

    private BulkProcessor getBulkProcessor(RestHighLevelClient client) {
        BulkProcessor bulkProcessor = null;
        try {
            BulkProcessor.Listener listener = new BulkProcessor.Listener() {
                @Override
                public void beforeBulk(long executionId, BulkRequest request) {
                    logger.info("Try to insert data number : " + request.numberOfActions());
                }

                @Override
                public void afterBulk(long executionId, BulkRequest request, BulkResponse response) {
                    logger.info("************** Success insert data number : " +
                            request.numberOfActions() + " , id: " + 
                                executionId);
                }

                @Override
                public void afterBulk(long executionId, BulkRequest request, Throwable failure) {
                    logger.error("Bulk is unsuccess : " + 
                                 failure + 
                                 ", executionId: " + 
                                 executionId);
                }
            };
            BiConsumer<BulkRequest, ActionListener<BulkResponse>> bulkConsumer = 
                (request, bulkListener) -> client
                    .bulkAsync(request, RequestOptions.DEFAULT, bulkListener);

            BulkProcessor.Builder builder = BulkProcessor.builder(bulkConsumer, listener);
            builder.setBulkActions(5000);
            builder.setBulkSize(new ByteSizeValue(100L, ByteSizeUnit.MB));
            builder.setConcurrentRequests(10);
            builder.setFlushInterval(TimeValue.timeValueSeconds(100L));
            builder.setBackoffPolicy(BackoffPolicy.constantBackoff(TimeValue.timeValueSeconds(1L), 3));
            // 注意点：让参数设置生效
            bulkProcessor = builder.build();
        } catch (Exception e) {
            e.printStackTrace();
            try {
                bulkProcessor.awaitClose(100L, TimeUnit.SECONDS);
            } catch (Exception e1) {
                logger.error(e1.getMessage());
            }
        }
        return bulkProcessor;
    }
}
```

BulkProcessor  官网介绍

https://www.elastic.co/guide/en/elasticsearch/client/java-api/7.3/java-docs-bulk-processor.html

### 2.2.6 控制器

```java
package com.turbo.es.controller;

import com.turbo.es.service.PositionService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.*;

import java.io.IOException;
import java.util.List;
import java.util.Map;

@Controller
public class PositionController {

    @Autowired
    private PositionService positionService;

    // 测试页面
    @GetMapping({"/","/index"})
    public String indexPage(){
        return "index";
    }

    @GetMapping("/search/{keyword}/{pageNo}/{pageSize}")
    @ResponseBody
    public List<Map<String,Object>> searchPosition(@PathVariable String keyword,
                                                   @PathVariable int pageNo,
                                                   @PathVariable int pageSize) throws IOException {
        final List<Map<String, Object>> searchPos = positionService.searchPos(keyword, 
                                                                              pageNo, 
                                                                              pageSize);
        System.out.println(searchPos);
        return searchPos;
    }

    @RequestMapping("/importAll")
    @ResponseBody
    public String importAll(){
        try {
            positionService.importAll();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return "success";
    }
}
```



### 2.2.7 启动引导

```java
package com.turbo.es;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ESBootApplication {
    public static void main(String[] args) {
        SpringApplication.run(ESBootApplication.class,args);
    }
}
```



### 2.2.8 页面 idex

```html
<!DOCTYPE html>
<html xmlns:th="http://www.w3.org/1999/xhtml">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8">
    <link rel="stylesheet" href="layout_8a42639.css">
    <link rel="stylesheet" href="main.html_aio_2_29de31f.css">
    <link rel="stylesheet" href="main.html_aio_16da111.css">
    <link rel="stylesheet" href="widgets_03f0f0e.css">

    <script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
    <script src="https://unpkg.com/axios/dist/axios.min.js"></script>
</head>
<body>
<div id="top_bannerC">
    <a rel="nofollow" href="https://zhuanti.lagou.com/qiuzhaozhc.html"
       style="background: url(https://www.lgstatic.com/i/image/M00/3C/48/CgqCHl8nkiOAOihGAAC5a9EQ2rU358.PNG) center center no-repeat"
       target="_blank" data-lg-tj-id="bd00" data-lg-tj-no="idnull" data-lg-tj-cid="11127" class=""></a>
</div>

<div id="lg_header">
    <!-- 页面主体START -->
    <div id="content-container">

        <div class="search-wrapper" style="height: 180px;">
            <div id="searchBar" class="search-bar" style="padding-top: 30px;">
                <div class="tab-wrapper" style="display: block;">
                    <a id="tab_pos" class="active" rel="nofollow" href="javascript:;">职位 ( <span>500+</span> ) </a>
                    <a id="tab_comp" class="disabled" rel="nofollow" href="javascript:;">公司 ( <span>0</span> ) </a>
                </div>
                <div class="input-wrapper" data-lg-tj-track-code="search_search" data-lg-tj-track-type="1">
                    <div class="keyword-wrapper">
                        <span role="status" aria-live="polite" class="ui-helper-hidden-accessible"></span><input
                            type="text" id="keyword" v-model="keyword" autocomplete="off" maxlength="64"
                            placeholder="搜索职位、公司或地点"
                            value="java" class="ui-autocomplete-input" @keyup.enter="searchPosition">
                    </div>
                    <input type="button" id="submit" value="搜索" @click="searchPosition">
                </div>
            </div>
        </div>

        <!-- 搜索输入框模块 -->
        <div id="main_container">
            <!-- 左侧 -->
            <div class="content_left">
                <div class="s_position_list " id="s_position_list">
                    <ul class="item_con_list" style="display: block;">
                        <li class="con_list_item default_list" v-for="element in results">
                            <span class="top_icon direct_recruitment"></span>
                            <div class="list_item_top">
                                <div class="position">
                                    <div class="p_top">
                                        <h3 style="max-width: 180px;">{{element.positionName}}</h3>
                                        <span class="add">[<em>{{element.workAddress}}</em>]</span>
                                        <span class="format-time"> {{element.createTime}} 发布</span>
                                    </div>
                                    <div class="p_bot">
                                        <div class="li_b_l">
                                            <span class="money">{{element.salary}}</span>
                                            经验 {{element.workYear}} 年 / {{element.education}}
                                        </div>
                                    </div>
                                </div>
                                <div class="company">
                                    <div class="company_name">
                                        <a href="#" target="_blank">{{element.companyName}}</a>
                                        <i class="company_mark"><span>该企业已经上传营业执照并通过资质验证审核</span></i>
                                    </div>
                                    <div class="industry">
                                        福利 【{{element.positionAdvantage}}】
                                    </div>
                                </div>
                                <div class="com_logo">
                                    <img src="//www.lgstatic.com/thumbnail_120x120/i/image2/M01/79/70/CgotOV1aS4qAWK6WAAAM4NTpXws809.png"
                                         alt="拉勾渠道" width="60" height="60">
                                </div>
                            </div>
                        </li>
                    </ul>
                </div>
            </div>
        </div>
    </div>
</div>
</div>
</body>
<script>
    new Vue({
        el: "#lg_header",
        data: {
            keyword: "",
            results: []
        },
        methods: {
            searchPosition() {
                var keyword = this.keyword;
                console.log(keyword);
                axios.get('http://localhost:8083/search/' + keyword + '/1/10').then(response =>{
                    console.log(response.data);
                    this.results = response.data;
                });
            }
        }
    });
</script>
</html>
```

