# [ThreadLocal源码解读](https://github.com/Winniekun/article/issues/15)

## 序
多线程共享可变数据时，涉及到线程见同步的问题，并不是所有的时候，都要用到共享数据，所以就需要线程封闭出场了。数据都被封闭在各自的线程之中，就不需要进行同步，这种通过将数据封闭在线程内而避免使用同步的技术称之为线程封闭。

## 适用场景
线程间数据隔离，各线程的 ThreadLocal 互不影响
方便同一个线程使用某一对象，避免不必要的参数传递
全链路追踪中的`traceId`或者流程引擎中上下文的传递一般采用`ThreadLocal`
`Spring`事务管理器采用了`ThreadLocal`
`Spring MVC` 的 `RequestContextHolder` 的实现使用了`ThreadLocal`


## Demo
```java
public class ThreadLocalDemo {
    public static final ThreadLocal<String> THREAD_LOCAL = new ThreadLocal<>();
    public static final ThreadLocal<String> THREAD_LOCAL_NEXT = new ThreadLocal<>();

    @Test
    public void demo() {
        new ThreadLocalDemo().threadLocalTest();
    }

    public void threadLocalTest() {
        THREAD_LOCAL.set("wkk");
        THREAD_LOCAL_NEXT.set("cjsq");
        String v = THREAD_LOCAL.get();
        String v2 = THREAD_LOCAL_NEXT.get();
        System.out.println("子线程 执行前， " + Thread.currentThread().getName() + "线程取到的值：" + v + " " + v2);

        Thread child = new Thread(new Runnable() {
            @Override
            public void run() {
                String v = THREAD_LOCAL.get();
                String v2 = THREAD_LOCAL_NEXT.get();
                System.out.println(Thread.currentThread().getName() + "线程取到的值：" + v + v2);
                // 设置 threadLocal
                THREAD_LOCAL.set("hhh");
                THREAD_LOCAL_NEXT.set("cjpl");
                v = THREAD_LOCAL.get();
                v2 = THREAD_LOCAL_NEXT.get();
                System.out.println("重新设置之后，" + Thread.currentThread().getName() + "线程取到的值为：" + v + " " + v2);
                System.out.println(Thread.currentThread().getName() + "线程执行结束");
            }
        }, "子线程");
        child.start();

        try {
            child.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        v = THREAD_LOCAL.get();
        v2 = THREAD_LOCAL_NEXT.get();
        System.out.println("子线程线程执行之后，" + Thread.currentThread().getName() + "线程取到的值：" + v + " " + v2);

    }
}
```

### 输出
```
子线程 执行前， main线程取到的值：wkk cjsq
子线程线程取到的值：nullnull
重新设置之后，子线程线程取到的值为：hhh cjpl
子线程线程执行结束
子线程线程执行之后，main线程取到的值：wkk cjsq
```

## 源码解读
### 重要属性
```java
// Thread类
ThreadLocal.ThreadLocalMap threadLocals = null;


// 当前threadlocal的hashcode
// 用于用于计算该ThreadLocal在线程的threadlocals（map）中的索引位置
private final int threadLocalHashCode = nextHashCode();
// hash表的阈值，黄金分割比
private static final int HASH_INCREMENT = 0x61c88647;
// 保证了在一台机器中每个 ThreadLocal 的 threadLocalHashCode 是唯一的
private static AtomicInteger nextHashCode = new AtomicInteger();
// 用于当前threadlocal的hashcode
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}

// 初始话的操作
protected T initialValue() {
    return null;
}
// lambda的初始化实现
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    return new SuppliedThreadLocal<>(supplier);
}
```

###  HashCode散列成果验证
可以通过和String的hashcode做个对比，然后验证其使用这种方式的hash结果如何
```java
private static AtomicInteger nextHashCode = new AtomicInteger();
private static final int HASH_INCREMENT = 0x61c88647;
private static final int SIZE = 32;
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}

@Test
public void test_idx() {
    int hashCode = 0;
    Set<Integer> setThreadLocal = new HashSet<>();
    Set<Integer> setNormal = new HashSet<>();
    for (int i = 0; i < SIZE; i++) {
        hashCode = nextHashCode();
        int threadLocalHashCode = hashCode & (SIZE - 1);
        int normalHashCode = String.valueOf(i).hashCode() & (SIZE - 1);
        setThreadLocal.add(threadLocalHashCode);
        setNormal.add(normalHashCode);
        System.out.println(i + " ThreadLocal的散列：" + threadLocalHashCode + " 普通散列：" + normalHashCode);
    }
    double threadLocalRate = setThreadLocal.size() / (double)SIZE * 100;
    double normalRate = setNormal.size() / (double)SIZE * 100;
    System.out.println("ThreadLocal的散列：" + threadLocalRate + "% " + " 普通散列：" + normalRate + "% ");
}
// ThreadLocal的散列：100.0%  普通散列：68.75%
```
### 形象理解
<img width="1073" alt="Image" src="https://github.com/user-attachments/assets/ed90aed9-2498-430f-b401-ee48d2429f32" />

如果类比为`HashMap`的话，我们可以将`threadlocal`类比为`key`，然后被封闭的数据类比为`value`。不过稍微不同的是，`HashMap`中将`key`和`value`做映射的操作是`map.put(key, value)`，而`threadlocals`中，将`Threadlocal`和被封闭的数据做映射的操作是`threadlocal.set(xxx)`。

### 思路转换
emmmm怎么说呢，就是我们使用普通map的时候，操作的是一个map，获取key的value时，直接通过map.get()即可。
但是我们使用threadlocal时，其本身是一个key，怎么获取对应的value呢？
1. 先获取map（每个线程独有的map，所以根据当前线程获取map）
2. 然后通过map.get(key)获取value
3. 返回value

### ThreadLocalMap
```java
static class ThreadLocalMap {
    /**
     * The entries in this hash map extend WeakReference, using
     * its main ref field as the key (which is always a
     * ThreadLocal object).  Note that null keys (i.e. entry.get()
     * == null) mean that the key is no longer referenced, so the
     * entry can be expunged from table.  Such entries are referred to
     * as "stale entries" in the code that follows.
     */
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;
        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
  // 下面的成员变量和hashmap同理
  // 初始化容量
    private static final int INITIAL_CAPACITY = 16;
    // 存储ThreadLocal的键值对，长度为2的幂次
    private Entry[] table;
    /**
     * The number of entries in the table.
     */
    private int size = 0;
    /**
     * The next size value at which to resize.
     */
    private int threshold; // Default to 0
}
```
ThreadLocalMap就是一个map，和`HashMap`类似，不过有些机制不同。

- hash冲突解决方式
- ThreadLocalMap处理hash冲突的方式为线型探测
- HashMap使用的是拉链法

## ThreadLocal内存泄漏
根据注释和代码，了解到了ThreadLocalMap内部的每个entry中，key设置为`弱引用`的原因，不过value还是正常的引用。这也就导致了如果ThreadLocal没有外部的强引用时，只要发生GC，就会被回收。这样ThreadMap中的key就变成了null，`但是value被Entry引用，Entry被ThreadLocalMap引用，ThreadLocalMap被Thread引用，这也就说明了只要，线程不终止，value的值一直无法被回收，所以可能会出现内存泄漏的现象`
为了避免这种情况，在需要使用threadlocal之后，需要我们手动remove掉。防止 ThreadLocalMap 中 Entry 一直保持对 value 的强引用，导致 value 不能被回收
## remove
```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}

// 根据当前线程获取其map
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

// 因为使用的线型探测法
// 所以要采用线型探测法找到对应的位置
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    // 根据threadlocal获取其hashcode值
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}

private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}

private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    // expunge entry at staleSlot
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;
    // Rehash until we encounter null
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;
                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```
- 通过当前线程获取该线程的map
- 然后调用map.remove方法

## set
```java
/**
 * Sets the current thread's copy of this thread-local variable
 * to the specified value.  Most subclasses will have no need to
 * override this method, relying solely on the {@link #initialValue}
 * method to set the values of thread-locals.
 *
 * @param value the value to be stored in the current thread's copy of
 *        this thread-local.
 */
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

```

set方法的作用是把我们想要存储的value给保存方法，只要流程是：

1. 先获取当亲的线程
2. 根据当前线程获取该线程的map
3. 判断map是否为空
     - 为空
              - 创建map
      - 不为空
              - 在map中放入元素

## set(ThreadLcoal<?> key, Object value)
```java
// 在map中存储键值对<key, value>
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    // 遍历一段连续的元素，以查找匹配的 ThreadLocal 对象
    for (Entry e = tab[i]; e != null; e = tab[i = nextIndex(i, len)]) {
        // / 获取该哈希值处的ThreadLocal对象
        ThreadLocal<?> k = e.get();
        // 键值ThreadLocal匹配，直接更改map中的value
        if (k == key) {
            e.value = value;
            return;
        }
        // 若 key 是 null，说明 ThreadLocal 被清理了，直接替换掉
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // 直到遇见了空槽也没找到匹配的ThreadLocal对象，那么在此空槽处安排ThreadLocal对象和缓存的value
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}

```
通过这两种set方式，就更加证明了思路的转换，我们在set时，都需要先获取当前线程获取map，然后再对整个map遍历然后放入value、或者直接放入key、value。

## get
```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}

// 根据threadlocal获取entry
private Entry getEntry(ThreadLocal<?> key) {
    // 计算hashcode，然后获取对应的索引位置
    int i = key.threadLocalHashCode & (table.length - 1);
    // 根据索引位置获取对应的entry
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private T setInitialValue() {
     T value = initialValue();
     Thread t = Thread.currentThread();
     ThreadLocalMap map = getMap(t);
     if (map != null)
         map.set(this, value);
     else
         createMap(t, value);
     return value;
 }
```
1. 先获取当前线程
2. 获取当前线程的map
3. map != null
     1. 通过key直接获取对应entry
     2. 返回entry.value
4. map == null
      1. 初始化map

## 总结
`threadlocal`本质属于某种map的一个key值，只不过该值通过泛型实现，支持各种类型。和基础的map不同的是，可以直接通过`key.set(value)`实现`key-value`的映射。而这种特殊的map是属于Thread级别的成员变量，多个线程之间该变量互不影响，所以这也就是我们所说的线程本地存储地方。不过需要注意的是，该key是虚引用，需要注意内存泄露的问题，所以在使用过threadlocal之后，记得及时remove。内部的threadlocalmap和hashmap的实现机制类似，不过区别就是对于hash冲突的解决方式为线型探测

### 内存泄露问题
#### 什么是内存泄漏
内存泄漏指的是，当某一个对象不再有用的时候，占用的内存却不能被回收，这就叫作内存泄漏。
因为通常情况下，如果一个对象不再有用，那么我们的垃圾回收器 GC，就应该把这部分内存给清理掉。这样的话，就可以让这部分内存后续重新分配到其他的地方去使用；否则，如果对象没有用，但一直不能被回收，这样的垃圾对象如果积累的越来越多，则会导致我们可用的内存越来越少，最后发生内存不够用的 OOM 错误。
下面我们来分析一下，在 ThreadLocal 中这样的内存泄漏是如何发生的。

#### Key 的泄漏
分析了 ThreadLocal 的内部结构之后，知道了每一个 Thread 都有一个 ThreadLocal.ThreadLocalMap 这样的类型变量，该变量的名字叫作 threadLocals。线程在访问了 ThreadLocal 之后，都会在它的 ThreadLocalMap 里面的 Entry 中去维护该 ThreadLocal 变量与具体实例的映射。

我们可能会在业务代码中执行了 ThreadLocal instance = null 操作，想清理掉这个 ThreadLocal 实例，但是假设我们在 ThreadLocalMap 的 Entry 中强引用了 ThreadLocal 实例，那么，虽然在业务代码中把 ThreadLocal 实例置为了 null，但是在 Thread 类中依然有这个引用链的存在。

GC 在垃圾回收的时候会进行可达性分析，它会发现这个 ThreadLocal 对象依然是可达的，所以对于这个 ThreadLocal 对象不会进行垃圾回收，这样的话就造成了内存泄漏的情况。

## references
- Java并发编程
- 面试官系统精讲Java源码及大厂真题
- Java并发编程78讲
- [【细谈Java并发】谈谈ThreadLocal](https://benjaminwhx.com/2018/04/28/[%E7%BB%86%E8%B0%88Java%E5%B9%B6%E5%8F%91]%E8%B0%88%E8%B0%88ThreadLocal/)
- [一文搞懂 ThreadLocal 原理](https://www.cnblogs.com/wupeixuan/p/12638203.html)
[为何每次用完 ThreadLocal 都要调用 remove()？——内存泄漏](https://blog.csdn.net/vincent_wen0766/article/details/108599292)



