title: semaphore
categories: java基础
tags: 
	- 多线程
---

### semaphore

java 1.5 后 引入Semaphore类

计数信号量(Counting Semaphore)用来控制同时访问某个特定资源的操作数量，或者同时执行某个指定操作的数量。计数信号量还可以用来实现某种资源池，或者对容器施加边界。

Semaphore中管理着一组虚拟的许可(permit)，许可的初始数量可通过构造函数来指定。在执行操作时可以首先获得许可(只要还有剩余的许可)，并在使用以后释放许可。如果没有许可，那么acquire将阻塞直到有许可(或者直到被中断或者操作超时)。release方法将返回一个许可给信号量。计算信号量的一种简化形式是二值信号量，即初始值为1的Semaphore。二值信号量可以用做互斥体(mutex)，并具备不可重入的加锁语义:谁拥有这个唯一的许可，谁就拥有了互斥锁。

概念上讲，一个信号量管理许多的许可证(permit)。为了通过信号量，线程通过调用`acquire`请求许可。其实没有实际的许可对象,信号量仅维护一个计数。许可的数目是固定的，由此限制了通过的线程数量。其他线程可以通过调用 `release`释放许可。而且，许可不是必须由获取它的线程释放。**事实上，任何线程都可以释放任意数目的许可，这可能会增加许可数目以至于超出初始数目。**

### 方法摘要

```
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```

构造方法，也分公平锁和非公平锁，公平锁遵循FIFO原则

- `void acquire()`

从此信号量获取一个许可，在提供一个许可前一直将线程阻塞，否则线程被中断

- `void release()`

释放一个许可，将其返回给信号量

- `int availablePermits()`

返回此信号量中当前可用的许可数。

- `boolean hasQueuedThreads()`

查询是否有线程正在等待获取

### 使用

#### 模拟加同步锁

```
class Store{
    private static int storage = 0;
    Semaphore semaphore = new Semaphore(1);
    public void add(int n) throws InterruptedException {
        semaphore.acquire();
        storage += n;
        semaphore.release();
    }

    public void out(){
        System.out.println(storage);
    }
}
```

设置信号量为1，这样可以保证多线程并发调用`add`方法时，只能单个线程运行，起到加锁效果

#### 生产者消费者模型

```

public class Test { 
    int count = 0; 
    final Semaphore put = new Semaphore(5);// 初始令牌个数 
    final Semaphore get = new Semaphore(0); 
    final Semaphore mutex = new Semaphore(1); 
 
    public static void main(String[] args) { 
        Test t = new Test(); 
        new Thread(t.new Producer()).start(); 
        new Thread(t.new Consumer()).start(); 
        new Thread(t.new Consumer()).start(); 
        new Thread(t.new Producer()).start(); 
    } 
 
    class Producer implements Runnable { 
        @Override 
        public void run() { 
            for (int i = 0; i < 5; i++) { 
                try { 
                    Thread.sleep(1000); 
                } catch (Exception e) { 
                    e.printStackTrace(); 
                } 
                try { 
                    put.acquire();// 注意顺序 
                    mutex.acquire(); 
                    count++; 
                    System.out.println("生产者" + Thread.currentThread().getName() 
                            + "已生产完成，商品数量：" + count); 
                } catch (Exception e) { 
                    e.printStackTrace(); 
                } finally { 
                    mutex.release(); 
                    get.release(); 
                } 
 
            } 
        } 
    } 
 
    class Consumer implements Runnable { 
 
        @Override 
        public void run() { 
            for (int i = 0; i < 5; i++) { 
                try { 
                    Thread.sleep(1000); 
                } catch (InterruptedException e1) { 
                    e1.printStackTrace(); 
                } 
                try { 
                    get.acquire();// 注意顺序 
                    mutex.acquire(); 
                    count--; 
                    System.out.println("消费者" + Thread.currentThread().getName() 
                            + "已消费，剩余商品数量：" + count); 
                } catch (Exception e) { 
                    e.printStackTrace(); 
                } finally { 
                    mutex.release(); 
                    put.release(); 
                } 
            } 
        } 
    } 
}
```

生产者受`put`初始信号量的限制，最多5个线程生产，生产完毕开始释放`get`信号量，虽然`get`信号量初始为0，但释放的信号量不受限制，所以生产完消费者线程开始消费，消费完毕又释放`put`信号量，启动生产

其中要注意`mutex`，初始信号量为1，即起到加锁效果，每次只能有一个线程运行，因为`count`为共享变量，还要注意`mutex`和`get`、`put`获取信号的顺序，`mute`确保只有本线程在运行，如果在`acquire`阻塞，将形成死锁