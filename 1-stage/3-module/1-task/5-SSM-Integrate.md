> 第五部分 SSM 整合

# 1 整合策略

SSM = Spring + SpringMVC + Mybatis = (Spring + Mybatis) + SpringMVC

先整合 Spring + Mybatis，然后再整合 SpringMVC

基于的需求：查询 Account 表的全部数据显示到页面。

新建一个 web、maven 项目。

# 2 Mybatis 整合 Spring

## 2.1 整合目标

- 数据库连接池以及事务管理都交给 Spring 容器来完成
- SqlSessionFactory 对象应该放到 Spring 容器中作为单例对象管理
- Mapper 动态代理对象交给 Spring 管理，我们从 Spring 容器中直接获得 Mapper 的代理对象

## 2.2 整合所需 Jar 分析

- Junit测试 jar （4.12 版本）
- Mybatis的 jar （3.4.5）
- Spring 相关 jar（spring-context、spring-text、spring-jdbc、spring-tx、spring-aop、aspectweaver）
- Mybatis/Spring 整合包 jar （mybatis-spring-xx.jar）
- MySQL 数据库驱动 jar
- Druid 数据库连接池的 jar

## 2.3 整合后的 pom 坐标

# 3 整合 SpringMVC

