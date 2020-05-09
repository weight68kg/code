[TOC]
### 注册
```java
EventBus.getDefault().register(this);
```
```java
public void register(Object subscriber) {
    //获取class
    Class<?> subscriberClass = subscriber.getClass();
    
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        //所有订阅方法 都 和被观察者绑定
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```

先通过`findSubscriberMethods()`方法找到 __SubscriberMethod__ 的集合。__SubscriberMethod__ 根据意思表示，订阅方法。也就是接收的方法。
`findSubscriberMethods()`遍历父类，找到`@Subscribe`方法，中间会配合反射寻找

```java
public class SubscriberMethod {
    final Method method;
    final ThreadMode threadMode;
    final Class<?> eventType;
    final int priority;
    final boolean sticky;
    /** Used for efficient comparison */
    String methodString;
}
```

接着调用`subscribe()` 传入 __SubscriberMethod__ 和订阅者。

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    //获取接收类型。
    Class<?> eventType = subscriberMethod.eventType;
    //创建Subscription订阅关系
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    //获取订阅关系集合 
    //subscriptionsByEventType在EventBus构造创建的，用来存储以eventType为key,SubscriberMethod为value的集合
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    //订阅集合为空 创建一个新的集合
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        // key=class value=订阅集合
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        // 不能重复注册 
        //Subscription重写了equals，订阅者和SubscriberMethod相同时返回true
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }

    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        //第一次为空集合 直接添加订阅    
        //之后添加之前判断优先级，优先级高的插在前面，否则放在最后
        if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
            subscriptions.add(i, newSubscription);
            break;
        }
    }

    //typesBySubscriber 以订阅者为key，eventType为value的集合
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        subscribedEvents = new ArrayList<>();
        //key=被观察者类 value=event集合
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    subscribedEvents.add(eventType);
    //粘性事件会调用 checkPostStickyEventToSubscription()方法
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```
总结：
* 通过反射找到带`@Subscribe`的方法生成 __SubscriberMethod__实例。
* 遍历 __SubscriberMethod__ 集合 ，绑定订阅者和被订阅者
* subscriptionsByEventType存储 eventType对应SubscriberMethod集合,然后对SubscriberMethod集合按优先级排序
* typesBySubscriber存储订阅者对应eventType
* 做sticky相关操作

### 发送

 两种发送方式，粘性和非粘性。`postSticky`：指事件消费者在事件发布之后才注册的也能接收到该事件的特殊类型。 

```java
EventBus.getDefault().post("6");
EventBus.getDefault().postSticky("6");
```

将sticky事件存到stickyEvents，然后嗲用post()
```java
public void postSticky(Object event) {
        synchronized (stickyEvents) {
            stickyEvents.put(event.getClass(), event);
        }
        // Should be posted after it is putted, in case the subscriber wants to remove immediately
        post(event);
    }
```

```java
public void post(Object event) {
        //currentPostingThreadState 是一个ThreadLocal变量
        PostingThreadState postingState = currentPostingThreadState.get();
        //获取event队列
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);
		//初始化 未开始post
        if (!postingState.isPosting) {
        	//设置是否是主线程
            postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
            //改变发送状态
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                //发送事件
                while (!eventQueue.isEmpty()) {
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```

接下来看发事件的主体

```java
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        //eventInheritance 默认为true，如果设为 true 的话，它会拿到 Event 父类的 class 类型，设为 false，可以在一定程度上提高性能；
        if (eventInheritance) {
            //得到 订阅类型 所有父类和接口
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                //如果有一个@Subscribe()注解，subscriptionFound就是ture，否则是false。
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
```

eventInheritance在EventBus构造初始化的，可以通过EventbusBuilder修改

```java
var eventBusBuilder = EventBus.builder()
var eventBus = eventBusBuilder.build()
eventBus.register(this)
```

遍历订阅类型

```java
private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
    CopyOnWriteArrayList<Subscription> subscriptions;
    synchronized (this) {
        //根据event 获取订阅集合
        //在注册是添加的
        subscriptions = subscriptionsByEventType.get(eventClass);
    }
    if (subscriptions != null && !subscriptions.isEmpty()) {
        //遍历订阅集合
        for (Subscription subscription : subscriptions) {
            postingState.event = event;
            postingState.subscription = subscription;
            boolean aborted = false;
            try {
                //发布订阅
                postToSubscription(subscription, event, postingState.isMainThread);
                aborted = postingState.canceled;
            } finally {
                postingState.event = null;
                postingState.subscription = null;
                postingState.canceled = false;
            }
            if (aborted) {
                break;
            }
        }
        return true;
    }
    return false;
}
```

最后一步发布订阅

```java
private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
    switch (subscription.subscriberMethod.threadMode) {
        case POSTING:
            invokeSubscriber(subscription, event);
            break;
        case MAIN:
            if (isMainThread) {
                invokeSubscriber(subscription, event);
            } else {
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                // temporary: technically not correct as poster not decoupled from subscriber
                invokeSubscriber(subscription, event);
            }
            break;
        case BACKGROUND:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + subscription.subscriberMethod.threadMode);
    }
}
```
调用订阅的方法
```java
void invokeSubscriber(Subscription subscription, Object event) {
    try {
        // 备注7 方法反射    订阅方法 与 event 匹配
        subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
    } catch (InvocationTargetException e) {
        handleSubscriberException(subscription, event, e.getCause());
    } catch (IllegalAccessException e) {
        throw new IllegalStateException("Unexpected exception", e);
    }
}
```

### 备注
1. Class.getDeclaredMethods(); 获取该类所有方法
2. Class.getMethods(); 获取该类所有公共方法
3. Method.getDeclaringClass(); 获取方法声明的所在类
4. methodClassOld.isAssignableFrom(methodClass);判定此 Class 对象所表示的类或接口与指定的 Class 参数所表示的类或接口是否相同，或是否是其超类或超接口。如果是则返回 true；否则返回 false。如果该Class 表示一个基本类型，且指定的 Class 参数正是该 Class 对象，则该方法返回 true；否则返回 false。 
5. Clazz.getInterfaces()；它能够获得这个对象所实现的所有接口
6. Clazz.getSuperclass();    获取父类
7. Method invoke(Object obj, Object... args);    第一个参数为类的实例，第二个参数为相应函数中的参数

所有订阅方法都在这一个集合里
final List<SubscriberMethod> subscriberMethods = new ArrayList<>();

key event。value 被观察者Class与所有订阅方法绑定 的集合
private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;

class SubscriberMethod 管理订阅的方法