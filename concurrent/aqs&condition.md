## AQS之条件队列

在[锁机制、CAS、AQS](/concurrent/cas&aqs.md)中分析了AQS的独占模式和共享模式，他们都是在AQS的同步队列里。
AQS中还有一个等待队列：Condition。

### AQS Condition 用法：
```java
public class UseCondition {
    
    // 新建一个可重入锁
    private Lock lock = new ReentrantLock();
    // 在可重入锁上加上condition
    private Condition condition = lock.newCondition();

    // t1线程调用这个方法
    public void method1() {
        try {
            // 获取锁
            lock.lock();
            System.out.println("当前线程: " + Thread.currentThread().getName() + "进入等待状态");
            TimeUnit.SECONDS.sleep(3);
            System.out.println("当前线程: " + Thread.currentThread().getName() + "释放锁");
            // 这里模拟继续执行的条件不满足，调用condition.await()进入等待并释放锁
            condition.await();// Object wait
            System.out.println("当前线程: " + Thread.currentThread().getName() + "继续执行");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
    
    // t2线程调用这个方法
    public void method2() {
        try {
            // 获取锁
            lock.lock();
            System.out.println("当前线程: " + Thread.currentThread().getName() + "进入....");
            TimeUnit.SECONDS.sleep(3);
            System.out.println("当前线程: " + Thread.currentThread().getName() + "发出唤醒..");
            // 这里模拟t2线程修改了条件，使t1线程满足了继续执行的条件，调用condition.signal()唤醒t1线程并使t1线程继续执行
            condition.signal();//Object notify
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        // 启动两个线程t1和t2

        UseCondition uc = new UseCondition();
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                uc.method1();
            }
        }, "t1");

        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                uc.method2();
            }
        }, "t2");

        t1.start();
        t2.start();
    }
}
```

### Condition 源码分析

Condition接口：
```java
public interface Condition {

    /**
     * 使当前线程进入等待状态，直到被signal方法唤醒或被中断
     */
    void await() throws InterruptedException;

    /**
     * 使当前线程进入等待状态，直到被signal方法唤醒就，不响应中断
     */
    void awaitUninterruptibly();

    /**
     * 使当前线程等待指定时间，被signal方法唤醒或被中断时取消等待
     * 返回值为正数表示当前线程还需等待的时间
     * 返回值为负数表示当前线程已没有在等待了
     */
    long awaitNanos(long nanosTimeout) throws InterruptedException;

    /**
     * 使当前线程等待指定时间，被signal方法唤醒或被中断时取消等待
     * 返回值为true表示当前现在还在等待
     * 返回值为false表示当前线程已不在等待
     */
    boolean await(long time, TimeUnit unit) throws InterruptedException;

    /**
     * 使当前线程等待指定时间，被signal方法唤醒或被中断时取消等待
     * 返回值为true表示当前现在还在等待
     * 返回值为false表示当前线程已不在等待
     */
    boolean awaitUntil(Date deadline) throws InterruptedException;

    /**
     * 唤醒一个等待的线程
     */
    void signal();

    /**
     * 唤醒所有等待的线程
     */
    void signalAll();
}
```

AQS中的Condition：
```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {

    /**
     * AQS队列节点，这里只列出了Condition用到的属性
     */
    static final class Node {

        /*
         * 指向条件队列中的下一个节点，或者特殊值SHARED
         * 因为条件队列只有在独占模式下才能访问，所以只是简单的用一个链接队列来表示条件队列上的节点
         * 条件队列上的节点之后会被转移到同步队列上去重新获取锁
         * 因为条件队列只能在独占模式下访问，所以可以用一个特殊值来表示共享模式
         */
        Node nextWaiter;

        Node(Node nextWaiter) {
            this.nextWaiter = nextWaiter;
            THREAD.set(this, Thread.currentThread());
        }
    }

    /**
     * 使用传入节点的state去释放锁，如果释放失败则将当前节点置为已取消并抛出异常
     */
    final int fullyRelease(Node node) {
        try {
            int savedState = getState();
            if (release(savedState))
                return savedState;
            throw new IllegalMonitorStateException();
        } catch (Throwable t) {
            node.waitStatus = Node.CANCELLED;
            throw t;
        }
    }

    /**
     * 节点取消之后，将节点从条件队列转换到同步队列
     * 在唤醒之前取消的则返回true
     */
    final boolean transferAfterCancelledWait(Node node) {
        // 如果CAS可以成功就说明没有被唤醒过，因为唤醒的时候节点状态被置为了0
        if (node.compareAndSetWaitStatus(Node.CONDITION, 0)) {
            // 加入同步队列
            enq(node);
            return true;
        }

        // 如果上面设置失败，说明节点已经被signal()方法唤醒，由于signal()方法会将节点加入到同步队列，这里只需要自旋等待即可！
        while (!isOnSyncQueue(node))
            Thread.yield();
        return false;
    }

    /**
     * 从条件队列转换一个节点到同步队列
     */
    final boolean transferForSignal(Node node) {
        // 唤醒一个线程，这里将节点状态置为了0，如果节点被取消了，这里的CAS会失败，返回false
        if (!node.compareAndSetWaitStatus(Node.CONDITION, 0))
            return false;
    
        // 将该结点添加到同步队列尾部,这里返回的p节点是当前节点的前置节点
        Node p = enq(node);

        /*
         * 如果前置节点被取消或者修改状态失败则直接唤醒当前节点
         * 此时当前节点已经处于同步队列中，唤醒会进行获取锁操作（设置节点状态为SIGNAL）或者挂起等待
         */
        int ws = p.waitStatus;
        if (ws > 0 || !p.compareAndSetWaitStatus(ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
    
    public class ConditionObject implements Condition, java.io.Serializable {
        /** 条件队列首节点 */
        private transient Node firstWaiter;
        /** 条件队列尾节点 */
        private transient Node lastWaiter;
    
        public ConditionObject() { }

        /**
         * 添加一个节点到条件队列
         */
        private Node addConditionWaiter() {
            // 当前线程不是锁的拥有者，直接抛出异常
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();

            Node t = lastWaiter;
            // 如果尾节点是已取消，则删除所有已取消的节点
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }

            // 新建一个条件节点，并添加到条件队列尾部
            Node node = new Node(Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }

        /**
         * 删除所有已取消节点
         * 这里算法逻辑没看懂，先贴在这里
         */
        private void unlinkCancelledWaiters() {
            Node t = firstWaiter;
            Node trail = null;
            while (t != null) {
                Node next = t.nextWaiter;
                if (t.waitStatus != Node.CONDITION) {
                    t.nextWaiter = null;
                    if (trail == null)
                        firstWaiter = next;
                    else
                        trail.nextWaiter = next;
                    if (next == null)
                        lastWaiter = trail;
                }
                else
                    trail = t;
                t = next;
            }
        }

        /**
         * 验证是否被中断过
         * 在唤醒之前被中断过则返回 THROW_IE: -1
         * 在唤醒之后被中断过则返回 REINTERRUPT: 1
         * 没有被中断过则返回 0
         */
        private int checkInterruptWhileWaiting(Node node) {
            return Thread.interrupted() ?
                (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
                0;
        }

        /**
         * 实现支持中断的条件等待
         */
        public final void await() throws InterruptedException {
            // 如果当前线程被中断，直接抛出异常
            if (Thread.interrupted())
                throw new InterruptedException();

            // 添加一个节点到条件队列
            Node node = addConditionWaiter();

            // 释放当前线程的锁，如果释放失败则将当前节点置为已取消并抛出异常
            int savedState = fullyRelease(node);

            // 线程是否被中断过
            int interruptMode = 0;
            // 如果当前线程不在同步队列中
            while (!isOnSyncQueue(node)) {
                // 挂起当前线程
                LockSupport.park(this);
                // 线程挂起后被唤醒，从这里开始执行。发现被中断过，退出循环
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            
            // 更新中断状态
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            // 因为重新唤醒，所以要重新删除条件队列中已取消的节点
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            // 被中断过，判断是抛出异常还是自我中断
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

        /**
         * 唤醒一个线程
         */
        public final void signal() {
            // 当前线程不是锁的拥有者，直接抛出异常
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();

            // 有其他线程在等待，唤醒第一个等待线程
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }

        // 唤醒等待线程
        private void doSignal(Node first) {
            do {
                // 将firstWaiter指针向后移动一位，指向first的下一个节点，如果等于null的话说明队列中只有一个节点first;
                if ( (firstWaiter = first.nextWaiter) == null)
                    // 因为条件队列中没有节点了所以lastWaiter也制为空
                    lastWaiter = null;
                // 将头结点从等待队列中移除
                first.nextWaiter = null;

                // 将节点加入到同步队列，如果transferForSignal操作失败就去唤醒下一个结点
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }

    }
}
```