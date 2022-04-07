第一部分 Spring MVC应用

# 1 SpringMVC 简介

## 1.1 MVC 体系结构

在 B/S （浏览器/服务器）架构中，系统标准的三层架构包括：表现层、业务层、持久层。

- 表现层，就是常说的 web 层。它负责接收客户端响应结果，通常客户端使用 http 协议请求 web 层，web 需要接收 http 请求，完成 http 响应。

  表现层包括**展示层**和**控制层**：控制层负责接收请求，展示层负责结果的展示。

  表现层依赖业务层，接收到客户端请求一般会调用业务层进行业务处理，并将处理结果响应给客户端。

  变现层的设计一般都使用 MVC 模型。（MVC 是表现层的设计模式，和其他层没有关系）

- 业务层

  就是常说的 service 层。它负责业务逻辑处理，和我们开发项目的需求息息相关。web 层依赖 业务层，但是业务层不依赖 web 层。

  业务层在业务处理时可能会依赖持久层，如果要对数据持久化需要保证事务一致性（事务应该放到业务层来控制）。

- 持久层

  dao 层，负责数据持久化，包括 数据层即数据库和数据访问层，数据库是对数据进行持久化的载体，数据访问层是业务层和持久层交互的接口，业务层需要通过数据层将数据持久化到数据库中。通俗讲，持久化层就是和数据库交互，对数据表进行增删改查。

**MVC 设计模式**

MVC 全名是 Model View Controller，是 模型（model）- 视图（view）- 控制器（controller）的缩写，是一种用于设计创建 web 应用程序表现层的模式。MVC 中每个部分各司其职：

- Model（模型）：模型包含业务模型和数据模型，数据模型用于封装数据，业务模式用于处理业务。
- View（视图）：通常指的就是 jsp 或者 html。作用一般是展示数据，通常视图是依据数据模型创建的。
- Controller（控制器）：是应用程序中处理用户交互的部分。作用一般就是处理程序逻辑的。

MVC 提倡：每一层只编写自己的东西，不编写任何其他的代码；分层是为了解耦，解耦是为了维护方便和分工协作。

## 1.2 Spring MVC 是什么

Spring MVC 全名叫 Spring Web MVC，是一种基于 Java 的实现 MVC 设计模型的请求驱动类型的轻量级 Web 框架，属于 Spring framework 的后续产品。

[Spring MVC 官网](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html)

![image-20211019185543778](assest/image-20211019185543778.png)

它通过一套注解，让一个简单的 java 类成为处理请求的控制器，而无需实现任何接口。同时它还支持 Rsetful 编程风格的请求。

Spring MVC 本质可以认为是对 servlet 的封装，简化了我们 servlet 的开发

![image-20220406185134088](assest/image-20220406185134088.png)

# 2 Spring Web MVC 工作流程

需求：前端浏览器请求 url：[http://localhost:8080/demo/handle01](http://localhost:8080/demo/handle01)，前端页面显示后台服务器的时间。

开发过程：

1. 配置 DispatcherServlet 前端控制器
2. 

## 2.1 Spring MVC 请求处理流程

## 2.2 Spring MVC 九大组件

# 3 请求参数绑定（串讲）



# 4 对 Restful 风格请求支持

## 4.1 什么是 Restful

# 5 Ajax Json 交互

## 5.1 什么是 Json

## 5.2 @ResponseBody 注解

## 5.2 分析 Spring MVC 使用 Json 交互