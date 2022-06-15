> 7 插件

# 1 Mybatis 的插件机制

Mybatis 允许你在已映射语句执行过程中的某一点进行拦截调用。默认情况下，Mybatis 允许使用插件来拦截的方法调用包括：

1. Executor（update, query, flushStatements, commit, rollback, getTransaction, close, isClosed）
2. ParameterHandler（getParameterObject，setParameters）
3. ResultSetHandler（handleResultSets, handleOutputParameters）
4. StatementHandler（prepare, parameterize, batch, update, query）

我们看到了可以拦截Executor接口的部分方法，比如 update，query，commit，rollback 等方法，还有其他接口的一些方法等。

总体概括为：

1. 拦截执行器的方法
2. 拦截参数的处理
3. 拦截结果集的处理
4. 拦截Sql语法构建的处理

拦截器示例：

```java
package com.turbo.plugin;

import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;

import java.util.Properties;

@Intercepts({
        @Signature(type = Executor.class,
                method = "update",
                args = {MappedStatement.class,Object.class})
})
public class MyInterceptor implements Interceptor {

    /**
     * 拦截方法：只要被拦截的目标对象的目标方法执行时，每次都会执行 intercept方法
     * @param invocation
     * @return
     * @throws Throwable
     */
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        System.out.println("增强逻辑");
        return invocation.proceed();
    }


    /**
     * 主要是为了把这个拦截器生成一个代理对象，放到拦截器链中
     * @param target 拦截的对象
     * @return 代理对象
     */
    @Override
    public Object plugin(Object target) {
        System.out.println("将要包装的目标对象："+target);
        return Plugin.wrap(target,this);
    }

    /**
     * 获取配置文件的属性
     * 插件初始化的调用，也只调用一次，插件配置的属性从这里设置进来
     * @param properties
     */
    @Override
    public void setProperties(Properties properties) {
        System.out.println("插件配置的初始化参数："+properties);
    }
}
```

SpringBoot 注入到 Spring 容器

```java
@Configuration
public class MybatisPlusConfig {

    /**
     * 自定义插件
     * @return
     */
    @Bean
    public MyInterceptor myInterceptor(){
        return new MyInterceptor();
    }
}
```

或者通过 xml 配置，mybatis-config.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <plugins>
        <plugin interceptor="com.turbo.plugin.MyPlugin">
            <property name="name" value="bob"/>
        </plugin>        
    </plugins>
</configuration>    
```



# 2 执行分析插件

# 3 性能分析插件

# 4 乐观锁插件

## 4.1 主要使用场景

## 4.2 插件配置

## 4.3 注解实体字段

## 4.4 测试

## 4.5 特别说明