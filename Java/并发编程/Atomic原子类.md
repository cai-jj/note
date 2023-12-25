# 问题

**1. 有哪些原子类？**

**2.LongAdder相对于AtomicLong为什么快？**

**3.LongAdder原理？**

**4.Unsafe对象**

**5.缓存行伪共享？**

# Atomic原子类

## 基本类型原子类

使用原子的方式更新基本类型

- `AtomicInteger`：整型原子类
- `AtomicLong`：长整型原子类
- `AtomicBoolean`：布尔型原子类

上面三个类提供的方法几乎相同，所以我们这里以 `AtomicInteger` 为例子来介绍。

**AtomicInteger 类常用方法**

```java
public final int get() //获取当前的值
public final int getAndSet(int newValue)//获取当前的值，并设置新的值
public final int getAndIncrement()//获取当前的值，并自增
public final int getAndDecrement() //获取当前的值，并自减
public final int getAndAdd(int delta) //获取当前的值，并加上预期的值
boolean compareAndSet(int expect, int update) //如果输入的数值等于预期值，则以原子方式将该值设置为输入值（update）
public final void lazySet(int newValue)//最终设置为newValue,使用 lazySet 设置之后可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
```

### AtomicInteger 线程安全原理简单分析

```java
    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe(); //获取unsafe对象
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset 
                (AtomicInteger.class.getDeclaredField("value")); //获取value在该类的偏移量
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
```

`AtomicInteger` 类主要利用 CAS + volatile 来保证原子操作。其底层是调用unsafe对象的`objectFieldOffset`方法，这个方法可以用来拿到类成员变量(value)的在该类中的偏移量。 value 是一个 volatile 变量，保证任何时刻任何线程总能拿到该变量的最新值。拿到最新值后调用unsafe对象的CAS方法进行CAS操作。

## 数组类型原子类

使用原子的方式更新数组里的某个元素

- `AtomicIntegerArray`：整型数组原子类
- `AtomicLongArray`：长整型数组原子类
- `AtomicReferenceArray`：引用类型数组原子类

## 引用类型原子类

+ `AtomicReference`：引用类型原子类

+ `AtomicStampedReference`：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于解决原子的更新数据和数据的版本号，可以解决使用 CAS 进行原子更新时可能出现的 ABA 问题

+ `AtomicMarkableReference`：原子更新带有标记的引用类型。该类将 boolean 标记与引用关联起来，不能解决ABA问题

## 字段更新器

`AtomicIntegerFieldUpdater`:原子更新整型字段的更新器

`AtomicLongFieldUpdater`：原子更新长整型字段的更新器

`AtomicReferenceFieldUpdater`：原子更新引用类型里的字段

## 原子累加器

### LongAdder原理

AtomicLong进行累加是多个线程对一个基础值进行累加操作

而LongAdder在没有线程竞争的时候是对一个基础值进行累加，当出现竞争时，就会创建多个累加单元（cell数组），各自线程对不同的cell进行累加，最后汇总。