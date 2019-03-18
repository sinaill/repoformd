title: 接口与抽象类
categories: java基础
---

### interface

1.接口中方法默认修饰符 public abstract,属性默认修饰符 public static final

2.接口可继承多接口

3.接口中可定义静态方法(有方法体)

4.接口中可定义default方法(有方法体，实现类可调用)

### abstract class

1.抽象类中变量可以为普通变量

2.抽象类中可以没有抽象方法

3.可以实现接口

### 区别

1.接口中除了default和静态方法可以定义方法体，其余不能。而抽象类中无此限制

2.接口中没有构造器，抽象类可以有

3.抽象方法修饰符可以为public,protected,default，而接口中方法要么为public，要么为有方法体的default

4.接口与普通类不同，不能有Main方法，抽象类可以

5.接口不能继承抽象类，抽象类可以实现接口