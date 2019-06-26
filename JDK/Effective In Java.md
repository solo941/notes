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

