---
title: Handler源码解析
date: 2018-05-26 10:36:58
tags: handler
---

#### 前言

> Handler是Android消息机制的上层接口，在Android多线程编程中可以说是不可或缺的角色，最常见的应用就是通过Handler在子线程更新UI。

<!-- more -->

Handler 机制的实现原理依赖于Message、MessageQueue 和 Looper 这三个类。

#### Handler

构造方法：

- ```java
  public Handler()  // 默认构造方法，与当前线程及其Looper实例绑定
  ```

- ```java
  public Handler(Callback callback)	// 与当前线程及其Looper实例绑定，同时传入一个回调函数
  ```

- ```java
  public Handler(Looper looper)	// 与指定的Looper实例绑定
  ```

- ```java
  public Handler(Looper looper, Callback callback)	// 与指定的Looper实例绑定，同事传入一个回调函数
  ```

- ```java
  public Handler(boolean async)	// 是否是异步的
  ```

不过，最终构造方法的代码还是以下两个之一：

> 没有指定Looper对象，则需要自己去获取当前线程的Looper对象

```java
public Handler(Callback callback, boolean async) {
    if (FIND_POTENTIAL_LEAKS) {
        final Class<? extends Handler> klass = getClass();
        if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                (klass.getModifiers() & Modifier.STATIC) == 0) {
            Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                klass.getCanonicalName());
        }
    }
	// 获取当前线程的Looper对象
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

> 创建了指定的Looper对象，直接进行赋值操作

```java
public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    mCallback = callback;
    mAsynchronous = async;
}
```

- `mQueue = mLooper.mQueue;`接着又从上一步获取的当前线程的Looper实例中获取了其保存的MessageQueue（消息队列），这样就保证了handler的实例与我们Looper实例中MessageQueue关联上了。
- `mCallback = callback;`保存了创建Handler实例时覆写的callback回调方法。
- `mAsynchronous = async;`保存了创建Handler实例时对异步的设置。

#### Looper

> Looper 是线程用以运行消息循环的类。默认情况下，线程并没有与之关联的Looper实例，需要通过在线程中调用`Looper.prepare()`方法获取，并通过`Looper.loop()`无限循环获取并分发MessageQueue中的消息。

典型用法如下：

```java
class LooperThread extends Thread {
      public Handler mHandler;

      public void run() {
          Looper.prepare();

          mHandler = new Handler() {
              public void handleMessage(Message msg) {
                  // process incoming messages here
              }
          };

          Looper.loop();
      }
  }
```

在Handler的构造方法中有获取Looper实例的代码：

```java
	// 获取当前线程的Looper对象
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
```

可以看到，如果获取的Looper实例是null，会抛出异常，说线程内部没有调用Looper.prepare()方法。但是平时我们在UI线程创建Handler实例的时候，并没有调用该方法，为什么没有抛出异常呢，其实这是因为在Activity的启动代码中已经在当前的UI线程调用了`Looper.prepare()`，同时也调用了`Looper.loop()`方法。<br>[Android程序入口AndroidThread](https://www.baidu.com)<br>在Looper方法内部：

```java
/**
 * Return the Looper object associated with the current thread.  Returns
 * null if the calling thread is not associated with a Looper.
 */
public static @Nullable Looper myLooper() {
    return sThreadLocal.get();
}
```

#### ThreadLocal

> sThreadLocal 是一个`ThreadLocal`实例，`ThreadLocal`是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，数据存储后，只有在指定线程中可以获取到存储的数据，而其它线程则无法获取。	

从ThreadLocal的set()和get()方法可看出，所操作的对象就是当前线程的内部变量threadLocals，该变量为ThreadLocal的静态内部类ThreadLocalMap。

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null)
            return (T)e.value;
    }
    return setInitialValue();
}

public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

这里呢，我看的是Android 7.0，也就是SDK版本25的源码，如果是别的版本的源码可能会有所不同，比如之前我看的是19的，用的是一个ThreadLocal.Values的静态内部类。

##### Looper.prepare()

首先是Looper.prepare()方法：

```java
 /** Initialize the current thread as a looper.
  * This gives you a chance to create handlers that then reference
  * this looper, before actually starting the loop. Be sure to call
  * {@link #loop()} after calling this method, and end it by calling
  * {@link #quit()}.
  */
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

可以看到，首先会判断sThreadLocal是否为空，不为空则会将新建的Looper实例放进sThreadLocal对象中，Looper的构造函数内部实现如下：

```java
private Looper(boolean quitAllowed) {
    mQueue = new MessageQueue(quitAllowed);
    mThread = Thread.currentThread();
}
```

##### Looper.loop()

获取当前线程的实例，如存在则接着获取该Looper实例的MessageQueue实例，开启死循环，不断的从MessageQueue的next方法获取消息，而**next方法是一个阻塞操作**，没有消息则一直阻塞，当有消息则通过`msg.target.dispatchMessage(msg);`处理，这里的msg.target就是发送消息的Handler对象，这将在下面分析Message得到印证。

```java
/**
 * Run the message queue in this thread. Be sure to call
 * {@link #quit()} to end the loop.
 */
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
        final Printer logging = me.mLogging;
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                    msg.callback + ": " + msg.what);
        }

        final long traceTag = me.mTraceTag;
        if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
        }
        try {
            msg.target.dispatchMessage(msg);
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }

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

        msg.recycleUnchecked();
    }
}
```

#### Message

> Message封装了需要传递的消息，并且本身可以作为链表的一个节点。

更多了解Message，这里不赘述了<br>

[Android message](Https://www.baidu.com)<br>

#### MessageQueue

> MessageQueue是用来存放Message的集合，并由Looper实例来分发里面的Message对象。虽然叫消息队列，但是它的内部存储结构并不是真正的队列，而是采用单链表的数据结构来存储消息列表。

##### 加入消息队列

- 好了说了这么多，那么消息是怎么加入到消息队列MessageQueue中的呢

```java
public final boolean sendMessage(Message msg)
{
       return sendMessageDelayed(msg, 0);
}
```

- 那么现在具体看看平时使用的sendMessage(Message msg)方法具体是如何实现的。

```java
public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
```

- 其中的`sendMessageDelayed`方法

```java
public final boolean sendMessageDelayed(Message msg, long delayMillis)
{
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
```

- 最后还是调用了`sendMessageAtTime`方法

```java
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

而`enqueueMessage`方法中

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

在进入方法的第一行，enqueueMessage中首先为meg.target赋值为this，也就是把当前的handler作为msg的target属性。最终会调用queue的enqueueMessage的方法，同时可以看出最终调用了`queue.enqueueMessage`方法，那么queue又是什么呢，其实就是MessageQueue的对象实例，也就是说handler发出的消息，最终会保存到消息队列中去。

- 接下来再来看`dispatchMessage`方法：

```java
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

就会发现其中的`handleMessage(msg);`方法中是空的，其实就是需要我们覆写的最终消息处理方法。

#### 最后总结

- 首先Looper.prepare()在本线程保存一个Looper实例，然后在该实例中创建并保存了一个MessageQueue对象，同时因为Looper.prepare()在一个线程中只能调用一次，所以MessageQueue在一个线程只会存在一个。
- Looper.loop()方法会让当前线程进入一个无限循环，不断的从MessageQueue的实例中读取消息，然后回调`msg.target.dispatchMessage(msg)`方法。
- 而target就是Handler的实例，那么再看Handler的构造函数，其首先得到了当前线程保存的Looper实例，进而又从Looper实例中获取了保存的MessageQueue实例，与其相关联。
- 接下来Handler的sendMessage方法，会给msg的target赋值为handler本身，然后加入到MessageQueue中。
- 那么回过头来看Looper.loop()获取到了消息调用的dispatchMessage(msg)方法其实最终调用了我们覆写的handleMessage方法。

