第三部分 线程池与Future

# 1 线程池的实现原理

下图所示为线程池的实现原理：调用方不断地向线程池中提交任务；线程池中有一组线程，不断地从队列中取任务，这是一个典型地生产者-消费者模型。

![image-20210921193536503](assest/image-20210921193536503.png)

要实现这样一个线程池，有几个问题需要考虑：



# 2 线程池的类继承体系

![image-20210921194814218](assest/image-20210921194814218.png)

# 3 ThreadPoolExecutor

## 3.1 核心数据结构

## 3.2 核心配置参数解释

## 3.3 线程池的优雅关闭

## 3.4 任务的提交过程分析

## 3.5 任务的执行过程分析

## 3.6 线程池的4中拒绝策略

# 4 Executors工具类

## 4.1 四种对比

## 4.2 最佳实践

# 5 ScheduledThreadPoolExecutor

## 5.1 延迟执行和周期性执行的原理

## 5.2 延迟执行

## 5.3 周期性执行

# 6 CompletableFuture用法

## 6.1 runAsync与supplyAsync

## 6.2 thenRun、thenAccept和thenApply

## 6.3 thenCompose与thenCombine

## 6.4 任意个ConpletableFuture的组合

## 6.5 四种任务原型

## 6.6. CompletionStage接口

## 6.7 CompletableFuture内部原理

## 6.8 任务的网状执行：有向无环图

## 6.9 allOf内部的计算图分析