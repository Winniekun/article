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

## 终止
TODO


# 实战
TODO

# 原理
![stream-source-5](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/stream-source-5.jpeg)

# references
