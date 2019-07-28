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

### Hashcode计算

和`HashMap`的最大的不同在于，`ThreadLocalMap`结构非常简单，没有next引用，也就是说`ThreadLocalMap`中解决Hash冲突的方式并非链表的方式，而是采用线性探测的方式。`ThreadLocalMap` 的 key 是 `ThreadLocalHashCode`，调用 `nextHashCode() `，具体运算如下：

```java
  private final int threadLocalHashCode = nextHashCode();

    //共享变量，所有的 ThreadLocal 共享一个 AtomicInteger，在其基础上 CAS 增加。
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

这里我们对ThreadLocal的hashcode产生过程进行了模拟

```Java
public class ThreadLocalTest {
    public static void main(String[] args) {
        printAllSlot(8);
        printAllSlot(16);
        printAllSlot(32);
    }

    static void printAllSlot(int len) {
        System.out.println("********** len = " + len + " ************");
        for (int i = 1; i <= 64; i++) {
            ThreadLocal<String> t = new ThreadLocal<>();
            int slot = getSlot(t, len);
            System.out.print(slot + " ");
            if (i % len == 0)
                System.out.println(); // 分组换行
        }
    }

    /**
     * 获取槽位
     *
     * @param t ThreadLocal
     * @param len 模拟map的table的length
     * @throws Exception
     */
    static int getSlot(ThreadLocal<?> t, int len) {
        //相邻线程相差HASH_INCREMENT
        int hash = getHashCode(t);
        //模拟了ThreadLocalMap获取hashcode的方法
        return hash & (len - 1);
    }

    /**
     * 反射获取 threadLocalHashCode 字段，因为其为private的
     */
    static int getHashCode(ThreadLocal<?> t) {
        Field field;
        try {
            field = t.getClass().getDeclaredField("threadLocalHashCode");
            field.setAccessible(true);
            return (int) field.get(t);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return 0;
    }
```

槽位结果如下：

```
********** len = 8 ************
2 1 0 7 6 5 4 3 
2 1 0 7 6 5 4 3 
2 1 0 7 6 5 4 3 
2 1 0 7 6 5 4 3 
2 1 0 7 6 5 4 3 
2 1 0 7 6 5 4 3 
2 1 0 7 6 5 4 3 
2 1 0 7 6 5 4 3 
********** len = 16 ************
10 1 8 15 6 13 4 11 2 9 0 7 14 5 12 3 
10 1 8 15 6 13 4 11 2 9 0 7 14 5 12 3 
10 1 8 15 6 13 4 11 2 9 0 7 14 5 12 3 
10 1 8 15 6 13 4 11 2 9 0 7 14 5 12 3 
********** len = 32 ************
10 17 24 31 6 13 20 27 2 9 16 23 30 5 12 19 26 1 8 15 22 29 4 11 18 25 0 7 14 21 28 3 
10 17 24 31 6 13 20 27 2 9 16 23 30 5 12 19 26 1 8 15 22 29 4 11 18 25 0 7 14 21 28 3 
```

### 解决hash冲突

```Java
private void set(ThreadLocal<?> key, Object value) {

          
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    // key = null，说明 key 已经被回收了，进入替换方法
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
    		//size表示Entry的数量
            int sz = ++size;
    		//过期值清理
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```

`ThreadLocalMap`解决Hash冲突的方式就是简单的步长加1或减1，寻找下一个相邻的位置。与我们常用的Map不同，`java`里大部分Map都是用链表发解决hash冲突的，而 `ThreadLocalMap` 采用的是开发地址法。

这里对比一下两种方法：

**开放地址法**：容易产生堆积问题；不适于大规模的数据存储；散列函数的设计对冲突会有很大的影响；插入时可能会出现多次冲突的现象，删除的元素是多个冲突元素中的一个，需要对后面的元素作处理，实现较复杂；结点规模很大时会浪费很多空间；

**链地址法**：处理冲突简单，且无堆积现象，平均查找长度短；链表中的结点是动态申请的，适合构造表不能确定长度的情况；相对而言，拉链法的指针域可以忽略不计，因此较开放地址法更加节省空间。插入结点应该在链首，删除结点比较方便，只需调整指针而不需要对其他冲突元素作调整。

避免hash冲突的可行方法：每个线程只存一个变量，这样的话所有的线程存放到map中的Key都是相同的`ThreadLocal`，如果一个线程要保存多个变量，就需要创建多个`ThreadLocal`。

### 弱引用引起的`OOM`

由于ThreadLocalMap的key是弱引用，而Value是强引用。这就导致了一个问题，ThreadLocal在没有外部对象强引用时，发生GC时弱引用Key会被回收，而Value不会回收，如果创建ThreadLocal的线程一直持续运行，那么这个Entry对象中的value就有可能一直得不到回收，发生内存泄露。

解决方法：在调用`ThreadLocal`的get()、set()方法时完成后再调用remove方法，将Entry节点和Map的引用关系移除。

```Java
ThreadLocal<Session> threadLocal = new ThreadLocal<Session>();
try {
    threadLocal.set(new Session(1, "Misout的博客"));
    // 其它业务逻辑
} finally {
    threadLocal.remove();
}
```

## 应用场景:Hibernate的session获取场景

```java
private static final ThreadLocal<Session> threadLocal = new ThreadLocal<Session>();

//获取Session
public static Session getCurrentSession(){
    Session session =  threadLocal.get();
    //判断Session是否为空，如果为空，将创建一个session，并设置到本地线程变量中
    try {
        if(session ==null&&!session.isOpen()){
            if(sessionFactory==null){
                rbuildSessionFactory();// 创建Hibernate的SessionFactory
            }else{
                session = sessionFactory.openSession();
            }
        }
        threadLocal.set(session);
    } catch (Exception e) {
        // TODO: handle exception
    }

    return session;
}
```

每个线程访问数据库都应当是一个独立的Session会话，如果多个线程共享同一个Session会话，有可能其他线程关闭连接了，当前线程再执行提交时就会出现会话已关闭的异常，导致系统异常。

要特别注意的一点，`ThreadLocal`对象使用static修饰，有一个好处就是，由于`ThreadLocal`有强引用在，那么在`ThreadLocalMap`里对应的Entry的键会永远存在，那么执行remove的时候就可以正确进行定位到并且删除。