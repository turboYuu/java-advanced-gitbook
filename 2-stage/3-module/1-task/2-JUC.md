第二部分 JUC

# 5 并发容器

## 5.1 BlockingQueue

### 5.1.1 ArrayBlockingQueue

### 5.1.2 LinkedBlockingQueue

### 5.1.3 PriorityBlockingQueue

### 5.1.4 DelayQueue

### 5.1.5 SynchronousQueue

## 5.2 BlockingDeque

## 5.3 CopyOnWrite

### 5.3.1 CopyOnWriteArrayList

### 5.3.2 CopyOnWriteArraySet

## 5.4 ConcurrentLinkQueue/Deque

## 5.5 ConcurrentHashMap

## 5.6 ConcurrentSkipListMap/Set

### 5.6.1 ConcurrentSkipListMap

### 5.6.2 ConcurrentSkipListSet

# 6 同步工具

## 6.1 Semaphore

## 6.2 CountDownLatch

### 6.2.1 CountDownLatch使用场景

### 6.2.2 await()实现分析

### 6.2.3 countDown()实现分析

## 6.3 CyclicBarrier

### 6.3.1 CyclicBarrier使用场景

### 6.3.2 CyclicBarrier实现原理

## 6.4 Exchanger

### 6.4.1 使用场景

### 6.4.2 实现原理

### 6.4.3 exchange(V x)实现分析

## 6.5 Phaser

### 6.5.1 用Phaser替代CyclicBarrier和CountDownLatch

### 6.5.2 Phaser新特性

### 6.5.3 state变量解析

### 6.5.4 阻塞与唤醒（Treiber Stack）

### 6.5.5. arrive()方法分析

### 6.5.6 awaitAdvance()方法分析

# 7 Atomic类

## 7.1 AtomicInteger和AtomicLong

## 7.2 AtomicBoolean和AtomicReference

## 7.3 AtomicStampedReference和AtomicMarkableReference

## 7.4 AtomicIntegerFieldUpdater、AtomicLongFieldUpdate和AtomicReferenceFieldUpdater

## 7.5 AtomicIntegerArray、AtomicLongArray和AtomicReferenceArray

## 7.6 Striped64与LongAdder

# 8 Lock与Condition

## 8.1 互斥锁

## 8.2 读写锁

## 8.3 Condition

## 8.4 StampedLock