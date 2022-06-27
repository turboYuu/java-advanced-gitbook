> 第五部分 SpringBoot部署与监控

# 1 SpringBoot 项目部署

目前，前后端分离的架构已成主流，而使用SpringBoot构建Web应用是非常快速的，项目发布到服务器上的时候，只需要打成一个 jar 包，然后通过命令：java -jar xxx.jar ，即可启动服务了。

## 1.1 jar包（官方推荐）

SpringBoot 项目默认打包称 jar 包

![image-20220627182510219](assest/image-20220627182510219.png)

> jar 包方式启动，也就是使用 SpringBoot 内置的 tomcat 运行。服务器上面只要你配置了 jdk1.8 及以上就可以，不需要外置 tomcat。

**SpringBoot将项目打包成 jar 包**

1. 首先在 pom.xml 文件中导入 Springboot 的 maven 依赖

   ```xml
   <!--将应用打包成一个可以执行的 jar 包-->
   <build>
       <plugins>
           <plugin>
               <groupId>org.springframework.boot</groupId>
               <artifactId>spring-boot-maven-plugin</artifactId>
           </plugin>
       </plugins>
   </build>
   ```

2. 执行 package

   ![image-20220627183138663](assest/image-20220627183138663.png)

3. package 完成之后，target 中会生成一个 .jar 包

   ![image-20220627185133145](assest/image-20220627185133145.png)

4. 可以将 jar 包上传到 Linux 服务器上，以jar 运行（此处本地验证打包成功）

   ```bash
   java -jar spring-boot-mytest-0.0.1-SNAPSHOT.jar
   ```

   

## 1.2 war 包

## 1.3 jar 包 和 war 包 方式对比

## 1.4 多环境部署

### 1.4.1 @Profile

#### 1.4.1.1 @Profile 的使用位置

#### 1.4.1.2 profile激活

### 1.4.2 多Profile的资源文件

#### 1.4.2.1 资源配置文件

#### 1.4.2.2 效果

### 1.4.3 Spring Profile 和 Maven Profile 融合

# 2 SpringBoot 监控