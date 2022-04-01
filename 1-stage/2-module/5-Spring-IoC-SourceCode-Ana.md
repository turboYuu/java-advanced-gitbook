第五部分 Spring IOC 源码深度剖析

# 1 Spring IoC 容器初始化主体流程

## 1.1 Spring IoC 的容器体系

IoC容器是 Spring 的核心模块，是抽象了对象管理，依赖关系管理的框架解决方案。Spring 提供了很多的容器，其中 BeanFactory 是顶层容器（根容器），不能被实例化，它定义了所有 IoC 容器必须遵从的一套原则，具体的容器实现可以增加额外的功能，比如我们常用到 ApplicationContext，其下更具体的实现如 ClassPathXmlApplicationContext 包含了解析 xml 等一系列的内容，AnnotationConfigApplicationContext 则是包含了注解解析等一系列的内容。Spring IoC 容器继承体系非常聪明，需要使用哪个层次就用哪个层次即可，不必使用功能大而全的。

BeanFactory 顶级接口方法栈如下：

![image-20220401105418489](assest/image-20220401105418489.png)

BeanFactory 容器继承体系：

![image-20220401134522833](assest/image-20220401134522833.png)

![image-20220401141956973](assest/image-20220401141956973.png)

通过其接口设置，我们可以看到我们一贯使用的 ApplicationContext 除了继承 BeanFactory 的子接口，还继承了 ResourceLoader、MessageSource 等接口，因此其提供的功能也就更丰富了。

下面以 ClassPathXmlApplicationContext 为例，深入源码说明 IoC 容器的初始化流程。

## 1.2 Bean 证明周期关键时机点

**思路**：创建一个类 TurboBean，让其实现几个特殊的接口，并分别在接口的实现构造器、接口方法中断点，观察线程调用栈，分析出 Bean 对象创建和管理关键点的触发时机。

TurboBean 类：

```java
package com.turbo;

import org.springframework.beans.factory.InitializingBean;

public class TurboBean implements InitializingBean {

	public TurboBean() {
		System.out.println("TurboBean 构造函数");
	}

	/**
	 * InitializingBean 接口实现
	 * @throws Exception
	 */
	@Override
	public void afterPropertiesSet() throws Exception {
		System.out.println("TurboBean afterPropertiesSet .....");
	}
}
```

BeanPostProcessor 接口实现类：

```java
package com.turbo;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;

public class MyBeanPostProcessor implements BeanPostProcessor {

	public MyBeanPostProcessor() {
		System.out.println("BeanPostProcessor 实现类构造函数...");
	}

	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if("turboBean".equalsIgnoreCase(beanName)){
			System.out.println("BeanPostProcessor 实现类 postProcessBeforeInitialization 方法调用中...");
		}
		return bean;
	}

	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if("turboBean".equalsIgnoreCase(beanName)){
			System.out.println("BeanPostProcessor 实现类 postProcessBeforeInitialization 方法调用中...");
		}
		return bean;
	}
}
```

BeanFactoryPostProcessor 接口实现类

```java
package com.turbo;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanFactoryPostProcessor;
import org.springframework.beans.factory.config.ConfigurableListableBeanFactory;

/**
 * BeanFactory级别的处理，是针对整个Bean的⼯⼚进⾏处理，典型应⽤:PropertyPlaceholderConﬁgurer
 */
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

	public MyBeanFactoryPostProcessor() {
		System.out.println("BeanFactoryPostProcessor的实现类构造函数...");
	}

	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) 
        throws BeansException {
		System.out.println("BeanFactoryPostProcessor 的实现方法调用中....");
	}
}
```

applicationContext.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	   xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

	<bean id="turboBean" class="com.turbo.TurboBean"></bean>
	<bean id="myBeanPostProcessor" class="com.turbo.MyBeanPostProcessor"></bean>
	<bean id="myBeanFactoryPostProcessor" class="com.turbo.MyBeanFactoryPostProcessor"></bean>
</beans>
```

IoC 容器源码分析用例：

```java
import org.junit.Test;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class IoCTest {

	@Test
	public void testIoC(){
		// ApplicationContext 是容器的高级接口，BeanFactory (顶级容器/根容器，规范了/定义了容器的基础行为)
		// Spring应用上下文，官方称之为 IoC 容器（错误认识：容器就是 map 而已；）准确来说 map 是ioc容器的一个成员 叫做单例池 即是 singletonObjects
		// 容器是一组组件和过程的集合，包括 BeanFactory,单例池，BeanPostProcessor 等以及之间的协作
		ApplicationContext applicationContext =
				new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
		Object turboBean = applicationContext.getBean("turboBean");
		System.out.println(turboBean);
	}
}
```

### 1.2.1 分析 Bean 的创建是在容器初始化时还是在 getBean 时

![image-20220401140524602](assest/image-20220401140524602.png)

根据断点调试，发现在未设置延迟加载的前提下，Bean 的创建是在容器初始化过程中完成的。

### 1.2.2 分析构造函数调用情况

![image-20220401141103084](assest/image-20220401141103084.png)

观察调用栈：

![image-20220401141248411](assest/image-20220401141248411.png)

通过如上观察，发现构造函数的调用时机在 `org.springframework.context.support.AbstractApplicationContext#refresh` 方法的 `finishBeanFactoryInitialization(beanFactory);` 处。

### 1.2.3 分析 InitializingBean 之 afterPropertiesSet 初始化方法调用情况

![image-20220401141541223](assest/image-20220401141541223.png)

观察调用栈：

![image-20220401141719323](assest/image-20220401141719323.png)

通过如上观察，发现 InitializingBean 中的 afterPropertiesSet 方法的调用时机也是在  `org.springframework.context.support.AbstractApplicationContext#refresh` 方法的 `finishBeanFactoryInitialization(beanFactory);` 处。

### 1.2.4 分析 BeanFactoryPostProcessor 初始化和调用情况

分别在 构造函数、postProcessBeanFactory 方法处打断点，观察调用栈，发现

**BeanFactoryPostProcessor 初始化**在 AbstractApplicationContext 类的 refresh 方法的 `invokeBeanFactoryPostProcessors(beanFactory);` 处：

![image-20220401142535427](assest/image-20220401142535427.png)

**postProcessBeanFactory** 调用在  AbstractApplicationContext 类的 refresh 方法的 `invokeBeanFactoryPostProcessors(beanFactory);` 处：

![image-20220401142714684](assest/image-20220401142714684.png)

### 1.2.5 分析 BeanPostProcessor 初始化和调用情况

分别在构造函数、postProcessBeforeInitialization、postProcessAfterInitialization 方法处打断点，观察调用栈，发现：

**BeanPostProcessor 初始化在** AbstractApplicationContext 类 refresh 方法的 `registerBeanPostProcessors(beanFactory);`处：

![image-20220401143437490](assest/image-20220401143437490.png)

**postProcessBeforeInitialization 调用在**  AbstractApplicationContext 类 refresh 方法的 `finishBeanFactoryInitialization(beanFactory);` 处：

![image-20220401143608362](assest/image-20220401143608362.png)

**postProcessAfterInitialization 调用在** AbstractApplicationContext 类 refresh 方法的 `finishBeanFactoryInitialization(beanFactory);` 处：

![image-20220401143719609](assest/image-20220401143719609.png)

### 1.2.6 总结

根据上面的调试分析，发现 Bean 对象创建的几个关键时机点代码层级的调用都在 AbstractApplicationContext 类 refresh 方法中，可见这个方法对于 Spring Ioc 容器初始化来说非常关键，汇总如下：

| 关键点                                                   | 触发代码                                             |
| -------------------------------------------------------- | ---------------------------------------------------- |
| Bean 构造器                                              | refresh#finishBeanFactoryInitialization(beanFactory) |
| BeanFactoryPostProcessor 初始化                          | refresh#invokeBeanFactoryPostProcessors(beanFactory) |
| BeanFactoryPostProcessor#postProcessBeanFactory 方法调用 | refresh#invokeBeanFactoryPostProcessors(beanFactory) |
| BeanPostProcessor 初始化                                 | refresh#registerBeanPostProcessors(beanFactory)      |
| BeanPostProcessor中的前后初始化 方法调用                 | refresh#finishBeanFactoryInitialization(beanFactory) |



## 1.3 Spring IoC 容器初始化主流程

# 2 BeanFactory 创建流程

## 2.1 获取 BeanFactory 子流程

## 2.2 BeanDefinition 加载解析及注册子流程

# 3 Bean 创建流程

# 4 lazy-init 延迟加载机制原理

# 5 Spring IoC 循环依赖问题

## 5.1 什么是循环依赖

## 5.2 循环依赖处理机制