title: 注解@ModelAttribute和@RequestParam加与不加
categories: 框架
tags: 
	- SpringMVC

---

我们知道`@ModelAttribute`注解和`@RequestParam`注解加与不加都起作用，其中`@ModelAttribute`是以类名作为`value`，`@RequestParam`是以变量名作为`value`

还是看参数解析器集合的初始化

```
private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
	List<HandlerMethodArgumentResolver> resolvers = new ArrayList<HandlerMethodArgumentResolver>();

	// Annotation-based argument resolution
	resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
	resolvers.add(new RequestParamMapMethodArgumentResolver());
	resolvers.add(new PathVariableMethodArgumentResolver());
	resolvers.add(new PathVariableMapMethodArgumentResolver());
	resolvers.add(new MatrixVariableMethodArgumentResolver());
	resolvers.add(new MatrixVariableMapMethodArgumentResolver());
	resolvers.add(new ServletModelAttributeMethodProcessor(false));
	resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters()));
	resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters()));
	resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
	resolvers.add(new RequestHeaderMapMethodArgumentResolver());
	resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
	resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));

	// Type-based argument resolution
	resolvers.add(new ServletRequestMethodArgumentResolver());
	resolvers.add(new ServletResponseMethodArgumentResolver());
	resolvers.add(new HttpEntityMethodProcessor(getMessageConverters()));
	resolvers.add(new RedirectAttributesMethodArgumentResolver());
	resolvers.add(new ModelMethodProcessor());
	resolvers.add(new MapMethodProcessor());
	resolvers.add(new ErrorsMethodArgumentResolver());
	resolvers.add(new SessionStatusMethodArgumentResolver());
	resolvers.add(new UriComponentsBuilderMethodArgumentResolver());

	// Custom arguments
	if (getCustomArgumentResolvers() != null) {
		resolvers.addAll(getCustomArgumentResolvers());
	}

	// Catch-all
	resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
	resolvers.add(new ServletModelAttributeMethodProcessor(true));

	return resolvers;
}
```

最下面还有注释`Catch-all`

初始化了相同的`RequestParamMethodArgumentResolver`和`ServletModelAttributeMethodProcessor`，不同之处在于传入的`Boolean`值不同

我们看两个的`support`方法

```
@Override
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

两个变量`annotationNotRequired`和`useDefaultResolution`，当为`true`时，不需要两个注解也能当成有注解的情况处理，当然是要排在前面的参数解析器都不适用当前参数的情况