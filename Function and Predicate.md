title: Predicate和Function
categories: java基础
tags: java8
---

### Predicate

Predicate接口为函数式接口

```
    /**
     * Evaluates this predicate on the given argument.
     *
     * @param t the input argument
     * @return {@code true} if the input argument matches the predicate,
     * otherwise {@code false}
     */
    boolean test(T t);
```
### Function

Function接口为函数式接口

```
    /**
     * Applies this function to the given argument.
     *
     * @param t the function argument
     * @return the function result
     */
    R apply(T t);
```

与Consumer接口类似，但Function的抽象方法有返回值

```
    /**
     * Returns a function that always returns its input argument.
     *
     * @param <T> the type of the input and output objects to the function
     * @return a function that always returns its input argument
     */
    static <T> Function<T, T> identity() {
        return t -> t;
    }
```

相当于用lambda实现apply抽象方法：u->u