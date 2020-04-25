title: await、signal和signalAll源码
categories: java基础
tags: 
	- 多线程
---

### Condition

`ReentrantLock`使用了`Condition`接口实现类`ConditionObject`来进行线程间通信，例如它的`await、signal`方法

```
public Condition newCondition() {
    return sync.newCondition();
}


final ConditionObject newCondition() {
    return new ConditionObject();
}
```

接着观察`ConditionObject`，用来挂起线程的队列结构和抢锁时抢不到锁的线程队列结构相同，使用的`Node`都为`AbstractQueuedSynchronizer`内部类，并且`ConditionObject`也创建`firstWaiter`和`lastWaiter`指向链表头尾

![](https://img-blog.csdnimg.cn/20190829203647414.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM1ODM1NjI0,size_16,color_FFFFFF,t_70)

### await()方法

```
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();//往conditionqueue中添加当前线程节点
    int savedState = fullyRelease(node);//完全释放锁(可重入)并且唤醒下个线程
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {//如果节点不在waitqueue中
        LockSupport.park(this);//线程挂起
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
	//acquireQueued抢占锁，抢不到则挂起，无限循环
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) 
        unlinkCancelledWaiters();//清除waitStatus为cancelled的节点
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);//处理中断，抛出异常或者中断当前线程
}
```

`addConditionWaiter`往`conditionQueue`中增加节点

```
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // If lastWaiter is cancelled, clean out.
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```
节点中依旧存放当前线程和`waitStatus`

```
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        int savedState = getState();
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            throw new IllegalMonitorStateException();
        }
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

`fullyRelease`完全释放锁，唤醒`waitqueue`中下一个线程(unpark)

接着一个`while`循环，`isOnSyncQueue(node)`判断这个节点是否在`waitqueue`，前面可以知道这个节点是在`conditionqueue`中，所以这里进入循环中，然后挂起线程

线程被唤醒(unpark)后，`checkInterruptWhileWaiting`接着检测等待中是否被中断过，被中断的话直接`break`，跳出循环

`signal`前被中断，这里返回`THROW_IE`，`signal`后被中断则返回`REINTERRUPT`，没有中断返回 0

而按照正常流程，这里跳出循环是由于`node`节点被从`conditionqueue`中被转移到`waitqueue`中

接着到`acquireQueued(node, savedState)`方法，之前已经看过这个方法

```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
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

当`node`被转移到`waitqueue`之后，这个方法就发挥作用了，尝试抢锁，失败则被挂起，等待其他线程释放锁

### signal()方法

```
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}
```

第一步就看到，想调用`signal`方法，必须先拿到锁

```
private void doSignal(Node first) {
    do {
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```

`transferForSignal(first)`遍历`conditionqueue`，而且是从第一个开始

```
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

CAS将节点`waitStatus`从`Node.CONDITION`转为0，失败的话则是节点被取消，`waitStatus`被改变了

`enq(node)`之前获取锁释放锁源码已经看了，就是将`node`入队到`waitqueue`中，并且这里还将前节点`p`的`waitStatus`由默认0变为`Node.SIGNAL`，获取锁释放锁源码中完成这个的是`shouldParkAfterFailedAcquire`方法，所以这里返回`true`

这个`doSignal`方法遍历`conditionqueue`，只要成功将一个节点转移到`waitqueue`中，就停止，这样整个唤醒过程就完成了，接着等待该线程在`waitqueue`中被唤醒

从源码看出，`await`被唤醒顺序按照谁先被`await`，因为是从`conditionqueue`第一个节点被放到`waitqueue`最后一个节点，被`signal`之后仍要等待`waitqueue`前面线程都运行完

### signalAll()

```
public final void signalAll() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignalAll(first);
}


private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        first.nextWaiter = null;
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```

和`signal()`方法相比，这里跳出循环的条件少了`transferForSignal`的返回值，所以这里会将`conditionqueue`中所有节点转移到`waitqueue`中