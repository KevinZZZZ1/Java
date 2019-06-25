# Java Object类的各个方法

[TOC]

先上个Object类源码，大体上认识一下：

```java
public class Object{
    public final native Class<?> getClass();

    public native int hashCode();

    public boolean equals(Object obj) {
        return (this == obj);
    }

    protected native Object clone() throws CloneNotSupportedException;

    public String toString() {
        return getClass().getName() + "@" + Integer.toHexString(hashCode());
    }

    public final native void notify();

    public final native void notifyAll();

    public final native void wait(long timeout) throws InterruptedException;

    public final void wait(long timeout, int nanos) throws InterruptedException {
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos > 0) {
            timeout++;
        }

        wait(timeout);
    }

    public final void wait() throws InterruptedException {
        wait(0);
    }

    protected void finalize() throws Throwable { }

}
```

## 一、getClass方法

```java
public final native Class<?> getClass();
```

- 这是一个`native`方法，而且是`final`的，说明这个方法不能再子类重写；

- `getClass()`方法返回的是当前实例对应的`Class`类，也就是说无论该类有多少实例，每个实例`getClass()`返回的`Class`对象是一样的；

  - ```java
    Integer i = new Integer(1);
    Class iClass = i.getClass();
    
    Integer j = new Integer(2);
    Class jClass = j.getClass();
    
    System.out.println(iClass == jClass);
    ```

  - ```
    运行结果为true，说明两个Integer实例的getClass方法返回的Class对象是同一个；
    ```

- 还可以通过使用`类.class`的方法获得`Class`对象，即：`Integer.class`:

  - ```java
    Integer num = new Integer(1);
    Class numClass = num.getClass();
    Class integerClass = Integer.class;
    
    System.out.println(numClass == integerClass)
    ```

  - ```
    运行结果为true
    ```

- 但是`Integer.class`和`int.class`的`Class`对象是不同的，`int.class`返回的是原生`int`的`Class`：

  - ```java
    Class intClass = int.class;
    Class intType = Integer.TYPE;
    Class integerClass = Integer.class;
    
    System.out.println(intClass == integerClass);
    System.out.println(intClass == intType);
    ```

  - ```java
    运行结果为false,true，这是因为Integer.class和int.class返回的不是一个对象，而int.class返回的和Integer.TYPE是同一个对象。
    Integer.TYPE定义如下：
    public static final Class<Integer> TYPE = (Class<Integer>) Class.getPrimitiveClass("int");
    ```

  - Java中为原生类型(boolean,byte,char,short,int,long,float,double)和void都创建了一个预先定义好的类型，可以通过包装类的TYPE静态属性获取;





## 二、hashCode方法

```java
public native int hashCode();
```

- `hashCode()`方法也是一个`native`方法，**该方法的主要用在HashMap、HashSet等使用到对象Hash值的地方**；
- `Java`对于对象的`hashCode`方法有以下规定：
  - 在`Java`程序运行过程中，只要对象没有发生改变，对象的`hashCode`一定不会变化；
  - 如果两个对象使用`equals`方法比较结果为`true`的话，这两个对象的`hashCode`值必须相等；
  - 如果两个对象使用`equals`方法比较结果为`false`的话，这两个对象的`hashCode`值不一定要不相等，最好是不相等从而提高`hash`函数的性能；

- 在实际工程中，经常会重写`hashCode`方法，可能是因为`equals`方法也被重写。至于重写`hashCode`方法，一般的写法如下：

  ```java
  1.定义一个初始值为17==》int result=17;
  2.分别解析自定义类中各个属性：
  	2.1出现boolean类型数据f：result = 31*result + (f?1:0);
  	2.2出现byte/char/short/int类型数据f：result = 31*result + (int)f;
  	2.3出现long类型数据f：result = 31*result + (int)(f^(f>>>32));
  	2.4出现float类型数据f：result = 31*result + Float.floatToIntBits(f);
  	2.5出现double类型数据f：long m = Double.doubleToLongBits(f);result = 31*result + (int)m;
  	2.6出现数据：调用java.util.Arrays.hashCode方法计算；
  3.最后返回result;
  ```



## 三、equals方法

```java
public boolean equals(Object obj) {
        return (this == obj);
    }
```

- `equals`方法用于判断两个对象是否相等，`Object`中的`equals`默认是**判断两个对象是否拥有相同地址**，也就是==对应的内存语义；
- 在`Java`中，==对于基本数据类型来说，就表示值是否相等，对于引用类型来说，表示内存地址是否相等；
- 在实际工作中，要根据需求来重写`equals`方法，重写时要注意以下几个性质：
  - `reflexive`：自反性，即：非空引用值x，`x.equals(x)`必须返回`true`;
  - `symmetric`：对称性，即：非空引用值x，y，`x.equals(y)`返回`true`，则`y.equals(x)`也返回`true`;
  - `transitive`：传递性，即：非空引用值x，y，z，`x.equals(y)`和`y.equals(z)`，则有`x.equals(z)`;
  - `consistent`：一致性，即：非空引用值x，y，多次调用`x.equals(y)`的返回值一定是确定的;
  - 对于任何非空x，`x.equals(null)`都要返回`false`；

- **重写了equals方法之后，一定要重写hashCode方法，不然会导致某两个对象的equals方法比较结果为true，而hashCode值不相等的情况**；

## 四、clone方法

```java
protected native Object clone() throws CloneNotSupportedException;
```

- 用于克隆一个对象，被克隆的对象要实现`Cloneable`接口，否则对象调用`clone()`方法会抛出`CloneNotSupportedException`异常；
- 一般克隆的对象满足以下三个规则：
  - `x.clone()!=x`：
  - `x.clone().getClass()==x.getClass()`：
  - `x.clone().equals(x)`：
- **调用clone()方法时，基本数据类型和引用类型克隆的原理不同：基本数据类型是直接复制；引用类型是将对象引用复制过去，不会对对象进行克隆，即浅拷贝，如果要实现深拷贝还需要重写clone()方法**；



## 五、toString方法

```java
public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
}
```

`Object`中的默认`toString()`方法如上，一般在需要的情况下，都会重写该方法；



## 六、notify方法

```java
public final native void notify();
```

- `notify()`方法也是一个`final`类型的`native`方法，不允许子类重写；

- `notify()`方法用于唤醒正在等待当前对象监视器的线程，但是唤醒机制是随机的；

- 一般`notify`方法和`wait`方法配合使用来达到多线程同步的目的；

- 一个线程在调用一个对象的`notify`方法之前必须获取到该对象的监视器(`synchronized`)，否则将抛出`IllegalMonitorStateException`异常。同样一个线程在调用一个对象的`wait`方法之前也必须获取到该对象的监视器。

- ```java
  public class TestNotify{
      
      public static void main(String[] args){
          
          Object lock = new Object();
          Thread t1 = new Thread(() -> {
              synchronized (lock) {
                  System.out.println(Thread.currentThread().getName() + " is going to wait on lock's monitor");
                  try {
                      lock.wait();
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
                  System.out.println(Thread.currentThread().getName() + " relinquishes the lock's monitor");
              }
          });
          Thread t2 = new Thread(() -> {
              synchronized (lock) {
                  System.out.println(Thread.currentThread().getName() + " is going to notify a thread that waits on lock's monitor");
                  lock.notify();
              }
          });
          t1.start();
          t2.start();
  
          
      }
      
  }
  ```

- ```
  运行结果如下：
  Thread-0 is going to wait on lock's monitor
  Thread-1 is going to notify a thread that waits on lock's monitor
  Thread-0 relinquishes the lock's monitor
  ```

## 七、notifyAll方法

```java
public final native void notifyAll();
```

- `notifyAll`方法用于唤醒所有等待对象监视器锁的进程，而`notify`方法只能唤醒所有等待线程中的一个；
- 如果当前线程没有持有对象监视器，那么调用`notifyAll`也会发生`IllegalMonitorStateException`;



## 八、wait方法

```java
public final native void wait(long timeout) throws InterruptedException;

    public final void wait(long timeout, int nanos) throws InterruptedException {
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos > 0) {
            timeout++;
        }

        wait(timeout);
    }

    public final void wait() throws InterruptedException {
        wait(0);
    }
```

- 一个线程调用一个对象的`wait()`方法后，线程将进入`WAITTING`状态或者`TIMED_WAITING`状态，直到其他线程唤醒该线程或者超时之后自动被唤醒；

- 线程在调用对象的wait方法之前必须获取到这个对象的monitor锁，否则将抛出IllegalMonitorStateException异常。

- 线程的等待是支持中断的，如果线程在等待过程中，被其他线程中断，则抛出InterruptedException异常。

  

## 九、finalize方法

```java
protected void finalize() throws Throwable { }
```

- 垃圾回收器在回收一个无用对象时，会调用对象的`finalize()`方法，所以可以重写`finalize()`方法来做一些清除工作；
- `finalize()`方法还能完成对对象的自救，避免垃圾回收器的回收；
- **JDK9开始将其标记为Deprecated，所以尽量不要使用该方法**；

