第二部分 JUC

Java并发编程核心在于java.util.concurrent包，而JUC当中的大多数同步器实现都是围绕着共同的基础行为，比如等待队列，条件队列、独占获取、共享获取等。而这个行为的抽象就是基于AbstractQueuedSynchronizer，简称AQS，AQS定义了一套多线程访问共享资源的同步框架，是一个依赖状态（<font color='red'>**state**</font>）的同步器。

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

队列通常是先进先出的，而PriorityQueue是按照元素的优先级从小到大出队列。因此，PriorityQueue中的2个元素之间需要比较大小，并实现Comparable接口。



其核心数据结构如下：

```java
public class PriorityBlockingQueue<E> extends AbstractQueue<E> 
		implements BlockingQueue<E>, java.io.Serializable {
		
   	//...
   	// 用数组实现的二插小根堆
   	private transient Object[] queue;
   	private transient int size;
   	private transient Comparator<? super E> comparator;    
   	
   	// 1个锁+一个条件，没有非满条件
   	private final ReentrantLock lock;    
   	private final Condition notEmpty;    
   	//...
}
```

其构造方法如下，如果不指定初始大小，内部会设定一个默认值11，当元素个数超过这个大小后，会自动扩容。

![image-20210927133243954](assest/image-20210927133243954.png)



put方法：

![image-20210927133411321](assest/image-20210927133411321.png)

![image-20210927134118679](assest/image-20210927134118679.png)



take方法：

![image-20210927134336216](assest/image-20210927134336216.png)

![image-20210927134739716](assest/image-20210927134739716.png)

从上面可以看出，在阻塞的实现方面，和ArrayBlockingQueue的机制相似，主要区别是用数组实现了一个二叉堆，从而实现按优先级从小到大出队列。另一个区别是没有notFull条件，当元素个数超出数组长度时，执行扩容操作。

### 5.1.4 DelayQueue

DelayQueue 延迟队列，是一个按照延迟时间从小到大出队列的PriorityQueue。所谓延迟时间，就是“未来将要执行的时间”减去“当前时间”。为此，放入DelayQueue中的元素，必须实现Delayed接口，如下：

![image-20210927135610705](assest/image-20210927135610705.png)

![image-20210927135644945](assest/image-20210927135644945.png)

关于接口：

1. 如果getDelay返回值小于等于0，则说明该元素到期，需要从队列中拿出来执行。
2. 该接口首先继承了Comparable接口，所以要实现该接口，必须实现Comparable接口。具体就是，基于getDelay()的返回值比较两个元素的大小。

下面看一下DelayQueue的核心数据结构。

```java
public class DelayQueue<E extends Delayed> extends AbstractQueue<E> 
		implements BlockingQueue<E> {
		
   	// ...
   	// 一把锁和一个非空条件
   	private final transient ReentrantLock lock = new ReentrantLock();    
   	private final Condition available = lock.newCondition();
   	// 优先级队列
   	private final PriorityQueue<E> q = new PriorityQueue<E>(); 
   	// ...
}
```

下面介绍put/take的实现，先从take说起，因为这样更能看出DelayQueue特性。

**take方法：**

![image-20210927141101250](assest/image-20210927141101250.png)

关于take()方法：

1. 不同于一般的阻塞队列，只在队列为空的时候，才阻塞。如果堆顶元素的延迟时间没到，也会阻塞。
2. 在上面的代码中使用了一个优化技术，用一个Thread leader变量记录了等待堆顶元素的第一个线程，为什么这样做？通过getDelay(...)可以知道堆顶元素何时到期，不必无限等待，可以使用condition.awaitNanos()等待一个有限时间；只有当发现还有其他线程也在等待堆顶元素（leader != NULL）时，才需要无限期等待。

**put的实现：**

![image-20210927143023061](assest/image-20210927143023061.png)

![image-20210927143329470](assest/image-20210927143329470.png)

注意：不是每放入一个元素，都需要通知等待的线程。放入的元素，如果其延迟时间大于当前堆顶的元素延迟时间，就没有必要通知等待的线程；只有当延迟时间是最小的，在堆顶时，才有必要通知等待的线程，也就是上面代码中的`if (q.peek() == e) {`部分。

### 5.1.5 SynchronousQueue

SynchronousQueue是一种特殊的BlockingQueue，它本身没有容量。先调用put(...)，线程会阻塞；直到另外一个线程调用了take()，连个线程才同时解锁，反之亦然。对于多个线程而言，例如3个线程，调用3次put(...)，3个线程都会阻塞；直到另外的线程调用3次take()，6个线程才同时解锁，反之亦然。



SynchronousQueue的实现，构造方法：

![image-20210927144531529](assest/image-20210927144531529.png)

和锁一样，也有公平和非公平模式。如果是公平模式，则用TransferQueue实现；如果是非公平模式，则用TransferStack实现。这两个类分别是什么？先看一下put/take的实现。

![image-20210927144812507](assest/image-20210927144812507.png)

![image-20210927144845606](assest/image-20210927144845606.png)

可以看到，put/take都调用了transfer(...)接口。而TransferQueue和TransferStack分别实现了这个接口。该接口在SynchronousQueue内部，如下。如果是put(...)，则第一个参数就是对应的元素；如果是take(...)，则第一个参数为null，后面两个参数分别为是否设置超时和对应的超时时间。

![image-20210927145739165](assest/image-20210927145739165.png)



接下来看一下什么是公平模式和非公平模式。假设3个线程分别调用了put(...)，3个线程会进入阻塞状态，直到其他线程调用3次take()，和3个put() 一一配对。

如果是**公平模式（队列模式）**，则第一个调用put(...)的线程1会在队列头部，第1个到来的take()线程和它进行配对，遵循先到先匹配的原则，所以是公平的；如果是**非公平模式（栈模式）**，则第3个调用put(...)的线程3会在栈顶，第1个到来的take()线程和它进行配对，遵行的是后到先配对的原则，所以是非公平的。

![image-20210927151008392](assest/image-20210927151008392.png)

TransferQueue和TransferStack的实现。

**1.TransferQueue：**

```java
public class SynchronousQueue<E> extends AbstractQueue<E> 
		implements BlockingQueue<E>, java.io.Serializable {
		
	// ...
   	static final class TransferQueue<E> extends Transferer<E> {        
   		static final class QNode {
			volatile QNode next;
           	volatile Object item;
           	volatile Thread waiter;            
           	final boolean isData;            
           	//...
     	}
       	transient volatile QNode head;        
       	transient volatile QNode tail;        
       	// ...
	} 
}
```

从上面的代码可以看出，TransferQueue是一个基于单向链表而实现的队列，通过head和tail 2指针记录头部和尾部。初始的时候，head和tail会指向一个空节点，构造方法：

![image-20210927152025381](assest/image-20210927152025381.png)

> 阶段(a)：队列是一个空的节点，head/tail都指向这个空节点。
>
> 阶段(b)：3个线程分别调用put，生成3个QNode，进入队列。
>
> 阶段(c)：来了一个线程调用take，会和队列头部的第一个QNode进行配对。
>
> 阶段(d)：第1个QNode出队列。



![image-20210927152406268](assest/image-20210927152406268.png)

这里有一个关键点：put节点和take节点节点一旦相遇，就会配对出队列，所以在队列中不可能同时存在put节点和take节点，要么全是put节点，要么全是take节点。

TransferQueue的代码实现：

![image-20210927153627629](assest/image-20210927153627629.png)

![image-20210927154245207](assest/image-20210927154245207.png)

![image-20210927154556764](assest/image-20210927154556764.png)



整个for循环有两个大的if-else分支，如果当前线程和队列中的元素是同一模式（都是put节点或者take节点），则与当前线程对应的节点被加入队列尾部并且阻塞；如果不是同一种模式，则选取队列头部的第1个元素进行配对。

这里的配对就是m.casltem(x, e)，把自己的item x换成对方的item e，如果CAS操作成功，则配对成功。如果是put节点，则isData=true，item != null；如果是take节点，则isData = false，item = null。如果CAS操作不成功，则isData和item之间将不一致，也就是isData != (x != null)，通过这个条件可以判断节点是否已经被匹配过了。



**2.TransferStack：**

TransferStack的定义如下，首先，它也是一个单向链表。不同于队列，只需要head指针就能实现入栈和出栈操作。

```java
static final class TransferStack extends Transferer {
   	static final int REQUEST = 0;
   	static final int DATA = 1;
   	static final int FULFILLING = 2;
   	static final class SNode {
       	volatile SNode next;     // 单向链表
       	volatile SNode match;    // 配对的节点
       	volatile Thread waiter;  // 对应的阻塞线程        
       	Object item;
       	int mode;                // 三种模式        
       	//...
 	}
   	volatile SNode head; 
}
```

链表中的节点有三种状态，REQUEST对应take节点，DATA对应put节点，二者配对之后，会生成一个FULLFILLING节点，入栈，然后FULLING节点和被配对的节点一起出栈。

> 阶段(a)：head指向NULL。不同于TransferQueue，这里没有空的头节点。
>
> 阶段(b)：3个线程调用3次put，依次入栈。
>
> 阶段(c)：线程4调用take，和栈顶的第一个元素配对，生成FULLFILLING节点，入栈。
>
> 阶段(d)：栈顶的2个元素同时出栈。

![image-20210927164448155](assest/image-20210927164448155.png)

具体代码实现：

![image-20210927165753844](assest/image-20210927165753844.png)

![image-20210927170315483](assest/image-20210927170315483.png)



## 5.2 BlockingDeque

BlockingQueue定义了一个阻塞的双端队列接口，如下：

```java
public interface BlockingDeque<E> extends BlockingQueue<E>, Deque<E> {    
	void putFirst(E e) throws InterruptedException;
   	void putLast(E e) throws InterruptedException;    
   	E takeFirst() throws InterruptedException;
   	E takeLast() throws InterruptedException;    
   	// ...
}
```

该接口继承了BlockingQueue接口，同时增加了对应的双端队列操作接口。该接口只有一个实现，就是LinkedBlockingDeque。

核心数据结构如下：是一个双向链表。

```java
public class LinkedBlockingDeque<E> extends AbstractQueue<E> 
		implements BlockingDeque<E>, java.io.Serializable {
   
   	static final class Node<E> {        
   		E item;
       	Node<E> prev;  // 双向链表的Node        
       	Node<E> next;Node(E x) {            
       		item = x;      
		}
	}
	
   	transient Node<E> first;  // 队列的头和尾    
   	transient Node<E> last;
   	private transient int count; // 元素个数    
   	private final int capacity;  // 容量    
   	
   	// 一把锁+两个条件
   	final ReentrantLock lock = new ReentrantLock();
   	private final Condition notEmpty = lock.netCondition();    
   	private final Condition notFull = lock.newCondition();    
   	// ...
}
```

对应的实现原理，和LinkedBlockingQueue基本一样，只是LinkedBlockingQueue是单向链表，而LinkedBlockingDeque是双向链表。

take/put：

![image-20210927171439946](assest/image-20210927171439946.png)

![image-20210927171514668](assest/image-20210927171514668.png)

## 5.3 CopyOnWrite

CopyOnWrite指在“写”的时候，不直接“写”源数据，而是把数据拷贝一份进行修改，在通过悲观锁或者乐观锁的方式写回。

拷贝一份再修改，是为了在“读”的时候不加锁。

### 5.3.1 CopyOnWriteArrayList

和ArrayList一样，CopyOnWriteArrayList的核心数据结构也是一个数组，代码如下：

```java
public class CopyOnWriteArrayList<E> 
		implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
   
   	// ...
   	private volatile transient Object[] array; 
}
```

下面是CopyOnWriteArrayList的几个“读”方法：

```java
	final Object[] getArray() {        
		return array;
	}
	//
	public E get(int index) {
		return elementAt(getArray(), index);  
	}
   	public boolean isEmpty() {        
   		return size() == 0;  
	}
   	public boolean contains(Object o) {        
   		return indexOf(o) >= 0;
 	}
   	public int indexOf(Object o) {        
   		Object[] es = getArray();
       	return indexOfRange(o, es, 0, es.length);  
	}
   	private static int indexOfRange(Object o, Object[] es, int from, int to){        
   		if (o == null) {
           	for (int i = from; i < to; i++)                
           		if (es[i] == null)
                   	return i;        
		} else {
           	for (int i = from; i < to; i++)                
           		if (o.equals(es[i]))
                   	return i;      
		}
       	return -1;  
	}
```

这些“读”方法都没有加锁，如何保证"线程安全"？答案在”写“方法中。

```java
public class CopyOnWriteArrayList<E>
   		implements List<E>, RandomAccess, Cloneable, java.io.Serializable {    
   	
   	// 锁对象
   	final transient Object lock = new Object();  
    
   	public boolean add(E e) {
       	synchronized (lock) { // 同步锁对象            
       		Object[] es = getArray();
           	int len = es.length;
           	es = Arrays.copyOf(es, len + 1); // CopyOnWrite，写的时候，先拷贝一 份之前的数组。
           	es[len] = e;            
           	setArray(es);            
           	return true;      
		}
 	}
 	
   	public void add(int index, E element) {        
   		synchronized (lock) { // 同步锁对象            
   			Object[] es = getArray();
           	int len = es.length;
           	if (index > len || index < 0)
           		throw new IndexOutOfBoundsException(outOfBounds(index, len));
           	Object[] newElements;
           	int numMoved = len - index;            
           	if (numMoved == 0){
               	newElements = Arrays.copyOf(es, len + 1);            
			} else {
               	newElements = new Object[len + 1];
               	System.arraycopy(es, 0, newElements, 0, index); // CopyOnWrite，写的时候，先拷贝一份之前的数组。
               	System.arraycopy(es, index, newElements, index + 1, numMoved);
       		}
           	newElements[index] = element;
           	setArray(newElements); // 把新数组赋值给老数组      
		}
	} 
}
```

其他”写“方法，例如remove和add类似。

### 5.3.2 CopyOnWriteArraySet

CopyOnWriteArraySet就是用Array实现的一个Set，保证所有元素都不重复。其内部是封装的一个CopyOnWriteArrayList。

```java
public class CopyOnWriteArraySet<E> extends AbstractSet<E> 
		implements java.io.Serializable {
   	// 新封装的CopyOnWriteArrayList
	private final CopyOnWriteArrayList<E> al;
	public CopyOnWriteArraySet() {
		al = new CopyOnWriteArrayList<E>();  
	}
   	public boolean add(E e) {
       	return al.addIfAbsent(e); // 不重复的加进去  
	}
}
```



## 5.4 ConcurrentLinkedQueue/Deque

AQS（AbstractQueuedSynchronizer）内部的阻塞队列实现原理：基于双向链表，通过对head/tail进行CAS操作，实现入队和出队。

ConcurrentLinkedQueue的实现原理和AQS内部的阻塞队列类似：同样是基于CAS，同样是通过head/tail指针记录队列头部和尾部，但是有稍许差别。

首先，它是一个单向链表，定义：

```java
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E> 
		implements Queue<E>, java.io.Serializable {
	
	private static class Node<E> {        
		volatile E item;
       	volatile Node<E> next;        
       	//...
 	}
   	private transient volatile Node<E> head;    
   	private transient volatile Node<E> tail;    
   	//...
}
```

其次，在AQS的阻塞队列中，每次入队后，tail一定会后移一个位置；每次出队，head一定后移一个位置，保证head指向队列头部，tail指向链表尾部。

但在ConcurrentLinkedQueue中，head/tail的更新可能落后于节点的入队和出队，因为它不是直接对head/tail指针进行CAS操作的，而是对Node中的item进行操作。分析如下：

**1.初始化**

初始的时候，`head`和`tail`都指向一个`null`节点，对应代码如下：

```java
public ConcurrentLinkedQueue() {
	head = tail = new Node<E>();
}
```

![image-20210927205638192](assest/image-20210927205638192.png)

**2.入队列**

代码如下：

```java
public boolean offer(E e) {
    final Node<E> newNode = new Node<E>(Objects.requireNonNull(e));

    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            if (NEXT.compareAndSet(p, null, newNode)) {
                if (p != t) // hop two nodes at a time; failure is OK
                    TAIL.weakCompareAndSet(this, t, newNode);
                return true;
            }
        }
        else if (p == q)
            p = (t != (t = tail)) ? t : head;
        else
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

![image-20210927210313970](assest/image-20210927210313970.png)

上面的入队，其实是每次在队尾追加2个节点时，才移动一次tail节点，如下：

初始化的时候，队列中有1个节点item1，tail指向该节点，假设线程1要入队item2节点：

step1：p=tail, q=p.next=null

step2：对p的next执行CAS操作，追加item2，成功后，p=tail。所以上面的casTail方法不会执行，直接返回。此时tail指针没有变化。

![image-20210928142848219](assest/image-20210928142848219.png)

之后，假设线程2要入队item3节点，如下：

step3：p=tail，q=p.next

step4：q != null，因此不会入队新节点。p，q都后移一位。

step5：q = null ，对p的next执行CAS操作，入队item3节点。

step6：p != t，满足条件，执行上面的casTail操作，tail后移2个位置，到达队列尾部。

![image-20210928143455076](assest/image-20210928143455076.png)

最后总结以下入队列的两个关键点：

1. 即使tail指针没有移动，只要对p的next指针成功进行CAS操作，就算成功入队列。
2. 只有当 p != tail 的时候，才会后移tail指针。也就是说，每连续追加2个节点，才后移1次tail指针。即使CAS失败也没关系，可以由下1个线程来移动tail指针。



**3.出队列**

上面说了入队列后，tail指针不变化，那是否会出现入队列之后，要出队列却没有元素可出的情况？

![image-20210928144418379](assest/image-20210928144418379.png)

出队列的代码和入队列类似，也有p、q两个指针，整个变化过程如下：，假设初始的时候head指向空节点，队列中有item1、item2、item3三个节点。

step1：p = head，q = p.next，p != q

step2：后移p指针，使得p=q

step3：出队列。关键点：此处并没有直接删除item1节点，只是把该节点的item通过CAS操作置为了NULL。

step4：p != head，此时队列中有了2个NULL节点，再次前移1次head指针，对其执行updateHead操作。



![image-20210928150007987](assest/image-20210928150007987.png)

最后总结一下出队列的关键点：

1. 出队列的判断并非观察tail指针的位置，而是依赖于head指针后续的节点是否为NULL这一条件。
2. 只要对节点的item执行CAS操作，置为NULL成功，则出队列成功，即使head指针没有成功移动，也可以下1个线程继续完成。



**4.队列判空**

因为head/tail并不是精确地指向队列头部和尾部，所以不能简单地通过比较head/tail指针来判断队列是否为空，而是需要从head指针开始遍历，找到第一个不为NULL的节点，如果找到，则队列不为空；如果找不到，则队列为空。代码如下：

![image-20210928151750417](assest/image-20210928151750417.png)

![image-20210928152104656](assest/image-20210928152104656.png)



## 5.5 ConcurrentHashMap

HashMap通常的实现方式是“**数组+链表**”，这种方式被称为“拉链法”。ConcurrentHashMap在这个基本原理之上进行了各种优化。

首先是所有数据都放在一个大的HashMap中；其次是**引入了红黑树**。

![image-20210928152653371](assest/image-20210928152653371.png)

如果头节点是Node类型，则尾随它的就是一个普通的链表；如果头节点是TreeNode类型，它的后面就是一个红黑树，TreeNode是Node的子类。

链表和红黑树之间可以相互转换：初始的时候是链表，当链表中的元素超过某个阈值时，把链表转换成红黑树；反之，当红黑树中的元素个数小于某个阈值时，在转换为链表。

为什么要做这种设计？

1. 使用红黑树，当一个槽里有很多元素时，其查询和更新速度会比链表快很多，Hash冲突的问题由此得到较好的解决。
2. 加锁的粒度，并非整个ConcurrentHashMap，而是对每个头节点分别加锁，即并发度，就是Node数组的长度，初始长度为16。
3. 并发扩容，这是难度最大的。当一个线程要扩容Node数组的时候，其他线程还要读写，因此处理过程很复杂，

上述对比可以总结出：这种设计一方面降低了Hash冲突，另一方面也提升了并发度。



下面从构造方法开始，一步步深入分析其实现过程：

### 5.5.1 构造方法分析

![image-20210928154358999](assest/image-20210928154358999.png)

![image-20210928154433430](assest/image-20210928154433430.png)

在上面的代码中，变量cap就是Node数组的长度，保持2的整数次方。tableSizeFor(...)方法是根据传入的初始容量，计算出一个合适的数组长度。具体而言：1.5倍的初始容量+1，再往上取最接近的2的整数次方，作为数组长度cap的初始值。

这里的sizeCtl，其含义是用于控制在初始化或者并发扩容时候的线程数，只不过其初始值设置成cap。



### 5.5.2 初始化

在上面的构造方法里只计算了数组的初始大小，并没有对数组进行初始化。当多个线程都往里面放入元素的时候，再进行初始化。这就存在一个问题：多个线程重复初始化。那看一下是如何处理的。

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // 自旋等待
        else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) { //重点：将sizeCtl设置为-1
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n]; // 初始化
                    table = tab = nt;
                    // sizeCtl不是数组长度，因此初始化成功后，就不再等于数组长度
                    // 而是n-(n>>>2)=0.75n，表示下一次扩容的阈值：n-n/4
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;// 设置sizeCtl的值为sc。
            }
            break;
        }
    }
    return tab;
}
```

通过上面的代码可以看到，uoge线程的竞争是通过对sizeCtl进行CAS操作实现的。如果某个线程成功地把sizeCtl设置为-1，它就拥有了初始化的权利，进入初始化的代码模块，等到初始化完成，再把sizeCtl设置回去；其他线程则一直执行while循环，自旋等待，直到数组不为null，即当初始化结束时，退出整个方法。

因为初始化的工作量很小，所以此处选择的策略是让其他线程一直等待，而没有帮助其初始化。



### 5.5.3 put(...)实现分析

![image-20210928162417972](assest/image-20210928162417972.png)

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh; K fk; V fv;
        // 分支1：整个数组初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        // 分支2：第i个元素初始化
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value)))
                break;                   // no lock when adding to empty bin
        }
        // 分支3：扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else if (onlyIfAbsent // check first node without acquiring lock
                 && fh == hash
                 && ((fk = f.key) == key || (fk != null && key.equals(fk)))
                 && (fv = f.val) != null)
            return fv;
        // 分支4：放入元素
        else {
            V oldVal = null;
            // 重点：加锁
            synchronized (f) {
                // 链表
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { // 红黑树
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                    else if (f instanceof ReservationNode)
                        throw new IllegalStateException("Recursive update");
                }
            }
            // 如果是链表，上面的binCount会一直累加
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i); // 超出阈值，转换为红黑树
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);//总元素个数累加1
    return null;
}
```

上面的for循环有4个大的分支：

第一个分支，是整个数组的初始化，

第二个分支，是所在的槽为空，说明该元素是该槽的第一个元素，直接新建一个头节点，然后返回；

第三个分支，说明该槽正在进行扩容，帮助其扩容；

第四个分支，即使把元素放入槽内，槽内可能是一个链表，也可能是一棵红黑树，通过头节点的类型可以判断是哪一种。第四个分支是包裹在synchronized(f)里面的，f对应的数组下标位置的头节点，意味着每个数组元素有一把锁，并发度等于数组的长度。

上面的binCount表示链表的元素个数，当这个数目超过TREEIFY_THRESHOLD=8时，把链表转换成红黑树，也就是treeifyBin(tab, i)方法。但在这个方法内部，不一定需要进行红黑树转换，可能只做扩容操作，所以接下来从扩容说起。



### 5.5.4 扩容

扩容的实现是最复杂的，下面从treeifyBin(Node<K,V>[] tab, int index)说起。

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            // 数组长度小于阈值64,不做红黑树，直接扩容
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            // 链表转换为红黑树
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    // 遍历链表，初始化红黑树
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

在上面的代码中，MIN_TREEIFY_CAPACITY=64，意味着当数组长度没有超过64的时候，数组的每个节点里都是链表，只会扩容，不会转换为红黑树。只有当数组长度大于等于64时，才考虑把链表转换为红黑树。

![image-20210928165801647](assest/image-20210928165801647.png)

在tryPresize(int size)内部调用了一个核心方法transfer(Node<K,V>[] tab, Node<K,V>[] nextTab)，先从这个方法分析说起。

```java
private final void tryPresize(int size) {
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1);
    int sc;
    while ((sc = sizeCtl) >= 0) {
        Node<K,V>[] tab = table; int n;
        if (tab == null || (n = tab.length) == 0) {
            n = (sc > c) ? sc : c;
            if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt;
                        sc = n - (n >>> 2);
                    }
                } finally {
                    sizeCtl = sc;
                }
            }
        }
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) {
            int rs = resizeStamp(n);
            if (U.compareAndSetInt(this, SIZECTL, sc,
                                    (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
        }
    }
}
```





```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // 计算步长
    if (nextTab == null) {            // 初始化新的HashMap
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];// 扩容两倍
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        // 初始化的transferIndex=旧HashMap的数组长度
        transferIndex = n;
    }
    int nextn = nextTab.length;
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    // 此处，i为遍历下标，bound为边界
    // 如果成功获取下一个任务，则i=nextIndex-1
    // bound=nextIndex-stride
    // 如果获取步到，则i=0,bound=0
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // advance表示从i=transferIndex-1遍历到bound位置的过程中，是否一直继续
        while (advance) {
            int nextIndex, nextBound;
            // 以下是那个分支中的advance都是false，表示如果三个分支都不执行，才可以一直while循环
            // 目的在于当对transferIndex执行CAS操作不成功的时候，需要自旋，以期获取一个stride的迁移任务
            if (--i >= bound || finishing)
                // 对数组遍历，通过这里的--i进行。如果成功执行了--i，就不需要继续while循环了，因为advance只能进一步
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                // transferIndex <= 0，整个HashMap完成
                i = -1;
                advance = false;
            }
            // 对transferIndex执行CAS操作，即为当前线程分配1个stride。
            // CAS操作成功，线程成功获取到一个stride的迁移任务
            // CAS操作不成功，线程没有抢到任务，会继续执行while循环，自旋
            else if (U.compareAndSetInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        // i越界，整个HashMap遍历完成
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // finishing表示整个HashMap扩容完成
            if (finishing) {
                nextTable = null;
                // 将nextTab赋值给当前table
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            if (U.compareAndSetInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // tab[i]迁移完毕，赋值一个ForwardingNode
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // tab[i]的位置已经在迁移过程中
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            // 对tab[i]进行迁移操作，tab[i]可能是一个链表或者红黑树
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // 链表
                    if (fh >= 0) {
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                // 表示lastRun之后的所有元素，hash值都是一样的
                                // 记录下这个最后的位置
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            // 链表迁移的优化做法
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                    // 红黑树，迁移做法和链表类似
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                        (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                        (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

该方法非常复杂，下面一步步分析：

1. 扩容的基本原理如下图，首先建一个新的HashMap，其数组长度是旧数组长度的2倍，然后把旧的元素逐个迁移过来，第一个参数tab是扩容之前的HashMap，第二个参数nextTab是扩容后的HashMap。当nextTab=null的时候，方法最初会对nextTab进行初始化。这里有一个关键点要说明：该方法会被多个线程调用，所以每个线程只是扩容旧的HashMap部分，这就涉及如何任务的问题。

   ![image-20210928180120551](assest/image-20210928180120551.png)

2. 上图为多个线程并行扩容-任务划分示意图，旧数组的长度是N，每个线程扩容一段，一段的长度用变量stride（步长）表示，transferIndex表示了整个数组扩容的进度。

   stride的计算供使如上面的代码所示，即：在单核模式下直接等于n，因为在单核模式下没有办法多个线程并行扩容，只需要一个线程来扩容整个数组；在多核模式下为(n>>>3)/NCPU，并且保证部长最小值是16。显然，需要的线程个数约为n/stride。

   ![image-20210928184446395](assest/image-20210928184446395.png)

   transferIndex是ConcurrentHashMap的一个成员变量，记录了扩容的进度。初始值为n，从达到小扩容，每次减stride个位置，最终减至n<=0，表示真个扩容完成。因此，从[0，transferIndex-1]的位置表示还没有分配到线程扩容的部分，从[transferIndex，n-1]的位置表示已经分配给某个线程进行扩容，当前正在扩容中，或者已经扩容成功。

   因为transferIndex会被多个线程并发修改，每次减stride，所以需要通过CAS进行操作，如下面代码所示：

   ![image-20210928191203351](assest/image-20210928191203351.png)
   
   ![image-20210928224939498](assest/image-20210928224939498.png)
   
3. 在扩容未完成之前，有的数字下标对应的槽已经迁移到了新的HashMap里面，有的还在旧的HashMap里面。这个时候，所有调用get(k，v)的线程还是会访问旧HashMap，怎么处理呢？

   下图为扩容过程中的转发示意图：当Node[0]已经迁移成功，而其他Node还在迁移过程中时，如果有线程要读取Node[0]的数据，就会访问失败。为此，新建一个ForwardingNode，即转发节点，在这个节点里面记录的是新的ConcurrentHashMap的引用。这样，当线程访问ForwardingNode之后，会去查询新的ConcurrentHashMap。

4. 因为数组的长度`tab.length`是2的整数次方，每次扩容又是2倍。而Hash函数是hashCode%tab.length，等价于<br>hashCode&(tab.length-1)。这意味着：处于第i个位置的元素，在新的Hash表的数组中一定处于第i个或者第i+n个位置。如下图所示。举个例子：假设数组长度是8，扩容之后是16：<br>若hashCode = 5，5%8=0，扩容后，5%16=0，位置保持不变；<br>若hashCode = 24，24%8=0，扩容后，24%16=8，后移8个位置；<br>若hashCode = 25，25%8=1，扩容后，25%16=9，后移8个位置；<br>若hashCode = 39，39%8=7，扩容后，39%16=7，位置保持不变；<br>

   ![image-20210928230334903](assest/image-20210928230334903.png)

   正因为有这样的规律，所以如下有代码：

   ![image-20210928232657677](assest/image-20210928232657677.png)

   也就是把tab[i]位置的链表或红黑树重新组装成两部分，一部分链接到nextTab[i]的位置，一部分链接到nextTab[i+n]的位置，如下图所示。然后把tab[i]的位置指向一个ForwardingNode节点。

   同时，当tab[i]后面是链表时，使用类似于JDK 7中在扩容时的优化方法，从lastRun往后的所有节点，不需要依次拷贝，而是直接链接到新的链表头部。从lastRum往前的所有节点，需要依次拷贝。

   了解了核心的迁移函数transfer(tab, nextTab)，在回头看tryPresize(int size)函数。这个函数的输入的是整个Hash表的元素个数，在函数里面，根据需要对整个Hash表进行扩容。想要看明白这个函数，需要透彻的理解sizeCtl变量，下面这段注释摘自源码。

   ![image-20210928234425117](assest/image-20210928234425117.png)

   当sizeCtl=-1，表示整个HashMap正在初始化；<br>当sizeCtl=某个其他复数时，表示多个线程在对HashMap做并发扩容；<br>当sizeCtl=cap，tab=null，表示未初始化之前的初始化容量（如上面的构造函数所示）；<br>扩容成功之后，sizeCtl存储的是下一次要扩容的阈值，即上面初始化代码中的 n-(n>>>2)=0.75n。

   所以，sizeCtl变量在Hash表处于不同状态时，表达不同的含义。明白了这个道理，再来看上面的tryPresize(int size)函数。

   ```
   private final void tryPresize(int size) {
       int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
           tableSizeFor(size + (size >>> 1) + 1);
       int sc;
       while ((sc = sizeCtl) >= 0) {
           Node<K,V>[] tab = table; int n;
           if (tab == null || (n = tab.length) == 0) {
               n = (sc > c) ? sc : c;
               if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
                   try {
                       if (table == tab) {
                           @SuppressWarnings("unchecked")
                           Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                           table = nt;
                           sc = n - (n >>> 2);
                       }
                   } finally {
                       sizeCtl = sc;
                   }
               }
           }
           else if (c <= sc || n >= MAXIMUM_CAPACITY)
               break;
           else if (tab == table) {
               int rs = resizeStamp(n);
               if (U.compareAndSetInt(this, SIZECTL, sc,
                                       (rs << RESIZE_STAMP_SHIFT) + 2))
                   transfer(tab, null);
           }
       }
   }
   ```

   tryPresize(int size) 是根据期望的元素个数对整个Hash表进行扩容，核心是调用transfer函数。在第一次扩容的时候，sizeCtl会被设置成一个很大的负数U.compareAndSwapInt(this，SIZECTL，sc，(rs < < RESIZE_STAMP_SHIFT)+2)；之后每一个线程扩容的时候，sizeCtl就加1，U.compareAndSwapInt(this，SIZECTL，sc，sc+1)，带扩容完成之后，sizeCtl减1。



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