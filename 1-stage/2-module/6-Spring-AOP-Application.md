第六部分 Spring AOP 应用

> AOP本质：在不改变原有业务逻辑的情况下增强横切逻辑，横切逻辑代码往往是权限检验，日子代码，事务控制代码，性能监控代码。

# 1 AOP 相关术语

## 1.1 业务主线

在讲解AOP之前，我们先看一下下面这两张图，它们就是该部分案例需求的扩展（针对这些扩展的需求，我们只进行分析，在此基础上）

![image-20220402181130552](assest/image-20220402181130552.png)

上图描述的就是未采用 AOP 思想设计的程序，当我们红色框中圈定的方法时，会带来大量的重复劳动。程序中充斥着大量的重复代码，使我们程序的独立性很差。而下图中是采用了AOP思想设计的程序，它把红框部分的代码抽取出来的同时，运用动态代理技术，在运行期对需要使用的业务逻辑方法进行增强。

![image-20220402183954632](assest/image-20220402183954632.png)

## 1.2 AOP 术语

| 名称              | 解释                                                         |
| ----------------- | ------------------------------------------------------------ |
| JoinPoint(连接点) | 指的是那些**可以**用于把增强代码加入到业务主线中的点，<br>那么由上图中我们可以看出，这些点指的就是方法。<br>在方法执行的前后通过动态代理技术加入增强的代码。<br>在Spring框架AOP思想的技术实现中，也只支持方法类型的连接点。 |
| Pointcut(切入点)  | 指的是那些**已经**把增强代码写入到业务主线进来之后的连接点。<br>由上图中，我们看出表现层 `register` 方法只是连接点，<br>因为判断访问权限的功能并没有对其增强。 |
| Advice(通知/增强) | 指的是切面类中用于提供增强功能的方法。并且不同的方法增强的时机是不一样的。<br>比如，开启事务肯定要在业务方法执行之前执行；<br>提交事务要在业务方法正常执行之后执行，而回滚事务要在业务方法执行产生异常之后执行等等。<br>那么这些就是通知的类型。其分类有：**前置通知 后置通知 异常通知 最终通知 环绕通知** |
| Target(目标)      | 指的是代理的目标对象。即被代理对象                           |
| Proxy(代理)       | 指的是一个类被 AOP 织入增强后，产生的代理类。即代理对象      |
| Weaving(织入)     | 指的是把增强应用到目标对象来创建新的代理对象的过程。<br>Spring 采用动态代理织入，而 AspectJ 采用编译期织入和类装载期织入 |
| Aspect(切面)      | 指的是增强的代码所关注的方面，把这些相关的增强代码定义到一个类中，这个类就是切面类。<br>例如，事务切面。它里面定义的方法就是和事务相关的，像开启事务，提交事务，回滚事务等等，<br>不会定义其他与事务无关的方法。前面案例中 `TransactionManager` 就是一个切面。 |

1. 连接点：方法开始时，结束时，正常运行完毕时、方法异常时等这些特殊的时机点，我们称之为连接点，项目中每个方法都有连接点，连接点是一种**候选点**。

2. 切入点：指定 AOP 思想要影响的具体方法是哪些，描述感兴趣的方法

3. Advice增强：

   第一个层次：指的是横切逻辑

   第二个层次：方位点（在某一些连接点上加入横切逻辑，那么这些连接点就叫做方位点，描述的是具体的特殊时机）

   

   Aspect 切面：切面是对上述概念的一个综合

   Aspect切面 = 切入点 + 增强

   ​					 = 切入点（锁定方法）+ 方位点（锁定方法中的特殊时机）+ 横切逻辑 

# 2 Spring 中AOP的代理选择

Spring 实现 AOP 思想使用的是动态代理技术

默认情况下，Spring 会根据被代理对象是否实现接口来选择使用 jdk 还是 cglib。当被代理对象没有实现任何接口时，Spring 会选择 cglib；当被代理对象实现了接口，Spring 会选择 jdk 官方的代理技术。不过可以通过配置的方式，让 Spring 强制使用 cglib。

# 3 Spring中AOP的配置方式

在 Spring 的 AOP 配置中，也和 IoC 配置一样，支持三类配置方式：

1. 使用 xml 配置
2. 使用 xml + 注解组合配置
3. 使用纯注解配置

# 4 Spring中AOP实现

需求：横切逻辑代码是打印日志，希望把打印日志的逻辑织入到目标方法的特定位置（service 层 transfer 方法）。

## 4.1 xml 模式

（复制 turbo-transfer-iocxml-anno 到 turbo-transfer-aopxml）

代码地址：https://gitee.com/turboYuu/spring-1-2/tree/master/lab/turbo-transfer-aopxml

Spring 是模块化开发的框架，使用 aop 就引入 aop 的 jar

- pom.xml

  ```xml
  <!--spring aop 的 jar 包支持-->
  <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-aop</artifactId>
      <version>5.1.12.RELEASE</version>
  </dependency>
  
  <!--第三方的 aop 框架 aspectjweaver 的jar-->
  <dependency>
      <groupId>org.aspectj</groupId>
      <artifactId>aspectjweaver</artifactId>
      <version>1.8.13</version>
  </dependency>
  ```

- AOP 核心配置

  ```xml
  <?xml version="1.0" encoding="UTF-8"?>
  <beans xmlns="http://www.springframework.org/schema/beans"
         xmlns:context="http://www.springframework.org/schema/context"
         xmlns:aop="http://www.springframework.org/schema/aop"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="
          http://www.springframework.org/schema/beans
          https://www.springframework.org/schema/beans/spring-beans.xsd
          http://www.springframework.org/schema/context
          https://www.springframework.org/schema/context/spring-context.xsd
          http://www.springframework.org/schema/aop
          https://www.springframework.org/schema/aop/spring-aop.xsd
  
  ">
  
      <!--.....-->
      
      <!--进行 aop 相关的 xml 配置, 配置aop的过程就是把相关术语落地-->
      <!--横切逻辑 bean-->
      <bean id="logUtils" class="com.turbo.edu.utils.LogUtils"></bean>
      <!--使用 config 标签表明aop配置，在其内部配置切面-->
      <aop:config>
          <!--aspect 切面 = 切入点（锁定方法）+ 方位点（锁定方法中的特殊时机）+ 横切逻辑 -->
          <aop:aspect id="logAspect" ref="logUtils">
              <!--切入点锁定我们感兴趣的方法，使用aspectj语法表达式-->
              <aop:pointcut id="pt1" expression="execution(public void com.turbo.edu.service.impl.TransferServiceImpl.transfer(java.lang.String, java.lang.String, int))"/>
              
              <!--方位信息 pointcut-ref 关联切入点-->
              <aop:before method="beforeMethod" pointcut-ref="pt1" />
          </aop:aspect>
      </aop:config>
  	
  </beans>
  ```

### 4.1.1 关于切入点表达式

- 概念及作用

  切入点表达式，也称之为 AspectJ 切入点表达式，**指的是遵循特定语法结构的字符串**，**其作用是用于对符合语法格式的连接点进行增强**。它是 AspectJ 表达式的一部分。

- 关于 AspectJ

  AspectJ 是一个基于 Java 语言的 AOP 框架，Spring 框架从 2.0 版本之后集成了 AspectJ 框架中切入点表达式的部分，开始支持 AspectJ 切入点表达式。

- 切入点表达式使用示例

  ```xml
  全限定方法名：访问修饰符  返回值  包名.包名.包名.包名.类型.方法名(参数列表)
  
  全匹配方式：
  public void com.turbo.edu.service.impl.TransferServiceImpl.transfer(java.lang.String, java.lang.String, int)
  
  访问修饰符可以省略：
  void com.turbo.edu.service.impl.TransferServiceImpl.transfer(java.lang.String, java.lang.String, int)
  
  返回值可以使用 *，标识任意返回值：
  * com.turbo.edu.service.impl.TransferServiceImpl.transfer(java.lang.String, java.lang.String, int)
  
  包名可以使用.表示任意包，但是有几级包，必须写几个
  * .....TransferServiceImpl.transfer(java.lang.String, java.lang.String, int)
  
  包名可以使用..表示当前包及其子包
  * ..TransferServiceImpl.transfer(java.lang.String, java.lang.String, int)
  
  类名和方法名，都可以使用.表示任意类，任意方法
  * ...(java.lang.String, java.lang.String, int)
  
  参数列表，可以使用具体类型
  基本类型直接写类型名称：int
  引用类型必须写全限定类名：java.lang.String
  参数列表可以使用*，表示任意参数类型，但是必须有参数 * *..*.*(*)
  参数列表可以使用..，表示有无参数均可。有参数可以是任意类型 * *..*.*(..)
  全统配方式 * *..*.*(..)
  ```

  

### 4.1.2 改变代理方式的配置

Spring 在创建代理对象时，会根据被代理对象的实际情况来选择代理方式。被代理对象实现了接口，则采用基于接口的动态代理。当被代理对象没有实现任何接口的时候，Spring会自动切换到基于子类的动态代理方式。

但是我们知道，无论被代理对象是否实现接口，只要不是final修饰的类都可以采用cglib提供的方式创建代理对象。所以 Spring 也考虑到这个情况，提供了配置的方式实现强制使用基于子类的动态代理（即cglib），配置的方式有两种：

- 使用 aop:config 标签配置

  ```xml
  <aop:config proxy-target-class="true">
  ```

- 使用 aop:aspectj-autoproxy 标签配置

  ```xml
  <!--此标签注解是基于XML和注解组合配置 AOP 时的必备标签，表示Spring开启注解配置AOP的支持-->
  <aop:aspectj-autoproxy proxy-target-class="true"></aop:aspectj-autoproxy>
  ```

  

### 4.1.3 五种通知类型

#### 4.1.3.1 前置通知

**配置方式**：aop:before 标签

```xml
<!--
	作用：前置通知
	出现位置：只能出现在 aop:aspect 标签内部
	属性：
		method：用于指定前置通知的方法名称
		pointcut：用于指定切面表达式
		pointcut-ref：用于指定切入点表达式的引用
	
-->
<aop:before method="beforeMethod" pointcut-ref="pt1" />
```

**执行时机**：前置通知永远都会在切入点方法（业务核心方法）执行之前执行。

**细节**：前置通知可以获取切入点方法的参数，并对其进行增强。

```java
public void beforeMethod(JoinPoint joinPoint){
    // 获取切入点的参数
    final Object[] args = joinPoint.getArgs();
    // ...
}
```



#### 4.1.3.2 后置通知

**配置方式**：aop:after-returning

```xml
<!--后置通知-->
<aop:after-returning method="successMethod" pointcut-ref="pt1"/>
```



#### 4.1.3.3 异常通知

#### 4.1.3.4 最终通知

#### 4.1.3.5 环绕通知

## 4.2 xml + 注解模式

## 4.3 注解模式

# 5 Spring 声明式事务支持

## 5.1 事务回顾

### 5.1.1 事务的概念

### 5.1.2 事务的四大特性

### 5.1.3 事务的隔离级别

### 5.1.4 事务的传播级别

## 5.2 Spring 中事务的 API

## 5.3 Spring 声明式事务配置