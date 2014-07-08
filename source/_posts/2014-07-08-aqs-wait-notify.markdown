---
layout: post
title: "JAVA AQS 解密"
date: 2014-07-08 19:42:36 +0800
comments: true
categories: Java
---

# JAVA AQS 实现细节
AQS(AbstractQueuedSynchronizer)是java lock的实现基础，如下类图描述，juc的很多实现都依赖AQS。AQS确实比较复杂，
所有不同锁的每个细节难以描述清楚，所以本文的重点是使用ReentrantLock作为例子只分析NonfairSync和condition的实现原理，
其它锁的实现原理都是以此作为基础进行扩展和优化。

{% img center /images/aqs/aqs-class.png %}

# 从Lock的一段简单代码开始
就以下面一段我们经常使用的简单代码作为例子，例子中就是使用Lock和Condition实现线程安全的生产者-消费者模型，借助这个简单的例子，
我们一步步分析一下Lock内部是如何实现线程同步的。

```java
		
		import java.util.concurrent.locks.Condition;
		import java.util.concurrent.locks.Lock;
		import java.util.concurrent.locks.ReentrantLock;
		
	public class ProductQueue<T> {

    private T[]       items;
    private Lock      lock         = new ReentrantLock();
    final static int  DEAFULT_SIZE = 10;

    //满队列条件
    private Condition notFull      = lock.newCondition();

    //空队列条件
    private Condition notEmpty     = lock.newCondition();

    private int       head, tail, count;

    @SuppressWarnings("unchecked")
    public ProductQueue(int size) {
        items = (T[]) new Object[size];
    }

    public ProductQueue() {
        this(DEAFULT_SIZE);
    }

    /**
     * 生产产品
     * 
     * @param t
     * @throws InterruptedException
     */
    public void put(T t) throws InterruptedException {
        lock.lock();//获取独占锁
        try {
            Thread.sleep(2 * 1000);
            while (count == getCapacity()) {//队列已满，挂起当前 线程，将当前线程加入条件锁队列中，直到收到队列不为满的信号，从队列中按照FIFO唤醒一个线程
                notFull.await(); //释放独占锁 ，让其他线程（消费者线程）获取，然后挂起当前线程，一旦后续条件满足，再次获取锁
            }
            items[tail] = t;
            if (++tail == getCapacity()) {
                tail = 0;
            }
            ++count;
            notEmpty.signalAll();//队列已不为空，可以唤醒消费者 消费产品
        } finally {
            lock.unlock();
        }

    }

    /**
     * 消费产品
     * 
     * @return
     * @throws InterruptedException
     */
    public T get() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {//队列已空，挂起空队列锁条件 线程，直到收到不为空的信号，被唤醒
                notEmpty.await();
            }
            T ret = items[head];
            items[head] = null;
            if (++head == getCapacity()) {
                head = 0;
            }
            --count;
            notFull.signalAll();//队列已不为满，可以唤醒生产者  生产产品
            return ret;

        } finally {
            lock.unlock();
        }
    }

    /**
     * 产品队列容量
     * 
     * @return
     */
    public int getCapacity() {
        return items.length;
    }

    /**
     * 当前产品数量
     * 
     * @return
     */
    public int size() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }

}

```

# AQS基础数据结构
在开始分析之前，先讲一下AQS实现中的基础数据结构（如下图所示）
AQS基础数据结构.png

*	state：32位整数描述的状态位来支持原子操作(volatile+CAS)
*	head：有序队列的队头结点，tail：有序队列的队尾结点
*	pre：当前节点的前一个结点引用 next:当前节点的下一个节点引用主要用于构建有序队列的双向链表
*	thread：当前节点绑定的线程
*	waitStatus：当前节点的等待状态
*	nextWaiter：当前节点的后继节点，主要结合Condition使用
*	firstWaiter：Condition队列的第一个Node
*	lastWaiter：Condition队列的最后一个Node，主要用于表示Condition队列的头和尾节点，需要区分AQS队列的head和tail

# Lock.lock()
从Lock的lock方法作为入口讲起，参考下面的处理流程图：

{% img center /images/aqs/lock-workflow.png %}

整个加锁执行逻辑：

*	1.如果tryAcquire(arg)成功，返回true，说明当前线程已经拿到锁，执行当前线程操作，整个lock()过程就结束了。如果失败进行操作2。  
*	2.创建一个独占节点（同时会新生成一个傀儡头结点）并且将此节点加入CHL队列末尾。进行操作3。  
*	3.acquireQueued(node,arg),自旋尝试获取锁，如果获取失败，则根据前一个节点来决定是否挂起（park()），不管是否挂起，都会自旋，直到成功获取到锁。如果在自旋的过程中线程被中断过，那么会置线程中断标志,  进行操作4。
*	4.如果当前线程已经中断过，那么就中断当前线程（清除中断位）。




# Lock.unlock()

{% img center /images/aqs/unlock-workflow.png %}

整个解锁执行逻辑：

*	1.tryRelease(arg)执行解锁操作，如果锁释放失败或者当前线程没有持有锁，则抛出异常(IllegalMonitorStateException),如果成功，则进行操作2。
*	2.获取CHL队列头结点，如果头结点为空或者状态为0，说明CHL队列为空，结束锁释放过程，否则进行操作3。
*	3.找到需要唤醒的继任节点，并进行线程唤醒操作（unpark()），同时会维护CHL队列，去除已中断或者超时的节点，结束锁释放过程。



# AQS队列
{% img center /images/aqs/AQS-queue.png %}

AQS队列

*	整个AQS队列为带头结点和尾节点的双向链表，其中的节点主要以waitStatus来表示其状态，并绑定对应的线程。
*	AQS队列的头节点为傀儡节点，唤醒的节点为傀儡节点的继任节点，移除节点从队列头节点开始，添加节点从队列尾节点开始，添加时主要以CAS操作保证线程安全。
*	AQS队列主要按FIFO来保证节点的执行顺序，以此来达到公平性。


# Condition.await()
{% img center /images/aqs/condit-await.png %}

执行逻辑如下：

*	1.将当前线程加入Condition锁队列。特别说明的是，这里不同于AQS的队列，这里进入的是Condition的FIFO队列。
*	2.释放锁，这里可以看到将锁释放了，否则别的线程就无法拿到锁而发生死锁。进行3。
*	3.自旋(while)挂起，直到被唤醒或者超时或者CACELLED。进行4。
*	4.获取锁(acquireQueued)，并将自己从Condition的FIFO队列中释放，表明自己不再需要锁（我已经拿到锁了）。


# Condition.signal()
{% img center /images/aqs/condition-signal.png %}

signal就是唤醒Condition队列中的第一个非CANCELLED节点线程，
而signalAll就是唤醒所有非CANCELLED节点线程。当然了遇到CANCELLED线程就需要将其从FIFO队列中剔除。


# aqs和condition2个队列
{% img center /images/aqs/aqs-condition-queue.png %}

AQS和Condition队列

*	以生产者-消费者模型来说，在产品的生成和消费过程中，会维护3个队列，一个AQS队列，一个生产者条件队列，一个消费者条件队列。
* AQS和Condition队列有一定的不同，条件队列为典型的FIFO队列，而AQS带有一定的特性（头结点和继任节点关系）。
* 当执行await()会向相应的条件队列中加入条件节点，并进行相应的维护，当执行signal()或者signalAll()时，会将条件队列中的节点维护到AQS中，进行队列间的交互。

