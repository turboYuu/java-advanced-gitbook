第四部分 Spring IOC应用

# 1 Spring IoC基础

![image-20220327120546407](assest/image-20220327120546407.png)

## 1.1 BeanFactory 和 ApplicationContext 区别

BeanFactory 是 Spring 框架中 IoC 容器的顶层接口，它只是用来定义一些基础功能，定义一些基础规范，而ApplicationContext 是它的一个子接口，所以 ApplicationContext 是具备 BeanFactory 提供的全部功能的。

通常，我们称 BeanFactory 为 SpringIOC 的基础容器，ApplicationContext 是容器的高级接口，比 BeanFactory 要拥有更多的功能，比如说国际化支持和资源访问（xml，java配置类）等等。



启动 IoC 容器的方式：

- Java 环境下启动 IoC 容器

  - ClassPathXmlApplicationContext：从类的根路径下加载配置文件（推荐使用）
  - FileSystemXmlApplicationContext：从磁盘路径上加载配置文件
  - AnnotationConfigApplicationContext：纯注解模式下启动Spring容器

- Web环境下启动 IoC 容器

  - 从 xml 启动容器

    ```xml
    
    ```

  - 从配置类启动容器

    ```xml
    
    ```

    

## 1.2 纯 xml 模式

本部分内容不采用一一讲解知识点的方式，而是采用 Spring IoC 纯 xml 模式改造前面手写的 IoC 和 AOP 实现，在改造的过程中，把各个知识点串起来。

### 1.2.1 xml 文件头

### 12.2 实例化 Bean 的 三种方式

- 方式一：使用无参构造函数

  在默认情况下，它会通过反射调用无参构造函数来创建对象。如果类中没有无参构造函数，将创建失败

  ```xml
  
  ```

- 方式二：使用静态方法创建

  在实际开发中，我们使用的对象有些时候不是直接通过构造函数就可以创建出来的，他可能在创建的过程中会做很多额外的操作。此时会提供一个创建对象的方法，恰好这个方法是 `static` 修饰的方法。

  例如：我们在做Jdbc 操作时，会用到 java.sql.Connection 接口的实现类，如果是 mysql 数据库，那么用的就是 JDBC4Connection，但是我们不会去写 `JDBC4Connection connection = new JDBC4Connection()`，因为我们要注册驱动，还要提供 URL 和凭证信息，用 `DriverManager.getConnection` 方法来获取链接。

  那么在实际开发中，尤其早期的项目中没有使用Spring框架来管理对象的创建，但是在设计时使用了工厂模式解耦，那么当接入 Spring 之后，工厂类创建对象就具有和上述例子相同特征，即可采用此种方式配置。

  ```xml
  
  ```

- 方式三：使用实例化方法创建

  此种方式和上面静态方法创建其实类似，区别是用于获取对象的方法不再是 static 修饰的了，而是类中的一个普通方法。此种方式比静态方法创建的使用几率要高一些。

  在早期开发的项目中，工厂类中的方法有可能是静态的，也有可能是非静态方法，当是非静态方法时，即可采用下面的配置方式：

  ```xml
  
  ```

### 1.2.3 Bean的作用范围及生命周期

- 作用范围的改变

  在Spring 框架管理 Bean 对象的创建时，Bean 对象默认都是单例的，但是它支持配置的方式改变作用范围。[作用范围官方提供的说明](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-factory-scopes)如下：

  ![image-20220328133401361](assest/image-20220328133401361.png)

  在上图中提供的这些选项中，我们实际开发中用到做多的作用范围就是 `singleton`（单例模式）和 `prototype` （原型模式，也叫多例模式）。配置方式参考下面的代码：

  ```xml
  
  ```

- 不同作用范围的生命周期

  **单例模式：singleton**

  对象出生：当创建容器时，对象就被创建。

  对象活着：只要容器在，对象一直活着

  对象死亡：当销毁容器时，对象就被销毁了。

  一句话总结：单例模式的 bean 对象生命周期与容器相同。

  **多例模式：prototype**

  对象出生：当使用对象时，创建新的对象实例。

  对象活着：只要对象在使用中，就一直活着

  对象死亡：当对象长时间不用时，被 java 的垃圾回收器回收

  一句话总结：多例模式的 bean 对象，Spring 框架只负责创建，不负责销毁。

- Bean 标签属性

  在基于xml的 IoC 配置中，bean 标签是最基础的标签。它表示了 IoC 容器中的一个对象。换句话说，如果一个对象想让 Spring 管理，在 xml 

## 1.3 xml 与注解相结合模式

## 1.4 纯注解模式

# 2 Spring IoC 高级特性

## 2.1 lazy-Init 延迟加载

## 2.2 FactoryBean 和 BeanFactory

## 2.3 后置处理器

### 2.3.1 BeanPostProcessor

### 2.3.2 BeanFactoryPostProcessor