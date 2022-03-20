第三部分 SpringBoot数据访问

# 1 数据源自动配置源码剖析

## 1.1 数据源配置方式

### 1.1.1 选择数据库驱动的库文件

在 maven 中配置数据库驱动

```xml

```



### 1.1.2 配置数据库连接

在application.properties中配置数据库连接

```properties

```



### 1.1.3 配置 spring-boot-starter-jdbc

```xml

```



### 1.1.4 编写测试类

## 1.2 连接池配置方式

### 1.2.1 选择数据库连接池的库文件

## 1.3 数据源自动配置

# 2 Druid连接池配置

## 2.1 整合效果实现

# 3 SpringBoot整合 Mybatis

## 3.1 整合效果实现

# 4 Mybatis 自动配置源码分析

# 5 SpringBoot + Mybatis 实现动态数据源切换

## 5.1 动态数据源介绍

## 5.2 环境准备

## 5.3 具体实现

### 5.3.1 配置多数据源

### 5.3.2 编写 RoutingDataSource

## 5.4 优化