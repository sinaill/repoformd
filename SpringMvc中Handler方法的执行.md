title: SpringMvc中Handler方法的执行
categories: 框架
tags: 
	- SpringMvc

---

### 需要注意的类

#### invocableHandlerMethod

```

```

### 流程

`Handler`方法的执行由`handlerAdapter`接口的实现类的`handle`方法为入口

```
// Actually invoke the handler.
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
``` 

本Demo中配置方式决定了`ha`变量为`RequestMappingHandlerAdapter`实例对象，调用该对象的`handle`方法
，`handle`方法为它父类`AbstractHandlerMethodAdapter`中的方法
```
@Override
public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
		throws Exception {

	return handleInternal(request, response, (HandlerMethod) handler);
}
```

接着又调用该对象的`handleInternal`方法

```
@Override
protected ModelAndView handleInternal(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

	//判断是否有@SessionAttributes注解
	if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
		// Always prevent caching in case of session attribute management.
		checkAndPrepare(request, response, this.cacheSecondsForSessionAttributeHandlers, true);
	}
	else {
		// Uses configured default cacheSeconds setting.
		checkAndPrepare(request, response, true);
	}

	// Execute invokeHandlerMethod in synchronized block if required.
	if (this.synchronizeOnSession) {
		HttpSession session = request.getSession(false);
		if (session != null) {
			Object mutex = WebUtils.getSessionMutex(session);
			synchronized (mutex) {
				return invokeHandleMethod(request, response, handlerMethod);
			}
		}
	}

	return invokeHandleMethod(request, response, handlerMethod);
}
```

跟入`invokeHandleMethod`

```
/**
 * Invoke the {@link RequestMapping} handler method preparing a {@link ModelAndView}
 * if view resolution is required.
 */
private ModelAndView invokeHandleMethod(HttpServletRequest request,
		HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

	ServletWebRequest webRequest = new ServletWebRequest(request, response);

	WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
	ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
	ServletInvocableHandlerMethod requestMappingMethod = createRequestMappingMethod(handlerMethod, binderFactory);

	ModelAndViewContainer mavContainer = new ModelAndViewContainer();
	mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
	modelFactory.initModel(webRequest, mavContainer, requestMappingMethod);
	mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

	AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
	asyncWebRequest.setTimeout(this.asyncRequestTimeout);

	final WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
	asyncManager.setTaskExecutor(this.taskExecutor);
	asyncManager.setAsyncWebRequest(asyncWebRequest);
	asyncManager.registerCallableInterceptors(this.callableInterceptors);
	asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

	if (asyncManager.hasConcurrentResult()) {
		Object result = asyncManager.getConcurrentResult();
		mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
		asyncManager.clearConcurrentResult();

		if (logger.isDebugEnabled()) {
			logger.debug("Found concurrent result value [" + result + "]");
		}
		requestMappingMethod = requestMappingMethod.wrapConcurrentResult(result);
	}

	requestMappingMethod.invokeAndHandle(webRequest, mavContainer);

	if (asyncManager.isConcurrentHandlingStarted()) {
		return null;
	}

	return getModelAndView(mavContainer, modelFactory, webRequest);
}
```

先看`WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);`，进入`getDataBinderFactory`

```
private WebDataBinderFactory getDataBinderFactory(HandlerMethod handlerMethod) throws Exception {
	Class<?> handlerType = handlerMethod.getBeanType();
	Set<Method> methods = this.initBinderCache.get(handlerType);
	if (methods == null) {
		methods = HandlerMethodSelector.selectMethods(handlerType, INIT_BINDER_METHODS);
		this.initBinderCache.put(handlerType, methods);
	}
	List<InvocableHandlerMethod> initBinderMethods = new ArrayList<InvocableHandlerMethod>();
	// Global methods first
	for (Entry<ControllerAdviceBean, Set<Method>> entry : this.initBinderAdviceCache .entrySet()) {
		if (entry.getKey().isApplicableToBeanType(handlerType)) {
			Object bean = entry.getKey().resolveBean();
			for (Method method : entry.getValue()) {
				initBinderMethods.add(createInitBinderMethod(bean, method));
			}
		}
	}
	for (Method method : methods) {
		Object bean = handlerMethod.getBean();
		initBinderMethods.add(createInitBinderMethod(bean, method));
	}
	return createDataBinderFactory(initBinderMethods);
}
```

`initBinderCache`字面看是`@initBinder`方法的缓存，看`selectMethod`方法

```
public static Set<Method> selectMethods(final Class<?> handlerType, final MethodFilter handlerMethodFilter) {
	final Set<Method> handlerMethods = new LinkedHashSet<Method>();
	Set<Class<?>> handlerTypes = new LinkedHashSet<Class<?>>();
	Class<?> specificHandlerType = null;
	if (!Proxy.isProxyClass(handlerType)) {
		handlerTypes.add(handlerType);
		specificHandlerType = handlerType;
	}
	handlerTypes.addAll(Arrays.asList(handlerType.getInterfaces()));
	for (Class<?> currentHandlerType : handlerTypes) {
		final Class<?> targetClass = (specificHandlerType != null ? specificHandlerType : currentHandlerType);
		ReflectionUtils.doWithMethods(currentHandlerType, new ReflectionUtils.MethodCallback() {
			@Override
			public void doWith(Method method) {
				Method specificMethod = ClassUtils.getMostSpecificMethod(method, targetClass);
				Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
				if (handlerMethodFilter.matches(specificMethod) &&
						(bridgedMethod == specificMethod || !handlerMethodFilter.matches(bridgedMethod))) {
					handlerMethods.add(specificMethod);
				}
			}
		}, ReflectionUtils.USER_DECLARED_METHODS);
	}
	return handlerMethods;
}
```

判断传入的`handlerType`是不是代理类，不是的话添加到集合`handlerTypes`中，再让`specificHandlerYtpe`指向它，如果事代理类的话，将代理类实现的接口添加到集合`handlerTypes`中。
下一步对`handlerTypes`进行处理，当`handlerType`不是代理类时，这一步处理的就是`handlerType`本身，
调用了类`ReflectionUtils`的`doWithMethods`方法对`handlerType`进行处理

```
public static void doWithMethods(Class<?> clazz, MethodCallback mc, MethodFilter mf) {
	// Keep backing up the inheritance hierarchy.
	Method[] methods = getDeclaredMethods(clazz);
	for (Method method : methods) {
		if (mf != null && !mf.matches(method)) {
			continue;
		}
		try {
			mc.doWith(method);
		}
		catch (IllegalAccessException ex) {
			throw new IllegalStateException("Not allowed to access method '" + method.getName() + "': " + ex);
		}
	}
	if (clazz.getSuperclass() != null) {
		doWithMethods(clazz.getSuperclass(), mc, mf);
	}
	else if (clazz.isInterface()) {
		for (Class<?> superIfc : clazz.getInterfaces()) {
			doWithMethods(superIfc, mc, mf);
		}
	}
}
```

`mf`为`MethodFilter`接口的实现类，作用为判断如果传入的`method`为非桥接方法和非`Object`中的方法就返回`true`，否则返回`false`。

```
public static MethodFilter USER_DECLARED_METHODS = new MethodFilter() {

	@Override
	public boolean matches(Method method) {
		return (!method.isBridge() && method.getDeclaringClass() != Object.class);
	}
}


```


接着获取上一步的`handlerType`的方法，`getDeclaredMethod`获取的是类中定义的所有类型的方法，不包括从父类继承的。然后通过`mf`过滤掉桥接方法，然后使用`mc`的`dowith`方法遍历所有它们，`mc`为`selectMehtod`中的用匿名内部类的方法构造。

```
new ReflectionUtils.MethodCallback() {
			@Override
			public void doWith(Method method) {
				Method specificMethod = ClassUtils.getMostSpecificMethod(method, targetClass);
				Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
				if (handlerMethodFilter.matches(specificMethod) &&
						(bridgedMethod == specificMethod || !handlerMethodFilter.matches(bridgedMethod))) {
					handlerMethods.add(specificMethod);
				}
			}
		}
```

其中`ClassUtils`的`getMostSpecificmethod`方法，通过`method`查找方法名和变量名在`targetClass`中相同的`method`，`BridgeMethodResolver.findBridgedMethod`api的英文解释，Find the original method for the supplied bridge Method.从源码看是通过传入的桥接方法获取到Class，然后获取所有方法，匹配名字和参数个数相同的方法。接下来是`handlerMethodFilter`过滤器的匹配，这个过滤器定义在`RequestMappingHandlerAdapter`中

```
public static final MethodFilter INIT_BINDER_METHODS = new MethodFilter() {

	@Override
	public boolean matches(Method method) {
		return AnnotationUtils.findAnnotation(method, InitBinder.class) != null;
	}
};
``` 

作用是查看方法是否有`@initBinder`注解，如果有返回`true`，没有就返回`false`，当`specificMethod`不是`@initBinder`注解的方法，切不是桥接方法或桥接方法为`@initBinder`方法时，将`specificMethod`添加到`handlerMethods`中，然后返回

```

private WebDataBinderFactory getDataBinderFactory(HandlerMethod handlerMethod) throws Exception {
	Class<?> handlerType = handlerMethod.getBeanType();
	Set<Method> methods = this.initBinderCache.get(handlerType);
	if (methods == null) {
		methods = HandlerMethodSelector.selectMethods(handlerType, INIT_BINDER_METHODS);
		this.initBinderCache.put(handlerType, methods);
	}
	List<InvocableHandlerMethod> initBinderMethods = new ArrayList<InvocableHandlerMethod>();
	// Global methods first
	for (Entry<ControllerAdviceBean, Set<Method>> entry : this.initBinderAdviceCache .entrySet()) {
		if (entry.getKey().isApplicableToBeanType(handlerType)) {
			Object bean = entry.getKey().resolveBean();
			for (Method method : entry.getValue()) {
				initBinderMethods.add(createInitBinderMethod(bean, method));
			}
		}
	}
	for (Method method : methods) {
		Object bean = handlerMethod.getBean();
		initBinderMethods.add(createInitBinderMethod(bean, method));
	}
	return createDataBinderFactory(initBinderMethods);
}

```

这样返回的`methods`集合中就有了和当前`handlerType`(当前`request`对应的`controller`)对应的`@initBinder`方法，并且存放在了缓存`initBinderCache`中，`initBinderAdviceCache`存放的是在`@controllerAdvice`中的全局`@initBinder`方法。

然后调用`createInitBinderMethod`方法创建一个`invocableHandlerMethod`

```
private InvocableHandlerMethod createInitBinderMethod(Object bean, Method method) {
	InvocableHandlerMethod binderMethod = new InvocableHandlerMethod(bean, method);
	binderMethod.setHandlerMethodArgumentResolvers(this.initBinderArgumentResolvers);
	binderMethod.setDataBinderFactory(new DefaultDataBinderFactory(this.webBindingInitializer));
	binderMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
	return binderMethod;
}
```

然后将全局`@initBinder`方法和控制器中的`@initBinder`方法放入集合`initBinderMethods`中，并且用它来创建数据绑定工厂

```
protected InitBinderDataBinderFactory createDataBinderFactory(List<InvocableHandlerMethod> binderMethods)
		throws Exception {

	return new ServletRequestDataBinderFactory(binderMethods, getWebBindingInitializer());
}
```

```
WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
```

以上主要查找了控制器中和全局`@initBinder`方法，然后和`webBindingInitializer`一同创建`servletRequestDataBinderFactory`

接下来看

```
ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory)
```

```
private ModelFactory getModelFactory(HandlerMethod handlerMethod, WebDataBinderFactory binderFactory) {
	SessionAttributesHandler sessionAttrHandler = getSessionAttributesHandler(handlerMethod);
	Class<?> handlerType = handlerMethod.getBeanType();
	Set<Method> methods = this.modelAttributeCache.get(handlerType);
	if (methods == null) {
		methods = HandlerMethodSelector.selectMethods(handlerType, MODEL_ATTRIBUTE_METHODS);
		this.modelAttributeCache.put(handlerType, methods);
	}
	List<InvocableHandlerMethod> attrMethods = new ArrayList<InvocableHandlerMethod>();
	// Global methods first
	for (Entry<ControllerAdviceBean, Set<Method>> entry : this.modelAttributeAdviceCache.entrySet()) {
		if (entry.getKey().isApplicableToBeanType(handlerType)) {
			Object bean = entry.getKey().resolveBean();
			for (Method method : entry.getValue()) {
				attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
			}
		}
	}
	for (Method method : methods) {
		Object bean = handlerMethod.getBean();
		attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
	}
	return new ModelFactory(attrMethods, binderFactory, sessionAttrHandler);
}
```

主要是处理`@modelAttribute`注解方法

先看`getSessionAttributesHandler`方法

```
private SessionAttributesHandler getSessionAttributesHandler(HandlerMethod handlerMethod) {
	Class<?> handlerType = handlerMethod.getBeanType();
	SessionAttributesHandler sessionAttrHandler = this.sessionAttributesHandlerCache.get(handlerType);
	if (sessionAttrHandler == null) {
		synchronized (this.sessionAttributesHandlerCache) {
			sessionAttrHandler = this.sessionAttributesHandlerCache.get(handlerType);
			if (sessionAttrHandler == null) {
				sessionAttrHandler = new SessionAttributesHandler(handlerType, sessionAttributeStore);
				this.sessionAttributesHandlerCache.put(handlerType, sessionAttrHandler);
			}
		}
	}
	return sessionAttrHandler;
}
```

用了单例模式创建`SessionAttrHandler`，`sessionAttrHandler`是`DefaultSessionAttributeStore`类的实例，由于在`handleInternal`调用过这个方法，所以这里有了缓存

接着从当前`modelAttributeCache`根据当前`handlerType`查找集合`methods`

```
methods = HandlerMethodSelector.selectMethods(handlerType, MODEL_ATTRIBUTE_METHODS);
```

熟悉的方法，不过过滤器换成了`MODEL_ATTRIBUTE_METHODS`

```
public static final MethodFilter MODEL_ATTRIBUTE_METHODS = new MethodFilter() {

	@Override
	public boolean matches(Method method) {
		return ((AnnotationUtils.findAnnotation(method, RequestMapping.class) == null) &&
				(AnnotationUtils.findAnnotation(method, ModelAttribute.class) != null));
	}
};
```

查找当前`handlerType`中没有被`@requestMapping`注解并且有`@modelAttribute`注解的方法，然后再从`@controllerAdvice`注解的类中也找相同的全局`@modelAttribute`方法，然后创建`modelFactory`

```
ServletInvocableHandlerMethod requestMappingMethod = createRequestMappingMethod(handlerMethod, binderFactory);
```

创建`ServletInvocableHandlerMethod`对象

```
private ServletInvocableHandlerMethod createRequestMappingMethod(
		HandlerMethod handlerMethod, WebDataBinderFactory binderFactory) {

	ServletInvocableHandlerMethod requestMethod;
	requestMethod = new ServletInvocableHandlerMethod(handlerMethod);
	requestMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
	requestMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
	requestMethod.setDataBinderFactory(binderFactory);
	requestMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
	return requestMethod;
}
```

`argumentResolver`是`HandlerMethodArgumentResolverComposite`类的实例，用来做参数解析，`returnValueHandlers`是`HandlerMethodReturnValueHandlerComposite`类的实例，用来对返回结果做处理，`parameterNameDiscoverer`用来记录参数名解析器

```
ModelAndViewContainer mavContainer = new ModelAndViewContainer();
```

ModelAndViewContainer主要是用来返回`Model`对象的，当然里面还有变量`view`存放视图

此时`Model`为默认`Model`，是新建的`BindingAwareModelMap`类的实例

```
modelFactory.initModel(webRequest, mavContainer, requestMappingMethod);
```

文档是这样写的

```
/**
 * Populate the model in the following order:
 * <ol>
 * 	<li>Retrieve "known" session attributes listed as {@code @SessionAttributes}.
 * 	<li>Invoke {@code @ModelAttribute} methods
 * 	<li>Find {@code @ModelAttribute} method arguments also listed as
 * 	{@code @SessionAttributes} and ensure they're present in the model raising
 * 	an exception if necessary.
 * </ol>
 * @param request the current request
 * @param mavContainer a container with the model to be initialized
 * @param handlerMethod the method for which the model is initialized
 * @throws Exception may arise from {@code @ModelAttribute} methods
 */
public void initModel(NativeWebRequest request, ModelAndViewContainer mavContainer, HandlerMethod handlerMethod)
		throws Exception {

	Map<String, ?> sessionAttributes = this.sessionAttributesHandler.retrieveAttributes(request);
	mavContainer.mergeAttributes(sessionAttributes);

	invokeModelAttributeMethods(request, mavContainer);

	for (String name : findSessionAttributeArguments(handlerMethod)) {
		if (!mavContainer.containsAttribute(name)) {
			Object value = this.sessionAttributesHandler.retrieveAttribute(request, name);
			if (value == null) {
				throw new HttpSessionRequiredException("Expected session attribute '" + name + "'");
			}
			mavContainer.addAttribute(name, value);
		}
	}
}
```

`sessionAttributes`变量当`@SessionAttribute`注解的变量名在控制器某个方法被实例化放入`model`中，也就是被放入`Session`中后，在此处会获得这个注解的变量名的属性，源码也就是变量`sessionAttributesHandler`中有个变量`knownAttributeNames`集合，存放了`@SessionAttribute`注解的变量名，这里通过`request`获取`Session`，然后取出对应属性

然后`mergeAttributes`将获取到的属性复制一份放到`mavContainer`的`model`中

查看`invokeModelAttributeMethods`方法

```
private void invokeModelAttributeMethods(NativeWebRequest request, ModelAndViewContainer mavContainer)
		throws Exception {

	while (!this.modelMethods.isEmpty()) {
		InvocableHandlerMethod attrMethod = getNextModelMethod(mavContainer).getHandlerMethod();
		String modelName = attrMethod.getMethodAnnotation(ModelAttribute.class).value();
		if (mavContainer.containsAttribute(modelName)) {
			continue;
		}

		Object returnValue = attrMethod.invokeForRequest(request, mavContainer);

		if (!attrMethod.isVoid()){
			String returnValueName = getNameForReturnValue(returnValue, attrMethod.getReturnType());
			if (!mavContainer.containsAttribute(returnValueName)) {
				mavContainer.addAttribute(returnValueName, returnValue);
			}
		}
	}
}
```

此处的`modelMethods`在前几步的`getModelFactory`中，是获取的控制器和全局`@initBinder`方法的集合。

我们看`getNextModelMethod(mavContainer).getHandlerMethod()`

```
private ModelMethod getNextModelMethod(ModelAndViewContainer mavContainer) {
	for (ModelMethod modelMethod : this.modelMethods) {
		if (modelMethod.checkDependencies(mavContainer)) {
			if (logger.isTraceEnabled()) {
				logger.trace("Selected @ModelAttribute method " + modelMethod);
			}
			this.modelMethods.remove(modelMethod);
			return modelMethod;
		}
	}
	ModelMethod modelMethod = this.modelMethods.get(0);
	if (logger.isTraceEnabled()) {
		logger.trace("Selected @ModelAttribute method (not present: " +
				modelMethod.getUnresolvedDependencies(mavContainer)+ ") " + modelMethod);
	}
	this.modelMethods.remove(modelMethod);
	return modelMethod;
}
```

对集合`modelMethods`进行遍历，查找合适条件的`modelMethod`，这里用的`modelMethod`的`checkDependencies`方法

```
public boolean checkDependencies(ModelAndViewContainer mavContainer) {
	for (String name : this.dependencies) {
		if (!mavContainer.containsAttribute(name)) {
			return false;
		}
	}
	return true;
}
```

关于`ModelMethod`的`denpendencies`变量，需要往前追溯一下变量`modelMethod`是怎么创建的

回到`getModelFactory`方法

```
attrMethods.add(createModelAttributeMethod(binderFactory, bean, method));
```

这一步是将获取到的带`@ModelAttribute`注解的方法放入到类型为`List<InvocableHandlerMethod>`的集合中`attrMethods`中，在创建`ModelFactory`类的实例的时候利用构造函数将其类型转化为`List<ModelMethod>`的集合`modelMethods`

```
public ModelFactory(List<InvocableHandlerMethod> invocableMethods, WebDataBinderFactory dataBinderFactory,
		SessionAttributesHandler sessionAttributesHandler) {

	if (invocableMethods != null) {
		for (InvocableHandlerMethod method : invocableMethods) {
			this.modelMethods.add(new ModelMethod(method));
		}
	}
	this.dataBinderFactory = dataBinderFactory;
	this.sessionAttributesHandler = sessionAttributesHandler;
}
```

在构造函数里，集合`attrMethods`中的`invocableHandlerMethod`类的实例被包装为`ModelMethod`类的实例
`new ModelMethod(method)`

我们看`ModelMethod`的构造函数是怎么处理`dependencies`变量的

```
private ModelMethod(InvocableHandlerMethod handlerMethod) {
	this.handlerMethod = handlerMethod;
	for (MethodParameter parameter : handlerMethod.getMethodParameters()) {
		if (parameter.hasParameterAnnotation(ModelAttribute.class)) {
			this.dependencies.add(getNameForParameter(parameter));
		}
	}
}
```

`getMethodParameters`获取方法的所有参数，返回一个`MethodParameter`集合，接下来遍历这些参数，如果有带`@ModelAttribute`注解的，获取参数的名字放入类型为`Set<String>`集合的`dependencies`中

接下来就知道`checkDependencies`方法是做什么的了

```
public boolean checkDependencies(ModelAndViewContainer mavContainer) {
	for (String name : this.dependencies) {
		if (!mavContainer.containsAttribute(name)) {
			return false;
		}
	}
	return true;
}
```

`!mavContainer.containsAttribute(name)`，对方法的所有参数名字遍历

```
public boolean containsAttribute(String name) {
	return getModel().containsAttribute(name);
}
```

以上将`@ModelAttribute`方法的集合做了包装，`invocableHandlerMethod -> ModelMethod`，多了个`dependencies`属性，类型为`Set<String>`，用来存放当前方法中带了`@ModelAttribute`注解的形参的参数名

接着遍历包装后的集合，`mavContainer`中的`model`必须拥有`dependencies`中所有的属性名，此时`checkDependencies`方法返回`true`，当`ModelMethod`的`dependencies`为空时，也返回`true`，其他返回`false`

回到`getNextModelMethod(mavContainer).getHandlerMethod()`

```
private ModelMethod getNextModelMethod(ModelAndViewContainer mavContainer) {
	for (ModelMethod modelMethod : this.modelMethods) {
		if (modelMethod.checkDependencies(mavContainer)) {
			if (logger.isTraceEnabled()) {
				logger.trace("Selected @ModelAttribute method " + modelMethod);
			}
			this.modelMethods.remove(modelMethod);
			return modelMethod;
		}
	}
	ModelMethod modelMethod = this.modelMethods.get(0);
	if (logger.isTraceEnabled()) {
		logger.trace("Selected @ModelAttribute method (not present: " +
				modelMethod.getUnresolvedDependencies(mavContainer)+ ") " + modelMethod);
	}
	this.modelMethods.remove(modelMethod);
	return modelMethod;
}

