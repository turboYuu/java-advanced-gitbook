第一部分 SpringBoot应用回顾

# 1 约定优于配置

概念：约定优于配置（Convention over Configuration），又称为约定编程，是一种软件设计规范。本质上是对系统、类库或者框架中一些东西假定一个大众化合理的默认值（缺省值）。



# 2 SpringBoot概念

## 2.1 什么是 SpringBoot

spring官方网站：[https://spring.io/projects/spring-boot](https://spring.io/projects/spring-boot)

![image-20220301154002113](assest/image-20220301154002113.png)

*通过 Spring Boot，可以轻松地创建独立的，基于生产级别的 Spring 的应用程序，并且可以 运行 它们*。

其实 Spring Boot 的设计是为了让你尽快的跑起来 Spring 应用程序并且尽可能减少你的配置文件。

**以下内容来自百度百科**

> SpringBoot是由Pivotal团队在2013年开始研发，2014年4月发布第一个版本的全新开源的轻量级框架。它基于Spring4.0设计，不仅继承了Spring框架原有的优秀特性，而且还通过简化配置来进一步简化了Spring应用的整个搭建和开发过程，另外SpringBoot通过集成大量的框架使得依赖包的版本冲突，以及引用的不稳定性等问题得到了很好的解决。



## 2.2 SpringBoot主要特性

1. SpringBoot Starter：它将常用的依赖分组进行了整合，将其合并到一个依赖中，这样就可以一次性添加到项目的Maven或Gradle构建中；
2. 使编码变得简单，SpringBoot采用 JavaConfig 的方式对Spring进行配置，并且提供了大量的注解，极大的提高了工作效率。
3. 自动配置：SpringBoot的自动配置特性利用了Spring对条件化配置的支持，合理的推测应用所需的bean并自动化配置它们；
4. 使部署变得简单，SpringBoot内置了三种Servlet容器，Tomcat，Jetty，undertow。我们只需要一个Java的运行环境就可以跑SpringBoot的项目了，SpringBoot的项目可以打成一个jar包。

# 3 SpringBoot 案例实现

## 3.1 案例实现

**案例需求**：使用Spring Initializr 方式构建 Spring Boot 项目，并请求 Controller 中的目标方法，将返回值相应到页面。

**1 使用Spring Initializr方式构建Spring Boot项目**

本质上说，Spring Initializr 是一个Web应用，它提供了一个基本的项目结构，能够帮助我们快速构建一个基础的Spring Boot项目。

![image-20220301164807510](assest/image-20220301164807510.png)

Project SDK 用于设置创建项目使用的JDK版本，这里，使用 JDK 11；在 Choose Initializr Service URL（选择初始化服务地址）下使用默认的初始化服务地址 https://start.spring.io/，进行Spring Boot项目创建（注意使用快捷方式创建Spring Boot现目时，所在主机必须在联网状态下）

![image-20220301165520803](assest/image-20220301165520803.png)

![image-20220301165756923](assest/image-20220301165756923.png)

SpringBoot项目就创建好了，创建好的Spring Boot项目结构如图：

![image-20220301171626729](assest/image-20220301171626729.png)

使用Spring Initializr方式构建的Spring Boot会默认生成项目的启动类，存放前端静态资源和页面的文件夹、编写项目配置的配置文件以及进行项目单元测试的测试类。

**2 创建Controller**

com.turbo包下创建名为 controller 的包，在该包下创建一个请求处理控制类 DemoController，并编写一个请求处理方法。

```java
@RestController // 该注解为组合注解，等同于Spring中 @Controller + @ResponseBody 注解
public class DemoController {

    @RequestMapping("/hello")
    public String demo(){
        return "hello spring boot";
    }
}
```

**3 运行项目**

运行主程序启动类：SpringbootDemoApplication，项目启动成功后，在控制台上会发现 Spring Boot 项目默认启动的端口号为 8080，此时，可以在浏览器上访问 ：http://localhost:8080/hello

![image-20220301173744124](assest/image-20220301173744124.png)

至此，构建Spring Boot项目就完成了

**附：解决中文乱码：**

解决方法一：

```java
@RequestMapping(produces = "application/json;charset=utf-8")
```

解决方法二：

```properties
#设置响应为 utf-8，（高版本的SpringBoot已经弃用）
spring.http.encoding.force-response=true
```

## 3.2 疑问

1. starter是什么？我们如何去使用这些starter？
2. 为什么包扫描只会扫描核心启动类所在的包及其子包？
3. 在SpringBoot启动的过程中，是如何完成自动装配的？
4. 内嵌Tomcat是如何被创建及启动的？
5. 使用了web场景对应的starter，springmvc是如何自动装配？

在源码剖析部分，解答以上疑问。



# 4 热部署

spring开发团队带来一个插件：spring-boot-devtools，很好的解决了本地验证缓慢的问题。

## 4.1 热部署实现演示

1. **添加spring-boot-devtools热部署依赖启动器**

   在Spring Boot项目进行热部署测试之前，需要现在项目的pom.xml文件中添加 spring-boot-devtools热部署依赖启动器：

   ```xml
   <!-- 引入热部署依赖 -->
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-devtools</artifactId>
       <version>2.4.2</version>
   </dependency>
   ```

   由于使用的是IDEA开发工具，添加热部署以来后可能没有任何效果，接下来还需要针对IDEA开发工具进行热部署相关的功能设置

2. **IDEA工具热部署设置**

   ![image-20220301181456700](assest/image-20220301181456700.png)

   在项目任意页面使用组合快捷键 “Ctrl+Shift+Alt+/”，打开 Registry 页面，

   ![image-20220301181559504](assest/image-20220301181559504.png)

   列表中找到“complier.automake.allow.when.app.running”，将该选项后的Value值勾选，用于指定 IDEA 工具在程序运行过程中自动编译，最后单击【Close】按钮完成设置。

3. **热部署效果测试**

   启动服务，http://localhost:8080/hello

   ![image-20220301173744124](assest/image-20220301173744124.png)

   接下来，在不关闭当前项目的情况下，将DemoController类中的请求处理方法 demo() 的返回值修改为 "你好 spring boot"，查看控制台信息会发现项目能够自动构建和编译，说明热部署生效。

   ![image-20220301183346162](assest/image-20220301183346162.png)

## 4.2 热部署原理分析

## 4.3 排除资源

# 5 全局配置文件

# 6 属性注入

# 7 SpringBoot日志框架