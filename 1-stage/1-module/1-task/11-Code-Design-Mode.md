> 第十一部分 设计模式

虽然我们都知道有 3 类 23 种设计模式，但是大多停留在概念层面，Mybatis 源码种使用了大量的设计模式，观察设计模式在其中的应用，能够更深入的理解设计模式。

Mybatis至少用到了以下的设计模式：

| 模式         | Mybatis 体现                                                 |
| ------------ | ------------------------------------------------------------ |
| Builder模式  | 例如 SqlSessionFactoryBuilder、Environment                   |
| 工厂方法模式 | 例如 SqlSessionFactory，TransactionFactory，LogFactory       |
| 单例模式     | 例如 ErrorContext 和 LogFactory                              |
| 代理模式     | Mybatis实现的核心，比如 MapperProxy、ConnectionLogger，<br>用的 jdk 的动态代理还有 executor.loader 包 使用了 cglib <br>或者 javassist 达到延迟加载的效果 |
| 组合模式     | 例如 SqlNode 和 各个子类 ChooseSqlNode 等                    |
| 模板方法模式 | 例如 BaseExecutor 和 SimpleExecutor，还有 BaseTypeHandler <br>和 所有的子类例如 IntegerTypeHandler |
| 适配器模式   | 例如 Log 的 Mybatis 接口和它对 jdbc、log4j 等各种日志框架的适配实现 |
| 装饰者模式   | 例如 Cache 包中的 cache.decorators 子包中的各种装饰者的实现  |
| 迭代器模式   | 例如迭代器模式 PropertyTokenizer                             |

接下来对 Builder 构建者模式、工厂模式、代理模式 进行解读，先介绍模式自身的知识，然后解读在 Mybatis 中怎样应用了该模式。

# 1 Builder 构建者模式

Builder 模式的定义是 **将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示**。它属于创建类模式，一般来说，如果一个对象的构建比较复杂，超出了构造函数所能包含的范围，就可以使用工厂模式和Builder模式，相对于工厂模式会产生一个完整的产品，Builder应用于更加复杂的对象的构建，甚至只会构建产品的一个部分，直白来说，就是使用多个简单的对象一步一步构建成一个复杂的对象



# 2 工厂模式

# 3 代理模式