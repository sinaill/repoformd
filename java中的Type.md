title: java中的Type
categories: java基础

---

### 什么是Type

在Java中，泛型与反射是两个重要的概念，我们几乎能够经常的使用到它们。而谈起Type，如果有人还比较陌生的话 ，那么说起一个它的直接实现类——Class的话，大家都应该明白了。Type是Java语言中所有类型的公共父接口。

![](https://raw.githubusercontent.com/sinaill/pic/master/Type.jpg)

其实在jdk1.5之前Java中只有原始类型而没有泛型类型，而在JDK 1.5 之后引入泛型，但是这种泛型仅仅存在于编译阶段，当在JVM运行的过程中，与泛型相关的信息将会被擦除，如List与List都将会在运行时被擦除成为List这个类型。而类型擦除机制存在的原因正是因为如果在运行时存在泛型，那么将要修改JVM指令集，这是非常致命的。
此外，**原始类型在会生成字节码文件对象，而泛型类型相关的类型并不会生成与其相对应的字节码文件(因为泛型类型将会被擦除)，因此，无法将泛型相关的新类型与class相统一。因此，为了程序的扩展性以及为了开发需要去反射操作这些类型，就引入了Type这个类型**，并且新增了ParameterizedType, TypeVariable, GenericArrayType, WildcardType四个表示泛型相关的类型，再加上Class，这样就可以用Type类型的参数来接受以上五种子类的实参或者返回值类型就是Type类型的参数。统一了与泛型有关的类型和原始类型Class。而且这样一来，我们也可以通过反射获取泛型类型参数
。

### 四个子类

#### ParameterizedType：参数化类型

参数化类型即我们通常所说的泛型类型，一提到参数，最熟悉的就是定义方法时有形参，然后调用此方法时传递实参。那么参数化类型怎么理解呢？顾名思义，就是将类型由原来的具体的类型参数化，类似于方法中的变量参数，此时类型也定义成参数形式（可以称之为类型形参），然后在使用/调用时传入具体的类型（类型实参）。那么我们的ParameterizedType就是这样一个类型，下面我们来看看它的三个重要的方法：

- Type getRawType();

该方法的作用是返回当前的ParameterizedType的类型。如一个List，返回的是List的Type，即返回当前参数化类型本身的Type。

- Type getOwnerType();

返回ParameterizedType类型所在的类的Type。如Map.Entry<String, Object>这个参数化类型返回的事Map(因为Map.Entry这个类型所在的类是Map)的类型。

- Type[] getActualTypeArguments();

该方法返回参数化类型<>中的实际参数类型， 如 Map<String,Person> map 这个 ParameterizedType 返回的是 String 类,Person 类的全限定类名的 Type Array。注意: 该方法只返回最外层的<>中的类型，无论该<>内有多少个<>。

```
class testType{
	HashMap<String,Object> map;
	List<Integer> list;
	String str;
}

@Test
public void testType() {
	Field fields[] = testType.class.getDeclaredFields();
	for(Field field:fields) {
		System.out.println("Field : "+field.getName() + " is parameterizedType :"+(field.getGenericType() instanceof ParameterizedType));
		
		if(field.getGenericType() instanceof ParameterizedType) {
			System.out.println("getRawType: "+((ParameterizedType)field.getGenericType()).getRawType());
			for(Type type:((ParameterizedType)field.getGenericType()).getActualTypeArguments())
			//getActualTypeArguments返回的类型为TypeVariable
			System.out.println("getActualTypeArguments: "+type.getTypeName());
		}
	}
}


Field : map is parameterizedType :true
getRawType: class java.util.HashMap
getActualTypeArguments: java.lang.String
getActualTypeArguments: java.lang.Object
Field : list is parameterizedType :true
getRawType: interface java.util.List
getActualTypeArguments: java.lang.Integer
Field : str is parameterizedType :false
```

#### TypeVariable

类型变量，即泛型中的变量；例如：T、K、V等变量，可以表示任何类；在这需要强调的是，TypeVariable代表着泛型中的变量，而ParameterizedType则代表整个泛型；

- Type[] getBounds();

返回当前类型的上边界，如果没有指定上边界，则默认为Object

- D getGenericDeclaration();

返回当前类型所在的类的Type

- String getName();

返回当前类型的类名

```
class testType<k extends Number,T>{
	HashMap<String,Object> map;
	List<Integer> list;
	String str;
}

@Test
public void testTypeVariable(){
	 Type types[] = testType.class.getTypeParameters();
	 //返回的数组包含泛型表示的类型变量K,T
	 for(Type type:types) {
		 //遍历K,T
		 TypeVariable t = (TypeVariable)type;
		 int index = t.getBounds().length-1;
         //返回的表示上边界的是一个数组
		 System.out.println("getBounds: "+t.getBounds()[index]);
		 System.out.println("getName: "+t.getName());
         //表示泛型变量所在的类
		 System.out.println("getGenericDeclaration: "+t.getGenericDeclaration());
	 }
}


getBounds: class java.lang.Number
getName: K
getGenericDeclaration: class test.testType
getBounds: class java.lang.Object
getName: T
getGenericDeclaration: class test.testType
```

#### GenericArrayType

泛型数组类型，用来描述ParameterizedType、TypeVariable类型的数组；即List<T>[] 、T[]等

```
class testType<K,T>{
	HashMap<K,T>[] map;
	List<K>[] list;
	T Array[];
}

@Test
public void testGenericArrayType() {
	Field fields[] = testType.class.getDeclaredFields();
	for(Field field:fields) {
		System.out.println(field.getName()+" instance of GenericArrayType: "+(field.getGenericType() instanceof GenericArrayType));
		System.out.println(field.getName()+" instance of GenericArrayType: "+(field.getGenericType() instanceof ParameterizedType));
		//对于getGenericComponentType
        //如果是参数化类型数组，则这里返回的是parameterizedType类型
		//普通泛型数组，返回的TypeVariable类型
		System.out.println(field.getName()+":getGenericComponentType "+((GenericArrayType)field.getGenericType()).getGenericComponentType());
	}
}


map instance of GenericArrayType: true
map instance of GenericArrayType: false
map:getGenericComponentType java.util.HashMap<K, T>
list instance of GenericArrayType: true
list instance of GenericArrayType: false
list:getGenericComponentType java.util.List<K>
Array instance of GenericArrayType: true
Array instance of GenericArrayType: false
Array:getGenericComponentType T
```


#### WildcardType

表示通配符类型，比如 <?>, <? Extends Number>等

- Type[] getUpperBounds()

得到上边界的type数组

- Type[] getLowerBounds();

得到下边界的type数组

**如果没有指定上边界，则默认为Object，如果没有指定下边界，则默认为String**

```
class testWildcardType{
	List<? extends Number> listNum;
	List<? super String> listStr;
}

@Test
public void testWildcardType() {
	Field fields[] = testWildcardType.class.getDeclaredFields();
	for(Field field:fields) {
		
		ParameterizedType t = (ParameterizedType) field.getGenericType();
		System.out.println(t.getTypeName());
		Type[] Type = t.getActualTypeArguments();
		System.out.println(field.getName()+".getGenericType.getActualTypeArguments(): instance of WildcardType is "+(Type[0] instanceof WildcardType));
		WildcardType wildcardType = (WildcardType) Type[0];
		if(wildcardType.getUpperBounds().length>0) {
			
			System.out.println("getUpperBounds: "+wildcardType.getUpperBounds()[0].getTypeName());
		}
		if(wildcardType.getLowerBounds().length>0) {
			
			System.out.println("getLowerBouds: "+wildcardType.getLowerBounds()[0].getTypeName());
		}
	}
}

java.util.List<? extends java.lang.Number>
listNum.getGenericType.getActualTypeArguments(): instance of WildcardType is true
getUpperBounds: java.lang.Number
java.util.List<? super java.lang.String>
listStr.getGenericType.getActualTypeArguments(): instance of WildcardType is true
getUpperBounds: java.lang.Object
getLowerBouds: java.lang.String
```

相关链接

[参考一](https://juejin.im/post/5adefaba518825670e5cb44d)
[参考二](https://blog.csdn.net/ZytheMoon/article/details/79241988)
