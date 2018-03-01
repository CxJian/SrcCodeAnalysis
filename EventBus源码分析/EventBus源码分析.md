## EventBus源码分析笔记

 EventBus源码版本：**org.greenrobot:eventbus:3.1.1**

### 1、创建EventBus源码分析

一般情况下，使用EventBus为我们提供的默认实例对象即可，先从该对象进行分析

	EventBus.getDefault()

源码如下：
	
	/** Convenience singleton for apps using a process-wide EventBus instance. */
    public static EventBus getDefault() {
        if (defaultInstance == null) {
            synchronized (EventBus.class) {
                if (defaultInstance == null) {
                    defaultInstance = new EventBus();
                }
            }
        }
        return defaultInstance;
    }

首先，默认提供给我的EventBus是一个单例对象。

    EventBus(EventBusBuilder builder) {
        logger = builder.getLogger();
        subscriptionsByEventType = new HashMap<>();
        typesBySubscriber = new HashMap<>();
        stickyEvents = new ConcurrentHashMap<>();
        mainThreadSupport = builder.getMainThreadSupport();
        mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
        backgroundPoster = new BackgroundPoster(this);
        asyncPoster = new AsyncPoster(this);
        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);
        logSubscriberExceptions = builder.logSubscriberExceptions;
        logNoSubscriberMessages = builder.logNoSubscriberMessages;
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        throwSubscriberException = builder.throwSubscriberException;
        eventInheritance = builder.eventInheritance;
        executorService = builder.executorService;
    }

构造方法中几个比较重要的成员变量：

* subscriptionsByEventType:以event(即事件类型类)为key，以订阅列表(Subscription)为value，事件发送之后，在这里寻找订阅者,而Subscription又是一个CopyOnWriteArrayList，这是一个线程安全的容器。Subscription是一个封装类，封装了订阅者、订阅方法这两个类。

* typesBySubscriber：以订阅者类为key，以event事件类为value，在进行register或unregister操作的时候，会操作这个map。

* stickyEvents:保存的是粘性事件


### 2、注册、注销EventBus源码分析


#### （1）注册源码分析

 	EventBus.getDefault().register(this)
	

**register()**

    public void register(Object subscriber) {
        Class<?> subscriberClass = subscriber.getClass();
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                subscribe(subscriber, subscriberMethod);
            }
        }
    }	

通过订阅方法查找器，寻找当前订阅者的订阅方法，并进行订阅。

首先我们需要搞清楚订阅方法查找器做了什么？（subscriberMethodFinder）

**findSubscriberMethods()**

    List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        if (subscriberMethods != null) {
            return subscriberMethods;
        }

        if (ignoreGeneratedIndex) {
			//通过反射获取满足条件的订阅方法
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

**findUsingReflection()**

    private List<SubscriberMethod> findUsingReflection(Class<?> subscriberClass) {
        FindState findState = prepareFindState();

		//initForSubscriber()方法会将我们传递的订阅者class对象赋值给findState.class成员变量
        findState.initForSubscriber(subscriberClass);
        while (findState.clazz != null) {
			//核心代码看这个方法
            findUsingReflectionInSingleClass(findState);
            findState.moveToSuperclass();
        }
        return getMethodsAndRelease(findState);
    }

**findUsingReflectionInSingleClass()**

	 private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        try {
			//反射当前订阅者的方法
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            methods = findState.clazz.getMethods();
            findState.skipSuperClasses = true;
        }
		//遍历获取满足条件的方法
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

							//最后将满足条件的方法，封装为一个订阅方法对象，并添加到findState的订阅方法集合中
                            findState.subscriberMethods.add(
								new SubscriberMethod(method, eventType, threadMode,
                            		subscribeAnnotation.priority(),
									subscribeAnnotation.sticky())
							);
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

**getMethodsAndRelease()**

最后让我们回到findUsingReflection()方法最后调用的getMethodsAndRelease()方法中：

    private List<SubscriberMethod> getMethodsAndRelease(FindState findState) {
        List<SubscriberMethod> subscriberMethods = 
			new ArrayList<>(findState.subscriberMethods);
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

可以看到将findState里存储的订阅方法集返回了回去，到此subscriberMethodFinder订阅方法查找器的工作就完成了，也就是说:**订阅方法查找器通过反射的方式校验当前的订阅者里是否包含订阅方法，并返回这些订阅方法。**
	

**subscribe()**

最后注册的源码就剩下遍历这些订阅方法，并进行订阅这一块了，继续看源码：

    private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
        Class<?> eventType = subscriberMethod.eventType;
        Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
        CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions == null) {
            subscriptions = new CopyOnWriteArrayList<>();
			//绑定 订阅事件 和 订阅类型
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
			//绑定 订阅者 和 订阅类型
            typesBySubscriber.put(subscriber, subscribedEvents);
        }
        subscribedEvents.add(eventType);
		

		//粘性事件相关
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


代码看起来比较多，实际只是在处理订阅方法、订阅事件集、订阅类型、订阅者这几个对象的关系，并把相关对象放到了之前EventBus构造方法中初始化时的那几个容器中（subscriptionsByEventType，typesBySubscriber，stickyEvents），到此注册工作也就完成了。

---


#### （2）注销源码分析

再来看看注销代码又做了什么？


	EventBus.getDefault().unregister(this)

**unregister()**

    public synchronized void unregister(Object subscriber) {
		//通过订阅者获取订阅类型
        List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
        if (subscribedTypes != null) {
            for (Class<?> eventType : subscribedTypes) {
				//遍历订阅类型，依次进行解绑操作
                unsubscribeByEventType(subscriber, eventType);
            }
			//在当前的订阅者中，清除掉该类型
            typesBySubscriber.remove(subscriber);
        } else {
            logger.log(Level.WARNING, "Subscriber to unregister was not registered before: " + subscriber.getClass());
        }
    }

通过语义上，我们可以大致看出以上注释部分代码含义，关键点还是在unsubscribeByEventType()方法中，我们继续往下看：

**unsubscribeByEventType()**

    private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
		//获取指定类型的事件集合
        List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
        if (subscriptions != null) {
            int size = subscriptions.size();
            for (int i = 0; i < size; i++) {
				//遍历事件集合
                Subscription subscription = subscriptions.get(i);
                if (subscription.subscriber == subscriber) {
					//清除掉事件集合里指定订阅者的相关事件
                    subscription.active = false;
                    subscriptions.remove(i);
                    i--;
                    size--;
                }
            }
        }
    }

可以看出，其实这里就是清除掉所有事件里指定类型的事件集合满足对应订阅者的相关事件。（说白了就是与注册进行对应，移除subscriptionsByEventType中需要注销的订阅者引用，避免内存泄漏）

到此，注册和注销的源码就分析完了。


### 3、订阅事件源码分析

	EventBus.getDefault().post(TestEvent())

**post()**

    public void post(Object event) {
		//获取当前的发送状态
        PostingThreadState postingState = currentPostingThreadState.get();

		//将当前事件添加到消息队列中
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
					//发送事件
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }


**postSingleEvent()**

    private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {	 //默认true
			//查询所有的该类型的事件类型字节码对象（包括父类,详细介绍见下断代码）
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
            int countTypes = eventTypes.size();
			//遍历所有满足该类型的字节码对象
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
				//接下来就是postSingleEventForEventType()方法了，在下方分析
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

**lookupAllEventTypes()**

该方法会将传入的事件类型字节码对象作为key，该对象以及该对象的所有父类字节码对象集合作为value，存储到eventTypesCache中，并最后只返回包含这个包含所有字节码对象的集合

	private static List<Class<?>> lookupAllEventTypes(Class<?> eventClass) {
        synchronized (eventTypesCache) {
            List<Class<?>> eventTypes = eventTypesCache.get(eventClass);
            if (eventTypes == null) {
                eventTypes = new ArrayList<>();
                Class<?> clazz = eventClass;
                while (clazz != null) {
					//添加自己到集合中
                    eventTypes.add(clazz);	
					//添加所实现的所以接口到集合中，该方法里面是递归操作
                    addInterfaces(eventTypes, clazz.getInterfaces());
					//切换到父类，准备重新循环
                    clazz = clazz.getSuperclass();
                }
                eventTypesCache.put(eventClass, eventTypes);
            }
            return eventTypes;
        }
    }

**postSingleEventForEventType()**

    private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
			//拿到该类型当前的所有事件
            subscriptions = subscriptionsByEventType.get(eventClass);
        }

        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted = false;
                try {
					//触发订阅
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

**postToSubscription（）**

该方法用于将触发事件调度到对应线程，最后执行之前反射后的method.invoke()方法

    private void postToSubscription(Subscription subscription, Object event, boolean isMainThread) {
        switch (subscription.subscriberMethod.threadMode) {
            case POSTING:
				//执行method里的invoke()方法，详细代码往下翻
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


**invokeSubscriber()**

	void invokeSubscriber(Subscription subscription, Object event) {
        try {
			//通过之前注册时反射后的method触发订阅
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }

到这里，一个基本的发送消息以及触发订阅者的流程也就分析完了。