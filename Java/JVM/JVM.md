## java内存结构

分为两大块：线程私有和线程共享

线程私有：程序计数器、java虚拟机栈、本地方法栈

线程共享：方法区、堆

### 1.程序计数器

记录的是当前线程下一条字节码指令执行的地址值

程序计数器是唯一一个不会出现 内存溢出的内存区域

### 2.Java虚拟机栈

每个线程运行的内存，java虚拟机每个方法执行时都会创建一个栈帧，让后将栈帧入栈

虚拟机栈由一个个栈帧组成

每个栈帧存放着局部变量表、操作数栈、动态链接等信息

局部变量表的大小在编译期间就确定了

- 局部变量表：存放方法的局部变量(this、方法参数、方法的局部变量) long 和double占两个槽，其余类型占用一个槽
- 操作数栈：用于存放临时变量值以及变量运算产生的结果
- 动态链接：符号引用转换直接引用

栈内存溢出(`**StackOverFlowError**`)：栈帧过多导致栈内存溢出

```
/**
 * 栈内存溢出
 * 通过-Xss配置栈的最大内存
 */
public class StackDemo1 {
    static int count = 0;
    public static void main(String[] args) {
        try {
            method();
        } catch (Throwable e) {
            e.printStackTrace();
            System.out.println(count);
        }
    }

    public static void method() {
        count++;
        method();
    }
}
```

问题：

1.垃圾回收器是否涉及栈内存？

不涉及，方法调用进栈，在执行完后会自动弹出栈

2.栈内存分配越大越好吗？

不一定，物理内存是固定大小，栈内存过大会使可分配的线程数变少

3.方法内的局部变量是否是线程安全的？

- 如果是方法的局部变量是在方法体内部声明的，是线程安全的
- 如果局部变量引用了对象，作为参数或者是返回值，不一定是线程安全的

### 3. 本地方法栈

存放运行native修饰的方法的

Object类中的native方法

```
private static native void registerNatives();

public final native Class<?> getClass();

public native int hashCode();

protected native Object clone() throws CloneNotSupportedException;

public final native void notify();

public final native void notifyAll();

public final native void wait(long timeout) throws InterruptedException;
```

### 4. 方法区

JDK1.8之前存放在堆内存的永久代中、1.8之后存放在元空间(使用的是直接内存)

存放class文件的类型信息、常量、静态变量

方法区内存溢出(元空间内存溢出)

```
/**
 * 演示元空间内存溢出 java.lang.OutOfMemoryError: Metaspace
 * 元空间使用的是物理内存，通过参数设置调节小一点
 * -XX:MaxMetaspaceSize=10m
 */
public class Demo1_8 extends ClassLoader { // 可以用来加载类的二进制字节码
    public static void main(String[] args) {
        int j = 0;
        try {
            Demo1_8 test = new Demo1_8();
            for (int i = 0; i < 10000; i++, j++) {
                // ClassWriter 作用是生成类的二进制字节码
                ClassWriter cw = new ClassWriter(0);
                // 版本号， public， 类名, 包名, 父类， 接口
                cw.visit(Opcodes.V1_8, Opcodes.ACC_PUBLIC, "Class" + i, null, "java/lang/Object", null);
                // 返回 byte[]
                byte[] code = cw.toByteArray();
                // 执行了类的加载
                test.defineClass("Class" + i, code, 0, code.length); // Class 对象
            }
        } finally {
            System.out.println(j);
        }
    }
}
```

### 5. 堆

存放实例对象

对象创建过程：

**1.类加载检查**

检查是否加载了对应的类

**2.分配内存**

类加载过程之后就开始创建对象，先为对象分配内存(对象所需要的内存在类加载完成之后可以确定)，所以只需要从堆内存中划出一块确定大小的空间即可；**分配内存方式**有 **“指针碰撞”** 和 **“空闲列表”** 两种

- 指针碰撞：
  - 适用场合：堆内存规整（即没有内存碎片）的情况下。
  - 原理：用过的内存全部整合到一边，没有用过的内存放在另一边，中间有一个分界指针，只需要向着没用过的内存方向将该指针移动对象内存大小位置即可。
  - 使用该分配方式的 `GC` 收集器：`Serial`, `ParNew`
- 空闲列表：
  - 适用场合：堆内存不规整的情况下。
  - 原理：虚拟机会维护一个列表，该列表中会记录哪些内存块是可用的，在分配的时候，找一块儿足够大的内存块儿来划分给对象实例，最后更新列表记录。
  - 使用该分配方式的 `GC` 收集器：`CMS`

选择以上两种方式中的哪一种，取决于 `Java` 堆内存是否规整。而 `Java` 堆内存是否规整，取决于 `GC` 收集器使用的算法是"标记-清除"(有内存碎片)，"标记-整理"(没有内存碎片)，复制算法(没有内存碎片)

内存分配并发问题：在创建对象时并非线程安全的(在给A对象分配内存时，指针还没来得及修改，B对象又使用了原来的指针)

解决方法：

- `CAS`+失败重试
- `TLAB`**：** 为每一个线程预先在 Eden 区分配一块儿内存，JVM 在给线程中的对象分配内存时，首先在 TLAB 分配，当对象大于 TLAB 中的剩余内存或 TLAB 的内存已用尽时，再采用上述的 CAS 进行内存分配

**3.初始零值**

内存分配完成后，虚拟机需要将分配到的内存空间都初始化为零值（不包括对象头）

**4.设置对象头**

**5.执行init()方法(构造函数)**

初始化所有的成员变量，收集所有 {} 代码块和成员变量赋值的代码

对象内存布局：对象头、实例数据、对齐填充

1.对象头：包括两部分信息，第一部分为`Mark Word`，用于存储对象自身的一些信息（哈希码、GC 分代年龄、锁状态标志等等），另一部分是类型指针，指向方法区类信息的指针，表明该对象是哪个类的实例对象。

32位虚拟机对象头大小为8个字节（`mark word`占4个字节(32bit), 类型指针占4个字节）
64位虚拟机对象头大小为16个字节 (`mark word`占8个字节(64bit), 类型指针占8个字节）

2.实例数据：对象真正存储的有效信息

3.对齐填充：凑位数（8的整数倍）

对象访问方式：直接指针和句柄

堆结构：新生代(1个Eden区、2个Survivor 区)、老年代、永久代（JDK1.8以后为元空间)

对象首先都会在 Eden 区分配，在一次新生代垃圾回收后，如果对象还存活，则会进入 Survivor区，并且对象的年龄还会加 1(Eden 区->Survivor 区后对象的初始年龄变为 1)，当它的年龄增加到一定程度（最大为15 岁，因为只使用了4个比特位存放GC年龄信息），就会被晋升到老年代中。对象晋升到老年代的年龄阈值，可以通过参数 `-XX:MaxTenuringThreshold` 来设置。

堆内存溢出( `OutOfMemoryError`) 

```
/**
 * 堆内存溢出
 * -Xms设置堆内存最大值
 */
public class HeapDemo {
    public static void main(String[] args) {
        int i = 0;
        try {
            List<String> list = new ArrayList<>();
            String s = "hello";
            while (true) {
                list.add(s);
                s = s + s;
                i++;
            }
        } catch (Throwable e) {
            e.printStackTrace();
            System.out.println(i);
        }
    }
}
```

堆内存诊断

1. jps 工具

查看当前系统中有哪些 java 进程

2. jmap 工具

查看堆内存占用情况 jmap - heap 进程id

3. jconsole 工具

图形界面的，多功能的监测工具，可以连续监测

4.javap -v + .class文件 反编译

```
/**
 * 演示堆内存查询工具
 * jps查看在运行的进程
 * jmap -heap + 进程id查看堆内存情况
 */
public class HeapDemo2 {

    public static void main(String[] args) throws InterruptedException {
        System.out.println("1...");
        Thread.sleep(30000);
        byte[] array = new byte[1024 * 1024 * 10]; // 10 Mb
        System.out.println("2...");
        Thread.sleep(20000);
        array = null;
        System.gc();
        System.out.println("3...");
        Thread.sleep(1000000L);
    }
}
```

### 6.字符串常量池

JDK1.7之前存放在方法区；JDK1.7之后存放在堆内存中

JDK 1.7 为什么要将字符串常量池移动到堆中？

主要是因为永久代（方法区实现）的 GC 回收效率太低，只有在整堆收集 (Full GC)的时候才会被执行 GC。Java 程序中通常会有大量的被创建的字符串等待回收，将字符串常量池放到堆中，能够更高效及时地回收字符串内存。

```
        String s1 = "a"; //若串池没有该字符串，则将该字符串加载到串池中 StringTable["a"]
        String s2 = "b";  //StringTable["a","b"]
        String s3 = "ab"; //StringTable["a","b","ab"]
        String s4 = s1 + s2; //字符串变量拼接操作，new StringBuilder().append(a).append(b).toString()
        //toString方法本质就是new String("ab") 在堆内存中
        String s5 = "a" + "b";//编译优化 相当于s5 == "ab" 在串池中
        System.out.println(s3 == s4);
        System.out.println(s3 == s5);
```

字符串intern操作

JDK1.7之后

将字符串对象放入串池当中，如果串池有则不会放入，如果没有则放入，最后都会把串池的对象返回

JDK1.7之前

将字符串对象放入串池当中，如果串池有则不会放入，如果没有会把对象复制一份放入，最后都会把串池的对象返回

```
        String s1 = new String("a") + new String("b");
        //串池只会存放两个,StringTable ["a","b"],"ab"不在串池
        System.out.println(s1 == "ab");
        String s2 = s1.intern(); //将这个字符串对象放入串池当中，如果串池有则不会放入，如果没有则放入
        // 最后都会把串池的对象返回，所以s2的对象一定存在串池
        //因为s1对象"ab"不在串池，所以调用s1.intern()后串池就会存在"ab"，StringTable ["a","b","ab"]
        System.out.println(s2 == "ab");
```

### 7. 直接内存

通过`new DirectByteBuffer(capacity)`对象分配直接内存

```
ByteBuffer byteBuffer = ByteBuffer.allocateDirect(_100M);
//调用new DirectByteBuffer
public static ByteBuffer allocateDirect(int capacity) {
        return new DirectByteBuffer(capacity);
}
```

底层真正 完成直接内存的分配回收是Unsafe 对象

```
//直接内存分配
//base是直接内存地址
base = unsafe.allocateMemory(size);
unsafe.setMemory(base, size, (byte) 0);
//释放内存
unsafe.freeMemory(base);
```

`ByteBuffer` 的实现类内部，使用了 `Cleaner `（虚引用）来监测 `ByteBuffer` 对象，一旦`ByteBuffer `对象被垃圾回收，那么就会由`ReferenceHandler` 线程通过 Cleaner类 的 clean 方法调用 freeMemory方法来释放直接内存

------

## 垃圾回收

### 1.如何判断一个对象可以被回收

#### 1.1引用计数法

给每个对象添加一个引用计数器，当一个地方引用这个对象时，引用计数器加1，当该引用失效，计数器减1；

引用计数器为0就表示可以被回收

问题：存在循环引用问题

#### 1.2可达性分析算法

一个对象是否可以由GCRoot对象的引用找到，如果能找到就不能被回收，不能找到就被回收

GCRoot对象？

- 启动类加载器加载的对象
- native方法中引用的对象
- 正在活动的线程的对象，如当前线程局部变量表中的局部变量引用的对象
- 加锁的对象
- 方法区中类静态属性引用的对象    (static修饰的变量引用的对象)
- 方法区中常量引用的对象     (常量引用的对象)

#### 1.3 四种引用类型

1.强引用

GCRoot可以到达的对象，不会被垃圾回收器回收

2.软引用

通过强引用引用一个软引用，软引用再引用软引用对象

在进行垃圾回收，软引用只有内存不足才会被回收，**主要用于缓存**

```
/**
 * 软引用
 * -Xmx200m
 */
public class Demo1 {
    public static void main(String[] args) {
        //强引用
        byte[] bytes = new byte[1024 * 1024 * 100];
        //创建软引用对象bytes
        SoftReference<byte[]> softReference = new SoftReference<>(bytes);
        //释放强引用
        bytes = null;
        //获取软引用对象
        System.out.println(softReference.get());
        //在堆中申请100m，内存不足，软引用会被回收
        byte[] bytes1 = new byte[1024 * 1024 * 100];
        System.out.println(softReference.get()); //值为null
//        byte[] bytes2 = new byte[1024 * 1024 * 100];
        softReference = null; //直接将softReference置为null，会导致软引用包含的对象全部被回收
        // 但是软引用包含的对象不应该立刻会被回收，后续可能还会使用，只有内存不足才会被回收
        //实现一种机制，软引用对象里面的数据回收，软引用也应该被回收
    }
}
```

当软引用包含的对象被回收掉后，因为软引用也是一个对象，就会把软引用放入引用队列中，通过软引用队列机制进行软引用回收（遍历该队列，依次弹出元素）

```
/**
 * 软引用队列用法
 * -Xmx200m
 */
public class Demo2 {
    public static void main(String[] args) {
        //创建软引用队列
        ReferenceQueue<byte[]> queue = new ReferenceQueue<>();
        byte[] bytes = new byte[1024 * 1024 * 100];
        //创建软引用对象bytes,关联软引用队列
        SoftReference<byte[]> softReference = new SoftReference<>(bytes, queue);
        bytes = null;
        System.out.println(softReference.get());
        //内存充足，队列为空
        System.out.println("queue:" + queue.poll());
        byte[] bytes1 = new byte[1024 * 1024 * 100];
        System.out.println(softReference.get());
        //内存不足，将软引用包含对象回收，会自动将软引用对象入队列，队列不为空
        System.out.println("queue:" + queue.poll());
    }
}
```

3.弱引用

通过强引用引用一个弱引用，弱引用再引用弱引用对象

不管内存够不够，只要进行垃圾回收，就会被回收，主要在`ThreadLocal`使用

```
/**
 * 弱引用
 */
public class Demo3 {
    public static void main(String[] args) {
        //强引用
        byte[] bytes = new byte[1024 * 1024 * 100];
        //创建弱引用对象bytes
        WeakReference<byte[]> weakReference = new WeakReference<>(bytes);
        //释放强引用
        bytes = null;
        //获取弱引用对象
        System.out.println(weakReference.get());
        //进行垃圾回收
        System.gc();
        //再次获取弱引用对象
        System.out.println(weakReference.get());
    }
}
```

当弱引用对象被回收掉后，因为弱引用也是一个对象，会把弱引用放入引用队列中

软引用和弱引用可以配合引用队列使用，也可以不配合引用队列使用；

而虚引用的使用需要关联一个虚引用队列

4.虚引用

配合`ByteBuffer`使用，当虚引用对象被回收时，虚引用入队列，由一个`Reference Handle`线程调用虚引用的方法释放直接内存。

### 2.垃圾回收算法

#### 2.1 标记清除算法

先标记所有需要回收的对象，然后清除

缺点：

- 导致内存碎片
- 堆中包含大量对象，而大多数又是要回收的，会导致标记和清除的时间都过长，效率不高

#### 2.2 复制算法

将内存一分为二，每次使用其中的一块， 当这一块的内存使用完后，就将存活的对象复制到另一块内存去，然后再把原来的内存一次清理掉。

缺点：可用的内存减为原来的一半

#### 2.3 标记整理算法

先标记存活的对象，将所有存活对象向内存空间的一端移动，再清理边界以外的对象

缺点：

- 需要移动存活对象，效率不高；
- 移动存活对象必须暂停用户线程(stop the world)

### 3.分代垃圾回收算法

`GC`种类

部分收集 (Partial GC)：

- 新生代收集（Minor GC / Young GC）：只对新生代进行垃圾收集；
- 老年代收集（Major GC / Old GC）：只对老年代进行垃圾收集。需要注意的是 Major GC 在有的语境中也用于指代整堆收集；
- 混合收集（Mixed GC）：对整个新生代和部分老年代进行垃圾收集。

整堆收集 (Full GC)：收集整个 Java 堆和方法区。

分代垃圾回收：

将堆空间分为**新生代**和**老年代**,，新生代分为1块Eden区和2块Suvivor区(from、to)；

对象首先分配在伊甸园区域，当新生代空间不足时，触发 `minor gc`，伊甸园和 `from` 存活的对象使用 **复制算法**到 `to` 中，存活的对象年龄加 1并且交换 from to指针的地址值；

`minor gc` 会引发 `stop the world`，暂停其它用户的线程，等垃圾回收结束，用户线程才恢复运行；

当对象寿命超过阈值时，会晋升至老年代，最大寿命是15（4bit），每个对象的对象头的`Mark Word`中使用`4bit`存储`GC`年龄标识；

当老年代空间不足，会先尝试触发 `minor gc`，如果之后空间仍不足，那么触发 `full gc`，`STW`的时间更长

新生代与老年代使用的垃圾回收算法取决于使用的垃圾回收器

### 4. 垃圾回收器

#### 4.1 Serial

单线程进行垃圾回收，进行垃圾回收时必须暂停所有的用户线程(Stop the world)

新生代-复制算法；老年代-标记整理算法

#### 4.2 ParNew

多线程进行垃圾回收，进行垃圾回收时必须暂停所有的用户线程(Stop the world)

新生代-复制算法；老年代-标记整理算法

#### 4.3 Parallel Scavenge

多线程进行垃圾回收，关注吞吐量，提高CPU利用率

新生代-标记复制算法；老年代-标记整理算法

#### 4.4 CMS

可以并发收集(用户线程和垃圾回收同时工作)

`CMS`具体过程：

1.初始标记

stop the world,标记与GCRoot直接关联的对象，速度很快

2.并发标记

用户线程和垃圾收集并发执行，标记从GCRoot直接关联的对象遍历整个对象

3.重新标记

stop the world,用户线程和垃圾收集并发会导致标记错误，所有需要重新标记那些标记发生变动的对象。

4.并发清除

并发执行，对没有标记的对象进行清除

缺点：

- 对处理器资源敏感(并发阶段，虽然不会暂停用户线程，但会占用一部分线程，从而导致应用程序变慢)
- 无法处理浮动垃圾（并发标记和并发清理的过程，因为用户线程在执行，就会产生新的垃圾对象，垃圾对象是在标记之后产生的，无法里面收集，只能等到下次，就需要预留足够空间给用户线程，否则就会导致提前full gc）
- 使用的是标记清除算法，会导致内存碎片

三色标记

#### 4.5 G1

------

## 类加载与字节码

### 1.类文件结构

### 2. 编译期优化处理(语法糖)

所谓的 `语法糖` ，其实就是指 java 编译器把 *.java 源码编译为 *.class 字节码的过程中，自动生成和转换的一些代码

#### 1. 默认构造器

```
public class Candy1 {
}
```

编译后的代码

```
public class Candy1 {
	// 这个无参构造是编译器帮助我们加上的
	public Candy1() {
		super(); // 即调用父类 Object 的无参构造方法，即调用 java/lang/Object."<init>":()V
	}
}
```

#### 2. 自动拆装箱

这个特性是 JDK 5 开始加入的

```
public class Demo2 {
    public static void main(String[] args) {
        //装箱
        Integer x = 1;
        //拆箱
        int y = x;
    }
}
```

编译后的代码

```
public class Demo2 {
	public static void main(String[] args) {
		Integer x = Integer.valueOf(1);
		int y = x.intValue();
	}		
}
```

#### 3. 泛型擦除

#### 4. 可变参数

#### 5. `foreach` 循环

#### 6. 桥接方法

方法重写时对返回值分两种情况：

- 父子类的返回值完全一致
- 子类返回值可以是父类返回值的子类（编译器还会生成一个桥接方法）

```
class A {
    public Number m() {
        return 1;
    }
}
class B extends A {
    @Override
    // 子类 m 方法的返回值是 Integer 是父类 m 方法返回值 Number 的子类
    public Integer m() {
        return 2;
    }
    //如果返回值类型相同，则不会生成桥接方法
    //编译器还会生成一个桥接方法
    /*
    // 此方法才是真正重写了父类 public Number m() 方法
    public synthetic bridge Number m() {
        // 调用 public Integer m()
        return m();
    }
     */
}
```

可以通过反射验证

```
        Method[] methods = B.class.getDeclaredMethods();
        for (Method method : methods) {
            System.out.println(method); // 会输出两个方法
        }
```

## 构造方法

<cinit>()方法

**类的初始化，初始化所有类变量**

编译器会按从上至下的顺序，收集所有 static 静态代码块和静态成员赋值的代码，合并为<cinit>()方法 

<init>()方法

**对象的初始化，初始化所有成员变量**

编译器会按从上至下的顺序，收集所有 {} 代码块和成员变量赋值的代码，形成<init>()方法

## 方法调用

静态绑定：final，private，构造方法，由 invokespecial 指令来调用；static修饰的方法，invokestatic调用

动态绑定：普通成员方法，是由 invokevirtual 调用，支持多态

多态原理：补充

## 异常处理

### **try catch**

```
public class Demo1 {
    public static void main(String[] args) {
        int i = 0;
        try {
            i = 10;
        } catch (Exception e) {
            i = 20;
        }
    }
}
```

字节码分析

```
/**
 *    Code:
 *       stack=1, locals=3, args_size=1
 *          0: iconst_0 // this放入0号槽
 *          1: istore_1 // i放入1号槽
 *          2: bipush        10 // 10放入操作数栈
 *          4: istore_1  //取出栈顶元素放入1号槽 i = 10
 *          5: goto          12
 *          8: astore_2   //异常对象的引用存入2号槽
 *          9: bipush        20
 *         11: istore_1
 *         12: return
 *       Exception table:
 *          from    to  target type
 *              2     5     8   Class java/lang/Exception //异常表，监测2-4行字节码指令，如果发生异常
 *              //将发生的异常异常表类型进行匹配，匹配成功就执行target值第8行
 *       
 *       LocalVariableTable:
 *         Start  Length  Slot  Name   Signature
 *             9       3     2     e   Ljava/lang/Exception;
 *             0      13     0  args   [Ljava/lang/String;
 *             2      11     1     i   I
 */
```

### **多个try c**atch

```
public class Demo2 {
    public static void main(String[] args) {
        int i = 0;
        try {
            i = 10;
        } catch (ArithmeticException e) {
            i = 30;
        } catch (NullPointerException e) {
            i = 40;
        } catch (Exception e) {
            i = 50;
        }
    }
}
```

字节码

```
/**
 *     Code:
 *       stack=1, locals=3, args_size=1
 *          0: iconst_0 //初始化this
 *          1: istore_1 // i = 0
 *          2: bipush        10 //10放入操作数栈
 *          4: istore_1   // i = 10
 *          5: goto          26
 *          8: astore_2  // ArithmeticException对象引用存入2号槽
 *          9: bipush        30
 *         11: istore_1
 *         12: goto          26
 *         15: astore_2 // NullPointerException对象引用存入2号槽
 *         16: bipush        40
 *         18: istore_1
 *         19: goto          26
 *         22: astore_2 // Exception对象引用存入2号槽
 *         23: bipush        50
 *         25: istore_1
 *         26: return
 *       Exception table: //监测2-4行异常，根据type匹配，匹配成功，执行对应的target行的指令
 *          from    to  target type
 *              2     5     8   Class java/lang/ArithmeticException
 *              2     5    15   Class java/lang/NullPointerException
 *              2     5    22   Class java/lang/Exception

 *       LocalVariableTable: //都是用2号槽存放e，因为异常只能发一个，复用
 *         Start  Length  Slot  Name   Signature
 *             9       3     2     e   Ljava/lang/ArithmeticException;
 *            16       3     2     e   Ljava/lang/NullPointerException;
 *            23       3     2     e   Ljava/lang/Exception;
 *             0      27     0  args   [Ljava/lang/String;
 *             2      25     1     i   I
 */
```

**为异常出现时，只能进入 Exception table 中一个分支，所以局部变量表 slot 2 位置被共用**

### **finally**

```
public class Demo3 {
    public static void main(String[] args) {
        int i = 0;
        try {
            i = 10;
        } catch (Exception e) {
            i = 20;
        } finally {
            i = 30;
        }
    }
}
```

字节码

```
/**
 *     Code:
 *       stack=1, locals=4, args_size=1
 *          0: iconst_0 // this
 *          1: istore_1 // i
 *          2: bipush        10
 *          4: istore_1 // i = 10
 *          5: bipush        30 // finally赋值操作会放在try、catch、catch剩余的异常类型流
 * 程，所以finally指令会被复制三份
 *          7: istore_1 // i = 30
 *          8: goto          27
 *         11: astore_2  // Exception对象引用存入2号槽
 *         12: bipush        20
 *         14: istore_1  // i = 20
 *         15: bipush        30 // catch下的finally复制
 *         17: istore_1  // i = 30
 *         18: goto          27
 *         21: astore_3   // 发生异常catch没捕获的异常，按剩余异常处理，将对象引用存入3号槽(隐藏槽号，
 *         局部变量表没有显示，但存在，因为局部变量表长度为4)
 *         22: bipush        30 // 剩余异常类型的finally复制
 *         24: istore_1
 *         25: aload_3
 *         26: athrow
 *         27: return
 *       Exception table:
 *          from    to  target type
 *              2     5    11   Class java/lang/Exception
 *              2     5    21   any // 发生异常catch没捕获的异常，
 *             11    15    21   any
 *
 *       LocalVariableTable:
 *         Start  Length  Slot  Name   Signature
 *            12       3     2     e   Ljava/lang/Exception;
 *             0      28     0  args   [Ljava/lang/String;
 *             2      26     1     i   I
 */
```

**finally 中的代码被复制了 3 份，分别放入 try 流程，catch 流程以及 catch 剩余的异常类型流程**

### finally与return

如果在 finally 中出现了 return，会吞掉异常

## 类加载阶段

1.加载 2.链接 3.初始化

类的生命周期阶段：加载、链接、初始化、使用、销毁

### 1.加载

通过不同的类加载器将字节码加载到方法区，生成该类的class对象

如果这个类还有父类没有加载，先加载父类

### 2.链接

验证、准备、解析

验证：对字节码格式进行验证

准备：

- 为 static 变量分配空间，设置默认值
- static 变量分配空间和赋值是两个步骤，分配空间在准备阶段完成，赋值在初始化阶段完成
- 如果 static  final 的基本类型和字符串常量，编译阶段值就确定了，赋值在准备阶段完成
- 如果 static  final 的引用类型，那么赋值也会在初始化阶段完成

解析：常量池符号引用转换为直接引用

### 3.初始化

执行cinit()方法（保证父类的cinit()方法已经执行）

cinit方法主要是将类中成员变量和静态语句块按顺序初始化给定的值

类初始化是懒惰的

初始化的时机：

- main 方法所在的类，总会被首先初始化
- 首次访问这个类的静态变量或静态方法时
- 子类初始化，如果父类还没初始化，父类先初始然后子类再初始化
- 子类访问父类的静态变量，只会触发父类的初始化
- Class.forName
- new 会导致初始化

不会初始化：

- 访问类的 static final 静态常量（基本类型和字符串）不会触发初始化
- 类对象.class 不会触发初始化
- 创建该类的数组不会触发初始化
- 类加载器的 loadClass 方法
- Class.forName 的参数 2 为 false 时

```
package com.cjj.init;

public class Demo1 {
    static {
        System.out.println("main init");
    }
    public static void main(String[] args) throws ClassNotFoundException {

        /*
        不会初始化的时候
         */
        //1. 静态常量（基本类型和字符串）不会触发初始化
        System.out.println(B.b);
        //2. 类对象.class 不会触发初始化
        System.out.println(B.class);
        // 3. 创建该类的数组不会触发初始化
        System.out.println(new B[0]);
        // 4. loadClass方法，不会初始化类 B，但会加载 B、A
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        cl.loadClass("com.cjj.init.B");

        /*
        会初始化的时候
         */
        // 1. 首次访问这个类的静态变量或静态方法时
        System.out.println(A.a);
        // 2. 子类初始化，如果父类还没初始化，父类先初始然后子类再初始化
        System.out.println(B.c);
        // 3. 子类访问父类静态变量，只触发父类初始化
        System.out.println(B.a);
        // 4. Class.forName会初始化
        Class.forName("com.cjj.init.B");
    }
}
class A {
    static int a = 0;
    static {
        System.out.println("a init");
    }
}
class B extends A {
    final static double b = 5.0;
    static boolean c = false;
    static {
        System.out.println("b init");
    }
}
```

------

## 类加载器

**1.**`**BootstrapClassLoader**`**(启动类加载器)**：最顶层的加载类，由 C++实现，通常表示为 null，并且没有父级，主要用来加载 JDK 内部的核心类库（ `%JAVA_HOME%/lib`目录下的 `rt.jar`、`resources.jar`、`charsets.jar`等 jar 包和类）以及被 `-Xbootclasspath`参数指定的路径下的所有类。

**2.**`**ExtensionClassLoader**`**(扩展类加载器)**：主要负责加载 `%JRE_HOME%/lib/ext` 目录下的 jar 包和类以及被 `java.ext.dirs` 系统变量所指定的路径下的所有类。

**3.**`**AppClassLoader**`**(应用程序类加载器)**：面向我们用户的加载器，负责加载当前应用 classpath 下的所有 jar 包和类

### 双亲委派机制

自底向上查找该类是否被加载，如果加载就返回，如果都没加载，就交给父类加载，如果父类为空，则调用启动类加载器来加载该类，自顶向下尝试加载该类

### 自定义类加载器

1. 继承 `ClassLoader` 父类
2. 要遵从双亲委派机制，重写 `findClass` 方法

- 注意不是重写 `loadClass` 方法，否则不会走双亲委派机制
- 读取类文件的字节码
- 调用父类的 `defineClass` 方法来加载类
- 使用者调用该类加载器的 `loadClass` 方法

```
class MyClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        String path = "D:\\temp\\bbb\\" + name + ".class";

        try {
            ByteArrayOutputStream os = new ByteArrayOutputStream();
            Files.copy(Paths.get(path), os);
            byte[] bytes = os.toByteArray();
            //字节数组->class对象
           return  defineClass(bytes, 0, bytes.length);
        } catch (IOException e) {
            e.printStackTrace();
            throw new ClassNotFoundException("类文件没找到");
        }
    }
}
```

### 打破双亲委派机制

1.重写 `ClassLoader`类的 `loadClass()`方法

2.线程上下文类加载器

`JDBC`通过启动类加载器加载`DriverManger`后，又通过线程上下文类加载器(启动类加载器)加载Jar包的MySQL驱动(一个线程创建完后，虚拟机会将应用程序类加载器放入线程上下文中，其他线程可以通过

getContextClassLoader()方法 获取它)