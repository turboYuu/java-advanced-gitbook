第四部分 Spring IOC应用

# 1 Spring IoC基础

![image-20220327120546407](assest/image-20220327120546407.png)

## 1.1 BeanFactory 和 ApplicationContext 区别

BeanFactory 是 Spring 框架中 IoC 容器的顶层接口，它只是用来定义一些基础功能，定义一些基础规范，而ApplicationContext 是它的一个子接口，所以 ApplicationContext 是具备 BeanFactory 提供的全部功能的。

通常，我们称 BeanFactory 为 SpringIOC 的基础容器，ApplicationContext 是容器的高级接口，比 BeanFactory 要拥有更多的功能，比如说国际化支持和资源访问（xml，java配置类）等等



## 1.2 纯 xml 模式

## 1.3 xml 与注解相结合模式

## 1.4 纯注解模式

# 2 Spring IoC 高级特性

## 2.1 lazy-Init 延迟加载

## 2.2 FactoryBean 和 BeanFactory

## 2.3 后置处理器

### 2.3.1 BeanPostProcessor

### 2.3.2 BeanFactoryPostProcessor