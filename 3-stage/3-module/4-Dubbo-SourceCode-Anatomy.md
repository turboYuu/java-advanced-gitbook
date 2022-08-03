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

# 3 服务注册与消费源码剖析

# 4 Dubbo 扩展 SPI 源码剖析

# 5 集群容错源码剖析

# 6 网络通信原理剖析