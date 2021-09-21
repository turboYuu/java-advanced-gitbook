第二部分 JUC

# 1 并发容器

## 1.1 BlockingQueue

### 1.1.1 ArrayBlockingQueue

### 1.1.2 LinkedBlockingQueue

### 1.1.3 PriorityBlockingQueue

### 1.1.4 DelayQueue

### 1.1.5 SynchronousQueue

## 1.2 BlockingDeque

## 1.3 CopyOnWrite

### 1.3.1 CopyOnWriteArrayList

### 1.3.2 CopyOnWriteArraySet

## 1.4 ConcurrentLinkQueue/Deque

## 1.5 ConcurrentHashMap

## 1.6 ConcurrentSkipListMap/Set

### 1.6.1 ConcurrentSkipListMap

### 1.6.2 ConcurrentSkipListSet

# 2 同步工具

## 2.1 Semaphore

## 2.2 CountDownLatch

### 2.2.1 CountDownLatch使用场景

### 2.2.2 await()实现分析

### 2.2.3 countDown()实现分析

## 2.3 CyclicBarrier

### 2.3.1 CyclicBarrier使用场景

### 2.3.2 CyclicBarrier实现原理

## 2.4 Exchanger

### 2.4.1 使用场景

### 2.4.2 实现原理

### 2.4.3 exchange(V x)实现分析

## 2.5 Phaser

### 2.5.1 用Phaser替代CyclicBarrier和CountDownLatch

### 2.5.2 Phaser新特性

### 2.5.3 state变量解析

### 2.5.4 阻塞与唤醒（Treiber Stack）

### 2.5.5. arrive()方法分析

### 2.5.6 awaitAdvance()方法分析

# 3 Atomic类

## 3.1 AtomicInteger和AtomicLong

## 3.2 AtomicBoolean和AtomicReference

## 3.3 AtomicStampedReference和AtomicMarkableReference

## 3.4 AtomicIntegerFieldUpdater、AtomicLongFieldUpdate和AtomicReferenceFieldUpdater

## 3.5 AtomicIntegerArray、AtomicLongArray和AtomicReferenceArray

## 3.6 Striped64与LongAdder

# 4 Lock与Condition

## 4.1 互斥锁

## 4.2 读写锁

## 4.3 Condition

## 4.4 StampedLock