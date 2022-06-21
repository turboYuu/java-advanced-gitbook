> 第四部分 Spring Data JPA 执行过程源码分析

Spring Data JPA 的源码很少有人去分析，原因如下：

1. Spring Data JPA 地位没有之前学习的框架高，大家习惯把它当成一个工具来用，不愿意对它进行源码层次的解读。
2. 开发Dao接口，接口的实现对象肯定是通过动态代理来完成的（增强），代理对象的产生过程在源码中很难跟踪，特别讲究技巧。

**源码剖析的主要过程，就是代理对象产生的过程**

发现 resumeDao 是一个代理对象，这个代理对象的类型是 SimpleJpaRepository

![image-20220617182315688](assest/image-20220617182315688.png)

# 1 疑问：这个代理对象是怎么产生的，过程是？

以往：如果要给一个对象产生代理对象，我们知道是在 AbstractApplicationContext 的 refresh 方法中，那么能不能在这个方法中找到什么当前场景的线索？

![image-20220617185523456](assest/image-20220617185523456.png)

![image-20220617185719895](assest/image-20220617185719895.png)

![image-20220617190938527](assest/image-20220617190938527.png)

新的问题又来了？

**问题1**：为什么会给它指定为一个 JpaRepositoryFactoryBean （getObject方法返回具体的对象）？

**问题2**：指定这个 FactoryBean 是在什么时候发生的？

首先解决问题2 ：

![image-20220618205037486](assest/image-20220618205037486.png)



![image-20220618205207029](assest/image-20220618205207029.png)

传入一个 resumeDao 就返回一个已经指定class为 JpaRepositoryFactoryBean  的 BeanDefinition 对象了，那么应该在上图中的 get 的时候就有了，所以断点进入

![image-20220618235841010](assest/image-20220618235841010.png)

问题来了，什么时候put到 `mergedBeanDefinitions` 这个map中去的？定位到了一个方法在做这件事：

*在断点中设置条件*

![image-20220620173803782](assest/image-20220620173803782.png)

![image-20220620174309869](assest/image-20220620174309869.png)

发现，传入该方法的时候，BeanDefinition 中的class就已经被指定为FactoryBean了，那么观察该方法的调用栈：

![image-20220620174916803](assest/image-20220620174916803.png)

 断点进入：

![image-20220620104626072](assest/image-20220620104626072.png)

在该类中 搜索 `beanDefinitionMap.put`：

![beanDefinitionMap.put/Ctrl+F](assest/image-20220620111128397.png)

观察调用栈：

![image-20220620180619314](assest/image-20220620180619314.png)

断点进入：

![image-20220620181041698](assest/image-20220620181041698.png)

断点跟踪：

![image-20220620181948130](assest/image-20220620181948130.png)

![image-20220620182401142](assest/image-20220620182401142.png)

通过上述追踪发现，<jpa:repositories base-package ，扫描到的接口，在进行BeanDefinition注册时候，class会被固定的指定为 JpaRepositoryFactoryBean。

至此问题2追踪完毕。

**那么接下来，再来追踪问题1 ，JpaRepositoryFactoryBean 是一个什么样的类？**

它是一个 FactoryBean，重点关注FactoryBean的getObject方法

![image-20220620183408902](assest/image-20220620183408902.png)

搜一下 `this.repository` 什么时候被赋值？在 afterPropertiesSet() 方法中。



其父类实现了 `InitializingBean` 接口，Spring 将调用 afterPropertiesSet() 方法。 

![image-20220620183849600](assest/image-20220620183849600.png)

![image-20220620184405078](assest/image-20220620184405078.png)



![image-20220620190322908](assest/image-20220620190322908.png)

![image-20220620191245657](assest/image-20220620191245657.png)

![image-20220620191429692](assest/image-20220620191429692.png)

![image-20220620191640585](assest/image-20220620191640585.png)

![image-20220620192318948](assest/image-20220620192318948.png)

![image-20220620192651205](assest/image-20220620192651205.png)

![image-20220620193807926](assest/image-20220620193807926.png)

由此可见，JdkDynamicAopProxy 会生成一个代理对象类型为 SimpleJpaRespository，而该对象的增强逻辑就在 JdkDynamicAopProxy 类的 invoke 方法中。

至此问题1 追踪完成。

# 2 疑问：这个代理对象 SimpleJpaRepository 有什么特别的？

![image-20220621105026091](assest/image-20220621105026091.png)

![image-20220621105107748](assest/image-20220621105107748.png)

原来 SimpleJpaRepository 类实现了 JpaRepository 接口 和 JpaSpecificationExecutor 接口

![image-20220621105429728](assest/image-20220621105429728.png)

