# RenntrantLock原理

![1702003036488](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/1702003036488.png)



`ReentrantLock` 默认使用非公平锁，也可以通过构造器来显式的指定使用公平锁 

```java
    public ReentrantLock() {
            sync = new NonfairSync(); //默认构造方法非公平锁
    }
    public ReentrantLock(boolean fair) { //通过true,false指定公平还是非公平锁
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

## 1. 非公平锁的实现原理

```java
static final class NonfairSync extends Sync {} 
abstract static class Sync extends AbstractQueuedSynchronizer{} 
//NonfairSync继承AQS
```

没有竞争时，加锁成功，直接将owner设置为当前线程，`state`设置为1

![1702003559168](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/1702003559168.png)

```java
final void lock() { // 加锁
    if (compareAndSetState(0, 1)) // CAS成功 state由0到1
        setExclusiveOwnerThread(Thread.currentThread());//owner设置为当前线程
    else
        acquire(1); //没有成功
}
```

出现竞争，线程0获得锁，还没释放，`state`为1

![1702003827236](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/1702003827236.png)

Thread-1 执行了 

1. `CAS` 尝试将 state 由 0 改为 1，结果失败 ，执行`acquire`方法

2. `acquire`方法内，执行 `tryAcquire `方法，这时 state 已经是1，结果仍然失败 

3. 失败后就会进入`addWaiter`方法，尝试创建一个节点对象，构造 Node 队列 

图中黄色三角表示该 Node 的`waitStatus` 状态，其中 0 为默认正常状态 

Node 的创建是懒惰的 

其中第一个 Node 称为 Dummy（哑元）或哨兵，用来占位，并不关联线程 

![1702005128143](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/1702005128143.png)

当前线程进入 `acquireQueued` 逻辑 

1. `acquireQueued` 会在一个死循环中不断尝试获得锁，失败后进入 park 阻塞 

2. 如果自己是紧邻着 head（排第二位），那么再次 `tryAcquire` 尝试获取锁，当然这时 `state` 仍为 1，失败 

3. 进入 `shouldParkAfterFailedAcquire` 逻辑，将前驱 node，即 head 的 `waitStatus` 改为 -1，这次返回 false 

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) //构造等待队列
        selfInterrupt();
}

final boolean acquireQueued(final Node node, int arg) { //加入等待队列
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) { //如果前驱节点是头节点，会尝试再次获取锁
                setHead(node); //如果当前线程获得了锁，会将当前节点设置为头节点，节点内的线程置为null, 然后将原来的头节点置为null，便于垃圾回收
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

![1702005537136](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/1702005537136.png)

4. `shouldParkAfterFailedAcquire `执行完毕回到 `acquireQueued` ，再次 `tryAcquire` 尝试获取锁，当然这时 state 仍为 1，失败 

5. 当再次进入 `shouldParkAfterFailedAcquire` 时，这时因为其前驱 node 的 `waitStatus` 已经是 -1，这次返回 true 

6. 进入 `parkAndCheckInterrupt`， Thread-1 park（灰色表示） 

再次有多个线程经历上述过程竞争失败，变成这个样子 

![1702005686841](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/1702005686841.png)

Thread-0 释放锁，进入 `tryRelease` 流程，如果成功 

+ 设置` exclusiveOwnerThread` 为 null 

+ state = 0 

```java
public final boolean release(int arg) { //释放锁
    if (tryRelease(arg)) {
        Node h = head;  
        if (h != null && h.waitStatus != 0) //头节点不为空，并且waitStatus不为0,唤醒后继节点
            unparkSuccessor(h); //找到最近的节点，调用unpark方法唤醒
        return true;
    }
    return false;
}
//exclusiveOwnerThread为 null,state为0
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

![1702006403971](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/1702006403971.png)

当前队列不为 null，并且 head 的 `waitStatus` = -1，进入 `unparkSuccessor` 流程 

找到队列中离 head 最近的一个 Node（没取消的），`unpark `恢复其运行，本例中即为 Thread-1 

回到 Thread-1 的 `acquireQueued` 流程

![1702006494083](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/1702006494083.png)

如果加锁成功（没有竞争），会设置 

+ `exclusiveOwnerThread` 为 Thread-1，state = 1 

+ head 指向刚刚 Thread-1 所在的 Node，该 Node 清空 Thread 

+ 原本的 head 因为从链表断开，而可被垃圾回收

如果这时候有其它线程来竞争（非公平的体现），例如这时有 Thread-4 来了 

![1702006878662](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/1702006878662.png)

如果不巧又被 Thread-4 占了先 

+ Thread-4 被设置为 `exclusiveOwnerThread`，`state` = 1 

+ Thread-1 再次进入 `acquireQueued` 流程，获取锁失败，重新进入 park 阻塞

## 2. 可重入原理

```java
 //尝试获得锁
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) { //发生锁重入
        int nextc = c + acquires; //等价于state++
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc); //每一次重入，state加，state的值为锁重入的次数
        return true;
    }
    return false;
}

//释放锁
protected final boolean tryRelease(int releases) { 
    int c = getState() - releases; //等价于state--
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) { //只有state为0时，free才为true,才表示释放锁成功
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

## 3. 可打断的原理



## 4. **公平锁实现原理** 

公平锁与非公平锁主要在于竞争锁时候的逻辑

```java
//公平锁竞争
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() && //公平锁在锁竞争时，先判断等待队列中是否有线程，再竞争
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
 //非公平锁竞争   
public final void acquire(int arg) {
    if (!tryAcquire(arg) && //非公平锁是不管队列是否有线程，直接竞争
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

# 读写锁

## 1.**ReentrantReadWriteLock** 

**案例**

```java
/**
 * 读写锁，读读不阻塞，读写阻塞，写写阻塞
 */
public class Demo1 {
    public static void main(String[] args) throws InterruptedException {
        DataContainer dataContainer = new DataContainer();
        new Thread(()->{
            dataContainer.read();
        }, "t1").start();

//        new Thread(()->{
//            dataContainer.read();
//        }, "t2").start();

        Thread.sleep(500);
        new Thread(()->{
            dataContainer.write();
        }, "t2").start();
    }
}

@Slf4j
class DataContainer {
    private Object data;
    ReentrantReadWriteLock rw = new ReentrantReadWriteLock();
    //读锁
    ReentrantReadWriteLock.ReadLock r = rw.readLock();
    //写锁
    ReentrantReadWriteLock.WriteLock w = rw.writeLock();
    //读操作
    public Object read() {
        log.debug("获取读锁");
        r.lock();
        try {
            log.debug("read...");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return data;
        }finally {
            log.debug("释放读锁");
            r.unlock();
        }
    }
    //写操作
    public void write() {
        log.debug("获取写锁...");
        w.lock();
        try {
            log.debug("写入");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        } finally {
            log.debug("释放写锁...");
            w.unlock();
        }
    }
}
```

**注意事项** 

+ 读锁不支持条件变量， 写锁支持条件变量

+ 重入时升级不支持：即持有读锁的情况下去获取写锁，会导致获取写锁永久等待

+ 重入时降级支持：即持有写锁的情况下去获取读锁

## 2. 读写锁原理



## 3.  **StampedLock** 

该类自 JDK 8 加入，是为了进一步优化读性能，它的特点是在使用读锁、写锁时都必须配合【戳】使用 

**加解读锁** 

```java
long stamp = lock.readLock();
lock.unlockRead(stamp);
```

**加解写锁** 

```java
long stamp = lock.writeLock();
lock.unlockWrite(stamp);
```

乐观读，`StampedLock` 支持 `tryOptimisticRead()` 方法（乐观读），读取完毕后需要做一次 **戳校验** 如果校验通过，表示这期间确实没有写操作，数据可以安全使用，如果校验没通过，需要重新获取读锁，保证数据安全。 

```java
long stamp = lock.tryOptimisticRead();
// 检验戳是否改变
if(!lock.validate(stamp)){
 // 戳改变了，锁升级
}
```

**案例**

```java
@Slf4j
class DataContainerStamped {
    private Object data;
    private final StampedLock lock = new StampedLock();
    public DataContainerStamped(int data) {
        this.data = data;
    }
    //读操作
    public Object read(int readTime) {
        //尝试乐观读
        long stamp = lock.tryOptimisticRead();
        log.debug("optimistic read locking...{}", stamp);
        try {
            Thread.sleep(readTime);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        //检验戳是否改变
        if (lock.validate(stamp)) { //没有改变，直接返回读到的数据
            log.debug("read finish...{}, data:{}", stamp, data);
            return data;
        }
        //改变, 进行锁升级-读锁
        log.debug("updating to read lock... {}", stamp);
        try {
            //加读锁
            stamp = lock.readLock();
            try {
                Thread.sleep(readTime);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.debug("read finish...{}, data:{}", stamp, data);
            return data;
        } finally {
            //释放读锁
            lock.unlockRead(stamp);
        }

    }

    //写操作
    public void write(Object data) {
        //加写锁
        long stamp = lock.writeLock();
        log.debug("write lock {}", stamp);
        try {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            this.data = data;
        } finally {
            log.debug("write unlock {}", stamp);
            //释放写锁
            lock.unlockWrite(stamp);
        }
    }
}
```

读与读的测试

```java
public class Demo1 {
    public static void main(String[] args) {
        DataContainerStamped dataContainer = new DataContainerStamped(1);
        //读与读测试
        new Thread(() -> {
            dataContainer.read(1000);
        }, "t1").start();
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(() -> {
            dataContainer.read(0);
        }, "t2").start();
    }
}
```

输出结果

```java
16:41:25.679 [t1] DEBUG com.cjj.stampedlock.DataContainerStamped - optimistic read locking...256
16:41:26.186 [t2] DEBUG com.cjj.stampedlock.DataContainerStamped - optimistic read locking...256
16:41:26.186 [t2] DEBUG com.cjj.stampedlock.DataContainerStamped - read finish...256, data:1
16:41:26.690 [t1] DEBUG com.cjj.stampedlock.DataContainerStamped - read finish...256, data:1
```

读与写测试

```java
public class Demo1 {
    public static void main(String[] args) {
        DataContainerStamped dataContainer = new DataContainerStamped(1);
        //读与写测试
        new Thread(() -> {
            dataContainer.read(1000);
        }, "t1").start();
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(() -> {
            dataContainer.write(100);
        }, "t2").start();
    }
}
```

输出结果

```java
16:42:22.302 [t1] DEBUG com.cjj.stampedlock.DataContainerStamped - optimistic read locking...256
16:42:22.791 [t2] DEBUG com.cjj.stampedlock.DataContainerStamped - write lock 384
16:42:23.310 [t1] DEBUG com.cjj.stampedlock.DataContainerStamped - updating to read lock... 256
16:42:24.803 [t2] DEBUG com.cjj.stampedlock.DataContainerStamped - write unlock 384
16:42:25.811 [t1] DEBUG com.cjj.stampedlock.DataContainerStamped - read finish...513, data:100
```

**注意事项**

+ `StampedLock` 不支持条件变量 

+ `StampedLock` 不支持可重入

# Semaphare

信号量，用来限制能同时访问共享资源的线程上限。

## 1. 基本使用

```java
/**
 * 限制同时访问共享资源的线程上限个数
 */
@Slf4j
public class Demo1 {
    public static void main(String[] args) {
        // 1. 创建 semaphore 对象
        Semaphore semaphore = new Semaphore(3);
        // 2. 10个线程同时运行
        for (int i = 0; i < 10; i++) {
            new Thread(() -> {
                // 3. 获取许可
                try {
                    semaphore.acquire();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                try {
                    log.debug("running...");
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    log.debug("end...");
                } finally {
                    // 4. 释放许可
                    semaphore.release();
                }
            }).start();
        }
    }
}
```

## 2. Semaphare应用(改写连接池)



## 3. Semaphare原理



# CountdownLatch

用来进行线程同步协作，等待所有线程完成倒计时

其中构造参数用来初始化等待计数值，await() 用来等待计数归零，countDown() 用来让计数减一

**用法1**

```java
/**
 * 用来进行线程同步协作，等待所有线程完成倒计时。
 * 构造参数用来初始化等待计数值，await() 用来等待计数归零，countDown() 用来让计数减一
 */
@Slf4j
public class Demo2 {
    public static void main(String[] args) {
        CountDownLatch latch = new CountDownLatch(3);
        ExecutorService es = Executors.newFixedThreadPool(4);
        es.execute(()->{
            log.debug("begin...");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            latch.countDown();
            log.debug("end...{}", latch.getCount());
        });
        es.execute(()->{
            log.debug("begin...");
            try {
                Thread.sleep(1500);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            latch.countDown();
            log.debug("end...{}", latch.getCount());
        });
        es.execute(()->{
            log.debug("begin...");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            latch.countDown();
            log.debug("end...{}", latch.getCount());
        });
        es.execute(()->{
            try {
                log.debug("waiting...");
                //等待所有线程完成
                latch.await();
                log.debug("wait end...");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        es.shutdown();
    }
}
```

**用法2**：**同步等待多线程准备完毕**

```java
/**
 * 等待所有线程都加载100%再游戏开始
 */
public class Demo3 {
    public static void main(String[] args) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(10);
        ExecutorService es = Executors.newFixedThreadPool(10);
        String[] arr = new String[10];
        Random random = new Random();
        for (int j = 0; j < 10; j++) {
            int k = j;
            es.submit(()->{
                for (int i = 0; i <= 100; i++) {
                    arr[k] = i + "%";
                    try {
                        Thread.sleep(random.nextInt(100));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    System.out.print("\r" + Arrays.toString(arr)); // 小技巧，每次输出行首，覆盖之前的
                }
                latch.countDown();
            });
        }
        latch.await();
        System.out.println("\n游戏开始");
        es.shutdown();
    }
}
```

**用法3**：多线程进行多个远程调用，等待多个远程调用结束。如果需要远程调用得到的返回值结果可以使用`Future`

# CyclicBarrier

用来进行线程协作，等待线程满足某个计数。构造时设置『计数个数』，每个线程执行到某个需要“同步”的时刻调用 await() 方法进行等待，当等待的线程数满足『计数个数』时，继续执行。

```java
/**
 * 每个线程执行到某个需要“同步”的时刻调用 await() 方法进行等待，
 * 当等待的线程数满足『计数个数』时，继续执行
 */
@Slf4j
public class Demo1 {
    public static void main(String[] args) {
        CyclicBarrier cb = new CyclicBarrier(2);
        new Thread(()->{
            log.debug("线程1开始..");
            try {
                cb.await(); // 当个数不足时，等待
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
            System.out.println("线程1继续向下运行...");
        },"t1").start();
        new Thread(()->{
            log.debug("线程2开始..");
            try {
                cb.await(); // 当个数不足时，等待
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
            System.out.println("线程2继续向下运行...");
        },"t2").start();
    }
}
```

