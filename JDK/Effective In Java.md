# JDK8新特性

## Optional

Optional是Java8提供的为了解决null安全问题的一个API。常用的场景如下：

## if判空

```java
if(deviceId != null){
    DummyOperator.doIt(deviceId);
}
```

简化后等价：

```java
Optional.ofNullable(deviceId).ifPresent(DummyOperator::doIt)
```

## 映射

```java
if(deviceId != null){
	String group = device.getGroup();
    DummyOperator.doIt(group);
}
```

简化后等价：

```java
Optional.ofNullable(device).map(d -> d.getGroup()).ifPresent(g -> DummyOperator.doIt(g));
```

## 简化if/else

```java
if(deviceId != null){
	String group = device.getGroup();
    return group;
}else return "unknown";
```

简化后等价：

```java
return Optional.ofNullable(device).map(Device::getGroup).orElse("unknown");
```

## try-with-resource

在Java编程过程中，如果打开了外部资源（文件、数据库连接、网络连接等），我们必须在这些外部资源使用完毕后，手动关闭它们。因为外部资源不由JVM管理，无法享用JVM的垃圾回收机制，如果我们不在编程时确保在正确的时机关闭外部资源，就会导致外部资源泄露，紧接着就会出现文件被异常占用，数据库连接过多导致连接池溢出等诸多很严重的问题。

为了确保外部资源一定要被关闭，通常关闭代码被写入finally代码块中，当然我们还必须注意到关闭资源时可能抛出的异常，于是变有了下面的经典代码：

```java
public static void main(String[] args) {
    FileInputStream inputStream = null;
    try {
        inputStream = new FileInputStream(new File("test"));
        System.out.println(inputStream.read());
    } catch (IOException e) {
        throw new RuntimeException(e.getMessage(), e);
    } finally {
        if (inputStream != null) {
            try {
                inputStream.close();
            } catch (IOException e) {
                throw new RuntimeException(e.getMessage(), e);
            }
        }
    }
}
```

代码结构复杂，做的事情却很简单。除此之外，还会引起异常抑制的问题：

```java
public static void main(String[] args) {
    try {
        send();
    } catch (Exception e) {
        System.out.println("send");
    } finally {
            try {
                close();
            } catch (IOException e) {
                 System.out.println("close");
        }
    }
}
```

如果send和close都会抛异常，此时会报错"close"，send的错误很难定位。

直到JDK7中新增了try-with-resource语法糖，才实现了自动关闭外部资源的功能。

 那什么是try-with-resource呢？简而言之，当一个外部资源的句柄对象（比如FileInputStream对象）实现了AutoCloseable接口，那么就可以将上面的板式代码简化为如下形式：

```
public static void main(String[] args) {
    try (FileInputStream inputStream = new FileInputStream(new File("test"))) {
        System.out.println(inputStream.read());
    } catch (IOException e) {
        throw new RuntimeException(e.getMessage(), e);
    }
}
```

将外部资源的句柄对象的创建放在try关键字后面的括号中，当这个try-catch代码块执行完毕后，Java会确保外部资源的close方法被调用。

try-with-resource时，如果对外部资源的处理和对外部资源的关闭均遭遇了异常，“关闭异常”将被抑制，“处理异常”将被抛出，但“关闭异常”并没有丢失，而是存放在“处理异常”的被抑制的异常列表中。这解决了异常抑制的问题。