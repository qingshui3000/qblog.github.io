---
title: Java多线程学习总结
permalink: Java多线程学习总结
date: 2020-01-05 16:45:27
tags: 多线程
---

# Java多线程学习笔记

## 为什么要使用多线程

<!--more-->

### 1.最大化的利用硬件性能

多物理核心和线程的CPU早已大量普及，显然并发的执行任务能更好的发挥硬件性能；

### 2.设计优势

**示例场景：**有两个任务需执行，任务一假定耗时10单位，任务二假定耗时1单位时间（实际中可能并不能在执行前确定任务耗时），如果不是多线程的环境下按顺序执行任务，则总耗时11，任务二虽然仅需要1单位时间执行，却要等待前面的任务一执行完后才被执行；而在多线程下执行时，因为cpu会在当前线程的时间片消耗完后切换至其他线程，所以虽然总耗时未变，但任务二无需等待任务一彻底执行完后再执行了。

**总结：多线程并不会提高程序的执行速度，因为上下文切换的原因，反而会降低速度。但如上述场景描述，可以减少用户的等待响应时间，提高了资源的利用效率和用户体验。**

### 3.使用多线程带来的问题

在并发编程时会出现数据的安全问题，线程与线程之间的竞争也会导致线程死锁和*锁死* 等**活性故障**，还有就是上文提到的上下文切换带来的额外开销。

[关于线程锁死]("https://blog.csdn.net/sinat_25991865/article/details/87949199")

## 如何创建线程

### 1. 继承Thread类 

 ```java
   public class MyThread1 extends Thread{
       public void run(){
           super.run();
           System.out.println("继承Thread类创建线程");
       }
       public static void main(String[] args) {
           MyThread1 thread1 = new MyThread1();
           thread1.start();
       }
   }
 ```

**运行结果**

```
继承Thread类创建线程
```



### 2.实现Runnable接口 

```JAVA
public class MyThread2 implements Runnable{
    public void run() {
        System.out.println("实现Runnable接口创建线程");
    }

    public static void main(String[] args) {
        Runnable myThread2 = new MyThread2();
        Thread t2 = new Thread(myThread2);
        t2.start();
    }
}
```

**运行结果**

```
实现Runnable接口创建线程
```

**两种方法有何区别？**

两者实现的功能是一致的，继承Thread类创建线程在设计上有局限性，因为Java不支持多继承，在一些业务情景下方法1会无法继承Thread类，而通过将一实现了Runnable接口的类作为Thread的构造方法的参数可以在保留原有继承关系的情况下创建线程。因此，总体来说，实现Runnable接口的方法创建线程更好一些，1是Java 不支持多重继承，因此继承了 Thread 类就无法继承其它类，但是可以实现多个接口；2是类可能只要求可执行就行，继承整个 Thread 类开销过大。

## 线程状态

**jdk1.8源码中对于Thread状态的枚举定义:**

#### 1.新建(NEW)


```JAVA
public enum State {
        /**
         * Thread state for a thread which has not yet started.
         */
        NEW,
```

线程还未调用start()方法启动



#### 2.可运行(RUNNABLE)

```JAVA
        /**
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         */
        RUNNABLE,
```

正在Java虚拟机中运行，但是在操作系统的层面有两种可能，一是处于运行状态，二是等待系统资源，资源调度完成后就开始运行，所以该状态指的是**可被运行**，具体有没有运行要结合操作系统资源调度的情况。



#### 3.阻塞(BLOCK)

```JAVA
        /**
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or
         * reenter a synchronized block/method after calling
         * {@link Object#wait() Object.wait}.
         */
        BLOCKED,
```

在线程准备进入synchronized同步块或者方法时需要请求获取一个监听器锁(monitor lock)，若此时其他线程已经占用了该监听器锁，则当前线程进入阻塞状态；在其他线程释放了该监听器锁后当前线程结束阻塞态。

#### 4.无限期等待(WAITING)

```JAVA
        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait()</tt>
         * on an object is waiting for another thread to call
         * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
         * that object. A thread that has called <tt>Thread.join()</tt>
         * is waiting for a specified thread to terminate.
         */
        WAITING,
```

无期限的等待其他线程显式的唤醒。与阻塞态不同的是，阻塞是被动的，它在等待获取监听器锁，而等待是主动的，通过调用Object.wait()进入；



| 进入方法                                   | 退出方法                             |
| ------------------------------------------ | ------------------------------------ |
| 没有设置 Timeout 参数的 Object.wait() 方法 | Object.notify() / Object.notifyAll() |
| 没有设置 Timeout 参数的 Thread.join() 方法 | 被调用的线程执行完毕                 |
| LockSupport.park() 方法                    | LockSupport.unpark(Thread)           |

#### 5.限期等待(TIMED_WAITING)

```JAVA
        /**
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
        TIMED_WAITING,
```

与上一状态类似，但在一定时间后会被系统唤醒。



| 进入方法                                 | 退出方法                                        |
| ---------------------------------------- | ----------------------------------------------- |
| Thread.sleep() 方法                      | 时间结束                                        |
| 设置了 Timeout 参数的 Object.wait() 方法 | 时间结束 / Object.notify() / Object.notifyAll() |
| 设置了 Timeout 参数的 Thread.join() 方法 | 时间结束 / 被调用的线程执行完毕                 |
| LockSupport.parkNanos() 方法             | LockSupport.unpark(Thread)                      |
| LockSupport.parkUntil() 方法             | LockSupport.unpark(Thread)                      |

#### 6.死亡(TERMINATED)

```JAVA
        /**
         * Thread state for a terminated thread.
         * The thread has completed execution.
         */
        TERMINATED;
    }
```

死亡态就是线程执行结束(run方法执行结束)或抛出异常而结束。



## 线程通信

待更。。。

## 线程相关方法

### 1.中断

**interrupt()**

**interrupted()**

**isInterrupted()**



### 2.sleep() 和wait()

* **sleep方法**是Thread类的静态方法，作用是将当前线程顺便n毫秒，线程挂起进入阻塞态，时间到了后会解除阻塞进入可运行态，**sleep方法不释放锁。**
* **wait()方法**是Object类的静态方法，只能在同步方法或同步块中使用，否则运行时会抛出IllegalMonitorStateException异常，调用后线程进入阻塞态，当调用notify或notifyAll方法后解除阻塞，等待**重新获得互斥锁**后进入可运行态，**因为wait方法释放锁**

### 3.如何避免死锁

我们只要破坏产生死锁的四个条件中的其中一个就可以了。

**1.破坏互斥条件**

这个条件我们没有办法破坏，因为我们使用锁本来就是想让他们互斥的（临界资源需要互斥访问）。

**2.破坏请求与保持条件**

一次性申请所有的资源。

**3.破坏不剥夺条件**

占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源。

**4.破坏循环等待条件**

靠按序申请资源来预防。按某一顺序申请资源，释放资源则反序释放。破坏循环等待条件。



### 4.多线程编程中的一些线程安全问题

#### 原子性、有序性和可见性(内存模型三大特性)

多线程环境下的线程安全主要体现在**原子性**、**可见性**和**有序性**方面。

**1.原子性**

Java内存模型为read、load、use、assign、store、write、lock和unlock操作保证了**原子性**；**特殊的**，对于long和double类型的数据，Java内存模型允许虚拟机将没有被volatile关键字修饰的64位变量的读、写操划分为两次对32位数据的操作进行，即在这种情况下，read、load、store、write操作可以不具有原子性。

**定义**

对于涉及到访问共享变量的操作，若当前操作是不可拆分的，即中间操作对于线程外部来说不可见，那么该操作就是原子操作1，该操作具有原子性。

**举例**

银行业务中的转账操作，A给B转账100元，基本步骤就是A账户减少100元，B账户就会多100元，虽然表面上可以拆分为这两步，但是我们不可能看到A账户少了100元、但B账户余额并未增加的情况。（现实中可能有延迟时间，这里忽略）

**如何保证原子性**

* 利用**互斥锁**的排他性，保证同一时刻只有一个线程在操作共享变量。
* 利用CAS保证。

**2.可见性**

**定义**

可见性是指一个线程对于共享变量的更新，对于后续访问该变量的线程是否可见的问题；Java内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值来实现可见性的。

```
处理器缓存的概念

现代处理器处理速度远大于主内存的处理速度，所以在主内存和处理器之间加入了寄存器，高速缓存，写缓冲器以及无效化队列等部件来加速内存的读写操作。也就是说，我们的处理器可以和这些部件进行读写操作的交互，这些部件可以称为处理器缓存。

处理器对内存的读写操作，其实仅仅是与处理器缓存进行了交互。一个处理器的缓存上的内容无法被另外一个处理器读取，所以另外一个处理器必须通过缓存一致性协议来读取的其他处理器缓存中的数据，并且同步到自己的处理器缓存中，这样保证了其余处理器对该变量的更新对于另外处理器是可见的。


```

**如何实现可见性**

* volatile关键字
* synchronized，对一个变量执行unlock操作前，必须把变量值同步回主内存。
* final关键字，被final关键字修饰的字段在构造器中一旦初始化完成，并且没有发生this逃逸（其他线程通过this引用到初始化一半的变量），那么其他线程就能看到该final字段的值。

**3.有序性**

**定义：**有序性是指一个处理器上运行的线程所执行的内存访问操作在另外一个处理器上运行的线程来看是否有序的问题。



**重排序：**
为了提高程序执行的性能，Java编译器在其认为不影响程序正确性的前提下，可能会对源代码顺序进行一定的调整，导致程序运行顺序与源代码顺序不一致。

重排序是对内存读写操作的一种优化，在单线程环境下不会导致程序的正确性问题，但是多线程环境下可能会影响程序的正确性。



**重排序举例：**
**Instance instance = new Instance()都发生了啥？**
**具体步骤如下所示三步：**

- 在堆内存上分配对象的内存空间
- 在堆内存上初始化对象
- 设置instance指向刚分配的内存地址



第二步和第三步可能会发生重排序，导致引用型变量指向了一个不为null但是也不完整的对象。（**在多线程下的单例模式中，我们必须通过volatile来禁止指令重排序**）



**总结：**

- **原子性**是一组操作要么完全发生，要么没有发生，其余线程不会看到中间过程的存在。注意，**原子操作+原子操作不一定还是原子操作。**
- **可见性**是指一个线程对共享变量的更新**对于另外一个线程是否可见**的问题。
- **有序性**是指一个线程对共享变量的更新在其余线程看起来是**按照什么顺序执行**的问题。
- 可以这么认为，**原子性 + 可见性 -> 有序性**

### 5.互斥同步

Java提供了两种锁机制来控制多个线程对于共享资源的互斥访问，一个是JVM实现的synchronized关键字，另一个是JDK实现的ReentrantLock

#### synchronized

* **同步代码块**

```JAVA
public void func(){
    synchronized (this){
        //...
    }
}
```

**只作用于同一个对象，如果分别调用两个对象上的同步代码块，就不会进行同步**

**示例：**

```JAVA
public class SynchronizedExample {

    public void func1() {
        synchronized (this) {
            for (int i = 0; i < 10; i++) {
                System.out.print(i + " ");
            }
        }
    }
}
```

示例1（同一个对象）：


```JAVA
public static void main(String[] args) {
    SynchronizedExample e1 = new SynchronizedExample();
    ExecutorService executorService = Executors.newCachedThreadPool();
    executorService.execute(() -> e1.func1());
    executorService.execute(() -> e1.func1());
}
```



运行结果：

```JAVA
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9
```



示例2（两个对象）：

```JAVA
public static void main(String[] args) {
    SynchronizedExample e1 = new SynchronizedExample();
    SynchronizedExample e2 = new SynchronizedExample();
    ExecutorService executorService = Executors.newCachedThreadPool();
    executorService.execute(() -> e1.func1());
    executorService.execute(() -> e2.func1());
}
```

运行结果：

```JAVA
0 0 1 1 2 2 3 3 4 4 5 5 6 6 7 7 8 8 9 9
```



* **同步方法**

```JAVA
public synchronized void func(){
    //...
}
```

与同步块一样，作用于同一个对象

* **同步一个类**

作用于整个类，也就是两个线程调用同一个类的不同对象的同步语句仍会进行同步

```java
public class SynchronizedExample {

    public void func2() {
        synchronized (SynchronizedExample.class) {
            for (int i = 0; i < 10; i++) {
                System.out.print(i + " ");
            }
        }
    }
}
public static void main(String[] args) {
    SynchronizedExample e1 = new SynchronizedExample();
    SynchronizedExample e2 = new SynchronizedExample();
    ExecutorService executorService = Executors.newCachedThreadPool();
    executorService.execute(() -> e1.func2());
    executorService.execute(() -> e2.func2());
}
```

```
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9
```

* **同步静态方法**

```JAVA
public synchronized static void fun(){
    // ...
}
```

作用于整个类

#### ReentrantLock和synchronized的区别

ReentrantLock**是显示锁**，其提供了一些内部锁不具备的特性，但并不是内部锁的替代品。**显式锁支持公平和非公平的调度方式**，默认采用非公平调度。

synchronized 内部锁简单，但是不灵活。显示锁支持在一个方法内申请锁，并且在另一个方法里释放锁。**显示锁定义了一个tryLock（）方法，尝试去获取锁**，成功返回true，失败并不会导致其执行的线程被暂停而是直接返回false，即可以**避免死锁**。

#### 比较

**1. 锁的实现**

synchronized 是 JVM 实现的，而 ReentrantLock 是 JDK 实现的。

**2. 性能**

新版本 Java 对 synchronized 进行了很多优化，例如自旋锁等，synchronized 与 ReentrantLock 大致相同。

**3. 等待可中断**

当持有锁的线程长期不释放锁的时候，正在等待的线程可以选择放弃等待，改为处理其他事情。

ReentrantLock 可中断，而 synchronized 不行。

**4. 公平锁**

公平锁是指多个线程在等待同一个锁时，必须按照申请锁的时间顺序来依次获得锁。

synchronized 中的锁是非公平的，ReentrantLock 默认情况下也是非公平的，但是也可以是公平的。

**5. 锁绑定多个条件**

一个 ReentrantLock 可以同时绑定多个 Condition 对象。

#### 使用选择

除非需要使用 ReentrantLock 的高级功能，否则优先使用 synchronized。这是因为 synchronized 是 JVM 实现的一种锁机制，JVM 原生地支持它，而 ReentrantLock 不是所有的 JDK 版本都支持。并且使用 synchronized 不用担心没有释放锁而导致死锁问题，因为 JVM 会确保锁的释放。

相比synchronized，ReentrantLock增加了一些高级功能。主要来说主要有三点：**①等待可中断；②可实现公平锁；③可实现选择性通知（锁可以绑定多个条件）**

- **ReentrantLock提供了一种能够中断等待锁的线程的机制**，通过lock.lockInterruptibly()来实现这个机制。也就是说正在等待的线程可以选择放弃等待，改为处理其他事情。
- **ReentrantLock可以指定是公平锁还是非公平锁。而synchronized只能是非公平锁。所谓的公平锁就是先等待的线程先获得锁。** ReentrantLock默认情况是非公平的，可以通过 ReentrantLock类的`ReentrantLock(boolean fair)`构造方法来制定是否是公平的。
- synchronized关键字与wait()和notify()/notifyAll()方法相结合可以实现等待/通知机制，ReentrantLock类当然也可以实现，但是需要借助于Condition接口与newCondition() 方法。Condition是JDK1.5之后才有的，它具有很好的灵活性，比如可以实现多路通知功能也就是在一个Lock对象中可以创建多个Condition实例（即对象监视器），**线程对象可以注册在指定的Condition中，从而可以有选择性的进行线程通知，在调度线程上更加灵活。 在使用notify()/notifyAll()方法进行通知时，被通知的线程是由 JVM 选择的，用ReentrantLock类结合Condition实例可以实现“选择性通知”** ，这个功能非常重要，而且是Condition接口默认提供的。而synchronized关键字就相当于整个Lock对象中只有一个Condition实例，所有的线程都注册在它一个身上。如果执行notifyAll()方法的话就会通知所有处于等待状态的线程这样会造成很大的效率问题，而Condition实例的signalAll()方法 只会唤醒注册在该Condition实例中的所有等待线程。

## 线程池

### 1.使用线程池的好处

线程池和Http连接池、数据库连接池等类似，都是为了降低资源消耗、提高访问效率。

**使用线程池的好处：**

* **降低资源消耗。**通过重复利用已创建的线程降低线程创建和销毁造成的消耗。
* **提高响应速度。**当任务到达时，任务可以**不需要等待线程创建**就能立即执行。
* **提高线程的可管理性。**线程是一种稀缺资源，如果无限制的创建，不仅会消耗系统资源，还会降低系统的稳定性，使用线程池可以进行统一的分配、调优和监控。

### 2.ThreadPoolExecutor

#### 基本参数

线程池实现类是Executor框架最核心的类

`ThreadPoolExecutor`的3个重要参数：

* `corePoolSize`：核心线程数，定义了最小可以同时运行的线程数量。
* `maximumPoolSize`：最大线程数。
* `workQueue`：存储线程的队列

`ThreadPoolExecutor`其他常见参数:

1. **`keepAliveTime`**:当线程池中的线程数量大于 `corePoolSize` 的时候，如果这时没有新的任务提交，核心线程外的线程不会立即销毁，而是会等待，直到等待的时间超过了 `keepAliveTime`才会被回收销毁；
2. **`unit`** : `keepAliveTime` 参数的时间单位。
3. **`threadFactory`** :executor 创建新线程的时候会用到。
4. **`handler`** :饱和策略。关于饱和策略下面单独介绍一下。

#### 线程池的排队策略：

* 如果正在运行的线程数小于核心线程数，则Executor始终首选添加新的线程进入线程池，而不排队。
* 如果正在运行的线程等于或多余核心线程数，则Executor始终首选将新请求加入队列进行排队。
* 如果无法将请求加入队列，即队列已满，则创建新的线程，若创建此线程后超出`maximumPoolSize`，则任务将会拒绝。



#### 常见的线程池类型

**newCachedThreadPool()**

* 核心线程池大小为0，最大线程数不受限，即每次请求都会创建一个线程。
* 适合用于执行大量耗时短且提交频率高的任务场景。

**newFixedThreadPool()**

* 固定大小的线程池
* 当线程池大小达到核心线程数时，会将新任务加入`LinkedBlockingQueue`。
* 线程池中的线程执行完手头的任务后，会在循环中反复从`LinkedBlockingQueue`中获取任务执行。

`LinkedBlockingQueue`是一个无界队列（容量为Integer.MAX_VALUE）。

为什么不推荐使用`FixedThreadPool`：

1.线程池达到核心线程数后不会继续增加。

2.由于无界队列的存在，`maximumPoolSize`将会是一个无效参数，即不可能存在任务队列满的情况（`FixedThreadPool`中的`maximumPoolSize`和`corePoolSize`被设置为同一个值）。

3.同理，由于1,2，`keepAliveTime`也将会是一个无效参数。

4.由于无界队列的存在，运行中的`FixedThreadPoll`不会拒绝任务，在任务比较多的时候会导致OOM（内存溢出）。

**newSingleThreadExecutor()**

便于实现生产者-消费者模式

#### 常见的阻塞队列

**ArrayBlockingQueue:**

- 内部使用一个**数组**作为其存储空间，数组的存储空间是**预先分配**的
- **优点是** put 和 take操作不会增加GC的负担（因为空间是预先分配的）
- **缺点是** put 和 take操作使用同一个锁，可能导致锁争用，导致较多的上下文切换。
- ArrayBlockingQueue适合在生产者线程和消费者线程之间的**并发程序较低**的情况下使用。



**LinkedBlockingQueue：**

- 是一个无界队列（其实队列长度是Integer.MAX_VALUE）
- 内部存储空间是一个**链表**，并且链表节点所需的**存储空间是动态分配**的
- **优点是** put 和 take 操作使用两个显式锁（putLock和takeLock）
- **缺点是**增加了GC的负担，因为空间是动态分配的。
- LinkedBlockingQueue适合在生产者线程和消费者线程之间的并发程序较高的情况下使用。



**SynchronousQueue：**
SynchronousQueue可以被看做一种特殊的有界队列。生产者线程生产一个产品之后，会等待消费者线程来取走这个产品，才会接着生产下一个产品，适合在生产者线程和消费者线程之间的处理能力相差不大的情况下使用。



## ThreadLocal

### 1.定义

ThreadLocal为每个使用该变量的线程提供独立的变量副本，所以每个线程都可以拥有独立的副本，不会影响到其他线程的变量副本。

ThreadLocal类用于实现**线程的本地存储**

### 2.ThreadLocal内部实现机制

* 每个线程内部都会维护一个类似HashMap的对象，称为**ThreadLocalMap**，里面会包含若干**Entry(KEY-VALUE键值对)**，相应的线程被称为这些Entry的属主线程。
* **Entry的KEY是一个ThreadLocal实例，VALUE是一个线程持有对象。**Entry的作用即为其属主线程建立起一个ThreadLocal实例和线程持有对象的对应关系。
* Entry对KEY是弱引用，对VALUE是强引用（GC时的区别）。

## 原子类

java.util.concurrent.atomic包下的AtomicInteger等原子类用于保证复合操作的原子性。

* **AtomicInteger类提供了getAndIncrement和incrementAndGet等原子性的自增自减等操作**。
* **Atomic等原子类内部使用了CAS来保证原子性。**

## 锁相关

代更。。