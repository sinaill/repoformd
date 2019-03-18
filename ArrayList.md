title: ArrayList
categories: java基础
tags: 
	- 集合
---
### 结构

内部维护了一个自动扩容的Object类型数组

### 成员变量


`private static final int DEFAULT_CAPACITY = 10;` 

默认初始化容量

`private static final Object[] EMPTY_ELEMENTDATA = {};` 

Shared empty array instance used for empty instances.

`private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};`

两个的区别是，上面的是初始化容量为0时共享的array instance，下面的是以默认容量初始化时共享的array instance.

```
/**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
	   transient Object[] elementData;
```

存储元素的Object数组，empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA will be expanded to DEFAULT_CAPACITY when the first element is added

### 构造函数
```
    /**
     * Constructs an empty list with an initial capacity of ten.
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```
无参构造函数时，Constructs an empty list with an initial capacity of ten

```
/**
     * Constructs an empty list with the specified initial capacity.
     *
     * @param  initialCapacity  the initial capacity of the list
     * @throws IllegalArgumentException if the specified initial capacity
     *         is negative
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```

int类型为参数的构造函数，构造确定容量的ArrayList，**而不是默认的10位**

```
/**
     * Constructs a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param c the collection whose elements are to be placed into this list
     * @throws NullPointerException if the specified collection is null
     */
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }
```

可传入List或者Set类型参数，由它们存放的内容来实例化ArrayList

### 遍历方法

```
		List<String> a = new ArrayList<String>();
		a.add("abc");
		a.add("ddd");
		a.add("ccc");

		for (String string : a) {
			System.out.println(string);
		}
		
		Iterator<String> it = a.iterator();
		while(it.hasNext()){	
			System.out.println(it.next());
		}

		a.forEach(str->system.out.println(str))
```

### 自动扩容

```
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

   private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

先1.5倍扩容，如果还是不够，直接设置为当前size，再比较MAX_ARRAY_SIZE，扩容后新容量大于它时，且minCapacity即当前包含元素个数<MAX_ARRAY_SIZE，则新容量设置为MAX_ARRAY_SIZE。
反之，扩容后新容量和当前包含元素个数都都大于MAX_ARRAY_SIZE时，新容量设置为Integer.MAX_VALUE

