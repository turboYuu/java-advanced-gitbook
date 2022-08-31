> 第五部分 常见问题及解决方案

本部分主要讲解 Eureka 服务发现慢的原因，Spring Cloud 超时设置问题。

如果你刚刚接触 Eureka，对 Eureka 的设计和实现都不是很了解，可能就会遇到一些无法快速解决的问题，这些问题包括：新服务上线后，服务消费者不能访问到刚上线的新服务，需要过一段时间才能访问？或是将服务下线后，服务还是会被调用到，一段时间后才彻底停止服务，访问前期会导致频繁报错？这些问题还会让你对 Spring Cloud 产生严重的怀疑，这难道不是一个 Bug ？

# 1 问题场景

> 上线一个新的服务实例，但是服务消费者无感知，过了一段时间才知道。
>
> 某一个服务实例下线了，服务消费者无感知，仍然向这个服务实例发起请求。

这其实就是服务发现的一个问题，当我们需要调用服务实例时，信息是从注册中心 Eureka 获取的，然后通过 Ribbon 选择一个服务实例发起调用，如果出现调用不到或者下线后还可以调用的问题，原因肯定是服务实例的信息更新不及时导致的。

# 2 Eureka 服务发现慢的原因

Eureka 服务发现慢的原因主要有两个，一部分是因为服务缓存导致的，另一部分是客户端缓存导致的。

![image-20220830181511537](assest/image-20220830181511537.png)

## 2.1 服务端缓存

服务注册到注册中心后，服务实例信息是存储在注册表中的，也就是内存中。但 Eureka 为了提供响应速度，在内部做了优化，加入了两层的缓存结构，将 Client 需要的实例信息，直接缓存起来。获取的时候直接从缓存中拿数据然后响应给 Client。

第一层缓存是 readOnlyCacheMap，readOnlyCacheMap 是采用 ConcurrentHashMap 来存储数据的，主要负责定时与 readWriteCacheMap 进行数据同步，默认同步时间为 30s 一次。

第二层缓存是 readWriteCacheMap，readWriteCacheMap 采用 Guava 来实现缓存。缓存过期时间默认为 180s，当服务下线、过期、注册、状态变更等操作都会清除此缓存中的数据。

Client 获取服务实例数据时，会先从一级缓存中获取，如果一级缓存中不存在，再从二级缓存中获取，如果二级缓存也不存在，会触发缓存的加载，从存储层拉取数据到缓存中，然后再返回给 Client。

Eureka 之所以设计二级缓存机制，也是为了提高 Eureka Server 的响应速度，缺点是缓存会导致 Client 获取不到最新的服务实例信息，然后导致无法快速发现新的服务和已下线的服务。

了解了服务端的实现后，想要解决这个问题就变得很简单了，我们可以缩短只读缓存的更新时间（eureka.server.response-cache-update-interval-ms）让服务发现变得更加及时，或者直接将只读缓存关闭（eureka.server.use-read-only-response-cache=false），多级缓存也导致 C 层面（数据一致性）很薄弱。

Eureka Server 中会有定时任务去检测失效的服务，将服务实例信息从注册表中移除，也可以将这个失效检测的时间缩短，这样服务下线后就能够及时从注册表中清除。

## 2.2 客户端缓存

客户端缓存主要分为两块内容，一部分是 Eureka Client 缓存，一部分是 Ribbon 缓存。

### 2.2.1 Eureka Client 缓存

Eureka Client 负责跟 Eureka Server 进行交互，在 Eureka Client 中的 `com.netflix.discovery.DiscoveryClient#initScheduledTasks` 方法中，初始化了一个 CacheRefreshThread 定时任务专门用来拉取 Eureka Server 的实例信息到本地。

所以我们需要缩短这个定时拉取服务信息的时间间隔（eureka.client.registry-fetch-interval-seconds）来快速发现新的服务。

### 2.2.2 Ribbon 缓存

Ribbon 会从 Eureka Client 中获取服务信息，ServerListUpdater 是 Ribbon 中负责服务实例更新的组件，默认的实现是 PollingServerListUpdater，通过线程定时去更新实例信息。定时刷新的时间间隔默认是 30s，当服务停止或者上线后，这边最快也需要 30s 才能将实例信息更新成最新的。我们可以将这个时间调短一点，比如 3s。

刷新间隔的参数是通过 getRefreshIntervalMs 方法来获取的，方法中的逻辑也是从 Ribbon 的配置中进行取值的。



将这些服务服务端缓存和客户端缓存的时间全部缩短后，跟默认的配置时间相比，快了很多。我们通过调整参数的方式来尽量加快服务发现的速度，但是还是不能完全解决报错的问题，间隔时间设置为 3s，也还是会有间隔。所以我们一般都会开启重试功能，当路由的服务出现问题时，可以重试到另一个服务来保证这次请求的成功。



# 3 Spring Cloud 各组件超时

在 SpringCloud 中，应用的组件较多，只要涉及通信，就有可能会发生超时。那么如何设置超时时间？在 Spring Cloud 中 ，超时时间只需要重点关注 Ribbon 和 Hystrix 即可。

## 3.1 Ribbon

如果采用的是服务发现方式，就可以通过服务名去进行转发，需要配置 Ribbon 的超时。Ribbon 的超时可以配置全局 ribbon.ReadTimeout 和 ribbon.ConnectTimeout。也可以在前面指定服务名，为每个服务单独配置。比如：user-service.ribbon.ReadTimeout。

其次是 Hystrix 的超时配置，Hystrix 的超时时间要大于 Ribbon 的超时时间，因为 Hystrix 将请求包装了起来，特别需要注意的是，如果 Ribbon 开启的重试机制，比如重试 3次，Ribbon 的超时为1s，那么Hystrix 的超时时间应该大于 3s，否则就会出现 Ribbon 还在重试中，而 Hystrix 已经超时的现象。

## 3.2 Hystrix

Hystrix 全局超时配置就可以用 default 来代替具体的 command 名称。

hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds=3000 ，如果相对具体的 command进行配置，那么就需要知道 command 名称的生成规则，才能准确的配置。

如果我们使用 @HystrixCommand 的话，可以自定义 commandKey。如果使用 FeignClient 的话，可以为 FeignClient 来指定超时时间：hystrix.command.UserRemoteClient.execution.isolation.thread.timeoutInMilliseconds = 3000

如果相对 FeignClient 中的某个接口设置单独的超时，可以在 FeignClient 名称后加上具体的方法：hystrix.command.UserRemoteClient#getUser(Long).execution.isolation.thread.timeoutInMilliseconds = 3000

## 3.3 Feign

Feign 本身也有超时时间的设置，如果此时设置了  Ribbon 的时间就以 Ribbon 的时间为准，如果没设置 Ribbon 的时间但配置了 Feign 的时间，就以 Feign 的时间为准。Feign 的时间同样也配置了连接超时时间（feign.client.config.服务名称.connectTimeout）和 读取超时时间（feign.client.config.服务名称.readTimeout）。

建议，我们配置 Ribbon 超时时间和 Hystrix 超时时间即可。

