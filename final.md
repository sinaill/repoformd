title: final关键字
categories: java基础
---

### final用法

1.final用来修饰类时，该类不可被继承

2.final用来修饰方法时，方法不可被复写

3.final修饰的基本变量时，值不可变，修饰的引用类型时，引用不可变，值可变

### final类型成员变量初始化

1.final修饰的成员变量没有默认值

2.final初始化可以在三个地方
(1)声明的时候初始化
(2)构造函数里初始化
(3)要是没有static修饰的话可以在非静态块里初始化,要是有static修饰的话可以在静态块里初始化

3.使用final成员前要确保已经初始化

### 深入理解

1.当final变量是基本数据类型以及String类型时，如果在编译期间能知道它的确切值，则编译器会把它当做编译期常量使用。也就是说在用到该final变量的地方，相当于直接访问的这个常量，不需要在运行时确定。

```
public class Test {
    public static void main(String[] args)  {
        String a = "hello2"; 
        final String b = "hello";
        String d = "hello";
        String c = b + 2; 
        String e = d + 2;
        System.out.println((a == c));
        System.out.println((a == e));
    }
}
```

`true false`

不过要注意，只有在编译期间能确切知道final变量值的情况下，编译器才会进行这样的优化

```
public class Test {
    public static void main(String[] args)  {
        String a = "hello2"; 
        final String b = getHello();
        String c = b + 2; 
        System.out.println((a == c));
 
    }
     
    public static String getHello() {
        return "hello";
    }
}
```

`false`

2.final也可以用来修饰方法的形参，防止方法内部修改形参

3.匿名内部类使用的外部变量必须为final

**外部类和外部局部变量是作为构造方法的参数传入匿名内部类的**

为什么在匿名内部类中引用外部对象要加final修饰符呢，因为，在匿名内部类中引用的外部对象受到外部线程的作用域的制约有其特定的生命周期，以线程为例，当外部的变量生命周期已经完结之后，内部的线程还在运行，怎么样解决这个外部生命周期已经结束而在内部却需要继续使用呢，这个时候就需要在外部变量中添加final修饰符，**其实内部匿名类使用的这个变量就是外部变量的一个“复制品”**，即使外部变量生命周期已经结束，内部的“复制品“依然可用。并且**final保证复制品与原始变量保持一致**



