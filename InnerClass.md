title: 内部类
categories: java基础
---

### 成员内部类

成员内部类是最普通的内部类，它在另一个类内部被定义，结构为

```
class Outter{

	class Inner{
	
	}

}
```

实例化内部类的方法为:

```
Outter.Inner inner = new Outter().new Inner();
```

1. 内部类中可以访问到外部类中所有方法和成员变量，即使是private或者static

2. **普通内部类中不能定义static变量和方法，但可以定义静态常量**

3. 外部类要访问内部类变量，必须先实例化一个内部类

4. 静态内部类的访问可以直接用Outter.Inner(初始化时new Outter.Inner())，访问内部类静态成员变量Outter.Inner.valueName

5. 静态内部类不能访问外部类的属性方法

### 局部内部类

局部内部类是定义在一个方法或者一个作用域里面的类，它和成员内部类的区别在于局部内部类的访问仅限于方法内或者该作用域内，结构为

```
class Outter{
	public void print(final int a){
		class Inner{
			public void say(){
				System.out.println(a);
			}
		}
		new Inner().say();
	}
}
```

1.拥有普通内部类特性

2.内部类的modifier只能为default

3.在内部类和方法传入的参数必须是final类型

### 匿名内部类

1.一个类用于继承其他类或是实现接口，并不需要增加额外的方法，只是对继承方法的事先或是覆盖。

2.只是为了获得一个对象实例，不需要知道其实际类型。

3.类名没有意义，也就是不需要使用到。

例如在创建线程时

```
 Thread thread = new Thread(new Runnable(){
	       public void run(){
					
		}
	});
```

java8我们可以使用lambda来简化匿名内部类创建，以上代码可简化为

```
Thread thread = new Thread(()->{});//Runnable为函数式接口
```

### 理解

关于匿名内部类和局部内部类访问局部变量或者形参时，这是因为局部变量或形参与内部类的生存周期不同，当局部变量或形参被回收后，内部类的实例仍在，若此时访问它们，将会出错，所以局部变量和形参都是以构造函数的参数形式传入到匿名内部类中，相当于是复制了一份放到匿名内部类中，为了保持原始数据和复制后的数据相同，所以要使用final。[查看原理](http://blog.csdn.net/u014805893/article/details/53310521?locationNum=5&fps=1)

### java8新修改

**java8开始匿名内部类使用的外部变量不再被强制用final修饰。外部变量要么是final的，要么自初始化后值不会被改变，这两种都是可以在匿名内部类中使用且编译通过。**

**Local variable br defined in an enclosing scope must be final or effectively final**