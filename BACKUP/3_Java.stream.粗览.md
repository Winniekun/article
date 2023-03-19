# [Java stream 粗览](https://github.com/Winniekun/article/issues/3)

# 背景

日常的开发和Stream会打很多的交道，譬如获取DTO列表中某个属性的列表，譬如将 DTO 列表按照某个属性进行聚类转化成 map 等。

# 使用

## 概览

![Stream 使用概览](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/Stream%20%E4%BD%BF%E7%94%A8%E6%A6%82%E8%A7%88.png)

图中4种`Stream`接口继承自`BaseStream`，其中`IntStream, LongStream, DoubleStream`对应三种基本类型（`int, long, double`，注意不是包装类型），`Stream`对应所有剩余类型的Stream视图。为不同数据类型设置不同Stream接口原因：

1. 提高性能
2. 增加特定接口函数

虽然大部分情况下`Stream`是容器调用`Collection.stream()`方法得到的，但`Stream`和Collection有以下不同：

- **无存储**。`Stream`不是一种数据结构，它只是某种数据源的一个视图，数据源可以是一个数组，Java容器或I/O channel等。
- **为函数式编程而生**。对`Stream`的任何修改都不会修改背后的数据源，比如对`Stream`执行过滤操作并不会删除被过滤的元素，而是会产生一个不包含被过滤元素的新`Stream`。
- **惰式执行**。`Stream`上的操作并不会立即执行，只有等到用户真正需要结果的时候才会执行。
- **可消费性**。`Stream`只能被“消费”一次，一旦遍历过就会失效，就像容器的迭代器那样，想要再次遍历必须重新生成。

对`Stream`的操作分为为两类，**中间操作(intermediate operations)和结束操作(terminal operations)**，二者特点是：

1. **中间操作总是会惰式执行**，调用中间操作只会生成一个标记了该操作的新`Stream`，仅此而已。
2. **结束操作会触发实际计算**，计算发生时会把所有中间操作积攒的操作以*pipeline*的方式执行，这样可以减少迭代次数。计算完成之后`Stream`就会失效。

具体的操作和**Apache Spark RDD**类似，对`Stream`的这个特点应该不陌生。

`Stream` 的使用步骤如下：

- **创建** Stream。
- 通过一个或多个**中间操作(intermediate operations)**将初始 Stream **转换**为另一个 Stream。
- 通过**终止**操作(terminal operation)获取结果；该操作触发之前的懒操作的执行，终止操作后，该 Stream 关闭，不能再使用了。

## 创建

- 通过 **`Stream`** 接口的`静态工厂方法`；
- 通过 **`Collection`** 接口的默认方法– `stream()`，为集合创建`串行流`。
- 通过 **`Collection`** 接口的默认方法– `parallelStream()`，为集合创建`并行流`。
- 通过 **`Arrays`** 类的 `stream()` 静态工厂方法。

## 常见操作介绍

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

**`reduce()`** 方法用于从 Stream 中生成一个值，其生成的值不是随意的，而是根据指定的计算模型。比如，`min()`、`max()` 和 `count()` 方法都是使用 `reduce` 操作。

![image-20230228225117791](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20230228225117791.png)

- `Optional<T> reduce(BinaryOperator<T> accumulator)`
- `T reduce(T identity, BinaryOperator<T> accumulator)`
- `<U> U reduce(U identity, BiFunction<U,? super T,U> accumulator, BinaryOperator<U> combiner)`



```java
List<Integer> list = Arrays.asList(1, 3, 2, 8, 11, 4);
// 求和1
Optional<Integer> sum1 = list.stream().reduce((x, y) -> x + y);
// 求和2
Optional<Integer> sum2 = list.stream().reduce(Integer::sum);
// 求和3
Integer sum3 = list.stream().reduce(0, Integer::sum);

Stream<String> stream = Stream.of("I", "am", "a", "programmer");
Integer lengthSum = stream.reduce(0,　// 初始值　// (1)
        (sum, str) -> sum+str.length(), // 累加器 // (2)
        (a, b) -> a+b);　// 部分和拼接器，并行执行时才会用到 // (3)
// 14
```



## Collect大法

> 如果在Stream中没有找到某个功能的接口，那么大概率可以通过`collect`方法实现，`collect()`是*Stream*接口方法中最灵活的一个，学会它才算真正入门Java函数式编程

```java
List<String> list = Lists.newArrayList("I", "am", "a", "programmer");
// Function.identity() 返回一个输出跟输入一样的Lambda表达式对象，等价于形如`t -> t`形式的Lambda表达式
Map<String, Integer> map = list.stream().collect(Collectors.toMap(Function.identity(), String::length));
List<String> newList = list.stream().collect(Collectors.toList());
Set<String> set = list.stream().collect(Collectors.toSet());
```

### Collectors

收集器（Collector）是为`Stream.collect()`方法量身打造的工具接口（类）。设想将一个Stream转换成一个容器（或者Map）需要做哪些工作？

1. 目标容器是什么？是ArrayList还是HashSet，或者是个HashMap。
2. 新元素如何添加到容器中？是`List.add()`还是`Map.put()`。

如果并行的进行规约，还需要告诉collect()

  3. 多个部分结果如何合并成一个。

结合以上分析，**collect()**方法定义为`<R> R collect(Supplier<R> supplier, BiConsumer<R,? super T> accumulator, BiConsumer<R,R> combiner)`，三个参数依次对应上述三条分析。不过每次调用**collect()**都要传入这三个参数太麻烦，收集器**Collector**就是对这三个参数的简单封装，所以**collect()**的另一定义为`<R,A> R collect(Collector<? super T,A,R> collector)`。**Collectors**工具类可通过静态方法生成各种常用的Collector。譬如，要将Stream转换成**List**可以通过如下两种方式实现：

```java
Stream<String> stream = Stream.of("I", "love", "you", "too");
// ① 直接使用 collect
List<String> list = stream.collect(ArrayList::new, ArrayList::add, ArrayList::addAll);
// ② 使用Collectors
List<String> list = stream.collect(Collectors.toList());// 方式2
System.out.println(list);

```

所以通常情况下都是调用`collect(Collector<? super T,A,R> collector)`方法，而不是手动指定**collect()**的三个参数。

```java
// 判断列表 A 中是否包含列表 B， 并返回 map 对象
@Test
public void testCollector() {
    List<String> list = Lists.newArrayList("I", "am", "a", "programmer");
    List<String> rest = Lists.newArrayList("I", "have", "a", "cat");
    Map<String, Boolean> map = list.stream().collect(Collectors.toMap(Function.identity(), ele -> contains(ele, rest)));
    System.out.println(map);
}
// {a=true, programmer=false, I=true, am=false}
private boolean contains(String ele, List<String> rest) {
    return rest.contains(ele);
}
```

#### 使用collect生成Collection

#### 使用collect生成map

> Stream 背后的数据源可以是数组、容器等，但不能是 Map，换言之 Stream->Map 是可以的，但是转换操作相比于转换为 collection稍加复杂，我们需要清楚构造Map的 key 和 value 分别代表什么。

1. `Collectors.toMap()`

   >  需要手动指定key 和 value，并且该方法对key 和 value 都有严格要求
   >
   >  - 不允许key 有重复，如果有重复未处理的话，会报错
   >   - [[Java9之后已经修复](https://bugs.openjdk.org/browse/JDK-8173464)](https://bugs.openjdk.org/browse/JDK-8173464)，我们可以通过手动指定当 key重复时，使用哪个 value 值
   >  - 不允许value 值为空，否则会报错。

   该操作上述内容已经介绍了，它和`Collectors.toCollection()`是并列的。

   ```java
   Employee a = Employee.builder().age(1).name("abc").build();
   Employee b = Employee.builder().name("def").build();
   Employee c = Employee.builder().age(100).name("def").build();
   // 报错，b对象中，age 属性为null
   Map<String, Integer> map = employeeList.stream().collect(Collectors.toMap(Employee::getName, Employee::getAge));
   // 解决① 模拟正常的 map的 put 操作(使用 collect直接写)
   HashMap<String, Integer> employeeMap = employeeList.stream().collect(HashMap::new, (map, item) -> map.put(item.getName(), item.getAge()), HashMap::putAll);
   // {abc=1, def=null}
   
   // 解决② 未value=null 的值填充默认值
   Map<String, Integer> employeeMap = employeeList.stream().collect(Collectors.toMap(Employee::getName, s -> Optional.ofNullable(s.getAge()).orElse(0)));
   // 解决② 未value=null 的值填充默认值，并且新增对重复 key 值得处理
   Map<String, Integer> employeeMap = employeeList.stream().collect(Collectors.toMap(Employee::getName, s -> Optional.ofNullable(s.getAge()).orElse(0), (key1, key2) -> key1));
   
   ```

   

2. `Collectors.partitioningBy()`

   使用`partitioningBy()`生成的收集器，这种情况适用于将`Stream`中的元素依据某个**二值逻辑**（满足条件，或不满足）分成互补相交的两部分，比如男女性别、成绩及格与否等。下列代码展示将学生分成成绩及格或不及格的两部分。

   ```java
   // 根据分数获取通过和不通过的学生名单
   Map<Boolean, List<Student>> passingFailing = students.stream()
            .collect(Collectors.partitioningBy(s -> s.getGrade() >= PASS_THRESHOLD));
   ```

3. `Collectors.groupingBy()`

   比较灵活的一种情况。跟SQL中的*group by*语句类似，这里的*groupingBy*()也是按照某个属性对数据进行分组，属性相同的元素会被对应到*Map的同一个key*上

   ```java
   // 按照部门对员工进行分组
   Map<Department, List<Employee>> byDept = employees.stream()
               .collect(Collectors.groupingBy(Employee::getDepartment));
   ```

   当然还能更灵活的使用，譬如我们只想根据部门对员工进行分组，然后想得到是员工的名字，而不是 Employee 对象

   ```
   Map<String, List<String>> employeeMap = employeeList.stream().collect(Collectors.groupingBy(Employee::getDepartment,
           Collectors.mapping(Employee::getName,  Collectors.toList())));
   ```

#### 使用collect 做字符串 join

> 从此，对字符串的操作，告别 for循环

```java
List<String> list = Lists.newArrayList("I", "am", "a", "programmer");
String joined = list.stream().collect(Collectors.joining());
// Iamaprogrammer
String joined = list.stream().collect(Collectors.joining(","));
// I,am,a,programmer
String joined = list.stream().collect(Collectors.joining(",", "{", "}"));
// {I,am,a,programmer}
```



## 操作总结

> 上述只介绍了常用的一些操作，如果想了解更多的流操作，可以参考《Java8函数式编程》

![分类](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20230319160752234.png)[[图片出处](https://www.cnblogs.com/CarpenterLee/p/6637118.html)](https://www.cnblogs.com/CarpenterLee/p/6637118.html)

> 流操作共分为两大类：中间操作和结束操作，中间操作只负责对操作进行记录，只有结束操作才会触发实际计算（所以说中间操作是惰性的，结束操作是非惰性的），同时其也是 Stream 在迭代大集合时高效的原因之一。中间操作又可以再详细划分为**无状态操作**与**有状态操作**：
>
> - 无状态操作
>   - 元素在处理时，不受之前元素影响
> - 有状态操作
>   - 只有拿到所有元素才能继续执行下去
>
> 结束操作可以详细划分为**短路操作**与**非短路操作**
>
> - 短路操作
>   - 遇到某些符合条件的元素就可以得到最终结果
> - 非短路操作
>   - 必须处理完所有元素才能得到最终结果

# 实战

[[20个示例玩转 Java8 Stream](https://mp.weixin.qq.com/s/2WYpN1hcTgVChfcXtnvK5g)](https://mp.weixin.qq.com/s/2WYpN1hcTgVChfcXtnvK5g)

# 原理

## 示例

统计b 班学生的兴趣爱好情况

```java
public class Student {
    /**
     * 班级
     */
    private String clazz;
    /**
     * 名字
     */
    private String name;
    /**
     * 年龄
     */
    private Integer age;
    /**
     * 爱好
     */
    private List<String>  hobbies;
}

public static List<Student> generateStudent() {
    Student first = Student.builder()
            .clazz("a")
            .name("fist")
            .hobbies(Lists.newArrayList("basketball", "table tennis", "football"))
            .build();
    Student second = Student.builder()
            .clazz("b")
            .name("bFirst")
            .hobbies(Lists.newArrayList("basketball", "table tennis", "football"))
            .build();
    Student third = Student.builder()
            .clazz("b")
            .name("bSecond")
            .hobbies(Lists.newArrayList("basketball", "badminton", "football"))
            .build();
    Student fourth = Student.builder()
            .clazz("b")
            .name("bThird")
            .hobbies(Lists.newArrayList("basketball", "tennis", "table tennis"))
            .build();
    Student fifth = Student.builder()
            .clazz("b")
            .name("bFourth")
            .hobbies(Lists.newArrayList("basketball", "tennis", "football"))
            .build();
    return Lists.newArrayList(first, second, third, fourth, fifth);
}
```

**计算**

```java
// ① 流操作
Map<String, Long> hobbyMap = students.stream()
        .filter(student -> Objects.equals(student.getClazz(), "b"))
        .flatMap(student -> student.getHobbies().stream())
        .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
System.out.println(hobbyMap);

// ② 非流操作
hobbyMap = Maps.newHashMapWithExpectedSize(16);
for (Student student : students) {
    if (!"b".equals(student.getClazz())) {
        continue;
    }
    List<String> hobbies = student.getHobbies();
    for (String hobby : hobbies) {
        hobbyMap.put(hobby, hobbyMap.getOrDefault(hobby, 0L) + 1);
    }
}
System.out.println(hobbyMap);
```

## Stream 中需要解决的问题

根据①②的思路，可以大致猜想 Stream 实现需要解决的问题：

- Stream 自身不存储数据，那么数据存储在哪儿？
- 中间操作是如何进行关联的？
- 中间操作是如何按照顺序执行的?
- 有状态的中间操作如何保存状态？
- 惰式执行如何实现
- 如何保证一个流一次只能执行一个终结方法

## stream 中类关系分析（无并发操作相关类）

![流的各种工厂类](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/%E6%B5%81%E7%9A%84%E5%90%84%E7%A7%8D%E5%B7%A5%E5%8E%82%E7%B1%BB.png)

主要是各种操作的工厂类、数据的存储结构以及收集器的工厂类等

- MatchOps：用于实现allMatch、anyMatch、noneMatch的终止操作
- SortedOps：用于实现流中元素的排序操作

![Stream 概览图](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/Stream%20%E6%A6%82%E8%A7%88%E5%9B%BE.png)

主要用于Stream的求值实现

- BaseStream规定了流的基本接口，比如iterator、spliterator等
- PipelineHelper主要用于Stream执行过程中相关结构的构建
- 各种Stream中定义了map、filter、flatmap等用户关注的常用操作
- ReferencePipeline
  - 数据源和中心阶段的核心实现类
- Head、StatelessOp、StatefulOp为ReferencePipeline中的静态内部类，当然其他 pipeline 也包含有这三种这三种静态内部类
  - Head： 代表数据源阶段
  - StateFulOp：流有状态中间阶段
  - StatelessOp：流的无状态中间阶段
- Sink
  - Consumer 的扩展，在流管道的各个阶段传递值。

## 概念解释

- Pipline ：流水线，表示一整个流程。
- Stage ：  表示流水线的其中一个阶段。是一个比较抽象层面的描述，因为 stage 主要表示一种逻辑上的顺序关系，而具体每一个阶段要干嘛、怎么干，使用 Sink 来进行描述。
- Sink：直译为水槽，生活中水槽的作用可以理解分为如下几个部分，Stream 中也有 Sink 的概念。
  - 打开水龙头，有水要来
  - 水在水槽里, 可以进行一些操作
  - 打开水闸放水，流入到其他地方

```java
interface Sink<T> extends Consumer<T> {
    /**
     * 开始遍历元素之前调用该方法，通知Sink做好准备
     */
    default void begin(long size) {}

    /**
     * 所有元素遍历完成之后调用，通知Sink没有更多的元素了
     */
    default void end() {}

    /**
     * 是否可以结束操作，可以让短路操作尽早结束
     */
    default boolean cancellationRequested() {
        return false;
    }

    /**
     *遍历元素时调用，接受一个待处理元素，并对元素进行处理。Stage把自己包含的操作和回调方法封装到该方法里，
     * 前一个Stage只需要调用当前Stage.accept(T t)方法就行了。
     */
    default void accept(int value) {
        throw new IllegalStateException("called wrong accept method");
    }

    /**
     * 
     */
    default void accept(long value) {
        throw new IllegalStateException("called wrong accept method");
    }

    /**
     * 
     */
    default void accept(double value) {
        throw new IllegalStateException("called wrong accept method");
    }
}
```



## 流水线结构图



![Stream流水线组织结构](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/Stream%E6%B5%81%E6%B0%B4%E7%BA%BF%E7%BB%84%E7%BB%87%E7%BB%93%E6%9E%84.png)

图中通过`Collection.stream()`方法得到**Head**也就是stage0，紧接着调用一系列的操作，不断产生新的Stream。**这些Stream对象以双向链表的形式组织在一起，构成整个流水线，并且每个Stage都记录了前一个Stage和本次的操作以及回调函数，依靠这种结构就能建立起对数据源的所有操作**。这就是Stream记录操作的方式。接下来以该图的顺序，过一遍 Stream 的执行逻辑

## 创建 Head

### 使用

```java
Stream<Integer> stream = Stream.of(1,1,1,2,1,3,2,4);
```

```java
public static <T> Stream<T> stream(Spliterator<T> spliterator, boolean parallel) {
    Objects.requireNonNull(spliterator);
    return new ReferencePipeline.Head<>(spliterator,
                                        StreamOpFlag.fromCharacteristics(spliterator),
                                        parallel);
}
```

调用了`ReferencePipeline.Head<>`，返回一个 Head 对象。可以理解为 Head 是流水线的第一个 stage。构造方法的直 super()到`AbstractPipeline`类。

```java
AbstractPipeline(Spliterator<?> source,
                 int sourceFlags, boolean parallel) {
    this.previousStage = null;
    // 使用一个字段指向数据集合的Spliterator,后续终结操作的时候，引用的方式操作数据
    this.sourceSpliterator = source;
    this.sourceStage = this;
    this.sourceOrOpFlags = sourceFlags & StreamOpFlag.STREAM_MASK;
    // The following is an optimization of:
    // StreamOpFlag.combineOpFlags(sourceOrOpFlags, StreamOpFlag.INITIAL_OPS_VALUE);
    this.combinedFlags = (~(sourceOrOpFlags << 1)) & StreamOpFlag.INITIAL_OPS_VALUE;
    this.depth = 0;
    this.parallel = parallel;
}
```

![image-20230319225943083](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20230319225943083.png)

> 1. **Stream 不存储数据，那么数据保存在那里**
>    - Head 中保存数据源的 Spliterator 对象，后续操作 Spliterator 的方式操作数据



## 中间操作

```java
Stream<Integer> st = headStream.filter(ele -> ele == 1)
  .distinct()
  .sorted();
//等同于
Stream<Integer> afterFilter = headStream.filter(e -> e == 1); // ①
Stream<Integer> afterDistinct = afterFilter.distinct();       // ②
Stream<Integer> afterSort = afterDistinct.sorted();           // ③
```

**先看第①步中 filter 的执行逻辑**

```java
@Override
public final Stream<P_OUT> filter(Predicate<? super P_OUT> predicate) {
    Objects.requireNonNull(predicate);
    return new StatelessOp<P_OUT, P_OUT>(this, StreamShape.REFERENCE,
                                 StreamOpFlag.NOT_SIZED) {
        @Override
        Sink<P_OUT> opWrapSink(int flags, Sink<P_OUT> sink) {
            return new Sink.ChainedReference<P_OUT, P_OUT>(sink) {
                @Override
                public void begin(long size) {
                    downstream.begin(-1);
                }
                @Override
                public void accept(P_OUT u) {
                    if (predicate.test(u))
                        downstream.accept(u);
                }
            };
        }
    };
}
```

返回一个`StatelessOp`类( filter 是一个无状态操作)，`StatelessOp`类,是一个静态抽象内部类，继承了`ReferencePipeline`类。在该类中的构造方法，会一直super()到`AbstractPipeline`类对应构造方法, 连接 stage 之间的关系。

```java
AbstractPipeline(AbstractPipeline<?, E_IN, ?> previousStage, int opFlags) {
    if (previousStage.linkedOrConsumed)
        throw new IllegalStateException(MSG_STREAM_LINKED);
    previousStage.linkedOrConsumed = true;
    previousStage.nextStage = this;
    this.previousStage = previousStage;
    this.sourceOrOpFlags = opFlags & StreamOpFlag.OP_MASK;
    this.combinedFlags = StreamOpFlag.combineOpFlags(opFlags, previousStage.combinedFlags);
    this.sourceStage = previousStage.sourceStage;
    if (opIsStateful())
        sourceStage.sourceAnyStateful = true;
    this.depth = previousStage.depth + 1;
}
```

![image-20230319231805090](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20230319231805090.png)

**第②步中 distinct 的执行逻辑**

调用 DistinctOps 类的 makeRef()方法，返回一个 StatefulOp 类，并重写了 4 个方法，封装成 Sink 对象的逻辑在 opWrapSink()中:

```java
@Override
public final Stream<P_OUT> distinct() {
    return DistinctOps.makeRef(this);
}

static <T> ReferencePipeline<T, T> makeRef(AbstractPipeline<?, T, ?> upstream) {
    // 返回StatefulOp对象
    return new ReferencePipeline.StatefulOp<T, T>(upstream, StreamShape.REFERENCE,
                                                  StreamOpFlag.IS_DISTINCT | StreamOpFlag.NOT_SIZED) {
        <P_IN> Node<T> reduce(PipelineHelper<T> helper, Spliterator<P_IN> spliterator) {}
        @Override
        <P_IN> Node<T> opEvaluateParallel(PipelineHelper<T> helper, Spliterator<P_IN> spliterator, IntFunction<T[]> generator) {}
        @Override
        <P_IN> Spliterator<T> opEvaluateParallelLazy(PipelineHelper<T> helper, Spliterator<P_IN> spliterator){}
          
        @Override
        Sink<T> opWrapSink(int flags, Sink<T> sink) {
            Objects.requireNonNull(sink);
            if (StreamOpFlag.DISTINCT.isKnown(flags)) {
                ...
            } else if (StreamOpFlag.SORTED.isKnown(flags)) {
                ...
            } else {
                 // 返回一个SinkChainedReference类
                return new Sink.ChainedReference<T, T>(sink) {
                    Set<T> seen;
                    
                    //当上游调用begin的时候，初始化Set
                    @Override
                    public void begin(long size) {
                        seen = new HashSet<>();
                        downstream.begin(-1);
                    }
                    @Override
                    public void end() {
                        seen = null;
                        downstream.end();
                    }
                    
                    //如果已经存在，之间抛弃
                    @Override
                    public void accept(T t) {
                        if (!seen.contains(t)) {
                            seen.add(t);
                            downstream.accept(t);
                        }
                    }
                };
            }
        }
    };
}

```

StatefulOp 类与 StatelessOp 类相似,都是继承了`ReferencePipeline`类，然后中间 super()会一直执行到`AbstractPipeline`类对应的构造方法, 连接 stage 之间的关系。

![image-20230319232909067](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20230319232909067.png)

**第③步中 distinct 的执行逻辑**

同样调用了SortedOps 类的 makeRef()方法，构造了StatefulOp 类，下面的代码是sorted 的一种可能封装的Sink代码

```java
private static final class RefSortingSink<T> extends AbstractRefSortingSink<
    private ArrayList<T> list;
    RefSortingSink(Sink<? super T> sink, Comparator<? super T> comparator) {
        super(sink, comparator);
    }
    @Override
    public void begin(long size) {
        if (size >= Nodes.MAX_ARRAY_SIZE)
            throw new IllegalArgumentException(Nodes.BAD_SIZE);
        // 创建一个存放排序元素的列表
        list = (size >= 0) ? new ArrayList<>((int) size) : new ArrayList<>()
    }
    @Override
    public void end() {
        list.sort(comparator);                // 只有元素全部接收之后才能开始排序
        downstream.begin(list.size());
        if (!cancellationRequestedCalled) {   // 下游Sink不包含短路操作
            list.forEach(downstream::accept); // 将处理结果传递给流水线下游的Sink
        }
        else {                                // 下游Sink包含短路操作
            for (T t : list) {                // 每次都调用cancellationRequested()询问是否可以结束处理。
                if (downstream.cancellationRequested()) break;
                downstream.accept(t);
            }
        }
        downstream.end();
        list = null;
    }
    @Override
    public void accept(T t) {
        list.add(t);                         // 使用当前Sink包装动作处理t，只是简单的将元素添加到中间列表当中
    }
}
```

上述代码完美的展现了Sink的四个接口方法是如何协同工作的：

1. 首先begin()方法告诉Sink参与排序的元素个数，方便确定中间结果容器的的大小；
2. 之后通过accept()方法将元素添加到中间结果当中，最终执行时调用者会不断调用该方法，直到遍历所有元素；
3. 最后end()方法告诉Sink所有元素遍历完毕，启动排序步骤，排序完成后将结果传递给下游的Sink；
4. 如果下游的Sink是短路操作，将结果传递给下游时不断询问下游cancellationRequested()是否可以结束处理。

![image-20230319233914377](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20230319233914377.png)



> 1. **各个中间操作是如何进行关联的？**
>    - 一个个的操作封装成了一个个的`statelessOp`或`stateFulOp`对象，以双向链表的方法串起来。
> 2. **如何执行完一个中间操作，然后执行下一个？**
>    - Sink 类负责流水线操作的承接上下游和执行操作的任务，核心方法为 begain()、accept()、end()。
> 3. **有状态的中间操作是怎么保存状态的？**
>    - 有状态的中间操作封装成`stateFulOp`对象，各自都有独立的逻辑，具体的参考`sort()`的实现逻辑。



## 终结操作

![image-20230319234412583](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/image-20230319234412583.png)

```java
// 计算流中元素数量，FindOP
long count = afterLimit.count();

// 遍历所有元素,ForEachOp
afterLimit.forEach(System.out::printl);

// 获取第一个元素,MatchOp
Optional<Integer> any = afterLimit.findFirst();

// 返回新列表,ReduceOp
List<Integer> collect = afterSort.collect(Collectors.toList());

```

以 collect方法为例，调用关系如下所示：

![collector 流程图](https://cdn.jsdelivr.net/gh/Winniekun/cloudImg@master/uPic/collector%20%E6%B5%81%E7%A8%8B%E5%9B%BE.png)



### wrapSink()方法

```java
@Override
@SuppressWarnings("unchecked")
final <P_IN> Sink<P_IN> wrapSink(Sink<E_OUT> sink) {
    Objects.requireNonNull(sink);
    // 从后向前遍历
    for ( @SuppressWarnings("rawtypes") AbstractPipeline p=AbstractPipeline.this; p.depth > 0; p=p.previousStage) {
        // //执行每个opWrapSink()方法，这个方法在每个操作类中都进行了重写
        sink = p.opWrapSink(p.previousStage.combinedFlags, sink);
    }
    // 返回头sink
    return (Sink<P_IN>) sink;
}
```

### copyInfo()方法

这个方法是整个流水线的启动开关，流程如下：

1. 调用第一个 sink 的 begin()方法
2. 执行数据源的 spliterator.forEachRemaining(wrappendSink)方法遍历调用 accept()方法
3. end() 通知结束

```java
@Override
final <P_IN> void copyInto(Sink<P_IN> wrappedSink, Spliterator<P_IN> spliterator) {
    Objects.requireNonNull(wrappedSink);
    if (!StreamOpFlag.SHORT_CIRCUIT.isKnown(getStreamAndOpFlags())) {
        // 通知第一个sink，做好准备接受流
        wrappedSink.begin(spliterator.getExactSizeIfKnown());
        // 执行
        spliterator.forEachRemaining(wrappedSink);
        wrappedSink.end();
    }
    else {
        copyIntoWithCancel(wrappedSink, spliterator);
    }
}
```



> 1. 终结方法是如何进行操作的？
>    - 终结操作的实现里面都有调用 evaluate()方法，这个方法最后会 wrap 所有操作变成一串 sink，然后从头开始执行 begin(),accept(),end()方法
> 2. 如何实现由终结操作触发流的运作的？
>    - 触发的开关是 wrapAndCopyInto(),这个方法只有在终结操作中才有被调用。
> 3. 如何保证一个流一次只能执行一个终结方法？
>    - evaluate()方法中执行一次后`linkedOrConsumed`设为 true，后续再出发 evaluate()方法就会异常。



# 总结

- JDK中Stream的实现是精炼的高度工程化代码
- Stream的载体虽然是AbstractPipeline，管道结构，但是只用其形，实际求值操作之前会转化为一个多层包裹的Sink结构，也就是前文一直说的Sink链，从编程模式来看，应用的是Reactor编程模式
- Stream目前支持的固有求值执行结构一定是Head(Source Spliterator) -> Op -> Op ... -> Terminal Op的形式，这算是一个局限性，没有办法做到像LINQ那样可以灵活实现类似内存视图的功能
- 不存储数据。 流不是一个存储元素的数据结构。 它只是传递源(source)的数据
- 功能性的(Functional in nature)。 在流上操作只是产生一个结果，不会修改源。 例如filter只是生成一个筛选后的stream，不会删除源里的元素





# references

- [Java8 Stream原理深度解析](https://www.cnblogs.com/Dorae/p/7779246.html)
- [Java 8 Stream 探秘](https://colobu.com/2014/11/18/Java-8-Stream/)
- [Java Stream 源码深入解析](https://xie.infoq.cn/article/a0e6dc7ed0735da5128b6825b)
- [JavaLambdaInternals](https://github.com/CarpenterLee/JavaLambdaInternals)