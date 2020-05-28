title: ReentrantReadWriteLock的共享锁和独享锁
categories: java基础
tags: 
	- 多线程
---

### 共享锁和独享锁操作state

ReentrantReadWriteLock中共享锁和独享锁对锁的操作还是共同操作一个变量`state`，那怎么区分这个`state`是共享锁取得锁还是独享锁取得锁呢

从ReentrantReadWriteLock的源码我们来看

```
static final int SHARED_SHIFT   = 16

static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1

/** Returns the number of shared holds represented in count  */
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
/** Returns the number of exclusive holds represented in count  */
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

下面两个方法传入`state`，返回共享锁取得锁次数和独享锁取得锁次数

其中获取共享锁次数的方式为将`state`无符号右移16位

获取独享锁次数的方式为将`state`与`EXCLUSIVE_MASK`进行并运算，`EXCLUSIVE_MASK`为32位int型，前16位为0，后16位为1

所以`state`区分独享锁和共享锁次数的方法为，将`state`32位一分为二，前16位用来表示共享锁获取次数，后16位用来表示独享锁获取次数

### 共享锁和独享锁在获取锁时的区别

读锁用的就是共享锁，我们知道多个线程是可以同时获取读锁的

读锁获取锁的源码如下

```
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

而写锁获取锁的源码如下

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

都是一样的先尝试获取锁，失败之后放入`waitingqueue`等待唤醒

我们看获取锁有什么区别，下面是获取独享锁

```
protected final boolean tryAcquire(int acquires) {
    /*
     * Walkthrough:
     * 1. If read count nonzero or write count nonzero
     *    and owner is a different thread, fail.
     * 2. If count would saturate, fail. (This can only
     *    happen if count is already nonzero.)
     * 3. Otherwise, this thread is eligible for lock if
     *    it is either a reentrant acquire or
     *    queue policy allows it. If so, update state
     *    and set owner.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    int w = exclusiveCount(c);
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0)
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

当c!=0且w==0时，说明存在线程获取了读锁，没有线程获取了写锁，此时想要获取写锁，则需要进入waitingqueue排队

当c==0时，即没有线程获取写锁和读锁，线程直接获取写锁

读锁想重入写锁需要进入waitingqueue

下面是获取共享锁

```
protected final int tryAcquireShared(int unused) {
    /*
     * Walkthrough:
     * 1. If write lock held by another thread, fail.
     * 2. Otherwise, this thread is eligible for
     *    lock wrt state, so ask if it should block
     *    because of queue policy. If not, try
     *    to grant by CASing state and updating count.
     *    Note that step does not check for reentrant
     *    acquires, which is postponed to full version
     *    to avoid having to check hold count in
     *    the more typical non-reentrant case.
     * 3. If step 2 fails either because thread
     *    apparently not eligible or CAS fails or count
     *    saturated, chain to version with full retry loop.
     */
    Thread current = Thread.currentThread();
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        return -1;
    int r = sharedCount(c);
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}
```
共享锁对应的也先获取独享锁获取次数，如果已经有获取独享锁的线程了，那就进waitingqueue

如果此时没有线程获取独享锁，且waitingqueue中也没有exclusive模式的Node在排队，直接获取读锁，并且因为不像独享锁，它的特性exclusive使得它可以用`state`来记录重入的次数，而共享锁可以多个线程同时获取锁，所以这里用了`ThreadLocal`在线程中存放每个读锁重入的次数

如果是写锁要重入读锁，可以直接获取读锁

**根据以上特性我们可以初步总结到，当没有线程获取到写锁时，多个线程都可以获取到读锁，并且不需要进入waitingqueue，但是当有线程尝试获取写锁时，由于已经有线程获取了读锁，所以它要进入waitingqueue，此时后续想要获取读锁的线程也只能进入waitingqueue**

### 共享锁和独享锁唤醒successor的区别

#### setHeadAndPropagate

我们知道，独享锁只有在`unlock()`的时候才会唤醒下个被挂起的线程，而waitingqueue中共享锁模式的Node总共有两次换新下个被挂起的线程，一次在被唤醒获取到锁之后，一次是在`unlock()`

```
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

当Node中线程获取到锁之后，会将自己设置为头节点Head，但是共享锁比起独享锁在这一步多了操作

```
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

可以看到最后调用了`doReleaseShared()`方法，唤醒下一个节点如果这个节点模式也为`Shared`

而在`unlock()`中

```
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

也调用了`doReleaseShared()`方法唤醒下一个节点

```
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

这就是唤醒下一个节点的源码，是一个for循环，只有`h==head`才能跳出循环

共享锁多出的这一步获取锁之后唤醒下一个模式为`shared`的节点的线程，目的应该是将多个连续的`shared`节点的线程都唤醒

但是在这一步可能会出现的问题是，如果唤醒下一个`shared`，在它获取锁将自己设置为Head之前，此时来到`h==head`，则循环结束，正好是一个链式唤醒
如果在下一个`shared`获取锁将自己设置为Head之后，此时h!=head，这样会造成什么后果，多个线程在执行自己代码之前，循环唤醒下个节点，这叫"调用风暴"，这极大地加速了唤醒后继节点的速度，提升了效率，同时该方法内部的CAS操作又保证了多个线程同时唤醒一个节点时，只有一个线程能操作成功。
直到下一个`exclusive`，因为有线程持有读锁，所以唤醒的线程获取不了写锁，唤醒终止，此时`head`不变，`h==head`恒成立，所有`doReleaseShared`循环结束，获取读锁的线程执行自己代码

#### unlock

两种锁在这一步相似

```
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current))
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        --rh.count;
    }
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```

最后这个for循环作用，只用当读锁全部释放完，且没有线程占有写锁的情况下，共享锁会唤醒下一个节点

为什么要强调没有线程占有写锁，我想可能是写锁重入读锁的问题，当写锁重入读锁，读锁`unlock()`的时候，也就是这一步，加这个条件，此时会尝试唤醒下一个线程，并且会改变`waitStatus`的状态为0(每次要唤醒下一个节点，都会把头节点`waitStatus`从`signal`变为0)，会导致什么后果呢

```
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

`waitStatus`变为0，这里无法唤醒下一个waitingqueue节点的线程