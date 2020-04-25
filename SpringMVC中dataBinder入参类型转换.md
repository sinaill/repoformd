title: SpringMVC中DataBinder对入参类型转换
categories: 框架
tags: 
	- SpringMVC

---

### DataBinder

`DataBinder`在`SpringMVC`中起的作用主要是为`Request`中参数转化为对应的入参类型

### 注册自己的PropertyEditor

由于`DataBinder`实现了`PropertyEditorRegistry`接口，所以有了注册属性编辑器的方法，我们可以手动添加我们需要的`PropertyEditor`，如下

```
@InitBinder
public void initBinder(WebDataBinder binder) {
	SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
	sdf.setLenient(false);
	binder.registerCustomEditor(Date.class, new CustomDateEditor(sdf, true));
}
```

这个方法会被收录在`WebDataBinderFactory`中，然后在创建和初始化`DataBinder`对象时，将`DataBinder`作为入参调用这个方法来注入`PropertyEditor`

### conversionservice

```
<mvc:annotation-driven />
```

这个配置默认注册了`FormattingConversionServiceFactoryBean`来创将一个`DefaultFormattingConversionService`，其中注册了许多`converter`用来进行类型转换

### 转化过程

既然是将参数转化为入参需要的类型，那应该发生在参数解析的时候，先看简单的例如`int`类型的转化

解析它的参数解析器为`RequestParamMethodArgumentResolver`


```
@Override
public final Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
		NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
	//获取这个入参的参数类型
	Class<?> paramType = parameter.getParameterType();
	//获取@RequestParam的值
	NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);
	//根据@RequestParam从request获取该参数的值
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
		//创建DataBinder
		WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
		//将从Request中获取的参数值转化为入参需要的类型paramType
		arg = binder.convertIfNecessary(arg, paramType, parameter);
	}

	handleResolvedValue(arg, namedValueInfo.name, parameter, mavContainer, webRequest);

	return arg;
}
```

转化过程发生在`arg = binder.convertIfNecessary(arg, paramType, parameter);`

先看`DataBinder`初始化

```
@Override
public final WebDataBinder createBinder(NativeWebRequest webRequest, Object target, String objectName)
		throws Exception {
	WebDataBinder dataBinder = createBinderInstance(target, objectName, webRequest);
	if (this.initializer != null) {
		//target为null的话，只将conversionservice注入到DataBinder
		//否则也一并注入到DataBinder内部变量BindingResult和它的
		//内部变量BeanWrapperImpl中
		this.initializer.initBinder(dataBinder, webRequest);
	}
	//调用@InitBinder方法
	initBinder(dataBinder, webRequest);
	return dataBinder;
}

//调用的构造函数
public DataBinder(Object target, String objectName) {
	if (target != null && target.getClass().equals(javaUtilOptionalClass)) {
		this.target = OptionalUnwrapper.unwrap(target);
	}
	else {
		this.target = target;
	}
	this.objectName = objectName;
}
```

在这里`target`为`null`

```
@Override
public <T> T convertIfNecessary(Object value, Class<T> requiredType, MethodParameter methodParam)
		throws TypeMismatchException {

	return getTypeConverter().convertIfNecessary(value, requiredType, methodParam);
}
```

`getTypeConverter()`获取转换器

```
protected TypeConverter getTypeConverter() {
	if (getTarget() != null) {
		return getInternalBindingResult().getPropertyAccessor();
	}
	else {
		return getSimpleTypeConverter();
	}
}
```

因为`target`为`null`，使用简单转换器

```
protected SimpleTypeConverter getSimpleTypeConverter() {
	if (this.typeConverter == null) {
		//创建typeConverter
		this.typeConverter = new SimpleTypeConverter();
		if (this.conversionService != null) {
			//注入conversionservice
			this.typeConverter.setConversionService(this.conversionService);
		}
	}
	return this.typeConverter;
}

//构造函数
public SimpleTypeConverter() {
	this.typeConverterDelegate = new TypeConverterDelegate(this);
	//注册默认PropertyEditor
	registerDefaultEditors();
}
```

使用简单转换器进行类型转化

```
@Override
public <T> T convertIfNecessary(Object value, Class<T> requiredType, MethodParameter methodParam)
		throws TypeMismatchException {

	return doConvert(value, requiredType, methodParam, null);
}



private <T> T doConvert(Object value, Class<T> requiredType, MethodParameter methodParam, Field field)
		throws TypeMismatchException {
	try {
		if (field != null) {
			return this.typeConverterDelegate.convertIfNecessary(value, requiredType, field);
		}
		else {
			return this.typeConverterDelegate.convertIfNecessary(value, requiredType, methodParam);
		}
	}
	catch (ConverterNotFoundException ex) {
		throw new ConversionNotSupportedException(value, requiredType, ex);
	}
	catch (ConversionException ex) {
		throw new TypeMismatchException(value, requiredType, ex);
	}
	catch (IllegalStateException ex) {
		throw new ConversionNotSupportedException(value, requiredType, ex);
	}
	catch (IllegalArgumentException ex) {
		throw new TypeMismatchException(value, requiredType, ex);
	}
}
```

调用`simpleTypeConverter`的成员变量`typeConverterDelegate`转换

```
public <T> T convertIfNecessary(Object newValue, Class<T> requiredType, MethodParameter methodParam)
		throws IllegalArgumentException {

	return convertIfNecessary(null, null, newValue, requiredType,
			(methodParam != null ? new TypeDescriptor(methodParam) : TypeDescriptor.valueOf(requiredType)));
}

```

然后到关键代码，这里的`propertyEditorRegistry`指向的就是之前的`simpleTypeConverter`

```
public <T> T convertIfNecessary(String propertyName, Object oldValue, Object newValue,
		Class<T> requiredType, TypeDescriptor typeDescriptor) throws IllegalArgumentException {

	Object convertedValue = newValue;

	// Custom editor for this type?
	//查找我们通过@InitBinder注入的PropertyEditor适用吗
	PropertyEditor editor = this.propertyEditorRegistry.findCustomEditor(requiredType, propertyName);

	ConversionFailedException firstAttemptEx = null;
	
	// No custom editor but custom ConversionService specified?
	//找不到则从conversionService中查找适用的`Converter`
	ConversionService conversionService = this.propertyEditorRegistry.getConversionService();
	//没有找到自定义的PropertyEditor时且conversionservice中有适用的converter
	//使用它来进行类型转换
	if (editor == null && conversionService != null && convertedValue != null && typeDescriptor != null) {
		TypeDescriptor sourceTypeDesc = TypeDescriptor.forObject(newValue);
		TypeDescriptor targetTypeDesc = typeDescriptor;
		if (conversionService.canConvert(sourceTypeDesc, targetTypeDesc)) {
			try {
				return (T) conversionService.convert(convertedValue, sourceTypeDesc, targetTypeDesc);
			}
			catch (ConversionFailedException ex) {
				// fallback to default conversion logic below
				firstAttemptEx = ex;
			}
		}
	}

	// Value not of required type?
	//自定义的PropertyEditor适用时或者值的类型不属于要转换的那个类型
	if (editor != null || (requiredType != null && !ClassUtils.isAssignableValue(requiredType, convertedValue))) {
		//String转集合
		if (requiredType != null && Collection.class.isAssignableFrom(requiredType) && convertedValue instanceof String) {
			TypeDescriptor elementType = typeDescriptor.getElementTypeDescriptor();
			if (elementType != null && Enum.class.isAssignableFrom(elementType.getType())) {
				convertedValue = StringUtils.commaDelimitedListToStringArray((String) convertedValue);
			}
		}
		//自定义PropertyEditor和conversionservice中都没有适用的
		//从默认注册的PropertyEditor中查找
		if (editor == null) {
			editor = findDefaultEditor(requiredType);
		}
		//进行转换，如果要转换的值convertedValue不属于String，转换用的是setValue方法
		convertedValue = doConvertValue(oldValue, convertedValue, requiredType, editor);
	}
	
	//以上对值的转换完成，由Object类型的convertedValue接收转换好的值
	boolean standardConversion = false;
	
	//以下应该是根据requiredType对转换好的值convertedValue进行再处理
	if (requiredType != null) {
		// Try to apply some standard type conversion rules if appropriate.

		if (convertedValue != null) {
			if (Object.class.equals(requiredType)) {
				return (T) convertedValue;
			}
			else if (requiredType.isArray()) {
				// Array required -> apply appropriate conversion of elements.
				if (convertedValue instanceof String && Enum.class.isAssignableFrom(requiredType.getComponentType())) {
					convertedValue = StringUtils.commaDelimitedListToStringArray((String) convertedValue);
				}
				return (T) convertToTypedArray(convertedValue, propertyName, requiredType.getComponentType());
			}
			else if (convertedValue instanceof Collection) {
				// Convert elements to target type, if determined.
				convertedValue = convertToTypedCollection(
						(Collection<?>) convertedValue, propertyName, requiredType, typeDescriptor);
				standardConversion = true;
			}
			else if (convertedValue instanceof Map) {
				// Convert keys and values to respective target type, if determined.
				convertedValue = convertToTypedMap(
						(Map<?, ?>) convertedValue, propertyName, requiredType, typeDescriptor);
				standardConversion = true;
			}
			if (convertedValue.getClass().isArray() && Array.getLength(convertedValue) == 1) {
				convertedValue = Array.get(convertedValue, 0);
				standardConversion = true;
			}
			if (String.class.equals(requiredType) && ClassUtils.isPrimitiveOrWrapper(convertedValue.getClass())) {
				// We can stringify any primitive value...
				return (T) convertedValue.toString();
			}
			else if (convertedValue instanceof String && !requiredType.isInstance(convertedValue)) {
				if (firstAttemptEx == null && !requiredType.isInterface() && !requiredType.isEnum()) {
					try {
						Constructor<T> strCtor = requiredType.getConstructor(String.class);
						return BeanUtils.instantiateClass(strCtor, convertedValue);
					}
					catch (NoSuchMethodException ex) {
						// proceed with field lookup
						if (logger.isTraceEnabled()) {
							logger.trace("No String constructor found on type [" + requiredType.getName() + "]", ex);
						}
					}
					catch (Exception ex) {
						if (logger.isDebugEnabled()) {
							logger.debug("Construction via String failed for type [" + requiredType.getName() + "]", ex);
						}
					}
				}
				String trimmedValue = ((String) convertedValue).trim();
				if (requiredType.isEnum() && "".equals(trimmedValue)) {
					// It's an empty enum identifier: reset the enum value to null.
					return null;
				}
				convertedValue = attemptToConvertStringToEnum(requiredType, trimmedValue, convertedValue);
				standardConversion = true;
			}
		}
		else {
			// convertedValue == null
			if (javaUtilOptionalEmpty != null && requiredType.equals(javaUtilOptionalEmpty.getClass())) {
				convertedValue = javaUtilOptionalEmpty;
			}
		}

		if (!ClassUtils.isAssignableValue(requiredType, convertedValue)) {
			if (firstAttemptEx != null) {
				throw firstAttemptEx;
			}
			// Definitely doesn't match: throw IllegalArgumentException/IllegalStateException
			StringBuilder msg = new StringBuilder();
			msg.append("Cannot convert value of type [").append(ClassUtils.getDescriptiveType(newValue));
			msg.append("] to required type [").append(ClassUtils.getQualifiedName(requiredType)).append("]");
			if (propertyName != null) {
				msg.append(" for property '").append(propertyName).append("'");
			}
			if (editor != null) {
				msg.append(": PropertyEditor [").append(editor.getClass().getName()).append(
						"] returned inappropriate value of type [").append(
						ClassUtils.getDescriptiveType(convertedValue)).append("]");
				throw new IllegalArgumentException(msg.toString());
			}
			else {
				msg.append(": no matching editors or conversion strategy found");
				throw new IllegalStateException(msg.toString());
			}
		}
	}

	if (firstAttemptEx != null) {
		if (editor == null && !standardConversion && requiredType != null && !Object.class.equals(requiredType)) {
			throw firstAttemptEx;
		}
		logger.debug("Original ConversionService attempt failed - ignored since " +
				"PropertyEditor based conversion eventually succeeded", firstAttemptEx);
	}

	return (T) convertedValue;
}
```

至此，参数解析器中对参数进行类型转化完成，接着后面会用反射调用方法，将这个作为转化后的值作为入参

当然这个属于单个属性的转换，我们经常会用到使用一个类来接收我们需要的多个属性参数，这种情况就使用到了`BeanWrapperImpl`这个类了，它继承了`AbstractPropertyAccessor`，可以对类中属性进行操作，同时它也有个形参`conversionservice`，和实现了接口`PropertyEditorRegistry`接口，有了给我们注入自定义`PropertyEditor`的功能

转换过程差不多，解析到参数为复杂类时，在创建`DataBinder`时，创建了内部变量`BindingResult`和它的内部变量`BeanWrapperImpl`，然后初始化注入自定义`PropertyEditor`和`conversionservice`，哦还有将这个参数类反射实例化通过构造方法传入`BeanWrapperImpl`，然后调用`BeanWrapperImpl`的`setProperty`将请求中参数名字和参数值设置到类中对应的属性，所以请求中参数名一定要和属性名相等

在这个过程中，通过`BeanWrapperImpl`中的`cachedIntrospectionResults`获取所有属性的`PropertyDescriptor`对象，通过这个对象中的属性的`get`,`set`方法确认属性的具体类型，然后开始和我们上面一样的转换过程

转换完之后调用`PropertyDescriptor`中属性的`set`方法将值注入类中

循环完成之后就完成了对将请求中属性绑定到入参的类中


