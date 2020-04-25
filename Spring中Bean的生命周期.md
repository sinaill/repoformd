title: Spring中Bean的生命周期
categories: 框架
tags: 
	- Spring

---

### 测试代码准备

#### 测试类

Sping Bean定义

```
public interface userDao {
	void query();
}


public class userDaoImpl implements userDao, InitializingBean {
	@Override
	public void query() {
		System.out.println("query");
	}

	@PostConstruct
	public void PostConstruct(){
		System.out.println("this is PostConstruct method");
	}
	@PreDestroy
	public void destroy(){
		System.out.println("this is a PreDestroy method");
	}

	@Override
	public void afterPropertiesSet(){
		System.out.println("this is a afterPropertiesSet method");
	}

	public void init(){
		System.out.println("this is init method");
	}
}
```

配置类

```
@Configuration
@ComponentScan("org.springframework.test")
public class springConfig {
	@Bean(initMethod = "init", name = "userDaoImpl")
	public userDaoImpl get(){
		return new userDaoImpl();
	}
}
```

自定义`BeanPostProcessor`

```
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		System.out.println("this is BeanPostProcessor method");
		return bean;
	}
}
```

启动`Spring`容器

```
public class Test {
	public static void main(String args[]){
		AnnotationConfigApplicationContext annotationConfigApplicationContext =
				new AnnotationConfigApplicationContext(springConfig.class);
		userDao userdao = (userDao) annotationConfigApplicationContext.getBean("userDaoImpl");
		userdao.query();
	}
}
```

### Bean的创建和初始化

`AnnotationConfigApplicationContext`的父类`GenericApplicationContext`的无参构造函数创建了`Spring`的工厂

```
public GenericApplicationContext() {
	this.beanFactory = new DefaultListableBeanFactory();
}
```

然后在它自身构造函数中的`refresh()`方法中开始创建和初始化`Spring Bean`

```
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
	this();
	register(componentClasses);
	refresh();
}
```

`refresh`方法中

```
// Instantiate all remaining (non-lazy-init) singletons.
finishBeanFactoryInitialization(beanFactory);
```

这里调用同个类中的`finishBeanFactoryInitialization`方法开始创建所有非懒加载的单例`Spring Bean`，在这个方法中调用之前创建好的工厂`DefaultListableBeanFactory`实例来创建`Spring Bean`

```
// Instantiate all remaining (non-lazy-init) singletons.
beanFactory.preInstantiateSingletons();
```

查看这个方法

```
@Override
public void preInstantiateSingletons() throws BeansException {
	if (logger.isTraceEnabled()) {
		logger.trace("Pre-instantiating singletons in " + this);
	}

	// Iterate over a copy to allow for init methods which in turn register new bean definitions.
	// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
	//BeanDefinition是存储要由Spring创建的类的信息，例如是否懒加载，单例等
	//beanDefinitionNames指代要实例化的类的名字
	List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
	//遍历开始实例化Spring Bean
	// Trigger initialization of all non-lazy singleton beans...
	for (String beanName : beanNames) {
		RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
		if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
			if (isFactoryBean(beanName)) {
				Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
				if (bean instanceof FactoryBean) {
					final FactoryBean<?> factory = (FactoryBean<?>) bean;
					boolean isEagerInit;
					if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
						isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
										((SmartFactoryBean<?>) factory)::isEagerInit,
								getAccessControlContext());
					}
					else {
						isEagerInit = (factory instanceof SmartFactoryBean &&
								((SmartFactoryBean<?>) factory).isEagerInit());
					}
					if (isEagerInit) {
						getBean(beanName);
					}
				}
			}
			else {
				getBean(beanName);
			}
		}
	}
```

从`getBean(beanName)`进入，直接找实例化部分的代码

在实例化`Spring Bean`之前，遍历了一次`BeanPostProcessor`，在`Spring Bean`生命周期中需要多次遍历这组`BeanPostProcessor`，它们是

![BeanPostProcessor.jpg](https://raw.githubusercontent.com/sinaill/pic/master/BeanPostProcessor.jpg)

除了`MyBeanPostProcessor`是我们自定义的`BeanPostProcessor`，其它都是由`Spring`注入

在实例化之前，进行了以下操作

```
@Nullable
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
	Object bean = null;
	if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
		// Make sure bean class is actually resolved at this point.
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			Class<?> targetType = determineTargetType(beanName, mbd);
			if (targetType != null) {
				bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
				if (bean != null) {
					bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
				}
			}
		}
		mbd.beforeInstantiationResolved = (bean != null);
	}
	return bean;
}
```

如果这里返回的`bean`不为`null`，则后面会跳过`Spring`实例化过程，将这里的`bean`作为实例化好的对象

```
@Nullable
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
	for (BeanPostProcessor bp : getBeanPostProcessors()) {
		if (bp instanceof InstantiationAwareBeanPostProcessor) {
			InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
			Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
			if (result != null) {
				return result;
			}
		}
	}
	return null;
}
```

这个方法属于`DefaultListableBeanFactory`，调用实现了`InstantiationAwareBeanPostProcessor`接口的`BeanPostProcessor`的`postProcessBeforeInstantiation`方法，传入的参数为要实例化的类的`Class`对象和类名

其中实现了这个接口的有`ImportAwareBeanPostProcessor`，`CommonAnnotationBeanPostProcessor`和`AutowiredAnnotationBeanPostProcessor`，这三个`BeanPostProcessor`均返回`null`

这里可以定义一个`BeanPostProcessor`实现`InstantiationAwareBeanPostProcessor`接口，可以根据传入的`Class`类型或者类名字决定要不要由我们自己来实例化这个类

如果返回的不是`null`而是我们自己实例化好的类，从代码看还要再遍历一次`BeanPostProcessor`

```
@Override
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
		throws BeansException {

	Object result = existingBean;
	for (BeanPostProcessor processor : getBeanPostProcessors()) {
		Object current = processor.postProcessAfterInitialization(result, beanName);
		//如果BeanPostProcessor返回一个null，则返回上一个BeanPostProcessor
		//返回的对象
		if (current == null) {
			return result;
		}
		result = current;
	}
	return result;
}
```

将由我们自己实例的对象传入，其他`Spring`注入的`BeanPostProcessor`基本直接返回对象本身，到我们自定义的那个替代`Spring`实例化的`BeanPostProcessor`时，也可以直接返回该对象，有点类似一条初始化链，中间一个`BeanPostProcessor`返回`null`时，链子断开然后返回上一个`BeanPostProcessor`处理后的对象

接着是实例化的代码

```
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
	// Make sure bean class is actually resolved at this point.
	Class<?> beanClass = resolveBeanClass(mbd, beanName);

	if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
		throw new BeanCreationException(mbd.getResourceDescription(), beanName,
				"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
	}

	Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
	if (instanceSupplier != null) {
		return obtainFromSupplier(instanceSupplier, beanName);
	}
	//工厂方法
	if (mbd.getFactoryMethodName() != null) {
		return instantiateUsingFactoryMethod(beanName, mbd, args);
	}

	// Shortcut when re-creating the same bean...
	boolean resolved = false;
	boolean autowireNecessary = false;
	if (args == null) {
		synchronized (mbd.constructorArgumentLock) {
			if (mbd.resolvedConstructorOrFactoryMethod != null) {
				resolved = true;
				autowireNecessary = mbd.constructorArgumentsResolved;
			}
		}
	}
	if (resolved) {
		if (autowireNecessary) {
			return autowireConstructor(beanName, mbd, null, null);
		}
		else {
			return instantiateBean(beanName, mbd);
		}
	}

	// Candidate constructors for autowiring?
	Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
	if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
			mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
		return autowireConstructor(beanName, mbd, ctors, args);
	}

	// Preferred constructors for default construction?
	ctors = mbd.getPreferredConstructors();
	if (ctors != null) {
		return autowireConstructor(beanName, mbd, ctors, null);
	}

	// No special handling: simply use no-arg constructor.
	return instantiateBean(beanName, mbd);
}
```



由于我们在配置类`springConfig`中使用的工厂方法来实例化类，所以这里走的

```
if (mbd.getFactoryMethodName() != null) {
	return instantiateUsingFactoryMethod(beanName, mbd, args);
}
```

找到我们在配置类中用`@Bean`注解的对应的方法进行实例化


这里如果我们不是用得工厂方法，而是正常的`@Repository`，则下面还要经过`determineConstructorsFromBeanPostProcessors`

```
@Nullable
protected Constructor<?>[] determineConstructorsFromBeanPostProcessors(@Nullable Class<?> beanClass, String beanName)
		throws BeansException {

	if (beanClass != null && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
				SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
				Constructor<?>[] ctors = ibp.determineCandidateConstructors(beanClass, beanName);
				if (ctors != null) {
					return ctors;
				}
			}
		}
	}
	return null;
}
```

主要是用来完成构造器注入的


实例化完之后，接着马上第二次遍历`BeanPostProcessor`

```
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
	for (BeanPostProcessor bp : getBeanPostProcessors()) {
		if (bp instanceof MergedBeanDefinitionPostProcessor) {
			MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
			bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
		}
	}
}
```

先放着，大致是完成了对注入元素注解的预解析

不过实现了这个接口的有`CommonAnnotationBeanPostProcessor`，`AutowiredAnnotationBeanPostProcessor`和`ApplicationListenerDetector`，最后这个类的实现的方法为将`beanName`添加到它类变量`singleonNames`中

`CommonAnnotationBeanPostProcessor`先调用父类`InitDestroyAnnotationBeanPostProcessor`解析`Bean`中有关生命周期的注解`@PostConstruct`和`@PreDestroy`，然后解析需要`@AutoWire`和`@Resource`注入的配置

然后是填充类属性

```
populateBean(beanName, mbd, instanceWrapper)
```

内部又遍历一次`BeanPostProcessor`

```
if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
	for (BeanPostProcessor bp : getBeanPostProcessors()) {
		if (bp instanceof InstantiationAwareBeanPostProcessor) {
			InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
			if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
				continueWithPropertyPopulation = false;
				break;
			}
		}
	}
}
```

这次与第一次遍历`BeanPostProcessor`向对应，第一次调用的`postProcessBeforeInstantiation`方法，这里则调用的`postProcessAfterInstantiation`

这里传入的参数为实例化好的对象和类名

其中实现了这个接口的有`ImportAwareBeanPostProcessor`，`CommonAnnotationBeanPostProcessor`和`AutowiredAnnotationBeanPostProcessor`，这三个`BeanPostProcessor`均返回`true`

如果有我们自定义的`BeanPostProcessor`并且实现了接口`InstantiationAwareBeanPostProcessor`，在这里返回`false`时，会停止后面的填充属性流程

然后又遍历一次`BeanPostProcessor`

```
for (BeanPostProcessor bp : getBeanPostProcessors()) {
	if (bp instanceof InstantiationAwareBeanPostProcessor) {
		InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
		PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
		if (pvsToUse == null) {
			if (filteredPds == null) {
				filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			}
			pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
			if (pvsToUse == null) {
				return;
			}
		}
		pvs = pvsToUse;
	}
}
```

主要是注入`@Resource`和`Autowire`，先放着

`populateBean`流程完成

```
exposedObject = initializeBean(beanName, exposedObject, mbd);
```

初始化流程

```
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
	if (System.getSecurityManager() != null) {
		AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
			invokeAwareMethods(beanName, bean);
			return null;
		}, getAccessControlContext());
	}
	else {
		invokeAwareMethods(beanName, bean);
	}

	Object wrappedBean = bean;
	if (mbd == null || !mbd.isSynthetic()) {
		//这里处理@PostConstruct
		wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
	}

	try {
		//这里处理了实现接口InitializingBean的afterPropertiesSet方法(void无参)
		//然后再调用init方法(Bean中init-method属性)
		invokeInitMethods(beanName, wrappedBean, mbd);
	}
	catch (Throwable ex) {
		throw new BeanCreationException(
				(mbd != null ? mbd.getResourceDescription() : null),
				beanName, "Invocation of init method failed", ex);
	}
	if (mbd == null || !mbd.isSynthetic()) {
		wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
	}

	return wrappedBean;
}
```

先看`invokeAwareMethods(beanName, bean)`

```
private void invokeAwareMethods(final String beanName, final Object bean) {
	if (bean instanceof Aware) {
		if (bean instanceof BeanNameAware) {
			((BeanNameAware) bean).setBeanName(beanName);
		}
		if (bean instanceof BeanClassLoaderAware) {
			ClassLoader bcl = getBeanClassLoader();
			if (bcl != null) {
				((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
			}
		}
		if (bean instanceof BeanFactoryAware) {
			((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
		}
	}
}
```

看`Bean`是否实现这几个接口，然后执行对应操作

然后到`wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);`

```
@Override
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
		throws BeansException {

	Object result = existingBean;
	for (BeanPostProcessor processor : getBeanPostProcessors()) {
		Object current = processor.postProcessBeforeInitialization(result, beanName);
		if (current == null) {
			return result;
		}
		result = current;
	}
	return result;
}
```

又遍历一次`BeanPostProcessor`，调用的`postProcessBeforeInitialization`方法

这里涉及到生命周期中的`@PostConstruct`方法，由`CommonAnnotationBeanPostProcessor`提供实现，`@PostConstruct`注解的方法就是在这被调用的

然后到`invokeInitMethods(beanName, wrappedBean, mbd);`

在这里面，如果`Bean`实现了`InitializingBean`接口，则调用`afterPropertiesSet`方法，在这之后接着调用`init`方法，这个`init`方法指的`Spring`配置中的`init-method`属性

接着`wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName)`，又遍历`BeanPostProcessor`

```
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
		throws BeansException {

	Object result = existingBean;
	for (BeanPostProcessor processor : getBeanPostProcessors()) {
		Object current = processor.postProcessAfterInitialization(result, beanName);
		if (current == null) {
			return result;
		}
		result = current;
	}
	return result;
}
```

### 总结

总共经过7次后置处理器

1. 第一次在创建实例之前，如果这里有后置处理器直接返回对应的类的实例，则跳过后面创建过程

```
protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
	for (BeanPostProcessor bp : getBeanPostProcessors()) {
		if (bp instanceof InstantiationAwareBeanPostProcessor) {
			InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
			Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
			if (result != null) {
				return result;
			}
		}
	}
	return null;
}
```

2. 之后是第二次调用后置处理器，同样也是在创建实例之前，找寻合适的构造器方法，主要是找到构造器注入方法，否则接下来使用默认无参构造方法来创建实例，如果这里找到构造器注入，则调用这个构造器实例化这个类，同时注入需要的属性

```
protected Constructor<?>[] determineConstructorsFromBeanPostProcessors(@Nullable Class<?> beanClass, String beanName)
		throws BeansException {

	if (beanClass != null && hasInstantiationAwareBeanPostProcessors()) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
				SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
				Constructor<?>[] ctors = ibp.determineCandidateConstructors(beanClass, beanName);
				if (ctors != null) {
					return ctors;
				}
			}
		}
	}
	return null;
}
```

3. 第三次调用后置处理器，主要解析`Bean`中有关生命周期的注解`@PostConstruct`和`@PreDestroy`，然后解析需要`@AutoWired`和`@Resource`注入的配置

```
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, Class<?> beanType, String beanName) {
	for (BeanPostProcessor bp : getBeanPostProcessors()) {
		if (bp instanceof MergedBeanDefinitionPostProcessor) {
			MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
			bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
		}
	}
}
```

4. 第四次调用后置处理器，还是在初始化之前，当有一个返回`false`，跳过实例内部属性注入过程

```
if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
	for (BeanPostProcessor bp : getBeanPostProcessors()) {
		if (bp instanceof InstantiationAwareBeanPostProcessor) {
			InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
			if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
				continueWithPropertyPopulation = false;
				break;
			}
		}
	}
}
```

5. 第五次调用后置处理器，进行内部属性注入，`@Autowired` `@Resource`

```
for (BeanPostProcessor bp : getBeanPostProcessors()) {
	if (bp instanceof InstantiationAwareBeanPostProcessor) {
		InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
		PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
		if (pvsToUse == null) {
			if (filteredPds == null) {
				filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			}
			pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
			if (pvsToUse == null) {
				return;
			}
		}
		pvs = pvsToUse;
	}
}
```

6. 第六次调用后置处理器，开始初始化，处理`@PostConstruct`注解方法

```
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
		throws BeansException {

	Object result = existingBean;
	for (BeanPostProcessor processor : getBeanPostProcessors()) {
		Object current = processor.postProcessBeforeInitialization(result, beanName);
		if (current == null) {
			return result;
		}
		result = current;
	}
	return result;
}
```

7. 第七次调用后置处理器

```
@Override
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
		throws BeansException {

	Object result = existingBean;
	for (BeanPostProcessor processor : getBeanPostProcessors()) {
		Object current = processor.postProcessAfterInitialization(result, beanName);
		if (current == null) {
			return result;
		}
		result = current;
	}
	return result;
}
```

### 附加

这几个后置处理器在`AnnotationConfigUtils`中的`registerBeanDefinition`方法中注册到`DefaultListableBeanFactory`的`beanDefinitionNames`和`beanDefinitionMap`中

使用注解式配置ssm时，这个过程发生在容器refresh的时候	，在创建工厂对象后调用`loadBeanDefinitions`方法开始注册

