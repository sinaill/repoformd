title: ThreadLocal
categories: java基础
tags: 
	- 多线程
---


### 什么是ThreadLocal

`ThreadLoca`l一般称为线程本地变量，它是一种特殊的线程绑定机制，将变量与线程绑定在一起，为每一个线程维护一个独立的变量副本。通过`ThreadLocal`可以将对象的可见范围限制在同一个线程内。

### 查看源码

先查看`get`方法

```
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

查看`getMap`方法

```
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
查看ThreadLocalMap类,为键值对的数据结构，另外Thread类中定义了一个ThreadLocalMap类的引用。get方法中先获取线程t，再获取线程t中的ThreadLocalMap实例，如果不为空，设置一个新键值对，key为当前ThreadLocal实例，value为传入的参数value。如果为空，进入createMap方法

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
```

给当前线程t的`ThreadLocalMap`类的引用传入新实例，`key`为当前`ThreadLocal`，值为传入的参数`firstValue`

查看`get`方法

```
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

先获取当前线程t，再获取t中的`ThreadLocalMap`实例`map`，如果`map`不为空

```
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

通过传入的`ThreadLocal`实例为`key`值获取对应键值对`Entry`

```
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

`Entry`中的`value`就是键值对中与当前`ThreadLocal`实例对应的`value`键值，`map`为空时，即调用`get`方法之前没有调用过`set`方法，`get`返回默认值(null)，可覆写

```
protected T initialValue() {
    return null;
}
```

### 总结

`Thread`类中存有`ThreadLocalMap`的引用，`ThreadLocalMap`类的定义在`ThreadLocal`中。

调用`ThreadLocal`的`set`方法时，先获取当前线程，查看线程中`ThreadLocalMap`引用为不为空，若为空，为当前线程创建`ThreadLocalMap`实例，传入键值对`key`为当前`ThreadLocal`实例，`value`为`set`方法传入的参数值。若不为空，为`ThreadLocalMap`实例添加新键值对，`key`为当前`ThreadLocal`实例，`value`为`set`方法传入的参数值。

调用`ThreadLocal`的`get`方法时，如果之前为调用过`set`方法或`ThreadLocalMap`实例中`key`没有与当前`ThreadLocal`实例对应的，则返回一个默认值`null`。照旧先获取当前线程，获取线程中`ThreadLocalMap`实例，再根据当前`ThreadLocal`实例获取对应的键值对`Entry`，从而获取线程中存储的变量副本。

为什么不用一个`map Entry<Thread,value>`来存储所有线程和线程变量副本，`map`过大会造成哈希冲突，造成性能变差。