title: jdk1.8新特性
categories: java基础
tags: java8
---

### 接口

1.可以定义静态方法(包括方法体)，实现接口后的类无法调用

2.可以定义default类型方法(包括方法体)，实现接口后可override，实现接口后的类的实例可以调用

### 函数式接口

如果一个接口定义个唯一一个抽象方法，那么这个接口就成为函数式接口，引入了一个新的Annotation：@FunctionalInterface。可以把他它放在一个接口前，表示这个接口是一个函数式接口。加上它的接口不会被编译。
**函数式接口中还可以额外定义Object类中的抽象方法，例如Comparator接口**

### lambda表达式

基本语法:(parameters) -> expression

```
    // 1. 不需要参数,返回值为 5  
    () -> 5  
      
    // 2. 接收一个参数(数字类型),返回其2倍的值  
    x -> 2 * x  
      
    // 3. 接受2个参数(数字),并返回他们的差值  
    (x, y) -> x – y  
      
    // 4. 接收2个int型整数,返回他们的和  
    (int x, int y) -> x + y  
      
    // 5. 接受一个 string 对象,并在控制台打印,不返回任何值(看起来像是返回void)  
    (String s) -> System.out.print(s)  
```

例子：
```
		Integer a[] = {1,2,3,4,5,6,7,8,9};
		List<Integer> list = Arrays.asList(a);
		list.forEach((Integer b)->System.out.println(b));
```

```
	String[] players = {"Rafael Nadal", "Novak Djokovic",   
    "Stanislas Wawrinka", "David Ferrer",  
    "Roger Federer", "Andy Murray",  
    "Tomas Berdych", "Juan Martin Del Potro",  
    "Richard Gasquet", "John Isner"}; 
	Arrays.sort(players, (String s1, String s2) -> (s1.compareTo(s2)));  
```

`Runnable r = () -> { System.out.println("Running!"); }`

### 捕获和非捕获的Lambda表达式

1.当Lambda表达式访问一个定义在Lambda表达式体外的非静态变量或者对象时，这个Lambda表达式称为“捕获的”。

2.在lambda表达式中访问外层作用域和老版本的匿名对象中的方式很相似。你可以直接访问标记了final的外层局部变量，或者实例的字段以及静态变量。

### Stream

这里的Stream可不是IO里面的OutputStream或InputStream之类，而是对java中Collection类聚合功能的增强。Stream API配合lamada表达式，极大的提升了编程效率和程序可读性。Stream中支持并行和串行两种聚合方式。

Stream接口中提供了更多对集合操作的api

#### map

```
    /**
     * Returns a stream consisting of the results of applying the given
     * function to the elements of this stream.
     *
     * <p>This is an <a href="package-summary.html#StreamOps">intermediate
     * operation</a>.
     *
     * @param <R> The element type of the new stream
     * @param mapper a <a href="package-summary.html#NonInterference">non-interfering</a>,
     *               <a href="package-summary.html#Statelessness">stateless</a>
     *               function to apply to each element
     * @return the new stream
     */
    <R> Stream<R> map(Function<? super T, ? extends R> mapper);
```

与forEach方法不同，Function中apply方法有返回值，而Consumer中的accept方法没有，map返回的Stream中的元素为Function中apply方法返回值，forEach只提供对元素操作，不返回Stream

```
		List<Integer> a = new ArrayList<Integer>();
		a.add(1);
		a.add(2);
		a.add(5);
		a = a.stream().map(i->++i).collect(Collectors.toList());
```
`2 3 6`
将集合中每个元素加1

```
		List<String> l = new ArrayList<String>();
		l.add("a");
		l.add("b");
		l.add("c");
		l = l.stream().map(String::toUpperCase).collect(Collectors.toList());
```

将集合中每个字符串变大写

#### filter

```
    /**
     * Returns a stream consisting of the elements of this stream that match
     * the given predicate.
     *
     * <p>This is an <a href="package-summary.html#StreamOps">intermediate
     * operation</a>.
     *
     * @param predicate a <a href="package-summary.html#NonInterference">non-interfering</a>,
     *                  <a href="package-summary.html#Statelessness">stateless</a>
     *                  predicate to apply to each element to determine if it
     *                  should be included
     * @return the new stream
     */
    Stream<T> filter(Predicate<? super T> predicate);
```

predicate接口返回值为布尔值，用来过滤不符合条件的集合中的元素

```
		List<Integer> a = new ArrayList<Integer>();
		a.add(1);
		a.add(2);
		a.add(5);
		List<Integer> t = new ArrayList<Integer>();
		t = a.stream().filter(i->i>2).collect(Collectors.toList());
		t.forEach(System.out::println);
```
`5`

作用是过滤了小于等于2的元素

#### mapToInt (mapToLong  mapToDouble)

```
    /**
     * Returns an {@code IntStream} consisting of the results of applying the
     * given function to the elements of this stream.
     *
     * <p>This is an <a href="package-summary.html#StreamOps">
     *     intermediate operation</a>.
     *
     * @param mapper a <a href="package-summary.html#NonInterference">non-interfering</a>,
     *               <a href="package-summary.html#Statelessness">stateless</a>
     *               function to apply to each element
     * @return the new stream
     */
    IntStream mapToInt(ToIntFunction<? super T> mapper);
```

```
public interface ToIntFunction<T> {

    /**
     * Applies this function to the given argument.
     *
     * @param value the function argument
     * @return the function result
     */
    int applyAsInt(T value);
}
```

实现ToIntFunction，将集合中元素转化为int类型，返回一个IntStream

#### flapMap

`    <R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper);`

与map方法的区别是，Function返回的是Stream，意味着Function方法中可以返回含有多个元素的Stream，最后会将所有Stream合并并返回一个Stream

例子：

```
		Stream<List<Integer>> integerListStream = Stream.of(
				  Arrays.asList(1, 4), 
				  Arrays.asList(3, 2), 
				  Arrays.asList(5)
				);
		integerListStream.flatMap(i->Stream.of(i.toArray())).forEach(System.out::println);
```

`1,4,3,2,5`

#### flapMapToInt(flapMapToDouble flapMapToLong)

```
    IntStream flatMapToInt(Function<? super T, ? extends IntStream> mapper);
```

与flapMap方法类似，Function返回的是IntStream，将多个IntStream合并然后返回一个IntStream

```
		Stream<List<String>> stringListStream = Stream.of(
				  Arrays.asList("1","4"), 
				  Arrays.asList("3","2"), 
				  Arrays.asList("5")
				);
		stringListStream.flatMapToInt(i->i.stream().mapToInt(Integer::valueOf)).forEach(System.out::println);
```

`1,4,3,2,5`

#### distinct

`    Stream<T> distinct();`

利用Object的equals方法去除重复元素

#### sorted

`Stream<T> sorted();`

对集合元素进行排序，如果元素没实现comparable接口，则抛出java.lang.ClassCastException异常

`    Stream<T> sorted(Comparator<? super T> comparator);`

元素没有实现comparable接口时，可以传入comparator实现类对集合元素进行排序

#### peek

`Stream<T> peek(Consumer<? super T> action);`

与forEach类似，不同之处在于peek继续返回Stream，而forEach无返回值

#### limit

`Stream<T> limit(long maxSize);`

根据maxSize截断Stream并返回长度为maxSize的新Stream，若maxSize为负数，抛出IllegalArgumentException

#### skip

` Stream<T> skip(long n);`

返回一个删除前n个元素的Stream

#### forEach

`    void forEach(Consumer<? super T> action);`

Consumer中accept方法对Stream中元素操作，无返回值(**遍历集合时依旧检测modCount==expectedModCount**)

#### forEachOrdered

`    void forEachOrdered(Consumer<? super T> action);`

按照集合中元素顺序来使用Consumer处理每个元素

#### toArray

`Object[] toArray();`

Stream转为数组

#### reduce

`T reduce(T identity, BinaryOperator<T> accumulator);`

accumulator函数式接口中，抽象方法为R apply(T t, U u);

先将identity作为第一个参数，Stream中第一个元素作为第二个参数，执行apply方法后，返回值作为第一个参数，下一个元素作为第二个参数,继续执行apply直到

sum，min，max, average,都是特殊的reduction

例如 sum = Stream.reduce(0,Integer::sum);

`    Optional<T> reduce(BinaryOperator<T> accumulator);`

同上，只是去掉identity，并且返回值为Optional类

#### collect

`    <R, A> R collect(Collector<? super T, A, R> collector);`

参数详见Collectors接口，Stream.collect(Collectors.tolist())将流转化回集合List

#### min

`    Optional<T> min(Comparator<? super T> comparator);`

返回Optional实例，值为传入的comparator排序后第一个元素

#### max 

`  Optional<T> max(Comparator<? super T> comparator);`

返回Optional实例，值为传入的comparator排序后最后一个元素

#### count

`    long count();`

返回流中元素个数

#### anyMatch

`    boolean anyMatch(Predicate<? super T> predicate);`

Predicate接口中boolean test(T t)处理所有元素，有一个返回值为true时，或者Stream为空时，返回true，否则返回false

#### allMatch

`boolean allMatch(Predicate<? super T> predicate);`

Predicate接口中boolean test(T t)处理所有元素，所有返回值都为true，或者Stream为空时返回true，否则返回false

#### noneMatch

与anyMatch相反

#### findFirst

`    Optional<T> findFirst();`

获取Stream第一个元素

#### findAny

`    Optional<T> findAny();`

返回任意元素

#### of

```
    /**
     * Returns a sequential {@code Stream} containing a single element.
     *
     * @param t the single element
     * @param <T> the type of stream elements
     * @return a singleton sequential stream
     */
    public static<T> Stream<T> of(T t) {
        return StreamSupport.stream(new Streams.StreamBuilderImpl<>(t), false);
    }
```

返回包含t元素的Stream

```
    /**
     * Returns a sequential ordered stream whose elements are the specified values.
     *
     * @param <T> the type of stream elements
     * @param values the elements of the new stream
     * @return the new stream
     */
    @SafeVarargs
    @SuppressWarnings("varargs") // Creating a stream from an array is safe
    public static<T> Stream<T> of(T... values) {
        return Arrays.stream(values);
    }
```

使用不定长度参数来创建Stream，参数可以为数组(可以Integer[]，不能是int[]）

### 方法引用与构造引用

1.方法引用，用来配合lambda表达式使用，使用情况有三种：
(1)对象::实例方法
(2)类::静态方法
(3)类::实例方法

前两种情况中，方法引用等同于提供方法参数的lambda表达式。比如：**Math::pow等同于(x, y) -> Math.pow(x, y)。**
第三种情况中，第一个参数会成为执行方法的对象。比如：**String::compareToIgnoreCase等同于(x, y) -> x.compareToIgnoreCase(y)。**

2.构造引用，配合supplier接口使用

```
@FunctionalInterface
public interface Supplier<T> {

    /**
     * Gets a result.
     *
     * @return a result
     */
    T get();
}
```

**()->new class()等同于class:new**