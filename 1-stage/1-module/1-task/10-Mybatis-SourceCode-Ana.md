> 第十部分 Mybatis源码剖析

# 1 传统方式源码分析

## 1.1 源码剖析-初始化

```java
InputStream resourceAsStream = Resources.getResourceAsStream("SqlMapConfig.xml");
// 这一行代码时初始化工作的开始
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(resourceAsStream);
```

进入源码分析：

```java
// 1. 我们最初调用的 build
public SqlSessionFactory build(InputStream inputStream) {
    // 调用了重载方法
    return build(inputStream, null, null);
}

// 2. 调用的重载方法
public SqlSessionFactory build(InputStream inputStream, 
                               String environment, 
                               Properties properties) {
    try {
        // XMLConfigBuilder 是专门解析 mybatis 的配置文件的类
        XMLConfigBuilder parser = 
            new XMLConfigBuilder(inputStream, environment, properties);
        // 这里又调用了一个重载方法。parser.parse() 的返回值是 Configuration 对象
        return build(parser.parse());
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
        ErrorContext.instance().reset();
        try {
            inputStream.close();
        } catch (IOException e) {
            // Intentionally ignore. Prefer previous error.
        }
    }
}
```

Mybatis 在初始化的时候，会将 Mybatis 的配置信息全部加载到内存中，使用 org.apache.ibatis.session.Configuration 实例来维护。

下面进入对配置文件解析部分：

首先对 Configuration 对象进行介绍：

```xml
Configuration 对象的结构和xml配置文件的对象几乎相同。

回顾一下 xml 中的配置标签有哪些：
properties(属性)，settings(设置)，typeAliases(类型名称)，typeHandlers(类型处理器)，objectFactory(对象工厂)，mappers(映射器)等 Configuration 也有对应的对象属性来封装它们

也就是说，初始化配置文件信息的本质就是创建 Configuration 对象，将解析的 xml 数据封装到 Configuration 内部属性中
```



```java
// 解析 xml 成 Configuration 对象
public Configuration parse() {
    // 若已经解析，抛出 BuilderException 异常
    if (parsed) {
        throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    // 标记已解析
    parsed = true;
    // 解析 XML Configuration 节点
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
}

// 解析 xml
private void parseConfiguration(XNode root) {
    try {
        //issue #117 read properties first
        // 解析 <properties/> 标签
        propertiesElement(root.evalNode("properties"));
        // 解析 <settings/> 标签
        Properties settings = settingsAsProperties(root.evalNode("settings"));
        // 加载自定义的 VFS 实现类
        loadCustomVfs(settings);
        // 解析 <typeAliases/> 标签
        typeAliasesElement(root.evalNode("typeAliases"));
        // 解析 <plugins/> 标签
        pluginElement(root.evalNode("plugins"));
        // 解析 <objectFactory/> 标签
        objectFactoryElement(root.evalNode("objectFactory"));
        // 解析 <objectWrapperFactory/> 标签
        objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
        // 解析 <reflectorFactory/> 标签
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
        // 赋值 <settings/> 至 Configuration 属性
        settingsElement(settings);
        // read it after objectFactory and objectWrapperFactory issue #631
        // 解析 <environments/> 标签
        environmentsElement(root.evalNode("environments"));
        // 解析 <databaseIdProvider/> 标签
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        // 解析 <typeHandlers/> 标签
        typeHandlerElement(root.evalNode("typeHandlers"));
        // 解析 <mappers/> 标签
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```



介绍一下 MappedStatement：

作用：MappedStatement 与 Mapper 配置文件中的一个 select/update/insert/delete 节点相对应。

mapper 中配置的标签都被封装到了此对象中，主要用途是描述一条 SQL 语句。

**初始化过程**：回顾刚开始的加载配置文件的过程，会对 mybatis-config.xml 中的各个标签都进行解析，其中有 mappers 标签用来引入 mapper.xml 文件或者配置 mapper 接口的目录。

```xml
<select id="selectUserById" parameterType="int" resultType="user">
    select * from user where id=#{id}
</select>
```

这样一个 select 标签会在初始化配置文件时被解析封装成一个 MappedStatement 对象，然后存储在 Configuration 对象的 mappedStatements属性中，mappedStatements 是一个 HashMap，存储时 key = 全限定类名 + 方法名，value = 对应的 MappedStatement 对象。

在 Configuration 中对应的属性为：

```java
protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection");
```

在 XMLConfigBuilder 中的处理：

```java
private void parseConfiguration(XNode root) {
    try {
        // ...
        // 解析 <mappers/> 标签
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```

到此对 xml 配置文件的解析就结束了，回到步骤2 中调用的重载 build 方法：

```java
// 调用的重载方法
public SqlSessionFactory build(Configuration config) {
    // 创建了 DefaultSqlSessionFactory 对象，传入 Configuration 对象
    return new DefaultSqlSessionFactory(config);
}
```

## 1.2 源码剖析-执行 SQL 流程

先简单介绍 **SqlSession**：

SqlSession 是一个接口，它有两个实现类：DefaultSqlSession（默认）和 SqlSessionManager（弃用，不做介绍）

SqlSession 是 Mybatis 中用于和数据库交互的顶层类，同城将它与 ThreadLocal 绑定，一个会话使用一个 SqlSession ，并且在使用完毕后需要 close。

```java
public class DefaultSqlSession implements SqlSession {

  private final Configuration configuration;
  private final Executor executor;
}
```

SqlSession 中的两个重要参数，configuration 与初始化时的相同，executor 为执行器。

### 1.2.1 Executor

Executor 也是一个接口，它有三个常用的实现类：

- BaseExecutor （重用语句并执行批量更新）
- ReuseExecutor（重用预处理语句 prepared statements）
- SimpleExecutor（普通的执行器，默认）

继续分析，初始化完毕后，我们就需要执行 SQL 了

```java
SqlSession sqlSession = factory.openSession();
List<User> list = sqlSession.selectList("com.turbo.mapper.UserMapper.findAllUser");
```

获得 sqlSession 

```java
// 进入 openSession 方法
public SqlSession openSession() {
    // getDefaultExecutorType() 传递的是 SimpleExecutor
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
}

// 进入 openSessionFromDataSource
// ExecutorType 为 Executor 的类型，TransactionIsolationLevel 为事务隔离级别，autoCommit 是否开启事务
// openSession 的入多个重载方法可以指定获得的 SqlSession 的 Executor 类型和事务的处理
private SqlSession openSessionFromDataSource(ExecutorType execType, 
                                             TransactionIsolationLevel level, 
                                             boolean autoCommit) {
    Transaction tx = null;
    try {
        final Environment environment = configuration.getEnvironment();
        final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
        tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
        // 根据参数创建指定类型的 Executor
        final Executor executor = configuration.newExecutor(tx, execType);
        // 返回的是 DefaultSqlSession
        return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
        closeTransaction(tx); // may have fetched a connection so lets call close()
        throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```

执行 sqlSession 中的 api

```java
// 进入 selectList 方法，多个重载方法
public <E> List<E> selectList(String statement) {
    return this.selectList(statement, null);
}

public <E> List<E> selectList(String statement, Object parameter) {
    return this.selectList(statement, parameter, RowBounds.DEFAULT);
}

public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
        // 
        MappedStatement ms = configuration.getMappedStatement(statement);
        return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```



## 1.3 源码剖析-executor



## 1.4 源码剖析-StatementHandler

# 2 Mapper代理方式

# 3 二级缓存源码剖析

# 4 延迟加载源码剖析