# `final`、`finally`以及`finalize`

[TOC]

## ` final`

- `final`可以用来修饰类、方法、变量，分别有不同的意义：

  - 修饰类：表示该类不可继承；
  - 修饰方法：表示该方法不可重写；
  - 修饰变量：表示该变量不可修改；

- `final`变量在某种程度上可以达到不可变（immutable）的效果，可以用于保护只读数据，比如在并发编程中，由于`final`变量是不可修改的，这样访问变量时就不需要加锁，提升了程序性能；

- 需要注意的是`final`变量并不等于immutable，比如下面这段代码：

  ```java
  final List<String> strList = new ArrayList<>();
  strList.add("1111");
  strList.add("2222");
  List<String> unmodifiableStrList = List.of("3333", "4444");
  unmodifiableStrList.add("5555");
  ```

  ```
  执行结果是：
  能对strList进行add操作，不能对unmodifiableStrList进行操作
  ```

  - `final`只能约束`strList`这个变量不能再被赋值，而对于`strList`所指向的对象不被`final`限制；
  - `List.of`方法创建的`List`本身就是不可变的，所以进行操作时会抛出异常；

- 实现immutable类，需要做到：

  - 将类定义为`final`；
  - 将所有成员变量定义为`final`和`private`，并且不要实现`setter`方法；
  - 在构造对象时，成员变量使用深拷贝来初始化，而不是直接赋值；
  - 如果需要实现`getter`方法，或者其他会返回内部状态的方法，使用`copy-on-write`原则；



## `finally`

- `finally`是`Java`保证重点代码必须会执行的机制，一般使用`try-finally`和`try-catch-finally`来进行如关闭`JDBC`操作，解锁等动作；

- `finally`在下面代码的情况下是无法执行的：

  ```java
  try{
      // do something
      System.exit(1);
  }finally{
      System.out.println("Print from finally");
  }
  ```

  





## `finalize`

- `finalize`是`java.lang.Object`类中的一个方法，设计的目的是保证在虚拟机垃圾回收之前完成特定资源回收，但是`finalized`机制已经不推荐使用，被设置为废弃方法；
- 不使用`finalize`原因：无法保证 finalize 什么时候执行，执行的是否符合预期，使用不当会影响性能、导致程序死锁；
- `finalize`与C++中的析构函数不是对应的。C++中的析构函数调用的时机是确定的（对象离开作用域或delete掉），但`Java`中的`finalize`的调用具有不确定性；
- `finalize`方法执行大体流程是：当对象变成`GC Roots`不可达时，GC会判断该对象是否覆盖了`finalize`方法，如未覆盖，则直接进行回收；否则，且该对象的`finalize`方法未执行，会将该对象放入`F-Queue`队列，由低优先级线程执行该队列中对象的`finalize`方法；执行完`finalize`方法之后，GC会再次判断对象是否可达，如不可达，直接回收；否则对象复活；








