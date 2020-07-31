---
title: 不怕难之BlockingQueue及其实现
date: 2020-07-31 02:04:34
tags: Algorithm
categories:
cover: https:////upload-images.jianshu.io/upload_images/4180398-89f0d2693361656e.png?imageMogr2/auto-orient/strip|imageView2/2/w/873/format/webp
---
<meta name="referrer" content="no-referrer" />

# 1. 前言

BlockingQueue即阻塞队列，它是基于ReentrantLock，依据它的基本原理，我们可以实现Web中的长连接聊天功能，当然其最常用的还是用于实现生产者与消费者模式，大致如下图所示：

![img](https:////upload-images.jianshu.io/upload_images/4180398-89f0d2693361656e.png?imageMogr2/auto-orient/strip|imageView2/2/w/873/format/webp)

> 在Java中，BlockingQueue是一个接口，它的实现类有ArrayBlockingQueue、DelayQueue、 LinkedBlockingDeque、LinkedBlockingQueue、PriorityBlockingQueue、SynchronousQueue等，它们的区别主要体现在存储结构上或对元素操作上的不同，但是对于take与put操作的原理，却是类似的。

# 2. 阻塞与非阻塞

## **入队**

> offer(E e)：如果队列没满，立即返回true； 如果队列满了，立即返回false-->不阻塞
>
> put(E e)：如果队列满了，一直阻塞，直到队列不满了或者线程被中断-->阻塞
>
> offer(E e, long timeout, TimeUnit unit)：在队尾插入一个元素,，如果队列已满，则进入等待，直到出现以下三种情况：-->阻塞
>
> 被唤醒
>
> 等待时间超时
>
> 当前线程被中断

## **出队**

> poll()：如果没有元素，直接返回null；如果有元素，出队
>
> take()：如果队列空了，一直阻塞，直到队列不为空或者线程被中断-->阻塞
>
> poll(long timeout, TimeUnit unit)：如果队列不空，出队；如果队列已空且已经超时，返回null；如果队列已空且时间未超时，则进入等待，直到出现以下三种情况：
>
> 被唤醒
>
> 等待时间超时
>
> 当前线程被中断



# 3. LinkedBlockingQueue 源码分析

LinkedBlockingQueue是一个基于链表实现的可选容量的阻塞队列。队头的元素是插入时间最长的，队尾的元素是最新插入的。新的元素将会被插入到队列的尾部。 

LinkedBlockingQueue的容量限制是可选的，如果在初始化时没有指定容量，那么默认使用int的最大值作为队列容量。

## **底层数据结构**

LinkedBlockingQueue内部是使用链表实现一个队列的，但是却有别于一般的队列，在于该队列至少有一个节点，头节点不含有元素。结构图如下：



![img](https:////upload-images.jianshu.io/upload_images/4180398-d987601194c3e199.png?imageMogr2/auto-orient/strip|imageView2/2/w/835/format/webp)

## **原理**

LinkedBlockingQueue中维持两把锁，一把锁用于入队，一把锁用于出队，这也就意味着，同一时刻，只能有一个线程执行入队，其余执行入队的线程将会被阻塞；同时，可以有另一个线程执行出队，其余执行出队的线程将会被阻塞。换句话说，虽然入队和出队两个操作同时均只能有一个线程操作，但是可以一个入队线程和一个出队线程共同执行，也就意味着可能同时有两个线程在操作队列，那么为了维持线程安全，LinkedBlockingQueue使用一个AtomicInterger类型的变量表示当前队列中含有的元素个数，所以可以确保两个线程之间操作底层队列是线程安全的。

## 源码分析

LinkedBlockingQueue可以指定容量，内部维持一个队列，所以有一个头节点head和一个尾节点last，内部维持两把锁，一个用于入队，一个用于出队，还有锁关联的Condition对象。主要对象的定义如下：

> //容量，如果没有指定，该值为Integer.MAX_VALUE;
>
> private final int capacity;
>
> //当前队列中的元素
>
> private final AtomicInteger count =new AtomicInteger();
>
> //队列头节点，始终满足head.item==null
>
> transient Node head;
>
> //队列的尾节点，始终满足last.next==null
>
> private transient Node last;
>
> //用于出队的锁
>
> private final ReentrantLock takeLock =new ReentrantLock();
>
> //当队列为空时，保存执行出队的线程
>
> private final Condition notEmpty = takeLock.newCondition();
>
> //用于入队的锁
>
> private final ReentrantLock putLock =new ReentrantLock();
>
> //当队列满时，保存执行入队的线程
>
> private final Condition notFull = putLock.newCondition();

### put(E e)方法

put(E e)方法用于将一个元素插入到队列的尾部，其实现如下：

> public void put(E e)throws InterruptedException {
>
> //不允许元素为null
>
>   if (e ==null)
>
> throw new NullPointerException();
>
>   int c = -1;
>
>   //以当前元素新建一个节点
>
>   Node node =new Node(e);
>
>   final ReentrantLock putLock =this.putLock;
>
>   final AtomicInteger count =this.count;
>
>   //获得入队的锁
>
>   putLock.lockInterruptibly();
>
>   try {
>
> ​    //如果队列已满，那么将该线程加入到Condition的等待队列中
>
> ​    while (count.get() == capacity) {
>
> ​       notFull.await();
>
> ​    }
>
> ​    //将节点入队
>
> ​    enqueue(node);
>
> ​    //得到插入之前队列的元素个数
>
> ​    c = count.getAndIncrement();
>
> ​    //如果还可以插入元素，那么释放等待的入队线程
>
> ​    if (c +1 < capacity){
>
> ​       notFull.signal();
>
> ​    }
>
> }finally {
>
> //解锁
>
> ​    putLock.unlock();
>
>   }
>
> //通知出队线程队列非空
>
>   if (c ==0)
>
> signalNotEmpty();
>
> }

####  

**3.1 具体入队与出队的原理图**：

图中每一个节点前半部分表示封装的数据x，后边的表示指向的下一个引用。



![img](https:////upload-images.jianshu.io/upload_images/4180398-ccb29b29b48e6192.png?imageMogr2/auto-orient/strip|imageView2/2/w/132/format/webp)

初始化之后，初始化一个数据为null，且head和last节点都是这个节点。

**3.2、入队两个元素过后**



![img](https:////upload-images.jianshu.io/upload_images/4180398-0d6b2ff6b3444abf.png?imageMogr2/auto-orient/strip|imageView2/2/w/436/format/webp)

**3.3、出队一个元素后**



![img](https:////upload-images.jianshu.io/upload_images/4180398-b482b1f470e2ad3a.png?imageMogr2/auto-orient/strip|imageView2/2/w/283/format/webp)

####  

#### put方法总结： 

\1. LinkedBlockingQueue不允许元素为null。 

\2. 同一时刻，只能有一个线程执行入队操作，因为putLock在将元素插入到队列尾部时加锁了 

\3. 如果队列满了，那么将会调用notFull的await()方法将该线程加入到Condition等待队列中。await()方法就会释放线程占有的锁，这将导致之前由于被锁阻塞的入队线程将会获取到锁，执行到while循环处，不过可能因为由于队列仍旧是满的，也被加入到条件队列中。 

\4. 一旦一个出队线程取走了一个元素，并通知了入队等待队列中可以释放线程了，那么第一个加入到Condition队列中的将会被释放，那么该线程将会重新获得put锁，继而执行enqueue()方法，将节点插入到队列的尾部 

\5. 然后得到插入一个节点之前的元素个数，如果队列中还有空间可以插入，那么就通知notFull条件的等待队列中的线程。 

\6. 通知出队线程队列为空了，因为插入一个元素之前的个数为0，而插入一个之后队列中的元素就从无变成了有，就可以通知因队列为空而阻塞的出队线程了。

## E take()方法

take()方法用于得到队头的元素，在队列为空时会阻塞，知道队列中有元素可取。其实现如下：

> public E take() throws InterruptedException {
>
> ​    E x;
>
> ​    int c = -1;
>
> ​    final AtomicInteger count = this.count;
>
> ​    final ReentrantLock takeLock = this.takeLock;
>
> ​    //获取takeLock锁    
>
> ​     takeLock.lockInterruptibly();
>
> ​    try {
>
> ​      //如果队列为空，那么加入到notEmpty条件的等待队列中      
>
> ​      while (count.get() == 0) {
>
> ​        notEmpty.await();
>
> ​      }
>
> ​      //得到队头元素      
>
> ​       x = dequeue();
>
> ​      //得到取走一个元素之前队列的元素个数      
>
> ​        c = count.getAndDecrement();
>
> ​      //如果队列中还有数据可取，释放notEmpty条件等待队列中的第一个线程      
>
> ​        if (c > 1)
>
> ​        notEmpty.signal();
>
> ​    } finally {
>
> ​      takeLock.unlock();
>
> ​    }
>
> ​    //如果队列中的元素从满到非满，通知put线程    
>
> ​      if (c == capacity)
>
> ​      signalNotFull();
>
> ​    return x;
>
>   }

#### take方法总结:

当队列为空时，就加入到notEmpty(的条件等待队列中，当队列不为空时就取走一个元素，当取完发现还有元素可取时，再通知一下自己的伙伴（等待在条件队列中的线程）；最后，如果队列从满到非满，通知一下put线程。 

## remove()方法 

remove()方法用于删除队列中一个元素，如果队列中不含有该元素，那么返回false；有的话则删除并返回true。入队和出队都是只获取一个锁，而remove()方法需要同时获得两把锁，其实现如下：

> public boolean remove(Object o) {
>
> ​    //因为队列不包含null元素，返回false   
>
> ​     if (o == null) return false;
>
> ​    //获取两把锁    fullyLock();
>
> ​    try {
>
> ​      //从头的下一个节点开始遍历      
>
> ​      for (Node trail = head, p = trail.next;
>
> ​         p != null;
>
> ​         trail = p, p = p.next) {
>
> ​         //如果匹配，那么将节点从队列中移除，trail表示前驱节点        
>
> ​        if (o.equals(p.item)) {
>
> ​          unlink(p, trail);
>
> ​          return true;
>
> ​        }
>
> ​      }
>
> ​      return false;
>
> ​    } finally {
>
> ​      //释放两把锁     
>
> ​      fullyUnlock();
>
> ​    }
>
>   }

> 
> void fullyLock() {
>
> ​    putLock.lock();
>
> ​    takeLock.lock();
>
>   }

## 提问：为什么remove()方法同时需要两把锁? 

## LinkedBlockingQueue总结:

LinkedBlockingQueue是允许两个线程同时在两端进行入队或出队的操作的，但一端同时只能有一个线程进行操作，这是通过两把锁来区分的；

为了维持底部数据的统一，引入了AtomicInteger的一个count变量，表示队列中元素的个数。count只能在两个地方变化，一个是入队的方法（可以+1），另一个是出队的方法（可以-1），而AtomicInteger是原子安全的，所以也就确保了底层队列的数据同步。 

# 4. ArrayBlockingQueue源码分析

ArrayBlockingQueue底层是使用一个数组实现队列的，并且在构造ArrayBlockingQueue时需要指定容量，也就意味着底层数组一旦创建了，容量就不能改变了，因此ArrayBlockingQueue是一个容量限制的阻塞队列。因此，在队列全满时执行入队将会阻塞，在队列为空时出队同样将会阻塞。

ArrayBlockingQueue的重要字段有如下几个：

> ​    /** The queued items */ 
>
> ​     final Object[] items;
>
>    /** Main lock guarding all access */ 
>
> ​    final ReentrantLock lock;
>
>   /** Condition for waiting takes */  
>
>    private final Condition notEmpty;
>
>   /** Condition for waiting puts */  
>
>    private final Condition notFull;

## put(E e)方法

put(E e)方法在队列不满的情况下，将会将元素添加到队列尾部，如果队列已满，将会阻塞，直到队列中有剩余空间可以插入。该方法的实现如下：

> public void put(E e) throws InterruptedException {
>
> ​    //检查元素是否为null，如果是，抛出NullPointerException    
>
> ​    checkNotNull(e);
>
> ​    final ReentrantLock lock = this.lock;
>
> ​    //加锁    
>
> ​    lock.lockInterruptibly();
>
> ​    try {
>
> ​      //如果队列已满，阻塞，等待队列成为不满状态      
>
> ​      while (count == items.length)
>
> ​        notFull.await();
>
> ​      //将元素入队      
>
> ​      enqueue(e);
>
> ​    } finally {
>
> ​      lock.unlock();
>
> ​    }
>
>   }

## put方法总结:

\1. ArrayBlockingQueue不允许元素为null 

\2. ArrayBlockingQueue在队列已满时将会调用notFull的await()方法释放锁并处于阻塞状态 

\3. 一旦ArrayBlockingQueue不为满的状态，就将元素入队

## **E take()方法**

take()方法用于取走队头的元素，当队列为空时将会阻塞，直到队列中有元素可取走时将会被释放。其实现如下：

>  public E take() throws InterruptedException {
>
> ​    final ReentrantLock lock = this.lock;
>
> ​    //首先加锁    
>
> ​     lock.lockInterruptibly();
>
> ​    try {
>
> ​      //如果队列为空，阻塞      
>
> ​      while (count == 0)
>
> ​        notEmpty.await();
>
> ​      //队列不为空，调用dequeue()出队      
>
> ​      return dequeue();
>
> ​    } finally {
>
> ​      //释放锁      
>
> ​     lock.unlock();
>
> ​    }
>
>   }

### take方法总结:

一旦获得了锁之后，如果队列为空，那么将阻塞；否则调用dequeue()出队一个元素。 

## ArrayBlockingQueue总结：

ArrayBlockingQueue的并发阻塞是通过ReentrantLock和Condition来实现的，ArrayBlockingQueue内部只有一把锁，意味着同一时刻只有一个线程能进行入队或者出队的操作。



# 5 总结

在上面分析LinkedBlockingQueue的源码之后，可以与ArrayBlockingQueue做一个比较。 

## ArrayBlockingQueue：

一个对象数组+一把锁+两个条件

入队与出队都用同一把锁

在只有入队高并发或出队高并发的情况下，因为操作数组，且不需要扩容，性能很高

采用了数组，必须指定大小，即容量有限

## LinkedBlockingQueue：

一个单向链表+两把锁+两个条件

两把锁，一把用于入队，一把用于出队，有效的避免了入队与出队时使用一把锁带来的竞争。

在入队与出队都高并发的情况下，性能比ArrayBlockingQueue高很多

采用了链表，最大容量为整数最大值，可看做容量无限