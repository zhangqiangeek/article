---
   title: Java内存模型（JMM）
   date: 2019-05-07 11:55:00
   tags: java
---

在学了Java并发相关的知识点后，有没有反过来去思考一下java内存模型是个什么东西？java内存模型(JMM)这个概念似乎已经上升到了操作系统层面，看着很高大上。翻开每一本介绍java并发编程的书籍，都会把把介绍JMM这一章节放到最前面，由此可见理解JMM对于理解Java并发编程的重要性。但是，知识需要贯通才能理解，每次上来就看JMM，生涩和突兀的概念每次都迫使我快速翻过这一章。在工作两年之后，再从新来梳理下JMM，看看能否从本质上理解这个知识点。
<!-- more -->
### JMM产生的原因
在并发编程中，需要处理两个关键问题：线程之间如何通信以及线程之间如何同步。通信是指线程之间以何种机制来交换信息。在命令式编程中，线程之间的通信机制有两种：共享内存和消息传递。

在共享内存的并发模型中，线程之间共享程序的公共状态，通过写-读内存中的公共状态来进行隐式通信。在消息传递的并发模型里，线程之间没有公共状态，线程之间必须通过发送消息来显式进行通信。

同步是指程序中用于控制两个不同线程间操作发生相对顺序的时机。在共享内存并发模型里，同步是显式进行的。程序员必须显式指定某个方法或某段代码需要在线程之间互斥执行。在消息传递的内存模型里，由于消息的发送必须在消息的接收之前，因此同步是隐式进行的。

Java的并发采用的是共享内存模型，Java线程之间的通信总是隐式执行的，整个通信过程对程序员完全透明。如果编写多线程程序的Java程序员不理解隐式进行的线程之间通信的工作机制，很可能会遇到各种奇怪的内存可见性问题。

由于并发程序要比串行程序复杂的多，其中一个重要的原因是并发程序下数据访问的一致性和安全性会受到严重挑战。那么如何保证一个线程可以看到正确的数据？我们需要在深入了解并行机制的前提下，再定义一种规则，保证多个线程间可以有效地、正确地协同工作，JMM正是为此而生的。

### JMM的抽象结构
Java线程之间的通信由Java内存模型控制，JMM决定了一个线程对共享变量的写入何时对另一个线程可见。从抽象的角度来看，JMM定义了线程和主存之间的抽象关系：线程之间的共享变量存储在主内存中，每个线程都有一个私有的本地内存，本地内存中存储了该线程以读/写共享变量的副本。本地内存是JMM的一个抽象的概念，并不真实存在。它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。
![JMM](http://pic.evilhex.com/2019-05-09-JMM.jpg)
如果线程A和线程B要进行通信的话，必须要经历两个步骤：
1. 线程A把本地内存A中更新过的共享变量刷新到主内存中；
2. 线程B到主内存中去读取线程A之前已经更新过的共享变量。

整体来看， 这两个步骤实质上是线程A在向线程B发送消息，而且这个通信过程必须经过主内存。JMM通过控制主内存和每个线程的本地内存之间的交互，来为Java程序提供内存可见性保证。

### JMM的实现机制
JMM的关键技术点都是围绕着多线程的原子性、可见性和有序性来建立的。原子性是指一个操作是不可中断的。即使是多个线程一起执行的时候，一个操作一旦开始，就不会被其它线程干扰。可见性是指当一个线程修改了某一个共享变量之后，其他线程是否能够立即知道这个修改。有序性是指出于性能考虑，在程序执行时，可能会进行指令重排，重排之后的指令与原指令的顺序未必一致。

#### 重排序
在程序执行时，为了提高性能，编译器和处理器会对指令做重排序。重排序分为三种：
1. 编译器优化的重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。
2. 指令级并行的重排序。现代处理器采用了指令级并行技术来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。
3. 内存系统的重排序。由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。

![指令重排序](http://pic.evilhex.com/2019-05-07-指令重排序.jpg)

其中，指令级重排序和内存系统重排序属于处理器重排序。对于编译器，JMM的编译器重排序规则会禁止特定类型的编译器重排序。对于处理器重排序，JMM的重排序规则会要求Java编译器在生成指令序列时，插入特定类型的内存屏障（Memory Barriers）指令，通过内存屏障指令来禁止特定类型的处理器重排序。
#### happens-before简介
虽然Java虚拟机和执行系统会对指令进行一定的重排序，但是指令重排序是有原则的，并非所有的指令都可以随便改变执行位置。happens-before原则是指令重排序不能违背的。

happens-before是JMM最核心的概念。理解happens-before是理解JMM的关键。

在JMM中，如果一个操作执行的结果需要对另外一个操作可见，那么这两个操作之间必须要存在happens-before关系。这两个操作既可以在一个线程之内，也可以在不同的线程之间。happens-before规则如下：
- 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。
- 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。
- volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
- 传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。

happens-before和JMM的关系如下图所示：
![happens-before](http://pic.evilhex.com/2019-05-07-happens-before.jpg)

一个happens-before规则对应于一个或多个编译器和处理器的重排序规则。对应Java程序员来说，happens-before规则简单易懂，避免了Java程序员为了理解JMM提供的内存可见性保证而去学习复杂的重排序规则以及这些规则的具体实现方法。

