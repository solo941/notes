## `ThreadLocal`

## 定义

`ThreadLocal` 用一种存储变量与线程绑定的方式，在每个线程中用自己的 `ThreadLocalMap` 安全隔离变量，如为每个线程创建一个独立的数据库连接。在很多场景也被用来实现线程参数传递，如 Spring 的 `RequestContextHolder`。

ThreadLocal归纳下来就2类用途：

- 保存线程上下文信息，在任意需要的地方可以获取
- 线程安全的，避免某些情况需要考虑线程安全必须同步带来的性能损失。`ThreadLocal`对象建议使用static修饰，针对一个线程所有操作共享。
- 每个线程往`ThreadLoca`l中读写数据是线程隔离，互相之间不会影响的，所以`ThreadLocal`无法解决共享对象的更新问题！

下图为`ThreadLocal`的内部结构图



## 源码解析

`ThreadLocal`类提供如下几个核心方法：

```java
public T get()
public void set(T value)
public void remove()
```

### get()

```java
public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            //ThreadLocalMap获取ThreadLocal对应的Entry
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

get()方法用于获取当前线程的副本变量值

### set()

```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

设置Thread对应的`ThreadLocal`的value，第一次调用时需要 `creatMap`。

### **remove()** 

```Java
public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null)
             m.remove(this);
     }
```

这里罗列了 `ThreadLocal` 的几个public方法，其实所有工作最终都落到了 `ThreadLocalMap` 的头上，`ThreadLocal `仅仅是从当前线程取到 `ThreadLocalMap` 而已，具体执行，请看下面对 `ThreadLocalMap` 的分析。

## ThreadLocalMap

`ThreadLocalMap`是`ThreadLocal`的内部类，没有实现Map接口，用独立的方式实现了Map的功能，其内部的Entry也独立实现。

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

在`ThreadLocalMap`中，也是用Entry来保存K-V结构数据的。Entry继承自`WeakReference`（弱引用，生命周期只能存活到下次`GC`前），但只有Key是弱引用类型的(key只能是`ThreadLocal`对象)，Value并非弱引用。

### 解决Hash冲突

和HashMap的最大的不同在于，ThreadLocalMap结构非常简单，没有next引用，也就是说ThreadLocalMap中解决Hash冲突的方式并非链表的方式，而是采用线性探测的方式。ThreadLocalMap 的 key 是 ThreadLocalHashCode，调用 nextHashCode() ，具体运算如下：

```java
  private final int threadLocalHashCode = nextHashCode();

    /**
     * The next hash code to be given out. Updated atomically. Starts at
     * zero.
     */
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */
    private static final int HASH_INCREMENT = 0x61c88647;

    /**
     * Returns the next hash code.
     */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
```

