# Queue-PriorityQueue

[TOC]

## 概述

优先级队列是一个基于堆的数据结构，默认情况下是最小堆，优先级队列中以数组作为存储结构保存元素，不允许插入null；

最小堆的情况下，优先级队列的头是按照指定排序方式确定的最小元素；

## 相关数据结构--堆

堆是一种特殊的二叉树，它具有以下两个特征：

- 堆是一个完全二叉树；
- 堆中每个节点的值必须小于等于（大于等于）其子节点的值；

完全二叉树是指：对于高度为k的二叉树，它的0到k-1层都是满的，且k-1层的节点的子节点不为单独的右孩子节点；如下图所示：

![avatar](F:\找工作\Java基础\Java\images\完全二叉树.png)

**完全二叉树从结构上来看特别适合使用数组实现，这样很方便的能得到某个节点的父节点和子节点**；

堆是一个完全二叉树，顶部是最小元素的叫小顶堆，顶部是最大元素的叫大顶堆；

## 继承关系

```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {
    省略部分代码
    //
}
```

- PriorityQueue继承了AbstractQueue，实现了Serializable接口，该接口是一个标识接口表示能够序列化；

## 主要属性

```java
// 优先级队列默认初始容量
private static final int DEFAULT_INITIAL_CAPACITY = 11;
// 存储优先级队列中元素的数组
transient Object[] queue;
// 记录队列中元素的个数
int size;
// 比较器
private final Comparator<? super E> comparator;
// 队列被修改的次数，实现fail-fast
transient int modCount;
```

从源码中可以看出：

- 优先级队列的初始默认大小为11，也就是说不指定容量的情况下，会创建一个大小为11的数组；
- 队列使用数组作为底层的数据结构来存储队列中的元素；
- 队列中元素的个数使用`size`属性记录；
- 使用一个`Comparator`对象来处理各种比较标准；
- `modCount`表示队列被修改的次数，防止在多线程情况下，一个线程使用迭代器遍历时，另一个线程对队列进行其他操作；

## 构造函数

优先级队列的构造函数有7个，可以分成两种：提供初始元素和不提供初始元素；

- 先看不提供初始元素的：

  ```java
  // 创建默认容量为11的优先级队列，按照自然序排序元素
  public PriorityQueue() {
  	this(DEFAULT_INITIAL_CAPACITY, null);
  }
  
  // 创建指定容量的优先级队列，按照自然序排序元素
  public PriorityQueue(int initialCapacity) {
      this(initialCapacity, null);
  }
  
  // 创建默认容量为11的优先级队列，使用给定Comparator对象进行排序
  public PriorityQueue(Comparator<? super E> comparator) {
      this(DEFAULT_INITIAL_CAPACITY, comparator);
  }
  
  // 创建指定容量的优先级队列，使用给定Comparator对象进行排序
  public PriorityQueue(int initialCapacity,
                           Comparator<? super E> comparator) {
      // Note: This restriction of at least one is not actually needed,
      // but continues for 1.5 compatibility
      if (initialCapacity < 1)
          throw new IllegalArgumentException();
      this.queue = new Object[initialCapacity];
      this.comparator = comparator;
  }
  ```

  从源码能够看出，这类构造函数比较简单，就是进行了一些创建对象以及赋值的操作；

- 提供初始元素的：

  ```java
  // 根据PriorityQueue来构造堆
  public PriorityQueue(PriorityQueue<? extends E> c) {
      // 比较器赋值
      this.comparator = (Comparator<? super E>) c.comparator();
      // 给定一个PriorityQueue来构建优先级队列
      initFromPriorityQueue(c);
  }
  
  // 根据SortedSet来构造堆
  public PriorityQueue(SortedSet<? extends E> c) {
      this.comparator = (Comparator<? super E>) c.comparator();
      // 因为SortedSet是有序的，所以可以直接将c中的元素拷贝到本优先级队列中
      initElementsFromCollection(c);
  }
  
  // 根据集合类来构造堆
  public PriorityQueue(Collection<? extends E> c) {
      // 如果c是SortedSet
      if (c instanceof SortedSet<?>) {
          SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
          this.comparator = (Comparator<? super E>) ss.comparator();
          initElementsFromCollection(ss);
      }
      // 如果c是PriorityQueue
      else if (c instanceof PriorityQueue<?>) {
          PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
          this.comparator = (Comparator<? super E>) pq.comparator();
          initFromPriorityQueue(pq);
      }
      // 如果是其他的集合类
      else {
          this.comparator = null;
          initFromCollection(c);
      }
  }
  
  // 将c中的元素直接添加到要创建的优先级队列中
  private void initElementsFromCollection(Collection<? extends E> c) {
      Object[] es = c.toArray();
      int len = es.length;
      // If c.toArray incorrectly doesn't return Object[], copy it.
      if (es.getClass() != Object[].class)
          es = Arrays.copyOf(es, len, Object[].class);
      // 判断es数组中是否有为null的元素
      if (len == 1 || this.comparator != null)
          for (Object e : es)
              if (e == null)
                  throw new NullPointerException();
      this.queue = ensureNonEmpty(es);
      this.size = len;
  }
  
  // 给定一个PriorityQueue来构建优先级队列
  private void initFromPriorityQueue(PriorityQueue<? extends E> c) {
      // 如果对象c就是一个优先级队列，直接赋值
      if (c.getClass() == PriorityQueue.class) {
          // 保证c不是空数组
          this.queue = ensureNonEmpty(c.toArray());
          this.size = c.size();
      } else {
          initFromCollection(c);
      }
  }
  
  // 
  private void initFromCollection(Collection<? extends E> c) {
      // 将c中元素拷贝到queue数组中
      initElementsFromCollection(c);
      // 堆化
      heapify();
  }
  
  ```

  从上面源码看出，在给定初始元素时，如果给定的初始元素是有序的话，直接将元素加入`queue`数组，否则加入`queue`数组之后还要多一步堆化的操作；

  可以说堆化是创建优先级队列中最重要的部分，接下来的代码主要就是堆化的过程：

  ```java
  // 进行堆化
  private void heapify() {
      // 得到存放元素的数组
      final Object[] es = queue;
      // 设置堆化的起始元素位置i（堆化都是从数组的中间位置开始，向前遍历数组）
      int n = size, i = (n >>> 1) - 1;
      final Comparator<? super E> cmp;
      // 如果没有设置比较器，使用默认自然排序对[i, 0]之间的元素进行堆化处理
      if ((cmp = comparator) == null)
          for (; i >= 0; i--)
              siftDownComparable(i, (E) es[i], es, n);
      else
      // 如果设置了比较器，使用指定的排序对[i, 0]之间的元素进行堆化处理
          for (; i >= 0; i--)
              siftDownUsingComparator(i, (E) es[i], es, n, cmp);
  }
  // 使用自然排序对元素x进行堆化操作，自上而下堆化
  private static <T> void siftDownComparable(int k, T x, Object[] es, int n) {
      // assert n > 0;
      Comparable<? super T> key = (Comparable<? super T>)x; // key用来记录要比较的节点
      // 只需要和前一半的节点进行比较即可
      int half = n >>> 1;           // loop while a non-leaf
      while (k < half) {
          // child用来记录最小值节点的索引，先假设最小值的节点的索引为左孩子的索引
          int child = (k << 1) + 1; // assume left child is least
          // 找到左孩子c，其实c的意图是记录有最小值的节点
          Object c = es[child];
          int right = child + 1;
          // 下面两个if语句的意义在于找出根节点、左孩子、右孩子中最小的节点
          // 如果右孩子存在且左孩子的值大于右孩子的话，将右孩子赋值给c，将右孩子的索引赋值给child
          if (right < n &&
              ((Comparable<? super T>) c).compareTo((T) es[right]) > 0)
              c = es[child = right];
          // 如果根节点的值是最小的，已经满足堆的特性，无需堆化
          if (key.compareTo((T) c) <= 0)
              break;
          // 否则，将最小值的节点赋值到k位置上，
          es[k] = c;
          k = child;
      }
      // 把要比较的节点放在合适的位置
      es[k] = key;
  }
  
  // 使用比较器来进行堆化
  private static <T> void siftDownUsingComparator(
      int k, T x, Object[] es, int n, Comparator<? super T> cmp) {
      // assert n > 0;
      int half = n >>> 1;
      while (k < half) {
          // 记录最小值的节点索引
          int child = (k << 1) + 1;
          // 记录最小值的节点
          Object c = es[child];
          int right = child + 1;
        // 在根节点、左孩子、右孩子中寻找最小值的节点
          if (right < n && cmp.compare((T) c, (T) es[right]) > 0)
              c = es[child = right];
          if (cmp.compare(x, (T) c) <= 0)
              break;
          es[k] = c;
          k = child;
      }
      es[k] = x;
  }
  ```
  
  从源码中我们可以看出无论是否使用比较器，堆化过程的思想都是相同的，过程是：
  
  - 用变量`k`表示当前需要比较的节点，而且k只需要和数组前一半元素比较即可；
  - 首先假设左孩子是最小值节点，用变量`child`记录最小值节点的索引，用变量`c`记录最小值节点；
  - 在左孩子、右孩子、根节点中找到最小值的节点，并且将相应的节点放到根节点的位置；
  - 然后重复以上过程直到`k`<0为止；



## 主要操作

对于所有的集合类来说，主要操作都是增删改查，所以以下对此进行说明：

### 添加元素

添加元素主要有`offer(E e)`方法和`add(E e)`方法，而且`add(E e)`方法调用`offer(E e)`方法来实现的，**都是在队尾添加一个元素**；

```java
// 将e插入队尾并进行堆化
public boolean add(E e) {
    return offer(e);
}

// 将e插入队尾并进行堆化
public boolean offer(E e) {
    // 如果e为空，抛出空指针异常，也就是说明优先级队列中不能加入null元素
    if(e == null)
        throw new NullPointerException();
    // 对优先级队列进行了修改，所以modCount要加1
    modCount++;
    int i = size;
    // 这里只能是i刚好等于queue.length，也就是数组刚刚好满，这时再加入一个元素就要扩容
    if(i >= queue.length)
        grow(i+1);
    // 保持堆的特性，对当前元素进行上浮操作，也就是自下而上的堆化过程
    siftUp(i, e);
    size = i + 1;
    
    return true;
}

// 对堆的底层数组进行扩容
private void grow(int minCapacity) {
    // oldCapacity表示新加节点前数组的大小
    int oldCapacity = queue.length;
    // Double size if small; else grow by 50%
    // 如果oldCapacity小于64，那么新数组的大小newCapacity为2*oldCapacity+2;
    // 如果oldCapacity大于64，那么新数组的大小newCapacity为1.5*oldCapacity;
    int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                     (oldCapacity + 2) :
                                     (oldCapacity >> 1));
    // overflow-conscious code
    // 求出的newCapacity溢出的情况
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // 数组扩容，复制
    queue = Arrays.copyOf(queue, newCapacity);
}
// 节点x上浮操作，自下而上堆化
private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x, queue, comparator);
    else
        siftUpComparable(k, x, queue);
}
// 使用比较器进行上浮操作
private static <T> void siftUpUsingComparator(
    int k, T x, Object[] es, Comparator<? super T> cmp) {
    while (k > 0) {
        // k节点的父节点的索引
        int parent = (k - 1) >>> 1;
        // k节点的父节点
        Object e = es[parent];
        // 如果满足新加入的节点x大于其父节点e的条件的话，则说明满足堆的特性直接跳出循环
        if (cmp.compare(x, (T) e) >= 0)
            break;
        // 父节点下沉，即x节点上浮
        es[k] = e;
        k = parent;
    }
    es[k] = x;
}
// 自然排序进行上浮操作
private static <T> void siftUpComparable(int k, T x, Object[] es) {
    Comparable<? super T> key = (Comparable<? super T>) x;
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = es[parent];
        if (key.compareTo((T) e) >= 0)
            break;
        es[k] = e;
        k = parent;
    }
    es[k] = key;
}
```

通过源码可以看出，`offer(E e)`方法的主要步骤是：

- 先判断`e`是否为null，如果为null，抛出异常（**这说明优先级队列中不能加入null元素**）；
- 然后再判断优先级队列底层的数组是否要扩容，如果要扩容：
  - 原数组大小 < 64，新数组大小 = 2*原数组大小+2；
  - 原数组大小 > 64，新数组大小 = 1.5*原数组大小；
- 将`e`加入队列尾部，并进行堆化；

至于上浮操作，这里以`siftUpUsingComparator(int k, T x, Object[] es, Comparator<? super T> cmp)`为例，进行说明：

- 使用变量`parent`来记录节点`x`的父节点的索引，变量`e`记录节点`x`的父节点；
- 比较`e`和`x`的大小，如果`x`大于`e`的话，那么新加入的节点`x`仍然能保证堆特性，直接跳出循环；
- 否则，对`x`进行上浮操作；

### 移除元素

```java
// 移除优先级队列中队首的节点，并返回该节点
public E poll() {
    final Object[] es; // 记录优先级队列的底层数组
    final E result; // 记录优先级队列的第一个节点
	// 如果第一个节点不为null
    if ((result = (E) ((es = queue)[0])) != null) {
        modCount++;
        final int n;
        final E x = (E) es[(n = --size)]; // 表示队尾的元素
        es[n] = null; // 方便gc
        if (n > 0) {
            final Comparator<? super E> cmp;
            if ((cmp = comparator) == null)
                // 对队尾元素x在队首位置自然排序进行下沉堆化
                siftDownComparable(0, x, es, n);
            else
                siftDownUsingComparator(0, x, es, n, cmp);
        }
    }
    return result;
}

```

`poll()`是从优先级队列的头部移除并返回该节点值，**在这个操作之后为了保证堆的特性，首先将队列尾部的元素放到队首，然后对现在的队首元素进行下沉操作，直到能满足堆特性**；

```java
// 在优先级队列中移除对象o
public boolean remove(Object o) {
    int i = indexOf(o);
    if (i == -1)
        return false;
    else {
        removeAt(i);
        return true;
    }
}
// 在优先级队列中找到第一个等于对象o的索引
private int indexOf(Object o) {
    // 如果对象o不为null
    if (o != null) {
        final Object[] es = queue;
        // 遍历底层数组，找到数组中第一个与o相等的对象，并返回其位置索引
        for (int i = 0, n = size; i < n; i++)
            if (o.equals(es[i]))
                return i;
    }
    return -1;
}

// 移除索引为i的对象
E removeAt(int i) {
    // assert i >= 0 && i < size;
    final Object[] es = queue;
    modCount++;
    int s = --size;
    // 移除的是最后一个元素，那么直接移除
    if (s == i) // removed last element
        es[i] = null;
    else {
        // 否则将队尾的元素移动到要移除的位置，再调用下沉堆化的操作保证堆的特性
        E moved = (E) es[s];
        es[s] = null;
        siftDown(i, moved);
        if (es[i] == moved) {
            siftUp(i, moved);
            if (es[i] != moved)
                return moved;
        }
    }
    return null;
}
```

`remove(Object o)`在优先级队列中移除对象o，具体的过程是：

- 先在优先级队列中找到对象o的位置索引；
- 在将对应位置上的对象从堆中移除；

### 查找元素

```java
public E peek() {
    return (E) queue[0];
}
```

获得优先级队列头部元素，直接返回数组的第一个元素值即可；

## 总结

- PriorityQueue 是基于堆的，堆是一个特殊的完全二叉树，它的每一个节点的值都必须小于等于（或大于等于）其子树中每个节点的值；

- PriorityQueue 的出队和入队操作时间复杂度都是 `O(log(n))`，仅与堆的高度有关；

- PriorityQueue 初始容量为 11，支持动态扩容。容量小于 64 时，扩容一倍。大于 64 时，扩容 0.5 倍；

- PriorityQueue 不允许 null 元素，不允许不可比较的元素；

- PriorityQueue 不是线程安全的，PriorityBlockingQueue 是线程安全的；



## 参考文献

https://juejin.im/post/5cbb2a996fb9a0689d6f9ca4#heading-0

https://juejin.im/post/5cdaa3d751882568ef1f3d28#heading-1







