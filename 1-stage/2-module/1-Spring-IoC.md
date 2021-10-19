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

# 7 Spring AOP源码深度剖析