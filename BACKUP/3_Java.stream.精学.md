# [Java stream 精学](https://github.com/Winniekun/article/issues/3)

# 背景

日常的开发和Stream会打很多的交道，譬如获取DTO列表中某个属性的列表，譬如将 DTO 列表按照某个属性进行聚类转化成 map 等。

# 使用

`Stream` 的使用步骤如下：

- **创建** Stream。
- 通过一个或多个中间操作(intermediate operations)将初始 Stream **转换**为另一个 Stream。
- 通过**终止**操作(terminal operation)获取结果；该操作触发之前的懒操作的执行，终止操作后，该 Stream 关闭，不能再使用了。

## 创建

- 通过 **`Stream`** 接口的`静态工厂方法`；
- 通过 **`Collection`** 接口的默认方法– `stream()`，为集合创建`串行流`。
- 通过 **`Collection`** 接口的默认方法– `parallelStream()`，为集合创建`并行流`。
- 通过 **`Arrays`** 类的 `stream()` 静态工厂方法。



### 通过 Stream 的静态方法来创建 Stream

`Stream` 接口提供了几个静态工厂方法用来创建 `Stream` 对象：

| 方法                                                         | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `static <T> Builder<T> builder()`                            | 返回一个 Stream 的可变构建器，可通过 Builder 的 build() 方法返回一个 Stream 实例 |
| `static <T> Stream<T> empty()`                               | 返回一个空的 Stream 实例                                     |
| `static <T> Stream<T> of(T t)`                               | 返回一个包含单一元素的 Stream 实例                           |
| `static <T> Stream<T> of(T... values)`                       | 返回一个包含指定元素并且按顺序排列的 Stream 实例             |
| `static <T> Stream<T> iterate(T seed, UnaryOperator<T> f)`   | 返回一个无限元素有序的 Stream 实例，其元素的生成是重复对给定的种子值(seed)调用指定函数来生成的，其中包含的元素可以认为是：seed，f(seed)，f(f(seed))无限循环 |
| `static <T> Stream<T> generate(Supplier<T> s)`               | 返回一个无限元素无序的 Stream 实例，其中每个元素由提供的 Supplier 生成，这适用于生成常量流、随机元素流等 |
| `static <T> Stream<T> concat(Stream<? extends T> a, Stream<? extends T> b)` | 创建一个延迟连接的 Stream 实例，其元素是第一个流的所有元素，然后是第二个流的所有元素 |

> `Stream` 的 `iterate()` 和 `generate()` 方法都用于生成无限元素的流，所以一般这种无限长度的 `Stream` 都会配合 `Stream` 的 `limit()` 方法来用。

### 通过 Collection 子类创建 Stream

`Collection` 接口提供了2个默认方法用来创建 `Stream` 对象：

| 方法                                 | 描述                                       |
| :----------------------------------- | :----------------------------------------- |
| `default Stream<E> stream()`         | 返回一个指定该集合为源的有序的 Stream 实例 |
| `default Stream<E> parallelStream()` | 返回一个指定该集合为源的并行的 Stream 实例 |

### 通过 Arrays 的静态方法来创建 Stream

`Arrays` 类提供了几个个静态工厂方法用来创建 `Stream` 对象：

| 方法                                                         | 描述                                                       |
| :----------------------------------------------------------- | :--------------------------------------------------------- |
| `static <T> Stream<T> stream(T[] array)`                     | 返回一个以指定数组为源的有序的 Stream 实例                 |
| `static <T> Stream<T> stream(T[] array, int startInclusive, int endExclusive)` | 返回一个以指定数组的指定范围为源的有序的 Stream 实例       |
| `static IntStream stream(int[] array)`                       | 返回一个以指定数组为源的有序的 IntStream 实例              |
| `static IntStream stream(int[] array, int startInclusive, int endExclusive)` | 返回一个以指定数组的指定范围为源的有序的 IntStream 实例    |
| `static LongStream stream(long[] array)`                     | 返回一个以指定数组为源的有序的 LongStream 实例             |
| `static static LongStream stream(long[] array, int startInclusive, int endExclusive)` | 返回一个以指定数组的指定范围为源的有序的 LongStream 实例   |
| `static DoubleStream stream(double[] array)`                 | 返回一个以指定数组为源的有序的 DoubleStream 实例           |
| `static DoubleStream stream(double[] array, int startInclusive, int endExclusive)` | 返回一个以指定数组的指定范围为源的有序的 DoubleStream 实例 |

## 转换

### distinct()

**`distinct()`** 方法对于 Stream 中包含的元素进行去重操作(去重逻辑依赖元素的 `equals()` 方法)，新生成的 Stream 中没有重复的元素。

> 如果需要去除的元素是对象，则根据对象对应的`equals` 和 `hashcode` 方法来判断元素是否重复

![image-20230227231711712](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20230227231711712.png)

```java
List<String> words = Lists.newArrayList("Hello", "World", "hello");
words.stream().distinct().forEach(System.out::println);
// Hello 
// World
```



### filter()

`filter()`方法用于对Stream中的元素进行过滤，具体的过滤逻辑根据`filter()`内部的函数进行定义，新生成的 Stream只包含有符合条件的元素

![image-20230227232544718](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20230227232544718.png)

```java
List<String> list = Lists.newArrayList("Java", "Python", "Goland", "Shell");
list.stream().filter(ele -> ele.length() < 5).forEach(System.out::println);
// Java
```

### map()

**`map()`** 方法对于 Stream 中包含的元素使用给定的转换函数进行转换操作，新生成的 Stream 只包含转换生成的元素。这个方法有三个对于原始类型的变种方法，分别是：`mapToInt`，`mapToLong` 和 `mapToDouble`。这三个方法也比较好理解，比如 `mapToInt` 就是把原始 Stream 转换成一个元素都是 int 类型的新 Stream。这样三个变种方法，可以免除自动装箱/拆箱的额外消耗。

![image-20230227233625311](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20230227233625311.png)

```java
List<String> strList = Lists.newArrayList("a", "b", "c");
strList.stream().map(ele -> ele.toUpperCase()).forEach(System.out::println);
// 也可以写成这种
strList.stream().map(String::toUpperCase).forEach(System.out::println);
// A
// B 
// C
```
### flatMap()

**`flatMap()`** 不同于 **`map`** 地方在于 **`map`** 只是提取属性放入流中，而 **`flatMap()`** 先提取属性放入一个比较小的流，然后再将所有的流合并为一个流。

![image-20230228215533915](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20230228215533915.png)

```java
List<String> list = Arrays.asList("k,w,k", "a,b,c");
//两个字符串，内部使用，分割，之后合并为一个新的字符数组
list.stream().flatMap(s -> {
    String[] split = s.split(",");
    return Arrays.stream(split);
}).forEach(System.out::println);
// k
// w
// k
// a
// b
// c
```

### peek()

**`peek()`** 操作接收的是一个 **`Consumer<T>`** 函数。该操作会按照 **`Consumer<T>`** 函数提供的逻辑去消费流中的每一个元素，同时有可能改变元素内部的一些属性。

> **`peek()`** 操作 一般用于不想改变流中元素本身的类型或者只想操作元素的内部状态，而 `map` 则用于改变流中元素本身类型，即从元素中派生出另一种类型的操作。

![image-20230228223701126](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20230228223701126.png)

```java
List<String> list = Lists.newArrayList("abandon", "boom", "com");
list.stream().filter(ele -> ele.length() < 7).peek(s -> System.out.println("After filer value: " + s))
        .map(String::toUpperCase).peek(s -> System.out.println("After map value: " + s)).forEach(System.out::println);
// After filer value: boom
// After map value: BOOM
// BOOM
// After filer value: com
// After map value: COM
// COM
```

### limit()

**`limit()`** 方法对一个 Stream 进行截断操作，获取其前 N 个元素，如果原 Stream 中包含的元素个数小于 N，那就获取其所有的素。

![image-20230228224803775](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20230228224803775.png)

```java
List<Integer> list = Lists.newArrayList(1, 2, 3, 4, 5, 6, 7);
list.stream().limit(3).forEach(System.out::println);
// 1
// 2
// 3
```

### reduce()

**`reduce()`** 方法用于从 Stream 中生成一个值，其生成的值不是随意的，而是根据指定的计算模型。比如，`min()`、`max()` 和 `count()` 方法，因为常用而被纳入标准库中。事实上，这些方法都是 `reduce` 操作。

![image-20230228225117791](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20230228225117791.png)
```java
List<Integer> list = Arrays.asList(1, 3, 2, 8, 11, 4);
// 求和1
Optional<Integer> sum1 = list.stream().reduce((x, y) -> x + y);
// 求和2
Optional<Integer> sum2 = list.stream().reduce(Integer::sum);
// 求和3
Integer sum3 = list.stream().reduce(0, Integer::sum);
```

## 终止
TODO


# 实战
TODO

# 原理
![stream-source-5](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/stream-source-5.jpeg)

# references
- [Java8 Stream原理深度解析](https://www.cnblogs.com/Dorae/p/7779246.html)
- [Java 8 Stream 探秘](https://colobu.com/2014/11/18/Java-8-Stream/)
- [Java Stream 源码深入解析](https://xie.infoq.cn/article/a0e6dc7ed0735da5128b6825b)
- [JavaLambdaInternals](