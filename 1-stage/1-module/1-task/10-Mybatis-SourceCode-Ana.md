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
        Properties settings = settingsAsProperties(root.evalNode("settings"));
        loadCustomVfs(settings);
        typeAliasesElement(root.evalNode("typeAliases"));
        pluginElement(root.evalNode("plugins"));
        objectFactoryElement(root.evalNode("objectFactory"));
        objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
        reflectorFactoryElement(root.evalNode("reflectorFactory"));
        settingsElement(settings);
        // read it after objectFactory and objectWrapperFactory issue #631
        environmentsElement(root.evalNode("environments"));
        databaseIdProviderElement(root.evalNode("databaseIdProvider"));
        typeHandlerElement(root.evalNode("typeHandlers"));
        mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
        throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
}
```



# 2 Mapper代理方式

# 3 二级缓存源码剖析

# 4 延迟加载源码剖析