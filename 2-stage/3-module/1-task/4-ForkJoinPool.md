第四部分 ForkJoinPool

# 1 ForkJoinPool用法

ForkJoinPool就是JDK7提供的一种“分治算法”的多线程并行计算框架。Fork意为分叉，Join意为合并，一分一和，相互配合，形成分治算法。此外，也可以将ForkJoinPool看作一个单机版的Map/Reduce，多个线程并行计算。

相比于ThreadPoolExecutor，ForkJoinPool可以更好的实现计算的负载均衡，提高资源利用率。

利用ForkJoinPool，可以把大的任务拆分成很多小任务，然后这些小任务被所有的线程执行，从而实现任务计算的负载均衡。

# 2 核心数据结构

# 3 工作窃取队列

