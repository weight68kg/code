__实现原理__ 线程池，Handler 
#### Callable 
```java 
@FunctionalInterface 
public interface Callable<V> { 
    V call() throws Exception; 
} 
``` 
和接口Runnable很类似，但是Callable 的 call方法有返回值。 
#### Future 
```java 
public interface Future<V> { 
    boolean cancel(boolean mayInterruptIfRunning); 
    boolean isCancelled(); 
    boolean isDone(); 
    V get() throws InterruptedException, ExecutionException; 
    V get(long timeout, TimeUnit unit) ;
    throws InterruptedException, ExecutionException,    TimeoutException; 
} 
``` 
__Future 有五个方法：__ 

* cancel()用来取消任务，如果取消任务成功则返回true，如果取消任务失败则返回false。参数mayInterruptIfRunning表示是否允许取消正在执行却没有执行完毕的任务，如果设置true，则表示可以取消正在执行过程中的任务。如果任务已经完成，则无论mayInterruptIfRunning为true还是false，此方法肯定返回false，即如果取消已经完成的任务会返回false；如果任务正在执行，若mayInterruptIfRunning设置为true，则返回true，若mayInterruptIfRunning设置为false，则返回false；如果任务还没有执行，则无论mayInterruptIfRunning为true还是false，肯定返回true。 
* isCancelled()方法表示任务是否被取消成功，如果在任务正常完成前被取消成功，则返回 true。 
* isDone()表示任务是否已经完成，若任务完成，则返回true。取消了也是完成。 
* get()用来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回； 
* get(long timeout, TimeUnit unit)获取执行结果，如果在指定时间内，还没获取到结果，就直接返回null。 
__也就是说Future提供了三种功能：判断任务是否完成。能够中断任务。能够获取任务执行结果。__ 
#### FutureTask 
FutureTask 是 RunnableFuture 唯一的实现类。 

```java 
public interface RunnableFuture<V> extends Runnable, Future<V>{ 
    void run(); 
} 
``` 
根据源码知道接口`RunnableFuture`继承了`Runnable`和`Future`。所以`FutureTask`也是一个 Runnable。 
接下来看 FutureTask 构造。 

```java 
public FutureTask(Callable<V> callable) { 
    if (callable == null) 
        throw new NullPointerException(); 
    this.callable = callable; 
    this.state = NEW; // ensure visibility of callable 
}
``` 
传进一个参数是`Callable`。 
接着看一下`run()`的具体实现 

```java 
public void run() { 
    if (state != NEW || !U.compareAndSwapObject(this, RUNNER, null, Thread.currentThread())) 
        return; 
    try { 
        Callable<V> c = callable; 
        if (c != null && state == NEW) { 
            V result; 
            boolean ran; 
            try { 
                result = c.call(); 
                ran = true; 
            } catch (Throwable ex) { 
                result = null; 
                ran = false; 
                setException(ex); 
            } 
            if (ran) 
                set(result); 
        } 
    } finally { 
    // runner must be non-null until state is settled to 
    // prevent concurrent calls to run() 
    runner = null; 
    // state must be re-read after nulling runner to prevent 
    // leaked interrupts 
    int s = state; 
    if (s >= INTERRUPTING) 
        handlePossibleCancellationInterrupt(s); 
    } 
} 
``` 
可以看到构造方法传入的Callable对象，调用了`call()`方法。 
#### 线程池 
有两个线程池，一个是串行线程`SerialExecutor`，一个是并行线程`ThreadPoolExecutor`。默认是串行。 
先看串行的声明 
```java 
public static final Executor SERIAL_EXECUTOR = new SerialExecutor(); 
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR; 
``` 
接下来是并行 
```java 
private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors(); 
// We want at least 2 threads and at most 4 threads in the core pool, 
// preferring to have 1 less than the CPU count to avoid saturating 
// the CPU with background work 
private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4)); 
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1; 
private static final int KEEP_ALIVE_SECONDS = 30; 
private static final ThreadFactory sThreadFactory = new ThreadFactory() { 
private final AtomicInteger mCount = new AtomicInteger(1); 
public Thread newThread(Runnable r) { 
    return new Thread(r, "AsyncTask #" + mCount.getAndIncrement()); 
} 
}; 
private static final BlockingQueue<Runnable> sPoolWorkQueue = 
new LinkedBlockingQueue<Runnable>(128); 
static { 
    ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor( 
    CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS, 
    sPoolWorkQueue, sThreadFactory); 
    threadPoolExecutor.allowCoreThreadTimeOut(true); 
    THREAD_POOL_EXECUTOR = threadPoolExecutor; 
} 
``` 
#### AsyncTask构建 
现在看一下AsyncTask 的构造方法。 
```java 
public AsyncTask(@Nullable Looper callbackLooper) { 
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper() 
    ? getMainHandler() 
    : new Handler(callbackLooper); 
    mWorker = new WorkerRunnable<Params, Result>() { 
        public Result call() throws Exception { 
            mTaskInvoked.set(true); 
            Result result = null; 
            try { 
                Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND); 
                //noinspection unchecked 
                result = doInBackground(mParams); 
                Binder.flushPendingCommands(); 
            } catch (Throwable tr) { 
                mCancelled.set(true); 
            throw tr; 
            } finally { 
              postResult(result); 
            } 
            return result; 
        } 
    }; 
    mFuture = new FutureTask<Result>(mWorker) { 
        @Override 
        protected void done() { 
            try { 
                postResultIfNotInvoked(get()); 
            } catch (InterruptedException e) { 
                android.util.Log.w(LOG_TAG, e); 
            } catch (ExecutionException e) { 
                throw new RuntimeException("An error occurred while executing doInBackground()", 
                e.getCause()); 
            } catch (CancellationException e) { 
                postResultIfNotInvoked(null); 
            } 
        } 
    }; 
} 
``` 
构造方法里主要做了三件事： 
* 声明 Handler 
* 声明 WorkerRunnable，也就是上米娜提到的 Callable。 
* 把WorkerRunnable对象传入FutureTask，声明。 
看 WorkerRunnable具体实现，在`call()`方法中，调用了`doInBackground()`，这也就是异步操作。 
在看 FutureTask 的`done()`方法，跟踪到源码，可以知道最后完成会调用。回头看`postResultIfNotInvoked()`，最后会调用`postResult()`，并使用handler传递到主线程。 
```java 
private Result postResult(Result result) { 
    @SuppressWarnings("unchecked") 
    Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT, 
    new AsyncTaskResult<Result>(this, result)); 
    message.sendToTarget(); 
    return result; 
} 
``` 
Handler接收 
```java 
private static class InternalHandler extends Handler { 
    public InternalHandler(Looper looper) { 
        super(looper); 
    } 
    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"}) 
    @Override 
    public void handleMessage(Message msg) { 
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj; 
        switch (msg.what) { 
            case MESSAGE_POST_RESULT: 
            // There is only one result 
                result.mTask.finish(result.mData[0]); 
            break; 
            case MESSAGE_POST_PROGRESS: 
                result.mTask.onProgressUpdate(result.mData); 
            break; 
        } 
    } 
} 
``` 
finish方法 
```java 
private void finish(Result result) { 
    if (isCancelled()) { 
        onCancelled(result); 
    } else { 
        onPostExecute(result); 
    } 
    mStatus = Status.FINISHED; 
} 
``` 
最后根据是否取消，而回调不同方法。 
#### AsyncTask 执行 
```java 
@MainThread 
public final AsyncTask<Params, Progress, Result> execute(Params... params) { 
    return executeOnExecutor(sDefaultExecutor, params); 
} 
``` 
传入`sDefaultExecutor`,调用`executeOnExecutor()`,这也是串行的执行方法。 
```java 
@MainThread 
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec, 
Params... params) { 
    if (mStatus != Status.PENDING) { 
        switch (mStatus) { 
        case RUNNING: 
            throw new IllegalStateException("Cannot execute task:" 
            + " the task is already running."); 
        case FINISHED: 
            throw new IllegalStateException("Cannot execute task:" 
            + " the task has already been executed " 
            + "(a task can be executed only once)"); 
        } 
    } 
    mStatus = Status.RUNNING; 
    onPreExecute(); 
    mWorker.mParams = params; 
    exec.execute(mFuture); 
    return this; 
} 
``` 
并行执行直接调用两个参数的方法 
```java 
new AsyncTask().executeOnExecutor(THREAD_POOL_EXECUTOR); 
``` 


