# Android的消息机制
### 概述
说到Android的消息机制，我们肯定会想到Handler。是的，Android的消息机制主要是指Handler的运行机制以及Handler所附带的MessageQueue和Looper的工作过程。当我们工作的时候我们只要接触到Handler就可以了。

Android的消息机制主要是指handlr的运行机制，handler的运行需要底层的messagequeue和looper来支撑。messagequeue就是消息队列，也就是说它内部存储了一组消息，以队列的形式对外提供插入和删除的工作，虽然叫队列，但其内部是用单链表的数据结构来存储消息列表的。looper意旨循环，在这也就说消息循环。由于messagequeue只是一个消息的存储单元。并不能去处理消息，送一我们就需要looper去填补。looper会以无限循环的形式去遍历messagequeue，如果有消息就取出。否则一直等待着。

提到线程，我们就需要提到threadlocal，threadlocal并不是线程，但它可以在每个线程中存储数据。后面我们会分析threadlocal的源码来解释它是如何在多个线程中存储数据的。

### 疑问
handler可以说是面试常客了，现在列出几个常问的面试题：

- handler消息机制流程
- 什么是ANR？什么情况下会出现ANR？
- looper.loop()为什么不阻塞主线程
- handler、messagequeue、looper三者的关系

### threadlocal工作原理
前面我们说到threadlocal可以在多个线程中存储数据，那么它到底是怎么做到的呢？我们只要弄清楚get和set方法就能理解它的工作原理。
首先看看Threadlocal的set方法：

```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```
通过源码我们可以发现这边是通过一个map来存储数据的，而它的key就是当前的线程，也就是我们前面说的它是怎么做多个线程中存储数据的。

下面我们看看get方法，看看它是如何获取这些线程的数据的。

```java
 public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
    
     private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
    
      protected T initialValue() {
        return null;
    }
```
这边我们可以看出它是通过当前线程去寻找对应的map，通过map在去寻找对应的value。如果map不存在，默认返回一个null。

### MessageQueue的工作原理
messagequeue是以一个单链表去维护一个消息队列。提高删除插入消息等操作的性能。插入和读取的对应方法分别为enqueueMessage和next。前者的作用是往消息队列中插入一条消息，而后者的作用则是从消息队列中取出一条消息并将其从消息队列中移除。下面我们看看enqueueMessage的源码：

```java
  boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
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
上述代码主要是单链表的插入，然后通过对应的标记，设置是否需要唤醒。下面看看next方法的实现：

```java
 Message next() {
        // Return here if the message loop has already quit and been disposed.
        // This can happen if the application tries to restart a looper after quit
        // which is not supported.
        final long ptr = mPtr;
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // -1 only during first iteration
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }

            nativePollOnce(ptr, nextPollTimeoutMillis);

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
                        if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // No more messages.
                    nextPollTimeoutMillis = -1;
                }

            ···
    }
```
这边就是一个无限循环的方法，去找出一个异步消息将其返回并且移除，如果没有消息，就一直阻塞在这里。

### Looper的工作原理
looper在消息机制中扮演着消息循环的角色。准确的说它会不停的从messagequeue中查看是否有新消息，如果有新消息就会处理掉。否则一直阻塞在那。我们可以通过looper.prepare()即可为当前线程创建一个looper，然后通过looper.loop()来开启消息循环队列。

looper除了prepare方法之外，还提供了prepareMainLooer方法，这个方法主要是给主线程创建looper使用的。其本质还是通过prepare来实现的。由于主线程的looper比较特殊，looper提供了getMainLooper方法来获取主线程的looper。当然looper也是可以退出的。它提供了quit和quitSafely来退出一个looper。前者会直接退出，而后者只是给了一个标记，当队列中消息处理完毕后，才会安全的退出。looper退出后，handler的sendmessage方法会返回false。

looper中最重要的一个方法是loop。只有调用了loop，才会真正的启动循环系统。下面看看它是如何实现的：

```java
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

            final long slowDispatchThresholdMs = me.mSlowDispatchThresholdMs;

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            final long start = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
            final long end;
            try {
                msg.target.dispatchMessage(msg);
                end = (slowDispatchThresholdMs == 0) ? 0 : SystemClock.uptimeMillis();
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            if (slowDispatchThresholdMs > 0) {
                final long time = end - start;
                if (time > slowDispatchThresholdMs) {
                    Slog.w(TAG, "Dispatch took " + time + "ms on "
                            + Thread.currentThread().getName() + ", h=" +
                            msg.target + " cb=" + msg.callback + " msg=" + msg.what);
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
此方法也很好理解，loop方法是一个死循环，唯一跳出循环的方法是messagequeue的next方法返回null，也就是looper的quit方法被调用后，会将messagequeue来通知消息队列退出。此时，它的next方法就会返回null，也就是说looper必须退出，否则它将一直循环下去。如果messagequeue的next方法返回了新消息，looper就会处理这条消息，它会将这条消息交给handler去处理。

### handler工作原理
handler主要是用来发送消息和接受消息的。消息的发送可以通过post去发送，不过最后还是通过send去实现的。下面我们看代码：

```java
 public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }
    
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
    
     private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
可以发现，handler发送消息仅仅是向消息队列中插入一条消息，然后messagequeue的next方法就会将消息返回给looper，looper收到消息后又把它交给handler处理。也就是会调用dispatchMessage方法。具体代码如下：

```java
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
handler处理消息的过程如下：
首先，检查message的callback是否为null，不为null就通过handlecallback来处理消息，message的callback是一个runnable对象，实际上就是handler的post方法所传的runnable参数。
其次，检查mcallback是否为null，不为null就调用mcallback的handlemessage方法来处理消息。
最后，调用handler的hanldemessage方法来处理消息。

### 主线程的消息循环
Android的主线程就是ActivityThread。主线程的入口是main，在mian方法中我们通过looper.prepareMainLooper()来创建主线程的looper以及messagequeue，并通过looper.loop()去开启主线程的消息循环。代码如下：

```java
 public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

        // CloseGuard defaults to true and can be quite spammy.  We
        // disable it here, but selectively enable it later (via
        // StrictMode) on debug builds, but using DropBox, not logs.
        CloseGuard.setEnabled(false);

        Environment.initForCurrentUser();

        // Set the reporter for event logging in libcore
        EventLogger.setReporter(new EventLoggingReporter());

        // Make sure TrustedCertificateStore looks in the right place for CA certificates
        final File configDir = Environment.getUserConfigDirectory(UserHandle.myUserId());
        TrustedCertificateStore.setDefaultUserDirectory(configDir);

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false);

        if (sMainThreadHandler == null) {
            sMainThreadHandler = thread.getHandler();
        }

        if (false) {
            Looper.myLooper().setMessageLogging(new
                    LogPrinter(Log.DEBUG, "ActivityThread"));
        }

        // End of event ActivityThreadMain.
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
        Looper.loop();

        throw new RuntimeException("Main thread loop unexpectedly exited");
    }

```
主线程的消息循环开启之后，activitythread还需要一个handler来和消息队列进行交互，也就是activitythread.H。其内部定义了一组消息的类型，主要是四大组件的停止和启动。

activitythread通过applicationthread和ams进行进程间通信。ams以进程间通信的方式完成activitythread的请求后会回调applicationthread中的binder方法。然后applicationthread会向H发送消息，H收到消息后会将applicationthread的逻辑切换到activitythread中去执行，即切换到主线程执行。这个过程就是主线程的消息循环模型。


