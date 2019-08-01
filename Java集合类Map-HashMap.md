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

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table; // 用于存储底层数组
    int oldCap = (oldTab == null) ? 0 : oldTab.length; // 得到数组的大小，0表示未被初始化的情况
    int oldThr = threshold; // 阈值 
    int newCap, newThr = 0; // 新的数组大小和阈值
    // 如果 table 不为空，表明已经初始化过了
    if (oldCap > 0) {
        // 当 table 容量超过容量最大值，则不再扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        } 
        // 按旧容量和阈值的2倍计算新容量和阈值的大小
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    } else if (oldThr > 0) // initial capacity was placed in threshold
        /* oldCap==0 && oldThr>0的情况，是调用HashMap(int)或HashMap(int, float)的情况
         * 初始化时，将 threshold 的值赋值给 newCap，
         * HashMap 使用 threshold 变量暂时保存 initialCapacity 参数的值
         */ 
        // 由构造函数里的源码可知，在给定initialCapacity参数的构造函数在调用tableSizeFor()函数后的返回结果存储在threshold中（也就是oldThr中），总之就是oldThr就是初始容量
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        /* oldCap==0 && oldThr==0 的情况就是HashMap为空，还没被初始化过
         * 调用无参构造方法时，桶数组容量为默认容量，
         * 阈值为默认容量与默认负载因子乘积
         */
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    
    // newThr 溢出导致为 0 时，按阈值计算公式进行计算
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    // 将newThr赋值给threshold
    threshold = newThr;
    // 创建新的桶数组，桶数组的初始化也是在这里完成的
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    
    table = newTab;
    if (oldTab != null) {
        // 如果旧的桶数组不为空，则遍历桶数组，并将键值对映射到新的桶数组中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e; // 表示当前处理的节点
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null; // 方便gc
                // oldTab[j]只有一个节点的情况
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 重新映射时，需要对红黑树进行拆分
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                // 将oldTab[j]对应链表中的元素映射到新的桶数组中
                else { // preserve order
                    // 假设e.hash&(oldCap-1)值为m
                    // 用于记录节点e.hash&(newCap-1)的值仍然为m的情况，也就是在原来位置情况
                    Node<K,V> loHead = null, loTail = null;
                    // 用于记录节点e.hash&(newCap-1)的值为m+oldCap的情况
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    // 遍历链表，并将链表节点按原顺序进行分组
                    do {
                        next = e.next;
                        // 该节点e仍然在原来桶位置
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 该节点e在m+oldCap的位置
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 将分组后的链表映射到新桶中
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

**所谓扩容是指，对HashMap底层数组的大小进行扩大**，从上面源码可以看出，`resize()`函数的逻辑是：

- 计算新桶数组的大小newCap，新阈值newThr；
- 根据计算出的newCap创建新的桶数组；
- 将键值对节点重新映射到新的桶数组中，如果节点是TreeNode类型，需要拆分红黑树，如果是链表节点则按节点顺序进行分组；

##### 计算newCap和newThr

计算newCap和newThr的逻辑过程如下：

```java
// 第一个分支
if(oldCap>0){
    ...
    // 嵌套分支
    if(oldCap>=MAXIMUM_CAPACITY){

    }else if((newCap=oldCap<<1)<MAXIMUM_CAPACITY && oldCap>=DEFAULT_INITIAL_CAPACITY){

    }    
    
}else if(oldThr>0){
    ...
}else{
    ...
}

// 第二个分支
if(newThr==0){
    ...
}
```

**第一个分支：**

|         条件         |                     情况说明                     |                             备注                             |
| :------------------: | :----------------------------------------------: | :----------------------------------------------------------: |
|       oldCap>0       | HashMap已经被初始化过，直接对HashMap底层数组扩容 |                                                              |
| oldCap==0&&oldThr>0  |       HashMap未被初始化过，而且threshold>0       | 在调用HashMap(int)、HashMap(int, float)构造方法时会出现这种情况 |
| oldCap==0&&oldThr==0 |      HashMap未被初始化过，而且threshold==0       |              在调用HashMap()无参构造函数时出现               |

第一个条件和第三个条件里的代码都很好理解，这里主要说一下第二个条件：

首先这里的`oldThr`也就是`threshold`，在介绍[构造函数3](#1)的时候，我们介绍过`tableSizeFor(int)`方法，该方法输入的参数`initialCapacity`，输出结果是第一个大于`initialCapacity`的2的幂；在构造函数3中把这个输出赋值给了`threshold`变量，所以现在我们有：

```java
newCap = oldThr = threshold = tableSizeFor(initialCapacity);
```

**所以我们初始化传入的initialCapacity参数会经过threshold赋值给新的容量**；

- 嵌套分支

  | 条件                      | 情况说明                                     | 备注                                                         |
  | ------------------------- | -------------------------------------------- | ------------------------------------------------------------ |
  | oldCap>= 2^30             | 桶数组大小大于2^30                           | 不再扩容                                                     |
  | newCap<2^30 && oldCap>=16 | 新桶数组大小小于最大值，旧桶数组的大小大于16 | 新桶数组大小变成旧桶数组大小的两倍，新的阈值也变成原来阈值的两倍； |

  在第二种情况下会出现`newThr`移位溢出的情况，具体如下图所示：

  ![avatar](D:\img\newThr移位溢出.jpg)

**第二个分支：**

|   条件    |               情况说明               |           备注           |
| :-------: | :----------------------------------: | :----------------------: |
| newThr==0 | 第一个分支在计算newThr出现溢出的情况 | 使用阈值计算公式重新计算 |

```java
float ft = (float)newCap * loadFactor;
```

##### 链表节点映射

HashMap底层的桶数组经过扩容之后，由于其数组大小发生了变化，所以原来在同一链表里的元素可能在新的桶数组中会在不同的位置（hash&(newCap-1)不一样了），但是这种变化并非没有规律的，首先看一个例子：

假设桶数组大小n为16，虽然hash1与hash2值不相同，但是在进行取余运算时只用到了后4位，所以得到了相同的结果；

![avatar](D:\img\hash求模.jpg)

扩容之后，n变成了32，再进行hash&(n-1)的运算结果如下图所示：

![avatar](D:\img\hash求模1.jpg)

**从上面可以看出，扩容后的取余运算会根据hash值分成两组：（记原来余数为m）**

- hash&(newCap-1)答案仍然为m；
- hash&(newCap-1)答案为m+oldCap;

**判断条件就是hash&oldCap==1**;



介绍完上面的理论之后，我们现在来谈谈链表节点映射部分，例如我们有个HashMap如下图所示，在进行映射操作的时候，我们需要遍历链表，判断hash&oldCap==1是否成立，如果成立就将该节点尾插到头指针为hiHead、尾指针为hiTail的链表中；否则将该节点尾插入到头指针为loHead、尾指针为loTail的链表中；

![avatar](D:\img\数组扩容链表映射1.jpg)

插入结果如下图所示：

![avatar](D:\img\数组扩容链表映射2.jpg)

最后把loHead链表赋值给table[m]，把hiHead链表赋值给table[m+oldCap]，具体如下图所示：

![avatar](D:\img\数组扩容链表映射3.jpg)



#### 3.7.2链表树化、红黑树链化与拆分

##### 链表树化

```java
// 当链表长度大于8是需要进行红黑树化
static final int TREEIFY_THRESHOLD = 8;

/**
 * 当桶数组容量小于该值时，优先进行扩容，而不是树化
 */
static final int MIN_TREEIFY_CAPACITY = 64;

static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }
}

/**
 * 将普通节点链表转换成树形节点链表
 */
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 桶数组容量小于 MIN_TREEIFY_CAPACITY，优先进行扩容而不是树化
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        // hd 为头节点（head），tl 为尾节点（tail）
        TreeNode<K,V> hd = null, tl = null;
        do {
            // 将普通节点替换成树形节点
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);  // 将普通链表转成由树形节点链表
        if ((tab[index] = hd) != null)
            // 将树形链表转换成红黑树
            hd.treeify(tab);
    }
}

TreeNode<K,V> replacementTreeNode(Node<K,V> p, Node<K,V> next) {
    return new TreeNode<>(p.hash, p.key, p.value, next);
}
```

在扩容过程中，红黑树化需要满足两个条件：

- 链表长度大于等于TREEIFY_THRESHOLD（也就是8）；
- 桶数组大小大于等于MIN_TREEIFY_CAPACITY（也就是64）；

对于第二个条件的原因是：**在桶数组大小比较小时，导致hash冲突的主要原因是桶数组太小，这个时候扩容才是解决问题的最好方法；而且桶数组大小比较小时，扩容比较频繁而且扩容时还需要拆分红黑树，代价过大**；

链表红黑树化的逻辑主要是在`treeifyBin(Node<K,V>[], int)`函数中进行的，该函数的主要逻辑是：

- 先判断链表红黑树化的条件是否满足，如果不满足则进行扩容；
- 否则使用指针hd记录链表的头，遍历链表构造TreeNode节点；
- 调用`treeify()`函数完成红黑树化；

**这里有一个需要注意的地方就是，由于TreeNode是继承Node类，所有TreeNode类中有next引用**，所以在遍历链表的时候，设置的prev，next字段其实就构成了一个双链表；

![avatar](D:\img\链表树化.jpg)

由于HashMap没有要求键实现Comparable接口，所以键之间的比较就是一个问题，为了解决这个问题，HashMap 是做了三步处理，确保可以比较出两个键的大小，如下：

1. 比较键与键之间 hash 的大小，如果 hash 相同，继续往下比较
2. 检测键类是否实现了 Comparable 接口，如果实现调用 compareTo 方法进行比较
3. 如果仍未比较出大小，就需要进行仲裁了，仲裁方法为 tieBreakOrder

![avatar](D:\img\链表树化后.jpg)

链表转成红黑树后，原链表的顺序仍然会被引用仍被保留了（红黑树的根节点会被移动到链表的第一位），我们仍然可以按遍历链表的方式去遍历上面的红黑树。这样的结构为后面红黑树的切分以及红黑树转成链表做好了铺垫 ；

##### 红黑树拆分

```java
// 红黑树转链表阈值
static final int UNTREEIFY_THRESHOLD = 6;

final void split(HashMap<K,V> map, Node<K,V>[] tab, int index, int bit) {
    TreeNode<K,V> b = this;
    // Relink into lo and hi lists, preserving order
    TreeNode<K,V> loHead = null, loTail = null;
    TreeNode<K,V> hiHead = null, hiTail = null;
    int lc = 0, hc = 0;
    /* 
     * 红黑树节点仍然保留了 next 引用，故仍可以按链表方式遍历红黑树。
     * 下面的循环是对红黑树节点进行分组，与上面类似
     */
    for (TreeNode<K,V> e = b, next; e != null; e = next) {
        next = (TreeNode<K,V>)e.next;
        e.next = null;
        if ((e.hash & bit) == 0) {
            if ((e.prev = loTail) == null)
                loHead = e;
            else
                loTail.next = e;
            loTail = e;
            ++lc;
        }
        else {
            if ((e.prev = hiTail) == null)
                hiHead = e;
            else
                hiTail.next = e;
            hiTail = e;
            ++hc;
        }
    }

    if (loHead != null) {
        // 如果 loHead 不为空，且链表长度小于等于 6，则将红黑树转成链表
        if (lc <= UNTREEIFY_THRESHOLD)
            tab[index] = loHead.untreeify(map);
        else {
            tab[index] = loHead;
            /* 
             * hiHead == null 时，表明扩容后，
             * 所有节点仍在原位置，树结构不变，无需重新树化
             */
            if (hiHead != null) 
                loHead.treeify(tab);
        }
    }
    // 与上面类似
    if (hiHead != null) {
        if (hc <= UNTREEIFY_THRESHOLD)
            tab[index + bit] = hiHead.untreeify(map);
        else {
            tab[index + bit] = hiHead;
            if (loHead != null)
                hiHead.treeify(tab);
        }
    }
}
```

在扩容后，和链表一样，红黑树也要进行节点映射，本来这个操作需要先将红黑树先转换为链表，再进行两边的节点映射，最后再把链表转换为红黑树；但是在将普通链表转成红黑树时，HashMap 通过两个额外的引用 next 和 prev 保留了原链表的节点顺序。这样再对红黑树进行重新映射时，完全可以按照映射链表的方式进行。这样就避免了将红黑树转成链表后再进行映射，无形中提高了效率；

从源码上可以看得出，重新映射红黑树的逻辑和重新映射链表的逻辑基本一致。不同的地方在于，重新映射后，会将红黑树拆分成两条由 TreeNode 组成的链表。如果链表长度小于 UNTREEIFY_THRESHOLD，则将链表转换成普通链表。否则根据条件重新将 TreeNode 链表树化；

![avatar](D:\img\红黑树节点映射.jpg)

##### 红黑树链化

```java
final Node<K,V> untreeify(HashMap<K,V> map) {
    Node<K,V> hd = null, tl = null;
    // 遍历 TreeNode 链表，并用 Node 替换
    for (Node<K,V> q = this; q != null; q = q.next) {
        // 替换节点类型
        Node<K,V> p = map.replacementNode(q, null);
        if (tl == null)
            hd = p;
        else
            tl.next = p;
        tl = p;
    }
    return hd;
}

Node<K,V> replacementNode(Node<K,V> p, Node<K,V> next) {
    return new Node<>(p.hash, p.key, p.value, next);
}
```

由于红黑树的节点仍然保留了原来链表的信息，所以红黑树转换为链表的操作很简单；



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
