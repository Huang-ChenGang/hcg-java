## Java线程模型
Java创建和销毁线程时会通过JVM调用操作系统，JVM和操作系统的对应关系就是线程模型。
Java线程模型分类：一对一、多对一、多对多
1. 一对一：当前主流Java虚拟机都采用这个模型。
2. 多对一：当一个用户线程阻塞，造成当前的内核线程阻塞，就会阻塞其他用户线程。
3. 多对多：实现起来比较难，但是能够实现更大并发量，GO语言就是采用这个模型。

## Java锁机制
Java锁机制的概念：在多线程并发情况下可能会有多个线程对同一个资源进行争抢，会导致数据不一致的问题。
Java就抽象了一个锁概念，用来对资源进行锁定，保证数据一致性。

在Java中每个对象都拥有一把锁，存放在对象头中，锁里边记录了当前对象被哪个线程所占用。

Java对象构成：对象头、实例对象、填充字节。
1. 对象头：MarkWord和ClassPointer组成。
    MarkWord：存储和当前对象运行时有关的数据，包含HashCode、分代年龄、锁标志等。
    ClassPointer：是一个指针，指向当前对象类型指向方法区中的类型数据。
2. 实例对象
3. 填充字节：保证每个Java对象大小是8bit(1Byte)的倍数，无用字节。

synchronized关键字原理（优化前）：synchronized会编译成monitorenter和monitorexit两个字节码命令。
资源被monitorenter之后不可被其他线程获取，monitorexit之后才能被其他线程获取。
依赖于操作系统，所以每当挂起或唤醒Java线程都需要切换到内核态，非常重量级。
Java6之后synchronized进行了优化，引入了偏向锁和轻量级锁。

Java锁状态由低到高：无锁、偏向锁、轻量级锁、重量级锁。
锁只能升级不能降级。

当对象的锁为偏向锁时，对象头中存有可以访问这个对象的线程ID，可以理解为这个对象偏爱这个线程，所以叫偏向锁。
当偏向锁对象发现有多个线程竞争时，就会升级为轻量级锁。

当对象锁为轻量级锁时，一个线程想要获取轻量级锁时，会在自己的虚拟机栈中开辟一块空间，叫LockRecord。
虚拟机栈是线程私有的，不存在同步问题。
LockRecord中存放的是对象头中的MarkWord副本以及Owner指针。
当线程用CAS获取到轻量级锁之后，会把对象头中的MarkWord复制一份到自己的LockRecord中，然后将Owner指针指向对象锁。
对象头中的MarkWord前30位就会生成一个指针，指向持有这个对象锁的线程的LockRecord。
这样对象和线程就会相互知道对方的存在，就会形成一个绑定的关系。
其他线程再想要获得这个轻量级锁就会进入自旋等待，当对象发现有多个线程自旋时，就会升级为重量级锁。

## 无锁编程
### CAS
CAS: Compare And Swap，比较并交换。
CAS是无锁方式竞争资源的方法。
CAS在操作系统中通过一条指令来实现，所以能够保证原子性。
线程首先回去读取资源，获得OldValue，然后用OldValue调用底层调用Cpu提供的compare and set的原子性操作。
减少了之前重量级锁的内核态和用户态切换的消耗，提高了性能。

### AQS
AQS: Abstract Queued Synchronizer，抽象队列同步器，AbstractQueuedSynchronizer.java。

AQS是构建Java同步组件的基础，我们期待它能够成为实现大部分同步需求的基础。
AQS的设计模式采用的模板方法模式，子类通过继承的方式，实现它的抽象方法来管理同步状态，
对于子类而言它并没有太多的活要做，AQS提供了大量的模板方法来实现同步，
主要是分为三类：独占式获取和释放同步状态、共享式获取和释放同步状态、查询同步队列中的等待线程情况。
自定义子类使用AQS提供的模板方法就可以实现自己的同步语义。

AQS 定义两种资源共享方式：独占和共享。
AQS主要是用state和FIFO(先进先出)的队列来管理多线程的同步状态。

成员变量：
```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    
    /*
     * 共享变量，使用volatile修饰保证线程可见性
     * 表示当前竞争资源对象的线程数量。
     * 使用int就是因为有独占和共享两种资源共享方式，当是共享的时候，占用资源的线程数量是有多个的。
     */
    private volatile int state;

    // 当前等待队列的链表头
    private transient volatile Node head;

    // 当前等待队列的链表尾
    private transient volatile Node tail;

    /**
     * 队列中的节点，Node类
     */
    static final class Node {
        // 共享
        static final Node SHARED = new Node();
        // 独占
        static final Node EXCLUSIVE = null;
    
        /**
         * 当前节点获取锁的请求处于取消状态
         * 因为超时或者中断，节点会被设置为取消状态
         * 被取消的节点是不会参与到竞争中的，他会一直保持取消状态不会转变为其他状态
         */
        static final int CANCELLED =  1;
        /**
         * 当前节点拥有锁或处于竞争锁的状态
         * 当前节点之后的后继节点的线程处于等待状态
         * 而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行
         */
        static final int SIGNAL    = -1;
        /**
         * 当前节点在等待队列中，节点线程等待在Condition上
         * 当其他线程对Condition调用了signal()后，改节点将会从等待队列中转移到同步队列中，加入到同步状态的获取中
         */
        static final int CONDITION = -2;
        /**
         * 传递共享模式下锁释放状态，和共享模式相关
         * 表示下一次共享式同步状态获取将会无条件地传播下去
         */
        static final int PROPAGATE = -3;
        
        /** 当前节点的线程在队列里的等待状态
         * 是一个枚举值，主要包含上边的四个状态
         * 0：节点初始化默认值，或节点已经释放锁
         */
        volatile int waitStatus;
        /** 前指针 */
        volatile Node prev;
        /** 后指针 */
        volatile Node next;
        /** 线程对象 */
        volatile Thread thread;
    }

}
```

独占式(Exclusive)：只有一个线程可以使用资源，如 ReentrantLock。
队列里只有一个线程在竞争锁，其他线程都被挂起。
被挂起的线程会在一个线程使用完了共享资源，将要释放锁的时候被唤醒。

独占方式代码分析：
```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    /**
     * 尝试获取锁(修改标记位)，无论有没有成功，都会立即返回
     * protected修饰：需要被继承
     * 参数arg：代表对state的修改
     * 返回值：代表是否成功获得锁
     *
     * 该方法必须要保证线程安全的获取同步状态
     */
    protected boolean tryAcquire(int arg) {
        // 直接抛出异常，需要继承类Override这个tryAcquire方法
        throw new UnsupportedOperationException();
    }

    /**
     * tryAcquire的增强版方法，添加超时控制，并且响应中断
     * 如果当前线程没有在指定时间内获取同步状态，则会返回false，否则返回true
     * 时间单位为纳秒
     */
    public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }

    /**
     * 获取锁(修改标记位)，如果没有成功就进入队列等待，直到成功获取
     * 修饰符：public final，不允许继承类Override
     *
     * 该方法不响应中断
     */
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            /*
             * 如果acquireQueued返回true，表明线程在挂起的过程中有外部中断请求
             * 因为线程在挂起的过程中无法响应外部的中断请求，之后改变状态，所以这里自己中断一下
             */
            selfInterrupt();
    }

    /**
     * 同acquire方法，但是会响应中断
     * 获取锁(修改标记位)，如果没有成功就进入队列等待，直到成功获取
     * 修饰符：public final，不允许继承类Override
     *
     * 如果当前线程被中断，会直接抛出InterruptedException
     */
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }

    /**
     * Creates and enqueues node for current thread and given mode.
     * 将当前线程封装为一个Node加入等待队列
     *
     * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
     * @return the new node， 当前节点
     */
    private Node addWaiter(Node mode) {
        Node node = new Node(mode);
    
        for (;;) {
            // 队列是先进先出，所以要插入到队尾
            Node oldTail = tail;
            if (oldTail != null) {
                node.setPrevRelaxed(oldTail);
                if (compareAndSetTail(oldTail, node)) {
                    oldTail.next = node;
                    return node;
                }
            } else {
                initializeSyncQueue();
            }
        }
    }

    /**
     * 方法整体流程：
     * 如果当前节点是头结点的后面一个，那么将会不断的去尝试拿锁，直到拿锁成功。
     * 否则进行判断是否需要挂起(也就是说，整个队列里只有一个线程是不断尝试拿锁的，其他线程都被挂起)。
     *
     * 这个方法主要是对线程进行挂起，不响应中断
     */
    final boolean acquireQueued(final Node node, int arg) {
        boolean interrupted = false;
        try {
            for (;;) {
                final Node p = node.predecessor();
                
                // 如果当前节点是头结点的后面一个，那么将会不断的去尝试拿锁，直到拿锁成功
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    return interrupted;
                }

                /*
                 * 判断是否需要挂起
                 * 判断条件：当前节点之前除了head还有其他节点，并且前一个节点的状态是SIGNAL(-1)，则当前节点的线程需要挂起。
                 * 这样就保证head节点之后只有一个在通过CAS获取锁。队列里边其他线程都已被挂起或正在被挂起，最大限度的避免无用的自旋消耗CPU
                 */
                if (shouldParkAfterFailedAcquire(p, node))
                    // 当线程被挂起的过程中有外部中断请求的话，这里会返回true
                    interrupted |= parkAndCheckInterrupt();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            if (interrupted)
                selfInterrupt();
            throw t;
        }
    }

    /**
     * 同acquireQueued，方法整体流程：
     * 如果当前节点是头结点的后面一个，那么将会不断的去尝试拿锁，直到拿锁成功。
     * 否则进行判断是否需要挂起(也就是说，整个队列里只有一个线程是不断尝试拿锁的，其他线程都被挂起)。
     *
     * 这个方法主要是对线程进行挂起，如果当前线程被中断，直接抛出InterruptedException
     */
    private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    // 响应中断，不再使用interrupted标记是否被中断过，而是直接抛出InterruptedException
                    throw new InterruptedException();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            throw t;
        }
    }
    
    /**
     * 同acquireQueued，添加超时控制
     * 程序首先记录唤醒时间deadline ，deadline = System.nanoTime() + nanosTimeout（时间间隔）
     * 如果获取同步状态失败，则需要计算出需要休眠的时间间隔nanosTimeout（= deadline - System.nanoTime()）
     * 如果nanosTimeout <= 0 表示已经超时了，返回false
     * 如果大于spinForTimeoutThreshold（1000L）则需要休眠nanosTimeout
     * 如果nanosTimeout <= spinForTimeoutThreshold ，就不需要休眠了，直接进入快速自旋的过程。
     * 原因在于 spinForTimeoutThreshold 已经非常小了，非常短的时间等待无法做到十分精确，
     * 如果这时再次进行超时等待，相反会让nanosTimeout 的超时从整体上面表现得不是那么精确，
     * 所以在超时非常短的场景中，AQS会进行无条件的快速自旋。
     */
    private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L) {
                    cancelAcquire(node);
                    return false;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > SPIN_FOR_TIMEOUT_THRESHOLD)
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            throw t;
        }
    }
    
    /**
     * 如果这个方法返回true，则表明当前节点需要被挂起
     */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        
        // 当前节点的前置节点waitStatus为SIGNAL，说明前置节点在等待获取锁，当前节点可以挂起
        if (ws == Node.SIGNAL)
            return true;
        if (ws > 0) {
            // 当前节点的前置节点waitStatus > 0，只可能为CANCELLED，所以直接删除前置节点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * 这里会更改前置节点的waitStatus
             * 当前置节点为0或其他非SIGNAL状态的负数时
             * 既然当前节点已经压入，那前置节点就应该做好准备竞争锁，所以waitStatus置为SIGNAL
             */
            pred.compareAndSetWaitStatus(ws, Node.SIGNAL);
        }
    
        // 后两种情况返回false，进行下一轮的判断
        return false;
    }

    /*
     * 挂起线程
     */
    private final boolean parkAndCheckInterrupt() {
        /*
         * 调用操作系统原语来将当前线程挂起，挂起后线程是停在这里的，不会往下执行
         *
         * 因为调用操作系统原语来挂起当前线程，
         * 所以挂起之后如果有其他地方调用了当前线程的interrupt()方法，是不会抛出异常的，只会改变当前线程内部的终端状态值
         * 简单来说就是当线程处于等待队列中时无法响应外部的中断请求
         */
        LockSupport.park(this);

        /*
         * 挂起的线程被唤醒后从这里开始继续执行
         * 当线程在挂起的过程中有外部终端请求的话这里会返回true
         */
        return Thread.interrupted();
    }

    /**
     * 取消当前线程获取资源
     */
    private void cancelAcquire(Node node) {
        if (node == null)
            return;

        // 清除线程
        node.thread = null;

        // 清除所有cancelled前置节点
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;

        // 获取到第一个非cancelled的前置节点，并获取他的下个节点，用于之后的CAS
        Node predNext = pred.next;

        // 当前节点状态设置为CANCELLED
        node.waitStatus = Node.CANCELLED;

        // 当前节点是尾结点，直接删除，并把前置节点设置为尾结点
        if (node == tail && compareAndSetTail(node, pred)) {
            pred.compareAndSetNext(predNext, null);
        } else {
            // 如果前置节点不是头节点并且是其他等待获取资源的状态(<=0)，则置为SIGNAL
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||
                 (ws <= 0 && pred.compareAndSetWaitStatus(ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    pred.compareAndSetNext(predNext, next);
            } else {
                // 否则直接唤醒下一节点，唤醒方法中会把已取消状态的节点删除
                unparkSuccessor(node);
            }

            node.next = node; // help GC
        }
    }

    /**
     * 尝试释放锁，给继承类去实现
     */
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }
    
    /**
     * 释放锁
     * 假如尝试释放锁成功，下一步就去唤醒等待队列里的其他节点
     */
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

    /**
     * 唤醒下一个节点
     */
    private void unparkSuccessor(Node node) {
        /*
         * 这里会更改节点的waitStatus，当前节点为负数的状态时，会被置为0，表示释放资源
         */
        int ws = node.waitStatus;
        if (ws < 0)
            node.compareAndSetWaitStatus(ws, 0);

        /*
         * 如果下一个节点存在则唤醒下一个节点，如果不存在则从尾部开始，唤醒最前边的一个
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node p = tail; p != node && p != null; p = p.prev)
                if (p.waitStatus <= 0)
                    s = p;
        }
        if (s != null)
            // 调用操作系统原语唤醒线程
            LockSupport.unpark(s.thread);
    }

}
```

共享式(Share)：多个线程可以共享资源，如Semaphore、CountDownLatCh、CyclicBarrier、ReadWriteLock。
共享式与独占式的最主要区别在于同一时刻独占式只能有一个线程获取同步状态，而共享式在同一时刻可以有多个线程获取同步状态。
例如读操作可以有多个线程同时进行，而写操作同一时刻只能有一个线程进行写操作，其他操作都会被阻塞。

跟独占锁相比，共享锁的主要特征在于当一个在等待队列中的共享节点成功获取到锁以后（它获取到的是共享锁），
既然是共享，那它必须要依次唤醒后面所有可以跟它一起共享当前锁资源的节点，毫无疑问，
这些节点必须也是在等待共享锁（这是大前提，如果等待的是独占锁，那前面已经有一个共享节点获取锁了，它肯定是获取不到的）。
当共享锁被释放的时候，无论是独占模式的节点还是共享模式的节点都会被唤醒。

共享方式代码分析：
```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    
    /**
     * 尝试以共享模式获取。 该方法应该查询对象的状态是否允许在共享模式下获取它，如果允许则获取它。
     *
     * 返回值为负数：不能获取，需要进入等待队列
     * 返回值为0：可以获取成功，但是共享模式下之后的线程不会获取成功，也就是不需要把它后面等待的节点唤醒
     * 返回值为正数：可以获取成功，共享模式下之后的线程也能获取成功，也就是说此时需要把后续节点唤醒让它们去尝试获取共享锁
     */
    protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }

    /**
     * 共享模式获取锁(修改标记位)，如果没有成功就进入队列等待，直到成功获取
     * 修饰符：public final，不允许继承类Override
     *
     * 该方法不响应中断
     */
    public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

    /**
     * 共享模式获取锁，不响应中断
     */
    private void doAcquireShared(int arg) {
        // 添加一个共享节点到等待队列，方法跟独占模式一样
        final Node node = addWaiter(Node.SHARED);
        boolean interrupted = false;
        try {
            for (;;) {
                // 如果当前节点是头结点的后面一个，那么将会不断的去尝试拿锁，直到拿锁成功
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    // r>=0 表示获取锁成功，>0 需要唤醒后面的线程
                    if (r >= 0) {
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        return;
                    }
                }
                // 挂起线程逻辑，跟独占模式一样，不支持中断
                if (shouldParkAfterFailedAcquire(p, node))
                    interrupted |= parkAndCheckInterrupt();
            }
        } catch (Throwable t) {
            cancelAcquire(node);
            throw t;
        } finally {
            if (interrupted)
                selfInterrupt();
        }
    }

    /**
     * 设置头结点并唤醒后面线程
     * 保证唤醒的只有共享节点
     */
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * 这里有两种情况需要执行唤醒操作：
         * 1. propagate > 0 表示调用方指明了后继节点需要被唤醒
         * 2. 头节点后面的节点需要被唤醒（waitStatus<0）
         */
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            /*
             * 当前节点没有后继节点或后继节点是共享型，则进行唤醒
             * 这里可以理解为除非明确指明不需要唤醒（后继等待节点是独占类型），否则都要唤醒
             */
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }

    /**
     * 共享模式释放锁
     * 接下来等待独占锁跟共享锁的线程都可以被唤醒进行尝试获取
     */
    private void doReleaseShared() {
        /*
         * 唤醒操作由头结点开始，注意这里的头节点已经是doAcquireShared方法中获取到锁的节点了
         * 其实就是唤醒doAcquireShared方法中获取到锁的节点的后继节点
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                // 表示后继节点需要被唤醒
                if (ws == Node.SIGNAL) {
                    // 这里需要控制并发，因为入口有setHeadAndPropagate跟release两个，避免两次唤醒
                    if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                // 如果后继节点暂时不需要唤醒，则把当前节点状态设置为PROPAGATE确保以后可以传递下去
                else if (ws == 0 &&
                         !h.compareAndSetWaitStatus(0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            /*
             * 如果头结点没有发生变化，表示设置完成，退出循环
             * 如果头结点发生变化，比如说其他线程获取到了锁，为了使自己的唤醒动作可以传递，必须进行重试
             */
            if (h == head)                   // loop if head changed
                break;
        }
    }

    /**
     * 尝试释放锁，给继承类去实现
     */
    protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }
    
    /**
     * 释放锁
     * 假如尝试释放锁成功，下一步就去唤醒等待队列里的其他节点
     */
    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            // 见如上分析
            doReleaseShared();
            return true;
        }
        return false;
    }

    /*
     * 跟独占模式一样，共享模式也提供了支持中断和支持超时的方法，这里不再赘述
     */

}
```

AQS方法说明：

void acquire(int arg)：以独占模式获取锁，不响应中断。

void acquireInterruptibly(int arg)：以独占模式获取，如果被中断则终止。

void acquireShared​(int arg)：以共享模式获取锁，不响应中断。

void acquireSharedInterruptibly​(int arg)：以共享模式获取锁，如果被中断则终止。

protected boolean compareAndSetState​(int expect, int update)：如果当前状态值等于预期值，则自动将同步状态设置为给定的更新值。

Collection<Thread> getExclusiveQueuedThreads()：返回等待以独占模式获取资源的线程集合。

Thread getFirstQueuedThread()：返回队列中的第一个（等待时间最长的）线程，如果当前没有线程排队，则返回 null。

Collection<Thread> getQueuedThreads()：返回等待获取资源的线程集合。

int getQueueLength()：返回等待获取资源的线程数。

Collection<Thread> getSharedQueuedThreads()：返回等待以共享模式获取资源的线程集合。

protected int getState()：返回当前的同步状态。

Collection<Thread> getWaitingThreads​(AbstractQueuedSynchronizer.ConditionObject condition)：返回以特定条件等待获取资源的线程集合。

int getWaitQueueLength​(AbstractQueuedSynchronizer.ConditionObject condition)：返回以特定条件等待获取资源的线程数。

boolean hasContended()：返回是否阻塞过其他线程。

boolean hasQueuedPredecessors()：返回是否有其他线程等待的时间比当前线程等待的时间更长。

boolean	hasQueuedThreads()：返回当前是否有其他线程在等待。

boolean	hasWaiters​(AbstractQueuedSynchronizer.ConditionObject condition)：返回是否有其他线程以特定条件在等待。

protected boolean isHeldExclusively()：返回是否以独占模式在进行。

boolean	isQueued​(Thread thread)：查询指定线程是否在等待。

boolean owns​(AbstractQueuedSynchronizer.ConditionObject condition)：查询给定的 ConditionObject 是否使用这个同步器作为它的锁。

boolean	release​(int arg)：以独占模式释放锁。

boolean	releaseShared​(int arg)：以共享模式释放锁。

protected void setState​(int newState)：设置同步器状态。

protected boolean tryAcquire​(int arg)：以独占模式尝试获取锁。

boolean	tryAcquireNanos​(int arg, long nanosTimeout)：以独占模式尝试获取锁，如果被中断或超时，则终止。

protected int tryAcquireShared​(int arg)：以共享模式尝试获取锁。

boolean	tryAcquireSharedNanos​(int arg, long nanosTimeout)：以共享模式尝试获取锁，如果被中断或超时，则终止。

protected boolean tryRelease​(int arg)：尝试以独占模式释放锁。

protected boolean tryReleaseShared​(int arg)：尝试以共享模式释放锁。

高并发集合问题主要为第三代线程安全集合类，位于 java.util.concurrent.* 下，
ConcurrentHashMap等，
底层是CAS和AQS。