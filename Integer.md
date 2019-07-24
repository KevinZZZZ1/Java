# Integer

[TOC]

## 概述

Java中的Integer类主要实现了对基本类型int的封装，并且提供了一些处理int的方法；

非new生成的Integer变量指向的是java常量池中的对象，而new Integer()生成的变量指向堆中新建的对象，两者在内存中的地址不同 

## 继承结构

```java
public final class Integer extends Number implements Comparable<Integer> {
    // 其他代码
    ...  
}
```

- Integer类被`final`关键字修饰，是一个不可继承的类；

- Integer类继承了Number抽象类，该抽象类中主要是实现数值的转换，在Integer中直接通过强转实现；

- Integer类实现了Comparable接口，因此具备了比较大小的能力；

  ```java
  public int compareTo(Integer anotherInteger) {
      return compare(this.value, anotherInteger.value);
  }
  
  public static int compare(int x, int y) {
      return (x < y) ? -1 : ((x == y) ? 0 : 1);
  }
  ```

## 主要属性

```java
public final class Integer extends Number implements Comparable<Integer> {
    
    private final int value; // Integer 类包装的值，真正用来存储 int 值

    public static final int   MIN_VALUE = 0x80000000; // int 最小值为 -2^31 -2147483648

    public static final int   MAX_VALUE = 0x7fffffff; // int 最大值为 2^31-1 2147483647

    public static final Class<Integer>  TYPE = (Class<Integer>) Class.getPrimitiveClass("int"); // 基本类型 int 包装类的实例

    public static final int SIZE = 32; // 以二进制补码形式表示 int 值所需的比特数

    public static final int BYTES = SIZE / Byte.SIZE; // 以二进制补码形式表示 int 值所需的字节数。1.8 新添加字段

    private static final long serialVersionUID = 1360826667806852920L; // 序列化

    ...
    
}
```

- Integer类中使用一个`final`修饰的`int`类型的变量`value`来保存值；
- 静态常量`MAX_VALUE`、`MIN_VALUE`表示int类型可表示的最大值（2147483647）、最小值（-2147483648）；
- 静态常量`TYPE`等同于`int.class`;

```java
final static char [] DigitTens = {
        '0', '0', '0', '0', '0', '0', '0', '0', '0', '0',
        '1', '1', '1', '1', '1', '1', '1', '1', '1', '1',
        '2', '2', '2', '2', '2', '2', '2', '2', '2', '2',
        '3', '3', '3', '3', '3', '3', '3', '3', '3', '3',
        '4', '4', '4', '4', '4', '4', '4', '4', '4', '4',
        '5', '5', '5', '5', '5', '5', '5', '5', '5', '5',
        '6', '6', '6', '6', '6', '6', '6', '6', '6', '6',
        '7', '7', '7', '7', '7', '7', '7', '7', '7', '7',
        '8', '8', '8', '8', '8', '8', '8', '8', '8', '8',
        '9', '9', '9', '9', '9', '9', '9', '9', '9', '9',
        } ;

final static char [] DigitOnes = {
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        '0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        } ;

final static char[] digits = {
        '0' , '1' , '2' , '3' , '4' , '5' ,
        '6' , '7' , '8' , '9' , 'a' , 'b' ,
        'c' , 'd' , 'e' , 'f' , 'g' , 'h' ,
        'i' , 'j' , 'k' , 'l' , 'm' , 'n' ,
        'o' , 'p' , 'q' , 'r' , 's' , 't' ,
        'u' , 'v' , 'w' , 'x' , 'y' , 'z'
    };
final static int [] sizeTable = { 9, 99, 999, 9999, 99999, 999999, 9999999,
                                      99999999, 999999999, Integer.MAX_VALUE };
```

- `DigitTens`和`DigitOnes`两个数组的作用主要是，**获得0~99之间某个数的十位和各位**，比如36，可以用`DigitTens`数组直接得到十位为3，`DigitOnes`数组直接得到个位为6；
- `digits`数组表示Integer类支持的进制表示，**从中可以看出，支持从2进制到36进制**；
- `sizeTable`数组是用来，**判断一个整数对应字符串的长度**，使用这种方法能高效的得到相应的长度，避免使用除法操作来求解整数有多少位；

## IntegerCache内部类

```java
private static class IntegerCache {
    static final int low = -128; // 下界一定是-128
    static final int high; // 不确定，默认是127，可以通过-Djava.lang.Integer.IntegerCache.high参数设置
    static final Integer cache[]; // 静态final数组作为缓存，避免重复实例化和回收

    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            try {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127); // 比较设置的参数和127的大小，选择大的
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h;

        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++); // 设计缓存中的值

        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }

    private IntegerCache() {}
}
```

- `IntegerCache`是Integer类中的内部类，**主要实现的是缓存的功能**，其内部使用一个`Integer[]`数组来存放Integer类型的对象；
- `IntegerCache`默认的范围是[-128, 127]，可以通过JVM参数设置最大值，但是该参数只有在大于127时才会有效（如果小于127，范围还是[-128, 127]）；
- 在调用`valueOf(int i)`方法时，如果i的值在`IntegerCache`的表示范围之内的话，会直接从`IntegerCache`的Integer数组中取得对象；否则才会创建一个新的Integer对象；

**在Long、Short等包装类中都存在相应的Cache，而且low值都是-128**

## 主要方法

### 构造函数

```java
public Integer(int value) {
    this.value = value;
}

public Integer(String s) throws NumberFormatException {
    this.value = parseInt(s, 10);
}
```

从源码中可以看出，`Integer`提供了两个构造函数：

- 第一个构造函数参数是一个整数`value`，直接将`value`赋值给Integer类中的属性；
- 第二个构造函数参数是一个字符串s，其逻辑是：
  - 先调用`parseInt(String s, int radix)`函数，将字符串s转换为radix进制的整数；
  - 然后把上一步得到的结果赋值给Integer类中的属性；

### parseInt方法

```java
public static int parseInt(String s, int radix) throws NumberFormatException {
    /*
         * WARNING: This method may be invoked early during VM initialization
         * before IntegerCache is initialized. Care must be taken to not use
         * the valueOf method.
         */

    if (s == null) {
        throw new NumberFormatException("null");
    }

    if (radix < Character.MIN_RADIX) {
        throw new NumberFormatException("radix " + radix +
                                        " less than Character.MIN_RADIX");
    }

    if (radix > Character.MAX_RADIX) {
        throw new NumberFormatException("radix " + radix +
                                        " greater than Character.MAX_RADIX");
    }

    int result = 0;
    boolean negative = false;
    int i = 0, len = s.length();
    int limit = -Integer.MAX_VALUE;
    int multmin;
    int digit;

    if (len > 0) {
        char firstChar = s.charAt(0);
        if (firstChar < '0') { // Possible leading "+" or "-"
            if (firstChar == '-') {
                negative = true;
                limit = Integer.MIN_VALUE;
            } else if (firstChar != '+')
                throw NumberFormatException.forInputString(s);

            if (len == 1) // Cannot have lone "+" or "-"
                throw NumberFormatException.forInputString(s);
            i++;
        }
        multmin = limit / radix;
        while (i < len) {
            // Accumulating negatively avoids surprises near MAX_VALUE
            digit = Character.digit(s.charAt(i++),radix);
            if (digit < 0) {
                throw NumberFormatException.forInputString(s);
            }
            if (result < multmin) {
                throw NumberFormatException.forInputString(s);
            }
            result *= radix;
            if (result < limit + digit) {
                throw NumberFormatException.forInputString(s);
            }
            result -= digit;
        }
    } else {
        throw NumberFormatException.forInputString(s);
    }
    return negative ? result : -result;
}
```





### getChars方法







### toString方法





### valueOf方法

```java
// 将字符串s转换为指定进制radix的整数
public static Integer valueOf(String s, int radix) throws NumberFormatException {
    return Integer.valueOf(parseInt(s,radix));
}
// 将字符串s转换为10进制的整数
public static Integer valueOf(String s) throws NumberFormatException {
    return Integer.valueOf(parseInt(s, 10));
}
// 判断i是否在IntegerCache的表示范围内，如果是的话，从IntegerCache中取值，否则新创建一个对象；
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

- `valueOf(int i)`是这一系列方法的核心，其逻辑是：

  - 先判断参数i是否在区间[IntegerCache.low, IntegerCache.high]内，如果在，则从IntegerCache中得到相应值的Integer对象；
  - 否则创建一个新的值为i的Integer对象；

- **这个方法会被运用到Java的自动装箱上：**

  ```java
  Integer a = 100;
  会被自动装箱成：
  Integer a = Integer.valueOf(100);
  ```

- **而自动拆箱使用的是intValue()方法**：

  ```java
  Integer a = new Integer(128);
  int m = a;
  会被自动拆箱为：
  int m = a.intValue();
  ```

- **简单来说，自动装箱就是Integer.valueOf(int i);自动拆箱就是 i.intValue(); **

### decode方法







### hashCode方法







### equals方法















###bitCount方法







###highestOneBit方法、lowestOneBit方法







###numerOfLeadingZeros方法、 numerOfTrailingZeros方法





## 总结





## 参考文献

<https://juejin.im/post/5c76ad1ae51d4572c95835d0> 

<https://juejin.im/post/5a9cfc7a51882555686838e3> 

<https://juejin.im/post/5992b1986fb9a03c3223ce32> 

<https://www.cnblogs.com/ysocean/p/8564466.html> 



