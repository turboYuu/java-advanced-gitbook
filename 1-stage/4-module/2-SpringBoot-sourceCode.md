第二部分 SpringBoot源码剖析

# 1 SpringBoot源码环境构建

参考：https://blog.csdn.net/chuanchengdabing/article/details/115871178?spm=1001.2014.3001.5501

## 1.1 下载源码

下载地址：https://github.com/spring-projects/spring-boot/releases

采用 spring-boot-2.2.9.RELEASE 

![image-20220305174341399](assest/image-20220305174341399.png)

## 1.2 环境准备

1. JDK 1.8 +
2. Maven 3.5 +

![image-20220305174617678](assest/image-20220305174617678.png)

注意：源码不要放在中文目录下，最好使用11版本的JDK（JDK1.8一直没有成功）

## 1.3 编译源码

进入 spring-boot 源码根目录

执行 mvn 命令：

```bash
# 跳过测试用例，会下载大量jar包，时间会长一些
mvn clean install -DskipTests -Pfast
```

![image-20220305175226419](assest/image-20220305175226419.png)

![image-20220305180115199](assest/image-20220305180115199.png)





## 1.4 导入IDEA

将编译后的项目导入 IDEA 中

![image-20220305180356893](assest/image-20220305180356893.png)

![image-20220305180440990](assest/image-20220305180440990.png)

![image-20220305180516061](assest/image-20220305180516061.png)

![image-20220305180718525](assest/image-20220305180718525.png)

依赖加载完毕后，此时 pom 配置文件中存在错误引用，打开 pom.xml ，关闭 maven 代码检查，pom 报错消失。

![image-20220305181319156](assest/image-20220305181319156.png)

在pom.xml中可能会出现 **java.lang.outofmemoryerror gc overhead limit exceeded** 的报错，修改 idea maven 的 import 的 vm 参数。

![image-20220305181403292](assest/image-20220305181403292.png)



## 1.5 新建一个module

![image-20220305181721647](assest/image-20220305181721647.png)

![image-20220305181701204](assest/image-20220305181701204.png)

![image-20220305181917403](assest/image-20220305181917403.png)

![image-20220305182013500](assest/image-20220305182013500.png)

**注意**：**修改新创建的子项目的父工程为当前构建的Springboot的版本**

![image-20220305182246908](assest/image-20220305182246908.png)

## 1.6 新建一个Controller

```java
package com.turbo.contorller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class TestController {

	@RequestMapping("/test")
	public String test(){
		System.out.println("源码环境构建完成，congratulations");
		return "源码环境构建完成，congratulations";
	}
}
```

启动测试

![image-20220305183632440](assest/image-20220305183632440.png)

![image-20220305183619590](assest/image-20220305183619590.png)

# 2 源码剖析-依赖管理

**问题**：为什么导入dependency时，不需要指定版本？

在Spring Boot 入门程序中，项目 pom.xml 文件有两个核心依赖，分别是 spring-boot-starter-parent 和 spring-boot-starter-web ，这两个依赖的相关介绍如下：

## 2.1 spring-boot-starter-parent

在新增module中的pom.xml文件中找到 spring-boot-starter-parent 依赖，如下：

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.2.9.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
```

上述代码中，将 spring-boot-starter-parent 依赖作为 Spring Boot 项目的统一父项目依赖管理，并将项目版本号统一为 2.2.9.RELEASE，可根据实际开发修改。

查看 spring-boot-starter-parent 底层源文件，先看 spring-boot-starter-parent 做了哪些事

**首先看 `spring-boot-starter-parent` 的 `properties` 节点** 

```xml
<properties>
    <main.basedir>${basedir}/../../..</main.basedir>
    <java.version>1.8</java.version>
    <resource.delimiter>@</resource.delimiter> <!-- delimiter that doesn't clash with Spring ${} placeholders -->
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <maven.compiler.source>${java.version}</maven.compiler.source>
    <maven.compiler.target>${java.version}</maven.compiler.target>
</properties>
```

在这里`spring-boot-starter-parent` 定义了：

1. 工程的Java版本为`1.8`
2. 工程代码的编译源文件编码格式为 `UTF-8`
3. 工程编译后文件编码格式为`UTF-8`
4. Maven 打包编译的版本

**再来看 `spring-boot-starter-parent` 的 `build` 节点** 

接下来看 POM 的`build`节点，分别定义了 `resource` 资源 和 `pluginManagement`

```xml
<!-- Turn on filtering by default for application properties -->
<resources>
    <resource>
        <directory>${basedir}/src/main/resources</directory>
        <filtering>true</filtering>
        <includes>
            <include>**/application*.yml</include>
            <include>**/application*.yaml</include>
            <include>**/application*.properties</include>
        </includes>
    </resource>
    <resource>
        <directory>${basedir}/src/main/resources</directory>
        <excludes>
            <exclude>**/application*.yml</exclude>
            <exclude>**/application*.yaml</exclude>
            <exclude>**/application*.properties</exclude>
        </excludes>
    </resource>
</resources>
```

详细看一下`resources`节点





## 2.2 spring-boot-starter-web

# 3 自动配置

# 4 Run方法执行流程

# 5 自定义Start

# 6 内嵌Tomcat

# 7 自动配置SpringMVC