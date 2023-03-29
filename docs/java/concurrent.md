# Java 并发篇

Java 并发首先要搞清楚**为什么**会有并发问题（为了解决 CPU、内存、IO 速度不一致，加快处理），通过引入多线程和缓存**解决这个问题**，但是又产生了**新问题**（可见性、原子性、有序性），**怎么解决这些新问题**（同步锁、无锁方式），**具体用法**（synchronized、JUC）。

## 多线程基础

### 线程状态（生命周期）有哪些？状态如何转变？

JDK 在 java.lang.Thread.State 里定义了 6 种线程状态：

```
public enum State {

    /**
     * 新建状态：尚未启动的线程的状态
     */
    NEW,

    /**
     * 可运行状态：可能正在运行，也可能正在等待 CPU 时间片。调用 start() 进入。
     */
    RUNNABLE,

    /**
     * 阻塞状态：等待获取一个排它锁，如果其线程释放了锁就会结束此状态。运行中的线程发生锁竞争的时候进入，获取到锁资源回到运行状态。
     */
    BLOCKED,

    /**
     * 无限期等待状态：等待其它线程显式地唤醒，否则不会被分配 CPU 时间片。
     * 进入方法：
     *    1. 调用没有设置 Timeout 参数的 Object.wait() 进入，调用 Object.notify() / Object.notifyAll() 退出；
     *    2. 调用没有设置 Timeout 参数的 Thread.join() 进入，被调用的线程执行完毕退出；
     */
    WAITING,

    /**
     * 限期等待：无需等待其它线程显式地唤醒，在一定时间之后会被系统自动唤醒。
     * 进入方法：
     *    1. 调用 Thread.sleep() 进入，时间结束退出；
     *    2. 调用设置了 Timeout 参数的 Object.wait() 进入，时间结束或调用 Object.notify() / Object.notifyAll() 退出；
     *    3. 调用设置了 Timeout 参数的 Thread.join()进入，时间结束或被调用的线程执行完毕退出；
     */
    TIMED_WAITING,

    /**
     * 终止状态：可以是线程结束任务之后自己结束，或者产生了异常而结束。
     */
    TERMINATED;
}
```

### 线程有哪些常见方法？

1. **sleep()**：休眠当前正在执行的线程。可能会抛出 InterruptedException，因为异常不能跨线程传播回 main() 中，因此必须在本地进行处理。线程中抛出的其它异常也同样需要在本地进行处理。

1. **setDaemon()**：设置守护线程。守护线程是程序运行时在后台提供服务的线程，不属于程序中不可或缺的部分。当所有非守护线程结束时，程序也就终止，同时会杀死所有守护线程。main() 属于非守护线程。

1. **yield()**：对静态方法 Thread.yield() 的调用声明了当前线程已经完成了生命周期中最重要的部分，可以切换给其它线程来执行。该方法只是对线程调度器的一个建议，而且也只是建议具有相同优先级的其它线程可以运行。

### 线程的中断方式有哪些？
1. 如果该线程处于阻塞、限期等待或者无限期等待状态，调用 interrupt() 会中断线程，抛出 InterruptedException 异常。但是不能中断 I/O 阻塞和 synchronized 锁阻塞。

1. 如果是运行状态，调用 interrupt() 不会中断线程，但会设置中断标记，导致 interrupted() 返回true，可以通过 interrupted() 返回值手动中断。

1. 调用 Executor 的 shutdown() 方法会等待线程都执行完毕之后再关闭，但是如果调用的是 shutdownNow() 方法，则相当于调用每个线程的 interrupt() 方法。

### 线程之间有哪些协作方式？
1. **join()**：在线程中调用另一个线程的 join() 方法，会将当前线程挂起，而不是忙等待，直到目标线程结束。

1. **wait()、notify()、notifyAll()**：调用 wait() 使得线程等待某个条件满足，线程在等待时会被挂起，当其他线程的运行使得这个条件满足时，其它线程会调用 notify() 或者 notifyAll() 来唤醒挂起的线程。

   它们都属于 Object 的一部分，而不属于 Thread。

   只能用在同步方法或者同步控制块中使用，否则会在运行时抛出 IllegalMonitorStateExeception。

   使用 wait() 挂起期间，线程会释放锁。这是因为，如果没有释放锁，那么其它线程就无法进入对象的同步方法或者同步控制块中，那么就无法执行 notify() 或者 notifyAll() 来唤醒挂起的线程，造成死锁。

1. **CountDownLatch**：允许一个或多个线程等待其他线程完成操作。

   **工作原理**：通过一个计数器来实现，计数器初始值为线程的数量，调用 await() 的线程会被阻塞，直到计数器减到 0 才能继续往下执行。通过调用 countDown() 让计数器减 1。底层通过 AQS 实现。

   **与 join() 方法的区别**：join() 必须等待线程执行完成才能继续往下执行，CountDownLatch 不需要，可以在任意地方计数器减到 0 即可继续执行，更灵活。


### sleep 和 wait 有什么区别？
* sleep() 是 Thread 的静态方法，而 wait() 是 Object 的方法；

* sleep 是让出指定 cpu 时间，时间到了恢复就绪，**不会释放对象锁**；wait 会放弃对象锁，进入锁等待池，需要 notify 方法恢复就绪；

### notify() 和 notifyAll() 有什么区别？
notify 是随机唤醒一个对象等待池里的线程加入锁池竞争该对象锁，notifyAll 是唤醒对象等待池里所有线程加入锁池竞争对象锁。

### 什么是锁池和等待池？
**锁池**：假设线程 A 已经拥有了某个对象的锁，而其它的线程想要调用这个对象的某个 synchronized 方法，由于这些线程在进入对象的 synchronized 方法之前必须先获得该对象的锁的拥有权，但是该对象的锁目前正被线程 A 拥有，所以这些线程就进入了该对象的锁池中。

**等待池**：假设一个线程 A 调用了某个对象的 wait() 方法，线程 A 就会释放该对象的锁后，进入到了该对象的等待池中。

### 线程死锁问题

**定义**：不同线程互相等待对方释放资源导致谁都不能往下执行

死锁产生原因：
* 锁顺序死锁：多个线程以不同顺序获取相同的锁导致
* 动态锁顺序死锁：多个线程以相同顺序获取相同锁，但是锁住的对象动态改变调换的情况
* 协作对象之间发生死锁：协作对象隐式获取当前对象锁并在业务中显示获取另一个对象锁

如何避免死锁：
* 固定加锁的顺序【针对锁顺序死锁】
* 开放调用【针对协作对象死锁】
* 使用定时锁：tryLock，如果等待获取锁时间超时则抛出异常而不是一直等待

死锁检测：
* Jconsole：JDK 自带的图形化界面工具
* Jstack：JDK 自带的命令行工具，主要用于线程 Dump 分析

## 并发基础

### 如何理解并发和并行的区别？
并发是指一个处理器同时处理多个任务。

并行是指多个处理器或者是多核的处理器同时处理多个不同的任务。

### 并发问题的本质是要解决什么问题？
CPU、内存、I/O 设备的速度是有极大差异的，为了合理利用 CPU 的高性能，平衡这三者的速度差异，计算机体系结构、操作系统、编译程序都做出了贡献，主要体现为：
* CPU 增加了缓存，以均衡与内存的速度差异，导致**可见性**问题：每个线程拥有自己的一个高速缓存区 —— 线程工作内存，普通变量被修改后什么时候写入主存是不确定的，其他线程读取主存可能读到旧值。

* 操作系统增加了进程、线程，以分时复用 CPU，进而均衡 CPU 与 I/O 设备的速度差异，导致**原子性**问题：只有简单的读取、基本类型赋值才是原子操作，其他都不是。

* 编译程序优化指令执行次序，使得缓存能够得到更加合理地利用，导致**有序性**问题。

### Java 怎么解决并发问题？
1. **阻塞同步**：能保证同一时刻只有一个线程修改数据，能同时保证原子性、可见性、有序性，一般通过 synchronized、ReentrantLock实现。但是性能往往不太好。

1. **非阻塞同步**：通过 CAS、volatile 等乐观策略，不阻塞线程，失败重试保证。
1. **无锁方式**：通过 final、ThreadLocal 等实现栈封闭保证线程安全。

### 什么是 volatile？
volatile 是一个 Java 关键字，一般用来修饰共享变量，可以**保证可见性**和**禁止指令重排序**。

* 可见性指：每次读取到 volatile 变量，一定是最新数据；实现原理：

   1. 当线程修改变量时，会强制刷新到主内存中
   1. 当线程读取变量时，会强制从主内存读取变量并且刷新到工作内存中

* JVM 为了获取更好的性能可能会对指令重排序，导致多线程出现有序性问题，volatile 可以禁止指令重排序，保证有序性；会降低一些性能。

### 什么是 CAS ？

CAS 全称是 Compare And Swap（比较并交换），通过原子操作判断 A 值在更新前原值是否仍为 A，如果是：执行更新，如果不是什么都不做。原子性通过**CPU 指令**（基于硬件平台的汇编指令）从硬件层面保证。是一条 CPU 的原子指令，让比较和更新变成一个原子操作。

CAS 为乐观锁，性能通常比悲观锁更好。通过 UnSafe 类的 native 方法实现。

CAS 有哪些**问题**：
1. ABA 问题：可以通过加版本号解决
1. 循环时间长开销大：锁竞争激烈时频繁重试带来的自旋会给 CPU 带来很大的开销
1. 只能保证一个共享变量的原子操作

### 常见锁概念

锁机制在宏观上分成悲观锁和乐观锁两大类，**悲观锁阻塞事务，乐观锁回滚重试**，乐观锁适合写少的时候，冲突少，没有加锁的开销，提高吞吐量，悲观锁适合冲突多多。

高并发时，同步调用应该去考量锁的性能损耗。能用无锁数据结构，就不要用锁；能锁区块，就不要锁整个方法体；能用对象锁，就不要用类锁。尽可能使加锁的代码块工作量尽可能的小，避免在锁代码块中调用 RPC 方法。

如果每次访问冲突概率小于20%，推荐使用乐观锁，否则使用悲观锁。

资金相关的金融敏感信息，推荐使用悲观锁策略。

1. **悲观锁**：认为每次获取数据都会被别人修改，所以每次都上锁，直到锁释放才能供下个人使用。

1. **乐观锁**：认为每次获取数据都认为别人不会修改，所以不会上锁，但是如果想要更新数据，则会在更新前检查在读取至更新这段时间别人有没有修改过这个数据，如果修改过，则重新读取，再次尝试更新，循环直到成功。乐观锁其实不是“锁”，它仅仅是一个循环重试 CAS 的算法而已。

1. **自旋锁**：CPU 无限循环获取锁。

1. **公平锁、非公平锁**：如果多个线程申请一把公平锁，那么当锁释放的时候，公平锁先申请的先得到，非公平锁不一定。非公平锁的吞吐量大，一般优先非公平锁。

1. **可重入锁**：允许同一个线程多次获取同一把锁，常用与解决死锁问题。

1. **可中断锁**：可以响应中断的锁。

1. **读写锁**：如果读取仅仅是读数据，加读锁（共享锁），其他线程读不需要等待；如果读取是为了更新，加写锁（互斥锁、排他锁），其他线程读或写都要等待。

### ThreadLocal
ThreadLocal 是 JDK 提供的，用来保证共享变量线程安全的类。通过为每个访问这个变量的线程创建一个副本来保证线程安全。

**实现原理**：每个线程都有一个 threadLocals 变量，是一个数组实现的 Map 结构，存储了 ThreadLocal 实例当做 key，共享变量当做 value 的 ThreadLocalMap。

ThreadLocal 可能会造成**内存泄漏**，**解决方案**：

线程池复用线程的时候，父线程无法将更新后的数据通过 ThreadLocal 传递给子线程的问题，**InheritableThreadLocal** 是怎么解决的还有哪些遗留问题，阿里巴巴开源的 TransmittableThreadLocal 工具类是怎么解决 InheritableThreadLocal 的遗留问题的，最后再讲讲 Skywalking 是怎么跨线程将 TranceID 传递到下一个链路的

## synchronized

synchronized 先天具有**重入性**。在同一锁程中，线程不需要再次获取同一把锁。每个对象拥有一个计数器，当线程获取该对象锁后，计数器就会加一，释放锁后就会将计数器减一。

synchronized 是非公平锁、不可中断锁。

**用法**：synchronized 不能被继承。修饰成员方法或者代码块锁 this 时，锁的是对象实例，修饰静态方法或代码块锁 .class 时锁的是类的 class 对象，并不是同一把锁，可以同时存在。

### synchronized 实现原理？
synchronized 编译后生成 monitorenter、monitorexit 指令，通过这两个指令实现加锁和释放锁。

* **monitorenter 指令**：会让对象在执行时，使其锁计数器加 1，每一个对象在同一时间只与一个 monitor（锁）相关联，而一个 monitor 在同一时间只能被一个线程获得。一个对象在尝试获得与这个对象相关联的 monitor 锁的所有权的时候，会发生如下 3 种情况之一：
   1. monitor 计数器为 0，意味着目前还没有被获得，那这个线程就会立刻获得然后把锁计数器 +1，一旦 +1，别的线程再想获取，就需要等待
   1. 如果这个 monitor 已经拿到了这个锁的所有权，又重入了这把锁，那锁计数器就会累加，变成 2，并且随着重入的次数，会一直累加
   1. 这把锁已经被别的线程获取了，等待锁释放

* **monitorexit 指令**：释放对于 monitor 的所有权，释放过程很简单，就是将 monitor 的计数器减 1，如果减完以后，计数器不是 0，则代表刚才是重入进来的，当前线程还继续持有这把锁的所有权，如果计数器变成 0，则代表当前线程不再拥有该 monitor 的所有权，即释放锁。

### JDK 1.6 对 synchronized 做了哪些优化？

jdk1.6 以前直接加重量锁，1.6 增加了**锁升级优化**，提升性能。只能按照偏向锁、轻量锁、重量锁升级，不允许降级。

synchronized 锁升级过程：
1. 初次执行到 synchronized 代码块时，锁对象是**偏向锁**（偏向于第一个获得它的线程的锁）

1. 一旦有第二个线程加入锁竞争时变成**轻量级锁**（自旋锁）
1. 锁竞争严重，达到最大自旋次数（计数器记录自旋次数，默认允许循环 10 次，可以通过虚拟机参数更改）时变成**重量级锁**，这时其他线程不再自旋，直接挂起，等待被唤醒。

其他优化：

1. **锁粗化**（Lock Coarsening）：也就是减少不必要的紧连在一起的 unlock，lock 操作，将多个连续的锁扩展成一个范围更大的锁。

1. **锁消除**（Lock Elimination）：通过运行时 JIT 编译器的逃逸分析来消除一些没有在当前同步块以外被其他线程共享的数据的锁保护，通过逃逸分析也可以在线程本地 stack 上进行对象空间的分配（同时还可以减少 heap 上的垃圾收集开销）。

1. **轻量级锁**（Lightweight Locking）：这种锁实现的背后基于这样一种假设，即在真实的情况下我们程序中的大部分同步代码一般都处于无锁竞争状态（即单线程执行环境），在无锁竞争的情况下完全可以避免调用操作系统层面的重量级互斥锁，取而代之的是在 monitorenter 和 monitorexit 中只需要依靠一条 CAS 原子指令就可以完成锁的获取及释放。当存在锁竞争的情况下，执行 CAS 指令失败的线程将调用操作系统互斥锁进入到阻塞状态，当锁被释放的时候被唤醒。

1. **偏向锁**（Biased Locking）：是为了在无锁竞争的情况下避免在锁获取过程中执行不必要的 CAS 原子指令，因为 CAS 原子指令虽然相对于重量级锁来说开销比较小但还是存在非常可观的本地延迟。

1. **适应性自旋**（Adaptive Spinning）：当线程在获取轻量级锁的过程中执行 CAS 操作失败时，在进入与 monitor 相关联的操作系统重量级锁（mutex semaphore）前会进入忙等待（Spinning）然后再次尝试，当尝试一定的次数后如果仍然没有成功则调用与该 monitor 关联的 semaphore（即互斥锁）进入到阻塞状态。

### synchronized 和 ReentrantLock 优缺点？比较和选择？
1. **锁的实现**：synchronized 是 JVM 实现的，而 ReentrantLock 是 JDK 实现的。

1. **锁的释放**：synchronized 是获取锁的线程执行完同步代码后或者线程执行发生异常时释放锁，不会产生死锁；Lock 一般在 finally 中释放锁，不然容易造成线程死锁。

1. **锁的获取**：synchronized 只能等待其他线程释放锁后获取锁；Lock 提供 tryLock 可以不等待

1. **性能**：新版本 Java 对 synchronized 进行了很多优化，例如自旋锁等；竞争小的时候 synchronized 略优于 ReentrantLock，数据量大时 synchronized 性能下降严重，ReentrantLock 维持常态。

1. **等待可中断**：当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待，改为处理其他事情。ReentrantLock 可中断，而 synchronized 不行。

1. **公平锁**：公平锁是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁。synchronized 中的锁是非公平的，ReentrantLock 默认情况下也是非公平的，但是也可以是公平的。

1. **锁绑定多个条件**：一个 ReentrantLock 可以同时绑定多个 Condition 对象。

1. **锁的状态**：Lock 可以判断锁的状态，synchronized 不可以，无法知道是否成功获得锁。

1. **调度**：synchronized: 使用 Object 对象本身的 wait 、notify、notifyAll 调度机制；Lock：可以使用 Condition 进行线程之间的调度

1. synchronized **效率低**：锁的释放情况少，只有代码执行完毕或者异常结束才会释放锁；试图获取锁的时候不能设定超时，不能中断一个正在使用锁的线程，相对而言，Lock 可以中断和设置超时

1. synchronized **不够灵活**：加锁和释放的时机单一，每个锁仅有一个单一的条件(某个对象)，相对而言，读写锁更加灵活

### synchronized 在使用时有何注意事项？
1. 锁对象不能为空，因为锁的信息都保存在对象头里
1. 作用域不宜过大，影响程序执行的速度，控制范围过大，编写代码也容易出错
1. 避免死锁
1. 在能选择的情况下，既不要用 Lock 也不要用 synchronized 关键字，用 java.util.concurrent 包中的各种各样的类，如果不用该包下的类，在满足业务的情况下，可以使用 synchronized 关键，因为代码量少，避免出错

## 参考

* [深入掌握底层源码常见的 CAS 原子编程](https://juejin.cn/post/6900372748397182983)

* [学会了volatile，你变心了，我看到了](https://juejin.im/post/6892664695988649997)

* [从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)

* [分布式发号器架构设计](https://blog.csdn.net/egworkspace/article/details/80435081)

* [Java线程池实现原理及其在美团业务中的实践](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html)

* [多线程之死锁就是这么简单](https://juejin.cn/post/6844903602494898190#heading-8)

* [通俗易懂 悲观锁、乐观锁、可重入锁、自旋锁、偏向锁、轻量/重量级锁、读写锁、各种锁及其Java实现！](https://zhuanlan.zhihu.com/p/71156910)

* [深入掌握底层源码常见的 CAS 原子编程](https://juejin.cn/post/6900372748397182983)

* [学会了volatile，你变心了，我看到了](https://juejin.im/post/6892664695988649997)

* [从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)

* [【基本功】不可不说的Java“锁”事](https://mp.weixin.qq.com/s?__biz=MjM5NjQ5MTI5OA==&mid=2651749434&idx=3&sn=5ffa63ad47fe166f2f1a9f604ed10091&chksm=bd12a5778a652c61509d9e718ab086ff27ad8768586ea9b38c3dcf9e017a8e49bcae3df9bcc8&scene=38#wechat_redirect)

## Java 并发工具包

![](http://cdn.liufq.com/FpTiVPdig41lGesPbwMt_gMU0Kn-)

JUC 是 Java 并发工具包（java.util.concurrent）的简称。通过 CAS、AQS 等机制提供了安全的并发操作类。

## 什么是 AQS ？
AQS（AbstractQueuedSynchronizer）是一个用来构建锁和同步器的框架，使用 AQS 能简单且高效地构造出应用广泛的大量的同步器，JUC 很多都是基于 AQS 实现。

AQS **核心思想**是：如果被请求的共享资源空闲，则将当前请求资源的线程设置为有效的工作线程，并且将共享资源设置为锁定状态。如果被请求的共享资源被占用，那么就需要一套线程阻塞等待以及被唤醒时锁分配的机制，这个机制 AQS 是用 CLH 队列的变体 FIFO 队列实现的，即将暂时获取不到锁的线程加入到队列中。

AQS 使用一个 volatile 的 int 类型的成员变量 state 来表示同步状态，通过内置的 FIFO 队列来完成资源获取的排队工作，通过 CAS 完成对 state 值的修改。

AQS 定义了两种**资源获取方式**：
* 独占：只有一个线程能访问执行，又根据是否按队列的顺序分为公平锁和非公平锁，如 ReentrantLock。
* 共享：多个线程可同时访问执行，如 Semaphore、CountDownLatch、 CyclicBarrier，ReentrantReadWriteLock 可以看成是组合式，允许多个线程同时对某一资源进行读。

AQS 使用了模板设计模式。

## ReentrantLock 有哪些特点？
* Lock 实现类、可重入锁；提供了可响应中断锁、可轮询锁请求、定时锁等方法避免死锁；

* 主要功能通过抽象类 Sync 实现，根据是否是公平锁通过策略模式有 FairSync 和 NonfairSync 两种实现

* 基于状态变量 state 实现，通过 CAS 更新 state，state = 0 表示空闲，>0 表示被占用，数值等于当前线程占用锁的次数

* 公平锁通过队列实现，非公平锁直接竞争锁资源

## 什么是池化思想？

为了最大化收益最小化风险，而将资源统一在一起管理的思想，应用场景：

1. **内存池**：预先申请内存，提升申请内存速度，减少内存碎片。

2. **连接池**：预先申请数据库连接，提升申请连接的速度，降低系统的开销。
3. **实例池**：循环使用对象，减少资源在初始化和释放时的昂贵损耗。

## 为什么要用线程池？线程池的作用是什么？有哪些应用场景？
线程池能够对线程进行统一分配，调优和监控：

1. 降低资源消耗
1. 提高响应速度
1. 提高线程的可管理性
1. 提高可扩展性，提供更多更强大的功能

应用场景：
1. 快速响应用户请求
2. 快速处理批量任务

## ThreadPoolExecutor 实现原理？
通过线程集合 workerSet 和阻塞队列 workQueue 实现，当用户向线程池提交一个任务（也就是线程）时，线程池会先把任务放入 workQueue，workerSet 会不断的从 workQueue 里取任务去执行，当 workQueue 中没有任务的时候，workerSet 就会阻塞，直到队列中有任务了就取出来继续执行。

当 ThreadPoolExecutor 创建新线程时，通过 CAS 来更新线程池的状态 ctl。

1. 线程池首先当前运行的线程数量是否少于corePoolSize。如果是，则创建一个新的工作线程来执行任务。
1. 如果都在执行任务，则判断 BlockingQueue 是否已经满了，倘若还没有满，则将线程放入 BlockingQueue 。否则，
1. 如果创建一个新的工作线程将使当前运行的线程数量超过 maximumPoolSize，则交给 RejectedExecutionHandler 来处理任务。

## ThreadPoolExecutor 源码分析
![](https://p0.meituan.net/travelcube/77441586f6b312a54264e3fcf5eebe2663494.png)

1. 线程池的本质是对任务和线程的管理，通过**缓存队列**将任务和线程解耦，任务作为生产者加入队列，线程作为消费者从队列拿数据。

2. 通过工作线程 **Worker** 掌握线程的状态并维护线程的生命周期。

3. 使用 **ctl 变量**存储线程池状态：高 3 位表示运行状态 runState，低 29 位表示工作线程数量 workerCount

## ThreadPoolExecutor 有哪些核心的配置参数？

```
/**
 * @param corePoolSize 核心线程数：当任务数小于核心线程数时会创建新线程执行任务，否则加入阻塞队列；
 *        如果执行了线程池的 prestartAllCoreThreads() 方法，线程池会提前创建并启动所有核心线程。
 * @param maximumPoolSize 最大线程数：如果当前阻塞队列满了，且继续提交任务，如果当前线程数小于最大线程数，则创建新的线程执行任务；
 *        当阻塞队列是无界队列, 则 maximumPoolSize 不起作用, 因为无法提交至核心线程池的线程会一直持续地放入 workQueue。
 * @param keepAliveTime 线程空闲时的存活时间：即当线程没有任务执行时，该线程继续存活的时间；默认情况下，该参数只在线程数大于 corePoolSize 时才有用, 超过这个时间的空闲线程将被终止；
 * @param unit keepAliveTime 的单位
 * @param workQueue 用来保存等待被执行的任务的阻塞队列
 * @param threadFactory 创建线程的工厂，通过自定义的线程工厂可以给每个新建的线程设置一个具有识别度的线程名。默认为DefaultThreadFactory
 * @param handler 线程池的拒绝策略，当阻塞队列满了，且没有空闲的工作线程，如果继续提交任务，必须采取一种策略处理该任务
 */
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
}
```

## 线程池有哪些阻塞队列？

1. **ArrayBlockingQueue**: 基于数组结构的有界阻塞队列，按 FIFO 排序任务；

1. **LinkedBlockingQuene**: 基于链表结构的阻塞队列，按 FIFO 排序任务，吞吐量通常要高于 ArrayBlockingQuene；
1. **SynchronousQuene**: 一个不存储元素的阻塞队列，每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于 LinkedBlockingQuene；
1. **PriorityBlockingQuene**: 具有优先级的无界阻塞队列；

LinkedBlockingQueue 比 ArrayBlockingQueue 在插入删除节点性能方面更优，但是二者在 put()、take() 任务的时均需要加锁，SynchronousQueue 使用无锁算法，根据节点的状态判断执行，而不需要用到锁，其核心是 Transfer.transfer()。

### 线程池有哪些拒绝策略？
1. AbortPolicy: 直接抛出异常，默认策略；
1. CallerRunsPolicy: 用调用者所在的线程来执行任务；
1. DiscardOldestPolicy: 丢弃阻塞队列中靠最前的任务，并执行当前任务；
1. DiscardPolicy: 直接丢弃任务；

当然也可以根据应用场景实现 RejectedExecutionHandler 接口，自定义饱和策略，如记录日志或持久化存储不能处理的任务。

### 线程池的参数如何配置？
1. 根据**简单公式**：（N = cpu num）coreSize = 2 * N；maxSize = 25 * N；**问题**：没有区分 IO 密集型和 CPU 密集型，也没用考虑应用有多个线程池情况，实际效果可能很差；

2. **线程池参数动态化**：通过 ThreadPoolExecutor 提供的 pulice set 方法动态调整参数；

### FutureTask 作用是什么？

FutureTask 为 Future 提供了基础实现，如获取任务执行结果和取消任务等。常用来封装 Callable 和 Runnable，也可以作为一个任务提交到线程池中执行。FutureTask 的线程安全由 CAS 来保证。

### 如何使用 Executors 创建线程池？

阿里开发手册不推荐通过 Executors 工具类创建线程池，容易产生 OOM ，推荐手动创建 ThreadPoolExecutor 的方式创建。

1. **newCachedThreadPool**：创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。

2. **newFixedThreadPool**：创建一个可重用固定线程数的线程池，以共享的无界队列方式来运行这些线程；
3. **newScheduledThreadPool**：创建一个线程池，它可安排在给定延迟后运行命令或者定期地执行；
4. **newSingleThreadExecutor**：创建一个只有一个线程的线程池；

## CountDownLatch 源码分析

## 阻塞队列
### 什么是阻塞队列（BlockingQueue）？

阻塞队列（BlockingQueue）被广泛使用在“生产者-消费者”问题中，其原因是 BlockingQueue 提供了可阻塞的插入和移除的方法。当队列容器已满，生产者线程会被阻塞，直到队列未满；当队列容器为空时，消费者线程会被阻塞，直至队列非空时为止。

### 阻塞队列实现类
1. ArrayBlockingQueue：数组有界队列
1. LinkedBlockingDeque：链表无界队列
1. DelayQeque：基于时间的调度无界队列
1. PriorityBlockingQueue：优先级堆支持的无界队列

### 阻塞队列使用场景
1. 线程池中使用
1. Eureka 三级缓存
1. Netty
1. Nacos
1. RokcetMQ