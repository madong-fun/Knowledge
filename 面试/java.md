# Java

## 多线程

### **多线程常见问题**

**1.上下文切换**

由于在线程切换的时候需要保存本次执行的信息，在该线程被 CPU 剥夺时间片后又再次运行恢复上次所保存的信息的过程就称为上下文切换。

解决方案：

- 采用无锁编程，比如将数据按照 `Hash(id)` 进行取模分段，每个线程处理各自分段的数据，从而避免使用锁。
- 采用 CAS(compare and swap) 算法，如 `Atomic` 包就是采用 CAS 算法。
- 合理的创建线程，避免创建了一些线程但其中大部分都是处于 `waiting` 状态，因为每当从 `waiting` 状态切换到 `running` 状态都是一次上下文切换。

**2.死锁**

死锁的场景一般是：线程 A 和线程 B 都在互相等待对方释放锁，或者是其中某个线程在释放锁的时候出现异常如死循环之类的。这时就会导致系统不可用。

解决方案：

- 尽量一个线程只获取一个锁
- 一个线程只占用一个资源
- 尝试使用定时锁，至少能保证锁最终会被释放

**3.资源限制**

当在带宽有限的情况下一个线程下载某个资源需要 `1M/S`,当开 10 个线程时速度并不会乘 10 倍，反而还会增加时间，毕竟上下文切换比较耗时。

如果是受限于资源的话可以采用集群来处理任务，不同的机器来处理不同的数据，就类似于开始提到的无锁编程。

### 多线程三大核心

#### 原子性

`Java` 的原子性就和数据库事务的原子性差不多，一个操作中要么全部执行成功或者失败。

`JMM` 只是保证了基本的原子性，但类似于 `i++` 之类的操作，看似是原子操作，其实里面涉及到:

- 获取`i`的值
- 自增
- 再赋值给`i`



这三步操作，所以想要实现 `i++` 这样的原子操作就需要用到 `synchronized` 或者是 `lock` 进行加锁处理。

如果是基础类的自增操作可以使用 `AtomicInteger` 这样的原子类来实现(其本质是利用了 `CPU` 级别的 的 `CAS` 指令来完成的)。

其中用的最多的方法就是: `incrementAndGet()` 以原子的方式自增。



#### 可见性

现代计算机中，由于 `CPU` 直接从主内存中读取数据的效率不高，所以都会对应的 `CPU` 高速缓存，先将主内存中的数据读取到缓存中，线程修改数据之后首先更新到缓存，之后才会更新到主内存。如果此时还没有将数据更新到主内存其他的线程此时来读取就是修改之前的数据。

`volatile` 关键字就是用于保证内存可见性，当线程A更新了 volatile 修饰的变量时，它会立即刷新到主线程，并且将其余缓存中该变量的值清空，导致其余线程只能去主内存读取最新值。

使用 `volatile` 关键词修饰的变量每次读取都会得到最新的数据，不管哪个线程对这个变量的修改都会立即刷新到主内存。

`synchronized`和加锁也能能保证可见性，实现原理就是在释放锁之前其余线程是访问不到这个共享变量的。但是和 `volatile` 相比开销较大。



#### 顺序性

在保证最终结果和代码顺序执行结果一致的情况下，`JVM` 为了提高整体的效率会进行指令进行重排。

重排在单线程中不会出现问题，但在多线程中会出现数据不一致的问题。

Java 中可以使用 `volatile` 来保证顺序性，`synchronized 和 lock` 也可以来保证有序性，和保证原子性的方式一样，通过同一段时间只能一个线程访问来实现的。

除了通过 `volatile` 关键字显式的保证顺序之外， `JVM` 还通过 `happen-before` 原则来隐式的保证顺序性。

其中有一条就是适用于 `volatile` 关键字的，针对于 `volatile` 关键字的写操作肯定是在读操作之前，也就是说读取的值肯定是最新的。

PS：`volatile` 关键字只能保证可见性，顺序性，**不能保证原子性**。



### 锁

#### synchronized 关键字原理

众所周知 `synchronized` 关键字是解决并发问题常用解决方案，有以下三种使用方式:

- 同步普通方法，锁的是当前对象
- 同步静态方法，锁的是当前class对象
- 同步代码块，锁的是（）中的对象

其实现原理：

`JVM` 是通过进入、退出对象监视器( `Monitor` )来实现对方法、同步块的同步的。

具体实现是在编译之后在同步方法调用前加入一个 `monitor.enter` 指令，在退出方法和异常处插入 `monitor.exit` 的指令。

其本质就是对一个对象监视器( `Monitor` )进行获取，而这个获取过程具有排他性从而达到了同一时刻只能一个线程访问的目的。

而对于没有获取到锁的线程将会阻塞到方法入口处，直到获取锁的线程 `monitor.exit` 之后才能尝试继续获取锁。



#### synchronized优化

`synchronized` 很多都称之为重量锁，`JDK1.6` 中对 `synchronized` 进行了各种优化，为了能减少获取和释放锁带来的消耗引入了`偏向锁`和`轻量锁`。

#### 轻量锁

当代码进入同步块时，如果同步对象为无锁状态时，当前线程会在栈帧中创建一个锁记录(`Lock Record`)区域，同时将锁对象的对象头中 `Mark Word` 拷贝到锁记录中，再尝试使用 `CAS` 将 `Mark Word` 更新为指向锁记录的指针。

如果更新**成功**，当前线程就获得了锁。

如果更新**失败** `JVM` 会先检查锁对象的 `Mark Word` 是否指向当前线程的锁记录。

如果是则说明当前线程拥有锁对象的锁，可以直接进入同步块。

不是则说明有其他线程抢占了锁，如果存在多个线程同时竞争一把锁，**轻量锁就会膨胀为重量锁**。

#### 解锁

轻量锁的解锁过程也是利用 `CAS` 来实现的，会尝试锁记录替换回锁对象的 `Mark Word` 。如果替换成功则说明整个同步操作完成，失败则说明有其他线程尝试获取锁，这时就会唤醒被挂起的线程(此时已经膨胀为`重量锁`)

轻量锁能提升性能的原因是：

认为大多数锁在整个同步周期都不存在竞争，所以使用 `CAS` 比使用互斥开销更少。但如果锁竞争激烈，轻量锁就不但有互斥的开销，还有 `CAS` 的开销，甚至比重量锁更慢。

#### 偏向锁

为了进一步的降低获取锁的代价，`JDK1.6` 之后还引入了偏向锁。

偏向锁的特征是:锁不存在多线程竞争，并且应由一个线程多次获得锁。

当线程访问同步块时，会使用 `CAS` 将线程 ID 更新到锁对象的 `Mark Word` 中，如果更新成功则获得偏向锁，并且之后每次进入这个对象锁相关的同步块时都不需要再次获取锁了。

#### 释放锁

当有另外一个线程获取这个锁时，持有偏向锁的线程就会释放锁，释放时会等待全局安全点(这一时刻没有字节码运行)，接着会暂停拥有偏向锁的线程，根据锁对象目前是否被锁来判定将对象头中的 `Mark Word` 设置为无锁或者是轻量锁状态。

偏向锁可以提高带有同步却没有竞争的程序性能，但如果程序中大多数锁都存在竞争时，那偏向锁就起不到太大作用。可以使用 `-XX:-UseBiasedLocking` 来关闭偏向锁，并默认进入轻量锁。



#### ReentrantLock

使用 `synchronized` 来做同步处理时，锁的获取和释放都是隐式的，实现的原理是通过编译后加上不同的机器指令来实现。而 `ReentrantLock` 就是一个普通的类，它是基于 `AQS(AbstractQueuedSynchronizer)`来实现的。

是一个**重入锁**：一个线程获得了锁之后仍然可以**反复**的加锁，不会出现自己阻塞自己的情况。

ps：`AQS` 是 `Java` 并发包里实现锁、同步的一个重要的基础框架.



ReentrantLock 分为**公平锁**和**非公平锁**，可以通过构造方法来指定具体类型,默认一般使用**非公平锁**，它的效率和吞吐量都比公平锁高的多(后面会分析具体原因)。

**锁的获取过程：**

```java
    public void lock() {
        sync.lock();
    }
```

可以看到是使用 `sync`的方法，而这个方法是一个抽象方法，具体是由其子类(`FairSync`)来实现的，以下是公平锁的实现:

```java
        final void lock() {
            acquire(1);
        }

        //AbstractQueuedSynchronizer 中的 acquire()
        public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
        }
```

第一步是尝试获取锁(`tryAcquire(arg)`),这个也是由其子类实现：

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
    }
```



首先会判断 `AQS` 中的 `state` 是否等于 0，0 表示目前没有其他线程获得锁，当前线程就可以尝试获取锁。

**注意**:尝试之前会利用 `hasQueuedPredecessors()` 方法来判断 AQS 的队列中中是否有其他线程，如果有则不会尝试获取锁(**这是公平锁特有的情况**)。

如果队列中没有线程就利用 CAS 来将 AQS 中的 state 修改为1，也就是获取锁，获取成功则将当前线程置为获得锁的独占线程(`setExclusiveOwnerThread(current)`)。

如果 `state` 大于 0 时，说明锁已经被获取了，则需要判断获取锁的线程是否为当前线程(`ReentrantLock` 支持重入)，是则需要将 `state + 1`，并将值更新



如果 `tryAcquire(arg)` 获取锁失败，则需要用 `addWaiter(Node.EXCLUSIVE)` 将当前线程写入队列中。

写入之前需要将当前线程包装为一个 `Node` 对象(`addWaiter(Node.EXCLUSIVE)`)。

PS: `AQS 中的队列是由 Node 节点组成的双向链表实现的。`

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```



首先判断队列是否为空，不为空时则将封装好的 `Node` 利用 `CAS` 写入队尾，如果出现并发写入失败就需要调用 `enq(node);` 来写入了.

```
rivate Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

这个处理逻辑就相当于`自旋`加上 `CAS` 保证一定能写入队列。



写入队列之后需要将当前线程挂起(利用`acquireQueued(addWaiter(Node.EXCLUSIVE), arg)`)：

```
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

首先会根据 `node.predecessor()` 获取到上一个节点是否为头节点，如果是则尝试获取一次锁，获取成功就万事大吉了。

如果不是头节点，或者获取锁失败，则会根据上一个节点的 `waitStatus` 状态来处理(`shouldParkAfterFailedAcquire(p, node)`)。

`waitStatus` 用于记录当前节点的状态，如节点取消、节点等待等。

`shouldParkAfterFailedAcquire(p, node)` 返回当前线程是否需要挂起，如果需要则调用 `parkAndCheckInterrupt()`：

```
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```



他是利用 `LockSupport` 的 `part` 方法来挂起当前线程的，直到被唤醒。



公平锁与非公平锁的差异主要在获取锁：

公平锁就相当于买票，后来的人需要排到队尾依次买票，**不能插队**。

而非公平锁则没有这些规则，是**抢占模式**，每来一个人不会去管队列如何，直接尝试获取锁。



非公平锁：

```
      final void lock() {
            //直接尝试获取锁
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
```

公平锁：

```
    final void lock() {
            acquire(1);
        }
```

还要一个重要的区别是在尝试获取锁时`tryAcquire(arg)`，非公平锁是不需要判断队列中是否还有其他线程，也是直接尝试获取锁：

```
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //没有 !hasQueuedPredecessors() 判断
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```



释放锁：

公平锁和非公平锁的释放流程都是一样的：

```
    public void unlock() {
        sync.release(1);
    }

    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                   //唤醒被挂起的线程
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

    //尝试释放锁
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

首先会判断当前线程是否为获得锁的线程，由于是重入锁所以需要将 `state` 减到 0 才认为完全释放锁。

释放之后需要调用 `unparkSuccessor(h)` 来唤醒被挂起的线程



**总结**

由于公平锁需要关心队列的情况，得按照队列里的先后顺序来获取锁(会造成大量的线程上下文切换)，而非公平锁则没有这个限制。

所以也就能解释非公平锁的效率会被公平锁更高。

### 线程池

创建线程池：

```
ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue, RejectedExecutionHandler handler) 
```



线程池核心参数：

- `corePoolSize` 为线程池的基本大小。
- `maximumPoolSize` 为线程池最大线程大小。
- `keepAliveTime` 和 `unit` 则是线程空闲后的存活时间。
- `workQueue` 用于存放任务的阻塞队列。
- `handler` 当队列和最大线程池都满了之后的饱和策略。

线程状态：

- `RUNNING` 自然是运行状态，指可以接受任务执行队列里的任务
- `SHUTDOWN` 指调用了 `shutdown()` 方法，不再接受新任务了，但是队列里的任务得执行完毕。
- `STOP` 指调用了 `shutdownNow()` 方法，不再接受新任务，同时抛弃阻塞队列里的所有任务并中断所有正在执行任务。
- `TIDYING` 所有任务都执行完毕，在调用 `shutdown()/shutdownNow()` 中都会尝试更新为这个状态。
- `TERMINATED` 终止状态，当执行 `terminated()` 后会更新为这个状态。

`ThreadPool.execute()` 方法是如何处理的：

1. 获取当前线程池的状态。
2. 当前线程数量小于 coreSize 时创建一个新的线程运行。
3. 如果当前线程处于运行状态，并且写入阻塞队列成功。
4. 双重检查，再次获取线程状态；如果线程状态变了（非运行状态）就需要从阻塞队列移除任务，并尝试判断线程是否全部执行完毕。同时执行拒绝策略。
5. 如果当前线程池为空就新创建一个线程并执行。
6. 如果在第三步的判断为非运行状态，尝试新建线程，如果失败则执行拒绝策略。

**如何配置线程池：**

- IO 密集型任务：由于线程并不是一直在运行，所以可以尽可能的多配置线程，比如 CPU 个数 * 2
- CPU 密集型任务（大量复杂的运算）应当分配较少的线程，比如 CPU 个数相当的大小。

#### **关闭线程池**

- `shutdown()` 执行后停止接受新任务，会把队列的任务执行完毕。
- `shutdownNow()` 也是停止接受新任务，但会中断所有的任务，将线程池状态变为 stop。

### 线程间通信

#### 等待与通知

两个线程通过对同一对象调用等待 wait() 和通知 notify() 方法来进行通讯。

```
public class TwoThreadWaitNotify {

    private int start = 1;

    private boolean flag = false;

    public static void main(String[] args) {
        TwoThreadWaitNotify twoThread = new TwoThreadWaitNotify();

        Thread t1 = new Thread(new OuNum(twoThread));
        t1.setName("A");


        Thread t2 = new Thread(new JiNum(twoThread));
        t2.setName("B");

        t1.start();
        t2.start();
    }

    /**
     * 偶数线程
     */
    public static class OuNum implements Runnable {
        private TwoThreadWaitNotify number;

        public OuNum(TwoThreadWaitNotify number) {
            this.number = number;
        }

        @Override
        public void run() {

            while (number.start <= 100) {
                synchronized (TwoThreadWaitNotify.class) {
                    System.out.println("偶数线程抢到锁了");
                    if (number.flag) {
                        System.out.println(Thread.currentThread().getName() + "+-+偶数" + number.start);
                        number.start++;

                        number.flag = false;
                        TwoThreadWaitNotify.class.notify();

                    }else {
                        try {
                            TwoThreadWaitNotify.class.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }

            }
        }
    }


    /**
     * 奇数线程
     */
    public static class JiNum implements Runnable {
        private TwoThreadWaitNotify number;

        public JiNum(TwoThreadWaitNotify number) {
            this.number = number;
        }

        @Override
        public void run() {
            while (number.start <= 100) {
                synchronized (TwoThreadWaitNotify.class) {
                    System.out.println("奇数线程抢到锁了");
                    if (!number.flag) {
                        System.out.println(Thread.currentThread().getName() + "+-+奇数" + number.start);
                        number.start++;

                        number.flag = true;

                        TwoThreadWaitNotify.class.notify();
                    }else {
                        try {
                            TwoThreadWaitNotify.class.wait();
                        } catch (InterruptedException e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }
    }
}
```

这里的线程 A 和线程 B 都对同一个对象 `TwoThreadWaitNotify.class` 获取锁，A 线程调用了同步对象的 wait() 方法释放了锁并进入 `WAITING` 状态。

B 线程调用了 notify() 方法，这样 A 线程收到通知之后就可以从 wait() 方法中返回。

这里利用了 `TwoThreadWaitNotify.class` 对象完成了通信。

有一些需要注意:

- wait() 、notify()、notifyAll() 调用的前提都是获得了对象的锁(也可称为对象监视器)。
- 调用 wait() 方法后线程会释放锁，进入 `WAITING` 状态，该线程也会被移动到**等待队列**中。
- 调用 notify() 方法会将**等待队列**中的线程移动到**同步队列**中，线程状态也会更新为 `BLOCKED`
- 从 wait() 方法返回的前提是调用 notify() 方法的线程释放锁，wait() 方法的线程获得锁。



#### Join

```
    private static void join() throws InterruptedException {
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                LOGGER.info("running");
                try {
                    Thread.sleep(3000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }) ;
        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                LOGGER.info("running2");
                try {
                    Thread.sleep(4000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }) ;

        t1.start();
        t2.start();

        //等待线程1终止
        t1.join();

        //等待线程2终止
        t2.join();

        LOGGER.info("main over");
    }
```



在 `t1.join()` 时会一直阻塞到 t1 执行完毕，所以最终主线程会等待 t1 和 t2 线程执行完毕。

其实从源码可以看出，join() 也是利用的等待通知机制：

在 join 线程完成后会调用 notifyAll() 方法，是在 JVM 实现中调用，所以这里看不出来。



#### Volatile 共享内存

因为 Java 是采用共享内存的方式进行线程通信的，所以可以采用以下方式用主线程关闭 A 线程:

#### CountDownLatch 

CountDownLatch 可以实现 join 相同的功能，但是更加的灵活。

CountDownLatch 也是基于 AQS(AbstractQueuedSynchronizer) 实现的

- 初始化一个 CountDownLatch 时告诉并发的线程，然后在每个线程处理完毕之后调用 countDown() 方法。
- 该方法会将 AQS 内置的一个 state 状态 -1 。
- 最终在主线程调用 await() 方法，它会阻塞直到 `state == 0` 的时候返回。

#### CyclicBarrier

CyclicBarrier 中文名叫做屏障或者是栅栏，也可以用于线程间通信。

它可以等待 N 个线程都达到某个状态后继续运行的效果。

1. 首先初始化线程参与者。
2. 调用 `await()` 将会在所有参与者线程都调用之前等待。
3. 直到所有参与者都调用了 `await()` 后，所有线程从 `await()` 返回继续后续逻辑。

#### interrupt

可以采用中断线程的方式来通信，调用了 `thread.interrupt()` 方法其实就是将 thread 中的一个标志属性置为了 true。

并不是说调用了该方法就可以中断线程，如果不对这个标志进行响应其实是没有什么作用(这里对这个标志进行了判断)。

**但是如果抛出了 InterruptedException 异常，该标志就会被 JVM 重置为 false。**



#### 线程池awaitTermination()方法

​	使用这个 `awaitTermination()` 方法的前提需要关闭线程池，如调用了 `shutdown()` 方法。

调用了 `shutdown()` 之后线程池会停止接受新任务，并且会平滑的关闭线程池中现有的任务。

#### 管道通信

```
   public static void piped() throws IOException {
        //面向于字符 PipedInputStream 面向于字节
        PipedWriter writer = new PipedWriter();
        PipedReader reader = new PipedReader();

        //输入输出流建立连接
        writer.connect(reader);


        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                LOGGER.info("running");
                try {
                    for (int i = 0; i < 10; i++) {

                        writer.write(i+"");
                        Thread.sleep(10);
                    }
                } catch (Exception e) {

                } finally {
                    try {
                        writer.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }

            }
        });
        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                LOGGER.info("running2");
                int msg = 0;
                try {
                    while ((msg = reader.read()) != -1) {
                        LOGGER.info("msg={}", (char) msg);
                    }

                } catch (Exception e) {

                }
            }
        });
        t1.start();
        t2.start();
    }
```



## JVM

java运行时内存划分：

![img](../Spark-Knowledge/resources/5d31384c568c531115.jpg)



### 程序计数器

记录当前线程所执行的字节码行号，用于获取下一条执行的字节码。

当多线程运行时，每个线程切换后需要知道上一次所运行的状态、位置。由此也可以看出程序计数器是每个线程**私有**的。

### 虚拟机栈

虚拟机栈由一个一个的栈帧组成，栈帧是在每一个方法调用时产生的。

每一个栈帧由`局部变量区`、`操作数栈`等组成。每创建一个栈帧压栈，当一个方法执行完毕之后则出栈。

- 如果出现方法递归调用出现死循环的话就会造成栈帧过多，最终会抛出 `StackOverflowError`。
- 若线程执行过程中栈帧大小超出虚拟机栈限制，则会抛出 `StackOverflowError`。
- 若虚拟机栈允许动态扩展，但在尝试扩展时内存不足，或者在为一个新线程初始化新的虚拟机栈时申请不到足够的内存，则会抛出 `OutOfMemoryError`。

**这块内存区域也是线程私有的。**

### 堆内存

`Java` 堆是整个虚拟机所管理的最大内存区域，所有的对象创建都是在这个区域进行内存分配。

可利用参数 `-Xms -Xmx` 进行堆内存控制。

这块区域也是垃圾回收器重点管理的区域，由于大多数垃圾回收器都采用`分代回收算法`，所有堆内存也分为 `新生代`、`老年代`，可以方便垃圾的准确回收。

**这块内存属于线程共享区域。**



### 方法区（jdk1.7）

方法区主要用于存放已经被虚拟机加载的类信息，如`常量，静态变量`。 这块区域也被称为`永久代`。

可利用参数 `-XX:PermSize -XX:MaxPermSize` 控制初始化方法区和最大方法区大小。

### 元数据区（jdk1.8）

在 `JDK1.8` 中已经移除了方法区（永久代），并使用了一个元数据区域进行代替（`Metaspace`）。

默认情况下元数据区域会根据使用情况动态调整，避免了在 1.7 中由于加载类过多从而出现 `java.lang.OutOfMemoryError: PermGen`。

但也不能无线扩展，因此可以使用 `-XX:MaxMetaspaceSize`来控制最大内存。



### 运行时常量池

运行时常量池是方法区的一部分，其中存放了一些符号引用。当 `new` 一个对象时，会检查这个区域是否有这个符号的引用。

### 直接内存

直接内存又称为 `Direct Memory（堆外内存）`，它并不是由 `JVM` 虚拟机所管理的一块内存区域。

有使用过 `Netty` 的朋友应该对这块并内存不陌生，在 `Netty` 中所有的 IO（nio） 操作都会通过 `Native` 函数直接分配堆外内存。

它是通过在堆内存中的 `DirectByteBuffer` 对象操作的堆外内存，避免了堆内存和堆外内存来回复制交换复制，这样的高效操作也称为`零拷贝`。

既然是内存，那也得是可以被回收的。但由于堆外内存不直接受 `JVM` 管理，所以常规 `GC` 操作并不能回收堆外内存。它是借助于老年代产生的 `fullGC` 顺便进行回收。同时也可以显式调用 `System.gc()` 方法进行回收（前提是没有使用 `-XX:+DisableExplicitGC` 参数来禁止该方法）。

**值得注意的是**：由于堆外内存也是内存，是由操作系统管理。如果应用有使用堆外内存则需要平衡虚拟机的堆内存和堆外内存的使用占比。避免出现堆外内存溢出。



![img](../Spark-Knowledge/resources/5d31384cbc79744624.jpg)



通过上图可以直观的查看各个区域的参数设置。

常见的如下：

- `-Xms64m` 最小堆内存 `64m`.
- `-Xmx128m` 最大堆内存 `128m`.
- `-XX:NewSize=30m` 新生代初始化大小为`30m`.
- `-XX:MaxNewSize=40m` 新生代最大大小为`40m`.
- `-Xss=256k` 线程栈大小。
- `-XX:+PrintHeapAtGC` 当发生 GC 时打印内存布局。
- `-XX:+HeapDumpOnOutOfMemoryError` 发送内存溢出时 dump 内存。

新生代和老年代的默认比例为 `1:2`，也就是说新生代占用 `1/3`的堆内存，而老年代占用 `2/3` 的堆内存。

可以通过参数 `-XX:NewRatio=2` 来设置老年代/新生代的比例。

### OOM分析

#### 堆内存溢出

在 Java 堆中只要不断的创建对象，并且 `GC-Roots` 到对象之间存在引用链，这样 `JVM` 就不会回收对象。

只要将`-Xms(最小堆)`,`-Xmx(最大堆)` 设置为一样禁止自动扩展堆内存。

当使用一个 `while(true)` 循环来不断创建对象就会发生 `OutOfMemory`，还可以使用 `-XX:+HeapDumpOutofMemoryErorr` 当发生 OOM 时会自动 dump 堆栈到文件中。

当出现 OOM 时可以通过工具来分析 `GC-Roots` [引用链](https://github.com/crossoverJie/Java-Interview/blob/master/MD/GarbageCollection.md#可达性分析算法) ，查看对象和 `GC-Roots` 是如何进行关联的，是否存在对象的生命周期过长，或者是这些对象确实改存在的，那就要考虑将堆内存调大了。

#### MetaSpace内存溢出

`JDK8` 中将永久代移除，使用 `MetaSpace` 来保存类加载之后的类信息，字符串常量池也被移动到 Java 堆。

DK 8 中将类信息移到到了本地堆内存(Native Heap)中，将原有的永久代移动到了本地堆中成为 `MetaSpace` ,如果不指定该区域的大小，JVM 将会动态的调整。

可以使用 `-XX:MaxMetaspaceSize=10M` 来限制最大元数据。这样当不停的创建类时将会占满该区域并出现 `OOM`。

## 对象的创建与内存分配

### 创建对象

当 `JVM` 收到一个 `new` 指令时，会检查指令中的参数在常量池是否有这个符号的引用，还会检查该类是否已经被[加载](https://github.com/crossoverJie/Java-Interview/blob/master/MD/ClassLoad.md)过了，如果没有的话则要进行一次类加载。

接着就是分配内存了，通常有两种方式：

- 指针碰撞
- 空闲列表

使用指针碰撞的前提是堆内存是**完全工整**的，用过的内存和没用的内存各在一边每次分配的时候只需要将指针向空闲内存一方移动一段和内存大小相等区域即可。

当堆中已经使用的内存和未使用的内存**互相交错**时，指针碰撞的方式就行不通了，这时就需要采用空闲列表的方式。虚拟机会维护一个空闲的列表，用于记录哪些内存是可以进行分配的，分配时直接从可用内存中直接分配即可。

堆中的内存是否工整是有**垃圾收集器**来决定的，如果带有压缩功能的垃圾收集器就是采用指针碰撞的方式来进行内存分配的。

分配内存时也会出现并发问题:

这样可以在创建对象的时候使用 `CAS` 这样的乐观锁来保证。

也可以将内存分配安排在每个线程独有的空间进行，每个线程首先在堆内存中分配一小块内存，称为本地分配缓存(`TLAB : Thread Local Allocation Buffer`)。

分配内存时，只需要在自己的分配缓存中分配即可，由于这个内存区域是线程私有的，所以不会出现并发问题。

可以使用 `-XX:+/-UseTLAB` 参数来设定 `JVM` 是否开启 `TLAB` 。

内存分配之后需要对该对象进行设置，如对象头。



### 对象访问

一个对象被创建之后自然是为了使用，在 `Java` 中是通过栈来引用堆内存中的对象来进行操作的。

对于我们常用的 `HotSpot` 虚拟机来说，这样引用关系是通过直接指针来关联的。





# Kafka

**1、请说明什么是Apache Kafka?**
Apache Kafka是由Apache开发的一种发布订阅消息系统，它是一个分布式的、分区的和重复的日志服务。

**2、请说明什么是传统的消息传递方法?**

传统的消息传递方法包括两种：

排队：在队列中，一组用户可以从服务器中读取消息，每条消息都发送给其中一个人。
发布-订阅：在这个模型中，消息被广播给所有的用户。
**3、请说明Kafka相对传统技术有什么优势?**

Apache Kafka与传统的消息传递技术相比优势之处在于：

快速:单一的Kafka代理可以处理成千上万的客户端，每秒处理数兆字节的读写操作。

可伸缩:在一组机器上对数据进行分区和简化，以支持更大的数据

持久:消息是持久性的，并在集群中进行复制，以防止数据丢失。

设计:它提供了容错保证和持久性

**4、在Kafka中broker的意义是什么?**

在Kafka集群中，broker术语用于引用服务器。

**5、Kafka服务器能接收到的最大信息是多少?**

Kafka服务器可以接收到的消息的最大大小由参数message.max.bytes决定，010版本默认值是1000012，可以配置为broker级别或者topic级别。

**6、解释Kafka的Zookeeper是什么?我们可以在没有Zookeeper的情况下使用Kafka吗?**

Zookeeper在分布式集群中有广泛的应用，它可以用作 ***数据发布与订阅***、***负载均衡***、***命名服务***、***分布式通知/协调***、***集群管理与Master选举***、***分布式锁***、***分布式队列*** 。

Zookeeper是一个开放源码的、高性能的协调服务，它用于Kafka的分布式应用。

不，不可能越过Zookeeper，直接联系Kafka broker。一旦Zookeeper停止工作，它就不能服务客户端请求。

Zookeeper主要用于在集群中不同节点之间进行通信
在Kafka中，它被用于提交偏移量，因此如果节点在任何情况下都失败了，它都可以从之前提交的偏移量中获取
除此之外，它还执行其他活动，如: leader检测、分布式同步、配置管理、负载均衡（识别新节点何时离开或连接、集群、节点实时状态）等等。


[zookeeper 在kafka中的作用](https://mp.weixin.qq.com/s?__biz=MzUyMDA4OTY3MQ==&mid=2247487331&amp;idx=1&amp;sn=e945f923cbc79bb19596ea206ac3ecfe&source=41#wechat_redirect)

**7、解释Kafka的用户如何消费信息?**

在Kafka中传递消息是通过使用sendfile API完成的。它支持将字节从套接口转移到磁盘，通过内核空间保存副本，并在内核用户之间调用内核。

消费者消费有各种客户端：

010:http://kafka.apache.org/0102/javadoc/index.html?org/apache/kafka/clients/consumer/KafkaConsumer.html

082 分高阶API和低阶API：

https://cwiki.apache.org/confluence/display/KAFKA/Consumer+Group+Example

https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example

**8、解释如何提高远程用户的吞吐量?**

如果用户位于与broker不同的数据中心，则可能需要调优套接口缓冲区大小，以对长网络延迟进行摊销。

**9、解释一下，在数据制作过程中，你如何能从Kafka得到准确的信息?**

在数据中，为了精确地获得Kafka的消息，你必须遵循两件事: 在数据消耗期间避免重复，在数据生产过程中避免重复。

这里有两种方法，可以在数据生成时准确地获得一个语义:

每个分区使用一个单独的写入器，每当你发现一个网络错误，检查该分区中的最后一条消息，以查看您的最后一次写入是否成功
在消息中包含一个主键(UUID或其他)，并在用户中进行反复制

**10、解释如何减少ISR中的扰动?broker什么时候离开ISR?**

ISR是一组与leaders完全同步的消息副本，也就是说ISR中包含了所有提交的消息。ISR应该总是包含所有的副本，直到出现真正的故障。如果一个副本从leader中脱离出来，将会从ISR中删除。

**11、Kafka为什么需要复制?**

Kafka的信息复制确保了任何已发布的消息不会丢失，并且可以在机器错误、程序错误或更常见些的软件升级中使用。

**12、如果副本在ISR中停留了很长时间表明什么?**

如果一个副本在ISR中保留了很长一段时间，那么它就表明，跟踪器可以像在leader收集数据那样快速地获取数据。

**13、请说明如果首选的副本不在ISR中会发生什么?**

如果首选的副本不在ISR中，控制器将无法将leadership转移到首选的副本。

**14、有可能在生产后发生消息偏移吗?**

在大多数队列系统中，作为生产者的类无法做到这一点，它的作用是触发并忘记消息。broker将完成剩下的工作，比如使用id进行适当的元数据处理、偏移量等。

作为消息的用户，你可以从Kafka broker中获得补偿。如果你注视SimpleConsumer类，你会注意到它会获取包括偏移量作为列表的MultiFetchResponse对象。此外，当你对Kafka消息进行迭代时，你会拥有包括偏移量和消息发送的MessageAndOffset对象。

**15、kafka提高吞吐量的配置**

最基础的配置是

batch.size 默认是单批次最大16384字节，超过该值就立即发送。

linger.ms 默认是0ms，超过该时间就立即发送。

上面两个条件满足其一，就立即发送消息否则等待。

**16、kafka支持事务吗？**

0.11版本以后开始支持事务的生产者和消费者。

**17、kafka可以指定时间范围消费吗？**

0.10.2版本以后支持指定时间戳范围消费kafka数据。

**18、新增分区Spark 能发现吗**

Spark Streaming针对kafka0.8.2及以前版本不能进行新增分区及topic发现，0.10以后版本是可以动态检测新增分区和topic。

**19、kafka分区数怎么设定呢？**

一般可以设置为broker或者磁盘的整数倍，然后再结合数据量和后段消费者处理的复杂度及消费者组的数来确定。

**20、kafka的重要监控指标有哪些**

磁盘对与kafka来说比较重要，尽量做矩阵和监控，避免集群故障，而且磁盘问题最容易引起吞吐量下降。

网卡流量，由于副本同步，消费者多导致网路带宽很容易吃紧，所以监控也比较重要。

topic流量波动情况，这个主要是为了后端应对流量尖峰作准备。

消费者lagsize，也即使消费者滞后情况。

**21、Kafka如何保证写入速度**
* 页缓存技术 + 磁盘顺序写
* 零拷贝

**22、Kafka如何做到数据不丢失不重复**

分为三种：

**最多一次（at most once）**: 消息可能丢失也可能被处理，但最多只会被处理一次。


**至少一次（at least once）**: 消息不会丢失，但可能被处理多次。


**精确传递一次（exactly once）**: 消息被处理且只会被处理一次。

**Producer** 生产消息需要需要配置ack参数，有三种设置，分别如下

 * 0： producer完全不管broker的处理结果 回调也就没有用了 并不能保证消息成功发送 但是这种吞吐量最高

*   -1或者all： leader broker会等消息写入 并且ISR都写入后 才会响应，这种只要ISR有副本存活就肯定不会丢失，但吞吐量最低。

 * 1： 默认的值 leader broker自己写入后就响应，不会等待ISR其他的副本写入，只要leader broker存活就不会丢失，即保证了不丢失，也保证了吞吐量。

所以设置为0时，实现了at most once，而且从这边看只要保证集群稳定的情况下，不设置为0，消息不会丢失。

但是还有一种情况就是消息成功写入，而这个时候由于网络问题producer没有收到写入成功的响应，producer就会开启重试的操作，直到网络恢复，消息就发送了多次。这就是at least once了。

kafka producer 的参数acks 的默认值为1，所以默认的producer级别是at least once。并不能exactly once。

**24、Kafka如何保证数据不丢失**

**Kafka如何保障数据不丢失**，即**Kafka的Broker提供了什么机制保证数据不丢失的。**

对于Kafka的Broker而言，Kafka 的**复制机制**和**分区的多副本**架构是Kafka 可靠性保证的核心。把消息写入多个副本可以使Kafka 在发生崩溃时仍能保证消息的持久性。

搞清楚了问题的核心，再来看一下该怎么回答这个问题：主要包括三个方面：

- Topic 副本因子个数：replication.factor >= 3
- 同步副本列表(ISR)：min.insync.replicas = 2
- 禁用unclean选举：unclean.leader.election.enable=false



**24.1 副本因子**

Kafka的topic是可以分区的，并且可以为分区配置多个副本，该配置可以通过`replication.factor`参数实现。

Kafka中的分区副本包括两种类型：领导者副本（Leader Replica）和追随者副本（Follower Replica)，每个分区在创建时都要选举一个副本作为领导者副本，其余的副本自动变为追随者副本。

