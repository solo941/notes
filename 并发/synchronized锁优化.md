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

总结：**每一个Java类，在被JVM加载的时候，JVM会给这个类创建一个instanceKlass，保存在方法区，用来在JVM层表示该Java类。当我们在Java代码中，使用new创建一个对象的时候，JVM会创建一个instanceOopDesc对象，这个对象头中包含了两部分信息，方法头以及元数据。对象头中有一些运行时数据，其中就包括和多线程相关的锁的信息。元数据其实维护的是指针，指向的是对象所属的类的instanceKlass。**

##  Java虚拟机的锁优化技术

