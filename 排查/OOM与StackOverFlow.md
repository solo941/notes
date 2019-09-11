## OOM

### 堆内存不足

最常见的OOM出现原因，堆内存不足

```
java.lang.OutOfMemoryError: Java heap space
```

原因分析：

在 Java 堆中只要不断的创建对象，并且 `GC-Roots` 到对象之间存在引用链，这样 `JVM` 就不会回收对象。只要将`-Xms(最小堆)`,`-Xmx(最大堆)` 设置为一样禁止自动扩展堆内存。

解决方法：

使用jmap 对2657进程dump堆内存，使用mat进行分析

```
jmap -heap:format=b 2657
```



### 元空间溢出

```
java.lang.OutOfMemoryError: Metaspace
```

JDK8后，元空间替换了永久代，元空间使用的是本地内存，还有其它细节变化：

- 字符串常量由永久代转移到堆中
- 和永久代相关的JVM参数已移除

出现永久代或元空间的溢出的原因可能有如下几种：

1、在Java7之前，频繁的错误使用String.intern()方法
 2、生成了大量的代理类，导致方法区被撑爆，无法卸载
 3、应用长时间运行，没有重启

可以使用 `-XX:MaxMetaspaceSize=10M` 来限制最大元数据。这样当不停的创建类时将会占满该区域并出现 `OOM`。

解决方案

dump之后通过mat检查是否存在大量由于反射生成的代理类；

设置元数据大小



### **方法栈溢出**

```
java.lang.OutOfMemoryError : unable to create new native Thread
```

原因分析：

出现这种异常，基本上都是创建的了大量的线程导致的。

解决方案

visualvm查看线程栈大小，通过`-Xss`降低每个线程栈的大小。

## 死锁分析

可以使用java visualvm进行死锁分析

## StackOverFlow

栈溢出，常见于递归写错时出现

解决方法：

java visualvm可以dump线程，定位递归语句的错误。

