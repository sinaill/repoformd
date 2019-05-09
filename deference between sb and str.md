title: StringBuffer与String的区别
categories: java基础
---

### 查阅

翻阅String和StringBuffer和源码可知
String中用来存储字符串的char[]为

`private final char value[]`

StringBuffer中用来存储字符串的char[]为

`char[] value;`

### 构造方法的差别

#### String

String的构造方法较为简单

```
 public String(String original) {
        this.value = original.value;
        this.hash = original.hash;
    }
```

通过反射获取两个String对象中的value[]，结果为true;

```
		String a = new String("abc");
		String b = new String("abc");
		//"abc"返回同一个String对象(常量池)，调用参数为String的构造方法
		Field af = a.getClass().getDeclaredField("value");
		Field bf = b.getClass().getDeclaredField("value");
		af.setAccessible(true);
		bf.setAccessible(true);
		System.out.println(af.get(a) == bf.get(b));
```

**由此我们可以看出，由于String常量池的存在，当我们用同一字符串来构造String对象时，他们中的value[]都指向了常量池中的该字符串对象的value[]**

#### StringBuffer

StringBuffer中参数为StringBuffer和String的构造函数，跟源码得

```
    public StringBuffer(String str) {
        super(str.length() + 16);
        append(str);
    }
	
    public AbstractStringBuilder append(String str) {
        if (str == null)
            return appendNull();
        int len = str.length();
        ensureCapacityInternal(count + len);
        str.getChars(0, len, value, count);
        count += len;
        return this;
    }
```

与String不同，它时将传进来的string复制到变量value中，所以用相同字符串初始化StringBuffer，利用反射获取value[]，比较它们的引用，结果为false

```
		StringBuffer a = new StringBuffer("abc");
		StringBuffer b = new StringBuffer("abc");
		Field af = a.getClass().getSuperclass().getDeclaredField("value");
		Field bf = a.getClass().getSuperclass().getDeclaredField("value");
		af.setAccessible(true);
		bf.setAccessible(true);
		System.out.println(af.get(a) == bf.get(b));
```

**可以看出StringBuffer中的value[]指向不同地址**

### 总结

**1.对于equals为true的String对象，它们内部value[]指向相同地址，且为final类型，初始化后无法改变其引用，虽然可以通过反射获取引用，来改变value中的值，但是会改变所有String对象中的value**

**2.StringBuffer中的value为default,所以上面我们不能直接用a.value或者b.vlaue获取value[],并且StringBuffer中即使用相同String对象或者StringBuffer初始化时，它们之间的value[]也是指向不同地址**

**3.String和StringBuffer类型为final**

**所以当为String对象改变值时，其实是创建了新的String对象再复制给它**

**而在StringBuffer中改变值时可以改变自身value[]，不用创建对象，节省资源**

**当我们需要频繁改变字符串内容时最好使用StringBuffer而不是String**
