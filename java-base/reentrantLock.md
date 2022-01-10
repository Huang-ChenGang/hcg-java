## ReentrantLock：独占模式的可重入锁

ReentrantLock基于ASQ，在并发编程中轻松实现公平锁和非公平锁对共享资源进行同步。
同时，和synchronized一样，ReentrantLock支持可重入。
除此之外，ReentrantLock在调度上更灵活，支持更多丰富的功能。

## ReentrantLock源码分析：

Lock类：
```java
package java.util.concurrent.locks;

/**
 * 实现提供了比使用synchronized方法和语句所能获得的更广泛的锁定操作。
 * 允许更灵活的结构，并且可能支持多个关联的Condition对象。
 */
public interface Lock {
    
    /**
     * 获取锁
     * 假如当前锁正在被其他线程占用将会继续等待，直到获取为止
     * 不响应中断
     */
    void lock();

    /**
     * 获取锁
     * 假如当前锁正在被其他线程占用将会继续等待，直到获取为止
     * 如果当前线程被中断则会停止等待，并抛出InterruptedException异常
     */
    void lockInterruptibly() throws InterruptedException;

    /**
     * 尝试获取锁并立即返回
     * 返回值表示释放获取成功
     */
    boolean tryLock();

    /**
     * 在一段时间内尝试获取锁
     * 如果获取锁期间被中断，将会抛出InterruptedException异常
     */
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    /**
     * 释放锁
     */
    void unlock();

    /**
     * 新建一个绑定在当前lock上的condition对象
     *
     * Condition可以理解为一个等待状态。
     * 获得锁的线程可能在某些时刻需要等待一些条件的完成才能继续执行，
     * 这个时候就可以通过condition的await进行等待，通过condition的signal方法进行唤醒。
     * 有点类似于Object的wait和notify方法，不同的是一个lock对象可以关联多个condition对象，
     * 多个线程可以绑定在不同的condition对象上。这样就可以做到分组唤醒。
     * Condition还提供了和限时、中断相关的功能。
     */
    Condition newCondition();
}
```

ReentrantLock类：
```java
package java.util.concurrent.locks;
public class ReentrantLock implements Lock, java.io.Serializable {
    
    // final修饰，一经初始化不可被修改
    private final Sync sync;

    /**
     * 抽象类，需要通过子类实现
     * 锁的同步控制基础，下面分为公平和非公平版本
     * 继承AQS，使用AQS状态来表示持有锁的次数
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {

        /**
         * 非公平方式直接获取锁
         * final修饰，不可被子类修改
         * 返回值表示是否成功获取锁
         */
        @ReservedStackAccess
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();

            // 获取当前锁状态
            int c = getState();

            // 锁空闲，直接需改锁状态(非公平模式体现)，并把当前线程设置为独占模式拥有者
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                // 当前线程为独占模式拥有者，则修改锁状态(拥有锁的线程数)，体现了可重入
                int nextc = c + acquires;

                // 当锁被获取的次数超过int的最大值就会溢出变为负数
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");

                setState(nextc);
                return true;
            }
            return false;
        }

        /**
         * 释放锁，返回值表示锁是否被完全释放
         */
        @ReservedStackAccess
        protected final boolean tryRelease(int releases) {
            // 获取剩余拥有锁的线程数
            int c = getState() - releases;
            // 不允许是否其他线程的锁
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                // 锁被完全释放，清空锁的独占拥有者
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

        /**
         * 检查当前线程是否是锁的独占线程
         */
        protected final boolean isHeldExclusively() {
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        /**
         * 新建Condition对象
         */
        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        /**
         * 获取当前锁的拥有者
         */
        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        /**
         * 获取当前锁的加锁次数
         */
        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        /**
         * 获取当前锁是否被锁定
         */
        final boolean isLocked() {
            return getState() != 0;
        }

        /**
         * 反序列化
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }

        /**
         * 非公平锁
         */
        static final class NonfairSync extends Sync {

            /**
             * AQS的tryAcquire方法
             * 直接调用父类的nonfairTryAcquire方法，非公平方式直接获取锁
             */
            protected final boolean tryAcquire(int acquires) {
                return nonfairTryAcquire(acquires);
            }
        }

        /**
         * 公平锁
         */
        static final class FairSync extends Sync {

            /**
             * AQS的tryAcquire方法
             */
            @ReservedStackAccess
            protected final boolean tryAcquire(int acquires) {
                final Thread current = Thread.currentThread();

                // 获取当前锁状态
                int c = getState();
    
                // 锁空闲
                if (c == 0) {
                    // 当前线程之前没有其他线程在等待时才更改锁状态，公平锁的体现
                    if (!hasQueuedPredecessors() &&
                        compareAndSetState(0, acquires)) {
                        // 把当前线程设置为独占模式拥有者
                        setExclusiveOwnerThread(current);
                        return true;
                    }
                }
                else if (current == getExclusiveOwnerThread()) {
                    // 当前线程为独占模式拥有者，则修改锁状态(拥有锁的线程数)，体现了可重入
                    int nextc = c + acquires;
                    if (nextc < 0)
                        throw new Error("Maximum lock count exceeded");
                    setState(nextc);
                    return true;
                }
                return false;
            }
        }
        
    }

    /**
     * 默认无参构造函数使用非公平锁模式
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    /**
     * 可以选择公平锁或非公平锁的构造函数
     */
    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }

    /**
     * 获取锁，不响应中断
     * 直接调用AQS的acquire方法，acquire方法会先调用tryAcquire方法，加锁次数是1
     * tryAcquire方法不成功时当前线程就会进入队列等待，直到获取锁，这个时候就变成了公平锁
     */
    public void lock() {
        sync.acquire(1);
    }

    /**
     * 获取锁，如果当前线程被中断则抛出InterruptedException异常
     */
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    /**
     * 尝试获取锁，直接调用sync的非公平方式直接获取锁
     * 无论当前锁的实现是公平的或非公平的，tryLock都是非公平的
     */
    public boolean tryLock() {
        return sync.nonfairTryAcquire(1);
    }

    /**
     * 在单位时间内尝试获取锁，直接调用AQS的tryAcquireNanos方法
     * 如果被中断则抛出InterruptedException异常
     */
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

    /**
     * 释放锁
     */
    public void unlock() {
        sync.release(1);
    }

    /**
     * 新建Condition对象
     */
    public Condition newCondition() {
        return sync.newCondition();
    }

    /**
     * 获取当前锁的加锁次数
     */
    public int getHoldCount() {
        return sync.getHoldCount();
    }

    /**
     * 返回当前线程是否是当前锁的拥有者
     */
    public boolean isHeldByCurrentThread() {
        return sync.isHeldExclusively();
    }

    /**
     * 当前锁是否被加锁
     */
    public boolean isLocked() {
        return sync.isLocked();
    }

    /**
     * 当前锁是否是公平锁
     */
    public final boolean isFair() {
        return sync instanceof FairSync;
    }

    /**
     * 返回当前锁的拥有者
     */
    protected Thread getOwner() {
        return sync.getOwner();
    }

    /**
     * 返回是否有线程在排队获取当前锁
     */
    public final boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }

    /**
     * 查询指定线程是否在等待获取锁
     */
    public final boolean hasQueuedThread(Thread thread) {
        return sync.isQueued(thread);
    }

    /**
     * 返回等待获取锁的线程数
     */
    public final int getQueueLength() {
        return sync.getQueueLength();
    }

    /**
     * 返回等待获取锁的线程集合
     */
    protected Collection<Thread> getQueuedThreads() {
        return sync.getQueuedThreads();
    }

    /**
     * 指定条件下是否有线程在等待获取锁
     */
    public boolean hasWaiters(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.hasWaiters((AbstractQueuedSynchronizer.ConditionObject)condition);
    }

    /**
     * 指定条件下等待获取锁的线程数
     */
    public int getWaitQueueLength(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitQueueLength((AbstractQueuedSynchronizer.ConditionObject)condition);
    }

    /**
     * 指定条件下指定获取锁的线程集合
     */
    protected Collection<Thread> getWaitingThreads(Condition condition) {
        if (condition == null)
            throw new NullPointerException();
        if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
            throw new IllegalArgumentException("not owner");
        return sync.getWaitingThreads((AbstractQueuedSynchronizer.ConditionObject)condition);
    }
}
```
