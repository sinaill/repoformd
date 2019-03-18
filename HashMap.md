title: HashMap
categories: java基础
tags: 
	- 集合
---

底层结构为数组加链表，插入元素是无序的，维护插入顺序时使用LinkedHashMap，它维护着一个双向循环链表，保证了我们遍历得到元素的顺序和我们插入的顺序一致

### 成员变量

```
	/**
     * The default initial capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
```

初始化默认容量为16，且容量必须为2的幂

`static final int MAXIMUM_CAPACITY = 1 << 30;`

最大容量2的30次幂

`static final float DEFAULT_LOAD_FACTOR = 0.75f;`

默认负载因子0.75f，**负载因子表示当元素个数达到负载因子乘于容量时就进行扩容，负载因子为final类型**

增大负载因子会减少hash表占用内存空间，但是会增加查询的时间开销

减少负载因子则相反

`transient Node<K,V>[] table;`

存储元素的链表数组

`transient Set<Map.Entry<K,V>> entrySet;`

HashMap中的键值对Entry组成的集合

### 构造函数

`public HashMap(int initialCapacity, float loadFactor)`

由容量和负载因子作为参数来初始化HashMap

```
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }
```

由容量作为参数，负载因子使用默认值来初始化HashMap

```
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
```

无参数时，负载因子为默认值，size为0

### 遍历方法

(1) foreach map.entrySet()

```
Map<String, String> map = new HashMap<String, String>();
map.entrySet().forEach(e->System.out.println(e.getValue()+e.getKey()));
```

(2) 显示调用map.entrySet()的集合迭代器

```
Iterator<Map.Entry<String, String>> iterator = map.entrySet().iterator();
while (iterator.hasNext()) {
	Map.Entry<String, String> entry = iterator.next();
	entry.getKey();
	entry.getValue();
}
```

(3) for each map.keySet()，再调用get获取

```
Map<String, String> map = new HashMap<String, String>();
for (String key : map.keySet()) {
	map.get(key);
}
```

(4) map.forEach java8新增方法

````
	map.forEach((k,v)->System.out.println(k+v))
````
### 原理

当我们使用put方法向HashMap中添加元素时，我们先对键调用hashCode()方法，返回的hashCode用于找到table中位置来储存Entry对象，当key的hashcode相同且equals不相等时，**它们将在同一位置迭代成链表**。此时调用get方法获取值对象时，先通过key的hashcode查找到位置，再通过key的equals方法在链表中查找目标Entry对象。

元素达到负载因子限制的容量时，自动扩容会重新分配Entry在table中的位置


