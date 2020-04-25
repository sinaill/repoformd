title: SpringMVC对Controller目标方法返回mav的处理
categories: 框架
tags: 
	- SpringMVC

---

### 流程

在`DispatchServlet`中

```
mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

执行完`Controller`的目标方法后，然后一个带有`Model`和`view`的`ModelAndView`对象

接下来就是对视图的处理

```
applyDefaultViewName(request, mv);


private void applyDefaultViewName(HttpServletRequest request, ModelAndView mv) throws Exception {
	if (mv != null && !mv.hasView()) {
		mv.setViewName(getDefaultViewName(request));
	}
}
```

判断当`ModelAndView`对象中的`view`为空时，设置默认视图名

```
@Override
public String getViewName(HttpServletRequest request) {
	String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
	return (this.prefix + transformPath(lookupPath) + this.suffix);
}
```

这是设置默认视图名的代码段，这里获取的`lookupPath`为`getServletPath()`的返回结果，也就是除了项目名名以外的后面部分，`transformPath`方法的作用是将`lookupPath`的前后的`/`去掉，然后将最后一个`.`和后面的字符去掉，取剩余的部分

然后开始处理视图结果

```
processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);




private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
		HandlerExecutionChain mappedHandler, ModelAndView mv, Exception exception) throws Exception {

	boolean errorView = false;

	if (exception != null) {
		if (exception instanceof ModelAndViewDefiningException) {
			logger.debug("ModelAndViewDefiningException encountered", exception);
			mv = ((ModelAndViewDefiningException) exception).getModelAndView();
		}
		else {
			Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
			mv = processHandlerException(request, response, handler, exception);
			errorView = (mv != null);
		}
	}

	// Did the handler return a view to render?
	if (mv != null && !mv.wasCleared()) {
		render(mv, request, response);
		if (errorView) {
			WebUtils.clearErrorRequestAttributes(request);
		}
	}
	else {
		if (logger.isDebugEnabled()) {
			logger.debug("Null ModelAndView returned to DispatcherServlet with name '" + getServletName() +
					"': assuming HandlerAdapter completed request handling");
		}
	}

	if (WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
		// Concurrent handling started during a forward
		return;
	}

	if (mappedHandler != null) {
		mappedHandler.triggerAfterCompletion(request, response, null);
	}
}
```

传进来的`excetpion`为`null`

判断`ModelAndView`引用不为空，并且`ModelAndView`内部的`Model`和`View`也不为空时，开始渲染视图

```
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
	// Determine locale for request and apply it to the response.
	//locale表示使用者当地语言
	Locale locale = this.localeResolver.resolveLocale(request);
	response.setLocale(locale);

	View view;
	if (mv.isReference()) {
		// We need to resolve the view name.
		view = resolveViewName(mv.getViewName(), mv.getModelInternal(), locale, request);
		if (view == null) {
			throw new ServletException("Could not resolve view with name '" + mv.getViewName() +
					"' in servlet with name '" + getServletName() + "'");
		}
	}
	else {
		// No need to lookup: the ModelAndView object contains the actual View object.
		view = mv.getView();
		if (view == null) {
			throw new ServletException("ModelAndView [" + mv + "] neither contains a view name nor a " +
					"View object in servlet with name '" + getServletName() + "'");
		}
	}

	// Delegate to the View object for rendering.
	if (logger.isDebugEnabled()) {
		logger.debug("Rendering view [" + view + "] in DispatcherServlet with name '" + getServletName() + "'");
	}
	try {
		view.render(mv.getModelInternal(), request, response);
	}
	catch (Exception ex) {
		if (logger.isDebugEnabled()) {
			logger.debug("Error rendering view [" + view + "] in DispatcherServlet with name '" +
					getServletName() + "'", ex);
		}
		throw ex;
	}
}
```

`mv.isReference()`判断`ModelAndView`中的`view`属于`String`类型的时候，对视图进行处理

```
protected View resolveViewName(String viewName, Map<String, Object> model, Locale locale,
		HttpServletRequest request) throws Exception {

	for (ViewResolver viewResolver : this.viewResolvers) {
		View view = viewResolver.resolveViewName(viewName, locale);
		if (view != null) {
			return view;
		}
	}
	return null;
}
```

这里的视图解析器`viewResolver`来自于我们在配置文件中注入

```
 <!-- 3.配置jsp 显示ViewResolver -->
 <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
 	<property name="viewClass" value="org.springframework.web.servlet.view.JstlView" />
 	<property name="prefix" value="/" />
 	<property name="suffix" value=".jsp" />
 </bean>
```

这里调用的`InternalResourceViewResolver`对视图名进行解析处理

```
@Override
public View resolveViewName(String viewName, Locale locale) throws Exception {
	if (!isCache()) {
		return createView(viewName, locale);
	}
	else {
		Object cacheKey = getCacheKey(viewName, locale);
		View view = this.viewAccessCache.get(cacheKey);
		if (view == null) {
			synchronized (this.viewCreationCache) {
				view = this.viewCreationCache.get(cacheKey);
				if (view == null) {
					// Ask the subclass to create the View object.
					view = createView(viewName, locale);
					if (view == null && this.cacheUnresolved) {
						view = UNRESOLVED_VIEW;
					}
					if (view != null) {
						this.viewAccessCache.put(cacheKey, view);
						this.viewCreationCache.put(cacheKey, view);
						if (logger.isTraceEnabled()) {
							logger.trace("Cached view [" + cacheKey + "]");
						}
					}
				}
			}
		}
		return (view != UNRESOLVED_VIEW ? view : null);
	}
}
```

前面是对于缓存的处理，然后是`createView(viewName, locale)`创建视图对象`View`

```
@Override
protected View createView(String viewName, Locale locale) throws Exception {
	// If this resolver is not supposed to handle the given view,
	// return null to pass on to the next resolver in the chain.
	if (!canHandle(viewName, locale)) {
		return null;
	}
	// Check for special "redirect:" prefix.
	if (viewName.startsWith(REDIRECT_URL_PREFIX)) {
		String redirectUrl = viewName.substring(REDIRECT_URL_PREFIX.length());
		RedirectView view = new RedirectView(redirectUrl, isRedirectContextRelative(), isRedirectHttp10Compatible());
		return applyLifecycleMethods(viewName, view);
	}
	// Check for special "forward:" prefix.
	if (viewName.startsWith(FORWARD_URL_PREFIX)) {
		String forwardUrl = viewName.substring(FORWARD_URL_PREFIX.length());
		return new InternalResourceView(forwardUrl);
	}
	// Else fall back to superclass implementation: calling loadView.
	return super.createView(viewName, locale);
}
```

先看`canHandle(viewName, locale)`

```
protected boolean canHandle(String viewName, Locale locale) {
	String[] viewNames = getViewNames();
	return (viewNames == null || PatternMatchUtils.simpleMatch(viewNames, viewName));
}
```

字符串数组`viewNames`也是在配置`InternalResourceViewResolver`的时候配置好的，根据这个决定是否处理这个视图

然后可以看到这里根据视图前缀创建不同类型的视图对象`View`，分别是以`redirect:`开头的重定向`RedirectView`，以`forward:`开头的内部转发`InternalResourceView`，和第三种没有前缀的如下，调用了父类的`createView`方法


```
protected View createView(String viewName, Locale locale) throws Exception {
	return loadView(viewName, locale);
}


@Override
protected View loadView(String viewName, Locale locale) throws Exception {
	AbstractUrlBasedView view = buildView(viewName);
	View result = applyLifecycleMethods(viewName, view);
	return (view.checkResource(locale) ? result : null);
}

@Override
protected AbstractUrlBasedView buildView(String viewName) throws Exception {
	InternalResourceView view = (InternalResourceView) super.buildView(viewName);
	if (this.alwaysInclude != null) {
		view.setAlwaysInclude(this.alwaysInclude);
	}
	view.setPreventDispatchLoop(true);
	return view;
}

protected AbstractUrlBasedView buildView(String viewName) throws Exception {
	AbstractUrlBasedView view = (AbstractUrlBasedView) BeanUtils.instantiateClass(getViewClass());
	view.setUrl(getPrefix() + viewName + getSuffix());

	String contentType = getContentType();
	if (contentType != null) {
		view.setContentType(contentType);
	}

	view.setRequestContextAttribute(getRequestContextAttribute());
	view.setAttributesMap(getAttributesMap());
	//决定是否将PathVariable合并到Model中
	Boolean exposePathVariables = getExposePathVariables();
	if (exposePathVariables != null) {
		view.setExposePathVariables(exposePathVariables);
	}
	Boolean exposeContextBeansAsAttributes = getExposeContextBeansAsAttributes();
	if (exposeContextBeansAsAttributes != null) {
		view.setExposeContextBeansAsAttributes(exposeContextBeansAsAttributes);
	}
	String[] exposedContextBeanNames = getExposedContextBeanNames();
	if (exposedContextBeanNames != null) {
		view.setExposedContextBeanNames(exposedContextBeanNames);
	}

	return view;
}
```

`getViewClass()`获取配置文件中`InternalResourceViewResolver`中的`viewClass`，这里我们配置的是`JstlView`，这里用反射的方式实例化，`getPrefix()`和`getSuffix()`对应我们配置的前后缀，来拼凑成有效的`url`，后面的都是读取`InternalResourceViewResolver`中配置的属性来初始化`JstlView`

此外我们看到创建好视图后，还对视图进行了一些处理`applyLifecycleMethods(viewName, view)`

```
private View applyLifecycleMethods(String viewName, AbstractView view) {
	return (View) getApplicationContext().getAutowireCapableBeanFactory().initializeBean(view, viewName);
}
```

因为`JstlView`实现了`BeanNameAware`接口，所以这里将`viewName`设置到`view`的`BeanName`属性中，然后还将`applicationContext`注入到`View`中，还没看`Spring`，这里跳过

回到`DispatchServlet`的`render`方法中，创建好视图对象之后，调用视图对象的`render`方法渲染视图

```
@Override
public void render(Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
	if (logger.isTraceEnabled()) {
		logger.trace("Rendering view with name '" + this.beanName + "' with model " + model +
			" and static attributes " + this.staticAttributes);
	}

	Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
	prepareResponse(request, response);
	renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
}
```

这里创建了一个`Map`类型的对象

```
protected Map<String, Object> createMergedOutputModel(Map<String, ?> model, HttpServletRequest request,
		HttpServletResponse response) {

	@SuppressWarnings("unchecked")
	Map<String, Object> pathVars = (this.exposePathVariables ?
			(Map<String, Object>) request.getAttribute(View.PATH_VARIABLES) : null);

	// Consolidate static and dynamic model attributes.
	int size = this.staticAttributes.size();
	size += (model != null ? model.size() : 0);
	size += (pathVars != null ? pathVars.size() : 0);

	Map<String, Object> mergedModel = new LinkedHashMap<String, Object>(size);
	mergedModel.putAll(this.staticAttributes);
	if (pathVars != null) {
		mergedModel.putAll(pathVars);
	}
	if (model != null) {
		mergedModel.putAll(model);
	}

	// Expose RequestContext?
	if (this.requestContextAttribute != null) {
		mergedModel.put(this.requestContextAttribute, createRequestContext(request, response, mergedModel));
	}

	return mergedModel;
}
```

`exposePathVariables`变量是我们在初始化`JstlView`的注入的，来源于`InternalResourceViewResolver`，默认为`true`，用来获取`url`中{}属性和值

这里合并`ModelAndView`中的`ModelMap`和`pathVars`，然后返回

然后使用合并之后的`ModelMap`继续渲染视图

```
@Override
protected void renderMergedOutputModel(
		Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {

	// Expose the model object as request attributes.
	exposeModelAsRequestAttributes(model, request);

	// Expose helpers as request attributes, if any.
	exposeHelpers(request);

	// Determine the path for the request dispatcher.
	String dispatcherPath = prepareForRendering(request, response);

	// Obtain a RequestDispatcher for the target resource (typically a JSP).
	RequestDispatcher rd = getRequestDispatcher(request, dispatcherPath);
	if (rd == null) {
		throw new ServletException("Could not get RequestDispatcher for [" + getUrl() +
				"]: Check that the corresponding file exists within your web application archive!");
	}

	// If already included or response already committed, perform include, else forward.
	if (useInclude(request, response)) {
		response.setContentType(getContentType());
		if (logger.isDebugEnabled()) {
			logger.debug("Including resource [" + getUrl() + "] in InternalResourceView '" + getBeanName() + "'");
		}
		rd.include(request, response);
	}

	else {
		// Note: The forwarded resource is supposed to determine the content type itself.
		if (logger.isDebugEnabled()) {
			logger.debug("Forwarding to resource [" + getUrl() + "] in InternalResourceView '" + getBeanName() + "'");
		}
		rd.forward(request, response);
	}
}
```

先看`exposeModelAsRequestAttributes(model, request);`

```
protected void exposeModelAsRequestAttributes(Map<String, Object> model, HttpServletRequest request) throws Exception {
	for (Map.Entry<String, Object> entry : model.entrySet()) {
		String modelName = entry.getKey();
		Object modelValue = entry.getValue();
		if (modelValue != null) {
			request.setAttribute(modelName, modelValue);
			if (logger.isDebugEnabled()) {
				logger.debug("Added model object '" + modelName + "' of type [" + modelValue.getClass().getName() +
						"] to request in view with name '" + getBeanName() + "'");
			}
		}
		else {
			request.removeAttribute(modelName);
			if (logger.isDebugEnabled()) {
				logger.debug("Removed model object '" + modelName +
						"' from request in view with name '" + getBeanName() + "'");
			}
		}
	}
}
```

这里将合并之后的`ModelMap`中不为空的属性全部放入`request`中，所以我们能在`jsp`中获取到我们在`Controller`给`Map`入参中添加的属性

`preventDispatchLoop`属性在创建`JstlView`的时候设置为了`true`

然后看`prepareForRendering(request, response)`

```
protected String prepareForRendering(HttpServletRequest request, HttpServletResponse response)
		throws Exception {

	String path = getUrl();
	if (this.preventDispatchLoop) {
		String uri = request.getRequestURI();
		if (path.startsWith("/") ? uri.equals(path) : uri.equals(StringUtils.applyRelativePath(uri, path))) {
			throw new ServletException("Circular view path [" + path + "]: would dispatch back " +
					"to the current handler URL [" + uri + "] again. Check your ViewResolver setup! " +
					"(Hint: This may be the result of an unspecified view, due to default view name generation.)");
		}
	}
	return path;
}
```

`getUrl`返回我们创建视图传入的目的资源的`url`

`request.getRequestURI`返回的结果是`/`+项目名称+剩余`uri`

这里要防止当前`url`和跳转后的`url`相同变成死循环，检测到相同时抛出异常

注意到`StringUtils.applyRelativePath(uri, path))`，应该是和拼接最终`url`相关

```
public static String applyRelativePath(String path, String relativePath) {
	int separatorIndex = path.lastIndexOf(FOLDER_SEPARATOR);
	if (separatorIndex != -1) {
		String newPath = path.substring(0, separatorIndex);
		if (!relativePath.startsWith(FOLDER_SEPARATOR)) {
			newPath += FOLDER_SEPARATOR;
		}
		return newPath + relativePath;
	}
	else {
		return relativePath;
	}
}
```

拼接路径的方式为原本请求路径中去掉最后一个`/`和后面的字符，当第二个值不存在`/`，也就是`url`中项目名和后面的字符全为空，这里则直接返回创建好的视图的`url`，否则去掉最后一个`/`和后面的字符，然后拼接上我们视图的`url`

回到

```
@Override
protected void renderMergedOutputModel(
		Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {

	// Expose the model object as request attributes.
	exposeModelAsRequestAttributes(model, request);

	// Expose helpers as request attributes, if any.
	exposeHelpers(request);

	// Determine the path for the request dispatcher.
	String dispatcherPath = prepareForRendering(request, response);

	// Obtain a RequestDispatcher for the target resource (typically a JSP).
	RequestDispatcher rd = getRequestDispatcher(request, dispatcherPath);
	if (rd == null) {
		throw new ServletException("Could not get RequestDispatcher for [" + getUrl() +
				"]: Check that the corresponding file exists within your web application archive!");
	}

	// If already included or response already committed, perform include, else forward.
	if (useInclude(request, response)) {
		response.setContentType(getContentType());
		if (logger.isDebugEnabled()) {
			logger.debug("Including resource [" + getUrl() + "] in InternalResourceView '" + getBeanName() + "'");
		}
		rd.include(request, response);
	}

	else {
		// Note: The forwarded resource is supposed to determine the content type itself.
		if (logger.isDebugEnabled()) {
			logger.debug("Forwarding to resource [" + getUrl() + "] in InternalResourceView '" + getBeanName() + "'");
		}
		rd.forward(request, response);
	}
}
```

剩下最后一个判断`useInclude(request, response)`，查了下，`include`的作用应该是属于内嵌的一种，就是将另一个servlet/jsp处理过后的内容拿过来与本身的servlet的内容一同输出

### 总结

1. 当`ModelAndView`的`View`为`null`时，取默认视图名，该视图名与当前`url`有关

2. 获取配置好的`InternalResourceViewResolver`，用来解析处理`ModelAndView`中的`View`(Object类型)

3. 根据`View`前缀决定创建的`View`视图对象的类型

4. 将`@PathVariable`的属性和`ModelAndView`的`Model`合并

5. 不是重定向时，将合并后的`Model`的值放入`request`中，然后跳转

