title: SpringMvc获取HandlerExecutionChain和HandlerAdapter
categories: 框架
tags: 
	- SpringMvc

---

### 重要类

#### HandlerExecutionChain和HandlerAdapter

`HandlerExecutionChain`中有如下成员变量

```
public class HandlerExecutionChain{
	private static final Log logger = LogFactory.getLog(HandlerExecutionChain.class);

	private final Object handler;

	private HandlerInterceptor[] interceptors;

	private List<HandlerInterceptor> interceptorList;

	private int interceptorIndex = -1;
}
```

- `handler`：执行一个`HandlerMethod`类型的对象
- `interceptors`：存放了所有拦截器的数组
- `interceptorList`：存放了所有拦截器的链表


#### HandlerMethod

`HandlerMethod`是一个接口，它的成员变量如下

```
public class HandlerMethod {

	/** Logger that is available to subclasses */
	protected final Log logger = LogFactory.getLog(getClass());

	private final Object bean;

	private final BeanFactory beanFactory;

	private final Class<?> beanType;

	private final Method method;

	private final Method bridgedMethod;

	private final MethodParameter[] parameters;
}
```

- `Object`：存放了与`Handler`有关的信息，如Demo中指向了一个方法所在的类的名称
- `method`：指向本次请求的目标`Handler`对象的目标方法
- `parameters`：存放了本次请求的入参
- `beanType`：指向了对应方法所在的`Handler`对象的类对象`Class`

#### RequestMappingHandlerMapping

Demo中用了`@RequestMapping`注解的方式来匹配url，所以只看它

在Debug中，查看`RequestMappingHandlerMapping`中的成员变量，发现

- `applicationContext`：Spring容器，可以用来获取容器中创建了的对象
- `servletContext`
- `urlMap`：属于`LinkedMultiValueMap`类，key值为请求中的`url(lookupPath)`，`value`值又是一个`linkedList`，可以存放多个`@RequestMappingInfo`类型的对象，该对象主要存放了`@RequestMapping`注解中的属性值，例如`value`和`method`属性，键值间的映射方式为`url(lookupPath)`与`@RequestMapping`注解的value值相同
- `handlerMethods`：是一个`LinkedHashMap`，键key为`RequestMapping`对象，存放了`@RequestMapping`注解的属性，值value为对应方法的`HandlerMethod`对象的映射
- `mappedInterceptor`:是一个`ArrayList`，指向了配置中的所有拦截器

#### RequestMappingInfo

[MVC请求映射信息RequestMappingInfo详解](https://my.oschina.net/u/157224/blog/974072)

### 获取HandlerExecutionChain

获取HandlerExecutionChain的关键代码如下

```
	// Determine handler for the current request.
	mappedHandler = getHandler(processedRequest);
```

跟进`getHandler`方法

```
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	for (HandlerMapping hm : this.handlerMappings) {
		if (logger.isTraceEnabled()) {
			logger.trace(
					"Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
		}
		HandlerExecutionChain handler = hm.getHandler(request);
		if (handler != null) {
			return handler;
		}
	}
	return null;
}
```

其中在`DispatchServelt`中，`this.HandlerMappings`为`SpringMvc`中系统已经注入了三种`HandlerMappings`，如图

![HandlerMapping](http://wx3.sinaimg.cn/large/96b7c0f4ly1g2blsrig3yj20bv04174a.jpg)

源码中依次让三种`HandlerMapping`来执行本次代码`HandlerExecutionChain handler = hm.getHandler(request);`，然后判断返回的`handler`对象是否为空，来决定是否返回该`handler`对象，所以接着查看`getHandler`方法，三种`HandlerMapping`的`getHandler`方法都继承自`AbstractHandlerMapping`

```
@Override
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	Object handler = getHandlerInternal(request);
	if (handler == null) {
		handler = getDefaultHandler();
	}
	if (handler == null) {
		return null;
	}
	// Bean name or resolved handler?
	if (handler instanceof String) {
		String handlerName = (String) handler;
		handler = getApplicationContext().getBean(handlerName);
	}
	return getHandlerExecutionChain(handler, request);
}
```

跟入`getHandlerInternal`方法发现，`RequestHandlerMapping`对象的`getHandlerInternal`来自`AbstractHandlerMapping`的继承类`AbstractHandlerMethodMapping`，另外两种`BeanNameUrlHandlerMapping`、`SimpleUrlHandlerMapping`则来自`AbstractHandlerMapping`的另一个继承类`AbstractUrlMethodMapping`

先看`AbstractHandlerMethodMapping`

```
	@Override
	protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
		String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
		if (logger.isDebugEnabled()) {
			logger.debug("Looking up handler method for path " + lookupPath);
		}
		HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
		if (logger.isDebugEnabled()) {
			if (handlerMethod != null) {
				logger.debug("Returning handler method [" + handlerMethod + "]");
			}
			else {
				logger.debug("Did not find handler method for [" + lookupPath + "]");
			}
		}
		return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
	}
```

其中变量`lookupPath`为除掉服务器根目录后剩余的url，例如在Demo中本次请求url为"http://localhost:8080/Person?name=zhang&age=13"(pom文件中配置了服务器path为"/")，则此处`lookupPath`为"/person"

接着`HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request)`返回了一个`HandlerMethod`类型的对象

```
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
	List<Match> matches = new ArrayList<Match>();
	List<T> directPathMatches = this.urlMap.get(lookupPath);
	if (directPathMatches != null) {
		addMatchingMappings(directPathMatches, matches, request);
	}
	if (matches.isEmpty()) {
		// No choice but to go through all mappings...
		addMatchingMappings(this.handlerMethods.keySet(), matches, request);
		//这条分支在Demo中测试得，由于urlMap映射中不包括模糊匹配/*，所以directPathMatches
		//将为空，不能精确匹配到RequestMappingInfo
		//当有匹配了模糊匹配的request传进来，是直接用RequestMappingHandlerMapping
		//中的HandlerMethods集合中的keySet，也就是使用RequestMappingInfo集合进行遍历寻找
		//匹配的RquestMappingInfo
		//由这个特性得，当既有精确匹配又有模糊匹配时，优先匹配精确，然后才是模糊匹配
	}

	if (!matches.isEmpty()) {
		Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
		Collections.sort(matches, comparator);
		if (logger.isTraceEnabled()) {
			logger.trace("Found " + matches.size() + " matching mapping(s) for [" + lookupPath + "] : " + matches);
		}
		Match bestMatch = matches.get(0);
		if (matches.size() > 1) {
			Match secondBestMatch = matches.get(1);
			if (comparator.compare(bestMatch, secondBestMatch) == 0) {
				Method m1 = bestMatch.handlerMethod.getMethod();
				Method m2 = secondBestMatch.handlerMethod.getMethod();
				throw new IllegalStateException(
						"Ambiguous handler methods mapped for HTTP path '" + request.getRequestURL() + "': {" +
						m1 + ", " + m2 + "}");
			}
		}
		handleMatch(bestMatch.mapping, lookupPath, request);
		return bestMatch.handlerMethod;
	}
	else {
		return handleNoMatch(handlerMethods.keySet(), lookupPath, request);
	}
}
```

根据传入的`lookupPath`为key值从`urlMap`中获取对应的映射`linkedList`类型的对象`directPahtMatches`，链表中存放的是`RequestMappingInfo`类型的对象，存放`@RequestMapping`注解中的属性值。然后以该链表为入参进入`addMatchingMappings`方法。

这一步查找到与项目`ServletPath`匹配的`RequestMappingInfo`，由于一个`ServletPath`可以匹配多个属性不同的`RequestMappingInfo`，所以还要进一步筛选最终`RequestMappingInfo`

```
private void addMatchingMappings(Collection<T> mappings, List<Match> matches, HttpServletRequest request) {
	for (T mapping : mappings) {
		T match = getMatchingMapping(mapping, request);
		if (match != null) {
			matches.add(new Match(match, this.handlerMethods.get(mapping)));
		}
	}
}
```

传入的`mappings`是与当前`url`映射的`RequestMappingInfo`链表，遍历该链表，进入`getMatchingMapping(mapping, request)`方法

这一步就是开始遍历上一步筛选的`RequestMappingInfo`，接下来是筛选方式

```
@Override
protected RequestMappingInfo getMatchingMapping(RequestMappingInfo info, HttpServletRequest request) {
	return info.getMatchingCondition(request);
}
```

调用了传进来的`RequestMappingInfo`对象的`getMatchingCondition`方法

```
@Override
public RequestMappingInfo getMatchingCondition(HttpServletRequest request) {
	RequestMethodsRequestCondition methods = this.methodsCondition.getMatchingCondition(request);
	//存放注解中指定请求的method类型， GET、POST、PUT、DELETE等
	ParamsRequestCondition params = this.paramsCondition.getMatchingCondition(request);
	/**
	 * 限定参数，用法例如
	 * 1.params="myParam=myValue",	必须存在参数myParam,并且值为myValue.
	 * 2.params="myParam"		必须存在参数myParam.
	 * /
	//存放注解中指定的header值
	HeadersRequestCondition headers = this.headersCondition.getMatchingCondition(request);
	ConsumesRequestCondition consumes = this.consumesCondition.getMatchingCondition(request);
	//存放注解中指定处理请求的提交内容类型（Content-Type），例如application/json, text/html;
	ProducesRequestCondition produces = this.producesCondition.getMatchingCondition(request);
	//存放注解中指定的返回的内容类型，例如Accept: */*
	if (methods == null || params == null || headers == null || consumes == null || produces == null) {
		return null;
	}

	PatternsRequestCondition patterns = this.patternsCondition.getMatchingCondition(request);
	if (patterns == null) {
		return null;
	}

	RequestConditionHolder custom = this.customConditionHolder.getMatchingCondition(request);
	if (custom == null) {
		return null;
	}

	return new RequestMappingInfo(this.name, patterns,
			methods, params, headers, consumes, produces, custom.getCondition());
}
```


以上代码主要查看`RequestMappingInfo`和`request`是否匹配，是的话创建一个新的`RequestMappingInfo`然后返回，否则返回空

这一步确定最终的`RequestMappingInfo`

```
private void addMatchingMappings(Collection<T> mappings, List<Match> matches, HttpServletRequest request) {
	for (T mapping : mappings) {
		T match = getMatchingMapping(mapping, request);
		if (match != null) {
			matches.add(new Match(match, this.handlerMethods.get(mapping)));
		}
	}
}
```

变量`match`接收返回的`RequestMappingInfo`，然后给传入的`ArrayList`类型的对象matches添加元素`new Match(match, this.handlerMethods.get(mapping))`

这一步将最终确定的`RequestMappingInfo`和它相对应的`HandlerMethods`对象封装到`Match`类中

```
private class Match {

	private final T mapping;

	private final HandlerMethod handlerMethod;

	public Match(T mapping, HandlerMethod handlerMethod) {
		this.mapping = mapping;
		this.handlerMethod = handlerMethod;
	}

	@Override
	public String toString() {
		return this.mapping.toString();
	}
}
```

`Match`类为定义在`AbstractHandlerMethodMapping`的内部类，

`this.handlerMethods.get(mapping)`，`handlerMethods`为`AbstractHandlerMethodMapping`类的成员变量，定义为`private final Map<T, HandlerMethod> handlerMethods = new LinkedHashMap<T, HandlerMethod>();`，键值key为`RequestMappingInfo`类型的对象，值value为`HandlerMethod`类型的对象，从Demo中看，映射关系为`Controller`中`RequestMappingInfo`和它对应的方法，在这里根据`RequestMapping`类的变量`mapping`获取对应的`HandlerMethod`对象，创建`Match`对象后添加到`matches`中

至此，`addMappingMatchings`结束，回到`lookupHandlerMethod`方法

```
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
	List<Match> matches = new ArrayList<Match>();
	List<T> directPathMatches = this.urlMap.get(lookupPath);
	if (directPathMatches != null) {
		addMatchingMappings(directPathMatches, matches, request);
	}
	if (matches.isEmpty()) {
		// No choice but to go through all mappings...
		addMatchingMappings(this.handlerMethods.keySet(), matches, request);
	}

	if (!matches.isEmpty()) {
		Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
		Collections.sort(matches, comparator);
		if (logger.isTraceEnabled()) {
			logger.trace("Found " + matches.size() + " matching mapping(s) for [" + lookupPath + "] : " + matches);
		}
		Match bestMatch = matches.get(0);
		if (matches.size() > 1) {
			Match secondBestMatch = matches.get(1);
			if (comparator.compare(bestMatch, secondBestMatch) == 0) {
				Method m1 = bestMatch.handlerMethod.getMethod();
				Method m2 = secondBestMatch.handlerMethod.getMethod();
				throw new IllegalStateException(
						"Ambiguous handler methods mapped for HTTP path '" + request.getRequestURL() + "': {" +
						m1 + ", " + m2 + "}");
			}
		}
		handleMatch(bestMatch.mapping, lookupPath, request);
		return bestMatch.handlerMethod;
	}
	else {
		return handleNoMatch(handlerMethods.keySet(), lookupPath, request);
	}
}
```

这一步又一次对经过上一步筛选的`RequestMappingInfo`进行筛选(这一步不知道为什么还会有剩多个`RequestMappingInfo`，单纯的复制`Handler`中方法，编译期就报错)

上面一大段代码主要通过`ServetPath`->`RequestMappingInfo`->`HandlerMethod`->`March`
此时变量`mathces`中就存放了这个`Match`，选取最优`Match`，返回`HandlerMethod`对象

执行完后回到`AbstractHandlerMethodMapping`的`getHandlerInternal`方法

```
@Override
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
	String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
	if (logger.isDebugEnabled()) {
		logger.debug("Looking up handler method for path " + lookupPath);
	}
	HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
	if (logger.isDebugEnabled()) {
		if (handlerMethod != null) {
			logger.debug("Returning handler method [" + handlerMethod + "]");
		}
		else {
			logger.debug("Did not find handler method for [" + lookupPath + "]");
		}
	}
	return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
}
```

获取到和当前`Request`对应的`HandlerMethod`之后，还要执行`handlerMethod.createWithResolvedBean()`

```
public HandlerMethod createWithResolvedBean() {
	Object handler = this.bean;
	if (this.bean instanceof String) {
		String beanName = (String) this.bean;
		handler = this.beanFactory.getBean(beanName);
	}
	return new HandlerMethod(this, handler);
}
```

这里获取了`HandlerMethod`中的`bean`变量，在Demo中Debug得，`Object`类型的`bean`变量此时指向了一个字符串`String`对象，该对象存放的是`HandlerMethod`代表的方法所在的`Handler`控制器的类的名称

而`beanFactory`实际指向了`org.springframework.beans.factory.support.DefaultListableBeanFactory`，经查询，是一个与Spring IOC相关的类，这里应该就是根据取得的`Handler`类的名称，从IOC容器中取出注解@controller的`Handler`，从而将`HandlerMethod`中的`Bean`变量从指向字符串转化为指向对应的`Handler`对象

返回后，执行回到`AbstractHandlerMapping`的`getHandler`方法

```
@Override
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
	Object handler = getHandlerInternal(request);
	if (handler == null) {
		handler = getDefaultHandler();
	}
	if (handler == null) {
		return null;
	}
	// Bean name or resolved handler?
	if (handler instanceof String) {
		String handlerName = (String) handler;
		handler = getApplicationContext().getBean(handlerName);
	}
	return getHandlerExecutionChain(handler, request);
}
```

`handler`变量接收了返回的`HandlerMethod`，如果为空，获取默认`Handler`，如果还为空，就返回null。
接着又对`handler`变量作了处理，最终调用`getHandlerExecutionChain(handler, request)`来获取我们最终需要的`HandlerExecutionChain`

```
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
	HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
			(HandlerExecutionChain) handler : new HandlerExecutionChain(handler));
	chain.addInterceptors(getAdaptedInterceptors());

	String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
	for (MappedInterceptor mappedInterceptor : this.mappedInterceptors) {
		if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
			chain.addInterceptor(mappedInterceptor.getInterceptor());
		}
	}

	return chain;
}
```

第一步对传进来的`handler`变量进行判断，是否属于类`HandlerExecutionChain`，如果是的话直接返回，否则用它作为形参调用构造方法创建一个`HandlerExecution`对象。

然后给它添加拦截器，根据当前查找到的`lookupPath`，也就是`ServletPath`，将配置了与`lookupPath`相关的拦截器添加到`HandlerMethod`中，然后返回

### 获取HandlerAdapter

```
// Determine handler adapter for the current request.
HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
```
为当前请求获取一个handler adapter

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

同样，Spring Mvc也配置了三种HandlerAdapter

![](https://github.com/sinaill/pic/blob/master/handlerAdapter.PNG?raw=true)

遍历，依次检测`handler`，也就是`HandlerMethod`对象，三种`HandlerAdapter`对应了三种不同的配置方式，Demo中配置方式为@Controller和@ReqeustMapping注解实现

所以接下来查看`AbstractHandlerMethodAdapter`的`supports`方法

```
@Override
public final boolean supports(Object handler) {
	return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
}
```

判断传入的`handler`是否属于`HandlerMethod`，由于`handler`是来自`mappedHandler.getHandler()`，
所以直接返回`handler`

而`supportsInternal((HandlerMethod) handler)`

```
@Override
protected boolean supportsInternal(HandlerMethod handlerMethod) {
	return true;
}
```

所以直接返回`RequestMappingHandlerAdapter`








