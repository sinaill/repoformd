title: volatile
categories: java基础
tags: 
	- 多线程
---

### 简介

Java编程语言允许线程访问共享变量，为了确保共享变量能被准确和一致地更新，线程应该确保通过排他锁单独获得这个变量。Java语言提供了volatile，在某些情况下比锁要更加方便。如果一个字段被声明成volatile，Java线程内存模型确保所有线程看到这个变量的值是一致的。

### 原理

![](http://wx3.sinaimg.cn/large/96b7c0f4gy1g0t2gabww2j20hj08edfq.jpg)

在 Java 内存模型中，所有的变量都存储在主存中，同时每个线程都拥有自己的工作线程，用于提高访问速度。线程会从主存中拷贝变量值到自己的工作内存中，然后在自己的工作线程中操作变量，而不是直接操作主存中的变量，由于每个线程在自己的内存中都有一个变量的拷贝，就会造成变量值不一致的问题。

如果对声明了volatile的变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存，同时让其它工作内存中这个变量的缓存失效，再次使用该变量的时候重新从主存中读取数据。

```
public class VolatileTest extends  Thread{
    private boolean isRunning = true;
    public boolean isRunning(){
        return isRunning;
    }
    public void setRunning(boolean isRunning){
        this.isRunning= isRunning;
    }
    public void run(){
        System.out.println("进入了run...............");
        while (isRunning){}
        System.out.println("isRunning的值被修改为为false,线程将被停止了");
    }
    public static void main(String[] args) throws InterruptedException {
        VolatileTest volatileThread = new VolatileTest();
        volatileThread.start();
        Thread.sleep(1000);
        volatileThread.setRunning(false);   //停止线程
    }
}
```

以上结果为：线程进入死循环，将`isRunning`变量设为`volatile`变量后，线程成功停止。

### volatile的特性

1. 可见性，当一个共享变量被`volatile`修饰时，它会保证修改的值立即被更新到主存，所以对其他线程是可见的。当其他线程需要读取该值时，其他线程会去主存中读取新值。相反普通的共享变量不能保证可见性，因为普通共享变量被修改后并不会立即被写入主存，何时被写入主存也不确定。当其他线程去读取该值时，此时主存可能还是原来的旧值，这样就无法保证可见性
2. 有序性，java内存模型中允许编译器和处理器对指令进行重排序，虽然重排序过程不会影响到单线程执行的正确性，但是会影响到多线程并发执行的正确性。这时可以通过`volatile`来保证有序性，除了`volatile`,也可以通过`synchronized`和`Lock`来保证有序性。`synchronized`和`Lock`保证每个时刻只有一个线程执行同步代码，这相当于让线程顺序执行同步代码，从而保证了有序性。如果不考虑原子性操作的话`volatile`比`synchronized`和`Lock`更轻量级，成本更低。
3. 不保障原子性`volatile`关键字只能保证共享变量的可见性和有序性
 
### 非原子性

`volatile`**仅仅用来保证该变量对所有线程的可见性，但不保证原子性**

```
public class Test{

	volatile static int n;
	
	    public static void main(String[] args) throws InterruptedException {
	        Thread t1 = new Thread(()->{
	        	for (int i = 0; i < 100; i++) {
					n++;
				}
	        });
	        Thread t2 = new Thread(()->{
	        	for (int i = 0; i < 100; i++) {
					n++;
				}
	        });
	        t1.start();
	        t2.start();
	        t1.join();
	        t2.join();
	        System.out.println(n);
	    }
}
```
在多次运行中，以上程序输出结果不全是200

不要将volatile用在getAndOperate场合（这种场合不原子，需要再加锁），仅仅set或者get的场景是适合volatile的。

volatile在进行自增自减操作的时候，不能保证原子性，n++实际上是三个操作，不是原子操作，先从主内存读取值到线程工作内存，然后在工作内存中对数据进行处理，最后刷新到主内存中，让其它线程可见。**一旦在这三步中线程被挂起，就会出现上面这种结果。**

### 禁止重排序

下面是一个单例模式的代码

```
public class Singleton {
    private volatile static Singleton instance;
    private Singleton (){}
    public static Singleton getInstance() {
        if (instance == null) {
			synchronized(this.class){
				if (instance == null){
					instance = new Singleton();
				}
        }
        return instance;
    }
}
```

为什么要对`instance`使用`volatile`，`instance = new Singleton()`这句，这并非是一个原子操作，事实上在 JVM 中这句话大概做了下面 3 件事情，给`instance`分配内存，调用 Singleton 的构造函数来初始化成员变量，将instance对象指向分配的内存空间（执行完这步`instance`就为非 null 了）。但是在 JVM 的即时编译器中存在指令重排序的优化。也就是说上面的第二步和第三步的顺序是不能保证的，最终的执行顺序可能是 1-2-3 也可能是 1-3-2。如果是后者，则在 3 执行完毕、2 未执行之前，线程2进入第一个`instance==null`，这时`instance`已经是非null了（但却没有初始化），所以线程二会直接返回`instance`，然后使用，然后顺理成章地报错。

总结
修改`volatile`变量时会强制将修改后的值刷新的主内存中

修改`volatile`变量后会导致其他线程工作内存中对应的变量值失效。因此，再读取该变量值的时候就需要重新从读取主内存中的值

在访问`volatile`变量时不会执行加锁操作，因此也就不会使执行线程阻塞，因此`volatile`变量是一种比`sychronized`关键字更轻量级的同步机制。

性能方面，`volatile`的读性能消耗与普通变量几乎相同，但是写操作稍慢，因为它需要在本地代码中插入许多内存屏障指令来保证处理器不发生乱序执行。

[对volatile更详细的参考](http://www.importnew.com/24082.html)
[彻底理解volatile](https://juejin.im/post/5ae9b41b518825670b33e6c4)