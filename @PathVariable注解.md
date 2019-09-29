title: @PathVariable注解
categories: 框架
tags: 
	- SpringMVC

---

### @PathVariable

通过 @PathVariable 可以将 URL 中占位符参数绑定到控制器处理方法的入参中：URL 中的 {xxx} 占位符可以通过@PathVariable("xxx") 绑定到操作方法的入参中

### 源码

#### 注解java.util.Map类型

依旧是查看处理`@PathVariable`注解的入参的参数解析处理器`PathVariableMapMethodArgumentResolver`

```
@Override
public boolean supportsParameter(MethodParameter parameter) {
	PathVariable ann = parameter.getParameterAnnotation(PathVariable.class);
	return (ann != null && (Map.class.isAssignableFrom(parameter.getParameterType()))
			&& !StringUtils.hasText(ann.value()));
}


@Override
public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

	@SuppressWarnings("unchecked")
	Map<String, String> uriTemplateVars =
			(Map<String, String>) webRequest.getAttribute(
					HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);

	if (!CollectionUtils.isEmpty(uriTemplateVars)) {
		return new LinkedHashMap<String, String>(uriTemplateVars);
	}
	else {
		return Collections.emptyMap();
	}
}
```

适用于`@PathVariable`注解的`java.util.Map`类型的入参，且`@PathVariable`注解的`value`为空

`resolveArgument`方法中，把使用PathVariable方法传参的属性名和值获取到`java.util.Map<String,Object>`中然后返回，作为反射调用`Controller`方法的入参，这个适用于使用一个`java.util.Map`变量接收请求中多个PathVariable变量

#### 注解其他变量

注解入参不是`java.util.Map`类型时，使用的参数解析器为`PathVariableMethodArgumentResolver`

```
@Override
public boolean supportsParameter(MethodParameter parameter) {
	if (!parameter.hasParameterAnnotation(PathVariable.class)) {
		return false;
	}
	if (Map.class.isAssignableFrom(parameter.getParameterType())) {
		String paramName = parameter.getParameterAnnotation(PathVariable.class).value();
		return StringUtils.hasText(paramName);
	}
	return true;
}

```

`PathVariableMethodArgumentResolver`和`RequestParamMethodArgumentResolver`继承了`AbstractNamedValueMethodArgumentResolver`

```
@Override
public final Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

	Class<?> paramType = parameter.getParameterType();
	NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);

	Object arg = resolveName(namedValueInfo.name, parameter, webRequest);
	if (arg == null) {
		if (namedValueInfo.defaultValue != null) {
			arg = resolveDefaultValue(namedValueInfo.defaultValue);
		}
		else if (namedValueInfo.required && !parameter.getParameterType().getName().equals("java.util.Optional")) {
			handleMissingValue(namedValueInfo.name, parameter);
		}
		arg = handleNullValue(namedValueInfo.name, arg, paramType);
	}
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

`getNamedValueInfo`获取存放`@PathVariable`注解的`value`、`required`、`default`

`resolveName`根据注解的`value`从请求中获取对应的值

```
@Override
@SuppressWarnings("unchecked")
protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
	Map<String, String> uriTemplateVars =
		(Map<String, String>) request.getAttribute(
				HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);
	return (uriTemplateVars != null) ? uriTemplateVars.get(name) : null;
}
```

一样先从请求中获取存放了PathVariable类型的变量名和值的`java.util.Map`，然后以传入的`value`为key取出对应值返回，作为反射调用对应`Controller`方法的入参，也就顺利传给了我们注解了`@PathVariable`变量

#### 关于获取PathVariable变量

```
@Override
protected void handleMatch(RequestMappingInfo info, String lookupPath, HttpServletRequest request) {
	super.handleMatch(info, lookupPath, request);

	String bestPattern;
	Map<String, String> uriVariables;
	Map<String, String> decodedUriVariables;

	Set<String> patterns = info.getPatternsCondition().getPatterns();
	if (patterns.isEmpty()) {
		bestPattern = lookupPath;
		uriVariables = Collections.emptyMap();
		decodedUriVariables = Collections.emptyMap();
	}
	else {
		bestPattern = patterns.iterator().next();
		uriVariables = getPathMatcher().extractUriTemplateVariables(bestPattern, lookupPath);
		decodedUriVariables = getUrlPathHelper().decodePathVariables(request, uriVariables);
	}

	request.setAttribute(BEST_MATCHING_PATTERN_ATTRIBUTE, bestPattern);
	request.setAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, decodedUriVariables);

	if (isMatrixVariableContentAvailable()) {
		Map<String, MultiValueMap<String, String>> matrixVars = extractMatrixVariables(request, uriVariables);
		request.setAttribute(HandlerMapping.MATRIX_VARIABLES_ATTRIBUTE, matrixVars);
	}

	if (!info.getProducesCondition().getProducibleMediaTypes().isEmpty()) {
		Set<MediaType> mediaTypes = info.getProducesCondition().getProducibleMediaTypes();
		request.setAttribute(PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE, mediaTypes);
	}
}
```

以上代码段属于在`RequestMappingHandlerMapping`获取`RequestMappingHandlerAdapter`的过程中，使用`url`匹配对应的`RequestMappingInfo`，匹配完之后最后对`url`解析，获取PathVariable

```
uriVariables = getPathMatcher().extractUriTemplateVariables(bestPattern, lookupPath);
```

`bestPattern`指向被匹配的`@RequestMapping`的`value`，`lookupPath`指向`ServletPath`

通过`extractUriTemplateVariables`方法获取{}中字符和对应`lookupPath`中的值

```
@Override
public Map<String, String> extractUriTemplateVariables(String pattern, String path) {
	Map<String, String> variables = new LinkedHashMap<String, String>();
	boolean result = doMatch(pattern, path, true, variables);
	Assert.state(result, "Pattern \"" + pattern + "\" is not a match for \"" + path + "\"");
	return variables;
}
```

返回的一个`java.util.Map<String,String>`类型的变量，存放了PathVariable的变量名和值

然后进行url解码

```
decodedUriVariables = getUrlPathHelper().decodePathVariables(request, uriVariables);
```

解码后将它以`HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE`为变量名调用`Requrest`的`SetAttribute`方法放入请求中



