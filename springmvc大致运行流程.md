title: SpringMVC大致运行流程
categories: 框架
tags: 
	- SpringMVC

---

### HandlerMapping

url和controller的映射方式


![HandlerMapping](http://wx3.sinaimg.cn/large/96b7c0f4ly1g2blsrig3yj20bv04174a.jpg)


默认加载的三种HandlerMapping():

- RequestMappingHandlerMapping:针对注解配置@RequestMapping
- BeanNameUrlHandlerMapping：通过对比url和bean的name找到对应的对象，[使用例子](https://www.tutorialspoint.com/springmvc/springmvc_beannameurlhandlermapping.htm)
- SimpleUrlHandlerMapping 也是直接配置url和对应bean，[使用例子](https://www.tutorialspoint.com/springmvc/springmvc_simpleurlhandlermapping.htm)

### HandlerAdapter

根据上面三种配置controller的方式决定HandlerAdapter，也就是调用controller目标方法的方式

![HandlerAdapter](http://wx2.sinaimg.cn/large/96b7c0f4ly1g2bm2cj768j20dv042aa3.jpg)

默认加载的三种HandlerAdapter

- RequestMappingHandlerApapter:HandlerExecutionChain中的handler实际类型为`HandlerMethod`
- HttpRequestHandlerAdapter：HandlerExecutionChain中的handler实际类型为`HttpRequestHandler`
- SimpleControllerHandlerAdapter:HandlerExecutionChain中的handler实际类型为`Controller`

`HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());`

HandlerAdapter由HandlerMapping中获取到的HandlerExecutionChain中的`private final Object handler`的实际类型决定

配置controller方式->获取对应HandlerMapping->得到HandlerExecutionChain(HandlerMapping决定了HandlerExecutionChain中handler实际变量类型)->根据HandlerExecutionChain中handler实际变量类型获取对应HandlerAdapter->执行HandlerAdapter中的handler方法

`mv = ha.handle(processedRequest, response, mappedHandler.getHandler());`

适配器的作用

```
	//RequestMappingHandlerApapter
	@Override
	public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return handleInternal(request, response, (HandlerMethod) handler);
	}

	//HttpRequestHandlerAdapter
	@Override
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		((HttpRequestHandler) handler).handleRequest(request, response);
		return null;
	}

	//SimpleControllerHandlerAdapter
	@Override
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {

		return ((Controller) handler).handleRequest(request, response);
	}
```

### HandlerExecutionChain



### 流程

![springmvc流程](http://wx4.sinaimg.cn/large/96b7c0f4ly1g2cfxsdnhfj20w20q878x.jpg)

1. 得到最高优先级HandlerMapping和由它获得的HandlerExecutionChain

```
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		for (HandlerMapping hm : this.handlerMappings) {
			if (logger.isTraceEnabled()) {
				logger.trace(
						"Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
			}
			//handler
			HandlerExecutionChain handler = hm.getHandler(request);
			
			if (handler != null) {
				return handler;
			}
		}
		return null;
	}
```

2. HandlerExecutionChain中handler变量实际类型决定HandlerAdatper

```
	protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		for (HandlerAdapter ha : this.handlerAdapters) {
			if (logger.isTraceEnabled()) {
				logger.trace("Testing handler adapter [" + ha + "]");
			}
			if (ha.supports(handler)) {
				return ha;
			}
		}
		throw new ServletException("No adapter for handler [" + handler +
				"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
	}
```

3. 调用拦截器PreHandler方法

```
	if (!mappedHandler.applyPreHandle(processedRequest, response)) {
		return;
	}
```

4. 适配器调用controller目标方法返回ModelAndView对象

```
	mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
```

5. 调用拦截器的postHandle方法

```
	mappedHandler.applyPostHandle(processedRequest, response, mv);

	//倒序调用
	void applyPostHandle(HttpServletRequest request, HttpServletResponse response, ModelAndView mv) throws Exception {
		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			for (int i = interceptors.length - 1; i >= 0; i--) {
				HandlerInterceptor interceptor = interceptors[i];
				interceptor.postHandle(request, response, this.handler, mv);
			}
		}
	}
```

6. 渲染视图

```
	protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
		// Determine locale for request and apply it to the response.
		Locale locale = this.localeResolver.resolveLocale(request);
		response.setLocale(locale);

		View view;
		if (mv.isReference()) {
			// 视图处理器处理得到视图
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
			//开始渲染视图
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

7. 调用拦截器的afterCompletion方法

```
	void triggerAfterCompletion(HttpServletRequest request, HttpServletResponse response, Exception ex)
			throws Exception {

		HandlerInterceptor[] interceptors = getInterceptors();
		if (!ObjectUtils.isEmpty(interceptors)) {
			for (int i = this.interceptorIndex; i >= 0; i--) {
				HandlerInterceptor interceptor = interceptors[i];
				try {
					interceptor.afterCompletion(request, response, this.handler, ex);
				}
				catch (Throwable ex2) {
					logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
				}
			}
		}
	}
```

