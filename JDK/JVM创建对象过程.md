# Jvm创建对象的过程

如果对象所属的类没有加载进内存，需要先通过类的全限定名来加载类信息，加载并初始化类完成后，再进行对象的创建工作。

## 类加载的过程

使用双亲委派模型进行类的加载。

双亲委派模型：一个类加载器首先将类加载请求转发到父类加载器，只有当父类加载器无法完成时才尝试自己加载。

好处：使得 Java 类随着它的类加载器一起具有一种带有优先级的层次关系，从而使得基础类得到统一，当程序中出现多个限定名相同的类时，类加载器在执行加载时，始终只会加载其中的某一个类。

### 类加载的五个主要阶段

包含了加载、验证、准备、解析和初始化这 5 个阶段。

#### 加载

 由类加载器负责根据一个类的全限定名来读取此类的二进制字节流到JVM内部，并存储在运行时内存区的方法区，然后将其转换为一个与目标类型对应的java.lang.Class对象实例，作为方法区中该类各种数据的访问入口。

18.8中，取消永久代，方法区存放于元空间中，元空间并不在虚拟机中，而是使用本地内存。因此，默认情况下，元空间的大小仅受本地内存限制。

#### 验证

格式验证：验证是否符合class文件规范

语义验证：检查一个被标记为final的类型是否包含子类；检查一个类中的final方法是否被子类进行重写；确保父类和子类之间没有不兼容的一些方法声明（比如方法签名相同，但方法的返回值不同）。
操作验证：在操作数栈中的数据必须进行正确的操作，对常量池中的各种符号引用执行验证（通常在解析阶段执行，检查是否可以通过符号引用中描述的全限定名定位到指定类型上，以及类成员信息的访问修饰符是否允许访问等）

#### 准备

为类中的所有静态变量分配内存空间，并为其设置一个初始值（由于还没有产生对象，实例变量不在此操作范围内）被final修饰的static变量（常量)会直接赋值；

#### 解析

将常量池中的符号引用转为直接引用（得到类或者字段、方法在内存中的指针或者偏移量，以便直接调用该方法）。解析需要静态绑定的内容，动态绑定可以在初始化之后再执行。

所有不会被重写的方法和域都会被静态绑定

#### 初始化

虚拟机执行类构造器 \<clinit>() 方法的过程,为静态变量赋值并执行static代码块。

因为子类存在对父类的依赖，所以类的加载顺序是先加载父类后加载子类，初始化也一样。最终，方法区会存储当前类类信息，包括类的**静态变量**、**类初始化代码**（**定义静态变量时的赋值语句** 和 **静态初始化代码块**）、**实例变量定义**、**实例初始化代码**（**定义实例变量时的赋值语句实例代码块**和**构造方法**）和**实例方法**，还有**父类的类信息引用。**

虚拟机会保证一个类的 \<clinit>() 方法在多线程环境下被正确的加锁和同步，如果多个线程同时初始化一个类，只会有一个线程执行这个类的 \<clinit>() 方法，其它线程都会阻塞等待，直到活动线程执行\<clinit>() 方法完毕。

## 创建对象

**1、在堆区分配对象需要的内存**

　　分配的内存包括本类和父类的所有实例变量，但不包括任何静态变量

**2、对所有实例变量赋默认值**

　　将方法区内对实例变量的定义拷贝一份到堆区，然后赋默认值

**3、执行实例初始化代码**

　　初始化顺序是先初始化父类再初始化子类，**初始化时先执行实例代码块然后是构造方法**

4、如果有类似于Child c = new Child()形式的符号引用的话，在栈区定义Child类型引用变量c，然后将堆区对象的地址赋值给它。

#### 方法调用

通过实例引用调用实例方法的时候，先从方法区中对象的实际类型信息找，找不到的话再去父类类型信息中找。

 如果继承的层次比较深，要调用的方法位于比较上层的父类，则调用的效率是比较低的，因为每次调用都要经过很多次查找。这时候大多系统会采用一种称为**虚方法表**的方法来优化调用的效率。

 虚方法表，就是在类加载的时候，为每个类创建一个表，这个表包括该类的对象所有动态绑定的方法及其地址，包括父类的方法，但一个方法只有一条记录，子类重写了父类方法后只会保留子类的。当通过对象动态绑定方法的时候，只需要查找这个表就可以了，而不需要挨个查找每个父类。

## 面试题

```java

class Bowl {
	Bowl(int marker) {
		System.out.println("Bowl(" + marker + ")");
	}
}
class Tableware {
	static Bowl bowl7 = new Bowl(7);
	static {

		System.out.println("Tableware静态代码块");

	}
	Tableware() {
		System.out.println("Tableware构造方法");	
	}
	Bowl bowl6 = new Bowl(6);	
}

class Table extends Tableware {
	{
		System.out.println("Table非静态代码块_1");
	}

	Bowl bowl5 = new Bowl(5);	// 9
	{
		System.out.println("Table非静态代码块_2");	
	}

	static Bowl bowl1 = new Bowl(1);	
	static {
		System.out.println("Table静态代码块");
	}
	
	Table() {
		System.out.println("Table构造方法");
	}

	static Bowl bowl2 = new Bowl(2);	

}

class Cupboard extends Tableware {
	Bowl bowl3 = new Bowl(3);	
	static Bowl bowl4 = new Bowl(4);	
	Cupboard() {
		System.out.println("Cupboard构造方法");
	}
	
	void otherMethod(int marker) {
		System.out.println("otherMethod(" + marker + ")");
	}
	static Bowl bowl5 = new Bowl(5);	
}

public class StaticInitialization {
	public static void main(String args[]) {	
		System.out.println("main()");
		cupboard.otherMethod(1);	
	}
	static Table table = new Table();
	static Cupboard cupboard = new Cupboard();
}

```

题目解析：在类的内部，变量定义的先后顺序决定了初始化顺序。即使**变量定义散布于方法定义之间，它们仍旧会在任何方法（包括构造方法）被调用之前得到初始化；**

静态初始化只有在必要时刻才进行，例如：类里面的静态变量，只有当类被调用时才会初始化（执行），并且静态变量不会再次被初始化（执行），即静态变量只会初始化（执行）一次；

当有父类时，完整的初始化顺序为：父类静态变量（静态代码块）->子类静态变量（静态代码块）->父类非静态变量（非静态代码块）->父类构造器 ->子类非静态变量（非静态代码块）->子类构造器 。

运行结果：

```
Bowl(7)
Tableware静态代码块
Bowl(1)
Table静态代码块
Bowl(2)
Bowl(6)
Tableware构造方法
Table非静态代码块_1
Bowl(5)
Table非静态代码块_2
Table构造方法
Bowl(4)
Bowl(5)
Bowl(6)
Tableware构造方法
Bowl(3)
Cupboard构造方法
main()
otherMethod(1)
```

初始化顺序：



## 参考文章

[java new一个对象的过程中发生了什么](https://www.cnblogs.com/JackPn/p/9386182.html)

[**Java 虚拟机.md**](https://github.com/CyC2018/CS-Notes/blob/master/notes/Java%20虚拟机.md#%E5%8F%8C%E4%BA%B2%E5%A7%94%E6%B4%BE%E6%A8%A1%E5%9E%8B)

[深入探讨 Java 类加载器](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/index.html#code6)

[**JDK8中JVM堆内存划分**](https://yq.aliyun.com/articles/446120)

