模块二 IoC容器设计实现及Spring源码分析

# 1 Spring概述

## 1.4 Spring的核心结构

![image-20211019185543778](assest/image-20211019185543778.png)

# 2 核心思想

注意：IoC和AOP不是spring提出来的，在spring之前就已经存在了，只不过更偏向于理论化，spring在技术层面把这两个思想做了非常好的实现（Java）。

## 2.1 IoC

### 2.1.1 什么是IoC

IoC（Inversion of Control）控制反转/反转控制，注意它是一个技术思想，不是一个技术实现。

描述的事情：Java开发领域对象的创建，管理的问题。

比如A依赖于B，IoC思想下，是由IoC容器（Spring框架）去帮助我们实例化对象并且管理它，我们需要使用哪个对象，去找IoC容器要即可。

为什么叫做控制反转？

控制：指的是对象创建（实例化，管理）的权利

反转：控制权利交给外部环境了（Spring框架、IoC容器）

![image-20211013174209370](assest/image-20211013174209370.png)



### 2.1.2 IoC解决了什么问题

IoC解决对象之间的耦合问题

![image-20211013174547361](assest/image-20211013174547361.png)

### 2.1.3 IoC和DI的区别

DI：Dependancy Injection（依赖注入）

IOC和DI描述的是同一件事，只不过角度不一样罢了

![image-20211013174744951](assest/image-20211013174744951.png)

## 2.2 AOP

### 2.2.1 什么是AOP

AOP：Aspect oriented Programming 面向切面编程/面向方面编程

AOP是OOP的延续，从OOP说起



### 2.2.2 AOP在解决什么问题

在不改变原有的业务逻辑的情况下，增强横切逻辑代码，根本上解耦合，避免横切逻辑代码重复。

### 2.2.3 为什么叫做面向切面编程

# 3 手写实现IoC和AOP

# 4 Spring IoC应用

## 4.1 Spring IoC基础

### 4.1.1 BeanFactory与ApplicationContext区别

BeanFactory是Spring框架中的IoC容器的顶层接口，它只是用来定义一些基础功能，定义一些基础规范。而ApplicationContext是它的一个子接口，所以ApplicationContext是具备BeanFactory提供的全部功能。

通常，我们称BeanFactory为Spring IoC的基础容器，ApplicationContext是高级接口，比BeanFactory要拥有更多的功能，比如说国际化支持和资源访问（xml，java配置类）等等。



启动IoC容器的方式

- Java环境下启动IoC容器
  - ClassPathXmlApplicationContext：从类的根路径下加载配置文件（推荐使用）
  - FileSystemXmlApplicationContext：从磁盘路径上加载配置文件
  - AnnotationConfigApplicationContext：纯注解模式下启动Spring容器
- Web环境下启动IoC容器
  - 从xml启动容器
  - 从配置类启动容器

# 5 Spring IoC源码深度剖析

# 6 Spring AOP应用

## 6.4 Spring种AOP实现

### 6.4.1 XML模式

**五种通知类型**

- 前置通知

- 正常执行通知

- 异常通知

- 最终通知

  执行时机：无论切入点方法执行是否产生异常，它都会在返回之前执行

- 环绕通知

### 6.4.2 XML+注解模式

### 6.4.3 注解模式

## 6.5 Spring声明式事务

### 6.5.1 事务回顾

#### 6.5.1.1 事务的概念

事务指逻辑上的一组操作，组成这组操作的各个单元，要么全部成功，要么全部不成功。从而确保了数据的准确与安全。

#### 6.5.1.2 事务的四大特性

原子性（Atomicity）

一致性（Consistency）

隔离性（Isolation）

永久性（Durability）



#### 6.5.1.3 事务的隔离级别

不考虑隔离级别，会出现以下情况：

- 脏读
- 不可重复读
- 幻读

数据库共定义了4中隔离级别，**级别依次升高，效率依次降低**

- Read uncommitted（读未提交）
- Read committed（读已提交）
- Repeatable read（可重复读）
- Serializable（串行化）

MySQL的默认隔离级别是：Repeatable Read

查询当前使用的隔离级别：`select @@tx_isolation;`



#### 6.5.1.4 事务的传播行为

事务往往在service层进行控制，如果出现service层方法A调用了另外一个service层方法B，A和B方法本身都已经被添加了事务控制，那么A调用B的时候，就需要进行事务的一些协商，这就叫做事务的传播行为。

A调用B，我们站在B的角度来观察定义事务的传播行为

| PROPAGATION_REQUIRED      | 如果已经存在一个事务中，加入到这个事务，如果当前没有事务，就新建一个事务，这是最常见的选择 |
| ------------------------- | ------------------------------------------------------------ |
| PROPAGATION_SUPPORTS      | 使用当前事务，如果当前没有事务，就以非事务方式执行。         |
| PROPAGATION_MANDATORY     | 使用当前事务，如果当前没有事务，就抛出异常。                 |
| PROPAGATION_REQUIRES_NEW  | 新建事务，如果当前存在事务，把当前事务挂起。                 |
| PROPAGATION_NOT_SUPPORTED | 以非事务方式执行，如果当前存在事务，就把当前事务挂起         |
| PROPAGATION_NEVER         | 以非事务方式执行，如果当前存在事务，则抛出异常               |
| PROPAGATION_NESTED        | 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与PROPAGATION_REQUIRED类似的操作。 |



# 7 Spring AOP源码深度剖析















