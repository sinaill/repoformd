title: SpringMVC的@ModelAttribute注解原理
categories: 框架
tags: 
	- SpringMVC

---

### @ModelAttribute注解的方法给模型中添加数据

```
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

这是在`RequestMappingHandlerAdapter`执行目标`Controller`的方法的主要流程

```
ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
```

这里创建了`ModelFactory`，里面有注入`@ModelAttribute`注解了的方法

```
modelFactory.initModel(webRequest, mavContainer, requestMappingMethod);
```

后面用这个来初始化`Model`

可以看源码，是怎么初始化的

```
public void initModel(NativeWebRequest request, ModelAndViewContainer mavContainer, HandlerMethod handlerMethod)
		throws Exception {

	Map<String, ?> sessionAttributes = this.sessionAttributesHandler.retrieveAttributes(request);
	mavContainer.mergeAttributes(sessionAttributes);
	//
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

进入`invokeModelAttributeMethods`，这里是调用`@ModelAttribute`注解的方法

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

#### 有返回值时

当`@ModelAttribute`注解的方法的返回值类型不为`void`时，也就是有返回值时，这里的`returnValueName`的命名规则可以看`getNameForReturnValue`方法

```
/**
 * Derive the model attribute name for the given return value using one of:
 * <ol>
 * 	<li>The method {@code ModelAttribute} annotation value
 * 	<li>The declared return type if it is more specific than {@code Object}
 * 	<li>The actual return value type
 * </ol>
 * @param returnValue the value returned from a method invocation
 * @param returnType the return type of the method
 * @return the model name, never {@code null} nor empty
 */
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


public static String getVariableNameForReturnType(Method method, Class<?> resolvedType, Object value) {
	Assert.notNull(method, "Method must not be null");

	if (Object.class.equals(resolvedType)) {
		if (value == null) {
			throw new IllegalArgumentException("Cannot generate variable name for an Object return type with null value");
		}
		return getVariableName(value);
	}

	Class<?> valueClass;
	boolean pluralize = false;

	if (resolvedType.isArray()) {
		valueClass = resolvedType.getComponentType();
		pluralize = true;
	}
	else if (Collection.class.isAssignableFrom(resolvedType)) {
		valueClass = GenericCollectionTypeResolver.getCollectionReturnType(method);
		if (valueClass == null) {
			if (!(value instanceof Collection)) {
				throw new IllegalArgumentException(
						"Cannot generate variable name for non-typed Collection return type and a non-Collection value");
			}
			Collection<?> collection = (Collection<?>) value;
			if (collection.isEmpty()) {
				throw new IllegalArgumentException(
						"Cannot generate variable name for non-typed Collection return type and an empty Collection value");
			}
			Object valueToCheck = peekAhead(collection);
			valueClass = getClassForValue(valueToCheck);
		}
		pluralize = true;
	}
	else {
		valueClass = resolvedType;
	}

	String name = ClassUtils.getShortNameAsProperty(valueClass);
	return (pluralize ? pluralize(name) : name);
}
```

看到命名规则为

- 取方法上`@ModelAttribute`注解的`value`值
- 直接取返回的值的类型`Class`的全限定名最后一个`.`之后的字符，但是当返回值的类型为`Object`时，要取其实际类型
- 返回值的类型为`Array`或者`Collection`，获取其中的值的类型的全限定名`.`后面的字符，然后在后面继续拼接`List`

然后看`mavContainer`中是否已经含有该属性名称的属性，如果没有的话，将该属性添加进去，否则则什么都不做

#### 没有返回值

`@ModelAttribute`注解的方法不用返回值往模型`mavContainer`中添加数据的方法是在入参中定义一个`java.util.Map`类型的入参

```
@ModelAttribute
public void test(Map map){

}
```

然后我们在目标方法也定义一个入参`java.util.Map`，就能在方法中在这个入参里获取到我们之前在`@ModelAttribute`方法中添加的属性

从什么地方入手，我们看反射调用`@ModelAttribute`注解的方法的入参是什么

首先查看对入参类型为`java.util.Map`时选择的参数解析器`ArgumentResolver`

```
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
```

选择的参数解析器`ArgumentResolver`为`MapMethodProcessor`

我们查看它的`supportsParameter`方法和`resolveArgument`方法

```
@Override
public boolean supportsParameter(MethodParameter parameter) {
	return Map.class.isAssignableFrom(parameter.getParameterType());
}

@Override
public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

	return mavContainer.getModel();
}
```

可以看到适用于入参为`Java.util.Map`的时候，所以`ModelMap`和`Model`也行

我们看它是怎么解析的，直接返回了`mavContainer`中的`BindingAwareModelMap`对象，也就是说我们在执行`Controller`中的方法的时候，入参为`java.util.Map`的时候，都回被解析成`mavContainer`中的`BindingAwareModelMap`，所以我们能在不同方法中使用同一个`java.util.Map`入参来共享变量，估计`@InitBinder`方法也可以这样

### 与@SessionAttributes合用

在执行`@ModelAttribute`的方法之前，还有对`@SessionAttributes`注解属性的行为

```
public void initModel(NativeWebRequest request, ModelAndViewContainer mavContainer, HandlerMethod handlerMethod)
		throws Exception {

	Map<String, ?> sessionAttributes = this.sessionAttributesHandler.retrieveAttributes(request);
	mavContainer.mergeAttributes(sessionAttributes);
	//
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

首先，`Controller`类上的注解`@SessionAttributes`的`value`值，当往`Model`中添加属性时，属性名与这个`value`值相同时，会被放入`Session`中，然后这里会以`@SessionAttributes`注解的值为`key`，从`Session`中取出，合并到`mavContainer`的`BindingAwareModelMap`中

然后看调用`@ModelAttribute`方法之前

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

这里先获取了`@ModelAttribute`的`value`，如果`BindingAwareModelMap`中已经有与`value`相同的`key`，则跳过这个`@ModelAttribute`方法的调用

还有一个特殊的用法，可以在`Controller`方法中使用`@ModelAttribute`注解入参，它的`value`与`SessionAttribute`的`value`相同时，这个入参也可以接收到`Session`中的属性

它的原理在于，我们看处理`@ModelAttribute`注解参数的处理解析器`ModelAttributeMethodProcessor`

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

它先获取了入参`@ModelAttribute`的`value`，然后如果`BindingAwareModelMap`有与该值相同的`key`，取出该属性`attribute`，后面就是将`request`请求中的属性绑定到这个`attribute`了，然后这个`attribute`就作为调用该`Controller`方法的参数来反射调用

此时在整个流程中，在完成以上动作前，`@ModelAttribute`方法已经执行完了，所以已经完成了将`Session`中属性合并到`BindingAwareModelMap`中

其实一般入参分两种情况，属于简单类的，例如`CharSequence`、`Number`、`Date`、`Class`，是由`RequestParamMethodArgumentResolver`这个类负责解析，其他复杂类，例如我们自定义的类，和用`@ModelAttribute`注解的入参都是由`ServletModelAttributeMethodProcessor`负责处理

所以复杂类入参没有`@ModelAttribute`注解也可以接收到`Session`中属性

但是如果我们要这样使用`@SessionAttributes`和`@ModelAttribute`的话，注意以下

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

private List<String> findSessionAttributeArguments(HandlerMethod handlerMethod) {
	List<String> result = new ArrayList<String>();
	for (MethodParameter parameter : handlerMethod.getMethodParameters()) {
		if (parameter.hasParameterAnnotation(ModelAttribute.class)) {
			String name = getNameForParameter(parameter);
			if (this.sessionAttributesHandler.isHandlerSessionAttribute(name, parameter.getParameterType())) {
				result.add(name);
			}
		}
	}
	return result;
}


public static String getNameForParameter(MethodParameter parameter) {
	ModelAttribute annot = parameter.getParameterAnnotation(ModelAttribute.class);
	String attrName = (annot != null) ? annot.value() : null;
	return StringUtils.hasText(attrName) ? attrName :  Conventions.getVariableNameForParameter(parameter);
}
```

可以看到执行完`@ModelAttribute`方法之后，在执行`Controller`目标方法之前，它要判断`BindingAwareModelMap`中有无要传递的属性，没有的话会抛出异常

也就是说要这样用的话，要保证在`@ModelAttribute`方法中就将要传递的属性和与`@SessionAttributes`和入参`@ModelAttribute`对应的属性名放入`BindingAwareModelMap`中，或者已经在`Session`中含有该属性和对应属性名，然后会在调用`@ModelAttribute`方法前被合并到`BindingAwareModelMap`中，否则会抛出异常