> 4 配置

在MP中有大量的配置，其中一部分是Mybatis原生的配置，另一部分是 MP 的配置，详情：https://baomidou.com/pages/56bac0/#%E5%9F%BA%E6%9C%AC%E9%85%8D%E7%BD%AE

下面对常用的配置做讲解

# 1 基本配置

## 1.1 configLocation

Mybatis 配置文件位置，如果有单独的Mybatis配置，请将其路径配置到 configLocation 中。Mybatis Configuration 的具体内容参考 Mybatis 官方文档

Spring Boot:

```properties
mybatis-plus.config-location=classpath:mybatis-config.xml
```

Spring :

```xml
<bean id="sqlSessionFactory" class="com.baomidou.mybatisplus.extension.spring.MybatisSqlSessionFactoryBean">
    <property name="configuration" value="classpath:mybatis-config.xml"/><!--非必须-->
</bean>
```



## 1.2 mapperLocations



## 1.3 typeAliasesPackage

# 2 进阶配置

# 3 DB策略配置