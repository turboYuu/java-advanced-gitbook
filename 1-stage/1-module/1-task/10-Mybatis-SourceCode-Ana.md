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
        // 根据传入的全限定名 + 方法名 从映射的 Map 中取出 MappedStatement 对象
        MappedStatement ms = configuration.getMappedStatement(statement);
        // 调用 executor 中的方法处理
        // RowBounds 是用来逻辑分页
        // wrapCollection(parameter) 是用来装饰集合或者数组参数
        return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
        throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
        ErrorContext.instance().reset();
    }
}
```



## 1.3 源码剖析-executor

继续源码中的步骤，进入 executor.query()

```java
// 此方法在 SimpleExecutor 的父类 BaseExecutor 中实现
public <E> List<E> query(MappedStatement ms, 
                         Object parameter, 
                         RowBounds rowBounds, 
                         ResultHandler resultHandler) throws SQLException {
    // 根据传入的参数动态获得 SQL 语句，最后返回用 BoundSql 对象表示
    BoundSql boundSql = ms.getBoundSql(parameter);
    // 为本次查询创建缓存的 key
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
}

// 进入 query 的重载方法中
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, 
                         ResultHandler resultHandler, CacheKey key, 
                         BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
        throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
        clearLocalCache();
    }
    List<E> list;
    try {
        queryStack++;
        list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
        if (list != null) {
            handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
        } else {
            // 如果缓存中没有本次查找的值，那么从数据库中查询
            list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
        }
    } finally {
        queryStack--;
    }
    if (queryStack == 0) {
        for (DeferredLoad deferredLoad : deferredLoads) {
            deferredLoad.load();
        }
        // issue #601
        deferredLoads.clear();
        if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
            // issue #482
            clearLocalCache();
        }
    }
    return list;
}


// 从数据库查询
private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, 
                                      RowBounds rowBounds, ResultHandler resultHandler, 
                                      CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
        // 查询的方法
        list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
        localCache.removeObject(key);
    }
    // 将查询结果放入缓存
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
        localOutputParameterCache.putObject(key, parameter);
    }
    return list;
}

// SimpleExecutor 中实现父类的 doQuery 抽象方法
public <E> List<E> doQuery(MappedStatement ms, Object parameter, 
                           RowBounds rowBounds, ResultHandler resultHandler, 
                           BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
        Configuration configuration = ms.getConfiguration();
        // 传入参数创建 StatementHandler 对象来执行查询
        StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
        // 创建 jdbc 中的 statement 对象
        stmt = prepareStatement(handler, ms.getStatementLog());
        // StatementHandler 进行处理
        return handler.<E>query(stmt, resultHandler);
    } finally {
        closeStatement(stmt);
    }
}

// 创建 Statement 的方法
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    // 代码中的 getConnection 方法经过重重调用最后会调用 openConnection 方法，从连接池中获得连接
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
}

// 从连接池过的连接的方法
protected void openConnection() throws SQLException {
    if (log.isDebugEnabled()) {
        log.debug("Opening JDBC Connection");
    }
    // 从连接池获得连接
    connection = dataSource.getConnection();
    if (level != null) {
        connection.setTransactionIsolation(level.getLevel());
    }
    setDesiredAutoCommit(autoCommmit);
}
```

上述的 Executor.query() 方法几经转折，最后会创建一个 StatementHandler 对象，然后将必要的参数传递给 StatementHandler ，使用 StatementHandler 来完成对数据库的查询，最终返回 List 结果集。

从上面的代码中我们可以看出，Executor 的功能和作用是：

```xml
1. 根据传递的参数，完成 SQL 语句的动态解析，生成 BoundSql 对象，供 StatementHandler 使用。
2. 为查询创建缓存，以提高性能。
3. 创建 JDBC 的 Statement 连接对象，传递给 StatementHandler 对象，返回 List 查询结果。
```



## 1.4 源码剖析-StatementHandler

StatementHandler 对象主要完成两个工作：

-  对于 JDBC 的 PreparedStatement 类型的对象，创建的过程中，我们使用的是 SQL 语句字符串会包含若干个 ? 占位符，我们齐后再对占位符进行设值。StatementHandler 通过 parameterize(statement) 方法对 Statement 进行设值
- StatementHandler 通过 List query(Statement statement, ResultHandler resultHandler) 方法来完成 Statement，和将 Statement 对象返回的 resultSet 封装成 List；

进入 StatementHandler 的 parameterize(statement) 方法的实现：

```java
public void parameterize(Statement statement) throws SQLException {
	// 使用 parameterHandler 对象来完成对 statement 的设值
    parameterHandler.setParameters((PreparedStatement) statement);
}
```

```java
// ParameterHandler 类的 setParameters(PreparedStatement ps) 实现
// 对某一个 Statement 进行设置参数
public void setParameters(PreparedStatement ps) {
    ErrorContext.instance()
        .activity("setting parameters")
        .object(mappedStatement.getParameterMap().getId());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    if (parameterMappings != null) {
        for (int i = 0; i < parameterMappings.size(); i++) {
            ParameterMapping parameterMapping = parameterMappings.get(i);
            if (parameterMapping.getMode() != ParameterMode.OUT) {
                Object value;
                String propertyName = parameterMapping.getProperty();
                if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params
                    value = boundSql.getAdditionalParameter(propertyName);
                } else if (parameterObject == null) {
                    value = null;
                } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                    value = parameterObject;
                } else {
                    MetaObject metaObject = configuration.newMetaObject(parameterObject);
                    value = metaObject.getValue(propertyName);
                }
                // 每一个Mapping都有一个 TypeHandler，根据 TypeHandler 来对 prepearedStatemnt 进行设置参数
                TypeHandler typeHandler = parameterMapping.getTypeHandler();
                JdbcType jdbcType = parameterMapping.getJdbcType();
                if (value == null && jdbcType == null) {
                    jdbcType = configuration.getJdbcTypeForNull();
                }
                try {
                    // 设置参数
                    typeHandler.setParameter(ps, i + 1, value, jdbcType);
                } catch (TypeException e) {
                    throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
                } catch (SQLException e) {
                    throw new TypeException("Could not set parameters for mapping: " + parameterMapping + ". Cause: " + e, e);
                }
            }
        }
    }
}
```

从上述代码可以看到，StatementHandler 的 parameterize(Statement) 方法调用了 parameterHandler.setParameters((PreparedStatement) statement) 方法，parameterHandler.setParameters((PreparedStatement) statement) 方法负责根据我们输入的参数，对 statement 对象的 ? 占位符处进行赋值。

进入到 StatementHandler  的 List query(Statement statement, ResultHandler resultHandler) 方法的实现：

```java
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    // 1. 调用 PreparedStatement.execute() 方法，然后将 resultSet 交给 resultSetHandler 处理
    PreparedStatement ps = (PreparedStatement) statement;
    ps.execute();
    // 2. 使用 resultSetHandler 来处理 ResultSet
    return resultSetHandler.<E> handleResultSets(ps);
}
```

从上述代码我们可以看出，StatementHandler  的 List query(Statement statement, ResultHandler resultHandler) 方法的实现，是调用了 ResultSetHandler 的 handleResultSets(Statement) 方法。

ResultSetHandler 的 handleResultSets(Statement) 方法 会将 Statement 语句执行后生成的 resultSet 结果集转换成 List 结果集。

```java
public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());
	// 多 ResultSet 的结果集合，每个 ResultSet 对应一个 Object 对象。而实际上，每个 Object 是 List<Object> 对象
    // 在不考虑存储过程的多ResultSet的情况，普通的查询，实际就是一个ResultSet,也就是说，multipleResults最多就一个元素
    final List<Object> multipleResults = new ArrayList<Object>();

    int resultSetCount = 0;
    // 获得首个 ResultSet 对象，并封装成ResultSetWrapper对象 
    ResultSetWrapper rsw = getFirstResultSet(stmt);
	// 获得 ResultMap数组
    // 在不考虑存储过程的多ResultSet的情况，普通的查询，实际就是一个ResultSet,也就是说，resultMaps最多就一个元素
    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null && resultMapCount > resultSetCount) {
        // 获得 ResultMap对象
        ResultMap resultMap = resultMaps.get(resultSetCount);
        // 处理ResultSet，将结果添加到multipleResults中
        handleResultSet(rsw, resultMap, multipleResults, null);
        // 获得下一个ResultSet对象，并封装成ResultSetWrapper对象 
        rsw = getNextResultSet(stmt);
        // 清理
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
    }
	// 因为 mappedStatement.resultSets 只在存储过程中使用，本系列暂时不考虑，忽略即可
    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
        while (rsw != null && resultSetCount < resultSets.length) {
            ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
            if (parentMapping != null) {
                String nestedResultMapId = parentMapping.getNestedResultMapId();
                ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
                handleResultSet(rsw, resultMap, null, parentMapping);
            }
            rsw = getNextResultSet(stmt);
            cleanUpAfterHandlingResultSet();
            resultSetCount++;
        }
    }
	// 如果是 multipleResults 单元素，则取首元素返回
    return collapseSingleResultList(multipleResults);
}
```



# 2 Mapper代理方式

回顾下写法：

```java
@Test
public void test02() throws IOException {
    // 1. 读取配置文件，读成字节输入流，注意：现在还没有解析
    InputStream resourceAsStream = Resources.getResourceAsStream("SqlMapConfig.xml");
    // 2. 这一行代码时初始化工作的开始
    SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(resourceAsStream);
    SqlSession sqlSession = factory.openSession();
    UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    List<User> all = userMapper.findAllUser();
    for (User user : all) {
        System.out.println(user);
    }
    sqlSession.close();
}
```

思考一个问题，通过的 Mapper 接口我们都没有实现的方法却可以使用，是为什么？答案很简单：动态代理

开始之前介绍一下 Mybatis 初始化时对接口的处理：MapperRegistry 是 Configuration 中的一个属性，它内部维护一个 HashMap 用于存放 mapper 接口的工厂类，每个接口对应一个工厂类。mappers 中可以配置接口的包路径，或者某个具体的接口类。

```xml
<mappers>
    <mapper resource="UserMapper.xml"></mapper>
    <mapper class="com.turbo.mapper.UserMapper"/>
    <package name="com.turbo.mapper"/>
</mappers>
```

当解析 mappers 标签时，它会判断解析到的是 <br>mapper 配置文件时，会再将对应配置文件中的增删改查 标签封装成 MappedStatement 对象，存入 mappedStatements 中；<br>判断解析到接口时，会建立接口对应的 MapperProxyFactory对象，存入 HashMap 中，key = 接口的字节码对象，value = 此接口对应的 MapperProxyFactory 对象。



## 2.1 源码剖析-getMapper

进入 sqlSession.getMapper(UserMapper.class)中

```java
// org.apache.ibatis.session.defaults.DefaultSqlSession#getMapper
public <T> T getMapper(Class<T> type) {
    return configuration.<T>getMapper(type, this);
}

// org.apache.ibatis.session.Configuration#getMapper
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
}

// org.apache.ibatis.binding.MapperRegistry#getMapper
public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    // 从 MapperRegistry 中的HashMap中拿 MapperProxyFactory
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    if (mapperProxyFactory == null) {
        throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
        // 通过动态代理工厂生成示例
        return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
        throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
}

// MapperProxyFactory#newInstance(org.apache.ibatis.session.SqlSession)
public T newInstance(SqlSession sqlSession) {
    // 创建了 jdk动态代理的Handler类
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    // 调用了重载方法
    return newInstance(mapperProxy);
}

// MapperProxy类，实现了InvocationHandler接口
public class MapperProxy<T> implements InvocationHandler, Serializable {

  // ...
  private final SqlSession sqlSession;
  private final Class<T> mapperInterface;
  private final Map<Method, MapperMethod> methodCache;

  // 构造，传入SqlSession，说明每个session中的代理对象的不同
  public MapperProxy(SqlSession sqlSession, Class<T> mapperInterface, 
                     Map<Method, MapperMethod> methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }
  // 省略部分源码
}

```





## 2.2 源码剖析-invoke()

在动态代理返回示例后，我们就可以直接调用 mapper 类中的方法了，但代理对象调用方法，执行的是在 MapperProxy中的invoke方法：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
        // 如果是Object定义的方法，直接调用
        if (Object.class.equals(method.getDeclaringClass())) {
            return method.invoke(this, args);
        } else if (isDefaultMethod(method)) {
            return invokeDefaultMethod(proxy, method, args);
        }
    } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
    }
    // 获得 MapperMethod 对象
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    // 重点在这：MapperMethod最终调用了执行的方法
    return mapperMethod.execute(sqlSession, args);
}
```

进入 execute 方法：

```java
public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    // 判断mapper中的方法类型，最终调用的还是 sqlSession 的方法
    switch (command.getType()) {
        case INSERT: {
            // 转换参数
            Object param = method.convertArgsToSqlCommandParam(args);
            // 执行 insert操作
            // 转换rowCount
            result = rowCountResult(sqlSession.insert(command.getName(), param));
            break;
        }
        case UPDATE: {
            // 转换参数
            Object param = method.convertArgsToSqlCommandParam(args);
            // 转换 rowCount
            result = rowCountResult(sqlSession.update(command.getName(), param));
            break;
        }
        case DELETE: {
            // 转换参数
            Object param = method.convertArgsToSqlCommandParam(args);
            // 转换 rowCount
            result = rowCountResult(sqlSession.delete(command.getName(), param));
            break;
        }
        case SELECT:
            // 无返回，并且有ResultHandler方法参数，则将查询的结果，提交给ResultHandler
            if (method.returnsVoid() && method.hasResultHandler()) {
                executeWithResultHandler(sqlSession, args);
                result = null;
            } else if (method.returnsMany()) { // 执行查询，返回列表
                result = executeForMany(sqlSession, args);
            } else if (method.returnsMap()) { // 执行查询，返回map
                result = executeForMap(sqlSession, args);
            } else if (method.returnsCursor()) { // 执行查询，返回Cursor
                result = executeForCursor(sqlSession, args);
            } else {
                // 执行拆线呢，返回单个对象
                // 转换参数
                Object param = method.convertArgsToSqlCommandParam(args);
                // 查询单条
                result = sqlSession.selectOne(command.getName(), param);
            }
            break;
        case FLUSH:
            result = sqlSession.flushStatements();
            break;
        default:
            throw new BindingException("Unknown execution method for: " + command.getName());
    }
    // 返回结果为 null，并且返回类型为基本类型，则抛出BindingException异常
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
        throw new BindingException("Mapper method '" + command.getName() 
                       + " attempted to return null from a method with a primitive return type (" 
                                   + method.getReturnType() + ").");
    }
    // 返回结果
    return result;
}
```



# 3 二级缓存源码剖析

二级缓存构建在一级缓存之上，在收到查询请求时，Mybatis首先会查询二级缓存，若二级缓存未命中，再去查询一级缓存，一级缓存没有，再查询数据库。

二级缓存 -->  一级缓存 --> 数据库

与一级缓存不同，二级缓存和具体的命名空间绑定，一个Mapper中有一个Cache，相同Mapper中的MappedStatement共用一个Cache，一级缓存则是和SqlSession绑定。



## 3.1 启用二级缓存

分为三步走：

1. 开启全局二级缓存配置

   ```xml
   <!--全局性地开启或关闭所有映射器配置文件中已配置的任何缓存。-->
   <settings>
       <setting name="cacheEnabled" value="true"/>
   </settings>
   ```

2. 在需要使用二级缓存的Mapper配置文件中配置标签

   ```xml
   <cache></cache>
   ```

3. 在具体CURD标签上配置 **useCache=true**

   ```xml
   <!--flushCache	将其设置为 true 后，只要语句被调用，都会导致本地缓存和二级缓存被清空，默认值：false。
       useCache	将其设置为 true 后，将会导致本条语句的结果被二级缓存缓存起来，默认值：对 select 元素为 true。-->
   <select id="selectUserById" parameterType="int" resultType="user" useCache="true" flushCache="fasle">
       select * from user where id=#{id}
   </select>
   ```

   

## 3.2 标签< cache/> 的解析

## 3.3 查询源码分析

## 3.4 为何只有SqlSession提交或关闭之后

## 3.5 二级缓存的刷新

## 3.6 二级缓存的刷新

## 3.7 总结

# 4 延迟加载源码剖析