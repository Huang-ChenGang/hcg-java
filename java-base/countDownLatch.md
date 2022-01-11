## CountDownLatch，倒计时器

CountDownLatch 允许 count 个线程阻塞在一个地方，直至所有线程的任务都执行完毕。

CountDownLatch 是共享锁的一种实现，它默认构造 AQS 的 state 值为 count。
当线程使用 countDown() 方法时，其实使用了tryReleaseShared方法以 CAS 的操作来减少 state，
直至 state 为 0 。当调用 await() 方法的时候，如果 state 不为 0，
那就证明任务还没有执行完毕，await() 方法就会一直阻塞，也就是说 await() 方法之后的语句不会被执行。
然后，CountDownLatch 会自旋 CAS 判断 state == 0，如果 state == 0 的话，就会释放所有等待的线程，
await() 方法之后的语句得到执行。

### CountDownLatch 用法

将 CountDownLatch 的计数器初始化为 n （new CountDownLatch(n)），
每当一个任务线程执行完毕，就将计数器减 1 （countdownlatch.countDown()），
当计数器的值变为 0 时，在 CountDownLatch 上 await() 的线程就会被唤醒。
一个典型应用场景就是启动一个服务时，主线程需要等待多个组件加载完毕，之后再继续执行。

CountDownLatch使用示例：
```java
import java.util.Arrays;
import java.util.List;
import java.util.Random;
import java.util.concurrent.CountDownLatch;

public class CountDownLatchTest {
    private final static Random random = new Random();

    static class SearchTask implements Runnable {
        private final Integer id;
        private final CountDownLatch latch;

        SearchTask(Integer id, CountDownLatch latch) {
            this.id = id;
            this.latch = latch;
        }

        @Override
        public void run() {
            System.out.println("开始寻找" + id + "号龙珠");
            int seconds = random.nextInt(10);
            try {
                Thread.sleep(seconds * 1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("花了" + seconds + "秒找到了" + id + "号龙珠");

            /*
             * 子线程通过调用countDown方法通知latch任务已完成。
             * 每调用一次这个方法，latch的 count 值就减 1。
             * 当latch的count为0时，主线程就能通过 await()方法，恢复执行自己的任务。
             */
            latch.countDown();
        }
    }

    public static void main(String[] args) {
        List<Integer> idList = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
        CountDownLatch latch = new CountDownLatch(idList.size());

        for (Integer id : idList) {
            new Thread(new SearchTask(id, latch)).start();
        }

        try {
            /*
             * 主线程必须在启动其他线程后立即调用 CountDownLatch.await() 方法。
             * 这样主线程的操作就会在这个方法上阻塞，直到其他线程完成各自的任务。
             */
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("七龙珠已集齐，开始召唤神龙");
    }
}
```

### CountDownLatch 源码解析

```java
/**
 * 一种同步辅助，允许一个或多个线程等待，直到在其他线程中执行的一组操作完成。
 */
public class CountDownLatch {
    
    /**
     * CountDownLatch的同步控制，使用AQS的state代表count
     */
    private static final class Sync extends AbstractQueuedSynchronizer {

        /*
         * 构造函数初始化，直接调用AQS的setState方法，把AQS的state设置为传进来的count
         */
        Sync(int count) {
            setState(count);
        }

        /*
         * 获取还未执行完的线程数，直接调用AQS的getState方法获取
         */
        int getCount() {
            return getState();
        }

        /*
         * 重写AQS的尝试以共享模式获取锁的方法，会在countDownLatch的await方法中被调用到
         * 这里返回1表示有一个后续线程需要被唤醒，也就是要唤醒主线程 
         */
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        /**
         * 重写AQS的尝试以共享模式释放锁的方法，会在countDownLatch的countDown方法中被调用到
         */
        protected boolean tryReleaseShared(int releases) {
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c - 1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }

    private final Sync sync;

    /**
     * 初始化，直接调用Sync的初始化方法
     */
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }

    /**
     * countDownLatch的await方法，
     * 直接调用AQS的以共享模式获取锁，如果被中断则抛出InterruptedException异常
     * 这里会调用tryAcquireShared方法
     */
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    /**
     * await设置超时并相应中断，直接调用AQS的tryAcquireSharedNanos方法
     * 这里会调用tryAcquireShared方法
     */
    public boolean await(long timeout, TimeUnit unit)
        throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    /**
     * countDownLatch的countDown方法，
     * 直接调用AQS的释放共享锁方法，让state减一，这里会调用tryReleaseShared方法
     */
    public void countDown() {
        sync.releaseShared(1);
    }

    /**
     * 获取当前count值
     */
    public long getCount() {
        return sync.getCount();
    }
}
```