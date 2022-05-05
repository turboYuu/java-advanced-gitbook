> 第十部分 Mybatis源码剖析

# 1 传统方式源码分析

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



# 2 Mapper代理方式

# 3 二级缓存源码剖析

# 4 延迟加载源码剖析