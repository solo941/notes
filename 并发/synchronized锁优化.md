## Synchronized的实现原理

Java中的每一个对象都可以作为锁。对于同步方法，锁是当前实例对象。对于静态同步方法，锁是当前对象的Class对象。对于同步方法块，锁是synchonized括号里配置的对象。

```java
public class SyncTest {

    private static double a = 1;

    public synchronized void plusNumber() {
        a++;
    }

    public void minusNumber() {
        System.out.println(a);
        synchronized (this) {
            a--;
        }
    }

    public synchronized static void divide() {

        a = a / 0.1;
    }

}
```

我们先来使用Javap来反编译以上代码，结果如下

```java
//同步方法
public synchronized void plusNumber();                                                                       
  descriptor: ()V                                                                                            
  flags: ACC_PUBLIC, ACC_SYNCHRONIZED                                                                        
  Code:                                                                                                      
    stack=4, locals=1, args_size=1                                                                           
       0: getstatic     #2                  // Field a:D                                                     
       3: dconst_1                                                                                           
       4: dadd                                                                                               
       5: putstatic     #2                  // Field a:D                                                     
       8: return                                                                                             
    LineNumberTable:                                                                                         
      line 12: 0                                                                                             
      line 13: 8                                                                                             

//同步块                                                                                                             
public void minusNumber();                                                                                   
  descriptor: ()V                                                                                            
  flags: ACC_PUBLIC                                                                                          
  Code:                                                                                                      
    stack=4, locals=3, args_size=1                                                                           
       0: getstatic     #3                  // Field java/lang/System.out:Ljava/io/PrintStream;              
       3: getstatic     #2                  // Field a:D                                                     
       6: invokevirtual #4                  // Method java/io/PrintStream.println:(D)V                       
       9: aload_0                                                                                            
      10: dup                                                                                                
      11: astore_1                                                                                           
      12: monitorenter                                                                                       
      13: getstatic     #2                  // Field a:D                                                     
      16: dconst_1                                                                                           
      17: dsub                                                                                               
      18: putstatic     #2                  // Field a:D                                                     
      21: aload_1                                                                                            
      22: monitorexit                                                                                        
      23: goto          31                                                                                   
      26: astore_2                                                                                           
      27: aload_1                                                                                            
      28: monitorexit                                                                                        
      29: aload_2                                                                                            
      30: athrow                                                                                             
      31: return                                                                                             
    Exception table:                                                                                         
       from    to  target type                                                                               
          13    23    26   any                                                                               
          26    29    26   any                                                                               
                                                                   
//静态同步方法                                                                                                             
public static synchronized void divide();                                                                    
  descriptor: ()V                                                                                            
  flags: ACC_PUBLIC, ACC_STATIC, ACC_SYNCHRONIZED                                                            
  Code:                                                                                                      
    stack=4, locals=0, args_size=0                                                                           
       0: getstatic     #2                  // Field a:D                                                     
       3: ldc2_w        #5                  // double 0.1d                                                   
       6: ddiv                                                                                               
       7: putstatic     #2                  // Field a:D                                                     
      10: return                                                                                             
    LineNumberTable:                                                                                         
      line 24: 0                                                                                             
      line 26: 10            
```

同步代码块是使用`monitorenter`和`monitorexit`指令实现的。可以把执行`monitorenter`指令理解为加锁，执行`monitorexit`理解为释放锁。 每个对象维护着一个记录着被锁次数的计数器。未被锁定的对象的该计数器为0，当一个线程获得锁（执行`monitorenter`）后，该计数器自增变为 1 ，当同一个线程再次获得该对象的锁的时候，计数器再次自增。当同一个线程释放锁（执行`monitorexit`指令）的时候，计数器再自减。当计数器为0的时候。锁将被释放，其他线程便可以获得锁。

同步方法和静态同步方法依靠的是方法修饰符上的ACC_SYNCHRONIZED实现。JVM根据该修饰符来实现方法的同步。当方法调用时，调用指令将会检查方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor。

## Java的对象模型

### oop-klass model

HotSpot JVM设计了一个`OOP-Klass Model`表示Java对象。OOP（`Ordinary Object Pointer`）指的是普通对象指针，而`Klass`用来描述对象实例的具体类型。

为什么HotSpot要设计一套`oop-klass model`呢？答案是：HotSopt JVM的设计者不想让每个对象中都含有一个`vtable`（虚函数表）

多态是面向对象的最主要的特性之一，是一种方法的动态绑定，实现运行时的类型决定对象的行为。

在C++中通过虚函数表的方式实现多态，每个包含虚函数的类都具有一个虚函数表（virtual table），在这个类对象的地址空间的最靠前的位置存有指向虚函数表的指针。在虚函数表中，按照声明顺序依次排列所有的虚函数。由于C++在运行时并不维护类型信息，所以在编译时直接在子类的虚函数表中将被子类重写的方法替换掉。

在Java中，在运行时会维持类型信息以及类的继承体系。每一个类会在方法区中对应一个数据结构用于存放类的信息，可以通过Class对象访问这个数据结构。其中，类型信息具有superclass属性指示了其超类，以及这个类对应的方法表（其中只包含这个类定义的方法，不包括从超类继承来的）。而每一个在堆上创建的对象，都具有一个指向方法区类型信息数据结构的指针，通过这个指针可以确定对象的类型。

### OOP

在Java程序运行过程中，每创建一个新的对象，在JVM内部就会相应地创建一个对应类型的OOP对象，这些OOPS在JVM内部有着不同的用途。

HotSpot虚拟机中，对象在内存中存储分为三块区域：对象头、实例数据和对齐填充。在虚拟机内部，一个Java对象对应一个instanceOopDesc的对象，该对象中有两个字段分别表示了对象头和实例数据。那就是\_mark和_metadata。

对象头默认存储结构如下：

![pic](https://github.com/solo941/notes/blob/master/并发/pics/20190510095530.png)

`_metadata`是一个共用体，其中`_klass`是普通指针，`_compressed_klass`是压缩类指针。在深入介绍之前，就要来到`oop-Klass`中的另外一个主角`klass`了。

```c++
union _metadata {
    wideKlassOop    _klass;
    narrowOop       _compressed_klass;
  } _metadata;
```

### Kclass

Klass向JVM提供两个功能：

- 实现语言层面的Java类（在Klass基类中已经实现）
- 实现Java对象的分发功能（由Klass的子类提供虚函数实现）

HotSopt JVM的设计者把对象一拆为二，分为klass和oop，其中oop的职能主要在于表示对象的实例数据，所以其中不含有任何虚函数。而klass为了实现虚函数多态，所以提供了虚函数表。

`_metadata`是一个共用体，其中`_klass`是普通指针，`_compressed_klass`是压缩类指针。这两个指针都指向**instanceKlass对象，它用来描述对象的具体类型。**

### instanceKlass

JVM在运行时，需要一种用来标识Java内部类型的机制。在HotSpot中的解决方案是：**为每一个已加载的Java类创建一个instanceKlass对象，用来在JVM层表示Java类。**

instanceKclass内部结构：

```
  //类拥有的方法列表
  objArrayOop     _methods;
  //描述方法顺序
  typeArrayOop    _method_ordering;
  //实现的接口
  objArrayOop     _local_interfaces;
  //继承的接口
  objArrayOop     _transitive_interfaces;
  //域
  typeArrayOop    _fields;
  //常量
  constantPoolOop _constants;
  //类加载器
  oop             _class_loader;
  //protected域
  oop             _protection_domain;
```

在JVM中，对象在内存中的基本存在形式就是oop。那么，对象所属的类，在JVM中也是一种对象，因此它们实际上也会被组织成一种oop，即klassOop。同样的，对于klassOop，也有对应的一个klass来描述，它就是klassKlass，也是klass的一个子类。在这种设计下，JVM对内存的分配和回收，都可以采用统一的方式来管理。oop-klass-klassKlass关系如图：

![pic](https://github.com/solo941/notes/blob/master/并发/pics/微信图片_20190821025851.jpg)

### 实例演示对象模型

```java
class Model
{
    public static int a = 1;
    public int b;

    public Model(int b) {
        this.b = b;
    }
}

public static void main(String[] args) {
    int c = 10;
    Model modelA = new Model(2);
    Model modelB = new Model(3);
}
```

HotSpot JVM中的OOP-Kclass模型如图：

![pic](https://github.com/solo941/notes/blob/master/并发/pics/微信图片_20190821031643.jpg)

总结：**每一个Java类，在被JVM加载的时候，JVM会给这个类创建一个instanceKlass，保存在方法区，用来在JVM层表示该Java类。当我们在Java代码中，使用new创建一个对象的时候，JVM会创建一个instanceOopDesc对象，这个对象头中包含了两部分信息，方法头以及元数据。对象头中有一些运行时数据，其中就包括和多线程相关的锁的信息。元数据其实维护的是指针，指向的是对象所属的类的instanceKlass。**

##  Java虚拟机的锁优化技术

要想把锁说清楚，一个重要的概念不得不提，那就是线程和线程的状态。对于线程来说，一共有五种状态，分别为：初始状态(New) 、就绪状态(Runnable) 、运行状态(Running) 、阻塞状态(Blocked) 和死亡状态(Dead) 。



### 自旋锁

`synchronized`的实现方式中使用`Monitor`进行加锁，这是一种互斥锁，为了表示他对性能的影响我们称之为重量级锁。这种互斥锁在互斥同步上对性能的影响很大，Java的线程是映射到操作系统原生线程之上的，如果要阻塞或唤醒一个线程就需要操作系统的帮忙，这就要从用户态转换到内核态，因此状态转换需要花费很多的处理器时间。

在程序中，Java虚拟机的开发工程师们在分析过大量数据后发现：共享数据的锁定状态一般只会持续很短的一段时间，为了这段时间去挂起和恢复线程其实并不值得。如果物理机上有多个处理器，可以让多个线程同时执行的话。我们就可以让后面来的线程“稍微等一下”，但是并不放弃处理器的执行时间，看看持有锁的线程会不会很快释放锁。这个“稍微等一下”的过程就是自旋。对于阻塞锁和自旋锁来说，都是要等待获得共享资源。但是阻塞锁是放弃了CPU时间，进入了等待区，等待被唤醒。而自旋锁是一直“自旋”在那里，时刻的检查共享资源是否可以被访问。

在 JDK 1.6 中引入了自适应的自旋锁。自适应意味着自旋的次数不再固定了，而是由前一次在同一个锁上的自旋次数及锁的拥有者的状态来决定。当线程数不停增加时，自旋锁性能下降明显，因为每个线程都需要执行，占用CPU时间。

### 锁消除

锁消除是指对于被检测出不可能存在竞争的共享数据的锁进行消除。在动态编译同步块的时候，JIT编译器可以借助一种被称为逃逸分析（Escape Analysis）的技术来判断同步块所使用的锁对象是否只能够被一个线程访问而没有被发布到其他线程。如果同步块所使用的锁对象通过这种分析被证实只能够被一个线程访问，那么JIT编译器在编译这个同步块的时候就会取消对这部分代码的同步。

```Java
public void f() {
   Object hollis = new Object();
   synchronized(hollis) {
       System.out.println(hollis);
   }
}
```

代码中对`hollis`这个对象进行加锁，但是`hollis`对象的生命周期只在`f()`方法中，并不会被其他线程所访问到，所以在JIT编译阶段就会被优化掉。

```Java
public void f() {
   Object hollis = new Object();
   System.out.println(hollis);
}
```

### 锁粗化

```Java
for(int i=0;i<100000;i++){  
   synchronized(this){  
       do();  
}
```

可以说，大部分情况下，减小锁的粒度是很正确的做法，只有一种特殊的情况下，会发生一种叫做锁粗化的优化。如果在一段代码中连续的对同一个对象反复加锁解锁，其实是相对耗费资源的，这种情况可以适当放宽加锁的范围，减少性能消耗。

```Java
synchronized(this){  
   for(int i=0;i<100000;i++){  
       do();  
}
```

### 轻量级锁

JDK 1.6 引入了偏向锁和轻量级锁，从而让锁拥有了四个状态：无锁状态（unlocked）、偏向锁状态（biasble）、轻量级锁状态（lightweight locked）和重量级锁状态（inflated）。

轻量级锁是相对于传统的重量级锁而言，它使用 CAS 操作来避免重量级锁使用互斥量的开销。对于绝大部分的锁，在整个同步周期内都是不存在竞争的，因此也就不需要都使用互斥量进行同步，可以先采用 CAS 操作进行同步，如果 CAS 失败了再改用互斥量进行同步。

当尝试获取一个锁对象时，如果锁对象标记为 0 01，说明锁对象的锁未锁定（unlocked）状态。此时虚拟机在当前线程的虚拟机栈中创建 Lock Record，然后使用 CAS 操作将对象的 Mark Word 更新为 Lock Record 指针。如果 CAS 操作成功了，那么线程就获取了该对象上的锁，并且对象的 Mark Word 的锁标记变为 00，表示该对象处于轻量级锁状态。

如果 CAS 操作失败了，虚拟机首先会检查对象的 Mark Word 是否指向当前线程的虚拟机栈，如果是的话说明当前线程已经拥有了这个锁对象，那就可以直接进入同步块继续执行，否则说明这个锁对象已经被其他线程线程抢占了。如果有两条以上的线程争用同一个锁，那轻量级锁就不再有效，要膨胀为重量级锁。

### 偏向锁

偏向锁的思想是偏向于让第一个获取锁对象的线程，这个线程在之后获取该锁就不再需要进行同步操作，甚至连 CAS 操作也不再需要。当锁对象第一次被线程获得的时候，进入偏向状态，标记为 1 01。同时使用 CAS 操作将线程 ID 记录到 Mark Word 中，如果 CAS 操作成功，这个线程以后每次进入这个锁相关的同步块就不需要再进行任何同步操作。当有另外一个线程去尝试获取这个锁对象时，偏向状态就宣告结束，此时撤销偏向（Revoke Bias）后恢复到未锁定状态或者轻量级锁状态。

可以这样理解，当只有一个线程获取对象时，使用偏向锁。如果有第二个线程尝试获取对象，锁会升级为轻量级锁。此时再来一个线程，就要膨胀为重量级锁。

下面举一个偏向锁升级的例子。假设线程1当前拥有偏向锁对象,线程2是需要竞争到偏向锁。

1. 线程2来竞争锁对象;
2. 判断当前对象头是否是偏向锁;
3. 判断拥有偏向锁的线程1是否还存在;
4. 线程1不存在,直接设置偏向锁标识为0(线程1执行完毕后,不会主动去释放偏向锁);
5. 使用cas替换偏向锁线程ID为线程2,锁不升级，仍为偏向锁;
6. 线程1仍然存在,暂停线程1；
7. 设置锁标志位为00(变为轻量级锁),偏向锁为0;
8. 从线程1的空闲monitor record中读取一条,放至线程1的当前monitor record中;
9. 更新mark word，将mark word指向线程1中monitor record的指针;
10. 继续执行线程1的代码;
11. 锁升级为轻量级锁;   
12. 线程2自旋来获取锁对象;

## 参考资料

[synchronized实现原理及锁优化](https://nicky-chen.github.io/2018/05/14/synchronized-principle/)

[Synchronized的实现原理（一）](https://mp.weixin.qq.com/s/637zy26W_fdeopX5dG8OKQ)

[深入理解多线程（二）—— Java的对象模型](https://mp.weixin.qq.com/s/mWWey3zngiqi-E40PR9U3A)

[深入理解多线程（五）—— Java虚拟机的锁优化技术](https://mp.weixin.qq.com/s/VDdsKp0uzmh_7vpD9lpLaA)

[Java对象头详解](https://www.jianshu.com/p/3d38cba67f8b)

[java 偏向锁怎么升级为轻量级锁](https://www.cnblogs.com/baxianhua/p/9391981.html)