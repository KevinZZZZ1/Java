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
	// 如果字符串为空，则抛出异常
    if (s == null) {
        throw new NumberFormatException("null");
    }
	// Character.MIN_RADIX的值为2
    if (radix < Character.MIN_RADIX) {
        throw new NumberFormatException("radix " + radix +
                                        " less than Character.MIN_RADIX");
    }
	// Character.MAX_RADIX的值为36
    if (radix > Character.MAX_RADIX) {
        throw new NumberFormatException("radix " + radix +
                                        " greater than Character.MAX_RADIX");
    }
	// 由于parseInt("1234",10)其实可以写成：1234 = (((1*10)+2)*10+3)*10+4
    // 虽然实际上是负数累减的形式，但是原理是一样的
    int result = 0; // 记录最后结果
    boolean negative = false; // 使用负数进行转换，因为负数的表示范围比正数大，所以更好处理
    int i = 0, len = s.length();// 从0位置开始遍历字符串中字符
    int limit = -Integer.MAX_VALUE; // 正数情况下允许的最大值
    int multmin; // 乘法允许的最大值
    int digit;
	// 字符串的长度要大于0
    if (len > 0) {
        // 对第一个字符进行判断，看是否是符号位
        char firstChar = s.charAt(0);
        if (firstChar < '0') { // Possible leading "+" or "-"
            if (firstChar == '-') {
                // 负数的情况
                negative = true;
                limit = Integer.MIN_VALUE;
            } else if (firstChar != '+')
                throw NumberFormatException.forInputString(s);

            if (len == 1) // Cannot have lone "+" or "-"
                throw NumberFormatException.forInputString(s);
            i++;
        }
        multmin = limit / radix; // result*radix的最大值，超过该值就会溢出
        while (i < len) {
            // Accumulating negatively avoids surprises near MAX_VALUE
            // 位置为i的字符对应的10进制数
            digit = Character.digit(s.charAt(i++),radix);
            if (digit < 0) {
                throw NumberFormatException.forInputString(s);
            }
            // result,multmin都是负数，这里其实是表示溢出
            if (result < multmin) {
                throw NumberFormatException.forInputString(s);
            }
            // 1234 = (((-1*10)-2)*10-3)*10-4
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

`parseInt(String s, int radix)`的功能是，将内容是s进制是radix的字符串转成10进制的整数，主要逻辑是：

- 先判断字符串是否为`null`，如果为`null`则抛出异常；
- 再判断参数radix是否在[2, 36]之间（因为Integer类只支持2进制~36进制）；
- 遍历字符串s中的每个字符，**以负数的形式（负数表示的范围更大）相乘累减**，即1234 = -(((-1* 10)-2) * 10-3)*10-4；

### getChars方法

```java
// 将整数i对应的字符串中的字符放入数组buf中，从index位置开始向前添加
static void getChars(int i, int index, char[] buf) {
    int q, r; 
    int charPos = index;
    char sign = 0;
	// 负数情况的处理
    if (i < 0) {
        sign = '-';
        i = -i;
    }

    // 一次循环产生两位
    while (i >= 65536) {
        q = i / 100;
        // really: r = i - (q * 100);
        // r就是i的最低两位
        r = i - ((q << 6) + (q << 5) + (q << 2)); // ((q << 6) + (q << 5) + (q << 2))相当于q*100，也就是q*2^6+q*2^5+q*2^2，即64q+32q+4q
        i = q;
        // 直接使用DigitOnes,DigitTens数组得到数字r的十位和个位
        buf [--charPos] = DigitOnes[r];
        buf [--charPos] = DigitTens[r];
    }

    // Fall thru to fast mode for smaller numbers
    // assert(i <= 65536, i);
    for (;;) {
        // 52429 = (2 ^ 19) / 10 +1,(i * 52429) >>> (16+3)相当于i/10
        q = (i * 52429) >>> (16+3); // 相当于q=i/10
        // ((q << 3) + (q << 1));相当于q*2^3+q*2^1，即8q+2q，r就是i的最低位
        r = i - ((q << 3) + (q << 1));  // r = i-(q*10) ...
        buf [--charPos] = digits [r];
        i = q;
        if (i == 0) break;
    }
    if (sign != 0) {
        buf [--charPos] = sign;
    }
}
```

`getChars(int i, int index, char[] buf)`主要实现的功能是，将参数i以字符的形式添加到buf数组中，上面的函数使用了很多优化的技巧，大概逻辑是：

- 如果i>65536，那么一次循环会产生i的最低位的两个字符；

  ```java
  while(i>65536){
      q = i/100;
      r = i - ((q<<6)+(q<<5)+(q<<2));
      i = q;
      
      buf[--charPos] = DigitOnes[r];
      buf[--charPos] = DigitOnes[r];
  }
  ```

  

- 否则，一次循环会产生i的最低位的字符；

  ```java
  for( ; ; ){
      q = (i*52429)>>(16+3);
      r = i - ((q<<3)+(q<<1));
      buf[--charPos] = digits[r];
      i = q;
      if(i==0)
          break;
  }
  ```

### toString方法

```java
public String toString() {
    return toString(value);
}

public static String toString(int i) {
    if (i == Integer.MIN_VALUE)
        return "-2147483648";
    int size = (i < 0) ? stringSize(-i) + 1 : stringSize(i);
    char[] buf = new char[size];
    getChars(i, size, buf);
    return new String(buf, true);
}
// 计算整数x的位数
static int stringSize(int x) {
    // 遍历sizeTable数组，该数组中记录着
    for (int i=0; ; i++)
        if (x <= sizeTable[i])
            return i+1;
}
```

`toString(int i)`方法实现了将整数i转换为字符串的功能，这里主要看`stringSize(int x)`这个方法，该方法通过在遍历`sizeTable[]`数组中的元素来判断x的位数（因为`sizeTable[]`数组中存储的是每种位的最大值）；

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

```java
public static Integer decode(String nm) throws NumberFormatException {
    int radix = 10;
    int index = 0;
    boolean negative = false;
    Integer result;

    if (nm.length() == 0)
        throw new NumberFormatException("Zero length string");
    char firstChar = nm.charAt(0);
    if (firstChar == '-') {
        negative = true;
        index++;
    } else if (firstChar == '+')
        index++;
    if (nm.startsWith("0x", index) || nm.startsWith("0X", index)) {
        index += 2;
        radix = 16;
    }
    else if (nm.startsWith("#", index)) {
        index ++;
        radix = 16;
    }
    else if (nm.startsWith("0", index) && nm.length() > 1 + index) {
        index ++;
        radix = 8;
    }

    if (nm.startsWith("-", index) || nm.startsWith("+", index))
        throw new NumberFormatException("Sign character in wrong position");

    try {
        result = Integer.valueOf(nm.substring(index), radix);
        result = negative ? Integer.valueOf(-result.intValue()) : result;
    } catch (NumberFormatException e) {
        String constant = negative ? ("-" + nm.substring(index))
            : nm.substring(index);
        result = Integer.valueOf(constant, radix);
    }
    return result;
}
```

decode方法主要作用是解码字符串转成Integer型，比如`Integer.decode("11")`的结果为11；`Integer.decode("0x11")`和`Integer.decode("#11")`结果都为17，因为0x和#开头的会被处理成十六进制；`Integer.decode("011")`结果为9，因为0开头会被处理成8进制；

### hashCode方法

```java
public int hashCode() {
    return Integer.hashCode(value);
}
public static int hashCode(int value) {
    return value;
}
```

`hashCode()`是直接返回int类型的值；

### equals方法

```java
public boolean equals(Object obj) {
    if (obj instanceof Integer) {
        return value == ((Integer)obj).intValue();
    }
    return false;
}
```

比较是否相同时先判断是不是Integer类型再比较值；

###bitCount方法

```java
public static int bitCount(int i) {
    i = i - ((i >>> 1) & 0x55555555);
    i = (i & 0x33333333) + ((i >>> 2) & 0x33333333);
    i = (i + (i >>> 4)) & 0x0f0f0f0f;
    i = i + (i >>> 8);
    i = i + (i >>> 16);
    return i & 0x3f;
}
```

- 该方法主要用于计算二进制数中1的个数。一看有点懵，都是移位和加减操作。先将重要的列出来，`0x55555555`等于`01010101010101010101010101010101`，`0x33333333`等于`110011001100110011001100110011`，`0x0f0f0f0f`等于`1111000011110000111100001111`。它的核心思想就是先每两位一组统计看有多少个1，比如`10011111`则每两位有1、1、2、2个1，记为`01011010`，然后再算每四位一组看有多少个1，而`01011010`则每四位有2、4个1，记为`00100100`，接着每8位一组就为`00000110`，接着16位，32位，最终在与`0x3f`进行与运算，得到的数即为1的个数；

###highestOneBit方法、lowestOneBit方法

```java
public static int highestOneBit(int i) {
    i |= (i >>  1);
    i |= (i >>  2);
    i |= (i >>  4);
    i |= (i >>  8);
    i |= (i >> 16);
    return i - (i >>> 1);
}
```

- 该方法返回i的二进制中最高位的1，其他全为0的值。比如i=10时，二进制即为1010，最高位的1，其他为0，则是1000。如果i=0，则返回0。如果i为负数则固定返回-2147483648，因为负数的最高位一定是1，即有`1000,0000,0000,0000,0000,0000,0000,0000`。这一堆移位操作是什么意思？其实也不难理解，将i右移一位再或操作，则最高位1的右边也为1了，接着再右移两位并或操作，则右边1+2=3位都为1了，接着1+2+4=7位都为1，直到1+2+4+8+16=31都为1，最后用`i - (i >>> 1)`自然得到最终结果；

```java
public static int lowestOneBit(int i) {
    return i & -i;
}
```

- 与highestOneBit方法对应，lowestOneBit获取最低位1，其他全为0的值。这个操作较简单，先取负数，这个过程需要对正数的i取反码然后再加1，得到的结果和i进行与操作，刚好就是最低位1其他为0的值了 ；

###numerOfLeadingZeros方法、 numerOfTrailingZeros方法

```java
 public static int numberOfLeadingZeros(int i) {
     if (i == 0)
         return 32;
     int n = 1;
     if (i >>> 16 == 0) { n += 16; i <<= 16; }
     if (i >>> 24 == 0) { n +=  8; i <<=  8; }
     if (i >>> 28 == 0) { n +=  4; i <<=  4; }
     if (i >>> 30 == 0) { n +=  2; i <<=  2; }
     n -= i >>> 31;
     return n;
 }
```

- 该方法返回i的二进制从头开始有多少个0。i为0的话则有32个0。这里处理其实是体现了二分查找思想的，先看高16位是否为0，是的话则至少有16个0，否则左移16位继续往下判断，接着右移24位看是不是为0，是的话则至少有16+8=24个0，直到最后得到结果；

```java
public static int numberOfTrailingZeros(int i) {
    int y;
    if (i == 0) return 32;
    int n = 31;
    y = i <<16; if (y != 0) { n = n -16; i = y; }
    y = i << 8; if (y != 0) { n = n - 8; i = y; }
    y = i << 4; if (y != 0) { n = n - 4; i = y; }
    y = i << 2; if (y != 0) { n = n - 2; i = y; }
    return n - ((i << 1) >>> 31);
}
```

- 与前面的numberOfLeadingZeros方法对应，该方法返回i的二进制从尾开始有多少个0。它的思想和前面的类似，也是基于二分查找思想，详细步骤不再赘述；





## 总结

- Integer类中的内部类IntegerCache中有一个Integer类型的数组，里面存储着Integer对象，默认值是在[-128, 127]之间，当使用Integer类型创建对象时，如果数值在[-128, 127]区间内，则会从IntegerCache的Integer数组中返回；
- Integer类的sizeTable数组属性记录了1,2,3,4,5...的整数的最大值，这样在求解一个整数的位数时速度很快；
- 从Integer类中的一些方法（`parseInt()`）可以看出，使用负数来处理Integer中的累加问题比较方便，因为负数的表示范围更大，无需特殊情况处理；
- 能使用移位解决的问题，就不要使用乘除法（q=i/10可以写成q=(i*52429)>>(19)）；



## 参考文献

<https://juejin.im/post/5c76ad1ae51d4572c95835d0> 

<https://juejin.im/post/5a9cfc7a51882555686838e3> 

<https://juejin.im/post/5992b1986fb9a03c3223ce32> 

<https://www.cnblogs.com/ysocean/p/8564466.html> 



