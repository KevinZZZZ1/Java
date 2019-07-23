# List-ArrayList

[TOC]

## ArrayList结构

### ArrayList的创建

**ArrayList也是一个普通类，所以在创建ArrayList对象时，也是在堆上创建的**；如下面这段代码（以下代码都是基于JDK1.8的）：

```java
List<Person> list1 = new ArrayList<>();
List<Person> list2 = new ArrayList<>();

Person person1 = new Person("张三");
list1.add(person);
```

上面这段代码执行完之后，JVM的运行时内存如下图所示:

![avatar](https://github.com/KevinZZZZ1/Java/blob/master/images/%E5%88%9B%E5%BB%BAArrayList.png)

其中`DEFAULTCAPACITY_EMPTY_ELEMENTDATA`是一个`Object`类型的数组，具体的用处下文再解释；

### ArrayList继承关系、接口实现

![avatar](https://github.com/KevinZZZZ1/Java/blob/master/images/ArrayListUML.png)

从图中可以看出ArrayList继承了AbstractList类，实现了List、RandomAccess、Serializable、Cloneable接口；

- 实现RandomAccess接口，说明能够支持随机访问，而且该接口是一个标记接口，ArrayList真正随机访问的功能是由底层的数组提供的，该接口只是起一个标识的作用；
- 实现Cloneable接口，能被克隆，该接口也是一个标记接口，没有任何方法要实现；
- 实现Serializable，能被序列化，该接口也是一个标记接口，没有任何方法要实现；

## 成员变量

```java
public class ArrayList<E> extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
    
    private static final long serialVersionUID = 8683452581122892189L;
    private static final int DEFAULT_CAPACITY = 10;
    private static final Object[] EMPTY_ELEMENTDATA = {};
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    transient Object[] elementData; // non-private to simplify nested class access
    private int size;
	// 省略其他代码...
}
```

根据以上代码我们能得出以下几个结论：

- ArrayList底层使用Object数组存储元素；

- size属性表示数组中元素的个数；

- DEFAULT_CAPACITY默认容量是10；

- EMPTY_ELEMENTDATA 空集合；

- DEFAULTCAPACITY_EMPTY_ELEMENTDATA 默认空集合 （与EMPTY_ELEMENTDATA有点区别，在不同的构造函数中用到），第一个元素添加时会扩容成默认容量10；

  

## 构造函数

```java
public ArrayList(int initialCapacity){
    if(initialCapacity>0){
        this.elementData = new Object[initialCapacity];
    } else if(initialCapacity == 0){
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
    
    
}
```

- 该构造函数是指定了初始化长度的，并且该长度必须大于等于0，如果等于0，会被赋值上一个EMPTY_ELEMENTDATA表示空集合；（**EMPTY_ELEMENTDATA和DEFAULTCAPACITY_EMPTY_ELEMENTDATA 的不同体现在添加元素的时候，下文将给出解释**）；

```java
public ArrayList(){
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

- 这是平时最常用的构造函数，其中会将elementData数组的大小设置为DEFAULT_CAPACITY，也就是10；
- **这里并没有着急创建长度为10的Object数组，而是将一个空数组DEFAULTCAPACITY_EMPTY_ELEMENTDATA赋值给了elementData，这么做的原因就是延迟内存分配，等到需要使用的时候才分配内存，而且DEFAULTCAPACITY_EMPTY_ELEMENTDATA被定义成了静态常量，是存在于方法区的常量池中的，当创建多个使用默认构造函数的ArrayList时，elementData数组都会指向该静态常量**；

```java
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if((size = elementData.length) != 0){
        if(elementData.getClass() != Object[].class)
			elementData = Arrays.copyOf(elementData, size, Object[].class);
            
    } else{
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

- 这里是利用已有集合来创建ArrayList，从代码中可以看出，首先把集合转换成数组赋值给elementData，然后判断elementData数组的长度是否大于0，如果是，再判断该数组类型是否为Object[]，如果是则直接使用该数组，否则使用Arrays.copyOf函数进行复制；



## 常用操作

对于ArrayList以及其他的集合类来说，最重要的就是增删改查四个操作；



### 添加操作

```java
public boolean add(E e){
    // 检查是否要扩容，如果要扩容也是在这步完成
    ensureCapacityInternal(size+1);
    // 向数组中添加元素
    elementData[size++] = e;
    return true;
}

// minCapacity表示需要的最小容量
private void ensureCapacityInternal(int minCapacity){
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private static int calculateCapacity(Object[] elementData, int minCapacity){
    // 这里就是判断elementData是否是刚刚使用无参构造函数创建的，返回的是默认值和minCapacity中最大的那个
    if(elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA){
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

private void ensureExplicitCapacity(int minCapacity){
    modCount++;
    // elementData的大小无法满足最小需求，则需要进行扩容
    if(minCapacity - elementData.length > 0)
        grow(minCapacity)
}

private void grow(int minCapacity){
    int oldCapacity = elementData.length;
    // 尝试扩容为原来的1.5倍
    int newCapacity  = oldCapacity + (oldCapacity>>1);
    // 尝试扩容大小仍然不够，使用最小需求作为扩容后的数组大小
    if(newCapacity - minCapacity<0)
        newCapacity = minCapacity;
    // 对于过大的newCapacity值进行处理，防止溢出
    if(newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // 扩容成功移动数据
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity){
    if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
    MAX_ARRAY_SIZE;
   
}

```

- size表示ArrayList的元素个数。在添加元素之前先要确保数组有足够容量。当当前数组需要的空间不够时，就需要扩容了，并且保证新容量不能比当前需要的容量小，然后调Arrays.copyOf()创建一个新的数组并将数据拷贝到新数组中，且把引用赋值给elementData

```java
public void add(int index, E element) {
    rangeCheckForAdd(index);
    
    ensureCapacityInternal(size + 1);
    // 既然要添加元素，就要保证有足够的数组空间；当然要在index位置插入元素，得让index后所有的元素往后移动一位，腾出index位置设置要添加的元素。
    System.arrayCopy(elementData, index, elementData, index+1, size-index);
    elementData[index] = element;
    size++;
}
private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
```

- 指定数组index位置添加元素。首先会检查index是否超出数组的范围，然后会把index位置以后的元素向后移一位，然后再添加；
- 注意这里`arraycopy()`函数的用法，先给出该函数的完整定义`public static void arraycopy(Object src, int srcPos, Object dest, int destPos, int length)`其中：　　Object src : 原数组 int srcPos : 从元数据的起始位置开始 Object dest : 目标数组 int destPos : 目标数组的开始起始位置 int length  : 要copy的数组的长度
- `arraycopy()`的作用主要是移位，比如ArrayList底层数组elementData为[0,1,2,3,4,5]，当index为2时，执行`arraycopy(elementData, index, elementData, index+1, size-index)`得到的结果是：[0,1,2,2,3,4,5]，**也就是说会从index=2+1=3的位置开始，将数组元素替换为从index起始位置为index=2，长度为6-2=4的数据。**


### 删除操作

```java
public E remove(int index) {
    rangeCheck(index);

    modCount++;
    E oldValue = elementData(index);
	// 删除之后需要向前移动的位置数
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

- 删除指定位置的元素。首先也是检查index的合法性，然后取得该位置的旧元素，计算需要移动的长度，如果需要移动的，则调用System.arraycopy方法将index位置后的元素所有往前移动一位，将数组最后一位置为null，方便GC工作，最后返回被删除的元素；

```java
  public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            // 对于null的处理
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            // 删除第一个等于o的元素
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}


 private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

- 删除指定元素，这里删除的是数组中第一个找到的元素，而且通过源码我们也可以知道，**ArrayList中是可以加入null的，因为在删除代码中有对于null的处理**



### 获取操作

```java
public E get(int index){
    rangeCheck(index);
    return elementData(index);
}

E elementData(int index){
    return (E) elementData[index];
}
```

- 获取指定index位置的元素。这个实现很简单，首先index范围检查，然后直接取数组中index位置元素返回

### 修改操作

```java
public E set(int index, E element){
	rangeCheck(index);
    
    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}
```

- 修改指定位置的元素，并返回旧值

### 迭代器

使用集合的都知道，在for循环遍历集合时不可以对集合进行删除操作，因为删除会导致集合大小改变，从而导致数组遍历时数组下标越界，严重时会抛ConcurrentModificationException异常，（这是为了防止多线程同时修改，导致数组下标越界）

```java
public Iterator<E> iterator() {
    return new Itr();
}

private class Itr implements Iterator<E> {
    int cursor;       // index of next element to return
    int lastRet = -1; // index of last element returned; -1 if no such
    int expectedModCount = modCount;

    Itr() {}

    public boolean hasNext() {
        return cursor != size;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        checkForComodification();
        int i = cursor;
        if (i >= size)
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length)
            throw new ConcurrentModificationException();
        cursor = i + 1;
        return (E) elementData[lastRet = i];
    }

    public void remove() {
        if (lastRet < 0)
            throw new IllegalStateException();
        checkForComodification();

        try {
            ArrayList.this.remove(lastRet);
            cursor = lastRet;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

    @Override
    @SuppressWarnings("unchecked")
    public void forEachRemaining(Consumer<? super E> consumer) {
        Objects.requireNonNull(consumer);
        final int size = ArrayList.this.size;
        int i = cursor;
        if (i >= size) {
            return;
        }
        final Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length) {
            throw new ConcurrentModificationException();
        }
        while (i != size && modCount == expectedModCount) {
            consumer.accept((E) elementData[i++]);
        }
        // update once at end of iteration to reduce heap write traffic
        cursor = i;
        lastRet = i - 1;
        checkForComodification();
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

- Itr对象有三个成员变量：
  - cursor：代表下一个要访问的数组下标
  - lastRet：代表上一个要访问的数组下标
  - expectedModCount：代表ArrayList修改次数的期望值，初始为modeCount
- Itr有三个主要函数：
  - hasNext：实现简单，判断下一个要访问的数组下标等于数组大小，表示遍历到最后；
  - next：首先判断 expectedModCount 和 modCount 是否相等。每调用一次 next 方法， cursor 和 lastRet 都会自增 1；
  -  remove ：首先会判断 lastRet  的值是否小于 0，然后在检查 expectedModCount 和 modCount 是否相等。然后直接调用 ArrayList 的 remove 方法删除下标为 lastRet 的元素。然后将 lastRet 赋值给 cursor ，将 lastRet 重新赋值为 -1，并将 modCount 重新赋值给 expectedModCount；
- remove方法弊端：调用 remove 之前必须先调用 next。因为 remove 开始就对 lastRet 做了校验。而 lastRet 初始化时为 -1。next 之后只可以调用一次 remove。因为 remove 会将 lastRet 重新初始化为 -1



### 序列化

```java
private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
    // Write out element count, and any hidden stuff
    int expectedModCount = modCount;
    s.defaultWriteObject();
    // Write out array length
    s.writeInt(elementData.length);
    // Write out all elements in the proper order.
    for (int i=0; i<size; i++)
               s.writeObject(elementData[i]);
        if (modCount != expectedModCount) {
               throw new ConcurrentModificationException();
        }
}
```

- elementData是由transient修饰的，说明在序列化ArrayList的时候不会序列化elementData；
- 但是重写了`writeObject()`保证序列化的时候虽然不序列化全部 但是有的元素都序列化；

## 总结

- ArrayList是一个可以自动扩容的动态数组；
- ArrayList的默认容量大小是10；
- 扩容为原来的1.5倍，如果1.5倍还不够的话，直接扩容成我们所需要的容量，1.5倍或所需容量太大的话，直接扩容成Integer.MAX_VALUE或MAX_ARRAY_SIZE；
- 扩容之后通过数组拷贝确保元素的准确性，尽量减少扩容机制；
- 复制和扩容使用了Arrays.copyOf和System.arraycopy方法；
- ArrayList查找效率高，插入删除操作效率相对低；
- size 为集合实际存储元素个数；
- elementData.length 为数组长度，表示数组可以存储多少个元素；
- 如果需要边遍历边 remove ，必须使用 iterator。且 remove 之前必须先 next，next 之后只能用一次 remove；



