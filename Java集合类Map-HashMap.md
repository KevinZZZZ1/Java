# Map-HashMap

[TOC]

## 1.概述

HashMap诞生于JDK1.2，是一个经常使用的集合类，它使用hash算法进行散列；允许null键和null值（在计算时，null键值为0）；HashMap不保证插入顺序；HashMap不是线程安全的类，多线程操作时可能会出现问题；



## 2.原理

在使用hash算法进行散列时，会出现hash值相同的情况（即碰撞情况），对于解决碰撞情况，有两种解决办法：

- 再散列探测法；
- 链表法；

HashMap采用链表法处理散列冲突，即如果有多个hash值相同的键值对插入时，会插入到同一个位置，然后在该位置后面形成一个链表，如下图所示：

![avatar](D:\img\HashMap数据结构示意图.jpg)

**在进行增删改等操作时，先是根据键的hash值定位到桶的位置，然后在遍历链表或红黑树来定位具体元素的位置**。

**HashMap的底层数据结构是：数组+链表+红黑树**；



## 3.源码分析

### 3.1继承结构

```java
public class HashMap<K,V> extends AbstractMap<K,V>
implements Map<K,V>, Cloneable, Serializable {
 //... 
}
```

HashMap继承了AbstractMap抽象类，实现了Map接口，Cloneable接口，Serializable接口：

- Map接口：定义了Map数据结构公共的方法；
- Cloneable接口：标识接口，表示实现该接口的类或接口能被克隆；
- Serializable接口：标识接口，表示实现该接口的类或接口能被序列化；

### 3.2属性

#### 3.2.1静态属性

```java
	// 默认HashMap中数组的大小，为16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
	// HashMap中数组的最大值，为2的30次方
    static final int MAXIMUM_CAPACITY = 1 << 30;
	// 默认的负载因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    // 链表红黑树化的阈值，当链表个数大于该阈值时会将链表红黑树化
    static final int TREEIFY_THRESHOLD = 8;
	// 红黑树链表化的阈值，当红黑树中节点个数小于该阈值时就会链表化
    static final int UNTREEIFY_THRESHOLD = 6;
	// 链表红黑树化的阈值，当底层数组的大小大于等于64时才允许链表红黑树化
    static final int MIN_TREEIFY_CAPACITY = 64;
```

从上面的静态属性可以看出：

- **链表想要红黑树化的条件为**：HashMap底层的数组大小要大于等于64，而且要操作链表的长度要大于等于8；

  ```java
  table.length>=64 && len(linkedlist)>=8
  ```

- **HashMap将底层数组的大小默认设置为16**；

- **HashMap允许底层数组最大大小为2的30次方**；

- **HashMap的负载因子为0.75**；

#### 3.2.2变量属性

```java
	// HashMap的底层数组
    transient Node<K,V>[] table;

    /**
     * Holds cached entrySet(). Note that AbstractMap fields are used
     * for keySet() and values().
     */
    transient Set<Map.Entry<K,V>> entrySet;
	// HashMap中键值对的个数
    transient int size;

   	// HashMap被修改的次数
    transient int modCount;
	// 其实是用来记录给定初始容量的大小
    int threshold;
	// 记录给定HashMap的负载因子
    final float loadFactor;
```



### 3.3内部类

#### 3.3.1链表节点类

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
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
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}
```

上面表示的是链表的节点类，包括：`hash`,`key`,`value`,`next`字段，以及处理节点的方法，都很简单此处不在细说；

#### 3.3.2红黑树节点类

```java
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }
    //...
    // 此处省略一些代码
}
```

上面是红黑树节点类，包括：`parent`,`left`,`right`,`prev`,`red`字段，



### 3.4构造方法

```java
/** 构造方法 1 */
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

/** 构造方法 2 */
public HashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

/** 构造方法 3 */
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

/** 构造方法 4 */
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
```

HashMap一共有4个构造函数，这几个构造函数都很简单，都是做一些设定底层数组初始容量，负载因子的操作；

从上面的代码可以看出，构造函数内并没有实际进行数组的创建工作，所以可以猜想（实际上就是）可能是在第一次插入时才进行底层数组的创建（这一点和ArrayList很想，都是延迟加载）；

#### 3.4.1初始容量，负载因子，阈值

一般情况下，我们都是使用第一个构造函数创建HashMap，当我们对时间复杂度和空间复杂度有要求时，我们会对初始容量`initialCapacity `，负载因子`loadFactor `进行调整；

它们之间的关系是：`threshold` = `loadFactor ` * `initialCapacity `

初始容量`initialCapacity `，负载因子`loadFactor `，阈值`threshold`的作用如下表所示：

|        名称        |                       用途                        |
| :----------------: | :-----------------------------------------------: |
| `initialCapacity ` |             HashMap底层数组的初始容量             |
|   `loadFactor `    |                     负载因子                      |
|    `threshold`     | 当前HashMap能放入的最多键值对，超过这个值需要扩容 |

需要注意的是在构造函数3中，有一个语句`this.threshold = tableSizeFor(initialCapacity);`，这里的意思是通过初始容量`initialCapacity `来设置阈值`threshold`，现在我们来看一下`tableSizeFor()`方法：

```java
// 这个方法的作用是：返回第一个大于等于cap值的2的幂
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}
```

刚开始看的时候并不能看懂这是在搞什么，那我们就举个例子吧，假设cap=2^29+1，计算过程如下图所示：

![avatar](D:\img\tableSizeFor方法示意图.jpg)

**所以这里我们大概知道了，在构造函数中threshold中存储的是第一个大于cap的2的幂，比如cap=20的话，threshold=32**；

### 3.5查找

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; // 记录底层数组
    Node<K,V> first, e; // e表示当前处理链表元素，first表示链表第一个元素
    int n; // 表示底层数组的长度，即容量
    K k;
    // 1. 定位键值对所在桶的位置
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 第一个节点就满足查找要求：hash相等 && key相等
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 第一个节点不满足查找要求，查找后面的节点
        if ((e = first.next) != null) {
            // 2. 如果 first 是 TreeNode 类型，则调用黑红树查找方法
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                
            // 2. 对链表进行查找
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}

/**
 * 计算键的 hash 值
 */
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

查找过程的逻辑是：

- 先计算`Key`的hash值；
- 再根据hash值定位到HashMap中桶的位置；
- 在对应位置处的链表或红黑树进行查找；

#### 3.5.1计算`key`的hash值

```java
static final int hash(Object key) {
    int h;
    return (key==null) ? 0 : (h=key.hashCode()) ^ (h>>>16);
}
```

**如果key为null的话，其hash值为0**；

HashMap并没有使用直接使用Object提供的hashCode()方法，而是在此基础上进行了修改，这么做有两个好处：

- 充分使用了h的高16位和低16位，让高16位也能进行计算中：我们在计算hash值所在的桶时，使用的是hash&(n-1)运算，**当n比较小时，`hash&(n-1)`只利用了hash的低位，使得碰撞几率变大，让高位数据与低位数据进行异或 ，可以使得高位也能参与到与计算中**；
- 增加了hash的随机性：如果对象的hashCode()方法产生的hashCode值如果随机性不够好的话，会导致碰撞几率变大，影响性能，**我们在这里对hashCode再进行操作，使得得到hash值随机性更好，不容易发生碰撞**；

#### 3.5.2在数组table中定位对应位置

```java
first = table[(n-1)&hash];
```

**当n为2的幂时，hash&(n-1)就等于hash%n**，由于移位运算速度很快，所以选用移位操作；



### 3.6遍历

```java
public Set<K> keySet() {
    Set<K> ks = keySet;
    if (ks == null) {
        ks = new KeySet();
        keySet = ks;
    }
    return ks;
}

/**
 * 键集合
 */
final class KeySet extends AbstractSet<K> {
    public final int size()                 { return size; }
    public final void clear()               { HashMap.this.clear(); }
    public final Iterator<K> iterator()     { return new KeyIterator(); }
    public final boolean contains(Object o) { return containsKey(o); }
    public final boolean remove(Object key) {
        return removeNode(hash(key), key, null, false, true) != null;
    }
    // 省略部分代码
}

/**
 * 键迭代器
 */
final class KeyIterator extends HashIterator 
    implements Iterator<K> {
    public final K next() { return nextNode().key; }
}

abstract class HashIterator {
    Node<K,V> next;        // next entry to return
    Node<K,V> current;     // current entry
    int expectedModCount;  // for fast-fail
    int index;             // current slot

    HashIterator() {
        expectedModCount = modCount;
        Node<K,V>[] t = table;
        current = next = null;
        index = 0;
        if (t != null && size > 0) { // advance to first entry 
            // 寻找第一个包含链表节点引用的桶
            do {} while (index < t.length && (next = t[index++]) == null);
        }
    }

    public final boolean hasNext() {
        return next != null;
    }

    final Node<K,V> nextNode() {
        Node<K,V>[] t;
        Node<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        if ((next = (current = e).next) == null && (t = table) != null) {
            // 寻找下一个包含链表节点引用的桶
            do {} while (index < t.length && (next = t[index++]) == null);
        }
        return e;
    }
    //省略部分代码
}
```

我们在使用for-each遍历HashMap时，每次遍历得到的顺序都是相同的（但是不同于插入顺序），这是为什么呢？

从源码中我们可以看到：HashMap使用HashIterator进行遍历， HashIterator的逻辑并不复杂，在初始化时，HashIterator 先从桶数组中找到包含链表节点引用的桶。然后对这个桶指向的链表进行遍历。遍历完成后，再继续寻找下一个包含链表节点引用的桶，找到继续遍历。找不到，则结束遍历 ；

如下图所示：

![avatar](D:\img\HashMap遍历.jpg)

- HashIterator 在初始化时，会先遍历桶数组，找到包含链表节点引用的桶，对应图中就是3号桶。随后由 nextNode 方法遍历该桶所指向的链表。遍历完3号桶后，nextNode 方法继续寻找下一个不为空的桶，对应图中的7号桶。之后流程和上面类似，直至遍历完最后一个桶。 遍历上图的最终结果是 `19 -> 3 -> 35 -> 7 -> 11 -> 43 -> 59` 
- **也就是说，在遍历HashMap的时候，是按照底层数组的顺序和链表（红黑树）的顺序进行遍历的**；

### 3.7插入

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; // 记录底层数组
    Node<K,V> p; // 记录对应桶中第一个元素
    int n, i; //n表示底层数组的大小，i表示key在数组中的位置
    // 初始化桶数组 table，table 被延迟到插入新数据时再进行初始化
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 如果桶中不包含键值对节点引用，则将新键值对节点的引用存入桶中即可
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; // 表示当前链表或红黑树处理的元素
        K k;
        // 如果键的值以及节点 hash 等于链表中的第一个键值对节点时，则将 e 指向该键值对
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
            
        // 如果桶中的引用类型为 TreeNode，则调用红黑树的插入方法
        else if (p instanceof TreeNode)  
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 对链表进行遍历，并统计链表长度
            for (int binCount = 0; ; ++binCount) {
                // 链表中不包含要插入的键值对节点时，则将该节点接在链表的最后
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 如果链表长度大于或等于树化阈值，则进行树化操作
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                
                // 条件为 true，表示当前链表包含要插入的键值对，终止遍历
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        
        // 判断要插入的键值对是否存在 HashMap 中
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            // onlyIfAbsent 表示是否仅在 oldValue 为 null 的情况下更新键值对的值
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 键值对数量超过阈值时，则进行扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

插入函数的大概逻辑是：

- 计算出键的hash值；
- 根据hash值定位到对应的桶；
- 判断桶是否为空，如果是空就直接插入；
- 否则在链表或红黑树中查找；
- 如果其中存在相等的键，则进行更新操作；
- 否则将键值对加入链表的尾部；

而插入函数的主要逻辑是在`V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict)`中：

- 当桶数组table为空时，说明数组还未初始化过，使用扩容方法`resize()`初始化table；
- 查找需要插入的键值对是否存在，存在的话根据条件判断是否用新值替换旧值；
- 如果不存在，则将键值对插入链表尾部，并根据链表的长度判断是否需要转为红黑树；
- 判断键值对数量是否大于阈值，如果大于需要扩容；

#### 3.7.1扩容机制





#### 3.7.2链表树化、红黑树链化与拆分





### 3.8删除

```java
public V remove(Object key) {
    Node<K,V> e;
    return (e = removeNode(hash(key), key, null, false, true)) == null ?
        null : e.value;
}

final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        // 1. 定位桶位置
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        // 如果键的值与链表第一个节点相等，则将 node 指向该节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {  
            // 如果是 TreeNode 类型，调用红黑树的查找逻辑定位待删除节点
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                // 2. 遍历链表，找到待删除节点
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
        
        // 3. 删除节点，并修复链表或红黑树
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
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

删除的逻辑是：

- 计算得到key对应的hash值；
- 根据hash值定位到桶的位置；
- 遍历链表或红黑树找到键相同的节点；
- 删除节点；



### 3.9其他

我们在阅读源码的时候会发现，底层数组`table[]`被声明为`transient`，说明数组table不会被序列化，这时就有一个问题，如果table不序列化的话，那么如何进行还原呢？

HashMap中使用`readObject()/writeObject()`进行序列化和反序列化，**而且只是把键值对序列化了**，再根据键值对重建HashMap，这么做的目的是：

- **table[]一般是不满的，序列化未使用部分浪费空间**；
- **同一个键值对在不同JVM下，会有不同的hash值**：因为hashCode()方法是一个本地方法，所以和具体的实现有关，同一个键在不同平台下回产生不同的hashCode；



## 4.总结







## 5.参考文献

<https://segmentfault.com/a/1190000012926722#articleHeader0> 

<https://allenwu.itscoder.com/hashmap-analyse> 

<https://juejin.im/entry/58f3052aac502e006c2e8e18> 

<https://zhuanlan.zhihu.com/p/34280652> 

<http://www.tianxiaobo.com/2018/01/11/%E7%BA%A2%E9%BB%91%E6%A0%91%E8%AF%A6%E7%BB%86%E5%88%86%E6%9E%90/> 
