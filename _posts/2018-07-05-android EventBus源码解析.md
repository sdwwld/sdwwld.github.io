---
layout: post
title: "android EventBus源码解析"
subtitle: 'android EventBus源码解析'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “自恨枝无叶，莫怨太阳偏。”

## 正文

eventBus3.1.1

eventBus主要用于数据之间的传递，使用也非常简单，就几个主要的方法，一个是register和unregister，这两个要成对出现，一般在onCreate中注册，在onDestroy中取消注册。还有几个方法post,postSticky,removeAllStickyEvents。其中post必须在register之后才有效，否则接收不到信息，postSticky可以在register之前和之后都可以。先看一下register方法

## register

```java
    /**
     * Registers the given subscriber to receive events. Subscribers must call {@link #unregister(Object)} once they
     * are no longer interested in receiving events.
     * <p/>
     * Subscribers have event handling methods that must be annotated by {@link Subscribe}.
     * The {@link Subscribe} annotation also allows configuration like {@link
     * ThreadMode} and priority.
     */
    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)这里面主要看一下findSubscriberMethods方法，他是找到你所在注册类的注解方法，因为一般情况下要想接收数据，必须要加注解的方法，比如@Subscribe(threadMode = ThreadMode.MAIN)或者@Subscribe(threadMode = ThreadMode.MAIN, sticky = true)，当然你也可以修改threadMode指定在其他线程中操作。
POSTING     ：表示发送事件和接收事件在相同的线程  
MAIN        ：表示在主线程中处理数据  
MAIN_ORDERED：和MAIN一样是在主线程中操作，但需要排队  
BACKGROUND  ：在后台线程中执行  
ASYNC       ：在另起一个异步线程中执行

## findSubscriberMethods

```java
    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    } 
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)上面代码中ignoreGeneratedIndex默认情况下是false，其中findUsingReflection和findUsingInfo有可能最终调用的都是findUsingReflectionInSingleClass，为啥说是有可能，是因为findUsingInfo取值的时候会从先从subscriberInfoIndexes中取，如果有就返回，没有就会调用findUsingReflectionInSingleClass方法，所以来看一下findUsingInfo方法

## findUsingInfo

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
                findUsingReflectionInSingleClass(findState);
            }
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)注意这里有个方法moveToSuperclass和上面的while循环，moveToSuperclass是获取FindState字段clazz的父类，就是在当前类中查找之后还要在父类中查找注解的方法，不断往上找，直到父类为空为止。其中FindState是一个数据池FindState的对象，默认值为4，private static final int POOL_SIZE = 4;如果有就从池中取，没有就创建，也是为了提高速度，看一下第一行代码prepareFindState

## prepareFindState

```java
    private FindState prepareFindState() {
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                FindState state = FIND_STATE_POOL[i];
                if (state != null) {
                    FIND_STATE_POOL[i] = null;
                    return state;
                }
            }
        }
        return new FindState();
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)来看一下上面的getSubscriberInfo方法，如果获取为空，就会执行findUsingReflectionInSingleClass方法，来看一下getSubscriberInfo的具体实现

## getSubscriberInfo

```java
    private SubscriberInfo getSubscriberInfo(FindState findState) {
        if (findState.subscriberInfo != null && findState.subscriberInfo.getSuperSubscriberInfo() != null) {
            SubscriberInfo superclassInfo = findState.subscriberInfo.getSuperSubscriberInfo();
            if (findState.clazz == superclassInfo.getSubscriberClass()) {
                return superclassInfo;
            }
        }
        if (subscriberInfoIndexes != null) {
            for (SubscriberInfoIndex index : subscriberInfoIndexes) {
                SubscriberInfo info = index.getSubscriberInfo(findState.clazz);
                if (info != null) {
                    return info;
                }
            }
        }
        return null;
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)下面的subscriberInfoIndexes是由SubscriberMethodFinder的构造方法传进来的值，这里有一个官方提供的优化方法，就是从subscriberInfoIndexes中取，这个最后在介绍。我们先往上看，会发现无论是findUsingInfo还是findUsingReflection方法，在最后都会调用getMethodsAndRelease方法，我们再来看一下它的具体实现

## getMethodsAndRelease

```java
    private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
        List<SubscriberMethod> subscriberMethods = new ArrayList<>(findState.subscriberMethods);
        findState.recycle();
        synchronized (FIND_STATE_POOL) {
            for (int i = 0; i < POOL_SIZE; i++) {
                if (FIND_STATE_POOL[i] == null) {
                    FIND_STATE_POOL[i] = findState;
                    break;
                }
            }
        }
        return subscriberMethods;
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)其实就相当于FindState的回收和订阅方法的返回。下面再来看一下重量级方法findUsingReflectionInSingleClass

## findUsingReflectionInSingleClass

```java
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            int modifiers = method.getModifiers();
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                Class<?>[] parameterTypes = method.getParameterTypes();
                if (parameterTypes.length == 1) {
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
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

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)通过反射的方式找到注解的方法，从上面代码可以看出，注解的方法只能有一个参数，其中checkAdd是根据方法和参数进行验证。然后把找到的存到FindState中。OK，关于注解方法的查找也就这些，下面看回过头来看一下register发具体实现，在subscriberMethodFinder.findSubscriberMethods(subscriberClass)方法查找之后，然后进行遍历

## subscribe

```java
    // Must be called in synchronized block
    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
            subscriptionsByEventType.put(eventType, subscriptions);
        } else {
            if (subscriptions.contains(newSubscription)) {
                throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                        + eventType);
            }
        }

        int size = subscriptions.size();
        for (int i = 0; i <= size; i++) {
            if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
                subscriptions.add(i, newSubscription);
                break;
            }
        }

        List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
        if (subscribedEvents == null) {
            subscribedEvents = new ArrayList<>();
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);

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

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)代码比较简单，这里要记住一下subscriptionsByEventType字段，为什么在register之前post会收不到消息，而在register之前postSticky确能收到消息。因为这里会把当前订阅的方法存入到subscriptionsByEventType中，post的时候如果还没有register，那么subscriptionsByEventType就会为空，当然收不到消息，而postSticky是粘性事件，会保存在stickyEvents中，在register的时候还可以在触发。看一下上面代码的第31行，如果之前发送的是粘性事件，也就是postSticky，那么这里就会执行下面的checkPostStickyEventToSubscription方法。上面的boolean eventInheritance = true;默认值为true。checkPostStickyEventToSubscription方法会调用postToSubscription，来看一下postToSubscription方法

## postToSubscription

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

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)这里就是上面说的几种线程中的操作，看一下MAIN和MAIN_ORDERED。我们以MAIN线程为例继续看，来看一下invokeSubscriber方法

## invokeSubscriber

```java
    void invokeSubscriber(Subscription subscription, Object event) {
        try {
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)这回是彻底明白了，找到订阅的方法，然后通过反射进行调用。这里只是在register中的调用，他只能调用sticky的事件。下面看一下最主要的两个方法post和postSticky

## postSticky

```cpp
    /**
     * Posts the given event to the event bus and holds on to the event (because it is sticky). The most recent sticky
     * event of an event's type is kept in memory for future access by subscribers using {@link Subscribe#sticky()}.
     */
    public void postSticky(Object event) {
        synchronized (stickyEvents) {
            stickyEvents.put(event.getClass(), event);
        }
        // Should be posted after it is putted, in case the subscriber wants to remove immediately
        post(event);
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)我们看到postSticky调用了post，和post唯一的区别就是他保存了event对象，保存在stickyEvents中，所以postSticky的事件可以在register之前调用原理就在这，把当前订阅的类保存在stickyEvents中，然后register的时候就可以调用，而post没有保存，所以register的时候自然没法触发，这里要注意在上面分析的subscribe方法中，我们知道他只能触发sticky的事件，我们接着往下看post方法

## post

```java
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

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)这里就不在过多介绍，主要看一下postSingleEvent方法

## postSingleEvent

```java
    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
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

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)接着看postSingleEventForEventType方法

## postSingleEventForEventType

```java
    private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
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

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)这里我们先看一下postToSubscription，就是上面刚分析的，就不在说了，这里我们来看一下subscriptionsByEventType，我们上面分析的subscriptionsByEventType是在register的时候才会把订阅的事件保存，如果在register之前调用post和postSticky方法，那么这里subscriptionsByEventType返回的自然是空，所以也就不会执行下面的代码了，但postSticky不同，虽然他不能执行，但它把订阅的对象保存在了stickyEvents中，在register的时候就会触发了。下面再来说说上面遗留的问题，上面说道subscriberInfoIndexes中取值的时候的问题，这个字段是在SubscriberMethodFinder构造方法中带过来的，而SubscriberMethodFinder是在EventBus类的构造函数中初始化的，而EventBus的构造函数传入的是EventBusBuilder，使用的是建造者模式，且subscriberInfoIndexes默认为空，我们看一下他传值的方法

## addIndex

```java
    /** Adds an index generated by EventBus' annotation preprocessor. */
    public EventBusBuilder addIndex(SubscriberInfoIndex index) {
        if (subscriberInfoIndexes == null) {
            subscriberInfoIndexes = new ArrayList<>();
        }
        subscriberInfoIndexes.add(index);
        return this;
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)这个很简单，实际上还可以使用一种更加高效的方法，自动为我们生成一个MyEventBusIndex类，它里面会包含我们注解的方法。具体实现是在app的defaultConfig中添加下面代码

## javaCompileOptions

```java
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [ eventBusIndex : 'com.example.myapp.MyEventBusIndex' ]
            }
        }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)然后在dependencies中添加下面代码

```java
    implementation 'org.greenrobot:eventbus:3.1.1'
    annotationProcessor 'org.greenrobot:eventbus-annotation-processor:3.1.1'
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)就会自动为我们生成一个类，具体位置如下，                      

 ![](/img/blog/2018/20180705163004151.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

这里我写了两个类，每个类都写了两个注解的方法

```java
    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onReceiveEvent(String event) {
        Log.d("wld_____", "FirstActivity：onReceiveEvent1：" + event);
    }

    @Subscribe(threadMode = ThreadMode.MAIN, sticky = true)
    public void onReceiveStickyEvent(String event) {
        Log.d("wld_____", "FirstActivity：onReceiveStickyEvent2：" + event);
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)和

```java
    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onReceiveEvent(String event) {
        Log.d("wld_____", "SecondActivity：onReceiveEvent：" + event);
    }

    @Subscribe(threadMode = ThreadMode.MAIN, sticky = true)
    public void onReceiveStickyEvent(String event) {
        Log.d("wld_____", "SecondActivity：onReceiveStickyEvent：" + event);
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)我们来看一下生成的MyEventBusIndex类

## MyEventBusIndex

```java
package com.example.myapp;

import org.greenrobot.eventbus.meta.SimpleSubscriberInfo;
import org.greenrobot.eventbus.meta.SubscriberMethodInfo;
import org.greenrobot.eventbus.meta.SubscriberInfo;
import org.greenrobot.eventbus.meta.SubscriberInfoIndex;

import org.greenrobot.eventbus.ThreadMode;

import java.util.HashMap;
import java.util.Map;

/** This class is generated by EventBus, do not edit. */
public class MyEventBusIndex implements SubscriberInfoIndex {
    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;

    static {
        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();

        putIndex(new SimpleSubscriberInfo(yiquan.xianquan.com.myapplication.SecondActivity.class, true,
                new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onReceiveEvent", String.class, ThreadMode.MAIN),
            new SubscriberMethodInfo("onReceiveStickyEvent", String.class, ThreadMode.MAIN, 0, true),
        }));

        putIndex(new SimpleSubscriberInfo(yiquan.xianquan.com.myapplication.FirstActivity.class, true,
                new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onReceiveEvent", String.class, ThreadMode.MAIN),
            new SubscriberMethodInfo("onReceiveStickyEvent", String.class, ThreadMode.MAIN, 0, true),
        }));

    }

    private static void putIndex(SubscriberInfo info) {
        SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);
    }

    @Override
    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
        if (info != null) {
            return info;
        } else {
            return null;
        }
    }
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)至于他的使用，可以这样

```java
        EventBus.builder().addIndex(new MyEventBusIndex()).installDefaultEventBus();
        EventBus eventBus = EventBus.getDefault();
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)这里最好把EventBus.builder().addIndex(new MyEventBusIndex()).installDefaultEventBus();放到Application中，只初始化一次，如果多次初始化会直接抛异常，我们看一下源码

## installDefaultEventBus

```java
    /**
     * Installs the default EventBus returned by {@link EventBus#getDefault()} using this builders' values. Must be
     * done only once before the first usage of the default EventBus.
     *
     * @throws EventBusException if there's already a default EventBus instance in place
     */
    public EventBus installDefaultEventBus() {
        synchronized (EventBus.class) {
            if (EventBus.defaultInstance != null) {
                throw new EventBusException("Default instance already exists." +
                        " It may be only set once before it's used the first time to ensure consistent behavior.");
            }
            EventBus.defaultInstance = build();
            return EventBus.defaultInstance;
        }
    }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)所以他只能初始化一次。OK，EventBus的原理基本已经分析完毕，下面来看一下具体使用。下面有两个类FirstActivity和SecondActivity，我们暂且标记为A和B

## FirstActivity

```java
public class FirstActivity extends AppCompatActivity {
    private Button button1;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_first);
        EventBus.getDefault().register(this);
        button1 = findViewById(R.id.button1);
        button1.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                EventBus.getDefault().postSticky("111111");
                startActivity(new Intent(FirstActivity.this, SecondActivity.class));
            }
        });
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onReceiveEvent(String event) {
        Log.d("wld_____", "FirstActivity：onReceiveEvent：" + event);
    }

    @Subscribe(threadMode = ThreadMode.MAIN, sticky = true)
    public void onReceiveStickyEvent(String event) {
        Log.d("wld_____", "FirstActivity：onReceiveStickyEvent：" + event);
    }


    @Override
    protected void onDestroy() {
        super.onDestroy();
        EventBus.getDefault().unregister(this);
    }
}
```

## ![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)SecondActivity

```java
public class SecondActivity extends AppCompatActivity {
    private Button button2;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_second);
        EventBus.getDefault().register(this);
        button2 = findViewById(R.id.button2);
        button2.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                finish();
            }
        });
    }

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onReceiveEvent(String event) {
        Log.d("wld_____", "SecondActivity：onReceiveEvent：" + event);
    }

    @Subscribe(threadMode = ThreadMode.MAIN, sticky = true)
    public void onReceiveStickyEvent(String event) {
        Log.d("wld_____", "SecondActivity：onReceiveStickyEvent：" + event);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        EventBus.getDefault().unregister(this);
    }
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)运行一下，看一下打印的log

![](/img/blog/2018/20180705170642495.png)

结果显然是正确的，因为在A和B中都是先注册，所以会获得他们注解的方法，当在A中发送消息的时候A的两个方法都是可以接收到消息的，但在B中由于B还没有启动，所以当B启动的时候只能接收到sticky注册的方法。改一下代码再看一下

```java
                EventBus.getDefault().post("111111");
                startActivity(new Intent(FirstActivity.this, SecondActivity.class));
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)看一下打印log，

![](/img/blog/2018/2018070517143569.png)

我们发现只有A的两个方法执行了，B的方法一个也没执行，这个也很好理解，因为B还没有注册就开始发送消息，所以收不到。再来改一下代码看看，调整一下顺序

```java
                startActivity(new Intent(FirstActivity.this, SecondActivity.class));
                EventBus.getDefault().post("111111");
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)看一下打印log

![](/img/blog/2018/20180705171724407.png)

一样B不会打印，这是因为Activity的启动是耗时的，而B还没启动就开始发送消息，自然是接收不到的，我们再改一下，延迟30毫秒在发送，30毫秒的时间Activity应该完全启动了吧，我们看一下

```java
                startActivity(new Intent(FirstActivity.this, SecondActivity.class));
                new Handler().postDelayed(new Runnable() {
                    @Override
                    public void run() {
                        EventBus.getDefault().post("111111");
                    }
                }, 30);
            }
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)再来看一下打印log

![](/img/blog/2018/20180705172353957.png)

看到没，A和B的两个注解的方法都执行了，这是因为延迟之后A和B都已经启动了，但A的onDestroy还没有执行，所以两个类的注解方法都会执行的。OK，到这里EventBus的原理就已经分析完了。