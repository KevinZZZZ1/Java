# String、StringBuffer和StringBuilder

## 一、StringBuffer和StringBuilder的历史

- String、StringBuffer、String Builder：
  - 三者都是final类，都不能被继承；
  - String类是不可变的，StringBuffer、StringBuilder是可变的；
  - StringBuffer是线程安全的，StringBuilder不是线程安全的；

由于Java字符串的不可变性，导致在使用+进行字符串拼接时，它们的内容都要被拷贝并生产一个新的字符串，这样的操作是很低效的，而且当内存中引用对象多了之后，JVM的GC就会开始工作，性能也会下降；

为了提升性能，JDK1.0提供了StringBuffer，它被设计为线程安全的，其实在实际项目中几乎不会遇到需要线程安全的场景，所以其加锁机制成为了性能负担；

从IDK1.5开始，提供了非线程安全的StringBuilder，内部少了加锁机制；



## 二、StringBuffer是否过时，对于使用StringBuffer代码的改进

**从Java1.5开始，优先使用StringBuilder；**

- **在没有使用循环的条件下，简单字符串拼接，直接用+完成，因为编译器会将+拼接优化成StringBuilder对字符串操作；**
- **简单的toString方法，建议使用+进行拼接，这样代码更简洁、更符合阅读习惯**



## 三、字符串拼接

- **简单字符串拼接直接使用+**：

  ```java
  String fileName = "LOG_"+date+suffix;
  ```

  

- **大批量字符串拼接**：

  - 无需考虑线程安全的情况下，使用StringBuilder，而且可以使用局部变量的方式来避免多线程争用的问题；

- **高性能字符串拼接：**

  - **由于在构造的字符串长度未知的情况下，StringBuilder的动态扩容机制是通过数组复制实现的**，如果一个字符非常大，那么在StringBuilder构造字符串过程中，会伴随着多次内存申请和数组复制操作，对性能消耗很大；
  - **StringBuilderHelper内部通过将数组扩容到最大值，并且重复使用数组来保证性能**；



## 四、源码分析

### String

```java
public final class String implements java.io.Serializable, Comparable<String>, CharSequence{
    
    private final char value[];
    
    // ...
    
}
```

`String`是继承`object`类，并实现了序列化接口，字符序列接口，字符串排序接口；

由于`String`中使用`final`的`char`数组存储字符，所以`String`是不可变的对象，每次对`String`类型改变的时候其实都等同于生成一个新的`String`对象，然后将指针指向新的`String`对象.

**创建字符串时有几点需要注意的：**

- ```java
  		 String str5 = "This is Java";
  		 String str6 = "This is";
  		 String str7 = "This is" + " Java";
  		 //输出true
  		 System.out.println(str5.equals(str7));
  		 //输出true，由于str7是利用字符串常量进行复制的，所以不会创建新对象
  		 System.out.println(str5 == str7); 
  
  		 String str8 = "This is Java";
  		 String str9 = "This is";
  		 String str10 = str9 + " Java";
  		 //输出true
  		 System.out.println(str8.equals(str10));
  		 //输出false，str10是通过对象引用来创建的，所以会创建新对象
  		 System.out.println(str8 == str10); 
  
  		 String str17 = new String("This is Java");
  		 String str9 = str17.intern();// 把字符串对象加入常量池中，
  		 String str18 = "This is Java";
  		 //输出false
  		 System.out.println(str9 == str17);
  		 //输出true
  		 System.out.println(str9 == str18);
  
  ```

  

- `String a = "xxx"` 可能创建一个或者不创建对象，当`xxx`这个字符常量在`String`常量池不存在，会在`String`常量池创建一个`String`对象，然后`a`会指向这个内存地址，后面继续用这种方式继续创建`xxx`字符串常量，始终只有一个内存地址被分配。

- `String a = new String("xxx")`至少创建一个对象。只要用到`new`就肯定在堆上创建一个`String`对象，并且检查`String`常量池中对应字符串是否存在，如果不存在就会在`Stirng`常量池创建该字符串常量，存在就不创建。







### StringBuffer





### StringBuilder





## 五、总结

- **未优化的字符串连接操作性能很差；**
- **简单的字符串拼接操作（+）会被编译器优化为StringBuilder.append；**
- **StringBuffer几乎不会被使用，StringBuilder在实际项目中使用最多；**
- **StringBuilderHelper更适合大数据量字符串拼接**；
- **StringBuilderHelper是一个线程持有一个StringBuilder实例**；



