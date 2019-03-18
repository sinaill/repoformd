title: HashMap与HashTable
categories: java基础
tags:
	- 集合
---

1.HashTable继承自Dictionary,而HashMap继承自AbstractMap

2.HashTable方法带有synchronized关键字，意味着HashTable是同步的，而HashMap是非同步的

3.HashTable中提供了一个类似Iterator的Enumeration(fail-safe),用来遍历自身，而HashMap中无迭代器

```
    public synchronized Enumeration<K> keys() {
        return this.<K>getEnumeration(KEYS);
    }

	public synchronized Enumeration<V> elements() {
        return this.<V>getEnumeration(VALUES);
    }
```

以keys为例

```
	Hashtable<String,String> a = new Hashtable<String,String>();
	a.put("1", "1");
	a.put("2", "2");
	a.put("3", "3");
	Enumeration<String> e = a.keys();
	while(e.hasMoreElements()){
		System.out.println(e.nextElement());
	}
	}
```

`3 2 1`

结果是倒序的，我们看下源码

```
		public T nextElement() {
            Entry<?,?> et = entry;
            int i = index;//index=table.length
            Entry<?,?>[] t = table;
            /* Use locals for faster loop iteration */
            while (et == null && i > 0) {
                et = t[--i];
            }
            entry = et;
            index = i;
            if (et != null) {
                Entry<?,?> e = lastReturned = entry;
                entry = e.next;
                return type == KEYS ? (T)e.key : (type == VALUES ? (T)e.value : (T)e);
            }
            throw new NoSuchElementException("Hashtable Enumerator");
        }
```

et = t[--i];所以结果是倒序的，但是我们还可以发现，与其他集合的Iterator不同，Enumerator的nextElement方法中没有检测modCount==expectedModCount，而HashMap中EntrySet的Iterator有检测这个。

```
	Hashtable<String,String> a = new Hashtable<String,String>();
	a.put("1", "1");
	a.put("2", "2");
	a.put("3", "3");
	a.put("4", "4");
	a.put("5", "5");
	a.put("6", "6");
	Enumeration<String> e = a.keys();
	String b = null;
	while(e.hasMoreElements()){
		b = e.nextElement();
		if(b.equals("3")){
			a.remove(b);
		}
	}
	a.forEach((c,d)->System.out.println(c+" "+d));
	}
```

输出

`1 1 2 2 4 4 5 5 6 6 `

确实没有抛出ConcurrentModificationException，Enumerator是一个fail-safe迭代器

4.HashMap允许null作为key或value

5.HashMap默认容量16，Hashtable默认容量11

