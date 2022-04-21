> 第八部分 Mybatis插件

# 1 插件简介

一般情况下，开源框架都会提供插件或其他形式的拓展点，供开发者自行拓展。这样的好处是显而易见的。一是增加了框架的灵活性；二是开发者可以结合实际需求，对框架进行拓展，使其能够更好的工作。以 Mybatis 为例，我们可基于 Mybatis 插件实现分页、分表，监控等功能。由于插件和业务无关，业务也无法感知插件的存在。因此可以无感植入插件，在无形中增强功能。



# 2 Mybatis 插件介绍

在四大组件（Executor、StatementHandler、ParameterHandler、ResultSetHandler）处提供了简单易用的插件扩展机制。Mybatis 对持久层的操作就是借助于四大核心对象。Mybatis 支持用插件对四大核心对象进行拦截，对 mybatis 来说插件就是拦截器，用来增强核心对象的功能，增强功能本质上是借助于底层的**动态代理**实现的，换句话说，Mybatis 中的四大对象都是代理对象。

![image-20220421155322887](assest/image-20220421155322887.png)

**Mybatis所允许拦截的方法如下**：

- 执行器 Executor（update、query、commit、rollback 等方法）；
- SQL 语法构建器 StatementHandler （prepare、parameterize、batch、update、query 等方法）；
- 参数处理器 ParameterHandler （getParameterObject、setParameters 方法）；
- 结果集处理器 ResultSetHandler（handleResultSets、handleOutputParameters 等方法）；

# 3 Mybatis 插件原理

在四大对象创建的时候：

1. 每个创建出来的对象不是直接返回的，而是 InterceptorChain.pluginAll(ParameterHandler);
2. 获取到所有的 Interceptor （拦截器）（插件需要实现的接口）；调用 interceptor.plugin(target); 返回 target 包装后的对象
3. 插件机制，我们可以使用插件为目标对象创建一个代理对象；AOP（面向切面）我们的插件可以为四大对象创建出代理对象，代理对象就可以拦截到四大对象的每一个执行；

**拦截**

插件具体是如何拦截并附加额外的功能的呢？以 ParameterHandler 来说

```java

```



# 4 自定义插件

# 5 源码分析

# 6 pageHelper 分页插件

# 7 通过 mapper

