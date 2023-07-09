# Java内存模型

## Java内存模型基础

### 并发编程模型需要解决的问题
并发编程中我们需要解决两个问题：

- 线程之间如何通信

- 线程之间如何同步

#### 线程之间的通信
通信是指线程之间进行信息交换，有两种方式：共享内存、消息传递

- 共享内存：在共享内存并发模型中，线程之间共享程序的公共状态，通过写-读内存中的公共状态进行隐式通信

- 消息传递：消息传递并发模型中，线程之间没有公共状态，线程之间必须通过发送消息进行显示通信

#### 线程之间的同步
同步指程序中用于控制不同线程间操作顺序发生相对顺序的机制。共享内存并发模型中，同步是显示进行，必须显示指定某个方法或某段代码需要在线程之间
互斥执行；在消息传递并发模型中，由于消息发送一定在消息接收之前，因此同步是隐式进行

#### Java中的并发模型
在Java中并发模型采用共享内存模型，Java线程间的通信是隐式进行，通信过程对开发人员透明。

### Java内存模型的抽象结构
Java中所有实例域、静态域和数组元素都存储在堆内存中，堆内存在线程之间共享。而局部变量、方法参数、异常处理器参数不会在线程间共享，也就不存在
内存可见性问题，也不受内存模型影响

Java线程之间的通信由Java内存模型控制，JMM决定一个线程对共享变量的写入何时对另一个线程可见。JMM定义了线程与内存之间的抽象关系：线程之间的
共享变量存储在主内存中，每个线程都有一个私有的本地内存，本地内存中存储了该线程读/写共享变量的副本

![img.png](img.png)

线程A要和线程B通信，需要经历如下两个步骤

- 线程A把本地内存中更新过的共享变量刷新到主内存中

- 线程B到主存中读取线程A之前已更新过的共享变量

### 源代码到指令序列的重排序
Java为提升程序的执行效率，编译器和处理器常常会对指令做重排序，重排序分为三种类型

- 编译器优化的重排序：编译器在不改变单线程程序执行语言的前提下，重新安排语句的执行顺序

- 指令级并行的重排序：现代处理器采用指令级并行技术来将多条指令重叠执行。如果不存在依赖性，处理器可以改变语句对应机器指令的执行顺序

- 内存系统的重排序：由于处理器使用缓存和读/写缓冲区，使得加载和存储操作看上去可能是在乱序执行

从Java源代码到最终实际执行的指令序列，会分别经历下面3种重排序，第一种是编译器重排序，后两种是处理器重排序

![img_2.png](img_2.png)

这些重排序可能导致内存可见性问题，因此对于JMM的编译器重排序规则会禁止特定类型的编译重排序；对于处理器重排序，JMM的处理器重排序规则会要求Java
编译器在生成指令序列时，插入特定类型的内存屏障指令，通过内存屏障指令禁止特定类型重排序

### 并发编程模型分类
处理器使用写缓冲区来临时保存向内存写入的数据，写缓冲区可以保证指令流水线持续运行，避免由处理器停顿等待向内存写入数据而产生的延迟。并且通过
批处理方式刷新写缓冲区，以及合并写缓冲区对同一内存地址的多次写，减少对内存总线的占用。但每个处理器上的写缓冲区，仅仅对它所在的处理器可见。这
个特性会导致处理器对内存的读/写操作顺序与内存实际发生的读/写操作顺序不一定一致

假设处理器A执行如下代码
```
a = 1; //A1
x = b; //A2
```

处理器B执行如下代码：
```
b = 2; //B1
y = a; //B2
```

处理器A、B按程序顺序并行执行内存访问，初始状态a = b = 0，最终可能得到的结果是x = y = 0。由于处理器A、B可以同时把共享变量写入自己写缓存中
即A1、B1，然后从共享内存中读取另一个共享变量A2、B2，最后把写缓存中的脏数据刷新到内存中，按这种时序执行最后就会得到x = y = 0的结果

实际上只有当写缓存中的数据刷新到内存中时，A1操作才算正在完成。因此虽然处理器A执行内存操作的顺序是：A1 -> A2，但内存操作的实际发生顺序是: A2 -> A1，
我们可以看到处理器的内存操作顺序被重排了

常见的处理器都应许对Store-Load操作重排序，都不允许对存在数据依赖项的操作重排序。为保证内存可见性，编译器在生成指令时会在适当位置添加内存屏障指令
来禁用特定类型的处理器重排序。JMM把内存屏障指令分为4类

屏障类型             | 指令示例                 | 说明
------------------- | ------------------------ | -----------  
LoadLoad Barriers   | Load1;LoadLoad;Load2     | 确保Load1数据的装载先于Load2及所有后续装载指令的装载
StoreStore Barriers | Store1;StoreStore;Store2 | 确保Store1刷新数据到内存先于Store2
LoadStore Barriers  | Load1;LoadStore;Store2   | 确保Load1的装载数据先于Store2及之后的存储指令刷新数据到内存
StoreLoad Barriers  | Store1;StoreLoad;Load2   | 确保Store1刷新数据到内存先于Load2及所有之后的装载指令的装载

### happens-before
从JDK 5开始，Java使用新的JSR-133内存模型。在JSR-133内存模型中使用happens-before概念来阐述操作之间的内存可见性。如果一个操作的执行结果需要
对另一个操作可见，这两个操作之间必须存在happens-before关系。这里说的两个操作既可以是同一个线程，也可以是两个不同的线程

与我们开发人员密切相关的happens-before关系如下

- 程序顺序规则：一个线程中的每个操作，happens-before于这个线程的后续操作

- 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁

- volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读

- 传递性：如果A happens-before于B，B happens-before于C，那么A happens-before于C

***两个操作之间具有happens-before关系并不能说前一个操作必须在后一个操作之前执行，只要保证前一个操作执行结果对后一个操作可见，且前一个操作按顺序排在第二个操作之前***

## 重排序
重排序是指编译器和处理器为了执行性能而重新排序执行指令序列的行为。

### 数据依赖性
如果两个操作访问同一个变量，且这两个操作中有一个为写操作，此时这两个操作之间就存在数据依赖性

名称                 | 代码示例                  | 说明
------------------- | ------------------------ | -----------  
写后读               |      a = 1; b = a;       | 写一个变量之后，再读这个变量
写后写               |      a = 1; a = 2;       | 写一个变量之后，再写一个变量
读后写               |      a = b; b = 1;       | 读一个变量之后，再写这个变量

上面的3种情况只要重排序两个操作的执行顺序，程序的执行结果就会被改变。编译器和处理器不会对存在数据依赖性的两个操作重排序，改变执行顺序。但
这里说的数据依赖性仅仅指单个处理器中执行的指令序列和单个线程中执行的操作，不同处理器和不同线程之间的数据依赖性编译器和处理器不会考虑

### as-if-serial语言
as-if-serial语言的意思是：不管怎么重排序程序的执行结果不能被改变。编译器、处理器、runtime都必须遵守as-if-serial语义。as-if-serial语义
把单线程程序保护起来，给用户的感觉就是单线程程序是按程序的顺序来执行的，用户无需担心单线程执行的内存可见性问题

### 重排序对线程的影响
```
package org.aim.reorder;

public class ReorderExample {
    int a = 0;
    boolean flag = false;

    public void writer() {
        a = 1;             // 1
        flag = true;       // 2
    }

    public void reader(){
        if (flag){         // 3
            int i = a * a; // 4
        }
    }
}
```
上述代码片段中flag用于标识变量a是否被写入。假设有两个线程A和B，A先执行writer()方法,随后B执行reader()方法。线程B在执行操作4时并不一定能
看到线程A对变量a的操作。由于操作1、2没有依赖关系，因此编译器和处理器可以对两个操作重排序，同样操作3、4也没有数据依赖关系，因此也可以进行重
排序。当操作1、2重排序时，可能出现线程A先执行操作2，接着线程B执行了操作3、4，之后线程A执行了操作1，在这种情况下线程B是没有看到对变量a的修改。

当我们对操作3、4重排序时，由于操作3、4存在控制依赖关系。因此编译器和处理器会采用猜测执行来克服控制相关性对并行度的影响。执行线程B的处理器可以
提前读取并计算a*a，并将计算结果临时保存在重排序缓冲的硬件缓存中。当操作3的判断为true时就将结果写入变量i中。在单线程中，对存在控制依赖的操作
重排序，不会改变执行结果，但在多线程中，对存在控制依赖的操作重排序，可能改变程序执行结果

## 顺序一致性
顺序一致性内存模型是一个理论参考模型，在设计时，处理器的内存模型和编程语言的内存模型都是以顺序一致性内存模型为参照

### 数据竞争与顺序一致性
程序如果未正确同步，可能会出现数据竞争。Java内存模型对数据竞争的定义如下

- 在一个线程中写一个变量，在另一个线程读取同一个变量，而且写和读没有通过同步来排序

代码中包含数据竞争时，程序的执行往往产生超出预期的结果。一个多线程程序正确同步，则将会是一个没有数据竞争的程序。JMM对正确同步的多线程的内存
一致性做如下保证

- 如果程序正确同步，程序执行将具有顺序一致性。即程序的执行结果与改程序在顺序一致性内存模型的执行结果相同

### 顺序一致性内存模型
顺序一致性内存模型是一个理想化的理论参考模型，提供了强内存可见性保证，具有以下两大特性

- 一个线程中的所有操作必须按照程序的顺序来执行

- 不论线程是否同步，所有线程都只能看到一个单一的操作执行顺序。在顺序一致性内存模型中，每个操作都必须原子执行且立即对所有线程可见

顺序一致性模型有一个单一的全局内存，每个时刻只能有一个线程连接到内存，每个线程必须按照程序的顺序来执行内存读/写操作。在多线程并发执行时，所有
线程的所有内存读/写是串行化的。对于正确同步的多线程，不但线程内部的执行是顺序的，线程之间也是顺序的，一个线程执行完再执行另一个线程；对于未
同步的多线程，线程之间是没有顺序的，但线程内的执行是有顺序的，也就是内存的每次操作可能是A线程也可能是B线程，并且每个线程看到的内存操作顺序
都是一样的

在JMM中，未同步线程不但线程间的执行是无序的，所有线程看到的操作执行顺序也是不一致的。因为每个线程有自己的缓存，如果没有把操作修改的结果从缓存
刷新到主内存中，这个操作就仅对当前线程可见。只有刷新到主内存中，其它线程才能看到这个操作

### 同步程序的顺序一致性
```
package org.aim.reorder;

public class ReorderExample {
    int a = 0;
    boolean flag = false;

    public synchronized void writer() {
        a = 1;             // 1
        flag = true;       // 2
    }

    public synchronized void reader(){
        if (flag){         // 3
            int i = a * a; // 4
        }
    }
}
```
对于正确同步的程序，在多线程并发执行时，JMM中对于同步代码块内的代码可以重排序，在获取同步锁进入同步代码块和退出同步代码块时间点，具有和顺序一致性模型
相同的内存视图，也就是说线程A执行write方法，线程B执行reader方法。线程A在操作1、2的重排序，对于线程B是无感的


### 未同步程序执行特性
未同步或未正确同步的多线程，JMM只提供最小安全性：线程执行时读取到的值，要么是之前某个线程写入的值，要么是默认值。为实现最小安全性，JVM在堆上
分配对象时，首先会对内存空间进行清零，然后才会在上面分配对象

JMM不保证未同步程序执行结果与该程序在顺序一致性模型中的执行结果一致。因为JMM不会去禁用处理器和编译器的优化，如果禁用这些优化会对程序执行性能
产生很大影响。并且未同步程序在顺序一致性模型中执行时，整体是无序的，执行结果无法预知

未同步程序在JMM和顺序一致性模型中执行有如下差异

- 顺序一致性模型保证单线程内的操作会按程序顺序执行，而在JMM不保证单线程内的操作会按程序的顺序执行

- 顺序一致性模型保证所有线程只能看到一致的操作执行顺序，JMM不保证所有线程能看到一致的操作执行顺序

- JMM不保证对long型和double型变量的写操作具有原子性，顺序一致性模型保证对所有内存读/写操作都具有原子性

计算机中数据是通过总线在内存与处理器之间传递。处理器与内存的每次数据传递都是通过一系列步骤完成，这一系列的步骤称为总线事务。总线事务包括读
事务和写事务。读事务负责把内存中的数据传递到处理器，写事务负责把处理器中的数据传递到内存。多处理器同时向总线发起总线事务时，总线会裁决选择一个
处理器进行总线事务,其他处理器的总线事务需要等到上一个正在执行的总线事务结束才行。总线事务的串行化执行确保了单个总线事务之内的内存读/写操作具有原子性。
对于一些32位的处理器，如果对64位数据进行需要是原子的，需要有较大的开销。因此Java语言规范鼓励但不强求JVM对64位的long型变量和double型变量的
写操作具有原子性。在这种处理器上，可能会把64位变量的写操作拆分成两个32位写操作，这两个操作可能分配到不同的总线事务中，因此对64位变量的写操作
将不具有原子性

## volatile的内存语义

### volatile特性
volatile修饰变量单个读/写操作，可以看成是使用同一个锁对单个读/写操作做了同步。根据锁的happens-before规则保证释放锁和获取锁两个线程的内存可见性，
对volatile变量的读，总是能看到任意线程最后对这个变量的写入。即使64位变量被volatile修饰，其读/写操作同样是原子的。但对volatile变量的复合操作将
不具有原子性。volatile修饰的变量具有以下特性

- 可见性：对一个volatile变量的读操作，总是能看到任意线程对这个变量最后的写入

- 原子性：对任意单个volatile变量的读/写操作具有原子性。但是复合操作不具有原子性

### volatile写-读内存语义
volatile写的内存语义：当写一个volatile变量，JMM会把该线程对应的本地内存中的共享变量刷新到主内存

volatile读的内存语义：当读一个volatile变量，JMM会把该线程对应的本地内存置为无效。直接从主内存读取共享变量，并更新本地内存共享变量使其与主内存中一致

### volatile内存语言的实现
为实现volatile内存语义，JMM会对编译器重排序和处理器重排序做限制

<table width="401.55" border="0" cellpadding="0" cellspacing="0" style='width:401.55pt;border-collapse:collapse;table-layout:fixed;'>
   <col width="82.45" style='mso-width-source:userset;mso-width-alt:3517;'/>
   <col width="89.95" style='mso-width-source:userset;mso-width-alt:3837;'/>
   <col width="105" style='mso-width-source:userset;mso-width-alt:4480;'/>
   <col width="124.15" style='mso-width-source:userset;mso-width-alt:5297;'/>
   <tr height="17.60" style='height:17.60pt;'>
    <td height="17.60" width="82.45" style='height:17.60pt;width:82.45pt;' x:str>是否能重排序</td>
    <td class="xl65" width="319.10" colspan="3" style='width:319.10pt;border-right:none;border-bottom:none;' x:str>第二个操作</td>
   </tr>
   <tr height="17.60" style='height:17.60pt;'>
    <td height="17.60" style='height:17.60pt;' x:str>第一个操作</td>
    <td x:str>普通读/写</td>
    <td x:str>volatile读</td>
    <td x:str>volatile写</td>
   </tr>
   <tr height="17.60" style='height:17.60pt;'>
    <td height="17.60" style='height:17.60pt;' x:str>普通读/写</td>
    <td colspan="2" style='mso-ignore:colspan;'></td>
    <td x:str>NO</td>
   </tr>
   <tr height="17.60" style='height:17.60pt;'>
    <td height="17.60" style='height:17.60pt;' x:str>volatile读</td>
    <td x:str>NO</td>
    <td x:str>NO</td>
    <td x:str>NO</td>
   </tr>
   <tr height="17.60" style='height:17.60pt;'>
    <td height="17.60" style='height:17.60pt;' x:str>volatile写</td>
    <td></td>
    <td x:str>NO</td>
    <td x:str>NO</td>
   </tr>
   <![if supportMisalignedColumns]>
    <tr width="0" style='display:none;'>
     <td width="82" style='width:82;'></td>
     <td width="90" style='width:90;'></td>
     <td width="105" style='width:105;'></td>
     <td width="124" style='width:124;'></td>
    </tr>
   <![endif]>
  </table>

- 第二个操作为volatile写时，无论第一个操作是什么，都不能重排序，确保volatile写之前的操作不会被编译器重排序到volatile写之后

- 第一个操作为volatile读时，无论第二个操作为什么，都不能重排序，确保volatile读之后的操作不会被编译器重排序到volatile读之前

- 第一个操作为volatile写时，第二个操作为volatile读，不能重排序

为实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。对编译器来说，发现一个最优布置来最
小化插入屏障的总数几乎不可能。为此JMM采取保守策略，以下为保守策略的JMM内存屏障插入策略

- 每个volatile写操作前面插入一个StoreStore屏障

- 每个volatile写操作后面插入一个StoreLoad屏障

- 每个volatile读操作的后面插入一个LoadLoad屏障

- 每个volatile读操作的后面插入一个LoadStore屏障

上述屏障插入策略非常保守，在实际执行时只要保证volatile写-读内存语义，编译器可以根据具体情况省略没有必要的屏障。以下述代码执行为例
```
class VolatileBarrierExample {
    int a;
    volatile int v1 = 1;
    volatile int v2 = 2;
    
    void readAndWrite() {
        int i = v1;    // 第一个volatile读
        int j = v2;    // 第二个volatile读
        a = i + j;     // 普通写
        v1 = i + 1;    // 第一个volatile写
        v2 = j * 2;    // 第二个volatile写 
    }
}
```
编译器在生成字节是可以做如下优化

![img_4.png](img_4.png)

## 锁的内存语义

### 锁的释放-获取建立的happens-before关系
锁除了让临界区互斥外，还可以让释放锁的线程向获取同一个锁的线程发送消息

```
class MonitorExample {
    int a = 0;
    
    public synchronized void writer() {     // 1
        a++;                                // 2
    }                                       // 3
    
    public synchronized void reader() {     // 4
       int i = a;                           // 5
       ......                               
    }                                       // 6
}
```
线程A先执行writer()方法，随后线程B执行reader()方法。根据happens-before规则，这个过程包含happens-before关系可分为3类

- 1 happens-before 2，2 happens-before 3; 4 happens-before 5, 5 happens-before 6

- 根据监视器锁规则，3 happens-before 4

- 根据happens-before的传递性，2 happens-before 5

### 锁的释放和获取的内存语义
***当线程释放锁时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存中***

***当线程获取锁时，JMM会把该线程对应的本地内存置为无效。使得监视器保护的临界区代码必须从主内存中读取共享变量***

对比锁的释放-获取与volatile变量的写-读可知：锁的释放与volatile写具有相同内存语义；锁的获取与volatile读具有相同的内存语义

锁的释放-获取内存语义总结

- 线程A释放一个锁，实质是线程A向接下来将要获取这个锁的某个线程发出了（线程A对共享变量所做修改的）消息

- 线程B获取一个锁，实质是线程B接收到之前某个线程发出的（在释放锁之前对共享变量所做修改的）消息

- 线程A释放锁，随后线程B获取这个锁，这个过程实质是线程A通过主内存向线程B发送消息

### 锁内存语义的实现
通过ReentrantLock锁的使用以及其源码我们了解一下锁内存语言的具体实现机制
```
public class ReentrantLockExample {
    int a = 0;
    ReentrantLock lock = new ReentrantLock(true);


    public static void main(String[] args){
        new ReentrantLockExample().writer();
    }

    public void writer() {
        lock.lock();            // 获取锁
        try {
            a++;
        } finally {
            lock.unlock();     // 释放锁
        }
    }

    public int reader() {
        lock.lock();          // 获取锁
        try {
            return a;
        } finally {
            lock.unlock();     // 释放锁
        }
    }
}
```
ReentrantLock包括两种锁，一种是公平锁；另一种是非公平锁。在新建ReentrantLock对象时可以指定使用公平锁还是非公平锁，默认使用非公平锁。在
ReentrantLock使用lock()方法获取锁；使用unlock()方法释放锁

ReentrantLock实现依赖于AbstractQueuedSynchronizer。AbstractQueuedSynchronizer属性state被volatile修饰，被用于维护同步状态

我们先看一下其公平锁的加锁过程

- 调用ReentrantLock对象的lock()方法实现加锁

- 内部调用FairSync内部类的lock()方法
  
- 接着调用AbstractQueuedSynchronizer的acquire方法传入参数1

- acquire方法首先会调用FairSync类的tryAcquire方法传入参数1，此方法实现真正的加锁

  - 首先获取当前线程及stage状态字段
    
  - 如果state状态字段为0，则表明锁没有被获取。通过hasQueuedPredecessors()方法查看是否有排在当前线程前面的线程还没有获得锁，如果没有这样的线程，
  则调用compareAndSetState方法尝试加锁，如果加锁成功，则将当前线程设置到exclusiveOwnerThread属性中
    
  - 如果state状态字段不为0，则表明已经有线程获取了锁，判断是不是当前线程持有锁，如果当前线程已经持有锁，则将state值加1，如果state值小于0，则
  表明超过了最大锁计数值
    
- tryAcquire方法加锁失败后，调用AbstractQueuedSynchronizer类的acquireQueued方法将当前线程加入到获取锁的线程队列中，并中断当前线程
 
```
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

公平锁与非公平锁的解锁过程相同，调用轨迹如下

- 调用ReentrantLock的unlock方法实现解锁

- unlock方法继续调用AbstractQueuedSynchronizer的release(int arg)方法并传入参数1

- release方法中会尝试释放锁并唤醒被阻塞的线程

  - 调用tryRelease方法完成锁的释放.其处理逻辑是，使用state字段记录的值减1并赋值c，如果c为0，则解锁成功，并置空exclusiveOwnerThread属性，否则解锁失败,
  不管解锁成功还是失败都会设置state字段值为c
  ```
  protected final boolean tryRelease(int releases) {
      int c = getState() - releases;
      if (Thread.currentThread() != getExclusiveOwnerThread())
          throw new IllegalMonitorStateException();
      boolean free = false;
      if (c == 0) {
          free = true;
          setExclusiveOwnerThread(null);
      }
      setState(c);
      return free;
  }
  ```

  - 如果解锁成功，调用unparkSuccessor方法唤醒后续的阻塞线程并使用compareAndSetWaitStatus方法即CAS操作，原子的更新state状态
  ```
  private void unparkSuccessor(Node node) {
      /*
       * If status is negative (i.e., possibly needing signal) try
       * to clear in anticipation of signalling.  It is OK if this
       * fails or if status is changed by waiting thread.
       */
      int ws = node.waitStatus;
      if (ws < 0)
          compareAndSetWaitStatus(node, ws, 0);

      /*
       * Thread to unpark is held in successor, which is normally
       * just the next node.  But if cancelled or apparently null,
       * traverse backwards from tail to find the actual
       * non-cancelled successor.
       */
      Node s = node.next;
      if (s == null || s.waitStatus > 0) {
          s = null;
          for (Node t = tail; t != null && t != node; t = t.prev)
              if (t.waitStatus <= 0)
                  s = t;
      }
      if (s != null)
          LockSupport.unpark(s.thread);
  }
  ```
  
ReentrantLock源码分析可以发现，锁释放-获取内存语义实现至少有下面两种方式

- 利用volatile变量的写-读所具有的内存语义

- 利用CAS所附带的volatile读和volatile写的内存语义

### concurrent包的实现
Java CAS操作同时具有volatile读和volatile写的内存语义，因此线程之间的通信会有4种方式

- A线程写volatile变量，随后B线程读这个volatile变量

- A线程写volatile变量，随后B线程用CAS更新这个volatile变量

- A线程用CAS更新一个volatile变量，随后B线程用CAS更新这个volatile变量

- A线程用CAS更新一个volatile变量，随后B线程读这个volatile变量

CAS会使用现代处理器上提供的高效机器级别的原子操作，这些原子指令以原子方式执行读-改-写操作，这是在多处理器中实现同步的关键。同时volatile变量的
读/写和CAS可以实现线程之间的通信。而concurrent包实现的基础就是基于volatile和CAS操作，其通用化的实现模式如下

- 首先申明共享变量为volatile
  
- 之后使用CAS原子条件更新来实现线程之间的同步

- 配合volatile读/写和CAS所具有的volatile读和写的内存语言来实现线程之间的通信

AQS、非阻塞数据结果、原子变量类，这些concurrent包中的基础类都是使用这种模式来实现，concurrent包中的高层类又是依赖于这些基础类来实现

## final域的内存语义
与锁和volatile相比，对final域的读和写更像是普通的变量访问

### final域的重排序规则
对final域，编译器和处理器需要遵循两个重排序规则

- 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序

- 初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序

示例程序如下
```
public class FinalDemo {
    int i;         //普通变量
    final int j;   // final 变量
    static FinalDemo obj;
    public FinalDemo(){             // 构造函数
        i = 1;                      // 写普通域
        j = 2;                      // 写final域
    }

    public static void writer(){    // 写线程A执行
        obj = new FinalDemo();
    }

    public static void reader(){    // 读线程B执行
        FinalDemo object = obj;     // 读对象引用
        int a = object.i;           // 读普通域
        int b = object.j;           // 读final域
    }
}
```

### 写final域的重排序规则
写final域重排序规则禁止把写final域重排序到构造函数之外，其中包含两层意思

- JMM禁止编译器把final域的写重排序到构造函数之外

- 编译器会在final域写之后，构造函数return之前，插入一个StoreStore屏障，这个屏障禁止处理器把final域的写重排序到构造函数之外

上例，我们假设线程A先执行writer方法，随后线程B执行reader方法。在writer方法中只有一个操作就是初始化obj对象，其实现包含两个步骤

- 调用构造函数，构造一个FinalDemo对象

- 将构造的对象赋值给obj属性

假设线程B读对象引用与读对象成员域之间没有重排序，则可能的执行时序如下
![img_5.png](img_5.png)

在上述可能的执行时序中，写普通域被重排序到了构造函数之外，读线程B错误的读取到了普通变量i初始化之前的值。写final域的操作，被写final域的重
排序规则限定在构造函数内，线程B正确的读取到了final域中初始化之后的值

写final域的重排序规则可以确保：在对象引用为任意线程可见之前，对象的final域已经被正确初始化过了，而普通域不具有这个保障。在上述过程中，读线程
B看到obj对象时，对象很有可能还没有完成构造

### 读final域重排序规则
读final域的重排序规则是，在一个线程中，初次读对象引用与初次读该对象的final域，JMM禁止处理器重排序这两个操作，这个规则仅针对处理器，编译器
会在读final域操作的前面插入LoadLoad屏障

初次读对象引用与初次读对象包含的final域，这两个操作具有间接依赖关系。编译器遵守间接依赖关系，因此编译器不会重排序这两个操作。大多数处理器也
遵循间接依赖，不会重排序这两个操作。但少数处理器允许对间接依赖关系操作重排序，这个规则就是针对这种处理器

### final域为引用类型
如下为final域为引用类型的例子
```
public class FinalReferenceDemo {
    final int[] intArrays;                   // final域为引用类型

    static FinalReferenceDemo obj;

    public FinalReferenceDemo() {            // 构造函数
        intArrays = new int[1];              // 1
        intArrays[0] = 1;                    // 2
    }

    public static void writerOne() {          // 写线程A执行
        obj = new FinalReferenceDemo();       // 3
    }

    public static void writerTwo() {          // 写线程B执行
        obj.intArrays[0] = 2;                 // 4
    }

    public static void reader() {             // 读线程C执行
        if (obj != null) {                    // 5
            int temp1 = obj.intArrays[0];     // 6
            System.out.println(temp1);
        }
    }

    public static void main(String[] args) {
        Thread tA = new Thread(new Runnable() {
            @Override
            public void run() {
                writerOne();
            }
        });
        tA.start();

        Thread tB = new Thread(new Runnable() {
            @Override
            public void run() {
                writerTwo();
            }
        });
        tB.start();

        Thread tC = new Thread(new Runnable() {
            @Override
            public void run() {
                reader();
            }
        });
        tC.start();
    }
}
```
对写final域的重排序规则对编译器和处理器增加如下约束：构造函数内对一个final引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用
赋值给一个引用变量，这两个操作之间不能重排序

假设首先线程A执行writeOne()方法，执行完成后线程B执行writeTwo()方法，执行完后线程C执行reader()方法。操作1、2与操作3都不能重排序，JMM可以
保证读线程C至少能看到写线程A在构造函数中对final引用对象的成员域的写入。即C至少能看到数组下标0的值为1。写线程B对数组元素的写入，读线程C可能
看不到，也可能看得到。JMM不保证线程B的写入对线程C可见，因为写线程B和读线程C之间存在数据竞争。如果要保证读线程C看到写线程B对数组元素的写入，
写线程B和读线程C之前需要使用同步原语（lock或volatile）来确保内存可见性

### final引用为什么不能从构造函数内"溢出"
写final域重排序规则可以确保：在引用变量为任意线程可见之前，该引用变量指向的对象的final域已经在构造函数中被正确初始化过。需要得到这个效果，还需要
一个保证：在构造函数内部，不能让这个被构造对象的引用为其他线程可见，就是说对象引用不能在构造函数中逸出。
```
package org.aim.finalkeyword;

public class FinalReferenceEscapeExample {
    final int i;
    static FinalReferenceEscapeExample obj;

    public FinalReferenceEscapeExample() {
        i = 1;                            //1 写final域
        obj = this;                       //2 this引用在此逸出
    }

    public static void writer() {
        new FinalReferenceEscapeExample();
    }

    public static void reader() {
        if (obj != null) {
            int temp =obj.i;
            System.out.println(temp);
        }
    }

    public static void main(String[] args){
        Thread A = new Thread(new Runnable() {
            @Override
            public void run() {
                writer();
            }
        });
        A.start();

        Thread B = new Thread(new Runnable() {
            @Override
            public void run() {
                reader();
            }
        });
        B.start();
    }
}
```
假设线程A执行writer方法，另一线程B执行reader方法。上面示例代码操作2使对象还没有完成构造前就为线程B可见。虽然操作2是最后一行代码，操作1
排在操作2前面，但操作1与操作2之间还是可能被重排序。可能出现如下的执行时序
![img_6.png](img_6.png)

在构造函数返回前，被构造对象的引用不能为其他线程所见，因为此时的final域可能还没有被初始化。在构造函数返回后，任意线程都将保证能看到final域
初始化后的值