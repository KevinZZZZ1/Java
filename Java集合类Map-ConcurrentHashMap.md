# Java集合类Map-ConcurrentHashMap

[TOC]

## 一、概述

`ConcurrentHashMap`通过部分加锁和CAS实现同步功能，key和value都不允许为null（HashMap允许key和value都为null），其针对的是Node[] table数组中的每个桶，进一步减小了锁粒度，并且当链表长度大于8时，会使用红黑树设计。

## 二、使用ConcurrentHashMap的原因

在并发场景下，为了得到线程安全的HashMap，有以下3中方法：

- 使用HashTable：
- 使用Collections.synchronizedMap()方法：
- 使用ConcurrentHashMap：

一般情况下都会选择ConcurrentHashMap，重要的原因就是效率问题；HashTable就是将HashMap的方法前加上synchronized关键字，而Collections.synchronizedMap()也是内部持有一个锁对象，然后内部还是用使用synchronized代码块进行同步的；本质上上面两个类都同时只能允许一个线程进入临界区；

而ConcurrentHashMap的锁粒度远远大于synchronized，使用CAS+synchronized来保证线程安全；

## 三、ConcurrentHashMap源码（基于JDK1.8）分析

### 3.1继承结构

```java
public class ConcurrentHashMap<K,V> extends AbstractMap<K,V>
    implements ConcurrentMap<K,V>, Serializable {
    	...
    }
```

可以看到ConcurrentHashMap继承了AbstractMap抽象类，实现了ConcurrentMap和Serializable接口；

### 3.2属性

```java

    /**
     * 存放node的数组，大小是2的幂次方
     */
    transient volatile Node[] table;
    
    /**
     * 扩容时用于存放数据的变量，平时为null
     */
    private transient volatile Node[] nextTable;
    
    /**
     * 通过CAS更新，记录容器的容量大小
     */
    private transient volatile long baseCount;
    
    /**
     * 容量控制
     * 负数: 代表正在进行初始化或扩容操作，其中-1表示正在初始化，-N 表示有N-1个线程正在进行扩容操作
     * 正数或0: 代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小，类似于扩容阈值
     * 它的值始终是当前ConcurrentHashMap容量的0.75倍，这与loadfactor是对应的。
     * 实际容量 >= sizeCtl，则扩容
     */
    private transient volatile int sizeCtl;
    
    /**
     * 下次transfer方法的起始下标index加上1之后的值
     */
    private transient volatile int transferIndex;
    
    /**
     * CAS自旋锁标志位
     */
    private transient volatile int cellsBusy;
    
    /**
     * counter cell表，长度总为2的幂次
     */
    private transient volatile CounterCell[] counterCells;

```

- 其中重要的属性有：
  - transient volatile Node[] table：Node类型的数组，作为ConcurrentHashMap的**数据容器**，采用**懒加载的方式**，直到第一次插入数据时才会进行数组初始化，**数组的大小为2的幂**；
  - private transient volatile Node[] nextTable：Node类型的数组，平时为null，**在扩容的时候使用**；
  - private transient volatile int sizeCtl：**该属性用于容量控制，表示threshold**，其取值有以下几种情况：
    - sizeCtl：为-1，表示正在初始化；
    - sizeCtl：为-N，表示当前有N-1个线程正在进行扩容；
    - sizeCtl：为正数，如果当前table数组为null，sizeCtl表示新建数组的长度；如果已经初始化了，sizeCtl就是该Map的threshold;

### 3.3重要内部类

#### 3.3.1Node节点类

```java
 static class Node<K,V> implements Map.Entry<K,V> {
         final int hash;
         final K key;
         volatile V val;
         volatile Node<K,V> next;
 		......
 }
```

- 其中表示值V的变量`val`以及表示下一个节点的变量`next`，都使用了`volatile`关键字修饰，保证这两个变量的可见性；

#### 3.3.2TreeNode树节点类

```java
 /**
  * Nodes for use in TreeBins
  */
 static final class TreeNode<K,V> extends Node<K,V> {
         TreeNode<K,V> parent;  // red-black tree links
         TreeNode<K,V> left;
         TreeNode<K,V> right;
         TreeNode<K,V> prev;    // needed to unlink next upon deletion
         boolean red;
 		......
 }

```

- 继承了存储数据的节点类Node，TreeNode为TreeBin内部类服务，**其hash值固定为-2**

#### 3.3.3TreeBin类

```java
 static final class TreeBin<K,V> extends Node<K,V> {
         TreeNode<K,V> root;
         volatile TreeNode<K,V> first;
         volatile Thread waiter;
         volatile int lockState;
         // values for lockState
         static final int WRITER = 1; // set while holding write lock
         static final int WAITER = 2; // set when waiting for write lock
         static final int READER = 4; // increment value for setting read lock
 		......
 }
```

- 该类用于包装很多TreeNode节点，在红黑树的情况下，实际上table数组中存放的是TreeBin对象，而不是TreeNode对象；

#### 3.3.4ForwardingNode节点类

```java
 static final class ForwardingNode<K,V> extends Node<K,V> {
     final Node<K,V>[] nextTable;
     ForwardingNode(Node<K,V>[] tab) {
         super(MOVED, null, null, null);
         this.nextTable = tab;
     }
    .....
 }

```

- 在**扩容时才会出现的特殊节点**，其key，value，hash全部为null，并且nextTable指向扩容时新的table数组；

### 3.4重要的CAS操作

#### 3.4.1tabAt

```java
 static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
     return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
 }
```

- 以CAS的方式获取tab[i]处的Node元素；

#### 3.4.2casTabAt

```java
 static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                     Node<K,V> c, Node<K,V> v) {
     return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
 }
```

- 以CAS的方式，将tab[i]处的Node元素由c替换为v；

#### 3.4.3setTabAt

```java
 static final <K,V> void setTabAt(Node<K,V>[] tab, int i, Node<K,V> v) {
     U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);
 }
```

- 以CAS的方式，将tab[i]处的Node元素替换为v；



### 3.5构造方法

构造一个ConcurrentHashMap对象，一共有5种构造方法：

```java
// 1. 构造一个空的map，即table数组还未初始化，初始化放在第一次插入数据时，默认大小为16
ConcurrentHashMap()
// 2. 给定map的大小
ConcurrentHashMap(int initialCapacity) 
// 3. 给定一个map
ConcurrentHashMap(Map<? extends K, ? extends V> m)
// 4. 给定map的大小以及加载因子
ConcurrentHashMap(int initialCapacity, float loadFactor)
// 5. 给定map大小，负载因子以及并发度（预计同时操作数据的线程）
ConcurrentHashMap(int initialCapacity,float loadFactor, int concurrencyLevel)
```

- 从上面我们可以看到，ConcurrentHashMap中可设置的参数有：`initialCapacity`，`loadFactor`，`concurrentLevel`；分别表示初始大小，负载因子，并发度；
- `concurrentLevel`表示的是同时操作ConcurrentHashMap的线程数；

现在，我们主要看一下第5个构造函数：

```java
public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    // 将初始数组的大小赋值给sizeCtl
    this.sizeCtl = cap;
}
```

- 该方法的大致逻辑是：
  - 先是判断输入的loadFactor，initialCapacity，concurrencyLevel是否合法；
  - 再判断并发程度（concurrencyLevel）和初始容量（initialCapacity）的关系，应该是：初始容量>=并发程度；
  - 计算数组table的初始容量，其中`tableSizeFor(int size)`方法和HashMap中的一样，都是得到大于等于size的最小2的幂；
  - 把数组的初始容量赋值给sizeCtl；

### 3.6initTable方法

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc; // tab变量记录当前的数组，sc记录当前的sizeCtl的值
    // 使用while循环进行条件判断
    while ((tab = table) == null || tab.length == 0) {
        // 1. sizeCtl小于0，表示已经有线程在进行初始化工作了，保证只有一个线程正在进行初始化操作
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { // 如果sizeCtl>=0，那么保证SIZECTL（sizeCtl）仍然等于sc的情况，当前线程使用CAS将sizeCtl值置为-1，说明已经有线程正在对ConcurrentHashMap进行初始化
            try {
                if ((tab = table) == null || tab.length == 0) {
					// 2. 得出数组的大小
                    // 使用构造函数是，将初始容量存在了sizeCtl中，如果sc大于0的话，那么说明设定了初始容量，否则就是没有设定初始容量，默认容量是16
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
					// 3. 这里才真正的初始化数组
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
					// 4. 计算数组中可用的大小：实际大小n*0.75（加载因子）
                    // 即n-n/4为0.75n表示threshold值
                    sc = n - (n >>> 2); // 
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

- 上面方法的大致逻辑是：
  - 先判断sizeCtl的值是否小于0，如果小于0表示**已经有线程对ConcurrentHashMap进行初始化**，则该线程让出自己的CPU时间；
  - 如果sizeCtl值大于等于0，说明**还没有线程初始化ConcurrentHashMap**，此时当前线程可以使用CAS将sizeCtl的值变为-1，表示当前对ConcurrentHashMap进行初始化；
    - 如果sizeCtl值大于0，说明**构造函数设定了初始值**（initialCapacity和sizeCtl是不一样的），使用sizeCtl的值作为数组大小
    - 如果sizeCtl值为0，使用默认的数组大小，为**16**；
    - 真正的创建Node类型数组table；
    - 设置当前table大小下的threshold，并赋值给sizeCtl；
  - 返回新建的table数组；

- `initTable()`是在第一次向ConcurrentHashMap中添加元素才会执行，这也就是所谓的**懒加载策略**；
- 初始化table数组的大小一定是2的幂，无论是默认的16，还是sizeCtl（sizeCtl中保存的是第一个大于initialCapacity的2的幂）；

### 3.7put方法

```java
/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // ConcurrentHashMap不允许key和value为null
    if (key == null || value == null) throw new NullPointerException();
	//1. 计算key的hash值
    int hash = spread(key.hashCode());
    int binCount = 0; // 
    for (Node<K,V>[] tab = table;;) {
        // n表示数组的大小，i表示hash值对应的索引，f表示table[i]处的节点也就是链表的头节点，fh表示f节点对应的hash值；
        Node<K,V> f; int n, i, fh;
		//2. 如果当前table还没有初始化先调用initTable方法将tab进行初始化
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
		//3. tab中索引为i的位置的元素为null，则直接使用CAS将值插入即可
        // hash&(n-1)就是hash%n，当n为2的幂时，这一点和HashMap一样
        // f = tabAt(tab, i)表示以CAS的方式，得到table[i]的元素，如果为null表示table数组的这个位置还没被占用
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 以CAS的方式，将table[i]处值为null的节点替换为需要插入的节点
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
		//4. ConcurrentHashMap正在扩容，当前线程也要帮助进行扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            // 使用CAS插入失败，只能使用Synchronized锁来实现同步
            V oldVal = null;
            // 锁住的是table[i]对应的链表的头节点
            synchronized (f) {
                if (tabAt(tab, i) == f) {
					//5. 当前为链表，在链表中插入新的键值对
                    if (fh >= 0) {
                        binCount = 1;
                        // 从前向后变量table[i]开始的链表，e表示当前在处理的元素
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 判断当前节点e是否与key有相同的hash值、key值，如果是的话直接用新value替换原来的val
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            // 在创建一个新节点并添加到链表尾部
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
					// 6.当前为红黑树，将新的键值对插入到红黑树中
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
			// 7.插入完键值对后再根据实际大小看是否需要转换成红黑树
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
	//8.对当前容量大小进行检查，如果超过了threshold（实际大小*加载因子）就需要扩容 
    addCount(1L, binCount);
    return null;
}

```

- 通过对上面代码的阅读，大致逻辑如下：

  - 先判断key和value是否为null，如果是直接抛出异常（**ConcurrentHashMap不允许key和value任意一个为null**）；
  - 根据key计算对应的hash值；
  - 判断当前的**table数组是否为空**，如果为空说明是第一次添加元素，先进行初始化；
  - table数组不为null，计算hash值对应table数组的索引i，并**使用CAS判断table[i]是否为null**；
    - **使用CAS尝试将新节点赋值给table[i]**；
  - 判断table[i]的节点的**hash值是否为-1**，如果是则说明ConcurrentHashMap**正在扩容**，当前线程也进入扩容操作，**帮助扩容**；
  - 否则，说明使用CAS尝试插入元素失败，**使用synchronized对table[i]处节点加锁**；
    - 如果table[i]节点的hash值大于等于0，说明当前结构还是链表：
      - 从头开始遍历链表，如果链表中的某个节点hash,k等于插入的hash和key，那么使用value对v值进行更新；否则新建一个节点并在链表尾部插入；
    - 否则说明当前结构已经是红黑树：
      - 遍历红黑树上的节点，更新或者增加节点；
  - 判断当前**链表长度是否超过8**，如果是的话将链表转化为红黑树；
  - 将ConcurrentHashMap中的节点数量+1，并**判断是否超过threshold**，超过的话需要扩容；

- key的hash值计算：

  ```java
  static final int spread(int h) {
      return (h ^ (h >>> 16)) & HASH_BITS;
  }
  ```

  - 和HashMap大致上相同，都是将key的hashCode的高16位和低16位异或，ConcurrentHashMap多了一个与HASH_BITS的操作；

- 正在扩容时，插入操作的处理：

  当链表的头节点的hash值为-1（MOVED）时，表示该节点类型为forwardingNode，即此ConcurrentHashMap有其他线程对其进行扩容操作，此时**当前线程也会帮助扩容操作**（执行`helpTransfer()`方法）；

  ```java
  // 当前节点f的类型为forwardingNode，帮助完成扩容操作
  final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
      // nextTab表示用来扩容的数组
      Node<K,V>[] nextTab; int sc;
      // 还在扩容的过程中
      if (tab != null && (f instanceof ForwardingNode) &&
          (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
          int rs = resizeStamp(tab.length);
          while (nextTab == nextTable && table == tab &&
                 (sc = sizeCtl) < 0) {
              if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                  sc == rs + MAX_RESIZERS || transferIndex <= 0)
                  break;
              if (U.compareAndSetInt(this, SIZECTL, sc, sc + 1)) {
                  // 调用transfer()方法帮助扩容
                  transfer(tab, nextTab);
                  break;
              }
          }
          return nextTab;
      }
      return table;
  }
  ```

  关于`transfer()`方法的解释看3.9节，这里就不再多说了；

- 插入完成后，检查当前ConcurrentHashMap中元素个数是否大于threshold：

  ```java
  
  private final void addCount(long x, int check) {
      CounterCell[] as; long b, s;
      //利用CAS方法更新baseCount的值
      if ((as = counterCells) != null ||
          !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
          CounterCell a; long v; int m;
          boolean uncontended = true;
          if (as == null || (m = as.length - 1) < 0 ||
              (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
              !(uncontended =
                U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
              //多线程CAS发生失败的时候执行  
              fullAddCount(x, uncontended);
              return;
          }
          if (check <= 1)
              return;
          s = sumCount();
      }
      ///如果check值大于等于0 则需要检验是否需要进行扩容操作
      if (check >= 0) {
          Node[] tab, nt; int n, sc;
          while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                 (n = tab.length) < MAXIMUM_CAPACITY) {
              int rs = resizeStamp(n);
              if (sc < 0) {
                  if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                      sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                      transferIndex <= 0)
                      break;
                  // 对应ConcurrentHashMap在扩容期间的情况
                  if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                      transfer(tab, nt);
              }
              // 对应ConcurrentHashMap还没被扩容的情况
              else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                           (rs << RESIZE_STAMP_SHIFT) + 2))
                  transfer(tab, null);
              s = sumCount();
          }
      }
  }
  
  ```

  其中，我们可以看到在计算完当前ConcurrentHashMap中元素的个数后，`addCount()`方法还检查了该ConcurrentHashMap是否要扩容（调用`transfer()`方法扩容）；

### 3.8get方法

```java
public V get(Object key) {
    // tab表示当前table数组，n表示table数组大小，e表示在table[h&(n-1)]处的节点
    // eh表示e节点的hash值，ek表示e节点的key
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 计算key的hash值为h
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        // table[i]处的节点就是要找的节点，直接返回
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 当前节点的hash值小于0表示节点是红黑树节点，所以在红黑树中查找
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        // 在table[i]的链表中，从头到尾的查找
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    // 没有找到，返回null
    return null;
}
```

- get()方法相比put()方法就简单的多了，get()方法的大致逻辑如下:
  - 计算key的hash值，以及hash值对应在table数组中的下标；
  - 如果table[i]的hash值，k值都等于要查找的键的hash值和key值，那么直接返回table[i]的value；
  - 否则再判断table[i]对应的节点的hash值是否小于0，如果是的话说明当前节点是红黑树，那么在红黑树中查找；
  - 否则，在table[i]指向的链表中从头到尾的查找；
  - 如果上面都没有找到，那么就返回null；



### 3.9transfer方法

`transfer()`方法就是对ConcurrentHashMap进行扩容的方法，当ConcurrentHashMap中的元素个数大于threshold（数组大小*负载因子，也就是在sizeCtl中的值）时，就需要扩容。ConcurrentHashMap的扩容支持多线程同时扩容，而且不用加锁；在扩容期间，只要对应的table[i]节点的类型不是forwardingNode，那么其他线程还能对该节点的链表（红黑树）进行操作；

```java
private final void transfer(Node[] tab, Node[] nextTab) {
    // n表示当前table数组的长度，stride表示每个CPU需要处理桶的个数
    int n = tab.length, stride;
    // 让每个CPU处理一样多的桶，避免出现任务分配不均匀
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    // 用于扩容的nextTab数组还没初始化
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            // 创建node数组，容量为当前的两倍
            Node[] nt = (Node[])new Node[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            // 若扩容时出现OOM异常，则将阈值设为最大，表明不支持扩容
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n; // transferIndex表示迁移数据的下标，现在是等于原数组的长度
    }
    int nextn = nextTab.length; // nextn表示新数组的长度
    // 创建ForwardingNode节点，作为标记位，表明当前位置桶已做过处理
    ForwardingNode fwd = new ForwardingNode(nextTab);
    // 表示是否要处理下一个桶
    boolean advance = true;
    // 扩容是否结束
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        // 
        Node f; int fh;
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            //通过CAS设置transferIndex属性值，并初始化i和bound值
            //i指当前处理的槽位序号，bound指需要处理的槽位边界
            //先处理最后一个桶的节点；
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound; 
                i = nextIndex - 1;
                advance = false;
            }
        }
        // 将原数组中节点复制到新数组中去
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            //如果所有的节点都已经完成复制工作  就把nextTable赋值给table 清空临时对象nextTable
            if (finishing) {
                nextTable = null;
                table = nextTab;
                //设置新扩容阈值
                sizeCtl = (n << 1) - (n >>> 1); // 2*n-n/2等于1.5n，而原来的threshold是0.75n，扩容之后的threshold是1.5n
                return;
            }
            //利用CAS方法更新扩容阈值，在这里面sizectl值减一，说明新加入一个线程参与到扩容操作
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 如果table[i]的节点为null，使用CAS将table[i]替换为forwardingNode类型节点
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd);
        // 如果table[i]的节点已经为forwardingNode节点，说明已经该桶已经被处理
        else if ((fh = f.hash) == MOVED)
            advance = true; // already processed
        else {
            //锁住i位置上桶的节点
            synchronized (f) {
                //确保f是i位置上桶的节点
                if (tabAt(tab, i) == f) {
                    // ln表示(hash&n==0)对应的链表
                    // hn表示(hash&n==1)对应的链表
                    Node ln, hn;
                    //当前桶是链式结构
                    if (fh >= 0) {
                        //构造两个链表
                        int runBit = fh & n;
                        Node lastRun = f;
                        // 遍历链表，lastRun到链表尾部都是同一种类型的节点，即从lastRun到尾部都是runBit类型的节点，这样的话在进行链表映射时，就只需要遍历头节点到lastRun节点前一个就行；
                        for (Node p = f.next; p != null; p = p.next) {
                            //n是就数组长度，不是长度-1
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        // 将ln(hn)指向原来最后一个处理的节点
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        //遍历链表，类似于1.8HashMap，只需要看(hash&n)是0还是1进行判断，
                        for (Node p = f; p != lastRun; p = p.next) {
                            // p表示正在处理的节点
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            // 使用头插法，完成逆序
                            if ((ph & n) == 0)
                                ln = new Node(ph, pk, pv, ln);
                            else
                                hn = new Node(ph, pk, pv, hn);
                        }
                        //在nextTable的i位置上插入一个链表
                        setTabAt(nextTab, i, ln);
                        //在nextTable的i+n的位置上插入另一个链表
                        setTabAt(nextTab, i + n, hn);
                        //在table的i位置上插入forwardNode节点  表示已经处理过该节点
                        setTabAt(tab, i, fwd);
                        //设置advance为true 返回到上面的while循环中 就可以执行i--操作
                        advance = true;
                    }
                    //当前桶是红黑树结构，操作和上面的类似
                    else if (f instanceof TreeBin) {
                        TreeBin t = (TreeBin)f;
                        TreeNode lo = null, loTail = null;
                        TreeNode hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode p = new TreeNode
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        //如果扩容后已经不再需要tree的结构 反向转换为链表结构
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                        (hc != 0) ? new TreeBin(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                        (lc != 0) ? new TreeBin(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}

```

- 从上面的代码可以看出ConcurrentHashMap的扩容操作挺复杂，大致逻辑是：

  - 先判断新table数组是否为空，如果为空则创建一个新Node类型数组，并且大小是当前的两倍；
  - 利用CAS操作设置每个线程处理的数组桶的个数；
  - 判断当前线程是否完成了对于某个数组中某个桶的操作；
  - 利用CAS判断table[i]处是否为null，如果是null就直接用forwardingNode类型节点代替；
  - 否则判断table[i]处是否为forwardingNode节点，说明该节点已经被处理过；
  - 否则使用synchronized锁定table[i]处的节点，根据桶节点的hash值判断当前节点是链表还是红黑树，然后再进行链表节点映射（红黑树节点映射）；

- 链表节点映射是需要注意的问题：

  和HashMap一样，ConcurrentHashMap也有**链表节点映射**，n表示原数组的大小，也是根据hash&n==1对应新table数组中的i+n位置，hash&n ==0对应新table数组中i位置；但是一点不同的是，HashMap中链表节点映射得到的两个链表中，节点的相对顺序是不会变化的，而在ConcurrentHashMap中得到的两个链表是HashMap中链表的逆序；

  如下图所示：

  ![avatar](F:\找工作\Java基础\Java\images\ConcurrentHashMap扩容.png)

  其中蓝色节点表示hash&n== 0，灰色表示hash&n==1：

  - 对于HashMap来说，有lo链表：A->B->F，hi链表：C->D->E；然后再把lo链表插入table[i]处，hi链表插入table[i+n]处；
  - 对于ConcurrentHashMap来说，第一次遍历链表可得runBit为0，lastRun为F，In为F，hn为null；第二次遍历链表时，**由于此时插入是头插法，所以得到In链表：B->A->F，hn链表：E->D->C**；然后再把In链表插入table[i]处，hn链表插入table[i+n]处；





## 四、ConcurrentHashMap 1.7和1.8的区别

在JDK1.7中ConcurrentHashMap主要是使用Segment来实现减小锁粒度的，分割成若干个Segment（继承ReentrantLock），在put的时候只对对应的Segment上锁然后再定位到具体的桶数组最后到具体的桶，get的时候不加锁（使用volatile保证可见性）；

在JDK1.8中锁的粒度更小了，直接针对的是table数组中的每一个桶，而且put先是使用CAS尝试获得，CAS失败再使用synchronized锁定桶，get的时候也不加锁；而且当前线程检测到ConcurrentHashMap在扩容，当前线程会帮助一起扩容；



## 五、参考文献

https://juejin.im/post/5b53d1adf265da0f70070e3d

https://juejin.im/post/5aeeaba8f265da0b9d781d16

https://www.jianshu.com/p/aaf769fdbd20