# [HashMap源码解读](https://github.com/Winniekun/article/issues/16)

## 序
常看常新，重点是思想

## 概述
![HashMap](https://i.loli.net/2020/03/13/q9tI7BCHXjJdopn.png)
HashMap用于存放键值对，通过哈希表方式实现Map接口，是常用的Java集合之一，**JDK1.8之前HashMap由数组+链表组成, 其使用的是数组进行存储, 之后使用链接法解决哈希冲突.** JDK1.8之后HashMap在解决哈希冲突时, 与之前不同的是, 当链表长度大于阈值(默认是8), 将链表转化为红黑树, 避免过长的链表从而降低性能(过长的话, 就相当于使用的是链表存储的了,每次的查找都会导致O(n)的时间), 同时转化为红黑树的时候也会进行判断（将链表转换成红黑树前会判断，如果当前数组的长度小于 64，那么会选择先进行数组扩容，而不是转换为红黑树），以减少搜索时间。

同时HashMap的源码中，充斥个各种位运算代替常规运算的地方，以提升效率：
- 与运算替代模运算。用 hash & (table.length-1) 替代 hash % (table.length)
- 用if ((e.hash & oldCap) == 0)判断扩容后，节点e处于低区还是高区。

HashMap的底层为数组, 可以将其称之为哈希桶，每个桶里放的是链表。其是线程不安全的，允许key为null，value为null，遍历时无序。


### 常见寻址方法

#### 开放寻址法

-  线性探测法
    - 如果哈希冲突发生，就按固定步长（通常是 1）向后寻找下一个空位。
- 二次探测法（平方探测法）
- 双重散列法

#### 链表法
> 对于不同的关键字可能会通过散列函数映射到同一个地址。为了避免发生分歧，将同样的哈希结果，但是不同关键字的元素，可以将它们存储在一个线性链表中。

**HashMap中的解决哈希冲突的方式就链表法**

## 相关依赖
![HashMap.png](https://i.loli.net/2020/06/20/gB13O6PxHU4GwhF.png)
- 实现了Serializable接口
- 实现了Cloneable接口
- 实现了Map接口，并继承了AbstractMap类

## 阅读套路
按照正常的逻辑，首先基础的数据构造, 然后了解其构造方法，之后了解常用的API(CRUD)操作即可，以下操作本文均已阐述。
- [x] 增
- [x] 删
- [x] 改
- [x] 查

## 基础数据结构
### 整体结构
![HashMap.png](https://i.loli.net/2020/06/20/8SI6gLjBp7kMHsv.png)
在添加元素的时候, 会先根据hash值计算出其应在的位置：
- 若是没有冲突，则直接放入
- 若是有冲突, 则将该元素以俩表的形式插入到链表尾部
- 若是链表的长度超过阀值(默认是8), 则将链表转化为红黑树, 提高效率


数组的查询为O(1)，链表的查询为O(k)，红黑树的查询为O(logk)，k表示为桶中的元素的个数，所以当元素很多的时候，转化为红黑树能提高查询效率。
### 单节点构造
```jav
static class Node<K,V> implements Map.Entry<K,V> {
    # has值
    final int hash;
    # key
    final K key;
    # value
    V value;
    # next引用
    Node<K,V> next;
    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }
    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }
    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }
    public final boolean equals(Object o) {
        if (o == this)
            return true;
        return o instanceof Map.Entry<?, ?> e
                && Objects.equals(key, e.getKey())
                && Objects.equals(value, e.getValue());
    }
}
```
**总结:**
- 一个节点的hash值是将key的哈希值和value的哈希值异或得到
- 单个节点包含的信息有
    - key
    - value
    - hash值
    - next指针（指向下一个节点）

### TreeNode构造（红黑树节点）
```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> 
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }
    /**
     * Returns root of tree containing this node.
     */
    final TreeNode<K,V> root() {
        for (TreeNode<K,V> r = this, p;;) {
            if ((p = r.parent) == null)
                return r;
            r = p;
        }
    }
```

### 类属性
```java
//初始容量为16 注释中说明了 必须为2的幂次
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
// 最大容量 2^30
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认负载因子0.75(若是构造函数中未说明)
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 桶中的节点数量大于8时转为红黑树
static final int TREEIFY_THRESHOLD = 8;
// 桶中的节点数量小于6时 转为链表
static final int UNTREEIFY_THRESHOLD = 6;
// 桶中结构转化为红黑树对应的table的最小大小
static final int MIN_TREEIFY_CAPACITY = 64;

// 哈希桶 存放链表 总是2的倍数
transient Node<K,V>[] table;
// 存放具体元素的集
transient Set<Map.Entry<K,V>> entrySet;
// 存放元素的个数，注意这个不等于数组的长度。
transient int size;
// 每次扩容和更改map结构的计数器
transient int modCount;
// threshold = 哈希桶.length * loadFactor; 当桶的使用数量操作该值, 进行扩容
int threshold;
// 装载因子，用于计算哈希表元素数量的阈值。  
final float loadFactor;
```

### 构造方法
#### 无参构造方法
```java
public HashMap() {
    // 默认构造方法 将负载因子设置为默认值 0.75, 其他的属性均为默认
    // 假设默认初始化空间大小为16, 则元素数量达到16*0.75=12时, 会进行扩容.
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```
> 这是一个默认构造器，潜在的问题是初始容量16太小了，可能中间需要不断扩容的问题，会影响插入的效率。

#### 指定初始化容量的构造方法
```java
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}
```
> 用这种构造函数创建HashMap的对象，如果知道map要存放的元素个数，可以直接指定容量的大小， 减除不停的扩容，提高效率，内部调用下述的构造方法

#### 指定初始化容量和负载因子的构造方法
```java
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}

static final int tableSizeFor(int cap) {
    int n = -1 >>> Integer.numberOfLeadingZeros(cap - 1);
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```
> 先是对初始化容量、装载因子的合理性检测，之后进行赋值，loadFactor: 自定义值， 默认为0.75，threshold: 取与cap最近的且大于cap的2的次方值


#### Map转换构造方法
```java
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}

// 其实就是一个一个取出m中的元素调用putVal,一个个放入table中的过程。
final void putMapEntries(Map<? extends K, ? extends V> m, boolean evict) {
    int s = m.size();
    if (s > 0) {
        // ① table为空。说明还没初始化，适合在构造方法的情况
        if (table == null) { // pre-size
            // +1.0F 的目的，是因为下面 (int) 直接取整，避免不够。
            float ft = ((float)s / loadFactor) + 1.0F;
            // 修正，防止超出
            int t = ((ft < (float)MAXIMUM_CAPACITY) ?
                     (int)ft : MAXIMUM_CAPACITY);
            // 如果计算出来的 t 大于阀值，则计算新的阀值
            if (t > threshold)
                threshold = tableSizeFor(t);
        }
        // ② 如果 table 非空，说明已经初始化，需要不断扩容到阀值超过 s 的数量
        else if (s > threshold)
            resize();
        // ③ 正常情况
        for (Map.Entry<? extends K, ? extends V> e : m.entrySet()) {
            K key = e.getKey();
            V value = e.getValue();
            // 添加元素
            putVal(hash(key), key, value, false, evict);
        }
    }
}

```
> 整体的意思就是必须保证新的map的table容量够，如何保证？确保原map的元素数量小于新map的threshold。


#### 总结
1. 为什么 threshold 要返回大于等于 initialCapacity 的最小 2 的 N 次方？
> 计算元素table中的位置的时候，概述中有讲，使用的是hash & (table.length-1) 而不是我们认为的 hash % table.length，而这两个方式在table.length是2的N次方的时候是等价的，从性能上而言，&性能更好，所以，仅需要满足数组的容量n尽可能为2的N次方即可。

2. 为什么给的是初始化容量， 计算的却是阀值，而不是容量？
> 1.8中， 将table的初始化放入了resize()中，并且在类属性中也注意到了没有capacity这个属性， 所以这里只能重新计算threshold，而resize()后面就会根据threshold来重新计算capacity，来进行 table数组的初始化，然后再重新按照装载因子计算threshold。



### 增、改

### 扩容机制

### 删

### 查
