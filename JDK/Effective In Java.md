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

