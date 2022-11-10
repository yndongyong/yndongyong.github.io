---
title: EventBus源码分析
categories:
  - - android
  - - EventBus
tags:
  - - android
date: 2022-10-18 16:05:11
---

### 是什么

> EventBus is a publish/subscribe event bus for Android and Java.

EventBus 是基于发布-订阅模式（也叫观察模式）的事件总线框架。在Android开发中，常用于组件之间的通信与数据传输，可以有效的进行组件之间的解耦。例如，Activity之间、与Fragment的参数传递，事件通知等。

### 有什么特点

- 使用简单，如果这也算一个优点的话。相比LocalBroadcastManager使用是真的简单。

- 有五种线程模式，支持订阅方法执行在指定的线程。
- 可以指定订阅者的优先级，按优先级处理事件，还可以拦截事件。
- 3.0版本之前使用的运行时的反射收集事件的订阅方法，有一定的性能损耗，但3.0之后使用了Apt技术，编译时期就完成了订阅事件的收集。

### 有什么缺点

- 没有生命周期感知的功能

注册和反注册需要成对的出现，如果忘记反注册可能会导致内存泄漏的问题发生，特别是在Activity、Fragment中使用时。毕竟这个框架出现的时间可比LifeCycle出现的早很多了。

- 不克制使用的话，会造成大量事件的满天飞，同时也导致POJO类的膨胀。

需要为Event事件专门定义java类，针对数据传输的场景，难以复用Event。

### 简单的使用

使用方式就不介绍了，参见官网：http://greenrobot.org/eventbus/

### 是如何运行的

基于3.6.0版本进行源码分析。

#### 基本概念

- Event ：事件，分为普通事件和粘性事件（StickyEvent），通常定义为一个键的pojo类。
- Publisher：发布者，使用post方法发布事件。
- Subscriber：订阅者，订阅关心的Event，在指定的线程执行。

先贴出一下EventBus类中的几个关键的缓存map

```
public class EventBus {

    static volatile EventBus defaultInstance;

    private static final EventBusBuilder DEFAULT_BUILDER = new EventBusBuilder();
    //key:event class ,values:event class 已经event class的父类
    private static final Map<Class<?>, List<Class<?>>> eventTypesCache = new HashMap<>();

    //    key:event，values:订阅了某个事件的所有订阅者的集合
    private final Map<Class<?>, CopyOnWriteArrayList<Subscription>> subscriptionsByEventType;
    //    key:订阅者，values:订阅的所有event class
    private final Map<Object, List<Class<?>>> typesBySubscriber;
    //粘性事件，key:eventType class ：事件对象 ，粘性消息重复添加的话，只会保存最后一个
    private final Map<Class<?>, Object> stickyEvents;

    .....
}
```

#### 注册订阅者

入口在Eventbus的register

```java
public void register(Object subscriber) {
        if (AndroidDependenciesDetector.isAndroidSDKAvailable() && !AndroidDependenciesDetector.areAndroidComponentsAvailable()) {
            // Crash if the user (developer) has not imported the Android compatibility library.
            throw new RuntimeException("It looks like you are using EventBus on Android, " +
                    "make sure to add the \"eventbus\" Android library to your dependencies.");
        }

        Class<?> subscriberClass = subscriber.getClass();
    //步骤1
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                //步骤2
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

#### 收集事件订阅

拿到`subscriber`的class，使用`subscriberMethodFinder`找出当前class以及父类下的所有订阅方法，跟一下`findSubscriberMethods`方法

```java
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
//        从缓存中找 事件的处理方法集合
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
        //evebuts 3.0 之前逻辑,反射收集
        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            //3.0之后的版本使用apt，编译时就生成了索引类，
            //索引类的添加EventBus.builder().addIndex(new MyEventBusIndex()).installDefaultEventBus();
            subscriberMethods = findUsingInfo(subscriberClass);
        }
    //进行了注册，但是没有@Subscriber的方法，就抛出异常。
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```

先看一下`HEAD_CACHE`的定义

```
//    key:订阅者类，values：某一事件的所有处理方法的集合
    private static final Map<Class<?>, List<SubscriberMethod>> METHOD_CACHE = new ConcurrentHashMap<>();
```

查找时，先从`METHOD_CACHE`缓存中查找，在`METHOD_CACHE`缓存没有命中时，3.0之前的版本使用反射的方式查找，3.0的版本冲Apt时期生成的索引类中的查找。找到是放入`METHOD_CACHE`中。

在进行重复注册与反注册的场景，可以有效避免反复查找的问题，比如，一个事件只在`Activity`可见阶段才感兴趣的情况，就会在`onResume`生命周期进行注册，在`onPause`的生命周期进行反注册。这个空间换时间的缓存机制，就可以避免二次查找。

##### 3.0之前反射方式

看一下3.0之前的反射方式的，主要跟一下`findUsingReflection`方法

```java
private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
//        prepareFindState object pool 防止频繁创建对象，
        FindState findState = prepareFindState();
        findState.initForSubscriber(subscriberClass);
//        clazz 订阅者，或者订阅者的分类
        while (findState.clazz != null) {
            //通过反射 收集订阅者的事件处理方法
            findUsingReflectionInSingleClass(findState);
            //在父类中，收集一遍，直接系统类时，clazz = false,结束递归
            findState.moveToSuperclass();
        }
        //返回事件处理集合，并且重置findState，将尝试将findState放回对象池中
        return getMethodsAndRelease(findState);
    }
```

使用迭代的方式，收集订阅者以及父类中的订阅方法，直到找到系统类是退出迭代。

`FindState`用于存放查找过程中的各种现相关信息，使用了对象池，避免频繁的对象创建，又再一次用空间换时间，同时也可以避免频繁的GC，造成内存抖动，引起卡顿。

该类的部分源码

```java
 static class FindState {
        //当前订阅类以及父类中，事件的所有处理订阅方法
        final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
        //key：eventType,value: eventType的处理的方法
        final Map<Class, Object> anyMethodByEventType = new HashMap<>();
        //key：处理方法的字符串化，value：订阅类
        final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
        final StringBuilder methodKeyBuilder = new StringBuilder(128);

        //订阅者
        Class<?> subscriberClass;
//        订阅者的父类，有可能有
        Class<?> clazz;
        boolean skipSuperClasses;
     //APT先关的索引信息
        SubscriberInfo subscriberInfo;
```

之后就到了`findUsingReflectionInSingleClass`方法。见名知意。

```java
private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            try {
                methods = findState.clazz.getMethods();
            } catch (LinkageError error) { // super class of NoClassDefFoundError to be a bit more broad...
                String msg = "Could not inspect methods of " + findState.clazz.getName();
                if (ignoreGeneratedIndex) {
                    msg += ". Please consider using EventBus annotation processor to avoid reflection.";
                } else {
                    msg += ". Please make this class visible to EventBus annotation processor to avoid reflection.";
                }
                throw new EventBusException(msg, error);
            }
            findState.skipSuperClasses = true;
        }
//        从这里可以看出来，为啥处理事件的方法需要是定义为public，已经方法参数只能有一个的要求，因为反射的查找的缘故。
        for (Method method : methods) {
            int modifiers = method.getModifiers();
//            只能是public，非静态、非抽象类
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        //关键点1
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            //将方法、event、threadmod、优先级、是否是粘性包装到SubscriberMethod中
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }
```

找到`@Subscribe`注解的非`static`的、`public`的，有且仅有一个参数的方法，包装到`SubscriberMethod`中，解析ThreadMode、优先级priority、是否是粘性事件。

其中`findState.checkAdd(method, eventType)`,处理了子类和父类中，是否存在订阅方法重新的问题。

```java
//检查了子类是否有重写重复的方法。有的话需要使用之类的方法。
        boolean checkAdd(Method method, Class<?> eventType) {
            // 2 level check: 1st level with event type only (fast), 2nd level with complete signature when required.
            // Usually a subscriber doesn't have methods listening to the same event type.
            Object existing = anyMethodByEventType.put(eventType, method);
            if (existing == null) {
                return true;
            } else {
                if (existing instanceof Method) {
                    if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                        // Paranoia check
                        throw new IllegalStateException();
                    }
                    // Put any non-Method object to "consume" the existing Method
                    anyMethodByEventType.put(eventType, this);
                }
                return checkAddWithMethodSignature(method, eventType);
            }
        }
```

##### 3.0之后APT技术

​	使用`EventBusAnnotationProcessor`在编译器，遍历所有的类 ，搜集订阅者与订阅方法，最终生成一个实现`SubscriberInfoIndex`的类，然后在EventBus初始化是调用addIndex方法使用。

使用findUsingInfo查找所有订阅者的订阅方法,思路和3.0之前的版本一直，但是多了一个降级策略.

```java
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
    FindState findState = prepareFindState();
    findState.initForSubscriber(subscriberClass);
    while (findState.clazz != null) {
        findState.subscriberInfo = getSubscriberInfo(findState);
        if (findState.subscriberInfo != null) {
            SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
            for (SubscriberMethod subscriberMethod : array) {
                if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                    findState.subscriberMethods.add(subscriberMethod);
                }
            }
        } else {
            //降级策略
            findUsingReflectionInSingleClass(findState);
        }
        findState.moveToSuperclass();
    }
    return getMethodsAndRelease(findState);
}
```

#### 粘性事件分发

通过步骤 1， 拿到所有的事件订阅方法，然后进行订阅

```java
synchronized (this) {
    for (SubscriberMethod subscriberMethod : subscriberMethods) {
    	//步骤2
    	subscribe(subscriber, subscriberMethod);
    }
}
```

```java
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
//        获取该事件的所有订阅者
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            //不能重复注册
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            // 优先级高的放在前面，优先级的低放在最后
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }
        //订阅者-订阅的所有事件的 映射
        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);

        //5.1 有粘性事件时，注册订阅者时直接分发
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
                        //粘性事件分发
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                //粘性事件分发
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
    }
```

粘性事件分发

有粘性事件是在注册阶段就判断了是否有响应的订阅者，有的话就`checkPostStickyEventToSubscription`最终调用到`postToSubscription`，普通事件的分发最终也是通过`postToSubscription`，该方法 ，具体的放到后面一起分析

#### 事件发布与分发

核心逻辑，就是找到事件的订阅者，执行相应的订阅方法，

```java
//6 发送event
    /** Posts the given event to the event bus. */
    public void post(Object event) {
        PostingThreadState postingState = currentPostingThreadState.get();
        List<Object> eventQueue = postingState.eventQueue;
        eventQueue.add(event);

        if (!postingState.isPosting) {
            postingState.isMainThread = isMainThread();
            postingState.isPosting = true;
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                //遍历当前线程的事件队列，一个一个发送
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



`currentPostingThreadState`是一个ThreadLocal变量，它提供线程本地变量，如果创建一个ThreadLocal变量，那么访问这个变量的每个线程都会有这个变量的一个副本，在实际多线程操作的时候，操作的是自己本地内存中的变量，从而规避了线程安全问题。通常EventBus都是process-wide范围的。

```java
final static class PostingThreadState {
    //存放事件
    final List<Object> eventQueue = new ArrayList<>();
    boolean isPosting;
    //是否是主线程post的
    boolean isMainThread;
    //订阅者和事件处理方法的包装类
    Subscription subscription;
    Object event;
    boolean canceled;
}
```

遍历eventQueue，从数组头部开始分发，`postSingleEvent`之后调用到`postSingleEventForEventType`，取出所有的events(继承的情况)，之后再到`postSingleEventForEventType`，从缓存中找到所有的订阅者与订阅方法的包装类，使用CopyOnWriteArrayList存放subscriptions，避免遍历时对Suscriber的反注册引起的问题。

```java
// 6.1 发送单个event，已经找到了eventClass
    private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted;
                try {
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

遍历分发，最终到`postToSubscription`

```java
//  7. 分发 event 到 suscription，
    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        //按线程模型分发
        switch (subscription.subscriberMethod.threadMode) {
//           在投递的线程处理，没有线程切换，在主线程post，invoke就在主线程
            case POSTING:
                invokeSubscriber(subscription, event);
                break;
            case MAIN:
//                当前线程是主线程直接调用
                if (isMainThread) {
                    invokeSubscriber(subscription, event);
                } else {
                    mainThreadPoster.enqueue(subscription, event);
                }
                break;
//                event 总是被排队执行
            case MAIN_ORDERED:
                if (mainThreadPoster != null) {
                    mainThreadPoster.enqueue(subscription, event);
                } else {
                    // temporary: technically not correct as poster not decoupled from subscriber
                    invokeSubscriber(subscription, event);
                }
                break;
            case BACKGROUND:
//                已经是后台线程就直接执行
                if (isMainThread) {
//                    Executors.newCachedThreadPool()
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



`ThreadMode`中的`POSTING`、`MAIN`、`MAIN_ORDERED`、`BACKGROUND`，都可能阻塞post的所在线程，最好不要执行耗时操作。

`mainThreadPoster`底层使用`Android`的`Handler`进行分发，`backgroundPoster`、`asyncPoster`都是依赖同一个`Executors.newCachedThreadPool()`进行分发，被分发之后在执行时都是通过`invokeSubscriber`进行调用。

其中`backgroundPoster`内部也使用了一个链表queue，使事件按post的顺序执行。

#### 取消注册

​	

### 总结

框架整体考虑的非常的完善，各种级别的缓存，用空间换时间，避免二次查询，多处使用对象池，避免对象的频繁创建，通过DCL单例提供EventBus对象，使用Builder模式，优化使用大量参数进行初始化。











