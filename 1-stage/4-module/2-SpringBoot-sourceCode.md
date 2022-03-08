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

> **问题1**：为什么导入dependency时，不需要指定版本？

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

详细看一下`resources`节点，里面定义了资源过滤，针对`application`的 `yml`、`properties` 格式进行了过滤，可以支持不同环境的配置，比如 `application-dev.yml`、`application-test.yml`、 `application-dev.properties`、`application-test.properties`等等。

`pluginManagement`则是引入了相应的插件和对应的版本依赖



最后来看 `spring-boot-starter-parent` 的父依赖 `spring-boot-dependencies`，`spring-boot-dependencies` 的 properties 节点。

看定义POM，这个才是 SpringBoot 项目真正管理依赖的项目，里面定义了 SpringBoot 相关的版本：

```xml
<properties>
    <main.basedir>${basedir}/../..</main.basedir>
    <!-- Dependency versions -->
    <activemq.version>5.15.13</activemq.version>
    <antlr2.version>2.7.7</antlr2.version>
    <appengine-sdk.version>1.9.81</appengine-sdk.version>
    <artemis.version>2.10.1</artemis.version>
    <aspectj.version>1.9.6</aspectj.version>
    <assertj.version>3.13.2</assertj.version>
    <atomikos.version>4.0.6</atomikos.version>
    <awaitility.version>4.0.3</awaitility.version>
    <bitronix.version>2.1.4</bitronix.version>
    <byte-buddy.version>1.10.13</byte-buddy.version>
    <caffeine.version>2.8.5</caffeine.version>
    <cassandra-driver.version>3.7.2</cassandra-driver.version>
    <classmate.version>1.5.1</classmate.version>
    ...
</properties>
```

spring-boot-dependencies 的 dependencyManagement 节点，在这里，dependencies 定义了 SpringBoot 版本的依赖组件以及相应版本。

```xml
<dependencyManagement>
    <dependencies>
        <!-- Spring Boot -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot</artifactId>
            <version>${revision}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-test</artifactId>
            <version>${revision}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-test-autoconfigure</artifactId>
            <version>${revision}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-actuator</artifactId>
            <version>${revision}</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-actuator-autoconfigure</artifactId>
            <version>${revision}</version>
        </dependency>
        ...
	<dependencies>
</dependencyManagement>
```

`spring-boot-starter-parent`  通过继承 `spring-boot-dependencies` 从而实现了 SpringBoot 的版本依赖管理，所以我们的SpringBoot 工程继承 `spring-boot-starter-parent` 后就已经具备版本锁定等配置了，这就是在 Spring Boot 项目中 **部分依赖 不需要写版本号**的原因。

## 2.2 spring-boot-starter-web

> 问题2：spring-boot-starter-parent 父依赖启动器的主要作用是进行版本统一管理，那么项目运行依赖的JAR包是从何而来？

查看 spring-boot-starter-web 依赖文件源码，核心代码具体如下：

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-json</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-tomcat</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
        <exclusions>
            <exclusion>
                <groupId>org.apache.tomcat.embed</groupId>
                <artifactId>tomcat-embed-el</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
    </dependency>
</dependencies>
```

从上述代码可以发现，spring-boot-starter-web 依赖启动器的主要作用就是打包Web开发场景需要的底层所有依赖（基于依赖传递，当前项目也存在对应的依赖 jar 包）。

正是如此，在pom.xml中引入 spring-boot-starter-web 依赖启动器时，就可以实现 web 场景 开发，而不需要额外导入 Tomcat 服务器以及其他Web依赖文件等。当然这些引入的依赖文件的版本还是由 `spring-boot-starter-parent` 父依赖进行统一管理。

Spring Boot 除了提供有上述介绍的Web依赖启动器外，还提供了其他许多开发场景的相关依赖，可以 [Spring Boot 官网](https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.build-systems.starters)，查询场景依赖启动器。

![image-20220305190259163](assest/image-20220305190259163.png)

上图这些依赖启动器适用于不同的场景开发，使用时只需要在 pom.xml 文件中导入对应的依赖启动器即可。

但Spring Boot官方并不是对所有场景开发的技术框架都提供场景启动器。例如 Druid 数据源 druid-spring-boot-starter，在 pom.xml问价中引入这些第三方依赖启动器，要配置版本号。

# 3 自动配置

自动配置：根据我们添加的 jar 包依赖，会自动将一些配置类的 bean 注册进 IOC 容器，可以在需要的地方使用 @Autowired 或者 @Resource 等注解来使用。

> 问题3：Spring Boot 到底是如何进行自动配置的，都把哪些组件进行了自动配置？

Spring Boot 应用的启动入口是 @SpringBootApplication 注解标注类中的 main() 方法，`@SpringBootApplication`：Spring Boot 应用标注在某个类上，说明这个类是 Spring Boot 的主配置类，Spring Boot 就应该运行这个类的 main() 方法启动 SpringBoot 应用。



## 3.1 @SpringBootApplication

下面查看 `@SpringBootApplication` 内部源码进行分析，核心代码具体如下：

```java
@Target(ElementType.TYPE) // 注解的适用范围，Type表示注解可以描述在类、接口、注解或枚举中
@Retention(RetentionPolicy.RUNTIME) // 表示注解的声明周期，Runtime 运行时
@Documented // 表示注解可以记录在 javadoc 中
@Inherited // 表示可以被子类继承该注解
@SpringBootConfiguration // 表明该类为配置类
@EnableAutoConfiguration // 启动自动配置功能
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

	// 根据class来排除特定的类，使其不能加入Spring容器，传入参数value类型是class类型
	@AliasFor(annotation = EnableAutoConfiguration.class)
	Class<?>[] exclude() default {};

	// 根据classname 来排除特定的类，使其不能加入spring容器，传入参数 value 类型是 class 的全类名字符串数组
	@AliasFor(annotation = EnableAutoConfiguration.class)
	String[] excludeName() default {};

	// 指定扫描包，参数是包名的字符串数组
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
	String[] scanBasePackages() default {};

	// 扫描特定的包，参数类似是Class类型数组
	@AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
	Class<?>[] scanBasePackageClasses() default {};
}
```

从上述源码可以看出，`@SpringBootApplication` 注解是一个组合注解，前面 4 个是注解元数据信息，主要看后面 3 个注解：@SpringBootConfiguration、@EnableAutoConfiguration、@ComponentScan  三个核心注解，下面具体说明：

## 3.2 @SpringBootConfiguration

@SpringBootConfiguration：SpringBoot 的配置类，标注在某个类上，表示这是一个 SpringBoot 的配置类。

查看 @SpringBootConfiguration 注解源码，核心代码如下：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration // 配置类的作用等同于配置文件，配置类也是容器中的一个对象
public @interface SpringBootConfiguration {
	@AliasFor(annotation = Configuration.class)
	boolean proxyBeanMethods() default true;
}
```

从上述源码可以看出，@SpringBootConfiguration 注解内部有一个核心注解 @Configuration ，该注解是 Spring 框架提供的，表示当前类是一个配置类（XML配置文件的注解表现形式），并可以被租价扫描器扫描。由此可见，@SpringBootConfiguration 注解的作用与 @Configuration 注解相同，**都是标识一个可以被组件扫描器扫描的配置类**，只不过 @SpringBootConfiguration 是被 SpringBoot进行了重新封装命名而已。

## 3.3 @EnableAutoConfiguration

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
// 自动配置包
// 导入的组件是 AutoConfigurationPackages.Registrar.class
@AutoConfigurationPackage 
// Spring的底层注解@Import，给容器中导入一个组件
@Import(AutoConfigurationImportSelector.class)
// 告诉SpringBoot开启自动配置功能，这样自动配置才能生效
public @interface EnableAutoConfiguration {

	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

	// 返回不会被导入到 Spring 容器中的类
	Class<?>[] exclude() default {};

	// 返回不会被导入到 Spring 容器中的类名
	String[] excludeName() default {};
}
```

Spring 中有很多以 `Enable` 开头的注解，其作用就是借助 `@Import` 来收集并注册特定场景相关的 `Bean`，并加载到`IOC`容器。

@EnableAutoConfiguration 就是借助 @Import 来收集所有符合自动配置条件的 bean 定义，并加载到 IoC 容器。

### 3.3.1 @AutoConfigurationPackage

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class) // 导入Registrar中注册的组件
public @interface AutoConfigurationPackage {

}
```

`@AutoConfigurationPackage`：自动配置包，也是一个组合注解，其中最重要的注解是 `@Import(AutoConfigurationPackages.Registrar.class)` ，是 Spring 框架的底层注解，它的作用就是给容器中导入某个组件类，例如 `@Import(AutoConfigurationPackages.Registrar.class)` ，它就是将 `Registrar` 这个组件类导入到容器中，可查看 `Registrar` 类中 `registerBeanDefinitions` 方法：

```java
@Override
public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
    // 将注解标注的元信息传入，获取到相应的包名
    register(registry, new PackageImport(metadata).getPackageName());
}
```

对 `new PackageImport(metadata).getPackageName()` 进行检索，看其结果是什么：

![image-20220306145603096](assest/image-20220306145603096.png)

再看 register 方法：

```java
public static void register(BeanDefinitionRegistry registry, String... packageNames) {
		// 这里参数 packageNames 缺省情况下就是一个字符串，是使用了注解
		// @SpringBootApplication 的 Spring Boot 应用程序入口所在的包
		if (registry.containsBeanDefinition(BEAN)) {
			// 如果该bean已经注册，则将要注册包名称添加进去
			BeanDefinition beanDefinition = registry.getBeanDefinition(BEAN);
			ConstructorArgumentValues constructorArguments = beanDefinition
                .getConstructorArgumentValues();
			constructorArguments.addIndexedArgumentValue(0, 
            	addBasePackages(constructorArguments, packageNames));
		}
		else {
			// 如果该 bean 尚未注册，则注册该bean，参数中提供的包名会被设置到 bean 定义中去
			GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
			beanDefinition.setBeanClass(BasePackages.class);
            // packageNames(com.turbo) = @AutoConfigurationPackage 这个注解的类所在包路径
			beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(0, packageNames);
			beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
			registry.registerBeanDefinition(BEAN, beanDefinition);
		}
	}
```

AutoConfigurationPackages.Registrar 这个类就干一个事，注册一个 `Bean` ，这个 `Bean` 就是 `org.springframework.boot.autoconfigure.AutoConfigurationPackages.BasePackages`，它有一个参数，这个参数是使用了 `@AutoConfigurationPackage` 这个注解的类所在包路径，保存自动配置类以供之后的使用，比如给 `JPA entity` 扫描器用来扫描开发人员通过注解 `@Entity`定义的 `entity`类。

### 3.3.2 @Import(AutoConfigurationImportSelector.class)

`@Import(AutoConfigurationImportSelector.class)`：将 `AutoConfigurationImportSelector` 这个类导入到 Spring 容器中，`AutoConfigurationImportSelector`  可以帮助 SpringBoot 应用将所有符合条件的 `@Configuration` 配置都加载到当前 Spring Boot 创建并使用的 IOC 容器（ApplicationContext）中。

![image-20220306152703080](assest/image-20220306152703080.png)

可以看到 `AutoConfigurationImportSelector` 重点是实现了 `DeferredImportSelector` 接口和各种 `Aware` 接口，然后 `DeferredImportSelector` 又继承了 `ImportSelector` 接口。

其实不止实现了 `ImportSelector` 接口，还实现了很多其他的 `Aware` 接口，分别表示在某个时机 会被回调。

![image-20220306190410305](assest/image-20220306190410305.png)

#### 3.3.2.1 确定自动配置实现逻辑的入口方法

因为 `AutoConfigurationImportSelector`  实现了 接口 `DeferredImportSelector`，那么在 SpringBoot启动时，就会执行 `org.springframework.context.annotation.ConfigurationClassParser.DeferredImportSelectorGrouping#getImports`方法。

所以，和 自动配置逻辑相关的入口方法在 `DeferredImportSelectorGrouping` 类的 `getImports` 方法处，因此我们就从 `org.springframework.context.annotation.ConfigurationClassParser.DeferredImportSelectorGrouping#getImports` 开始分析 SpringBoot的自动配置源码。

先看 `getImports` 方法代码：

```java
// org.springframework.context.annotation.ConfigurationClassParser.DeferredImportSelectorGrouping
public Iterable<Group.Entry> getImports() {
    // 遍历 DeferredImportSelectorHolder 对象集合 deferredImport，deferredImport 集合装了各种 ImportSelector，当然这里装的是 AutoConfigurationImportSelector
    for (DeferredImportSelectorHolder deferredImport : this.deferredImports) {
        // 【1】 利用 AutoConfigurationGroup 的 process 方法来处理自动配置的相关逻辑，决定导入哪些配置类（这是分析重点，自动配置逻辑全在这里）
        this.group.process(deferredImport.getConfigurationClass().getMetadata(),
                           deferredImport.getImportSelector());
    }
    // 【2】 经过上面的处理后，然后再进行选择导入哪些配置类
    return this.group.selectImports();
}
```

标 【1】 处的代码是分析的**重点**，自动配置相关的绝大部分逻辑全在这里。那么 `this.group.process(deferredImport.getConfigurationClass().getMetadata(),deferredImport.getImportSelector())` 主要做的事情就是在 `this.group` 即 `AutoConfigurationGroup` 对象的 `process` 方法中，传入的`AutoConfigurationImportSelector` 对象 来选择一些符合条件的自动配置类，过滤掉一些不符合条件的自动配置类。

![image-20220307105636455](assest/image-20220307105636455.png)

> 注：
> AutoConfigurationGroup：是 AutoConfigurationImportSelector 的内部类，主要用来处理自动配置相关的逻辑，拥有 process 和 selectImports 方法，然后拥有 entries 和 autoConfigurationEntries 集合属性，这两个集合分别存储被处理后的符合条件的自动配置类；
>
> AutoConfigurationImportSelector ：承担自动配置的绝大部分逻辑，负责选择一些符合条件的自动配置类；
>
> Metadata：标注在 SpringBoot 启动类上的 @SpringBootApplication 注解元数据，标【2】的 this.group.selectImports 的方法主要针对前面的 process 方法处理后的自动配置类再进一步有选择地导入

再进入到 AutoConfigurationImportSelector.AutoConfigurationGroup#process 方法：

![image-20220307105917669](assest/image-20220307105917669.png)

通过图中可以看到，自动配置逻辑相关的入口方法在 process 方法中。

#### 3.3.2.2 分析自动配置的主要逻辑

```java
// 这里用来处理自动配置类，比如过滤不符合匹配条件的自动配置类
// org.springframework.boot.autoconfigure.AutoConfigurationImportSelector.AutoConfigurationGroup#process
@Override
public void process(AnnotationMetadata annotationMetadata, 
                    DeferredImportSelector deferredImportSelector) {
    Assert.state(deferredImportSelector instanceof AutoConfigurationImportSelector,
                 () -> String.format("Only %s implementations are supported, got %s",
                                     AutoConfigurationImportSelector.class.getSimpleName(),
                                     deferredImportSelector.getClass().getName()));
    // 【1】 调用 getAutoConfigurationEntry 方法得到自动配置类放入 autoConfigurationEntry 对象中
    AutoConfigurationEntry autoConfigurationEntry = 
        ((AutoConfigurationImportSelector) deferredImportSelector)
        .getAutoConfigurationEntry(getAutoConfigurationMetadata(), annotationMetadata);
    // 【2】 又封装了自动配置类的 autoConfigurationEntry 对象装进 autoConfigurationEntries 集合
    this.autoConfigurationEntries.add(autoConfigurationEntry);
    // 【3】 遍历刚获取的自动配置类
    for (String importClassName : autoConfigurationEntry.getConfigurations()) {
        // 这里符合条件的自动配置类作为 key,annotationMetadata作为值 放进 entries 集合
        this.entries.putIfAbsent(importClassName, annotationMetadata);
    }
}
```

上面代码中我们再来看 标【1】的方法 `getAutoConfigurationEntry`，这个方法主要是用来获取自动配置类，承担了自动配置类的主要逻辑，代码：

```java
// org.springframework.boot.autoconfigure.AutoConfigurationImportSelector#getAutoConfigurationEntry
// 获取符合条件自动配置类，避免加载不必要的自动配置类从而造成内存浪费
protected AutoConfigurationEntry getAutoConfigurationEntry(
    AutoConfigurationMetadata autoConfigurationMetadata,
    AnnotationMetadata annotationMetadata) {
    // 获取是否有配置 spring.boot.enableautoconfiguration 属性，默认返回true
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    // 获得 @Configuration 标注的 Configuration 类即被视为 introspectedClass 的注解数据
    // 比如：@SpringBootApplication(exclude = FreeMarkerAutoConfiguration.class)
    // 将会获取到 exclude = FreeMarkerAutoConfiguration.class 和 excludeName = "" 的注解数据
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // 【1】 得到 spring.factories 文件配置的所有自动配置类
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    // 利用LinkedHashSet移除重复的配置类
    configurations = removeDuplicates(configurations);
    // 得到要排除的自动配置类，比如注解属性 exclude 的配置类
    // 比如：@SpringBootApplication(exclude = FreeMarkerAutoConfiguration.class)
    // 将会获取到 exclude = FreeMarkerAutoConfiguration.class 的注解数据
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    // 检查要被排除的配置类，因为有些不是自动配置类，故要抛出异常
    checkExcludedClasses(configurations, exclusions);
    // 【2】 将要排除的配置类移除
    configurations.removeAll(exclusions);
    // 【3】 因为从  spring.factories 文件获取的自动配置类太多，如果有些不必要的自动配置类都加载进内存，会造成内存浪费，因此这里需要进行过滤
    // 注意这里会调用 AutoConfigurationImportFilter 的 match 方法来判断是否符合 @ConditionalOnBean，@ConditionalOnClass 或 @ConditionalOnWebApplication，后面重点分析
    configurations = filter(configurations, autoConfigurationMetadata);
    // 【4】 获取了符合条件的自动配置类后，此时触发 AutoConfigurationImportEvent 事件
    // 目的是告诉 ConditionEvaluationReport 条件评估器对象来记录符合条件的自动配置类
    // 该事件什么时候会被触发？ --> 在刷新容器时调用 invokeBeanFactoryPostProcessors 后置处理器时触发
    fireAutoConfigurationImportEvents(configurations, exclusions);
    // 【5】 将符合条件和要排除的自动配置类封装进 AutoConfigurationEntry 对象，并返回
    return new AutoConfigurationEntry(configurations, exclusions);
}
```

#### 3.3.2.3 深入 getCandidateConfigurations 方法

这个方法中有一种重要方法 `loadFactoryNames`，这个方法是让 `SpringFactoriesLoader` 去加载一些组件的名字。

```java
//org.springframework.boot.autoconfigure.AutoConfigurationImportSelector#getCandidateConfigurations
protected List<String> getCandidateConfigurations(
    AnnotationMetadata metadata, AnnotationAttributes attributes) {
    
    // 这个方法需要传入两个参数 getSpringFactoriesLoaderFactoryClass() 和 getBeanClassLoader()
    // getSpringFactoriesLoaderFactoryClass() 返回的是 EnableAutoConfiguration.class
    // getBeanClassLoader() 返回是 beanClassLoader（类加载器）
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(
        getSpringFactoriesLoaderFactoryClass(),
        getBeanClassLoader());
    Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
                    + "are using a custom packaging, make sure that file is correct.");
    return configurations;
}
```

继续点开 `loadFactoryNames` 方法

```java
// org.springframework.core.io.support.SpringFactoriesLoader#loadFactoryNames
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    // 获取注入的键 （在自动配置时=EnableAutoConfiguration.class）
    String factoryTypeName = factoryType.getName();
    return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
}
// org.springframework.core.io.support.SpringFactoriesLoader#loadSpringFactories
private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    try {
        // 如果类加载器不为 null，则加载类路径下 spring.factories 文件，将其中设置的 配置类的全路径信息封装为 Enumeration 类对象
        Enumeration<URL> urls = (classLoader != null ?
                                 classLoader.getResources("META-INF/spring.factories") :
                                 ClassLoader.getSystemResources("META-INF/spring.factories");
        result = new LinkedMultiValueMap<>();
		// 循环 Enumeration 类对象，根据相应的节点信息生成 Properties 对象，通过传入的键获取值，在将值切割为一个个小的字符串转化为 Array,方法 result 集合中
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                for (String factoryImplementationName : 
                     StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryTypeName, factoryImplementationName.trim());
                }
            }
        }
        cache.put(classLoader, result);
        return result;
    }
    catch (IOException ex) {
        throw new IllegalArgumentException("Unable to load factories from location [" +
                                           FACTORIES_RESOURCE_LOCATION + "]", ex);
    }
}
```

从代码中可以知道，这个方法中会遍历整个ClassLoader中所有所有 jar 包下的 spring.factories 文件。spring.factories 里面保存着 springboot的默认提供的自动配置类。

![image-20220306171833733](assest/image-20220306171833733.png)





`getAutoConfigurationEntry` 方法主要所做的事情就是获取符合条件的自动配置类，避免加载不必要的自动配置类从而造成内存浪费。**下面总结下 `getAutoConfigurationEntry` 方法主要做的事情**：

1. 从 `spring.factories` 配置文件中加载  `EnableAutoConfiguration` 自动配置类，获取的自动配置类如上图。
2. 若 `@EnableAutoConfiguration`  等注解标有要 `exclude` 的自动配置类，那么再将这个自动配置类排除掉。
3. 排除掉 `exclude`的自动配置类后，然后调用 `filter`方法进一步的过滤，再次排除一些不符合条件的自动配置类。
4. 经过重重过滤后，此时再触发 `AutoConfigurationImportEvent` 事件，告诉 `ConditionEvaluationReport` 条件评估报告器来记录符合条件的自动配置类。
5. 最后再将符合条件的自动配置类返回。

再细看 `org.springframework.boot.autoconfigure.AutoConfigurationImportSelector#filter`  方法：

```java
private List<String> filter(List<String> configurations, 
                            AutoConfigurationMetadata autoConfigurationMetadata) {
    long startTime = System.nanoTime();
    // 将从spring.factories 中获取的自动配置类转出字符串数组
    String[] candidates = StringUtils.toStringArray(configurations);
    // 定义skip数组，是否跳过。注意skip数组与candidates数组顺序一一对应
    boolean[] skip = new boolean[candidates.length];
    boolean skipped = false;
    // getAutoConfigurationImportFilters 方法：拿到 OnBeanCondition，OnClassCondition 和 OnWebApplicationCondition
    // 然后遍历这三个条件类 去过滤从 spring.factories 加载的大量配置类
    for (AutoConfigurationImportFilter filter : getAutoConfigurationImportFilters()) {
        // 调用各种 aware 方法，将beanClassLoader，beanFactory 等注入到 filter 对象中，
        // 这里的 filter 对象即 OnBeanCondition，OnClassCondition 或 OnWebApplicationCondition
        invokeAwareMethods(filter);
        // 判断各种filter，来判断每个 candidates （这里实质要通过 candidates [自动配置类] 拿到其标注的
        // @ConditionalOnClass,@ConditionalOnBean 和 @ConditionalOnWebApplication 里面的注解值）是否匹配。
        // 注意 candidates 数组与 match 数组一一对应
        /************************【主线，重点】******************************/
        boolean[] match = filter.match(candidates, autoConfigurationMetadata);
        // 遍历 match 数组，注意 match 顺序跟  candidates(自动配置类) 一一对应
        for (int i = 0; i < match.length; i++) {
            // 若不匹配的话
            if (!match[i]) {
                // 不匹配的将记录在 skip 数组，标志 skip[i]=true，也与 candidates 数组一一对应
                skip[i] = true;
                // 因为不匹配，将相应的自动配置类置空
                candidates[i] = null;
                // 标注 skipped 为 true
                skipped = true;
            }
        }
    }
    // 这里表示若所有自动配置类经过  OnBeanCondition，OnClassCondition， OnWebApplicationCondition 过滤后，全部都匹配的话，则全部原样返回
    if (!skipped) {
        return configurations;
    }
    // 建立 result 集合来装配的自动配置类
    List<String> result = new ArrayList<>(candidates.length);
    for (int i = 0; i < candidates.length; i++) {
        // 若 skip[i] 为 false，则说明是符合条件的自动配置类，添加到result集合中
        if (!skip[i]) {
            result.add(candidates[i]);
        }
    }
    // 打印日志
    if (logger.isTraceEnabled()) {
        int numberFiltered = configurations.size() - result.size();
        logger.trace("Filtered " + numberFiltered + " auto configuration class in "
                     + TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - startTime) + " ms");
    }
    // 最后返回符合条件的自动配置类
    return new ArrayList<>(result);
}
```

`AutoConfigurationImportSelector`的`filter` 方法主要所得事情就是 **调用** `org.springframework.boot.autoconfigure.AutoConfigurationImportFilter`接口的`match`  方法来判断每一个自动配置类上的条件注解 `@ConditionalOnClass`，`@ConditionalOnBean` 或 `@ConditionalOnWebApplication` 是否满足条件，若满足，返回true；否则返回 false。

### 3.3.3 **关于条件注解的讲解**

@Conditional 是 Spring4 新提供的注解，它的作用是按照一定的条件进行判断，满足条件就向容器注册bean。

1. @ConditionalOnBean：仅仅在当前上下文中存在某个对象时，才会实例化一个Bean。
2. @ConditionalOnMissingBean：仅仅在当前上下文中不存在某个对象时，才会实例化一个Bean。
3. @ConditionalOnClass：某个class位于类路径上，才会实例化一个Bean。
4. @ConditionalOnMissingClass：某个class在类路径上不存在时，才会实例化一个Bean。
5. @ConditionalOnExpression：当表达式为true的时候，才会实例化一个Bean。基于SpEL表达式的条件判断。
6. @ConditionalOnWebApplication：当项目是一个Web项目时，才进行实例化。
7. @ConditionalOnNotWebApplication：当一个项目不是Web项目时进行实例化。
8. @ConditionalOnProperty：当指定的属性有指定的值时进行实例化。
9. @ConditionalOnJava：当JVM版本为指定的版本范围时触发实例化。
10. @ConditionalOnResource：当类路径下有指定的资源时触发实例化。
11. @ConditionalOnJndi：在JNDI村子啊的条件下触发实例化。
12. @ConditionalOnSingleCandidate：当指定的Bean在容器中只有一个，或者有多个但是指定了首选的Bean时触发实例化。



**有选择地导入自动配置类**

`this.group.selectImports()`方法时如何进一步有效的导入自动配置类的：

```java
// AutoConfigurationImportSelector.AutoConfigurationGroup#selectImports
public Iterable<Entry> selectImports() {
    if (this.autoConfigurationEntries.isEmpty()) {
        return Collections.emptyList();
    }
    // 这里得到所有要排除的自动配置类的set集合
    Set<String> allExclusions = this.autoConfigurationEntries.stream()
        .map(AutoConfigurationEntry::getExclusions)
        .flatMap(Collection::stream)
        .collect(Collectors.toSet());
    // 这里得到过滤后所有符合条件的自动配置类的set集合
    Set<String> processedConfigurations = this.autoConfigurationEntries.stream()
        .map(AutoConfigurationEntry::getConfigurations)
        .flatMap(Collection::stream)
        .collect(Collectors.toCollection(LinkedHashSet::new));
    // 移除掉要排除的自动配置类
    processedConfigurations.removeAll(allExclusions);
    // 对标注有 @Order 注解的自动配置类进行排序
    return sortAutoConfigurations(processedConfigurations, getAutoConfigurationMetadata()).stream()
        .map((importClassName) -> new Entry(this.entries.get(importClassName), importClassName))
        .collect(Collectors.toList());
}
```

可以看到，`selectImports` 方法主要是针对经过排除掉 `exclude` 的 和 被 `AutoConfigurationImportFilter` 接口过滤后的满足条件的自动配置类，**再进一步排除 `exclude` 的自动配置类**，然后再排序。

最后，**总结下SpringBoot自动配置的原理**，主要做了以下事情：

1. 从spring.factories 配置文件中加载自动配置类；
2. 加载的自动配置类中排除掉 `@EnableAutoConfiguration`注解的 `exclude` 属性指定的自动配置类；
3. 然后再用 `AutoConfigurationImportFilter` 接口去过滤自动配置类是否符合其标注注解（若有的话）`@ConditionalOnClass`，`@ConditionalOnBean` 或 `@ConditionalOnWebApplication` 的条件，若都符合的话则返回匹配结果；
4. 再触发 `AutoConfigurationImportEvent` 事件，告诉 `ConditionEvaluationReport` 条件评估报告器来记录符合条件的自动配置类和 `exclude`的自动配置类；
5. 最后Spring再将最后筛选后的自动配置类导入 IOC 容器中。

### 3.3.4 以 `HttpEncodingAutoConfiguration` （`Http`编码自动配置）为例解释自动配置原理

```java
// org.springframework.boot.autoconfigure.web.servlet.HttpEncodingAutoConfiguration

// 表示这是一个配置类，和以前编写的配置文件一样，也可以给容器中添加组件
@Configuration(proxyBeanMethods = false)
// 启动指定类的 ConfigurationProperties 功能：将配置文件中对应的值和 HttpProperties 绑定起来；
@EnableConfigurationProperties(HttpProperties.class)
// Spring 底层 @Conditional注解，根据不同的条件，如果满足指定的条件，整个配置类里面的配置就会生效
// 判断当前应用是否是web应用，如果是，当前配置类生效。
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
// 判断当前项目中有没有这个 CharacterEncodingFilter ：SpringMVC中进行乱码解决的过滤器
@ConditionalOnClass(CharacterEncodingFilter.class)
// 判断配置文件中是否存在某个配置 spring.http.encoding ，如果不存在，判断也是成立的
// matchIfMissing = true 表示即使配置文件中不配置  spring.http.encoding.enabled = true,也是默认生效的
@ConditionalOnProperty(prefix = "spring.http.encoding", value = "enabled", matchIfMissing = true)
public class HttpEncodingAutoConfiguration {

	// 它已经和 SpringBoot 配置文件中的值进行映射了
	private final HttpProperties.Encoding properties;

	// 只有一个有参构造器的情况下，参数的值就会从容器中拿
	public HttpEncodingAutoConfiguration(HttpProperties properties) {
		this.properties = properties.getEncoding();
	}

	@Bean // 给容器中添加一个组件，这个组件中的某些值需要从 properties 中获取
	@ConditionalOnMissingBean // 判断容器中 没有 这个组件
	public CharacterEncodingFilter characterEncodingFilter() {
		CharacterEncodingFilter filter = new OrderedCharacterEncodingFilter();
		filter.setEncoding(this.properties.getCharset().name());
		filter.setForceRequestEncoding(this.properties.shouldForce(Type.REQUEST));
		filter.setForceResponseEncoding(this.properties.shouldForce(Type.RESPONSE));
		return filter;
	}
}
```

根据当前不同的条件判断，决定这个配置类是否生效。

一旦这个配置类生效，这个配置类就会给容器中添加各种组件；这些组件的属性是从对应的 `properties` 类中获取的，这些类里面的每一个属性又是和配置文件绑定的。

```properties
spring.http.encoding.enabled=true
spring.http.encoding.charset=utf-8
spring.http.encoding.force=true
```

所有在配置文件中能配置的属性都是在 `xxxProperties` 类中封装着，配置文件能配置什么就可以参照某个功能对应的这个属性类。

```java
@ConfigurationProperties(prefix = "spring.http")
public class HttpProperties {
    public static class Encoding {
		public static final Charset DEFAULT_CHARSET = StandardCharsets.UTF_8;
    }
}
```



### 3.3.5 精髓

1. SpringBoot 启动会加载大量的自动配置类
2. 看我们需要实现的功能有没有 `SpringBoot` 默认写好的自动配置类
3. 再来看看这个自动配置类中到底设置了哪些组件（只要有我们要用的组件，我们就不需要再来配置了）
4. 给容器中自动配置类添加组件的时候，会从 `properties` 类中获取某些属性，我们就可以在配置文件中指定这些属性的值。



`xxxAutoConfiguration`：自动配置类，用于给容器中添加组件从而代替之前我们手动完成大量繁琐的配置。

`xxxProperties`：封装了对应自动配置类的默认属性值，如果我们需要自定义属性值，只需要根据 `xxxProperties` 寻找相关属性在配置文件设置即可。

## 3.4 @ComponentScan  注解

### 3.4.1 @ComponentScan 使用

主要是从定义的扫描路径中，找出标识了需要装配的类自动装配到 Spring 的 Bean 容器中。

常用属性如下：

1. basePackages, value：指定扫描路径，如果为空则以 @ComponentScan 注解的类所在的包为基本的扫描路径
2. basePackageClasses：指定具体扫描类
3. includeFilters：指定满足Filter条件的类
4. excludeFilters：指定排除Filter条件的类

includeFilters 和 excludeFilters 的 FilterType 可选：ANNOTATION（注解类型 默认）、ASSIGNABLE_TYPE（指定固定类）、ASPECTJ（ASPECTJ类型），REGEX（正则表达式），CUSTOM（自定义类型），自定义的Filter需要实现 TypeFilter 接口。

```java
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) }) 
```

借助 excludeFilters 将 TypeExcludeFilter 及 AutoConfigurationExcludeFilter 这两个类进行排除。

当前 @ComponentScan 注解没有标注 basePackages 及 value，所以扫描路径默认为 @ComponentScan  注解所在的包路径为基本的扫描路径（也就是标注了 @SpringBootApplication 注解的项目启动类所在的路径）。



> 问题：@EnableAutoConfiguration 注解是通过 @Import 注解加载了自动配置的 bean，
>
> ​			@ComponentScan 注解自动进行扫描
>
> **那么 真正根据包扫描，把组件类生成实例对象存到 IOC 容器中，又是怎么来完成的**？

# 4 Run方法执行流程

SpringBoot 项目的 main 函数

```java
@SpringBootApplication // 标注一个主程序类，说明这是一个 Spring Boot 应用
public class SpringBootMytestApplication {
	public static void main(String[] args) {
		SpringApplication.run(SpringBootMytestApplication.class, args);
	}
}
```

点击 `run`  方法 

```java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
    // 调用重载方法
    return run(new Class<?>[] { primarySource }, args);
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    // 两件事：1.初始化 SpringApplication 2.执行 run 方法
    return new SpringApplication(primarySources).run(args);
}
```

## 4.1 SpringApplication() 构造方法

继续查看源码，SpringApplication 实例化过程，首先是进入带参数的构造方法，最终回来到两个参数的构造方法。

```java
public SpringApplication(Class<?>... primarySources) {
    this(null, primarySources);
}

@SuppressWarnings({ "unchecked", "rawtypes" })
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    // 设置资源加载器为 null
    this.resourceLoader = resourceLoader;

    // 断言加载资源类不能为null
    Assert.notNull(primarySources, "PrimarySources must not be null");

    // 将 primarySources 数组转换为 List，最后 放到 LinkedHashSet 集合中
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 【1.1】 推断应用类型，后面会根据类型初始化对应的环境。常用的一般都是 servlet 环境
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    // 【1.2】 初始化 classpath 下 META-INFO/spring.factories 中已配置的 ApplicationContextInitializer
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    // 【1.3】 初始化 classpath 下所有已配置的 ApplicationListener
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    // 【1.4】 根据调用栈，推断出 main 方法的类名 
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

### 4.1.1 deduceFromClasspath()

```java
// org.springframework.boot.WebApplicationType
public enum WebApplicationType {
    
	// 应用程序不是web应用，也不应该用web服务器去启动
	NONE,

	// 应用程序应作为基于servlet的web应用程序运行，并应启动嵌入式servlet web（tomcat）服务器。
	SERVLET,

	// 应用程序应作为reactive web应用程序运行，并应启动嵌入式reactive web服务器。
	REACTIVE;
    
    private static final String[] SERVLET_INDICATOR_CLASSES = { "javax.servlet.Servlet",
                "org.springframework.web.context.ConfigurableWebApplicationContext" };

    private static final String WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler";

    // 判断应用类型
    static WebApplicationType deduceFromClasspath() {
        // classpath 下必须存在 org.springframework.web.reactive.DispatcherHandler
        if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) 
            && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
            && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
            return WebApplicationType.REACTIVE;
        }
        for (String className : SERVLET_INDICATOR_CLASSES) {
            // classpath 环境下不存在 javax.servlet.Servlet 或 org.springframework.web.context.ConfigurableWebApplicationContext
            if (!ClassUtils.isPresent(className, null)) {
                return WebApplicationType.NONE;
            }
        }
        return WebApplicationType.SERVLET;
    }
}
```

返回类型是 WebApplicationType 的枚举类型，WebApplicationType 有三个枚举，三个枚举的解释：

- WebApplicationType.REACTIVE classpath下存在
  org.springframework.web.reactive.DispatcherHandler；
- WebApplicationType.SERVLET classpath下存在javax.servlet.Servlet或者 org.springframework.web.context.ConfigurableWebApplicationContext；
- WebApplicationType.NONE 不满足以上条件。

### 4.1.2 setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class))

初始化 classpath 下 META-INF/spring.factories 中已配置的  ApplicationContextInitializer

```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}

/**
	 * 通过指定的 classloader 从 META-INF/spring.factories 获取指定的 Spring 的工厂实例
	 * @param type
	 * @param parameterTypes
	 * @param args
	 * @param <T>
	 * @return
	 */
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, 
                                                      Class<?>[] parameterTypes, 
                                                      Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    // 通过指定的classLoader 从 META-INF/spring.factories 的资源文件中，
    // 读取 key 为 type.getName() 的 value (ApplicationContextInitializer.class)
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 创建 Spring 工厂实例
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    // 对 Spring 工厂实例排序（org.springframework.core.annotation.Order注解指定的顺序）
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

看看 getSpringFactoriesInstances 都做了什么，看源码，有一个很重要的  `SpringFactoriesLoader.loadFactoryNames` 方法，这个方法很重要，这个方法是 spring-core 中提供的从 META-INF/spring.factories中获取指定的类（key）的统一入口方法。

在这里，获取的key 为 `org.springframework.context.ApplicationContextInitializer` 的类。

![image-20220308103548821](assest/image-20220308103548821.png)

上面说了，是从classpath 下 META-INF/spring.factories 中获取的，验证一下：

![image-20220308103831027](assest/image-20220308103831027.png)

![image-20220308103941201](assest/image-20220308103941201.png)

发现在上图所示的两个工程中找到了 debug 中看到的结果。

`ApplicationContextInitializer`  是 Spring 框架的类，这个类的主要目的就是在 ConfigurableApplicationContext 调用 refresh() 方法之前，回调这个类`ApplicationContextInitializer`  的 initialize 方法。

通过 ConfigurableApplicationContext 的实例获取容器的环境 Environment，从而实现对配置文件的修改完善等工作。

### 4.1.3 setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class))

初始化classpath 下 META-INF/spring.facotires 中已配置的 ApplicationListener。

ApplicationListener 的加载过程 和 上面的 ApplicationContextInitializer 类的加载过程是一样的。至于 ApplicationListener 是 Spring 的事件监听器，典型的观察者模式，通过 ApplicationEvent 类 和 ApplicationListener 接口，可以实现对 Spring 容器全生命周期的监听，当然也可以自定义监听事件。

### 4.1.4 总结

关于 SpringApplication 类的构造过程，到这里就梳理完了。纵观 SpringApplication 类的实例化过程，我们可以看到，合理的利用该类，我们能在 Spring 容器创建之前做一些预备工作，和定制化的需求。

比如，自定义 SpringBoot 的 Banner，自定义事件监听器，再比如 在容器 refresh 之前通过自定义的 ApplicationContextInitializer 修改一些配置 或 获取指定的bean 都可以。

## 4.2 run(String... args) 

这一节，总结SpringBoot 启动流程最重要的部分 run 方法。通过 run 方法梳理出 SpringBoot 的启动流程。

经过深入分析，会发现 SpringBoot 也就是给 Spring 包了一层，事先替我们准备好 Spring 所需的环境及一些基础。

```java
/**
	 * Run the Spring application, creating and refreshing a new
	 * {@link ApplicationContext}.
	 * @param args the application arguments (usually passed from a Java main method)
	 * @return a running {@link ApplicationContext}
	 * 运行 Spring 应用，并刷新一个新的 ApplicationContext （Spring 的上下文）
	 * ConfigurableApplicationContext 是 ApplicationContext 接口的子接口，
	 * 在 ApplicationContext 的基础上增加了配置上下文的工具，
	 * ConfigurableApplicationContext 是容器的高级接口。
	 */
public ConfigurableApplicationContext run(String... args) {
    // 记录运行事件
    StopWatch stopWatch = new StopWatch();
    stopWatch.start();
    // ConfigurableApplicationContext Spring de 上下文
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    configureHeadlessProperty();
    // 从 META-INF/spring.factories 中获取监听器
    // 1. 获取并启动监听器
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting();
    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        // 2. 构造应用上下文环境
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        // 处理需要忽略的Bean
        configureIgnoreBeanInfo(environment);
        // 打印 banner
        Banner printedBanner = printBanner(environment);
        // 3. 初始化应用上下文
        context = createApplicationContext();
        // 实例化 SpringBootExceptionReporter.class，用来支持报告关于启动的错误
        exceptionReporters = getSpringFactoriesInstances(
            SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, 
            context);
        // 4. 刷新应用上下文前的准备阶段
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        // 5. 刷新应用上下文
        refreshContext(context);
        // 6. 刷新应用上下文后的扩展接口
        afterRefresh(context, applicationArguments);
        // 时间记录停止
        stopWatch.stop();
        if (this.logStartupInfo) {
            new StartupInfoLogger(
                this.mainApplicationClass).logStarted(getApplicationLog(), 
                                                      stopWatch);
        }
        // 发布容器启动完成事件
        listeners.started(context);
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }

    try {
        listeners.running(context);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```

在以上的代码中，启动过程中的重要步骤 分为六步：

1. 获取并启动监听器
2. 构造应用上下文环境
3. 初始化应用上下文
4. **刷新应用上下文前的准备阶段**  :star:
5. **刷新应用上下文** :star:
6. 刷新应用上下文后的扩展接口

下马 SpringBoot 的启动流程分析，就根据这六大步骤进行详细解读。最重要的是 第4、第5步。

### 4.2.1 获取并启动监听器

事件机制在 Spring 中是很重要的一部分内容，通过事件机制我们可以监听Spring容器中正在发生的一些事件，同样也可以自定义监听事件。Spring的事件为 Bean 和 Bean 之间的消息传递提供支持。当一个对象处理完某种任务后，通知另外的对象进行某些处理，常用的场景有进行某些操作后发通知、消息、邮件等。

```java
private SpringApplicationRunListeners getRunListeners(String[] args) {
    Class<?>[] types = new Class<?>[] { SpringApplication.class, String[].class };
    return new SpringApplicationRunListeners(
        logger,
        getSpringFactoriesInstances(SpringApplicationRunListener.class, types, this, args));
}
```

这里是不是看到一个熟悉的方法：`getSpringFactoriesInstances()`，可以看下面的注释，前面的小结已经详细介绍过该方法是怎么一步步获取到 META-INF/spring.factories 中的指定 key 的 value，获取到以后怎么实例化类的。

```java
/**
	 * 通过指定的 classloader 从 META-INF/spring.factories 获取指定的 Spring 的工厂实例
	 * @param type
	 * @param parameterTypes
	 * @param args
	 * @param <T>
	 * @return
	 */
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, 
                                                      Class<?>[] parameterTypes, 
                                                      Object... args) {
    ClassLoader classLoader = getClassLoader();
    // Use names and ensure unique to protect against duplicates
    // 通过指定的classLoader 从 META-INF/spring.factories 的资源文件中，
    // 读取 key 为 type.getName() 的 value
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    // 创建 Spring 工厂实例
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    // 对 Spring 工厂实例排序（org.springframework.core.annotation.Order注解指定的顺序）
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

回到 run 方法，debug 这个代码 `SpringApplicationRunListeners listeners = getRunListeners(args);` 看一下获取到的是哪个监听器：

![image-20220308144358708](assest/image-20220308144358708.png)

EventPublishingRunListener 监听器是Spring容器的启动监听器。`listeners.starting();` 开启了监听事件。

### 4.2.2 构造应用上下文环境

### 4.2.3 初始化应用上下文

### 4.2.4 刷新应用上下文前的准备阶段

### 4.2.5 刷新应用上下文（IOC容器的初始化过程）

### 4.2.6 刷新应用上下文后的扩展接口

# 5 自定义Start

# 6 内嵌Tomcat

# 7 自动配置SpringMVC



































