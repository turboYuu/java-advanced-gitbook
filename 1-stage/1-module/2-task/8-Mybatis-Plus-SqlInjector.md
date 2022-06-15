> 8 Sql注入器

https://baomidou.com/pages/42ea4a/

在 MP 中，通过 AbstractSqlInjector 将 BaseMapper 中的方法注入到了 Mybatis 容器，这样这些方法才可以正常执行。

那么，如果我们需要扩充 BaseMapper 中的方法，又该如何实现？

# 1 编写MyBaseMapper

```java
package com.turbo.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;

import java.util.List;

public interface MyBaseMapper<T> extends BaseMapper<T> {

    List<T> findAll();
}
```

其他的Mapper都可以继承该 Mapper，这样实现了统一的扩展

如：

```java
package com.turbo.mapper;

import com.turbo.pojo.User;

public interface UserMapper extends MyBaseMapper<User> {

    User findById(Long id);
}
```



# 2 编写MySqlinjector

如果直接继承 AbstractSqlInjector 的话，原有的BaseMapper中的方法将失效，所以选择继承 DefaultSqlInjector 进行扩展。

```java
package com.turbo.sqlInjector;

import com.baomidou.mybatisplus.core.injector.AbstractMethod;
import com.baomidou.mybatisplus.core.injector.DefaultSqlInjector;

import java.util.List;

public class MySqlInjector extends DefaultSqlInjector {

    @Override
    public List<AbstractMethod> getMethodList() {
        List<AbstractMethod> methodList = super.getMethodList();
        // 扩充自定义的方法
        methodList.add(new FindAll());
        return methodList;
    }
}
```



# 3 编写FindAll

```java
package com.turbo.sqlInjector;

import com.baomidou.mybatisplus.core.injector.AbstractMethod;
import com.baomidou.mybatisplus.core.metadata.TableInfo;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.SqlSource;

public class FindAll extends AbstractMethod {
    @Override
    public MappedStatement injectMappedStatement(Class<?> mapperClass, Class<?> modelClass, TableInfo tableInfo) {
        String sqlMethod = "findAll";
        String sql = "select * from " + tableInfo.getTableName();
        SqlSource sqlSource = languageDriver.createSqlSource(configuration, sql, modelClass);

        return this.addSelectMappedStatement(mapperClass,sqlMethod,sqlSource,modelClass,tableInfo);
    }
}
```



# 4 注册到 Spring 容器

```java
/**
     * 自定义 sql 注入器
     * @return
     */
@Bean
public MySqlInjector mySqlInjector(){
    return new MySqlInjector();
}
```



# 5 测试

```java
@Autowired
private UserMapper userMapper;

@Test
public void testMySqlInjector(){
    List<User> users = userMapper.findAll();
    for (User user : users) {
        System.out.println(user);
    }
}
```

输出sql：

```sql
17:14:09,376 DEBUG findAll:143 - ==>  Preparing: select * from td_user 
17:14:09,415 DEBUG HikariPool:729 - HikariPool-1 - Added connection com.mysql.jdbc.JDBC4Connection@acaffd9
17:14:09,418 DEBUG findAll:143 - ==> Parameters: 
17:14:09,500 DEBUG findAll:143 - <==      Total: 6
```

至此实现了全局扩展SQL注入器。