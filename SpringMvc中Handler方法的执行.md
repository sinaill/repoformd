title: SpringMVC中Handler方法的执行
categories: 框架
tags: 
	- SpringMVC

---

### 需要注意的类

#### invocableHandlerMethod

#### HandlerMethodParameter

`HandlerMethodParameter`为`HandlerMethod`的内部类和`MethodParameter`的子类

先看`MethodParameter`

```
private final Method method
//指向方法
private final int parameterIndex;
//本方法参数在方法中的形参位置index，从0开始

```

 

```

```


#### argumentResolver

创建`InvocableHandlerMethod`的时候，将`Spring`给`RequestMappingHandlerAdapter`注入的`argumentResovlers`参数传递给`InvocableHandlerMethod`，用来对通过`GenericTypeResolver`确定方法参数类型之后的`MethodParameter`进行处理

例如`RequestParamMethodArgumentResolver`，它的`supportParameter`方法，即验证是否适用于当前参数

```
@Override
public boolean supportsParameter(MethodParameter parameter) {
	Class<?> paramType = parameter.getParameterType();
	if (parameter.hasParameterAnnotation(RequestParam.class)) {
		//与instanceof作用相同，只不过这个是判断类之间的管理
		//判断传入的类与这个类是否相同或者是不是传入的这个类的超类或者超接口
		if (Map.class.isAssignableFrom(paramType)) {
			String paramName = parameter.getParameterAnnotation(RequestParam.class).value();
			return StringUtils.hasText(paramName);
		}
		else {
			return true;
		}
	}
	else {
		if (parameter.hasParameterAnnotation(RequestPart.class)) {
			return false;
		}
		else if (MultipartFile.class.equals(paramType) || "javax.servlet.http.Part".equals(paramType.getName())) {
			return true;
		}
		else if (this.useDefaultResolution) {
			return BeanUtils.isSimpleProperty(paramType);
		}
		else {
			return false;
		}
	}
}
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

`getSessionAttributesHandler`方法

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

单例模式创建`SessionAttributesHandler`，看构造方法

```
public SessionAttributesHandler(Class<?> handlerType, SessionAttributeStore sessionAttributeStore) {
	Assert.notNull(sessionAttributeStore, "SessionAttributeStore may not be null.");
	this.sessionAttributeStore = sessionAttributeStore;

	SessionAttributes annotation = AnnotationUtils.findAnnotation(handlerType, SessionAttributes.class);
	if (annotation != null) {
		this.attributeNames.addAll(Arrays.asList(annotation.value()));
		this.attributeTypes.addAll(Arrays.<Class<?>>asList(annotation.types()));
	}

	for (String attributeName : this.attributeNames) {
		this.knownAttributeNames.add(attributeName);
	}
}
```

查看`Controller`有没有`@SessionAttributes`注解

把注解中的`value`值放入成员变量`knownAttributeNames`集合中

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

`getSessionAttributesHandler`方法获取创建好的`SessionAttributesHandler`

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

`argumentResolver`是`HandlerMethodArgumentResolver`类的实例，用来做参数解析，`returnValueHandlers`是`HandlerMethodReturnValueHandlerComposite`类的实例，用来对返回结果做处理，`parameterNameDiscoverer`用来记录参数名解析器

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

当与`@SessionAttribute`注解的`value`值相同的变量名在控制器某个方法被实例化放入`model`中，也就是被放入`Session`中后，在此处会获得这个注解的变量名的属性，源码也就是变量`sessionAttributesHandler`中有个变量`knownAttributeNames`集合，存放了`@SessionAttribute`注解的`value`值，这里通过`request`获取`Session`，然后取出`value`对应属性

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

此处的`modelMethods`在前几步的`getModelFactory`中，是获取的控制器和全局`@ModelAttribute`方法的集合。

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

将获取到的带`@ModelAttribute`注解的方法`method`包装为`InvocableHandlerMethod`类型然后添加到`List<InvocableHandlerMethod>`的集合中`attrMethods`中，在创建`ModelFactory`类的实例的时候利用构造函数将其集合类型转化为`List<ModelMethod>`的集合`modelMethods`

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

以上将`@ModelAttribute`方法的集合做了包装，`invocableHandlerMethod -> ModelMethod`，多了个`dependencies`属性，类型为`Set<String>`，用来存放当前方法中带了`@ModelAttribute`注解的**形参**的参数名

接着遍历包装后的集合，`mavContainer`中的`model`必须拥有`dependencies`中所有的属性名，此时`checkDependencies`方法返回`true`，当`ModelMethod`的`dependencies`为空时，也返回`true`，其他返回`false`

回到变量`ModelFactory`的`getNextModelMethod(mavContainer).getHandlerMethod()`

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

由于当前Demo中`@ModelAttribute`方法的形参没有带`@ModelAttribute`注解，所以这里`dependencies`为空，返回`true`

将符合条件的`ModelMethod`返回，然后移除，下面再依次返回不符合条件的`ModelMethod`

往回`invokeModelAttributeMethods`方法

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

`getNextModelMethod`获取到`ModelMetod`后，将它解包装回`InvocableHandlerMethod`，获取当前方法注解`@ModelAttribute`的`value`值，如果当前`mavContainer`中的`Model`已经有与`value`同名的属性名，则跳过，**在之前的代码，我们知道此时**`Model`**中的属性来自于**`@SessionAttributes`

然后调用`InvocableHandlerMethod`的`invokeForRequest`方法来完成执行`ModelAttribute`方法，取得返回值

```
public Object invokeForRequest(NativeWebRequest request, ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {

	Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
	if (logger.isTraceEnabled()) {
		StringBuilder sb = new StringBuilder("Invoking [");
		sb.append(getBeanType().getSimpleName()).append(".");
		sb.append(getMethod().getName()).append("] method with arguments ");
		sb.append(Arrays.asList(args));
		logger.trace(sb.toString());
	}
	Object returnValue = doInvoke(args);
	if (logger.isTraceEnabled()) {
		logger.trace("Method [" + getMethod().getName() + "] returned [" + returnValue + "]");
	}
	return returnValue;
}
```

获取方法参数数组

```
private Object[] getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {

	MethodParameter[] parameters = getMethodParameters();
	Object[] args = new Object[parameters.length];
	for (int i = 0; i < parameters.length; i++) {
		MethodParameter parameter = parameters[i];
		parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
		GenericTypeResolver.resolveParameterType(parameter, getBean().getClass());
		args[i] = resolveProvidedArgument(parameter, providedArgs);
		if (args[i] != null) {
			continue;
		}
		if (this.argumentResolvers.supportsParameter(parameter)) {
			try {
				args[i] = this.argumentResolvers.resolveArgument(
						parameter, mavContainer, request, this.dataBinderFactory);
				continue;
			}
			catch (Exception ex) {
				if (logger.isDebugEnabled()) {
					logger.debug(getArgumentResolutionErrorMessage("Error resolving argument", i), ex);
				}
				throw ex;
			}
		}
		if (args[i] == null) {
			String msg = getArgumentResolutionErrorMessage("No suitable resolver for argument", i);
			throw new IllegalStateException(msg);
		}
	}
	return args;
}
```

获取了方法参数类型数组`MethodParameter[]`

遍历该数组，`resolveParameterType`确定每个方法参数的类型，`getBean().getClass()`传入方法所在的类的`Class`

```
public static Class<?> resolveParameterType(MethodParameter methodParam, Class<?> clazz) {
	Assert.notNull(methodParam, "MethodParameter must not be null");
	Assert.notNull(clazz, "Class must not be null");
	methodParam.setContainingClass(clazz);
	methodParam.setParameterType(ResolvableType.forMethodParameter(methodParam).resolve());
	return methodParam.getParameterType();
}
```

设置成员变量`containClass`和`parameterType`，先看`forMethodParameter`方法

```
public static ResolvableType forMethodParameter(MethodParameter methodParameter) {
	return forMethodParameter(methodParameter, (Type) null);
}

public static ResolvableType forMethodParameter(MethodParameter methodParameter, Type targetType) {
	Assert.notNull(methodParameter, "MethodParameter must not be null");
	//获取方法参数所在的类的ResolvableType,其它参数TypeProvider和variableResolve为空
	ResolvableType owner = forType(methodParameter.getContainingClass()).as(methodParameter.getDeclaringClass());
	//typeProvider由methodParameter为参数，提供参数的真正类型
	//以owner创建它的内部类new DefaultVariableResolver
	return forType(targetType, new MethodParameterTypeProvider(methodParameter), owner.asVariableResolver()).
			getNested(methodParameter.getNestingLevel(), methodParameter.typeIndexesPerLevel);
}
```

`forType`是用来创建`ResolveType`类型的变量

```
public static ResolvableType forType(Type type) {
	return forType(type, null, null);
}

static ResolvableType forType(Type type, TypeProvider typeProvider, VariableResolver variableResolver) {
	if (type == null && typeProvider != null) {
		type = SerializableTypeWrapper.forTypeProvider(typeProvider);
	}
	if (type == null) {
		return NONE;
	}

	// Purge empty entries on access since we don't have a clean-up thread or the like.
	cache.purgeUnreferencedEntries();

	// For simple Class references, build the wrapper right away -
	// no expensive resolution necessary, so not worth caching...
	if (type instanceof Class) {
		return new ResolvableType(type, typeProvider, variableResolver, null);
	}

	// Check the cache - we may have a ResolvableType which has been resolved before...
	ResolvableType key = new ResolvableType(type, typeProvider, variableResolver);
	ResolvableType resolvableType = cache.get(key);
	if (resolvableType == null) {
		resolvableType = new ResolvableType(type, typeProvider, variableResolver, null);
		cache.put(resolvableType, resolvableType);
	}
	return resolvableType;
}
```

这里先用单独一个`type`，这里是`containClass`即方法参数所在的类，创建一个`ResolveType`

```
private ResolvableType(
		Type type, TypeProvider typeProvider, VariableResolver variableResolver, ResolvableType componentType) {

	this.type = type;
	this.typeProvider = typeProvider;
	this.variableResolver = variableResolver;
	this.componentType = componentType;
	this.resolved = resolveClass();
}

private Class<?> resolveClass() {
	if (this.type instanceof Class || this.type == null) {
		return (Class<?>) this.type;
	}
	if (this.type instanceof GenericArrayType) {
		Class<?> resolvedComponent = getComponentType().resolve();
		return (resolvedComponent != null ? Array.newInstance(resolvedComponent, 0).getClass() : null);
	}
	return resolveType().resolve();
}
```

因为传来的`type`是`invocableHandlerMethod`中的`bean`，所以`type`==`resolved`

第二次使用了为`null`的`targetType`，`new MethodParameterTypeProvider(methodParameter)`和`owner.asVariableResolver()`

```
public MethodParameterTypeProvider(MethodParameter methodParameter) {
	if (methodParameter.getMethod() != null) {
		this.methodName = methodParameter.getMethod().getName();
		this.parameterTypes = methodParameter.getMethod().getParameterTypes();
	}
	else {
		this.methodName = null;
		this.parameterTypes = methodParameter.getConstructor().getParameterTypes();
	}
	this.declaringClass = methodParameter.getDeclaringClass();
	this.parameterIndex = methodParameter.getParameterIndex();
	this.methodParameter = methodParameter;
}

VariableResolver asVariableResolver() {
	if (this == NONE) {
		return null;
	}
	return new DefaultVariableResolver();
}
```

创建这个`TypeProvider`通过构造函数传入了`methodParameter`，并且将它的一些属性注入到自己中

因为传入的`targetType`为`null`

```
if (type == null && typeProvider != null) {
	type = SerializableTypeWrapper.forTypeProvider(typeProvider);
}

static Type forTypeProvider(final TypeProvider provider) {
	Assert.notNull(provider, "Provider must not be null");
	if (provider.getType() instanceof Serializable || provider.getType() == null) {
		return provider.getType();
	}
	Type cached = cache.get(provider.getType());
	if (cached != null) {
		return cached;
	}
	for (Class<?> type : SUPPORTED_SERIALIZABLE_TYPES) {
		if (type.isAssignableFrom(provider.getType().getClass())) {
			ClassLoader classLoader = provider.getClass().getClassLoader();
			Class<?>[] interfaces = new Class<?>[] { type,
				SerializableTypeProxy.class, Serializable.class };
			InvocationHandler handler = new TypeProxyInvocationHandler(provider);
			cached = (Type) Proxy.newProxyInstance(classLoader, interfaces, handler);
			cache.put(provider.getType(), cached);
			return cached;
		}
	}
	throw new IllegalArgumentException("Unsupported Type class " + provider.getType().getClass().getName());
}
```

`provider.getType()`，变量`provider`为前面传来`MethodParameterTypeProvider`类型的变量，决定了参数的具体类型，它的父类`TypeProvoider`实现了`serializable`接口

```
@Override
public Type getType() {
	return this.methodParameter.getGenericParameterType();
}

public Type getGenericParameterType() {
	if (this.genericParameterType == null) {
		if (this.parameterIndex < 0) {
			this.genericParameterType = (this.method != null ? this.method.getGenericReturnType() : null);
		}
		else {
			this.genericParameterType = (this.method != null ?
				this.method.getGenericParameterTypes()[this.parameterIndex] :
				this.constructor.getGenericParameterTypes()[this.parameterIndex]);
		}
	}
	return this.genericParameterType;
}
```

调用了类`TypeProvider`的`getType`，通过之前注入的变量`MethodParameter`

因为创建类`HandlerMethodParameter`的时候注入了`Method`

```
public HandlerMethodParameter(int index) {
	super(HandlerMethod.this.bridgedMethod, index);
}
```

所以这里能获取到参数所在的方法的所有方法参数类型`this.method.getGenericParameterTypes()[this.parameterIndex]`

然后利用这返回的方法参数的类型用来创建`ResolveType`

```
if (type instanceof Class) {
	return new ResolvableType(type, typeProvider, variableResolver, null);
}
```

返回的参数类型的`type`，又用来和之前的`TypeProvider`和`VariableResolve`创建新`ResolveType`

再回到之前的方法

```
public static Class<?> resolveParameterType(MethodParameter methodParam, Class<?> clazz) {
	Assert.notNull(methodParam, "MethodParameter must not be null");
	Assert.notNull(clazz, "Class must not be null");
	methodParam.setContainingClass(clazz);
	methodParam.setParameterType(ResolvableType.forMethodParameter(methodParam).resolve());
	return methodParam.getParameterType();
}
```

这里再次调用创建好的`ResolveType`的`resolve`方法

```
public Class<?> resolve() {
	return resolve(null);
}

public Class<?> resolve(Class<?> fallback) {
	return (this.resolved != null ? this.resolved : fallback);
}
```

这里`ResolveType`的成员变量`resolve`和`type`指向的都是方法的参数类型

所以这里通过`ResolvableType.forMethodParameter(methodParam).resolve()`来找到方法参数的具体类型，然后设置到`methodParam`的`ParameterType`中

通过这样确定了方法参数的具体类型

回到`InvocalbleHandlerMethod`

```
private Object[] getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {

	MethodParameter[] parameters = getMethodParameters();
	Object[] args = new Object[parameters.length];
	for (int i = 0; i < parameters.length; i++) {
		MethodParameter parameter = parameters[i];
		parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
		GenericTypeResolver.resolveParameterType(parameter, getBean().getClass());
		args[i] = resolveProvidedArgument(parameter, providedArgs);
		if (args[i] != null) {
			continue;
		}
		if (this.argumentResolvers.supportsParameter(parameter)) {
			try {
				args[i] = this.argumentResolvers.resolveArgument(
						parameter, mavContainer, request, this.dataBinderFactory);
				continue;
			}
			catch (Exception ex) {
				if (logger.isDebugEnabled()) {
					logger.debug(getArgumentResolutionErrorMessage("Error resolving argument", i), ex);
				}
				throw ex;
			}
		}
		if (args[i] == null) {
			String msg = getArgumentResolutionErrorMessage("No suitable resolver for argument", i);
			throw new IllegalStateException(msg);
		}
	}
	return args;
}
```

确定方法参数的具体类型之后，接着调用`InvocableHandlerMethod`自身的`resolveProvidedArgument`方法处理对方法参数进行处理

```
private Object resolveProvidedArgument(MethodParameter parameter, Object... providedArgs) {
	if (providedArgs == null) {
		return null;
	}
	for (Object providedArg : providedArgs) {
		if (parameter.getParameterType().isInstance(providedArg)) {
			return providedArg;
		}
	}
	return null;
}
```

Demo中源码传入得入参`providedArgs`为`[]`，所以这里返回空

接着判断`this.argumentResolvers.supportsParameter(parameter)`

```
@Override
public boolean supportsParameter(MethodParameter parameter) {
	return getArgumentResolver(parameter) != null;
}


private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
	HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
	if (result == null) {
		for (HandlerMethodArgumentResolver methodArgumentResolver : this.argumentResolvers) {
			if (logger.isTraceEnabled()) {
				logger.trace("Testing if argument resolver [" + methodArgumentResolver + "] supports [" +
						parameter.getGenericParameterType() + "]");
			}
			if (methodArgumentResolver.supportsParameter(parameter)) {
				result = methodArgumentResolver;
				this.argumentResolverCache.put(parameter, result);
				break;
			}
		}
	}
	return result;
}
```

`argumentResolvers`属于类`HandlerMethodArgumentResolverComposite`，实现了`HandlerMethodArgumentResolver`接口，用来解析方法参数

遍历成员变量`argumentResolvers`，查找一个支持当前参数类型得参数解析器，放入缓存

当前Demo正在执行`@ModelAttribute`方法

```
@ModelAttribute("attr")
public void mda(Map map) {
	Person person = new Person("li", 12);
	map.put("person", new Person("zh", 14));
}
```

这里适用于这个的解析器为`MapMethodProcessor`

有支持当前参数的参数解析器，所以返回`true`

之后对方法参数变量`MethodParameter`进行处理

```
args[i] = this.argumentResolvers.resolveArgument(
		parameter, mavContainer, request, this.dataBinderFactory);
```

接下来

```
@Override
public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

	HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
	Assert.notNull(resolver, "Unknown parameter type [" + parameter.getParameterType().getName() + "]");
	return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
}
```

`getArgumentResover(parameter)`返回`MapMethodProcessor`

查看`MapMethodProcessor`的`resolveArgument`方法

```
@Override
public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

	return mavContainer.getModel();
}
```

返回容器中的默认`BindingAwareModelMap`

也就是说，此时`@ModelAttribute`方法中的`Map`类型的参数指向了`mavContainer`中的`Model`

方法在最后返回方法参数`args[]`数组

回到`InvocableHandlerMethod`的`invokeForRequest`方法

```
public Object invokeForRequest(NativeWebRequest request, ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {

	Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
	if (logger.isTraceEnabled()) {
		StringBuilder sb = new StringBuilder("Invoking [");
		sb.append(getBeanType().getSimpleName()).append(".");
		sb.append(getMethod().getName()).append("] method with arguments ");
		sb.append(Arrays.asList(args));
		logger.trace(sb.toString());
	}
	Object returnValue = doInvoke(args);
	if (logger.isTraceEnabled()) {
		logger.trace("Method [" + getMethod().getName() + "] returned [" + returnValue + "]");
	}
	return returnValue;
}
```

这里接收返回的参数之后，调用`doInvoke(args)`真正对`@ModelAttribute`方法执行

```
protected Object doInvoke(Object... args) throws Exception {
	ReflectionUtils.makeAccessible(getBridgedMethod());
	try {
		return getBridgedMethod().invoke(getBean(), args);
	}
	catch (IllegalArgumentException ex) {
		assertTargetBean(getBridgedMethod(), getBean(), args);
		throw new IllegalStateException(getInvocationErrorMessage(ex.getMessage(), args), ex);
	}
	catch (InvocationTargetException ex) {
		// Unwrap for HandlerExceptionResolvers ...
		Throwable targetException = ex.getTargetException();
		if (targetException instanceof RuntimeException) {
			throw (RuntimeException) targetException;
		}
		else if (targetException instanceof Error) {
			throw (Error) targetException;
		}
		else if (targetException instanceof Exception) {
			throw (Exception) targetException;
		}
		else {
			String msg = getInvocationErrorMessage("Failed to invoke controller method", args);
			throw new IllegalStateException(msg, targetException);
		}
	}
}
```

这里使用了反射的方式来执行方法，然后返回返回值，所以我们现在也知道了操作`@ModelAttribute`方法中的`Map`入参，就等于是在操作`mavContainer`的`Model`

回到`ModelFactory`的`invokeModelAttributeMethods`

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

如果`@ModelAttribute`方法不是`void`类型

传入参数为`@ModelAttribute`方法的返回值和`attrMethod.getReturnType()`

```
public MethodParameter getReturnType() {
	return new HandlerMethodParameter(-1);
}
```

接下来获取返回值名

```
public static String getNameForReturnValue(Object returnValue, MethodParameter returnType) {
	ModelAttribute annotation = returnType.getMethodAnnotation(ModelAttribute.class);
	if (annotation != null && StringUtils.hasText(annotation.value())) {
		return annotation.value();
	}
	else {
		Method method = returnType.getMethod();
		Class<?> resolvedType = GenericTypeResolver.resolveReturnType(method, returnType.getContainingClass());
		return Conventions.getVariableNameForReturnType(method, resolvedType, returnValue);
	}
}
```

第一行获取方法参数所在方法是否有`@ModelAttribute`注解

```
public <T extends Annotation> T getMethodAnnotation(Class<T> annotationType) {
	return getAnnotatedElement().getAnnotation(annotationType);
}

public AnnotatedElement getAnnotatedElement() {
	// NOTE: no ternary expression to retain JDK <8 compatibility even when using
	// the JDK 8 compiler (potentially selecting java.lang.reflect.Executable
	// as common type, with that new base class not available on older JDKs)
	if (this.method != null) {
		return this.method;
	}
	else {
		return this.constructor;
	}
}
```

如果存在注解，返回注解的`value`属性

然后检测`mavContainer`中的`Model`是否已经包含该`value`属性的值为`key`的键值对，如果不包含，则将`value`和返回值添加到`Model`中

回到`ModelFactory`的`initModel`方法

```
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

上面执行完`invokeModelAttributeMethods(request, mavContainer)`之后

`findSessionAttributeArguments(handlerMethod)`返回的结果是响应的`Handler`方法中参数带`@ModelAttribute`注解的`value`值和`@SessionAttributes`注解的`value`相同时的`value`，或者带`@ModelAttribute`注解的方法参数属于`@SessionAttributes`注解的`Types`范围内的`value`

然后将`value`作为`key`取出因为`Session`中相应属性，添加到`mavContainer`的`Model`中

取出的值为null时，抛出异常，也就是说当`@SessionAttributes`和`Handler`方法中参数的`@ModelAttribute`对应时，例如`value`相同，在执行完ModelAttribute方法之后，`mavContainer`中必须有值能传给这个`@ModelAttribute`方法参数

以上就是`modelFactory.initModel(webRequest, mavContainer, requestMappingMethod)`完成的事

接下来是对`Controller`中响应网页请求的方法的执行

该方法

```
@RequestMapping("/Person")
public String testParam(Person person,HttpSession session) {
	System.out.println(person.getName()+" "+person.getAge());
	session.setAttribute("person", person);
	return "person";
}
```

依旧回到`RequestMappingHandlerAdapter`

```
requestMappingMethod.invokeAndHandle(webRequest, mavContainer)
```

其中`requestMappingMethod`是由`InvocableHandlerMethod`包装成的`ServletInvocableHandlerMethod`，它指向响应了客户端请求的方法

```
ServletInvocableHandlerMethod requestMappingMethod = createRequestMappingMethod(handlerMethod, binderFactory)


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

进入`invokeAndHandle(webRequest, mavContainer)`，调用响应方法

```
public void invokeAndHandle(ServletWebRequest webRequest,
		ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {

	Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
	setResponseStatus(webRequest);

	if (returnValue == null) {
		if (isRequestNotModified(webRequest) || hasResponseStatus() || mavContainer.isRequestHandled()) {
			mavContainer.setRequestHandled(true);
			return;
		}
	}
	else if (StringUtils.hasText(this.responseReason)) {
		mavContainer.setRequestHandled(true);
		return;
	}

	mavContainer.setRequestHandled(false);
	try {
		this.returnValueHandlers.handleReturnValue(
				returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
	}
	catch (Exception ex) {
		if (logger.isTraceEnabled()) {
			logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
		}
		throw ex;
	}
}
```

`invokeForRequest`方法调用响应方法获取返回值

```
public Object invokeForRequest(NativeWebRequest request, ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {

	Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
	if (logger.isTraceEnabled()) {
		StringBuilder sb = new StringBuilder("Invoking [");
		sb.append(getBeanType().getSimpleName()).append(".");
		sb.append(getMethod().getName()).append("] method with arguments ");
		sb.append(Arrays.asList(args));
		logger.trace(sb.toString());
	}
	Object returnValue = doInvoke(args);
	if (logger.isTraceEnabled()) {
		logger.trace("Method [" + getMethod().getName() + "] returned [" + returnValue + "]");
	}
	return returnValue;
}
```

接下来的步骤与执行`@ModelAttribute`方法差不多，通过`GenericTypeResolver`确认了`MethodParameter`中的`ParameterType`

接着获取适用`HandlerMethodArgumentResolver`时，这里使用的是`ServletModelAttributeMethodProcessor`

```
public boolean supportsParameter(MethodParameter parameter) {
	if (parameter.hasParameterAnnotation(ModelAttribute.class)) {
		return true;
	}
	else if (this.annotationNotRequired) {
		return !BeanUtils.isSimpleProperty(parameter.getParameterType());
	}
	else {
		return false;
	}
}
```

`BeanUtils.isSimpleProperty(parameter.getParameterType()`方法确认了`MethodParameter`不是简单属性，英文解释如下

Check if the given type represents a "simple" value type:a primitive, a String or other CharSequence, a Number, a Date,a URI, a URL, a Locale or a Class.

然后使用`argumentResolver`处理`MethodParameter`

```
public final Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

	String name = ModelFactory.getNameForParameter(parameter);
	Object attribute = (mavContainer.containsAttribute(name) ?
			mavContainer.getModel().get(name) : createAttribute(name, parameter, binderFactory, webRequest));

	WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
	if (binder.getTarget() != null) {
		bindRequestParameters(binder, webRequest);
		validateIfApplicable(binder, parameter);
		if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
			throw new BindException(binder.getBindingResult());
		}
	}

	// Add resolved attribute and BindingResult at the end of the model
	Map<String, Object> bindingResultModel = binder.getBindingResult().getModel();
	mavContainer.removeAttributes(bindingResultModel);
	mavContainer.addAllAttributes(bindingResultModel);

	return binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
}
```

`getNameForParameter(parameter)`决定了变量的名字，如果变量被`@ModelAttribute`注解修饰，则`name`取此`value`属性，当变量为`Array`或者`Collection`或者普通`Class`，则取成员变量或者全限定名最后一个.之后的字符作为`name`

接着如果`mavContainer`中存在`key`为`name`，则取出对应的值，否则`createAttribute`方法获取`MethodParameter`的`Class`来创建新对象

接着`binderFactory.createBinder`创建一个数据绑定器

```
@Override
public final WebDataBinder createBinder(NativeWebRequest webRequest, Object target, String objectName)
		throws Exception {
	WebDataBinder dataBinder = createBinderInstance(target, objectName, webRequest);
	if (this.initializer != null) {
		this.initializer.initBinder(dataBinder, webRequest);
	}
	initBinder(dataBinder, webRequest);
	return dataBinder;
}
```

创建`binderFactory`的时候，`RequestMappingHandlerAdapter`给它注入了一个`ConfigurableWebBindinginitializer`类的变量，然后用它来初步初始化`dataBinder`，这个`dataBinder`的类型为`ExtendedServletRequestDataBinder`

```
@Override
public void initBinder(WebDataBinder binder, WebRequest request) {
	binder.setAutoGrowNestedPaths(this.autoGrowNestedPaths);
	if (this.directFieldAccess) {
		binder.initDirectFieldAccess();
	}
	if (this.messageCodesResolver != null) {
		binder.setMessageCodesResolver(this.messageCodesResolver);
	}
	if (this.bindingErrorProcessor != null) {
		binder.setBindingErrorProcessor(this.bindingErrorProcessor);
	}
	if (this.validator != null && binder.getTarget() != null &&
			this.validator.supports(binder.getTarget().getClass())) {
		binder.setValidator(this.validator);
	}
	if (this.conversionService != null) {
		binder.setConversionService(this.conversionService);
	}
	if (this.propertyEditorRegistrars != null) {
		for (PropertyEditorRegistrar propertyEditorRegistrar : this.propertyEditorRegistrars) {
			propertyEditorRegistrar.registerCustomEditors(binder);
		}
	}
}
```

`initializer`中有个重要变量`conversionService`，属于接口`ConversionService`，它指向了一个`DefaultFormattingConversionService`类的实例，该实例也是由`SpringMvc`注入，存放`SpringMvc`自带的类型转换器converters

接着由`binderFactory`的`init`方法初始化`dataBinder`

```
@Override
public void initBinder(WebDataBinder binder, NativeWebRequest request) throws Exception {
	for (InvocableHandlerMethod binderMethod : this.binderMethods) {
		if (isBinderMethodApplicable(binderMethod, binder)) {
			Object returnValue = binderMethod.invokeForRequest(request, null, binder);
			if (returnValue != null) {
				throw new IllegalStateException("@InitBinder methods should return void: " + binderMethod);
			}
		}
	}
}
```

这里真正调用我们使用`@InitBinder`注解的方法，同样的`invokeForRequest`方法，这里传入的`binder`，会作为调用该方法的参数

```
@InitBinder
public void initBinder(WebDataBinder binder) {
	SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
	sdf.setLenient(false);
	binder.registerCustomEditor(Date.class, new CustomDateEditor(sdf, true));
}
```

所以我们`@InitBinder`的方法，其实就是在初始化`SpringMvc`为每个请求创建的`dataBinder`

我们这里使用了`registerCustomEditor`注册了它`SpringMvc`中已经定义好的`PropertyEditor`，用来将字符串转化为`java.util.Date`，它继承了`PropertyEditorSupport`，如果要使用我们自定义的`PropertyEditor`，也是如此，覆写`getAsText`和`setAsText`方法，然后注册

```
@Override
public String getAsText() {
	Date value = (Date) getValue();
	return (value != null ? this.dateFormat.format(value) : "");
}

@Override
public void setAsText(String text) throws IllegalArgumentException {
	if (this.allowEmpty && !StringUtils.hasText(text)) {
		// Treat empty String as null value.
		setValue(null);
	}
	//exactDateLength可以指定需要转化成Date类型的字符串长度
	else if (text != null && this.exactDateLength >= 0 && text.length() != this.exactDateLength) {
		throw new IllegalArgumentException(
				"Could not parse date: it is not exactly" + this.exactDateLength + "characters long");
	}
	else {
		try {
			//使用我们注入的DateFormat将字符串转化成Date
			setValue(this.dateFormat.parse(text));
		}
		catch (ParseException ex) {
			throw new IllegalArgumentException("Could not parse date: " + ex.getMessage(), ex);
		}
	}
}

```

我们看是怎么用`dataBinder`注册的

```
@Override
public void registerCustomEditor(Class<?> requiredType, PropertyEditor propertyEditor) {
	getPropertyEditorRegistry().registerCustomEditor(requiredType, propertyEditor);
}
```

获取`PropertyEditorRegistry`

```
protected PropertyEditorRegistry getPropertyEditorRegistry() {
	if (getTarget() != null) {
		return getInternalBindingResult().getPropertyAccessor();
	}
	else {
		return getSimpleTypeConverter();
	}
}

protected AbstractPropertyBindingResult getInternalBindingResult() {
	if (this.bindingResult == null) {
		initBeanPropertyAccess();
	}
	return this.bindingResult;
}

public void initBeanPropertyAccess() {
	Assert.state(this.bindingResult == null,
			"DataBinder is already initialized - call initBeanPropertyAccess before other configuration methods");
	this.bindingResult = new BeanPropertyBindingResult(
			getTarget(), getObjectName(), isAutoGrowNestedPaths(), getAutoGrowCollectionLimit());
	if (this.conversionService != null) {
		this.bindingResult.initConversion(this.conversionService);
	}
}
```

这里创建了一个`BeanPropertyBindingResult`类的实例，将`dataBinder`中的`target`和`objectName`传入，让`dataBinder`的`bindingResult`指向它，然后返回这个对象

然后调用这个实例的`getPropertyAccessor`方法，获取属性存取器

```
@Override
public final ConfigurablePropertyAccessor getPropertyAccessor() {
	if (this.beanWrapper == null) {
		this.beanWrapper = createBeanWrapper();
		this.beanWrapper.setExtractOldValueForEditor(true);
		this.beanWrapper.setAutoGrowNestedPaths(this.autoGrowNestedPaths);
		this.beanWrapper.setAutoGrowCollectionLimit(this.autoGrowCollectionLimit);
	}
	return this.beanWrapper;
}

protected BeanWrapper createBeanWrapper() {
	Assert.state(this.target != null, "Cannot access properties on null bean instance '" + getObjectName() + "'!");
	return PropertyAccessorFactory.forBeanPropertyAccess(this.target);
}

public static BeanWrapper forBeanPropertyAccess(Object target) {
	return new BeanWrapperImpl(target);
}
```

最后返回的是一个`BeanWrapperImpl`类的实例，并且让`BeanPropertyBindingResult`的`beanWrapper`指向它，这个类继承了`AbstractPropertyAccessor`和`PropertyEditorRegistrySupport`

然后开始注册`PropertyEditor`，也就是执行我们自定义的`@InitBinder`注解的方法

```
@Override
public void registerCustomEditor(Class<?> requiredType, PropertyEditor propertyEditor) {
	registerCustomEditor(requiredType, null, propertyEditor);
}

@Override
public void registerCustomEditor(Class<?> requiredType, String propertyPath, PropertyEditor propertyEditor) {
	if (requiredType == null && propertyPath == null) {
		throw new IllegalArgumentException("Either requiredType or propertyPath is required");
	}
	if (propertyPath != null) {
		if (this.customEditorsForPath == null) {
			this.customEditorsForPath = new LinkedHashMap<String, CustomEditorHolder>(16);
		}
		this.customEditorsForPath.put(propertyPath, new CustomEditorHolder(propertyEditor, requiredType));
	}
	else {
		if (this.customEditors == null) {
			this.customEditors = new LinkedHashMap<Class<?>, PropertyEditor>(16);
		}
		this.customEditors.put(requiredType, propertyEditor);
		this.customEditorCache = null;
	}
}
```

然后将`PropertyEditor`放入`customEditors`，此时，`@InitBinder`注解的方法执行完毕

```
@Override
public void initBinder(WebDataBinder binder, NativeWebRequest request) throws Exception {
	for (InvocableHandlerMethod binderMethod : this.binderMethods) {
		if (isBinderMethodApplicable(binderMethod, binder)) {
			Object returnValue = binderMethod.invokeForRequest(request, null, binder);
			if (returnValue != null) {
				throw new IllegalStateException("@InitBinder methods should return void: " + binderMethod);
			}
		}
	}
}
```

这里可以看到`@InitBinder`注解的方法不能有返回值

```
@Override
public final WebDataBinder createBinder(NativeWebRequest webRequest, Object target, String objectName)
		throws Exception {
	WebDataBinder dataBinder = createBinderInstance(target, objectName, webRequest);
	if (this.initializer != null) {
		this.initializer.initBinder(dataBinder, webRequest);
	}
	initBinder(dataBinder, webRequest);
	return dataBinder;
}

public final Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

	String name = ModelFactory.getNameForParameter(parameter);
	Object attribute = (mavContainer.containsAttribute(name) ?
			mavContainer.getModel().get(name) : createAttribute(name, parameter, binderFactory, webRequest));

	WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
	if (binder.getTarget() != null) {
		bindRequestParameters(binder, webRequest);
		validateIfApplicable(binder, parameter);
		if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
			throw new BindException(binder.getBindingResult());
		}
	}
```

初始化完`dataBinder`之后，开始绑定请求数据`bindRequestParameters(binder, webRequest)`

```
	protected void bindRequestParameters(WebDataBinder binder, NativeWebRequest request) {
		ServletRequest servletRequest = request.getNativeRequest(ServletRequest.class);
		ServletRequestDataBinder servletBinder = (ServletRequestDataBinder) binder;
		servletBinder.bind(servletRequest);
	}

```

`RequestFacade`在前面被封装成`ServletWebRequest`，这里又解包装
然后将`binder`从`ExtendedServletRequestDataBinder`向上转型为`ServletRequestDataBinder`

```
public void bind(ServletRequest request) {
	MutablePropertyValues mpvs = new ServletRequestParameterPropertyValues(request);
	MultipartRequest multipartRequest = WebUtils.getNativeRequest(request, MultipartRequest.class);
	if (multipartRequest != null) {
		bindMultipart(multipartRequest.getMultiFileMap(), mpvs);
	}
	addBindValues(mpvs, request);
	doBind(mpvs);
}
```

看mpvs是什么

```
public ServletRequestParameterPropertyValues(ServletRequest request) {
	this(request, null, null);
}

public ServletRequestParameterPropertyValues(ServletRequest request, String prefix, String prefixSeparator) {
	super(WebUtils.getParametersStartingWith(
			request, (prefix != null ? prefix + prefixSeparator : null)));
}

public static Map<String, Object> getParametersStartingWith(ServletRequest request, String prefix) {
	Assert.notNull(request, "Request must not be null");
	Enumeration<String> paramNames = request.getParameterNames();
	Map<String, Object> params = new TreeMap<String, Object>();
	if (prefix == null) {
		prefix = "";
	}
	while (paramNames != null && paramNames.hasMoreElements()) {
		String paramName = paramNames.nextElement();
		if ("".equals(prefix) || paramName.startsWith(prefix)) {
			String unprefixed = paramName.substring(prefix.length());
			String[] values = request.getParameterValues(paramName);
			if (values == null || values.length == 0) {
				// Do nothing, no values found at all.
			}
			else if (values.length > 1) {
				params.put(unprefixed, values);
			}
			else {
				params.put(unprefixed, values[0]);
			}
		}
	}
	return params;
}
```

这里`request.getParameterNames()`获取请求中传来的参数的名字的枚举集合，然后`request.getParameterValues(paramName)`获取请求中参数的值

继续构造方法

```
public MutablePropertyValues(Map<?, ?> original) {
	// We can optimize this because it's all new:
	// There is no replacement of existing property values.
	if (original != null) {
		this.propertyValueList = new ArrayList<PropertyValue>(original.size());
		for (Map.Entry<?, ?> entry : original.entrySet()) {
			this.propertyValueList.add(new PropertyValue(entry.getKey().toString(), entry.getValue()));
		}
	}
	else {
		this.propertyValueList = new ArrayList<PropertyValue>(0);
	}
}
```

`original`存放了请求中参数的名和值，将其从`Map`类型转换到`PropertyValue`的数组队列

```
public void bind(ServletRequest request) {
	MutablePropertyValues mpvs = new ServletRequestParameterPropertyValues(request);
	MultipartRequest multipartRequest = WebUtils.getNativeRequest(request, MultipartRequest.class);
	if (multipartRequest != null) {
		bindMultipart(multipartRequest.getMultiFileMap(), mpvs);
	}
	addBindValues(mpvs, request);
	doBind(mpvs);
}
```

`addBindValues(mpvc,request)`将`pathVariable`变量添加入`mpvs`中，`key`重复时跳过

然后`doBind(mpvs)`把`mvps`中的参数绑定到`dataBinder`的`target`中，也就是目标`handlerMethod`方法得参数中，中间利用到的转换器有默认的`ConversionService`中注入的转换器，例如`java.lang.String`转换到`java.lang.Integer`的转换器`StringToNumber`

```
final class StringToNumberConverterFactory implements ConverterFactory<String, Number> {

	@Override
	public <T extends Number> Converter<String, T> getConverter(Class<T> targetType) {
		return new StringToNumber<T>(targetType);
	}

	private static final class StringToNumber<T extends Number> implements Converter<String, T> {

		private final Class<T> targetType;

		public StringToNumber(Class<T> targetType) {
			this.targetType = targetType;
		}

		@Override
		public T convert(String source) {
			if (source.length() == 0) {
				return null;
			}
			return NumberUtils.parseNumber(source, this.targetType);
		}
	}

}
```

回到`ServletModelAttributeMethodProcessor`的`resolveArgument`

```
public final Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

	String name = ModelFactory.getNameForParameter(parameter);
	Object attribute = (mavContainer.containsAttribute(name) ?
			mavContainer.getModel().get(name) : createAttribute(name, parameter, binderFactory, webRequest));

	WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
	if (binder.getTarget() != null) {
		bindRequestParameters(binder, webRequest);
		validateIfApplicable(binder, parameter);
		if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
			throw new BindException(binder.getBindingResult());
		}
	}

	// Add resolved attribute and BindingResult at the end of the model
	Map<String, Object> bindingResultModel = binder.getBindingResult().getModel();
	mavContainer.removeAttributes(bindingResultModel);
	mavContainer.addAllAttributes(bindingResultModel);

	return binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
}
```

`validateIfApplicable(binder,webRequest)`这句的作用在`@validated`注解的使用，对`POJO`类的属性进行jrs303验证

最后将`dataBinder`中的`BeanPropertyBindingResult`的已经和请求中的参数做了数据绑定之后的`target`和处理`@validated`注解之后的`BeanPropertyBindingResult`也就是它自身绑定到`mavContainer`的`Model`中

回到最初执行控制器`Controller`的`ServletInvocableHandlerMethod`

```
public Object invokeForRequest(NativeWebRequest request, ModelAndViewContainer mavContainer,
		Object... providedArgs) throws Exception {

	Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
	if (logger.isTraceEnabled()) {
		StringBuilder sb = new StringBuilder("Invoking [");
		sb.append(getBeanType().getSimpleName()).append(".");
		sb.append(getMethod().getName()).append("] method with arguments ");
		sb.append(Arrays.asList(args));
		logger.trace(sb.toString());
	}
	Object returnValue = doInvoke(args);
	if (logger.isTraceEnabled()) {
		logger.trace("Method [" + getMethod().getName() + "] returned [" + returnValue + "]");
	}
	return returnValue;
}
```

`getMethodArgumentValues`方法中，`argumentResolver`对形参`methodParameter`解析之后，获取到要执行的方法的参数的具体的值

然后利用反射执行方法，然后返回返回值

```
public void invokeAndHandle(ServletWebRequest webRequest,
		ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {

	Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
	setResponseStatus(webRequest);

	if (returnValue == null) {
		if (isRequestNotModified(webRequest) || hasResponseStatus() || mavContainer.isRequestHandled()) {
			mavContainer.setRequestHandled(true);
			return;
		}
	}
	else if (StringUtils.hasText(this.responseReason)) {
		mavContainer.setRequestHandled(true);
		return;
	}

	mavContainer.setRequestHandled(false);
	try {
		this.returnValueHandlers.handleReturnValue(
				returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
	}
	catch (Exception ex) {
		if (logger.isTraceEnabled()) {
			logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
		}
		throw ex;
	}
}
```

`Controller`中方法返回值一般就是`view`了

`RequestMappingHandlerAdapter`在将`Controller`中目标方法包装成`ServletInvocableHandlerMethod`时注入了`HandlerMethodReturnValueHandlerComposite`的实例，其中包含了许多处理返回值的`HandlerMethodReturnValueHandler`


其中这里处理`view`视图的是`ViewNameMethodReturnValueHandler`

```
@Override
public void handleReturnValue(Object returnValue, MethodParameter returnType,
		ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

	if (returnValue == null) {
		return;
	}
	else if (returnValue instanceof String) {
		String viewName = (String) returnValue;
		mavContainer.setViewName(viewName);
		if (isRedirectViewName(viewName)) {
			mavContainer.setRedirectModelScenario(true);
		}
	}
	else {
		// should not happen
		throw new UnsupportedOperationException("Unexpected return type: " +
				returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
	}
}
```



返回值为`String`类型的时候，将它作为`viewName`放入`mavContainer`，然后判断`viewName`中是否以`redirect:`开头的设置`redirectModelScenario`属性

此时，`RequestMappingHandlerAdapter`的`InvokeHandleMethod`，也就是`handler`方法执行的入口就只剩下最后一步

```
return getModelAndView(mavContainer, modelFactory, webRequest);


private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
		ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {

	modelFactory.updateModel(webRequest, mavContainer);
	if (mavContainer.isRequestHandled()) {
		return null;
	}
	ModelMap model = mavContainer.getModel();
	ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model);
	if (!mavContainer.isViewReference()) {
		mav.setView((View) mavContainer.getView());
	}
	if (model instanceof RedirectAttributes) {
		Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
		HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
		RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
	}
	return mav;
}
```

前面看到返回值为空时，设置`requestHandled`为`true`，使用`mavContainer`中的`Model`和`viewName`创建一个`ModelAndView`返回，接下来就是对视图进行处理

### 步骤总结

1. 创建`WebDataBinderFactory`，用来处理请求中属性绑定到形参

2. 创建`ModelFactory`，处理`@ModelAttribute`方法

3. 创建`ModelAndViewContainer`，用来作为`Model`和`View`的容器

4. 执行`@ModelAttribute`方法，在这之前先将`@SessionAttributes`注解的属性从`Session`中取出放入`mavContainer`的`Model`中

5. 执行`Controller`目标方法，主要通过获取方法的形参，然后根据形参选择适用的`HandlerMethodArgumentResolver`对形参进行解析处理，并且在其中使用`WebDataBinderFactory`创建`DataBinder`将请求中附带的属性绑定到形参中，然后利用反射的方式来执行方法

6. 获取到方法的返回值过后，利用合适的`HandlerMethodReturnValueHandler`对返回值也就是`view`进行处理，然后放入`mavContainer`