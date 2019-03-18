title: consumer接口
categories: java基础
tags: java8
---

### consumer

```
@FunctionalInterface
public interface Consumer<T> {

    /**
     * Performs this operation on the given argument.
     *
     * @param t the input argument
     */
    void accept(T t);

    /**
     * Returns a composed {@code Consumer} that performs, in sequence, this
     * operation followed by the {@code after} operation. If performing either
     * operation throws an exception, it is relayed to the caller of the
     * composed operation.  If performing this operation throws an exception,
     * the {@code after} operation will not be performed.
     *
     * @param after the operation to perform after this operation
     * @return a composed {@code Consumer} that performs in sequence this
     * operation followed by the {@code after} operation
     * @throws NullPointerException if {@code after} is null
     */
    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```

consumer是一个函数式接口，可配合实现了Iterable的类和Iterator使用。

```
		List<student> a = new ArrayList<student>();
		a.add(new student(1, "aa"));
		a.add(new student(2, "bb"));
		a.add(new student(3, "cc"));
		a.forEach(stu->stu.id=4);
		a.forEach(stu->System.out::println)
```

`4 4 4`

List继承自Collection，而Collection又实现了Iterable接口，接口的foreach方法如下：

```
    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }
```

**所以，此接口是用来处理集合中所有元素，Performs this operation on the given argument.**



