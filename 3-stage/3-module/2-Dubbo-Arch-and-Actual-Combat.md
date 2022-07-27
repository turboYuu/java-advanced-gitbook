> 第二部分 Dubbo架构与实战

# 1 Dubbo 架构概述

## 1.1 什么是Dubbo

Apache Dubbo 是一款 RPC 服务开发框架，用于解决微服务架构下的服务治理与通信问题。

## 1.2 Dubbo 的特性

参考官网 [Dubbo 核心特性](https://dubbo.apache.org/zh/overview/what/overview/#dubbo-%E6%A0%B8%E5%BF%83%E7%89%B9%E6%80%A7)

## 1.3 Dubbo 的服务治理

服务治理（SOA governance），企业为了确保项目顺利完成二实施的过程，包括最佳实践，架构原则、治理规程、规律以及其他决定性的因素。服务治理指的是用来管理 SOA 的采用和实现的过程。

参考官网 [服务治理](https://dubbo.apache.org/zh/docs/v2.7/user/preface/requirements/)

![image](assest/dubbo-service-governance.jpg)

# 2 Dubbo 处理流程

![dubbo-architucture](assest/dubbo-architecture.jpg)

节点角色说明：

| 节点      | 角色名称                                 |
| --------- | ---------------------------------------- |
| Provide   | 暴露服务的服务提供方                     |
| Consumer  | 调用远程服务的服务消费方                 |
| Registry  | 服务注册与发现的注册中心                 |
| Monitor   | 统计服务的调用次数 和 调用时间的监控中心 |
| Container | 服务运行容器                             |

**调用关系说明**

0. 服务容器负责启动，加载，运行服务提供者。
1. 服务提供者在启动时，向注册中心注册自己提供的服务。
2. 服务消费者在启动时，向注册中心订阅自己所需的服务。
3. 注册中心返回服务提供者地址列表给消费者，如有变更，注册中心将基于长连接推送变更数据给消费者。
4. 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
5. 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。

# 3 服务注册中心 Zookeeper

通过前面的 Dubbo 架构图可以看到，Registry（服务注册中心）在其中起着至关重要的作用。Dubbo 官方推荐使用 Zookeeper 作为服务注册中心。Zookeeper 是 Apache Hadoop 的子项目，作为 Dubbo 服务注册中心，工业强度较高，可用于生产环境，推荐使用。

# 4 Dubbo 开发实战

## 4.1 实战案例介绍

在 Dubbo 中所有的服务调用都是基于接口去进行双方交互的。双方协定好 Dubbo 调用的中接口，提供者来提供实现类 并且 注册到注册中心上。

调用方则只需要引入该接口，并且同样注册到相同的注册中心上（消费者订阅）。即可利用注册中心来实现集群感知功能，之后消费者即可对提供者进行调用。

我们所有的项目都是基于 Maven 进行创建，这样相互在引用的时候只需要以依赖的形式进行扩展就可以了。

并且会通过 maven 的父工程来统一依赖的版本。

程序实现分为以下几个步骤：

1. 建立 Maven 工程 并且创建 API 模块；用于规范双方接口协定。
2. 提供 provider 模块，引入 API 模块，并且对其中的服务进行实现，将其注册到注册中心上，对外来统一提供服务。
3. 提供 consumer 模块，引入 API 模块，并且引入与提供者相同的注册中心，再进行服务调用。

## 4.2 开发过程

官网参考 [以注解配置的方式来配置你的 Dubbo 应用](https://dubbo.apache.org/zh/docsv2.7/user/configuration/annotation/) 示例中使用的 dubbo 版本：`2.7.3`

![image-20220727155516722](assest/image-20220727155516722.png)

### 4.2.1 接口协定

1. 定义 maven

   ```xml
   <parent>
       <artifactId>demo-base</artifactId>
       <groupId>com.turbo</groupId>
       <version>1.0-SNAPSHOT</version>
   </parent>
   <modelVersion>4.0.0</modelVersion>
   
   <artifactId>service-api</artifactId>
   ```

2. 定义接口，这里为了方便，知识写一个基本的方法

   ```java
   package com.turbo.service;
   
   public interface HelloService {
   
       String sayHello(String name);
   }
   ```

   

### 4.2.2 创建接口提供者

1. 引入 API 模块

   ```xml
   <dependency>
       <groupId>com.turbo</groupId>
       <artifactId>service-api</artifactId>
       <version>1.0-SNAPSHOT</version>
   </dependency>
   ```

2. 引入Dubbo相关依赖，这里使用注解方式

   ```xml
   <dependency>
       <groupId>org.apache.dubbo</groupId>
       <artifactId>dubbo</artifactId>
   </dependency>
   <dependency>
       <groupId>org.apache.dubbo</groupId>
       <artifactId>dubbo-registry-zookeeper</artifactId>
   </dependency>
   <dependency>
       <groupId>org.apache.dubbo</groupId>
       <artifactId>dubbo-rpc-dubbo</artifactId>
   </dependency>
   <dependency>
       <groupId>org.apache.dubbo</groupId>
       <artifactId>dubbo-remoting-netty4</artifactId>
   </dependency>
   <dependency>
       <groupId>org.apache.dubbo</groupId>
       <artifactId>dubbo-serialization-hessian2</artifactId>
   </dependency>
   ```

3. 编写实现类。注意这里也是用了 Dubbo 中的 `@Service` 注解来声明它是一个服务的提供者。

   ```java
   package com.turbo.service.impl;
   
   import com.turbo.service.HelloService;
   import org.apache.dubbo.config.annotation.Service;
   
   /**
    * @author yutao
    */
   @Service
   public class HelloServiceImpl implements HelloService {
       @Override
       public String sayHello(String name) {
           return "Hello:"+name;
       }
   }
   ```

4. 编写配置文件，用于配置 dubbo。比如这里叫 `dubbo-provider.properties`，放到 `resources` 目录下：

   ```properties
   dubbo.application.name=service-provider
   dubbo.protocol.name=dubbo
   dubbo.protocol.port=20880
   ```

   - dubbo.application.name ：当前提供者的名称。
   - dubbo.protocol.name：对外提供的时候使用的协议。
   - dubbo.protocol.port：该服务对外暴露的端口是什么，在消费者使用时，则会使用这个端口并且使用指定的协议与提供者建立连接。

5. 编写启动的 `main` 函数。

   ```java
   package com.turbo;
   
   import org.apache.dubbo.config.RegistryConfig;
   import org.apache.dubbo.config.spring.context.annotation.EnableDubbo;
   import org.springframework.context.annotation.AnnotationConfigApplicationContext;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.context.annotation.PropertySource;
   
   import java.io.IOException;
   
   /**
    * @author yutao
    */
   public class DubboPureMain {
   
       public static void main(String[] args) throws IOException {
           AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ProviderConfiguration.class);
           context.start();
           System.in.read();
       }
   
       /**
        * 服务提供者的配置类
        */
       @Configuration
       @EnableDubbo(scanBasePackages = "com.turbo.service.impl")
       @PropertySource("classpath:/dubbo-provider.properties")
       static class ProviderConfiguration{
           @Bean
           public RegistryConfig registryConfig(){
               RegistryConfig registryConfig = new RegistryConfig();
               registryConfig.setAddress("zookeeper://152.136.177.192:2181");
               return registryConfig;
           }
       }
   }
   ```

   

### 4.2.3 创建消费者

1. 引入 API 模块

   ```xml
   <dependency>
       <groupId>com.turbo</groupId>
       <artifactId>service-api</artifactId>
       <version>1.0-SNAPSHOT</version>
   </dependency>
   ```

2. 引入 Dubbo 依赖，同服务提供者。

3. 编写服务，用于真实的引用 dubbo 接口并使用。这里面 `@Reference` 所指向的就是真实的第三方服务接口。

   ```java
   package com.turbo.bean;
   
   import com.turbo.service.HelloService;
   import org.apache.dubbo.config.annotation.Reference;
   import org.springframework.stereotype.Component;
   
   /**
    * @author yutao
    */
   @Component
   public class ConsumerComponet {
   
       /**
        * 引用dubbo的组件  @Reference
        */
       @Reference
       private HelloService helloService;
   
       public String sayHello(String name){
           return helloService.sayHello(name);
       }
   }
   ```

4. 编写消费者的配置文件。这里比较简单，主要就是指定了当前消费者的名称和注册中心的地址。通过这个注册中心地址，消费者就会注册到这里并且也可以根据这个注册中心找到真正的服务提供者列表。

   ```properties
   dubbo.application.name=service-consumer
   dubbo.registry.address=zookeeper://152.136.177.192:2181
   ```

5. 编写启动类，用户在控制台输入一次换行后，则会发起一次请求。

   ```java
   package com.turbo;
   
   import com.turbo.bean.ConsumerComponet;
   import org.apache.dubbo.config.spring.context.annotation.EnableDubbo;
   import org.springframework.context.annotation.AnnotationConfigApplicationContext;
   import org.springframework.context.annotation.ComponentScan;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.context.annotation.PropertySource;
   
   import java.io.IOException;
   
   public class AnnotationConsumerMain {
   
       public static void main(String[] args) throws IOException {
           AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(ConsumerConfiguration.class);
           context.start();
           // 获取消费者组件
           ConsumerComponet service = context.getBean(ConsumerComponet.class);
           while (true){
               System.in.read();
               try{
                   String world = service.sayHello("world");
                   System.out.println("result:"+world);
               }catch (Exception e){
                   e.printStackTrace();
               }
   
           }
       }
   
       /**
        *
        */
       @Configuration
       @EnableDubbo
       @PropertySource("classpath:/dubbo-consumer.properties")
       @ComponentScan(basePackages = {"com.turbo.bean"})
       static class ConsumerConfiguration{
   
       }
   }
   ```



### 4.2.4 QOS 概述

启动 服务提供者，然后启动服务消费者，消费者报错。

![image-20220727163359441](assest/image-20220727163359441.png)

错误提示 Qos 22222 端口已经被占用。

参考官网 [QoS 参数配置](https://dubbo.apache.org/zh/docs3-v2/java-sdk/reference-manual/qos/overview/)

使用系统属性方式配置，在服务端或消费端（主要是避免服务端和消费端 Qos 端口冲突）

![image-20220727163559900](assest/image-20220727163559900.png)

`telnet localhost 22222(qos.port)`

![image-20220727163811181](assest/image-20220727163811181.png)



## 4.3 配置方式介绍

## 4.4 XML 方式

### 4.4.1 provider 模块

### 4.4.2 consumer 模块

# 5 Dubbo 管理控制台

# 6 Dubbo配置项说明