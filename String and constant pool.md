title: String与常量池
categories: java基础
---

### 创建方法

`String str=new String("abc")`

**一共创建两个对象，其中new方法在堆中创建一个String对象，""为String特有的创建对象的方法，这行代码被执行的时候，JAVA虚拟机首先在字符串池中查找是否已经存在了值为"abc"的这么一个对象，它的判断依据是String类equals(Object obj)方法的返回值。如果有，则不再创建新的对象，直接返回已存在对象的引用；如果没有，则先创建这个对象，然后把它加入到字符串池中，再将它的引用返回**

```
		String str1 = "abc";
		String str2 = new String("abc");
		System.out.println(str1 == str2);
```

`false`

**str2指向堆中的对象，str1指向常量池中的对象**

### intern方法

`string.intern()`

实例1：

```
		String str1 = "abc";
		String str2 = new String("abc");
		str2 = "abc".intern();
		System.out.println(str1 == str2);
```

`true`

**会检查常量池中是否存在string，存在则返回池里的字符串引用；如果不存在则将string引用放入常量池中(jdk1.7后)。这样做可以避免在堆中不断的创建字符串对象，起到节省空间的作用。**

实例2：

```
String s3 = new String("1") + new String("1");  
s3.intern();  
String s4 = "11";  
System.out.println(s3 == s4);
```

`true`

实例3：

```
String s3 = new String("1") + new String("1");  
String s4 = "11";  
s3.intern();  
System.out.println(s3 == s4);
```

`false`

**调换顺序后出现不同结果的原因**

**1.String s3 = new String("1") + new String("1")，字符串常量池中生成“1” ，并在堆空间中生成s3引用指向的对象（内容为"11"）。注意此时常量池中是没有 “11”对象的**

**2.实例2中s3.intern()，此时常量池中没有"11"对象，所以在常量池中存储s3引用(jdk1.7以后)，String s4 = "11",首先从常量池中查看(equals方法)有无"11"对象，所以将s3引用传回去，所以结果true，而在实例3中，String s4 = "11"在s3.intern()方法前，此时常量池中已经有"11"对象，s3.intern()无效，所以s3还是指向堆中对象，所以结果false**