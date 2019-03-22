title: 接口中的default方法
categories: java基础
tags: 
	- java8
---


### 特性

java接口中的default方法是在java 8之后引入的，即在不破坏java现有实现架构的情况下能往接口里增加新方法，这个特征又叫做虚拟扩展方法（Virtual extension methods），通常也称之为 defender 方法，它目前可以添加到接口中，为声明的方法提供默认的实现，子类可以直接调用或者复写该方法。作用为优化接口的同时，避免跟现有实现架构的兼容问题。

### 例子

```
List<Integer> list = new ArrayList<Integer>();
list.add(1);
list.add(2);
list.forEach(System.out::println);
```

java8之后list多了一个forEach方法来方便我们遍历集合，向上查找父类和实现的接口发现Iterable接口中多了一个default方法

```
/**
 * Performs the given action for each element of the {@code Iterable}
 * until all elements have been processed or the action throws an
 * exception.  Unless otherwise specified by the implementing class,
 * actions are performed in the order of iteration (if an iteration order
 * is specified).  Exceptions thrown by the action are relayed to the
 * caller.
 *
 * @implSpec
 * <p>The default implementation behaves as if:
 * <pre>{@code
 *     for (T t : this)
 *         action.accept(t);
 * }</pre>
 *
 * @param action The action to be performed for each element
 * @throws NullPointerException if the specified action is null
 * @since 1.8
 */
default void forEach(Consumer<? super T> action) {
    Objects.requireNonNull(action);
    for (T t : this) {
        action.accept(t);
    }
}
```

与此相同的还有Collection接口新增的stream方法，用来帮助我们更方便地处理集合元素

```
/**
 * Returns a sequential {@code Stream} with this collection as its source.
 *
 * <p>This method should be overridden when the {@link #spliterator()}
 * method cannot return a spliterator that is {@code IMMUTABLE},
 * {@code CONCURRENT}, or <em>late-binding</em>. (See {@link #spliterator()}
 * for details.)
 *
 * @implSpec
 * The default implementation creates a sequential {@code Stream} from the
 * collection's {@code Spliterator}.
 *
 * @return a sequential {@code Stream} over the elements in this collection
 * @since 1.8
 */
default Stream<E> stream() {
    return StreamSupport.stream(spliterator(), false);
}
```