# Linux IO模式和Java NIO

[TOC]



## 一、Linux IO模式以及select、poll、epoll

### 1.1Linux IO模式

#### 1.1.1概念说明

- **用户空间和内核空间**：

  现代操作系统都采用虚拟内存，对于32位操作系统而言，其寻址空间为4G（$2^{32}$）。操作系统的核心是内核，内核独立于其他应用程序，有着独立的内存空间以及底层硬件访问权限。为了保证内核安全，将虚拟内存分成两部分：`内核空间`和`用户空间`。

  对于Linux来说，最高的1G内存（0xC0000000~0xFFFFFFFF）共内核使用，成为内核空间；

  较低的3G内存（0x00000000~0xBFFFFFFF）供应用系统使用，称为用户空间；

- **进程切换**：

  进程切换指：为了控制进程的执行，内核有能力将CPU上正在执行的进程挂起，恢复之前挂起的某个进程；

  进程切换会进过下面一些列变化：

  - 保存处理器上下文，包括寄存器和程序计数器中的内容；
  - 更新PCB信息；
  - 将进程的PCB移入阻塞队列中；
  - 选择另一个线程执行，并更新其PCB；
  - 更新内存管理的数据结构；
  - 恢复处理的上下文；

  总之，**进程的切换很消耗资源**；

- **进程阻塞**：

  进程阻塞是指：正在执行的进程，所期待的事件未发生（例如：请求系统资源失败，等待某种操作完成等），由系统执行阻塞原语将进程变成阻塞状态，**进程阻塞是一种进程自身的主动行为，进入阻塞状态是不占用CPU资源的**；

- **文件描述符fd**：

  文件描述符（File Descriptor）是指：对一个文件引用的抽象化概念；文件描述符形式上是个非负整数，实际上**文件描述符是一个索引值，指向内核为该进程打开文件的记录表**，当程序打开文件或创建文件时，内核向进程返回一个文件描述符。

- **缓存I/O**：

  缓存I/O是指：**操作系统在将I/O数据缓存在文件系统页缓存中，即数据先被拷贝到操作系统内核缓冲区，然后再从操作系统内核的缓存区拷贝到应用程序的地址内存**；

  数据传输时这种在应用程序内存和内核内存多次拷贝的形式是十分占用CPU和内存的；



#### 1.1.2I/O模式

对于一次I/O访问（比如Read），数据先是被拷贝到操作系统内核的缓冲区中，再将操作系统内核缓冲区中的数据拷贝到应用程序的地址空间中；

- **阻塞I/O（blocking I/O）**：

  在Linux中，默认情况下所有socket都是阻塞的，典型的阻塞I/O如下图所示：

  ![avatar](F:\找工作\Java基础\Java\images\blockingIO.png)

  用户进程调用`recvfrom`，则内核就开始了IO的第一个阶段，即准备数据：

  - 数据被拷贝到操作系统缓存区的过程内核需要等待，比如对于网络IO来说，可能是还没有收到一个完整的UDP包；
  - 用户进程在调用`recvfrom`之后就会进入阻塞状态；

  当内核等到数据准备好之后，就会进行第二阶段，即从内核缓冲区拷贝数据到用户内存：

  - 内核会一直将数据拷贝到用户内存，直到拷贝结束；
  - 拷贝结束后，用户进程解除阻塞状态，开始处理数据；

  **所谓阻塞I/O就是指，在IO执行的两个阶段用户进程都是阻塞的**。

  

- **非阻塞I/O（nonblocking I/O）**：

  对于socket，可以通过设置将其变为非阻塞I/O，非阻塞I/O如下图所示：

  ![avatar](F:\找工作\Java基础\Java\images\nonblockingIO.png)

  当用户调用`recvfrom`，内核就进入了IO的第一个阶段，即准备数据：

  - 当数据未准备好时，内核直接返回一个error给用户进程；
  - 用户进程在调用`recvfrom`后立刻会得到返回结果，所以不会被阻塞；

  当操作系统缓冲区中数据准备好后，并且用户进程调用`recvfrom`，这时进入第二个阶段：

  - 内核会将缓冲区中的数据拷贝到用户内存中；
  - 用户进程会在该阶段阻塞，直到内核拷贝数据结束；

  **非阻塞I/O在数据准备阶段是不会被阻塞的，但是要不断主动询问内核数据是否准备好，在数据拷贝阶段会进入阻塞状态**；

  

- **I/O多路复用（I/O multiplexing）**：

  I/O多路复用就是我们平常说的`select`、`poll`、`epoll`。它们都是使用单线程处理多个网络连接的IO。**基本原理是：select、poll、epoll会不断轮询所有socket，当某个socket有数据到达了，通知用户线程**。I/O多路复用如下图所示：

  ![avatar](F:\找工作\Java基础\Java\images\IOMultiplexing.png)

  当用户调用`select`，内核就进入了IO的第一个阶段，即准备数据：

  - 当数据未准备好时，内核会监视所有`select`负责的socket，当其中有个socket中的数据准备完毕，`select`就会返回；
  - 用户进程在调用`select`后马上被阻塞，直到数据准备完毕；

  当操作系统缓冲区中数据准备好后，然后用户进程调用`recvfrom`，这时进入第二个阶段：

  - 内核会将缓冲区中的数据拷贝到用户内存中；
  - 用户进程会在该阶段阻塞，直到内核拷贝数据结束；

  **I/O多路复用的特点是，调用了两个系统调用（select和recvfrom），使得一个进程能同时等待多个文件描述符，一旦某个文件描述符进入就绪状态，`select`函数就能返回**。

  

- **信号驱动I/O**：

  ![avatar](F:\找工作\Java基础\Java\images\signalIO.png)

- **异步I/O**：

  异步I/O如下图所示：

  ![avatar](F:\找工作\Java基础\Java\images\asynchronousIO.png)

  当用户调用`system call`，内核就进入了IO的第一个阶段，即准备数据：

  - 当数据未准备好时，内核会等待数据准备完成；
  - 用户进程在调用`system call`后马上得到返回值，不会被阻塞；

  当操作系统缓冲区中数据准备好后，这时进入第二个阶段：

  - 内核会将缓冲区中的数据拷贝到用户内存中，并且在拷贝完成之后通知用户线程；
  - 用户进程会在得到内核通知时才开始处理数据；



#### 1.1.3同步I/O和异步I/O、阻塞和非阻塞的总结

- **各个I/O模型的比较**：

  ![avatar](F:\找工作\Java基础\Java\images\各种IO比较.png)

  

- **阻塞和非阻塞**：

  - 阻塞是指：用户线程在IO的第一阶段和第二阶段一直被阻塞；
  - 非阻塞是指：用户线程在IO的第一阶段会立即返回，不会被阻塞，而是在第二阶段才被阻塞；

- **同步I/O和异步I/O**：

  - 同步I/O：是指在进行I/O操作，即将数据由操作系统缓冲区拷贝到用户内存的过程中会出现阻塞的情况；
  - 异步I/O：是指当进程发起IO操作之后，不需要用户进程关注，直到内核给用户进程发送IO完成的消息，在整个过程中进程完全没有被阻塞；



### 1.2I/O多路复用：select、poll、epoll

首先由一点是需要明确的，`select`、`poll`、`epoll`都是属于I/O多路复用的，即一个进程可以监视多个文件描述符，一旦某个文件描述符就绪能够通知程序进行读写操作。**使用select、poll只能知道I/O事件发生了，但是却不知道是哪几个流，所以只能轮询所有流找出读出或写入数据，这样在处理多个流的时候会导致每次轮询的时间很长**；**而epoll则是会告知用户线程哪个流、发生了什么I/O事件，这时我们就能直接处理I/O**

#### 1.2.1select

```c++
int select(int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

- `select`函数监视的文件描述符分成3类：`writefds`、`readfds`、`exceptfds`。
- 用户调用`select`函数会阻塞，直到由文件描述符就绪或者超时；
- 当`select`函数返回后，可以遍历`fd_set`来找到就绪的文件描述符；
- 优点：
  - 几乎在所有平台上都有支持，有良好的跨平台能力；
- 缺点：
  - 单个线程监视的文件描述符数量有上限，在Linux上一般为1024；



#### 1.2.2poll

```c++
int poll(struct pollfd *fds, unsigned int nfd, int timeout);
```

```c++
struct pollfd{
    int fd; // 文件描述符
    short events; // 需要监视的事件
    short revents; // 发生的事件
}
```

- 不同于`select`使用3个指针表示3个`fdset`，`poll`使用一个`pollfd`的指针实现；
- `pollfd`结构中包含了要监视的事件和发生的事件；
- `poll`返回后，也需要轮询`pollfd`来获得就绪的描述符；
- 优点：
  - 没有文件描述符最大值的限制；



#### 1.2.3epoll

`epoll`是`select`和`poll`的增强版本，`epoll`更加灵活，使用一个文件描述符管理多个文件描述符，将用户关心的文件描述符的事件放到内核的事件表中，这样在用户内存和内核空间的copy只要一次；

```c++
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

- `epoll`操作过程需要三个接口，分别表示`epoll`过程；

  - int epoll_create(int size);

    **创建一个`epoll`对象**，其实`size`参数并没有什么用；

    当创建好`epoll`句柄之后，它也会占用一个`fd`值，所以在使用完`epoll`后必须调用`close()`关闭，否则`fd`可能被耗尽；

  - int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);

    **向`epoll`对象中添加或删除某个流的某个事件**：

    - ```c++
      epoll_ctl(epollfd, EPOLL_CTL_ADD, socket, EPOLLIN);//注册缓冲区非空事件，即有数据流入
      epoll_ctl(epollfd, EPOLL_CTL_DEL, socket, EPOLLOUT);//注册缓冲区非满事件，即流可以被写入
      ```

    - `epfd`是`epoll_create`的返回值；

    - `op`表示操作，用三个宏表示：添加（EPOLL_CTL_ADD）、删除（EPOLL_CTL_DEL）、修改（EPOLL_CTL_MOD）

    - `fd`需要监视的文件描述符；

    - `epoll_event`告诉内核需要监视的事件：

      ```c++
      struct epoll_event{
          __uint32_t events;
          epoll_data_t data;
      }
      //events可以是以下几个宏的集合：
      EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
      EPOLLOUT：表示对应的文件描述符可以写；
      EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
      EPOLLERR：表示对应的文件描述符发生错误；
      EPOLLHUP：表示对应的文件描述符被挂断；
      EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
      EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
      ```

      

  - int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);

    **等待直到注册的事件发生**；

    - `events`用来从内核得到事件集合；
    - `maxevents`告诉内核该`events`有多大；
    - `timeout`表示超时时间；

- 工作模式：`epoll`对于文件描述符的操作有两种模式：LT（Level trigger）模式、ET（Edge trigger）模式，LT是默认模式；

  - LT模式：当`epoll_wait`检测到描述符事件发生并将此事件通知应用程序，**应用程序可以不立即处理该事件**，下次调用`epoll_wait`时，会再次响应应用程序并通知此事件；
  - ET模式：当`epoll_wait`检测到描述符事件发生并将此事件通知应用程序，**应用程序必须立即处理该事件**，如果不处理，下次调用`epoll_wait`时，不会再次响应应用程序并通知此事件；



## 二、Java NIO





