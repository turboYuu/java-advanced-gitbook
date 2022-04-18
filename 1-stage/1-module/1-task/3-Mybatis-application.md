> 第三部分 Mybatis基本应用

# 1 快速入门

[Mybatis 官网](https://mybatis.org/mybatis-3/)

![image-20220417170009150](assest/image-20220417170009150.png)

## 1.1 开发步骤

1. 添加 Mybatis 的坐标
2. 创建 user 数据表
3. 编写User实体类
4. 编写映射文件 UserMapper.xml 
5. 编写核心文件 SqlMapConfig.xml
6. 编写测试类

## 1.2 环境搭建

[mybatis-quickStart gitee代码地址](https://gitee.com/turboYuu/mybatis-1-1/tree/master/lab-mybatis/mybatis-quickStart)

![image-20220417171421830](assest/image-20220417171421830.png)

![image-20220417171527689](assest/image-20220417171527689.png)

1. 导入 Mybatis的坐标和其他相关坐标

   ```xml
   <properties>
       <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
       <maven.compiler.encoding>UTF-8</maven.compiler.encoding>
       <java.version>1.8</java.version>
       <maven.compiler.source>1.8</maven.compiler.source>
       <maven.compiler.target>1.8</maven.compiler.target>
   </properties>
   
   <dependencies>
       <!--mybatis坐标-->
       <dependency>
           <groupId>org.mybatis</groupId>
           <artifactId>mybatis</artifactId>
           <version>3.4.5</version>
       </dependency>
       <!--mysql驱动坐标-->
       <dependency>
           <groupId>mysql</groupId>
           <artifactId>mysql-connector-java</artifactId>
           <version>5.1.46</version>
       </dependency>
       <!--单元测试坐标-->
       <dependency>
           <groupId>junit</groupId>
           <artifactId>junit</artifactId>
           <version>4.13.2</version>
       </dependency>
   </dependencies>
   ```

   

2. 创建user数据表

   ![image-20220417171836699](assest/image-20220417171836699.png)

3. 编写User实体

   ```java
   package com.turbo.pojo;
   
   public class User {
   
       private Integer id;
   
       private String username;
   
       private String password;
   
       public Integer getId() {
           return id;
       }
   
       public void setId(Integer id) {
           this.id = id;
       }
   
       public String getUsername() {
           return username;
       }
   
       public void setUsername(String username) {
           this.username = username;
       }
   
       public String getPassword() {
           return password;
       }
   
       public void setPassword(String password) {
           this.password = password;
       }
   
       @Override
       public String toString() {
           return "User{" +
                   "id=" + id +
                   ", username='" + username + '\'' +
                   ", password='" + password + '\'' +
                   '}';
       }
   }
   
   ```

   

4. 编写UserMapper映射文件

   ```xml
   <?xml version="1.0" encoding="UTF-8" ?>
   <!DOCTYPE mapper
           PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
           "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
   
   <mapper namespace="user">
       <!--namespace:名称空间 与 id 组成sql的唯一标识-->
   
       <!--resultType:表明返回值类型-->
       <select id="findAll" resultType="com.turbo.pojo.User">
           select * from user
       </select>
   </mapper>
   ```

   

5. 编写Mybatis核心文件

   ```xml
   <?xml version="1.0" encoding="UTF-8" ?>
   <!DOCTYPE configuration
           PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
           "http://mybatis.org/dtd/mybatis-3-config.dtd">
   <configuration>
   
       <!--environments：运行环境-->
       <environments default="development">
           <environment id="development">
               <!--表示当前事务交由JDBC进行管理-->
               <transactionManager type="JDBC"></transactionManager>
               <!--当前使用mybatis提供的连接池-->
               <dataSource type="POOLED">
                   <property name="driver" value="com.mysql.jdbc.Driver"/>
                   <property name="url" value="jdbc:mysql://152.136.177.192:3306/turbine"/>
                   <property name="username" value="root"/>
                   <property name="password" value="123456"/>
               </dataSource>
           </environment>
       </environments>
   
       <!--引入映射配置文件-->
       <mappers>
           <mapper resource="UserMapper.xml"></mapper>
       </mappers>
       
   </configuration>
   ```

6. 编写测试代码

   ```java
   package com.turbo.test;
   
   import com.turbo.pojo.User;
   import org.apache.ibatis.io.Resources;
   import org.apache.ibatis.session.SqlSession;
   import org.apache.ibatis.session.SqlSessionFactory;
   import org.apache.ibatis.session.SqlSessionFactoryBuilder;
   import org.junit.Test;
   
   import java.io.IOException;
   import java.io.InputStream;
   import java.util.List;
   
   public class MybatisTest {
   
       @Test
       public void test1() throws IOException {
           InputStream resourceAsStream = Resources.getResourceAsStream("SqlMapConfig.xml");
           SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(resourceAsStream);
           SqlSession sqlSession = sqlSessionFactory.openSession();
   
           List<User> list = sqlSession.selectList("user.findAll");
           System.out.println(list);
           sqlSession.close();
       }
   }
   ```




## 1.3 Mybatis 的插入操作

1. 编写UserMapper映射文件

   ```xml
   <insert id="add" parameterType="com.turbo.pojo.User">
       insert into user values (#{id},#{username},#{password})
   </insert>
   ```

   

2. 编写插入实体 User 的代码

   ```java
   @Test
   public void addTest() throws IOException {
       InputStream resourceAsStream = Resources.getResourceAsStream("SqlMapConfig.xml");
       SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(resourceAsStream);
       SqlSession sqlSession = sqlSessionFactory.openSession();
       User user = new User();
       user.setUsername("gaowuren");
       user.setPassword("123456");
       int insert = sqlSession.insert("user.add", user);
       System.out.println(insert);
       sqlSession.commit();
       sqlSession.close();
   }
   ```

   

3. 插入操作注意问题

   - 插入语句使用insert标签
   - 在映射文件中使用 parameterType属性指定要插入的数据类型
   - sql语句中使用 #{实体属性名} 方式引用实体中的属性值
   - 插入操作使用的是 API 是 sqlSession.insert("命名空间.id", "实体对象")
   - 插入操作涉及数据库数据变化，所以要使用 sqlSession 对象显示的提交事务，即 sqlSession.commit()



## 1.4 Mybatis 的修改数据操作

1. 编写

# 2 Mybatis 的 Dao 层实现

## 2.1 传统开发方式

## 2.2 代理开发方式













