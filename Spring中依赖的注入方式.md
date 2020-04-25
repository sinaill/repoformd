title: Spring中依赖的注入方式
categories: 框架
tags: 
	- Spring

---

### 构造器注入

构造器注入作用在第二次后置处理器调用，找到合适构造器后，用它实例化类的同时，注入依赖属性

```
public class userServiceImpl implements userService{

	private userDao userdao;
	@Autowired
	public userServiceImpl(userDao userdao){
		this.userdao = userdao;
	}

	@Override
	public void query() {
		userdao.query();
	}
}
```

`Autowired`注解可省略，当不需要无参构造函数时

### set方法注入

```
public class userServiceImpl implements userService{

	private userDao userdao;
	@Override
	public void query() {
		userdao.query();
	}
	@Autowired
	public void setuserDao(userDao userdao){
		this.userdao = userdao;
	}


}
```

### 普通属性注入

使用`@Value`注解

### Method Injection方法注入

Spring文档

In most application scenarios, most beans in the container are singletons. When a singleton bean needs to collaborate with another singleton bean or a non-singleton bean needs to collaborate with another non-singleton bean, you typically handle the dependency by defining one bean as a property of the other. A problem arises when the bean lifecycles are different. Suppose singleton bean A needs to use non-singleton (prototype) bean B, perhaps on each method invocation on A. The container creates the singleton bean A only once, and thus only gets one opportunity to set the properties. The container cannot provide bean A with a new instance of bean B every time one is needed.

当一个singleton的Bean A，内部依赖为prototype的Bean B，因为A只初始化一次，所以它的内部依赖B始终为同一个实例，如果要使每次用到B时都是新实例，就需要用到Method Injection

文档要我们实现`ApplicationContextAware`接口，从而获得容器对象

```
public class CommandManager implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    public Object process(Map commandState) {
        // grab a new instance of the appropriate Command
        Command command = createCommand();
        // set the state on the (hopefully brand new) Command instance
        command.setState(commandState);
        return command.execute();
    }

    protected Command createCommand() {
        // notice the Spring API dependency!
        return this.applicationContext.getBean("command", Command.class);
    }

    public void setApplicationContext(
            ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }
}
```

官方例子中，实现接口，获取容器对象，每次使用这个类时都手动从容器获取这个对象的实例，从而每次都是新的实例B

### depends-on

文档中

The depends-on attribute can explicitly force one or more beans to be initialized before the bean using this element is initialized

depends-on属性可以强制在初始化使用这个元素的类之前初始化一个指定的类


