title: SpringMVC拦截器运行流程源码
categories: 框架
tags: 
	- SpringMVC

---

### HandlerInterceptorAdapter

`HandlerInterceptorAdapter`是SpringMvc给我们用来实现拦截器的抽象类

```
public abstract class HandlerInterceptorAdapter implements AsyncHandlerInterceptor {

	/**
	 * This implementation always returns {@code true}.
	 */
	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
		throws Exception {
		return true;
	}

	/**
	 * This implementation is empty.
	 */
	@Override
	public void postHandle(
			HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView)
			throws Exception {
	}

	/**
	 * This implementation is empty.
	 */
	@Override
	public void afterCompletion(
			HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
			throws Exception {
	}

	/**
	 * This implementation is empty.
	 */
	@Override
	public void afterConcurrentHandlingStarted(
			HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
	}

}
```

### preHandle

SpringMvc执行拦截器`preHandle`方法

在`DispatcherServlet`中执行拦截器的`preHandle`方法，刚好在`Handler`方法被执行前执行

```
if (!mappedHandler.applyPreHandle(processedRequest, response)) {
	return;
}
```

跟入`HandlerExecutionChain`的`applyPreHandle`方法

```
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) throws Exception {
	HandlerInterceptor[] interceptors = getInterceptors();
	if (!ObjectUtils.isEmpty(interceptors)) {
		for (int i = 0; i < interceptors.length; i++) {
			HandlerInterceptor interceptor = interceptors[i];
			if (!interceptor.preHandle(request, response, this.handler)) {
				triggerAfterCompletion(request, response, null);
				return false;
			}
			this.interceptorIndex = i;
		}
	}
	return true;
}
```

`getInterceptors`方法获取`HandlerExecutionChain`中的拦截器数组`interceptors`，按照配置顺序从前到后执行拦截器的`preHandle`方法，如果有拦截器返回的结果为`false`，则进入`triggerAfterCompletion`方法

```
/**
 * Trigger afterCompletion callbacks on the mapped HandlerInterceptors.
 * Will just invoke afterCompletion for all interceptors whose preHandle invocation
 * has successfully completed and returned true.
 */
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

从英文注释中看出这个方法会在所有拦截器被成功执行并且返回`true`时触发，从`applyPreHandle`方法可以看到，当有拦截器返回了`false`时，这个方法也会被触发，然后执行所有拦截器的`afterCompletion`方法


### applyPostHandle

```
mappedHandler.applyPostHandle(processedRequest, response, mv);
```

`applyPostHandle`方法在执行完`Handler`的目标方法后被执行，所以形参中有类型为`ModelAndView`变量`mv`

```
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

注意到获取到拦截器数组`interceptors`后，是从高位往地位顺序执行拦截器的`postHandle`方法，在其中，我们可以对`Handler`返回的视图`ModelAndView`进行修改

### triggerAfterCompletion

`triggerAfterCompletion`的调用是在`processDispatchResult`方法里，在渲染完视图之后


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

执行完拦截器的`applyHandle`，处理器`Controller Handler`的方法，然后执行`postHandle`方法，再渲染视图，最后调用这个方法。

