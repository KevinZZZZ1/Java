# Exception和Error

## `Exception`和`Error`的联系

- `Exception`和`Error`都继承了`Throwable`类，`Java`中只有Throwable类型的实例才能被抛出（throw）或捕获（catch）；
- `Error`和`RuntimeException`及其子类被称为未检查异常，其他异常被称为检查异常；

![avatar](https://github.com/KevinZZZZ1/Java/blob/master/images/Excption%E5%92%8CError.png)

## `Exception`和`Error`的区别

- `Exception`是程序正常运行中，可预料会出现的情况，应该被捕获；
- `Error`是指在正常情况下，不大可能出现的情况，一般是指和虚拟机有关的问题，比如`StackOverFlowError`和`OutOfMemoryError`；





## 运行时异常和受检查异常

`Exception`可以分为可检查异常和不检查异常：

- 可检查异常，是指在源码中必须显示处理，这是编译器检查的一部分；
- 不检查异常，也就是运行时异常，通常来说都是程序写的有问题；



## 使用异常时需要注意的点

- 尽量捕获特定异常，而不是捕获通用异常；
- 不要生吞（swallow）异常；
- 尽量使用日志系统记录异常抛出时信息；
- Throw early，catch late原则；



## `throw`和`throws`关键字的不同

- `throw`是用来抛出任意异常的，可以抛出`Throwable`、自定义异常类对象等；
- `throws`出现在函数头中，用来表明该函数可能会抛出的异常，调用这个方法要进行异常处理；



## `try-catch-finally-return`的执行顺序

- 不管异常是否发生，`finally`块中的代码都会执行（除非在`try`或`catch`语句中执行了`System.exit(0)`方法）；

- 当`try`或`catch`中有`return`语句时，`finally`块仍然执行；

- `finally`是在执行完`return`之后执行的，所以函数的返回值在`return`执行完后就确定，`finally`块无法修改；

- `finally`块最好不要包含`return`，否则程序会提前退出；

- 实例代码1：

  - ```java
    public static void demo(){
    
            try {
                System.out.println("try");
            }catch (Exception e){
                System.out.println(e.getMessage());
            }finally {
                System.out.println("finally");
            }
    
    }
    ```

  - ```
    执行结果为：
    try
    finally
    ```

- 实例代码2：

  - ```java
    public class TestTryCatch {
    
        public static int demo2(){
    
            try {
                System.out.println("try");
                return 0;
            }catch (Exception e){
                System.out.println(e.getMessage());
                return -1;
            }finally {
                System.out.println("finally");
            }
    
        }
    
        public static void main(String[] args) {
            System.out.println("main "+demo2());
        }
    }
    ```

  - ```
    执行结果是：
    try
    finally
    main 0
    // 执行完try和finally语句之后最后执行return
    ```

- 实例代码3：

  - ```java
    public class TestTryCatch {
    
        public static int demo3(){
            int i=0;
            try {
                i=6;
                System.out.println("try "+i);
                return i;
            }catch (Exception e){
                System.out.println(e.getMessage());
                return -1;
            }finally {
                i=12;
                System.out.println("finally "+i);
            }
    
        }
    
        public static void main(String[] args) {
            System.out.println("main "+demo3());
        }
    }
    ```

  - ```
    执行结果为：
    try 6
    finally 12
    main 6
    // 返回值仍然为6，也就是在finally中对i的赋值并未改变i的返回值
    ```

- 实例代码4：

  - ```java
    public class TestTryCatch {
    
        public static int demo4(){
            int i=0;
            try {
                i=6;
                System.out.println("try "+i);
                return i;
            }catch (Exception e){
                System.out.println(e.getMessage());
                return -1;
            }finally {
                i=12;
                System.out.println("finally "+i);
                return i;
            }
    
        }
    
        public static void main(String[] args) {
            System.out.println("main "+demo4());
        }
    }
    ```

  - ```
    执行结果为：
    try 6
    finally 12
    main 12
    // 在程序还未执行try中的return语句时就先执行了finally里面的return语句所以返回结果为12。
    ```

- 实例代码5：

  - ```java
    public class TestTryCatch {
    
        public static int demo5(){
            try {
                System.out.println("try");
                return printX();
            }catch (Exception e){
                System.out.println(e.getMessage());
                return -1;
            }finally {
                System.out.println("finally");
    
            }
    
        }
    
        public static int printX(){
            System.out.println("X");
            return 0;
        }
    
        public static void main(String[] args) {
            System.out.println("main "+demo5());
        }
    }
    ```

  - ```
    执行结果为：
    try
    X
    finally
    main 0
    // 程序顺序执行时先执行printX（）函数，此时得到返回值0并且将0保存到variable中对应的用于保存返回值的区域，此时程序在执行finally语句因为finally语句中没有return语句，所以程序将返回值区域的0返回给上一级函数。
    ```
