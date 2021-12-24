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
AQS: Abstract Queued Synchronizer，抽象队列同步器。
AQS 定义两种资源共享方式：独占和共享。
Exclusive（独占）：只有一个线程可以使用资源，如 ReentrantLock。
Share（共享）：多个线程可以共享资源，如Semaphore、CountDownLatCh、CyclicBarrier、ReadWriteLock。
独占和共享在表现的意义上不一样，但是在底层的处理逻辑上没有太大的差别。

AQS主要是用state和FIFO(先进先出)的队列来管理多线程的同步状态。

AbstractQueuedSynchronizer.java 源码解析。

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
        /** Marker to indicate a node is waiting in shared mode */
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        static final Node EXCLUSIVE = null;
    
        /**
         * waitStatus value to indicate thread has cancelled.
         * 当前节点获取锁的请求已经被取消了
         */
        static final int CANCELLED =  1;
        /**
         * waitStatus value to indicate successor's thread needs unparking.
         * 当前节点之后的线程需要被唤醒
         */
        static final int SIGNAL    = -1;
        /**
         * waitStatus value to indicate thread is waiting on condition.
         * 当前节点正在等待某一个condition对象，和条件模式相关
         */
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should unconditionally propagate.
         * 传递共享模式下锁释放状态，和共享模式相关
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

获取锁的方法：
```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    /**
     * 尝试获取锁(修改标记位)，无论有没有成功，都会立即返回
     * protected修饰：需要被继承
     * 参数arg：代表对state的修改
     * 返回值：代表是否成功获得锁
     */
    protected boolean tryAcquire(int arg) {
        // 直接抛出异常，需要继承类Override这个tryAcquire方法
        throw new UnsupportedOperationException();
    }

    /**
     * 获取锁(修改标记位)，如果没有成功就进入队列等待，直到成功获取
     * 修饰符：public final，不允许继承类Override
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
     * 这个方法主要是对线程进行挂起
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
     * 如果这个方法返回true，则表明当前节点需要被挂起
     */
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        
        // 当前节点的前置节点waitStatus为SIGNAL，说明前置节点在等待获取锁，当前节点可以挂起
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            // 当前节点的前置节点waitStatus > 0，只可能为CANCELLED，所以直接删除前置节点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            // 前置节点为其他状态，既然当前节点已经压入，那前置节点就应该做好准备竞争锁，所以waitStatus置为SIGNAL
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

}
```

队列里只有一个线程在竞争锁，其他线程都被挂起。
被挂起的线程会在一个线程使用完了共享资源，将要释放锁的时候被唤醒。

释放锁的方法：
```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    /**
     * 给继承类去实现
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
     * 如果下一个节点存在则唤醒下一个节点，如果不存在则从尾部开始唤醒第一个
     */
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            node.compareAndSetWaitStatus(ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
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

高并发集合问题主要为第三代线程安全集合类，位于 java.util.concurrent.* 下，
ConcurrentHashMap等，
主要是CAS和AQS。