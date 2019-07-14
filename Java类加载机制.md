# Java类加载机制

[TOC]



Java程序要想能够执行，要经历以下步骤：

- 将`.java`源文件编译成`.class`的文件，该文件中保存着Java程序对应的字节码指令以及要使用的数据；
- 当需要使用某个类时，虚拟机将会加载该类的`.class`文件，并创建相应的Class对象，将`.class`文件加载到内存，这个过程称为类加载；
- 然后JVM再对字节码进行链接；
- 链接完毕后执行初始化工作；
- 然后再交由执行子系统执行；

大概流程如下图所示：

![avatar](F:\找工作\Java基础\Java\images\Java程序执行流程图.png)

一个类的生命周期可以用下图表示：

![avatar](F:\找工作\Java基础\Java\images\类的生命周期.png)

其中验证、准备、解析三个部分又被合起来称为链接；

以下套类加载过程，包括加载、验证、准备、解析、初始化这五个阶段，其中加载、准备、初始化这三个阶段比较重要；

## 加载

- 类的加载是指，将类的`.class`文件中的二进制数据读入内存中，将其放入运行时数据区域的方法区内，并且再堆中创建一个`java.lang.Class`对象，作为方法区数据结构的入口；
  - 通过全限定类名获取类文件的字节数组，可以来自本地，jar包，网络；
  - 在方法区/元空间保存类的描述信息，静态属性；
  - 在JVM堆中创建一个对应的`java.lang.Class`对象；
- 什么时候要对类进行加载：Java虚拟机有预加载功能，也就是说不用等到某个类“首次使用时”才加载该类；
- Java中类的加载使用类加载器完成，Java默认提供了三个类加载器：
  - **Bootstrap ClassLoader**：负责加载Java基础类，主要是`%JRE_HOME%/lib/`目录下的`rt.jar`、`resource.jar`、`charsets.jar`等，出于安全考虑，Bootstrap启动类加载器只加载包名为java、javax、sun等开头的类，由C++实现，是JVM的一部分；
  - **Extension ClassLoader**：负责加载Java扩展类，主要是`%JRE_HOME%/lib/ext`目录下的jar文件，由Java实现；
  - **Application ClassLoader**：负责加载当前应用的ClassPath中的所有类，由Java实现；

![avatar](F:\找工作\Java基础\Java\images\JVM架构图.png)

### 类加载器详解

首先先看一段代码：

```java
package com.kevin;

public class ClassPath {

    public static void main(String[] args) {
        System.out.println("Bootstrap ClassLoader path: ");
        System.out.println(System.getProperty("sun.boot.class.path"));
        System.out.println("----------------------------");

        System.out.println("Extension ClassLoader path: ");
        System.out.println(System.getProperty("java.ext.dirs"));
        System.out.println("----------------------------");

        System.out.println("App ClassLoader path: ");
        System.out.println(System.getProperty("java.class.path"));
        System.out.println("----------------------------");
    }

}
```

这里先给出执行结果，然后再分析原因，执行结果为：

```java
Bootstrap ClassLoader path: 
null
----------------------------
Extension ClassLoader path: 
null
----------------------------
App ClassLoader path: 
F:\IdeaProjects\Javajichu\out\production\Javajichu
----------------------------
```

`URLClassLoader`，`AppClassLoader`等类加载器都是继承了顶层抽象类`ClassLoader`

#### ClassLoader类

在`ClassLoader`中有几个比较重要的方法：

- `loadClass(String)`：该方法是指定要加载的类名，loadClass()方法是ClassLoader类自己实现的，该方法中的逻辑就是双亲委派模式的实现，其中的resolve参数代表是否生成class对象的同时进行解析相关操作；

  ```java
  protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 先从缓存查找该class对象，找到就不用重新加载
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        //如果找不到，则委托给父类加载器去加载
                        c = parent.loadClass(name, false);
                    } else {
                    //如果没有父类，则委托给启动加载器去加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }
  
                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // 如果都没有找到，则通过自定义实现的findClass去查找并加载
                    c = findClass(name);
  
                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {//是否需要在加载时进行解析
                resolveClass(c);
            }
            return c;
        }
    }
  ```

  当类加载请求到来时，先从缓存中查找该类对象，如果存在直接返回，如果不存在则交给该类加载去的父加载器去加载，倘若没有父加载则交给顶级启动类加载器去加载，最后倘若仍没有找到，则使用findClass()方法去加载

- `findClass(String)`：findClass()方法是在loadClass()方法中被调用的，当loadClass()方法中父加载器加载失败后，则会调用自己的findClass()方法来完成类加载，自定义的类加载逻辑写在findClass()方法中；

  ClassLoader类中并没有实现findClass()方法的具体代码逻辑，取而代之的是抛出ClassNotFoundException异常，同时应该知道的是findClass方法通常是和defineClass方法一起使用的；

  ```java
  protected Class<?> findClass(String name) throws ClassNotFoundException {
          throw new ClassNotFoundException(name);
  }
  ```

- `defineClass(byte[] b, int off, int len)`：defineClass()方法是用来将byte字节流解析成JVM能够识别的Class对象，一般情况下，在自定义类加载器时，会直接覆盖ClassLoader的findClass()方法并编写加载规则，取得要加载类的字节码后转换成流，然后调用defineClass()方法生成类的Class对象；

### 双亲类加载机制

- Java类加载机制使用双亲委派模式：即如果一个类加载器收到了类加载请求，它并不会自己先去加载，而是把这个请求委托给父类的加载器去执行，如果父类加载器还存在其父类加载器，则进一步向上委托，依次递归，请求最终将到达顶层的启动类加载器，如果父类加载器可以完成类加载任务，就成功返回，倘若父类加载器无法完成此加载任务，子加载器才会尝试自己去加载；如下图所示：

  ![avatar](F:\找工作\Java基础\Java\images\双亲类加载机制.png)

- 双亲委派模式的优势：Java类随着它的类加载器一起具备了一种带有优先级的层次关系，

  - 通过这种层级关可以避免类的重复加载，当父亲已经加载了该类时，就没有必要子ClassLoader再加载一次；
  - 其次是考虑到安全因素，java核心api中定义类型不会被随意替换，假设通过网络传递一个名为java.lang.Integer的类，通过双亲委托模式传递到启动类加载器，而启动类加载器在核心Java API发现这个名字的类，发现该类已被加载，并不会重新加载网络传递的过来的java.lang.Integer，而直接返回已加载过的Integer.class，这样便可以防止核心API库被随意篡改；

  

- 类加载器之间的关系：

  - 启动类加载器，由C++实现，没有父类；

  - 拓展类加载器(ExtClassLoader)，由Java语言实现，父类加载器为null；

  - 系统类加载器(AppClassLoader)，由Java语言实现，父类加载器为ExtClassLoader；
  - 自定义类加载器，父类加载器肯定为AppClassLoader；

  我们可以通过代码来分析上面的关系：

  ```java
  class FileClassLoader extends  ClassLoader{
      private String rootDir;
  
      public FileClassLoader(String rootDir) {
          this.rootDir = rootDir;
      }
      // 编写获取类的字节码并创建class对象的逻辑
      @Override
      protected Class<?> findClass(String name) throws ClassNotFoundException {
         //...省略逻辑代码
      }
  	//编写读取字节流的方法
      private byte[] getClassData(String className) {
          // 读取类文件的字节
          //省略代码....
      }
  }
  
  public class ClassLoaderTest {
  
      public static void main(String[] args) throws ClassNotFoundException {
         
  			 FileClassLoader loader1 = new FileClassLoader(rootDir);
  			
  			  System.out.println("自定义类加载器的父加载器: "+loader1.getParent());
  			  System.out.println("系统默认的AppClassLoader: "+ClassLoader.getSystemClassLoader());
  			  System.out.println("AppClassLoader的父类加载器: "+ClassLoader.getSystemClassLoader().getParent());
  			  System.out.println("ExtClassLoader的父类加载器: "+ClassLoader.getSystemClassLoader().getParent().getParent());
      }
  }
  
  ```

  执行结果为：

  ```
  自定义类加载器的父加载器: sun.misc.Launcher$AppClassLoader@29453f44
  系统默认的AppClassLoader: sun.misc.Launcher$AppClassLoader@29453f44
  AppClassLoader的父类加载器: sun.misc.Launcher$ExtClassLoader@6f94fa3e
  ExtClassLoader的父类加载器: null
  ```

  我们自定义了一个FileClassLoader，接着在main方法中，通过`ClassLoader.getSystemClassLoader()`获取到系统默认类加载器；

  上面执行结果的原因是：sun.misc.Launcher类是Java程序的入口，通过查看Launcher源码我们可以得出答案：

  ```java
  public Launcher() {
          // 首先创建扩展类加载器
          ClassLoader extcl;
          try {
              extcl = ExtClassLoader.getExtClassLoader();
          } catch (IOException e) {
              throw new InternalError(
                  "Could not create extension class loader");
          }
  
          // Now create the class loader to use to launch the application
          try {
  	        //再创建AppClassLoader并把extcl作为父加载器传递给AppClassLoader
              loader = AppClassLoader.getAppClassLoader(extcl);
          } catch (IOException e) {
              throw new InternalError(
                  "Could not create application class loader");
          }
  
          //设置线程上下文类加载器
          Thread.currentThread().setContextClassLoader(loader);
  //省略其他没必要的代码......
          }
      }
  ```

  

  通过查看Lancher构造函数的源码可知，Lancher初始化时首先会创建ExtClassLoader类加载器，然后再创建AppClassLoader并把ExtClassLoader传递给它作为父类加载器；

  ```java
  //Launcher中创建ExtClassLoader
  extcl = ExtClassLoader.getExtClassLoader();
  
  //getExtClassLoader()方法
  public static ExtClassLoader getExtClassLoader() throws IOException{
  
    //........省略其他代码 
    return new ExtClassLoader(dirs);                     
    // .........
  }
  
  //构造方法
  public ExtClassLoader(File[] dirs) throws IOException {
     //调用父类构造URLClassLoader传递null作为parent
     super(getExtURLs(dirs), null, factory);
  }
  ```

  显然ExtClassLoader的父类为null，而AppClassLoader的父加载器为ExtClassLoader，所有自定义的类加载器其父加载器只会是AppClassLoader；



### 自定义类加载器

- 实现自定义类加载器需要继承ClassLoader或者URLClassLoader：
  - 继承ClassLoader则需要自己重写findClass()方法并编写加载逻辑；
  - 继承URLClassLoader则可以省去编写findClass()方法以及class文件加载转换成字节码流的代码；
- 编写自定义类加载器的意义：
  - 当class文件不在ClassPath路径下，默认系统类加载器无法找到该class文件，在这种情况下我们需要实现一个自定义的ClassLoader来加载特定路径下的class文件生成class对象；
  - 当一个class文件是通过网络传输并且可能会进行相应的加密操作时，需要先对class文件进行相应的解密后再加载到JVM内存中，这种情况下也需要编写自定义的ClassLoader并实现相应的逻辑；
  - 当需要实现热部署功能时(一个class文件通过不同的类加载器产生不同class对象从而实现热部署功能)，需要实现自定义ClassLoader的逻辑；
- 在JVM中表示两个Class对象是否为同一个类对象存在两个必要条件：
  - 类的完整类名必须一致，包括包名；
  - 加载这个类的ClassLoader(指ClassLoader实例对象)必须相同；
- 自定义File类加载器：
- 自定义网络类加载器：







## 验证

- 验证阶段主要是确保 Class 文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全；
- 主要包括四种验证，文件格式验证，元数据验证，字节码验证，符号引用验证。



## 准备

- 准备阶段为类变量（`static`修饰的变量）分配内存并设置初始值，使用的是方法区的内存；

- 实例变量不会在这阶段分配内存，它会在对象实例化时随着对象一起被分配在堆中，实例化不是类加载的一个过程，类加载发生在所有实例化操作之前，并且类加载只进行一次，实例化可以进行多次；

- 初始值一般为 0 值，例如下面的类变量 value 被初始化为 0 而不是 123：

  ```java
  public static int value = 123;
  ```

- 如果类变量是常量，那么它将初始化为表达式所定义的值而不是 0。例如下面的常量 value 被初始化为 123 而不是 0：

  ```java
  public static final int value = 123;
  ```

  

## 解析

- 将常量池的符号引用替换为直接引用的过程；其中解析过程在某些情况下可以在初始化阶段之后再开始，这是为了支持 Java 的动态绑定；
- 符号引用就是一组符号来描述目标，可以是任何字面量，而直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。有类或接口的解析，字段解析，类方法解析，接口方法解析；



## 初始化

- 初始化阶段才是真正执行Java程序代码，初始化阶段是虚拟机执行类构造器 <clinit>() 方法的过程。在准备阶段，类变量已经赋过一次系统要求的初始值，而在初始化阶段，根据程序员通过程序制定的主观计划去初始化类变量和其它资源；

- 对于初始化阶段来说，其发生时机是**首次主动使用**，有下面六种主动初始化的方式：

  - **创建对象实例，使用new创建对象时，会引发初始化，前提是该类没有被初始化过**；

  - **调用类的静态属性或给静态属性赋值**；
  - **调用类的静态方法**；
  - **通过class对象反射创建对象**；
  - **初始化一个类的子类，使用子类时要先初始化父类**；
  - **Java虚拟机启动时被标记的启动类，也就是main方法中的类**；

- 在主动初始化的过程中有以下几点需要注意的：

  - 在同一个类加载器下只能初始化类一次，这是因为类加载的最终结果就是在堆中有一个唯一的Class对象，Class对象的唯一性就使得类只需要初始化一次即可；
  - 调用编译时可以确定的静态变量不会对类进行初始化；
  - 调用编译时不能确定的静态变量会对类进行初始化;
  - 如果类中存在初始化语句，按语句出现顺序执行；

  例1：

  ```java
  public class Test1{
      public static void main(String[] args){
          System.out.println(FinalTest.x);
      }
  }
  class FinalTest{
      public static final int x = 6/3;
      static{
          System.out.println("FinalTest static block");
      }
  }
  ```

  执行结果为：

  ```java
  2
  原因是，FinalTest类中的常量x在编译器就能确定值，这个时候就不会造成类初始化
  ```

  例2：

  ```java
  public class Test2{
      public static void main(String[] args){
          System.out.println(FinalTest2.x);
      }
  }
  class FinalTest2{
      public static final int x = new Random().nextInt(100);
      static{
          System.out.println("FinalTest2 static block");
      }
  }
  ```

  执行结果为：

  ```java
  FinalTest2 static block
  66
  原因是，FinalTest2类中的常量x在编译器不能确定值，是个运行时常量，这个时候就需要进行类初始化；
  ```

  

- 类的初始化步骤：

  - 没有父类情况：
    1. 类的静态属性；
    2. 类的静态代码块；
    3. 类的非静态属性；
    4. 类的非静态代码块；
    5. 构造函数；
  - 有父类情况：
    1. 父类的静态属性；
    2. 父类的静态代码块；
    3. 子类的静态属性；
    4. 子类的静态代码块；
    5. 父类的非静态属性；
    6. 父类的非静态代码块；
    7. 父类构造函数；
    8. 子类的非静态属性；
    9. 子类的非静态代码块；
    10. 子类构造函数；

  其中静态代码块和静态属性是等价的，他们是按照代码顺序执行的

- 例1：

  ```java
  public class Singleton{
      
      private static Singleton singleton = new Singleton();
      public static int count1;
      public static int count2=0;
      
      private Singleton(){
          count1++;
          count2++;
      }
      
      public static Singleton getSingleton(){
          return singleton;
      }
      
  }
  ```

  ```java
  public class TestSingleton{
      public static void main(String[] args){
          Singleton singleton = Singleton.getSingleton();
          System.out.println("count1= "+Singleton.count1);
          System.out.println("count2= "+Singleton.count2);
      }
  }
  ```

  执行结果为：

  ```
  count1= 1
  count2= 0
  ```

  原因是：

  - 执行`TestSingleton`第一句的时候，因为我们没有对`Singleton`类进行加载和链接，所以我们首先需要对它进行加载和链接操作，在链接操作中会将静态变量赋值为默认值，即：
    `singleton=null`
    `count1=0`
    `count2=0`

  - 加载和链接完毕之后，我们再进行初始化工作。初始化工作是从上往下依次执行的（注意这个时候还没有调用`getSinleton()`方法），
    - 先执行`private static Singleton singleton = new Singleton();`这时会调用`Singleton`的构造函数对`count1`、`count2`的值进行修改，此时`count1==1`、`count2==1`；
    - 再执行`public static int count1`；
    - 最后执行`public static int count2=0;`此时`count2==0`；
  - 初始化完毕之后我们就要调用静态方法`Singleton.getSingleton()`，此时返回的`Singleton`已经初始化了；



## 参考文献

https://juejin.im/entry/5b6d38e0e51d45195a718fd2

https://www.jianshu.com/p/b6547abd0706

https://www.jianshu.com/p/8c8d6cba1f8e

https://github.com/CyC2018/CS-Notes

https://blog.csdn.net/justloveyou_/article/details/72217806

https://blog.csdn.net/javazejian/article/details/73413292