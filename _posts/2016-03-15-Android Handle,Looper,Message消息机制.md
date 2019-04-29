---
layout: post
title: "Android Handle,Looper,Message消息机制"
subtitle: 'Android Handle,Looper,Message消息机制'
author: "山大王"
header-style: text
catalog: true
tags:
  - 源码
  - android
---
> “非学无以广才，非志无以成学。”
	--诸葛亮

## 正文

我们知道在Android中更新UI都是在主线程中，而操作一些耗时的任务则需要在子线程中，如果存在多个线程共同更新UI，可能会导致页面显示混乱，所以在Android中不允许多线程来共同操作UI，只允许在主线程中更新，下面我们就分析一下Android的消息机制，我们首先要了解这几个类：Handler，Message，Looper，MessageQueue。除了Handler以外，其他的都是final类型，我们来先看一下Handler类的源码，在初始化的时候有这样一段代码

## Handler

```
    public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
 
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

我们看到上面2-9行，打印的是一个警告，我们知道在Android中如果是匿名内部类，或者是一个内部成员类，且不是静态的，可能会出现内存泄漏，因为默认情况下它是持有外部类的引用的，在Handler中一般会伴随着一些耗时的操作，如果外部类退出，可能会导致一直持有外部引用不能被回收，导致内存泄漏，内存泄漏不是ANR，但会影响性能。我们可以这样修改，当持有的外部类被回收之后就不在处理。

## MyHandler

```
	static class MyHandler extends Handler {
		WeakReference<Activity> mActivityReference;
 
		MyHandler(Activity activity) {
			mActivityReference = new WeakReference<Activity>(activity);
		}
 
		@Override
		public void handleMessage(Message msg) {
			final Activity activity = mActivityReference.get();
			if (activity != null) {
				// do something
			}
		}
	}
```

我们继续看上面Handler初始化的第11行的代码，获取looper对象，然后再在looper中获取MessageQueue的实例mQueue，MessageQueue是消息队列，存储消息的，我们还看到上面有一个异常RuntimeException，提示没有调用Looper.prepare()方法，这个异常可能很多人都见过比较熟悉，因为如果我们在子线程中处理Handler消息的时候不调用这个方法，Looper.myLooper()就会返回为空，就会报上面的异常，但是在主线程中我们不需要调上面的方法，因为在主线程中已经默认的为我们调用了，我们在前面<a href="https://androidboke.com/2016/03/15/Android-setContentView%E6%96%B9%E6%B3%95%E8%A7%A3%E6%9E%90-%E4%B8%80" target="_blank">Android setContentView方法解析（一）</a>中大致提到过，他是在ActivityThread的main方法中调用的，我们看一下

## main

```
    public static void main(String[] args) {
        SamplingProfilerIntegration.start();
 
        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);
 
        Environment.initForCurrentUser();
 
        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());
 
        Security.addProvider(new AndroidKeyStoreProvider());
 
        Process.setArgV0("<pre-initialized>");
 
        Looper.prepareMainLooper();
 
        ActivityThread thread = new ActivityThread();
        thread.attach(false);
 
        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }
 
        AsyncTask.init();
 
        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }
 
        Looper.loop();
 
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

我们看到上面18和34行，系统已经默认的为我们调用了，所以我们在主线程中处理消息的时候是不需要再调用的，我们看一下prepareMainLooper的源码

## prepareMainLooper

```
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

我们再来看一下Looper.myLooper()这个方法

```
    static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
 
    public static Looper myLooper() {
        return sThreadLocal.get();
    }
```

ThreadLocal是存储数据的一个类，专门存储到当前线程，且各个线程之间互不影响，这个以后再分析，在这里存储的是Looper对象，这个Looper就是在prepare方法中初始化的，我们看一下

## prepare

```
    public static void prepare() {
        prepare(true);
    }
 
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
```

初始化之后保存到ThreadLocal中，且ThreadLocal中之前不能存在Looper，否则会报上面的异常，因为要保证每个thread中只有一个Looper，如果一个thread存在多个looper就会创建多个MessageQueue，消息发送的时候可能就会出现不知道该发送到哪个消息队列的情况，所以他只能有一个。ThreadLocal保存在当前的线程中，以后取的时候就从当前的线程中取，上面我们还看到prepare方法和prepareMainLooper是有区别的，主要在传的参数quitAllowed，我们看一下

```
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

在Looper中初始化了一个MessageQueue对象

```
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
```

mQuitAllowed判断是否可以退出消息循环，我们看一下MessageQueue的quit方法

## quit

```
    void quit(boolean safe) {
        if (!mQuitAllowed) {
            throw new RuntimeException("Main thread not allowed to quit.");
        }
 
        synchronized (this) {
            if (mQuitting) {
                return;
            }
            mQuitting = true;
 
            if (safe) {
                removeAllFutureMessagesLocked();
            } else {
                removeAllMessagesLocked();
            }
 
            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }
```

看到没，子线程的looper和主线程的looper主要区别就在这，子线程是可以退出消息循环的，但主线程不可以，如果退出会报上面的异常，我们在看一下ActivityThread的main方法的最后一行throw new RuntimeException("Main thread loop unexpectedly exited");如果不明原因退出，也会报上面异常。  
我们再来看一下消息的发送，在Handler中消息的发送有两种方式，一种是send一种是post，这两种方式被重构了好多，通过Message.obtain()方法获取Message对象，在Android中获取Message对象推荐的是使用obtain，不推荐使用new，我们来看一下Message的obtain的源码，

## obtain

```
    public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
```

我们看到在Message中维护了一个消息池，所有的Message对象都是从消息池中取的，如果没有就new一个，因为Message中有一个Message类型的next对象，是以一种单链表的形式维护的，它的最大容量是50，是在消息发送完回收的时候存的，我们来看一下

```
    private static final int MAX_POOL_SIZE = 50;
	…………………………
    public void recycle() {
        clearForRecycle();
 
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```

所以如果我们获取Message实例的时候最好调用它的obtain方法而不是new一个。我们接着看上面的，在上面的两种发送消息的时候最终调用的都是同一个方法，我们看一下

## sendMessageAtTime

```
    public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }
```

这个消息队列mQueue就是刚才Handler初始化的时候从Looper中获取的，我们上面分析过，Looper是从当前的线程中的ThreadLocal获取的，所以这个消息队列也是属于当前线程的，我们再来看一下enqueueMessage方法的源码

## enqueueMessage

```
    private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```

我们注意上面的msg.target = this，因为Message有一个Handler类型target，所以在这里赋值给了他，待会取出消息处理的时候也是通过target来发送的，我们具体来看一下MessageQueue中的enqueueMessage方法的源码，

```
    boolean enqueueMessage(Message msg, long when) {
        if (msg.isInUse()) {
            throw new AndroidRuntimeException(msg + " This message is already in use.");
        }
        if (msg.target == null) {
            throw new AndroidRuntimeException("Message must have a target.");
        }
 
        synchronized (this) {
            if (mQuitting) {
                RuntimeException e = new RuntimeException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w("MessageQueue", e.getMessage(), e);
                return false;
            }
 
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
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

我们看一下20-24行，如果消息队列中没有消息，或者时间为0，或者时间小于当前的消息队列头的时间，就会把它加入到消息队列的前面，这里的消息是按时间存放的，因为在发送消息的时候有好几个重构的方法，其中就有延迟发送，时间越往后的越排在后面。在发送的时候还有一个sendMessageAtFrontOfQueue方法是把消息放到队列的最前面的，我们接着再看30-39行，通过不断的循环找到合适的位置，找的时候一般判断最后的是否为空，如果为空就表示到了队列的最后了，就是最后的位置，如果不为空就和当前Message的时间对比，如果时间比后一个短，说明要比他先执行，位置就在他前面，然后在41-42行，把消息存进去。

然后在最后调用Looper的loop方法执行循环，我们看一下loop的源码

## loop

```
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;
 
        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();
 
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
 
            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }
 
            msg.target.dispatchMessage(msg);
 
            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }
 
            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }
 
            msg.recycle();
        }
    }
```

我们看第14行，通过消息队列取出消息，这个取出消息调用的是MessageQueue的next()方法，这个我们待会再分析，我们知道他取出消息就好了，我们在看27行，就是消息的处理，再看44行，就是消息的回收，在上面我们提过，回收之后加入到消息池中，如果下次用到直接从里面取就是了，但是加入的时候把数据都清空了，我们看到Message的recycle方法中调用了这样一个方法clearForRecycle，我们看一下

## clearForRecycle

```
    /*package*/ void clearForRecycle() {
        flags = 0;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        when = 0;
        target = null;
        callback = null;
        data = null;
    }
```

就是把数据全部清空之后然后加入到消息池中，我们还看上面的27行的代码msg.target.dispatchMessage(msg);我们刚才在上面提过在发送消息的时候把Handler赋给了Message的target，所以这里再从Message中取出target，调用它的dispatchMessage方法来处理消息，我们看一下

## Callback

```
    public interface Callback {
        public boolean handleMessage(Message msg);
    }
    
    /**
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) {
    }
    
    /**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }
```

我们看到Handler中有个handleMessage方法，返回类型为void，还有一个接口Callback，里面也有这样一个方法，返回类型为boolean，在调用dispatchMessage方法处理消息的时候首先调用的是Message的Callback，因为Message的Callback其实就是一个Runnable，我们来看一下handleCallback方法，

```
    private static void handleCallback(Message message) {
        message.callback.run();
    }
```

很简单，就一行代码它所以在上面的处理消息的时候首先判断是否是post发送，如果是就调用它的run方法，如果不是，说明是send方式发送，就判断他自己的mCallback是否为空，如果不为空就调用他自己的接口，如果返回true就表示事件被消耗，就不在往下执行，否则就调用自己的handleMessage方法。好了，Android的消息机制到这来已经分析完了，上面还遗留了一个问题，就是从消息队列中获取消息的方法，我们来看一下

## next

```
    Message next() {
        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
 
            // We can assume mPtr != 0 because the loop is obviously still running.
            // The looper will not call this method after the loop quits.
            nativePollOnce(mPtr, nextPollTimeoutMillis);
 
            synchronized (this) {
                // Try to retrieve the next message.  Return if found.
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                if (msg != null && msg.target == null) {
                    // Stalled by a barrier.  Find the next asynchronous message in the queue.
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // Next message is not ready.  Set a timeout to wake up when it is ready.
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
                        if (false) Log.v("MessageQueue", "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }
 
                // Process the quit message now that all pending messages have been handled.
                if (mQuitting) {
                    dispose();
                    return null;
                }
 
                // If first time idle, then get the number of idlers to run.
                // Idle handles only run if the queue is empty or if the first message
                // in the queue (possibly a barrier) is due to be handled in the future.
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // No idle handlers to run.  Loop and wait some more.
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
                    Log.wtf("MessageQueue", "IdleHandler threw exception", t);
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

代码量比较多，我们找关键的看,  通过不断的循环来取出消息，第11行是延迟等待的时间， 28行如果执行的时间没到就计算延迟的时间nextPollTimeoutMillis，30到40行如果取出消息则返回，并把它置为markInUse,表示在使用。48行如果退出就返回为null，否则可能就会一直等待消息的到来，上面的pendingIdleHandlerCount如果小于等于0就表示消息已经被处理完了，在等待更多的消息。

OK，目前为止Android的消息机制就分析完了，在上面我们说如果在主线程中处理消息的时候在ActivityThread的main方法中已经为我们初始化了looper，我们不需要再初始化，但如果在子线程中处理消息的时候就必须要手动初始化looper，因为子线程默认是没有初始化的，在主线程处理消息大家早就已经很熟悉了，在这里就不在演示，下面我们就为大家演示一下在子线程处理消息要注意的事项，我们把子线程写在一个click事件中。

```
_ll.setOnClickListener(new OnClickListener() {
	@Override
	public void onClick(View v) {
		new Thread() {
			public void run() {
				new Handler() {
					public void handleMessage(Message msg) {
						Log.d("wld", "Android源码");
					}
				}.sendEmptyMessage(0);
			}
		}.start();
	}
});
```

我们点击直接crash，我们看一下打印的log

![](/img/blog/2016/20160416211603604.png)

这个就是最上面第一段代码的那个异常，因为默认情况下子线程是没有初始化looper的，所以直接报错，我们再修改一下

```
_ll.setOnClickListener(new OnClickListener() {
	@Override
	public void onClick(View v) {
		new Thread() {
			public void run() {
				Looper.prepare();
				new Handler() {
					public void handleMessage(Message msg) {
						Log.d("wld", "Android源码");
					}
				}.sendEmptyMessage(0);
				Looper.loop();
			}
		}.start();
	}
});
```

我们再运行一下，发现没有问题，并且log也能正常打印，因为在子线程中我们已经初始化了looper，所以就没有报错。如果我们不想在子线程中调用Looper的方法，我们也可以修改一下Handler

## MyHandler

```
public class MyHandler extends Handler {
 
	public MyHandler() {
		super(getMyLooper());
	}
 
	private static Looper getMyLooper() {
		/*从子线程中获取looper，如果我们在子线程中没有调用Looper.prepare();则会返回为
		 * null，如果为空我们就获取主线程的looper，主线程的looper肯定不会为null的，因
		 * 为主线程的looper在程序启动的时候就已经在ActivityThread的main方法中初始化了。*/
		Looper mLooper = Looper.myLooper();
		if (mLooper == null)
			mLooper = Looper.getMainLooper();
		return mLooper;
	}
}
```

通过修改我们直接把looper传进去了，就不需要再创建了，我们再来修改一下之前的代码

```
_ll.setOnClickListener(new OnClickListener() {
	@Override
	public void onClick(View v) {
		new Thread() {
			public void run() {
				new MyHandler() {
					public void handleMessage(Message msg) {
						Log.d("wld", "Android源码");
					}
				}.sendEmptyMessage(0);
			}
		}.start();
	}
});
```

我们看到这次并没有报错，并且还正常的打印了log。  
下面我就画一个流程图来详细说明一下Android的消息机制。

![](/img/blog/2016/20160417223643746.jpg)

可能大家会对上面的图感到有点困惑，Handler既然能发送Message，为什么不直接发送到Handler的dispatchMessage方法中自己处理，如果真的这样做是没有任何意义的，如果这样我们还不如直接在子线程直接处理，因为我们一般都是在子线程中发送消息，而在主线程中处理消息，因为这样做可以操作UI，像上面那种直接在子线程中发送消息又在子线程中处理消息的毕竟不多，但不管怎样在子线程中是不能操作UI的，但Toast是个奇葩，在子线程中通过修改是可以正常弹出Toast的，这个我们以后讲到Toast源码的时候在为大家分析

OK，Android的消息机制到现在已经分析完毕。