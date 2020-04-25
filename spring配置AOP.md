title: Spring配置AOP
categories: 框架
tags: 
	- Spring

---

### AOP面向切面编程

AOP的底层原理就是动态代理，就是对目标方法进行增强，**将相同逻辑的重复代码横向抽取出来，使用动态代理技术将这些重复代码织入到目标对象方法中，实现和原来一样的功能**，例如访问控制，事务管理以及日志记录

### AOP

- 切面(Aspect)： 一些横跨多个类的公共模块，如日志，安全，事务等。简单地说，日志模块就是一个切面
- 连接点(Joint Point)： 目标类中插入代码的方法，在`Spring AOP`中连接点为方法
- 通知(Advice)： 在连接点插入的实际代码(即切面的方法)
- 切点(Pointcut)： 定义了连接点的条件，匹配所有对应的`joint Point`
- 目标对象(target object): 目标对象(可以理解为连接点所在的类)
- 代理对象(proxy object): 目标对象的代理对象
- 织入(Weaving): 利用代理实现切面方法增强连接点方法的过程叫织入

#### Aspect

声明一个切面有两种方法，一种是通过xml配置的方法

```
<bean id="myAspect" class="org.xyz.NotVeryUsefulAspect">
    <!-- configure properties of the aspect here -->
</bean>
```

当然，配置的这个类上需要有`@Aspect`注解

第二种就是注解配置的方法

在需要声明为切面的类上配置`@Aspect`注解，但是，注意Spring文档中，However, note that the @Aspect annotation is not sufficient for autodetection in the classpath. For that purpose, you need to add a separate @Component annotation

意思就是采用注解的方式声明，不仅需要`@Aspect`注解，还需要配置上`@Component`注解才行

#### Advice

总共有五种`Advice`

- Before advice: 在执行连接点方法之前被调用，并且无法阻止方法执行，除非抛出异常
- After returning advice: 在连接点方法被完整执行之后，返回值并且没有抛出异常
- After throwing advice: 当方法执行后抛出异常之后被调用
- After advice: 在方法运行完之后，无论是正常执行或者抛出异常都被调用
- Around advice: 围绕连接点方法被调用，可以在执行连接点方法前后执行我们自定义的操作，干预方法例如修改返回值和修改方法的调用参数等

#### Pointcut

`Pointcut`用来确定连接点，它的值通过`point designator`指定，规则如下

1. execution

```
@Pointcut("execution(public String org.baeldung.dao.FooDao.findById(Long))")

@Pointcut("execution(* org.baeldung.dao.FooDao.*(..))")
```

通过第一种指定特定方法或者第二种模糊匹配

2. within

within是用来指定类型的，指定类型中的所有方法将被拦截

```
@Pointcut("within(org.baeldung.dao.FooDao)")

@Pointcut("within(org.baeldung..*)")
```

指定某个类或者某个包下所有类的所有方法(子类型无效)

3. this and target

```
@Pointcut("target(org.baeldung.dao.BarDao)")

@Pointcut("this(org.baeldung.dao.FooDao)")
```

其中target指代被代理的类(连接点方法所在的类)，this表示代理类，匹配条件为是给定值的类的实例

4. args

args是用来匹配方法参数的

- `args()`匹配任何不带参数的方法。
- `args(java.lang.String)`匹配任何只带一个参数，而且这个参数的类型是`String`的方法。
- `args(..)`带任意参数的方法。
- `args(java.lang.String,..)`匹配带任意个参数，但是第一个参数的类型是`String`的方法。
- `args(..,java.lang.String)`匹配带任意个参数，但是最后一个参数的类型是`String`的方法。

5. @target

与上面的target是不同的，这个是指定目标类(子类无效)要有指定的注解才能匹配，

```
@Pointcut("@target(org.springframework.stereotype.Repository)")
```

6. @args

用来匹配参数带有指定注解的方法

```
@Pointcut("@args(com.elim.spring.support.MyAnnotation)")
```

7. @within

和`@target`差不多，但是`@within`范围更大，被注解类的子类中未被覆写的方法也匹配

```
@Pointcut("@target(org.springframework.stereotype.Repository)")
```


8. @annotation

@annotation用于匹配方法上拥有指定注解的情况。

```
@Point("@annotation(com.elim.spring.support.MyAnnotation)")
```

### 五种通知方法

```
@Aspect
@Component
public class Log {
	@Pointcut("execution(* crj.ssm.service.*.*(..))")
	private void startLog() {
		
	}

	@Before("startLog()")
	public void beforeMethod(JoinPoint point) {
		System.out.println("@Before方法开始");
		System.out.println("getSignature方法："+point.getSignature().getName());
		//获取目标方法的参数，返回的是Object[]
		System.out.println("getArgs方法： "+point.getArgs());
		//返回目标方法所在的类
		System.out.println("getTarget方法："+point.getTarget());
		//也是返回目标方法所在的类
		System.out.println("getThis方法" +point.getThis());
		System.out.println("@Before方法结束");
	}

	@Around("startLog()")
	public Object aroundMethod(ProceedingJoinPoint point) throws Throwable{
		System.out.println("@aroundMethod方法开始");
		//同上
		System.out.println(point.getArgs());
		Object object = point.proceed(point.getArgs());
		System.out.println("@aroundMethod方法结束");
		return object;
	}

	@After("startLog()")
	public void afterMethod(JoinPoint point) {
		System.out.println("@afterMethod方法开始");
		System.out.println("this is after-method");
		System.out.println("@afterMethod方法结束");
	}

	@AfterReturning(value = "startLog()", returning = "result")
	public void afterReturningMethod(JoinPoint point, Object result) {
		System.out.println("@AfterReturning方法开始");
		System.out.println(result);
		System.out.println("@AfterReturning方法结束");
	}

	@AfterThrowing(value = "startLog()", throwing = "ex")
	public void afterThrowingMethod(JoinPoint point, Throwable ex) {
		System.out.println("this is afterThrowing-method");
	}
}
```

执行顺序为

```
prehandle方法
@aroundMethod方法开始
getArgs方法[Ljava.lang.Object;@493d696c
@Before方法开始
getSignature方法：selectById
getArgs方法： [Ljava.lang.Object;@72e6ef43
getTarget方法：crj.ssm.service.impl.PersonServiceImpl@177185be
getThis方法crj.ssm.service.impl.PersonServiceImpl@177185be
@Before方法结束
-----------------------------------------
22:44:37.933 [http-bio-8080-exec-2] DEBUG org.mybatis.spring.SqlSessionUtils - Creating a new SqlSession
22:44:37.946 [http-bio-8080-exec-2] DEBUG org.mybatis.spring.SqlSessionUtils - Registering transaction synchronization for SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@711b6130]
22:44:37.960 [http-bio-8080-exec-2] DEBUG o.m.s.t.SpringManagedTransaction - JDBC Connection [com.mchange.v2.c3p0.impl.NewProxyConnection@21118e0c] will be managed by Spring
22:44:37.967 [http-bio-8080-exec-2] DEBUG crj.ssm.dao.PersonDao.selectById - ==>  Preparing: SELECT * from person where id = ? 
22:44:38.029 [http-bio-8080-exec-2] DEBUG crj.ssm.dao.PersonDao.selectById - ==> Parameters: 1(Integer)
22:44:38.065 [http-bio-8080-exec-2] DEBUG crj.ssm.dao.PersonDao.selectById - <==      Total: 1
22:44:38.078 [http-bio-8080-exec-2] DEBUG org.mybatis.spring.SqlSessionUtils - Releasing transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@711b6130]
@aroundMethod方法结束
@afterMethod方法开始
this is after-method
@afterMethod方法结束
@AfterReturning方法开始
crj.ssm.entity.Person@77e5c255
@AfterReturning方法结束
22:44:38.106 [http-bio-8080-exec-2] DEBUG org.mybatis.spring.SqlSessionUtils - Transaction synchronization committing SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@711b6130]
22:44:38.108 [http-bio-8080-exec-2] DEBUG org.mybatis.spring.SqlSessionUtils - Transaction synchronization deregistering SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@711b6130]
22:44:38.108 [http-bio-8080-exec-2] DEBUG org.mybatis.spring.SqlSessionUtils - Transaction synchronization closing SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@711b6130]
postHandle方法
afterCompletion方法
```

先执行`@Around`方法，直到调用proceed方法前，调用`@Before`方法，然后就是执行目标方法，执行完目标方法，开始执行`@AfterMethod`方法，然后`@Around`方法执行完proceed返回值，然后是`@AfterReturning`方法。