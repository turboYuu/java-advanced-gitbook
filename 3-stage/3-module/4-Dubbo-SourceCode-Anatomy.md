> 第四部分 Dubbo源码剖析

# 1 源码下载和编译

1. dubbo 的项目在 github 中的地址为 ：https://github.com/apache/dubbo
2. 进行需要进行下载的目录，执行 `git clone https://github.com/apache/dubbo.git`
3. 为了防止 master 中的代码不稳定，进入 dubbo 项目 `cd dubbo` 可以及切换到 2.7.6 分支 `git checkout 2.7.6`
4. 进行本地编译，进入 dubbo 项目 `cd dubbo`，进行编译操作 `mvn clean install -DskipTests`
5. 使用 IDE 引入项目。



![image-20220803124139668](assest/image-20220803124139668.png)

# 2 架构整体设计

[Dubbo 框架设计-官网说明](https://dubbo.apache.org/zh/docsv2.7/dev/design/)

## 2.1 Dubbo 调用关系说明

![/dev-guide/images/dubbo-relation.jpg](assest/dubbo-relation.jpg)

在这里主要由四部分组成：

1. Provider：暴露服务的服务提供方
   - Protocol：负责提供者 和 消费者 之间协议交互数据。
   - Service：真实的业务服务信息，可以理解成接口 和 实现。
   - Container：Dubbo 的运行环境。
2. Consumer：调用远程服务的服务消费方
   - Protocol：负责提供者 和 消费者 之间协议交互数据。
   - Cluster：感知提供者端的列表信息。
   - Proxy：可以理解成 提供者的服务调用代理类，由它接管 Consumer 中的接口调用逻辑。
3. Register：注册中心。用于作为服务发现 和 路由配置等工作，提供者和消费者都会在这里进行注册。
4. Monitot：用于提供者和消费者中的数据统计，比如调用频次，成功失败次数等信息。



**启动 和 执行流程说明**：

提供者端启动，容器负责把 Service 信息加载，并通过 Protocol 注册到注册中心；

消费者端启动，通过监听提供者列表来感知提供者信息，并在提供者发生改变时，通过注册中心及时通知消费端；

消费方发起请求，通过 Proxy 模块；

利用 Cluster 模块，来选择真实的要发送给的提供者信息；

交由 Consumer 中的 Protocol 把信息发送给提供者；

提供者同样需要通过 Protocol 模块来处理消费者的信息；

最后由真正的服务提供者 Service 来进行处理。

## 2.2 整体调用链路

![/dev-guide/images/dubbo-extension.jpg](assest/dubbo-extension.jpg)

说明：淡绿色代表了 服务生产者的范围；淡蓝色 代表 服务消费者的范围；红色箭头代表了调用的方向。

业务逻辑层：RPC层（远程过程调用） Remoting（远程数据传输）

整体链路调用的流程：

1. 消费者通过 Interface 进行方法调用，统一交由消费端的 Proxy，通过 ProxyFactory 来进行代理对象的创建，使用到了 jdk、javassist 技术。
2. 交给 Filter 这个模块，做一个统一的过滤请求，在 SPI 案例中涉及过。
3. 接下来会进入最主要的 Invoker 调用逻辑。
   - 通过 Directory 去配置中新读取信息，最终通过 list 方法获取所有的 Invoker
   - 通过 Cluster 模块，根据选择的具体路由规则，来选取 Invoker 列表
   - 通过 LoadBalance 模块，根据负载均衡策略，选择一个具体的 Invoker 来处理我们的请求
   - 如果执行中出现错误，并且 Consumer 阶段配置了重试机制，则会重新尝试执行。
4. 继续经过 FIlter ，进行执行功能的前后封装，Invoker 选择具体的执行协议
5. 客户端 进行编码 和 序列化，然后发送数据
6. 到达 Provider 中的 Server，在这里进行反编码 和 反序列化 的接收数据
7. 使用 Exporter 选择执行执行器
8. 交给 Filter 进行一个提供者端的过滤，到达 Invoker 执行器
9. 通过 Invoker 调用接口的具体实现，然后返回

## 2.3 Dubbo源码整体设计

![/dev-guide/images/dubbo-framework.jpg](assest/dubbo-framework.jpg)

图例说明：

- 图中左边淡蓝背景 的为 服务消费方使用的接口；右边淡绿色背景的为服务提供方使用的接口；位于中轴线上的为双方都用到的接口。
- 图中从下至上分为 十层，各层均为单向依赖，右边的黑色箭头代表层之间的依赖关系，每一层都可以剥离上层被复用。其中 ，Service 和 Config 层为 API，其它各层均为 SPI。
- 图中绿色小块的为扩展接口，蓝色小块为实现类，图中只显示用于关联各层的实现类。
- 图中蓝色虚线为初始化过程，即启动时组装链；红色实线为方法调用过程，即运行时调用链；紫色三角箭头为继承，可以把子类看作父类的同一个节点，线上的文字为调用的方法。



### 2.3.1 分层介绍

[各层说明-官网说明](https://dubbo.apache.org/zh/docsv2.7/dev/design/#%E5%90%84%E5%B1%82%E8%AF%B4%E6%98%8E)

# 3 服务注册与消费源码剖析

## 3.1 注册中心 Zookeeper 剖析

[Zookeeper注册中心参考手册-官网说明](https://dubbo.apache.org/zh/docsv2.7/user/references/registry/zookeeper/)

注册中心是 Dubbo 的重要组成部分，主要用于服务的注册与发现，可以选择 Redis、Nacos、Zookeeper 作为 Dubbo 的注册中心，Dubbo 推荐用户使用 Zookeeper 作为注册中心。

### 3.1.1 注册中心 Zookeeper 目录结构

使用一个最基本的服务注册与消费的 Demo 来进行说明。

例如：只有一个提供者和消费者。`com.turbo.service.HelloService` 为所提供的服务。

```java
public interface HelloService {
    String sayHello(String name);
}
```

则 Zookeeper 的目录结构如下：

![image-20220803173449891](assest/image-20220803173449891.png)

- 可以在这里看到所有的都是在 dubbo 层级下的；
- dubbo 根节点下面是当前所拥有的接口名称，如果有多个接口，则会以多个子节点的形式展开；
- 每个服务下面有分别有四个配置项
  - consumers：当前服务下面所有的消费者列表（URL）
  - providers：当前服务下面所有的提供者列表（URL）
  - configurators：当前服务下面的配置信息，provider 或者 consumer 会通过读取这里的配置信息来获取配置
  - routers：当消费者在进行获取提供者时，会通过这里配置好的路由来进行适配匹配规则
- 可以看到，dubbo基本上很多时候都是通过 URL 的形式来进行交互获取数据的，在 URL 中也会保存很多的信息。后面也会对 URL 的规则做详细介绍。

![/user-guide/images/zookeeper.jpg](assest/zookeeper.jpg)

通过这张图我们可以了解到如下信息：

- 提供者会在 `providers` 目录下进行自身的注册。
- 消费者会在 `consumers` 目录下进行自身注册，并且监听 `providers` 目录，以此通过监听提供者的变化，实现服务发现。
- Monitor 模块会对整个服务级别做监听，用来得知整体的服务情况。以此就能更多的对整体情况做监控。



## 3.2 服务注册过程分析

[服务注册（暴露）过程](https://dubbo.apache.org/zh/docsv2.7/dev/implementation/#%E6%9C%8D%E5%8A%A1%E6%8F%90%E4%BE%9B%E8%80%85%E6%9A%B4%E9%9C%B2%E4%B8%80%E4%B8%AA%E6%9C%8D%E5%8A%A1%E7%9A%84%E8%AF%A6%E7%BB%86%E8%BF%87%E7%A8%8B)

![/dev-guide/images/dubbo_rpc_export.jpg](assest/dubbo_rpc_export.jpg)

首先 `ServiceConfig` 类拿到对外提供服务的实际类 ref（如：HelloServiceImpl），然后通过 `ProxyFactory` 接口实现类中的 `getInvoker` 方法使用 ref 生成一个 `AbstractProxyInvoker` 实例，到这一步就完成具体服务到 `Invoker` 的转化。接下来就是 `Invoker` 转换到 `Exporter` 的过程。



查看 ServiceConfig 类：重点查看 ProxyFactory 和 Protocol 类型的属性，以及 ref。



下面我们就看一下 Invoker  转换成 Exporter 的过程：

其中会涉及到 RegistryService 接口、`RegistryFactory` 接口 和 注册 provider 到注册中心流程的过程。

1. RegistryService 代码解读，这里的代码比较简单，主要是对指定的路径进行注册、解绑、监听 和 取消监听、查询操作。也是注册中心最为基础的类。

   ```java
   package org.apache.dubbo.registry;
   
   import org.apache.dubbo.common.URL;
   
   import java.util.List;
   
   public interface RegistryService {
   
       /**
        * 进行对 URL的注册操作,比如: provider service, consumer address, route rule, override rule and other data.
        */
       void register(URL url);
   
       /**
        * 解除对指定URL的注册，比如provider，consumer，routers等
        */
       void unregister(URL url);
   
       /**
        * 增加对指定URL的路径监听，当有变化的时候进行通知操作
        */
       void subscribe(URL url, NotifyListener listener);
   
       /**
        * 解除对指定 URL 的路径监听，取消指定的listener
        */
       void unsubscribe(URL url, NotifyListener listener);
   
       /**
        * 查询指定URL下面的URL列表，比如查询指定服务下面的consumer列表
        */
       List<URL> lookup(URL url);
   
   }
   ```

2. 再来看 `RegistryFactory`，是通过它来生成真实的注册中心。通过这种方式，也可以保证一个应用中可以使用多个注册中心。可以看到这里也是通过不同的 protocol 参数，来选择不同的协议。

   ```java
   package org.apache.dubbo.registry;
   
   import org.apache.dubbo.common.URL;
   import org.apache.dubbo.common.extension.Adaptive;
   import org.apache.dubbo.common.extension.SPI;
   
   
   @SPI("dubbo")
   public interface RegistryFactory {
       /**
        * 获取注册中心地址
        */
       @Adaptive({"protocol"})
       Registry getRegistry(URL url);
   }
   ```

3. 下面就来跟踪一下，一个服务是如何注册到注册中心上去的。其中比较关键的一个类是 `RegistryProtocol`，它负责管理整个注册中心相关的协议，并且统一对外提供服务。这里我们主要以 `RegistryProtocol#export` 方法作为入口，这个方法主要的作用就是将我们需要执行的信息注册并且导出。

   ```java
   @Override
   public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
       // 获取注册中心的地址
       // zookeeper://152.136.177.192:2181/org.apache.dubbo.registry.RegistryService?application=service-provider&dubbo=2.0.2&export=dubbo%3A%2F%2F192.168.56.1%3A20880%2Fcom.turbo.service.HelloService%3Fanyhost%3Dtrue%26application%3Dservice-provider%26bind.ip%3D192.168.56.1%26bind.port%3D20880%26deprecated%3Dfalse%26dubbo%3D2.0.2%26dynamic%3Dtrue%26generic%3Dfalse%26interface%3Dcom.turbo.service.HelloService%26methods%3DsayHello%26pid%3D12816%26qos.accept.foreign.ip%3Dfalse%26qos.enable%3Dtrue%26qos.port%3D33333%26release%3D2.7.6%26side%3Dprovider%26timestamp%3D1659586078756&pid=12816&qos.accept.foreign.ip=false&qos.enable=true&qos.port=33333&release=2.7.6&timeout=60000&timestamp=1659586078724
       URL registryUrl = getRegistryUrl(originInvoker);
       // 获取当前提供者需要注册的地址
       // dubbo://192.168.56.1:20880/com.turbo.service.HelloService?anyhost=true&application=service-provider&bind.ip=192.168.56.1&bind.port=20880&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.turbo.service.HelloService&methods=sayHello&pid=12816&qos.accept.foreign.ip=false&qos.enable=true&qos.port=33333&release=2.7.6&side=provider&timestamp=1659586078756
       URL providerUrl = getProviderUrl(originInvoker);
   
       // 获取进行注册 override协议的访问地址
       // provider://192.168.56.1:20880/com.turbo.service.HelloService?anyhost=true&application=service-provider&bind.ip=192.168.56.1&bind.port=20880&category=configurators&check=false&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.turbo.service.HelloService&methods=sayHello&pid=12816&qos.accept.foreign.ip=false&qos.enable=true&qos.port=33333&release=2.7.6&side=provider&timestamp=1659586078756
       final URL overrideSubscribeUrl = getSubscribedOverrideUrl(providerUrl);
       // 增加override的监听器
       final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
       overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
   
       // 根据现有的override协议，对注册地址进行改写操作
       providerUrl = overrideUrlWithConfig(providerUrl, overrideSubscribeListener);
       // 对当前的服务进行本地导出
       // 完成后即可以看到本地的20880端口已经启动，并且暴露服务
       final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker, providerUrl);
   
       // 获取真实的注册中心，比如我们常用的 ZookeeperRegistry
       final Registry registry = getRegistry(originInvoker);
       // 获取当前服务需要注册到注册中心的 providerURL，主要用于去除一些没有必要的参数（比如在本地导出时所使用的 qos 参数等值）
       // dubbo://192.168.56.1:20880/com.turbo.service.HelloService?anyhost=true&application=service-provider&deprecated=false&dubbo=2.0.2&dynamic=true&generic=false&interface=com.turbo.service.HelloService&methods=sayHello&pid=12816&release=2.7.6&side=provider&timestamp=1659586078756
       final URL registeredProviderUrl = getUrlToRegistry(providerUrl, registryUrl);
       ProviderInvokerWrapper<T> providerInvokerWrapper = ProviderConsumerRegTable.registerProvider(originInvoker,
                                                                                                    registryUrl, registeredProviderUrl);
       // 获取当前url是否需要进行注册参数
       boolean register = registeredProviderUrl.getParameter(REGISTER_KEY, true);
       if (register) {
           // 将当前的提供住注册到注册中心上去
           register(registryUrl, registeredProviderUrl);
       }
   
       // 对override协议进行注册，用于在接收到overrider请求时做适配，这种方式用于适配2.6.x 及之前的版本(混用)
       registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
   
       // 设置当前导出中的相关信息
       exporter.setRegisterUrl(registeredProviderUrl);
       exporter.setSubscribeUrl(overrideSubscribeUrl);
       // 返回导出对象（对数据进行封装）
       return new DestroyableExporter<>(exporter);
   }
   ```

   

# 4 Dubbo 扩展 SPI 源码剖析

# 5 集群容错源码剖析

# 6 网络通信原理剖析





