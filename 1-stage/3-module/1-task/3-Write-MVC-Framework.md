> 第三部分 手写 MVC 框架

回顾 SpringMVC 执行的大致流程，后续根据这个图模仿手写自己的 mvc 框架

![image-20220409173650257](assest/image-20220409173650257.png)

![image-20220409174916299](assest/image-20220409174916299.png)

![image-20220409175009378](assest/image-20220409175009378.png)

![image-20220409175024973](assest/image-20220409175024973.png)

![image-20220409175109712](assest/image-20220409175109712.png)

然后项目打开后，修改 jdk 级别为 11，删除 pom 中 build，增加 tomcat7 插件。增加 java 和 resources 目录（如果不存在的话，并且 正确标记目录）。

```xml
<!--配置 tomcat 插件-->
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.tomcat.maven</groupId>
            <artifactId>tomcat7-maven-plugin</artifactId>
            <version>2.2</version>
            <configuration>
                <port>8080</port>
                <path>/</path>
            </configuration>
        </plugin>
    </plugins>
</build>
```



# 1 手写MVC 框架之注解开发

- TurboController
- TurboService
- TurboAutowired
- TurboRequestMapping

# 2 TbDispatcherServlet

