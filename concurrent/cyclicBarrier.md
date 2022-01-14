## CyclicBarrier，循环障碍

CyclicBarrier 和 CountDownLatch 非常类似，它也可以实现线程间的技术等待，
但是它的功能比 CountDownLatch 更加复杂和强大。主要应用场景和 CountDownLatch 类似。

CountDownLatch 的实现是基于 AQS 的，而 CyclicBarrier 是基于 ReentrantLock 和 Condition 的。

CyclicBarrier 的字面意思是可循环使用（Cyclic）的屏障（Barrier）。
它要做的事情是：让一组线程到达一个屏障（也可以叫同步点）时被阻塞，
直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活。

CyclicBarrier 可以用于多线程计算数据，最后合并计算结果的应用场景。
比如我们用一个 Excel 保存了用户所有银行流水，每个 Sheet 保存一个帐户近一年的每笔银行流水，
现在需要统计用户的日均银行流水，先用多线程处理每个 sheet 里的银行流水，都执行完之后，得到每个 sheet 的日均银行流水，
最后，再用 barrierAction 用这些线程的计算结果，计算出整个 Excel 的日均银行流水。

### CyclicBarrier 用法

CyclicBarrier 默认的构造方法是 CyclicBarrier(int parties)，
其参数表示屏障拦截的线程数量，每个线程调用 await() 方法告诉 CyclicBarrier 我已经到达了屏障，然后当前线程被阻塞。

CyclicBarrier使用示例：
```java
public class CyclicBarrierExample2 {
  // 请求的数量
  private static final int threadCount = 550;
  // 需要同步的线程数量
  private static final CyclicBarrier cyclicBarrier = new CyclicBarrier(5);

  public static void main(String[] args) throws InterruptedException {
    // 创建线程池
    ExecutorService threadPool = Executors.newFixedThreadPool(10);

    for (int i = 0; i < threadCount; i++) {
      final int threadNum = i;
      Thread.sleep(1000);
      threadPool.execute(() -> {
        try {
          test(threadNum);
        } catch (InterruptedException e) {
          e.printStackTrace();
        } catch (BrokenBarrierException e) {
          e.printStackTrace();
        }
      });
    }
    threadPool.shutdown();
  }

  public static void test(int threadNum) throws InterruptedException, BrokenBarrierException {
    System.out.println("threadNum:" + threadNum + "is ready");
    try {
      /* 等待60秒，保证子线程完全执行结束 */
      cyclicBarrier.await(60, TimeUnit.SECONDS);
    } catch (Exception e) {
      System.out.println("-----CyclicBarrierException------");
    }
    System.out.println("threadNum:" + threadNum + "is finish");
  }

}
```

### CyclicBarrier 源码解析

```java
/**
 * 允许一组线程互相等待，直到全部线程到达一个共同的障碍点
 */
public class CyclicBarrier {

    /**
     * 每次使用障碍都代表了Generation的一个实例
     * 当障碍开启或重置的时候generation就会变更
     */
    private static class Generation {
        Generation() {}                 // prevent access constructor creation
        boolean broken;                 // initially false
    }

    // 用于保护障碍入口的锁
    private final ReentrantLock lock = new ReentrantLock();
    // 等待所有线程到达的条件
    private final Condition trip = lock.newCondition();
    // 需要等待的线程数
    private final int parties;
    // 所有线程都到达后需要执行的操作
    private final Runnable barrierCommand;
    // generation，表示当前使用的障碍，当前正在执行的循环等待
    private Generation generation = new Generation();

    /*
     * 还在等待的线程数，每当一个线程到达障碍点就会减一，直到为0，为0时则打开障碍
     * 每次打开障碍或被打断的时候就会重置为需要等待的线程数
     */
    private int count;

    /**
     * 开启一个新的循环障碍
     */
    private void nextGeneration() {
        // 唤醒所有在当前条件上等待的线程
        trip.signalAll();
        // 重置等待线程数
        count = parties;
        // 开启一个新的循环障碍
        generation = new Generation();
    }

    /**
     * 中断当前执行的障碍
     */
    private void breakBarrier() {
        // 当前障碍设置为中断
        generation.broken = true;
        // 重置等待线程数
        count = parties;
        // 唤醒所有在当前条件上等待的线程
        trip.signalAll();
    }

    /**
     * 主要的障碍代码，涵盖各种情况
     * 参数timed表示是否定时，也就是是否有超时限制
     */
    private int dowait(boolean timed, long nanos)
        throws InterruptedException, BrokenBarrierException,
               TimeoutException {

        // 获取当前循环障碍的可重入锁，可以看出循环障碍是依赖于可重入锁实现的
        final ReentrantLock lock = this.lock;
        lock.lock();

        try {
            final Generation g = generation;

            if (g.broken)
                throw new BrokenBarrierException();

            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            // 有线程到达，则把等待线程数减一
            int index = --count;
            // 等待线程数为0，说明全部到达，打开障碍点
            if (index == 0) {  // tripped

                // 执行预定义操作
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    if (command != null)
                        command.run();
                    ranAction = true;
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }

            /*
             * 自旋等待，如果线程全部到达、选择障碍被中断、线程被中断、超时，
             * 有以上四种情况之一的，就会退出等待
             */
            for (;;) {
                try {
                    // 使用可重入锁的condition来进行等待
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        // 等待指定时间
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    // 当前循环障碍中的线程被中断，会走到这个分支，抛出中断异常
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // 如果循环障碍被reset，reset方法中会重新开启一个generation，会走到这里，中断当前线程
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                if (g != generation)
                    return index;

                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            lock.unlock();
        }
    }

    /**
     * 创建一个循环障碍，当指定数量的线程到达障碍时打开障碍
     * 打开障碍时先做预定义的操作
     */
    public CyclicBarrier(int parties, Runnable barrierAction) {
        if (parties <= 0) throw new IllegalArgumentException();
        this.parties = parties;
        this.count = parties;
        this.barrierCommand = barrierAction;
    }
    
    /**
     * 创建一个循环障碍，当指定数量的线程到达障碍时打开障碍
     * 打开障碍时不做任何预定义的操作
     */
    public CyclicBarrier(int parties) {
        this(parties, null);
    }

    // 获取需要等待的线程数
    public int getParties() {
        return parties;
    }

    /**
     * 等待所有线程到达障碍点
     * 返回值表示还未到达的线程数
     */
    public int await() throws InterruptedException, BrokenBarrierException {
        try {
            return dowait(false, 0L);
        } catch (TimeoutException toe) {
            throw new Error(toe); // cannot happen
        }
    }

    /**
     * 等待所有线程到达障碍点，如果指定时间内未抵达，则抛出TimeoutException异常
     * 返回值表示还未到达的线程数
     */
    public int await(long timeout, TimeUnit unit)
        throws InterruptedException,
               BrokenBarrierException,
               TimeoutException {
        return dowait(true, unit.toNanos(timeout));
    }

    /**
     * 查询当前障碍是否是中断状态
     * 返回true表示：一个或多个线程被中断或超时，或预定义的操作抛出异常
     * false：正常状态
     */
    public boolean isBroken() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return generation.broken;
        } finally {
            lock.unlock();
        }
    }

    /**
     * 重置障碍点，这个时候在当前障碍点上等待的线程将会抛出BrokenBarrierException异常
     */
    public void reset() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            breakBarrier();   // break the current generation
            nextGeneration(); // start a new generation
        } finally {
            lock.unlock();
        }
    }

    /**
     * 获取当前等待的线程数
     */
    public int getNumberWaiting() {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            return parties - count;
        } finally {
            lock.unlock();
        }
    }
}
```