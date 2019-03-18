title: LinkedHashMap与LRU算法
categories: java基础
tags:
	- 集合
---

使用LRU时，我们要用到这个构造函数

```
    /**
     * Constructs an empty <tt>LinkedHashMap</tt> instance with the
     * specified initial capacity, load factor and ordering mode.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @param  accessOrder     the ordering mode - <tt>true</tt> for
     *         access-order, <tt>false</tt> for insertion-order
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
```

对于变量accessOrder，传入false时，集合中元素顺序基于插入顺序，传入true是，集合中元素基于访问顺序

```
		LinkedHashMap<Integer, String> lhm = new LinkedHashMap<Integer,String>(8, 0.75f, true);
		lhm.put(3, "3");
		lhm.put(2, "2");
		lhm.put(1, "1");
		lhm.put(6, "6");
		lhm.put(5, "5");
		lhm.put(4, "4");
		lhm.get(2);
		lhm.forEach((a,b)->System.out.println(a));
```

得到输出

`3 1 6 5 4 2`

查看get方法

```
    public V get(Object key) {
        Node<K,V> e;
        if ((e = getNode(hash(key), key)) == null)
            return null;
        if (accessOrder)//access为true时，将访问元素放到尾部
            afterNodeAccess(e);
        return e.value;
    }

	    void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
            if (b == null)
                head = a;
            else
                b.after = a;
            if (a != null)
                a.before = b;
            else
                last = b;
            if (last == null)
                head = p;
            else {
                p.before = last;
                last.after = p;
            }
            tail = p;
            ++modCount;
        }
    }
```

调用get节点访问元素后，会继续调用afterNodeAccess将元素移动到尾部，要实现LinkedHashMap中元素个数大于容量*负载因子时，删除头部，即最近最少被访问的元素,先查看put方法，找到关键代码

```
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
```

进入该方法：

```
    void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }
    }
```

发现removeEldestEntry(first)，接着查看removeEldestEntry(first)

```
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
```

方法总是放回一个false，所以要实现移除头部，需要覆写本方法，我们新定义一个类继承LinkedHashMap

```
class MyLinkedHashMap<K,V> extends LinkedHashMap<K, V>{
	/**
	 * 
	 */
	private static final long serialVersionUID = 1L;
	private int MaxSize;

	public MyLinkedHashMap(int capacity,float factor,boolean accessOrder,int MaxSize){
		super(capacity,factor,accessOrder);
		this.MaxSize = MaxSize;
	}

	/* (non-Javadoc)
	 * @see java.util.LinkedHashMap#removeEldestEntry(java.util.Map.Entry)
	 */
	@Override
	protected boolean removeEldestEntry(Entry<K, V> eldest) {
		// TODO Auto-generated method stub
		return this.size()>MaxSize;
	}
	
	
}
```

构造方法调用父类上文提到那个，再加一个MaxSize表示集合元素最大个数，在removeEldestEntry方法中比较size和MaxSize大小，这样当集合中元素个数size超过设定最大个数MaxSize时，就会触发removeNode，移除头结点即最近最少使用的元素

```
		LinkedHashMap<Integer, String> lhm = new MyLinkedHashMap<Integer,String>(8, 0.75f, true,6);
		lhm.put(3, "3");
		lhm.put(2, "2");
		lhm.put(1, "1");
		lhm.put(6, "6");
		lhm.put(5, "5");
		lhm.put(4, "4");
		lhm.put(7,"7");
		lhm.forEach((a,b)->System.out.println(a+" "+b));
```

输出

`2 1 6 5 4 7`

我们可以看到头结点Entry 3=3被移除