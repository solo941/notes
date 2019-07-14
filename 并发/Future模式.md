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

## FutureTask

```java
public class ResponseFuture implements Future<String> {
    private final ResponseCallback callback;
    private String responsed;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition condition = lock.newCondition();

    public ResponseFuture(ResponseCallback callback) {
        this.callback = callback;
    }


    @Override
    public boolean cancel(boolean mayInterruptIfRunning) {
        return false;
    }

    @Override
    public boolean isCancelled() {
        return false;
    }

    @Override
    public boolean isDone() {
        return null != this.responsed;

    }

    @Override
    public String get() throws InterruptedException, ExecutionException {

            try {
                this.lock.lock();
                System.out.println("还没准备好");
                condition.await();
                System.out.println("收到回应:" + this.responsed);
            } finally {
                this.lock.unlock();
            }
        return this.responsed;

    }
    public void done(String responsed) throws Exception{
        try {
            this.lock.lock();
            if(null != this.callback) this.responsed = this.callback.call(responsed);
            this.condition.signal();
            System.out.println("已发送回应："+this.responsed);
        } finally {
            this.lock.unlock();
        }
    }


    @Override
    public String get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
        return null;
    }


}
```

```java
public class ResponseCallback {
    public String call(String o) {
        System.out.println("ResponseCallback：处理完成，返回："+o);
        return o;
    }
}
```



```java
public void test(){
        final ResponseFuture responseFuture = new ResponseFuture(new ResponseCallback());
        ExecutorService executorService = Executors.newCachedThreadPool();
        executorService.execute(() ->{
            try {
                responseFuture.get();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (ExecutionException e) {
                e.printStackTrace();
            }
        });
        executorService.execute(() ->{
            try {
                responseFuture.done("ok");// 处理完成
            } catch (Exception e) {
                e.printStackTrace();
            }
        });
        executorService.shutdown();
    }
```

