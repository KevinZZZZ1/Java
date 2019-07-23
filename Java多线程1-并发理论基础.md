# 并发理论基础

[TOC]

## 可见性、原子性和有序性

- 可见性：多核环境下，缓存造成的变量可见性被破坏；
- 原子性：由于高级语言的一条语句，会分解成多个CPU指令，多线程时分复用的操作系统中，在任意指令结束都可能会发生线程切换，这样导致高级语言语句的原子性无法保证；
- 有序性：由于编译器对代码的优化会导致程序运行的顺序和代码书写顺序不一样，这就导致程序的有序性无法保证，在一些场合下会导致并发程序出现问题；

硬件设备的不断迭代，使得CPU的处理速度越来越快，但是这会导致一个问题：**CPU、内存、I/O设备的速度不匹配的问题**，CPU与内存的处理速度差异相对来说还行，内存和I/O设备之间的速度差异则是巨大的，**所以I/O操作成为了大部分系统的瓶颈**，为了解决这一问题，有以下三种解决方案：

- CPU增加了缓存，均衡CPU和内存间的速度差异；
- 操作系统增加了线程、进程，以分时复用CPU，均衡内存和I/O设备间的速度差异；
- 编译程序对代码指令执行次序进行优化，能更充分的使用缓存；

但是，并发编程问题多的原因正是来自于上面的成果；

### 缓存导致的可见性问题

在单核CPU时代，所有线程都是运行在同一个CPU上，使用同一缓存，即使线程A对缓存中的变量V进行了修改，切换到线程B之后，线程A对变量V的修改，线程B是可知的，这叫做**可见性**；

**在多核时代，缓存就会带来可见性问题**，即如果线程A在CPU1上执行，并在其缓存中修改了变量V的值，而线程B在CPU2上执行，其缓存中并不能知道线程A对于变量V的修改，所以变量的可见性就无法保证；



### 线程切换带来的原子性问题

由于I/O速度太慢，如果让进程一直阻塞等待I/O响应，导致CPU利用率低，会浪费CPU资源，所以**多进程分时复用操作系统的出现完美的解决了CPU利用率低的问题**；

多进程分时复用操作系统运行某个进程执行一个时间片的时间，在下一个时间片操作系统会重新选择一个进程来执行，这就叫做任务切换；早期操作系统使用进程作为任务切换的单位，**但是进程不共享内存空间，切换起来很麻烦，而一个进程创建的所有线程是共享内存空间的，切换起来成本很低，所以现在使用线程进行任务调度**；

**线程切换（任务切换）是导致并发编程结果不确定的原因之一，由于高级语言的一条语句是由多条CPU指令组成的**，线程切换一般发生在时间片结束时，此时一条语句的CPU指令可能还没执行完，导致高级语言语句的原子性无法保证；（CPU指令的原子性可以保证）

比如这个执行count++的例子：

![avatar](F:\找工作\Java基础\Java\images\并发编程原子性.png)

**我们潜意识会任务count++是一个不可分割的整体，其实并不是这样的，这也是并发程序出现bug的原因之一**；

### 编译优化带来的有序性问题

有序性是指，程序按照代码的书写的先后顺序进行执行，**但是在编译的时候编译器可能会进行优化，导致程序执行的时候并不是按照代码的书写顺序执行**，这样的优化可能就会导致并发编程出现bug；

一个经典的例子就是，利用双重检查创建单例对象：

```java
public class Singleton{
    private static Singleton instance;
    public static Singleton getInstance(){
        if(instance == null){
            synchronized(Singleton.class){
                if(instance == null)
                    instance = new Singleton();
            }
            
        }
        return instance;
    }
}
```

表面上看假设有两个线程 A、B 同时调用 getInstance()方法，他们会同时发现 instance == null ，于是同时instance == null ，于是同时对 Singleton.class 加锁，此时 JVM 保证只有一个线程能够加锁成功（假设是线程 A），另外一个线程则会处于等待状态（假设是线程 B）；线程 A 会创建一个 Singleton 实例，之后释放锁，锁释放后，线程 B 被唤醒，线程 B 再次尝试加锁，此时是可以加锁成功的，加锁成功后，线程 B 检查 instance == null时会发现，已经创建过 Singleton 实例了，所以线程 B 不会再创建一个 Singleton 实例。

目前为止，感觉上面程序并没有问题，**但是在考虑过有序性问题之后就会发现上面代码的问题**：

我们以为的new操作是：

- 分配一块内存M；
- 在内存M上初始化Singleton对象；
- 把M的地址赋值给instance变量；

其实，由于编译器优化代码导致有序性无法保证，实际的执行是这样的：

- 分配一块内存M；
- 将M的地址赋值给instance变量；
- 在内存M上初始化Singleton对象；

这就会导致如下图的问题：

![avatar](F:\找工作\Java基础\Java\images\并发编程有序性.png)

如何修改这个bug呢？由于问题出在无法保证有序性，**所以需要一种手段来保证原子性，即使用volatile关键字来修饰instance变量，从而保证有序性**

```java
public class Singleton{
    private volatile static Singleton instance;
    public static Singleton getInstance(){
        if(instance == null){
            synchronized(Singleton.class){
                if(instance == null)
                    instance = new Singleton();
            }
            
        }
        return instance;
    }
}
```



## Java内存模型

上面讨论了造成并发程序容易产生Bug的原因，也就是：**可见性、原子性、有序性得不到满足**；

Java中使用**Java内存模型来解决可见性和有序性问题**；

### Java内存模型

导致可见性的原因是缓存，导致有序性的原因是编译器指令重排，所以一种暴力解决可见性和有序性的方法就是，**禁用缓存和编译器优化**，但是这样会对性能带来极大的影响；

**所以这就需要我们在合适的地方禁用缓存和编译器优化，也就是所说的按需禁用缓存以及编译优化**，那么至于按需，就是由程序员来决定，Java只需要提供给程序员禁用缓存和编译优化的方法即可；

Java内存模型本质上就是实现了上面所说的，**提供给程序员禁用缓存和编译优化的方法**，具体来说就是：

- `volatile`关键字：禁用缓存；
- `synchronized`关键字：
- `final`关键字：
- `Happens-Before`规则：约束编译优化，使得编译器优化结果符合`Happens-Before`原则；

**正是上面几项技术保证了按需禁用缓存和编译优化，使得控制可见性和有序性变得可能**；

### volatile

`volatile`关键字的本意就是禁用CPU缓存，所以当我们声明一个变量为`volatile`时，就是告诉编译器，对于该变量的读写只能到内存中操作，不能使用CPU缓存；

对于`volatile`变量的写操作`Happens-Before`对于该变量的读操作；

### synchronized

`synchronized`关键字实现了加锁操作，能保证在同一时间，只有一个线程能访问`synchronized`修饰的方法或者代码块；

对于一个锁的解锁操作`Happens-Before`对于该锁的加锁操作；

### final

**`final`关键字修饰变量时，final修饰的实例字段则是涉及到新建对象的发布问题。当一个对象包含final修饰的实例字段时，其他线程能够看到已经初始化的final实例字段，这是安全的；**



### Happens-Before规则

`Happens-Before`规则并不是字面意思的前面一个操作时间上优先于后面一个操作，**而是指。前面一个操作的结果对后面的操作是可见的**；

比较正式的说法就是，`Happens-Before`规则**约束**了编译器优化，使得代码优化后符合`Happens-Before`规则；

下面介绍一下`Happens-Before`规则

#### 1.程序的顺序性规则

内容：**在同一个线程中，按照程序顺序，前面的操作Happens-Before后面的操作**；

比如，例1：

```java
class VolatileExample {
    int x = 0;
    volatile boolean v = false;
    public void writer() {
        x = 42;
        v = true;
    }
    public void reader() {
        if (v == true) {
            System.out.println(x);
        }
    }
    
    public static void main(String[] args){
        VolatileExample e = new VolatileExample();
        
        Thread t1 = new Thread(new Runnable(){
            public void run(){
                e.writer();
            }
        });
        
        Thread t2 = new Thread(new Runnable(){
            public void run(){
                e.reader();
            }
        });
        
        t1.start();
        t2.start();
        
    }
    
}
```

- 在这段代码中，`x=42;`在位置上先出现，`v=true;`在位置上后出现，所以一定有，**x=42 Happens-Before v=true**；

**这条规则保证了，在同一线程中，对于某个变量的修改对于后序程序是一定可见的，这样解释了为什么即使在单线程中且有指令优化的情况下也能使得程序正常运行**；



#### 2.volatile变量规则

内容：对于一个`volatile`变量的写操作，`Happens-Before`对该变量的读操作；



#### 3.传递性

内容：如果A操作`Happens-Before`B操作，B操作`Happens-Before`C操作，那么A操作`Happens-Before`C操作；

对于例1来说：

- 根据规则1：x=42; Happens-Before v=true;

- 根据规则2：v=true; Happens-Before v==true;

- 根据规则3：x=42; Happens-Before v==true;

  ![avatar](F:\找工作\Java基础\Java\images\传递性.png)

**即：在读变量v时，x为42这个操作是可见的**；

也就是说，对于例1来说，会输出结果：42



#### 4.管程中的锁规则

内容：**一个锁的解锁`Happens-Before`后续对该锁的加锁**；

**管程就是解决并发的一种通用手段，在Java中的实现就是`synchronized`关键字**；

管程中的锁在Java中是隐式实现的，在进入同步块之前，JVM会自动加锁，退出代码块时，JVM会自动解锁；

可以这么理解这个规则：**在某个锁解锁之后，会将强制所有共享变量的值从缓存刷新到内存中，这样之前的线程对于共享变量的操作对于后续的线程就是可见的**



#### 5.线程start()规则

内容：主线程A启动子线程B后，线程B能够看见线程A在启动线程B前的操作；

也就是说：**如果线程A调用线程B的start()方法，那么该start()操作`Happens-Before`线程B中任意操作**；

```java
Thread b = new Thread(()->{
    // 主线程调用 B.start() 之前，所有对共享变量的修改此处都可见
    // 即在此处var变量的值为77
});
var = 77;
b.start();
```

- 即：**线程B能看见所有在b.start()方法执行之前对于共享变量的修改**；



#### 6.线程join()规则

内容：主线程A等待子线程B完成（也就是主线程A通过调用子线程B的`join()`方法来实现），当线程B执行完成之后（也就是线程A中的`join()`方法返回了），线程A能够看到线程B对于共享变量的修改；

也就是说：**在线程A中调用线程B的`join()`方法并成功返回，那么线程B的所有操作`Happens-Before`该`join()`方法后的操作**

```java
Thread B = new Thread(()->{
  // 此处对共享变量 var 修改
  var = 66;
});
B.start();
// 如果线程A在此处访问var，那么线程B对于var的修改线程A时不一定能看到的
B.join()
// 子线程所有对共享变量的修改，在主线程调用 B.join() 之后皆可见
// 此例中，var的值为66

```

- 即：**在执行完B.join()方法之后，线程B对于共享变量的操作，在线程A中都能看到**



#### 7.线程中断规则

内容：对于线程的`interrupt()`方法调用`Happens-Before`被中断线程的代码检测到中断事件的发生；



#### 8.对象终结规则

内容：一个对象的初始化完成（构造函数执行结束）`Happens-Before`其`finalize()`方法的开始；



#### `Happens-Before`总结

**`Happens-Before`规则实际上是对可见性的一种约束，A `Happens-Before` B 意味着A事件对于B事件是可见的，无论A事件和B事件是否发生在同一线程里**；



### Java内存模型总结

- Java内存模型涉及的几个关键词：锁、volatile字段、final修饰符与对象的安全发布。其中：

  - 第一是锁，锁操作是具备happens-before关系的，解锁操作happens-before之后对同一把锁的加锁操作。实际上，**在解锁的时候，JVM需要强制刷新缓存，使得当前线程所修改的内存对其他线程可见**。
  - 第二是volatile字段，volatile字段可以看成是一种**不保证原子性的同步但保证可见性**的特性，其性能往往是优于锁操作的。但是，频繁地访问 volatile字段也会出现因为不断地强制刷新缓存而影响程序的性能的问题。
  - 第三是final修饰符，final修饰的实例字段则是涉及到新建对象的发布问题。当一个对象包含final修饰的实例字段时，其他线程能够看到已经初始化的final实例字段，这是安全的。

- Java内存模型底层怎么实现的？主要是通过**内存屏障(memory barrier)禁止重排序的**，即时编译器根据具体的底层体系架构，将这些内存屏障替换成具体的 CPU 指令。对于编译器而言，内存屏障将限制它所能做的重排序优化。而对于处理器而言，内存屏障将会导致缓存的刷新操作。比如，对于volatile，编译器将在volatile字段的读写操作前后各插入一些内存屏障。

  

## 互斥锁

之前讨论过，导致并发程序出现bug的原因主要有三个：

- 可见性；
- 原子性；
- 有序性；

**而Java内存模型解决了可见性和有序性问题，至于原子性问题，需要由互斥锁来解决**；

### 原子性问题的原因

原子性问题的原因是：**线程切换**，所以如果能禁止线程切换，那么原子性问题就能很好的解决了；

操作系统做线程切换是依赖于CPU中断的，所以禁止CPU中断，就能够禁止线程切换；

在单核CPU的时代，禁止CPU中断来禁止线程切换的方法是可行的，但是在多核CPU场景下，可能会出现，CPU-1运行线程1，CPU-2运行线程2，此时禁止CPU中断只能保证线程1、线程2能一直执行，**并不能保证同一时刻只有一个线程执行**；

**同一时刻只有一个线程执行，这就是互斥的意义，如果能保证互斥的话，原子性问题也就解决了**；



### 锁模型

实现互斥的一种重要方法就是锁，锁模型如下图所示：

![avatar](F:\找工作\Java基础\Java\images\互斥锁模型.png)

说明一下锁模型的各个部分：

- 首先，我们要确定好**受保护的资源R**，对R的操作都出现在临界区中，进入临界区前要加锁，退出临界区前要解锁；
- 其次，要给受保护的资源R创建一把锁LR；
- 最后，一定要确立锁LR和受保护资源R之间的关联关系；

### Synchronized

Java中的`synchronized`关键字就是锁的一种实现，`synchronized`关键字可以用来修饰方法和代码块，具体的使用如下所示：

```java
class Test{
    // 修饰非静态方法
    synchronized void test1(){
        // 临界区
        
    }
    
    // 修饰静态方法
    synchronized static void test2(){
        // 临界区
    }
    
    // 修饰代码块
    Object obj = new Object();
    void test3(){
        synchronized(obj){
            // 临界区
        }
    }
}
```

-  在`synchronized`关键字中，加锁和解锁的动作由JVM自动添加；
- `synchronized`关键字修饰静态方法时，**锁定当前类的class对象，在这个例子中就是Test.class；**
- `synchronized`关键字修饰非静态方法时，**锁定当前实例对象this；**



### 锁和受保护资源的关系







## 死锁





## 等待-通知机制





## 安全性、活跃性以及性能问题





## 管程





## Java线程





## 面向对象和并发编程



