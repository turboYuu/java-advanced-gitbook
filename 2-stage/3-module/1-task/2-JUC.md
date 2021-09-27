第二部分 JUC

# 5 并发容器

## 5.1 BlockingQueue

在所有的并发容器中，BlockingQueue是最常见的一种。BlockingQueue是一个带有阻塞功能的队列，当入队时，若队列已满，则阻塞调用者；当出队，队列空，则阻塞调用者。

在Concurrent包中，BlockingQueue是一个接口，有许多个不同的实现类，如图所示。

![image-20210927111427901](assest/image-20210927111427901.png)

接口定义：

```java
public interface BlockingQueue<E> extends Queue<E> {    
	//...
   	boolean add(E e);    
   	boolean offer(E e);
   	void put(E e) throws InterruptedException;    
   	boolean remove(Object o);
   	E take() throws InterruptedException;
   	E poll(long timeout, TimeUnit unit) throws InterruptedException;    
   	//...
}
```

该接口和JDK集合包中的Queue接口是兼容的，同时在其基础上增加了阻塞功能。入队提供了add(...)、offer(...)、put(...) 3个方法，区别：add(...)和offer(...)的返回值是布尔类型，而put无返回值，还会抛出中断异常。所以add(...)和offer(...)是无阻塞的，也就是Queue本身定义的接口，而put(...)是阻塞的。add(...)和offer(...)的区别不大，当队列满时，前者会排除异常，后者直接返回false。

出队列与之类似，提供了remove()、poll()、take()等方法，remove()是非阻塞式的，take()和poll()式阻塞式的。

### 5.1.1 ArrayBlockingQueue

ArrayBlockingQueue是一个用数组实现的环形队列，在构造方法中，会传入数组的容量。

```java
public ArrayBlockingQueue(int capacity) {    
	this(capacity, false);
}
public ArrayBlockingQueue(int capacity, boolean fair) { 
	// ...
}
public ArrayBlockingQueue(int capacity, boolean fair, Collection<? extends E> c) {
   	this(capacity, fair);    
   	// ...
}
```

其核心数据结构如下：

```java
public class ArrayBlockingQueue<E> extends AbstractQueue<E> 
		implements BlockingQueue<E>, java.io.Serializable {

	//...
   	final Object[] items;    
   	// 队头指针
   	int takeIndex;    
   	// 队尾指针
   	int putIndex;    
   	int count;
   	
   	// 核心为1个锁外加两个条件    
   	final ReentrantLock lock;
   	private final Condition notEmpty;    
   	private final Condition notFull;    
   	//...
}
```

其put/take方法也很简单，如下所示：

put方法：

![image-20210927113351092](assest/image-20210927113351092.png)

![image-20210927113526204](assest/image-20210927113526204.png)

take方法：

![image-20210927124114130](assest/image-20210927124114130.png)

![image-20210927124227070](assest/image-20210927124227070.png)

### 5.1.2 LinkedBlockingQueue

LinkedBlockingQueue是一种基于单向链表的阻塞队列。因为队头和队尾是2个指针分开操作的，所以用了2把锁+2个条件，同时有1个AutomicInteger的原子变量记录count数。

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E> 
		implements BlockingQueue<E>, java.io.Serializable {
   
   	// ...
   	private final int capacity;    
   	// 原子变量
   	private final AtomicInteger count = new AtomicInteger(0);    
   	// 单向链表的头部
   	private transient Node<E> head;    
   	// 单向链表的尾部
   	private transient Node<E> last;    
   	
   	// 两把锁，两个条件
   	private final ReentrantLock takeLock = new ReentrantLock();
   	private final Condition notEmpty = takeLock.newCondition();
   	private final ReentrantLock putLock = new ReentrantLock();
   	private final Condition notFUll = putLock.newCondition();    
   	// ...
}
```

在其构造方法中，也可以指定队列的总容量。如果不指定，默认为Integer.MAX_VALUE。

![image-20210927125307508](assest/image-20210927125307508.png)

put方法：

![image-20210927125553832](assest/image-20210927125553832.png)

take方法：

![image-20210927125831595](assest/image-20210927125831595.png)

LinkBlockingQueue和ArrayBlockingQueue的差异：

1. 为了提高并发度，用了2把锁，分别控制对头，队尾的操作。意味着在put(...)和put(...)之间、take(...)和take(...)之间是互斥的，put(...)和take(...)之间并不互斥。但对于count变量，双方都需要操作，所以必须是原子类型。
2. 因为各自拿了一把锁，所以当需要调用对方的condition的signal时，还必须再加上对方的锁，就是signalNotEmpty()和signalNotFull()方法，如下：

![image-20210927131429252](assest/image-20210927131429252.png)

3. 不仅put会通知take，take也会通知put。当put发现非满的时候，也会通知其他put线程；当take发现非空时，也会通知其他take线程。



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