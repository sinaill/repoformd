title: @PathVariable参与到将参数绑定到入参Bean
categories: 框架
tags: 
	- SpringMVC

---

在参数为复杂类时，处理参数的参数处理器为`ModelAttributeMethodProcessor`

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

创建要绑定到类中变量的所有参数的key和value集合mpvs，构造函数中先从请求中获取所有变量名和值，不包括PathVariable变量，然后通过`addBindValues(mpvs, request)`

```
protected void addBindValues(MutablePropertyValues mpvs, ServletRequest request) {
	String attr = HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE;
	Map<String, String> uriVars = (Map<String, String>) request.getAttribute(attr);
	if (uriVars != null) {
		for (Entry<String, String> entry : uriVars.entrySet()) {
			if (mpvs.contains(entry.getKey())) {
				logger.warn("Skipping URI variable '" + entry.getKey()
						+ "' since the request contains a bind value with the same name.");
			}
			else {
				mpvs.addPropertyValue(entry.getKey(), entry.getValue());
			}
		}
	}
}
```

首先获取了所有PathVariable变量`uriVars`

当`uriVars`的key值不与`mpvs`中的任意key相同时，将PathVariable变量也绑入mpvs中。

然后将`mpvs`绑定到对应入参`Bean`

所以也可以通过`@PathVariable`的方法将PathVariable绑入入参`Bean`中，当然参数名字和PathVariable的名字要对上