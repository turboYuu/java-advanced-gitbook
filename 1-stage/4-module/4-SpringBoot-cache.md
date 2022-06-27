第四部分 SpringBoot缓存深入

# 1 JSR 107

JSR 是 Java Specification Requests 的缩写，Java 规范请求，顾名思义提交Java规范，JSR-107 就是关于如何使用缓存的规范，是 Java 提供的一个接口规范，类似于 JDBC 规范，没有具体的实现，具体的实现就是 Redis 等这些缓存。

## 1.1 JSR 107 核心接口

Java Caching（JSR-107）定义了5个核心接口，分别是 CahcingProvider 、CacheManager、Cache、Entry、Expiry。

- CachingProvider（缓存提供者）：创建、配置、获取、管理和控制多个 CacheManager
- CacheManager（缓存管理器）：创建、配置、获取、管理和控制多个唯一命名的Cache，Cache存在于 CacheManager的上下文中。一个 CacheManager仅对应一个 CachingProvider。
- Cache（缓存）：是由 CacheManager管理的，CacheManager管理Cache的生命周期，Cache存在于 CacheManager 的上下文中，是一个类似 map 的数据结构，并临时存储以 key 为索引的值，一个 Cache 仅被一个 CacheManager所拥有。
- Entry（缓存键值对）是一个存储在 Cache 中的 key-value对。
- Expiry（缓存时效）：每一个存储在 Cache 中的条目都有一个定义的有效期。一旦超过这个时间，条目就自动过期，过期后，条目将不可访问、更新和删除操作。缓存有效期可以通过ExpiryPolicy设置。



## 1.2 JSR 107 图示

![image-20220624164448656](assest/image-20220624164448656.png)

一个应用里面可以由多个缓存提供者（CachingProvider），一个缓存缓存提供者可以获取到多个缓存管理器（CacheManager），一个缓存管理器管理着不同的缓存（Cache），缓存中是一个个的缓存键值对（Entry），每个entry都有一个有效期（Expiry）。缓存管理器和缓存之间的关系有点类似于数据库中的连接池和连接的关系。

使用 JSR-107 需导入的依赖

```xml

```

在实际使用时，并不会使用 JSR-107 提供的接口，而是具体使用 Spring 提供的缓存抽象。

# 2 Spring 的缓存抽象

[Spring中cache的官方文档](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache)

## 2.1 缓存抽象定义

Spring从3.1开始定义了 org.springframework.cache.Cache 和 org.springframework.cache.CacheManager 接口来统一不同的缓存技术；并支持使用 Java Caching （JSR-107）注解简化缓存开发。

Spring Cache 只负责维护抽象层，具体的实现由自己的技术选型来决定。将缓存处理和缓存技术解除耦合。

每次调用需要缓存功能的方法时，Spring会检查指定参数的指定的目标方法是否已经被调用过，如果有直接从缓存中获取方法调用后的结果，如果没有就调用方法并缓存结果返回给用户。下次调用直接从缓存中获取。

使用Spring缓存抽象时我们需要关注以下两点：

1. 确定哪些方法需要被缓存
2. 缓存策略

## 2.2 重要接口

- Cache：缓存抽象的规范接口，缓存实现有：RedisCache、EhCache、ConcurrentMapCache等。
- CacheManager：缓存管理器，管理 Cache 的声明周期

# 3 Spring 缓存使用

## 3.1 重要概念 缓存注解

案例实践之前，先介绍 Spring 提供的重要缓存及几个重要概念。

| 概念/注解      | 作用                                                         |
| -------------- | ------------------------------------------------------------ |
| Cache          | 缓存接口、定义缓存操作。实现有：RedisCache、EhCacheCache、ConcurrentMapCache 等 |
| CacheManager   | 缓存管理器，管理各种缓存（Cache）组件                        |
| @Cacheable     | 主要针对方法配置，能够根据方法的请求参数对其结果进行缓存     |
| @CacheEvict    | 清空缓存                                                     |
| @CachePut      | 保证方法被调用，又希望结果被缓存                             |
| @EnableCaching | 开启基于注解的缓存                                           |
| keyGenerator   | 缓存数据时 key 生成策略                                      |
| serialize      | 缓存数据时 value 序列化策略                                  |

**说明**：

1. @Cacheable 标注在方法上，表示该方法的结果需要被缓存起来，缓存的键由 keyGenerator 的策略决定，缓存的值的形式则由 serialize 序列化策略决定（序列化还是 json 格式）；标注上该注解之后，在缓存时效内再次调用该方法时将不会调用方法本身而是直接从缓存换取结果。
2. @CachePut 也标注在方法上，和 @Cacheable 相似也会将该方法的返回值缓存起来，不同的是标注 @CachePut 的方法每次都会被调用，而且每次都会将结果缓存起来，适用于对象的更新。

## 3.2 环境搭建

![image-20220624184817641](assest/image-20220624184817641.png)

![image-20220624184914689](assest/image-20220624184914689.png)

![image-20220624185100262](assest/image-20220624185100262.png)

![image-20220624185135007](assest/image-20220624185135007.png)

1. 创建 SpringBoot应用，选中 Mysql、Mybatis、Web 模块

2. 创建数据库

   ```sql
   DROP TABLE IF EXISTS `department`;
   
   CREATE TABLE `department` (
   	`id` INT (11) NOT NULL AUTO_INCREMENT,
   	`departmentName` VARCHAR (255) DEFAULT NULL,
   	PRIMARY KEY (`id`)
   ) ENGINE = INNODB DEFAULT CHARSET = utf8;
   
   DROP TABLE IF EXISTS `employee`;
   
   CREATE TABLE `employee` (
   	`id` INT (11) NOT NULL AUTO_INCREMENT,
   	`lastName` VARCHAR (255) DEFAULT NULL,
   	`email` VARCHAR (255) DEFAULT NULL,
   	`gender` INT (2) DEFAULT NULL,
   	`d_id` INT (11) DEFAULT NULL,
   	PRIMARY KEY (`id`)
   ) ENGINE = INNODB DEFAULT CHARSET = utf8;
   
   
   INSERT INTO `department` (`departmentName`) VALUES ('开发部');
   INSERT INTO `employee` (`lastName`, `email`, `gender`, `d_id`) VALUES (威廉', 'oath@gmail.com', '1', '1');
   ```

3. 创建表对应的实体Bean

   ```java
   package com.turbo.pojo;
   
   import lombok.Data;
   
   @Data
   public class Employee {
   	private Integer id;
   	private String lastName;
   	private String email;
   	//性别  1男  0女
   	private Integer gender;
   	private Integer dId;
   
   }
   ```

   ```java
   package com.turbo.pojo;
   
   import lombok.Data;
   
   @Data
   public class Department {
   	private Integer id;
   	private String departmentName;
   }
   ```

4. 整合Mybatis操作数据库

   数据源配置：驱动可以不写，SpringBoot会根据连接自动判断

   ```properties
   spring.datasource.url=jdbc:mysql://152.136.177.192:3306/turbine
   spring.datasource.username=root
   spring.datasource.password=123456
   #spring.datasource.driver-class-name=com.mysql.jdbc.Driver
   
   # 开启驼峰
   mybatis.configuration.map-underscore-to-camel-case=true
   ```

   使用注解版 Mybatis：使用@MapperScan指定mapper接口所在的包

   ```java
   package com.turbo;
   
   import org.mybatis.spring.annotation.MapperScan;
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   
   @SpringBootApplication
   @MapperScan("com.turbo.mappers")
   public class SpringBoot04CacheApplication {
   
   	public static void main(String[] args) {
   		SpringApplication.run(SpringBoot04CacheApplication.class, args);
   	}
   }
   ```

   创建对应的 mapper 接口

   ```java
   package com.turbo.mappers;
   
   import com.turbo.pojo.Employee;
   import org.apache.ibatis.annotations.Delete;
   import org.apache.ibatis.annotations.Insert;
   import org.apache.ibatis.annotations.Select;
   import org.apache.ibatis.annotations.Update;
   
   public interface EmployeeMapper {
   
   	@Select("select * from employee where id = #{id}")
   	public Employee getEmpById(Integer id);
   
   	@Insert("insert into employee (lastName,email,gender,d_id) VALUES (#{lastName},#{email},#{gender},#{d_id})")
   	public void insertEmp(Employee employee);
   
   	@Update({"update empolyee SET lastName = #{lastName},email = #{email},gender = #{gender},d_id=#{d_is}"})
   	public void updateEmp(Employee employee);
   
   	@Delete("delete from employee where id=#{id}")
   	public void deleteEmp(Integer id);
   }
   ```

   编写service：

   ```java
   package com.turbo.service;
   
   import com.turbo.mappers.EmployeeMapper;
   import com.turbo.pojo.Employee;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.stereotype.Service;
   
   @Service
   public class EmployeeService {
   
   	@Autowired
   	private EmployeeMapper employeeMapper;
   
   
   	public Employee getEmpId(Integer id){
   		Employee employee = employeeMapper.getEmpById(id);
   		return employee;
   	}
   }
   ```

   编写Controller：

   ```java
   package com.turbo.controller;
   
   import com.turbo.pojo.Employee;
   import com.turbo.service.EmployeeService;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.PathVariable;
   import org.springframework.web.bind.annotation.RestController;
   
   @RestController
   public class EmployeeController {
   
   	@Autowired
   	EmployeeService employeeService;
   
   	@GetMapping("/emp/{id}")
   	public Employee getEmp(@PathVariable("id") Integer id){
   		return employeeService.getEmpId(id);
   	}
   }
   ```

5. 测试

   测试之前可以先配置一下 Logger日志，让控制台将SQL打印出来：

   ```properties
   logging.level.com.turbo.mappers=debug
   ```

   ![image-20220627135958504](assest/image-20220627135958504.png)

   结论：当前还没有看到缓存效果，因为还没有进行缓存的相关配置。

## 3.3 @Cacheable初体验

1. 开启基于注解的缓存功能：主启动类标注 @EnableCaching

   ```java
   @SpringBootApplication
   @MapperScan("com.turbo.mappers")
   @EnableCaching
   public class SpringBoot04CacheApplication {
   
   	public static void main(String[] args) {
   		SpringApplication.run(SpringBoot04CacheApplication.class, args);
   	}
   }
   ```

2. 标注缓存相关注解：@Cacheable、@CacheEvict、@CachePut

   @Cacheable：将方法运行的结果进行缓存，以后获取相同的数据时，直接从缓存中换取，不再调用方法。

   ```java
   @Cacheable(cacheNames = {"emp"})
   public Employee getEmpId(Integer id){
       Employee employee = employeeMapper.getEmpById(id);
       return employee;
   }
   ```

   

   

@Cacheable 注解的属性：

| 属性名           | 描述                                                         |
| ---------------- | ------------------------------------------------------------ |
| cacheNames/vlaue | 指定缓存的名字，缓存使用CacheManager管理多个缓存组件Cache，<br>这些Cache组件就是根据这个名字进行区分的。<br>对缓存的真正CRUD操作在Cache中定义，<br>每个缓存组件Cache都有自己唯一的名字，通过cacheNames或者value属性指定，<br>相当于是将缓存的键值对进行分组，缓存的名字是一个数组，<br>也就是说可以将一个缓存键值对分到多个组里面。 |
| key              | 缓存数据时的key值，默认是使用方法参数的值，可以使用 SpEL表达式计算key的值 |
| keyGenerator     | 缓存的生成策略，和key二选一，都是生成键的，keyGenerator可自定义。 |
| cacheManager     | 指定缓存管理器（如 ConcurrentHashMap、Redis等）              |
| cacheResolver    | 和cacheManager功能一样，和cacheManager二选一                 |
| condition        | 指定缓存的条件（**满足什么条件时才缓存**），可用SpEL表达式（如#id>0，表示当入参id大于0时才缓存） |
| unless           | 否定缓存，**即满足unless指定的条件时，方法的结果不进行缓存**，使用unless时可以在调用的方法获取到结果之后再进行判断（如#result==null，表示如果结果为null时不缓存） |
| sync             | 是否使用异步模式进行缓存                                     |

**注意**：**既满足condition又满足unless条件的也不进行缓存**，**使用异步模式进行缓存时 (sync=true):unless条件将不被支持**

可用的[SpEL表达式](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache-spel-context)见下表：

| 名字          | 位置               | 描述                                                         | 示例                 |
| ------------- | ------------------ | ------------------------------------------------------------ | -------------------- |
| methodName    | root object        | 当前被调用的方法名                                           | #root.methodName     |
| method        | root object        | 当前被调用的方法                                             | #root.method.name    |
| target        | root object        | 当前被调用的目标对象                                         | #root.target         |
| targetClass   | root object        | 当前被调用的目标对象类                                       | #root.targetClass    |
| args          | root object        | 当前被调用的方法的参数列表                                   | #root.args[0]        |
| caches        | root object        | 当前方法调用使用的缓存列表，<br>(如 @Cacheable=(value={"cache1","cache2"}))，<br>则有两个cache | #root.caches[0].name |
| argument name | Evaluation context | 方法参数的名字，可以直接 `#参数名`，<br>也可以使用 `#p0` 或 `#a0` 的形式，0代表参数的索引 | #iban、#a0、#p0      |
| result        | Evaluation context | 方法执行后的返回值（仅当方法执行之后的判断有效，<br>如 "unless","cache put" 的表达式，<br>"cache evict" 的表达式 beforeInvocation=false） | #result              |



# 4 缓存自动配置原理源码剖析

# 5 @Cacheable 源码分析

# 6 @CahcePut、@CacheEvict、@CacheConfig

# 7 基于Redis的缓存实现

# 8 自定义 RedisCacheManager