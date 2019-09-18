# final关键字

## 基本用法

### 修饰类

声明类不允许被继承，final类中的所有成员方法都会被隐式地指定为final方法。

### 修饰方法

声明方法不能被子类重写。private 方法隐式地被指定为 final，如果在子类中定义的方法和基类中的一个 private 方法签名相同，此时子类的方法不是重写基类方法，而是在子类中定义了一个新的方法。

### **修饰变量**

声明数据为常量，可以是编译时常量，也可以是在运行时被初始化后不能被改变的常量。

- 对于基本类型，final 使数值不变；
- 对于引用类型，final 使引用不变，也就不能引用其它对象，但是被引用的对象本身是可以修改的。

 当用final作用于类的成员变量时，成员变量（注意是类的成员变量，局部变量只需要保证在使用之前被初始化赋值即可）必须在定义时或者构造器中进行初始化赋值，而且final变量一旦被初始化赋值之后，就不能再被赋值了。

还有一点很重要：当final变量是基本数据类型以及String类型时，如果在编译期间能知道它的确切值，则编译器会把它当做编译期常量使用。也就是说在用到该final变量的地方，相当于直接访问的这个常量，不需要在运行时确定。

```java
public class Test { 
    public static void main(String[] args)  { 
        String a = "hello2";   
        final String b = "hello"; 
        String d = "hello"; 
        String c = b + 2; //编译时确定为hello2  
        String e = d + 2; 
        System.out.println((a == c)); 
        System.out.println((a == e)); 
    } 
}
```

输出结果：true、false

## **final属性值能被反射修改吗？**

```java
public class Test {
    private final String NAME = "亦袁非猿";
    
    public String getName() {
        return NAME;
    }
}
public class Client {
    public static void main(String[] args) {
        Test test = new Test();
        Class mClass = test.getClass();
        // 获取NAME变量进行修改
        Field field = mClass.getDeclaredField("NAME");
        if (field != null) {
           field.setAccessible(true);
           System.out.println("modify before "+field.get(test));
           // 进行修改 
           field.set(test, "钢丝");
           System.out.println("modify after "+field.get(test));
            //编译时优化为：getName = 亦袁非猿
           System.out.println("getName = "+test.getName());
         }
    }
}

// 输出：
modify before 亦袁非猿
modify after 钢丝
getName = 亦袁非猿
```

这里使用反射修改了NAME的值，但是存在编译器优化。

## 参考资料

[浅谈Java中的final关键字](https://www.cnblogs.com/xiaoxi/p/6392154.html)

[**final属性值能被反射修改吗？**](https://www.jianshu.com/p/50830768bd52)