第四部分 ForkJoinPool

# 15 ForkJoinPool用法

ForkJoinPool就是JDK7提供的一种“分治算法”的多线程并行计算框架。Fork意为分叉，Join意为合并，一分一和，相互配合，形成分治算法。此外，也可以将ForkJoinPool看作一个单机版的Map/Reduce，多个线程并行计算。

相比于ThreadPoolExecutor，ForkJoinPool可以更好的实现计算的负载均衡，提高资源利用率。

利用ForkJoinPool，可以把大的任务拆分成很多小任务，然后这些小任务被所有的线程执行，从而实现任务计算的负载均衡。

# 16 核心数据结构

# 17 工作窃取队列

# 18 ForkJoinPool状态控制

## 18.1 状态变量ctl解析

## 18.2 阻塞栈Treiber Stack

## 18.3 ctl变量的初始值

## 18.4 ForkJoinWorkerThread状态与个数分析

# 19 Worker线程的阻塞-唤醒机制

## 19.1 阻塞-入栈

## 19.2 唤醒-出栈

# 20 任务的提交过程分析

## 20.1 内部提交任务push

## 20.2 外部提交任务

# 21 工作窃取算法：任务的执行过程分析

# 22 ForkJoinTask的fork/join

## 22.1 fork

## 22.2 join的嵌套

# 23 ForkJoinPool的优雅关闭

## 23.1 工作线程的退出

## 23.2 shutdown()与shutdownNow的区别