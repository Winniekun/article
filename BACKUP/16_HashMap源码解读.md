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
    - hash值 (key经过hash之后的值)
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
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
```

```java
// 为了使计算出的hash更分散
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```
> 哈希计算， 取得key的hashcode后，高16位与整个Hash异或运算重新计算hash值

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    // 对应table数组
    Node<K,V>[] tab;
    // 对应位置的 Node 节点
    Node<K,V> p; 
    // n table的长度
    // 原tab中对应放入Node的位置
    int n, i;
    // 如果 table 未初始化，或者容量为 0 ，则进行扩容
    // (第一次见这种赋值和判断融合的操作，学到了学到了)
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 如果对应的位置的Node节点为空，则直接创建 Node 节点即可。
    // (n-1)&hash 获取所在table的位置
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        // 哈希冲突解决
        // key 在 HashMap 对应的老节点
        Node<K,V> e; 
        K k;
        // hash相等 key相等 执行覆盖
        if (p.hash == hash &&((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 为红黑树的节点
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        // 为Node节点，则说明是链表，且不是覆盖操作。需要遍历查找
        else {
            // 判断什么时候进行链表转红黑树，什么时候红黑树转链表
            for (int binCount = 0; ; ++binCount) {
                // 如果链表遍历完了都没有找到相同key的元素，说明该key对应的元素不存在，则在链表                   // 最后插入一个新节点
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 链表的长度如果数量达到 TREEIFY_THRESHOLD（8）时，则进行树化。
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果待插入的key在链表中找到了，则退出循环
                if (e.hash == hash&&((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // 覆盖操作， 也就是相同的key的value进行覆盖
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // onlyIfAbsent进行判断是否需要覆盖oldValue
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 如果超过阀值，则进行扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
以下图片为`putVal`方法的总体逻辑
![hashmap_put.png](https://i.loli.net/2020/03/27/w281NB3JMPZEbSG.png)

### 扩容机制
```java
// 扩容方法
final Node<K,V>[] resize() {
    	// oldTab 表示当前的哈希桶
        Node<K,V>[] oldTab = table;
    	// oldCap表示当前哈希桶的容量 length
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
    	// 当前哈希桶的阀值
        int oldThr = threshold; 
    	// 初始化新的容量和阀值
        int newCap, newThr = 0;
    	// 1. 如果当前的容量不为空
        if (oldCap > 0) {
            // 如果当前的容量已经达到了上限
            if (oldCap >= MAXIMUM_CAPACITY) {
                // 阀值为2^32-1
                threshold = Integer.MAX_VALUE;
                // 返回当前的哈希桶不再扩容
                return oldTab;
            }
            // 否者新的容量为原先容量的两倍
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)// 如果旧的容量大于等于默认的初始容量						16
                // 新的阀值也为原先的两倍
                newThr = oldThr << 1; // double threshold
        }
    	// 2. 如果当前表是空的，但是有阈值。代表是初始化时指定了容量、阈值的情况(上述的第三种构造方法)
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr; // 新的容量为旧的阀值
        // 3. 此处的判断用于初始化 
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
    	//如果新的阈值是0，对应的是  当前表是空的，但是有阈值的情况
        if (newThr == 0) {
            //根据新表容量 和 加载因子 求出新的阈值
            float ft = (float)newCap * loadFactor;
            // 进行越界修复
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
    	// 更新阀值
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        // 更新哈希桶的引用
        table = newTab;
    	// 如果以前的哈希桶有元素
    	// 以下执行将原哈希桶中的元素转入新的哈希桶中
        if (oldTab != null) {
            // 遍历老的哈希桶
            for (int j = 0; j < oldCap; ++j) {
                // 取出当前的节点e
                Node<K,V> e;
                // 如果当前桶中有元素, 则将链表复制给e
                if ((e = oldTab[j]) != null) {
                    // 将原哈希桶置空以便GC
                    oldTab[j] = null;
                    // 当前链表中只有一个元素, 没有发生哈希碰撞
                    if (e.next == null)
                        //直接将这个元素放置在新的哈希桶里。
                        //注意这里取下标 是用 哈希值 与 桶的长度-1 。 
                        //由于桶的长度是2的n次方，这么做其实是等于 一个模运算。但是效率更高
                        newTab[e.hash & (newCap - 1)] = e;
                    // 发生碰撞, 而且节点的个数超过8, 则转化为红黑树
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //发生碰撞, 节点个数＜8 则进行处理, 依次放入新的哈希桶对应的下标位置
                    //因为扩容是容量翻倍，所以原链表上的每个节点，现在可能存放在原来的下								    //标,即low位， 或者扩容后的下标，即high位。 high位=low位+原哈希桶容量
                    else { // preserve order
                         //低位链表的头结点、尾节点
                        Node<K,V> loHead = null, loTail = null;
                         //高位链表的头结点、尾节点
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                       // 将同一桶中的元素根据(e.hash & oldCap)是否为0进行分割，分成高位链表和低位							链表以完成rehash
                        do {
                            next = e.next;
                              //这里又是一个利用位运算 代替常规运算的高效点： 利用哈希值与旧的容量，
                              //可以得到哈希值取模后，是大于等于oldCap还是小于oldCap，等于0代表小于                               //oldCap，应该存放在低位，否则存放在高位
                            if ((e.hash & oldCap) == 0) {
                               
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```

1. 如果使用的是默认构造方法,  则第一次插入元素的时候初始化为默认的容量16, 扩容阀值为12
2. 如果使用的是非默认方法,  则第一次插入元素时初始化容量为扩容阀值, 扩容的阀值为初始化的时候, 扩容门槛在构造方法里等于传入容量向上最近的2的n次方
3. 如果旧容量＞0, 则新容量为旧容量的2倍, 但不超过最大容量2^30, 新的扩容阀值为旧的扩容阀值2倍
4. 创建新的容量桶
5. 搬移元素，原链表分化成两个链表，低位链表存储在原来桶的位置，高位链表搬移到原来桶的位置加旧容量的位置

### 删
```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    // 这些和增里面是差不多的
    // tab：桶数组 p：待删除节点的前驱节点 n: 桶数组大小 index: 桶数组第i个位置
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null&&(n = tab.length)>0 &&(p=tab[index=(n-1)&hash])!= null) {
        // node为需要删除的节点
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        // 找到对应的key进行remove核心
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            // 如果node ==  p，说明是链表头是待删除节点
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```
> 除去复杂的判断，也就是单链表删除指定的节点，共有两种情况： 1.删除的为头结点。2. 删除的为中间节点

### 查
```java
public V get(Object key) {
    HashMap.Node e;
    return (e = this.getNode(hash(key), key)) == null ? null
}
// 根据key获取对应的节点
final HashMap.Node<K, V> getNode(int hash, Object key) {
    HashMap.Node[] tab;
    HashMap.Node first;
    int n;
    if ((tab = this.table) != null && (n = tab.length) > 0 && (first = tab[n - 1 & hash]) != null) {
        Object k;
        if (first.hash == hash && ((k = first.key) == key || key != null && key.equals(k))) {
            return first;
        }
        HashMap.Node e;
        if ((e = first.next) != null) {
            if (first instanceof HashMap.TreeNode) {
                return ((HashMap.TreeNode)first).getTreeNode(hash, key);
            }
            do {
                if (e.hash == hash && ((k = e.key) == key || key != null && key.equals(k))) {
                    return e;
                }
            } while((e = e.next) != null);
        }
    }
    return null;
}
```
> 使用`first.hash == hash && ((k = first.key) == key || key != null && key.equals(k))`判断是否为同一个key(因为相同的hash可能并不是同一个key，因为先是使用`(n - 1) & hash`寻找桶的位置，之后再依次查找key)

## 总结&面试相关问题

1. HashMap是一种散列表, 使用(数组+链表+红黑树)的结构

2. HashMap的默认初始容量为16, 默认装载因子为0.75, 且容量总是2^n

3. 正常情况下,  扩容为原来的两倍

4. 桶的长度小于64 时, 不会进行树化

5. 桶的长度大于64且链表长度大于8时, 进行树化

6. 当单个桶中的元素小于6时, 进行反树化(转化为链表)

7. 查询时间为$O(1)$ 添加时间O(1)
   1.判断key，根据key算出索引。
   2.根据索引获得索引位置所对应的键值对链表。
   3.遍历键值对链表，根据key找到对应的Entry键值对。
   4.拿到value。
   **分析：**
   以上四步要保证HashMap的时间复杂度O(1)，需要保证每一步都是O(1)，现在看起来就第三步对链表的循环的时间复杂度影响最大，链表查找的时间复杂度为O(n)，与链表长度有关。我们要保证那个链表长度为1，才可以说时间复杂度能满足O(1)。但这么说来只有那个hash算法尽量减少冲突，才能使链表长度尽可能短，理想状态为1。因此可以得出结论：HashMap的查找时间复杂度只有在最理想的情况下才会为O(1)，

8. HashMap是非线程安全的
