# AtomicLong原子类原理剖析

JUC并发包提供了一系列的原子操作类，这些类都是使用非阻塞算法CAS实现的，相比锁实现原子性操作这在性能上有很大提高。主要有AtomicInteger，AtomicBoolean，AtomicLong等操作类。



## 1. AtomicLong

```java
// 通过BootStarp类加载器进行加载
public class AtomicLong extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // 获取unsafe实例
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // 存放变量value的偏移量
    private static final long valueOffset;
    // 判断JVM是否支持Long类型无锁CAS
    static final boolean VM_SUPPORTS_LONG_CAS = VMSupportsCS8();
	
    static {
        try {
            // 获取value在AtomicLong中的偏移量
            valueOffset = unsafe.objectFieldOffset
                (AtomicLong.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
	// 实际变量值
    private volatile long value;
    
    public AtomicLong(long initialValue) {
        value = initialValue;
    }
    ......
}

```

### 1.1 递增递减操作

```java
// 设置原始值+1，返回值为原始值
public final long getAndIncrement() {
    return unsafe.getAndAddLong(this, valueOffset, 1L);
}
// 设置原始值-1，返回为原始值
public final long getAndDecrement() {
        return unsafe.getAndAddLong(this, valueOffset, -1L);
}
// 设置原始值+1，返回递增后的值
public final long incrementAndGet() {
    return unsafe.getAndAddLong(this, valueOffset, 1L) + 1L;
}
// 设置原始值-1，返回递减后的值
public final long decrementAndGet() {
    return unsafe.getAndAddLong(this, valueOffset, -1L) - 1L;
}
/**
 * @param var1 AtomicLong变量实例
 * @param var2 value变量在AtomicLong中的偏移量
 * @param var4 加减的值
 */
public final long getAndAddLong(Object var1, long var2, int var4) {
    int var5;
    do {
        // 获取变量当前内存中的值
        var5 = this.getLongVolatile(var1, var2);
        // 判断入参与内存中的值是否一致，一致则将值进行修改
    } while(!this.compareAndSwapLong(var1, var2, var5, var5 + var4));
    return var5;
}
// 如果原子变量中的value值等于expect，则使用update更新该值，并返回true，否则返回false
unsafe.compareAndSwapLong(this,valueOffset,expect,update);
```

// TODO 具体的CAS步骤，待画图表示



AtomicLong通过CAS提供了非阻塞的原子性操作，相比阻塞算法的同步器性能已经很好了，但是在高并发下大量线程会同时竞争更新一个原子变量，由于同时只有一个线程能够CAS操作成功，就会造成大量线程竞争失败，通过自旋尝试CAS，就白白的浪费了CPU资源。如下图所示：

![](./image\原子变量AtomicLong多线程竞争.png)

既然瓶颈在于过多线程同时去竞争一个变量的更新而产生，那么如果把一个变量分解成多个变量，让同样多的线程去竞争多个资源就能解决这个性能问题，在JDK8中新增的LongAdder类就是这么设计的。



