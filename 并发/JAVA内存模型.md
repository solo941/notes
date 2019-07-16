# Java内存模型（JMM）

不同于Java内存结构，内存模型指一种符合内存模型规范的，屏蔽了各种硬件和操作系统的访问差异的，保证Java程序在各种平台下对内存的访问能得到一致结果的机制与规范。解决多线程通过共享内存通信时，存在的原子性，可见性和有序性问题。首先谈一下这三种性质：

## 原子性

`Java` 的原子性就和数据库事务的原子性差不多，一个操作中要么全部执行成功或者失败。

 `JMM` 只是保证了基本的原子性，但类似于 `i++` 之类的操作，看似是原子操作，其实里面涉及到:

- 获取 i 的值。
- 自增。
- 再赋值给 i。

 在执行完读改之后，时间片耗完，线程就会被要求放弃CPU，并等待重新调度，此时，读改写不是原子性操作。因此这三步操作，所以想要实现 `i++` 这样的原子操作就需要用到 `synchronized` 或者是 `lock` 进行加锁处理。

 如果是基础类的自增操作可以使用 `AtomicInteger.incrementAndGet()` 来实现(其本质是利用了 `CPU` 级别的 的 `CAS` 指令来完成的)。

```java
public final int incrementAndGet() {
        return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
    }
```

```java
public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```

`getIntVolatile()`方法，返回被`volatile`修饰的关键字，保证可见性;`CAS`和`getIntVolatile()`都是native方法。

## 可见性

在多核CPU，多线程的场景中，每个核都至少有一个`L1 `缓存。多个线程访问进程中的某个共享内存，且这多个线程分别在不同的核心上执行，则每个核心都会在各自的`caehe`中保留一份共享内存的缓冲。由于多核是可以并行的，可能会出现多个线程同时写各自的缓存的情况，关于同一个数据的缓存内容可能不一致。

Java中的volatile关键字提供了一个功能，那就是被其修饰的变量在被修改后可以立即同步到主内存，被其修饰的变量在每次是用之前都从主内存刷新。因此，可以使用volatile来保证多线程操作时变量的可见性。

`synchronized`和加锁也能能保证可见性，实现原理就是在释放锁之前其余线程是访问不到这个共享变量的。但是和 `volatile` 相比开销较大。

## 有序性

除了引入了时间片以外，由于处理器优化和指令重排等，CPU还可能对输入代码进行乱序执行，比如load->add->save 有可能被优化成load->save->add 。这就是有序性问题。举个例子：

```java
int a = 100 ; //1
int b = 200 ; //2
int c = a + b ; //3
```

正常情况下的执行顺序应该是 `1>>2>>3`。但是有时 `JVM` 为了提高整体的效率会进行指令重排导致执行的顺序可能是 `2>>1>>3`。但是 `JVM` 也不能是什么都进行重排，是在保证最终结果和代码顺序执行结果一致的情况下才可能进行重排。重排在单线程中不会出现问题，但在多线程中会出现数据不一致的问题。

使用volatile关键字会禁止指令重排。具体如下图所示：

以单例模式的线程安全懒汉式为例：

```java
public class Singleton {
        private static volatile Singleton singleton;

        private Singleton() {
        }

        public static Singleton getInstance() {
            if (singleton == null) {
                synchronized (Singleton.class) {
                    if (singleton == null) {
                        singleton = new Singleton();
                    }
                }
            }
            return singleton;
        }

    }
```

这里的 `volatile` 关键字主要是为了防止指令重排。 如果不用 `volatile` ，`singleton = new Singleton();`，这段代码其实是分为三步：

- 分配内存空间。(1)
- 初始化对象。(2)
- 将 `singleton` 对象指向分配的内存地址。(3)

 加上 `volatile` 是为了让以上的三步操作顺序执行，反之有可能第三步在第二步之前被执行就有可能导致某个线程拿到的单例对象还没有初始化。加锁是为了保证原子性，保证`singleton`指向分配的内存。

总结一下，多CPU多级缓存导致的可见性问题、CPU时间片机制导致的原子性问题、以及处理器优化和指令重排导致的有序性问题等。

