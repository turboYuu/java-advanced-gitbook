> 第九部分 Mybatis架构原理

# 1 架构设计

我们把Mybatis的功能架构分为三层：

1. API 接口层：
2. 数据处理层
3. 基础支撑层

# 2 主要构建及其相关关系

| 构件             | 描述 |
| ---------------- | ---- |
| SqlSession       |      |
| Executor         |      |
| StatementHandler |      |
| ParameterHandler |      |
| ResultSetHandler |      |
| TypeHandler      |      |
| MappedStatement  |      |
| SqlSource        |      |
| BoundSql         |      |



# 3 总体流程