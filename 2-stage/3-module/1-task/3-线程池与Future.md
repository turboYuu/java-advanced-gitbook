第三部分 线程池与Future

# 9 线程池的实现原理

下图所示为线程池的实现原理：调用方不断地向线程池中提交任务；线程池中有一组线程，不断地从队列中取任务，这是一个典型地生产者-消费者模型。

![image-20210921193536503](assest/image-20210921193536503.png)

要实现这样一个线程池，有几个问题需要考虑：



# 10 线程池的类继承体系

![image-20210921194814218](assest/image-20210921194814218.png)

# 11 ThreadPoolExecutor

## 11.1 核心数据结构

```java
public class ThreadPoolExecutor extends AbstractExecutorService {    
	//...
	private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));    
	// 存放任务的阻塞队列
   	private final BlockingQueue<Runnable> workQueue;    
   	// 对线程池内部各种变量进行互斥访问控制
   	private final ReentrantLock mainLock = new ReentrantLock();    
   	// 线程集合
   	private final HashSet<Worker> workers = new HashSet<Worker>();    
   	//...
}
```



## 11.2 核心配置参数解释

![image-20210921200342643](assest/image-20210921200342643.png)

上面各个参数：



## 11.3 线程池的优雅关闭

![image-20210923133944932](assest/image-20210923133944932.png)

## 11.4 任务的提交过程分析

提交任务的方法如下：

```

```



## 11.5 任务的执行过程分析

## 11.6 线程池的4中拒绝策略

在execute(Runnable command)的最后，调用reject(command)执行拒绝策略，代码如下：

![image-20210923105445356](assest/image-20210923105445356.png)

![image-20210923105550124](assest/image-20210923105550124.png)

handler就是可以设置的拒绝策略管理器：

![image-20210923105639572](assest/image-20210923105639572.png)

RejectedExecutionHandler是一个接口，定义了四种实现，分别对应四种不同放入拒绝策略，默认是：`AbortPolicy`

![image-20210923110451253](assest/image-20210923110451253.png)

ThreadPoolExecutor类中默认的实现是：

![image-20210923112531561](assest/image-20210923112531561.png)

![image-20210923112624544](assest/image-20210923112624544.png)

四种策略的实现代码如下：

### 11.6.1 策略一 CallerRunsPolicy

调用者直接在自己的线程里执行，线程池不处理。

![image-20210923113121884](assest/image-20210923113121884.png)

### 11.6.2 策略二

线程池抛异常：AbortPolicy

![image-20210923113321852](assest/image-20210923113321852.png)

### 11.6.3 策略三

线程池直接丢任务，神不知鬼不觉：

![image-20210923113437460](assest/image-20210923113437460.png)

### 11.6.4 策略四

![image-20210923113533985](assest/image-20210923113533985.png)





示例程序：

```java
package com.turbo.concurrent.demo;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class ThreadPoolExecutorDemo {

    public static void main(String[] args) {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                3,
                5,
                1,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(3),
                //new ThreadPoolExecutor.AbortPolicy()
                //new ThreadPoolExecutor.CallerRunsPolicy()
                //new ThreadPoolExecutor.DiscardPolicy()
                new ThreadPoolExecutor.CallerRunsPolicy());


        for (int i = 0; i < 20; i++) {
            int finali = i;
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    System.out.println(Thread.currentThread().getId() + "["+finali+"] - 开始");
                    try {
                        Thread.sleep(5000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.println(Thread.currentThread().getId() + "["+finali+"] - 结束");
                }
            });
            try {
                Thread.sleep(200);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        executor.shutdown();
        boolean flag = true;

        try {
            do {
                flag = executor.awaitTermination(1,TimeUnit.SECONDS);

            }while (false);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("线程池关闭成功...");
        System.out.println(Thread.currentThread().getId());
    }
}
```



# 12 Executors工具类

## 12.1 四种对比

## 12.2 最佳实践

# 13 ScheduledThreadPoolExecutor

## 13.1 延迟执行和周期性执行的原理

## 13.2 延迟执行

## 13.3 周期性执行

# 14 CompletableFuture用法

## 14.1 runAsync与supplyAsync

## 14.2 thenRun、thenAccept和thenApply

## 14.3 thenCompose与thenCombine

## 14.4 任意个ConpletableFuture的组合

## 14.5 四种任务原型

| 四种任务原型 | 无参数                                            | 有参数                                     |
| ------------ | ------------------------------------------------- | ------------------------------------------ |
| 无返回值     | Runnable接口<br>对应的提交方法：runAsync，thenRun | Consumer接口<br>对应的提交方法：thenAccept |
| 有返回值     | Supplier接口：<br>对应的提交方法：supplierAsync   | Function接口<br>对应的提交方法：thenApply  |



## 14.6. CompletionStage接口

## 14.7 CompletableFuture内部原理



![image-20210923172826831](assest/image-20210923172826831.png)

![image-20210923181138791](assest/image-20210923181138791.png)

![image-20210923181903255](assest/image-20210923181903255.png)







## 14.8 任务的网状执行：有向无环图

## 14.9 allOf内部的计算图分析