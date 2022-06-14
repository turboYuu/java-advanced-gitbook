> 3 通用CRUD

通过继承 BaseMapper就可以获取到各种各样的表单操作，接下来将详细讲解这些操作。

![image-20220614103809811](assest/image-20220614103809811.png)

# 1 插入操作

## 1.1 方法定义

插入一条记录

![image-20220614104939109](assest/image-20220614104939109.png)

## 1.2 测试用例

```java
@RunWith(SpringRunner.class)
@SpringBootTest
class TurboMpSpringbootApplicationTests {

    @Autowired
    private UserMapper userMapper;

    @Test
    public void testInsert(){
        User user = new User();
        user.setAge(18);
        user.setName("echo");
        user.setEmail("echo@126.com");
        // 返回的insert是受影响的行数，并不是自增后的id
        int insert = userMapper.insert(user);
        System.out.println(insert);
        System.out.println(user.getId());
    }
}
```

结果：

```xml
1
1536542121278881794
```

![image-20220614105454301](assest/image-20220614105454301.png)

可以看到，数据已经写入到了数据库。但是，id的值不正确，期望的是数据库自增，实际是MP生成了id的值写入到了数据。

**如何设置id的生成策略？**

MP支持的id策略：

```java
package com.baomidou.mybatisplus.annotation;

import lombok.Getter;

/**
 * 生成ID类型枚举类
 *
 * @author hubin
 * @since 2015-11-10
 */
@Getter
public enum IdType {
    /**
     * 数据库ID自增
     */
    AUTO(0),
    /**
     * 该类型为未设置主键类型(注解里等于跟随全局,全局里约等于 INPUT)
     */
    NONE(1),
    /**
     * 用户输入ID
     * <p>该类型可以通过自己注册自动填充插件进行填充</p>
     */
    INPUT(2),

    /* 以下3种类型、只有当插入对象ID 为空，才自动填充。 */
    /**
     * 分配ID (主键类型为number或string）,
     * 默认实现类 {@link com.baomidou.mybatisplus.core.incrementer.DefaultIdentifierGenerator}(雪花算法)
     *
     * @since 3.3.0
     */
    ASSIGN_ID(3),
    /**
     * 分配UUID (主键类型为 string)
     * 默认实现类 {@link com.baomidou.mybatisplus.core.incrementer.DefaultIdentifierGenerator}(UUID.replace("-",""))
     */
    ASSIGN_UUID(4),
    /**
     * @deprecated 3.3.0 please use {@link #ASSIGN_ID}
     */
    @Deprecated
    ID_WORKER(3),
    /**
     * @deprecated 3.3.0 please use {@link #ASSIGN_ID}
     */
    @Deprecated
    ID_WORKER_STR(3),
    /**
     * @deprecated 3.3.0 please use {@link #ASSIGN_UUID}
     */
    @Deprecated
    UUID(4);

    private final int key;

    IdType(int key) {
        this.key = key;
    }
}
```

修改User对象

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@TableName("td_user")
public class User {

    @TableId(type = IdType.AUTO)
    private Long id;
    private String name;
    private Integer age;
    private String email;
}
```

数据插入成功：

![image-20220614110424450](assest/image-20220614110424450.png)

## 1.3 @TableField

官网说明：https://baomidou.com/pages/223848/#tablefield

在MP中通过@TableField注解可以指定自定字段的一些属性，常常解决的问题有两个：

1. 对象中的属性和字段名不一致的问题（非驼峰）
2. 对象中的属性字段在表中不存在的问题

![image-20220614110810072](assest/image-20220614110810072.png)

![image-20220614111318222](assest/image-20220614111318222.png)

其他用法，如大的字段不加入查询：

![image-20220614111638910](assest/image-20220614111638910.png)

![image-20220614111608408](assest/image-20220614111608408.png)



# 2 更新操作

# 3 删除操作

# 4 查询操作

# 5 SQL注入原理