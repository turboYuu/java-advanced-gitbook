> 第二部分 Tomcat 服务器核心配置详解



问题一：去哪配置？核心配置在 tomcat 目录下下 conf/server.xml 文件

问题二：怎么配置？

注意：

- tomcat 作为服务器的配置，主要是 server.xml 文件的配置；
- server.xml 中包含了 Servlet 容器的相关配置，即 Catalina 的配置；
- xml 文件的讲解主要是标签的使用



**主要标签结构如下**：

```xml
<!-- 
	Server 根元素，创建一个 Server 实例，子标签有 Listener、GlobalNamingResources、Service
 -->
<Server>
  <!--定义监听器-->
  <Listener/>
  <!--定义服务器的全局 JNDI资源-->
  <GlobalNamingResources></GlobalNamingResources>
  <Service name="Catalina"> <!--定义一个Service服务，一个Server标签可以有多个Service服务实例-->
      <Executor/>
      <Connector/>
      <Engine>
          <Host>
              <Context /><!---->
          </Host>
      </Engine>
  </Service>
</Server>
```



# 1 Server 标签

```xml
<!--port:关闭服务器的监听端口
	shutdown:关闭服务器的指令字符串-->
<Server port="8005" shutdown="SHUTDOWN">
  <!--以日志形式输出服务器、操作系统、JVM的版本信息-->	    
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <!-- Security listener. Documentation at /docs/config/listeners.html
  <Listener className="org.apache.catalina.security.SecurityListener" />
  -->
  <!--APR library loader. Documentation at /docs/apr.html -->
  <!--加载（服务器启动）和 销毁（服务器停止）APR。如果找不到 APR 库，则会输出日志，并不影响 tomcat 启动-->  
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <!-- Prevent memory leaks due to use of particular java/javax APIs-->
  <!-- 避免JRE内存泄露问题 -->  
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <!--加载(服务器启动)和销毁(服务器停止) 全局命名服务-->  
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <!--在Context停止重建 Executor 池中的线程，以避免 ThreadLocal相关的内存泄露--> 
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <!-- Global JNDI resources
       Documentation at /docs/jndi-resources-howto.html
		GlobalNamingResources 中定义了全局命名服务
  -->
  <GlobalNamingResources>
    <!-- Editable user database that can also be used by
         UserDatabaseRealm to authenticate users
    -->
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <!-- A "Service" is a collection of one or more "Connectors" that share
       a single "Container" Note:  A "Service" is not itself a "Container",
       so you may not define subcomponents such as "Valves" at this level.
       Documentation at /docs/config/service.html
   -->
  <Service name="Catalina">
	...
  </Service>
</Server>
```

# 2 Service 标签

```xml
<!-- 
	该标签用于创建 Service 实例，默认使用 org.apache.core,StandardService.
	默认情况下，tomcat 仅制定了 Service 的名称，值为 “Catallina”。
	Service 子标签为：Executor、Connector、Engine、、
	其中：
		Executor 用于配置 Service 共享线程池；
		Connector 用于配置 Service 包含的链接器；
		Engine 用于配置 Service 中链接器对应的 Servlet 容器引擎		
-->
<Service name="Catalina">
	...
</Service>
```

# 3 Executor 标签

```xml
<!--The connectors can use a shared executor, you can define one or more named thread pools
默认情况下，Service 并未添加共享线程池配置。如果我们想添加一个线程池，可以在<Service>下添加如下配置：
	name: 线程池名称，用于Connector中指定
	namePrefix: 所创建的每个线程的名称前缀，一个单独的线程名称为 namePrefix + threadNumber
	maxThreads: 池中最大线程数
	minSpareThreads：活跃线程数，也就是核心池线程数，这些线程不会被销毁，会一直存在
	maxIdleTime: 线程空闲时间，超过该时间后，空闲线程会被销毁，默认值为 6000(1分钟)，单位ms
	maxQueueSize: 在被执行前最大线程排队数目，默认为 Int 的最大值，也就是广义的无限。除非特殊情况，这个值不需要更改，否则会有请求不会被处理的情况发生
	prestartminSpareThreads: 启动线程池时是否启动 minSpareThreads 部分线程。默认值为 false，即不启动。
	threadPriority: 线程池中线程优先级，默认值为 5，从1到10
	className: 线程池实现类，未指定情况下，默认实现类为：org.apache.catalina.core.StandardThreadExecutor，如果想使用自定义线程池首先需要实现 org.apache.catalina.Executor接口
-->
<Executor name="tomcatThreadPool" 
          namePrefix="catalina-exec-"
          maxThreads="150" 
          minSpareThreads="4"
          maxIdleTime="6000"
          maxQueueSize="Integer.MAX_VALUE"
          prestartminSpareThreads="false"
          threadPriority="5"
          className="org.apache.catalina.core.StandardThreadExecutor"/>
```

# 4 Connector 标签

Connector 标签用于创建链接器实例

默认情况下，server.xml 配置了两个链接器，一个支持HTTP协议，一个支持 AJP 协议。

大多数情况下，我们并不需要新增链接器配置，只是根据需要对已有链接器进行优化

```xml
<!--
port: 端口号，Connector用于创建服务端 Socket并进行监听，以等待客户端请求链接。如果该属性设置为0，tomcat将会随机选择
	  一个可用的端口号给当前 Connector 使用
protocol: 当前 Connector 支持的访问协议，默认为 HTTP/1.1，并采用自动切换机制选择一个基于JAVA NIO的链接器
		  或者基于本地ARP的链接器（根据本地是否含有 Tomcat 的本地库判定）
connectionTimeout：Connector 接收链接后的等待超时时间，单位为毫秒。-1表示不超时。
redirectPort：当前 Connector不支持SSL请求，接收到了一个请求，并且也符合 security-constraint约束，需要SSL传输，
			  CataLina自动将请求重定向到指定的端口。
executor：指定共享线程池的名称，也可以通过 maxThreads、minSpareThreads 等属性配置内部线程池。
URIEncoding：用于指定编码URI的字符编码，Tomcat 8.x 版本默认的编码为 UTF-8,Tomcat 7.x 版本默认为 IOS-8859-1
-->
<!--org.apache.coyote.http11.Http11NioProtocol, 非阻塞式 Java NIO 链接器-->
<Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />

<Connector protocol="AJP/1.3" address="::1" port="8009" redirectPort="8443" />
```

可以使用共享线程池

```xml
<Connector port="8080" 
           protocol="HTTP/1.1"
           executor="commonThreadPool"
           maxThreads="150"
           minSpareThreads="4"
           acceptCount="1000" 
           maxConnections="1000" 
           connectionTimeout="20000" 
           compression="on" 
           compressionMinSize="2048" 
           disableUploadTimeout="true" 
           redirectPort="8443"
           URIEncoding="UTF-8"/>
```

# 5 Engine 标签

Engine 表示 Servlet 引擎

```xml
<!--
	name：用于指定 Engine 的名称，默认为 Catalina
	defaultHost：默认使用的虚拟主机名称，当客户端请求指定的主机无效时，将交由默认的虚拟主机处理，默认为 localhost
-->
<Engine name="Catalina" defaultHost="localhost">
    ...
</Engine>    
```

## 5.1 Host 标签

Host 标签用于配置一个虚拟主机

```xml
<Host name="localhost"  appBase="webapps"
      unpackWARs="true" autoDeploy="true">
</Host>
```

### 5.1.1 Context 标签

Context 标签用于配置一个 Web 应用，如下：

```xml
<Host name="localhost"  appBase="webapps" unpackWARs="true" autoDeploy="true">
    <!--
		docBase: web应用目录或者war包的部署路径。可以是绝对路径，也可以是相对于 Host appBase 的相对路径。
		path：Web应用的Context路径。如果我们Host名为localhost，则该 web 应用访问的根路径为 ：http://localhost:8080/web_demo
	-->
    <Context docBase="/usr/web_demo" path="/web3"></Context>
    <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
</Host>
```

