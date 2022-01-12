## Semaphore，信号量

synchronized 和 ReentrantLock 都是一次只允许一个线程访问某个资源，Semaphore(信号量)可以指定多个线程同时访问某个资源。

Semaphore 与 CountDownLatch 一样，也是共享锁的一种实现。它默认构造 AQS 的 state 为 permits。
当执行任务的线程数量超出 permits，那么多余的线程将会被放入阻塞队列，并自旋判断 state 是否大于 0。
只有当 state 大于 0 的时候，阻塞的线程才能继续执行，此时先前执行任务的线程继续执行 release() 方法，
release() 方法使得 state 的变量会加 1，那么自旋的线程便会判断成功。
如此，每次只有最多不超过 permits 数量的线程能自旋成功，便限制了执行任务线程的数量。
Semaphore 经常用于限制获取某种资源的线程数量。

### Semaphore 用法

Semaphore使用示例：
```java
public class SemaphoreExample1 {
  // 请求的数量
  private static final int threadCount = 550;

  public static void main(String[] args) throws InterruptedException {
    // 创建一个具有固定线程数量的线程池对象（如果这里线程池的线程数量给太少的话你会发现执行的很慢）
    ExecutorService threadPool = Executors.newFixedThreadPool(300);
    // 一次只能允许执行的线程数量。
    final Semaphore semaphore = new Semaphore(20);

    for (int i = 0; i < threadCount; i++) {
      final int threadNum = i;
      threadPool.execute(() -> {// Lambda 表达式的运用
        try {
          semaphore.acquire();// 获取一个许可
          test(threadNum);
          semaphore.release();// 释放一个许可
        } catch (InterruptedException e) {
          e.printStackTrace();
        }

      });
    }

    threadPool.shutdown();
    System.out.println("finish");
  }

  public static void test(int threadNum) throws InterruptedException {
    Thread.sleep(1000);// 模拟请求的耗时操作
    System.out.println("threadNum:" + threadNum);
    Thread.sleep(1000);// 模拟请求的耗时操作
  }
}
```

### Semaphore 源码解析
```java
/**
 * 一个计数的信号，从概念上来说一个信号保持了一组许可证，
 * acquire方法用来获取许可证，如果获取不到就阻塞，一直等到可以获取许可证
 * release方法用来增加许可证，如果有阻塞的线程等待获取许可证，那么就会唤醒一个阻塞的线程通过acquire方法去获取许可证
 * 实际上并没有真正的许可证，Semaphore只是维护了一个数字来表示有效的许可证数量
 */
public class Semaphore implements java.io.Serializable {
    private final Sync sync;

    /**
     * Semaphore的同步实现，使用AQS的state来表示许可证数量
     * 提供公平锁和非公平锁两种模式
     */
    abstract static class Sync extends AbstractQueuedSynchronizer {

        /**
         * 构造函数直接调用AQS的setState方法，使用AQS的state来表示许可证数量
         */
        Sync(int permits) {
            setState(permits);
        }

        /**
         * 调用AQS的getState方法来获取许可证数量
         */
        final int getPermits() {
            return getState();
        }

        /**
         * 非公平模式获取锁，也就是直接获取许可证
         */
        final int nonfairTryAcquireShared(int acquires) {
            // 自旋
            for (;;) {
                // 获取state，也就是有效的许可证数量
                int available = getState();
                // 计算剩余的有效许可证
                int remaining = available - acquires;
                // 获取锁失败或修改锁状态成功，也就是获取许可证失败或获取到许可证并设置剩余许可证数量成功
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    // 返回剩余许可证数量
                    return remaining;
            }
        }

        /**
         * 尝试共享模式释放锁，这里会增加许可证数量
         */
        protected final boolean tryReleaseShared(int releases) {
            for (;;) {
                int current = getState();
                int next = current + releases;
                if (next < current) // overflow
                    throw new Error("Maximum permit count exceeded");
                if (compareAndSetState(current, next))
                    return true;
            }
        }

        /**
         * 减少指定数量的许可证，和acquire的区别是不会被挂起等待
         */
        final void reducePermits(int reductions) {
            for (;;) {
                int current = getState();
                int next = current - reductions;
                if (next > current) // underflow
                    throw new Error("Permit count underflow");
                if (compareAndSetState(current, next))
                    return;
            }
        }

        /**
         * 返回有效许可证数量，并将有效许可证数量设置为0
         */
        final int drainPermits() {
            for (;;) {
                int current = getState();
                if (current == 0 || compareAndSetState(current, 0))
                    return current;
            }
        }
        
        /**
         * 非公平锁模式
         */
        static final class NonfairSync extends Sync {

            /**
             * 调用父类的初始化方法
             */
            NonfairSync(int permits) {
                super(permits);
            }

            /**
             * 尝试获取许可证，会在acquire方法中被调用
             * 如果返回负数那么就表明获取失败，线程就会一直等待(自旋或挂起)，直到获取成功
             */
            protected int tryAcquireShared(int acquires) {
                // 调用父类的nonfairTryAcquireShared方法
                return nonfairTryAcquireShared(acquires);
            }
        }

        /**
         * 公平锁模式
         */
        static final class FairSync extends Sync {

            /**
             * 调用父类的初始化方法
             */
            FairSync(int permits) {
                super(permits);
            }

            /**
             * 尝试获取许可证，会在acquire方法中被调用
             * 如果返回负数那么就表明获取失败，线程就会一直等待(自旋或挂起)，直到获取成功
             */
            protected int tryAcquireShared(int acquires) {
                for (;;) {
                    // 如果前边有线程在等待，直接返回获取失败，公平锁的提现
                    if (hasQueuedPredecessors())
                        return -1;
                    int available = getState();
                    int remaining = available - acquires;
                    if (remaining < 0 ||
                        compareAndSetState(available, remaining))
                        return remaining;
                }
            }
        }
    }

    /**
     * 构造函数，传入许可证数量，默认为非公平锁模式
     */
    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }

    /**
     * 构造函数，指定使用公平锁还是非公平锁
     */
    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }

    /**
     * 获取许可证，直接调用AQS的共享模式获取锁并相应中断方法，可以看出默认的获取许可证方法是响应中断的
     * 传入的数字是1，也就是获取一个许可证，会使有效许可证数量减一，这里会调用tryAcquireShared方法
     */
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    /**
     * 获取许可证，会使有效许可证数量减一，不响应中断
     */
    public void acquireUninterruptibly() {
        sync.acquireShared(1);
    }
    
    /**
     * 尝试获取许可证，会使有效许可证数量减一，
     * 无论声明Semaphore的时候是公平的还是非公平的，这里都是非公平模式
     */
    public boolean tryAcquire() {
        return sync.nonfairTryAcquireShared(1) >= 0;
    }

    /**
     * 尝试获取许可证，会使有效许可证数量减一，设置超时并响应中断
     * 这里会调用tryAcquireShared方法
     */
    public boolean tryAcquire(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    /**
     * 释放许可证，会让有效许可证数量加一
     * 这里会调用tryReleaseShared方法
     */
    public void release() {
        sync.releaseShared(1);
    }

    /**
     * 获取指定数量的许可证，响应中断
     */
    public void acquire(int permits) throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireSharedInterruptibly(permits);
    }

    /**
     * 获取指定数量的许可证，不响应中断
     */
    public void acquireUninterruptibly(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.acquireShared(permits);
    }

    /**
     * 尝试获取指定数量的许可证
     */
    public boolean tryAcquire(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        return sync.nonfairTryAcquireShared(permits) >= 0;
    }

    /**
     * 尝试获取指定数量的许可证，加入超时并响应中断
     */
    public boolean tryAcquire(int permits, long timeout, TimeUnit unit)
        throws InterruptedException {
        if (permits < 0) throw new IllegalArgumentException();
        return sync.tryAcquireSharedNanos(permits, unit.toNanos(timeout));
    }

    /**
     * 释放指定数量的许可证
     */
    public void release(int permits) {
        if (permits < 0) throw new IllegalArgumentException();
        sync.releaseShared(permits);
    }

    /**
     * 获取有效许可证数量
     */
    public int availablePermits() {
        return sync.getPermits();
    }

    /**
     * 返回有效许可证数量，并将有效许可证数量重置为0
     */
    public int drainPermits() {
        return sync.drainPermits();
    }

    /**
     * 减少指定数量的许可证，和acquire的区别是不会被挂起等待
     */
    protected void reducePermits(int reduction) {
        if (reduction < 0) throw new IllegalArgumentException();
        sync.reducePermits(reduction);
    }

    /**
     * 是否是公平锁
     */
    public boolean isFair() {
        return sync instanceof FairSync;
    }

    /**
     * 是否有线程在排队获取许可证
     */
    public final boolean hasQueuedThreads() {
        return sync.hasQueuedThreads();
    }

    /**
     * 等待获取许可证的线程数
     */
    public final int getQueueLength() {
        return sync.getQueueLength();
    }

    /**
     * 等待获取许可证的线程集合
     */
    protected Collection<Thread> getQueuedThreads() {
        return sync.getQueuedThreads();
    }
}
```