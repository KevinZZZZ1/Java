# Exception和Error

## `Exception`和`Error`的联系

- `Exception`和`Error`都继承了`Throwable`类，`Java`中只有Throwable类型的实例才能被抛出（throw）或捕获（catch）；

- `Error`和`RuntimeException`及其子类被称为未检查异常，其他异常被称为检查异常；

![avatar](F:\找工作\Java基础\Java\images\Excption和Error.png)

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

- 不管异常是否发生，`finally`块中的代码都会执行；

- 当`try`或`catch`中有`return`语句时，`finally`块仍然执行；

- `finally`是在执行完`return`之后执行的，所以函数的返回值在`return`执行完后就确定，`finally`块无法修改；

- `finally`块最好不要包含`return`，否则程序会提前退出；

- 实例代码：

  - ```java
    
    ```

  - 