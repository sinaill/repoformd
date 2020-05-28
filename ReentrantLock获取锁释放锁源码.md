title: ReentrantLock获取锁释放锁源码
categories: java基础
tags: 
	- 多线程
---

### AbstractQueuedSynchronizer

`ReentrantLock`中主要用到的核心类就是`AbstractQueuedSynchronizer`，例如最基本的获取锁和释放锁

`private volatile int state`: The synchronization state，标记当前加锁状态的变量，初始为0，当有线程调用`lock`方法获取到锁时，它会变为1(重入时继续增1)

`private transient volatile Node head`: 线程队列头

`private transient volatile Node tail`: 线程队列尾

它有个内部类`Node`，每个线程为一个节点，以链表的结构实现多个线程排队

`volatile Node prev;`: 上一个节点

`volatile Node next`: 下一个节点

`volatile int waitStatus`: 线程等待状态

`volatile Thread thread`： 节点中的线程

`volatile Node nextWaiter`: 节点锁的属性(shared和exclusive)

等待线程队列wait queue结构如下

![](https://img-blog.csdnimg.cn/20190829203640524.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1ODM1NjI0,size_16,color_FFFFFF,t_70)

### 公平锁和非公平锁

![](https://raw.githubusercontent.com/sinaill/pic/master/Sync.png) 

ReentrantLock中有两种锁，公平锁`FairSync`和非公平锁`NonfairSync`，公平锁遵循FIFO原则

```
/**
 * Creates an instance of {@code ReentrantLock}.
 * This is equivalent to using {@code ReentrantLock(false)}.
 */
public ReentrantLock() {
    sync = new NonfairSync();
}

/**
 * Creates an instance of {@code ReentrantLock} with the
 * given fairness policy.
 *
 * @param fair {@code true} if this lock should use a fair ordering policy
 */
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

分别创建方法如上，默认创建非公平锁，如果想要创建公平锁，则需要构造函数传入`True`

### 加锁

```
public void lock() {
    sync.lock();
}
```

这里可以看两种锁的实现方式

```
/**
 * 非公平锁
 */
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}

/**
 * 公平锁
 */
final void lock() {
    acquire(1);
}
```

可以看到非公平锁在线程获取锁时，是采用`compareAndSetState(0, 1)`马上尝试获取锁

接着来到`acquire(1)`

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

`tryAcquire`方法两种锁还是分别有自己的实现逻辑

```
/**
 * 公平锁的tryAcquire方法
 */
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
		//当前锁空闲时，查看是否有线程队列排队中
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
	//锁被占用时，如果是用一个线程重入，state继续加1
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

公平锁即使在锁空闲时，仍会`hasQueuedPredecessors()`查看时候后面还有线程在队列派对中

```
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

非公平锁的`tryAcquire`继续调用`nonfairTryAcquire(int acquires)`

```
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
		//锁空闲时，继续抢锁
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
还是一样，`state==0`时继续抢锁

以上就是两种锁在抢锁时不同的地方，非公平锁相当于抢两次锁，没抢到就去排队，而公平锁会先看有没有线程在排队，没有时才抢锁

然后我们回到`acquire(int arg)`方法，这里可以看到线程是怎么排队的

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

`addWaiter(Node.EXCLUSIVE)`方法创建新节点存放排队的线程

```
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);//将创建的节点入队
    return node;
}
```

每个线程创建一个节点，此时节点中`waitStatus`为0，然后将这个节点设置为`tail`，并且和上一个`tail`串联起来(将节点中`prev`指向上一个`tail`)，就这样，线程呈链表的形式排成队，接着进入`acquireQueued`方法

```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
			//查看上一个节点是不是头节点，是的话抢锁
            if (p == head && tryAcquire(arg)) {
                setHead(node);//抢到锁将自己变为头节点
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
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

这里接受的`node`是线程创建好的节点，然后线程开始以这样的形式，Head->线程节点1->线程节点2···，在for无条件循环中，每一个节点的线程检测上一个节点是否为头节点，是的话，尝试抢锁，成功之后把自己节点设置为头节点，Head->线程节点2···

非头节点时，进入`shouldParkAfterFailedAcquire(p, node)`

```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
    }
```
初始`waitStatus`为0，所以这里把前节点的`waitStatus`都设置为`Node.SIGNAL(-1)`，返回`false`，然后短路，第二次循环这里返回`true`，然后进入`parkAndCheckInterrupt()`，挂起线程

```
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

这里把两次循环都没抢到锁的线程挂起

### 释放锁

```
public void unlock() {
    sync.release(1);
}


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

`tryRelease(arg)`开始释放锁

```
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
	//被锁的线程才能释放锁，否则抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

这里将`state`减1，等于0的时候释放锁，将`state`置0，返回`true`，此时如果有线程在排队(挂起状态)，`unparkSuccessor(h)`将线程回到运行状态

```
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
		//唤醒Head后一个节点中的线程
        LockSupport.unpark(s.thread);
}
```

这里传入`Head`节点，从前面可以知道，`Head`节点后面有线程节点排队时，`head`节点的`waitStatus`被置为`Node.SIGNAL`，即-1，这里重新置0，然后获取`Head`节点的下一个节点，如果不为空，则将节点中存放的创建该节点的线程唤醒

