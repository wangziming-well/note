# Stream概述

JavaSE8提供了Stream流库，以更高效、可读性更强的方式计算/处理集合。

在Stream之前，我们通常通过迭代遍历集合，在集合的每个元素上执行某项操作以处理集合

现在可以使用Stream库的api来替代上述过程以处理结合，并且Stream库底层可以对这个计算过程进行优化，比如使用多线程来完成数据的迭代处理并将结果合并。

例如，某个场景下，需要计算某个单词集合中长度超过12的某个元素；如果通过迭代处理，可以是：

~~~java
int count = 0;
for (String w:words){
    if(w.length() >12) count++;
}
~~~

而使用Stream库可以这样处理：

~~~java
count =(int) words.stream()
    .filter(w -> w.length() > 12)
    .count();
~~~

可以看出，流的版本比迭代循环的版本更加易读，因为我们不需要像阅读迭代代码一样，需要了解在整个迭代中对每个元素做了哪些操作才能明白这段代码的作用，只需要了解流调用的那些函数，这些函数的函数名就能告诉我们这段代码的作用。

而且如果不是终结方法，那么流的方法的返回值大部分依然是流对象，所以使用Stream API时，通常可以连续调用相关的方法；这被称为流式编程；这种编程思想在其他许多类库/框架中都有应用。它可以极大的方便客户端开发人员的编码。

流看起来和集合类似，都可以让我们转换和获取数据，但是实际上他们存在显著差异：

集合和流虽然表面上有一些相似之处，但它们有不同的目标。集合主要关注对其元素的有效管理和访问。相比之下，流不提供直接访问或操作其元素的方法，而是关注声明性地描述其源和将在该源上聚合执行的计算操作。

* 流不存储元素，流操作的元素可能存储在导出该流的集合(数据源)，或者是按需生成的
* 流的操作不会修改其数据源。通常会生成一个新的流，或者收集为新的集合
* 流的操作尽可能惰性执行：在达到终止条件前不会处理元素，达到终止条件后逐个处理每个元素。如果遇到短路操作，那么只要满足所有条件，流处理就会终止。

==todo==流的惰性执行

# BaseStream

Stream接口继承自BaseStream，先了解以下它：

~~~java
public interface BaseStream<T, S extends BaseStream<T, S>>
        extends AutoCloseable {
    Iterator<T> iterator();
    //返回此流的元素的迭代器。
    Spliterator<T> spliterator();
    //返回此流的元素的拆分器
    boolean isParallel();
	//返回如果要执行终端操作，此流是否将并行执行。在调用终端流操作方法后调用此方法可能会产生不可预知的结果。
    S sequential();
	//返回顺序的等效流。如果流已经是顺序的，或者因为基础流状态已修改为顺序，则会返回自身。
    S parallel();
	//返回并行的等效流。若流已经并行，或者因为基础流状态已修改为并行，则会返回自身。
    S unordered();
	//返回无序的等效流。若流已无序，或者因为基础流状态已修改为无序，则会返回自身
    S onClose(Runnable closeHandler);
	//返回具有附加关闭处理程序的等效流。关闭处理程序在流上调用 close（） 方法时运行，并按添加顺序执行。运行所有关闭处理程序，即使较早的关闭处理程序引发异常也是如此。如果任何关闭处理程序引发异常，则抛出的第一个异常将被中继到 close（） 的调用方，并将任何剩余的异常作为抑制异常添加到该异常中（除非剩余异常之一与第一个异常相同，因为异常无法抑制自身。可能会自己返回。
    @Override
    void close();
    //关闭此流，导致调用此流管道的所有关闭处理程序。
}
~~~





# Stream相关的函数式接口

使用Stream需要调用方自己设计如何对元素的操作和判断，所以stream方法入参常常使用函数式接口，以方便调用方使用 lambda表达式传入元素的处理逻辑，以下是StreamAPI会用到的函数式接口

## Predicate

它的定义如下：

~~~java
@FunctionalInterface
public interface Predicate<T> {

    boolean test(T t);

    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }

    default Predicate<T> negate() {
        return (t) -> !test(t);
    }

    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }

    static <T> Predicate<T> isEqual(Object targetRef) {
        return (null == targetRef)
                ? Objects::isNull
                : object -> targetRef.equals(object);
    }
}
~~~

核心方法是`test()`，它返回给定参数的谓词

除此之外，它定义了三个默认函数以方便谓词之间进行逻辑运算`and(),or(),negate()`

和一个静态方法，`isEqual()`，它返回的谓词用以判断两个参数是否相等

它还有几个类似的方法，为了入参能兼容基本数据类型：

### 使用例

`Stream`的`filter()`方法接受一个谓词，对所有元素进行谓词判断，过滤保留谓词判断结果为true的元素

使用Stream的filter方法获取集中中指定范围的元素：我们可以直接使用2次filter：

~~~java
List<Integer> list = Arrays.asList(1, 2, 3,4,5,6,7,8);
List<Integer> collect = list.stream()
        .filter(i -> i > 4)
        .filter(i -> i < 7)
        .collect(Collectors.toList());
~~~

也可以使用谓词的逻辑运算：

定义一个PredicateUtils:

~~~java
public class PU {
    private PredicateUtils(){};
    public static Predicate<Integer> biggerThen(int num){
        return i -> i > num;
    }
    public static Predicate<Integer> smallerThen(int num){
        return i -> i < num;
    }
}
~~~

然后使用一个filter：

~~~java
List<Integer> collect = list.stream()
        .filter(PU.biggerThen(3).and(PU.smallerThen(6)))
        .collect(Collectors.toList());
~~~

## Supplier

表示结果的提供者

```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

## Consumer

表示接受单个输入参数且不返回结果的操作

```java
@FunctionalInterface
public interface Consumer<T> {

    void accept(T t);
	//返回一个组合的Consumer，该Consumer按顺序执行此操作，然后执行after操作
    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```

## Function

表示接受一个参数并产生结果的函数。

~~~java
@FunctionalInterface
public interface Function<T, R> {

    R apply(T t);
	//返回一个组合函数，该函数首先将before函数应用于其输入，然后将此函数应用于结果。
    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }
	//返回一个组合函数，该函数首先将此函数应用于其输入，然后将after函数应用于结果。
    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }
	//返回一个总是返回其输入参数的函数
    static <T> Function<T, T> identity() {
        return t -> t;
    }
}
~~~

## BiFunction

表示接受两个参数并产生结果的函数。这是函数的二元特化。

```java
@FunctionalInterface
public interface BiFunction<T, U, R> {

    R apply(T t, U u);

    default <V> BiFunction<T, U, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t, U u) -> after.apply(apply(t, u));
    }
}
```

它有一个子接口`BinaryOperator`，表示入参和返回值类型相同的`BiFunction`：

~~~java
@FunctionalInterface
public interface BinaryOperator<T> extends BiFunction<T,T,T> {

    public static <T> BinaryOperator<T> minBy(Comparator<? super T> comparator) {
        Objects.requireNonNull(comparator);
        return (a, b) -> comparator.compare(a, b) <= 0 ? a : b;
    }

    public static <T> BinaryOperator<T> maxBy(Comparator<? super T> comparator) {
        Objects.requireNonNull(comparator);
        return (a, b) -> comparator.compare(a, b) >= 0 ? a : b;
    }
}
~~~





# 获取Stream对象

Collection接口定义，下面方法返回以当前容器为源数据的流：

~~~java
default Stream<E> stream() {
    return StreamSupport.stream(spliterator(), false);
}
default Stream<E> parallelStream() {
    return StreamSupport.stream(spliterator(), true);
}
~~~

Arrays工具类，获取以入参数组为源数据的流：

~~~java
Stream<T> stream(T[] array);
Stream<T> stream(T[] array, int startInclusive, int endExclusive);
IntStream stream(int[] array);
IntStream stream(int[] array, int startInclusive, int endExclusive);
......
~~~

Stream接口中提供静态方法：

~~~java
Stream<T> of(T... values);
~~~







# Stream延迟方法

延迟方法表示对流中的元素进行过滤/映射/处理，返回新的流对象；但对元素的过滤/映射/处理不是在调用延迟方法时，而是在调用终结方法时，会对每个元素作用进行当前流经过的所有延迟方法

即调用延迟方法时不会执行方法操作，而是只返回一个新的流；在调用终端方法后，才会遍历流中元素，对每个元素依次执行延迟方法。

## filter()

~~~java
Stream<T> filter(Predicate<? super T> predicate);
//返回由此流中与给定谓词匹配的元素组成的流。(元素对应谓词结果为true的)
~~~

演示：

~~~java
List<Integer> collect = Stream.of(1, 2, 3, 4, 5, 6)
        .filter(i -> i > 3)
        .collect(Collectors.toList());
System.out.println(collect);//[4, 5, 6]
~~~

## map()

~~~java
Stream<R> map(Function<? super T, ? extends R> mapper);
IntStream mapToInt(ToIntFunction<? super T> mapper);
LongStream mapToLong(ToLongFunction<? super T> mapper);
DoubleStream mapToDouble(ToDoubleFunction<? super T> mapper);
//返回由将给定函数应用于this的元素的结果组成的流(对当前流的所用的元素应用mapper，其结果组成的新流)
~~~

演示：

~~~java
List<Integer> collect = Stream.of(1, 2, 3, 4, 5, 6)
        .map(i -> i + 1)
        .collect(Collectors.toList());
System.out.println(collect);//[2, 3, 4, 5, 6, 7]
~~~

## flatMap()

~~~java
Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper);
IntStream flatMapToInt(Function<? super T, ? extends IntStream> mapper);
LongStream flatMapToLong(Function<? super T, ? extends LongStream> mapper);
DoubleStream flatMapToDouble(Function<? super T, ? extends DoubleStream> mapper);
//返回一个流，其中包含将此流的每个元素替换为通过对每个元素应用所提供的映射函数生成的映射流的内容的结果。入参mapper的返回值必须是一个流;在处理嵌套容器时很好用
~~~

演示

~~~java
ArrayList<List<Integer>> lists = new ArrayList<>();
for (int i = 0; i < 9; i+=3) {
    lists.add(Arrays.asList(i,i+1,i+2));
}
List<Integer> collect = lists.stream()
        .flatMap(Collection::stream)
        .collect(Collectors.toList());
System.out.println(collect);//[0, 1, 2, 3, 4, 5, 6, 7, 8]
~~~

## distinct()

~~~java
Stream<T> distinct();
//返回由该流的不同元素组成的流(根据Object.equals(Object));即去掉该流中相同的元素
~~~

演示：

~~~java
List<Integer> collect = Stream.of(1, 1, 2, 2, 3, 3)
        .distinct()
        .collect(Collectors.toList());
System.out.println(collect);//[1, 2, 3]
~~~

## sorted()

~~~java
Stream<T> sorted();
//返回由此流的元素组成的流，按自然顺序排序。如果此流的元素不是Comparable的，则在执行终端操作时可能会抛出java.lang.ClassCastException。
Stream<T> sorted(Comparator<? super T> comparator);
//返回由此流的元素组成的流，根据提供的比较器排序。
~~~

演示：

~~~java
List<Integer> collect = Stream.of(3, 1, 2)
        .sorted()
        .collect(Collectors.toList());
System.out.println(collect);//[1, 2, 3]
~~~

## peek()

~~~java
Stream<T> peek(Consumer<? super T> action);
//返回由此流的元素组成的流，并在从结果流中消费元素时对每个元素执行所提供的操作。
~~~

演示：

~~~java
Stream.of("one", "two")
        .peek(e -> System.out.println("Original value: " + e))
        .map(String::toUpperCase)
        .peek(e -> System.out.println("Mapped value: " + e))
        .collect(Collectors.toList());
/**
Original value: one
Mapped value: ONE
Original value: two
Mapped value: TWO
*/
~~~

并且由该程序的输出顺序，我们也可以看出，Stream的延迟方法确实是惰性的；并且在终结时执行顺序也是一个元素执行完所有延迟方法，才执行下一个元素；而不是所有元素执行完一个延迟方法，再执行下一个延迟方法的。

## limit()

~~~java
Stream<T> limit(long maxSize);
//在截断为长度不超过maxSize后，返回由此流的元素组成的流
~~~

演示：

~~~java
List<Integer> collect = Stream.of(1, 2, 3, 4, 5, 6)
        .limit(3)
        .collect(Collectors.toList());
System.out.println(collect);//[1, 2, 3]
~~~

## skip()

~~~java
Stream<T> skip(long n);
//在丢弃流的前n个元素后，返回由该流的剩余元素组成的流。如果此流包含少于n个元素，则返回空流。
~~~

演示：

~~~java
List<Integer> collect = Stream.of(1, 2, 3, 4, 5, 6)
        .skip(3)
        .collect(Collectors.toList());
System.out.println(collect); //[4, 5, 6]
~~~

# Stream终结方法

终结方法会对流中的元素进行最终处理，返回一个新的集合或者其他结果；同一个流最多只能调用一次终结方法；

在介绍Stream的终结方法之前，需要先介绍一个类：Optional，用于解决NullPointerException问题和使代码更加健壮。

他是一个容器对象，只包含一个元素。它可以包含也可以不包含非空值。如果存在一个值，isPresent()将返回true，而get()将返回该值。

并且它同样有`map()`、`filter()`、`flatMap()`方法，作用和Stream的一样

一些返回单个元素的终结方法会返回Optional

## 处理元素

### forEach()

~~~java
void forEach(Consumer<? super T> action);
//对该流的每个元素执行一个操作
~~~

演示

~~~java
Stream.of(1,2,3,4,5,6)
        .forEach(System.out::print);
//123456
~~~

### forEachOrdered()

~~~java
void forEachOrdered(Consumer<? super T> action);
//如果流具有已定义的相遇顺序，则按照流的相遇顺序对该流的每个元素执行操作。对并发流也是同样的
~~~

演示

对并发流，`foreach()`方法不能保证调用顺序：

~~~java
Stream.of(1,2,3,4,5,6)
        .parallel()
        .forEach(System.out::print);
//456321
~~~

但`forEachOrdered()`方法可以，但是这样会大大损耗并发流的性能

~~~java
Stream.of(1,2,3,4,5,6)
        .parallel()
        .forEachOrdered(System.out::print);
~~~

## 收集元素

### toArray()

~~~java
Object[] toArray();
//返回包含此流元素的数组
<A> A[] toArray(IntFunction<A[]> generator);
//通过给定的generator，生成一个A[]数组;IntFunction<A[]>入参为数组长度，返回值是数组
~~~

演示：

~~~java
Object[] objects = Stream.of(1, 2, 3, 4, 5, 6)
        .toArray();
System.out.println(Arrays.toString(objects));//[1, 2, 3, 4, 5, 6]
Integer[] integers = Stream.of(1, 2, 3, 4, 5, 6)
        .toArray(i -> new Integer[i]);
System.out.println(Arrays.toString(integers));//[1, 2, 3, 4, 5, 6]
~~~

### reduce()

~~~java
T reduce(T identity, BinaryOperator<T> accumulator);
//使用提供的标识值和关联累加函数对该流的元素执行约简，并返回约简后的值
//相当于执行了以下操作：先让identity和第一个元素作为入参调用BinaryOperator.apply()获取结果，然后讲上一个结果和第二个元素作为入参调用BinaryOperator.apply()获取结果，依次类推;相等于以下代码
T result = identity;
for (T element : thisStream)      
    result = accumulator.apply(result, element)  
return result;
//-----------------------------------------------
Optional<T> reduce(BinaryOperator<T> accumulator);
//使用关联累加函数对该流的元素执行约简操作，并返回一个描述约简值的Optional(如果有的话)
//相当于执行了以下操作：让第一个元素和第二个元素作为入参调用BinaryOperator.apply()获取结果，然后将上一个结果和第三个元素作为入参调用BinaryOperator.apply()获取结果，以此类推;相当于以下代码:
 boolean foundAny = false;  
T result = null;  
for (T element : this stream) {
    if (!foundAny) {
        foundAny = true;
        result = element;
    } else
        result = accumulator.apply(result, element);  
}  
return foundAny ? Optional.of(result) : Optional.empty();
~~~

演示：

~~~java
Integer reduce = Stream.of(1, 2, 3, 4, 5, 6)
        .reduce(-21, (i, j) -> i + j);
System.out.println(reduce); // 0
~~~

#### combiner

~~~java
<U> U reduce(U identity,BiFunction<U, ? super T, U> accumulator,BinaryOperator<U> combiner);
//如果想要将流中的元素reduce为另外一种类型，那么就必须使用带combiner参数的reduce()方法;用于合并
//combiner组合器功能必须与accumulator累加器功能兼容，这意味着对所有的u和t,必须满足
 combiner.apply(u, accumulator.apply(identity, t)) == accumulator.apply(u, t);
~~~

比如说要获取字符串元素中最大的字符长度，可以使用：

~~~java
Integer reduce = Stream.of("one", "tow", "three", "four")
        .reduce(0, (l, str) -> l > str.length() ? l : str.length(), (i, j) -> i > j ? i : j);
System.out.println(reduce);//5
~~~

它的大致过程如图：

![image-20230629163539416](https://gitee.com/wangziming707/note-pic/raw/master/img/reduce%E8%BF%87%E7%A8%8B.png)

从上述过程来看，combiner好像不是必须的，我们只使用accumulator依次累加获取最大的字符长度；但这样的过程只能在串行时使用；并行时无法只使用accumulator；所以该接口设计必须有combiner，保证串行流和并发流操作的一致性；

### collect()

`collect()`方法一般用来将流转换为集合

~~~java
<R> R collect(Supplier<R> supplier,BiConsumer<R, ? super T> accumulator,BiConsumer<R, R> combiner);
//对该流的元素执行可变约简操作。可变约简是指被约简后的值是一个可变的结果容器，如ArrayList，通过更新结果的状态而不是替换结果来合并元素。
/**
supplier：创建新结果容器的函数。由累加器(accumulator)和组合器(combiner)填充，最后由collect()方法返回。于并行执行，这个函数可能被调用多次，并且每次都必须返回一个新的值
accumulator：它在结果中加入了额外的元素。它的入参时supplier的结果和流中元素
combiner：它组合了两个必须与累加器兼容的值。组合器(combiner)工作在并行处理中。它的入参时两个supplier的结果
*/
//这产生的结果相当于:
R result = supplier.get();  
for (T element : this stream)      
    accumulator.accept(result, element);  
return result;
//与reduce(Object, BinaryOperator)类似，收集操作可以并行化，而不需要额外的同步。
~~~

演示：

~~~java
ArrayList<String> arrayList = Stream.of("one", "three", "tow")
        .collect(() -> new ArrayList<String>(), (list, str) -> list.add(str),
                (list, list2) -> list.addAll(list2));
System.out.println(arrayList);//[one, three, tow]
~~~

也可以简化成：

~~~java
ArrayList<String> arrayList = Stream.of("one", "three", "tow")
        .collect(ArrayList::new, ArrayList::add, ArrayList::addAll);
System.out.println(arrayList);//[one, three, tow]
~~~

### findFirst()/findAny()

~~~java
Optional<T> findFirst();
//返回描述此流的第一个元素的 Optional,如果流为空则返回空的Optional。如果流没有遭遇顺序，则可以返回任何元素。
Optional<T> findAny();
//返回描述此流的某些元素的 Optional;如果流为空，则返回空的 Optional。
~~~

## 计算元素

有如下方法对流中元素进行一些计算并返回结果：

~~~Java
Optional<T> min(Comparator<? super T> comparator);
Optional<T> max(Comparator<? super T> comparator);
long count();
boolean anyMatch(Predicate<? super T> predicate);
boolean allMatch(Predicate<? super T> predicate);
boolean noneMatch(Predicate<? super T> predicate);
~~~

使用例：

~~~java
Stream<Integer> stream = Stream.of(1, 2, 3, 4, 5);
System.out.println(stream.count());//5
stream = Stream.of(1, 2, 3, 4, 5);
System.out.println(stream.min(Comparator.comparingInt(i -> i)).get());//1
stream = Stream.of(1, 2, 3, 4, 5);
System.out.println(stream.allMatch(i -> i < 3));//false
stream = Stream.of(1, 2, 3, 4, 5);
System.out.println(stream.anyMatch(i -> i < 3));//true
stream = Stream.of(1, 2, 3, 4, 5);
System.out.println(stream.noneMatch(i -> i < 0));//true
~~~

# Stream静态方法

Stream类提供的静态方法基本是用来获取Stream对象的

~~~java
<T> Stream<T> of(T t);
//返回包含单个元素的顺序流
<T> Stream<T> of(T... values);
//返回其元素为指定值的顺序有序流
<T> Builder<T> builder() ;
//返回流的生成器
<T> Stream<T> empty();
//返回一个空的序列 Stream
Stream<T> iterate(final T seed, final UnaryOperator<T> f);
//返回由函数 f 迭代应用于初始元素种子生成的无限顺序有序流，生成由seed、f(seed)、f(f(seed))等组成的流。流中的第一个元素(位置 0)将是提供的种子。对于 n > 0，位置 n 的元素将是将函数 f 应用于位置 n - 1 的元素的结果。
<T> Stream<T> generate(Supplier<T> s);
//返回无限顺序无序流，其中每个元素都由提供的供应商生成。这适用于生成恒定流、随机元素流等。
<T> Stream<T> concat(Stream<? extends T> a, Stream<? extends T> b);
//创建一个延迟串联的流，其元素是第一个流的所有元素，后跟第二个流的所有元素。如果两个输入流都按顺序排序，则生成的流是有序的，如果其中一个输入流是并行的，则结果流是并行的。当生成的流关闭时，将调用两个输入流的关闭处理程序。
~~~

使用例：

~~~java
//生成一个随机数流
Random random = new Random();
Stream.generate(() -> random.nextInt(100))
    .limit(10)
    .forEach(i-> System.out.print(i+","));//61,3,12,38,98,9,97,14,42,68,
//生成一个斐波那契数列
int[] before  = {1};
Stream.iterate(1,i ->{int result = before[0] + i;  before[0] = i; return result;} )
    .limit(100)
    .forEach(i-> System.out.print(i+","));//1,2,3,5,8,13,21,34,55,89,144,233,···
~~~

# 基本数据流

如果要操作基本数据类型元素流，直接使用`Stream`就需要将每个基本数据类型包装然后放入流中；这显然是很低效的；所以Java提供了专门的类型`IntStream、LongStream、DoubleStream`用来直接储存基本类型值；

使用IntStream存储`short、char、byte、boolean、int`

使用LongStream存储`long`

使用`DoubleStream`存储`double、float`

# Collector和Collectors

Stream有以`Collector`作为入参的`collect()`重载函数：

~~~java
<R, A> R collect(Collector<? super T, A, R> collector);
//使用收集器对此流的元素执行可变约简操作。Collector封装了用作collect参数的函数(Supplier、bicconsumer、bicconsumer)，允许重用收集策略和组合收集操作(如多级分组或分区)。
~~~

Collector：

~~~java
public interface Collector<T, A, R> {
    Supplier<A> supplier();
    BiConsumer<A, T> accumulator();
    BinaryOperator<A> combiner();
    Function<A, R> finisher();
    Set<Characteristics> characteristics();
}
~~~

直接实现一个Collector以调用collect方法十分麻烦；java提供Collectors工具类，它提供的静态方法支持获取一些现成的常用的Collector.

Collectors常用静态方法：

## toXXX()

~~~java
<T, C extends Collection<T>> Collector<T, ?, C> toCollection(Supplier<C> collectionFactory) ;
//返回一个Collector，它按遇到的顺序将输入元素累积到一个新的集合中。集合由提供的工厂创建，使用例:
<T> Collector<T, ?, List<T>> toList();
//返回一个Collector，它将输入元素累加到一个新的List中;该List实际是ArrayList;如果需要对返回的List进行更多控制，则使用toCollection(Supplier)
<T> Collector<T, ?, Set<T>> toSet();
//返回一个Collector，将输入元素累加到一个新的Set中。该Set实际是HashSet;如果需要对返回的集合进行更多的控制，使用toCollection(provider)。
<T, K, U> Collector<T, ?, Map<K,U>> toMap(Function<? super T, ? extends K> keyMapper, Function<? super T, ? extends U> valueMapper);
<T, K, U> Collector<T, ?, Map<K,U>> toMap(Function<? super T, ? extends K> keyMapper,Function<? super T, ? extends U> valueMapper,BinaryOperator<U> mergeFunction);
<T, K, U, M extends Map<K, U>> Collector<T, ?, M> toMap(Function<? super T, ? extends K> keyMapper, Function<? super T, ? extends U> valueMapper, BinaryOperator<U> mergeFunction, Supplier<M> mapSupplier);
//返回一个Collector，它将元素累积到一个Map中，该Map的键和值是将提供的映射函数应用于输入元素的结果。
/**
keyMapper – 生成键的映射函数 
valueMapper – 生成值的映射函数 
mergeFunction – 一个合并函数，用于解决与同一键相关联的值之间的冲突，提供给Map.merge(Object, Object, BiFunction)
mapSupplier – 返回一个新的空Map，将结果插入其中的函数
*/
~~~

使用例:

~~~java
HashSet<String> collect1 = Stream.of("one", "three", "tow")
        .collect(Collectors.toCollection(HashSet::new));
System.out.println(collect1);//[one, three, tow]
~~~

~~~java
List<Integer> collect2 = Stream.of(1, 2, 3)
        .collect(Collectors.toList());
System.out.println(collect2);//[1, 2, 3]
~~~

~~~java
LinkedHashMap<Integer, String> map = Stream.of("three", "sleep", "four", "one", "then")
    .collect(Collectors.toMap(String::length, 
                              str -> str, (s1, s2) -> s1 + "," + s2, 
                              LinkedHashMap::new));
System.out.println(map);//{5=three,sleep, 4=four,then, 3=one}
~~~

## joining()

~~~java
Collector<CharSequence, ?, String> joining();
Collector<CharSequence, ?, String> joining(CharSequence delimiter);
Collector<CharSequence, ?, String> joining(CharSequence delimiter,CharSequence prefix, CharSequence suffix);
//返回一个Collector，按相遇顺序将输入元素连接成一个String
//delimiter - 分隔符 prefix – 前缀 suffix – 后缀
~~~

使用例：

~~~java
String str = Stream.of("one", "tow")
        .collect(Collectors.joining(",", "zero,", ",three"));
System.out.println(str);//zero,one,tow,three
~~~

## groupingBy()

~~~java
<T, K> Collector<T, ?, Map<K, List<T>>> groupingBy(Function<? super T, ? extends K> classifier);
<T, K, A, D> Collector<T, ?, Map<K, D>> groupingBy(Function<? super T, ? extends K> classifier, Collector<? super T, A, D> downstream);
<T, K, D, A, M extends Map<K, D>> <T, ?, M> groupingBy(Function<? super T, ? extends K> classifier, Supplier<M> mapFactory, Collector<? super T, A, D> downstream);
//返回一个Collector，该Collector对类型为T的输入元素执行级联“group by”操作，根据分类函数classifier对元素进行分组，然后使用指定的下游Collector对与给定键相关的值执行约简操作。Collector生成的Map是用提供的工厂函数创建的
~~~

使用例：

~~~java
//将流中字符元素按照字符长度分类
HashMap<Integer, LinkedList<String>> collect = Stream.of("three", "sleep", "four", "one","then")
        .collect(Collectors.groupingBy(String::length, HashMap::new,
                Collectors.toCollection(LinkedList::new)));
System.out.println(collect);//{3=[one], 4=[four, then], 5=[three, sleep]}
~~~

## groupingByConcurrent()

~~~java
<T, K> Collector<T, ?, ConcurrentMap<K, List<T>>> groupingByConcurrent(Function<? super T, ? extends K> classifier);
<T, K, A, D> Collector<T, ?, ConcurrentMap<K, D>> groupingByConcurrent(Function<? super T, ? extends K> classifier,Collector<? super T, A, D> downstream);
<T, K, A, D, M extends ConcurrentMap<K, D>> Collector<T, ?, M> groupingByConcurrent(Function<? super T, ? extends K> classifier,Supplier<M> mapFactory, Collector<? super T, A, D> downstream);
//返回一个并发收集器，该收集器对类型为T的输入元素执行级联“group by”操作，根据分类函数对元素进行分组，然后使用指定的下游收集器对与给定键相关的值执行约简操作。由Collector生成的ConcurrentMap是用提供的工厂函数创建的。这是一个并发的无序收集器。
~~~

## partitioningBy()

~~~java
<T> Collector<T, ?, Map<Boolean, List<T>>> partitioningBy(Predicate<? super T> predicate);
<T, D, A> Collector<T, ?, Map<Boolean, D>> partitioningBy(Predicate<? super T> predicate,Collector<? super T, A, D> downstream) ;
//返回一个Collector，该Collector根据Predicate对输入元素进行分区，根据另一个Collector对每个分区中的值进行约简，并将它们组织成一个Map<Boolean, D>，其值是下游约简的结果。
~~~

使用例：

~~~java
Map<Boolean, Set<Integer>> collect = Stream.of(1, 2, 3, 4, 5, 6)
    .collect(Collectors.partitioningBy(i -> i > 3, Collectors.toSet()));
System.out.println(collect);
//{false=[1, 2, 3], true=[4, 5, 6]}
~~~

## mapping()

~~~java
<T, U, A, R> Collector<T, ?, R> mapping(Function<? super T, ? extends U> mapper,Collector<? super U, A, R> downstream);
//通过在累加前对每个输入元素应用映射函数，将一个接受U类型元素的Collector适配为一个接受T类型元素的Collector。
~~~

使用例：

~~~java
//将流中字符元素按照字符长度分类,并将字符串转换为大写后收集到map中
Map<Integer, List<String>> collect = Stream.of("three", "sleep", "four", "one", "then")
            .collect(Collectors.groupingBy(String::length,
                    Collectors.mapping(String::toUpperCase, Collectors.toList())));
System.out.println(collect);//{3=[ONE], 4=[FOUR, THEN], 5=[THREE, SLEEP]}
~~~

## collectingAndThen()

~~~java
<T,A,R,RR> Collector<T,A,RR> collectingAndThen(Collector<T,A,R> downstream, Function<R,RR> finisher); 
//调整收集器以执行额外的整理转换
~~~

使用例：

~~~java
//执行完收集操作后，将容器转换成不可变list
List<Integer> collect = Stream.of(1, 2, 3, 4, 5).collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList));
System.out.println(collect.getClass());
//class java.util.Collections$UnmodifiableRandomAccessList
~~~

## counting()

~~~java
<T> Collector<T, ?, Long> counting();
//返回一个接受类型为T的元素的Collector，用于统计输入元素的数量。如果没有元素存在，结果为0。
~~~

使用例

~~~java
//将流中字符元素按照字符长度分类；并只保存每个长度对应的字符数；而不是字符本身
Map<Integer, Long> collect = Stream.of("three", "sleep", "four", "one", "then")
    .collect(Collectors.groupingBy(String::length, Collectors.counting()));
System.out.println(collect);//{3=1, 4=2, 5=2}
~~~

## min/maxBy()

~~~java
<T> Collector<T, ?, Optional<T>> minBy(Comparator<? super T> comparator);
<T> Collector<T, ?, Optional<T>> maxBy(Comparator<? super T> comparator);
//返回一个Collector，该Collector根据给定的Comparator生成最小/最大元素
~~~

## summingXXX()

~~~java
<T> Collector<T, ?, Integer> summingInt(ToIntFunction<? super T> mapper);
<T> Collector<T, ?, Long> summingLong(ToLongFunction<? super T> mapper);
<T> Collector<T, ?, Double> summingDouble(ToDoubleFunction<? super T> mapper);
//返回一个Collector，该Collector生成应用于输入元素的数值函数的和。如果没有元素存在，结果为0。
~~~

## averagingXXX()

~~~java
<T> Collector<T, ?, Double> averagingInt(ToIntFunction<? super T> mapper);
<T> Collector<T, ?, Double> averagingLong(ToLongFunction<? super T> mapper);
<T> Collector<T, ?, Double> averagingDouble(ToDoubleFunction<? super T> mapper);
//返回一个Collector，该Collector生成应用于输入元素的数值函数的算术平均值。如果没有元素存在，结果为0
~~~

使用例：

~~~java
//求字符长度的平均值
Double ave = Stream.of("three", "sleep", "four", "one", "then")
    .collect(Collectors.averagingInt(String::length));
System.out.println(ave);//4.2
~~~

## reducing()

~~~java
<T, U> Collector<T, ?, U> reducing(U identity, Function<? super T, ? extends U> mapper, BinaryOperator<U> op);
<T> Collector<T, ?, T> reducing(T identity, BinaryOperator<T> op);
<T> Collector<T, ?, Optional<T>> reducing(BinaryOperator<T> op);
//返回一个Collector，在指定入参的规则下对其输入元素进行约简。
~~~







