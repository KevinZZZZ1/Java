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



#### Happens-Before总结

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

例：

```java
class SafeCalc {
    long value = 0L;
    long get(){
        return value;
    }
    synchronized void addOne(){
        value += 1;
    }
    
}
```

- 对于上面的例子来说，使用`get()`和`addOne()`方法会不会出现并发问题？

  - 分析：

    - 根据`Happens-Before`规则中的**管程锁规则**：对一个锁的解锁`Happens-Before`对该锁的加锁，再结合**程序的顺序性规则**（对于变量value的修改`Happens-Before`对管程的解锁），所以在`addOne()`函数中，之前的线程对于变量value的操作，之后的线程是一定可见的；
    - 但是对于`get()`函数来说，value的可见性是无法保证的，即一个线程调用`addOne()`函数之后，变量value的可见性是无法保证的；

  - 所以上面的例子，`addOne()`函数线程安全，`get()`函数线程不安全；

  - 可以通过在`get()`函数前也加上同步关键字，来使得它线程安全；

    ```java
    class SafeCalc {
        long value = 0L;
        synchronized long get(){
            return value;
        }
        synchronized void addOne(){
            value += 1;
        }
    }
    ```

    上面代码这种情况下，`get()`和`addOne()`函数要访问变量value，都要获得this这把锁，所以`get()`和`addOne()`是互斥的，也就是同一事件只能有一个方法访问变量value；

    ![avatar](F:\找工作\Java基础\Java\images\Synchronized可见性.png)

### 锁和受保护资源的关系

一般来说，**锁和受保护资源的关系是1：N的关系，也就是说一把锁可以保护多个资源**；

#### 多个锁保护一个资源带来的并发问题

**不能使用多个锁来保护同一个资源**，这样会造成并发问题，例：

```java
class SafeCalc {
  static long value = 0L;
  synchronized long get() {
    return value;
  }
  synchronized static void addOne() {
    value += 1;
  }
}
```

- 上面这段代码有两个锁，第一个是`get()`函数的this锁，第二个是`addOne()`函数的SafeCalc.class；而且这两个锁同时保护了变量value，这样就会造成问题：由于临界区 `get()` 和 `addOne()` 是用两个锁保护的，因此这两个临界区没有互斥关系，所以`addOne()`对于value的修改`get()`是不可见的；

![avatar](F:\找工作\Java基础\Java\images\两个锁保护同一个资源.png)

#### 保护没有关联的多个资源

**当我们要保护多个资源时，首先要区分这些资源之间是否存在关联，保护没有关联的多个资源可以为每个资源都设置一把锁**；

例如，在银行业务中有针对账号的余额取款的操作，也有对于账号密码的修改操作，但是账号余额和账号密码是两个不关联的资源，所以可以使用两把锁；

```java
class Account {
    // 锁：保护账户余额
    private final Object balLock = new Object();
    // 账户余额  
    private Integer balance;

    // 锁：保护账户密码
    private final Object pwLock = new Object();
    // 账户密码
    private String password;

    // 取款
    void withdraw(Integer amt) {
        synchronized(balLock) {
            if (this.balance > amt)
                this.balance -= amt;
        }
    } 

    // 查看余额
    Integer getBalance() {
        synchronized(balLock) {
            return balance;
        }
    }


    // 更改密码
    void updatePassword(String pw){
        synchronized(pwLock) {
            this.password = pw;
        }
    } 
    // 查看密码
    String getPassword() {
        synchronized(pwLock) {
            return password;
        }
    }

}

```

- 正如上面代码所描述的，使用对象`balLock`作为余额的锁，对象`pwLock`作为密码的锁，不同的资源使用不同的锁进行保护；
- 也可以使用一把锁来保护余额、密码这两个资源，但是这么做会导致取款、查看余额、修改密码、查看密码这四个操作是串行执行的，如果使用两把锁，那么取款和修改密码是可以并行的；
- **这种使用不同锁对受保护资源进行细化管理，能够提升性能，也叫做细化锁**；

#### 保护有关联关系的多个资源

例如，在银行业务里面的转账操作，账户A转给账户B100元，那么这两个账户就是有关联关系的，**这里有个重要的问题就是只能使用自己的锁锁住自己的资源，不能锁住其他人的资源**；

下面的代码就犯了这个错误：

```java
class Account {
    private int balance;
    // 转账
    synchronized void transfer(
        Account target, int amt){
        if (this.balance > amt) {
            this.balance -= amt;
            target.balance += amt;
        }
    }
    
}

```

- 这里的锁是this，this这个锁能够保护资源this.balance，但是不能保护别人的资源target.balance；
- 比如说，假设有 A、B、C 三个账户，余额都是 200 元，我们用两个线程分别执行两个转账操作：账户 A 转给账户 B 100 元，账户 B 转给账户 C  100元；
- 如果使用上面的代码会出现账户B余额最后可能是300，也可能是100，但绝对不可能是200；
- 原因是，假设线程1执行账户 A 转给账户 B 100 元，线程2执行账户 B 转给账户 C  100元，根据上面的代码，线程1和线程2是可能在不同CPU上同时执行的，因为线程1的锁是账户A对象，线程2的锁是账户B对象；这两个线程可以同时进入临界区，当这两个线程同时进入临界区时，都会读到账户 B 的余额为 200，导致最终账户 B 的余额可能是 300（线程 1 后于线程2 写 B.balance，线程 2 写的 B.balance 值被线程 1 覆盖），可能是100（线程 1 先于线程 2 写 B.balance，线程 1 写的 B.balance 值被线程 2 覆盖）；

上面代码的问题主要是，使用对象级别的锁，这样的锁并不能覆盖所有受保护的资源，如何正确使用锁来保护关联关系的资源？**就是让有关联关系的资源公用一把锁**；

```java
class Account {
    private int balance;
    // 转账
    void transfer(Account target, int amt){
        synchronized(Account.class) {
            if (this.balance > amt) {
                this.balance -= amt;
                target.balance += amt;
            }
        }
    } 
}

```

- 上面的代码使用过Account.class来保护不同对象的临界区，Account.class是JVM在Account类加载时创建的，它一定是唯一的；

![avatar](F:\找工作\Java基础\Java\images\关联关系资源的保护.png)

**如果资源之间没有关系，很好处理，每个资源一把锁就可以了。如果资源之间有关联关系，就要选择一个粒度更大的锁，这个锁应该能够覆盖所有相关的资源；**



## 死锁

之前说过了，导致并发程序容易出现bug的原因：可见性、原子性、有序性；

然后又介绍了解决上面问题的方法：使用JVM模型（`Happens-Before`规则、`volatile`修饰符）来解决可见性和有序性；使用互斥锁来解决原子性；

但是使用大粒度的互斥锁会导致操作是串行的，这样的执行效率很低，使用细粒度的锁能够使一些操作并发，但是容易造成死锁问题；

### 细粒度锁

对于之前那个转账的例子来说，使用细粒度锁是：转入账户一把锁，转出账户一把锁，在`transfer()`方法内部，首先先尝试锁定，转出账户this，然后再尝试锁定转入账户target，只有二者都成功时，才能进行转账；

```java
class Account {
    private int balance;
    
    void transfer(Account target, int amt) {
        // 锁定转出账户
        synchronized(this){
            // 锁定转入账户
            synchronized(target){
                if(this.balance>amt){
                    this.balance -= amt;
                    target.balance += amt;
                }
            }
        }
    }
    
}
```

- 上面的锁this，锁target就称为细粒度锁，**使用细粒度锁能提高并行度，从而优化性能**；
- 但是上面的方法并不是没有问题的，锁细化之后容易出现死锁问题；
- 比如有两个线程T1，T2，T1执行账户 A 转账户 B 的操作，账户 A.transfer(账户B)，T2执行账户 B 转账户 A 的操作，账户 B.transfer(账户A)；那么当T1和T2执行`synchronized(this)`时，T1会获得对象A的锁，T2会获得对象B的锁，接着执行`synchronized(target)`时，T1需要对象B的锁，T2需要对象A的锁，这是T1，T2就会进行无限期的等待；

### 死锁

死锁是指，**一组相互竞争资源的线程，因为等待资源而永久阻塞的现象**；

对于死锁的判断，**可以使用资源分配图，如果资源分配图有环存在，则会出现死锁**；

**导致死锁的原因有：**

- **互斥**：贡献资源X、Y只能被一个线程使用；
- **占用等待**：当一个线程占有资源X，在等待资源Y的过程中不会释放X；
- **不可抢占**：当一个线程占有资源X后，其他线程无法抢占；
- **循环等待**：线程T1等待线程T2的资源，线程T2等待线程T1的资源；

一旦造成死锁，大多数情况都是通过重启系统来解决，所以解决死锁最好的办法是预防死锁；



### 死锁预防

**只要破坏造成死锁原因4个中的1个就能预防死锁**，其中互斥是无法破坏的，因为我们用锁就是看上了锁互斥的特性，所以只能从剩下的3个条件入手；

#### 1.破坏占用等待条件

也就是说，**一次性申请所有需要的资源**，对于上面那个转账的例子来说，此时需要一个对象来同时申请所有资源和同时释放所有资源，具体代码如下：

```java
class Allocator {
    // 用来记录申请的资源
    private List<Object> als = new ArrayList<>();
    // 一次性申请所有资源
    synchronized boolean apply(Object from, Object to) {
        if(als.contains(from) || als.contains(to)){
            return false;  
        } else {
            als.add(from);
            als.add(to);  
        }
        return true;
    }
    // 归还资源
    synchronized void free(
        Object from, Object to){
        als.remove(from);
        als.remove(to);
    }
}

class Account {
    // actr 应该为单例
    private Allocator actr;
    private int balance;
    // 转账
    void transfer(Account target, int amt){
        // 一次性申请转出账户和转入账户，直到成功
        while(!actr.apply(this, target));
        try{
            // 锁定转出账户
            synchronized(this){              
                // 锁定转入账户
                synchronized(target){           
                    if (this.balance > amt){
                        this.balance -= amt;
                        target.balance += amt;
                    }
                }
            }
        } finally {
            actr.free(this, target)
        }
    } 
}
```







#### 2.破坏不可抢占条件

**线程能主动释放当前占有的资源**，`synchronized`关键字是无法做到这点的；

但是JDK中的java.util.concurrent包下的Lock是能实现破环不可抢占条件的；



#### 3.破坏循环等待条件

也就是说，**需要对资源进行排序，然后按顺序申请资源**，比如还是上面转账的例子，我们假设每个账户都有不同的属性 id，这个 id 可以作为排序字段，申请的时候，我们可以按照从小到大的顺序来申请；①~⑥处的代码对转出账户（this）和转入账户（target）排序，然后按照序号从小到大的顺序锁定账户；

```java
class Account {
    private int id;
    private int balance;
    // 转账
    void transfer(Account target, int amt){
    	Account left = this        ①
        Account right = target;    ②
        if (this.id > target.id) { ③
        	left = target;           ④
            right = this;            ⑤
         }                          ⑥
         // 锁定序号小的账户
         synchronized(left){
         	// 锁定序号大的账户
            synchronized(right){ 
            	if (this.balance > amt){
                	this.balance -= amt;
                    target.balance += amt;
                }
            }
        }
    } 
}
```



## 等待-通知机制

之前我们说过了引起并发编程产生众多bug的原因：可见性，有序性，原子性；以及解决这些问题使用的方法：Java内存模型（`Happens-Before`规则）解决可见性和有序性，互斥锁解决原子性；以及在使用互斥锁会碰到的问题：死锁，以及预防死锁的办法（破坏占有等待条件；破坏不可抢占条件；破坏循环等待条件）；

在使用破坏占有等待条件是指，**如果一个线程无法得到所有资源那么该线程会主动放弃自己已得到的资源进入等待状态**；那么这里就有一个问题，进入等待状态的线程是如何知道自己要的资源可用了呢？

- 使用一个`while`循环一直轮询等待，直到条件被满足；这种做法在轮询条件执行时间很多而且并发量不大的情况下是不错的，但是一旦并发量很大，这种循环方式对CPU的占用就很大了；
- **使用等待-通知机制**：如果线程发现条件不满足，则阻塞自己进入等待状态，当线程要求的条件得到满足时，通知线程重新执行；

**等待-通知机制是指，线程首先获得互斥锁，当线程要求的条件得不到满足时，线程会释放互斥锁并进入等待状态，当线程要求的条件得到满足时，通知等待的线程，重新获得互斥锁**；

### 用synchronized实现等待-通知机制

使用`synchronized`关键字配上`wait()`、`notify()`、`notifyAll()`三个方法就能实现等待-通知机制；

#### wait()

线程为了进入临界区，都要获得互斥锁，对应这个**互斥锁有一个等待队列**，当某个线程获得互斥锁进入临界区之后，发现自己要求的条件没有达成，**那么此时线程会调用互斥锁对象的wait()方法释放互斥锁并进入等待状态**；此时线程会进入**互斥锁的另一个等待队列**；具体如下图所示：

![avatar](F:\找工作\Java基础\Java\images\wait的原理图.png)

#### notify()、notifyAll()

当对象要求的条件得到满足时，调用互斥锁对象的`notify()`或`notifyAll()`方法就能通知**互斥锁等待队列（就是等待状态线程进入的队列）**中的线程条件已经满足；

此时，被通知的线程想要进入临界区的话，还是要重新获取互斥锁的，**重新获得互斥锁之后才能继续执行wait()之后的代码**；

![avatar](F:\找工作\Java基础\Java\images\notify原理图.png)

**需要注意的是**：

- 对于等待队列来说，它是属于互斥锁的，所以wait()、notify()、notifyAll()方法都是针对互斥锁来说的，具体的就是：
  - 如果synchronized锁住的是this对象，那么对应的是this.wait()、this.notify()、this.notifyAll()；
  - 如果synchronized锁住的是target对象，那么对应的是target.wait()、target.notify()、target.notifyAll()；
- 由于一定要先获得锁再使用等待-通知机制，所以一定是在synchronized同步块内调用wait()、notify()、notifyAll()方法；



#### 等待-通知机制的应用

下面我们使用等待-通知机制来实现一次性获得转入账户和转出账户：

```java
class Allocator{
    private List<Object> als;
    // 一次性申请所有资源
    synchronized void apply(Object from, Object to){
        while(als.contains(from) || als.contains(to)){
            try{
                wait();
            }catch(Exception e){
                
            }
        }
        als.add(from);
        als.add(to);
    }
    
    // 规划所有资源
    synchronized void free(Object from, Object to){
        als.remove(from);
        als.remove(to);
        notifyAll();
    }
    
}
```



## 安全性、活跃性以及性能问题

之前我们学习了导致并发编程bug多的原因：可见性，有序性，原子性；以及解决方法：Java内存模型，互斥锁；而且解决了在使用互斥锁时可能会出现的死锁情况（破坏占有等待，破坏不可抢占；破坏循环等待），以及在申请所有资源时遇到条件不满足的情况（使用等待-通知机制）；

现在来总结一下并发编程中要注意的问题：

### 安全性问题

也就是我们通常说的线程安全问题，线程安全的本质就是正确性，**也就是说程序能够按照期望执行**；不会产生任何意外；

**只要保证不会出现可见性问题、原子性问题、有序性问题，那么线程安全是一定能保证的**；

而且安全性问题发生有一个前提条件：**会出现多个线程同时访问或修改共享变量的情况**，如果这个前提条件不满足，那么线程一定是安全的，比如说线程本地存储(Thread Local Storage)；

总结来说，安全性问题可以分成两个方面：

- 数据竞争：多个线程同时对共享变量进行读写；
- 竞态条件：程序的执行解决依赖于线程的执行顺序，也就是程序执行的结果是不确定的，再次执行该程序可能会出现不一样的结果；

**数据竞争和竞态条件问题都可以使用互斥，也就是锁来解决；**

### 活跃性问题

活跃性问题可以分为：

- 死锁：两个或多个线程占有某个资源，等待其他资源的过程；

  解决方法：破坏等待占有条件；破坏不可抢占条件；破坏循环等待条件；

- 活锁：两个或多个进程同时放弃自己的资源，然后又重试竞争所需的资源；

  解决方法：随机等待一段时间；

- 饥饿：线程因无法访问所需资源而无法执行下去的情况；

  解决方法：保证资源充足；公平分配资源；避免持有锁的进程长时间执行；

### 性能问题

**由于锁其实就是将并行操作串行化**，如果串行化范围过大，多线程的性能优势就会收到影响，性能问题主要的解决办法就是：

- 使用无锁的算法和数据结构：比如线程本地存储，写入时复制，乐观锁等；
- 减少持有锁的时间：比如使用细粒度的锁；

性能的衡量标准有：

- 吞吐量：是指单位时间内能处理的请求数量；
- 延迟：是指发出请求到收到响应的时间；
- 并发量：是指能同时处理的请求量；

## 管程

之前我们讨论了关于并发编程的容易产生bug的原因：可见性，原子性，有序性；然后又说明了解决方法：Java内存模型（`Happens-Before`规则）解决可见性和有序性，互斥锁解决原子性；在使用互斥锁的时候可能会碰到死锁的情况，有3种方法可以预防死锁的发生：破坏等待占有条件，破坏不可抢占条件，破坏循环等待条件；对于使用破坏等待占有条件来预防死锁的应用来说，主要是通过一次性申请所有资源来实现的，那么做到这一点的呢？使用等待-通知机制就可以做到这一点，线程获得互斥锁之后发现要求的条件不满足，则会进入等待状态，当要求的条件被满足时，线程会收到一个通知，然后线程需要再次获得互斥锁继续执行后面的代码；之后我们总结了并发编程中的问题：安全性（线程执行的结果会符合预期，不会出现可见性、有序性、原子性的程序一定是线程安全的）、活跃性（死锁、活锁、饥饿）、性能（吞吐量、延迟、并发量）；

那么有没有一种核心技术能解决一切并发问题？实际上是有的，**管程技术可以解决一切并发问题**；

### 什么是管程

学习过操作系统课程都知道，在操作系统中，我们是使用信号量来解决并发问题的，而Java中实际使用的是管程技术，**管程和信号量是等价的，所谓的等价是指，可以用信号量来实现管程，也可以使用管程来实现信号量**；

管程(Monitor)是指，**管理共享变量，以及共享变量的访问操作方式，使它们支持并发**，在Java中管程的体现就是：`synchronized`关键字、`wait()`、`notify()`、`notifyAll()`；

### MESA模型

在管程的发展史上，出现过三种不同的管程模型：Hasen模型，Hoare模型、MESA模型；在Java中使用的使MESA模型，所以我们今天重点介绍MESA模型；

在并发编程领域，有两个核心问题：

- **互斥**：在同一时间内，只允许一个线程访问共享资源；
- **同步**：访问共享资源的线程如何进行协作、通信；

#### 互斥

管程（Monitor）解决互斥问题的方法很简单：**将共享资源以及对共享资源的操作封装起来**；

这个思想很符合面向对象的思想，这可能也是Java选择管程作为解决并发问题方法的原因吧；

![avatar](F:\找工作\Java基础\Java\images\管程解决互斥.png)

正如上图所示，管程X将共享资源`queue`以及相关的入队操作`enq()`、出队操作`deq()`都封装起来，访问共享资源操作的互斥性由管程来保证；

#### 同步

**管程中使用等待-同步机制来实现线程之间的同步**；共享资源及其访问操作是被封装起来的，在访问的入口处有个等待队列存放想访问共享资源的线程，管程中还有条件变量，当某个线程不满足条件变量时，会进入阻塞状态并释放当前锁，所以也会有一个对应的条件等待队列；

具体来说就是：

- 比如有线程T1想要获取共享资源，当时没有获得互斥锁，所以T1就会进入入口等待队列；
- 当T1获得互斥锁时，可以访问临界区，如果发现所要求的条件C1不能满足时，会阻塞并且释放互斥锁，然后加入该条件的条件等待队列；
- 当在某种情况下使得条件C1能够满足时，**此时会通知C1对应的条件等待队列中的线程，条件以满足**，然后这些线程再次竞争互斥锁，获得互斥锁的线程从上次阻塞处的代码继续执行；

我们现在使用一个阻塞队列来说明管程模型，该阻塞队列有两个操作，分别是入队操作，出队操作；

- 对于入队操作，如果队列已满，无法入队需要等到队列不满，使用notFull.await()；

- 对于出队操作，如果队列已空，无法出队需要等到队列不空，使用notEmpty()；

- 如果入队成功，说明队列不空，可以执行出队操作，使用notEmpty.signalAll()；

- 如果出队成功，说明队列不满，可以执行入队操作，使用notFull.signalAll()；

  ```java
  public class BlockedQueue<T>{
      final Lock lock = new ReentrantLock();
      // 条件变量：队列不满  
      final Condition notFull = lock.newCondition();
      // 条件变量：队列不空  
      final Condition notEmpty = lock.newCondition();
  
      // 入队
      void enq(T x) {
          lock.lock();
          try {
              while (队列已满){
                  // 等待队列不满 
                  notFull.await();
              }  
              // 省略入队操作...
              // 入队后, 通知可出队
              notEmpty.signal();
          }finally {
              lock.unlock();
          }
      }
      
      // 出队
      void deq(){
          lock.lock();
          try {
              while (队列已空){
                  // 等待队列不空
                  notEmpty.await();
              }
              // 省略出队操作...
              // 出队后，通知可入队
              notFull.signal();
          }finally {
              lock.unlock();
          }  
      }
  }
  ```

#### 管程模型

MESA模型如下图所示：

![avatar](F:\找工作\Java基础\Java\images\MESA模型.png)

### 管程中的wait()

对于MESA模型来说（也就是`synchronized`关键字来说），有一个编程范式，**就是要在一个while循环内调用wait()**，即：

```java
while(条件不满足){
    wait();
}
```

不同的模型的一个核心区别就是：当条件满足之后，如何通知相关线程；也就是说，当线程T2的操作使得线程T1等待的条件满足时，T1和T2哪个线程是可执行的？

- 在Hasen模型中，**要求notify()放在代码的最后**，这样T2通知完T1之后，T2就结束了，然后再执行T1；
- 在Hoare模型中，T2通知完T1之后，T2阻塞，T1马上执行，等T1执行完，再唤醒T2，执行T2剩下的动作；相比Hasen模型Hoare模型多了T2的阻塞唤醒动作；
- **在MESA模型中，T2通知完T1之后，T2继续执行，T1不会立即执行仅仅是从条件等待队列进入入口等待队列，这样会导致当T1执行的时候，很有可能条件变量就不满足了**；

**解释一下为什么要使用while循环**：原因是当线程被唤醒的时候，是从wait()方法后开始执行的，而且执行的时间点和唤醒的时间点基本上是不一致的，也就是说在线程执行的时间点条件变量可能是不满足的，**如果使用if循环判断，是不能起到再次验证的效果的**；



### 管程中的notify()、notifyAll()

在实际中一般都是使用`notifyAll()`方法唤醒所有线程，而`notify()`方法只是在下面条件满足时才使用：

- 所有等待线程拥有相同的等待条件；
- 所有等待线程被唤醒后，执行相同的操作；
- 只需要唤醒一个线程；



### Java管程模型

Java的管程和MESA管程有一点小小的不同，**Java管程只有一个条件变量以及条件等待队列**；

具体的如下图所示：

![avatar](F:\找工作\Java基础\Java\images\Java中的管程模型.png)



## Java线程



### Java线程的生命周期

在Java领域，实现并发程序的主要手段就是多线程，线程是一个操作系统的概念，Java的多线程会在操作系统的线程上进行一个封装；

**对于操作系统的以及Java线程的生命周期，需要掌握生命周期各个节点的状态转移机制就行了**；



#### 操作系统线程生命周期

在操作系统中，线程的生命周期可以分为：**初始状态，可运行状态，运行状态，休眠状态，终止状态**；具体如下图所示：

![avatar](F:\找工作\Java基础\Java\images\操作系统线程状态.png)

接下来详细介绍这五种状态：

- 初始状态：**是指编程语言将线程创建出来，此时在操作系统层面线程并未被创建，该状态属于编程语言特有的**；
- 可运行状态：**在该状态下，操作系统已经将线程创建出来了，等待分配CPU进行执行**；
- 运行状态：**操作系统将空闲CPU分配给线程执行**；
- 休眠状态：**运行的线程调用阻塞API（阻塞方式读文件）或等待某个事件（如条件变量），那么该线程就会进入休眠状态，同时释放CPU使用权，休眠状态下的线程只有在阻塞API返回或者某个事件发生才会重新进入可运行状态重新争夺CPU使用权**；
- 终止状态：**当线程执行完成或者线程出现异常终止时，就会进入终止状态，终止状态下的线程不能切换到其他任何状态**；

以上就是操作系统中线程的生命周期，在具体的编程语言中有着不同的实现，可能会对上面的生命周期做一些改变；

#### Java中线程的生命周期

在Java语言中，其生命周期包括：**初始化状态（NEW）、可运行／运行状态（RUNNABLE）、阻塞状态（BLOCKING）、等待状态（WAITING）、有时限等待（TIMED＿WAITING）、终止状态（TERMINATED）**；具体如下图所示：

![avatar](F:\找工作\Java基础\Java\images\Java线程状态.png)



其中Java把操作系统线程的休眠状态细分为：阻塞状态、等待状态、有时限等待状态；将操作系统中的可运行状态和运行状态合并成RUNNABLE状态；

接下来我们具体介绍一下上面的各个状态的转换：

- **RUNNABLE->BLOCKED**：

  只有一种情况能触发这个转换：**线程在等待`synchronized`关键字对应的隐式锁时会进入BLOCKED状态**；当线程获得`synchronized`关键字的隐式锁时会切换到RUNNABLE状态；

  **在BLOCKED状态下，线程是无法响应中断的，也就是说无法使用interrupted()进行中断**；

  在JVM层面，当线程调用阻塞式API时仍然保持RUNNABLE状态，**这是因为JVM把等待资源的线程都看成一种状态（RUNNABLE），无论是等待CPU使用权还是等待I/O，对于JVM来说都是一样的**；

  我们平时说调用阻塞式API使得线程进入阻塞状态指的是操作系统中的阻塞状态；

  

- **RUNNABLE->WAITING**：

  共有三种场景能做到这种转换：

  - **线程获得`synchronized`对应的隐式锁之后，调用锁的wait()方法**；
  - **线程A在执行时，调用了线程B的jion()方法**，此时线程A会进入WAITING状态，等待线程B执行完之后才会RUNNABLE状态；
  - **调用LockSupport.park()方法**，该方法会阻塞线程进入WAITING状态，当调用LockSupport.unpark(Thread thread)方法会唤醒目标线程，重新进入RUNNABLE状态；

  

- **RUNNABLE->TIMED_WAITING**

  有五种场景会触发这种转换：

  - **调用带超时参数的Thread.sleep(long millis)方法**；
  - 获得`synchronized`隐式锁之后，**调用带超时参数的Object.wait(long timeout)方法**；
  - **调用带超时参数的Thread.join(long millis)方法**；
  - **调用带超时参数的LockSupport.parkNanos(Object blocker, long deadline)方法**；
  - **调用带超时参数的LockSupport.parkUntil(long deadline)方法**；

  

- **NEW->RUNNABLE**

  **调用线程的start()方法就能进入RUNNABLE状态**；

  

- **RUNNABLE->TERMINATED**：

  当线程执行的run()方法执行完成之后，线程会自动切换到TERMINATED状态；

  如果在执行run()方法时异常抛出，会导致线程终止，也可以通过interrupt()方法中断处于WAITING、TIMED_WAITING状态的线程；

  线程如何得知自己被中断有两种方式：

  - **异常**：

    - 线程A处于WAITING、TIMED_WAITING状态时，调用线程A的interupt()方法可以使得线程进入进入RUNNABLE状态，同时线程A的代码会触发InterruptedException异常；
    - 线程A处于RUNNABLE状态，并且阻塞在I/O操作上时，其他线程条用线程A的interrupt()方法会触发I/O方法对于中断的处理；

  - **主动检测中断**：

    线程A处于RUNNABLE状态，如果其他线程调用线程A的interrupt()方法，那么线程A可以通过isInterrupted()方法检测自己是不是被中断了；

下面有一段有问题的代码，代码的原意是当当前线程被中断之后退出while(true)循环：

```java
Thread th = Thread.currentThread();
while(true){
    if(th.isInterrupted()){
        break;
    }
    // 省略业务代码
    
    try{
        Thread.sleep(100);
    }catch(InterruptedException e){
        e.printStackTrace();
    }
    
}
```

- 上面代码出现的主要问题是，**线程可能在sleep期间就被中断了，此时会抛出一个InterruptedException异常，并且中断标志会被自动清除掉**，所以应该在catch块中重置一下中断标示，修改后的代码如下：

  ```java
  Thread th = Thread.currentThread();
  while(true){
      if(th.isInterrupted()){
          break;
      }
      // 省略业务代码
      
      try{
          Thread.sleep(100);
      }catch(InterruptedException e){
          Thread.currentThread().interrupt();
          e.printStackTrace();
      }
      
  }
  ```

  

### 为什么需要多线程

多线程的本质上是要提升程序性能，而程序性能的衡量指标有：**延迟和吞吐量**；

- 延迟是指发出请求到收到响应的事件间隔，延迟越小说明性能越好；

- 吞吐量是指单位时间内能够处理请求的数量，吞吐量越大说明性能越好；

而提升性能的主要目的就是**降低延迟，提高吞吐量**，那么如何做到呢？主要有下面两种方法：

- 优化算法：
- 将硬件的性能发挥到极致；

第二个方法就和并发编程息息相关，**在并发编程领域，提升性能的本质就是提升硬件利用率，具体来说就是I/O的利用率和CPU的利用率，而使用多线程就能很好的提升I/O和CPU的综合利用率**；

对于设置多少线程合适，工程上有一个经验公式：

- 对于I/O密集型计算场景，最佳线程数是与程序中CPU计算和I/O操作的耗时比相关的：

  ```
  最佳线程数 = 1+(I/O耗时 / CPU耗时);
  ```

- 对于CPU密集型计算场景，最佳线程数是CPU核数+1：

  ```
  最佳线程数 = 1+CPU核数;
  ```

### 局部变量的线程安全性

对于局部变量的数据是否会出现数据竞争问题，其实我们可以直接给出答案就是：**局部变量是线程安全的**；

但至于为什么是线程安全的，这时我们在这里要讨论的问题：

#### 方法是如何被执行的

例如有以下代码：

```java
int a=7;
int[] b = fibonacci(a);
int[] c = b;
```

其执行过程如图所示：

![avatar](F:\找工作\Java基础\Java\images\方法的调用过程.png)

上面的图很好理解，但是有一个问题：**CPU是如何知道调用方法的参数以及返回地址的？**通过CPU的调用栈来实现的，每次调用一个方法时都会在调用栈中创建相应的栈帧，而且调用栈的生命周期和方法的生命周期是一样的，**栈帧中保存着参数、局部变量、返回地址，而对于每个线程都有自己独立的调用栈，这就是为什么局部变量是线程安全的原因**；

![avatar](F:\找工作\Java基础\Java\images\线程和调用栈的关系.png)

**从这里我们也能理解，为什么Java要设置线程独立的调用栈结构和线程共享的堆结构：如果你只想让变量只在方法内使用，那么创建局部变量创建在调用栈中是最好的方法了，而如果你是想让变量能跨越方法的使用，那么使用对象变量创建在堆中就能实现**；

栈中的变量仅仅是供本方法使用的，堆中的变量能够跨方法，跨线程的使用，也起到了线程通信的效果；

## 面向对象和并发编程

面向对象和并发编程其实是没有什么关系的，但是在Java语言中，将面向对象和并发编程融合在一起，而且面向对象思想能够让并发编程变得更简单；

### 封装共享变量

面向对象一个重要的思想就是封装，就是将类内部的属性和实现细节封装在对象内部，外界对象只能通过类实例提供的公共方法来简介访问这些内部属性；

并发编程领域，封装可以理解为将共享变量作为对象属性封装在内部，对所有公共方法制定并发访问策略；

实际中，并不是所有共享变量都会发生变化，对于那些不会发生变化的共享变量，建议使用final关键字修饰；

### 识别共享变量的约束条件

在实际的场景下，共享变量有着特定的要求，比如某商品的库存肯定要大于等于0，诸如此类的规则；



### 制定并发访问策略

制定并发访问策略有三点：

- 避免共享：利用线程本地存储以及为每个任务分配独立的线程；
- 不变模式：在Java领域使用很少；
- 管程及其其他同步工具：

