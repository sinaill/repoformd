title: BeanWrapperImpl初探
categories: 框架
tags: 
	- SpringMVC
	- Spring


---

### BeanWrapperImpl

`BeanWrapper`接口是用来操作javaBean属性，例如在SpringMVC中，在`Databinder`中也使用了`BeanWrapperImpl`来对入参的类进行属性设置和对应的将`Request`中传来的类型为`String`的参数转化为入参类对应的属性的类型

### 操作属性

#### BeanWrapperImpl

定义一个测试类

```
class Man{
    private String name;
    private int age;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return this.name+ " "+ this.age;
    }
}
```

然后获取它的`BeanWrapper`实例

```
@Test
public void test2(){
    BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(new Man());
    bw.setPropertyValue("name", "张三");
    bw.setPropertyValue("age", "11");
    System.out.println(bw.getWrappedInstance().toString());
}
```

正好输出结果为

```
张三 11
```

在没有调用`Man`对象自身的`set`方法的情况下，为`Man`对象的属性成功赋了值

#### CachedIntrospectionResults

`CachedIntrospectionResults`为`BeanWrapperImpl`中的`final`私有属性

`BeanWrapperImpl`类能对属性进行操作主要在于`CachedIntrospectionResults`的作用

看`CachedIntrospectionResults`的构造函数

```
private CachedIntrospectionResults(Class<?> beanClass) throws BeansException {
	try {
		if (logger.isTraceEnabled()) {
			logger.trace("Getting BeanInfo for class [" + beanClass.getName() + "]");
		}

		BeanInfo beanInfo = null;
		for (BeanInfoFactory beanInfoFactory : beanInfoFactories) {
			beanInfo = beanInfoFactory.getBeanInfo(beanClass);
			if (beanInfo != null) {
				break;
			}
		}
		if (beanInfo == null) {
			// If none of the factories supported the class, fall back to the default
			beanInfo = (shouldIntrospectorIgnoreBeaninfoClasses ?
					Introspector.getBeanInfo(beanClass, Introspector.IGNORE_ALL_BEANINFO) :
					Introspector.getBeanInfo(beanClass));
		}
		this.beanInfo = beanInfo;

		if (logger.isTraceEnabled()) {
			logger.trace("Caching PropertyDescriptors for class [" + beanClass.getName() + "]");
		}
		this.propertyDescriptorCache = new LinkedHashMap<String, PropertyDescriptor>();

		// This call is slow so we do it once.
		PropertyDescriptor[] pds = this.beanInfo.getPropertyDescriptors();
		for (PropertyDescriptor pd : pds) {
			if (Class.class.equals(beanClass) &&
					("classLoader".equals(pd.getName()) ||  "protectionDomain".equals(pd.getName()))) {
				// Ignore Class.getClassLoader() and getProtectionDomain() methods - nobody needs to bind to those
				continue;
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Found bean property '" + pd.getName() + "'" +
						(pd.getPropertyType() != null ? " of type [" + pd.getPropertyType().getName() + "]" : "") +
						(pd.getPropertyEditorClass() != null ?
								"; editor [" + pd.getPropertyEditorClass().getName() + "]" : ""));
			}
			pd = buildGenericTypeAwarePropertyDescriptor(beanClass, pd);
			this.propertyDescriptorCache.put(pd.getName(), pd);
		}

		this.typeDescriptorCache = new ConcurrentReferenceHashMap<PropertyDescriptor, TypeDescriptor>();
	}
	catch (IntrospectionException ex) {
		throw new FatalBeanException("Failed to obtain BeanInfo for class [" + beanClass.getName() + "]", ex);
	}
}
```

主要看这段代码`PropertyDescriptor[] pds = this.beanInfo.getPropertyDescriptors();`，由传入的`Class`对象获取到`BeanInfo`对象，然后从这个对象获取`PropertyDescriptor[]`数组，有关于`PropertyDescriptor`类的作用为，存放了目标类中某个属性的`get`，`set`方法，就是用它们来对类的属性进行填充设置

由于源代码太长，我们进行简单模拟一下实现原理

```
@Test
public void test3() throws Exception{
	//构建`BeanWrapperImpl对象`
    BeanWrapperImpl bw = new BeanWrapperImpl(new Man());
	//反射获取getCachedIntrospectionResults放啊
    Method getCachedIntrospectionResults = bw.getClass().getDeclaredMethod("getCachedIntrospectionResults");
	//设置私有方法可访问
    getCachedIntrospectionResults.setAccessible(true);
	//反射调用方法获取CachedIntrospectionResults属性
    CachedIntrospectionResults cir = (CachedIntrospectionResults)getCachedIntrospectionResults.invoke(bw);
	//获取CachedIntrospectionResults变量内部的getPropertyDescriptor方法对象
    Method method = cir.getClass().getDeclaredMethod("getPropertyDescriptor",new Class[]{String.class});
    method.setAccessible(true);
	//反射获取属性名为age的PropertyDescriptor变量
    PropertyDescriptor agePropertyDescriptor = (PropertyDescriptor)method.invoke(cir, "age");
	//从PropertyDescriptor中获取`set`方法
    Method ageWriteMethod = agePropertyDescriptor.getWriteMethod();
	//调用set方法对属性进行设置
    ageWriteMethod.invoke(bw.getWrappedInstance(), 11);
    System.out.println(bw.getWrappedInstance().toString());

}
```

结果输出的是`null 11`，成功对属性进行了设置

#### PropertyEditor

可以注意到我们在直接使用`BeanWrapperImpl`设置属性的时候，传入的属性会由`String`自动转化为对应的属性类型，比如我们给`age`传入的是`String`类型的11，自动转化成了`int`类型

`BeanWrapperImpl`在调用`PropertyDescrptor`的`WriteMethod`之前，对属性进行了转化，这个工作是由`PropertyEditor`在起作用，它内置了多个类型的转化器`PropertyEditor`

我们先看`BeanWrapperImpl`的构造函数

```
public BeanWrapperImpl(Object object) {
	registerDefaultEditors();
	setWrappedInstance(object);
}

protected void registerDefaultEditors() {
	this.defaultEditorsActive = true;
}
```

可以看到注册了默认属性编辑器，并且它是属于懒加载

```
public PropertyEditor getDefaultEditor(Class<?> requiredType) {
	if (!this.defaultEditorsActive) {
		return null;
	}
	if (this.overriddenDefaultEditors != null) {
		PropertyEditor editor = this.overriddenDefaultEditors.get(requiredType);
		if (editor != null) {
			return editor;
		}
	}
	if (this.defaultEditors == null) {
		createDefaultEditors();
	}
	return this.defaultEditors.get(requiredType);
}
```

在需要对属性进行转化的时候，获取默认属性编辑器时，开始初始化内置的`PropertyEditor`

```
private void createDefaultEditors() {
	this.defaultEditors = new HashMap<Class<?>, PropertyEditor>(64);

	// Simple editors, without parameterization capabilities.
	// The JDK does not contain a default editor for any of these target types.
	this.defaultEditors.put(Charset.class, new CharsetEditor());
	this.defaultEditors.put(Class.class, new ClassEditor());
	this.defaultEditors.put(Class[].class, new ClassArrayEditor());
	this.defaultEditors.put(Currency.class, new CurrencyEditor());
	this.defaultEditors.put(File.class, new FileEditor());
	this.defaultEditors.put(InputStream.class, new InputStreamEditor());
	this.defaultEditors.put(InputSource.class, new InputSourceEditor());
	this.defaultEditors.put(Locale.class, new LocaleEditor());
	this.defaultEditors.put(Pattern.class, new PatternEditor());
	this.defaultEditors.put(Properties.class, new PropertiesEditor());
	this.defaultEditors.put(Resource[].class, new ResourceArrayPropertyEditor());
	this.defaultEditors.put(TimeZone.class, new TimeZoneEditor());
	this.defaultEditors.put(URI.class, new URIEditor());
	this.defaultEditors.put(URL.class, new URLEditor());
	this.defaultEditors.put(UUID.class, new UUIDEditor());
	if (zoneIdClass != null) {
		this.defaultEditors.put(zoneIdClass, new ZoneIdEditor());
	}

	// Default instances of collection editors.
	// Can be overridden by registering custom instances of those as custom editors.
	this.defaultEditors.put(Collection.class, new CustomCollectionEditor(Collection.class));
	this.defaultEditors.put(Set.class, new CustomCollectionEditor(Set.class));
	this.defaultEditors.put(SortedSet.class, new CustomCollectionEditor(SortedSet.class));
	this.defaultEditors.put(List.class, new CustomCollectionEditor(List.class));
	this.defaultEditors.put(SortedMap.class, new CustomMapEditor(SortedMap.class));

	// Default editors for primitive arrays.
	this.defaultEditors.put(byte[].class, new ByteArrayPropertyEditor());
	this.defaultEditors.put(char[].class, new CharArrayPropertyEditor());

	// The JDK does not contain a default editor for char!
	this.defaultEditors.put(char.class, new CharacterEditor(false));
	this.defaultEditors.put(Character.class, new CharacterEditor(true));

	// Spring's CustomBooleanEditor accepts more flag values than the JDK's default editor.
	this.defaultEditors.put(boolean.class, new CustomBooleanEditor(false));
	this.defaultEditors.put(Boolean.class, new CustomBooleanEditor(true));

	// The JDK does not contain default editors for number wrapper types!
	// Override JDK primitive number editors with our own CustomNumberEditor.
	this.defaultEditors.put(byte.class, new CustomNumberEditor(Byte.class, false));
	this.defaultEditors.put(Byte.class, new CustomNumberEditor(Byte.class, true));
	this.defaultEditors.put(short.class, new CustomNumberEditor(Short.class, false));
	this.defaultEditors.put(Short.class, new CustomNumberEditor(Short.class, true));
	this.defaultEditors.put(int.class, new CustomNumberEditor(Integer.class, false));
	this.defaultEditors.put(Integer.class, new CustomNumberEditor(Integer.class, true));
	this.defaultEditors.put(long.class, new CustomNumberEditor(Long.class, false));
	this.defaultEditors.put(Long.class, new CustomNumberEditor(Long.class, true));
	this.defaultEditors.put(float.class, new CustomNumberEditor(Float.class, false));
	this.defaultEditors.put(Float.class, new CustomNumberEditor(Float.class, true));
	this.defaultEditors.put(double.class, new CustomNumberEditor(Double.class, false));
	this.defaultEditors.put(Double.class, new CustomNumberEditor(Double.class, true));
	this.defaultEditors.put(BigDecimal.class, new CustomNumberEditor(BigDecimal.class, true));
	this.defaultEditors.put(BigInteger.class, new CustomNumberEditor(BigInteger.class, true));

	// Only register config value editors if explicitly requested.
	if (this.configValueEditorsActive) {
		StringArrayPropertyEditor sae = new StringArrayPropertyEditor();
		this.defaultEditors.put(String[].class, sae);
		this.defaultEditors.put(short[].class, sae);
		this.defaultEditors.put(int[].class, sae);
		this.defaultEditors.put(long[].class, sae);
	}
}

```

`createDefaultEditors`方法初始化了以上属性编辑器

这些编辑器继承了`PropertyEditorSupport`，覆写它内部的`setAsText`方法来对`String`转化为其他类型变量

例如我们这里使用的应该是`CustomNumberEditor`来将字符串`11`转化为`int`类型`11`


```
@Override
public void setAsText(String text) throws IllegalArgumentException {
	if (this.allowEmpty && !StringUtils.hasText(text)) {
		// Treat empty String as null value.
		setValue(null);
	}
	else if (this.numberFormat != null) {
		// Use given NumberFormat for parsing text.
		setValue(NumberUtils.parseNumber(text, this.numberClass, this.numberFormat));
	}
	else {
		// Use default valueOf methods for parsing text.
		setValue(NumberUtils.parseNumber(text, this.numberClass));
	}
}
```

将我们传入的`String`类型属性转化为对应的`numberClass`

接着我们使用代码模拟下这个过程，根据目标属性选择一个合适的`PropertyEditor`

```
@Test
public void test3() throws Exception{
    BeanWrapperImpl bw = new BeanWrapperImpl(new Man());
    Method getCachedIntrospectionResults = bw.getClass().getDeclaredMethod("getCachedIntrospectionResults");
    getCachedIntrospectionResults.setAccessible(true);
    CachedIntrospectionResults cir = (CachedIntrospectionResults)getCachedIntrospectionResults.invoke(bw);
    Method method = cir.getClass().getDeclaredMethod("getPropertyDescriptor",new Class[]{String.class});
    method.setAccessible(true);
    PropertyDescriptor agePropertyDescriptor = (PropertyDescriptor)method.invoke(cir, "age");
    Method ageWriteMethod = agePropertyDescriptor.getWriteMethod();
	//根据目标属性的类选择对应的PropertyEditor
    PropertyEditor propertyEditor = bw.getDefaultEditor(ageWriteMethod.getParameterTypes()[0]);
	//转化
    propertyEditor.setAsText("11");
	//获取转化后的值
    Object value = propertyEditor.getValue();
    ageWriteMethod.invoke(bw.getWrappedInstance(), value);
    System.out.println(bw.getWrappedInstance().toString());

}
```

### 在SpringMVC中

在`SpringMVC`中`BeanWrapperImpl`帮助我们将客户端发来的`Request`中的参数由`String`类型转化为我们需要的类型，它与`DataBinder`对象一起使用，在`DataBinder`中被初始化，然后传入了另一些`Converter`转换器，`BeanWrapperImpl`中有个`ConversionService`类型的成员变量，所有的转换器是以它为载体传入的

常用场景为将`Request`中多个参数填充到方法的的某个入参类中，在其中进行必要的类型转换

