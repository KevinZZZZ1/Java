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

![avatar](F:\找工作\Java基础\Java\images\LinkedListUML.png)

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

在`node()`函数中，就体现了`first`指针和`last`指针的用处：将LinkedList根据长度平分成前后两个部分,如果index在前半部分，则从第一个节点开始遍历；否则就从最后一个节点开始遍历；



## LinkedList的操作



### LinkedList添加节点的方法





### LinkedList删除节点的方法



### LinkedList查询节点的方法





### LinkedList修改节点的方法





### LinkedList元素查询方法





## LinkedList作为双向队列的增删改查





### Deque双端队列







### Deque和Queue添加元素的方法





### Deque和Queue删除元素的方法





### Deque和Queue获取队列头部元素的实现







## LinkedList的遍历





## 总结





