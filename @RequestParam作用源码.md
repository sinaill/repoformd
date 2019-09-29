title: @RequestParam作用源码
categories: 框架
tags: 
	- SpringMVC

---

### @RequestParam作用

在`Controller`方法的入参使用`@RequestParam`注解可以获取`Request`请求中的参数

### 原理

#### 普通变量

依旧是查看解析`@RequestParam`注解的`RequestParamMethodArgumentResolver`

`supportsParameter`方法如下

```
	@Override
public boolean supportsParameter(MethodParameter parameter) {
	Class<?> paramType = parameter.getParameterType();
	if (parameter.hasParameterAnnotation(RequestParam.class)) {
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

再看解析参数的`resolveArgument`方法

```
@Override
public final Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

	Class<?> paramType = parameter.getParameterType();
	NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);
	//取出请求中对应的属性arg
	Object arg = resolveName(namedValueInfo.name, parameter, webRequest);
	//请求中没有对应属性，arg为null
	if (arg == null) {
		//除了设置`defaultValue`，其他都会抛出异常
		if (namedValueInfo.defaultValue != null) {
			arg = resolveDefaultValue(namedValueInfo.defaultValue);
		}
		else if (namedValueInfo.required && !parameter.getParameterType().getName().equals("java.util.Optional")) {
			handleMissingValue(namedValueInfo.name, parameter);
		}
		arg = handleNullValue(namedValueInfo.name, arg, paramType);
	}
	//arg为""时并且设置了defaultValue时，arg接收处理后的defaultValue
	else if ("".equals(arg) && namedValueInfo.defaultValue != null) {
		arg = resolveDefaultValue(namedValueInfo.defaultValue);
	}

	if (binderFactory != null) {
		WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
		arg = binder.convertIfNecessary(arg, paramType, parameter);
	}

	handleResolvedValue(arg, namedValueInfo.name, parameter, mavContainer, webRequest);

	return arg;
}
```

`NamedValueInfo`是用来存放`@RequestParam`注解中参数的类，包括`value`，`required`，`defaultValue`

```
private NamedValueInfo getNamedValueInfo(MethodParameter parameter) {
	NamedValueInfo namedValueInfo = this.namedValueInfoCache.get(parameter);
	if (namedValueInfo == null) {
		namedValueInfo = createNamedValueInfo(parameter);
		namedValueInfo = updateNamedValueInfo(parameter, namedValueInfo);
		this.namedValueInfoCache.put(parameter, namedValueInfo);
	}
	return namedValueInfo;
}


@Override
protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
	RequestParam ann = parameter.getParameterAnnotation(RequestParam.class);
	//不存在`@requestParam`时，用无参构造方法
	return (ann != null ? new RequestParamNamedValueInfo(ann) : new RequestParamNamedValueInfo());
}

//构造方法
	public RequestParamNamedValueInfo() {
		super("", false, ValueConstants.DEFAULT_NONE);
	}

	public RequestParamNamedValueInfo(RequestParam annotation) {
		super(annotation.value(), annotation.required(), annotation.defaultValue());
	}
```

然后要对`NameValueInfo`进行update，处理当没有`@RequestParam`注解或者`@RequestParam`注解没有设置`value`值的导致`NameValueInfo`的`name`属性为`""`的情况

```
private NamedValueInfo updateNamedValueInfo(MethodParameter parameter, NamedValueInfo info) {
	String name = info.name;
	if (info.name.length() == 0) {
		name = parameter.getParameterName();
		if (name == null) {
			throw new IllegalArgumentException("Name for argument type [" + parameter.getParameterType().getName() +
					"] not available, and parameter name information not found in class file either.");
		}
	}
	String defaultValue = (ValueConstants.DEFAULT_NONE.equals(info.defaultValue) ? null : info.defaultValue);
	return new NamedValueInfo(name, info.required, defaultValue);
}
```
当`name`长度为0，取入参的变量名，作为`name`，如果没有设置`defaultValue`，这里则将`defaultValue`设置为`ValueConstants.DEFAULT_NONE`

然后看`resloveName`方法

```
@Override
protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest webRequest) throws Exception {
	HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
	MultipartHttpServletRequest multipartRequest =
			WebUtils.getNativeRequest(servletRequest, MultipartHttpServletRequest.class);
	Object arg;

	if (MultipartFile.class.equals(parameter.getParameterType())) {
		assertIsMultipartRequest(servletRequest);
		Assert.notNull(multipartRequest, "Expected MultipartHttpServletRequest: is a MultipartResolver configured?");
		arg = multipartRequest.getFile(name);
	}
	else if (isMultipartFileCollection(parameter)) {
		assertIsMultipartRequest(servletRequest);
		Assert.notNull(multipartRequest, "Expected MultipartHttpServletRequest: is a MultipartResolver configured?");
		arg = multipartRequest.getFiles(name);
	}
	else if (isMultipartFileArray(parameter)) {
		assertIsMultipartRequest(servletRequest);
		Assert.notNull(multipartRequest, "Expected MultipartHttpServletRequest: is a MultipartResolver configured?");
		List<MultipartFile> multipartFiles = multipartRequest.getFiles(name);
		arg = multipartFiles.toArray(new MultipartFile[multipartFiles.size()]);
	}
	else if ("javax.servlet.http.Part".equals(parameter.getParameterType().getName())) {
		assertIsMultipartRequest(servletRequest);
		arg = servletRequest.getPart(name);
	}
	else if (isPartCollection(parameter)) {
		assertIsMultipartRequest(servletRequest);
		arg = new ArrayList<Object>(servletRequest.getParts());
	}
	else if (isPartArray(parameter)) {
		assertIsMultipartRequest(servletRequest);
		arg = RequestPartResolver.resolvePart(servletRequest);
	}
	else {
		arg = null;
		if (multipartRequest != null) {
			List<MultipartFile> files = multipartRequest.getFiles(name);
			if (!files.isEmpty()) {
				arg = (files.size() == 1 ? files.get(0) : files);
			}
		}
		if (arg == null) {
			String[] paramValues = webRequest.getParameterValues(name);
			if (paramValues != null) {
				arg = (paramValues.length == 1 ? paramValues[0] : paramValues);
			}
		}
	}

	return arg;
}
```

看最下面，以`name`从`request`请求中取出对应属性，然后返回给`resolveArgument`方法，经`dataBinder`转换为入参类型之后返回，然后作为反射调用`Controller`方法的参数

#### java.util.Map

入参为`java.util.Map`类型，且前面有`@ReqeustParam`注解并且`value`为空时，作用在入参上的参数解析器为`RequestParamMapMethodArgumentResolver`

```
@Override
public boolean supportsParameter(MethodParameter parameter) {
	RequestParam ann = parameter.getParameterAnnotation(RequestParam.class);
	if (ann != null) {
		if (Map.class.isAssignableFrom(parameter.getParameterType())) {
			return !StringUtils.hasText(ann.value());
		}
	}
	return false;
}

@Override
public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

	Class<?> paramType = parameter.getParameterType();

	Map<String, String[]> parameterMap = webRequest.getParameterMap();
	if (MultiValueMap.class.isAssignableFrom(paramType)) {
		MultiValueMap<String, String> result = new LinkedMultiValueMap<String, String>(parameterMap.size());
		for (Map.Entry<String, String[]> entry : parameterMap.entrySet()) {
			for (String value : entry.getValue()) {
				result.add(entry.getKey(), value);
			}
		}
		return result;
	}
	else {
		Map<String, String> result = new LinkedHashMap<String, String>(parameterMap.size());
		for (Map.Entry<String, String[]> entry : parameterMap.entrySet()) {
			if (entry.getValue().length > 0) {
				result.put(entry.getKey(), entry.getValue()[0]);
			}
		}
		return result;
	}
}
```

获取请求中参数时是用的`getParameterMap`方法以参数名为key和参数值为value的方式，获取一个`Map`类型的入参，将它返回作为反射调用该`Controller`方法的参数
