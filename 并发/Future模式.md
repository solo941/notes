## Future

Future表示异步计算的结果，可以看成线程在执行时留给调用者的一个存根。Future接口提供了如下方法，这些方法都有返回值：

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

## 基于lock和condition的Future实现

实现了阻塞的get()方法，通过上一篇讲的ReentrantLock和Condition实现。

```java
public class ResponseFuture implements Future<String> {
    private final ResponseCallback callback;
    private volatile String responsed;
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

```
还没准备好
ResponseCallback：处理完成，返回：ok
已发送回应：ok
收到回应:ok
```

## FutureTask

在J.U.C的组件中提供了一种组建FutureTask，它实现了RunnableFuture<V>接口，该接口继承了Runnable 和 Future 接口，因此FutureTask 既可以当做一个任务执行，也可以有返回值。

FutureTask 可用于异步获取执行结果或取消执行任务的场景。当一个计算任务需要执行很长时间，那么就可以用 FutureTask 来封装这个任务，主线程在完成自己的任务之后再去获取结果。

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask<Integer> futureTask = new FutureTask<Integer>(new Callable<Integer>() {
            @Override
            public Integer call() throws Exception {
                int result = 0;
                for (int i = 0; i < 100; i++) {
                    Thread.sleep(10);
                    result += i;
                }
                return result;
            }
        });

        Thread computeThread = new Thread(futureTask);
        computeThread.start();

        Thread otherThread = new Thread(() -> {
            System.out.println("other task is running...");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        otherThread.start();
        System.out.println(futureTask.get());
    }
```

```
other task is running...
4950
```

使用FutureTask封装耗时的任务,主线程可以在昨晚自己的任务后，异步获取分线程计算结果，还可以通过cancel()取消分线程任务。我们将前面的代码稍加修改：

```java
private static volatile int result = 0;
Thread otherThread = new Thread(() -> {
            System.out.println("other task is running...");
            try {
                Thread.sleep(800);
                futureTask.cancel(true);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally {
                System.out.println(result);
            }
        });
        otherThread.start();
```

```
other task is running...
3081
```

