# String、StringBuffer和StringBuilder

[TOC]

## 一、StringBuffer和StringBuilder的历史

- String、StringBuffer、String Builder：
  - 三者都是final类，都不能被继承；
  - ssString类是不可变的，StringBuffer、StringBuilder是可变的；
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
    
    /** The value is used for character storage. */
    private final byte[] value;
 
    /** The count is the number of characters in the String. */
    private final int count;
 
    /** Cache the hash code for the string */
    private int hash; // Default to 0
 
    /** use serialVersionUID from JDK 1.0.2 for interoperability */
    private static final long serialVersionUID = -6849794470754667710L;
    
}
```

`String`是继承`object`类，并实现了序列化接口，字符序列接口，字符串排序接口；

由于**`String`中使用`final`的`byte`数组存储字符，所以`String`是不可变的对象**，每次对`String`类型改变的时候其实都等同于生成一个新的`String`对象，然后将指针指向新的`String`对象.

**String.intern()方法：**

对于Java的8种基本数据类型和`String`类型，都提供了常量池，这是为了它们在运行过程中速度更快、更节省内存；**常量池类似于一个Java系统级别提供的缓存**。8种基本类型的常量池是由系统协调的，而`String`类型的常量池比较特殊也很重要，其用法有：

- **直接使用双引号声明出来的String对象会直接进入常量池；**例如：String s="124"; 
- **如果不是用双引号声明的String对象，可以使用String的intern()方法，intern()方法会从字符串常量池中查询当前字符串是否存在，若存在，则从字符串常量池中返回该字符串；若不存在就会将当前字符串放入常量池中，并且返回对该字符串的引用**

intern()方法的大致原理：

- Java使用jni调用C++实现的`StringTable`的`intern（）`方法，`intern()`方法和Java中的`HashMap`的实现是差不多的，但是在JDK6中不能扩容，默认大小为1009；
- 由于在JDK6中`String`的`String Pool`是一个固定大小（1009）的`HashTable`，所以常量池中的字符串过多会导致Hash冲突严重，链表长度很长，导致调用`String.intern`性能大幅下降；
- 在JDK7中，可以通过参数来调整`StringTable`的大小：`--XX:StringTableSize=99991`

第一次使用**String s = new String("abc");**会创建2个对象，即：

- "abc"字符串对象存储在常量池中；

- 在Java堆上创建了String对象；

  

代码分析1：

```java
pulic static void main(String[] args){
    String s = new String("1");
    s.intern();
    String s2 = "1";
    System.out.println(s == s2); 
    
    String s3 = new String("1")+new String("1");
    s3.intern();
    String s4 = "11";
    System.out.println(s3 == s4);
}
```

在JDK6下结果为：`false false`;

在JDK7下结果为：`false true`;

分析：

![avatar](F:\找工作\Java基础\Java\images\String.intern1.png)

- **在JDK6中全为false的原因是，JDK6的常量池是放在Perm区的，Perm区和Java堆是完全分开的**，在s创建的时候，会在Java堆创建`String obj`和在常量池中创建常量"1"，s2则是直接使用的字符串常量池中的常量“1”，所以s引用的是Java堆上的字符串对象，而s2引用的是字符串常量池中的常量，所以为false；同理对于s3、s4也是一样；

![avatar](F:\找工作\Java基础\Java\images\String.intern1.2.png)

- **在JDK7中字符串常量是放在Java堆中**，对于`String s3 = new String("1") + new String("1");`，这句代码中现在生成了2最终个对象，是字符串常量池中的“1” 和 Java堆中的 s3引用指向的对象。中间还有2个匿名的`new String("1")`我们不去讨论它们。此时s3引用对象内容是”11”，但此时常量池中是没有 “11”对象的；
- `s3.intern();`是将 s3中的“11”字符串放入 String 常量池中，因为此时常量池中不存在“11”字符串，由于 jdk7 中常量池不在 Perm 区域了，所以常量池中不需要再存储一份对象了，可以直接存储堆中的引用。这份引用指向 s3 引用的对象。 也就是说引用地址是相同的；
- 最后`String s4 = "11";` 这句代码中”11”是显示声明的，因此会直接去常量池中创建，创建的时候发现已经有这个对象了，此时也就是指向 s3 引用对象的一个引用。所以 s4 引用就指向和 s3 一样了。因此最后的比较 `s3 == s4` 是 true；
- s和s2对象的分析也如上；



代码分析2：

```java
public static void main(String[] args){
    String s = new String("1");
    String s2 = "1";
    s.intern();
    System.out.println(s == s2);
    
    String s3 = new String("1")+new String("1");
    String s4 = "11";
    s3.intern();
    System.out.println(s3 == s4);
}
```

在JDK6下结果为：`false false`;

在JDK7下结果为：`false false`;

分析：

- 对于JDK6的情况，和第一段代码的分析是一样的，就是**s是指向Java堆上的对象的，s2是指向字符串常量池中的常量**，所以结果一定是false；（s3和s4也是类似的分析）；

![avatar](F:\找工作\Java基础\Java\images\String.intern2.1.png)

- 对于JDK7来说，首先执行`String s4 = "11";`声明 s4 的时候常量池中是不存在“11”对象的，执行完毕后，“11“对象是 s4 声明产生的新对象。然后再执行`s3.intern();`时，常量池中“11”对象已经存在了，因此 s3 和 s4 的引用是不同的；
- 对于s和s2也是一样的分析；



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

  

- `String a = "xxx"` 是声明一个字符串常量，在编译器就确定，可能创建一个或者不创建对象，当`xxx`这个字符常量在`String`常量池不存在，会在`String`常量池创建一个`String`对象，然后`a`会指向这个内存地址，后面继续用这种方式继续创建`xxx`字符串常量，始终只有一个内存地址被分配。

- `String a = new String("xxx")`创建的字符串不是常量，不能在编译器确定，至少创建一个对象。只要用到`new`就肯定在堆上创建一个`String`对象，并且检查`String`常量池中对应字符串是否存在，如果不存在就会在`Stirng`常量池创建该字符串常量，存在就不创建。

### AbstractStringBuilder

```java
 public final class StringBuffer
    extends AbstractStringBuilder
    implements java.io.Serializable, Comparable<StringBuffer>, CharSequence

 public final class StringBuffer
    extends AbstractStringBuilder
    implements java.io.Serializable, Comparable<StringBuffer>, CharSequence
```

可以看到`StringBuffer`和`StringBuilder`的继承和实现关系是一模一样的，`Serializable`是可以序列化的标志。`CharSequence`接口包含了`charAt()`、`length()` 、`subSequence()`、`toString()`这几个方法；重点是抽象类`AbstractStringBuilder`，这个类封装了`StringBuilder`和`StringBuffer`大部分操作的实现。（模板设计模式？）

下面是`AbstractStringBuilder`的部分源码：

- 数据结构、构造函数：

  ```java
  abstract class AbstractStringBuilder implements Appendable, CharSequence{
      /**
       * The value is used for character storage.
       */
      byte[] value;
      
      /**
       * The id of the encoding used to encode the bytes in {@code value}.
       */
      byte coder;
      
      /**
       * The count is the number of characters used.
       */
      int count;
      
      /**
       * Creates an AbstractStringBuilder of the specified capacity.
       */
      AbstractStringBuilder(int capacity) {
          if (COMPACT_STRINGS) {
              value = new byte[capacity];
              coder = LATIN1;
          } else {
              value = StringUTF16.newBytesFor(capacity);
              coder = UTF16;
          }
      }
  }
  ```

  - 从上面可以看出，`AbstractStringBuilder`使用`byte[]`数组保存字符串，可以在构造时指定初始容量；

  - coder有两个取值：

    ```java
    @Native static final byte LATIN1 = 0;
    @Native static final byte UTF16  = 1;
    ```

    这是在JDK9的String类中，维护了这样的一个属性coder，它是一个编码格式的标识，使用LATIN1还是UTF-16，如果字符串中都是能用LATIN1就能表示的就是0，否则就是UTF-16。由于字符串的存储由原来的`char[]`数组变成了`byte[]`数组，所以在求字符长度的时候有一些变化，字符长度是等于字符中的Unicode编码单元数，而不同的编码方式占用的字节数也是不同的，LATIN1占用一个字节，UTF16占用两个字节：

    ```java
    public int length() {
        return value.length >> coder();
    }
    
    ```

    

- 扩容方法：

  ```java
  public void ensureCapacity(int minimumCapacity) {
          if (minimumCapacity > 0) {
              ensureCapacityInternal(minimumCapacity);
          }
  }
  
  private void ensureCapacityInternal(int minimumCapacity) {
          // overflow-conscious code
          int oldCapacity = value.length >> coder; // 求出value数组的代表字符串的长度
          if (minimumCapacity - oldCapacity > 0) {
              // 如果预期字符串长度大于value数组长度，则进行扩容
              // value是一个byte[]数组，所以最后还要进行右移操作
              value = Arrays.copyOf(value, newCapacity(minimumCapacity) << coder);
          }
  }
  // 计算新的value数组的大小
  private int newCapacity(int minCapacity) {
          // overflow-conscious code
          int oldCapacity = value.length >> coder; // value数组所存储的字符串的长度
          int newCapacity = (oldCapacity << 1) + 2; // 先尝试将新容量扩大为原来的2倍+2
          if (newCapacity - minCapacity < 0) { // newCapacity达不到预期值
              newCapacity = minCapacity;
          }
      	// 进行溢出检查
          int SAFE_BOUND = MAX_ARRAY_SIZE >> coder;
          return (newCapacity <= 0 || SAFE_BOUND - newCapacity < 0)
              ? hugeCapacity(minCapacity)
              : newCapacity;
  }
  
  private int hugeCapacity(int minCapacity) {
          int SAFE_BOUND = MAX_ARRAY_SIZE >> coder;
          int UNSAFE_BOUND = Integer.MAX_VALUE >> coder;
          if (UNSAFE_BOUND - minCapacity < 0) { // overflow
              throw new OutOfMemoryError();
          }
          return (minCapacity > SAFE_BOUND)
              ? minCapacity : SAFE_BOUND;
  }
  
  // 将最大数组大小设置为该值的原因：有些VM中会在数组中保留head words
  private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
  
  ```

  由上面源码我们可知，主要是解决两个问题：

  - 数组使用上面方法扩容复制？
  - 新数组的容量是多少？

  解决方法是：

  - `byte[]`数组扩容时，使用的是`Arrays.copyOf()`方法来进行数组复制；

  - 需要扩容时，先尝试间新的value数组的容量`newCapacity`设置为原来的2倍+2，如果仍小于指定的容量，那么就把`newCapacity`设为预期值`minimumCapacity`；

  - 最后判断`newCapacity`是否溢出；

    

- `append()方法`：

  ```java
  // Object
  public AbstractStringBuilder append(Object obj) {
          return append(String.valueOf(obj));
  }
  
  public AbstractStringBuilder append(String str) {
          if (str == null) {
              return appendNull(); // 直接把字符串"null"添加到byte[]数组中
          }
          int len = str.length(); // 新加入字符串的长度
          ensureCapacityInternal(count + len); // 判断是否需要扩容
          putStringAt(count, str); // 将str字符串加入到value数组中
          count += len; 
          return this;
  }
  
  private AbstractStringBuilder appendNull() {
          ensureCapacityInternal(count + 4);
          int count = this.count;
          byte[] val = this.value;
      	// 根据字符串编码的不同将"null"添加到value数组中
          if (isLatin1()) {
              val[count++] = 'n';
              val[count++] = 'u';
              val[count++] = 'l';
              val[count++] = 'l';
          } else {
              count = StringUTF16.putCharsAt(val, count, 'n', 'u', 'l', 'l');
          }
          this.count = count;
          return this;
  }
  
  private final void putStringAt(int index, String str) {
          if (getCoder() != str.coder()) {
              inflate();
          }
      	// 调用String类型的getBytes()将str添加到value数组末尾
          str.getBytes(value, index, coder);
  }
  
  ```

  ```java
  // char
  public AbstractStringBuilder append(char[] str) {
      int len = str.length;
      ensureCapacityInternal(count + len);
      appendChars(str, 0, len);
      return this;
  }
  
  private final void appendChars(char[] s, int off, int end) {
          int count = this.count;
          if (isLatin1()) {
              byte[] val = this.value;
              for (int i = off, j = count; i < end; i++) {
                  char c = s[i];
                  if (StringLatin1.canEncode(c)) {
                      val[j++] = (byte)c;
                  } else {
                      this.count = count = j;
                      inflate();
                      StringUTF16.putCharsSB(this.value, j, s, i, end);
                      this.count = count + end - i;
                      return;
                  }
              }
          } else {
              StringUTF16.putCharsSB(this.value, count, s, off, end);
          }
          this.count = count + end - off;
  }
  
  ```

  

  - `append()`是最常用的方法，它有很多形式的重载。首先说明一点`AbstractStringBuilder`内部用一个`byte[]`数组保存字符串，可以在构造的时候指定初始容量方法。
  - `append()`各种重载方法的处理流程都是类似的，根据参数不同大概可以分成下面几种类型：
    - 参数是`String`：首先判断输入字符串是否为`null`，如果是`null`，直接将`null`字符串添加到`value`数组中（包括判断是否要扩容）；如果不是`null`，首先判断是否要扩容，再调用`System.arraycopy()`方法将输入字符串添加到`value`数组末尾；
    - 参数是`char`或`char[]`:同上；
    - 参数是`boolean`:就是把`true`和`false`以字符串的形式添加到`value`数组中；
    - 参数是`int`等基本数据类型：将基本数据类型转换为`String`字符串加到`value`数组中

- `replace()方法`：

  ```java
  public AbstractStringBuilder replace(int start, int end, String str) {
          int count = this.count;
          if (end > count) {
              end = count; // 超出部分无法替换
          }
          checkRangeSIOOBE(start, end, count); // 进行边界检查
          int len = str.length();
          int newCount = count + len - (end - start); 
          ensureCapacityInternal(newCount);
          shift(end, newCount - count); 
          this.count = newCount;
          putStringAt(start, str);
          return this;
  }
  // 
  private void shift(int offset, int n) {
      System.arraycopy(value, offset << coder,
                       value, (offset + n) << coder, (count - offset) << coder);
  }
  
  ```

  - `replace`方法用于替换字符串。会对开始和结束为止进行大小比较，如果有误的话会报错。之后会算一下现在的容量和替换之后的容量大小，之后扩容，然后调用`String`的`getChars()`方法将`str`追加到`value`末尾;

  - arraycopy(被复制的数组, 从第几个元素开始复制, 要复制到的数组, 从第几个元素开始粘贴, 一共需要复制的元素个数)

  - `shift()`方法是指对`value`数组中元素进行平移操作，例如：

    - ```java
      StringBuilder sb = new StringBuilder();
      sb.append("01234");
      sb.replace(2,3,"zyn");
      // 结果是："01234"==>"01zyn34"
      
      ```

    - `shift()`的过程是："01234"==>"0123434"，最后调用`putStringAt()`方法变成："01zyn34"

- `delete()方法`：

  ```java
  public AbstractStringBuilder deleteCharAt(int index) {
      checkIndex(index, count);
      shift(index + 1, -1);
      count--;
      return this;
  }
  
  private void shift(int offset, int n) {
      System.arraycopy(value, offset << coder,
                       value, (offset + n) << coder, (count - offset) << coder);
  }
  
  ```

- `insert()方法`：

  ```java
  public AbstractStringBuilder insert(int offset, String str) {
          checkOffset(offset, count);
          if (str == null) {
              str = "null";
          }
          int len = str.length();
          ensureCapacityInternal(count + len);
          shift(offset, len);
          count += len;
          putStringAt(offset, str);
          return this;
  }
  
  ```

  - `insert`也有很多形式的重载。都是通过调用`arraycopy`数组来实现的，如果插入一个字符串的话，先判断是否是`null`,是`null`的话相当于插入一个"null"字符串，第一个参数是想要插入的位置，之后也是想扩容，之后调用arraycopy方法，之后调用`String`的`getChars()`方法将`str`追加到`value`末尾。

`AbstractStringBuilder`已经实现了大部分需要的方法，`StringBuilder`和`StringBuffer`只需要调用即可（模块设计模式）；



### StringBuffer

```java
 public final class StringBuffer
    extends AbstractStringBuilder
    implements java.io.Serializable, Comparable<StringBuffer>, CharSequence{
    
    	......
    }

```

- 构造方法：

  ```java
  public StringBuffer() {
          super(16);
  }
  
  public StringBuffer(int capacity) {
          super(capacity);
  }
  
  public StringBuffer(String str) {
          super(str.length() + 16);
          append(str);
  }
  
  public StringBuffer(CharSequence seq) {
          this(seq.length() + 16);
          append(seq);
  }
  
  ```

  - `StringBuffer`默认大小为16，也可以指定其大小，若使用`String`或`CharSequence`类型参数，则其大小为参数大小+16；

- `append()`方法：

  ```java
  public synchronized StringBuffer append(String str) {
      toStringCache = null;
      super.append(str);
      return this;
  }
  
  ```

  - 使用了`synchronized`关键字来保证线程安全；

- `toString()`方法：

  ```java
  public synchronized String toString() {
          if (toStringCache == null) {
              return toStringCache =
                      isLatin1() ? StringLatin1.newString(value, 0, count)
                                 : StringUTF16.newString(value, 0, count);
          }
          return new String(toStringCache);
  }
  
  ```

  - `toStringCache`是一个用来缓存上次`toString()`方法的返回值，只要`StringBuffer`发生变化就清空；



### StringBuilder

```java
 public final class StringBuffer
    extends AbstractStringBuilder
    implements java.io.Serializable, Comparable<StringBuffer>, CharSequence{
    	......
    
    }

```

- 构造方法：

  ```java
  public StringBuilder() {
          super(16);
  }
  
  public StringBuilder(int capacity) {
          super(capacity);
  }
  
  public StringBuilder(String str) {
          super(str.length() + 16);
          append(str);
  }
  
  public StringBuilder(CharSequence seq) {
          this(seq.length() + 16);
          append(seq);
  }
  
  ```

  - `StringBuilder`默认大小为16，也可以指定其大小，若使用`String`或`CharSequence`类型参数，则其大小为参数大小+16；

- `append()`方法:

  ```java
  public StringBuilder append(String str) {
          super.append(str);
          return this;
  }
  
  ```

  

- `toString()`方法：

  ```java
  public String toString() {
          // Create a copy, don't share the array
          return isLatin1() ? StringLatin1.newString(value, 0, count)
                            : StringUTF16.newString(value, 0, count);
  }
  
  ```

  





## 五、总结

- **未优化的字符串连接操作性能很差；**
- **简单的字符串拼接操作（+）会被编译器优化为StringBuilder.append；**
- **StringBuffer几乎不会被使用，StringBuilder在实际项目中使用最多；**
- **StringBuilderHelper更适合大数据量字符串拼接**；
- **StringBuilderHelper是一个线程持有一个StringBuilder实例**；



