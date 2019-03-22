title: fail-fast与fail-safe
categories: java基础
tags:
	- 集合
---

### fail-fast

java.util 包中的集合类都返回 fail-fast 迭代器，意味着在使用iterator遍历集合时，如果改变集合结构，将会抛出ConcurrentModificationException

下面以ArrayList为例分析

ArrayList继承于AbstractList，而AbstractList中定义了一个protected transient int modCount = 0，那么这个变量有什么作用呢

我们看ArrayList中实现了Iterator接口的私有内部类ltr，有个变量接收了这个modCount

```
int expectedModCount = modCount;
```

当我们获取集合的迭代器对象Iterator时，这个动作发生

```
    public Iterator<E> iterator() {
        return new Itr();
    }
```

然后看看改变集合结构的几个方法，例如add,remove

add方法：

```
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

```

remove方法：

```
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```

**我们发现在这些操作中都会将集合自身变量modCount自增1，会造成什么影响呢，接着往下看**

迭代器Iterator中的next()方法中调用了checkForComodification()

查看方法checkForComodification：

```
		final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
```

可以看到我们在获取下个元素时，Iterator会先比较modCount和expectedModCount，如果我们在迭代器遍历时调用了add或者remove等改变集合结构的方法时，modCount自增了，而expectedModCount不变，所以在此处会抛出ConcurrentModificationException

但是如果是在我们自己遍历集合时想修改集合结构，而且不是多线程环境中的话，我们可以看Iterator中给我们提供的方法，在ArrayList中的内部类Iterator中给我们提供了一个remove方法

```
        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
```

我们可以看到在移除元素后，重新对expectedModCount赋值，所以checkForComodification()不会抛出异常

ArrayList还提供了一个更丰富的ListIterator内部类，是一个双向迭代器，可以双向遍历，在这里比Iterator多了add方法

```
    public void add(E e) {
        checkForComodification();

        try {
            int i = cursor;
            ArrayList.this.add(i, e);
            cursor = i + 1;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }
}
```

依旧是处理了expectedModCount = modCount。

### fail-safe

以ArrayList的替代类CopyOnWriteArrayList为例分析，CopyOnWriteArrayList为何物？ArrayList 的一个线程安全的变体，其中所有可变操作（add、set 等等）都是通过对底层数组进行一次新的复制来实现的。 该类产生的开销比较大，但是在两种情况下，它非常适合使用。
1. 在不能或不想进行同步遍历，但又需要从并发线程中排除冲突时。
2. 当遍历操作的数量大大超过可变操作的数量时。

查看迭代器

```
public E next() {
    if (! hasNext())
        throw new NoSuchElementException();
    return (E) snapshot[cursor++];
}
```

没有像ArrayList的迭代器一样，它不需要检查expectedModCount = modCount

再看add方法

```
public boolean add(E paramE) {    
        ReentrantLock localReentrantLock = this.lock;    
        localReentrantLock.lock();    
        try {    
            Object[] arrayOfObject1 = getArray();    
            int i = arrayOfObject1.length;    
            Object[] arrayOfObject2 = Arrays.copyOf(arrayOfObject1, i + 1);    
            arrayOfObject2[i] = paramE;    
            setArray(arrayOfObject2);    
            int j = 1;    
            return j;    
        } finally {    
            localReentrantLock.unlock();    
        }    
    }    
    final void setArray(Object[] paramArrayOfObject) {    
        this.array = paramArrayOfObject;    
    }  
```

看关键代码：

```
            Object[] arrayOfObject2 = Arrays.copyOf(arrayOfObject1, i + 1);    
            arrayOfObject2[i] = paramE;    
            setArray(arrayOfObject2)
```

所以在多线程环境时，可以使用CopyOnWriterArrayList来替代，它们的原理为先复制原有集合，然后在复制集合上作修改，然后将原引用指向复制修改后的集合，而迭代器遍历集合是先接收CopyOnWriterArrayList中存储元素的数组，然后对其遍历。这样就达到了遍历和修改集合结构分别在不同的存储数组中。

