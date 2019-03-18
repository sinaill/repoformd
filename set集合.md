title: set集合
categories: java基础
tags: 
	- 集合
---

### 特点

`Set`集合中的元素是唯一的，不可重复（取决于`hashCode和equals`方法），也就是说具有唯一性。

`Set`集合中元素不保证存取顺序，并不存在索引。

### HashSet

`HashSet`实际上是基于`HashMap`实现的，HashSet中有一成员变量

`private transient HashMap<E,Object> map`

`HashSet`使用了`HashMap`的`key`来存储集合元素，也就是说`HashSet`可以存储一个`null`元素，并且不能有重复元素

```
private static final Object PRESENT = new Object()

   public boolean add(E e) {
       return map.put(e, PRESENT)==null;
   }
```

由于`HashMap`的无序性，所以`HashSet`也是无序性的

### LinkedHashSet

`LinkedHashSet`是`HashSet`的一个子类，`LinkedHashSet`也根据`HashCode`的值来决定元素的存储位置，但同时它还用一个链表来维护元素的插入顺序，插入的时候既要计算`hashCode`还要维护链表，而遍历的时候只需要按照链表来访问元素。

`LinkedHashSet`只有四个构造方法，构造方法调用了`HashSet`一个构造方法

```
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

可以看出，是基于`LinkedHashMap`实现，`HashSet`存储元素用的`HashMap`，而LinkedHashSet存储元素用的`LinkedHashMap`。

### TreeSet

`TreeSet`是一个有序集合，本质上属于二叉树，实现了`SortedSet`接口所以具有了对元素排序的功能，要实现对元素的排序，需要在构造方法中传入一个`comparator`或者元素实现了`comparable`接口。