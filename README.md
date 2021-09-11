# 一、多线程和并发编程

​       并发编程的目的是为了让程序运行得更快，但是，并不是启动更多的线程就能让程序最 

大限度地并发执行。在进行并发编程时，如果希望通过多线程执行任务让程序运行得更快，会 

面临非常多的**挑战**，比如**上下文切换的问题、死锁**的问题，以及受限于**硬件和软件的资源限制** 

问题。

## 第1章 并发编程的挑战 

### 1.1 上下文切换

* 时间片一般是几十毫秒（ms）

* 线程有创建和上下文切换的开销

* 使用Lmbench3[1]可以测量上下文切换的时长

* 使用vmstat可以测量上下文切换的次数

  ![image-20210911094620143](C:\Users\wangky\AppData\Roaming\Typora\typora-user-images\image-20210911094620143.png)

> CS（Content Switch）表示上下文切换的次数，从上面的测试结果中我们可以看到，上下文 
>
> 每1秒切换1000多次。 

* 减少上下文切换的方法有无锁并发编程、CAS算法、使用最少线程和使用协程

  * 无锁并发编程。多线程竞争锁时，会引起上下文切换，所以多线程处理数据时，可以用一 

    些办法来避免使用锁，如将数据的ID按照Hash算法取模分段，不同的线程处理不同段的数据。

  * CAS算法。Java的Atomic包使用CAS算法来更新数据，而不需要加锁

  * 使用最少线程。避免创建不需要的线程，比如任务很少，但是创建了很多线程来处理，这 

    样会造成大量线程都处于等待状态。

  * 协程：在单线程里实现多任务的调度，并在单线程里维持多个任务间的切换

### 1.2 死锁

避免死锁的几个常见方法

* 避免一个线程同时获取多个锁。 

* 避免一个线程在锁内同时占用多个资源，尽量保证每个锁只占用一个资源。 

* 尝试使用定时锁，使用lock.tryLock（timeout）来替代使用内部锁机制。 

* 对于数据库锁，加锁和解锁必须在一个数据库连接里，否则会出现解锁失败的情况。

### 1.3 资源限制的挑战

**什么是资源限制** 

> 资源限制是指在进行并发编程时，程序的执行速度受限于计算机硬件资源或软件资源。 
>
> 例如，服务器的带宽只有2Mb/s，某个资源的下载速度是1Mb/s每秒，系统启动10个线程下载资 
>
> 源，下载速度不会变成10Mb/s，所以在进行并发编程时，要考虑这些资源的限制。硬件资源限 
>
> 制有带宽的上传/下载速度、硬盘读写速度和CPU的处理速度。软件资源限制有数据库的连接 
>
> 数和socket连接数等。 

**如何解决资源限制的问题** 

> 对于硬件资源限制，可以考虑使用集群并行执行程序。既然单机的资源有限制，那么就让 
>
> 程序在多机上运行。比如使用ODPS、Hadoop或者自己搭建服务器集群，不同的机器处理不同 
>
> 的数据。可以通过“数据ID%机器数”，计算得到一个机器编号，然后由对应编号的机器处理这 
>
> 笔数据。 
>
> 对于软件资源限制，可以考虑使用资源池将资源复用。比如使用连接池将数据库和Socket 
>
> 连接复用，或者在调用对方webservice接口获取数据时，只建立一个连接。

**在资源限制情况下进行并发编程**

> 如何在资源限制的情况下，让程序执行得更快呢？方法就是，根据不同的资源限制调整 
>
> 程序的并发度，比如下载文件程序依赖于两个资源——带宽和硬盘读写速度。有数据库操作 
>
> 时，涉及数据库连接数，如果SQL语句执行非常快，而线程的数量比数据库连接数大很多，则 
>
> 某些线程会被阻塞，等待数据库连接。

**强烈建议多使用JDK并发包提供的并发容器和工具类来解决并发** 

**问题，因为这些类都已经通过了充分的测试和优化，均可解决了本章提到的几个挑战**

## 第2章 Java并发机制的底层实现原理

> Java代码在编译后会变成Java字节码，字节码被类加载器加载到JVM里，JVM执行字节 
>
> 码，最终需要转化为汇编指令在CPU上执行，Java中所使用的并发机制依赖于JVM的实现和 
>
> CPU的指令。本章我们将深入底层一起探索下Java并发机制的底层实现原理。

### 1、volatile的应用 

* volatile是轻量级的 synchronized，它在多处理器开发中保证了共享变量的“可见性”。

  ***可见性的意思是当一个线程 修改一个共享变量时，另外一个线程能读到这个修改的值。***

  

  **volatile的定义与实现原理** 

  > Java语言规范第3版中对volatile的定义如下：Java编程语言允许线程访问共享变量，为了 
  >
  > 确保共享变量能被准确和一致地更新，线程应该确保通过排他锁单独获得这个变量。Java语言 
  >
  > 提供了volatile，在某些情况下比锁要更加方便。如果一个字段被声明成volatile，Java线程内存 
  >
  > 模型确保所有线程看到这个变量的值是一致的。

  在了解volatile实现原理之前，我们先来看下与其实现原理相关的CPU术语与说明。表2-1 

  是CPU术语的定义。 

  ![image-20210911101422960](C:\Users\wangky\AppData\Roaming\Typora\typora-user-images\image-20210911101422960.png)

  **volatile是如何来保证可见性的呢？**

  让我们在X86处理器下通过工具获取JIT编译器生成的 

  汇编指令来查看对volatile进行写操作时，CPU会做什么事情。 

  **Java代码如下。** 

  `instance = new Singleton(); // instance是volatile变量` 

  转变成汇编代码，如下。 

  `0x01a3de1d: movb $0×0,0×1104800(%esi);0x01a3de24: lock addl $0×0,(%esp);` 

  有volatile变量修饰的共享变量进行写操作的时候会多出第二行汇编代码，通过查IA-32架 

  **构软件开发者手册可知，Lock前缀的指令在多核处理器下会引发了两件事情[1]。** 

  1）将当前处理器缓存行的数据写回到系统内存。 

  2）这个写回内存的操作会使在其他CPU里缓存了该内存地址的数据无效。 

  为了提高处理速度，处理器不直接和内存进行通信，而是先将系统内存的数据读到内部

  **在多处理器下，为了保证各个处理器的缓存是一致的？**

  在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一 

  致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当 

  处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状 

  态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存 

  里。 

  **IA-32处理器和Intel 64处 理器使用MESI（修改、独占、共享、无效）**控制协议去维护内部缓存和其他处理器缓存的一致 

  性。

  **volatile的使用优化**

  将共享变量追加到64字节。

### 2、synchronized的实现原理与应用 

### **synchronized实现同步的基础：Java中的每一个对象都可以作为锁。具体表现** 

为以下3种形式

* ·对于普通同步方法，锁是当前实例对象。 

* ·对于静态同步方法，锁是当前类的Class对象。 

* ·对于同步方法块，锁是Synchonized括号里配置的对象。

**当一个线程试图访问同步代码块时，它首先必须得到锁，退出或抛出异常时必须释放锁。** 

#### 那么锁到底存在哪里呢？锁里面会存储什么信息呢？ 

`从JVM规范中可以看到Synchonized在JVM里的实现原理，JVM基于进入和退出Monitor对` 

`象来实现方法同步和代码块同步，但两者的实现细节不一样。代码块同步是使用monitorenter` 

`和monitorexit指令实现的，而方法同步是使用另外一种方式实现的，细节在JVM规范里并没有` 

`详细说明。但是，方法的同步同样可以使用这两个指令来实现。` 

`monitorenter指令是在编译后插入到同步代码块的开始位置，而monitorexit是插入到方法结` 

`束处和异常处，JVM要保证每个monitorenter必须有对应的monitorexit与之配对。任何对象都有` 

`一个monitor与之关联，当且一个monitor被持有后，它将处于锁定状态。线程执行到monitorenter` 

`指令时，将会尝试获取对象所对应的monitor的所有权，即尝试获得对象的锁。`

#### 2.1、Java对象头

synchronized用的锁是存在Java对象头里的。如果对象是数组类型，则虚拟机用3个字宽 

（Word）存储对象头，如果对象是非数组类型，则用2字宽存储对象头。在32位虚拟机中，1字宽 

等于4字节，即32bit，如表2-2所示。

![image-20210911145840104](C:\Users\wangky\AppData\Roaming\Typora\typora-user-images\image-20210911145840104.png)

在64位虚拟机下，Mark Word是64bit大小的，其存储结构如表2-5所示。

![image-20210911145903623](C:\Users\wangky\AppData\Roaming\Typora\typora-user-images\image-20210911145903623.png)

#### 2.2 锁的升级与对比

Java SE 1.6为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁”，在 

Java SE 1.6中，锁一共有4种状态，级别从低到高依次是：无锁状态、偏向锁状态、轻量级锁状 

态和重量级锁状态，这几个状态会随着竞争情况逐渐升级。锁可以升级但不能降级，意味着偏 

向锁升级成轻量级锁后不能降级成偏向锁。这种锁升级却不能降级的策略，目的是为了提高 

获得锁和释放锁的效率

* 偏向锁

HotSpot[1]的作者经过研究发现，大多数情况下，锁不仅不存在多线程竞争，而且总是由同 

一线程多次获得，为了让线程获得锁的代价更低而引入了偏向锁。

`当一个线程访问同步块并 获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出` 

`同步块时不需要进行CAS操作来加锁和解锁，只需简单地测试一下对象头的Mark Word里是否` 

`存储着指向当前线程的偏向锁。如果测试成功，表示线程已经获得了锁。如果测试失败，则需` 

`要再测试一下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁）：如果没有设置，则` 

`使用CAS竞争锁；如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。` 

* .轻量级锁 

加锁：线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并 

将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用 

CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失 

败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。 

解锁：轻量级解锁时，会使用原子的CAS操作将Displaced Mark Word替换回到对象头，如果成 

功，则表示没有竞争发生。如果失败，表示当前锁存在竞争，锁就会膨胀成重量级锁。图2-2是 

两个线程同时争夺锁，导致锁膨胀的流程图

### 3、原子操作的实现原理 

> 原子（atomic）本意是“不能被进一步分割的最小粒子”，而原子操作（atomic operation）意 
>
> 为“不可被中断的一个或一系列操作”。在多处理器上实现原子操作就变得有点复杂。让我们 
>
> 一起来聊一聊在Intel处理器和Java里是如何实现原子操作的。 

**Java如何实现原子操作 ：**

`在Java中可以通过锁和循环CAS的方式来实现原子操作`

**本章我们一起研究了volatile、synchronized和原子操作的实现原理。Java中的大部分容器** 

**和框架都依赖于本章介绍的volatile和原子操作的实现原理，了解这些原理对我们进行并发编** 

**程会更有帮助。**

## 第3章 Java内存模型 

Java线程之间的通信对程序员完全透明，内存可见性问题很容易困扰Java程序员，本章将 

揭开Java内存模型神秘的面纱。

本章大致分4部分：

## 1、Java内存模型的基础，主要介绍内存模型相关的基本概念；

### 3.1.1 并发编程模型的两个关键问题 

​        在并发编程中，需要处理两个关键问题：线程之间如何通信及线程之间如何同步（这里的 

线程是指并发执行的活动实体）。通信是指线程之间以何种机制来交换信息。在命令式编程 

中，**线程之间的通信机制有两种**：`共享内存和消息传递。`

​		**在共享内存的并发模型里，线程之间共享程序的公共状态，通过写-读内存中的公共状态** 

**进行隐式通信。**在消息传递的并发模型里，线程之间没有公共状态，线程之间必须通过发送消 

息来显式进行通信。 

​		Java的并发采用的是共享内存模型，Java线程之间的通信总是隐式进行，整个通信过程对 

程序员完全透明。如果编写多线程程序的Java程序员不理解隐式进行的线程之间通信的工作 

机制，很可能会遇到各种奇怪的内存可见性问题

### 3.1.2 Java内存模型的抽象结构 

在Java中，所有实例域、静态域和数组元素都存储在堆内存中，堆内存在线程之间共享 

（本章用“共享变量”这个术语代指实例域，静态域和数组元素）。局部变量（Local Variables），方 

法定义参数（Java语言规范称之为Formal Method Parameters）和异常处理器参数（Exception 

Handler Parameters）不会在线程之间共享，它们不会有内存可见性问题，也不受内存模型的影 

响。 

Java线程之间的通信由Java内存模型（本文简称为JMM）控制，JMM决定一个线程对共享 

变量的写入何时对另一个线程可见。从抽象的角度来看，JMM定义了线程和主内存之间的抽 

象关系：线程之间的共享变量存储在主内存（Main Memory）中，每个线程都有一个私有的本地 

内存（Local Memory），本地内存中存储了该线程以读/写共享变量的副本。本地内存是JMM的 

一个抽象概念，并不真实存在。它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优 

化。Java内存模型的抽象示意如图3-1所示。

![image-20210911152705266](C:\Users\wangky\AppData\Roaming\Typora\typora-user-images\image-20210911152705266.png)

**从图3-1来看，如果线程A与线程B之间要通信的话，必须要经历下面2个步骤。** 

1）线程A把本地内存A中更新过的共享变量刷新到主内存中去。 

2）线程B到主内存中去读取线程A之前已更新过的共享变量。 

**如图3-2所示，本地内存A和本地内存B由主内存中共享变量x的副本。假设初始时，这3个** 

内存中的x值都为0。线程A在执行时，把更新后的x值（假设值为1）临时存放在自己的本地内存 

A中。当线程A和线程B需要通信时，线程A首先会把自己本地内存中修改后的x值刷新到主内 

存中，此时主内存中的x值变为了1。随后，线程B到主内存中去读取线程A更新后的x值，此时 

线程B的本地内存的x值也变为了1。 

从整体来看，这两个步骤实质上是线程A在向线程B发送消息，而且这个通信过程必须要 

经过主内存。**JMM通过控制主内存与每个线程的本地内存之间的交互，来为Java程序员提供** 

**内存可见性保证。**



**3.1.3 从源代码到指令序列的重排序** 

在执行程序时，为了提高性能，编译器和处理器常常会对指令做重排序。重排序分3种类 

型。 

1）编译器优化的重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句 

的执行顺序。 

2）指令级并行的重排序。现代处理器采用了指令级并行技术（Instruction-Level 

Parallelism，ILP）来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应 

机器指令的执行顺序。 

3）内存系统的重排序。由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上 

去可能是在乱序执行。 

从Java源代码到最终实际执行的指令序列，会分别经历下面3种重排序，如图3-3所示。 

![image-20210911153155684](C:\Users\wangky\AppData\Roaming\Typora\typora-user-images\image-20210911153155684.png)

上述的1属于编译器重排序，2和3属于处理器重排序。这些重排序可能会导致多线程程序 

出现内存可见性问题。对于编译器，JMM的编译器重排序规则会禁止特定类型的编译器重排 

序（不是所有的编译器重排序都要禁止）。对于处理器重排序，JMM的处理器重排序规则会要 

求Java编译器在生成指令序列时，**插入特定类型的内存屏障（Memory Barriers，Intel称之为** 

**Memory Fence）指令，通过内存屏障指令来禁止特定类型的处理器重排序。** 

JMM属于语言级的内存模型，它确保在不同的编译器和不同的处理器平台之上，通过禁止特定类型的编译器重排序和处理器重排序，为程序员提供一致的内存可见性保证。3.1.4 并发编程模型的分类 

## 2、Java内存模型中的顺序一致性，主要介绍重排序与顺序一致性内存模型；

## 3、同步 原语，主要介绍3个同步原语（synchronized、volatile和final）的内存语义及重排序规则在处理器 中的实现；

## 4、Java内存模型的设计，主要介绍Java内存模型的设计原理，及其与处理器内存模型 

和顺序一致性内存模型的关系

## 第4章 Java并发编程基础 

> **Java从诞生开始就明智地选择了内置对多线程的支持，这使得Java语言相比同一时期的** 
>
> **其他语言具有明显的优势。线程作为操作系统调度的最小单元，多个线程能够同时执行，这将** 
>
> **显著提升程序性能，在多核环境中表现得更加明显。但是，过多地创建线程和对线程的不当管** 
>
> **理也容易造成问题。本章将着重介绍Java并发编程的基础知识，从启动一个线程到线程间不同** 
>
> **的通信方式，最后通过简单的线程池示例以及应用（简单的Web服务器）来串联本章所介绍的** 
>
> **内容**

**线程的状态** 

![image-20210911153707836](C:\Users\wangky\AppData\Roaming\Typora\typora-user-images\image-20210911153707836.png)





## 第5章 Java中的锁

本章将介绍Java并发包中与锁相关的API和组件，以及这些API和组件的使用方式和实现 

细节。内容主要围绕两个方面：使用，通过示例演示这些组件的使用方法以及详细介绍与锁相 

关的API；实现，通过分析源码来剖析实现细节，因为理解实现的细节方能更加得心应手且正 

确地使用这些组件。希望通过以上两个方面的讲解使开发者对锁的使用和实现两个层面有一 

定的了解

5.1 Lock接口

5.2 队列同步器

5.3 重入锁

5.4 读写锁

5.5 LockSupport工具

5.6 Codition接口



## 第6章 Java并发容器和框架 

Java程序员进行并发编程时，相比于其他语言的程序员而言要倍感幸福，因为并发编程大 

师Doug Lea不遗余力地为Java开发者提供了非常多的并发容器和框架。本章让我们一起来见 

识一下大师操刀编写的并发容器和框架，并通过每节的原理分析一起来学习如何设计出精妙 

的并发程序。

**ConcurrentHashMap** 

ConcurrentHashMap所使用的锁分段技术。首先将数据分成一段一段地存 

储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数 

据也能被其他线程访问。

**ConcurrentLinkedQueue**

ConcurrentLinkedQueue由head节点和tail节点组成，每个节点（Node）由节点元素（item）和 

指向下一个节点（next）的引用组成，节点与节点之间就是通过这个next关联起来，从而组成一 

张链表结构的队列。默认情况下head节点存储的元素为空，tail节点等于head节点。

**堵塞队列**

阻塞队列常用于生产者和消费者的场景，生产者是向队列里添加元素的线程，消费者是 

从队列里取元素的线程。阻塞队列就是生产者用来存放元素、消费者用来获取元素的容器

![image-20210911160740584](C:\Users\wangky\AppData\Roaming\Typora\typora-user-images\image-20210911160740584.png)

JDK 7提供了7个阻塞队列，如下。 

·ArrayBlockingQueue：一个由数组结构组成的有界阻塞队列。 

·LinkedBlockingQueue：一个由链表结构组成的有界阻塞队列。 

·PriorityBlockingQueue：一个支持优先级排序的无界阻塞队列。 

·DelayQueue：一个使用优先级队列实现的无界阻塞队列。 

·SynchronousQueue：一个不存储元素的阻塞队列。 

·LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。 

·LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列

**Fork/Join框架** 

Fork/Join框架是Java 7提供的一个用于并行执行任务的框架，是一个把大任务分割成若干 

个小任务，最终汇总每个小任务结果后得到大任务结果的框架。 

我们再通过Fork和Join这两个单词来理解一下Fork/Join框架。Fork就是把一个大任务切分 

为若干子任务并行的执行，Join就是合并这些子任务的执行结果，最后得到这个大任务的结 

果。比如计算1+2+…+10000，可以分割成10个子任务，每个子任务分别对1000个数进行求和， 

最终汇总这10个子任务的结果。Fork/Join的运行流程如图6-6所示。

![image-20210911161301188](C:\Users\wangky\AppData\Roaming\Typora\typora-user-images\image-20210911161301188.png)

**Fork/Join框架的设计** 

我们已经很清楚Fork/Join框架的需求了，那么可以思考一下，如果让我们来设计一个 

Fork/Join框架，该如何设计？这个思考有助于你理解Fork/Join框架的设计。 

步骤1 分割任务。首先我们需要有一个fork类来把大任务分割成子任务，有可能子任务还 

是很大，所以还需要不停地分割，直到分割出的子任务足够小。 

步骤2 执行任务并合并结果。分割的子任务分别放在双端队列里，然后几个启动线程分 

别从双端队列里获取任务执行。子任务执行完的结果都统一放在一个队列里，启动一个线程 

从队列里拿数据，然后合并这些数据

```java
package com.dyg.juc;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.Future;
import java.util.concurrent.RecursiveTask;

public class CountTask extends RecursiveTask<Integer> {
    private static final int THRESHOLD = 2; // 阈值
    private int start;
    private int end;

    public CountTask(int start, int end) {
        this.start = start;
        this.end = end;
    }

    @Override
    protected Integer compute() {
        int sum = 0; // 如果任务足够小就计算任务
        boolean canCompute = (end - start) <= THRESHOLD;
        if (canCompute) {
            for (int i = start; i <= end; i++) {
                sum += i;
            }
        } else {// 如果任务大于阈值，就分裂成两个子任务计算
            int middle = (start + end) / 2;
            CountTask leftTask = new CountTask(start, middle);
            CountTask rightTask = new CountTask(middle + 1, end);
            // 执行子任务
            leftTask.fork();
            rightTask.fork();
            // 等待子任务执行完，并得到其结果
            int leftResult = leftTask.join();
            int rightResult = rightTask.join();
            // 合并子任务
            sum = leftResult + rightResult;
        }
        return sum;
    }

    public static void main(String[] args) {
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        // 生成一个计算任务，负责计算1+2+3+4
        CountTask task = new CountTask(1, 4); // 执行一个任务
        Future<Integer> result = forkJoinPool.submit(task);
        try {
            System.out.println(result.get());
        } catch (InterruptedException e) {
        } catch (ExecutionException e) {
        }
    }
}
```



## 第7章 Java中的13个原子操作类 

Atomic包里一共提供了13个类，属于4种类型的原子更 新方式，分别是原子更新基本类型、原子更新数组、原子更新引用和原子更新属性（字段）。 

Atomic包里的类基本都是使用Unsafe实现的包装类。

### 7.1 原子更新基本类型类 

使用原子的方式更新基本类型，Atomic包提供了以下3个类。 

·AtomicBoolean：原子更新布尔类型。 

·AtomicInteger：原子更新整型。 

·AtomicLong：原子更新长整型。 

以上3个类提供的方法几乎一模一样，所以本节仅以AtomicInteger为例进行讲解，

那么getAndIncrement是如何实现原子操作的呢？让我们一起分析其实现原理， 

getAndIncrement的源码如代码清单7-2所示。 

代码清单7-2 AtomicInteger.java 

```java
    public final int getAndIncrement() {
        for (; ; ) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next)) return current;
        }
    }

    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
```

源码中for循环体的第一步先取得AtomicInteger里存储的数值，第二步对AtomicInteger的当 

前数值进行加1操作，关键的第三步调用compareAndSet方法来进行原子更新操作，该方法先检 

查当前数值是否等于current，等于意味着AtomicInteger的值没有被其他线程修改过，则将 

AtomicInteger的当前数值更新成next的值，如果不等compareAndSet方法会返回false，程序会进 

入for循环重新进行compareAndSet操作。 

Atomic包提供了3种基本类型的原子更新，但是Java的基本类型里还有char、float和double等。那么问题来了，如何原子的更新其他的基本类型呢？Atomic包里的类基本都是使用Unsafe 

实现的，让我们一起看一下Unsafe的源码，如代码清单7-3所示。 

![image-20210911162924284](C:\Users\wangky\AppData\Roaming\Typora\typora-user-images\image-20210911162924284.png)

### 7.2 原子更新数组 

### 7.3 原子更新引用类型 

### 7.4 原子更新字段类 

```java
package com.dyg.juc;

import java.util.concurrent.atomic.AtomicIntegerFieldUpdater;

public class AtomicIntegerFieldUpdaterTest {
    // 创建原子更新器，并设置需要更新的对象类和对象的属性
    private static AtomicIntegerFieldUpdater<User> a = AtomicIntegerFieldUpdater.newUpdater(User.class, "old");

    public static void main(String[] args) {
        // 设置柯南的年龄是10岁 
        User conan = new User("conan", 10);
        // 柯南长了一岁，但是仍然会输出旧的年龄 
        System.out.println(a.getAndIncrement(conan));
        // 输出柯南现在的年龄 
        System.out.println(a.get(conan));
    }

    public static class User {
        private String name;
        public volatile int old;

        public User(String name, int old) {
            this.name = name;
            this.old = old;
        }

        public String getName() {
            return name;
        }

        public int getOld() {
            return old;
        }
    }

```

## 第8章 Java中的并发工具类 

在JDK的并发包里提供了几个非常有用的并发工具类。CountDownLatch、CyclicBarrier和 

Semaphore工具类提供了一种并发流程控制的手段，Exchanger工具类则提供了在线程间交换数 

据的一种手段。本章会配合一些应用场景来介绍如何使用这些工具类

### 8.1 等待多线程完成的CountDownLatch 

> CountDownLatch的构造函数接收一个int类型的参数作为计数器，如果你想等待N个点完 
>
> 成，这里就传入N。 当我们调用CountDownLatch的countDown方法时，N就会减1，CountDownLatch的await方法 
>
> 会阻塞当前线程，直到N变成零。由于countDown方法可以用在任何地方，所以这里说的N个 
>
> 点，可以是N个线程，也可以是1个线程里的N个执行步骤。用在多个线程时，只需要把这个 
>
> CountDownLatch的引用传递到线程里即可。 

假如有这样一个需求：我们需要解析一个Excel里多个sheet的数据，此时可以考虑使用多 

线程，每个线程解析一个sheet里的数据，等到所有的sheet都解析完之后，程序需要提示解析完 

成。在这个需求中，要实现主线程等待所有线程完成sheet的解析操作，最简单的做法是使用 

join()方法

### 8.2 同步屏障CyclicBarrier 

> CyclicBarrier的字面意思是可循环使用（Cyclic）的屏障（Barrier）。它要做的事情是，让一 
>
> 组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会 
>
> 开门，所有被屏障拦截的线程才会继续运行。



**CyclicBarrier默认的构造方法是CyclicBarrier（int parties），其参数表示屏障拦截的线程数** 

**量，每个线程调用await方法告诉CyclicBarrier我已经到达了屏障，然后当前线程被阻塞**。示例 

代码如代码清单8-3所示。 

```java
public class CyclicBarrierTest {
    static CyclicBarrier c = new CyclicBarrier(2);

    public static void main(String[] args) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    c.await();
                } catch (Exception e) {
                }
                System.out.println(1);
            }
        }).start();
        try {
            c.await();
        } catch (Exception e) {
        }
        System.out.println(2);
    }
}
```

**CyclicBarrier的应用场景** 

CyclicBarrier可以用于多线程计算数据，最后合并计算结果的场景。例如，用一个Excel保 

存了用户所有银行流水，每个Sheet保存一个账户近一年的每笔银行流水，现在需要统计用户 

的日均银行流水，先用多线程处理每个sheet里的银行流水，都执行完之后，得到每个sheet的日 

均银行流水，最后，再用barrierAction用这些线程的计算结果，计算出整个Excel的日均银行流 

水，如代码清单8-5所示。 

```java
package com.dyg.juc;

import java.util.Map;
import java.util.concurrent.*;

/**
 * 银行流水处理服务类
 *
 * @authorftf
 */
public class BankWaterService implements Runnable {
    //创建4个屏障，处理完之后执行当前类的run方法
    private CyclicBarrier c = new CyclicBarrier(4, this);
    //假设只有4个sheet，所以只启动4个线程
    private Executor executor = Executors.newFixedThreadPool(4);
    //保存每个sheet计算出的银流结果
    private ConcurrentHashMap<String, Integer> sheetBankWaterCount = new ConcurrentHashMap<String, Integer>();

    private void count() {
        for (int i = 0; i < 4; i++) {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    // 计算当前sheet的银流数据，计算代码省略
                    sheetBankWaterCount.put(Thread.currentThread().getName(), 1);
                    // 银流计算完成，插入一个屏障
                    try {
                        c.await();
                    } catch (InterruptedException |
                            BrokenBarrierException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }

    @Override
    public void run() {
        int result = 0;
        // 汇总每个sheet计算出的结果
        for (Map.Entry<String, Integer> sheet : sheetBankWaterCount.entrySet()) {
            result += sheet.getValue();
        }
        // 将结果输出
        sheetBankWaterCount.put("result", result);
        System.out.println(result);
    }

    public static void main(String[] args) {
        BankWaterService bankWaterCount = new BankWaterService();
        bankWaterCount.count();
    }
}

```

**CyclicBarrier和CountDownLatch的区别** 

CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重 

置。所以CyclicBarrier能处理更为复杂的业务场景。例如，如果计算发生错误，可以重置计数 

器，并让线程重新执行一次。 

CyclicBarrier还提供其他有用的方法，比如getNumberWaiting方法可以获得Cyclic-Barrier 

阻塞的线程数量。isBroken()方法用来了解阻塞的线程是否被中断。

### 8.3 控制并发线程数的Semaphore

Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，以 

保证合理的使用公共资源。 

多年以来，我都觉得从字面上很难理解Semaphore所表达的含义，只能把它比作是控制流 

量的红绿灯。比如××马路要限制流量，只允许同时有一百辆车在这条路上行使，其他的都必须 

在路口等待，所以前一百辆车会看到绿灯，可以开进这条马路，后面的车会看到红灯，不能驶 

入××马路，但是如果前一百辆中有5辆车已经离开了××马路，那么后面就允许有5辆车驶入马 

路，这个例子里说的车就是线程，驶入马路就表示线程在执行，离开马路就表示线程执行完 

成，看见红灯就表示线程被阻塞，不能执行。 

**应用场景** 

Semaphore可以用于做流量控制，特别是公用资源有限的应用场景，比如数据库连接。假 

如有一个需求，要读取几万个文件的数据，因为都是IO密集型任务，我们可以启动几十个线程 

并发地读取，但是如果读到内存后，还需要存储到数据库中，而数据库的连接数只有10个，这 

时我们必须控制只有10个线程同时获取数据库连接保存数据，否则会报错无法获取数据库连 

接。这个时候，就可以使用Semaphore来做流量控制，如代码清单8-7所示。 

```java
public class SemaphoreTest {
    private static final int THREAD_COUNT = 30;
    private static ExecutorService threadPool = Executors.newFixedThreadPool(THREAD_COUNT);
    private static Semaphore s = new Semaphore(10);

    public static void main(String[] args) {
        for (int i = 0; i < THREAD_COUNT; i++) {
            threadPool.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        s.acquire();
                        System.out.println("save data");
                        s.release();
                    } catch (InterruptedException e) {
                    }
                }
            });
        }
        threadPool.shutdown();
    }
}
```



### 8.4 线程间交换数据的Exchanger 

Exchanger（交换者）是一个用于线程间协作的工具类。Exchanger用于进行线程间的数据交 

换。它提供一个同步点，在这个同步点，两个线程可以交换彼此的数据。这两个线程通过 

exchange方法交换数据，如果第一个线程先执行exchange()方法，它会一直等待第二个线程也 

执行exchange方法，当两个线程都到达同步点时，这两个线程就可以交换数据，将本线程生产 

出来的数据传递给对方。 

下面来看一下Exchanger的应用场景。 

Exchanger可以用于遗传算法，遗传算法里需要选出两个人作为交配对象，这时候会交换 

两人的数据，并使用交叉规则得出2个交配结果。Exchanger也可以用于校对工作，比如我们需 

要将纸制银行流水通过人工的方式录入成电子银行流水，为了避免错误，采用AB岗两人进行 

录入，录入到Excel之后，系统需要加载这两个Excel，并对两个Excel数据进行校对，看看是否 

录入一致

## 第9章 Java中的线程池 

Java中的线程池是运用场景最多的并发框架，几乎所有需要异步或并发执行任务的程序 

都可以使用线程池。在开发过程中，合理地使用线程池能够带来3个好处。 

第一：降低资源消耗。通过重复利用已创建的线程降低线程创建和销毁造成的消耗。 

第二：提高响应速度。当任务到达时，任务可以不需要等到线程创建就能立即执行。 

第三：提高线程的可管理性。线程是稀缺资源，如果无限制地创建，不仅会消耗系统资源， 

还会降低系统的稳定性，使用线程池可以进行统一分配、调优和监控。但是，要做到合理利用 

线程池，必须对其实现原理了如指掌

### 9.1 线程池的实现原

当向线程池提交一个任务之后，线程池是如何处理这个任务的呢？本节来看一下线程池 

的主要处理流程，处理流程图如图9-1所示。 

从图中可以看出，当提交一个新任务到线程池时，线程池的处理流程如下。 

1）线程池判断核心线程池里的线程是否都在执行任务。如果不是，则创建一个新的工作 

线程来执行任务。如果核心线程池里的线程都在执行任务，则进入下个流程。 

2）线程池判断工作队列是否已经满。如果工作队列没有满，则将新提交的任务存储在这 

个工作队列里。如果工作队列满了，则进入下个流程。 

3）线程池判断线程池的线程是否都处于工作状态。如果没有，则创建一个新的工作线程 

来执行任务。如果已经满了，则交给饱和策略来处理这个任务。 

![image-20210911170814940](C:\Users\wangky\AppData\Roaming\Typora\typora-user-images\image-20210911170814940.png)

![image-20210911170908303](C:\Users\wangky\AppData\Roaming\Typora\typora-user-images\image-20210911170908303.png)

ThreadPoolExecutor执行execute方法分下面4种情况。 

1）如果当前运行的线程少于corePoolSize，则创建新线程来执行任务（注意，执行这一步骤 

需要获取全局锁）。 

2）如果运行的线程等于或多于corePoolSize，则将任务加入BlockingQueue。 

3）如果无法将任务加入BlockingQueue（队列已满），则创建新的线程来处理任务（注意，执行这一步骤需要获取全局锁）。 

4）如果创建新线程将使当前运行的线程超出maximumPoolSize，任务将被拒绝，并调用 

RejectedExecutionHandler.rejectedExecution()方法。 

ThreadPoolExecutor采取上述步骤的总体设计思路，是为了在执行execute()方法时，尽可能 

地避免获取全局锁（那将会是一个严重的可伸缩瓶颈）。在ThreadPoolExecutor完成预热之后 

（当前运行的线程数大于等于corePoolSize），几乎所有的execute()方法调用都是执行步骤2，而 

步骤2不需要获取全局锁。 

### 9.2 线程池的使用

**线程池的创建** 

```java
new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, milliseconds,runnableTaskQueue, handler);
```

创建一个线程池时需要输入几个参数，如下。 

1）corePoolSize（线程池的基本大小）：当提交一个任务到线程池时，线程池会创建一个线 

程来执行任务，即使其他空闲的基本线程能够执行新任务也会创建线程，等到需要执行的任 

务数大于线程池基本大小时就不再创建。如果调用了线程池的prestartAllCoreThreads()方法， 

线程池会提前创建并启动所有基本线程。 

2）runnableTaskQueue（任务队列）：用于保存等待执行的任务的阻塞队列。可以选择以下几 

个阻塞队列。 

·ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按FIFO（先进先出）原 

则对元素进行排序。 

·LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO排序元素，吞吐量通 

常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列。 

·SynchronousQueue：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用 

移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于Linked-BlockingQueue，静态工 

厂方法Executors.newCachedThreadPool使用了这个队列。 

·PriorityBlockingQueue：一个具有优先级的无限阻塞队列。3）maximumPoolSize（线程池最大数量）：线程池允许创建的最大线程数。如果队列满了，并 

且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是，如 

果使用了无界的任务队列这个参数就没什么效果。 

4）ThreadFactory：用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程设 

置更有意义的名字。使用开源框架guava提供的ThreadFactoryBuilder可以快速给线程池里的线 

程设置有意义的名字，代码如下。 

new ThreadFactoryBuilder().setNameFormat("XX-task-%d").build(); 

5）RejectedExecutionHandler（饱和策略）：当队列和线程池都满了，说明线程池处于饱和状 

态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法 

处理新任务时抛出异常。在JDK 1.5中Java线程池框架提供了以下4种策略。 

·AbortPolicy：直接抛出异常。 

·CallerRunsPolicy：只用调用者所在线程来运行任务。 

·DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。 

·DiscardPolicy：不处理，丢弃掉。 

当然，也可以根据应用场景需要来实现RejectedExecutionHandler接口自定义策略。如记录 

日志或持久化存储不能处理的任务。 

·keepAliveTime（线程活动保持时间）：线程池的工作线程空闲后，保持存活的时间。所以， 

如果任务很多，并且每个任务执行的时间比较短，可以调大时间，提高线程的利用率。 

·TimeUnit（线程活动保持时间的单位）：可选的单位有天（DAYS）、小时（HOURS）、分钟 

（MINUTES）、毫秒（MILLISECONDS）、微秒（MICROSECONDS，千分之一毫秒）和纳秒（NANOSECONDS，千分之一微秒）。



**向线程池提交任务**

可以使用两个方法向线程池提交任务，分别为execute()和submit()方法。 

execute()方法用于提交不需要返回值的任务，所以无法判断任务是否被线程池执行成功。

```java
threadsPool.execute(
  new Runnable() {
  @Override public void run() {
  // TODO Auto-generated method stub 
  } });
```

submit()方法用于提交需要返回值的任务。线程池会返回一个future类型的对象，通过这个 

future对象可以判断任务是否执行成功，并且可以通过future的get()方法来获取返回值，get()方 

法会阻塞当前线程直到任务完成，而使用get（long timeout，TimeUnit unit）方法则会阻塞当前线 

程一段时间后立即返回，这时候有可能任务没有执行完。 

```java
Future<Object> future = executor.submit(harReturnValuetask); 
try {
    Object s = future.get(); 
    } catch (InterruptedException e) { // 处理中断异常 
} catch (ExecutionException e) { // 处理无法执行任务异常 
} finally { // 关闭线程池 
    executor.shutdown(); 
}
```

**关闭线程池** 

可以通过调用线程池的shutdown或shutdownNow方法来关闭线程池。它们的原理是遍历线 

程池中的工作线程，然后逐个调用线程的interrupt方法来中断线程，所以无法响应中断的任务 

可能永远无法终止。但是它们存在一定的区别，shutdownNow首先将线程池的状态设置成 

STOP，然后尝试停止所有的正在执行或暂停任务的线程，并返回等待执行任务的列表，而 

shutdown只是将线程池的状态设置成SHUTDOWN状态，然后中断所有没有正在执行任务的线 

程。 

只要调用了这两个关闭方法中的任意一个，isShutdown方法就会返回true。当所有的任务 

都已关闭后，才表示线程池关闭成功，这时调用isTerminaed方法会返回true。至于应该调用哪 

一种方法来关闭线程池，应该由提交到线程池的任务特性决定，**通常调用shutdown方法来关闭** 

**线程池，如果任务不一定要执行完，则可以调用shutdownNow方法。**

## 第10章 Executor框架 

在Java中，使用线程来异步执行任务。Java线程的创建与销毁需要一定的开销，如果我们 

为每一个任务创建一个新线程来执行，这些线程的创建与销毁将消耗大量的计算资源。同时， 

为每一个任务创建一个新线程来执行，这种策略可能会使处于高负荷状态的应用最终崩溃。 

Java的线程既是工作单元，也是执行机制。从JDK 5开始，把工作单元与执行机制分离开 

来。工作单元包括Runnable和Callable，而执行机制由Executor框架提供。

本节将介绍Executor框架的主要成员：ThreadPoolExecutor、ScheduledThreadPoolExecutor、 

Future接口、Runnable接口、Callable接口和Executors。 



主线程首先要创建实现Runnable或者Callable接口的任务对象。工具类Executors可以把一 

个Runnable对象封装为一个Callable对象（Executors.callable（Runnable task）或 

Executors.callable（Runnable task，Object resule））。 

然后可以把Runnable对象直接交给ExecutorService执行（ExecutorService.execute（Runnable 

command））；或者也可以把Runnable对象或Callable对象提交给ExecutorService执行（Executor- 

Service.submit（Runnable task）或ExecutorService.submit（Callable<T>task））。 

如果执行ExecutorService.submit（…），ExecutorService将返回一个实现Future接口的对象 

（到目前为止的JDK中，返回的是FutureTask对象）。由于FutureTask实现了Runnable，程序员也可 

以创建FutureTask，然后直接交给ExecutorService执行。 

最后，主线程可以执行FutureTask.get()方法来等待任务执行完成。主线程也可以执行FutureTask.cancel（boolean mayInterruptIfRunning）来取消此任务的执行

![image-20210911172502288](C:\Users\wangky\AppData\Roaming\Typora\typora-user-images\image-20210911172502288.png)

## 第11章 Java并发编程实践 

# 二、设计模式



# 三、JVM

## Jvm基础

## JMM

## 调优方案

## 调优参数模板

## 调优实战

# 四、MySQL

## 索引

## mvcc

## 事务

# 五、Spring

# 六、SpringBoot

# 七、SpringCloud

## 7.1 Config

## 7.2 Consul

## 7.3 Ribbon

## 7.4 Feign

## 7.5 Hystrix

## 7.6 zuul

## 7.7 gateway

# 八、Dubbo

# 九、MyBatis

# 十、消息队列

## 10.1 ActiveMQ

## 10.2 RibbitMQ

## 10.3 RocketMQ

## 10.4 Kafka

# 十一、缓存

## Redis

# 十二、zookeeper

# 十三、高并发

## 13.1 高并发概念

## 13.2 解决方案

## 13.3 项目实战场景

# 十四、高可用

## 14.1 高可用概念

## 14.2 解决方案

## 14.3 项目实战场景

# 十五、常见解决方案

## 15.1 分布式事务解决方案

## 15.2 分布式锁解决方案

## 15.3 分布式服务追踪与调用链

## 15.4 分布式生成全局ID

## 15.5 分布式日志收集

## 15.6 开放平台设计

## 15.7 红包设计

## 15.8 秒杀设计

## 15.9 十万tps的方案设计（下单）

## 15.10 数据迁移后校对方案（30多亿）

# 十六、数据结构和算法

# 十七、做的最成功项目

