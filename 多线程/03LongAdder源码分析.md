# LongAdder源码分析

## 开篇

![](/image/原子变量LongAdder多线程竞争.png)

如上图所示，LongAdder在内部维护多个Cell变量，每个Cell里面有个初始值为0的long型变量，在同等并发情况下，竞争单个变量的线程数减少，变相的减少了争夺共享资源的并发量。下面我们通过源码看看内部是如何实现的。

我们从以下几个话题来分析以下LongAdder是如何实现的？

（1）LongAdder结构是怎么样的？

（2）当前线程应该访问Cell数组里面的哪一个Cell元素？

（3）如何初始化Cell数组？

（4）Cell数组如何扩容？

（5）线程访问分配的Cell元素时又冲突后如何处理？

（6）如何保证线程操作被分配的Cell元素的原子性？

## 源码分析

### 类图

首先看下LongAdder的类图结构，继承了Striped64类。内部维护着3个变量：base，cells，cellsBusy。LongAdder的真实值是base的值加上Cell数组里面所有Cell元素中的value值的累加。

![](/image/LongAdder类图.png)

```java
// cell元素数组，非空时数量为2的n次方    
transient volatile Cell[] cells;
// 基础值，主要在没有竞争的情况下使用，通过CAS更新
transient volatile long base;
// 用来实现自旋（CAS方式），当进行初始化或扩容Cells数组，新增Cell元素的锁标识
transient volatile int cellsBusy;
```

base是个基础值，默认0；cellsBusy用来实现自旋，状态值只有0和1，当创建Cell元素，扩容Cell数组或者初始化Cell数组时，使用CAS操作该变量来保证同时只有一个线程可进行其中之一的操作。(回答问题1)

### Cell类

```java
// 使用注解为了避免伪共享
@sun.misc.Contended static final class Cell {
    // 使用volatile，是因为线程在操作value变量时没有锁，保证变量的内存可见性
    volatile long value;
    Cell(long x) { value = x; }
    // cas函数，使用CAS操作，保证当前线程更新时被分配的Cell元素中的value值的原子性（问题6）
    final boolean cas(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long valueOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> ak = Cell.class;
            valueOffset = UNSAFE.objectFieldOffset
                (ak.getDeclaredField("value"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

### sum方法

```java
public long sum() {
    Cell[] as = cells; Cell a;
    // 累加base
    long sum = base;
    if (as != null) {
        // 遍历cell进行累加
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

我们从上面的sum方法中可以看到，内部通过累加所有Cell内部的value值后再累加base。但在计算综合时没有对Cell数组进行加锁，所以在累加的过程中，可能有其他线程对Cell中的值进行修改，也可能对数组进行了扩容，所以sum返回的值并不精确。返回的值并不是一个原子快照。

### reset方法

```java
public void reset() {
    Cell[] as = cells; Cell a;
    // 将base重置为0
    base = 0L;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                // cell元素值重置为0
                a.value = 0L;
        }
    }
}
```

reset方法为重置操作，将base和cell数组中的元素值都重置为0

### sumThenReset方法

```java
public long sumThenReset() {
    Cell[] as = cells; Cell a;
    long sum = base;
    // 重置base为0
    base = 0L;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null) {
                sum += a.value;
                // 累加过后，重置cell元素值为0
                a.value = 0L;
            }
        }
    }
    return sum;
}
```

sumTenReset是sum的改造版本，在进行累加后对当前的base和Cell的值重置为0，多线程调用仍会存在问题，第一个调用线程清空Cell的值，后一个线程调用时累加的值都是0。

### add方法

```java
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    // cells为空，则当前在base上进行累加  （1）
    if ((as = cells) != null || !casBase(b = base, b + x)) { 
        // cells不为空或者cas设置base值失败，执行下面代码
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 || // (2)
            (a = as[getProbe() & m]) == null ||     // getProbe()获取随机值  (3)
            !(uncontended = a.cas(v = a.value, v + x))) // 数组下标元素不为null，执行cas设置值
            // 当前线程映射的元素不存在，或者执行cas操作失败，执行下面代码
            longAccumulate(x, null, uncontended);  // (4)
    }
}
final boolean casBase(long cmp, long val) {
    return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
}
static final int getProbe() {
    return UNSAFE.getInt(Thread.currentThread(), PROBE);
}
```

（问题2）当前线程应该访问Cell数组里面的哪一个Cell元素呢，从上面代码我们能看到是通过getProbe() & m 进行计算得到的数组索引位置。m：数组元素个数-1，getProbe()：获取当前	线程中变量threadLocalRandomProbe的值，初始为0

### longAccumulate方法

```java
final void longAccumulate(long x, LongBinaryOperator fn,
                          boolean wasUncontended) {
    int h;
    // 初始化当前线程的变量threadLocalRandomProbe的值  （1）
    if ((h = getProbe()) == 0) {
        ThreadLocalRandom.current(); // force initialization
        h = getProbe();
        wasUncontended = true;
    }
    // 碰撞标识，最后一个槽不为空才为true
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        Cell[] as; Cell a; int n; long v;
        // Cell数组不为null且长度大于0
        if ((as = cells) != null && (n = as.length) > 0) {  // (2)
            // 计算要访问的Cell数组下标，对应下标的元素值为null
            if ((a = as[(n - 1) & h]) == null) {            // (3)
                // 锁可用
                if (cellsBusy == 0) {       // Try to attach new Cell
                    // 新建一个Cell元素
                    Cell r = new Cell(x);   // Optimistically create
                    // 获取锁
                    if (cellsBusy == 0 && casCellsBusy()) {   // (4)
                        boolean created = false;
                        try {               // Recheck under lock
                            Cell[] rs; int m, j;
                            if ((rs = cells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {  // (m-1)&h计算数组下标p，且元素为null，则设置cell数组下标p的元素为r（新建的）    (4.1)
                                rs[j] = r;   // 往数组中加入新Cell
                                created = true;
                            }
                        } finally {
                            // 释放锁  (4.2)
                            cellsBusy = 0;  
                        }
                        // 创建成功，跳出循环
                        if (created)
                            break;
                        continue;           // Slot is now non-empty
                    }
                }
                // 标记碰撞
                collide = false;
            }
            else if (!wasUncontended)       // CAS already known to fail
                wasUncontended = true;      // Continue after rehash
            // 当前Cell存在，执行CAS更新value值 (5)
            else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                         fn.applyAsLong(v, x))))
                break;
            // 当前Cell元素个数大于CPU个数 （6）
            else if (n >= NCPU || cells != as)
                collide = false;            // At max size or stale
            // 是否有冲突   (7)
            else if (!collide)
                collide = true;
            // 当前元素个数没有达到CPU个数，（6）,（7）都不满足，进行扩容  (8)
            else if (cellsBusy == 0 && casCellsBusy()) {
                try {
                    if (cells == as) {      // Expand table unless stale
                        // 定义新数组，容量扩大为原来的2倍   (8.1)
                        Cell[] rs = new Cell[n << 1];
                        // 将旧数组数据迁移至新数组         (8.2)
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        cells = rs;
                    }
                } finally {
                    // 扩容完毕，释放锁     (8.3)
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            // 重新计算hash值，找到一个空闲的Cell   (9)
            h = advanceProbe(h);
        }
        // 初始化  (10)
        else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
            boolean init = false;
            try {                           // Initialize table
                if (cells == as) {
                    // 数组初始化长度为2  (10.1)
                    Cell[] rs = new Cell[2];
                    rs[h & 1] = new Cell(x); // 找到一个Cell，(10.2)
                    cells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0;  // 重置标记
            }
            if (init) // 初始化完成，退出循环
                break;
        }
        // 竞争cell失败，回过头重新尝试去设置base   (11)
        else if (casBase(v = base, ((fn == null) ? v + x :
                                    fn.applyAsLong(v, x))))
            break;                          // Fall back on using base
    }
}
final boolean casCellsBusy() {
    return UNSAFE.compareAndSwapInt(this, CELLSBUSY, 0, 1);
}
static final int advanceProbe(int probe) {
    probe ^= probe << 13;   // xorshift
    probe ^= probe >>> 17;
    probe ^= probe << 5;
    UNSAFE.putInt(Thread.currentThread(), PROBE, probe);
    return probe;
}
```

这个方法是用来处理Cell数组初始化和扩容的地方。

## 主要流程

下面我们用图形化的方式来更直观的看一下，LongAdder的执行流程

### 初始化Cell流程

代码步骤10

![1584189677693](/image\LongAdder初始化1.png)

T1和T2线程同时执行add操作，线程T1执行casBase成功，对base进行累加，线程T2执行casBase失败，cell为null执行初始化流程（代码标注10）

若执行casCellsBusy，初始化数组长度为2，通过 h&1 计算下标，新建Cell并赋值

若执行失败，则走步骤（11）

### 新增Cell流程

代码步骤2

![1584191193944](E:\JavaKnowledge\多线程\image\LongAdder新增Cell.png)

线程T4执行add操作，执行casBase失败后，根据getProbe() & m计算Cell数组下标，当前下标元素为null，且cellsBusy为0（没有其他线程正在执行初始化，新增Cell，扩容的流程），则执行casCellsBusy，设置标识cellsBusy为1，新建Cell并添加到数组当中。

### 扩容流程

代码步骤8，在6,7步骤都不满足的情况下才执行

![1584193734485](E:\JavaKnowledge\多线程\image\LongAdder冲突.png)

线程T5执行add操作，执行casBase失败后，同样根据getProbe() & m计算Cell数组下标，当前下标元素为null，发现cellsBusy为1，与T4线程产生冲突，将cell数组扩容为原来数量的两倍，把原有数组中的元素复制到新的数组当中。

## 总结

本文讲述了LongAdder源码和流程，在此总结下开头提出的几个问题

1、当前线程访问Cell数组中的哪个Cell元素？

通过执行getProbe() & m进行计算数组下标，其中m是数组元素个数-1，getProbe()则用于获取当前线程变量threadLocalRandomProbe的值，这个值一开始是0

2、

