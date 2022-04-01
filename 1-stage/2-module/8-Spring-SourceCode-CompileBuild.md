第八部分 Spring 源码编译和环境搭建

# 1 工具

| 软件名称         | 版本     |
| ---------------- | -------- |
| jdk              | 11.0.6   |
| spring-framework | 5.1.x    |
| gradle           | 5.6.3    |
| idea             | 2019.1.3 |

# 2 Spring 和 Gradle

## 2.1 官网下载 spring 源码

官方地址：https://github.com/spring-projects/spring-framework

下载版本 5.1.x

![image-20220331222658346](assest/image-20220331222658346.png)

## 2.2 下载配置 gradle

https://services.gradle.org/distributions/

选择想要安装的发布版本，gradle-x.x-bin.zip 是需要下载的安装发布版，gradle-x.x-src.zip 是源码，gradle-x.x-all.zip 则是下载全部文件。选择下载 gradle-5.6.3-all.zip。

![image-20220331223716840](assest/image-20220331223716840.png)

### 2.2.1 安装 gradle

配置环境变量

![image-20220331224127065](assest/image-20220331224127065.png)

![image-20220331224225251](assest/image-20220331224225251.png)

![image-20220331224043423](assest/image-20220331224043423.png)

### 2.2.2 idea 中配置 gradle

![image-20220331224945246](assest/image-20220331224945246.png)

![image-20220331224344060](assest/image-20220331224344060.png)



## 2.3 idea 导入 spring 源码

![image-20220331224958846](assest/image-20220331224958846.png)

![image-20220331225045649](assest/image-20220331225045649.png)

![image-20220331225117174](assest/image-20220331225117174.png)



**注意：gradle开始进行源码项目构建的时候，会自动下载默认gradle版本进行项目构建，此时，强制结束下载进程**

修改 build.gradle ，将仓库源改为阿里源，这样下载构建速度更快

![image-20220401010438047](assest/image-20220401010438047.png)

```xml
repositories {
    maven { url "https://maven.aliyun.com/repository/spring-plugin" }
    maven { url "https://maven.aliyun.com/nexus/content/repositories/spring-plugin" }
    maven { url "https://repo.spring.io/plugins-release" }
}
```

![image-20220401010527504](assest/image-20220401010527504.png)

```xml
repositories {
    maven { url "https://maven.aliyun.com/repository/central" }
    maven { url "https://repo.spring.io/libs-release" }
    mavenLocal()
}
```

idea 中 gradle 的配置：

![image-20220401010643093](assest/image-20220401010643093.png)

设置完成后，刷新 gradle，让其重新构建源码：

![image-20220401010757177](assest/image-20220401010757177.png)



接下来整个过程会比较长时间。

## 2.4 编译工程

顺序：

```
core	⼯程  —>tasks  —>other  —>compileTestJava
oxm	    ⼯程  —>tasks  —>other  —>compileTestJava
context	⼯程  —>tasks  —>other  —>compileTestJava
beans	⼯程  —>tasks  —>other  —>compileTestJava
aspects	⼯程  —>tasks  —>other  —>compileTestJava
aop	    ⼯程  —>tasks  —>other  —>compileTestJava
```

### 2.4.1 core

![image-20220331231239964](assest/image-20220331231239964.png)

![image-20220331231258425](assest/image-20220331231258425.png)

### 2.4.2 oxm

![image-20220331231443206](assest/image-20220331231443206.png)

![image-20220401010820574](assest/image-20220401010820574.png)

### 2.4.3 context

![image-20220401010918198](assest/image-20220401010918198.png)

![image-20220401010945091](assest/image-20220401010945091.png)

### 2.4.4 beans

![image-20220401011040610](assest/image-20220401011040610.png)

![image-20220401011102028](assest/image-20220401011102028.png)

### 2.4.5 aspects

![image-20220401011143052](assest/image-20220401011143052.png)

![image-20220401011538265](assest/image-20220401011538265.png)

### 2.4.6 aop

![image-20220401011619527](assest/image-20220401011619527.png)

![image-20220401011645393](assest/image-20220401011645393.png)

# 3 创建项目

新建 Model

![image-20220401012114497](assest/image-20220401012114497.png)

![image-20220401012454778](assest/image-20220401012454778.png)

![image-20220401012519562](assest/image-20220401012519562.png)

在当前项目中添加 spring-context 依赖

![image-20220401012846196](assest/image-20220401012846196.png)



创建测试类

![image-20220401014026201](assest/image-20220401014026201.png)

编写配置文件

![image-20220401014100771](assest/image-20220401014100771.png)

编写测试类

![image-20220401014125679](assest/image-20220401014125679.png)



运行：报错：

![image-20220401013841835](assest/image-20220401013841835.png)

解决：

步骤1：

编译spring-context 模块的 spring-context.gradle

将 optional 修改为 compile

![image-20220401014421156](assest/image-20220401014421156.png)

步骤2：

重新编译 spring-context 模块：

![image-20220401014718424](assest/image-20220401014718424.png)

![image-20220401014751526](assest/image-20220401014751526.png)

再次测试，测试成功：

![image-20220401015449526](assest/image-20220401015449526.png)