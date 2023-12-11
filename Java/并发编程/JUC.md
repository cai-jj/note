# `ThreadLocal`

创建了一个`ThreadLocal`变量，那么访问这个变量的每个线程都会有这个变量的本地副本。当多线程访问的时候，操作的是自己本地内存的变量。可以使用 `get()` 和 `set()` 方法来获取默认值或将其值更改为当前线程所存的副本的值，从而避免了线程安全问题。

## 1. `ThreadLocal`用法

**`ThreadLocal`常用方法**

![image-20231211170241672](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/image-20231211170241672.png)

```java
/**
 * ThreadLocal用法
 */
public class Demo1 {
    private ThreadLocal<String> t = new ThreadLocal<>();
    private String content;
    private String getContent() {
//        return content;
        return t.get();
    }
    private void setContent(String content) {
//        this.content = content;
        t.set(content);
    }
    public static void main(String[] args) {
        Demo1 demo = new Demo1();
        for (int i = 0; i < 5; i++) {
            Thread thread = new Thread(() -> {
                demo.setContent(Thread.currentThread().getName() + "的数据");
                System.out.println(Thread.currentThread().getName() + "--->" + demo.getContent());
            });
            thread.setName("线程" + i);
            thread.start();
        }
    }
}
```

没有使用`ThreadLocal`打印结果

```java
线程4--->线程4的数据
线程3--->线程4的数据
线程1--->线程4的数据
线程2--->线程4的数据
线程0--->线程4的数据
```

使用了`ThreadLocal`打印结果

```java
线程0--->线程0的数据
线程1--->线程1的数据
线程4--->线程4的数据
线程2--->线程2的数据
线程3--->线程3的数据
```

使用`synchronized`关键字也可以解决，,但是使用`ThreadLocal`更为合适,因为这样可以使程序拥有更高的并发性。



## 2.`ThreadLocal`与`synchronized`的区别

 虽然`ThreadLocal`模式与`synchronized`关键字都用于处理多线程并发访问变量的问题, 不过两者处理问题的角度和思路不同。

![image-20231211171336966](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/image-20231211171336966.png)


## 3. `ThreadLocal`原理

如果我们不去看源代码的话，可能会猜测`ThreadLocal`是这样子设计的：每个`ThreadLocal`都创建一个`Map`，然后用线程作为`Map`的`key`，要存储的局部变量作为`Map`的`value`，这样就能达到各个线程的局部变量隔离的效果。这是最简单的设计方法，`JDK`最早期的`ThreadLocal `确实是这样设计的，但现在早已不是了。

![image-20231211171744356](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/image-20231211171744356.png)

### 3.1 `ThreadLocal`的设计

但是，JDK后面优化了设计方案，在JDK8中` ThreadLocal`的设计是：每个Thread维护一个`ThreadLocalMap`，这个Map的key是`ThreadLocal`实例本身，value才是真正要存储的值Object。

具体的过程是这样的：

+ 每个Thread线程内部都有一个`Map` (`ThreadLocalMap`静态内部类)

+ Map里面存储`ThreadLocal`对象（key）和线程的变量副本（value）

+ Thread内部的Map是由`ThreadLocal`维护的，由`ThreadLocal`负责向map获取和设置线程的变量值。

+ 对于不同的线程，每次获取副本值时，别的线程并不能获取到当前线程的副本值，形成了副本的隔离，互不干扰。

![image-20231211172222779](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/image-20231211172222779.png)

**这样设计的好处：**

+ 这样设计之后每个Map存储的Entry数量就会变少。因为之前的存储数量由Thread的数量决定，现在是由`ThreadLocal`的数量决定。在实际运用当中，往往`ThreadLocal`的数量要少于Thread的数量。
+ 当Thread销毁之后，对应的`ThreadLocalMap`也会随之销毁，能减少内存的使用

### 3.2 `ThreadLocal`源码

除了构造方法之外， `ThreadLocal`对外暴露的方法有以下4个

![image-20231211173111884](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/image-20231211173111884.png)

**set方法**

```java
/**
     * 设置当前线程对应的ThreadLocal的值
     *
     * @param value 将要保存在当前线程对应的ThreadLocal的值
     */
    public void set(T value) {
        // 获取当前线程对象
        Thread t = Thread.currentThread();
        // 获取此线程对象中维护的ThreadLocalMap对象
        ThreadLocalMap map = getMap(t);
        // 判断map是否存在
        if (map != null)
            // 存在则调用map.set设置此实体entry
            map.set(this, value);
        else
            // 1）当前线程Thread 不存在ThreadLocalMap对象
            // 2）则调用createMap进行ThreadLocalMap对象的初始化
            // 3）并将 t(当前线程)和value(t对应的值)作为第一个entry存放至ThreadLocalMap中
            createMap(t, value);
    }

 /**
     * 获取当前线程Thread对应维护的ThreadLocalMap 
     * 
     * @param  t the current thread 当前线程
     * @return the map 对应维护的ThreadLocalMap 
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
	/**
     *创建当前线程Thread对应维护的ThreadLocalMap 
     *
     * @param t 当前线程
     * @param firstValue 存放到map中第一个entry的值
     */
	void createMap(Thread t, T firstValue) {
        //这里的this是调用此方法的threadLocal
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```

**set执行流程** 

+ 首先获取当前线程，并根据当前线程获取一个Map

+ 如果获取的Map不为空，则调用`ThreadLocalMap`方法将参数设置到Map中（当前`ThreadLocal`的引用作为key）

+ 如果Map为空，则给该线程创建 Map，并设置初始值

**get方法**

```java
 /**
     * 返回当前线程中保存ThreadLocal的值
     * 如果当前线程没有此ThreadLocal变量，
     * 则它会通过调用{@link #initialValue} 方法进行初始化值
     *
     * @return 返回当前线程对应此ThreadLocal的值
     */
    public T get() {
        // 获取当前线程对象
        Thread t = Thread.currentThread();
        // 获取此线程对象中维护的ThreadLocalMap对象
        ThreadLocalMap map = getMap(t);
        // 如果此map存在
        if (map != null) {
            // 以当前的ThreadLocal 为 key，调用getEntry获取对应的存储实体e
            ThreadLocalMap.Entry e = map.getEntry(this);
            // 对e进行判空 
            if (e != null) {
                @SuppressWarnings("unchecked")
                // 获取存储实体 e 对应的 value值
                // 即为我们想要的当前线程对应此ThreadLocal的值
                T result = (T)e.value;
                return result;
            }
        }
        /*
        	初始化 : 有两种情况有执行当前代码
        	第一种情况: map不存在，表示此线程没有维护的ThreadLocalMap对象
        	第二种情况: map存在, 但是没有与当前ThreadLocal关联的entry
         */
        return setInitialValue();
    }

    /**
     * 初始化
     *
     * @return the initial value 初始化后的值
     */
    private T setInitialValue() {
        // 调用initialValue获取初始化的值
        // 此方法可以被子类重写, 如果不重写默认返回null
        T value = initialValue();
        // 获取当前线程对象
        Thread t = Thread.currentThread();
        // 获取此线程对象中维护的ThreadLocalMap对象
        ThreadLocalMap map = getMap(t);
        // 判断map是否存在
        if (map != null)
            // 存在则调用map.set设置此实体entry
            map.set(this, value);
        else
            // 1）当前线程Thread 不存在ThreadLocalMap对象
            // 2）则调用createMap进行ThreadLocalMap对象的初始化
            // 3）并将 t(当前线程)和value(t对应的值)作为第一个entry存放至ThreadLocalMap中
            createMap(t, value);
        // 返回设置的值value
        return value;
    }
```

**get执行流程**

+ A. 首先获取当前线程, 根据当前线程获取一个Map

+ B. 如果获取的Map不为空，则在Map中以`ThreadLocal`的引用作为key来在Map中获取对应的Entry  e，为空则执行D

+ C. 如果e不为null，则返回`e.value`，否则转到D

+  D. Map为空或者e为空，则通过`initialValue`函数获取初始值value，然后用`ThreadLocal`的引用和`value`作为`firstKey`和`firstValue`创建一个新的Map

**remove方法**

```java
/**
     * 删除当前线程中保存的ThreadLocal对应的实体entry
     */
     public void remove() {
        // 获取当前线程对象中维护的ThreadLocalMap对象
         ThreadLocalMap m = getMap(Thread.currentThread());
        // 如果此map存在
         if (m != null)
            // 存在则调用map.remove
            // 以当前ThreadLocal为key删除对应的实体entry
             m.remove(this);
     }
```

 **remove执行流程**

+ A. 首先获取当前线程，并根据当前线程获取一个Map

+  B. 如果获取的Map不为空，则移除当前`ThreadLocal`对象对应的`entry`

**`initialValue`方法**

```java
/**
  * 返回当前线程对应的ThreadLocal的初始值
  
  * 此方法的第一次调用发生在，当线程通过get方法访问此线程的ThreadLocal值时
  * 除非线程先调用了set方法，在这种情况下，initialValue 才不会被这个线程调用。
  * 通常情况下，每个线程最多调用一次这个方法。
  *
  * <p>这个方法仅仅简单的返回null {@code null};
  * 如果程序员想ThreadLocal线程局部变量有一个除null以外的初始值，
  * 必须通过子类继承{@code ThreadLocal} 的方式去重写此方法
  * 通常, 可以通过匿名内部类的方式实现
  *
  * @return 当前ThreadLocal的初始值
  */
protected T initialValue() {
    return null;
}
```

### 3.3 `ThreadLocalMap`源码

![image-20231211180515137](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/image-20231211180515137.png)

 `ThreadLocalMap`是`ThreadLocal`的静态内部类，没有实现Map接口，用独立的方式实现了Map的功能，其内部的Entry也是独立实现。

```java
   static class ThreadLocalMap { // ThreadLocalMap是ThreadLocal的静态内部类
       //Entry是ThreadLocalMap静态内部类，继承了虚引用
        static class Entry extends WeakReference<ThreadLocal<?>> { 
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

**成员变量**

```java
    /**
     * 初始容量 —— 必须是2的整次幂
     */
    private static final int INITIAL_CAPACITY = 16;

    /**
     * 存放数据的table，Entry类的定义在下面分析
     * 同样，数组长度必须是2的整次幂。
     */
    private Entry[] table;

    /**
     * 数组里面entrys的个数，可以用于判断table当前使用量是否超过阈值。
     */
    private int size = 0;

    /**
     * 进行扩容的阈值，表使用量大于它的时候进行扩容。
     */
    private int threshold; // Default to 0
```

跟`HashMap`类似，INITIAL_CAPACITY代表这个Map的初始容量；table 是一个Entry 类型的数组，用于存储数据；size 代表表中的存储数目； threshold 代表需要扩容时对应 size 的阈值。

**存储结构 - Entry**

```java
/*
 * Entry继承WeakReference，并且用ThreadLocal作为key.
 * 如果key为null(entry.get() == null)，意味着key不再被引用，
 * 因此这时候entry也可以从table中清除。
 */
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

在`ThreadLocalMap`中，也是用Entry来保存K-V结构数据的。不过Entry中的key只能是`ThreadLocal`对象，这点在构造方法中已经限定死了。另外，Entry继承`WeakReference`，也就是key（`ThreadLocal`）是弱引用，其目的是将`ThreadLocal`对象的生命周期和线程生命周期解绑。

## 4. `ThreadLocal`内存泄漏

内存泄漏是指程序中已动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。内存泄漏的堆积终将导致内存溢出

`ThreadLocalMap` 中使用的 key 为 `ThreadLocal` 的弱引用，而 value 是强引用。所以，如果 `ThreadLocal` 没有被外部强引用的情况下，在垃圾回收的时候，key 会被清理掉，而 value 不会被清理掉。

这样一来，`ThreadLocalMap` 中就会出现 key 为 null 的 Entry。假如我们不做任何措施的话，value 永远无法被 GC 回收，这个时候就可能会产生内存泄露。`ThreadLocalMap` 实现中已经考虑了这种情况，在调用 `set()`、`get()`、`remove()` 方法的时候，会清理掉 key 为 null 的记录。使用完 `ThreadLocal`方法后最好手动调用`remove()`方法

# 线程池

![image-20231211154038527](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/image-20231211154038527.png)

## 1. 自定义线程池

一个自定义的线程池应该包含线程池集合、核心线程数、阻塞队列、超时时间

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

# 线程安全的集合类

![image-20231211155902995](https://note-1322176778.cos.ap-guangzhou.myqcloud.com/java/juc/image-20231211155902995.png)

线程安全集合类可以分为三大类：

+ 遗留的线程安全集合如 `Hashtable`， `Vector`

+ 使用 Collections 装饰的线程安全集合，如：
  +  `Collections.synchronizedCollection`
  + `Collections.synchronizedList`
  + `Collections.synchronizedMap`
  + `Collections.synchronizedSet`
  + `Collections.synchronizedNavigableMap`
  + `Collections.synchronizedNavigableSet `
  + `Collections.synchronizedSortedMap`
  + `Collections.synchronizedSortedSet`

+ `java.util.concurrent.*`

`java.util.concurrent.* `下的线程安全集合类，可以发现它们有规律，里面包含三类关键词：

`Blocking`、`CopyOnWrite`、`Concurrent`

+ Blocking 大部分实现基于锁，并提供用来阻塞的方法

+ `CopyOnWrite` 之类容器修改开销相对较重

+ Concurrent 类型的容器
  + 内部很多操作使用 `cas `优化，一般可以提供较高吞吐量
  + 弱一致性，遍历时弱一致性，例如，当利用迭代器遍历时，如果容器发生修改，迭代器仍然可以继续进行遍历，这时内容是旧
  + 求大小弱一致性，size 操作未必是 100% 准确
  + 读取弱一致性

## 1. `ConcurrentHashMap`

### 1.1 使用（单词计数）

```java
/**
 * 单词计数
 */
public class Demo1 {
    public static void main(String[] args) {
        //一是提供一个 map 集合，用来存放每个单词的计数结果，key 为单词，value 为计数
        //二是提供一组操作，保证计数的安全性，会传递 map 集合以及 单词 List
        demo(
                //线程不安全
//                ()->new HashMap<String, Integer>(),
                //使用ConcurrentHashMap也不行，因为map.get() 和map.put()不是原子操作，必须保证两者是原子操作
                ()->new ConcurrentHashMap<String, LongAdder>(),
                (map, words)->{
                    //1. 使用synchronized锁住整个代码块，影响并发度
                    //2. 使用ConcurrentHashMap的computeIfAbsent方法
                    //如果缺少key，则生成一个value，然后将key,value放入map中
                    for (String word : words) {
                        //使用原子累加器，保证累加操作时原子的
                        LongAdder value = map.computeIfAbsent(word, (key) -> new LongAdder());
                        value.increment();
                    }
                }
        );
    }
    
    //开启多线程读取文件中的单词
    private static <V> void demo(Supplier<Map<String, V>> supplier,
                                 BiConsumer<Map<String, V>, List<String>> consumer) {
        Map<String, V> counterMap = supplier.get();
        List<Thread> ts = new ArrayList<>();
        for (int i = 1; i <= 26; i++) {
            int idx = i;
            Thread thread = new Thread(() -> {
                List<String> words = readFromFile(idx);
                consumer.accept(counterMap, words);
            });
            ts.add(thread);
        }
        ts.forEach(t -> t.start());
        ts.forEach(t -> {
            try {
                t.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        System.out.println(counterMap);
    }

    //读取文件
    public static List<String> readFromFile(int i) {
        ArrayList<String> words = new ArrayList<>();
        try (BufferedReader in = new BufferedReader(new InputStreamReader(
                new FileInputStream("src/main/resources/tmp/" + i + ".txt")))) {
            while (true) {
                String word = in.readLine();
                if (word == null) {
                    break;
                }
                words.add(word);
            }
            return words;
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

### 1.2 原理

