# AQS


## 1.AQS简介与原理阐述

AQS(AbstractQueuedSynchronizer)是一个用来构建锁和同步器的框架，它内部使用了一个int类型的state变量来表示同步状态。该变量用volatile修饰，确保并发下同步状态的正确性。AQS内部维护着一个CLH队列，对于那些获取共享资源失败的线程，将会被加入到CLH队列中。并提供了一套完整的机制，来唤醒队列中被阻塞的线程。Lock包中的锁也都是基于AQS实现的(如ReentrantLock、Semaphore)。

### 1.1 CLH队列

> CLH(Craig,Landin,and Hagersten)队列就是一个虚拟的双向FIFO队列。(虚拟的，即不存在队列实例，仅存在节点之间的关系)。AQS将每个请求共享资源的线程封装成一个Node节点，加入到CLH队列中。

下图是一个CLH队列的描述，AQS持有队列的head和tail节点。

![image-20200108142138687.png](https://i.loli.net/2020/01/09/r4HFdEMDiOVYfly.png)

AQS内部用一个int类型的变量表示同步状态，并提供了一系列的原子操作对其进行修改。

```java
private volatile int state; //volatile修饰，保证了线程的可见性

//获取当前同步状态
protected final int getState() {  
    return state;
}

//设置当前同步状态
protected final void setState(int newState) {
    state = newState;
}

//CAS修改，如果当前值等于期望值(expect)，则进行修改(update)
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

### 1.2 AQS内部类Node

现在我们知道了请求共享资源的线程是被封装成一个Node节点加入CLH队列的，那么我们来看一下这个内部类Node的结构

```java
static final class Node {
	
    static final Node SHARED = new Node(); //共享模式节点
    
    static final Node EXCLUSIVE = null; //独占模式节点
	
  
    static final int CANCELLED =  1; 

    static final int SIGNAL    = -1;

    static final int CONDITION = -2;

    static final int PROPAGATE = -3;
    
    //节点状态，有CANCELLED、SIGNAL、CONDITION、PROPAGATE和默认值0几种取值
    volatile int waitStatus;

    volatile Node prev; //前驱节点

    volatile Node next; //后继节点

    volatile Thread thread; //请求资源的线程

    Node nextWaiter;
}
```

对于waitStatus几种取值的解释：

* CANCELLED：该节点的线程可能由于超时或被中断而处于"取消"状态，且这个状态是有永久的，因此应该移出队列。

* SIGNAL：如果节点处于SIGNAL状态，则后继节点应该被阻塞。所以当前节点释放或被取消时，应该唤醒后继节点。

* CONDITION：该节点从LCH同步队列转移到condition的等待队列中，直到被唤醒，重新加入同步队列。

* PROPAGATE：和共享模式节点相关，处于这个状态的节点一旦被释放，将会把这个状态传播下去。

* 0：新加入的节点默认值



## 2.锁的获取/释放

### 2.1 独占式的获取

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

tryAcquire(arg)方法在AQS中并没有实现，需要开发者自己去实现，成功返回true，失败返回false。然后调用addWaiter()方法将当前线程构造成一个独享模式的节点。

```java
private Node addWaiter(Node mode) {
    // 将当前线程封装成一个独占式的节点
    Node node = new Node(Thread.currentThread(), mode);
  	// 获取同步队列的尾节点
    Node pred = tail;
    // 如果尾节点不为null，采用尾插法把当前节点加入到末尾
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node; //返回旧的尾节点
        }
    }
    enq(node); // 尾节点为null,即队列为空，初始化队列。
    return node;
}
```

可以看到，如果队列的尾节点不为null，则CAS把当前线程的节点设置为尾节点。若为null则调用enq(node)方法初始化队列。

```java
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { 
            // 队列为空，创建一个空节点，CAS初始化队列
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // CAS把当前节点插入队尾
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;  // 返回旧的尾节点
            }
        }
    }
}
```

接着，当addWaiter()把当前线程成功加入同步队列中后，并不会立即挂起线程，而是调用acquireQueued()方法，自旋的重新尝试获取锁，因为在加入同步队列的过程中，可能前面的线程已经释放锁了。当满足条件后退出自旋。

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        // 中断标志位，表示该线程在获取锁的过程中是否被中断过
        boolean interrupted = false; 
        for (;;) {
            final Node p = node.predecessor(); // 获取当前节点的前驱节点
            // 如果前驱节点是头节点，则尝试快速获取锁且获取成功
            if (p == head && tryAcquire(arg)) {
                setHead(node); // 把当前节点设置为头节点
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // shouldParkAfterFailedAcquire判断线程释放应该被挂起
            // parkAndCheckInterrupt挂起线程并返回线程被挂起时是否被中断过
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

shouldParkAfterFailedAcquire根据其前驱节点的状态来判断当前线程是否应该被挂起。

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    // 前驱节点的状态为SIGNAL，则当前线程应该被挂起
    if (ws == Node.SIGNAL)
        return true;
    // 如果前驱节点状态为CANCELLED，则前驱节点开始一直往前遍历，找到第一个节点状态非CANCELLED的，
    // 把找到节点作为当前线程的前驱节点。被遍历过的CANCELLED节点就会移出队列
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        // 如果前驱节点状态既不是SIGNAL也不是CANCELLED，把前驱节点设置为SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

parkAndCheckInterrupt调用LockSupport.park()阻塞线程，当线程被唤醒时检查线程是否被中断过。

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

现在我们对独占式的锁获取做个总结：

1. 通过模板方法acquire()，调用子类实现的tryAcquire()方法获取同步状态失败后，执行步骤2

2. 调用addWaiter()方法，将当前线程构造成一个Node节点，加入到同步队列的尾部。如果队列为空，则先初始化队列。继续执行步骤3

3. 调用acquireQueued()方法，判断前驱节点是否是头节点。

   3.1 如果是头节点则调用tryAcquire()尝试获取同步状态。若获取成功，则设置当前节点为新的头	    	  节点，返回并检查线程是否需要被中断。

   3.2 如果不是头节点或尝试获取失败，执行步骤4

4. 调用shouldParkAfterFailedAcquire()方法，判断前驱节点的状态。

   4.1 如果状态为SIGNAL，返回ture表示当前线程需要被阻塞。执行步骤5

   4.2 如果状态>0，即CANCELLED状态，从前驱节点开始一直往前遍历，直到找到第一个状态非  	  	  CANCELLED的节点。然后把这个非CANCELLED节点设置为当前线程的前驱节点，被遍历过	   	  CANCELLED节点会被移出同步队列。然后返回false从步骤开始重新执行。

   4.3 如果状态<0，则CAS把前驱节点状态设置为SIGNAL。然后返回false从步骤开始重新执行。

5. 调用parkAndCheckInterrupt()方法阻塞当前线程，当被唤醒时，返回线程是否被中断过。然后从步骤3开始重新执行

   

文字说起来可能不太好理解，可以结合下面的流程图来理解。
![image-20200109104052079.png](https://i.loli.net/2020/01/09/AS7WRbyzFLx6Vj4.png)

### 2.2 独占式的释放

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

和锁得获取一样，tryRelease同样需要子类去实现。成功返回true，失败返回false。既然是释放锁，那当然是获取锁的线程才能释放，所以这里就是释放head。

unparkSuccessor释放操作，就是把头节点状态改为0，然后唤醒后继节点。注意头节点的出队是在后继节点获取锁时发生的。

```java
private void unparkSuccessor(Node node) {
 
    int ws = node.waitStatus;
    // 如果头节点的状态<0，则CAS把头节状态修改为0，以便后继节点被唤醒时可以竞争锁。
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    Node s = node.next;
   	// 如果头节点的后继节点是null或是CANCELLED状态
    if (s == null || s.waitStatus > 0) {
        s = null;
        // 从后往前遍历，找到队列最前面那个状态非CANCELLED的节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    // 如果存在状态非CANCELLED的后继节点，则唤醒该节点线程
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

下面我们来总结一下独占式锁释放的流程：

1. 通过模板方法release()调用子类的实现tryRelease()，释放同步状态和头节点中的线程。如果成功则执行步骤2
2. 判断头节点的状态释是否<0，如果是则把状态修改为0。然后从后往前遍历找到队列最前面那个状态非CANCELLED的节点，如果存在则唤醒该节点中的线程。该线程被唤醒后就会重新竞争锁。