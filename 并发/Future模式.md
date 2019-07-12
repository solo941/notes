## Future

Future表示异步计算的结果，可以看成线程在执行时留给调用者的一个存根。Future接口提供了如下方法：

```java
public interface Future<V> {
    /** 取消，mayInterruptIfRunning-false:不允许在线程运行时中断 **/
    boolean cancel(boolean mayInterruptIfRunning);
    /** 是否取消**/
    boolean isCancelled();
    /** 是否完成 **/
    boolean isDone();
    /** 同步获取结果 **/
    V get() throws InterruptedException, ExecutionException;
    /** 同步获取结果，响应超时 **/
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

