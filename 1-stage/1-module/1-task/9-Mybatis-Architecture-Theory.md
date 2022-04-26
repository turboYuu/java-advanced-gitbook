> 第九部分 Mybatis架构原理

# 1 架构设计

我们把Mybatis的功能架构分为三层：

1. API 接口层：提供给外部使用的接口 API，开发人员通过这些本地 API 来操纵数据库。接口层一接收到，调用请求就会调用数据处理层来完成具体的数据处理。

   Mybatis和数据库的交互有两种方式：

   - 使用传统的 Mybatis 提供的 API
   - 使用 Mapper 代理的方式

2. 数据处理层：负责具体的SQL查找、SQL解析、SQL执行和执行结果映射处理等。它的主要目的是根据调用的请求完成一次数据库操作。

3. 基础支撑层：负责最基础的功能支撑，包括连接管理、事务管理、配置加载和缓存处理，这些都是共用的东西，将他们抽取出来作为最基础的组件。为上层的数据处理提供最基础的支撑

# 2 主要构建及其相关关系

| 构件             | 描述                                                         |
| ---------------- | ------------------------------------------------------------ |
| SqlSession       | 作为Mybatis工作的主要顶层 API，表示和数据库交互的会话，完成必要数据库增删改查功能 |
| Executor         | Mybatis执行器，是 Mybatis调度的核心，负责 SQL 语句的生成和查询缓存的维护 |
| StatementHandler | 封装了 JDBC Statement 操作，负责对 JDBC statement 的操作，如设置参数，将 Statement 结果转换成 List 集合 |
| ParameterHandler |                                                              |
| ResultSetHandler |                                                              |
| TypeHandler      |                                                              |
| MappedStatement  |                                                              |
| SqlSource        |                                                              |
| BoundSql         |                                                              |



# 3 总体流程