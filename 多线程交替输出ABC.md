title: 多线程交替输出ABC
categories: java基础
tags: 
	- 多线程
---

### synchronized

```
public class Mythread2 {
	public static void main(String[] args) throws InterruptedException {
		Object A = new Object();
		Object B = new Object();
		Object C = new Object();
		new ThreadABC(C,A,'A').start();
		Thread.sleep(100);
		new ThreadABC(A,B,'B').start();
		Thread.sleep(100);
		new ThreadABC(B,C,'C').start();
	}
}

class ThreadABC extends Thread{
	private Object pre;
	private Object self;
	private char c;
	public ThreadABC(Object pre, Object self, char c) {
		this.pre = pre;
		this.self = self;
		this.c = c;
	}
	@Override
	public void run() {
		// TODO Auto-generated method stub
		super.run();
		while(true){
			synchronized (pre) {
				synchronized (self) {
					System.out.println(c);
					self.notifyAll();//执行完同步代码块才会释放锁
				}
				try {
					pre.wait();
				} catch (InterruptedException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				}
			}
		}
	}
	
}
```

### ReentrantLock

```
public class Mythread3 {

	public static void main(String[] args) {
		ReentrantLock lock = new ReentrantLock();
		Condition A = lock.newCondition();
		Condition B = lock.newCondition();
		Condition C = lock.newCondition();
		new ThreadReen(C, A, 'A', lock).start();
		new ThreadReen(A, B, 'B', lock).start();
		new ThreadReen(B, C, 'C', lock).start();
	}
	
}

class ThreadReen extends Thread{
	private Condition pre;
	private Condition self;
	private char c;
	private ReentrantLock lock;
	
	public ThreadReen(Condition pre, Condition self, char c, ReentrantLock lock) {
		this.pre = pre;
		this.self = self;
		this.c = c;
		this.lock = lock;
	}

	@Override
	public void run() {
		// TODO Auto-generated method stub
		super.run();
		while(true){
			lock.lock();
			System.out.println(c);
			self.signalAll();//执行完同步代码块才会释放锁
			try {
				pre.await();
			} catch (InterruptedException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
			lock.unlock();
		}
	}
	
}
```