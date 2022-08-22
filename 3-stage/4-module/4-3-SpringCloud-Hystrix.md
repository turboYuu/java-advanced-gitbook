> 4-3 Hystrix 熔断器

属于一种容错机制

# 1 微服务中的雪崩效应

**什么是微服务中的雪崩效应呢？**

微服务中，一个请求可能需要多个微服务接口才能实现，会形成复杂的调用链路。



扇入：代表着该微服务被调用的次数，扇入大，说明该模块复用性好。

扇出：该微服务调用其他微服务的个数，扇出大，说明业务逻辑复杂。

扇入大是一个好事，扇出大不一定是好事。

在微服务架构中，一个应用可能会有多个微服务组成，微服务之间的数据交互通过**远程过程调用**（RPC）完成。这就带来一个问题：假设微服务A 调用 微服务B 和 微服务C，微服务B 和 微服务C 又调用其他的微服务，这就是所谓的“扇出”。如果扇出的链路上某个微服务的调用响应时间过程挥着不可用，对微服务A 的调用就会占用越来越多的系统资源，进而引起系统崩溃，所谓的 “雪崩效应”。

最下游 微服务响应时间过程，大量请求阻塞，大量线程不会释放，会导致服务器资源耗尽，，最终导致上游整个系统瘫痪。

# 2 雪崩效应解决方案

从可用性可靠性着想，为防止系统的整体缓慢甚至崩溃，采用的技术手段：

下面，介绍三种技术手段应对微服务中的雪崩效应，这三种手段都是从系统可用性、可靠性角度出发，尽量防止系统整体缓慢甚至瘫痪。

1. 服务熔断

   熔断机制是应对雪崩效应的一种微服务链路保护机制。

   当扇出链路的某个微服务不可用或者响应时间太长时，熔断该节点微服务的调用，进行服务的降级，快速返回错误的响应信息。当检测到该节点微服务调用响应正常后，恢复调用链路。

   **注意**：

   - 服务熔断重点在 **断**，切断对下游服务的调用
   - 服务熔断和服务降级往往是一起使用的，Hystrix 就是这样。

2. 服务降级

   通俗讲就是整体资源不够用，先将一些不关紧的服务停掉（调用我的时候，给你返回一个预留的值，也叫做**兜底数据**），待度过难关高峰过去，再把那些微服务打开。

   服务降级一般是从整体考虑，就是当某个服务熔断之后，服务器将不再被调用，此刻客户端可以自己准备一个本地的 fallback 回调，返回一个缺省值，这样做虽然服务水平下降，但好歹可用，比直接挂掉要强。

3. 服务限流

   限流的措施也很多，比如：

   - 限制总并发数（比如数据库连接池、线程池）
   - 限制瞬时并发数（如 nginx 限制瞬时并发连接数）
   - 限制时间窗口内的平均速率（如 Guava 的 RateLimiter、nginx的limit_req模块，限制每秒的平均速率）
   - 限制远程接口调用速率、限制 MQ 的消费速率等

# 3 Hystrix 简介

Hystrix 主要通过以下几点实现延迟和容错。

- 包裹请求：使用 HystirxCommand 包裹对调用的依赖。
- 跳闸机制：当某服务的错误率超过一定的阈值时，Hystrix可以跳闸，停止请求该服务一段时间
- 资源隔离：Hystrix为每个依赖都维护了一个小型的线程池（舱壁模式）（或者信号量）。如果线程池已满，发往该依赖的请求就被立即拒绝，而不是排队等待，从而加速失败判定。
- 监控：Hystrix 可以近乎实时地监控运行指标和配置的变化，例如成功、失败、超时、以及被拒绝的请求等。
- 回退机制：当请求失败、超时、被拒绝，或当断路器打开时，执行回退逻辑。回退逻辑由开发人员自定提供，例如返回一个缺省值。
- 自我修复：断路器打开一段时间后，会自动进入“半开”状态。

# 4 Hystrix 熔断应用

目的：简历微服务长时间没有响应，服务消费者 -> **自动投递微服务**快速失败给用户提示。

![image-20220822114307649](assest/image-20220822114307649.png)

1. 服务消费者工程（`turbo-service-autodeliver-8092-hystrix`）中引入 Hystrix 依赖坐标（也可以添加在父工程中）

   ```xml
   <!--熔断器 Hystrix-->
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
   </dependency>
   ```

2. 服务消费者工程（`turbo-service-autodeliver-8092-hystrix`）的启动类中添加熔断器开启注解 `@EnableCircuitBreaker`

   ```java
   /**
    * 注解简化写法：@SpringCloudApplication = @SpringBootApplication + @EnableDiscoveryClient + @EnableCircuitBreaker
    **/
   @SpringBootApplication
   @EnableDiscoveryClient // 开启服务发现
   @EnableCircuitBreaker // 开启熔断
   public class AutoDeliverApplication8092 {
       public static void main(String[] args) {
           SpringApplication.run(AutoDeliverApplication8092.class,args);
       }
   
       @Bean
       @LoadBalanced // ribbon负载均衡
       public RestTemplate getRestTemplate(){
           return new RestTemplate();
       }
   }
   ```

3. 定义服务降级处理方法，并在业务方法上使用 `@HystrixCommand` 的 fallbacKMethod 属性关联到服务降级处理方法。

   ```java
   @RestController
   @RequestMapping("/autodeliver")
   public class AutodeliverController {
   
       @Autowired
       RestTemplate restTemplate;
     
       /**
        * 提供者模拟超时处理，调用方法添加 Hystrix 控制
        * http://localhost:8092/autodeliver/checkStateTimeout/2195320
        * @param userId
        * @return
        */
       // 使用 @HystrixCommand注解进行熔断控制
       @HystrixCommand(
               threadPoolKey = "findResumeOpenStateTimeout",
               threadPoolProperties = {
                       @HystrixProperty(name = "coreSize",value = "1"), // 线程数
                       @HystrixProperty(name = "maxQueueSize",value = "20") //等待队列长度
               },
               // commandProperties熔断的一些细节属性配置
               commandProperties = {
                       @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "2000")
               }
       )
       @GetMapping("/checkStateTimeout/{userId}")
       public Integer findResumeOpenStateTimeout(@PathVariable Long userId){
           // 使用 Ribbon不需要自己获取获取服务实例然后选择一个访问
           String url = "http://turbo-service-resume/resume/openState/"+userId; // 指定服务名
           Integer forObject = restTemplate.getForObject(url, Integer.class);
           return forObject;
       }
       
       // http://localhost:8092/autodeliver/checkStateTimeoutFallback/2195320
       @HystrixCommand(
               threadPoolKey = "findResumeOpenStateTimeoutFallback",
               threadPoolProperties = {
                       @HystrixProperty(name = "coreSize",value = "1"), // 线程数
                       @HystrixProperty(name = "maxQueueSize",value = "20") //等待队列长度
               },
               // commandProperties熔断的一些细节属性配置
               // com.netflix.hystrix.contrib.javanica.conf.HystrixPropertiesManager
               commandProperties = {
                       @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds",value = "2000"),
                       // hystrix 高级配置，定制工作过程细节
                       // 统计时间窗口定义
                       @HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds",value = "8000"),
                       // 统计时间窗口内的最小请求数
                       @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold",value = "2"),
                       // 统计时间窗口内的错误数量百分比阈值
                       @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage",value ="50" ),
                       // 自我修复时的活动窗口长度
                       @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds",value = "3000")
               },fallbackMethod = "myFallback" // 回退方法
       )
       @GetMapping("/checkStateTimeoutFallback/{userId}")
       public Integer findResumeOpenStateTimeoutFallback(@PathVariable Long userId){
           // 使用 Ribbon不需要自己获取获取服务实例然后选择一个访问
           String url = "http://turbo-service-resume/resume/openState/"+userId; // 指定服务名
           Integer forObject = restTemplate.getForObject(url, Integer.class);
           return forObject;
       }
   
       /**
        * 定义回退方法，返回预设默认值
        * 注意：该方法形参和返回值与原始方法保持一致
        * @param userId
        * @return
        */
       public Integer myFallback(Long userId){
           return -1;
       }
   }
   ```

   **注意**：

   - 降级（兜底）方法必须和被降级方法相同的方法签名（相同参数列表、相同返回值）

4. 可以在类上使用 `@DefaultProperties` 注解统一指定整个类中公用的降级（兜底方法）。

5. 服务提供者模拟请求超时（线程休眠3s），只修改 8080 实例，对比观察

![hystrix-timeout](assest/hystrix-timeout.gif)

![hystrix-timeout-fallback](assest/hystrix-timeout-fallback.gif)

# 5 Hystrix 舱壁模式（线程池隔离策略）

为了避免问题服务请求过多导致正常服务无法访问，Hystrix 不是采用增加线程数，而是单独的为每一个控制方法创建一个线程池的方式，这种模式叫做 “舱壁模式”，也是线程隔离的手段。

# 6 Hystrix 工作流程与高级应用

# 7 Hystrix Dashboard 断路监控仪表盘

# 8 Hystrix Turbine 聚合监控

# 9 Hystrix 核心源码剖析