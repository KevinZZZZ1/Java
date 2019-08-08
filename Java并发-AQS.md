# AQS

## 1.简介

`AbstractQueuedSynchronizer  `抽象队列同步器，是由并发大师Doug Lea编写的，AQS是很多同步器的基础框架，比如`ReentrantLock`、`CountDownLatch`、`Semaphore`等都是基于AQS实现的；

**同步器与锁的比较：**

- 同步器：主要面向锁的开发人员，简化了锁的实现方式，无需处理同步状态管理、线程排队、线程等待和唤醒；
- 锁：主要面向使用者，隐藏了锁具体的实现细节；

我们还可以基于AQS编写自定义的同步器：自定义同步器使用一个内部类继承AQS实现同步功能，AQS注释中给出了一个自定义同步器的例子，如下所示：

```java
class Mutex implements Lock, java.io.Serializable {

   // Our internal helper class 内部类继承AQS
   private static class Sync extends AbstractQueuedSynchronizer {
     // Reports whether in locked state
     protected boolean isHeldExclusively() {
       return getState() == 1;
     }

     // Acquires the lock if state is zero 只需要重写tryAcquire()和tryRelease()以及isHeldExclusively()方法
     public boolean tryAcquire(int acquires) {
       assert acquires == 1; // Otherwise unused
       if (compareAndSetState(0, 1)) {
         setExclusiveOwnerThread(Thread.currentThread());
         return true;
       }
       return false;
     }

     // Releases the lock by setting state to zero
     protected boolean tryRelease(int releases) {
       assert releases == 1; // Otherwise unused
       if (getState() == 0) throw new IllegalMonitorStateException();
       setExclusiveOwnerThread(null);
       setState(0);
       return true;
     }

     // Provides a Condition
     Condition newCondition() { return new ConditionObject(); }

     // Deserializes properly
     private void readObject(ObjectInputStream s)
         throws IOException, ClassNotFoundException {
       s.defaultReadObject();
       setState(0); // reset to unlocked state
     }
   }

   // The sync object does all the hard work. We just forward to it.
   private final Sync sync = new Sync();

   public void lock()                { sync.acquire(1); }
   public boolean tryLock()          { return sync.tryAcquire(1); }
   public void unlock()              { sync.release(1); }
   public Condition newCondition()   { return sync.newCondition(); }
   public boolean isLocked()         { return sync.isHeldExclusively(); }
   public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
   public void lockInterruptibly() throws InterruptedException {
     sync.acquireInterruptibly(1);
   }
   public boolean tryLock(long timeout, TimeUnit unit)
       throws InterruptedException {
     return sync.tryAcquireNanos(1, unit.toNanos(timeout));
   }
 }
```

## 2.原理概述

在AQS内部，维护一个阻塞队列来保存获取互斥锁失败的线程，在公平竞争的情况下，无法获取互斥锁的线程会被封装成队列的一个节点放入队列尾部，该线程并不是马上阻塞而是先自旋尝试获取互斥锁，如果获取失败才会进入阻塞状态；

![avatar](D:\img\AQS阻塞队列.jpg)



## 3.重要方法介绍

AQS中的方法可以分为3组：

- 访问/设置同步状态：

  |                        方法                        |        说明         |
  | :------------------------------------------------: | :-----------------: |
  |                   int getState()                   |    获取同步状态     |
  |                  void setState()                   |    设置同步状态     |
  | boolean compareAndSetState(int expect, int update) | 通过CAS设置同步状态 |

- 由自定义同步器重写的方法：

  |               方法                |            说明            |
  | :-------------------------------: | :------------------------: |
  |    boolean tryAcquire(int arg)    |     独占式获取同步状态     |
  |    boolean tryRelease(int arg)    |     独占式释放同步状态     |
  |   int tryAcquireShared(int arg)   |     共享式获取同步状态     |
  | boolean tryReleaseShared(int arg) |     共享式释放同步状态     |
  |    boolean isHeldExclusively()    | 检测当前线程是否获取独占锁 |

- 模板方法，自定义同步器可以直接调用：

  |                       方法                        |                             说明                             |
  | :-----------------------------------------------: | :----------------------------------------------------------: |
  |               void acquire(int arg)               | 独占式获取同步状态，该方法将会调用tryAcquire尝试获取同步状态； |
  |        void  acquireInterruptibly(int arg)        |                      响应中断的acquire                       |
  |   boolean tryAcquireNanos(int arg, long nanos)    |                    超时+响应中断的acquire                    |
  |           void  acquireShared(int arg)            |     共享式获取同步状态，同一时刻会有多个线程获得同步状态     |
  |     void acquireSharedInterruptibly(int arg)      |                   响应中断的acquireShared                    |
  | boolean tryAcquireSharedNanos(int arg,long nanos) |                 超时+响应中断的acquireShared                 |
  |             boolean release(int arg)              |                      独占式释放同步状态                      |
  |          boolean releaseShared(int arg)           |                      共享式释放同步状态                      |

  



## 4.源码分析

### 4.1同步状态

AQS中维护了一个state变量，由volatile修饰，表示同步状态：

```java
private volatile int state;
```

第三节中所说的设置/获取同步状态，指的就是这个变量；我们可以设置state变量的值来定义同步器所在的模式：

- 独占模式：同一时刻只能有一个线程持有锁；

  在独占模式下，我们可以把state设置为0；每次需要获取锁时都要先判断state的值，如果state不为0的话，说明已经有线程持有该锁，当前线程需要阻塞等待，如果state为0，该线程将state的值置为1；**这个过程就是获取同步状态的过程**；

- 共享模式：同一时刻可以有多个线程持有锁；

  在共享模式下，我们可以将state设置为10，意思是允许有10个线程同时持有该锁，每次线程尝试获取同步状态的时候都会先判断state值是否大于0，如果state小于0说明已经有10个线程同时持有该锁了；



### 4.2节点结构

在并发的情况下，AQS会将未获取到同步状态的线程封装成节点，并且置于队列的尾部；

```java
static final class Node {

    /** 共享类型节点，标记节点在共享模式下等待 */
    static final Node SHARED = new Node();
    
    /** 独占类型节点，标记节点在独占模式下等待 */
    static final Node EXCLUSIVE = null;

    /** 等待状态 - 取消 */
    static final int CANCELLED =  1;
    
    /** 
     * 等待状态 - 通知。某个节点是处于该状态，当该节点释放同步状态后，
     * 会通知后继节点线程，使之可以恢复运行 
     */
    static final int SIGNAL    = -1;
    
    /** 等待状态 - 条件等待。表明节点等待在 Condition 上 */
    static final int CONDITION = -2;
    
    /**
     * 等待状态 - 传播。表示无条件向后传播唤醒动作，详细分析请看第五章
     */
    static final int PROPAGATE = -3;

    /**
     * 等待状态，取值如下：
     *   SIGNAL,
     *   CANCELLED,
     *   CONDITION,
     *   PROPAGATE,
     *   0
     * 
     * 初始情况下，waitStatus = 0
     */
    volatile int waitStatus;

    // 前驱节点
    volatile Node prev;

    // 后继节点
    volatile Node next;

    // 节点对应线程
    volatile Thread thread;

    // 下一个等待节点，用在ConditionObject中
    Node nextWaiter;

    // 判断节点是否为共享节点
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    // 获取当前节点的前驱
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    /** addWaiter 方法会调用该构造方法 */
    Node(Thread thread, Node mode) {
        this.nextWaiter = mode;
        this.thread = thread;
    }

    /** Condition 中会用到此构造方法 */
    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

Node节点可以形式化为：

![avatar](D:\img\AQS中的Node节点.jpg)



AQS中定义了同步队列头结点引用和尾节点引用：

```java
private transient volatile Node head;
private transient volatile Node tail;
```

Node类中的`waitStatus`变量表示的是等待状态，具体的如下图所示：

|    静态变量    |  值  |                       描述                       |
| :------------: | :--: | :----------------------------------------------: |
| Node.CANCELLED |  1   |            节点对应的线程已经被取消了            |
|  Node.SIGNAL   |  -1  |        表示后面节点对应的线程处于等待状态        |
| Node.CONDITION |  -2  |               表示节点在等待队列中               |
| Node.PROPAGATE |  -3  | 表示下一次共享式同步状态的获取将被无条件传播下去 |
|       无       |  0   |                     初始状态                     |



### 4.3独占模式分析



#### 4.3.1获取同步状态

```java
/**
 * 该方法将会调用子类复写的 tryAcquire 方法获取同步状态，
 * - 获取成功：直接返回
 * - 获取失败：将线程封装在节点中，并将节点置于同步队列尾部，
 *     通过自旋尝试获取同步状态。如果在有限次内仍无法获取同步状态，
 *     该线程将会被 LockSupport.park 方法阻塞住，直到被前驱节点唤醒
 */
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

/** 向同步队列尾部添加一个节点 */
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // 尝试以快速方式将节点添加到队列尾部
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    
    // 快速插入节点失败，调用 enq 方法，不停的尝试插入节点
    enq(node);
    return node;
}

/**
 * 通过 CAS + 自旋的方式插入节点到队尾
 */
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            // 设置头结点，初始情况下，头结点是一个空节点
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            /*
             * 将节点插入队列尾部。这里是先将新节点的前驱设为尾节点，之后在尝试将新节点设为尾节
             * 点，最后再将原尾节点的后继节点指向新的尾节点。除了这种方式，我们还先设置尾节点，
             * 之后再设置前驱和后继，即：
             * 
             *    if (compareAndSetTail(t, node)) {
             *        node.prev = t;
             *        t.next = node;
             *    }
             *    
             * 但但如果是这样做，会导致一个问题，即短时内，队列结构会遭到破坏。考虑这种情况，
             * 某个线程在调用 compareAndSetTail(t, node)成功后，该线程被 CPU 切换了。此时
             * 设置前驱和后继的代码还没带的及执行，但尾节点指针却设置成功，导致队列结构短时内会
             * 出现如下情况：
             *
             *      +------+  prev +-----+       +-----+
             * head |      | <---- |     |       |     |  tail
             *      |      | ----> |     |       |     |
             *      +------+ next  +-----+       +-----+
             *
             * tail 节点完全脱离了队列，这样导致一些队列遍历代码出错。如果先设置
             * 前驱，在设置尾节点。及时线程被切换，队列结构短时可能如下：
             *
             *      +------+  prev +-----+ prev  +-----+
             * head |      | <---- |     | <---- |     |  tail
             *      |      | ----> |     |       |     |
             *      +------+ next  +-----+       +-----+
             *      
             * 这样并不会影响从后向前遍历，不会导致遍历逻辑出错。
             * 
             * 参考：
             *    https://www.cnblogs.com/micrari/p/6937995.html
             */
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

/**
 * 同步队列中的线程在此方法中以循环尝试获取同步状态，在有限次的尝试后，
 * 若仍未获取锁，线程将会被阻塞，直至被前驱节点的线程唤醒。
 */
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 循环获取同步状态
        for (;;) {
            final Node p = node.predecessor();
            /*
             * 前驱节点如果是头结点，表明前驱节点已经获取了同步状态。前驱节点释放同步状态后，
             * 在不出异常的情况下， tryAcquire(arg) 应返回 true。此时节点就成功获取了同
             * 步状态，并将自己设为头节点，原头节点出队。
             */ 
            if (p == head && tryAcquire(arg)) {
                // 成功获取同步状态，设置自己为头节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            
            /*
             * 如果获取同步状态失败，则根据条件判断是否应该阻塞自己。
             * 如果不阻塞，CPU 就会处于忙等状态，这样会浪费 CPU 资源
             */
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        /*
         * 如果在获取同步状态中出现异常，failed = true，cancelAcquire 方法会被执行。
         * tryAcquire 需同步组件开发者覆写，难免不了会出现异常。
         */
        if (failed)
            cancelAcquire(node);
    }
}

/** 设置头节点 */
private void setHead(Node node) {
    // 仅有一个线程可以成功获取同步状态，所以这里不需要进行同步控制
    head = node;
    node.thread = null;
    node.prev = null;
}

/**
 * 该方法主要用途是，当线程在获取同步状态失败时，根据前驱节点的等待状态，决定后续的动作。比如前驱
 * 节点等待状态为 SIGNAL，表明当前节点线程应该被阻塞住了。不能老是尝试，避免 CPU 忙等。
 *    —————————————————————————————————————————————————————————————————
 *    | 前驱节点等待状态 |                   相应动作                     |
 *    —————————————————————————————————————————————————————————————————
 *    | SIGNAL         | 阻塞                                          |
 *    | CANCELLED      | 向前遍历, 移除前面所有为该状态的节点               |
 *    | waitStatus < 0 | 将前驱节点状态设为 SIGNAL, 并再次尝试获取同步状态   |
 *    —————————————————————————————————————————————————————————————————
 */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    /* 
     * 前驱节点等待状态为 SIGNAL，表示当前线程应该被阻塞。
     * 线程阻塞后，会在前驱节点释放同步状态后被前驱节点线程唤醒
     */
    if (ws == Node.SIGNAL)
        return true;
        
    /*
     * 前驱节点等待状态为 CANCELLED，则以前驱节点为起点向前遍历，
     * 移除其他等待状态为 CANCELLED 的节点。
     */ 
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * 等待状态为 0 或 PROPAGATE，设置前驱节点等待状态为 SIGNAL，
         * 并再次尝试获取同步状态。
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}

private final boolean parkAndCheckInterrupt() {
    // 调用 LockSupport.park 阻塞自己
    LockSupport.park(this);
    return Thread.interrupted();
}
```

独占式获取同步状态的大致流程做个总结，如下：

1. 调用 tryAcquire 方法尝试获取同步状态
2. 获取成功，直接返回
3. 获取失败，将线程封装到节点中，并将节点入队
4. 入队节点在 acquireQueued 方法中自旋获取同步状态
5. 若节点的前驱节点是头节点，则再次调用 tryAcquire 尝试获取同步状态
6. 获取成功，当前节点将自己设为头节点并返回
7. 获取失败，可能再次尝试，也可能会被阻塞。这里简单认为会被阻塞。

接下来，我们仔细分析一下上面的代码，从`acquire()`方法开始：

```java
public final void acquire(int arg) {
    if(!tryAcquire(arg) && aquireQueued(addWriter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

上面的代码说明，只有在获取同步状态失败的情况下，才会将当前线程封装成Node类，这个操作主要由`addWriter(Node.EXCLUSIVE)`方法实现：

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);  //构造一个新节点
    Node pred = tail;
    if (pred != null) { //尾节点不为空，插入到队列最后
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {       //更新tail，并且把新节点插入到列表最后
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}

private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) {    //tail节点为空，初始化队列
            if (compareAndSetHead(new Node()))  //设置head节点
                tail = head;
        } else {    //tail节点不为空，开始真正插入节点
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

`addWriter(Node.EXCLUSIVE)`方法中会**先尝试使用CAS将当前节点快速入队**，如果快速入队失败，则调用`enq(Node)`方法入队；

`enq(Node)`方法中需要特别注意的是：**当tail节点为空时，表示同步队列还为空，使用new Node()创建一个节点来初始化同步队列**；当前tail节点不为空时，再把当前线程封装成的节点加入队尾；如下图所示：

![avatar](D:\img\AQS同步队列加入节点.jpg)

图中0号节点就是new Node()创建的新节点，1号节点就是当前线程t1对应的节点；

生成完节点之后，接着会调用`acquireQueued (Node, int)`方法：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();  //获取前一个节点
            if (p == head && tryAcquire(arg)) { // 前一个节点是头节点再次尝试获取同步状态
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

**如果新插入节点的前一个节点是头结点的话，会再次调用tryAcquire()方法尝试获取同步状态**，这么做的原因是：怕得到同步状态的头结点很快释放同步状态，这样当前线程又需要一次状态切换，所以当前线程在阻塞之前再尝试一次获得同步状态；如果获得成功那么就使用`setHead(Node)`方法将自己设置为头节点；

```java
private void setHead(Node node){
    head = node;
    node.thread = null;
    node.prev = null;
}
```

同时把Node节点的thread设置为null，变成0号节点；

如果尝试获取同步状态失败，就需要执行`shouldParkAfterFailedAcquire `方法，主要是对`waitStatus`的各种操作，判断当前节点是否需要阻塞：

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;   //前一个节点的状态
    if (ws == Node.SIGNAL)  //Node.SIGNAL的值是-1
        return true;
    if (ws > 0) {   //当前线程已被取消操作，把处于取消状态的节点都移除掉
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {    //设置前一个节点的状态为-1
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

从源码中我们可以知道，`shouldParkAfterFailedAcquire `是根据当前节点的前一个节点的`waitStatus`的来判断当前节点应该处于的状态：

- 前一个节点的`waitStatus`为-1（Node.SIGNAL）时，直接返回true，表示当前节点需要阻塞；
- 前一个节点的`waitStatus`大于0（Node.CANCELLED）时，先将当前节点前面的所有CANCELLED状态的节点移除，再返回false;
- 前一个节点的`waitStatus`为0（初始状态）或者-3（Node.PROPAGATE），将当前节点的前一个节点的`waitStatus`设置为-1（Node.SIGNAL），再返回false；

对于初始状态如下图的同步队列来说：

![avatar](D:\img\AQS同步队列加入节点.jpg)

在一开始，所有的`Node`节点的`waitStatus`都是`0`，所以在第一次调用`shouldParkAfterFailedAcquire`方法时，当前节点的前一个节点，也就是`0号节点`的`waitStatus`会被设置成`Node.SIGNAL`立即返回`false`，这个状态的意思就是说`0号节点`后边的节点都处于等待状态；

![avatar](D:\img\AQS同步队列加入节点1.jpg)

由于`acquireQueued() `是在一个循环中在调用`shouldParkAfterFailedAcquire() `的，所以第二次调用的时候，0号节点的`waitStatus`已经变成-1，所以会返回true；

接下来`acquireQueued(Node, int)`继续调用`parkAndCheckInterrupt ()`方法：

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}

public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(false, 0L);     //调用底层方法阻塞线程
    setBlocker(t, null);
}
```

此时，线程会被阻塞；

整个独占获取同步模式的流程图如下所示：

![avatar](D:\img\独占模式获取同步状态.jpg)

#### 4.3.2释放同步状态

当持有同步状态的线程执行完操作之后，需要释放同步状态；这个过程需要调用`release(int arg)`方法：

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;   //节点的等待状态
    if (ws < 0) 
        compareAndSetWaitStatus(node, ws, 0);
    Node s = node.next; 
    if (s == null || s.waitStatus > 0) {    //如果node为最后一个节点或者node的后继节点被取消了
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)   
            if (t.waitStatus <= 0)  //找到离头节点最近的waitStatus为负数的节点
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);   //唤醒该节点对应的线程
}

```

释放同步状态的操作相比于获取同步状态操作很简单，结合上面的注释都能看懂；

#### 4.3.3独占式同步工具示例

```java
public class PlainLock implements Lock {

    private Sync sync = new Sync();

    @Override
    public void lock() {
        sync.tryAcquire(1);
    }

    @Override
    public void unlock() {
        sync.release(1);
    }
    
    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int arg) {
            return compareAndSetState(0, 1);
        }

        @Override
        protected boolean tryRelease(int arg) {
            setState(0);
            return true;
        }

        @Override
        protected boolean isHeldExclusively() {
            return getState()==1;
        }
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        
    }

    @Override
    public boolean tryLock() {
        return false;
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return false;
    }

    @Override
    public Condition newCondition() {
        return null;
    }
}
```

```java
public class Test {

    private int i;

    private PlainLock lock = new PlainLock();

    private void increase(){
        lock.lock();
        i++;
        lock.unlock();
    }
    private int get(){
        return i;
    }

    public static void test(int numThreads, int loopTimes) {
        Test test = new Test();
        Thread[] threads = new Thread[numThreads];
        for (int j = 0; j < numThreads; j++) {
            Thread t = new Thread(new Runnable() {
                @Override
                public void run() {
                    test.increase();
                }
            });
            threads[j] = t;
            t.start();
        }
        for (Thread t: threads){
            try {
                t.join();
            }catch (InterruptedException e){
                e.printStackTrace();
            }
        }
        System.out.println(numThreads + "个线程，循环" + loopTimes + "次结果：" + test.get());
    }


    public static void main(String[] args) {
        test(20, 1);
        test(20, 10);
        test(20, 100);
        test(20, 1000);
        test(20, 10000);
        test(20, 100000);
        test(20, 1000000);
        test(30, 1);
        test(30, 10);
        test(30, 100);
        test(30, 1000);
        test(30, 10000);
        test(30, 100000);
        test(30, 1000000);

        System.out.println("main thread 结束");
    }

}
```

```
运行结果为：
20个线程，循环1次结果：20
20个线程，循环10次结果：20
20个线程，循环100次结果：20
20个线程，循环1000次结果：20
20个线程，循环10000次结果：20
20个线程，循环100000次结果：20
20个线程，循环1000000次结果：20
30个线程，循环1次结果：30
30个线程，循环10次结果：30
30个线程，循环100次结果：30
30个线程，循环1000次结果：30
30个线程，循环10000次结果：30
30个线程，循环100000次结果：30
30个线程，循环1000000次结果：30
main thread 结束
```



### 4.4共享模式分析



#### 4.4.1获取同步状态





#### 4.4.2释放共享状态





## 5.PROPAGATE状态的意义



## 6.Condition实现原理





### 6.1实现原理





### 6.2源码解析



#### 6.2.1等待



#### 6.2.2通知







### 6.3其他



