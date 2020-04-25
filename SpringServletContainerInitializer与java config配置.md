title: SpringServletContainerInitializer与java config配置
categories: 框架
tags: 
	- Spring
	- SpringMVC
---

### ServletContainerInitializer

这是`serlvet3.0`的特性，它主要是用来实现`java config`的方式来配置框架，比如ssm，它通过编程的方式代理`web.xml`配置文件，通过编程的方式注册`serlvet`、`filter`、`Listener`等组件

`serlvet`容器在启动时会扫描应用中所有`jar`包下的`META-INF/services/javax.servlet.ServletContainerInitializer`文件，文件中的内容为实现了`ServletContainerInitializer`的实现类的全限定名

容器会自动创建它的实例，并且调用接口方法`onStartup`

`@handlesTypes`注解在实现了`servletContainerInitializer`的类上，决定可以这个类可以处理的类型，作为第一个参数`set`类型，被传到`onStartup`方法中


### SpringServletContainerInitializer

在`spring-web-4.17.RELEASE.jar`的`META-INF`目录的`services`下找到文件`javax.servlet.ServlerContainerInitializer`，其中内容为

```
org.springframework.web.SpringServletContainerInitializer
```

查看这个类

```
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {

	@Override
	public void onStartup(Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {

		List<WebApplicationInitializer> initializers = new LinkedList<WebApplicationInitializer>();

		if (webAppInitializerClasses != null) {
			for (Class<?> waiClass : webAppInitializerClasses) {
				// Be defensive: Some servlet containers provide us with invalid classes,
				// no matter what @HandlesTypes says...
				if (!waiClass.isInterface() && !Modifier.isAbstract(waiClass.getModifiers()) &&
						WebApplicationInitializer.class.isAssignableFrom(waiClass)) {
					try {
						initializers.add((WebApplicationInitializer) waiClass.newInstance());
					}
					catch (Throwable ex) {
						throw new ServletException("Failed to instantiate WebApplicationInitializer class", ex);
					}
				}
			}
		}

		if (initializers.isEmpty()) {
			servletContext.log("No Spring WebApplicationInitializer types detected on classpath");
			return;
		}

		AnnotationAwareOrderComparator.sort(initializers);
		servletContext.log("Spring WebApplicationInitializers detected on classpath: " + initializers);

		for (WebApplicationInitializer initializer : initializers) {
			initializer.onStartup(servletContext);
		}
	}

}
```

我们看到类上有`handlesTypes`注解，值为`WebApplicationInitializer.class`

断点调试查看`onStartup`方法第一个参数

![webApplicationInitializerClasses](https://raw.githubusercontent.com/sinaill/pic/master/wenAppInitializerClasses.jpg)

确实把我们定义的继承了`WebApplicationInitializer`的子类`AbstractAnnotationConfigDispatcherServletInitializer`的类传入了

```
public class WebInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {
    protected Class<?>[] getRootConfigClasses() {
        return new Class<?>[]{RootConfig.class};
    }

    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[]{MvcConfig.class};
    }

    protected String[] getServletMappings() {
        return new String[]{"/"};
    }

}
```

### 开始配置

最后容器会调用`WebApplicationInitializer`接口的`onStartup`方法，也就是调用我们上面的那个类来开始配置

由于我们是继承的`AbstractAnnotationConfigDispatcherServletInitializer`，所以该方法调用的是`AbstractDispatcherServletInitializer`的`onStartup`方法

```
@Override
public void onStartup(ServletContext servletContext) throws ServletException {
	super.onStartup(servletContext);
	registerDispatcherServlet(servletContext);
}
```

#### super.onStartup(servletContext)

调用了父类`AbstractContextLoaderInitializer`的`onStartup`方法

```
@Override
public void onStartup(ServletContext servletContext) throws ServletException {
	registerContextLoaderListener(servletContext);
}



protected void registerContextLoaderListener(ServletContext servletContext) {
	WebApplicationContext rootAppContext = createRootApplicationContext();
	if (rootAppContext != null) {
		servletContext.addListener(new ContextLoaderListener(rootAppContext));
	}
	else {
		logger.debug("No ContextLoaderListener registered, as " +
				"createRootApplicationContext() did not return an application context");
	}
}
```

这里调用了`createRootApplicationContext`方法创建`Spring`容器，然后创建了`ContextLoaderListener`监听器，注册到`servletContext`中，所以使用注解式配置ssm的时候，在`contextLoaderListener`被触发的时候，容器已经被创建好了，而使用配置文件的方式就是在触发`contextLoaderListener`的时候创建`Spring`容器和`SpringMVC`容器

```
@Override
protected WebApplicationContext createRootApplicationContext() {
	Class<?>[] configClasses = getRootConfigClasses();
	if (!ObjectUtils.isEmpty(configClasses)) {
		//创建Spring容器
		AnnotationConfigWebApplicationContext rootAppContext = new AnnotationConfigWebApplicationContext();
		rootAppContext.register(configClasses);
		return rootAppContext;
	}
	else {
		return null;
	}
}
```

可以看到创建了`Spring`容器并且调用了我们实现的`getRootConfigClasses`方法，把获得的`Class`注册到`Spring容器中`，然后只要调用`Spring`容器的`refresh`方法就能在容器中实例化它们

#### registerDispatcherServlet(servletContext)

```
protected void registerDispatcherServlet(ServletContext servletContext) {
	String servletName = getServletName();
	Assert.hasLength(servletName, "getServletName() must not return empty or null");

	WebApplicationContext servletAppContext = createServletApplicationContext();
	Assert.notNull(servletAppContext,
			"createServletApplicationContext() did not return an application " +
			"context for servlet [" + servletName + "]");

	DispatcherServlet dispatcherServlet = new DispatcherServlet(servletAppContext);
	ServletRegistration.Dynamic registration = servletContext.addServlet(servletName, dispatcherServlet);
	Assert.notNull(registration,
			"Failed to register servlet with name '" + servletName + "'." +
			"Check if there is another servlet registered under the same name.");

	registration.setLoadOnStartup(1);
	registration.addMapping(getServletMappings());
	registration.setAsyncSupported(isAsyncSupported());

	Filter[] filters = getServletFilters();
	if (!ObjectUtils.isEmpty(filters)) {
		for (Filter filter : filters) {
			registerServletFilter(servletContext, filter);
		}
	}

	customizeRegistration(registration);
}
```

`createServletApplicationContext`创建`SpringMVC`容器

```
@Override
protected WebApplicationContext createServletApplicationContext() {
	AnnotationConfigWebApplicationContext servletAppContext = new AnnotationConfigWebApplicationContext();
	Class<?>[] configClasses = getServletConfigClasses();
	if (!ObjectUtils.isEmpty(configClasses)) {
		servletAppContext.register(configClasses);
	}
	return servletAppContext;
}
```

同样创建了一个`AnnotationConfigWebApplicationContext`对象，然后调用我们实现的`getServletConfigClasses`将获取的`Class`注册到`SpringMVC`容器中

接着开始注册`DispatcherServlet`，创建`DispatcherServlet`对象绑定到`servetContext`中，并将`SpringMVC`容器传入`DispatcherServlet`

```
registration.setLoadOnStartup(1);
registration.addMapping(getServletMappings());
registration.setAsyncSupported(isAsyncSupported());
```

配置启动优先级，配置`serlvet`映射方式

还有后面调用`getServletFilters`方法获取过滤器

### 疑问

在使用配置文件方式的时候，创建`SpringMVC`容器

```
wac = createWebApplicationContext(rootContext);
```

将`Spring`作为父容器传入，所以`SpringMVC`容器中能取出在`Spring`容器中的对象

而这里使用注解方式了，没发现两个容器的关联

原来在两个容器关联的代码在`ContextLoaderListener`中的`initWebApplicationContext`方法中

```
if (this.context instanceof ConfigurableWebApplicationContext) {
	ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
	if (!cwac.isActive()) {
		// The context has not yet been refreshed -> provide services such as
		// setting the parent context, setting the application context id, etc
		if (cwac.getParent() == null) {
			// The context instance was injected without an explicit parent ->
			// determine parent for root web application context, if any.
			ApplicationContext parent = loadParentContext(servletContext);
			cwac.setParent(parent);
		}
		configureAndRefreshWebApplicationContext(cwac, servletContext);
	}
}
```

调用了`setParent`方法将`Spring`容器作为`SpringMVC`的父容器