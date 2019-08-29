## java的cpu使用率过高排查

第一步：先执行top命令，查看cpu消耗最高的进程

```Java
PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
  1893 admin     20   0 7127m 2.6g  38m S 181.7 32.6  10:20.26 java
```

第二步，定位PID = 1893进程的各线程CPU使用情况

```
$top -Hp 1893
   PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
  4519 admin     20   0 7127m 2.6g  38m R 18.6 32.6   0:40.11 java
```

第三步，定位代码，把4519这个线程转成16进制

```
printf %x 4519
```

第四步：使用jstack查看栈信息

```
$sudo -u admin  jstack 1893 |grep -A 200 11a7
"HSFBizProcessor-DEFAULT-8-thread-5" #500 daemon prio=10 os_prio=0 tid=0x00007f632314a800 nid=0x11a2 runnable [0x000000005442a000]
   java.lang.Thread.State: RUNNABLE
  at sun.misc.URLClassPath$Loader.findResource(URLClassPath.java:684)
  at sun.misc.URLClassPath.findResource(URLClassPath.java:188)
  at java.net.URLClassLoader$2.run(URLClassLoader.java:569)
  at java.net.URLClassLoader$2.run(URLClassLoader.java:567)
  at java.security.AccessController.doPrivileged(Native Method)
  at java.net.URLClassLoader.findResource(URLClassLoader.java:566)
  at java.lang.ClassLoader.getResource(ClassLoader.java:1093)
  at java.net.URLClassLoader.getResourceAsStream(URLClassLoader.java:232)
  at org.hibernate.validator.internal.xml.ValidationXmlParser.getInputStreamForPath(ValidationXmlParser.java:248)
  at org.hibernate.validator.internal.xml.ValidationXmlParser.getValidationConfig(ValidationXmlParser.java:191)
  at org.hibernate.validator.internal.xml.ValidationXmlParser.parseValidationXml(ValidationXmlParser.java:65)
  at org.hibernate.validator.internal.engine.ConfigurationImpl.parseValidationXml(ConfigurationImpl.java:287)
  at org.hibernate.validator.internal.engine.ConfigurationImpl.buildValidatorFactory(ConfigurationImpl.java:174)
  at javax.validation.Validation.buildDefaultValidatorFactory(Validation.java:111)
  at com.test.common.util.BeanValidator.validate(BeanValidator.java:30)
```

从而判断BeanValidator.java的第30行是有可能存在问题的。