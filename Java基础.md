# 综述

- java的基本原理，记录面试时容易考的。

- 记录平时容易混淆的。

- 内容与《JDK》中的有部分重复



参考：

《java核心技术面试精讲-40讲》



# 面向对象设计

> ####  基本要素：封装、继承、多态。

- 封装的目的是隐藏事务内部的实现细节，以便提高安全性和简化编程。封装提供了合理的边界，避免外部调用者接触到内部的细节。
- 继承是代码复用的基础机制
- 多态，你可能立即会想到重写(override)和重载(overload)、向上转型。简单说，重写是父子类中相同名字和参数的方法，不同的实现;重载则是相同名字的方法，但是不同的 参数，本质上这些方法签名是不一样的



> #### S.O.L.I.D原则

进行面向对象编程，掌握基本的设计原则是必须的，我今天介绍最通用的部分，也就是所谓的S.O.L.I.D原则。

- 单一职责(Single Responsibility)，类或者对象最好是只有单一职责，在程序设计中如果发现某个类承担着多种义务，可以考虑进行拆分。
- 开关原则(Open-Close, Open for extension, close for modifcation)，设计要对扩展开放，对修改关闭。换句话说，程序设计应保证平滑的扩展性，尽量避免因为新增同类功能而修改已有实现，这样可以少产出些回归(regression)问题。
- 里氏替换(Liskov Substitution)，这是面向对象的基本要素之一，进行继承关系抽象时，凡是可以用父类或者基类的地方，都可以用子类替换。
- 接口分离(Interface Segregation)，我们在进行类和接口设计时，如果在一个接口里定义了太多方法，其子类很可能面临两难，就是只有部分方法对它是有意义的，这就破坏 了程序的内聚性。对于这种情况，可以通过拆分成功能单一的多个接口，将行为进行解耦。在未来维护中，如果某个接口设计有变，不会对使用其他接口的子类构成影响。
- 依赖反转(Dependency Inversion)，实体应该依赖于抽象而不是实现。也就是说高层次模块，不应该依赖于低层次模块，而是应该基于抽象。实践这一原则是保证产品代码之 间适当耦合度的法宝。



# 注解

见《JVM》相关章节



# 线程

内容与JDK那边的有重复，大部分都写在那边了，记录几个问题



> #### 线程到底是什么以及Java底层实现方式。

从操作系统的角度，可以简单认为，线程是系统调度的最小单元，一个进程可以包含多个线程，作为任务的真正运作者，有自己的栈(Stack)、寄存器(Register)、本地存储 (Thread Local)等，但是会和进程内其他线程共享文件描述符、虚拟地址空间等。

在具体实现中，线程还分为内核线程、用户线程，Java的线程实现其实是与虚拟机相关的。对于我们最熟悉的Sun/Oracle JDK，其线程也经历了一个演进过程，基本上在Java 1.2之后，JDK已经抛弃了所谓的Green Thread，也就是用户调度的线程，现在的模型是一对一映射到操作系统内核线程。



> #### 从线程生命周期的状态开始展开，那么在Java编程中，有哪些因素可能影响线程的状态呢?

主要有: 

- 线程自身的方法，除了start，还有多个join方法，等待线程结束;yield是告诉调度器，主动让出CPU;另外，就是一些已经被标记为过时的resume、stop、suspend之类，据我所知，在JDK最新版本中，destory/stop方法将被直接移除。 

- 基类Object提供了一些基础的wait/notify/notifyAll方法。如果我们持有某个对象的Monitor锁，调用wait会让当前线程处于等待状态，直到其他线程notify或者notifyAll。所以，本质上是提供了Monitor的获取和释放的能力，是基本的线程间通信方式。

- 并发类库中的工具，比如CountDownLatch.await()会让当前线程进入等待状态，直到latch被基数为0，这可以看作是线程间通信的Signal。



# 解释执行与编译执行

相关内容在《JVM中也有很多重复》



**解释执行**

由解释器根据输入的数据当场执行而不生成任何目标程序。解释执行程序是高级语言翻译程序的一种，它将源语言（如VASIC）书写的源程序作为输入，解释一句后就提交给计算机执行一句，并不生成目标程序。

**编译执行**
先将源代码编译成目标语言之后通过连接程序生成目标程序进行执行。简单来说就是，先需要对源程序进行一个编译，生成一个目标文件，计算机再对这个目标程序进行执行。
这是一类很重要的语言处理程序，它把高级语言（如Pascal、C等）源程序作为输入，进行翻译转换，产生出机器语言的目标程序，然后再让计算机去执行这个目标程序，得到计算结果。编译程序工作时，先分析，后综合，从而得到目标程序。所谓分析，是指词法分析和语法分析；所谓综合是指代码优化，存储分配和代码生成。

```
解释执行：将编译好的字节码一行一行地翻译为机器码执行。

编译执行：以方法为单位，将字节码一次性翻译为机器码后执行。
```



> #### Java是解释执行吗？

对于“Java是解释执行”这句话，这个说法不太准确。我们开发的Java的源代码，首先通过Javac编译成为字节码（bytecode），然后，在运行时，通过 Java虚拟机（JVM）内嵌的解释器将字节码转换成为最终的机器码。

但是常见的JVM，比如我们大多数情况使用的Oracle JDK提供的Hotspot JVM，都提供了JIT（Just-In-Time）编译器，也就是通常所说的动态编译器，JIT能够在运行时将热点代码编译成机器码，这种情况下部分热点代码就属于编译执行，而不是解释执行了。

目前主流的JVM 都是混合模式（-Xmixed），即解释运行 和编译运行配合使用。



# 接口和抽象类有什么区别

> #### 接口

接口是对行为的抽象，它是抽象方法的集合，利用接口可以达到**API定义和实现分离的目的**。接口，不能实例化;不能包含任何非常量成员，任何field都是隐含着public static final的意义;同时，没有非静态方法实现，也就是说要么是抽象方法，要么是静态方法。

内部结构：
    jdk7：接口只有常量和抽象方法，无构造器
    jdk8：接口增加了 默认方法 和 静态方法，无构造器
    jdk9：接口允许 以 private 修饰的方法，无构造器



> #### 抽象类

抽象类是不能实例化的类，用abstract关键字修饰class，**其目的主要是代码重用**。除了不能实例化，形式上和一般的Java类并没有太大区别，可以有一个或者多个抽象方法，也可以没有抽象方法。抽象类大多用于抽取相关Java类的共用方法实现或者是共同成员变量，然后通过继承的方式达到代码复用的目的。

Java类实现interface使用implements关键词，继承abstract class则是使用extends关键词。



- 接口可以被类多实现（被其他接口多继承），抽象类只能被单继承。
- 接口中没有 `this` 指针，没有构造函数，不能拥有实例字段（实例变量）或实例方法，无法保存 状态（state），抽象方法中可以。
- 抽象类不能在 java 8 的 lambda 表达式中使用。
- 从设计理念上，接口反映的是 “like-a” 关系，抽象类反映的是 “is-a” 关系。



# Java IO

## 输入输出流

在java io的中，我们经常会提到“输入流”、“输出流”等概念。所谓**“流”，就是一种抽象的数据的总称，它的本质是能够进行传输。**

1. 按照“流”的数据流向，可以将其化分为：**输入流**和**输出流**。
2. 按照“流”中处理数据的单位，可以将其区分为：**字节流**和**字符流**。在java中，字节是占1个Byte，即8位；而字符是占2个Byte，即16位。而且，需要的是，java的字节是有符号类型，而字符是无符号类型！



**以字节为单位的输入流**

![以字节为单位的输入流的框架图](截图/Java基础/以字节为单位的输入流的框架图.jpg)

1. InputStream 是以字节为单位的输入流的超类。InputStream提供了read()接口从输入流中读取字节数据。
2. ByteArrayInputStream 是字节数组输入流。它包含一个内部缓冲区，该缓冲区包含从流中读取的字节；通俗点说，它的内部缓冲区就是一个字节数组，而ByteArrayInputStream本质就是通过**字节数组**来实现的。
3. PipedInputStream 是管道输入流，它和PipedOutputStream一起使用，能实现多线程间的管道通信。
4. FilterInputStream 是过滤输入流。它是DataInputStream和BufferedInputStream的超类。
5. DataInputStream 是数据输入流。它是用来装饰其它输入流，它“允许应用程序以与机器无关方式从底层输入流中读取基本 Java 数据类型”。
6. BufferedInputStream 是缓冲输入流。它的作用是为另一个输入流添加缓冲功能。
7. File 是“文件”和“目录路径名”的抽象表示形式。关于File，注意两点：
        a), File不仅仅只是表示文件，它也可以表示目录！
        b), File虽然在io保重定义，但是它的超类是Object，而不是InputStream。
8. FileDescriptor 是“文件描述符”。它可以被用来表示开放文件、开放套接字等。
9. FileInputStream 是文件输入流。它通常用于对文件进行读取操作。
10. ObjectInputStream 是对象输入流。它和ObjectOutputStream一起，用来提供对“基本数据或对象”的持久存储。



**以字节为单位的输出流**

![以字节为单位的输出流](截图/Java基础/以字节为单位的输出流.jpg)



1. OutputStream 是以字节为单位的输出流的超类。OutputStream提供了write()接口从输出流中读取字节数据。

2. ByteArrayOutputStream 是字节数组输出流。写入ByteArrayOutputStream的数据被写入一个 byte 
   数组。缓冲区会随着数据的不断写入而**自动增长**。可使用 toByteArray() 和 toString() 获取数据。

3. PipedOutputStream 是管道输出流，它和PipedInputStream一起使用，能实现多线程间的管道通信。

4. FilterOutputStream 是过滤输出流。它是DataOutputStream，BufferedOutputStream和PrintStream的超类。

5. DataOutputStream 是数据输出流。它是用来**装饰**其它输出流，它“允许应用程序以与机器无关方式向底层写入基本 Java 数据类型”。

6. BufferedOutputStream 是缓冲输出流。它的作用是为另一个输出流添加缓冲功能。

7. PrintStream 是打印输出流。它是用来装饰其它输出流，能为其他输出流添加了功能，使它们能够方便地打印各种数据值表示形式。

8. FileOutputStream 是文件输出流。它通常用于向文件进行写入操作。

9. ObjectOutputStream 是对象输出流。它和ObjectInputStream一起，用来提供对“基本数据或对象”的持久存储。



**以字符为单位的输入流**

![以字符为单位的输入流框架图](截图/Java基础/以字符为单位的输入流框架图.jpg)

1. Reader 是以字符为单位的输入流的超类。它提供了read()接口来取字符数据。
2. CharArrayReader 是字符数组输入流。它用于读取字符数组，它继承于Reader。操作的数据是以字符为单位！
3. PipedReader 是字符类型的管道输入流。它和PipedWriter一起是可以通过管道进行线程间的通讯。在使用管道通信时，必须将PipedWriter和PipedReader配套使用。
4. FilterReader 是字符类型的过滤输入流。
5. BufferedReader 是字符缓冲输入流。它的作用是为另一个输入流添加缓冲功能。
6. InputStreamReader 是字节转字符的输入流。它是字节流通向字符流的桥梁：它使用指定的 charset 读取字节并将其解码为字符。
7. FileReader 是字符类型的文件输入流。它通常用于对文件进行读取操作。



**字节流与字符流可以互相转化**

![字节转换为字符流的框架图](截图/Java基础/字节转换为字符流的框架图.jpg)

1. FileReader继承于InputStreamReader，而InputStreamReader依赖于InputStream。具体表现在InputStreamReader的构造函数是以InputStream为参数。我们传入InputStream，在InputStreamReader内部通过转码，将字节转换成字符。
2. FileWriter继承于OutputStreamWriter，而OutputStreamWriter依赖于OutputStream。具体表现在OutputStreamWriter的构造函数是以OutputStream为参数。我们传入OutputStream，在OutputStreamWriter内部通过转码，将字节转换成字符。



**以字符为单位的输出流**

![以字符为单位的输出流的框架图](截图/Java基础/以字符为单位的输出流的框架图.jpg)

1. Writer 是以字符为单位的输出流的超类。它提供了write()接口往其中写入数据。
2. CharArrayWriter 是字符数组输出流。它用于读取字符数组，它继承于Writer。操作的数据是以字符为单位！
3. PipedWriter 是字符类型的管道输出流。它和PipedReader一起是可以通过管道进行线程间的通讯。在使用管道通信时，必须将PipedWriter和PipedWriter配套使用。
4. FilterWriter 是字符类型的过滤输出流。
5. BufferedWriter 是字符缓冲输出流。它的作用是为另一个输出流添加缓冲功能。
6. OutputStreamWriter 是字节转字符的输出流。它是字节流通向字符流的桥梁：它使用指定的 charset 将字节转换为字符并写入。
7. FileWriter 是字符类型的文件输出流。它通常用于对文件进行读取操作。
8. PrintWriter 是字符类型的打印输出流。它是用来装饰其它输出流，能为其他输出流添加了功能，使它们能够方便地打印各种数据值表示形式。





> #### 字节流与字符流的区别

- 字节流在操作时本身不会用到缓冲区（内存），是文件本身直接操作的
- 而字符流在操作时使用了缓冲区，通过缓冲区再操作文



[java io系列01之 "目录"](https://www.cnblogs.com/skywang12345/p/io_01.html)



## IO模型 

> https://www.bilibili.com/video/BV1hK4y1N7Ky?p=82

	在读取文件时会涉及I/O操作，在发送和接收网络数据时也会涉及IO操作。所有涉及到I/O的操作都要直接或者间接的经过内核程序。网络传输数据，首先是内核先接收到数据，然后内核将数据拷贝到用户态中供应用进程使用。这里使用网络I/O进行演示。



**概念解释**

- 阻塞与非阻塞指的是当不能进行读写（网卡满时的写/网卡空的时候的读）的时候，I/O 操作立即返回还是阻塞；**阻塞/非阻塞关注的是程序（线程）等待消息通知时的状态**。当在进行阻塞操作时，当前线程会处于阻塞状态，无法从事其他任务，只有当条件就绪才能继续。而非阻塞则是不管IO操作是否结束，直接返回，相应操作在后台继续处理



- 同步异步指的是，当数据已经ready的时候，读写操作是同步读还是异步读。**同步与异步主要是从消息通知机制角度来说的。**`异步的概念和同步相对`。当一个同步调用发出后，`调用者要一直等待返回消息（结果）通知后`，才能进行后续的执行；当一个异步过程调用发出后，调用者不需要立刻得到返回消息（结果）。`实际处理这个调用的部件在完成后，通过状态、通知和回调来通知调用者`。



但是需要注意了，`同步非阻塞形式实际上是效率低下的`，想象一下你一边打着电话一边还需要抬头看到底队伍排到你了没有。如果把打电话和观察排队的位置看成是程序的两个操作的话，这个程序需要在这两种不同的行为之间来回的切换，效率可想而知是低下的；而`异步非阻塞形式却没有这样的问题`，因为打电话是你(等待者)的事情，而通知你则是柜台(消息触发机制)的事情，程序没有在两种不同的操作中来回切换。

----------------------



所有的系统I/O都分为两个阶段：等待就绪和操作。举例来说，读函数，分为等待系统可读和真正的读；同理，写函数分为等待网卡可以写和真正的写。

需要说明的是等待就绪的阻塞是不使用CPU的，是在“空等”；而真正的读写操作的阻塞是使用CPU的，真正在"干活"，而且这个过程非常快，属于memory copy，带宽通常在1GB/s级别以上，可以理解为基本不耗时。

![常见IO模型](截图/Java基础/常见IO模型.jpg)

**5种IO模型**

### 同步阻塞I/O
	最传统的一种IO模型，即在读写数据过程中会发生阻塞现象。
	在应用进程通过内核调用recvfrom（）函数，其系统调用直到数据包到达且被复制到应用进程的缓冲区或者发生错误时才会返回，在此期间会一直等待。

![阻塞式IO](截图/Java基础/阻塞式IO.png)



在Linux中，首先可以创建一个socket，可以拿到对应的文件描述符fd号，之后调用 `bind(fd,port)` 把这个socket绑定到具体的端口上，之后再调用listen方法，对端口进行监听，之后调用accept方法，尝试拿到客户端的fd，此时是堵塞在这里的。当有客户端连上来之后，可以拿到客户端的fd，之后尝试从fd6中读取他的输出流，recvfrom也是堵塞的，此时如果有另外的一个客户端尝试连接，那么是无法连上来的

<img src="截图/Java基础/BI模型.png" alt="image-20210507204544683" style="zoom:67%;" />



<img src="截图/Java基础/每线程对应一个客户端模型.png" alt="image-20210507204940504" style="zoom:50%;" />

BIO的方式，每个线程对应一个客户端。




### 同步非阻塞I/O 	

	应用系统还是调用recvfrom，但是他不会阻塞与此，而是不断的去轮询的是否有数据准备好，如果没有准备好，就直接返回一个EWOULDBLOCK错误。这个过程通常被称之为轮询。轮询检查内核数据，直到数据准备好，再拷贝数据到进程，进行数据处理。需要注意，拷贝数据整个过程，进程仍然是属于阻塞的状态。
	
	与同步阻塞I/O相比，如果数据未准备好，不会一直阻塞与此，而是直接返回错误，接收到错误之后，就可以干点别的事，这是他的优点。但是缺点也很明显，任务完成的响应延迟增大了。因为很可能在两次轮询之间，socketfd就处于read状态了，所以导致整体的吞吐量下降了。





<img src="截图/Java基础/同步非阻塞IO.png" alt="同步非阻塞IO" style="zoom:67%;" />





### I/O复用

由于同步非阻塞方式需要不断主动轮询，轮询占据了很大一部分过程，轮询会消耗大量的CPU时间，而 “后台” 可能有多个任务在同时进行，人们就想到了循环查询多个任务的完成状态，只要有任何一个任务完成，就去处理它。如果轮询不是进程的用户态，而是有人帮忙就好了。`那么这就是所谓的 “IO 多路复用”`。UNIX/Linux 下的 select、poll、epoll 就是干这个的（epoll 比 poll、select 效率高，做的事情是一样的）。

 `IO多路复用有两个特别的系统调用select、poll、epoll函数`。select调用是内核级别的，select轮询相对非阻塞的轮询的区别在于---`前者可以等待多个socket，能实现同时对多个IO端口进行监听`，当其中任何一个socket的数据准好了，`就能返回进行可读`，`然后进程再进行recvform系统调用，将数据由内核拷贝到用户进程，当然这个过程是阻塞的`。select或poll调用之后，会阻塞进程，与blocking IO阻塞不同在于，`此时的select不是等到socket数据全部到达再处理, 而是有了一部分数据就会调用用户进程来处理`。如何知道有一部分数据到达了呢？`监视的事情交给了内核，内核负责数据到达的处理。也可以理解为"非阻塞"吧`。

 `I/O复用模型会用到select、poll、epoll函数，这几个函数也会使进程阻塞，但是和阻塞I/O所不同的的，这两个函数可以同时阻塞多个I/O操作`。而且可以同时对多个读操作，多个写操作的I/O函数进行检测，直到有数据可读或可写时（注意不是全部数据可读或可写），才真正调用I/O操作函数。

对于多路复用，也就是轮询多个socket。`多路复用既然可以处理多个IO，也就带来了新的问题，多个IO之间的顺序变得不确定了`，当然也可以针对不同的编号。具体流程，如下图所示：



   <img src="截图/Java基础/IO多路复用.png" alt="IO多路复用" style="zoom: 67%;" />



#### Select、Poll、Epoll :star:

> [Select、Poll、Epoll详解](https://www.jianshu.com/p/722819425dbd)
>
> [一道搜狗面试题：IO多路复用中select、poll、epoll之间的区别](https://www.jianshu.com/p/e47e53e82908)



select、poll、epoll之间的区别：

|     \      |                       select                       |                       poll                       |                            epoll                             |
| :--------: | :------------------------------------------------: | :----------------------------------------------: | :----------------------------------------------------------: |
|  操作方式  |                        遍历                        |                       遍历                       |                             回调                             |
|  底层实现  |                        数组                        |                       链表                       |                            哈希表                            |
|   IO效率   |      每次调用都进行线性遍历，时间复杂度为O(n)      |     每次调用都进行线性遍历，时间复杂度为O(n)     | 事件通知方式，每当fd就绪，系统注册的回调函数就会被调用，将就绪fd放到rdllist里面。时间复杂度O(1) |
| 最大连接数 |             1024（x86）或 2048（x64）              |                      无上限                      |                            无上限                            |
|   fd拷贝   | 每次调用select，都需要把fd集合从用户态拷贝到内核态 | 每次调用poll，都需要把fd集合从用户态拷贝到内核态 |  调用epoll_ctl时拷贝进内核并保存，之后每次epoll_wait不拷贝   |

**select**

select本质上是通过设置或者检查存放fd标志位的数据结构来进行下一步处理。这样所带来的缺点是：

1. 单个进程可监视的fd数量被限制，即能监听端口的大小有限。

2. 对socket进行扫描时是线性扫描，即采用轮询的方法，效率较低：

   当套接字比较多的时候，每次select()都要通过遍历FD_SETSIZE个Socket来完成调度，不管哪个Socket是活跃的，都遍历一遍。这会浪费很多CPU时间。如果能给套接字注册某个回调函数，当他们活跃时，自动完成相关操作，那就避免了轮询，这正是epoll与kqueue做的。

3. 需要维护一个用来存放大量fd的数据结构，这样会使得用户空间和内核空间在传递该结构时复制开销大

<img src="截图/Java基础/io-select.png" alt="image-20210507210109484" style="zoom:50%;" />

当有m个客户端需要监听的时候，把所有需要监听的fd丢给系统的select方法，让kernel去帮你监听文件的状态，从而避免用户程序的不断轮询。

每次调用select方法会返回可以读的m个fd，此时再去调用recvfrom方法。但是读数据还是用户程序自己读，所以还是一个同步模型。

他的问题是

1. 每次都需要把要遍历的fd传给select方法
2. 在内核中把所有要遍历的fd都遍历一遍



**poll**

poll本质上和select没有区别，它将用户传入的数组拷贝到内核空间，然后查询每个fd对应的设备状态，如果设备就绪则在设备等待队列中加入一项并继续遍历，如果遍历完所有fd后没有发现就绪设备，则挂起当前进程，直到设备就绪或者主动超时，被唤醒后它又要再次遍历fd。这个过程经历了多次无谓的遍历。

它没有最大连接数的限制，原因是它是基于链表来存储的，但是同样有一个缺点：

- 大量的fd的数组被整体复制于用户态和内核地址空间之间，而不管这样的复制是不是有意义。
- poll还有一个特点是“水平触发”，如果报告了fd后，没有被处理，那么下次poll时会再次报告该fd。



**epoll**

// todo 待完善

epoll有EPOLLLT和EPOLLET两种触发模式，LT是默认的模式，ET是“高速”模式。LT模式下，只要这个fd还有数据可读，每次 epoll_wait都会返回它的事件，提醒用户程序去操作，而在ET（边缘触发）模式中，它只会提示一次，直到下次再有数据流入之前都不会再提示了，无论fd中是否还有数据可读。

所以在ET模式下，read一个fd的时候一定要把它的buffer读光，也就是说一直读到read的返回值小于请求值，或者遇到EAGAIN错误。还有一个特点是，epoll使用“事件”的就绪通知方式，通过epoll_ctl注册fd，一旦该fd就绪，内核就会采用类似callback的回调机制来激活该fd，epoll_wait便可以收到通知。



> #### epoll为什么要有EPOLLET触发模式？

如果采用EPOLLLT模式的话，系统中一旦有大量你不需要读写的就绪文件描述符，它们每次调用epoll_wait都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率.。而采用EPOLLET这种边沿触发模式的话，当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。

如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你！！！这种模式比水平触发效率高，系统不会充斥大量你不关心的就绪文件描述符

<img src="截图/Java基础/io-epoll.png" alt="image-20210507212144531" style="zoom:50%;" />

epoll会在内存中开辟一块空间，用来存储要监听的fd，所以就不用像select一样每次都传一遍了。在有事件到达之后，把事件放到右边蓝色的内存中去，每次用户都只是去查看一下，哪些fd已经就绪了。

- 在调用epoll_create的时候，返回这个绿色空间对应的fd，之后调用epoll_ctl_add添加对fd5也就是上面的socket的 accept 事件的监听到fd8中去。
- 之后调用epoll_wait fd8，让用户程序去等待fd8，他其实就会去等待蓝色区的结果。
- 当下方来了一个圆圈（client）之后，fd5的accept事件被触发了，那么就把fd5放到右边去。
- 那么就可以从fd5中拿到这个新建立的客户端fd6
- 拿到客户端对应的socket的fd之后，还需要建立对这个fd6的读/写的事件监听

黄色的区域中有个问号，指的是绿色区域监听的内容是发现，并怎么移动蓝色的区域的。与select一样都是把所有的fd都轮询遍历一遍吗？这里是使用事件驱动的方式。他是怎么实现的呢？

假设，网卡，系统会专门为网卡驱动开辟一块DMA（直接内存访问）空间，当网卡收到数据之后，会把数据放到DMA中。之后网卡这个硬件向CPU发起硬中断，cpu收到中断之后，会根据中断号去中断向量表中找到对应的callback，去DMA中读到数据，然后就可以把这个fd移动到右边的区域中去了。





### 信号驱动I/O

首先开启套接字信号驱动I/O功能，并通过系统调用sigaction执行一个信号处理函数，此时系统继续运行，并不会阻塞。

当数据准备就绪时，就为该进程生成一个SIGIO信号，通过信号回调通知，通知应用进程调用recvfrom来读取数据。



<img src="截图/Java基础/信号驱动IO.png" alt="信号驱动IO" style="zoom:67%;" />



### 异步非阻塞

相对于同步IO，异步IO不是顺序执行。`用户进程进行aio_read系统调用之后，无论内核数据是否准备好，都会直接返回给用户进程，然后用户态进程可以去做别的事情`。等到socket数据准备好了，内核直接复制数据给进程，`然后从内核向进程发送通知`。`IO两个阶段，进程都是非阻塞的`。

<img src="截图/Java基础/异步非阻塞IO.png" alt="异步非阻塞IO" style="zoom:67%;" />

### 总结

通过上面的图片，可以发现non-blocking IO和asynchronous IO的区别还是很明显的。`在non-blocking IO中，虽然进程大部分时间都不会被block，但是它仍然要求进程去主动的check`，并且当数据准备完成以后，也需要进程主动的再次调用recvfrom来将数据拷贝到用户内存。而asynchronous IO则完全不同。`它就像是用户进程将整个IO操作交给了他人（kernel）完成，然后他人做完后发信号通知`。在此期间，`用户进程不需要去检查IO操作的状态，也不需要主动的去拷贝数据`。



阻塞I/O：同步阻塞
非阻塞I/O：同步（轮询）非阻塞
I/O多路复用：同步阻塞（不过可以同时监听多个socket状态，效率高了）
信号驱动I/O：异步非阻塞
异步I/O：真正意义上的异步非阻塞（上面的都只是数据准备阶段，这个是数据准备和数据处理阶段）  





链接：

[聊聊Linux 五种IO模型](https://www.jianshu.com/p/486b0965c296)
[聊聊同步、异步、阻塞与非阻塞](https://www.jianshu.com/p/aed6067eeac9)



## JAVA NIO

上面更多的是站在操作系统角度的分析，从java角度来看。（这里其实更应该直接聊netty）

JAVA NIO有两种解释：一种叫非阻塞IO（Non-blocking I/O），另一种也叫新的IO（New I/O），其实是同一个概念。它是一种同步非阻塞的I/O模型，也是I/O多路复用的基础。

NIO的主要组成部分

- Buffer，高效的数据容器，除了布尔类型，所有原始数据类型都有相应的Buffer实现。 
- Channel，类似在Linux之类操作系统上看到的文件描述符，是NIO中被用来支持批量式IO操作的一种抽象。 File或者Socket，通常被认为是比较高层次的抽象，而Channel则是更加操作系统底层的一种抽象，这也使得NIO得以充分利用现代操作系统底层机制，获得特定场景的性能优化，例如，DMA（Direct Memory Access）等。不同层次的抽象是相互关联的，我们可以通过Socket获取Channel，反之亦然。
-  Selector，是NIO实现多路复用的基础，它提供了一种高效的机制，可以检测到注册在Selector上的多个Channel中，是否有Channel处于就绪状态，进而实现了单线程对 多Channel的高效管理。 Selector同样是基于底层操作系统机制，不同模式、不同版本都存在区别，例如，在最新的代码库里，相关实现如下： Linux上依赖于epoll。 Windows上NIO2（AIO）模式则是依赖于iocp。
-  Chartset，提供Unicode字符串定义，NIO也提供了相应的编解码器等，例如，通过下面的方式进行字符串到ByteBufer的转换



其中，主要需要了解Buffer相关内容

基本属性： 

- capacity，它反映这个Buffer到底有多大，也就是数组的长度。 
- position，要操作的数据起始位置。 
- limit，相当于操作的限额。在读取或者写入时，limit的意义很明显是不一样的。比如，读取操作时，很可能将limit设置到所容纳数据的上限；而在写入时，则会设置容量或容量 以下的可写限度。 
- mark，记录上一次position的位置，默认是0，算是一个便利性的考虑，往往不是必须的。



**类型**：

Direct Buffer：如果我们看Buffer的方法定义，你会发现它定义了isDirect()方法，返回当前Buffer是否是Direct类型。这是因为Java提供了堆内和堆外（Direct）Buffer，我们可以以它的allocate或者allocateDirect方法直接创建。 

MappedByteBuffer：它将文件按照指定大小直接映射为内存区域，当程序访问这个内存区域时将直接操作这块儿文件数据，省去了将数据从内核空间向用户空间传输的损耗。我 们可以使用FileChannel.map创建MappedByteBuffer，它本质上也是种Direct Buffer



**垃圾收集**：

- 大多数垃圾收集过程中，都不会主动收集Direct Buffer，它的垃圾收集过程，就是基于Cleaner（一个内部实现）和幻象引用 （PhantomReference）机制，其本身不是public类型，内部实现了一个Deallocator负责销毁的逻辑。对它的销毁往往要拖到full GC的时候，所以使用不当很容易导致OutOfMemoryError。 
- 对于Direct Buffer的回收，有几个建议： 
  - 在应用程序中，显式地调用System.gc()来强制触发。
  -  另外一种思路是，在大量使用Direct Buffer的部分框架中，框架会自己在程序中调用释放方法，Netty就是这么做的。
  -  重复使用Direct Buffer。



**跟踪和诊断Direct Bufer内存占用**

因为通常的垃圾收集日志等记录，并不包含Direct Buffer等信息。在JDK 8之后的版本，我们可以方便地使用Native Memory Tracking（NMT）特性来进行诊断

```cmd
-XX:NativeMemoryTracking={summary|detail}
// 打印NMT信息
jcmd <pid> VM.native_memory detail
// 进行baseline，以对比分配内存变化
jcmd <pid> VM.native_memory baseline
// 进行baseline，以对比分配内存变化
jcmd <pid> VM.native_memory detail.dif
```







## 零拷贝

零拷贝是指避免在用户态(User-space) 与内核态(Kernel-space) 之间来回拷贝数据的技术。

<img src="截图/Java基础/传统网络io模型.png" alt="640?wx_fmt=gif" style="zoom: 100%;" />

1. read()调用导致上下文从用户态切换到内核态。内核通过sys_read()（或等价的方法）从文件读取数据。DMA引擎执行第一次拷贝：从文件读取数据并存储到内核空间的缓冲区。
2. 请求的数据从内核的读缓冲区拷贝到用户缓冲区，然后read()方法返回。read()方法返回导致上下文从内核态切换到用户态。现在待读取的数据已经存储在用户空间内的缓冲区。至此，完成了一次IO的读取过程。
3. send()调用导致上下文从用户态切换到内核态。第三次拷贝数据从用户空间重新拷贝到内核空间缓冲区。但是，这一次，数据被写入一个不同的缓冲区，一个与目标套接字相关联的缓冲区。
4. send()系统调用返回导致第四次上下文切换。当DMA引擎将数据从内核缓冲区传输到协议引擎缓冲区时，第四次拷贝是独立且异步的。
   

![640?wx_fmt=gif](截图/Java基础/io零拷贝流程.png)

NIO的零拷贝由transferTo方法实现。transferTo方法将数据从FileChannel对象传送到可写的字节通道（如Socket Channel等）。在transferTo方法内部实现中，由native方法transferTo0来实现，它依赖底层操作系统的支持。在UNIX和Linux系统中，调用这个方法会引起sendfile()系统调用，实现了数据直接从内核的读缓冲区传输到套接字缓冲区，避免了用户态(User-space) 与内核态(Kernel-space) 之间的数据拷贝。

使用NIO零拷贝，流程简化为两步：

1. transferTo方法调用触发DMA引擎将文件上下文信息拷贝到内核读缓冲区，接着内核将数据从内核缓冲区拷贝到与套接字相关联的缓冲区。
2. DMA引擎将数据从内核套接字缓冲区传输到协议引擎（第三次数据拷贝）。



> #### Java有几种文件拷贝方式？

- 利用java.io类库，直接为源文件构建一个FileInputStream读取，然后再为目标文件构建一个FileOutputStream，完成写入工作。

  使用输入输出流进行读写时，实际上是进行了多次上下文切换，比如应用读取数据时，先在内核态将数据从磁盘读取到内核缓存，再切换到用户态将数据从内核缓存读取到用户缓存

- 利用java.nio类库提供的transferTo或transferFrom方法实现。

  在Linux和Unix上，则会使用到零拷贝技术，数据传输并不需要用户态参与，省去了上下文切换的开销和不必要的内存拷贝，进而可能提高应用 拷贝性能。



# 直接内存映射

Linux提供的mmap系统调用, 它可以将一段用户空间内存映射到内核空间, 当映射成功后, 用户对这段内存区域的修改可以直接反映到内核空间；同样地， 内核空间对这段区域的修改也直接反映用户空间。正因为有这样的映射关系, 就不需要在用户态(User-space)与内核态(Kernel-space) 之间拷贝数据， 提高了数据传输的效率，这就是内存直接映射技术。



**NIO的直接内存映射**

JDK1.4加入了NIO机制和直接内存，目的是**防止Java堆和Native堆之间数据复制**带来的性能损耗，此后NIO可以使用Native的方式直接在 Native堆分配内存。



**实际使用场景**

文件的拷贝复制，socket对拷，文件压缩，文件下载。







# 参数传递

一道经典的面试题目是通过一个函数交换两个对象。

这里主要考察的是对形参与实参的理解。值传递与引用传递的理解，与函数调用的原理。

> 值传递与引用传递

java的方法传递都是分类型的，如果是简单对象或者说是基本对象则直接是值传递，如果是复杂类则是传递地址的引用。这里明确一下概念，**java参数传递只有一种情况，那就是值传递，一般说的"引用传递"，在实际中传递的不过是引用对象的地址值**。如果是地址，那么还有一步隐藏的地址到目标内存的转换。也就是说

**值传递：**此传递过程就是将实参的值复制一份传递到函数中，这样如果在函数中对该值（形参的值）进行了操作将不会影响实参的值。

**引用传递：**引用传递就是将实参的地址复制一份传递到函数中。形参和实参的地址相同，指向同一块内存地址，操作的其实都是同一份数据，所以如果在函数中对该值（形参的值）进行了操作将会影响实参的值。



> 形参和实参

**形式参数：**用于定义方法的时候使用的参数，是用来接收调用者传递的参数的。 形参只有在方法被调用的时候，虚拟机才会分配内存单元，在方法调用结束之后便会释放所分配的内存单元。 因此,形参只在方法内部有效，所以针对引用对象的改动也无法影响到方法外。

**实际参数：**用于调用时传递给方法的参数。实参在传递给别的方法之前是要被预先赋值的。



> 明确一种说法

假如是引用传递，引用指向的是目标的内存区。那题目是相当于传入了2个引用，要求交换，则把这两个引用指向的地址内存对换，由于形参与实参都是指向同一个地址，而此地址指向的内存被交换了，所以完成了两个参数的交换。

var a(实参) -> *p -> 0x00aaa

var a(形参) -> *p -> 0x00aaa



var b(实参) -> *q -> 0x00bbb

var b(形参) -> *q -> 0x00bbb

形参a与实参a同时指向一个指针，此指针指向一个地址。那么在方法内部的时候，只交换*p -> 0x00aaa后半段

var a(实参) -> *p -> 0x00bbb

var a(形参) -> *p -> 0x00bbb



var b(实参) -> *q -> 0x00aaa

var b(形参) -> *q -> 0x00aaa

由于前半段没变，所以完成了交换。



在java中

1. 引用传递就是地址传递，指的就是传一个地址，不存在传一个指针类型的变量，此变量指向真实地址，中间那一层*p其实是不存在的。
2. 复杂对象分配在堆上，局部变量分配在栈上。变量标识符指向栈中的一块内存，内存里面存的是堆中的地址。函数调用时，形参与实参在栈上分配了两个空间，传值的时候相当于把两个栈空间中的地址复制了一下，没有给“指针对象”再分配一块空间了。
3. 地址里存的就是值了，在栈中的地址到堆中地址是一一对应的也与内存块是一一对应的，不存在一个（指针）地址指向不同的内存。



主要是引用传值，传的是地址，指针的概念就是等于地址，没有指针对象做一个桥接的二次跳转了。

C中的*p，其实是一个对象，只不过是指针类型，函数调用的时候引用传递不是传 *p对象，而是传地址。



参考：

[从一道面试题看你对java的理解程度](https://segmentfault.com/a/1190000018621882)
[Java基础【二】 - 值传递和引用传递](https://segmentfault.com/a/1190000016597411)
[Java中的值传递和地址传递](https://blog.csdn.net/qq_35109096/article/details/81105320)
[C语言指针之形参和实参](https://blog.csdn.net/aann518/article/details/53035419)



# 对象引用

不同的引用类型，主要体现的是对象不同的可达性（reachable）状态和对垃圾收集的影响。 

- 所谓强引用（"Strong" Reference），就是当一个对象可以有一个或多个线程可以不通过各种引用访问到的情况。只要还有强引用指向一个对象，就能表明对象还“活着”，垃圾收集器不会碰这种对象。对于一个普通的对象，如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应（强）引用赋值为null，就是可以被垃圾收集的了，当然具体回收时机还是要看垃圾收集策略。 

- 软引用（SoftReference），是一种相对强引用弱化一些的引用，就是当我们只能通过软引用才能访问到对象的状态。可以让对象豁免一些垃圾收集，**只有当JVM认为内存不足时，才会去试图回收软引用指向的对象**。JVM会确保在抛 出OutOfMemoryError之前，清理软引用指向的对象。

- 弱引用（WeakReference）并不能使对象豁免垃圾收集，仅仅是提供一种访问在弱引用状态下对象的途径，就是无法通过强引用或者软引用访问，只能通过弱引用访问时的状态。 

  目前用来处理ThreadLocal对象的key的内存回收。

- 幻象引用（虚引用），你不能通过它访问对象，get()方法返回总是null，也就是没有强、软、弱引用关联，并且finalize过了，只有幻象引用指向这个对象的时候。

  这东西主要是用来管理java的直接内存，在虚引用对象被回收的时候，会把他丢到队列里面去，相当于一个通知，在具体的应用中，直接内存被回收的时候，会丢到队列中，jvm监听这个队列，然后特殊处理直接内存的回收。



## 作用

- 软引用

  通常用来实现内存敏感的缓存，如果还有空闲内存，就可以暂时保留缓存，当内存不足时清理掉，这样就保证了使用缓存的同时，不会耗尽内存。 

- 弱引用

  可以用来构建一种没有特定约束的关系，比如，维护一种非强制性的映射关系，如果试图获取时对象还在，就使用它，否则重现实例化。

  它同样是很多缓存实现的选择。

- 幻象引用

  提供了一种确保对象被finalize以后，做某些事情的机制，比如，通常用来做所谓的Post-Mortem清理机制，Java平台自身Cleaner机制等，也有人利用幻象引用监控对象的创建和销毁。



## 引用队列（ReferenceQueue）

我们在创建各种引用并关联到响应对象时，可以选择是否需要关联引用队列，JVM会在特定时机将引用enqueue到队列里，我们可以从队列里获取引用，进行相关后续逻辑。尤其是幻象引用，get方法只返回null，如果再不指定引用队列，基本就没有意义了。



## Reachability Fence

通过底层API来达到强引用的效果。

为什么需要这种机制？按照Java语言规范，如果一个对象没有指向强引用，就符合垃圾收集的标准，有些时候，对象本身并没有强引用，但是也许它的部分 属性还在被使用。类似与函数式编程，或者直接调用后没有显示的分配一个对象。

```java
new Resource().action()
```

在JDK源码中，reachabilityFence大多使用在Executors或者类似新的HTTP/2客户端代码中，大部分都是异步调用的情况。编程中，可以将需 要reachability保障的代码段利用try-finally包围起来，在finally里明确声明对象强可达。



# Java8 

## 新特性

Java8 新增了非常多的特性，我们主要讨论以下几个：

- **Lambda 表达式** − Lambda 允许把函数作为一个方法的参数（函数作为参数传递到方法中）。
- **方法引用（双冒号）** −  方法引用提供了非常有用的语法，可以直接引用已有Java类或对象（实例）的方法或构造器。与lambda联合使用，方法引用可以使语言的构造更紧凑简洁，减少冗余代码。
- **默认方法** − 默认方法就是一个在接口里面有了一个实现的方法。
- **新工具** − 新的编译工具，如：Nashorn引擎 jjs、 类依赖分析器jdeps。
- **Stream API** −新添加的Stream API（java.util.stream） 把真正的函数式编程风格引入到Java中。
- **Date Time API** − 加强对日期与时间的处理。
- **Optional 类** − Optional 类已经成为 Java 8 类库的一部分，用来解决空指针异常。
- **Nashorn, JavaScript 引擎** −  Java 8提供了一个新的Nashorn javascript引擎，它允许我们在JVM上运行特定的javascript应用。



## 函数式接口

> [必看：深入学习Java8中的函数式接口](https://www.sohu.com/a/123958799_465959)

就是interface里有且仅有一个抽象方法，但是可以有多个非抽象方法的接口，可以有default方法，也可以有Object类的public方法。

Java8里关于函数式接口的包是java.util.function，里面全部是函数式接口。主要包含几大类：Function、Predicate、Supplier、Consumer和*Operator（没有Operator接口，只有类似BinaryOperator这样的接口）



## default method

> [Java 8 默认方法（Default Methods）](https://www.cnblogs.com/sidesky/p/9287710.html)

默认方法是在接口中的方法签名前加上了 `default` 关键字的实现方法。

`ClassA` 类并没有实现 `InterfaceA` 接口中的 `foo` 方法，`InterfaceA` 接口中提供了 `foo` 方法的默认实现，因此可以直接调用 `ClassA` 类的 `foo` 方法。

> #### 为什么要有默认方法?

在 java 8 之前，接口与其实现类之间的耦合度太高了（tightly  coupled），当需要为一个接口添加方法时，所有的实现类都必须随之修改。默认方法解决了这个问题，它可以为接口添加新的方法，而不会破坏已有的接口的实现。这在 lambda 表达式作为 java 8 语言的重要特性而出现之际，为升级旧接口且保持向后兼容（backward  compatibility）提供了途径。

> #### 默认方法的继承

- 不覆写默认方法，直接从父接口中获取方法的默认实现。
- 覆写默认方法，这跟类与类之间的覆写规则相类似。
- 覆写默认方法并将它重新声明为抽象方法，这样新接口的子类必须再次覆写并实现这个抽象方法。



> #### 默认方法的多继承

Java 使用的是单继承、多实现的机制，为的是避免多继承带来的调用歧义的问题。当接口的子类同时拥有具有相同签名的方法时，就需要考虑一种解决冲突的方案。

在 `ClassA` 类中，它实现的 `InterfaceA` 接口和 `InterfaceB` 接口中的方法不存在歧义，可以直接多实现。

在 `ClassB` 类中，它实现的 `InterfaceB` 接口和 `InterfaceC` 接口中都存在相同签名的 `foo` 方法，需要手动解决冲突。覆写存在歧义的方法，并可以使用 **`InterfaceName.super.methodName();`** 的方式手动调用需要的接口默认方法。



> #### 接口与抽象类

当接口继承行为发生冲突时的另一个规则是，类的方法声明优先于接口默认方法，无论该方法是具体的还是抽象的。



> #### 其他注意点

- `default` 关键字只能在接口中使用（以及用在 `switch` 语句的 `default` 分支），不能用在抽象类中。

- 接口默认方法不能覆写 `Object` 类的 `equals`、`hashCode` 和 `toString` 方法。

- 接口中的静态方法必须是 `public` 的，`public` 修饰符可以省略，`static` 修饰符不能省略。

  



## lambda





# Java新版本

## 9

- G1成为默认垃圾回收器
- 集合加强



## *11

- 本地变量类型推断
- 字符串加强
- 集合加强
- 在模块方面移除Java EE以及CORBA模块
- 在JVM方面引入了实验性的ZGC
- 在API方面正式提供了HttpClient类替换原有的HttpURLConnection。



## 12

- 对switch进行了增强，除了使用statement还可以使用expression



## 13

- Java13主要新增了如下特性
  - 350:    Dynamic CDS Archives
  - 351:    ZGC: Uncommit Unused Memory
  - 353:    Reimplement the Legacy Socket API
  - 354:    Switch Expressions (Preview)
  - 355:    Text Blocks (Preview)
- 语法层面，改进了Switch Expressions，新增了Text Blocks，二者皆处于Preview状态；API层面主要使用NioSocketImpl来替换JDK1.0的PlainSocketImpl
- GC层面则改进了ZGC，以支持Uncommit Unused Memory



# 排序算法

> Java提供的默认排序算法是什么？

这个问题本身就是有点陷阱的意味，因为需要区分是Arrays.sort()还是Collections.sort() （底层是调用Arrays.sort()）；什么数据类型；多大的数据集（太小的数据集，复杂排序是没必要的，Java会直接进行二分插入排序）等。 

- 对于原始数据类型，目前使用的是所谓**双轴快速排序**（Dual-Pivot QuickSort），是一种改进的快速排序算法，早期版本是相对传统的快速排序。 
- 而对于对象数据类型，目前则是使用**TimSort**，思想上也是一种归并和二分插入排序（binarySort）结合的优化排序算法。TimSort并不是Java的独创，简单说它的思路是查找数据集中已经排好序的分区（这里叫run），然后合并这些分区来达到排序的目的。 
- 另外，Java 8引入了并行排序算法（直接使用parallelSort方法），这是为了充分利用现代多核处理器的计算能力，底层实现基于fork-join框架（专栏后面会对fork-join进行相对详细的介绍），当处理的数据集比较小的时候，差距不明显，甚至还表现差一点；但是，当数据集增长到数万或百万以上时，提高就非常大了，具体还是取决于处理器和系统环境。



> 8. 对比Vector、ArrayList、LinkedList有何区别？.pdf



# 反射 — 动态代理

> 谈谈Java反射机制，动态代理是基于什么原理？ ——  《java核心技术面试精讲-40讲 — 06动态代理是基于什么原理？》
>
> 07 | JVM是如何实现反射的？  —— 《深入拆解 Java 虚拟机 07 JVM是如何实现反射的？》

这个题目有点诱导的嫌疑，会下意识地以为动态代理就是利用反射机制实现的，这么说不算错但稍微有些不全面。

实现动态代理的方式很多，比如：

- JDK自身提供的动态代理，就是主要利用了**反射机制**。
- 比如利用传说中更高性能的**字节码操作**机制，类似ASM、cglib（基于ASM）、Javassist等。



最简单的通常的使用，我们可以通过 Class 对象枚举该类中的所有方法，我们还可以通过 `Method.setAccessible`绕过 Java 语言的访问权限，在私有方法所在类之外的地方调用该方法。例如

```java
Method method = test.class.getMethod("main1");
Annotation[] annotations = test.class.getAnnotations();
method.invoke("name");
```

这里观察一下反射具体是怎么实现的。

java.lang.reflect.Method#invoke

```java
public Object invoke(Object obj, Object... args)
    throws IllegalAccessException, IllegalArgumentException,
       InvocationTargetException
{
    // 权限检查
    if (!override) {
        if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
            Class<?> caller = Reflection.getCallerClass();
            checkAccess(caller, clazz, obj, modifiers);
        }
    }
    // 委派给MethodAccessor来处理具体任务
    MethodAccessor ma = methodAccessor;             // read volatile
    if (ma == null) {
        ma = acquireMethodAccessor();
    }
    return ma.invoke(obj, args);
}
```

`MethodAccessor `是一个接口，它有两个已有的具体实现：一个通过本地方法来实现反射调用，另一个则使用了委派模式。

反射调用先是调用了Method.invoke，然后进入委派实现（DelegatingMethodAccessorImpl），再然后进入本地实现（NativeMethodAccessorImpl），最后到达目标方法。 



> #### 为什么反射调用还要采取委派实现作为中间层？直接交给本地实现不可以么？

Java 的反射调用机制还设立了另一种动态生成字节码的实现（下称动态实现），直接使用 invoke 指令来调用目标方法。之所以采用委派实现，便是为了能够在本地实现以及动态实现中切换。

sun.reflect.ReflectionFactory#newMethodAccessor

```
public MethodAccessor newMethodAccessor(Method var1) {
    checkInitted();
    if (noInflation && !ReflectUtil.isVMAnonymousClass(var1.getDeclaringClass())) {
    		// 使用asm等一些动态字节码操作工具
        return (new MethodAccessorGenerator()).generateMethod(var1.getDeclaringClass(), var1.getName(), var1.getParameterTypes(), var1.getReturnType(), var1.getExceptionTypes(), var1.getModifiers());
    } else {
    		// 使用本地实现
        NativeMethodAccessorImpl var2 = new NativeMethodAccessorImpl(var1);
        DelegatingMethodAccessorImpl var3 = new DelegatingMethodAccessorImpl(var2);
        var2.setParent(var3);
        return var3;
    }
}
```

动态实现和本地实现相比，其运行效率要快上 20 倍 。这是因为**动态实现无需经过 Java 到 C++ 再到 Java 的切换**。

但由于生成字节码十分耗时，仅调用一次的话，反而是本地实现要快 上 3 到 4 倍 [3]。 考虑到许多反射调用仅会执行一次，Java 虚拟机设置了一个阈值 15（可以通过 - Dsun.reflect.inflationThreshold= 来调整），当某个反射调用的调用次数在 15 之下时，采用本地实现；当达到 15 时，便开始动态生成字节码，并将委派实现的委派对象切换至动态实现， 这个过程我们称之为 Inflation。

在默认情况下，方法的反射调用为委派实现，委派给本地实现来进行方法调用。在调用超过 15 次之后，委派实现便会将委派对象切换至动态实现。这个动态实现的字节码是自动生成的，它将直接使用 invoke 指令来调用目标方法。



> #### 反射调用的开销

一个简单的调用例子

```java
  public static void main(String[] args) throws Exception {
    Class<?> klass = Class.forName("Test");
    Method method = klass.getMethod("target", int.class);
    method.invoke(null, 0);
  }
```

Class.forName 和 Class.getMethod。

Class.forName 会调用本地方法，Class.getMethod 则会遍历该类的公有方法。如果没有匹配到，它还将遍历父类的公有方法。所以能缓存就缓存一下。



Method.invoke

- 第一，由于 Method.invoke 是一个变长参数方法，在字节码层面它的最后一个参数会是 Object 数组。Java 编译器会在方法调用处生成一个长度为传入参数数量的 Object 数组，并将传入参数一一存储进该数组中。 
- 第二，由于 Object 数组不能存储基本类型，Java 编译器会对传入的基本类型参数进行自动装箱。
- 由于 Java 虚拟机的关于上述调用点的类型 profile无法同时记录这么多个类，因此可能造成所测试的反射调用没有被内联的情况。
- 除了没有内联之外，另外一个原因是逃逸分析不再起效（这两个原因慎用）

这两个操作除了带来性能开销外，还可能占用堆内存，使得 GC 更加频繁。



## Class.forName 与 ClassLoader区别

在java中Class.forName()和ClassLoader都可以对类进行加载。ClassLoader就是遵循**双亲委派模型**最终调用启动类加载器的类加载器，实现的功能是“通过一个类的全限定名来获取描述此类的二进制字节流”，获取到二进制流后放到JVM中。Class.forName()方法实际上也是调用的CLassLoader来实现的。

```java
		@CallerSensitive
    public static Class<?> forName(String className)
                throws ClassNotFoundException {
        Class<?> caller = Reflection.getCallerClass();
        return forName0(className, true, ClassLoader.getClassLoader(caller), caller);
    }
```

最后调用的方法是forName0这个方法，在这个forName0方法中的第二个参数被默认设置为了true，这个参数代表是否对加载的类进行初始化，设置为true时会类进行初始化，代表会执行类中的静态代码块，以及对静态变量的赋值等操作。



- **Class.forName会执行静态代码块，而obj.getClass.getClassLoader不会执行任何构造方法、静态代码块和非静态代码块**

- **class.newInstance()时与使用无参构造方法new对象一样，会自动调用非静态代码块和无参构造方法** 



> [在Java的反射中，Class.forName和ClassLoader的区别](https://www.cnblogs.com/jimoer/p/9185662.html)



> #### ClassLoader.getSystemClassLoader（）和Thread.currentThread().getContextClassLoader()

ClassLoader.getSystemClassLoader方法无论何时均会返回ApplicationClassLoader,其只加载classpath下的class文件。



> #### Class.forName(com.mysql.jdbc.Driver) 加载数据库驱动

Class.forName 遵循双亲委派机制。

从源码可以看出是使用 ClassLoader.getClassLoader(caller) 类加载器进行加载的，默认是使用的用户类加载器，还是符合双亲委派机制的。

为什么必须要破坏?
DriverManager::getConnection 方法需要根据参数传进来的 url 从所有已经加载过的 Drivers 里找到一个合适的 Driver 实现类去连接数据库.
Driver 实现类在第三方 jar 里, 要用 AppClassLoader 加载. 而 DriverManager 是 rt.jar 里的类, 被 BootstrapClassLoader 加载, DriverManager 没法用 BootstrapClassLoader 去加载 Driver 实现类, 所以只能破坏双亲委派模型, 用它下级的 AppClassLoader 去加载 Driver.

如何破坏的?
\> SPI 机制
Driver 具体的实现类 jar 包在 META-INF/services/java.sql.Driver 文件里写上自己 Driver 实现类的全限定名, 比如 `org.postgresql.Driver` , JDK8 的 DriverManager 在 static 代码块里(JDK14 是在第一次 getConnection 时)会用 ServiceLoader 获取所有的 Driver 并使用 Thread.currentThread().getContextClassLoader() 获取到的 AppClassLoader 加载该类, Driver 实现类的 static 代码块会去调用 DriverManager::registerDriver 将自己注册到 DriverManager 里.



> [Java 关于 Class.forName(com.mysql.jdbc.Driver) 未遵守了双亲委派模型这件事有什么比较好资料么?](https://www.v2ex.com/t/670286)



# SPI

SPI ，全称为 Service Provider Interface，是一种服务发现机制。它通过在ClassPath路径下的META-INF/services文件夹查找文件，自动加载文件里所定义的类。

这一机制为很多框架扩展提供了可能，比如在Dubbo、JDBC中都使用到了SPI机制。

使用例子：

```java
public class Test {
    public static void main(String[] args) {    
        Iterator<SPIService> providers = Service.providers(SPIService.class);
        ServiceLoader<SPIService> load = ServiceLoader.load(SPIService.class);

        while(providers.hasNext()) {
            SPIService ser = providers.next();
            ser.execute();
        }
        System.out.println("--------------------------------");
        Iterator<SPIService> iterator = load.iterator();
        while(iterator.hasNext()) {
            SPIService ser = iterator.next();
            ser.execute();
        }
    }
}
```

简单的跟一下ServiceLoader的代码：

```dart
    private ServiceLoader(Class<S> svc, ClassLoader cl) {
      	// 目标接口
        service = Objects.requireNonNull(svc, "Service interface cannot be null");
      	// 使用的类加载器，如果不指定则使用ApplicationClassLoader进行加载
        loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
        acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
        reload();
    }
```

![image-20210224193218208](截图/Java基础/SPI.png)



比较典型的例子就是JDBC的加载了。在早期版本中，需要先设置数据库驱动的连接，再通过DriverManager.getConnection获取一个Connection。在后续的版本中，设置数据库驱动连接，这一步骤就不再需要了。

```java
String url = "jdbc:mysql:///consult?serverTimezone=UTC";
String user = "root";
String password = "root";

Class.forName("com.mysql.jdbc.Driver");
Connection connection = DriverManager.getConnection(url, user, password);
```

`DriverManager`类，它在静态代码块里面做了一件比较重要的事，它通过SPI机制， 把数据库驱动连接初始化了。

```java
public class DriverManager {
    static {
        loadInitialDrivers();
        println("JDBC DriverManager initialized");
    }
  
    private static void loadInitialDrivers() {
        AccessController.doPrivileged(new PrivilegedAction<Void>() {
            public Void run() {
                //很明显，它要加载Driver接口的服务类，Driver接口的包为:java.sql.Driver
                //所以它要找的就是META-INF/services/java.sql.Driver文件
                ServiceLoader<Driver> loadedDrivers = ServiceLoader.load(Driver.class);
                Iterator<Driver> driversIterator = loadedDrivers.iterator();
                try{
                    //查到之后创建对象
                    while(driversIterator.hasNext()) {
                        driversIterator.next();
                    }
                } catch(Throwable t) {
                    // Do nothing
                }
                return null;
            }
        });
    }
}
```

上一步已经找到了MySQL中的com.mysql.cj.jdbc.Driver全限定类名，当调用next方法时，就会创建这个类的实例。它就完成了一件事，向DriverManager注册自身的实例。

```java
public class Driver extends NonRegisteringDriver implements java.sql.Driver {
    static {
        try {
            //注册
            //调用DriverManager类的注册方法
            //往registeredDrivers集合中加入实例
            java.sql.DriverManager.registerDriver(new Driver());
        } catch (SQLException E) {
            throw new RuntimeException("Can't register driver!");
        }
    }
    public Driver() throws SQLException {
        // Required for Class.forName().newInstance()
    }
}
```







> [深入理解SPI机制](https://www.jianshu.com/p/3a3edbcd8f24)







# 中断

[Java里一个线程调用了Thread.interrupt()到底意味着什么？](https://www.zhihu.com/question/41048032/answer/89431513)

简单来说就是在java中，interrupt() 并不能真正的中断线程。

>具体来说，当对一个线程，调用 interrupt() 时，
>① 如果线程处于被阻塞状态（例如处于sleep, wait, join 等状态），那么线程将立即退出被阻塞状态，并抛出一个InterruptedException异常。仅此而已。
>② 如果线程处于正常活动状态，那么会将该线程的中断标志设置为 true，仅此而已。被设置中断标志的线程将继续正常运行，不受影响。
>
>
>
>作者：Intopass
>链接：https://www.zhihu.com/question/41048032/answer/89431513
>来源：知乎
>著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



# 实操

## 数据库死锁

常规死锁见《jdk》中相关内容。

> [一次诡异的线上数据库的死锁问题排查过程](https://mp.weixin.qq.com/s/bRKcuUo3Pbfv6CPK82Y01A)



## CPU100%

> [【CPU100%排查】CPU100%问题排查方案](https://www.cnblogs.com/july-sunny/p/12611409.html)
>
> [【原创】谈谈线上CPU100%排查套路](https://www.cnblogs.com/rjzheng/p/10315250.html)

1. 使用`top -c `查看CPU 占用情况 ，按P(大写)可以倒序查看占CPU占用率

2. 找到占用率高的进程以后，再定位到具体线程

   比如 此时进程ID 14724 CPU占用高，进一步使用`top -Hp 14724`定位该进程内所有的线程使用情况

3. 定位到该进程内，15153 的线程CPU占用高，进一步分析内存堆栈的情况

   使用`jstack -l 14724 (进程id) > 14724.stack `将进程内的线程情况重定向到14724.stack这个文件，方便查看

4. 之后可以把那个线程号转为16进制，这是因为最后文件里的线程号是16进制的方便grep定位



## 内存溢出

> [生产出现oom问题，怎么排查？](https://www.cnblogs.com/c-xiaohai/p/12489336.html)

1. 使用ps命令查看到底是哪个进程内存溢出了，找到pid。

2. 使用jstat命令。用`jstat -gcutil 20886 1000 10`命令，就是用jstat工具，对指定java进程（20886就是进程id，通过ps -aux | grep java命令就能找到），按照指定间隔，看一下统计信息，这里会每隔一段时间显示一下，包括新生代的两个S0、s1区、Eden区，以及老年代的内存使用率，还有young gc以及full gc的次数。

   使用 `jstat -gcutil 8968 500 5 `表示每500毫秒打印一次Java堆状况（各个区的容量、使用容量、gc时间等信息），打印5次

   ![img](截图/JVM/jstat.png)

3. 使用jmap命令查看。执行`jmap  -histo pid`可以打印出当前堆中所有每个类的实例数量和内存占用，class  name是每个类的类名（[B是byte类型，[C是char类型，[I是int类型），bytes是这个类的所有示例占用内存大小，instances是这个类的实例数量。

4. 把当前堆内存的快照转储到dumpfile_jmap.hprof文件中，然后对内存快照进行分析。使用`jmap -dump:format=b,file=文件名 [pid]`，就可以把指定java进程的堆内存快照搞到一个指定的文件里去，但是`jmap -dump:format`其实一般会比较慢一些，也可以用gcore工具来导出内存快照

   例如：`jmap -dump:format=b,file=D:/log/jvm/dumpfile_jmap.hprof 20886`

5. 使用**eclipse memory analyer**等MAT工具来进行详细的分析。

线上jvm必须配置-XX:+HeapDumpOnOutOfMemoryError，-XX:HeapDumpPath=/path/heap/dump。因为这样就是说OOM的时候自动导出一份内存快照。





## 接口响应慢



## 查看jvm日志



# 面试题

> #### 抽象类必须要有抽象方法吗？

抽象类中不一定要包含abstrace方法。也就是抽象中可以没有abstract方法。



> #### 抽象类能使用 final 修饰吗？

final关键字不能用来抽象类和接口。



> #### Files的常用方法都有哪些？

```java
public static void main(String[] args) throws IOException {
        //创建方法
        File file = new File("F://a.txt");
        System.out.println("创建成功了吗？"+file.createNewFile());
        System.out.println("单级文件夹创建成功了吗？"+file.mkdir());
        System.out.println("多级文件夹创建成功了吗？"+file.mkdirs());
        File dest = new File("F://电影//c.txt");
        System.out.println("重命名成功了吗？"+file.renameTo(dest));
        
        //删除方法
        File file = new File("F://电影");
        System.out.println("删除成功了吗？"+file.delete());
        file.deleteOnExit();
        
        //判断方法
        File file = new File("F://a.txt");
        System.out.println("文件或者文件夹存在吗？"+file.exists());
        System.out.println("是一个文件吗？"+file.isFile());
        System.out.println("是一个文件夹吗？"+file.isDirectory());
        System.out.println("是隐藏文件吗？"+file.isHidden());
        System.out.println("此路径是绝对路径名？"+file.isAbsolute());
        
        //获取方法
        File file = new File("f://a.txt");
        System.out.println("文件或者文件夹得名称是："+file.getName());
        System.out.println("绝对路径是："+file.getPath());
        System.out.println("绝对路径是："+file.getAbsolutePath());
        System.out.println("文件大小是（以字节为单位）:"+file.length());
        System.out.println("父路径是"+file.getParent());
        //使用日期类与日期格式化类进行获取规定的时间
        long  lastmodified= file.lastModified();
        Date data = new Date(lastmodified);
        SimpleDateFormat simpledataformat = new SimpleDateFormat("YY年MM月DD日 HH:mm:ss");
        System.out.println("最后一次修改的时间是："+simpledataformat.format(data));

        
        //文件或者文件夹的方法
        File[] file = File.listRoots();
        System.out.println("所有的盘符是：");
        for(File item : file){
            System.out.println("/t"+item);
        }
        File filename =new File("F://Java workspace//Java");
        String[] name = filename.list();
        System.out.println("指定文件夹下的文件或者文件夹有：");
        for(String item : name){
            System.out.println("/t"+item);
        }
        File[] f = filename.listFiles();
        System.out.println("获得该路径下的文件或文件夹是：");
        for(File item : f){
            System.out.println("/t"+item.getName());
            }
        }
```



> #### Collection 和 Collections 有什么区别？

java.util.Collection 是一个**集合接口**。Collections则是集合类的一个工具类



> #### Array 和 ArrayList 有何区别？

数组(Array)和列表(ArrayList)

Array可以包含基本类型和对象类型，ArrayList只能包含对象类型。
Array大小是固定的，ArrayList的大小是动态变化的。



> #### 在 Queue 中 poll()和 remove()有什么区别？

1. queue的增加元素方法add和offer的区别在于，add方法在队列满的情况下将选择抛异常的方法来表示队列已经满了，而offer方法通过返回false表示队列已经满了；在有限队列的情况，使用offer方法优于add方法；
2. remove方法和poll方法都是删除队列的头元素，remove方法在队列为空的情况下将抛异常，而poll方法将返回null；
3. element和peek方法都是返回队列的头元素，但是不删除头元素，区别在与element方法在队列为空的情况下，将抛异常，而peek方法将返回null.



> #### 怎么确保一个集合不能被修改？

LIst<?>

//todo



> #### 如何实现对象克隆？

有两种方式：

1. 实现Cloneable接口并重写Object类中的clone()方法；
2. 实现Serializable接口，通过对象的序列化和反序列化实现克隆，可以实现真正的深度克隆



> #### 深拷贝的几种实现

- `clone()`方法其实还是一个浅拷贝，虽然会返回一个新对象，但是指向同一块内存地址，还是需要重写`clone()`方法。
- `BeanUtiles.copyProperties(Object source, Object target)`
- 序列化

org.springframework.beans.BeanUtils#copyProperties(java.lang.Object, java.lang.Object, java.lang.Class<?>, java.lang.String...)

```java
private static void copyProperties(Object source, Object target, @Nullable Class<?> editable,
      @Nullable String... ignoreProperties) throws BeansException {

   Assert.notNull(source, "Source must not be null");
   Assert.notNull(target, "Target must not be null");

   Class<?> actualEditable = target.getClass();
   if (editable != null) {
      if (!editable.isInstance(target)) {
         throw new IllegalArgumentException("Target class [" + target.getClass().getName() +
               "] not assignable to Editable class [" + editable.getName() + "]");
      }
      actualEditable = editable;
   }
   PropertyDescriptor[] targetPds = getPropertyDescriptors(actualEditable);
   List<String> ignoreList = (ignoreProperties != null ? Arrays.asList(ignoreProperties) : null);

   for (PropertyDescriptor targetPd : targetPds) {
      Method writeMethod = targetPd.getWriteMethod();
      if (writeMethod != null && (ignoreList == null || !ignoreList.contains(targetPd.getName()))) {
         PropertyDescriptor sourcePd = getPropertyDescriptor(source.getClass(), targetPd.getName());
         if (sourcePd != null) {
            Method readMethod = sourcePd.getReadMethod();
            if (readMethod != null &&
                  ClassUtils.isAssignable(writeMethod.getParameterTypes()[0], readMethod.getReturnType())) {
               try {
                  if (!Modifier.isPublic(readMethod.getDeclaringClass().getModifiers())) {
                     readMethod.setAccessible(true);
                  }
                  Object value = readMethod.invoke(source);
                  if (!Modifier.isPublic(writeMethod.getDeclaringClass().getModifiers())) {
                     writeMethod.setAccessible(true);
                  }
                  writeMethod.invoke(target, value);
               }
               catch (Throwable ex) {
                  throw new FatalBeanException(
                        "Could not copy property '" + targetPd.getName() + "' from source to target", ex);
               }
            }
         }
      }
   }
}
```



> [细说 Java 的深拷贝和浅拷贝](https://segmentfault.com/a/1190000010648514)



> #### throw 和 throws 的区别？

throws是用来声明一个方法可能抛出的所有异常信息，throws是将异常声明但是不处理，而是将异常往上传，谁调用我就交给谁处理。而throw则是指抛出的一个具体的异常类型。

```java
 public int div(int i,int j) throws Exception{}
 
 throw new RuntimeException("a的值大于0，不符合要求");
```



> #### final、finally、finalize 有什么区别？

[final、finally与finalize的区别](https://www.cnblogs.com/ktao/p/8586966.html)



> #### try-catch-finally 中，如果 catch 中 return 了，finally 还会执行吗？

会执行，在return 前执行



[try-catch-finally中，如果在catch中return了，finally中的代码还会执行么，原理是什么？（异常相关四）](https://blog.csdn.net/qq_40180411/article/details/81428382)



> 大数相加



在java里面的线程是共享进程的堆空间的，但是每个进程的栈空间是独立的