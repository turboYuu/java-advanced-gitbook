> 第七部分 Mybatis缓存

在 [mybatis-multitable](https://gitee.com/turboYuu/mybatis-1-1/tree/master/lab-mybatis/mybatis-multitable) 工程中操作

# 1 一级缓存

> 1 在一个 sqlSession 中，对 user 表根据 id 进行两次查询，查看它们发出 sql 语句的情况

```java
@Test
public void test0() throws IOException {
    InputStream resourceAsStream = Resources.getResourceAsStream("SqlMapConfig.xml");
    SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder()
        .build(resourceAsStream);
    SqlSession sqlSession = sqlSessionFactory.openSession();
    // 获得 Mybatis 框架生成的 UserMapper 接口的实现类
    UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
    // 第一次查询，发出sql语句，并将查询出来的结果放进缓存中
    User user = userMapper.selectUserById(1);
    System.out.println(user);
    // 第二次查询，由于是同一个 sqlSession，会在缓存中查询结果
    // 如果有，则直接从缓存中取出来，不和数据库进行交互
    User user1 = userMapper.selectUserById(1);
    System.out.println(user1);
    sqlSession.close();
}
```

![image-20220420185145427](assest/image-20220420185145427.png)

> 2 同样是对user表进行两次查询，只不过两次查询之间进行了一次 update 操作

```java
@Test
    public void test1() throws IOException {
        InputStream resourceAsStream = Resources.getResourceAsStream("SqlMapConfig.xml");
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(resourceAsStream);
        SqlSession sqlSession = sqlSessionFactory.openSession();
        // 获得 Mybatis 框架生成的 UserMapper 接口的实现类
        UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
        // 第一次查询，发出sql语句，并将查询出来的结果放进缓存中
        User user = userMapper.selectUserById(1);
        System.out.println(user);

        user.setPassword("111111");
        userMapper.updateUserById(user);
        sqlSession.commit();
        // 第二次查询，由于是同一个 sqlSession.commit()，会清空缓存信息
        // 则次此查询也会发出 sql 语句
        User user1 = userMapper.selectUserById(1);
        System.out.println(user1);
        sqlSession.close();
    }
```

![image-20220420190218891](assest/image-20220420190218891.png)

**总结**

1. 第一次发起查询用户id为1的用户信息，先去找缓存中是否有 id 为 1 的用户信息，如果没有，从数据库查询用户信息。得到用户信息，将用户信息存储到一级缓存中。
2. 如果中间 sqlSession 去执行 commit 操作（执行插入、更新、删除），则会清空 sqlSession 中的一级缓存，这样做的目的为了让缓存中存储的是最新的信息，避免脏读。
3. 第二次发起查询用户 id 为 1 的用户信息，先去找缓存中是否有 id 为 1 的用户信息，缓存中有，直接重缓存中获取用户信息。

![image-20220420221710486](assest/image-20220420221710486.png)

## 1.1 一级缓存原理探究与源码分析

一级缓存到底是什么？一级缓存什么时候被创建、一级缓存的工作流程是怎样的？

先从 SqlSession 中看看有没有创建缓存或者与缓存有关的属性或者方法？

![image-20220420222144346](assest/image-20220420222144346.png)

发现上述所有方法中，好像只有 clearCache() 和缓存有点关系，那么直接就从这个方法入手，分析源码，**我们要看它（此类）是谁，它的父类和子类分别又是谁**，对如上关系了解了，你才会对这个类有更深的认识，分析一圈，你可能会得到如下这个流程图：

![image-20220420222954139](assest/image-20220420222954139.png)

再深入分析，流程走到 **PerpetualCache** 中的 clear() 方法之后，会调用其 **cache.clear()** 方法，那么这个 cache 是什么东西呢？点进去发现，cache 其实就是 `private Map<Object, Object> cache = new HashMap<Object, Object>();` 也就是一个 Map ，所以说 cache.clear() 其实就是 map.clear()，也就是说，缓存其实就是 本地存放的一个 map 对象，每一个 SqlSession 都会存放一个 map 对象的引用，那么这个 cache 是何时创建的呢？

发现 Executor 中有一个 createCacheKey 方法，这个方法像是创建缓存的方法，跟进去看看，你发现 createCacheKey 方法是由 BaseExecutor 执行的，代码如下：

```java
@Override
public CacheKey createCacheKey(MappedStatement ms, 
                               Object parameterObject, 
                               RowBounds rowBounds, 
                               BoundSql boundSql) {
    
    CacheKey cacheKey = new CacheKey();
    // MappedStatement 的 id (namespace.id)
    cacheKey.update(ms.getId());
    // offset 就是 0
    cacheKey.update(rowBounds.getOffset());
    // limit 就是 Integer.MAXVALUE
    cacheKey.update(rowBounds.getLimit());
    // 具体的 sql 语句
    cacheKey.update(boundSql.getSql());
    
    // 后面是 update 了 sql 中带的参数
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
        if (parameterMapping.getMode() != ParameterMode.OUT) {
            Object value;
            String propertyName = parameterMapping.getProperty();
            if (boundSql.hasAdditionalParameter(propertyName)) {
                value = boundSql.getAdditionalParameter(propertyName);
            } else if (parameterObject == null) {
                value = null;
            } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
                value = parameterObject;
            } else {
                MetaObject metaObject = configuration.newMetaObject(parameterObject);
                value = metaObject.getValue(propertyName);
            }
            cacheKey.update(value);
        }
    }
    if (configuration.getEnvironment() != null) {
        // issue #176
        cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
}
```

创建缓存 key 会经过一系列的 update 方法，update 方法由一个 CacheKey 这个对象来执行的，这个 update 方法最终由 updateList 把这 六 个值存进去，对照上面的代码和下面的图示，可以理解这六个值分别是什么。

![image-20220420224830460](assest/image-20220420224830460.png)

这里需要注意以下最后一个值，`configuration.getEnvironment().getId()` 是什么，其实就是定义在 SqlMapConfig.xml 中的标签，见下：

```xml
<!--environments：运行环境-->
<environments default="development">
    <environment id="development">
        <!--表示当前事务交由JDBC进行管理-->
        <transactionManager type="JDBC"></transactionManager>
        <!--当前使用mybatis提供的连接池-->
        <dataSource type="POOLED">
            <property name="driver" value="${jdbc.driver}"/>
            <property name="url" value="${jdbc.url}"/>
            <property name="username" value="${jdbc.username}"/>
            <property name="password" value="${jdbc.password}"/>
        </dataSource>
    </environment>
</environments>
```

查看调用栈，发现调用 createCacheKey 的方法 `CachingExecutor#query(org.apache.ibatis.mapping.MappedStatement, java.lang.Object, org.apache.ibatis.session.RowBounds, org.apache.ibatis.session.ResultHandler)`

![image-20220420225627696](assest/image-20220420225627696.png)



![image-20220420230102728](assest/image-20220420230102728.png)

如果缓存中没有，就查询数据库，进入 **`queryFromDatabase`**：

![image-20220420230716972](assest/image-20220420230716972.png)

在 queryFromDatabase 中会对 localCache 进行写入。交由 **PerpetualCache** 的 cache 进行存放。

![image-20220420231234558](assest/image-20220420231234558.png)

# 2 二级缓存

# 3 三级缓存整合 redis

