> 4-6 Spring Cloud Config 分布式配置中心

# 1 分布式配置中心应用场景

往往，我们使用配置文件管理一些配置信息，比如 application.yml

**单体应用架构**，配置信息的管理，维护并不会显得特别麻烦，手动操作就可以，因为就一个工程；

**微服务架构**，因为我们的分布式集群环境中可能有很多个微服务，我们不可能一个一个去修改配置然后重启生效，在一定场景下我们还需要在运行期间动态调整配置信息，比如：根据各个微服务的负载情况，动态调整数据源连接池大小，我们希望配置内容发生变化的时候，微服务可以自动更新。

场景总结如下：

1. 集中配置管理，一个微服务架构中可能有成百上千个微服务，所以集中配置管理是很重要的（一次修改，到处生效）
2. 不同环境不同配置，比如数据源配置在不同环境（开发 dev，测试 test，生产 prod）中是不同的
3. 运行期间可动态调整。例如，可根据各个微服务的负载情况，动态调整数据源连接池大小等配置，修改后可自动更新
4. 如配置内容发生变化，微服务可以自动更新配置

那么，我们就需要对配置文件进行**集中式管理**，这也是分布式配置中心的作用。

# 2 Spring Cloud Config

## 2.1 Config 简介

Spring Cloud Config 是一个分布式配置管理方案，包含了 Server 端 和 Client 端 两个部分。

![image-20220825172230488](assest/image-20220825172230488.png)

![image-20220825172534868](assest/image-20220825172534868.png)

- Server 端：提供配置文件的存储，以接口的形式将配置文件的内容提供出去，通过使用 @EnableConfigServer 注解在 SpringBoot 应用中非常简单的嵌入。
- Client 端：通过接口获取配置数据并初始化自己的应用。

## 2.2 Config 分布式配置应用

**说明：Config Server 是集中式的配置服务，用于集中管理应用程序各个环境下的配置。默认使用Git存储配置文件内容，也可以 SVN**。

比如，我们要对 “简历微服务” 的 application.yml 进行管理（区分开发环境、测试环境、生产环境）

准备：

1. 登录码云，创建项目 `turbo-config-repo`

2. 上传 yml 配置文件，命名规则如下：

   {application}-{profile}.yml 或者 {application}-{profile}.properties

   其中，application 为应用名称，profile 指的是环境（用于区分开发环境、测试环境、生产环境等）

   示例：turbo-service-resume-dev.yml、turbo-service-resume-test.yml、turbo-service-resume-prod.yml



### 2.2.1 构建 Config Server 统一配置中心

1. 新建 SpringBoot 工程 `turbo-cloud-configserver-9006`，引入依赖坐标（需要注册自己到 Eureka）

   ```xml
   <!--eureka client 客户端依赖引入-->
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
   </dependency>
   
   <!--config 配置中心服务端-->
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-config-server</artifactId>
   </dependency>
   ```

2. 配置启动类，使用注解 开启配置中心服务器功能

   ```java
   package com.turbo;
   
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
   import org.springframework.cloud.config.server.EnableConfigServer;
   
   @SpringBootApplication
   @EnableDiscoveryClient
   @EnableConfigServer // 开启配置中心功能
   public class ConfigServerApplication9006 {
       public static void main(String[] args) {
           SpringApplication.run(ConfigServerApplication9006.class,args);
       }
   }
   ```

3. application.yml 配置

   ```yaml
   server:
     port: 9006
   
   eureka:
     client:
       service-url: #eureka server 的路径
         # 把所有 eureka 集群中的所有url都填写进来，可以只写一台，因为各个 eureka server 可以同步注册表
         defaultZone: http://TurboCloudEurekaServerB:8762/eureka,http://TurboCloudEurekaServerA:8761/eureka
     instance:
       #服务实例中显示ip，而不是显示主机名，(为了兼容老版本,新版本经过实验都是ip)
       prefer-ip-address: true
       # 实例名称： 192.168.1.3:turbo-service-resume:8080  可以自定义实例显示格式，加上版本号，便于多版本管理，注意是ip-address，早期版本是ipAddress
       instance-id: ${spring.cloud.client.ip-address}:${spring.application.name}:${server.port}:@project.version@
   
   
   # springboot中暴露健康检查等断点接口
   management:
     endpoints:
       web:
         exposure:
           include: "*"
     # 暴露健康检查细节
     endpoint:
       health:
         show-details: always
   
   # http://localhost:9006/master/turbo-service-resume-dev.yml
   spring:
     application:
       name: turbo-cloud-configserver
     cloud:
       config:
         server:
           git:
             uri: https://gitee.com/turboYuu/turbo-config-repo.git
             username: yutao2013@126.com
             password: yutao@#1990
             search-paths:
               - turbo-config-repo
         # 读取分支
         label: master
   ```



启动 eureka 注册中心 和 config 微服务，访问：http://localhost:9006/master/turbo-service-resume-dev.yml，

![image-20220825182232707](assest/image-20220825182232707.png)

### 2.2.2 构建 Client 客户端（在已有的简历微服务基础上）

![image-20220825184759500](assest/image-20220825184759500.png)

![image-20220825184820433](assest/image-20220825184820433.png)

1. 已有工程中添加依赖坐标

   ```xml
   
   ```

   



# 3 Config 配置手动刷新

# 4 Config 配置自动更新