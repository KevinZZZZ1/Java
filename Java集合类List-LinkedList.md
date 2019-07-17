# List-LinkedList

[TOC]

## LinkedList概述

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable {
    transient int size = 0;

    /**
     * Pointer to first node.
     */
    transient Node<E> first;

    /**
     * Pointer to last node.
     */
    transient Node<E> last;
    ...
    // 省略其他代码
}
```

从上面的源码可以看出，LinkedList继承了AbstractSequentialList抽象类，实现了List、Deque、Cloneable、Serializable接口；

![avatar](https://github.com/KevinZZZZ1/Java/blob/master/images/LinkedListUML.png)

- 实现Serializable、Cloneable两个标识接口，表示LinkedList能实现序列化和能被克隆；
- 实现List、Deque接口，表示LinkedList不但有List的相关操作，还有Deque的操作；



**LinkedList的主要特性：**

- **LinkedList底层实现是一个双向链表**；
- **LinkedList允许添加的元素为null**；
- **LinkedList允许添加重复的数据**；
- **LinkedList是非线程安全的，如果想使用线程安全的LinkedList，要使用：List list = Collections.synchronizedList(new LinkedList(...));来生成一个线程安全的LinkedList**



**LinkedList中的双向链表**:

- **包括：data、prev、next三个数据域**；



## LinkedList双向链表节点实现及成员变量

#### 双向链表节点的实现

```java
private static class Node<E> {
    // 数据域，存放需要存储的数据
    E item;
    // 指向下一个节点
    Node<E> next;
    // 指向前一个节点
    Node<E> prev;
    
    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

可以从源码中看出，双向链表的节点的实现还是比较简单直观的；

#### 成员变量

```java
// 表示LinkedList中元素的个数
transient int size = 0;
// 指向双向链表的第一个节点
transient Node<E> first;
// 指向双向链表的最后一个节点
transient Node<E> last;
```

其中，之所以保存链表的第一个节点和最后一个节点，是因为在遍历LinkedList的时候能够比较index和size/2的大小，从而选择是从头开始查找还是从尾部开始查找，在一定程度上加快了遍历的速度；



## LinkedList的构造函数

LinkedList有2个构造函数：

- 无参构造函数；
- Collection作为参数的构造函数；

```java
public LinkedList(){
}
```

该构造函数会创建一个空的LinkedList，其中first=last=null;

```java
// 传入一个集合类，将该集合类变成一个LinkedList
public LinkedList(Collection<? extends E> c){
    this();
    addAll(c);
}

public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);
}

// 在index节点处插入集合c中的所有元素
public boolean addAll(int index, Collection<? extends E> c){
    // 检查index是否有效，即0<=index<=size
    checkPositionIndex(index);
    // 将集合类转化为数组
    Object[] a = c.toArray();
    int numNew = a.length;
    // 如果数组为空，直接return false表示没有添加任何元素
    if(numNew == 0)
        return false;
    // pred表示index位置处的前一个节点，succ表示位于index位置的节点
    Node<E> pred, succ;
    // 在尾部插入
    if(index == size){
        succ = null;
        pred = last;
    } else {
        succ = node(index);
        pred = succ.prev;
    }
    
    // 遍历数组，将对应元素加入LinkedList中
    for(Object o: a) {
        E e = (E) o;
        Node<E> newNode = new Node<>(pred, e, null);
        
        // pred为空就是说，是在是在头部插入
        if(pred == null)
        	first = newNode;
        else
            pred.next = newNode;
        
        pred = newNode;
    }
    // 在尾部插入时，最后要将last指向最后一个节点
    if(succ == null){
        last = pred;
    } else{
        // 普通情况
        pred.next = succ;
        succ.prev = pred
    }
    // 更新size
    size += numNew;
    modCount++;
    return true
}

// 获得指定位置上的节点，将LinkedList根据长度平分成前后两个部分,如果index在前半部分，则从第一个节点开始遍历；否则就从最后一个节点开始遍历
Node<E> node(int index){
    // index在前半部分，则从第一个节点开始遍历
    if(index < (size>>1)){
        Node<E> x = first;
        for(int i=0; i<index; i++)
            x = x.next;
        return x;
    } else {
    // index在后半部分，从最后一个节点开始遍历
        Node<E> x = last;
        for(int i=size-1; i>index; i--)
            x = x.prev;
        return x;
    }
    
}

```

从上面的源码可以看出，集合类作为参数的LinkedList构造函数，调用 `addAll(c)` 这个方法，实际上这方法调用了 `addAll(size, c)` 方法，在外部单独调用时，将指定集合的元素作为节点，添加到 `LinkedList` 链表尾部： 而 `addAll(size, c)` 可以将集合元素插入到指定索引节点；

`addAll()`函数的处理流程如下：

- 检查index是否合法，不合法抛出异常；
- 保存index位置和index-1位置的节点，方便之后链表的链接；
- 将参数集合转化为数组，遍历数组将数组中元素加入链表中；
- 更新链表长度并返回true表示添加成功；

**在`node()`函数中，就体现了`first`指针和`last`指针的用处：将LinkedList根据长度平分成前后两个部分,如果index在前半部分，则从第一个节点开始遍历；否则就从最后一个节点开始遍历；**



## LinkedList的操作
由于LinkedList是基于链表实现的，所以其插入、删除操作比较方便，但是随机访问的性能较差


### LinkedList添加节点的方法

```java
public boolean add(E e) {
    linkLast(e);
    return true;
}
// 在链表的末尾添加一个元素
void linkLast(E e) {
    final Node<E> 1 = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null) // 之前的链表是空链表
        first = newNode;
    else
        l.next = newNode;
    size++;
    modCount++;
}
```

通过上面源码可以看出，LinkedList的`add(E e)`方法是默认在链表尾部添加一个元素；

```java
// 在index位置处添加节点
public void add(int index, E element) {
    // 对index进行检查，保证0<=index<=size
    checkPositionIndex(index);
	
    if (index == size) // 链表尾部添加节点
        linkLast(element);
    else
        // 先通过node(index)函数得到位于index位置的节点
        linkBefore(element, node(index));
}

// 在succ节点前插入一个值为e的新节点
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null) // 在链表头部插入
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

LinkedList的`add(int index, E element)`方法是在指定位置的index上添加节点：

- 先检查index是否在区间[0, size]中，如果不是，抛出IndexOutOfBoundsException异常；
- 再判断index在链表中的位置：
  - 如果index在链表尾部，则调用`linkLast()`进行添加;
  - 否则先使用`node()`函数找到index位置上的节点，再使用`linkBefore`函数在index对应的节点之间添加一个节点

```java
// 在链表头部添加元素
public void addFirst(E e) {
    linkFirst(e);
}
// 在链表尾部添加元素
public void addLast(E e) {
    linkLast(e);
}

private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null) // 在添加之前是一个空链表
        last = newNode;
    else
        f.prev = newNode;
    size++;
    modCount++;
}
```

LinkedList的`addFirst(E d)`和`addLast(E e)`方法

### LinkedList删除节点的方法

```java
public void clear() {
    // Clearing all of the links between nodes is "unnecessary", but:
    // - helps a generational GC if the discarded nodes inhabit
    //   more than one generation
    // - is sure to free memory even if there is a reachable Iterator
    for (Node<E> x = first; x != null; ) {
        // 先记录下当前处理节点x的下一个节点
        Node<E> next = x.next;
        x.item = null;
        x.next = null;
        x.prev = null;
        x = next;
    }
    first = last = null;
    size = 0;
    modCount++;
}
```

`clear()`方法，清除LinkedList中所有节点的数据，将其赋值为null，方便GC

```java
// remove方法默认移除链表头部的节点
public E remove() {
    return removeFirst();
}

public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}
// 移除链表头部的节点
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    first = next;
    if (next == null) // 移除之前，链表中只有一个节点的情况
        last = null;
    else
        next.prev = null;
    size--;
    modCount++;
    return element;
}
```

LinkedList的`remove()`方法，默认是移除链表首部的节点

```java
// 移除指定index位置的节点
public E remove(int index) {
    checkElementIndex(index);
    // node(index)获得index位置的节点，
    return unlink(node(index));
}
// 移除节点x
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) { // x是头结点的情况
        first = next;
    } else {
        prev.next = next;
        x.prev = null;
    }

    if (next == null) { // x是尾节点的情况
        last = prev;
    } else {
        next.prev = prev;
        x.next = null;
    }
	// 如果上面两个if都满足的话，说明链表中只有x一个节点
    
    x.item = null;
    size--;
    modCount++;
    return element;
}
```

LinkedList中的`remove(int index)`方法：

- 先判读index是否在[0, size]区间内，如果不是抛出IndexOutOfBoundsException异常；
- 使用`node(index)`函数得到index位置对应的节点；
- 再使用`unlink()`函数将index位置的节点移除链表；

```java
public boolean remove(Object o) {
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```

LinkedList中的`remove(Object o)`方法：就是遍历链表，比较链表中每个元素是否等于o，也就是说，**移除的是第一个相等的元素**，需要注意的是，**要对null进行特殊处理，因为判断等于null要用==，而判断对象是否相等用equals()函数**；

```java
public E removeFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return unlinkFirst(f);
}

public E removeLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return unlinkLast(l);
}
```

`removeFirst()`、`removeLast()`都是调用之前实现的函数，分别移除链表头部节点和链表尾部节点；

### LinkedList查询节点的方法

```java
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}
```

LinkedList中的`get(int index)`函数，返回index位置的节点的值：

- 先对index进行边界检查；
- 再调用`node(index)`函数找到index位置的节点，并返回其值；

```java
public E getFirst() {
    final Node<E> f = first;
    if (f == null)
        throw new NoSuchElementException();
    return f.item;
}

public E getLast() {
    final Node<E> l = last;
    if (l == null)
        throw new NoSuchElementException();
    return l.item;
}
```

`getFirst()`和`getLast()`分别返回链表的头节点的值和尾节点的值；

### LinkedList修改节点的方法

```java
public E set(int index, E element) {
    checkElementIndex(index);
    Node<E> x = node(index);
    E oldVal = x.item;
    x.item = element;
    return oldVal;
}
```

`set(int index, E element)`函数是将index位置的节点值置为element：

- 先检查index是否越界；
- 使用`node(index)`函数找到index位置的节点；
- 将新值存入index对应的节点，并返回旧值；

### LinkedList元素查询方法

```java
public boolean contains(Object o) {
    return indexOf(o) != -1;
}

public int indexOf(Object o) {
    int index = 0;
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null)
                return index;
            index++;
        }
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item))
                return index;
            index++;
        }
    }
    return -1;
}
```

`indexOf(Object o)`函数是**从链表头部开始遍历链表，**返回对象o在链表中的位置，如果返回-1表示不存在（**还是要注意判断是否等于null的特殊情况**）；

```java
public int lastIndexOf(Object o) {
    int index = size;
    if (o == null) {
        for (Node<E> x = last; x != null; x = x.prev) {
            index--;
            if (x.item == null)
                return index;
        }
    } else {
        for (Node<E> x = last; x != null; x = x.prev) {
            index--;
            if (o.equals(x.item))
                return index;
        }
    }
    return -1;
}
```

`lastIndexOf(Object o)`函数是**从链表尾部开始遍历链表，**返回对象o在链表中的位置，如果返回-1表示不存在；

## LinkedList作为双向队列的增删改查

LinkedList实现Deque的操作大部分都是复用了在List接口中的操作，所以只要知道对应关系就行，代码都是一样的；

### Deque双端队列

由于 `Deque` 接口继承 `Queue` 接口，当 `Deque` 当做队列使用时（FIFO），只需要在头部删除，尾部添加即可。我们现在复习下 `Queue` 中的方法及区别：

1. `Queue` 的 `offer` 和 `add` 都是在队列中插入一个元素，具体区别在于，对于一些 Queue 的实现的队列是有大小限制的，因此如果想在一个满的队列中加入一个新项，多出的项就会被拒绝。此时调用 `add()`方法会抛出异常，而 `offer()` 只是返回的 false。
2. `remove()` 和 `poll()` 方法都是从队列中删除第一个元素。remove()也将抛出异常，而 `poll()` 则会返回 `null`
3. `element()` 和 `peek()` 用于在队列的头部查询元素。在队列为空时， `element()` 抛出一个异常，而 `peek()` 返回 `null`。

### Deque和Queue添加元素的方法

```java
// queue 的添加方法实现，
public boolean add(E e) {
   linkLast(e);
   return true;
}
// Deque 的添加方法实现，
public void addLast(E e) {
   linkLast(e);
} 
  
// queue 的添加方法实现，
public boolean offer(E e) {
   return add(e);
}

// Deque 的添加方法实现，
public boolean offerLast(E e) {
        addLast(e);
        return true;
}
```

### Deque和Queue删除元素的方法

```java
// Queue 删除元素的实现 removeFirst 会抛出 NoSuchElement 异常
public E remove() {
   return removeFirst();
}

// Deque 的删除方法实现
public E removeFirst() {
   final Node<E> f = first;
   if (f == null)
       throw new NoSuchElementException();
   return unlinkFirst(f);
}
    
// Queue 删除元素的实现 不会抛出异常 如果链表为空则返回 null 
public E poll() {
   final Node<E> f = first;
   return (f == null) ? null : unlinkFirst(f);
}

// Deque 删除元素的实现 不会抛出异常 如果链表为空则返回 null 
public E pollFirst() {
   final Node<E> f = first;
   return (f == null) ? null : unlinkFirst(f);
}
```

### Deque和Queue获取队列头部元素的实现

```java
// Queue 获取队列头部的实现 队列为空的时候回抛出异常
 public E element() {
    return getFirst();
 }
// Deque 获取队列头部的实现 队列为空的时候回抛出异常
public E getFirst() {
   final Node<E> f = first;
   if (f == null)
       throw new NoSuchElementException();
   return f.item;
}

// Queue 获取队列头部的实现 队列为空的时候返回 null
public E peek() {
   final Node<E> f = first;
   return (f == null) ? null : f.item;
}

// Deque 获取队列头部的实现 队列为空的时候返回 null
public E peekFirst() {
   final Node<E> f = first;
   return (f == null) ? null : f.item;
}
```



## LinkedList的遍历

```java
private class ListItr implements ListIterator<E> {
   // 上一个遍历的节点
   private Node<E> lastReturned;
   // 下一次遍历返回的节点
   private Node<E> next;
   // cursor 指针下一次遍历返回的节点
   private int nextIndex;
   // 期望的操作数
   private int expectedModCount = modCount;
    
   // 根据参数 index 确定生成的迭代器 cursor 的位置
   ListItr(int index) {
       // assert isPositionIndex(index);
       // 如果 index == size 则 next 为 null 否则寻找 index 位置的节点
       next = (index == size) ? null : node(index);
       nextIndex = index;
   }

   // 判断指针是否还可以移动
   public boolean hasNext() {
       return nextIndex < size;
   }
    
  // 返回下一个带遍历的元素
  public E next() {
       // 检查操作数是否合法
       checkForComodification();
       // 如果 hasNext 返回 false 抛出异常，所以我们在调用 next 前应先调用 hasNext 检查
       if (!hasNext())
           throw new NoSuchElementException();
        // 移动 lastReturned 指针
       lastReturned = next;
        // 移动 next 指针
       next = next.next;
       // 移动 nextIndex cursor
       nextIndex++;
       // 返回移动后 lastReturned
       return lastReturned.item;
   }

  // 当前游标位置是否还有前一个元素
   public boolean hasPrevious() {
       return nextIndex > 0;
   }
  
  // 当前游标位置的前一个元素
   public E previous() {
       checkForComodification();
       if (!hasPrevious())
           throw new NoSuchElementException();
        // 等同于 lastReturned = next；next = (next == null) ? last : next.prev;
        // 发生在 index = size 时
       lastReturned = next = (next == null) ? last : next.prev;
       nextIndex--;
       return lastReturned.item;
   }
    
   public int nextIndex() {
       return nextIndex;
   }

   public int previousIndex() {
       return nextIndex - 1;
   }
    
    // 删除链表当前节点也就是调用 next/previous 返回的这节点，也就 lastReturned
   public void remove() {
       checkForComodification();
       if (lastReturned == null)
           throw new IllegalStateException();

       Node<E> lastNext = lastReturned.next;
       //调用LinkedList 的删除节点的方法
       unlink(lastReturned);
       if (next == lastReturned)
           next = lastNext;
       else
           nextIndex--;
       //上一次所操作的 节点置位空    
       lastReturned = null;
       expectedModCount++;
   }

    // 设置当前遍历的节点的值
   public void set(E e) {
       if (lastReturned == null)
           throw new IllegalStateException();
       checkForComodification();
       lastReturned.item = e;
   }
    // 在 next 节点位置插入及节点
   public void add(E e) {
       checkForComodification();
       lastReturned = null;
       if (next == null)
           linkLast(e);
       else
           linkBefore(e, next);
       nextIndex++;
       expectedModCount++;
   }
    //简单哈操作数是否合法
   final void checkForComodification() {
       if (modCount != expectedModCount)
           throw new ConcurrentModificationException();
   }
}
```





## 总结

- `LinkedList` 基于双向链表实现，内存中不连续，不具备随机访问，插入和删除效率较高，查找效率较低。使用上没有大小限制，天然支持扩容。 
- 双向链表由于可以反向遍历，相较于单向链表在某些操作上具有性能优势，但是由于每个结点都需要额外的内存空间来存储前驱指针，所以双向链表相对来说需要占用更多的内存空间，这也是 **空间换时间** 的一种体现。 

 ## 参考文献

<https://juejin.im/post/5abfde3f6fb9a028e46ec51c#heading-9> 

 

 

 


