# StringBuffer和StringBuilder

## 1.StringBuffer和StringBuilder的历史

由于Java字符串的不可变性，导致在使用+进行字符串拼接时，它们的内容都要被拷贝并生产一个新的字符串，这样的操作是很低效的；

为了提升性能，JDK1.0提供了StringBuffer，它被设计为线程安全的，其实在实际项目中几乎不会遇到需要线程安全的场景，所以其加锁机制成为了性能负担；

从IDK1.5开始，提供了非线程安全的StringBuilder，内部少了加锁机制；



## 2.StringBuffer是否过时，对于使用StringBuffer代码的改进

**从Java1.5开始，优先使用StringBuilder；**

- **简单字符串拼接，直接用+完成；**
- **简单的toString方法，建议使用+进行拼接，这样代码更简洁、更符合阅读习惯**



## 3.字符串拼接

- **简单字符串拼接直接使用+**：

  ```java
  String fileName = "LOG_"+date+suffix;
  ```

  

- **大批量字符串拼接**：

  - 无需考虑线程安全的情况下，使用StringBuilder，而且可以使用局部变量的方式来避免多线程争用的问题；

- **高性能字符串拼接：**

  - **由于在构造的字符串长度未知的情况下，StringBuilder的动态扩容机制是通过数组复制实现的**，如果一个字符非常大，那么在StringBuilder构造字符串过程中，会伴随着多次内存申请和数组复制操作，对性能消耗很大；
  - **StringBuilderHelper内部通过将数组扩容到最大值，并且重复使用数组来保证性能**；



## 4.总结

- **未优化的字符串连接操作性能很差；**
- **简单的字符串拼接操作（+）会被编译器优化为StringBuilder.append；**
- **StringBuffer几乎不会被使用，StringBuilder在实际项目中使用最多；**
- **StringBuilderHelper更适合大数据量字符串拼接**；
- **StringBuilderHelper是一个线程持有一个StringBuilder实例**；



