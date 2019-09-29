title: SpringMVC配置拦截器
categories: 框架
tags: 
	- SpringMVC

---

### 拦截器

Java 里的拦截器是动态拦截action调用的对象。它提供了一种机制可以使开发者可以定义在一个action执行的前后执行的代码，也可以在一个action执行前阻止其执行，同时也提供了一种可以提取action中可重用部分的方式。

### HandlerInterceptorAdapter类

它有以下三个方法

```
	//handler中方法被执行前被调用
	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
		throws Exception {
		return true;
	}

	//handler中方法被执行后被调用
	public void postHandle(
			HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)
			throws Exception {
	}

	//handler方法被调用，渲染视图之后被调用
	@Override
	public void afterCompletion(
			HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
			throws Exception {
	}
```

- `preHandle(HttpServletRequest request, HttpServletResponse response, Object handle)`方法，该方法在请求处理之前进行调用。`SpringMVC`中的`Interceptor`是链式调用的，在一个应用中或者说是在一个请求中可以同时存在多个`Interceptor`。每个`Interceptor`的调用会依据它的声明顺序依次执行，而且最先执行的都是`Interceptor`中的`preHandle`方法，所以可以在这个方法中进行一些前置初始化操作或者是对当前请求做一个预处理，也可以在这个方法中进行一些判断来决定请求是否要继续进行下去。该方法的返回值是布尔（Boolean）类型的，当它返回为`false`时，表示请求结束，后续的`Interceptor`和控制器（Controller）都不会再执行；当返回值为`true`时，就会继续调用下一个`Interceptor`的`preHandle`方法，如果已经是最后一个Interceptor的时候，就会是调用当前请求的控制器中的方法。
- `postHandle(HttpServletRequest request, HttpServletResponse response, Object handle, ModelAndView modelAndView)`方法，通过`preHandle`方法的解释，我们知道这个方法包括后面要说到的`afterCompletion`方法都只能在当前所属的`Interceptor`的`preHandle`方法的返回值为`true`的时候，才能被调用。`postHandle`方法在当前请求进行处理之后，也就是在控制器中的方法调用之后执行，但是它会在`DispatcherServlet`进行视图返回渲染之前被调用，所以我们可以在这个方法中对控制器处理之后的`ModelAndView`对象进行操作。`postHandle`方法被调用的方向跟`preHandle`是相反的，也就是说，先声明的`Interceptor`的`postHandle`方法反而会后执行。
- afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handle, Exception ex)方法，也是需要当前对应的Interceptor的preHandle方法的返回值为true时才会执行。因此，该方法将在整个请求结束之后，也就是在DispatcherServlet渲染了对应的视图之后执行，这个方法的主要作用是用于进行资源清理的工作。

### 配置方法

```
 <mvc:interceptors>
 	<mvc:interceptor>
 		<mvc:mapping path="/**"/>
 		<bean class="crj.ssm.intercepter.MyIntercepter"></bean>
 	</mvc:interceptor>
 </mvc:interceptors>
```

注意匹配所有不是使用/，而是/**，/代表服务器根目录，/*代表一级目录，两个星号代表多级目录





