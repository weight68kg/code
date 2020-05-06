[TOC] 
#### Looper 
###### 创建 
每个线程只允许创建一个`Looper`,创建的方法是 _Looper.prepare()_ ，得到的`Looper实例`会存在`ThreadLocal`中。 

```java 
private static void prepare(boolean quitAllowed) { 
    if (sThreadLocal.get() != null) { 
        throw new RuntimeException("Only one Looper may be created per thread"); 
    } 
    sThreadLocal.set(new Looper(quitAllowed)); 
} 
``` 
主线程会在`main函数`下调用 _Looper.prepareMainLooper();_ 创建一个主线程的 `Looper`。并会直接调用 _Looper.loop();_ 先看 _prepareMainLooper()_ 在看 _loop()_ 
```java 
public static void prepareMainLooper() { 
    prepare(false); 
    synchronized (Looper.class) { 
        if (sMainLooper != null) { 
            throw new IllegalStateException("The main Looper has already been prepared."); 
        } 
        sMainLooper = myLooper(); 
    } 
} 
``` 
_Looper.myLooper();_ 就是从`ThreadLocal`中取出当前线程的`Looper实例` 
```java 
public static @Nullable Looper myLooper() { 
    return sThreadLocal.get(); 
} 
``` 
###### Looper.loop() 
接下来看说 _Looper.loop();_ 但是重点在 _queue.next();_ 
```java 
public static void loop() { 
    //省略代码 判断当前线程有没有looper实例，也就是有没有调用prepare() 
    final MessageQueue queue = me.mQueue; 
    boolean slowDeliveryDetected = false; 
    for (;;) { 
        //从队列里取message，队列没有消息会阻塞线程。取到message=null是退出消息队列。 
        Message msg = queue.next(); // might block 
        if (msg == null) { 
            // No message indicates that the message queue is quitting. 
            return; 
        } 
        //msg.target是Handler dispatchMessage(msg)最后会回调handleMessage(msg) 
        msg.target.dispatchMessage(msg); 
        //释放消息占据的资源 
        msg.recycleUnchecked(); 
    } 
} 
``` 
#### Handler 
###### 创建 
使用`Handler`是需要`Looper`的，所以使用场景分为三种情况，在主线程，子线程和自定义Looper。 
* 主线程 已经创建过了Looper了，直接使用Handler就可以，构造函数会调用 _mLooper = Looper.myLooper();_ 
* 子线程 没有创建`Looper`，直接调用 _mLooper = Looper.myLooper();_ 会主动抛异常。所以在创建`Hanlder`之前要先调用 _Looper.prepare()_ 在该线程中创建一个`Looper`，还要接着调用 _Looper.loop();_ ,之后才可以使用。 
* 自定义Looper 子线程更新ui 
###### 构造函数 
`Handler`构造主要是初始化`Looper实例`和`MessageQueue实例` 
```java 
public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) { 
    mLooper = looper; 
    mQueue = looper.mQueue; 
    mCallback = callback; 
    mAsynchronous = async; 
} 
``` 
###### 具体使用 
不论是使用`sendMessage()`还是`post()`最后都会调用`enqueueMessage()` 
```java 
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg, 
long uptimeMillis) { 
    msg.target = this; 
    msg.workSourceUid = ThreadLocalWorkSource.getUid(); 
    if (mAsynchronous) { 
        msg.setAsynchronous(true); 
    } 
    return queue.enqueueMessage(msg, uptimeMillis); 
} 
``` 
给`Message`绑定当前`Handler`，然后调用`MessageQueue.enqueueMessage(Message msg, long when)`方法。把`Message`存进`MessageQueue`。接下来看 `MessageQueue` 
#### MessageQueue 
###### 创建 
　在创建`Looper`时创建。 
```java 
private Looper(boolean quitAllowed) { 
    mQueue = new MessageQueue(quitAllowed); 
    mThread = Thread.currentThread(); 
} 
``` 
###### next() 
部分内容没看懂，总体意思就是取message，message 为空就阻塞，直到被唤醒。 
```java 
Message next() { 
    //省略代码 如果队列要退出等情况 return null 
    int pendingIdleHandlerCount = -1; // -1 only during first iteration 
    int nextPollTimeoutMillis = 0; 
    for (;;) { 
        // nativePollOnce方法在native层，若是nextPollTimeoutMillis为-1，此时消息队列处于等待状态 
        nativePollOnce(ptr, nextPollTimeoutMillis); 
        synchronized (this) { 
            // Try to retrieve the next message. Return if found. 
            final long now = SystemClock.uptimeMillis(); 
            Message prevMsg = null; 
            //MessageQueue 队列 
            Message msg = mMessages; 
            if (msg != null && msg.target == null) { 
                // Stalled by a barrier. Find the next asynchronous message in the queue. 
                //寻找异步的message 
                do { 
                    prevMsg = msg; 
                    msg = msg.next; 
                } while (msg != null && !msg.isAsynchronous()); 
            } 
            if (msg != null) { 
                if (now < msg.when) { 
                //msg.when delay时间 
                // 设置一个超时时间 
                nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE); 
                } else { 
                    // Got a message. 
                    mBlocked = false; 
                    if (prevMsg != null) { 
                        prevMsg.next = msg.next; 
                    } else { 
                        mMessages = msg.next; 
                    } 
                    msg.next = null; 
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg); 
                    msg.markInUse(); 
                    return msg; 
                } 
            } else { 
                // 若 消息队列中已无消息，则将nextPollTimeoutMillis参数设为-1 
                // 下次循环时，消息队列则处于等待状态 
                nextPollTimeoutMillis = -1; 
            } 
            // Process the quit message now that all pending messages have been handled. 
            if (mQuitting) { 
                dispose(); 
                return null; 
            } 
            // If first time idle, then get the number of idlers to run.                 // Idle handles only run if the queue is empty or if the first message 
            // in the queue (possibly a barrier) is due to be handled in the future. 
            if (pendingIdleHandlerCount < 0 
&& (mMessages == null || now < mMessages.when)) { 
                pendingIdleHandlerCount = mIdleHandlers.size(); 
            } 
            if (pendingIdleHandlerCount <= 0) { 
                // No idle handlers to run. Loop and wait some more. 
                mBlocked = true; 
                continue; 
            } 
            if (mPendingIdleHandlers == null) { 
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)]; 
            } 
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers); 
        } 
        // Run the idle handlers. 
        // We only ever reach this code block during the first iteration. 
        for (int i = 0; i < pendingIdleHandlerCount; i++) { 
            final IdleHandler idler = mPendingIdleHandlers[i]; 
            mPendingIdleHandlers[i] = null; // release the reference to the handler 
            boolean keep = false; 
            try { 
                keep = idler.queueIdle(); 
            } catch (Throwable t) { 
                Log.wtf(TAG, "IdleHandler threw exception", t); 
            } 
            if (!keep) { 
                synchronized (this) { 
                mIdleHandlers.remove(idler); 
            } 
        } 
    } 
    // Reset the idle handler count to 0 so we do not run them again. 
    pendingIdleHandlerCount = 0; 
    // While calling an idle handler, a new message could have been delivered 
    // so go back and look again for a pending message without waiting. 
    nextPollTimeoutMillis = 0; 
    } 
} 
``` 
###### Message存到MessageQueue 
```java 
boolean enqueueMessage(Message msg, long when) { 
    //省略代码 判断msg没有Handler抛异常 
    //省略代码 判断msg正在使用抛异常 
    synchronized (this) { 
        //省略代码 如果队列正在退出，释放msg资源，并返回false 
        msg.markInUse(); 
        msg.when = when; 
        Message p = mMessages; 
        boolean needWake; 
        //第一次创建Looper 和 MessageQueue 
        if (p == null || when == 0 || when < p.when) { 
            // New head, wake up the event queue if blocked. 
            //给队列赋值，如果blocked 就在后面唤醒 
            msg.next = p; 
            mMessages = msg; 
            needWake = mBlocked; 
        } else { 
            // Inserted within the middle of the queue. Usually we don't have to wake 
            // up the event queue unless there is a barrier at the head of the queue 
            // and the message is the earliest asynchronous message in the queue. 
            needWake = mBlocked && p.target == null && msg.isAsynchronous(); 
            Message prev; 
            for (;;) { 
                prev = p; 
                p = p.next; 
                if (p == null || when < p.when) { 
                    break; 
                } 
                if (needWake && p.isAsynchronous()) { 
                    needWake = false; 
                } 
            } 
            msg.next = p; // invariant: p == prev.next 
            prev.next = msg; 
        } 
        // We can assume mPtr != 0 because mQuitting is false. 
        if (needWake) { 
            nativeWake(mPtr); 
        } 
    } 
    return true; 
} 
``` 




