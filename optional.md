title: option接口
categories: java基础
tags: java8
---

Optional是Java8提供的为了解决null安全问题的一个API，与String类似，也是final类型

### 成员变量

```
    /**
     * Common instance for {@code empty()}.
     */
    private static final Optional<?> EMPTY = new Optional<>();
```
值为空的Optional共享此实例

```
    /**
     * If non-null, the value; if null, indicates no value is present
     */
    private final T value;
```

Optional类中用来存储值得变量

### 成员方法

#### empty

```
    public static<T> Optional<T> empty() {
        @SuppressWarnings("unchecked")
        Optional<T> t = (Optional<T>) EMPTY;
        return t;
    }
```

返回类中定义好的空的Optional实例

#### of

```
    public static <T> Optional<T> of(T value) {
        return new Optional<>(value);
    }
```

指定value返回一个Optional实例，若value为空，抛出空指针异常

#### ofNullable

```
    public static <T> Optional<T> ofNullable(T value) {
        return value == null ? empty() : of(value);
    }
```

指定value返回一个Optional实例，若value为空，则返回类中定义好的空的optional实例

#### get

```
    public T get() {
        if (value == null) {
            throw new NoSuchElementException("No value present");
        }
        return value;
    }
```

如果 Optional中存在值，则返回值，否则抛出 NoSuchElementException 

#### isPresent

```
    public boolean isPresent() {
        return value != null;
    }
```

值不为空时返回true，否则返回false

#### ifPresent

```
    public void ifPresent(Consumer<? super T> consumer) {
        if (value != null)
            consumer.accept(value);
    }
```

使用consumer处理value，若consumer为空，抛出空指针异常

#### filter

```
    public Optional<T> filter(Predicate<? super T> predicate) {
        Objects.requireNonNull(predicate);
        if (!isPresent())
            return this;
        else
            return predicate.test(value) ? this : empty();
    }
```

predicate接口处理value，返回true时，filter方法返回自身，返回false时，filter返回Optional中定义的空实例

#### map

```
    public<U> Optional<U> map(Function<? super T, ? extends U> mapper) {
        Objects.requireNonNull(mapper);
        if (!isPresent())
            return empty();
        else {
            return Optional.ofNullable(mapper.apply(value));
        }
    }
```

Function接口处理value，value为空时，返回Optional中定义的空实例。value不为空时，当Function中apply返回值为空时，也是返回空实例，否则以apply返回值创建Optional实例

```
    public<U> Optional<U> flatMap(Function<? super T, Optional<U>> mapper) {
        Objects.requireNonNull(mapper);
        if (!isPresent())
            return empty();
        else {
            return Objects.requireNonNull(mapper.apply(value));
        }
    }
```

与map方法类似，区别在于Function返回值，apply方法直接返回Optional实例

#### orElse

```
    public T orElse(T other) {
        return value != null ? value : other;
    }
```

value为空则返回other，否则返回value

#### orElseGet

```
    public T orElseGet(Supplier<? extends T> other) {
        return value != null ? value : other.get();
    }
```

与orElse方法相同，参数变为Supplier接口实现。Optional.orElse(new class)与Optional.orElseGet(class:new)区别在于，orElseGet在value为空时才创建对象，而orElse一开始就创建了对象

#### orElseThrow

```
    public <X extends Throwable> T orElseThrow(Supplier<? extends X> exceptionSupplier) throws X {
        if (value != null) {
            return value;
        } else {
            throw exceptionSupplier.get();
        }
    }
```

value为空时抛出自定义异常，否则返回value，value和Supplier的get方法返回值都为空时抛出空指针异常