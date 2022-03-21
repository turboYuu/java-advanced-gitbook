第三部分 SpringBoot数据访问

# 1 数据源自动配置源码剖析

## 1.1 数据源配置方式

### 1.1.1 选择数据库驱动的库文件

在 maven 中配置数据库驱动

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
</dependency>
```



### 1.1.2 配置数据库连接

在application.properties中配置数据库连接

```properties
# 数据库连接相关配置
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://152.136.177.192:3306/springboot_h?useUnicode=true&characterEncoding=utf-8&useSSL=true&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=123456
```



### 1.1.3 配置 spring-boot-starter-jdbc

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```



### 1.1.4 编写测试类

```java
@RunWith(SpringRunner.class)
@SpringBootTest
class SpringBoot03DataaccessApplicationTests {

	@Autowired
	private DataSource dataSource;

	@Test
	void contextLoads() throws SQLException {
		final Connection connection = dataSource.getConnection();
		System.out.println(connection);
	}
}
```

结果：

![image-20220320233939433](assest/image-20220320233939433.png)

## 1.2 连接池配置方式

### 1.2.1 选择数据库连接池的库文件

SpringBoot 提供了三种数据库连接池：

- HikariCP
- Commons DBCP2
- Tomcat JDBC Connection Pool

其中 SpringBoot 2.x 版本默认使用 HikariCP，Maven 中配置如下：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
```

如果不使用HikariCP，改用 Commons DBCP2 ，则配置如下：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
    <exclusions>
        <exclusion>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-dbcp2</artifactId>
</dependency>
```

如果不使用HikariCP，改用 Tomcat JDBC Connection Pool ，则配置如下：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
    <exclusions>
        <exclusion>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>tomcat-jdbc</artifactId>
</dependency>
```

> 思考：为什么说SpringBoot默认使用的连接池类型时HikariCP，在哪里指定的？

## 1.3 数据源自动配置

spring.factories 中找到数据源的配置类：

![image-20220321000528589](assest/image-20220321000528589.png)

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class, 
         DataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {

	@Configuration(proxyBeanMethods = false)
	@Conditional(EmbeddedDatabaseCondition.class)
	@ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
	@Import(EmbeddedDataSourceConfiguration.class)
	protected static class EmbeddedDatabaseConfiguration {

	}

	@Configuration(proxyBeanMethods = false)
	@Conditional(PooledDataSourceCondition.class)
	@ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
	@Import({ DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
			DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.Generic.class,
			DataSourceJmxConfiguration.class })
	protected static class PooledDataSourceConfiguration {

	}
    // ...
}
```

@Conditional(PooledDataSourceCondition.class) 根据判断条件，实例化这个类，指定了配置文件中必须有 type 这个属性：

![image-20220321103309686](assest/image-20220321103309686.png)

另外 SpringBoot 默认支持 type 类型设置的数据源：

![image-20220321103642742](assest/image-20220321103642742.png)

```java
abstract class DataSourceConfiguration {

	@SuppressWarnings("unchecked")
	protected static <T> T createDataSource(DataSourceProperties properties, 
                                            Class<? extends DataSource> type) {
		return (T) properties.initializeDataSourceBuilder().type(type).build();
	}

	/**
	 * Tomcat Pool DataSource configuration.
	 * 2.0 之后默认不再使用 tomcat 连接池，或者使用 tomcat 容器
	 */
	@Configuration(proxyBeanMethods = false)
	// 如果导入 tomcat jdbc 连接池，则使用此连接池，在使用 tomcat 容器时候，或者导入此包时候
	@ConditionalOnClass(org.apache.tomcat.jdbc.pool.DataSource.class)
	@ConditionalOnMissingBean(DataSource.class)
	// 并且配置的时 org.apache.tomcat.jdbc.pool.DataSource 会采用 tomcat 连接池
	@ConditionalOnProperty(name = "spring.datasource.type", // name 用来从 application.properties中读取某个属性值
			havingValue = "org.apache.tomcat.jdbc.pool.DataSource", // 缺少该 property 时是否可以加载：如果true，没有该property也会正常加载；反之报错
			matchIfMissing = true) // 不管配不配置，都以 tomcat 连接池作为连接池 默认值 false
	static class Tomcat {

		// 给容器中加数据源
		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.tomcat")
		org.apache.tomcat.jdbc.pool.DataSource dataSource(DataSourceProperties properties) {
			org.apache.tomcat.jdbc.pool.DataSource dataSource = createDataSource(properties,
					org.apache.tomcat.jdbc.pool.DataSource.class);
			DatabaseDriver databaseDriver = DatabaseDriver.fromJdbcUrl(properties.determineUrl());
			String validationQuery = databaseDriver.getValidationQuery();
			if (validationQuery != null) {
				dataSource.setTestOnBorrow(true);
				dataSource.setValidationQuery(validationQuery);
			}
			return dataSource;
		}
	}

	/**
	 * Hikari DataSource configuration.
	 * 2.0 之后默认使用 hikari 连接池
	 */
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(HikariDataSource.class)
	@ConditionalOnMissingBean(DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", 
                           havingValue = "com.zaxxer.hikari.HikariDataSource",
			matchIfMissing = true)
	static class Hikari {

		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.hikari")
		HikariDataSource dataSource(DataSourceProperties properties) {
			HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);
			if (StringUtils.hasText(properties.getName())) {
				dataSource.setPoolName(properties.getName());
			}
			return dataSource;
		}
	}

	/**
	 * DBCP DataSource configuration.
	 * dbcp2 连接池
	 */
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(org.apache.commons.dbcp2.BasicDataSource.class)
	@ConditionalOnMissingBean(DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", 
                           havingValue = "org.apache.commons.dbcp2.BasicDataSource",
			matchIfMissing = true)
	static class Dbcp2 {

		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.dbcp2")
		org.apache.commons.dbcp2.BasicDataSource dataSource(DataSourceProperties properties) {
			return createDataSource(properties, org.apache.commons.dbcp2.BasicDataSource.class);
		}
	}

	/**
	 * Generic DataSource configuration.
	 * 自定义连接池接口，spring.datasource.type
	 */
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingBean(DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type")
	static class Generic {

		@Bean
		DataSource dataSource(DataSourceProperties properties) {
			// 创建数据源 initializeDataSourceBuilder DataSourceBuilder
			return properties.initializeDataSourceBuilder().build();
		}
	}
}
```

如果在类路径没有找到 jar 包，则会抛出异常：

> 

配置文件中没有指定数据源的时候，会根据注解判断，然后选择相应的实例化数据源对象。

```java
/**
* Hikari DataSource configuration.
* 2.0 之后默认使用 hikari 连接池
*/
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(HikariDataSource.class)
@ConditionalOnMissingBean(DataSource.class) // 注解判断是否执行初始化代码，即如果用户已经创建了bean，则相关的初始化代码不再执行
@ConditionalOnProperty(
    name = "spring.datasource.type", // 获取配置文件中的 type，如果为空返回 false                   
    havingValue = "com.zaxxer.hikari.HikariDataSource", // type 不为空则和 havingValue 对比，相同则true,否则false                   
    matchIfMissing = true) // 不管上面文件中是否匹配，默认都进行加载，默认值 false
static class Hikari {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.hikari")
    HikariDataSource dataSource(DataSourceProperties properties) {
        // 创建数据源
        HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);
        if (StringUtils.hasText(properties.getName())) {
            dataSource.setPoolName(properties.getName());
        }
        return dataSource;
    }
}
```

createDataSource 方法：

```java
protected static <T> T createDataSource(DataSourceProperties properties, 
                                        Class<? extends DataSource> type) {
    // 使用 DataSourceBuilder 创建数据库，利用反射创建type数据源，然后绑定相关属性
    return (T) properties.initializeDataSourceBuilder().type(type).build();
}
```

**DataSourceBuilder** 类

![image-20220321143707871](assest/image-20220321143707871.png)

设置 type

```java
public <D extends DataSource> DataSourceBuilder<D> type(Class<D> type) {
    this.type = type;
    return (DataSourceBuilder<D>) this;
}
```

```java
@SuppressWarnings("unchecked")
public T build() {
    // getType 点进去
    Class<? extends DataSource> type = getType();
    DataSource result = BeanUtils.instantiateClass(type);
    maybeGetDriverClassName();
    bind(result);
    return (T) result;
}
```

根据设置选择type类型：

```java
private Class<? extends DataSource> getType() {
    // 如果没有配置 type 则为空 默认选择 findType
    Class<? extends DataSource> type = (this.type != null) ? this.type : findType(this.classLoader);
    if (type != null) {
        return type;
    }
    throw new IllegalStateException("No supported DataSource type found");
}
```

```java
public static Class<? extends DataSource> findType(ClassLoader classLoader) {
    for (String name : DATA_SOURCE_TYPE_NAMES) {
        try {
            return (Class<? extends DataSource>) ClassUtils.forName(name, classLoader);
        }
        catch (Exception ex) {
            // Swallow and continue
        }
    }
    return null;
}
```

数组 `DATA_SOURCE_TYPE_NAMES`：

```java
private static final String[] DATA_SOURCE_TYPE_NAMES = new String[] 
{ "com.zaxxer.hikari.HikariDataSource",
 "org.apache.tomcat.jdbc.pool.DataSource", 
 "org.apache.commons.dbcp2.BasicDataSource" };
```

取出来的第一个值就是 `com.zaxxer.hikari.HikariDataSource`，那么证实在没有指定 type 的情况下，默认类型为：`com.zaxxer.hikari.HikariDataSource`。



# 2 Druid连接池配置

## 2.1 整合效果实现

1. 在pom.xml中引入druid数据源

   ```xml
   <dependency>
       <groupId>com.alibaba</groupId>
       <artifactId>druid-spring-boot-starter</artifactId>
       <version>1.1.10</version>
   </dependency>
   ```

2. 在application.yml中引入druid的相关配置

   ```yaml
   spring:
     datasource:
       username: root
       password: 123456
       url: jdbc:mysql://152.136.177.192:3306/springboot_h?useUnicode=true&characterEncoding=utf-8&useSSL=true&serverTimezone=UTC
       driver-class-name: com.mysql.cj.jdbc.Driver
       initialization-mode: always
       # 使用druid数据源
       type: com.alibaba.druid.pool.DruidDataSource
       # 数据源其他配置
       initialSize: 5
       minIdle: 5
       maxActive: 20
       maxWait: 60000
       timeBetweenEvictionRunsMillis: 60000
       minEvictableIdleTimeMillis: 300000
       validationQuery: SELECT 1 FROM DUAL
       testWhileIdle: true
       testOnBorrow: false
       testOnReturn: false
       poolPreparedStatements: true
       # 配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙
       filters: stat,wall,log4j
       maxPoolPreparedStatementPerConnectionSize: 20
       useGlobalDataSourceStat: true
       connectionProperties: druid.stat.mergeSql=true;druid.stat.slowSqlMillis=500
   ```

3. 进行测试：

   ![image-20220321155504683](assest/image-20220321155504683.png)

   但是debug查看 DataSource 的值，会发现有些属性是没有生效的：

   ![image-20220321155432592](assest/image-20220321155432592.png)

   这是因为：如果单纯在 yml 文件中编写如上的配置，SpringBoot肯定是读取不到 druid 的相关配置的。因为它并不像原生的 jdbc，系统默认就使用 `DataSourceProperties` 与其属性进行了绑定。所以我们应该编写一个类与其属性进行绑定

4. 编写整合 druid 的配置类 DruidConfig

   ```java
   package com.turbo.config;
   
   import com.alibaba.druid.pool.DruidDataSource;
   import org.springframework.boot.context.properties.ConfigurationProperties;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   
   import javax.sql.DataSource;
   
   @Configuration
   public class DruidConfig {
   
   	@Bean
   	@ConfigurationProperties(prefix = "spring.datasource")
   	public DataSource dataSource(){
   		return new DruidDataSource();
   	}
   }
   ```

   测试的时候，发现报错。发现是yml文件里的：

   ```yaml
   # 配置监控统计拦截的filters，去掉后监控界面sql无法统计，'wall'用于防火墙
   filters: stat,wall,log4j
   ```

   因为 SpringBoot 2.0 以后使用的日志框架已经不再使用 log4j。此时引入相应的适配器，可以在 pom.xml 文件上加入：

   ```xml
   <!--引入适配器-->
   <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-log4j12</artifactId>
   </dependency>
   ```

   OK。

   ![image-20220321160520452](assest/image-20220321160520452.png)

# 3 SpringBoot整合 Mybatis

Mybatis 是一款优秀的持久层框架，SpringBoot 官方虽然没有对 Mybatis 进行整合，但是 Mybatis 团队自定适配了对应的启动器，进一步简化了使用 Mybatis 进行数据的操作

因为 SpringBoot 框架开发的便利性，所以实现 SpringBoot 与 数据访问层框架（例如 Mybatis）的整合非常简单，主要是引入对应的依赖启动器，并进行数据库相关参数设置即可。

## 3.1 整合效果实现

1. spring-boot-03-dataaccess 项目，并导入 mybatis 的 pom.xml 配置

   ```xml
   <dependency>
       <groupId>org.mybatis.spring.boot</groupId>
       <artifactId>mybatis-spring-boot-starter</artifactId>
       <version>1.3.2</version>
   </dependency>
   <dependency>
       <groupId>org.projectlombok</groupId>
       <artifactId>lombok</artifactId>
       <optional>true</optional>
   </dependency>
   ```

   application.yml

   ```yaml
   spring:
     datasource:
       username: root
       password: 123456
       url: jdbc:mysql://152.136.177.192:3306/springboot_h?useUnicode=true&characterEncoding=utf-8&useSSL=true&serverTimezone=UTC
       driver-class-name: com.mysql.cj.jdbc.Driver
       initialization-mode: always
       # 使用druid数据源
       type: com.alibaba.druid.pool.DruidDataSource
   ```

2. 基础类

   ```java
   package com.turbo.pojo;
   
   import lombok.Data;
   
   @Data
   public class User {
   	private Integer id;
   	private String username;
   	private Integer age;
   }
   ```

   

3. 测试 dao（mybatis 使用注解开发）

   ```java
   package com.turbo.mapper;
   
   import com.turbo.pojo.User;
   import org.apache.ibatis.annotations.Select;
   
   import java.util.List;
   
   public interface UserMapper {
   
   	@Select("select * from user")
   	public List<User> findAllUser();
   }
   ```

   在启动类上增加注解：`@MapperScan("com.turbo.mapper")`

4. 测试 service

   ```java
   package com.turbo.service;
   
   import com.turbo.mapper.UserMapper;
   import com.turbo.pojo.User;
   import org.slf4j.Logger;
   import org.slf4j.LoggerFactory;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.stereotype.Service;
   import java.util.List;
   
   @Service
   public class UserService {
   
   	final Logger logger = LoggerFactory.getLogger(UserService.class);
   
   	@Autowired
   	private UserMapper userMapper;
   
   	public List<User> findAllUser(){
   		final List<User> allUser = userMapper.findAllUser();
   		logger.info("查询出来的用户信息："+allUser.toString());
   		return allUser;
   	}
   }
   ```

5. service对应的 test类

   ```java
   @Autowired
   private UserService userService;
   
   @Test
   public void test1(){
       final List<User> allUser = userService.findAllUser();
   }
   ```

6. 运行测试类输出结果

   ![image-20220321235100946](assest/image-20220321235100946.png)

# 4 Mybatis 自动配置源码分析

1. SpringBoot 项目最核心的就是自动加载配置，该功能则依赖的是一个注解 `@SpringBootApplication` 中的 `@EnableAutoConfiguration`
2. `@EnableAutoConfiguration` 主要是通过 `AutoConfigurationImportSelector` 类来加载。

以 mybatis 为例，`AutoConfigurationImportSelector`  通过反射加载 spring.factories 中指定的 java 类，也就是加载 MybatisAutoConfiguration 类（该类有 @Configuration 注解，属于配置类）



# 5 SpringBoot + Mybatis 实现动态数据源切换

## 5.1 动态数据源介绍

## 5.2 环境准备

## 5.3 具体实现

### 5.3.1 配置多数据源

### 5.3.2 编写 RoutingDataSource

## 5.4 优化